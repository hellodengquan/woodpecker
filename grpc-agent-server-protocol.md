# Woodpecker Agent-Server gRPC 协议分析

## 1. 架构总览

Woodpecker CI 采用 **Agent-Server** 架构，通过 gRPC **一元 RPC（Unary RPC）** 进行通信，而非流式 RPC。
虽然文档标题提到"流式同步"，但实际协议使用的是多个阻塞式的一元调用模拟长连接语义。

```
┌───────────────────────────────────────┐         ┌───────────────────────────────────────┐
│           Agent (执行端)              │         │           Server (编排端)             │
│                                       │         │                                       │
│  ┌─────────────┐    ┌──────────────┐  │  gRPC   │  ┌──────────────┐    ┌─────────────┐  │
│  │   Runner    │───▶│ Client (gRPC)│──┼────────▶│  │WoodpeckerServ│───▶│  RPC (业务) │  │
│  └─────────────┘    └──────────────┘  │◀────────│  │     er       │    └─────────────┘  │
│         │                              │         │  └──────────────┘          │          │
│         │ EnqueueLog                   │         │                            ▼          │
│         ▼                              │         │                   ┌─────────────┐    │
│  ┌─────────────┐                       │         │                   │ Scheduler   │    │
│  │ logs channel│──── processLogs ──────┼────────▶│                   │  ┌───────┐  │    │
│  └─────────────┘                       │         │                   │  │ Queue │  │    │
│                                       │         │                   │  └───────┘  │    │
│                                       │         │                   └─────────────┘    │
└───────────────────────────────────────┘         └───────────────────────────────────────┘
```

### 核心文件定位

| 角色 | 文件 | 作用 |
|------|------|------|
| 协议定义 | `rpc/proto/woodpecker.proto:26-38` | gRPC 服务与消息定义 |
| 统一接口 | `rpc/peer.go:70-315` | Peer 接口，定义所有通信方法 |
| Agent 客户端 | `agent/rpc/client_grpc.go:55-442` | gRPC 客户端实现 + 重试/日志批处理 |
| Server gRPC 适配层 | `server/rpc/server.go:34-218` | protobuf ⇄ 内部类型转换 |
| Server 业务逻辑 | `server/rpc/rpc.go:53-632` | 状态校验、DB 更新、队列操作 |
| 任务队列 | `server/queue/fifo.go:45-424` | FIFO 内存队列，Poll/Wait/Done/Extend |
| Agent 主循环 | `cmd/agent/core/agent.go:52-321` | 连接建立、Runner 池、心跳 |
| Agent 工作流执行 | `agent/runner.go:63-217` | 单工作流执行、取消监听、心跳续期 |
| Step 状态追踪 | `agent/tracer.go:30-60` | 将 step 状态通过 Update RPC 上报 |

---

## 2. gRPC 服务定义

定义于 `rpc/proto/woodpecker.proto:26-38`：

```protobuf
service Woodpecker {
  rpc Version         (Empty)                returns (VersionResponse) {}   // 版本检查
  rpc Next            (NextRequest)          returns (NextResponse) {}      // 拉取任务（阻塞）
  rpc Init            (InitRequest)          returns (Empty) {}             // 标记工作流已开始
  rpc Wait            (WaitRequest)          returns (WaitResponse) {}      // 等待取消信号（阻塞）
  rpc Done            (DoneRequest)          returns (Empty) {}             // 标记工作流完成
  rpc Extend          (ExtendRequest)        returns (Empty) {}             // 续期任务租约
  rpc Update          (UpdateRequest)        returns (Empty) {}             // 上报 Step 状态
  rpc Log             (LogRequest)           returns (Empty) {}             // 批量上报日志
  rpc RegisterAgent   (RegisterAgentRequest) returns (RegisterAgentResponse) {}
  rpc UnregisterAgent (Empty)                returns (Empty) {}
  rpc ReportHealth    (ReportHealthRequest)  returns (Empty) {}             // 心跳
}
```

### 关键数据结构

#### WorkflowState — 工作流状态 (`rpc/proto/woodpecker.proto:55-60`)
```protobuf
message WorkflowState {
  int64  started = 1;   // 开始时间戳
  int64  finished = 2;  // 结束时间戳
  string error = 3;     // 错误信息
  bool   canceled = 4;  // 是否被取消
}
```

#### StepState — 步骤状态 (`rpc/proto/woodpecker.proto:44-53`)
```protobuf
message StepState {
  string step_uuid = 1;
  int64  started = 2;
  int64  finished = 3;
  bool   exited = 4;
  int32  exit_code = 5;
  string error = 6;
  bool   canceled = 7;
  bool   skipped = 8;
}
```

#### LogEntry — 日志条目 (`rpc/proto/woodpecker.proto:62-68`)
```protobuf
message LogEntry {
  string step_uuid = 1;
  int64  time = 2;       // 相对 step 启动的秒数
  int32  line = 3;       // 行号
  int32  type = 4;       // 0=stdout, 1=stderr, 2=exit-code, 3=metadata, 4=progress
  bytes  data = 5;
}
```

---

## 3. Agent 生命周期消息流

### 3.1 启动阶段

```
Agent                                          Server
  │                                               │
  │──1. Auth(agent_token, agent_id)─────────────▶│  WoodpeckerAuth.Auth
  │◀──────────────────────(access_token)─────────│
  │                                               │
  │──2. Version()───────────────────────────────▶│  Woodpecker.Version
  │◀────(grpc_version, server_version)───────────│  校验版本兼容
  │                                               │
  │──3. RegisterAgent(AgentInfo)────────────────▶│  Woodpecker.RegisterAgent
  │◀──────────────────────(agent_id)─────────────│  创建/更新 agent 记录
  │                                               │
  │──4. 启动 ReportHealth 协程 (每10s)───────────│
  │    ReportHealth("I am alive!")──────────────▶│  Woodpecker.ReportHealth
  │◀──────────────────────(Empty)────────────────│  更新 LastContact
  │                                               │
  │──5. 启动 N 个 Runner (每 capacity 一个)──────│
```

