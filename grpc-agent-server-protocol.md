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
