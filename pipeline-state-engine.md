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