**关键代码**：
- 连接建立与认证：`cmd/agent/core/agent.go:106-126`
- 版本检查：`cmd/agent/core/agent.go:145-158`
- Agent 注册：`cmd/agent/core/agent.go:189-198`
- 心跳协程：`cmd/agent/core/agent.go:245-264`，间隔 `reportHealthInterval = 10s`

### 3.2 关闭阶段

```
Agent                                          Server
  │                                               │
  │  (收到 SIGTERM / agentCtx canceled)           │
  │   │                                           │
  │   ├── 停止新建 Runner 循环                     │
  │   ├── 等待进行中工作流完成                      │
  │   │                                           │
  │   └── UnregisterAgent()─────────────────────▶│  仅对系统 token agent 生效
  │◀──────────────────────(Empty)────────────────│  store.AgentDelete()
```

**关键代码**：`cmd/agent/core/agent.go:200-220`

---

## 4. 单个 Workflow 完整消息往返

这是最核心的流程，每个 Runner 协程循环执行：

### 4.1 时序图

```
Agent (Runner)                                 Server (RPC + Queue)
      │                                               │
      │                                               │
      │  ┌─── Next(filter) ─────────────────────────▶│  RPC.Next()
      │  │  (阻塞调用)                                │    │
      │  │                                            │    ├── scheduler.Poll()
      │  │                                            │    │   注册 worker，阻塞等待
      │  │                                            │    │   (每 100ms 队列调度匹配)
      │  │  ◀──── 队列 matched，推送 Task ────────────│    │
      │  │  NextResponse{Workflow}                    │    └── 返回 workflow(json反序列化)
      │  │                                            │
      │  │  [启动两个后台协程]                         │
      │  │                                            │
      │  ├── Init(workflowID, Started=now) ──────────▶│  RPC.Init()
      │  │  ◀──────────────── Empty ──────────────────│    - 校验工作流状态
      │  │                                            │    - DB: workflow → running
      │  │                                            │    - 更新 forge 状态(Git commit status)
      │  │                                            │    - PubSub 通知前端
      │  │                                            │
      │  ├── [Wait 协程]                              │
      │  │    Wait(workflowID) ─────────────────────▶│  RPC.Wait()
      │  │    (阻塞)                                  │    │
      │  │                                            │    └── scheduler.Wait()
      │  │                                            │         监听 done channel
      │  │                                            │
      │  ├── [Extend 协程]                            │
      │  │    every TaskTimeout/3                     │
      │  │    Extend(workflowID) ───────────────────▶│  RPC.Extend()
      │  │    ◀────────────── Empty ─────────────────│    scheduler.Extend()
      │  │                                            │    deadline = now + TaskTimeout
      │  │                                            │
      │  │  [执行各个 Step]                            │
      │  │                                            │
      │  │  ┌── Step A 启动 ──┐                       │
      │  │  │                 │                       │
      │  │  Update(step, Exited=false) ─────────────▶│  RPC.Update()
      │  │  ◀──────────── Empty ─────────────────────│    - StepState 校验
      │  │  │                 │                       │    - DB: step → running
      │  │  │                 │                       │    - PubSub 通知
      │  │  │   [Log Streaming]                       │
      │  │  │   EnqueueLog() → 内部缓冲                │
      │  │  │                 │                       │
      │  │  │    每 1s 或 1MB 批量发送                 │
      │  │  │    Log([LogEntry...]) ────────────────▶│  RPC.Log()
      │  │  │    ◀────────── Empty ──────────────────│    - 按 stepUUID 分组写入
      │  │  │                 │                       │    - logging.Write() → PubSub
      │  │  │                 │                       │    - LogStore 持久化
      │  │  │   [Step 完成]                           │
      │  │  │                 │                       │
      │  │  Update(step, Exited=true, ExitCode=X) ──▶│  RPC.Update()
      │  │  ◀──────────── Empty ─────────────────────│    - Step → success/failure
      │  │  └─────────────────┘                       │    - 通知 forge
      │  │                                            │
      │  │  ┌── Step B (并行/串行) ... ─┐             │
      │  │  └───────────────────────────┘             │
      │  │                                            │
      │  ├── Done(workflowID, FinalState) ───────────▶│  RPC.Done()
      │       Finished, Error, Canceled               │    - Workflow → done/killed
      │  ◀──────────────── Empty ─────────────────────│    - scheduler.Done() → 关闭 done channel
      │                                               │    - [Wait 协程返回]
      │                                               │    - Pipeline 状态聚合
      │                                               │    - 更新 forge 状态
      │                                               │    - 关闭日志流
      │                                               │    - PubSub 通知
      │                                               │
      │  [Wait 协程被释放]                             │
      │    if canceled=false → 正常结束                │
      │    if canceled=true  → 不应发生（Done在前）    │
```

### 4.2 关键 RPC 详细说明

#### Next — 拉取任务（长轮询）

**Agent 端** (`agent/rpc/client_grpc.go:208-234`)：
- 发送 `Filter{Labels}`（hostname/platform/backend/自定义标签）
- 调用阻塞直到收到任务或 context 取消
- 内置 `retryRPC` 重试机制（指数退避）

**Server 端** (`server/rpc/rpc.go:63-108` + `server/queue/fifo.go:87-111`)：
1. 从 gRPC metadata 提取 agent 身份
2. 合并服务端强制标签到 agent 标签
3. 调用 `scheduler.Poll()` 注册一个 `worker{channel, filter}`
4. **队列调度循环**每 100ms (`processTimeInterval`) 运行一次：
   - `resubmitExpiredPipelines()`：超时任务重新入队
   - `filterWaiting()`：依赖完成的任务移入 pending
   - `assignToWorker()`：遍历 pending，匹配 worker.filter，命中则通过 channel 发送

> ⚠️ **重点**：Next 不是 gRPC Server Streaming，而是靠 Server 端 goroutine 阻塞在 channel 上实现的"伪流式"。

#### Wait — 监听取消信号

**Agent 端** (`agent/runner.go:117-131`)：
- 在独立 goroutine 中调用
- 返回 `canceled=true` 时调用 `cancelWorkflowCtx(ErrCancel)` 停止执行
- context 取消（如 agent shutdown）仅退出协程，不等同于 workflow 取消

