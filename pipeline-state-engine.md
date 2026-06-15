# 流水线步骤在调度引擎中的状态变更机制

## 1. 状态定义

### 1.1 状态枚举 (`server/model/const.go:55-68)

```go
type StatusValue string

const (
    StatusSkipped  StatusValue = "skipped"   // 按条件跳过
    StatusPending  StatusValue = "pending"   // 等待执行
    StatusRunning  StatusValue = "running"   // 正在运行
    StatusSuccess  StatusValue = "success"   // 成功完成
    StatusFailure  StatusValue = "failure"   // 执行失败
    StatusKilled   StatusValue = "killed"    // 用户终止
    StatusCanceled StatusValue = "canceled"  // 未启动即取消
    StatusError    StatusValue = "error"     // 系统错误
    StatusBlocked  StatusValue = "blocked"   // 等待审批
    StatusDeclined StatusValue = "declined"  // 审批拒绝
    StatusCreated  StatusValue = "created"   // 已创建（内部状态）
)
```

### 1.2 步骤执行状态 (`pipeline/backend/types/state.go:18-32)

```go
type State struct {
    Started   int64  // 开始时间
    ExitCode  int    // 退出码
    Exited    bool   // 是否已退出
    Skipped   bool   // 是否被跳过
    OOMKilled bool  // 是否OOM终止
    Error     error  // 错误信息
}
```

---

## 2. 完整的状态变更流程

### 2.1 第一阶段：任务入队（服务端）

#### 2.1.1 触发条件

**代码位置：`server/pipeline/create.go:35-150`

1. **流水线创建流程：

```
Create()
├─ 设置 pipeline.Status = StatusCreated  (create.go:66)
├─ 解析流水线配置，创建 pipelineItems
├─ 检查是否需要审批
│  ├─ 需要审批 → StatusBlocked，不入队
│  └─ 无需审批 → updatePipelinePending() → StatusPending
└─ start()
   └─ queuePipeline() → PushAtOnce()
```