**Server 端** (`server/rpc/rpc.go:112-136` + `server/queue/fifo.go:164-183`)：
1. 校验 agent 对 workflow 的权限
2. 从 `running` map 中取出对应 `entry{done channel}`
3. `select` 阻塞在 `entry.done` 或 `ctx.Done()`
4. **触发释放的两种情况**：
   - `scheduler.Error()` 且错误为 `ErrCancel` → 返回 `canceled=true`
   - `scheduler.Done()` → 返回 `canceled=false`（正常完成）

#### Extend — 心跳续期

**Agent 端** (`agent/runner.go:134-148`)：
- 独立 goroutine，每 `TaskTimeout / 3` 调用一次
- 失败只记录日志不中止 workflow

**Server 端** (`server/queue/fifo.go:186-200`)：
- `state.deadline = time.Now().Add(q.extension)`
- 验证 `AgentID` 匹配，防止其他 agent 续期
- 若 deadline 过期仍未续期 → 下次 process 循环中 `resubmitExpiredPipelines()` 将其重新入队

#### Update — Step 状态上报

**触发时机** (`pipeline/runtime/step.go:267-292` → `agent/tracer.go:30-60`)：
1. **Step 开始**：`Exited=false, Started=now`
2. **Step 跳过**：`Skipped=true`
3. **Step 完成**：`Exited=true, Finished=now, ExitCode`
4. **Step 启动失败**：`Exited=true, Error`

**Server 端** (`server/rpc/rpc.go:158-226`)：
1. 解析 workflowID（string → int64）
2. 查 DB 加载 workflow/pipeline/repo/step
3. **状态校验** `checkWorkflowAllowsStepUpdate()`：只有 workflow 处于运行态才允许更新
4. `pipeline.UpdateStepStatus()` 更新 DB
5. step 完成时调用 `LogStore.StepFinished()`
6. `notify()` PubSub 广播给前端/订阅者

#### Log — 批量日志上报

**Agent 端缓冲策略** (`agent/rpc/client_grpc.go:351-396`)：
- `logs` channel 容量 10（`agent/rpc/client_grpc.go:70`）
- `processLogs` goroutine 聚合：
  - **大小触发**：累计 protobuf 序列化大小 ≥ 1MB (`maxLogBatchSize`)
  - **时间触发**：每 1s 刷新一次 (`maxLogFlushPeriod`)
  - **channel 关闭**：context done 时强制 flush
- 单次发送失败直接丢弃（内存有限，不做无限重试）

**Server 端** (`server/rpc/server.go:155-190` → `server/rpc/rpc.go:415-475`)：
1. **按 stepUUID 分组**：同批中不同 step 的日志拆分为多次 `peer.Log()` 调用
2. `allowAppendingLogs()` 校验 pipeline/step 状态允许追加
3. **异步 PubSub**：`go s.logger.Write()` 推送给 WebSocket 订阅者
4. **同步持久化**：`LogStore.LogAppend()` 写入 DB

#### Done — 工作流完成

**Agent 端** (`agent/runner.go:186-214`)：
- workflow 执行结束（成功/失败/取消）后调用
- 若 `runnerCtx` 已取消，则使用独立的 `shutdownCtx`（5秒超时）
- 填充 `WorkflowState{Started, Finished, Error, Canceled}`

**Server 端** (`server/rpc/rpc.go:296-411`)：
1. 加载 workflow 及所有 step
2. 校验 workflow 状态（未 finish/blocked）
3. `completeChildrenIfParentCompleted()`：强制结束还在 running 的子 step（如 service 容器）
4. `pipeline.UpdateWorkflowStatusToDone()` 更新 DB
5. **队列处理**：
   - 非取消 + 失败 → `scheduler.Error(workflowError)`
   - 非取消 + 成功 → `scheduler.Done(workflow.State)`
   - 已取消 + 已启动 → `scheduler.Done(StatusKilled)`
   - 已取消 + 未启动 → `scheduler.Done(StatusCanceled)`
   - **核心作用**：关闭 `entry.done` channel → 释放 Wait 调用
6. 若所有 workflow 完成 → `pipeline.UpdateStatusToDone()` 聚合 Pipeline 最终状态
7. 更新 forge 状态、关闭日志流、PubSub 通知、记录 Prometheus 指标

---

## 5. 取消流程的消息往返

### 5.1 从 Server 端发起取消（UI/API 点击取消）

```
User/API → Server 内部调用 queue.Error(workflowID, ErrCancel)
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │ queue.finished()      │
                              │  - 设置 entry.error   │
                              │  - close(entry.done)  │
                              └──────────┬────────────┘
                                         │
                ┌────────────────────────┼────────────────────────┐
                ▼                        ▼                        ▼
          Agent1 Wait()              Agent2 Wait()           Agent3 Wait()
     返回 canceled=true         返回 canceled=true       ...
          │                         │
          ▼                         ▼
cancelWorkflowCtx(ErrCancel)  cancelWorkflowCtx(ErrCancel)
          │                         │
          ▼                         ▼
pipeline runtime 停止执行      pipeline runtime 停止执行
          │                         │
          ▼                         ▼
Done(Canceled=true)          Done(Canceled=true)
          │                         │
          ▼                         ▼
scheduler.Done(StatusKilled)  ...(幂等，entry 已删除)
```

**关键代码**：
- 队列释放所有 Wait 阻塞：`server/queue/fifo.go:133-160`
- Agent 端收到取消后中止：`agent/runner.go:124-126`

### 5.2 从 Agent 端异常断开

```
Agent 崩溃 / 网络中断
        │
        ▼
  Server ReportHealth 超时 / Next context 断开
        │
        ▼
  队列 process() 循环中：
  resubmitExpiredPipelines()
    for taskID, state in running:
      if now.After(state.deadline):
        pending.PushFront(state.item)   // 重新入队
        delete(running, taskID)
        close(state.done)              // 释放 Wait（若还在等）
        state.error = ErrTaskExpired
        │
        ▼
  另一个可用 Agent 的 Poll 匹配到任务 → 重新执行
```

---

## 6. 重试与容错机制

### 6.1 Agent 端 RPC 重试 (`agent/rpc/client_grpc.go:129-166`)

**统一重试函数** `retryRPC[T]`：
- 指数退避：`InitialInterval=10ms`, `MaxInterval=10s`
- 最大超时：由 `connectionRetryTimeout` 配置（cli flag `--retry-timeout`）
- 可重试 gRPC code：`Aborted, DataLoss, DeadlineExceeded, Internal, Unavailable`
- 不可重试（Permanent）：其余所有 code 及本地 context 取消

**特殊处理**：
- `Next/Wait/Update/Init/Done/Extend/ReportHealth`：全部走 `retryRPC`
- `Log`：也走重试，但只在 `processLogs` 聚合层触发，**失败则丢弃**（`agent/rpc/client_grpc.go:369`）
- `IsConnected()` 为 false 时返回 `errNotConnected` → Retry 会等并重试

### 6.2 Server 端状态校验

每次 Agent 上报状态前，Server 都会执行：

| 校验方法 | 位置 | 作用 |
|---------|------|------|
| `getAgentFromContext()` | `server/rpc/rpc.go:596-607` | 从 JWT 解析 agentID，查 DB |
| `checkAgentPermissionByWorkflow()` | 多次调用 | 确保 agent 是该 workflow 的执行者 |
| `checkWorkflowState()` | Init/Done | 拒绝在 finish/blocked/canceled 状态上重复操作 |
| `checkWorkflowAllowsStepUpdate()` | Update | workflow 非运行态拒绝 step 更新 |
| `allowAppendingLogs()` | Log | pipeline/step 已完成拒绝追加日志 |

---

## 7. 消息往返完整路径表

下表列出了"从 Agent 哪一行发起 → 经过哪几层 → 到达 Server 哪一行核心逻辑"：

| Agent 发起 | Client 封装 | gRPC 方法 | Server 适配层 | Server 核心逻辑 | 队列操作 |
|-----------|------------|-----------|--------------|----------------|---------|
| `cmd/agent/core/agent.go:146` `client.Version()` | `client_grpc.go:196-205` | `Version` | `server.go:70-75` | — | — |
| `cmd/agent/core/agent.go:189` `client.RegisterAgent()` | `client_grpc.go:411-424` | `RegisterAgent` | `server.go:192-205` | `rpc.go:477-501` store.AgentUpdate | — |
| `cmd/agent/core/agent.go:209` `client.UnregisterAgent()` | `client_grpc.go:426-429` | `UnregisterAgent` | `server.go:208-211` | `rpc.go:504-518` store.AgentDelete | — |
| `cmd/agent/core/agent.go:247` `client.ReportHealth()` | `client_grpc.go:431-442` retryRPC | `ReportHealth` | `server.go:214-218` | `rpc.go:520-534` update LastContact | — |
| `agent/runner.go:71` `r.client.Next()` | `client_grpc.go:208-234` retryRPC | `Next` | `server.go:78-95` | `rpc.go:63-108` | `queue.Poll()` 注册 worker |
| `agent/runner.go:120` `r.client.Wait()` | `client_grpc.go:237-254` retryRPC | `Wait` | `server.go:141-146` | `rpc.go:112-136` | `queue.Wait()` 监听 done |
| `agent/runner.go:143` `r.client.Extend()` | `client_grpc.go:301-312` retryRPC | `Extend` | `server.go:149-153` | `rpc.go:139-155` | `queue.Extend()` 更新 deadline |
| `agent/runner.go:154` `r.client.Init()` | `client_grpc.go:257-276` retryRPC | `Init` | `server.go:98-107` | `rpc.go:229-293` | — |
| `agent/tracer.go:58` `r.client.Update()` | `client_grpc.go:315-338` retryRPC | `Update` | `server.go:110-124` | `rpc.go:158-226` | — |
| `agent/runner.go:210` `r.client.Done()` | `client_grpc.go:279-298` retryRPC | `Done` | `server.go:127-137` | `rpc.go:296-411` | `queue.Done()/Error()` close(done) |
| `agent/log/line_writer.go:69` `peer.EnqueueLog()` | `client_grpc.go:341-349` chan缓冲 → `processLogs`批量 → `sendLogs:398-409` retryRPC | `Log` | `server.go:155-190` (按stepUUID分组) | `rpc.go:415-475` | — |

---

## 8. 设计要点总结

### 8.1 为什么用"阻塞一元调用"而非 gRPC Streaming？

1. **简单性**：每个方向的语义（Next=拉任务, Wait=等取消）是**单一事件**，不需要 stream 的多消息语义
2. **错误隔离**：每个 RPC 独立重试，一个 stream 断开会丢失所有上下文
3. **队列模型天然契合**：Go channel + 阻塞 select 是 Go 原生模式，直接映射到 RPC Handler 的 goroutine 阻塞

### 8.2 关键同步原语

| 原语 | 位置 | 作用 |
|------|------|------|
| `worker.channel` | `server/queue/fifo.go:38-43` | 向阻塞的 Poll 推送任务 |
| `entry.done` | `server/queue/fifo.go:31-36` | 向阻塞的 Wait 发完成/取消信号 |
| `fifo.Mutex` | `server/queue/fifo.go:46` | 保护 pending/running/workers 等结构 |
| `client.logs chan` | `agent/rpc/client_grpc.go:58` | 日志异步批量缓冲 |
| `workflowCtx` cancel | `agent/runner.go:103` | Wait 返回 canceled=true 时中止整个执行 |

### 8.3 容易混淆的概念

| 概念 | 说明 | 对应实现 |
|------|------|---------|
| **Workflow.Timeout** | 用户配置的单 workflow 最大执行时长 | `agent/runner.go:80-83` 的 context.WithTimeout |
| **TaskTimeout (constant)** | 队列租约超时，用于死 agent 检测 | `server/queue/fifo.go:69` 的 `q.extension`，Agent 每 TaskTimeout/3 调用 Extend |
| **Wait canceled=true** | Server 主动取消（UI/API） | `queue.ErrCancel` 错误在 Wait 中被特殊处理 |
| **context.Canceled** | 进程/协程退出，与业务取消无关 | 所有 retryRPC 中特殊处理，返回 (zero, nil) |

---

## 9. 长连接断开重连及状态恢复机制