2. **入队触发条件** (`server/pipeline/queue.go:29-61`

```go
func queuePipeline(ctx context.Context, repo *model.Repo, activePipeline *model.Pipeline, pipelineItems []*builder.Item) error {
    for _, item := range pipelineItems {
        task := &model.Task{
            ID: fmt.Sprint(item.Workflow.ID),
            // ...
            Dependencies: getTaskDependencies(...),
            DepStatus: make(map[string]model.StatusValue),
        }
        task.Data, _ = json.Marshal(rpc.Workflow{...})
        tasks = append(tasks, task)
    }
    return server.Config.Services.Scheduler.PushAtOnce(ctx, tasks)
}
```

**入队前置条件：
- 流水线配置解析成功
- 无需审批或已通过审批
- 流水线状态不为空

---

### 2.2 第二阶段：任务出队与分配（调度引擎）

#### 2.2.1 调度引擎核心循环

**代码位置：`server/queue/fifo.go:257-287

```go
func (q *fifo) process() {
    for {
        select {
        case <-time.After(processTimeInterval): // 每100ms执行一次
        case <-q.ctx.Done():
            return
        }
        q.Lock()
        if q.paused { /* 暂停检查 */ }
        
        q.resubmitExpiredPipelines()  // 重新提交过期任务
        q.filterWaiting()             // 过滤依赖未满足的任务
        
        // 分配任务给worker
        for pending, worker := q.assignToWorker(); pending != nil && worker != nil; {
            task, _ := pending.Value.(*model.Task)
            task.AgentID = worker.agentID
            delete(q.workers, worker)
            q.pending.Remove(pending)
            
            // 标记为运行中
            q.running[task.ID] = &entry{
                item:     task,
                done:     make(chan bool),
                deadline: time.Now().Add(q.extension),
            }
            worker.channel <- task  // 发送给agent
        }
        q.Unlock()
    }
}
```

#### 2.2.2 出队触发条件

**代码位置：`server/queue/fifo.go:314-336`

```go
func (q *fifo) assignToWorker() (*list.Element, *worker) {
    for element := q.pending.Front(); element != nil; element = element.Next() {
        task, _ := element.Value.(*model.Task)
        
        for worker := range q.workers {
            matched, score := worker.filter(task)
            if matched && score > bestScore {
                bestWorker = worker
                bestScore = score
            }
        }
        if bestWorker != nil {
            return element, bestWorker
        }
    }
    return nil, nil
}
```

**出队前置条件：
1. **依赖满足**：`filterWaiting()` 确保依赖任务移至 `waitingOnDeps` 队列
   - 代码：`server/queue/fifo.go:289-312
   - 检查：`depsInQueue()` 确认依赖任务不在 pending 或 running 中

2. **标签匹配**：agent 的标签过滤函数返回 true
   - agent 通过 `FilterFn` 检查任务标签

3. **队列未暂停**：`q.paused == false`

4. **有空闲 worker**：有 agent 正在 Poll 等待任务

#### 2.2.3 Agent 拉取任务

**代码位置：`server/rpc/rpc.go:63-108

```go
func (s *RPC) Next(c context.Context, agentFilter rpc.Filter) (*rpc.Workflow, error) {
    filterFn := createFilterFunc(agentFilter)
    for {
        task, err := s.scheduler.Poll(c, agent.ID, filterFn)
        if task.ShouldRun() {
            workflow := new(rpc.Workflow)
            json.Unmarshal(task.Data, workflow)
            return workflow, nil
        }
    }
}
```

**代码位置：`agent/runner.go:63-77

```go
func (r *Runner) Run(runnerCtx context.Context) error {
    workflow, err := r.client.Next(runnerCtx, r.filter)
    if workflow == nil { return nil }
    
    // 初始化工作流状态
    state := rpc.WorkflowState{Started: time.Now().Unix()}
    r.client.Init(runnerCtx, workflow.ID, state)  // 触发 StatusRunning
```

---

### 2.3 第三阶段：步骤执行与状态追踪（Agent端）

#### 2.3.1 步骤执行入口

**代码位置：`pipeline/runtime/step.go:34-60

```go
func (r *Runtime) executeStep(runnerCtx context.Context, step *backend_types.Step) error {
    if r.shouldSkipStep(step) {
        return r.traceStep(&backend_types.State{Skipped: true}, nil, step)
    }
    
    // 发送"步骤开始"追踪
    r.traceStep(nil, nil, step)
    
    if step.Detached {
        return r.runDetachedStep(runnerCtx, step)
    }
    return r.runBlockingStep(runnerCtx, step)
}
```

#### 2.3.2 步骤跳过判断

**代码位置：`pipeline/runtime/step.go:65-85

```go
func (r *Runtime) shouldSkipStep(step *backend_types.Step) bool {
    currentErr := r.err.Get()
    
    // 前面有错误且步骤不设置 OnFailure → 跳过
    if currentErr != nil && !step.OnFailure { return true }
    
    // 前面无错误且步骤不设置 OnSuccess → 跳过
    if currentErr == nil && !step.OnSuccess { return true }
    
    return false
}
```

#### 2.3.3 步骤状态追踪（traceStep）

**代码位置：`pipeline/runtime/step.go:267-292

```go
func (r *Runtime) traceStep(processState *backend_types.State, err error, step *backend_types.Step) error {
    s := new(state.State)
    s.Workflow.Started = r.started
    s.CurrStep = step
    s.Workflow.Error = r.err.Get()
    
    switch {
    case processState == nil && err != nil:
        // 步骤启动失败
        s.CurrStepState = backend_types.State{Error: err, Exited: true}
    case processState != nil:
        // 步骤完成
        s.CurrStepState = *processState
    default:
        // 步骤刚启动，CurrStepState 零值
    }
    
    return r.tracer.Trace(s)  // 调用 tracer 上报状态
}
```

#### 2.3.4 Tracer 上报至服务端

**代码位置：`agent/tracer.go:30-60

```go
func (r *Runner) createTracer(...) tracing.TraceFunc {
    return func(state *state.State) error {
        stepState := rpc.StepState{
            StepUUID: state.CurrStep.UUID,
            Exited:   state.CurrStepState.Exited,
            ExitCode: state.CurrStepState.ExitCode,
            Started:  state.CurrStepState.Started,
            Canceled: errors.Is(state.CurrStepState.Error, pipeline_errors.ErrCancel),
            Skipped:  state.CurrStepState.Skipped,
        }
        if state.CurrStepState.Error != nil {
            stepState.Error = state.CurrStepState.Error.Error()
        }
        if state.CurrStepState.Exited {
            stepState.Finished = time.Now().Unix()
        }
        return r.client.Update(ctxMeta, workflow.ID, stepState)
    }
}
```

---

### 2.4 第四阶段：服务端处理步骤状态更新

#### 2.4.1 步骤状态计算

**代码位置：`server/pipeline/step_status.go:31-109

```go
func CalcStepStatus(step model.Step, state rpc.StepState) (*model.Step, cancelPipelineFromStep bool, _ error) {
    switch step.State {
    case model.StatusPending:
        // 1. 跳过处理
        if state.Skipped {
            step.State = model.StatusSkipped
            return &step, false, nil
        }
        // 2. 开始运行
        if state.Finished == 0 {
            step.State = model.StatusRunning
            step.Started = state.Started
        }
        // 3. 直接完成（启动错误）
        if state.Exited || state.Error != "" {
            step.Finished = state.Finished
            step.ExitCode = state.ExitCode
            step.Error = state.Error
            if state.ExitCode == 0 && state.Error == "" {
                step.State = model.StatusSuccess
            } else {
                step.State = model.StatusFailure
                if step.Failure == model.FailureCancel {
                    cancelPipelineFromStep = true
                }
            }
        }
        
    case model.StatusRunning:
        // 检查是否完成
        if state.Exited || state.Error != "" {
            step.Finished = state.Finished
            step.ExitCode = state.ExitCode
            step.Error = state.Error
            if state.ExitCode == 0 && state.Error == "" {
                step.State = model.StatusSuccess
            } else {
                step.State = model.StatusFailure
            }
        }
    }
    
    // 取消处理
    if state.Canceled && step.State != model.StatusKilled {
        step.State = model.StatusKilled
    }
    return &step, cancelPipelineFromStep, nil
}
```

#### 2.4.2 状态切换触发条件汇总

| 当前状态 | 触发条件 | 目标状态 |
|---------|---------|---------|
| **StatusPending** | `state.Skipped == true` | **StatusSkipped** |
| **StatusPending** | `state.Finished == 0` (未完成) | **StatusRunning** |
| **StatusPending** | `state.Exited || state.Error != ""` | **StatusSuccess** / **StatusFailure** |
| **StatusRunning** | `state.Exited || state.Error != ""` | **StatusSuccess** / **StatusFailure** |
| **StatusRunning** | `state.Canceled == true` | **StatusKilled** |

#### 2.4.3 RPC Update 处理流程

**代码位置：`server/rpc/rpc.go:158-226

```go
func (s *RPC) Update(c context.Context, strWorkflowID string, state rpc.StepState) error {
    // 1. 加载 workflow、pipeline、step
    // 2. 检查 agent 权限
    // 3. 检查 workflow 状态是否允许更新
    checkWorkflowAllowsStepUpdate(workflow.State, step, state)
    
    // 4. 更新步骤状态
    pipeline.UpdateStepStatus(c, s.store, step, state)
    
    // 5. 步骤完成时通知 LogStore
    if state.Exited {
        server.Config.Services.LogStore.StepFinished(step)
    }
    
    // 6. 重新加载 workflow 树并通知 pubsub
    currentPipeline.Workflows, _ = s.store.WorkflowGetTree(currentPipeline)
    return s.notify(c, repo, currentPipeline)
}
```

---

### 2.5 第五阶段：工作流完成

#### 2.5.1 Workflow Init（初始化运行状态

**代码位置：`server/rpc/rpc.go:229-293

```go
func (s *RPC) Init(c context.Context, strWorkflowID string, state rpc.WorkflowState) error {
    // 如果 pipeline 还是 StatusPending
    if currentPipeline.Status == model.StatusPending {
        pipeline.UpdateToStatusRunning(s.store, *currentPipeline, state.Started)
    }
    // 更新 workflow 状态为 StatusRunning
    workflow, err = pipeline.UpdateWorkflowStatusToRunning(s.store, *workflow, state)
}
```

#### 2.5.2 Workflow Done（完成）

**代码位置：`server/rpc/rpc.go:296-...

```go
func (s *RPC) Done(c context.Context, strWorkflowID string, state rpc.WorkflowState) error {
    // 完成所有仍在运行的子步骤
    // 计算 workflow 最终状态
    // 更新 pipeline 状态
    s.scheduler.Done(c, strWorkflowID, exitStatus)
    
    // 通知 pubsub
}
```

---

## 3. 依赖状态传播机制

### 3.1 依赖检查

**代码位置：`server/queue/fifo.go:350-367

```go
func (q *fifo) depsInQueue(task *model.Task) bool {
    // 检查 pending 队列中的依赖
    for element := q.pending.Front(); element != nil; element = element.Next() {
        possibleDep, _ := element.Value.(*model.Task)
        for _, dep := range task.Dependencies {
            if possibleDep.ID == dep { return true }
        }
    }
    // 检查 running 队列中的依赖
    for possibleDepID := range q.running {
        if slices.Contains(task.Dependencies, possibleDepID) { return true }
    }
    return false
}
```

### 3.2 依赖状态更新

**代码位置：`server/queue/fifo.go:370-396

```go
func (q *fifo) updateDepStatusInQueue(taskID string, status model.StatusValue) {
    // 更新 pending 队列中依赖该 task 的任务
    for element := q.pending.Front(); element != nil; element = element.Next() {
        pending, _ := element.Value.(*model.Task)
        for _, dep := range pending.Dependencies {
            if taskID == dep {
                pending.DepStatus[dep] = status
            }
        }
    }
    // 同样更新 running 和 waitingOnDeps 队列
}
```

---

## 4. 状态变更时序图

```
服务端                                Agent
  │                                    │
  │ 1. CreatePipeline → StatusCreated        │
  │ 2. queuePipeline → PushAtOnce        │
  │ 3. process() 循环调度             │
  │    ├─ filterWaiting()              │
  │    └─ assignToWorker() → 出队        │
  │ 4. Poll() 等待任务 ←───────────────┤
  │ 5. Next() 返回 workflow             │
  │                                    │ 6. Init() → StatusRunning
  │                                    │ 7. executeStep()
  │                                    │    ├─ shouldSkipStep()?
  │                                    │    ├─ traceStep(started)
  │ Update(stepState) ←──────────────┤
  │ 8. CalcStepStatus()                 │
  │    StatusPending → StatusRunning  │
  │ notify(pubsub)                     │
  │                                    │ 9. runBlockingStep()
  │                                    │    ├─ startStep()
  │                                    │    ├─ completeStep()
  │                                    │    └─ traceStep(finished)
  │ Update(stepState) ←──────────────┤
  │ 10. CalcStepStatus()                │
  │     StatusRunning → StatusSuccess    │
  │ notify(pubsub)                     │
  │                                    │ 11. Done(workflowState)
  │ Done() ←───────────────────────────┤
  │ 12. scheduler.Done() → 移除 running │
  │     updateDepStatusInQueue()        │
  │     触发依赖任务调度                 │
  └────────────────────────────────────┘
```

---

## 5. 关键触发条件总结

### 5.1 任务出队触发条件
1. **时间触发**：每 100ms 执行一次 `process()` 循环 (`fifo.go:59, 260`)
2. **依赖满足**：`depsInQueue()` 返回 false，即依赖任务已完成
3. **标签匹配**：Agent 过滤函数匹配成功
4. **有空闲 Worker**：Agent 正在 `Poll()` 等待
5. **队列未暂停**

### 5.2 步骤状态切换触发条件
1. **Pending → Skipped**：`shouldSkipStep()` 返回 true
   - 前置步骤失败且当前步骤未设置 `OnFailure: true`
   - 前置步骤成功且当前步骤未设置 `OnSuccess: true`
   
2. **Pending → Running**：`traceStep(nil, nil, step)` 被调用（步骤开始）

3. **Pending/Running → Success**：
   - `state.Exited == true` 且 `state.ExitCode == 0` 且 `state.Error == ""`

4. **Pending/Running → Failure**：
   - `state.Exited == true` 且 (`state.ExitCode != 0` 或 `state.Error != ""`

5. **任何状态 → Killed**：`state.Canceled == true`

### 5.3 流水线状态切换触发条件
1. **Created → Blocked**：需要审批
2. **Created/Pending → Running**：第一个 workflow `Init()` 被调用
3. **Running → Success/Failure/Killed**：所有 workflow 完成后根据结果计算

---

## 6. Agent 与 Server 的 gRPC 通信链路

### 6.1 连接建立过程

**完整调用链：`cmd/agent/core/agent.go:52-321**

```
run()
├─ agent_rpc.Dial(authCtx, cfg)
│  ├─ grpc.NewClient(serverAddr) → authConn       // 认证连接
│  ├─ NewAuthGrpcClient(authConn, token, agentID)
│  ├─ NewAuthInterceptor(authCtx, authClient)      // JWT 自动续签
│  └─ grpc.NewClient(serverAddr,                  // 业务连接
│       WithUnaryInterceptor(authInterceptor),
│       WithStreamInterceptor(authInterceptor))
│
├─ agent_rpc.NewGrpcClient(ctx, mainConn,          // 封装 Peer 接口
│       SetConnectionRetryTimeout(retryTimeout))
│
├─ client.Version(ctx)                             // 版本校验
├─ client.RegisterAgent(ctx, info)                 // 注册 agent
│
└─ for i := range maxWorkflows {                   // 每个槽位一个 runner
     go runner.Run(agentCtx)                       // 持续轮询
  }
```

### 6.2 gRPC 服务定义

**代码位置：`rpc/proto/woodpecker.proto:26-38**

```protobuf
service Woodpecker {
  rpc Next            (NextRequest)          returns (NextResponse) {}   // 长轮询：拉取下一个 workflow
  rpc Init            (InitRequest)          returns (Empty) {}         // 通知 workflow 已启动
  rpc Wait            (WaitRequest)          returns (WaitResponse) {}  // 长轮询：等待取消信号
  rpc Done            (DoneRequest)          returns (Empty) {}         // 通知 workflow 已完成
  rpc Extend          (ExtendRequest)        returns (Empty) {}         // 续期 lease
  rpc Update          (UpdateRequest)        returns (Empty) {}         // 上报步骤状态
  rpc Log             (LogRequest)           returns (Empty) {}         // 批量发送日志
  rpc RegisterAgent   (RegisterAgentRequest) returns (RegisterAgentResponse) {}
  rpc UnregisterAgent (Empty)                returns (Empty) {}
  rpc ReportHealth    (ReportHealthRequest)  returns (Empty) {}
}
```

### 6.3 双层连接架构：Auth + Main

**代码位置：`agent/rpc/dial.go:68-112**

Agent 建立两条 gRPC 连接：

1. **Auth 连接**（`authConn`）：仅用于 `WoodpeckerAuth.Auth` 获取 JWT token
2. **Main 连接**（`mainConn`）：承载所有业务 RPC，通过 `AuthInterceptor` 自动注入 JWT

```
┌───────────────────────────────────────────────────┐
│  Agent                                            │
│                                                   │
│  authConn ──→ WoodpeckerAuth.Auth() ──→ JWT      │
│                        │                          │
│  AuthInterceptor ◄─────┘  (定时刷新 token)        │
│       │                                           │
│  mainConn ──→ [Unary/Stream Interceptor] ──→     │
│       │        Next / Init / Wait / Done / ...    │
└───────┼───────────────────────────────────────────┘
        │ gRPC
┌───────┼───────────────────────────────────────────┐
│  Server                                          │
│  grpc.NewServer(                                 │
│    StreamInterceptor(authorizer.Stream),          │
│    UnaryInterceptor(authorizer.Unary),            │
│    KeepaliveEnforcementPolicy(...)                │
│  )                                               │
│  ├─ WoodpeckerAuthServer                         │
│  └─ WoodpeckerServer ──→ RPC struct              │
│       ├─ Next()   → scheduler.Poll()             │
│       ├─ Wait()   → scheduler.Wait()             │
│       ├─ Init()   → UpdateToStatusRunning()      │
│       ├─ Done()   → scheduler.Done()/Error()     │
│       ├─ Update() → UpdateStepStatus()           │
│       └─ Log()    → LogStore.LogAppend()         │
└───────────────────────────────────────────────────┘
```

**Server 端注册流程：`server/rpc/serve.go:55-88**

```go
func Serve(ctx context.Context, cfg ServeConfig) error {
    jwtManager := NewJWTManager(cfg.JWTSecret)
    authorizer := NewAuthorizer(jwtManager)

    grpcServer := grpc.NewServer(
        grpc.StreamInterceptor(authorizer.StreamInterceptor),
        grpc.UnaryInterceptor(authorizer.UnaryInterceptor),
        grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime: cfg.KeepaliveMinTime,
        }),
    )
    proto.RegisterWoodpeckerServer(grpcServer, NewWoodpeckerServer(...))
    proto.RegisterWoodpeckerAuthServer(grpcServer, NewWoodpeckerAuthServer(...))

    grpcServer.Serve(cfg.Listener)
}
```

### 6.4 Agent 主循环与 gRPC 调用顺序

**代码位置：`agent/runner.go:63-217 + `cmd/agent/core/agent.go:267-310**

每个 runner 槽位持续执行以下循环：

```
for {
    runner.Run(agentCtx)
    │
    ├─ 1. client.Next(ctx, filter)          ← 长轮询，阻塞直到获得 workflow
    │     └─ gRPC Next → scheduler.Poll() → 阻塞等待任务出队
    │
    ├─ 2. client.Init(ctx, workflowID, state)  ← 通知 server workflow 已启动
    │     └─ gRPC Init → UpdateToStatusRunning()
    │
    ├─ 3. 并发启动三个后台协程：
    │     ├─ go client.Wait(ctx, workflowID)     ← 长轮询，监听取消信号
    │     │     └─ gRPC Wait → scheduler.Wait() → 阻塞直到完成或取消
    │     │
    │     ├─ go client.Extend(ctx, workflowID)   ← 每 TaskTimeout/3 续期一次
    │     │     └─ gRPC Extend → scheduler.Extend() → 更新 deadline
    │     │
    │     └─ pipeline_runtime.New(...).Run()     ← 执行工作流
    │        ├─ traceStep(started) → client.Update()
    │        ├─ traceStep(running) → client.Update()
    │        └─ traceStep(finished) → client.Update()
    │
    ├─ 4. client.Done(ctx, workflowID, state)   ← 通知 server workflow 已完成
    │     └─ gRPC Done → scheduler.Done()/Error()
    │
    └─ 如果 ErrConnectionLost → 关闭整个 agent
       其他错误 → 等待 5s 后继续轮询
}
```

### 6.5 gRPC 消息流与状态变更对应关系

| gRPC 方法 | 方向 | 触发时机 | 引发的状态变更 |
|-----------|------|---------|--------------|
| **Next** | Agent → Server | runner 循环开头 | 无（仅出队，状态仍为 Pending） |
| **Init** | Agent → Server | workflow 开始执行 | Pipeline: Pending→Running, Workflow: Pending→Running |
| **Wait** | Agent → Server | 并发监听 | 无（阻塞等待 Done/Cancel） |
| **Extend** | Agent → Server | 每 TaskTimeout/3 | 仅更新 queue deadline，无数据库变更 |
| **Update** | Agent → Server | 每次 traceStep | Step: Pending→Running→Success/Failure/Killed |
| **Done** | Agent → Server | workflow 执行结束 | Workflow→终态, Pipeline 汇总, Queue 清理 |
| **Log** | Agent → Server | 步骤日志产出 | 无状态变更，写入 LogStore + pubsub |

---

## 7. 队列出队的 Retry / Backoff 机制

### 7.1 三层 retry 架构总览

```
┌────────────────────────────────────────────────────────┐
│  Layer 1: Agent gRPC Retry (backoff/v5)                │
│  位置: agent/rpc/client_grpc.go retryRPC()             │
│  作用: 网络层重试，处理 gRPC 连接抖动                     │
│  参数: InitialInterval=10ms, MaxInterval=10s            │
│        MaxElapsedTime=connectionRetryTimeout(可配置)     │
├────────────────────────────────────────────────────────┤
│  Layer 2: Queue Lease / Expiry (FIFO 内存队列)          │
│  位置: server/queue/fifo.go resubmitExpiredPipelines() │
│  作用: 检测 agent 失联，将过期 running 任务重新入队        │
│  参数: TaskTimeout=1min (deadline 续期)                  │
├────────────────────────────────────────────────────────┤
│  Layer 3: Persistent Queue (数据库持久化)                │
│  位置: server/queue/persistent.go WithTaskStore()       │
│  作用: Server 重启后从数据库恢复未完成任务                  │
└────────────────────────────────────────────────────────┘
```

### 7.2 Layer 1: gRPC 指数退避重试

**代码位置：`agent/rpc/client_grpc.go:98-166**

```go
func (c *client) retryOpts(op string) []backoff.RetryOption {
    b := backoff.NewExponentialBackOff()
    b.MaxInterval = 10 * time.Second
    b.InitialInterval = 10 * time.Millisecond
    return []backoff.RetryOption{
        backoff.WithBackOff(b),
        backoff.WithMaxElapsedTime(c.connectionRetryTimeout),
        backoff.WithNotify(notify),
    }
}
```

**错误分类策略 (`client_grpc.go:171-193`)**：

```go
func classifyRPCErr(ctx context.Context, err error) error {
    switch status.Code(err) {
    case codes.Canceled:
        if ctx.Err() != nil {
            return backoff.Permanent(ctx.Err())  // 自己的 ctx 取消 → 永久终止
        }
        return backoff.Permanent(err)            // 服务端取消 → 永久终止
    case codes.Aborted, codes.DataLoss,
         codes.DeadlineExceeded, codes.Internal,
         codes.Unavailable:
        return err                               // 可重试错误 → 继续重试
    default:
        return backoff.Permanent(err)            // 其他错误 → 永久终止
    }
}
```

**可重试 vs 永久终止的 gRPC 状态码**：

| gRPC Code | 行为 | 说明 |
|-----------|------|------|
| `Canceled` | **Permanent** | 除非是自身 ctx 取消，否则不重试 |
| `Unavailable` | **Retry** | 服务端不可达，核心重试场景 |
| `DeadlineExceeded` | **Retry** | 超时，可能是暂时的 |
| `Aborted` | **Retry** | 事务冲突，可重试 |
| `DataLoss` | **Retry** | 数据丢失，尝试恢复 |
| `Internal` | **Retry** | 服务端内部错误，可能恢复 |
| `NotFound/PermissionDenied/...` | **Permanent** | 不可恢复，停止重试 |

**重试终止条件 (`client_grpc.go:143-166`)**：

```go
func retryRPC[T any](ctx context.Context, c *client, opName string, op backoff.Operation[T]) (T, error) {
    res, err := backoff.Retry(ctx, op, c.retryOpts(opName)...)
    
    // 1. ctx 被取消 → 返回零值+nil（吞掉错误，由上层判断）
    if ctxErr := context.Cause(ctx); ctxErr != nil && errors.Is(err, ctxErr) {
        return zero, nil
    }
    // 2. connectionRetryTimeout 耗尽且仍断连 → ErrConnectionLost
    if errors.Is(err, errNotConnected) {
        return zero, ErrConnectionLost
    }
    // 3. 其他永久错误 → 直接返回
    return zero, err
}
```

**ErrConnectionLost 的处理 (`cmd/agent/core/agent.go:278-283`)**：

```go
for {
    if err := runner.Run(agentCtx); err != nil {
        if errors.Is(err, agent_rpc.ErrConnectionLost) {
            ctxCancel(err)   // 关闭整个 agent
            return nil
        }
        // 其他错误 → 等待 5s 后继续
        time.Sleep(time.Second * 5)
    }
}
```

### 7.3 Layer 2: Queue Lease 与过期重提交

**核心配置 (`shared/constant/constant.go:40-41`)**：

```go
var TaskTimeout = time.Minute  // running 任务的默认 lease 时长
```

**Lease 续期机制（双向保障）**：

**Agent 端**（`agent/runner.go:134-148`）：
```go
go func() {
    for {
        select {
        case <-workflowCtx.Done():
            return
        case <-time.After(constant.TaskTimeout / 3):  // 每 20s 续期一次
            r.client.Extend(workflowCtx, workflow.ID)
        }
    }
}()
```

**Server 端**（`server/queue/fifo.go:186-200`）：
```go
func (q *fifo) Extend(_ context.Context, agentID int64, taskID string) error {
    state, ok := q.running[taskID]
    if ok && state.item.AgentID == agentID {
        state.deadline = time.Now().Add(q.extension)  // 重置 deadline
        return nil
    }
    return ErrNotFound
}
```

**过期重提交（`server/queue/fifo.go:338-348`）**：

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            log.Info().Msgf("queue: resubmitting expired task %s", taskID)
            taskState.error = ErrTaskExpired
            q.pending.PushFront(taskState.item)   // 重新放回 pending 队列前端
            delete(q.running, taskID)
            close(taskState.done)                  // 通知 Wait() 返回
        }
    }
}
```

**过期重提交流程图**：

```
                    Agent 正常                        Agent 失联
                    ─────────                       ─────────────
Task 出队          │                                │
→ running[id]      │                                │
deadline=now+1m    │                                │
                    │                                │
    ┌─────┐        │                                │
    │20s  │ Extend()│                                │
    │     ├───────►│ deadline = now + 1m             │
    │     │        │                                │ deadline 到期
    │40s  │ Extend()│                                │ (无 Extend)
    │     ├───────►│ deadline = now + 1m             │ ↓
    │     │        │                                │ resubmitExpired()
    │60s  │ Extend()│                                │ pending.PushFront(task)
    │     ├───────►│ deadline = now + 1m             │ delete(running, id)
    └─────┘        │                                │ close(entry.done)
                   │                                │
                   │                                │ → Wait() 返回 ErrTaskExpired
                   │                                │ → 新 Agent Poll() 可获取此任务
```

### 7.4 Layer 3: Persistent Queue（服务端重启恢复）

**代码位置：`server/queue/persistent.go:32-111`

```go
func WithTaskStore(ctx context.Context, q Queue, s store.Store) Queue {
    tasks, _ := s.TaskList()          // 从数据库加载所有未完成任务
    q.PushAtOnce(ctx, tasks)          // 重新推入内存队列
    return &persistentQueue{q, s}
}
```

**持久化包装器的关键方法**：

| 方法 | 行为 | 持久化操作 |
|------|------|-----------|
| `PushAtOnce` | 入队时同步写入数据库 | `store.TaskInsert(task)` |
| `Poll` | 出队时从数据库删除 | `store.TaskDelete(task.ID)` |
| `Error` | 错误完成时从数据库删除 | `store.TaskDelete(id)` |
| `ErrorAtOnce` | 批量错误时批量删除 | `store.TaskDelete(id)` × N |

**恢复场景**：
1. Server 正常运行时，内存队列（`fifo`）是主存储，数据库是备份
2. Server 重启时，`WithTaskStore()` 从数据库加载所有 task，重建内存队列
3. Agent 重新 Poll 后获取这些 task，实现无缝恢复

---

## 8. Canceled 状态清理的完整调用栈

### 8.1 取消触发入口

取消流水线有三种触发来源：

#### 入口 1: 用户 API 请求

**代码位置：`server/api/pipeline.go:475-500**

```
HTTP DELETE /api/repos/{repo}/pipelines/{number}/cancel
    │
    └─ CancelPipeline(c *gin.Context)
        └─ pipeline.Cancel(ctx, forge, store, repo, user, pl, &model.CancelInfo{
               CanceledByUser: user.Login,
           })
```

#### 入口 2: 步骤失败触发取消

**代码位置：`server/pipeline/step_status.go:129-152`

```
UpdateStepStatus(ctx, store, step, state)
    │
    ├─ CalcStepStatus() → cancelPipelineFromStep = true
    │  (当 step.Failure == model.FailureCancel)
    │
    └─ cancelPipelineFromStep(ctx, store, step)
        └─ Cancel(ctx, forge, store, repo, repoUser, pipeline, &model.CancelInfo{
               CanceledByStep: step.Name,
           })
```

#### 入口 3: 新流水线替换旧流水线

**代码位置：`server/pipeline/cancel.go:100-159`

```
start() → cancelPreviousPipelines(ctx, forge, store, pipeline, repo, user)
    │
    └─ for _, active := range activeBuilds {
           if pipelineNeedsCancel(active) {
               Cancel(ctx, forge, store, repo, user, active, &model.CancelInfo{
                   SupersededBy: pipeline.Number,
               })
           }
       }
```

### 8.2 Cancel 函数完整执行流程

**代码位置：`server/pipeline/cancel.go:32-98**

```
Cancel(ctx, forge, store, repo, user, pipeline, cancelInfo)
│
│  前置检查: pipeline.Status 必须是 Running/Pending/Blocked
│
├─ 1. 队列层面：批量驱逐任务
│     workflowsToCancel = [所有 Running/Pending 的 workflow ID]
│     server.Config.Services.Scheduler.ErrorAtOnce(ctx, workflowsToCancel, queue.ErrCancel)
│     │
│     └─ fifo.ErrorAtOnce() → fifo.finished(ids, StatusKilled, ErrCancel)
│         ├─ 对 running 中的任务: close(entry.done) → Wait() 返回 ErrCancel
│         ├─ 对 pending/waiting 中的任务: removeFromPendingAndWaiting()
│         └─ updateDepStatusInQueue(id, StatusKilled) → 传播依赖状态
│
├─ 2. 数据库层面：更新 workflow/step 状态
│     for _, workflow := range workflows {
│         if workflow.State == StatusPending {
│             UpdateWorkflowToStatusSkipped(store, workflow)
│             → workflow.State = StatusSkipped
│         }
│         for _, step := range workflow.Children {
│             if step.State == StatusPending {
│                 UpdateStepToStatusSkipped(store, step, 0, StatusCanceled)
│                 → step.State = StatusCanceled
│             }
│         }
│     }
│
├─ 3. 数据库层面：更新 pipeline 状态
│     if hasPendingOnly {
│         plState = StatusCanceled    // 全部是 pending → canceled
│     } else {
│         plState = StatusKilled      // 有 running → killed
│     }
│     UpdateToStatusKilled(store, pipeline, cancelInfo, plState)
│     → pipeline.Status = plState
│     → pipeline.Finished = now
│     → pipeline.CancelInfo = cancelInfo
│
├─ 4. 通知 Forge
│     updatePipelineStatus(ctx, forge, killedPipeline, repo, user)
│
└─ 5. 通知 PubSub
      publishToTopic(ctx, killedPipeline, repo)
```

### 8.3 Agent 端的取消检测与清理

**代码位置：`agent/runner.go:117-131`

```
// 并发监听取消信号
go func() {
    canceled, err := r.client.Wait(workflowCtx, workflow.ID)
    if canceled {
        cancelWorkflowCtx(pipeline_errors.ErrCancel)  // 取消 workflow 上下文
    }
}()
```

**取消信号在 Agent 端的传播路径**：

```
Server: Cancel() → ErrorAtOnce() → close(entry.done)
    │
    │  (Wait() 返回 ErrCancel)
    ↓
Agent: client.Wait() → canceled = true
    │
    ├─ cancelWorkflowCtx(pipeline_errors.ErrCancel)
    │     │
    │     ├─ workflowCtx 被取消
    │     │     │
    │     │     ├─ completeStep() 检测到 context.Canceled
    │     │     │     → waitState.Error = pipeline_errors.ErrCancel
    │     │     │     → traceStep() → Canceled=true
    │     │     │
    │     │     └─ workflow.Run() 返回 ErrCancel
    │     │
    │     └─ extend 协程退出（workflowCtx.Done()）
    │
    └─ Runner 主循环:
          state.Canceled = true
          state.Error = "cancel"
          client.Done(doneCtx, workflow.ID, state)
              │
              ↓
          Server: RPC.Done()
              ├─ workflow.Failing() → scheduler.Error()
              ├─ !state.Canceled → scheduler.Done(id, workflow.State)
              └─ state.Canceled:
                    ├─ Started > 0 → scheduler.Done(id, StatusKilled)
                    └─ Started == 0 → scheduler.Done(id, StatusCanceled)
```

### 8.4 Workflow Done 的取消分支

**代码位置：`server/rpc/rpc.go:356-369**

```go
var queueErr error
if !state.Canceled {
    if workflow.Failing() {
        queueErr = s.scheduler.Error(c, strWorkflowID, ...)
    } else {
        queueErr = s.scheduler.Done(c, strWorkflowID, workflow.State)
    }
} else {
    if workflow.Started > 0 {
        queueErr = s.scheduler.Done(c, strWorkflowID, model.StatusKilled)
    } else {
        queueErr = s.scheduler.Done(c, strWorkflowID, model.StatusCanceled)
    }
}
```

**关键区分**：
- `StatusKilled`：已启动的 workflow 被取消（Running → Killed）
- `StatusCanceled`：未启动的 workflow 被取消（Pending → Canceled）

### 8.5 取消后仍需完成的清理工作

**代码位置：`server/rpc/rpc.go:536-548**

```go
func (s *RPC) completeChildrenIfParentCompleted(completedWorkflow *model.Workflow, finished int64) {
    for _, c := range completedWorkflow.Children {
        if c.Running() {
            // 仍在运行的子步骤（如 service 容器）标记为 Killed
            pipeline.UpdateStepToStatusSkipped(s.store, *c, finished, model.StatusKilled)
        }
    }
}
```

**日志流清理**（`server/rpc/rpc.go:388-396`）：

```go
go func() {
    for _, step := range workflow.Children {
        if step.State != model.StatusSkipped {
            s.logger.Close(c, step.ID)   // 关闭日志流
        }
    }
}()
```

### 8.6 Step 状态上报的安全检查

**代码位置：`server/rpc/sanitize.go:97-119**

```go
func checkWorkflowAllowsStepUpdate(workflowState model.StatusValue, step *model.Step, state rpc.StepState) error {
    // 活跃状态的 workflow 允许任何 step 更新
    if isActiveState(workflowState) {   // Created/Pending/Running
        return nil
    }
    // 已终止的 workflow 只允许 step 转换到终态
    newStep, _, err := pipeline.CalcStepStatus(*step, state)
    if isDoneState(newStep.State) {     // Success/Failure/Killed/Canceled/Skipped/Error/Declined
        return nil
    }
    return ErrAgentIllegalWorkflowReRunStateChange
}
```

**这确保了**：即使 workflow 已被 cancel，agent 仍然可以上报步骤的最终状态（如 `Killed`），但绝不允许将已终止的步骤重新变回 `Running`。

---

## 9. 状态合并与优先级机制

### 9.1 状态优先级

**代码位置：`server/pipeline/status.go:20-44**

```go
var statusPriorityOrder = []model.StatusValue{
    model.StatusDeclined,    // 0 - 最高优先级
    model.StatusBlocked,     // 1
    model.StatusCreated,     // 2
    model.StatusError,       // 3
    model.StatusKilled,      // 4
    model.StatusCanceled,    // 5
    model.StatusRunning,     // 6
    model.StatusPending,     // 7
    model.StatusFailure,     // 8
    model.StatusSuccess,     // 9
    model.StatusSkipped,     // 10 - 最低优先级
}
```

### 9.2 MergeStatusValues 规则

**代码位置：`server/pipeline/status.go:56-69**

```go
func MergeStatusValues(s, t model.StatusValue) model.StatusValue {
    // 两个都是 canceled → canceled
    if s == StatusCanceled && t == StatusCanceled {
        return StatusCanceled
    }
    // 只有一个 canceled → 升级为 killed（因为已经有 workflow 在运行）
    if s == StatusCanceled { s = StatusKilled }
    if t == StatusCanceled { t = StatusKilled }
    // 取优先级更高的（索引更小的）
    return statusPriorityOrder[min(priorityMap[s], priorityMap[t])]
}
```

**合并示例**：

| Workflow A | Workflow B | Pipeline 结果 | 说明 |
|-----------|-----------|-------------|------|
| Success | Failure | **Failure** | 有一个失败则整体失败 |
| Success | Killed | **Killed** | 被取消优先于成功 |
| Canceled | Canceled | **Canceled** | 全部未启动即取消 |
| Canceled | Failure | **Killed** | 部分运行过，取消视为 killed |
| Running | Success | **Running** | 仍在运行 |
| Error | Success | **Error** | 系统错误优先级最高 |
| Skipped | Skipped | **Skipped** | 全部跳过 |

### 9.3 Pipeline 最终状态计算

**代码位置：`server/pipeline/pipeline_status.go:27-35 + `server/rpc/rpc.go:379-383**

```go
// PipelineStatus 遍历所有 workflow，合并得到最终状态
func PipelineStatus(workflows []*model.Workflow) model.StatusValue {
    status := model.StatusSuccess
    for _, p := range workflows {
        status = MergeStatusValues(status, p.State)
    }
    return status
}

// Done() 中判断是否所有 workflow 都已完成
if !model.IsThereRunningStage(currentPipeline.Workflows) {
    pipeline.UpdateStatusToDone(store, *currentPipeline,
        pipeline.PipelineStatus(currentPipeline.Workflows),
        workflow.Finished)
}
```

---

## 10. 全链路时序图（含 gRPC + Retry + Cancel）

```
Agent                                 Server
  │                                     │
  │ ══════ 连接建立 ══════              │
  │                                     │
  │  Dial() → authConn + mainConn      │
  │  Version() ──────────────────────►  │  版本校验
  │  RegisterAgent() ───────────────►   │  注册 agent
  │                                     │
  │ ══════ 工作流执行 ══════           │
  │                                     │
  │  Next(filter) ──────────────────►   │  scheduler.Poll() 阻塞
  │        ◄────────────────────────    │  出队后返回 workflow
  │                                     │
  │  Init(id, state) ───────────────►   │  Pipeline: Pending→Running
  │                                     │  Workflow: Pending→Running
  │                                     │
  │  ┌─ Wait(id) ───────────────────►   │  阻塞，监听取消
  │  │                                   │
  │  ├─ Extend(id) ────────────────►    │  每 20s 续期 lease
  │  │  Extend(id) ────────────────►    │
  │  │  ...                             │
  │  │                                   │
  │  └─ [执行步骤]                       │
  │     Update(id, stepState) ──────►   │  Step: Pending→Running
  │     Update(id, stepState) ──────►   │  Step: Running→Success
  │     ...                             │
  │                                     │
  │ ══════ 正常完成 ══════             │
  │                                     │
  │  Done(id, state) ───────────────►   │  scheduler.Done(id, StatusSuccess)
  │                                     │  Workflow: → Success
  │                                     │  Pipeline: 汇总所有 workflow
  │                                     │  → 最终状态
  │                                     │
  │ ══════ 取消场景 ══════             │
  │                                     │
  │                       [API: CancelPipeline()] 
  │                       Cancel() → ErrorAtOnce(ids, ErrCancel)
  │                                     │  └─ close(entry.done)
  │                                     │     └─ Wait() 返回 ErrCancel
  │  ◄── Wait() canceled=true ──────   │
  │                                     │
  │  cancelWorkflowCtx(ErrCancel)       │
  │  └─ 步骤检测到 ctx 取消            │
  │     └─ traceStep(canceled)          │
  │  Update(id, {canceled:true}) ───►   │  Step: → Killed
  │                                     │
  │  Done(id, {canceled:true}) ─────►   │  scheduler.Done(id, StatusKilled)
  │                                     │  Workflow: → Killed
  │                                     │  Pipeline: → Killed
  │                                     │
  │ ══════ Agent 失联场景 ══════       │
  │                                     │
  │  [网络断开，Extend 停止]             │
  │                                     │
  │                       [1min 后 deadline 过期]
  │                       resubmitExpiredPipelines()
  │                       └─ pending.PushFront(task)
  │                       └─ delete(running, id)
  │                       └─ close(entry.done)
  │                                     │
  │                       [新 Agent Poll() 获得该任务]
  │                       Next() → 返回同一 workflow
  │                                     │
  │ ══════ gRPC 重试场景 ══════        │
  │                                     │
  │  Update() ──────► Unavailable       │
  │  10ms 后重试 ──────► Unavailable    │
  │  20ms 后重试 ──────► Unavailable    │
  │  40ms 后重试 ──────► ...            │
  │  (指数退避直到 10s 上限)             │
  │  成功 ─────────────────────────►    │
  │                                     │
  │  [若 connectionRetryTimeout 耗尽]    │
  │  → ErrConnectionLost               │
  │  → 整个 agent 关闭                  │
```

---

## 11. Agent 心跳与重连机制

### 11.1 双重心跳：HTTP 服务端/客户端监控

#### 11.1.1 本地 HTTP Healthcheck（Dockerflow 协议）

**代码位置：`cmd/agent/core/health.go:35-104`**

Agent 在启动时（`runWithRetry` → `initHealth()`）注册三个 HTTP 端点：

```
┌───────────────────────────────────────────────────────────┐
│  HTTP Server（默认 :3000                                    │
│                                                         │
│  GET /healthz   → handleHeartbeat()                   │
│     └─ counter.Healthy()                               │
│        200 OK  /  500 Internal Server Error       │
│                                                         │
│  GET /version   → handleVersion()                         │
│     └─ 返回 JSON: {source, version}                      │
│                                                         │
│  GET /varz   → handleStats()                            │
│     ├─ Healthy() 检查 + 返回统计信息                     │
│     └─ counter.WriteTo(w) 输出 JSON 统计                  │
│                                                         │
└───────────────────────────────────────────────────────────┘
```

**健康检查核心——`agent/state.go`的判定逻辑**：

- `counter.Healthy()` 基于统计数据汇总所有 runner 工作流数量判断。

#### 11.1.2 gRPC ReportHealth 心跳（Agent → Server

**代码位置：`cmd/agent/core/agent.go:47-49, 245-264`**

```go
const reportHealthInterval = time.Second * 10  // 每 10 秒报告一次
```

Agent 启动后在一个独立 goroutine 内持续上报：

```go
go func() {
    for {
        err := client.ReportHealth(grpcCtx)
        if err != nil {
            if grpcCtx.Err() != nil || agentCtx.Err() != nil {
                return nil   // context 取消则退出
            }
        }
        select {
        case <-agentCtx.Done():
            return nil
        case <-time.After(reportHealthInterval):   // 10s
        }
    }
}()
```

**gRPC 客户端实现**（`agent/rpc/client_grpc.go:431-442）：

```go
func (c *client) ReportHealth(ctx context.Context) error {
    req := &proto.ReportHealthRequest{Status: "I am alive!"}
    return retryRPC(ctx, c, "report_health", func() (*proto.Empty, error) {
        if !c.IsConnected() {
            return nil, errNotConnected
        }
        r, err := c.client.ReportHealth(ctx, req)
        return r, classifyRPCErr(ctx, err)
    })
}
```

**Server 端处理**（`server/rpc/rpc.go:520-534`

```go
func (s *RPC) ReportHealth(ctx context.Context, status string) error {
    agent, _ := s.getAgentFromContext(ctx)
    if status != "I am alive!" {
        return errors.New("Are you alive?")
    }
    agent.LastContact = time.Now().Unix()
    return s.store.AgentUpdate(agent)
}
```

**心跳作用总结表**：

| 机制 | 方向 | 频率 | 作用 |
|--------|------|------|------|
| HTTP Health | 外部→Agent | 按需 | Kubernetes/容器健康检查，决定是否重启 |
| ReportHealth gRPC | Agent→Server | 每 10s | 更新数据库 Agent 表的 LastContact 字段 |
| Extend gRPC | Agent→Server | 每 20s | 续期队列 lease |
| gRPC Keepalive | 双向 | 配置化 | 底层 TCP 保活 |

### 11.2 Agent ID 持久化与重连（崩溃重启重用 Agent 崩溃循环重连策略

**代码位置：`cmd/agent/core/agent.go:128-135, 222-226

Agent 启动时通过 `agent-config` 持久化自己的 ID：

```
┌─────── 首次启动 ─────────────────────────────────────────────┐
│                                                         │
│  1. grpcClientCtx, ... Dial(grpcClientCtx, DialConfig{│
│        AgentID: agentConfig.AgentID (初次 = 0)             │
│                                                         │
│     └─ Auth 阶段：如果 AgentID == 0  → 自动创建新      │
│        服务端：AgentCreate() → 返回新 agent.ID             │
│                                                         │
│  2. RegisterAgent() → 更新：Platform/Backend/Capacity/Version  │
│        └─ store.AgentUpdate(agent)                       │
│                                                         │
│  3. Persist agent ID 到本地文件：                         │
│     agentConfig.AgentID = agentConn.AgentID               │
│     writeAgentConfig(agentConfig, agentConfigPath)        │
│     agentConfigPersisted.Store(true)                      │
│                                                         │
└───────────────────────────────────────────────────────────┘

┌─────── 崩溃重启 ─────────────────────────────────────┐
│                                                         │
│  1. 读取 agentConfigPath → 读出上次持久化的 AgentID       │
│                                                         │
│  2. Dial() 时携带已有 AgentID                         │
│     → Auth 阶段：                                                        │
│        服务端：AgentFind(agentID)                          │
│        ├─ 存在 → 复用原 ID，不会在数据库里新建          │
│        └─ 不存在 → 自动创建新 ID（避免 leak 旧 agent             │
│                                                         │
│  3. RegisterAgent() → 更新 agent 最新信息             │
│                                                         │
│  4. agentConfigPersisted = true → 优雅关闭时不会 Unregister   │
│                                                         │
└───────────────────────────────────────────────────────────┘
```

**关键配置选项**：
- `agentConfigPersisted == true 的 Agent：关闭时 Unregister，重启后数据库保留 ID
- `agentConfigPersisted == false` 的无状态 Agent：关闭时 Unregister，彻底删除

**Agent→ 1.3 连接失败重连架构

**代码位置：`cmd/agent/core/agent.go:323-344

```go
func runWithRetry(backendEngines []types.Backend) func(...) error {
    retryCount := c.Int("connect-retry-count")
    retryDelay := c.Duration("connect-retry-delay")
    for range retryCount {
        if err = run(ctx, c, backendEngines); status.Code(err) == codes.Unavailable {
            log.Warn().Err(err).Msg(...)
            time.Sleep(retryDelay)          // 线性等待
        } else {
            break
        }
    }
    return err
}
```

**三层重连层级**：

```
                    ┌───────────────────────────────────────────────┐
│ Layer 1: runWithRetry()                               │
│  位置: cmd/agent/core/agent.go:323-344                  │
│  触发: run() 返回 codes.Unavailable                    │
│  策略: 固定间隔 retryDelay，次数: retryCount 次          │
│  作用: 整个 agent 全量重启（重新 Dial/Register/Runner）        │
│                                                         │
│  （codes.Unavailable    │
│  └── 非 Unavailable → 停止（Version 版本不兼容等 fatal         │
│                                                         │
├───────────────────────────────────────────────────────────────┤
│ Layer 2: retryRPC()                                    │
│  位置: agent/rpc/client_grpc.go:143-166                 │
│  触发: 每个 RPC 调用（Next/Init/Update/Done/...            │
│  策略: 指数退避 10ms→10s，MaxElapsedTime           │
│  作用: 单次 RPC 重试                                    │
│                                                         │
│  ctx 取消        → 不重试，返回零值                          │
│  MaxElapsed 耗尽且仍未连接 → ErrConnectionLost          │
│  其他永久错误           → 返回原始错误                          │
│                                                         │
├───────────────────────────────────────────────────────────────┤
│ Layer 3: Runner 循环                                    │
│  位置: cmd/agent/core/agent.go:272-309                  │
│  触发: runner.Run() 返回非 ErrConnectionLost 的普通错误     │
│  策略: 等待 5s 后继续循环                           │
│  作用: 单个 runner 槽位重试                            │
│                                                         │
│  ErrConnectionLost → 触发 ctxCancel → 整个 agent 关闭        │
│  singleWorkflow=true → ctxCancel                    │
│  其他错误       → sleep 5s，继续下一次循环           │
│                                                         │
└───────────────────────────────────────────────────────────────┘
```

### 11.4 优雅关闭：Unregister 流程

**代码位置：`cmd/agent/core/agent.go:200-220

```go
serviceWaitingGroup.Go(func() error {
    defer grpcClientCtxCancel(nil)
    <-agentCtx.Done()

    if !agentConfigPersisted.Load() {  // 无状态 agent
        if client.IsConnected() {
            client.UnregisterAgent(grpcClientCtx)
            // UnregisterAgent()
        }
    }
    return nil
})
```

Server 端处理 `cmd/server/rpc/rpc.go:504-515`：

```go
func (s *RPC) UnregisterAgent(ctx context.Context) error {
    agent, _ := s.getAgentFromContext(ctx)
    server.Config.Services.Scheduler.Queue.RemoveAgent(agent.ID)
    return s.store.AgentDelete(agent)
}
```

---

## 12. Dead Letter 落盘：重试失败后的日志持久化机制

### 12.1 Agent 端日志批量处理架构

**代码位置：`agent/rpc/client_grpc.go:340-409`

Agent 端不做日志不直接发送到 `logs := make(chan *proto.LogEntry, 10) 通道，由后台 goroutine 合并批量发送：

```
┌────────────────────────────────────────────────────────────┐
│  Agent 端日志管道                          │
│                                                          │
│  pipeline.Logger(step, rc) ──→ 后端日志流             │
│        │                                                  │
│        │  每条日志行                              │
│        ▼                                                  │
│  EnqueueLog(logEntry)  ──→ c.logs channel (缓冲 10     │
│        │                                                  │
│        ▼                                                  │
│  processLogs(ctx)                               │
│        │                                                  │
│        ├─ 触发 1：bytes >= maxLogBatchSize (1 MiB)    │
│        ├─ 触发 2：time.After(maxLogFlushPeriod (1s)     │
│        ├─ 触发 3：ctx.Done() / channel 关闭               │
│        │                                                  │
│        ▼                                                  │
│  sendLogs(ctx, entries) → retryRPC("log")                   │
│        │                                                  │
│        ├─ 指数退避重试                                     │
│        └─ 重试耗尽：log.Error()     丢弃                │
│                                                          │
│  ⚠️ 关键: 即使 sendLogs 失败后 entries[:0]            │
│     （注释：even if send failed, we don't have infinite memory│
│                retry has already been used                         │
│                                                          │
└────────────────────────────────────────────────────────────┘
```

**processLogs 的实现细节**：

```go
func (c *client) processLogs(ctx context.Context) {
    var entries []*proto.LogEntry
    var bytes int
    send := func() {
        if len(entries) == 0 { return }
        if err := c.sendLogs(ctx, entries); err != nil {
            log.Error().Err(err)
        }
        // 即使失败也清空内存！
        entries = entries[:0]
        bytes = 0
    }
    for {
        select {
        case <-ctx.Done():
            return
        case entry, ok := <-c.logs:
            entries = append(entries, entry)
            bytes += grpc_proto.Size(entry)
            if bytes >= maxLogBatchSize { send() }
        case <-time.After(maxLogFlushPeriod):
            send()
        }
    }
}
```

### 12.2 Server 端日志落盘：内存广播 + 持久化双通道

**代码位置：`server/rpc/rpc.go:415-475

Server 收到 `RPC.Log() 使用双通道写入：

```
RPC.Log(c, stepUUID, rpcLogEntries)
    │
    │  ┌───────────────────────────────────────────────────┐
    │  安全检查:                              │
    │  - Step 存在性检查                   │
    │  - Agent 权限检查                          │
    │  - allowAppendingLogs() 终态检查           │
    │  - 更新 Agent LastWork                │
    │  └───────────────────────────────────────────────┘
    │
    ├─ 通道 A（异步）                 │
    │    （go func() {                               │
    │    │      s.logger.Write(c, step.ID, logEntries)    │
    │    │  └─ 内存 pubsub/webclient 实时流        │
    │    │      └─ s.streams[stepID].list = append    │
    │    │     ：subscriber channel 满              │
    │    │        log.Info("subscriber channel is full │
    │    │             -- dropping logs")                    │
    │    │   （慢消费者丢弃          │
    │    └─ }()                                         │
    │
    └─ 通道 B（同步，持久化   持久化存储
         s.LogStore.LogAppend(step, logEntries)
         │
         └─ file/file.go:90-115
             os.OpenFile(O_APPEND|O_CREATE|O_WRONLY)
             for _, entry := range logEntries {
                 json.Marshal(entry) → 逐行 JSON Lines 格式
                 file.Write(jsonBytes)
             }
             file.Close()
```

### 12.3 File LogStore（file.go:37-121

Dead Letter 概念映射关系：

```
┌────────────────────────────────────────────────────────────┐
│                                                          │
│  {base}/{step.ID}.json                                │
│                                                          │
│  存储格式: JSON Lines (每行一条)              │
│                                                          │
│  示例内容:                                               │
│  {"StepID":42,"Time":1718457600,"Line":1,   │
│    "Data":"+ go build ./...","Type":0}                  │
│  {"StepID":42,"Time":1718457601,"Line":2,   │
│    "Data":"go: downloading...","Type":0}                     │
│                                                          │
│  LogFind(step): 逐行扫描读回 []*model.LogEntry      │
│  LogAppend(step, entries): APPEND 追加写          │
│  LogDelete(step): os.Remove(file)                       │
│  StepFinished(step): 空操作（file）                        │
│                                                          │
└────────────────────────────────────────────────────────────┘
```

**配置来源：`server/config.go → `server/services/log/file/file.go:41-52

```go
func NewLogStore(base string) (service_log.Service, error {
    if base == "" { return nil, ... }
    os.MkdirAll(base, 0o700)
    return logStore{base: base}, nil
}
```

### 12.4 日志落盘保障级别汇总

| 环节 | 失败处理 | 是否落盘 |
|------|---------|-------|
| **Agent 端 batch sendLogs 连接中断 | MaxElapsedTime 耗尽后丢弃 | ❌ 不 |
| **Server 端内存 pubsub** | subscriber channel 满 → 慢消费者丢弃 | ❌ 仅内存 |
| **Server 端 LogStore** | 磁盘 IO 失败 → log.Error() 记录 | ✅ JSON 文件持久化 |
| **步骤完成后 StepFinished** | 触发文件保留文件 | ✅ 持久化 |
| **Server 重启** | 通过 LogFind 读取持久化内容 | ✅ 恢复 |

> **核心设计权衡**：Agent 侧重内存优先，不做本地落盘；Server 端同步持久化。这样设计上保证**日志至少一次**落盘。

---

## 13. Cancel 事件跨 step 的传播分发栈

### 13.1 Cancel 事件源

Cancel 事件传播两条路径传播

```
                    ┌─────────────────────────────────────────────────┐
│ 路径 A: Context 取消（硬终止             │
│  workflowCtx 被 cancelWorkflowCtx(ErrCancel)       │
│                                                         │
│ 路径 B: err.Protected[error] 错误状态（软传播）          │
│  r.err.Set(ExitError/OomError/ErrCancel)       │
└──────────────────────────────────────────────────┘
```

### 13.2 路径 A：Context 级硬终止

**代码位置：`agent/runner.go:100-148` 的 context tree

```
Workflow Runtime.│
│  workflowCtx = WithTimeout(WithCancelCause(WithTimeout(ctxMeta, timeout)))
│                                                           │
│  Cancel 信号来源:                                       │
│  ├─ 来源 1: Wait() 返回 canceled=true                 │
│  │    cancelWorkflowCtx(pipeline_errors.ErrCancel)             │
│  │                                                         │
│  ├─ 来源 2: utils.WithContextSigtermCallback                │
│  │    SIGTERM → cancelWorkflowCtx(ErrCancel)            │
│  │                                                         │
│  └─ 来源 3: Workflow超时                                   │
│       cancelWorkflowCtx(nil)  (deadline exceeded            │
│                                                           │
│  workflowCtx 通过 WithContext(workflowCtx) 注入到 Runtime.ctx      │
│                                                           │
└──────────────────────────────────────────────────────────────┘
```

**Runtime.ctx 传播的具体传递路径**：

```go
// agent/runner.go:172-184
pipeline_runtime.New(
    workflow.Config,backend,
    pipeline_runtime.WithContext(workflowCtx),  ← 注入 Runtime.ctx
    ...
).Run(runnerCtx)
```

在 Runtime 内部**双重 context 设计**：

```go
// pipeline/runtime/workflow.go:35-86
func (r *Runtime) Run(runnerCtx context.Context) error {
    ...
    for _, stage := range r.spec.Stages {
        stageChan := r.runStage(runnerCtx, stage.Steps)
        select {
        case <-r.ctx.Done():   // workflowCtx 取消
            <-stageChan         // 等待当前 stage 优雅退出
            return ErrCancel
        case err := <-stageChan:
            r.err.Set(err)       // 记录错误，供后续 stage 使用
        }
    }
    ...
}
```

**Context 取消在 step 内部的渗透点**：

| 调用链
--------+---------------+---------------------------+
| 位置 | 检查点 | 行为 |
|------|--------|------|
| `startStep()` | `r.engine.StartStep(r.ctx, ...)` | backend 接收取消 | backend 用 r.ctx → StartStep 失败 → traceStep(startError) |
| `startStep()` | `r.engine.TailStep(r.ctx, ...) 日志流终止 | rc.Close() |
| `completeStep()` | `r.engine.WaitStep(r.ctx, ...)` | errors.Is(err, context.Canceled) | waitState.Error = ErrCancel |
| `completeStep()` | r.ctx.Err() 二次检查| 重新检测 race 与 context.Canceled 检查 | waitState.Error = ErrCancel |
| `runBlockingStep()` | errors.Is(err, context.Canceled) → ErrCancel |
| `runStage()` | stageChan 返回 ErrCancel → r.err.Set() |
| `Run()` 主循环 | `<-r.ctx.Done()` → 提前返回 ErrCancel |

### 13.3 路径 B：err 状态软传播（OnFailure/OnSuccess 决策

**代码位置：`pipeline/runtime/step.go:62-85

```go
func (r *Runtime) shouldSkipStep(step *backend_types.Step) bool {
    currentErr := r.err.Get()  // 读取前序错误
    
    // 前序有错误，当前 step.OnFailure == false → 跳过
    if currentErr != nil && !step.OnFailure { return true }
    
    // 前序无错误，当前 step.OnSuccess == false → 跳过
    if currentErr == nil && !step.OnSuccess { return true }
    
    return false
}
```

**OnFailure / OnSuccess 配置矩阵**：

| r.err | OnSuccess | OnFailure | 是否执行 | 典型场景
----------|-----------|----------|---------|---------|
| nil (无错) | true | - | ✅ 执行 | 默认，普通 step |
| nil (无错) | false | - | ❌ 跳过 | 手动设置只在失败时执行（失败清理 |
| 有错误 | - | true | ✅ 执行 | cleanup/notify/rollback |
| 有错误 | - | false | ❌ 跳过 | 普通 step，失败链断裂 |

**step 错误设置点**：

```
Stage 0                                  Stage 1
┌─────────────┐ 阶段：                           ┌──────────────────────┐
│ Step A (build     │                         │ Step E (notify) │
│ OnSuccess: true     │  r.err = ErrExit(1)                │ OnFailure: true │
│                   │ ─────────────────────────────────► │               │
│ ExitCode: 1      │                         │ E 执行✅       │
└─────────────┘                         │ (slack 通知     │
                                          └───────────────┘
                                                   │
                                                   │
                                                   ▼
                                          ┌──────────────────────┐
                                          │ Step F (deploy) │
                                          │ OnSuccess: true    │
                                          │ OnFailure: false  │
                                          │                  │
                                          │ F 跳过❌        │
                                          │ (不部署失败不部署失败)  │
                                          └──────────────┘
```

### 13.4 step 启动之前的并行 step 间的 event 分发时序

**代码位置：`pipeline/runtime/workflow.go:141-159

```go
func (r *Runtime) runStage(runnerCtx context.Context, steps []*backend_types.Step) <-chan error {
    var g errgroup.Group
    done := make(chan error)
    
    for _, step := range steps {
        g.Go(func() error {
            return r.executeStep(runnerCtx, step)
        })
    }
    
    go func() { done <- g.Wait(); close(done) }()
    return done
}
```

**同 stage 并行 step 间的传播**：

```
Stage (并行执行
    │
    ├─ Step 1 (clone)       Step 2 (service)
    │   │                  │
    │   │ startStep()       │ startStep()
    │   │ completeStep()    │ completeStep()
    │   │ traceStep()    │ traceStep()
    │   │                  │
    │   ▼                  ▼
    │   err1               err2
    │   │                  │
    │   └────────┬──────────┘
    │            │
    │            ▼
    │        g.Wait() → 返回第一个非 nil error
    │
    │            │
    │            ▼
    └── Stage error (若有一个错误 → 下一个 stage shouldSkipStep()

关键设计：**同一 stage 内的所有 step 同时启动，并行执行，
               step 间的通过 errgroup.Group 的首个错误传播给下一个 stage 的所有 step 的 OnFailure/OnSuccess 判定

跨 stage 顺序执行，err 传播
```

### 13.5 Cancel 事件在 step.go:99-109：CI_PIPELINE_STATUS 环境变量注入

**代码位置：`pipeline/runtime/step.go:99-109

```go
func (r *Runtime) setStepEnv(step *backend_types.Step) error {
    if r.err.Get() != nil {
        step.Environment["CI_PIPELINE_STATUS"] = "failure"
    } else {
        step.Environment["CI_PIPELINE_STATUS"] = "success"
    }
}
```

这样 OnFailure step 在执行时**可以感知整个 pipeline 已经处于失败状态，
据此决定行为（例如发送失败通知）。

### 13.6 detached step 的特殊处理

**代码位置：`pipeline/runtime/step.go:227-258`

```go
func (r *Runtime) runDetachedStep(...) error {
    waitForLogs, startTime, err := r.startStep(step)
    if err != nil { return r.traceStep(nil, err, step) }
    
    r.uploadWait.Add(1)
    go func() {                              // 后台goroutine
        defer r.uploadWait.Done()
        
        processState, err := r.completeStep(...)
        if errors.Is(err, context.Canceled) {
            err = pipeline_errors.ErrCancel
        }
        r.traceStep(processState, err, step)  // 最终状态上报
    }()
    
    return nil   // 立即返回，不阻塞 pipeline
}
```

**detached 的传播注意事项**：

- detach step**提前返回 nil，不阻塞 stage 继续
- 其完成结果通过 uploadWait 延迟到 workflow 完成前确保完成
- **不会通过 err.Set(workflow 返回，不会影响下一个 stage 的 skip 影响
- 但 context 取消时，completeStep 通过 ErrCancel 处理

### 13.7 Failure 传播完整栈图

```
┌────────────────────────────────────────────────────────────┐
│                    Server 触发                                │
│                                                    │
│  Cancel()                                       │
│    │                                              │
│    ├── queue.ErrorAtOnce(ids, ErrCancel)                 │
│    │       │                                         │
│    │       └── close(entry.done)                         │
│    │                                                │
│    └── 数据库 Pending workflow/step 批量设置为终态        │
│                                                    │
└──────────────────────────────┬─────────────────────┘
                               │
                               ▼
                    ┌────────────────────────────┐
                    │  Agent: Wait() 返回 canceled=true          │
                    │  cancelWorkflowCtx(ErrCancel)          │
                    └──────────────┬────────────────────┘
                                   │
              ┌────────────────────────┼──────────────────────┐
              │                    │                     │
              ▼                    ▼                     ▼
┌─────────────────────┐  ┌───────────────────┐  ┌──────────────────┐
│ 路径 A: r.ctx 被取消     │  │ 路径 B: r.err 设置     │  │ Extend 协程退出       │
│ 上下文取消│  │ ErrCancel│  │ workflowCtx.Done()│
└──────────┬───────────┘  └──────┬────────┘  └──────────────────┘
           │                   │
           ▼                   ▼
┌────────────────────────────┐  ┌─────────────────────────────────────┐
│ 运行中的步骤:                │  │ 后续 stage 的步骤:             │
│ completeStep() 检测到            │  │ shouldSkipStep() 检查 r.err  │
│ context.Canceled               │  │   ├─ OnSuccess=false → 正常执行       │
│ → ErrCancel               │  │   └─ OnFailure=true → 执行     │
│                                 │
└──────────┬──────────┘  └───────────────┬──────────────┘
           │                           │
           ▼                           ▼
┌──────────────────────────────────────────────────────┐
│  traceStep(Canceled=true)                 │
│       client.Update(stepState)           │
│       通知 Server Step → Killed         │
└──────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Run() 主循环检测 r.ctx.Done()                │
│       return ErrCancel                  │
└──────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Runner.Run() 收尾                                    │
│  state.Canceled = true                       │
│  client.Done(doneCtx, workflow.ID, state)           │
└──────────────────────────────────────────────────────┘
```

---

## 14. Matrix Expansion：矩阵拆分为多 Workflow 的调度机制

### 14.1 Matrix 的两种定义形式

**代码位置：`pipeline/frontend/yaml/matrix/matrix.go:46-135`**

矩阵有两种写法，`Parse()` 函数会按顺序尝试：

#### 形式 1：`matrix.include` 列表（显式枚举）

```yaml
matrix:
  include:
    - GO_VERSION: "1.21"
      OS: linux
    - GO_VERSION: "1.22"
      OS: darwin
```

由 `parseList()` 解析，直接返回 `[]Axis`。

#### 形式 2：`matrix` key-value 笛卡尔积（自动展开）

```yaml
matrix:
  GO_VERSION: ["1.21", "1.22"]
  OS: [linux, darwin, windows]
```

由 `parse()` 解析后，通过 `calc()` 计算笛卡尔积。

### 14.2 笛卡尔积展开算法

**代码位置：`pipeline/frontend/yaml/matrix/matrix.go:70-112`**

```go
func calc(matrix Matrix) []Axis {
    // 计算总排列数
    var perm int
    var tags []string
    for k, v := range matrix {
        perm *= len(v)
        if perm == 0 { perm = len(v) }
        tags = append(tags, k)
    }

    var axisList []Axis
    for p := 0; p < perm; p++ {
        axis := map[string]string{}
        decrease := perm
        for i, tag := range tags {
            elems := matrix[tag]
            decrease /= len(elems)
            elem := p / decrease % len(elems)   // 模数展开
            axis[tag] = elems[elem]
            if i > limitTags { break }          // 最多 10 个 tag
        }
        axisList = append(axisList, axis)
        if p > limitAxis { break }              // 最多 25 个组合
    }
    return axisList
}
```

**限制常量**：
- `limitTags = 10`：最多 10 个维度（key）
- `limitAxis = 25`：最多 25 个组合（展开后的 workflow 数）

### 14.3 PipelineBuilder：从矩阵到多 Workflow

**代码位置：`pipeline/frontend/builder/builder.go:52-97`**

```go
func (b *PipelineBuilder) Build() (items []*Item, errorsAndWarnings error) {
    b.Yamls = SortYamlFilesByName(b.Yamls)
    pidSequence := 1

    for _, y := range b.Yamls {
        // 1. 解析矩阵
        axes, err := matrix.ParseString(string(y.Data))
        if len(axes) == 0 {
            axes = append(axes, matrix.Axis{})  // 无矩阵也生成一个空 axis
        }

        // 2. 每个 axis 生成一个 workflow
        for i, axis := range axes {
            workflow := &Workflow{
                PID:     pidSequence,
                Environ: axis,           // 矩阵变量作为环境变量
                Name:    SanitizePath(y.Name),
            }
            if len(axes) > 1 {
                workflow.AxisID = i + 1  // 多矩阵时标记 axis id
            }

            item, err := b.genItemForWorkflow(workflow, axis, string(y.Data))
            items = append(items, item)
            pidSequence++
        }
    }

    items = filterMissingDependencies(items)
    return items, errorsAndWarnings
}
```

**执行流程**：

```
PipelineBuilder.Build()
    │
    ├─ 遍历每个 YAML 文件
    │
    ├─ matrix.ParseString(yaml)
    │    ├─ parseList()   → 尝试 matrix.include
    │    └─ parse()+calc() → 笛卡尔积展开
    │
    ├─ 对每个 axis 执行 genItemForWorkflow()
    │    ├─ 环境变量合并: metadata.Environ() + axis
    │    ├─ 变量替换: metadata.EnvVarSubst(data, environ)
    │    ├─ yaml.ParseString(substituted)
    │    ├─ linter.Lint()
    │    ├─ when 条件过滤
    │    ├─ toInternalRepresentation() → compiler.Compile()
    │    └─ 生成 Item{Workflow, Config, Labels, DependsOn, RunsOn}
    │
    └─ filterMissingDependencies(items)
         → 去掉依赖不存在的 workflow，确保依赖图完整
```

### 14.4 矩阵变量注入路径

矩阵变量作为环境变量，在三层传递：

| 层次 | 代码位置 | 作用 |
|------|---------|------|
| **Builder 层** | `builder.go:70` `workflow.Environ = axis` | 存入 workflow 元数据 |
| **环境变量层** | `builder.go:207-211` `environmentVariables()` | 合并到 environ 用于变量替换 |
| **Compiler 层** | `compiler/option.go:143-147` `WithEnviron()` | 注入到每个 step 的 Environment map |
| **运行时层** | `runtime/step.go:90-93` `step.Environment` | 容器启动时作为环境变量 |

### 14.5 多 Workflow 的依赖调度

矩阵展开后生成的多个 workflow 通过 `DependsOn` 形成依赖图，由队列调度：

```
workflow-a (axis 1)
workflow-b (axis 2)
workflow-c (axis 3)

depends_on: [workflow-a]
    ↓
workflow-deploy
```

在 `server/pipeline/queue.go:31-45` 的 `getTaskDependencies()` 中，依赖以 workflow 名称解析。

每个 matrix axis 生成的 workflow 使用**同一个基础名称**，通过 `AxisID` 区分。
如果 `depends_on` 引用矩阵 workflow，则**所有矩阵实例完成**后才会执行依赖者。

---

## 15. Global Secret 注入与权限隔离

### 15.1 Secret 的三层级架构

**代码位置：`server/services/secret/db.go:41-72`**

```
┌────────────────────────────────────────────────────────────┐
│                       优先级从高到低                           │
│                                                          │
│  ┌─────────────┐  ┌────────────────┐  ┌───────────────┐   │
│  │ Repo Secret │  │ Org Secret     │  │ Global Secret │   │
│  │ (仓库级)     │  │ (组织级)        │  │ (全局级)       │   │
│  └─────────────┘  └────────────────┘  └───────────────┘   │
│                                                          │
│  同名字段冲突时，Repo 覆盖 Org，Org 覆盖 Global          │
│  （SecretListPipeline 去重时按优先级遍历）               │
└────────────────────────────────────────────────────────────┘
```

**去重逻辑**（`db.go:48-70`）：

```go
// Priority order: repository > user/organization > global
for _, condition := range []struct {
    IsRepository, IsOrganization, IsGlobal bool
}{
    {IsRepository: true},
    {IsOrganization: true},
    {IsGlobal: true},
} {
    for _, secret := range s {
        if _, ok := uniq[secret.Name]; ok { continue }
        uniq[secret.Name] = struct{}{}
        secrets = append(secrets, secret)
    }
}
```

### 15.2 Secret 服务组合链

**代码位置：`server/services/manager.go:90, 104-110 + secret/combined.go:37-74`**

完整的 secret 获取链：

```
Manager.SecretServiceFromRepo(repo)
    │
    ├─ repo.SecretExtensionEndpoint 非空
    │    └─ secret.NewCombined(base, httpExtension)
    │          ├─ extension 优先级更高（同名字段覆盖 base）
    │          └─ extension 失败时仅记录警告，不中断
    │
    └─ 否则返回 m.secret（DB 来源）
```

**Combined 合并逻辑**（`combined.go:56-71`）：

```go
// extension 优先级最高（同名覆盖 base）
merged := append(merged, extensionSecrets...)  // 先放 extension
for _, s := range baseSecrets {                 // 再放 base
    if _, ok := exists[s.Name]; ok { continue }  // 跳过重名
    merged = append(merged, s)
}
```

最终的完整优先级链（从高到低）：
1. **HTTP Extension**（repo 级别的 secret 扩展）
2. **Repo Secret**（仓库级）
3. **Org Secret**（组织级）
4. **Global Secret**（全局级）

### 15.3 Secret 注入编译期：Compiler 的注入时机

**代码位置：`pipeline/frontend/yaml/compiler/compiler.go:35-48, 119-142, 164-193**

Secret 在编译期（compiler 阶段）完成注入，而非运行时。

#### 编译期 Secret 列表（用于日志脱敏）

```go
func (c *Compiler) Compile(conf *yaml_types.Workflow) (*backend_types.Config, error) {
    // 将所有 secret 名+值加入 config.Secrets 列表
    for _, sec := range c.secrets {
        config.Secrets = append(config.Secrets, &backend_types.Secret{
            Name:  sec.Name,
            Value: sec.Value,
        })
    }
    // backend 用这个列表做日志脱敏（masking）
}
```

#### Step 级别 Secret 注入：`from_secret` 语法

**代码位置：`pipeline/frontend/yaml/compiler/convert.go:101-125 + settings/params.go`**

```go
getSecretValue := func(name string) (string, error) {
    name = strings.ToLower(name)
    secret, ok := c.secrets[name]
    if !ok {
        return "", fmt.Errorf("secret %q not found", name)
    }
    // 权限检查：事件类型 + 允许的镜像
    event := c.metadata.Curr.Event
    err := secret.Available(event, container)
    if err != nil {
        return "", err
    }
    return secret.Value, nil
}

// settings 中使用 from_secret
settings.ParamsToEnv(container.Settings, environment, "PLUGIN_", true, getSecretValue, secretMapping)
// environment 中使用 from_secret
settings.ParamsToEnv(container.Environment, environment, "", false, getSecretValue, secretMapping)
```

**YAML 使用方式**：

```yaml
steps:
  - name: deploy
    image: plugins/docker
    settings:
      password:
        from_secret: docker_password  # ← 编译期替换为真实值
```

### 15.4 Secret 权限隔离机制

**代码位置：`pipeline/frontend/yaml/compiler/compiler.go:49-64`**

每个 secret 有两个维度的权限限制：

```go
func (s *Secret) Available(event metadata.Event, container *yaml_types.Container) error {
    // 限制 1: 仅允许特定 plugin 镜像使用
    onlyAllowSecretForPlugins := len(s.AllowedPlugins) > 0
    if onlyAllowSecretForPlugins && !container.IsPlugin() {
        return fmt.Errorf("secret %q is only allowed to be used by plugins", s.Name)
    }
    if onlyAllowSecretForPlugins && !utils.MatchImageDynamic(container.Image, s.AllowedPlugins...) {
        return fmt.Errorf("secret %q is not allowed with image %q", s.Name, container.Image)
    }

    // 限制 2: 仅允许特定事件类型
    if !s.Match(event) {
        return fmt.Errorf("secret %q is not allowed with event %q", s.Name, event)
    }
    return nil
}
```

**权限检查矩阵**：

| 限制维度 | 配置字段 | 默认 | 检查时机 |
|---------|---------|------|---------|
| **Plugin 镜像白名单** | `AllowedPlugins` / `Images` | 空（所有可用） | 编译期 `getSecretValue` 回调时 |
| **事件类型白名单** | `Events` | 空（所有事件） | 编译期 `getSecretValue` 回调时 |
| **作用域隔离** | `OrgID` / `RepoID` | 强制 | 列表查询时数据库过滤 |

### 15.5 Server 端 Secret 获取完整调用栈

```
pipeline.Create()
  │
  └─ parsePipeline()                # server/pipeline/items.go:38
      │
      ├─ SecretServiceFromRepo(repo) → 返回 combined/db secret service
      │
      ├─ secretService.SecretListPipeline(ctx, repo, pipeline, netrc)
      │    │
      │    ├─ combined: base.SecretListPipeline() + extension.SecretListPipeline()
      │    │
      │    └─ db: store.SecretList(repo, true, ...)  # 含值（for pipeline use）
      │         （第 2 个参数 true = 返回 Value 字段）
      │
      ├─ 转换为 compiler.Secret{Name, Value, AllowedPlugins, Events}
      │
      └─ compiler.WithSecret(secrets...)
           │
           └─ compiler.secrets[name] = secret  # 小写 key 存储
                │
                └─ Compile() 时通过 getSecretValue() 注入到步骤环境变量
```

### 15.6 Secret 的数据保护

- **API 返回时**：`SecretList(repo, false, ...)` 第 2 参数 false → 不返回 Value 字段
- **Pipeline 执行时**：`SecretListPipeline` 返回完整 value，但仅在编译期使用
- **日志脱敏**：所有 secret 值在日志输出前被 backend 替换为 `***`
- **数据库加密**：通过 `encryption/wrapper/store/secret_store_wrapper.go` 可加密存储

---

## 16. Agent Capacity 与 Quota 上报调用栈

### 16.1 Capacity 的来源与含义

**代码位置：`cmd/agent/core/agent.go:69-77, 193, 267, 316`**

`capacity` = agent 能同时执行的最大 workflow 数 = runner 槽位数

```go
maxWorkflows := c.Int("max-workflows")  // CLI 参数

// 初始化 state counter
counter.Polling = maxWorkflows
counter.Running = 0

// 注册时上报给 server
client.RegisterAgent(grpcCtx, rpc.AgentInfo{
    Capacity: maxWorkflows,  // ← capacity = max-workflows
    ...
})

// 启动对应数量的 runner goroutine
for i := range maxWorkflows {
    serviceWaitingGroup.Go(func() error {
        runner := agent.NewRunner(...)
        for { runner.Run(agentCtx) }  // 每个 runner 独立循环 Poll
    })
}
```

**capacity 与 runner 的关系**：

```
Agent Capacity = 3
    │
    ├─ Runner #0 → Poll() → worker #0（queue.workers 中占一个槽位）
    ├─ Runner #1 → Poll() → worker #1
    └─ Runner #2 → Poll() → worker #2

每个 runner:
  - 独立调用 scheduler.Poll()
  - 在 queue.workers 中注册一个独立 worker
  - 一次只能处理一个 workflow
  - 完成后自动重新 Poll
```

### 16.2 Capacity 上报流程：Agent → Server

**完整调用栈**：

```
Agent 端 (cmd/agent/core/agent.go)
  │
  ├─ maxWorkflows = c.Int("max-workflows")
  │
  ├─ client.RegisterAgent(ctx, rpc.AgentInfo{Capacity: maxWorkflows})
  │    │
  │    └─ gRPC Woodpecker.RegisterAgent()
  │         └─ proto.RegisterAgentRequest{Info: {Capacity: ...}}
  │
  ▼
Server gRPC 层 (server/rpc/server.go:193-205)
  │
  └─ WoodpeckerServer.RegisterAgent()
       └─ s.peer.RegisterAgent(ctx, rpc.AgentInfo{Capacity: int(agentInfo.GetCapacity()), ...})
  │
  ▼
RPC 业务层 (server/rpc/rpc.go:477-500)
  │
  └─ RPC.RegisterAgent()
       ├─ getAgentFromContext(ctx) → 从 JWT 取出 agent 对象
       ├─ agent.Capacity = int32(info.Capacity)  ← 更新 capacity
       ├─ agent.Backend/Platform/Version/CustomLabels 一并更新
       ├─ s.store.AgentUpdate(agent)  ← 持久化到数据库
       └─ server.Config.Services.Scheduler.Queue.AddAgent(agent)  ← 加入调度池
```

**关键代码**（`server/rpc/rpc.go:489-492`）：

```go
agent.Backend = info.Backend
agent.Platform = info.Platform
agent.Capacity = int32(info.Capacity)  // ← 更新 capacity
agent.Version = info.Version
```

### 16.3 Queue 中的 Worker 注册与 Capacity 关系

**代码位置：`server/queue/fifo.go:87-111`**

每个 runner 的 Poll 调用在 queue 中注册一个独立的 `worker`：

```go
func (q *fifo) Poll(c context.Context, agentID int64, filter FilterFn) (*model.Task, error) {
    ctx, stop := context.WithCancelCause(c)
    w := &worker{
        agentID: agentID,
        channel: make(chan *model.Task, 1),
        filter:  filter,
        stop:    stop,
    }
    q.workers[w] = struct{}{}  // 加入 worker 池
    defer delete(q.workers, w)

    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case t := <-w.channel:
            return t, nil
        }
    }
}
```

**调度时的分配**（`fifo.go:314-336`）：

```go
func (q *fifo) assignToWorker() (*list.Element, *worker) {
    for element := q.pending.Front(); element != nil; element = element.Next() {
        task := element.Value.(*model.Task)
        for worker := range q.workers {
            matched, score := worker.filter(task)
            if matched && score > bestScore {
                bestWorker = worker
                bestScore = score
            }
        }
        if bestWorker != nil {
            return element, bestWorker
        }
    }
    return nil, nil
}
```

**重要发现**：
- `capacity` 在数据库中**仅作元数据记录**（用于 UI 展示）
- **实际并发控制**由 queue 中的 `workers` 数量决定（即 agent 实际 Poll 的 runner 数）
- 每个 runner 独立 Poll → 独立 worker → 自然形成并发上限

### 16.4 Worker 的生命周期

```
Agent 启动
  │
  ├─ Runner 1: Poll() → 加入 queue.workers
  ├─ Runner 2: Poll() → 加入 queue.workers
  └─ Runner N: Poll() → 加入 queue.workers
        │
        │  (阻塞等待任务)
        │
        ▼
  process() 循环分配任务
    assignToWorker() → 选中某个 worker
    worker.channel <- task
        │
        ▼
  Runner 收到 task
    → 从 queue.workers 移除？ ❌ 不会移除
    → 执行 workflow
    → 完成后再次 Poll() → 重新加入 workers

Worker 移除时机：
  - ctx 取消（agent 关闭、runner 退出）
  - Poll() 返回时 defer delete(q.workers, w)
```

### 16.5 Agent 注销与清理

**代码位置：`server/rpc/rpc.go:504-515` + `cmd/agent/core/agent.go:200-213`**

```go
// Server 端
func (s *RPC) UnregisterAgent(ctx context.Context) error {
    agent, _ := s.getAgentFromContext(ctx)
    server.Config.Services.Scheduler.Queue.RemoveAgent(agent.ID)
    return s.store.AgentDelete(agent)
}
```

**仅无状态 agent 会注销**：
- `agentConfigPersisted == false`：关闭时 Unregister，数据库删除
- `agentConfigPersisted == true`（持久化 agent）：关闭时不注销，保留 ID

### 16.6 与 Capacity 相关的状态上报

Agent 有两个独立的状态追踪体系：

| 体系 | 位置 | 作用 |
|------|------|------|
| **Agent State** (agent/state.go) | Agent 本地内存 | `Polling / Running / Metadata` 计数，用于健康检查和 varz |
| **Agent Capacity** (model/agent.go) | Server 数据库 | 声明的最大并发数，注册时上报 |

**健康检查逻辑**（`agent/state.go:62-73`）：

```go
func (s *State) Healthy() bool {
    now := time.Now()
    buf := time.Hour  // 1 小时缓冲
    for _, item := range s.Metadata {
        // 运行超过 timeout+1 小时的 workflow 视为不健康
        if now.After(item.Started.Add(item.Timeout).Add(buf)) {
            return false
        }
    }
    return true
}
```

### 16.7 配额与限流总结

Woodpecker 中**没有独立的 quota 限额系统**，并发控制完全由以下机制实现：

1. **Agent 侧**：`--max-workflows` 参数 → capacity → runner 数量 → 自然并发上限
2. **Server 侧**：每个 runner 注册一个 worker，worker 数即为并发槽位
3. **调度侧**：`assignToWorker()` 只能分配给空闲 worker
4. **标签路由**：通过 label filter 实现资源分区（不同 agent 处理不同类型任务）
5. **Org 隔离**：`agent.OrgID` 限制 agent 只能处理对应组织的任务

---

## 17. Workspace Volume 共享与 Cache 协议

### 17.1 Workspace 的编译期生成

**代码位置：`pipeline/frontend/yaml/compiler/compiler.go:130-134 + `pipeline/frontend/yaml/compiler/convert.go:54-56**

Compiler 在编译 workflow 时创建唯一的 volume 和 network 名称：

```go
// compiler.go:130-134
config.Volume  = fmt.Sprintf("%s_default", c.prefix)   // 例: wp_01HXYZ_default
config.Network = fmt.Sprintf("%s_default", c.prefix)   // 例: wp_01HXYZ_default
```

`c.prefix` 由 `WithPrefix()` 选项注入，通常使用 taskUUID 作为前缀，确保不同 workflow 之间资源隔离。

### 17.2 Workspace Volume 挂载路径

**代码位置：`pipeline/frontend/yaml/compiler/convert.go:34-56**

```go
pluginWorkspaceBase = "/woodpecker"              // plugin 步骤的固定基路径
DefaultWorkspaceBase = pluginWorkspaceBase        // 默认基路径

// createProcess 中:
if stepType == backend_types.StepTypeService || container.IsPlugin() {
    workspaceBase = pluginWorkspaceBase           // plugin/service: /woodpecker
}
workspaceVolume := fmt.Sprintf("%s_default:%s", c.prefix, workspaceBase)
// 生成: "wp_01HXYZ_default:/woodpecker"
```

**环境变量注入**：

```go
environment["CI_WORKSPACE"] = path.Join(workspaceBase, c.workspacePath)
// 生成: "/woodpecker/src/go.woodpecker-ci.org/woodpecker"
```

### 17.3 Docker Backend 的 Workspace 实现

**代码位置：`pipeline/backend/docker/docker.go:156-175**

```
SetupWorkflow(ctx, conf, taskUUID)
    │
    ├─ VolumeCreate(ctx, client.VolumeCreateOptions{
    │      Name:   conf.Volume,     // "wp_01HXYZ_default"
    │      Driver: volumeDriver,    // 默认 "local"
    │  })
    │  → 创建命名 Docker Volume
    │
    └─ NetworkCreate(ctx, conf.Network, client.NetworkCreateOptions{
           Driver:     networkDriver,  // Linux: "bridge", Windows: "nat"
           EnableIPv6: &e.config.enableIPv6,
       })
       → 创建隔离 Docker Network
```

**Step 启动时挂载**（`pipeline/backend/docker/convert.go:120-122`）：

```go
// toHostConfig 中:
if len(step.Volumes) != 0 {
    config.Binds = step.Volumes   // 包含 workspace volume 绑定
}
// 例: ["wp_01HXYZ_default:/woodpecker"]
```

**Volume 清理**（`docker.go:385-407`）：

```go
DestroyWorkflow(ctx, conf, taskUUID)
    │
    ├─ VolumeRemove(ctx, conf.Volume, Force: true)
    │    └─ 指数退避重试（"volume is in use" 场景）
    │       InitialInterval: volumeRetryWait
    │       Multiplier: 2
    │       MaxTries: maxRetry
    │
    └─ NetworkRemove(ctx, conf.Network)
```

### 17.4 Kubernetes Backend 的 Workspace 实现

**代码位置：`pipeline/backend/kubernetes/kubernetes.go:226-253 + `pipeline/backend/kubernetes/volume.go:28-86**

```
SetupWorkflow(ctx, conf, taskUUID)
    │
    ├─ [可选] mkNamespace()        → EnableNamespacePerOrg 时按组织创建 K8s namespace
    │
    ├─ startVolume(ctx, engine, conf.Volume, namespace)
    │    │
    │    └─ mkPersistentVolumeClaim(engineConfig, name, namespace)
    │         ├─ AccessMode:  StorageRwx → ReadWriteMany / else → ReadWriteOnce
    │         ├─ StorageClassName: 可配置
    │         └─ Resources.Requests[storage]: VolumeSize（默认 "10Gi"）
    │
    └─ startHeadlessService(ctx, engine, namespace, taskUUID)
         └─ 用于 workflow 内 pod 间 DNS 解析
```

**Pod 挂载 Workspace PVC**（`pipeline/backend/kubernetes/pod.go`）：

每个 step 的 Pod 通过 `PersistentVolumeClaim` 挂载 workspace：

```yaml
# 生成的 PVC 示例
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-01hxyz-default
  namespace: woodpecker
spec:
  accessModes: [ReadWriteMany]     # StorageRwx=true
  storageClassName: local-storage
  resources:
    requests:
      storage: 10Gi
```

### 17.5 Step 间 Workspace 共享机制

```
┌─────────────────────────────────────────────────────────────┐
│  同一 Workflow 内所有 Step 共享相同的 Volume                   │
│                                                           │
│  Docker:                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  clone   │  │  build   │  │  deploy  │                │
│  │          │  │          │  │          │                │
│  │ /woodpecker │ /woodpecker │ /woodpecker │              │
│  │    ↕     │  │    ↕     │  │    ↕     │                │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                │
│       │              │              │                       │
│       └──────────────┼──────────────┘                       │
│                      │                                      │
│            wp_01HXYZ_default                                │
│            (Docker Named Volume)                            │
│                                                           │
│  Kubernetes:                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │  Pod     │  │  Pod     │  │  Pod     │                │
│  │  clone   │  │  build   │  │  deploy  │                │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                │
│       │              │              │                       │
│       └──────────────┼──────────────┘                       │
│                      │                                      │
│            PVC: wp-01hxyz-default                           │
│            (ReadWriteMany)                                  │
└─────────────────────────────────────────────────────────────┘
```

**关键设计**：
- **同一 Workflow 内**：所有 step 共享同一个 volume（clone 写入 → build 读取 → deploy 使用）
- **不同 Workflow 间**：完全隔离（不同 prefix → 不同 volume name）
- **Matrix 展开后**：每个 axis 是独立 workflow → 各自有独立 volume

### 17.6 Cache Step 类型

**代码位置：`pipeline/backend/types/step.go:53-58**

```go
const (
    StepTypeClone    StepType = "clone"      // git clone 步骤
    StepTypeService  StepType = "service"    // 后台服务
    StepTypePlugin   StepType = "plugin"     // plugin 步骤
    StepTypeCommands StepType = "commands"   // 命令步骤
    StepTypeCache    StepType = "cache"      // 缓存步骤（预留类型）
)
```

**当前状态**：`StepTypeCache` 是预留类型，代码中**没有使用处**。Woodpecker 的缓存机制不是通过专用的 cache step type 实现的，而是通过 **plugin 生态** 解决：

**缓存的实际实现方式**：

1. **Volume 级缓存**：Docker volume 在 workflow 生命周期内持续存在（但 workflow 结束后删除）
2. **Plugin 级缓存**：如 `woodpecker-plugins/plugin-cache` 通过 volume 挂载实现目录级缓存恢复/保存
3. **外部存储**：用户可在 YAML 中使用缓存 plugin（如 S3 缓存）将构建产物上传/下载

```yaml
steps:
  restore_cache:
    image: meltwater/drone-cache
    settings:
      restore: true
      mount:
        - .cache
      bucket: my-cache-bucket
    volumes:
      - cache:/cache        # 自定义 volume（非 workspace）

  build:
    image: golang
    commands:
      - go build ./...

  rebuild_cache:
    image: meltwater/drone-cache
    settings:
      rebuild: true
      mount:
        - .cache
      bucket: my-cache-bucket
```

### 17.7 自定义 Volume 与 Workspace 的区别

| 维度 | Workspace Volume | 自定义 Volume |
|------|-----------------|-------------|
| **生命周期** | SetupWorkflow 创建 → DestroyWorkflow 删除 | 由 YAML volumes 定义 |
| **挂载路径** | `/woodpecker`（固定） | 用户指定 |
| **共享范围** | 同一 workflow 所有 step | 同一 workflow 所有 step |
| **命名** | `{prefix}_default` | YAML 中定义 |
| **Docker** | Named Volume | Named Volume 或 Host Bind |
| **Kubernetes** | PVC（ReadWriteMany） | PVC 或 emptyDir |

---

## 18. Cron Scheduler 触发器调用栈

### 18.1 Cron 数据模型

**代码位置：`server/model/cron.go:24-36**

```go
type Cron struct {
    ID        int64             `xorm:"pk autoincr 'id'"`
    Name      string            `xorm:"name UNIQUE(s) INDEX"`
    RepoID    int64             `xorm:"repo_id UNIQUE(s) INDEX"`
    CreatorID int64             `xorm:"creator_id INDEX"`
    NextExec  int64             `xorm:"next_exec"`        // Unix 时间戳
    Schedule  string            `xorm:"schedule NOT NULL"` // cron 表达式
    Timezone  string            `xorm:"timezone NOT NULL DEFAULT 'UTC'"`
    Branch    string            `xorm:"branch"`
    Enabled   bool              `xorm:"enabled NOT NULL DEFAULT TRUE"`
    Variables map[string]string `xorm:"json 'variables'"` // 额外环境变量
}
```

**唯一约束**：`(Name, RepoID)` 联合唯一。

### 18.2 Cron 调度循环

**代码位置：`server/cron/cron.go:41-65**

```go
func Run(ctx context.Context, store store.Store) error {
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-time.After(checkTime):   // checkTime = 1 minute
            go func() {
                now := time.Now()
                crons, _ := store.CronListNextExecute(now.Unix(), checkItems)  // checkItems = 10
                for _, cron := range crons {
                    runCron(ctx, store, cron, now)
                }
            }()
        }
    }
}
```

**启动入口**：`cmd/server/server.go` 中作为后台服务启动。

### 18.3 Cron 执行流程（含分布式锁）

**代码位置：`server/cron/cron.go:88-113**

```
runCron(ctx, store, cron, now)
    │
    ├─ 1. 计算下次执行时间
    │     CalcNewNext(cron.Schedule, cron.Timezone, now)
    │     │
    │     ├─ time.LoadLocation(tzLoc)  → 时区转换
    │     ├─ cron.NewDefaultParser(StandardOptions) → 解析器
    │     └─ parser.Parse(schedule).Next(now) → 计算下次时间
    │
    ├─ 2. 获取分布式锁（防多实例重复执行）
    │     gotLock, err := store.CronGetLock(cron, newNext.Unix())
    │     │
    │     ├─ 锁获取成功 → 继续执行
    │     │    同时将 cron.NextExec 更新为 newNext
    │     │
    │     └─ 锁获取失败 → return nil
    │        （另一个 server 实例已经抢到了）
    │
    ├─ 3. 创建 Pipeline
    │     CreatePipeline(ctx, store, cron)
    │     │
    │     ├─ store.GetRepo(cron.RepoID)       → 获取仓库
    │     ├─ ForgeFromRepo(repo)               → 获取 forge 实例
    │     ├─ cron.Branch 空时 fallback 到 repo.Branch
    │     ├─ store.GetUser(repo.UserID)        → 获取仓库 owner
    │     ├─ forge.Refresh(ctx, _forge, store, repoUser) → 刷新 token
    │     ├─ _forge.BranchHead(ctx, repoUser, repo, cron.Branch) → 获取最新 commit
    │     │
    │     └─ 返回 &model.Pipeline{
    │            Event:               model.EventCron,        // 事件类型: cron
    │            Commit:              commit.SHA,
    │            Ref:                 "refs/heads/" + cron.Branch,
    │            Branch:              cron.Branch,
    │            Timestamp:           cron.NextExec,
    │            Cron:                cron.Name,
    │            ForgeURL:            commit.ForgeURL,
    │            AdditionalVariables: cron.Variables,          // cron 附加变量
    │        }
    │
    └─ 4. 提交 Pipeline
          pipeline.Create(ctx, store, repo, newPipeline)
          → 走标准 pipeline 创建流程（解析 YAML、入队、调度）
```

### 18.4 CronGetLock 的原子性保障

**数据库层**：`CronGetLock` 通过 CAS（Compare-And-Swap）实现乐观锁：

```sql
UPDATE crons
SET next_exec = {newNext}
WHERE id = {cronID}
  AND next_exec = {cron.NextExec}   -- CAS 条件：只有当前值未变时才更新
```

**多实例安全**：
- 多个 server 实例同时读取到同一个 cron 记录
- 只有第一个成功执行 CAS 更新的实例会获得锁
- 其他实例的 CAS 失败，gotLock=false，跳过执行

### 18.5 Cron 触发 vs Webhook 触发对比

| 维度 | Webhook 触发 | Cron 触发 |
|------|-------------|----------|
| **事件源** | Forge 推送 webhook | Server 内部定时器 |
| **Pipeline.Event** | push / pull_request / tag | cron |
| **Commit SHA** | webhook 携带 | `BranchHead()` 实时查询 |
| **触发分支** | webhook 决定 | cron.Branch 或 repo 默认分支 |
| **额外变量** | 无 | cron.Variables 注入 |
| **Pipeline.Cron** | 空 | cron.Name |
| **Token 刷新** | PostHook 中 Refresh() | CreatePipeline 中 Refresh() |

---

## 19. Forge Integration 与 Webhook 调用栈

### 19.1 Forge 接口定义

**代码位置：`server/forge/forge.go:56-178**

Forge 是 Woodpecker 与 Git 托管平台的统一抽象层。核心方法：

| 方法 | 用途 | 调用时机 |
|------|------|---------|
| `Login()` | OAuth2 登录 | 用户授权回调 |
| `Hook()` | 解析 webhook | `/api/hook` 接收推送 |
| `File()` | 获取仓库文件 | pipeline 创建时读取 `.woodpecker.yml` |
| `Dir()` | 获取目录文件列表 | 多文件 pipeline 配置 |
| `Status()` | 上报 workflow 状态 | 步骤状态变更时 |
| `Netrc()` | 生成 clone 凭证 | clone 步骤执行时 |
| `Activate()` | 注册 webhook | 用户激活仓库 |
| `Deactivate()` | 注销 webhook | 用户停用仓库 |
| `BranchHead()` | 获取分支最新 commit | cron 触发 |
| `Branches()` | 列出分支 | UI/API |
| `PullRequests()` | 列出 PR | UI/API |
| `Teams()` | 列出团队/组织 | 用户登录 |
| `OrgMembership()` | 检查组织成员权限 | 权限校验 |
| `Repo()`/`Repos()` | 获取仓库信息 | UI/API |

### 19.2 ForgeManager 多实例管理

**代码位置：`server/services/manager.go`**

Woodpecker 支持同时接入多个 Forge 实例（如 GitHub + GitLab）：

```
ForgeManager
    │
    ├─ forges: map[int64]forge.Forge   ← ForgeID → Forge 实例
    │
    ├─ ForgeFromRepo(repo) → forges[repo.ForgeID]
    │    └─ 每个仓库绑定到特定 Forge
    │
    ├─ ForgeAccessURL(forgeID) → forges[forgeID].URL()
    │
    └─ MainForge() → 主 Forge（用于全局操作）
```

### 19.3 Webhook 处理完整调用栈

**代码位置：`server/api/hook.go:69-213**

```
POST /api/hook
    │
    ├─ 1. 验证 Webhook Token
    │     token.ParseRequest([HookToken], req, func(t) {
    │         repo = getRepoFromToken(store, t)
    │         return repo.Hash, nil
    │     })
    │     │
    │     ├─ 从 query string 或 header 中提取 token
    │     ├─ 解析 JWT → 获取 forge-id + repo-forge-remote-id
    │     └─ store.GetRepoForgeID(forgeID, remoteID) → 定位仓库
    │
    ├─ 2. 路由到对应 Forge 解析
    │     _forge = ForgeFromRepo(repo)
    │     repoFromForge, pipelineFromForge = _forge.Hook(ctx, req)
    │     │
    │     ├─ GitHub:  github.Hook() → 解析 X-GitHub-Event header
    │     ├─ GitLab:  gitlab.Hook() → 解析 X-GitLab-Event header
    │     ├─ Gitea:   gitea.Hook()  → 解析 X-Gitea-Event header
    │     ├─ Forgejo: forgejo.Hook() → 解析 X-Forgejo-Event header
    │     └─ Bitbucket: bitbucket.Hook() → 解析 X-Event-Key header
    │
    │  Hook() 返回语义:
    │     (repo, pipeline, nil)       → 正常执行 pipeline
    │     (repo, nil, nil)            → 有效 webhook 但不触发 pipeline
    │     (nil, nil, ErrIgnoreEvent)  → 忽略的事件类型
    │     (nil, nil, error)           → 解析错误
    │
    ├─ 3. 安全校验
    │     ├─ repoFromForge.ForgeRemoteID == repo.ForgeRemoteID  → 防 webhook 伪造
    │     ├─ repo.IsActive  → 仓库必须已激活
    │     ├─ repo.UserID != 0  → 仓库必须有 owner
    │     └─ pipelineFromForge.IsPullRequest() → repo.AllowPull 检查
    │
    ├─ 4. 仓库信息同步
    │     ├─ FullName 变更 → 创建 Redirection
    │     └─ repo.Update(repoFromForge) → store.UpdateRepo(repo)
    │
    ├─ 5. Token 刷新
    │     forge.Refresh(ctx, _forge, store, user)
    │     └─ 如果 token 将在 30 分钟内过期 → 使用 refresh_token 刷新
    │
    └─ 6. 创建 Pipeline
          pipeline.Create(ctx, store, repo, pipelineFromForge)
```

### 19.4 Forge Hook 事件类型映射

以 GitHub 为例（`server/forge/github/convert.go`）：

| GitHub Event | Woodpecker Event | 触发条件 |
|-------------|-----------------|---------|
| `push` | `EventPush` | 分支推送 |
| `pull_request` (opened/synchronized) | `EventPullRequest` | PR 创建/更新 |
| `pull_request` (closed/merged) | 忽略 | PR 关闭 |
| `deployment` | `EventDeploy` | 部署触发 |
| `release` (published) | `EventRelease` | 发布创建 |
| `create` (tag) | `EventTag` | 标签创建 |

**事件过滤**（`pipeline/frontend/yaml/constraint/constraint.go`）：

YAML 中的 `when` 条件在编译期过滤步骤：

```yaml
steps:
  deploy:
    when:
      event: [push, deployment]
      branch: main
```

### 19.5 Forge Status 回调

**代码位置：`server/forge/forge.go:116` + `server/forge/common/status.go`**

当 workflow 状态变更时，Server 向 Forge 报告：

```
pipeline.Create() / step_status.Update() / pipeline.Cancel()
    │
    └─ updatePipelineStatus(ctx, forge, pipeline, repo, user)
         │
         ├─ forge.Status(ctx, user, repo, pipeline, workflow)
         │    │
         │    ├─ GitHub:  创建 Commit Status（pending/success/failure/error）
         │    ├─ GitLab:  创建 Pipeline Status
         │    ├─ Gitea:   创建 Commit Status
         │    └─ Forgejo: 创建 Commit Status
         │
         └─ 失败仅记录日志，不阻塞 pipeline 执行
```

**Status 与 Forge UI 的关系**：

```
Forge UI 显示:
  commit abc123  ──→  ✓ woodpecker/build   (success)
                      ✓ woodpecker/test    (success)
                      ✗ woodpecker/deploy  (failure)

每个 workflow 对应 Forge 中的一个 commit status，
status 名称 = "woodpecker/" + workflow.Name
```

### 19.6 Activate / Deactivate：Webhook 注册

**代码位置：`server/api/repo.go`**

```
POST /api/repos/{repo}/activate
    │
    ├─ forge.Activate(ctx, user, repo, webhookURL)
    │    │
    │    ├─ 生成 webhook URL:  {woodpecker_url}/hook?access_token={JWT}
    │    │    JWT 包含: forge-id + repo-forge-remote-id
    │    │
    │    └─ GitHub: 创建 Repository Webhook
    │       POST /repos/{owner}/{repo}/hooks
    │       events: [push, pull_request, deployment, release, create]
    │
    └─ repo.IsActive = true → store.UpdateRepo(repo)

DELETE /api/repos/{repo}
    │
    ├─ forge.Deactivate(ctx, user, repo, webhookURL)
    │    └─ 删除 Forge 侧的 Webhook
    │
    └─ repo.IsActive = false
```

### 19.7 Forge Token 刷新机制

**代码位置：`server/forge/refresh.go`**

```go
func Refresh(ctx context.Context, f forge.Forge, store store.Store, user *model.User) {
    if user.Expiry == 0 { return }  // 无过期时间 → 不需要刷新

    now := time.Now().Unix()
    expiresAt := user.Expiry

    // 提前 30 分钟刷新
    if now < expiresAt && now > expiresAt-refreshTokenBeforeExpiry {
        refreshedUser, err := f.Refresh(ctx, user)
        if err != nil {
            log.Error().Err(err)
            return
        }
        store.UpdateUser(refreshedUser)
    }
}
```

### 19.8 Netrc 凭证生成（私有仓库 Clone）

**代码位置：`server/forge/forge.go:120` + 各 forge 实现**

Clone 步骤需要访问私有仓库时，Forge 提供 `.netrc` 格式的凭证：

```go
// GitHub
func (g *GitHub) Netrc(u *model.User, r *model.Repo) (*model.Netrc, error) {
    return &model.Netrc{
        Machine:  "github.com",
        Login:    "x-oauth-basic",     // GitHub 固定用户名
        Password: u.AccessToken,       // OAuth token 作为密码
    }, nil
}

// GitLab
func (g *GitLab) Netrc(u *model.User, r *model.Repo) (*model.Netrc, error) {
    return &model.Netrc{
        Machine:  "gitlab.com",
        Login:    "oauth2",            // GitLab 固定用户名
        Password: u.AccessToken,
    }, nil
}
```

**Netrc 注入到 clone 步骤**（`server/pipeline/items.go` 编译期）：

```go
compiler.WithCloneEnv(map[string]string{
    "CI_NETRC_MACHINE":  netrc.Machine,
    "CI_NETRC_USERNAME": netrc.Login,
    "CI_NETRC_PASSWORD": netrc.Password,
})
```

Clone plugin（如 `woodpecker-plugin-git`）读取这些环境变量，写入 `.netrc` 文件后执行 git clone。

---

## 20. 完整事件流转总图

```
┌─────────────── Forge (GitHub/GitLab/Gitea/...) ──────────────┐
│                                                            │
│  Webhook Push ────► POST /api/hook                           │
│  Webhook PR   ────► POST /api/hook                           │
│  Cron Timer   ────► server/cron.Run() → BranchHead() → hook   │
│                                                            │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌─────────────── Server: Pipeline Create ──────────────────────┐
│                                                            │
│  pipeline.Create()                                          │
│    ├─ Forge.File() → 读取 .woodpecker.yml                   │
│    ├─ PipelineBuilder.Build() → Matrix 展开多 workflow       │
│    ├─ Compiler.Compile()                                     │
│    │    ├─ Secret 注入 (from_secret → getSecretValue)        │
│    │    ├─ Netrc 注入 (CI_NETRC_*)                            │
│    │    ├─ Workspace Volume 生成 ({prefix}_default:/woodpecker)│
│    │    ├─ Network 生成 ({prefix}_default)                    │
│    │    └─ Clone/Service/Commands/Plugin 步骤生成              │
│    ├─ 审批检查 → Blocked/Pending                               │
│    └─ queuePipeline() → PushAtOnce() → 入队                  │
│                                                            │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌─────────────── Queue: 调度 ──────────────────────────────────┐
│                                                            │
│  FIFO.process() 每 100ms                                    │
│    ├─ 依赖检查 depsInQueue()                                 │
│    ├─ assignToWorker() → label filter + 评分                  │
│    ├─ worker.channel <- task → Agent Poll() 收到             │
│    └─ resubmitExpiredPipelines() → lease 过期重入队            │
│                                                            │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌─────────────── Agent: 执行 ──────────────────────────────────┐
│                                                            │
│  Runner.Run() 循环                                           │
│    ├─ client.Next() → 获取 workflow                         │
│    ├─ client.Init() → Pipeline: Pending→Running             │
│    ├─ 并发: Wait() + Extend() + Runtime.Run()                │
│    │    │                                                    │
│    │    └─ Runtime.Run()                                     │
│    │       ├─ engine.SetupWorkflow()                         │
│    │       │    ├─ Docker: VolumeCreate + NetworkCreate       │
│    │       │    └─ K8s: PVC + HeadlessService                │
│    │       │                                                 │
│    │       ├─ for stage in stages:                           │
│    │       │    runStage() → errgroup 并行执行 steps          │
│    │       │    │                                            │
│    │       │    └─ executeStep(step)                         │
│    │       │       ├─ shouldSkipStep(OnFailure/OnSuccess)    │
│    │       │       ├─ engine.StartStep()                     │
│    │       │       ├─ engine.TailStep() → 日志流              │
│    │       │       ├─ engine.WaitStep() → 等待完成            │
│    │       │       ├─ engine.DestroyStep()                    │
│    │       │       └─ traceStep() → client.Update() 上报状态  │
│    │       │                                                 │
│    │       └─ engine.DestroyWorkflow()                        │
│    │            ├─ Docker: VolumeRemove + NetworkRemove       │
│    │            └─ K8s: PVC 删除 + Service 删除               │
│    │                                                         │
│    └─ client.Done() → Workflow→终态, Pipeline 汇总            │
│                                                            │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌─────────────── Server: 状态汇总 ──────────────────────────────┐
│                                                            │
│  RPC.Done() / RPC.Update()                                  │
│    ├─ CalcStepStatus() → Step 终态                          │
│    ├─ UpdateWorkflowStatusToDone() → Workflow 终态           │
│    ├─ PipelineStatus() → MergeStatusValues() → Pipeline 终态 │
│    ├─ forge.Status() → Forge UI 更新                         │
│    ├─ publishToTopic() → PubSub 通知                         │
│    └─ scheduler.Done() → 队列清理, 触发下游 workflow           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```