本章节详解 gRPC 连接在网络波动、Agent 重启、Server 滚动升级等场景下的断连恢复与状态重建机制。

### 9.1 连接分层架构

Woodpecker Agent 与 Server 之间建立 **两条独立的 gRPC 连接**，各自承担不同职责：

```
Agent                                          Server
  │                                               │
  │  ┌─ AuthConn (端口复用) ─┐                     │
  │  │ 用途: 仅 Auth RPC     │                     │
  │  │ 生命周期: 与 agent    │                     │
  │  │  进程同生命周期       │                     │
  │  └───────────────────────┘                     │
  │                                               │
  │  ┌─ MainConn (端口复用) ─┐                     │
  │  │ 用途: 所有业务 RPC     │                     │
  │  │ (Next/Init/Wait/...)  │                     │
  │  │ 生命周期: 自动重连     │                     │
  │  └───────────────────────┘                     │
```

**关键代码**：`agent/rpc/dial.go:45-111`，两条连接共享相同的 `keepalive` 参数和 TLS 配置。

### 9.2 gRPC 层自动重连（透明恢复）

gRPC Go 客户端内置了连接状态机和自动重连逻辑，无需应用层干预：

```
┌─────────┐      ┌──────────┐      ┌──────────┐
│  Ready  │─────▶│Transient │─────▶│  Idle    │
│ (正常)  │      │ Failure  │      │ (空闲)   │
└─────────┘      └──────────┘      └──────────┘
     ▲                │
     └── 指数退避重试 ──┘
```

**Agent 端连接状态检测** (`agent/rpc/client_grpc.go:91-96`)：
```go
func (c *client) IsConnected() bool {
    state := c.conn.GetState()
    return state == connectivity.Ready || state == connectivity.Idle
}
```

**重要**：当连接状态为 `TransientFailure` 或 `Connecting` 时，`retryRPC` 会主动返回 `errNotConnected` 触发上层等待（`agent/rpc/client_grpc.go:115-118`），避免无意义的 RPC 调用。

### 9.3 Keepalive 心跳参数配置

| 配置项 | Agent 端 | Server 端 | 作用 |
|-------|---------|----------|------|
| `KeepaliveTime` | `--grpc-keepalive-time` (default: ?) | — | 空闲多久后发送 PING 帧 |
| `KeepaliveTimeout` | `--grpc-keepalive-timeout` (default: ?) | — | PING 后等待响应超时 |
| `KeepaliveMinTime` | — | `--keepalive-min-time` | Server 强制的最小 PING 间隔（防攻击） |

**Agent 端配置** (`agent/rpc/dial.go:78-81`)：
```go
keepaliveOpts := grpc.WithKeepaliveParams(keepalive.ClientParameters{
    Time:    cfg.KeepaliveTime,
    Timeout: cfg.KeepaliveTimeout,
})
```

**Server 端强制策略** (`server/rpc/serve.go:62-64`)：
```go
grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
    MinTime: cfg.KeepaliveMinTime,
})
```

> ⚠️ **已知坑点**：Agent 发送 PING 过于频繁会触发 Server 返回 `GOAWAY` 错误 `"too_many_pings"`。代码中已做降级处理，仅输出 Trace 日志（`agent/rpc/client_grpc.go:110-113`）。

### 9.4 Token 自动刷新（连接重连后的身份恢复）

重连后必须重新获取 access token，否则所有业务 RPC 都会报 `Unauthenticated`。

**双 Token 架构** (`server/rpc/authorizer.go:17-46`)：
1. **Agent Token**（长寿命）：配置在启动参数，仅用于 `WoodpeckerAuth.Auth()`
2. **JWT Access Token**（短寿命，默认 1 小时）：通过 `Auth()` 获取，注入到所有业务 RPC 的 `metadata["token"]`

**自动刷新机制** (`agent/rpc/auth_interceptor.go:78-103`)：
```
scheduleRefreshToken(ctx, refreshInterval=30min)
    ├── 立即调用 refreshToken() 获取初始 token
    └── 后台循环:
        ├── 成功: 等待 refreshInterval 后再次刷新
        └── 失败: 等待 1s 后重试（快速重试）
```

**关键代码**：
- Token 刷新协程：`agent/rpc/auth_interceptor.go:84-100`
- Auth 请求超时：`agent/rpc/auth_client_grpc.go:26` 的 `authClientTimeout = 5s`
- Interceptor 注入：`agent/rpc/dial.go:98-99` 的 `grpc.WithUnaryInterceptor`

### 9.5 Agent ID 持久化（重连后身份不变）

Agent 重启或重连后，为避免在 Server 端创建大量孤儿 agent 记录，Agent ID 会持久化到本地文件：

**恢复流程** (`cmd/agent/core/agent.go:104-135`):
1. 启动时读取 `agent-config.json` 中的 `AgentID`（若无则为 -1）
2. `Auth()` 时带上 `AgentID` → Server 若 ID 存在则复用，不存在则创建
3. `RegisterAgent()` 返回最终的 `AgentID`
4. 写回 `agent-config.json` 持久化

**关键代码**：
- 配置读取：`cmd/agent/core/agent.go:104` `readAgentConfig()`
- 配置写回：`cmd/agent/core/agent.go:130-135`, `222-226`
- Auth 带上已有 ID：`agent/rpc/auth_client_grpc.go:48-65`

### 9.6 运行时工作流的状态恢复

当 gRPC 连接短暂断开又恢复时，**正在运行的工作流不会丢失**：

| 场景 | 恢复行为 | 关键机制 |
|------|---------|---------|
| **Next 阻塞中断** | `retryRPC` 自动重连重试 | `retryRPC` 的指数退避 + 可重试 code 判定 |
| **Wait 阻塞中断** | 同上，连接恢复后继续等待 | 不丢失取消信号，因为信号在 Server 队列中持久化 |
| **Update/Log 发送失败** | `retryRPC` 自动重试 | 但 Log 发送失败最终会丢弃（内存有限） |
| **Extend 中断** | 失败只打日志，下一轮继续 | 只要在 `TaskTimeout` 内恢复就不会被当死 agent |
| **Done 发送失败** | 使用独立 `shutdownCtx` (5s 超时) 重试 | `agent/runner.go:203-208`，尽力保证最终状态上报 |

### 9.7 队列任务持久化（Server 重启恢复）

Server 重启后，内存队列会丢失所有 pending/running 任务。`persistentQueue` 通过 DB 备份解决此问题：

**持久化流程** (`server/queue/persistent.go:32-111`):
```
PushAtOnce(tasks)
    ├── 先写入 store.TaskInsert() 到 DB
    └── 再写入内存队列
        └── 失败则回滚 DB 删除

Poll() → task
    ├── 从内存队列取出
    └── store.TaskDelete(task.ID) 从 DB 删除
```

**Server 启动恢复** (`server/queue/persistent.go:32-38`):
```go
tasks, _ := s.TaskList()    // 从 DB 读出所有未完成任务
q.PushAtOnce(ctx, tasks)    // 重新入队
```

### 9.8 连接彻底丢失的兜底（Agent 退出）

当 `retryRPC` 的 `MaxElapsedTime`（即 `--retry-timeout`）耗尽仍未恢复连接时，Agent 会优雅退出：

**触发路径** (`cmd/agent/core/agent.go:277-282`):
```
runner.Run() 失败
    ↓
errors.Is(err, ErrConnectionLost) 为 true
    ↓
log.Error("connection to server lost, shutting down agent")
    ↓
ctxCancel(err) → 所有 Runner 协程退出 → 等待进行中工作流结束 → 进程退出
```

---

## 10. 消息背压时的反向限流处理

本章节分析在高并发场景下（如大量 Agent 同时上报、日志爆发式输出），系统各层如何处理消息积压与反向限流。

### 10.1 背压发生的三个层次

```
Agent 端                        网络层                       Server 端
┌──────────┐                  ┌──────────┐                  ┌──────────┐
│ 日志生产 │─── 积压点1 ─────▶│ gRPC     │─── 积压点2 ─────▶│ RPC      │
│ (Step    │                  │ HTTP/2   │                  │ 处理     │
│  stdout) │  logs channel    │ 流控     │  服务端处理能力   │  goroutine│
└──────────┘  容量=10         │ WINDOW   │  不足            │  池耗尽  │
                              │ UPDATE   │                  └────┬─────┘
                              └──────────┘                       │
                                                                 ▼
                                                          ┌────────────┐
                                                          │ PubSub     │─── 积压点3
                                                          │ 订阅者     │    Web 客户端
                                                          │ 消费缓慢   │    消费不过来
                                                          └────────────┘
```

### 10.2 Agent 端日志背压：logs channel 满阻塞

**Agent 日志通道配置** (`agent/rpc/client_grpc.go:70`):
```go
client.logs = make(chan *proto.LogEntry, 10) // 容量仅 10
```

**阻塞行为分析**：
- 正常路径：`LineWriter.Write()` → `EnqueueLog()` → `c.logs <- entry`（非阻塞，因为有缓冲）
- **背压触发**：当 `processLogs` 发送慢（网络拥塞/Server 慢），而日志生产速度 > 发送速度，10 个槽位占满后
  - `EnqueueLog()` 会 **阻塞** 调用方（即 `LineWriter.Write()`）
  - `LineWriter.Write()` 被阻塞 → `pipeline_utils.CopyLineByLine()` 阻塞 → 容器 stdout 管道缓冲区填满 → **容器进程写 stdout 也会被阻塞**

> ⚠️ **这是一个有意的设计**：通过反向压力让日志产生速度自动匹配发送速度，避免 Agent 内存无限增长。但极端情况下可能影响 CI 任务执行（容器被阻塞）。

### 10.3 Agent 端日志背压的缓解策略

**策略1：批量聚合发送** (`agent/rpc/client_grpc.go:351-396`)
- 最多 1MB 一批 (`maxLogBatchSize`)，减少 RPC 次数
- 最多 1s 延迟 (`maxLogFlushPeriod`)，平衡实时性与吞吐量

**策略2：发送失败直接丢弃** (`agent/rpc/client_grpc.go:369-371`)
```go
if err := c.sendLogs(ctx, entries); err != nil {
    log.Error().Err(err).Msg("log drain: could not send logs to server")
}
// even if send failed, we don't have infinite memory; retry has already been used
entries = entries[:0]  // 直接清空，不重试
bytes = 0
```

> 💡 **权衡**：宁愿丢日志也不能让 Agent 内存爆掉或阻塞 CI 任务。`retryRPC` 在 `sendLogs` 内部已经尝试过重试，外层不再重试。

### 10.4 Server 端 RPC 处理背压

gRPC Server 本身没有显式的限流，但通过以下机制间接限流：

**1. JWT 认证过滤** (`server/rpc/authorizer.go:120-145`)
- 每个 RPC 先经过 `authorize()` 拦截器
- token 无效直接返回 `Unauthenticated`，不进入业务逻辑

**2. 业务层状态校验**（见第 6.2 节）
- 非法/过期状态更新直接拒绝，减少无效处理
- 如 workflow 已完成后收到的 Update/Log 全部拒绝

**3. goroutine 模型 = 隐式限流**
- gRPC 默认每个请求一个 goroutine
- 当请求过多时，Go runtime 调度压力增大，但不会崩溃
- 实际瓶颈在 DB 连接池和队列锁竞争

### 10.5 PubSub 日志订阅背压：慢消费者问题

这是最容易被忽视但影响最大的背压场景。

**日志 PubSub 结构** (`server/logging/log.go:46-109`):
```
stream (per step)
    ├── list: []*LogEntry        // 历史回放缓冲
    ├── subs: map[*subscriber]   // 订阅者集合
    └── done: chan struct{}

subscriber
    └── receiver: LogChan (chan []*LogEntry)
```

**发送逻辑** (`server/logging/log.go:99-105`):
```go
for sub := range s.subs {
    select {
    case sub.receiver <- entries:    // 非阻塞发送
    default:
        log.Info().Msgf("subscriber channel is full -- dropping logs for step %d", stepID)
        // 直接丢弃！
    }
}
```

**Web 端订阅者的 channel 容量** (`server/api/stream.go:42`, `213`):
```go
maxQueuedBatchesPerClient int = 30  // 每个浏览器客户端最多缓存 30 批
batches := make(logging.LogChan, maxQueuedBatchesPerClient)
```

**背压场景**：
- 浏览器打开多个 pipeline 页面，网络差/页面休眠导致 SSE 消费慢
- `logChan` 满 → `select` 走 `default` → **日志丢弃**
- 页面恢复后，只能收到后续日志，丢失的日志需从 LogStore 重新拉取

**已知问题**：代码注释明确指出 TODO (`server/logging/log.go:26-28`)：
> writing to subscribers is currently a blocking operation and does not protect against slow clients from locking the stream. This should be resolved.

实际上当前实现已改为非阻塞+丢弃，但仍不完美——没有滑动窗口或流控反馈机制。

### 10.6 Pipeline 状态 PubSub 背压

Pipeline 状态更新（非日志）走另一条 PubSub 通道，背压策略不同：

**Publisher 实现** (`server/pubsub/memory/pub.go:41-65`):
```go
for s, tl := range p.subs {
    go (*s)(message)  // 每个订阅者一个 goroutine，异步发送
}
```

**分析**：
- ✅ 不阻塞主流程：发送方立即返回，goroutine 在后台执行
- ⚠️ 订阅者多时有 goroutine 爆炸风险：1000 个订阅者 = 1000 个 goroutine per publish
- ⚠️ 没有队列：如果 `(*s)(message)` 阻塞（如写 HTTP 响应慢），goroutine 会累积

**Web 端接收缓冲** (`server/api/stream.go:88`):
```go
eventChan := make(chan []byte, 10)  // 事件通道容量 10
```

```go
func(m pubsub.Message) {
    select {
    case <-ctx.Done():
    case eventChan <- m.Data:  // 如果 eventChan 满，goroutine 阻塞在这里
    }
}
```

### 10.7 队列层背压：任务堆积

当 Agent 不足，pending 队列无限增长时：

**FIFO 队列的隐式限制** (`server/queue/fifo.go`):
- 没有显式的最大长度限制
- pending 使用 `container/list`，理论上可无限增长
- 实际限制来自内存和 `TaskList()` 的 DB 写入性能

**缓解手段**：
1. Agent 端 `Filter` 标签匹配：只有能力匹配的 Agent 才能拉取任务
2. `NoSchedule` 标志：可在 Agent 上设置，不再分配新任务 (`server/rpc/rpc.go:73-76`)
3. 依赖调度：`depsInQueue()` 确保依赖满足才执行，避免无效阻塞

### 10.8 反向限流总结表

| 层级 | 缓冲点 | 容量 | 满时行为 |
|-----|--------|------|---------|
| Agent 日志 | `client.logs` chan | 10 | 阻塞调用方（反向压到容器 stdout） |
| Agent 日志批 | 内存累计 | 1MB 或 1s | 触发发送 |
| Server → Web 日志 | `LogChan` per subscriber | 30 批 | 非阻塞，满则丢弃 |
| Server → Web 事件 | `eventChan` per client | 10 | 阻塞后台 goroutine |
| Server 任务队列 | `pending` list | 无限制 | 内存增长 + DB 写入压力 |
| 全局 RPC | gRPC goroutine 池 | 无限制 | Go runtime 调度，DB 连接池瓶颈 |

---

## 11. 日志流与状态流的多路复用问题

本章节分析两种数据流如何在 gRPC/HTTP/2 连接上多路复用，以及它们之间的相互影响。

### 11.1 数据流分类

从 Agent 流向 Server 的数据分为两类，走不同 RPC：

| 流类型 | 对应 RPC | 消息模式 | 实时性要求 | 数据量 |
|-------|---------|---------|-----------|--------|
| **状态流** (Control) | `Next/Init/Wait/Extend/Update/Done/RegisterAgent/ReportHealth` | 请求-响应（单条） | 高（阻塞调用） | 小（<1KB） |
| **日志流** (Data) | `Log` | 批量异步 | 中（可延迟 1s） | 大（可达 MB 级） |

### 11.2 gRPC/HTTP/2 多路复用底层机制

所有 RPC 共享 **同一个 TCP 连接** 和 **同一个 HTTP/2 会话**，通过 HTTP/2 Stream 多路复用：

```
TCP 连接 (端口复用)
  └── HTTP/2 会话
        ├── Stream 1: Next (长轮询，可能持续数分钟)
        ├── Stream 2: Wait (长轮询，可能持续数分钟)
        ├── Stream 3: Init (快速往返)
        ├── Stream 4: Extend (快速往返)
        ├── Stream 5: Update (快速往返)
        ├── Stream 6: Log (批量，可能 1MB 数据)
        ├── Stream 7: Update ...
        └── ...
```

**HTTP/2 流控保证**：
- 每个 Stream 有独立的流量控制窗口（初始 65535 字节）
- 连接级也有统一流量控制窗口
- 避免单个大流量 Stream 饿死其他 Stream

**关键代码**：两条连接（AuthConn 和 MainConn）都设置了相同的 keepalive 参数，但 MainConn 承担了所有多路复用压力。

### 11.3 状态流 vs 日志流的相互影响

#### 场景 1：日志大爆发阻塞状态 Update

```
Step 疯狂输出日志（如编译某个大项目）
    │
    ├── EnqueueLog 快速填满 logs channel (10个槽)
    ├── processLogs 攒满 1MB 触发发送
    ├── Log RPC 发送 1MB 数据 → 占用 HTTP/2 带宽和 Stream
    │
    └── 此时 Step 完成，需要发送 Update RPC
        ├── retryRPC 检查 IsConnected() → true（因为 HTTP/2 本身还通）
        ├── 但 Update RPC 的 HTTP/2 Stream 被 Log 的大数据块延迟
        ├── 最坏情况：Update 延迟几百毫秒到几秒
```

**实际影响**：
- 前端 UI 上 Step 状态更新不及时
- 但不会阻塞工作流执行（Update 失败只打日志不中断流程）

#### 场景 2：Wait/Next 长轮询与日志流的和平共处

Next 和 Wait 都是 **长生命周期 RPC**（可能持续几分钟到几小时）。它们与日志流的多路复用关系：

```
时间轴 →
  0s: Next() 发送请求 → HTTP/2 Stream 打开，等待 Server 推送任务
  5s: Log() 批量发送 → 独立 Stream，不影响 Next
 30s: Log() 再次发送 → 独立 Stream
 ...
120s: Server 有任务匹配 → Next 响应返回 → Stream 关闭
120s: 立即发送 Init() → 新 Stream
     同时启动 Wait() → 新的长生命周期 Stream
```

**优点**：HTTP/2 多路复用天然支持这种混合模式，长轮询 Stream 只占内存不占带宽。

**潜在风险**：
- 每个 Agent 的每个 capacity 占 2 个长生命周期 Stream（Next + Wait）
- 100 个 Agent × 4 capacity = 800 个常驻 Stream
- HTTP/2 默认最大并发 Stream 数是 100（Server 端可配），需要注意调优

### 11.4 长轮询 RPC 的多路复用设计取舍

为什么用两个独立的长轮询 RPC（Next + Wait），而不是一个双向 stream？

**当前设计（两个独立 Unary RPC）**：
- **Next**：Agent 拉任务
- **Wait**：Agent 等取消信号

**假设的流式设计（一个 Bidirectional Stream）**：
```
stream TaskOrSignal {
    oneof message {
        Task task = 1;
        CancelSignal cancel = 2;
        Ack ack = 3;
    }
}
```

**选择当前设计的理由**（从代码推断）：

1. **错误隔离**：
   - Next 失败重试不影响 Wait，反之亦然
   - 流式设计下 stream 断开需要重建整个状态机

2. **语义清晰**：
   - Next 是"拉"语义，可能返回 nil（context 取消）
   - Wait 是"等"语义，有明确的布尔返回值
   - 一个 stream 上 multiplex 多种消息需要自己写状态机

3. **重试策略不同**：
   - Next 可以无限等（context 不取消就一直重试）
   - Wait 与具体 workflow 绑定，有明确的生命周期
   - 统一 stream 无法差异化重试策略

4. **调试友好**：
   - 每个 RPC 独立日志，独立 trace
   - gRPC 工具（grpcurl, wireshark）更容易分析

### 11.5 服务端的多流分发

Server 端接收两类数据流，通过不同路径分发到不同订阅者：

```
gRPC 接收
    │
    ├── Log RPC → rpc.Log() → logging.Log.Write() ──┐
    │                                                ├── logging 多路复用
    │                                                │     │
    │                                                │     ├── step1.subs → 浏览器1 (SSE)
    │                                                │     ├── step1.subs → 浏览器2 (SSE)
    │                                                │     └── step2.subs → 浏览器3 (SSE)
    │                                                │
    └── Update/Done/Init RPC → rpc.*() → notify() ──┘
                                                     │
                                                     ├── PubSub.Publish()
                                                           │
                                                           ├── topic:"repo/123" → Web 事件流
                                                           └── topic:"public"    → 匿名用户
```

**日志流多路复用** (`server/logging/log.go:55-59`):
- `streams map[int64]*stream`：按 stepID 分桶
- 每个 stream 有自己的 `subs map[*subscriber]struct{}`
- 写入时遍历该 step 的所有 subscribers，非阻塞发送

**状态流多路复用** (`server/pubsub/memory/pub.go:41-65`):
- `subs map[*pubsub.Receiver][]string`：按订阅者分桶，每个订阅者有自己的 topic 列表
- 发布时遍历所有订阅者，匹配 topic 则异步 goroutine 发送

### 11.6 前端的双流接收

浏览器前端通过两条独立的 SSE 连接接收数据，与 Agent → Server 的双流对应：

| 前端流 | Endpoint | 对应 Agent → Server 流 |
|-------|----------|-----------------------|
| 事件流 | `/stream/events` | Update/Init/Done 产生的 PubSub 事件 |
| 日志流 | `/stream/logs/:repo_id/:pipeline/:step_id` | Log RPC 产生的 logging 消息 |

**前端流多路复用注意点** (`server/api/stream.go`):
- 两条 SSE 连接是 **独立的 HTTP/2 Stream**（如果浏览器支持 HTTP/2），否则是两个独立 TCP 连接
- 每条连接有独立的 `idlePingTime = 30s` 保活心跳
- 日志流支持 `Last-Event-ID` 重连恢复（`server/api/stream.go:245-250`），但事件流不支持

### 11.7 多路复用的已知问题与权衡

| 问题 | 表现 | 设计权衡 |
|-----|------|---------|
| **日志大消息延迟状态消息** | UI 上状态更新比日志晚几秒 | 接受延迟，状态流失败不影响工作流执行 |
| **大量长轮询 Stream 占用内存** | 每个 Agent × capacity = 2 Stream | HTTP/2 Stream 很轻量（几 KB 每个），1000 个 Stream 也只有几 MB |
| **Agent 重连时所有并发 RPC 同时重试** | 连接恢复瞬间流量突增 | `retryRPC` 的指数退避已错开重试时间 |
| **Server 端日志 PubSub 慢消费者丢日志** | 浏览器卡顿/休眠时丢日志 | 接受丢日志，可从 LogStore 重放 |
| **双 SSE 连接开销** | 移动端多费电 | HTTP/2 复用 TCP，实际开销小 |

### 11.8 多路复用关键代码路径表

| 方向 | 流类型 | 发送方 | 接收方 | 传输层 |
|-----|-------|--------|--------|--------|
| Agent → Server | 状态流 | `client_grpc.go` retryRPC | `server/rpc/rpc.go` 各方法 | gRPC Unary，MainConn 多路复用 |
| Agent → Server | 日志流 | `client_grpc.go` processLogs → sendLogs | `server/rpc/rpc.go:415-475` | gRPC Unary，MainConn 多路复用 |
| Server → Web | 状态流 | `pubsub/memory/pub.go:58` go (*s)(message) | `server/api/stream.go:56-128` EventStreamSSE | SSE over HTTP/1.1 or HTTP/2 |
| Server → Web | 日志流 | `logging/log.go:99-105` sub.receiver <- entries | `server/api/stream.go:140-281` LogStreamSSE | SSE over HTTP/1.1 or HTTP/2 |

