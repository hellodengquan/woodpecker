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

---

## 12. gRPC TLS 认证与 mTLS 链路

本章节详解 gRPC 连接的 TLS 加密机制、证书配置、以及当前是否支持 mTLS（双向 TLS）。

### 12.1 TLS 连接分层架构

Woodpecker 的 TLS 配置分为 **两层独立的 TLS 端点**，各自服务不同的流量：

```
┌───────────────────────────────────────────────────────────────────┐
│                            Agent                                  │
│                                                                   │
│  ┌─ AuthConn (TLS) ─┐   ┌─ MainConn (TLS) ─┐                      │
│  │  port: 9000      │   │  port: 9000       │                      │
│  │  仅认证 RPC      │   │  所有业务 RPC      │                      │
│  └────────┬─────────┘   └────────┬──────────┘                      │
│           │                       │                                 │
│           └───────────┬───────────┘                                 │
│                       │                                             │
│          TLS over HTTP/2 (同一 TCP 连接，端口复用)                  │
└───────────────────────┼─────────────────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────────────┐
│                          Server (gRPC)                             │
│  port: 9000                                                         │
│  ┌──────────────────────────────────────────────────┐              │
│  │  gRPC Server (是否启用 TLS 取决于 Agent 配置)     │              │
│  │  - 无内置 TLS：依赖反向代理/负载均衡终止 TLS      │              │
│  └──────────────────────────────────────────────────┘              │
└───────────────────────────────────────────────────────────────────┘
```

### 12.2 Agent 端 TLS 配置

Agent 端的 TLS 行为完全由启动参数控制。

**核心配置参数** (`cmd/agent/core/flags.go:44-54`)：

| 参数 | 环境变量 | 默认值 | 作用 |
|-----|----------|--------|------|
| `--grpc-secure` | `WOODPECKER_GRPC_SECURE` | `false` | 是否使用 TLS 加密连接 |
| `--grpc-skip-insecure` | `WOODPECKER_GRPC_VERIFY` | `true` | 是否验证 Server 证书 |

> ⚠️ **命名注意**：Flag 名 `grpc-skip-insecure` 与环境变量 `WOODPECKER_GRPC_VERIFY` 语义相反——Flag 设为 `true` 表示"跳过不安全"（即验证证书），而环境变量设为 `true` 也表示"验证"。

**TLS 连接建立代码** (`agent/rpc/dial.go:69-76`)：
```go
var transport grpc.DialOption
if cfg.Secure {
    transport = grpc.WithTransportCredentials(grpc_credentials.NewTLS(
        &tls.Config{InsecureSkipVerify: cfg.SkipTLSVerify},
    ))
} else {
    transport = grpc.WithTransportCredentials(insecure.NewCredentials())
}
```

### 12.3 Server 端 TLS 配置

**重要发现**：**gRPC Server 本身不支持 TLS 终止**，必须依赖反向代理（如 Nginx、Traefik）做 TLS 终止。

**证据**：`cmd/server/grpc_server.go:30-45` 中 `runGrpcServer` 直接使用 `net.Listen("tcp", ...)` 创建普通 TCP listener，没有任何 TLS 包装：
```go
lis, err := net.Listen("tcp", c.String("grpc-addr"))
// ...
return server_rpc.Serve(ctx, server_rpc.ServeConfig{
    Listener:         lis,  // 普通 TCP listener，无 TLS
    // ...
})
```

对比 **HTTP Server** 的 TLS 配置（`cmd/server/server.go:180-211`）：
```go
// HTTP Server 有完整的 TLS 支持
tlsServer := &http.Server{
    Addr:    server.Config.Server.PortTLS,
    Handler: handler,
    TLSConfig: &tls.Config{
        NextProtos: []string{"h2", "http/1.1"},
    },
}
err := tlsServer.ListenAndServeTLS(
    c.String("server-cert"),  // 证书路径
    c.String("server-key"),   // 私钥路径
)
```

### 12.4 TLS 部署拓扑

由于 gRPC Server 本身不终止 TLS，实际生产部署必须采用以下架构：

```
Agent (TLS client)
      │
      │ TLS over HTTP/2
      ▼
反向代理 / 负载均衡 (TLS 终止)
  ├── 证书: server-cert + server-key
  ├── 协议: HTTP/2 (必需，gRPC 依赖 HTTP/2)
  └── 端口: 9000 (或其他)
      │
      │ 明文 HTTP/2 到后端
      ▼
Woodpecker gRPC Server (:9000)
  ├── 无 TLS
  └── 依赖反向代理提供加密
```

**关键要求**：反向代理必须支持 **HTTP/2 直通**（HTTP/2 Passthrough），因为 gRPC 协议底层依赖 HTTP/2 的多路复用、流控等特性，不能降级到 HTTP/1.1。

### 12.5 mTLS（双向 TLS）支持现状

**结论**：**当前代码不支持 mTLS**，即 Server 端不会请求也不会验证 Agent 的客户端证书。

**证据链**：
1. `server/rpc/serve.go` 中创建 gRPC Server 时，没有配置 `grpc.Creds()` 选项（仅有 `KeepaliveEnforcementPolicy`）
2. `agent/rpc/dial.go` 中创建 TLS Config 时，没有设置 `Certificates` 字段（仅可能设置 `InsecureSkipVerify`）
3. `cmd/server/grpc_server.go` 中没有任何 mTLS 相关的 flag（如 `--grpc-client-ca`、`--grpc-require-client-cert`）
4. 所有认证逻辑通过 **JWT token** 实现，而非客户端证书

**Agent 端 TLS Config 内容** (`agent/rpc/dial.go:71-72`)：
```go
&tls.Config{
    InsecureSkipVerify: cfg.SkipTLSVerify,  // 仅此一项
    // 缺失: Certificates []tls.Certificate
    // 缺失: RootCAs *x509.CertPool
    // 缺失: ServerName string
}
```

### 12.6 证书错误处理

虽然不支持 mTLS，但 Agent 端有完整的证书验证错误处理链。

**错误识别** (`shared/httputil/http_error.go:111-126`)：
```go
var certErr *x509.CertificateInvalidError
if errors.As(err, &certErr) {
    return fmt.Errorf("TLS certificate invalid: ...")
}
var unknownAuthErr *x509.UnknownAuthorityError
if errors.As(err, &unknownAuthErr) {
    return fmt.Errorf("TLS certificate verification failed: ...")
}
var hostErr *x509.HostnameError
if errors.As(err, &hostErr) {
    return fmt.Errorf("TLS hostname mismatch: ...")
}
```

### 12.7 TLS 安全配置建议

| 场景 | 推荐配置 | 说明 |
|------|---------|------|
| **生产环境** | `WOODPECKER_GRPC_SECURE=true`, `WOODPECKER_GRPC_VERIFY=true` | 强制 TLS + 证书验证 |
| **开发环境** | `WOODPECKER_GRPC_SECURE=false` | 明文连接，方便调试 |
| **自签名证书** | `WOODPECKER_GRPC_SECURE=true`, `WOODPECKER_GRPC_VERIFY=false` | 跳过证书验证（不推荐） |
| **mTLS 需求** | 需在反向代理层实现 | Woodpecker 本身不支持，由 Nginx/Traefik 处理客户端证书验证 |

---

## 13. Agent 注册与心跳协议字段详解

本章节详细拆解 `RegisterAgent` 和 `ReportHealth` 两个 RPC 的所有协议字段、数据结构、以及字段在业务逻辑中的具体作用。

### 13.1 协议定义回顾

**proto 定义** (`rpc/proto/woodpecker.proto:118-149`)：
```protobuf
message ReportHealthRequest {
  string status = 1;
}

message AgentInfo {
  string platform = 1;
  int32  capacity = 2;
  string backend  = 3;
  string version  = 4;
  map<string, string> customLabels = 5;
}

message RegisterAgentRequest {
  AgentInfo info = 1;
}

message RegisterAgentResponse {
  int64 agent_id = 1;
}
```

**内部 Go 类型** (`rpc/types.go:59-66`)：
```go
type AgentInfo struct {
    Version      string            `json:"version"`
    Platform     string            `json:"platform"`
    Backend      string            `json:"backend"`
    Capacity     int               `json:"capacity"`
    CustomLabels map[string]string `json:"custom_labels"`
}
```

### 13.2 Agent 数据模型（Server 端存储）

Server 端存储的 Agent 模型 (`server/model/agent.go:26-43`) 比 proto 定义更丰富：

| 字段 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `ID` | `int64` | DB autoincr | 主键，Agent 唯一标识 |
| `Created` | `int64` | xorm created | 创建时间戳 |
| `Updated` | `int64` | xorm updated | 更新时间戳 |
| `Name` | `string` | Hostname 或生成 | Agent 名称，默认 `WOODPECKER_HOSTNAME` |
| `OwnerID` | `int64` | Auth 时设置 | 所有者 ID，系统 agent 为 -1 |
| `Token` | `string` | Auth 时设置 | Agent token，系统 agent 用 master token |
| `LastContact` | `int64` | ReportHealth 更新 | 最后心跳时间戳 |
| `LastWork` | `int64` | Init/Done 更新 | 最后一次工作时间（自动扩缩容用） |
| `Platform` | `string` | RegisterAgent.info.platform | 操作系统平台（linux/amd64 等） |
| `Backend` | `string` | RegisterAgent.info.backend | 执行后端（docker/kubernetes/local 等） |
| `Capacity` | `int32` | RegisterAgent.info.capacity | 并行工作流数量 |
| `Version` | `string` | RegisterAgent.info.version | Agent 版本号 |
| `NoSchedule` | `bool` | 默认 false | 是否禁止调度新任务 |
| `CustomLabels` | `map[string]string` | RegisterAgent.info.customLabels | 用户自定义标签 |
| `OrgID` | `int64` | Auth 时设置 | 组织 ID，系统 agent 为 -1 |

### 13.3 RegisterAgent 字段详细说明

#### 13.3.1 platform — 平台标识

**来源**：`pipeline/backend` 自动检测
**典型值**：
- `linux/amd64`
- `linux/arm64`
- `darwin/arm64`
- `windows/amd64`

**作用**：
1. 作为 **隐式标签** 参与任务匹配（`server/rpc/rpc.go:79-88`）
2. 实际存储在 `CustomLabels["platform"]` 中

**代码证据** (`server/rpc/rpc.go:82-86`)：
```go
if info.Platform != "" {
    labels["platform"] = info.Platform
}
```

#### 13.3.2 capacity — 并行容量

**来源**：`WOODPECKER_MAX_WORKFLOWS` flag，默认 1
**作用**：
1. 决定 Agent 启动多少个 Runner 协程 (`cmd/agent/core/agent.go:174-181`)
2. 每个 Runner 发起一个 `Next()` 长轮询
3. 所以 capacity = 同时可执行的工作流数

**多 Runner 启动代码** (`cmd/agent/core/agent.go:174-181`)：
```go
for i := 0; i < capacity; i++ {
    serviceWaitingGroup.Go(func() error {
        return r.Run(runnerCtx)
    })
}
```

#### 13.3.3 backend — 执行后端

**来源**：`WOODPECKER_BACKEND` flag，默认 `auto-detect`
**典型值**：`docker`, `kubernetes`, `local`, `ssh`
**作用**：
1. 作为隐式标签 `CustomLabels["backend"]` 参与任务匹配
2. UI 展示 Agent 能力

#### 13.3.4 version — Agent 版本

**来源**：编译时注入的 version 变量
**作用**：
1. 与 Server 版本比较，确保兼容
2. 存储用于排障

#### 13.3.5 customLabels — 自定义标签

**来源**：`WOODPECKER_AGENT_LABELS` flag，支持多值
**格式**：`key=value` 或 `key=*`（通配符）或 `!key=value`（反向匹配）
**作用**：任务调度的核心匹配依据

**特殊标签前缀** (`server/rpc/filter.go:37-40`)：
```go
// ignore internal labels for filtering
for k := range labels {
    if strings.HasPrefix(k, pipeline.InternalLabelPrefix) {
        delete(labels, k)
    }
}
```
> `pipeline.InternalLabelPrefix = "woodpecker.org.cn/"` 开头的标签是系统内部标签，不参与匹配。

### 13.4 RegisterAgent 业务逻辑流程

```
Agent                                          Server
  │                                               │
  │  RegisterAgent(AgentInfo) ───────────────────▶│
  │   {platform, capacity, backend,               │
  │    version, customLabels}                     │
  │                                               │
  │                                               │  1. getAgentFromContext() 解析 agentID
  │                                               │  2. store.AgentFind(agentID) 加载
  │                                               │  3. 用 AgentInfo 更新字段:
  │                                               │     - Platform
  │                                               │     - Backend
  │                                               │     - Capacity
  │                                               │     - Version
  │                                               │     - CustomLabels (合并 platform/backend)
  │                                               │  4. 计算服务端强制标签:
  │                                               │     - OrgID 标签 (如 org=123 或 org=*)
  │                                               │  5. store.AgentUpdate() 保存
  │                                               │
  │  ◀────────── RegisterAgentResponse ────────────│
  │     {agent_id}                                │
```

**关键代码** (`server/rpc/rpc.go:477-501`)：
```go
agent, err := s.getAgentFromContext(ctx)
// ...
agent.Platform = info.Platform
agent.Backend = info.Backend
agent.Capacity = int32(info.Capacity)
agent.Version = info.Version
agent.CustomLabels = info.CustomLabels
if agent.CustomLabels == nil {
    agent.CustomLabels = map[string]string{}
}
// 注入隐式标签
if info.Platform != "" {
    agent.CustomLabels["platform"] = info.Platform
}
if info.Backend != "" {
    agent.CustomLabels["backend"] = info.Backend
}
// 注入服务端强制标签
serverLabels, _ := agent.GetServerLabels()
for k, v := range serverLabels {
    agent.CustomLabels[k] = v
}
s.store.AgentUpdate(agent)
```

### 13.5 ReportHealth 心跳字段详解

#### 13.5.1 status 字段

**proto 定义**：`string status = 1`
**实际发送值** (`cmd/agent/core/agent.go:263`)：
```go
if err := r.grpcClient.ReportHealth(ctx, &rpc.ReportHealthRequest{
    Status: "I am alive!",  // 硬编码常量
}); err != nil {
```

> 💡 **注意**：`status` 字段目前是固定值 `"I am alive!"`，没有实际语义。设计上预留为未来扩展（如 Agent 负载、资源使用率等），但当前未使用。

#### 13.5.2 心跳间隔

**默认值**：10 秒（`cmd/agent/core/agent.go:30`）
```go
reportHealthInterval = 10 * time.Second
```

**代码** (`cmd/agent/core/agent.go:245-264`)：
```go
serviceWaitingGroup.Go(func() error {
    ticker := time.NewTicker(reportHealthInterval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-ticker.C:
            if err := r.grpcClient.ReportHealth(ctx, &rpc.ReportHealthRequest{
                Status: "I am alive!",
            }); err != nil {
                log.Error().Err(err).Msg("could not send healthcheck")
            }
        }
    }
})
```

### 13.6 ReportHealth 业务逻辑

```
Agent (每 10s)                                Server
  │                                               │
  │  ReportHealth("I am alive!") ────────────────▶│
  │                                               │
  │                                               │  1. getAgentFromContext()
  │                                               │  2. store.AgentFind(agentID)
  │                                               │  3. agent.LastContact = time.Now().Unix()
  │                                               │  4. store.AgentUpdate()
  │                                               │
  │  ◀──────────── Empty ─────────────────────────│
```

**关键代码** (`server/rpc/rpc.go:520-534`)：
```go
agent, err := s.getAgentFromContext(ctx)
// ...
agent.LastContact = time.Now().Unix()
return s.store.AgentUpdate(agent)
```

### 13.7 心跳超时检测机制

Server 端并**没有主动检测**心跳超时的逻辑。心跳超时是**被动检测**的：

1. `LastContact` 字段仅作为信息展示（UI 上 Agent 状态）
2. 实际超时检测发生在 **队列调度循环** 中（`server/queue/fifo.go:338-348`）
3. `resubmitExpiredPipelines()` 通过 `taskState.deadline` 判断 agent 是否死亡

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            // 重新入队，任务会分配给其他可用 Agent
            q.pending.PushFront(taskState.item)
            delete(q.running, taskID)
            close(taskState.done)
        }
    }
}
```

### 13.8 心跳与续期的区别

| 机制 | ReportHealth 心跳 | Extend 续期 |
|------|------------------|------------|
| **粒度** | Agent 级别 | Workflow 级别 |
| **间隔** | 固定 10s | TaskTimeout/3（动态） |
| **更新字段** | `LastContact` | `deadline` |
| **作用** | UI 展示 + 信息收集 | 防止任务被重新调度 |
| **失败影响** | 仅打日志，不影响运行 | 超过 TaskTimeout 后任务被重新入队 |
| **代码位置** | `cmd/agent/core/agent.go:245-264` | `agent/runner.go:134-148` |

### 13.9 UnregisterAgent 协议

**proto 定义**：无请求字段，无响应字段
**调用时机**：Agent 优雅退出时（仅对系统 agent 有效）

**业务逻辑** (`server/rpc/rpc.go:504-518`)：
1. 校验 agent 是 **系统 agent**（`IsSystemAgent()` = true）
2. 非系统 agent 直接返回，不删除（因为它们可能通过 UI/API 创建）
3. `store.AgentDelete(agentID)` 从 DB 删除记录

---

## 14. 多 Agent 任务分配调度算法

本章节详解当有多个 Agent 在线时，Server 如何决定将任务分配给哪个 Agent。

### 14.1 调度整体架构

调度发生在队列的 `process()` 循环中，每 100ms 执行一次：

```
每 100ms process() 循环
    │
    ├── 1. resubmitExpiredPipelines()  ← 超时任务重新入队
    ├── 2. filterWaiting()              ← 依赖检查，ready 的入 pending
    └── 3. assignToWorker() 循环        ← 核心调度算法
             │
             └── 对每个 pending task:
                  └── 遍历所有在线 worker:
                       ├── filter(task) → (matched, score)
                       └── 选择 score 最高的 worker
```

### 14.2 任务入队时的标签注入

任务在进入队列前会被注入多个层级的标签，这些标签是调度匹配的基础：

```
Task Labels (最终用于匹配)
    │
    ├── 1. Pipeline YAML 中定义的 labels
    │       (如: size=large, gpu=true)
    │
    ├── 2. Repo 信息标签 (ApplyLabelsFromRepo)
    │       ├── woodpecker.org.cn/repo = org/repo
    │       └── woodpecker.org.cn/org  = 123
    │
    └── 3. 系统内部标签 (InternalLabelPrefix)
            ├── woodpecker.org.cn/...
            └── (调度前会被 filter 过滤掉)
```

**标签注入代码** (`server/model/task.go:48-58`)：
```go
func (t *Task) ApplyLabelsFromRepo(r *Repo) error {
    if t.Labels == nil {
        t.Labels = make(map[string]string)
    }
    t.Labels[pipeline.LabelFilterRepo] = r.FullName
    t.Labels[pipeline.LabelFilterOrg] = fmt.Sprintf("%d", r.OrgID)
    return nil
}
```

### 14.3 Worker 注册与 Filter 构建

每个 Agent 的每个 capacity 会注册一个 worker，每个 worker 携带一个 `Filter`：

```
Agent capacity = 3
    │
    ├── Runner 1 → Next(filter) → worker {filter, channel}
    ├── Runner 2 → Next(filter) → worker {filter, channel}
    └── Runner 3 → Next(filter) → worker {filter, channel}
```

**Filter 构建流程** (`server/rpc/rpc.go:71-88`)：
```go
labels := map[string]string{}
// 1. 从 Agent.CustomLabels 复制
for k, v := range agent.CustomLabels {
    labels[k] = v
}
// 2. 服务端强制标签
serverLabels, _ := agent.GetServerLabels()
for k, v := range serverLabels {
    labels[k] = v
}
// 3. NoSchedule 标志
if agent.NoSchedule {
    return nil, fmt.Errorf("agent has NoSchedule flag, will not receive tasks")
}
return &rpc.Filter{Labels: labels}, nil
```

**服务端强制标签** (`server/model/agent.go:63-74`)：
```go
func (a *Agent) GetServerLabels() (map[string]string, error) {
    filters := make(map[string]string)
    if a.OrgID != IDNotSet {
        filters[pipeline.LabelFilterOrg] = fmt.Sprintf("%d", a.OrgID)
    } else {
        filters[pipeline.LabelFilterOrg] = "*"
    }
    return filters, nil
}
```
> 系统 Agent 的 `OrgID = -1`，所以 `org = "*"`（通配符），可以匹配任何组织的任务。

### 14.4 核心调度算法：filter + 打分

调度的核心在 `createFilterFunc` (`server/rpc/filter.go:27-74`)，返回 `(matched bool, score int)`。

#### 14.4.1 匹配阶段（必须全部满足）

**Step 1：反向标签检查（Agent side required labels）** (`filter.go:76-86`)
```go
func requiredLabelsMissing(taskLabels, agentLabels map[string]string) bool {
    for label, value := range agentLabels {
        if len(label) > 0 && label[0] == '!' {
            // Agent 要求任务必须带有某标签
            // 如 Agent labels: {"!gpu": "true"}
            // 表示任务必须有 gpu=true 标签
            val, ok := taskLabels[label[1:]]
            if !ok || val != value {
                return true  // 缺失，不匹配
            }
        }
    }
    return false
}
```

**Step 2：内部标签过滤** (`filter.go:37-41`)
```go
for k := range labels {
    if strings.HasPrefix(k, pipeline.InternalLabelPrefix) {
        delete(labels, k)  // woodpecker.org.cn/ 开头的不参与匹配
    }
}
```

**Step 3：任务标签全匹配** (`filter.go:43-71`)
```go
for taskLabel, taskLabelValue := range labels {
    if taskLabelValue == "" {
        continue  // 空值忽略
    }

    // 任务的每个标签必须在 Agent labels 中找到匹配
    agentLabelValue, ok := agentFilter.Labels[taskLabel]
    if !ok {
        // 检查反向匹配: Agent labels 有 "!taskLabel" = value
        agentLabelValue, ok = agentFilter.Labels["!"+taskLabel]
        if !ok {
            return false, 0  // 不匹配
        }
    }

    switch agentLabelValue {
    case "*":
        score++  // 通配符匹配，得 1 分
    case taskLabelValue:
        score += 10  // 精确匹配，得 10 分
    default:
        return false, 0  // 不匹配
    }
}
return true, score
```

#### 14.4.2 打分规则总结

| 匹配类型 | 分数 | 说明 |
|---------|------|------|
| 精确匹配 | +10 | Agent label 值 == Task label 值 |
| 通配符匹配 | +1 | Agent label 值 == "*" |
| 不匹配 | - | 整个任务对该 Agent 不匹配 |

**优先级**：精确匹配 > 通配符匹配，分数越高越优先。

### 14.5 assignToWorker 选择逻辑

`assignToWorker()` (`server/queue/fifo.go:314-336`) 遍历所有 pending task 和所有 worker，找到**最佳匹配**：

```go
func (q *fifo) assignToWorker() (*list.Element, *worker) {
    var bestWorker *worker
    var bestScore int

    // 按 pending 顺序遍历（FIFO）
    for element := q.pending.Front(); element != nil; element = element.Next() {
        task, _ := element.Value.(*model.Task)

        // 遍历所有在线 worker
        for worker := range q.workers {
            matched, score := worker.filter(task)
            if matched && score > bestScore {
                bestWorker = worker
                bestScore = score
            }
        }
        if bestWorker != nil {
            return element, bestWorker  // 找到即返回，保证 FIFO
        }
    }
    return nil, nil
}
```

**关键特性**：
1. **FIFO 顺序**：按 pending 队列顺序逐个尝试，先入队的先分配
2. **最高分优先**：对同一个 task，选择 score 最高的 worker
3. **贪心匹配**：找到一个匹配就立即返回，不继续看后面的 task（保证公平）

### 14.6 调度匹配示例

#### 场景 1：单 Agent 多标签

```
Task labels: {platform: "linux/amd64", size: "large", org: "123"}

Agent1 labels: {platform: "linux/amd64", size: "*", org: "*"}
  → platform 精确匹配 (+10), size 通配 (+1), org 通配 (+1) = 12 分 ✓

Agent2 labels: {platform: "linux/arm64", size: "large", org: "*"}
  → platform 不匹配 (arm64 ≠ amd64) = 不匹配 ✗

选择：Agent1 (12 分)
```

#### 场景 2：多 Agent 精确 vs 通配

```
Task labels: {platform: "linux/amd64", backend: "docker", org: "123"}

Agent1 labels: {platform: "linux/amd64", backend: "*", org: "*"}
  → 精确(10) + 通配(1) + 通配(1) = 12 分

Agent2 labels: {platform: "linux/amd64", backend: "docker", org: "123"}
  → 精确(10) + 精确(10) + 精确(10) = 30 分 ✓

选择：Agent2 (30 分)
```

#### 场景 3：系统 Agent vs 组织 Agent

```
Task labels: {org: "123", platform: "linux/amd64"}

Agent1 (系统 Agent, OrgID=-1):
  → labels = {org: "*", platform: "linux/amd64"}
  → 通配(1) + 精确(10) = 11 分

Agent2 (组织 Agent, OrgID=123):
  → labels = {org: "123", platform: "linux/amd64"}
  → 精确(10) + 精确(10) = 20 分 ✓

选择：Agent2 (20 分)
```

> **重要**：组织 Agent 优先于系统 Agent 处理本组织任务，因为 `org=123` 精确匹配比 `org=*` 通配分数高。

#### 场景 4：反向标签（Agent 要求任务有某标签）

```
Agent labels: {"!gpu": "true", platform: "linux/amd64"}
  → "!gpu" 表示 Agent 要求任务必须有 gpu=true 标签

Task1 labels: {gpu: "true", platform: "linux/amd64"}
  → requiredLabelsMissing() 检查通过
  → 精确匹配 + 精确匹配 = 20 分 ✓

Task2 labels: {platform: "linux/amd64"}
  → requiredLabelsMissing() 返回 true（缺少 gpu=true）
  → 不匹配 ✗
```

### 14.7 依赖调度

任务有依赖时，即使匹配了 Agent 也不会立即执行，而是进入 `waitingOnDeps` 队列：

**依赖检查** (`server/queue/fifo.go:350-367`)：
```go
func (q *fifo) depsInQueue(task *model.Task) bool {
    // 检查 pending 队列中是否有依赖
    for element := q.pending.Front(); element != nil; element = element.Next() {
        possibleDep, _ := element.Value.(*model.Task)
        for _, dep := range task.Dependencies {
            if possibleDep.ID == dep {
                return true  // 依赖还在 pending
            }
        }
    }
    // 检查 running 队列中是否有依赖
    for possibleDepID := range q.running {
        if slices.Contains(task.Dependencies, possibleDepID) {
            return true  // 依赖还在 running
        }
    }
    return false
}
```

**依赖流转**：
1. 任务入队时先到 `pending`
2. `filterWaiting()` 发现有依赖未完成 → 移到 `waitingOnDeps`
3. 下一轮循环 `filterWaiting()` 再检查，依赖完成 → 移回 `pending`
4. `assignToWorker()` 正常分配

### 14.8 并发安全与锁

整个调度过程在 `q.Lock()` 保护下执行 (`server/queue/fifo.go:265, 285`)：
```go
select {
case <-time.After(processTimeInterval):
case <-q.ctx.Done():
    return
}

q.Lock()
// ... 所有调度逻辑 ...
q.Unlock()
```

这保证了：
- 不会出现同一个 task 被分配给多个 worker
- 不会出现 worker 被分配多个 task
- pending/waitingOnDeps/running/workers 四个结构的一致性

### 14.9 调度算法的优缺点

| 优点 | 缺点 |
|------|------|
| 实现简单，FIFO 保证公平 | 遍历所有 pending × all workers，O(n×m) 复杂度 |
| 精确匹配优先于通配，符合直觉 | 高并发场景下（如 1000 task × 1000 worker）每 100ms 一次遍历可能有性能问题 |
| 组织 Agent 优先处理本组织任务 | 不支持 worker 负载感知（不会优先分配给负载低的 Agent） |
| 反向标签支持 Agent 端强制需求 | 不支持任务优先级（所有 pending task 平等） |
| 依赖调度自动处理 | 不支持任务抢占（高优先级任务不能打断低优先级任务） |
| 全程锁保护，无竞态 | 全局锁在高并发下可能成为瓶颈 |

### 14.10 调度算法关键代码路径

| 阶段 | 代码位置 | 作用 |
|------|---------|------|
| 任务入队标签注入 | `server/model/task.go:48-58` | 注入 repo/org 标签 |
| Agent Filter 构建 | `server/rpc/rpc.go:71-88` | 合并 Agent 标签 + 服务端强制标签 |
| 匹配 + 打分 | `server/rpc/filter.go:27-74` | 核心过滤与评分逻辑 |
| 依赖检查 | `server/queue/fifo.go:350-367` | 判断依赖是否满足 |
| 最佳匹配选择 | `server/queue/fifo.go:314-336` | 遍历并选择最高分 worker |
| 调度循环 | `server/queue/fifo.go:257-287` | 每 100ms 执行一次完整调度 |

---

## 15. Agent 标签与 Capability 匹配机制详解

本章节深入解析 Agent 的标签系统、Capability（容量）如何与任务调度交互、以及标签匹配的完整生命周期。

### 15.1 标签系统的三层结构

Agent 的最终标签由 **三层叠加** 而成，每层都有不同的来源和优先级：

```
Agent 最终 Labels (用于调度匹配)
    │
    ├── 第1层: 用户自定义标签 (customLabels)
    │   来源: WOODPECKER_AGENT_LABELS
    │   示例: gpu=true, size=large, team=backend
    │
    ├── 第2层: 隐式系统标签 (注入到 customLabels)
    │   ├── platform   ← RegisterAgent.info.platform
    │   ├── backend    ← RegisterAgent.info.backend
    │   └── (如果为空则不注入)
    │
    └── 第3层: 服务端强制标签 (Server-side enforced)
        ├── woodpecker.org.cn/repo  ← 任务侧注入，Agent 侧不匹配
        └── woodpecker.org.cn/org   ← Agent.GetServerLabels()
            ├── 系统Agent: org = "*" (通配)
            └── 组织Agent: org = "123" (精确)
```

**标签注入代码** (`server/rpc/rpc.go:79-88`)：
```go
// 第2层: 注入隐式标签
if info.Platform != "" {
    agent.CustomLabels["platform"] = info.Platform
}
if info.Backend != "" {
    agent.CustomLabels["backend"] = info.Backend
}
// 第3层: 服务端强制标签
serverLabels, _ := agent.GetServerLabels()
for k, v := range serverLabels {
    agent.CustomLabels[k] = v
}
```

### 15.2 Capability 与 Worker 的映射关系

Capability 决定了 Agent 能同时执行多少个工作流，也决定了会注册多少个 Worker 到调度队列。

**映射关系**：
```
Agent capacity = 3
    │
    ├── Runner 1 (goroutine) → Next(filter) → worker_1 {filter, channel}
    ├── Runner 2 (goroutine) → Next(filter) → worker_2 {filter, channel}
    └── Runner 3 (goroutine) → Next(filter) → worker_3 {filter, channel}
```

**关键特性**：
1. **每个 Runner 独立调用 Next**：每个 Runner 都发起一个阻塞的 `Next()` RPC 调用
2. **每个 Next 对应一个 Worker**：Server 端每收到一个 Next 请求，就在 workers 集合中注册一个 worker
3. **所有 Worker 共享相同的 Filter**：同一 Agent 的所有 capacity 标签完全相同
4. **Worker 是临时的**：任务分配后 worker 从集合中删除，Runner 完成任务后再次调用 Next 重新注册

**多 Runner 启动代码** (`cmd/agent/core/agent.go:174-181`)：
```go
for i := 0; i < capacity; i++ {
    serviceWaitingGroup.Go(func() error {
        return r.Run(runnerCtx)
    })
}
```

### 15.3 Worker 注册的完整生命周期

```
Agent                                      Server (Queue)
  │                                            │
  │  1. Next(filter) ────────────────────────▶│  scheduler.Poll()
  │   (阻塞)                                   │
  │                                            │  workers.add(worker{filter, channel})
  │                                            │
  │  ◀──── 2. 任务分配 (Task) ────────────────│  匹配成功，worker.channel <- task
  │                                            │  workers.delete(worker)
  │                                            │
  │  3. [执行工作流... 可能几分钟到几小时]      │
  │                                            │
  │  4. Done(workflowID) ────────────────────▶│  scheduler.Done()
  │                                            │  running.delete(taskID)
  │                                            │
  │  5. Next(filter) ────────────────────────▶│  重新注册 worker
  │      (再次阻塞等待)                         │
```

**队列层实现** (`server/queue/fifo.go:87-111`)：
```go
func (q *fifo) Poll(ctx context.Context, f FilterFn) (*model.Task, error) {
    worker := &worker{
        channel:  make(chan *model.Task, 1),  // 缓冲 1，非阻塞发送
        filter:   f,
        agentID:  agentID,
        done:     make(chan struct{}),
    }

    q.Lock()
    q.workers[worker] = true  // 注册到 workers 集合
    q.Unlock()

    select {
    case task := <-worker.channel:
        return task, nil    // 收到任务，返回
    case <-ctx.Done():
        q.Lock()
        delete(q.workers, worker)  // context 取消，移除 worker
        q.Unlock()
        return nil, ctx.Err()
    }
}
```

### 15.4 标签匹配的详细流程

`createFilterFunc` 返回的 filter 函数是调度的核心，执行以下 **四步检查**：

```
任务 Task {Labels} + Agent Filter {Labels}
        │
        ▼
  Step 1: 反向标签检查 (requiredLabelsMissing)
    对 Agent 的每个 "!key=value" 标签:
      → 检查 Task 是否有 key=value
      → 缺少任何一个 → 不匹配 ✗
        │
        ▼ 通过
  Step 2: 内部标签过滤
    从 Task.Labels 中删除 woodpecker.org.cn/ 前缀的标签
        │
        ▼ 通过
  Step 3: 任务标签全匹配
    对 Task 剩余的每个标签 (key=value):
      → 在 Agent.Labels 中查找 key
      → 找不到 → 找 "!key" (反向匹配)
      → 都找不到 → 不匹配 ✗
      → 找到后比较 value:
          = "*" → 匹配，+1 分
          = taskValue → 匹配，+10 分
          = 其他 → 不匹配 ✗
        │
        ▼ 通过
  Step 4: 返回 (true, score)
```

### 15.5 匹配边界场景分析

#### 场景 1：空标签任务 vs 多标签 Agent

```
Task labels: {} (空)
Agent labels: {platform: "linux/amd64", gpu: "true"}

匹配结果: ✓ (true, 0 分)
```
> **原因**：Step 3 只遍历 **任务的标签**，任务没有标签就不需要匹配任何东西。空标签任务可以分配给任何 Agent。

#### 场景 2：任务有标签但 Agent 没有对应标签

```
Task labels: {gpu: "true"}
Agent labels: {platform: "linux/amd64"}

匹配结果: ✗ (false, 0)
```
> **原因**：任务要求 `gpu=true`，但 Agent 没有 `gpu` 标签，也没有 `!gpu` 反向标签，所以不匹配。

#### 场景 3：Agent 有额外标签

```
Task labels: {platform: "linux/amd64"}
Agent labels: {platform: "linux/amd64", gpu: "true", size: "large"}

匹配结果: ✓ (true, 10 分)
```
> **原因**：Agent 的额外标签（`gpu`, `size`）不影响匹配——只有任务的标签需要被满足。Agent 有多余的能力是完全没问题的。

#### 场景 4：通配符 vs 精确匹配的优先级

```
Task labels: {platform: "linux/amd64", size: "large"}

Agent1 labels: {platform: "linux/amd64", size: "*"}
  → 10 + 1 = 11 分

Agent2 labels: {platform: "*", size: "large"}
  → 1 + 10 = 11 分

Agent3 labels: {platform: "linux/amd64", size: "large"}
  → 10 + 10 = 20 分 ✓ 最高分
```

> **注意**：Agent1 和 Agent2 分数相同时，谁先被遍历到谁就获得任务（因为 `assignToWorker` 找到第一个最高分就返回）。

### 15.6 Capability 与并发的关系

**Capability 是逻辑并发数**，不等于物理资源限制：

| 层面 | 限制因素 | 说明 |
|------|---------|------|
| **调度层** | Capability | 队列最多给 Agent 分配 N 个任务 |
| **执行层** | Backend 资源 | 实际运行受 CPU/内存/Docker 限制 |
| **网络层** | gRPC 并发 Stream 数 | HTTP/2 默认 MaxStreams=100 |
| **日志层** | logs channel 容量 | 每个工作流独立的日志缓冲 |

**常见配置建议**：
- Docker 后端：capacity ≈ CPU 核心数
- Kubernetes 后端：capacity 可以很大（由 K8s 调度资源）
- Local 后端：capacity = 1（避免资源竞争）

### 15.7 NoSchedule 标志

Agent 可以被标记为 `NoSchedule=true`，此时不再分配新任务：

**触发时机**：
- 管理员通过 UI/API 设置（用于 Agent 下线维护）
- 系统自动设置（如检测到 Agent 异常）

**行为** (`server/rpc/rpc.go:73-76`)：
```go
if agent.NoSchedule {
    return nil, fmt.Errorf("agent has NoSchedule flag, will not receive tasks")
}
```

> 已经分配的任务会继续执行完成，只是不再分配新任务。

---

## 16. 任务执行超时与强制终止协议链路

本章节详解工作流超时的完整链路、强制终止的多种触发方式、以及各层之间的超时优先级。

### 16.1 超时体系的三层结构

Woodpecker 有 **三层独立的超时机制**，各自作用于不同层面：

```
┌───────────────────────────────────────────────────────────────┐
│  第3层: Workflow.Timeout (业务超时)                            │
│  单位: 分钟                                                    │
│  来源: Repo 设置 → 默认值 → 最大值限制                         │
│  作用: Agent 端 context.WithTimeout，超时后取消执行             │
│  触发: Done RPC 上报 finished=true, error="context deadline"  │
└───────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌───────────────────────────────────────────────────────────────┐
│  第2层: TaskTimeout (租约超时)                                 │
│  单位: 1 分钟 (constant.TaskTimeout)                           │
│  来源: 硬编码常量                                              │
│  作用: Server 端队列检测 Agent 是否存活                         │
│  触发: resubmitExpiredPipelines 重新入队                       │
└───────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌───────────────────────────────────────────────────────────────┐
│  第1层: Step 超时 (单步超时)                                   │
│  单位: 秒 (pipeline YAML 配置)                                 │
│  来源: .woodpecker.yaml 中 step.timeout                        │
│  作用: 单个 Step 的执行超时                                    │
│  触发: Step 退出码非 0 + error 消息                            │
└───────────────────────────────────────────────────────────────┘
```

### 16.2 Workflow.Timeout 的来源与计算

**超时值的三层决策链**（从高到低优先级）：

1. **Repo 级配置** (`repo.Timeout`)
   - 用户在仓库设置中配置
   - 单位：分钟
   - 0 表示使用默认值

2. **全局默认值** (`Config.Pipeline.DefaultTimeout`)
   - Server 启动参数 `--default-pipeline-timeout`
   - 单位：分钟

3. **全局最大值限制** (`Config.Pipeline.MaxTimeout`)
   - Server 启动参数 `--max-pipeline-timeout`
   - 超过此值的配置会被截断（普通用户）
   - Admin 用户可以超过

**计算逻辑** (`server/api/repo.go:113-116`)：
```go
if repo.Timeout == 0 {
    repo.Timeout = server.Config.Pipeline.DefaultTimeout
} else if repo.Timeout > server.Config.Pipeline.MaxTimeout {
    repo.Timeout = server.Config.Pipeline.MaxTimeout
}
```

### 16.3 Workflow 超时的完整协议链路

```
Agent                                          Server
  │                                               │
  │  1. Next() ─────────────────────────────────▶│
  │                                               │
  │  ◀──── Workflow {Timeout: 60} ──────────────│  从 DB 读取 repo.Timeout
  │       (分钟)                                  │
  │                                               │
  │  2. Agent 创建 workflowCtx:                   │
  │     context.WithTimeout(60 * time.Minute)    │
  │                                               │
  │  3. Init() ─────────────────────────────────▶│  标记 running
  │                                               │
  │  [执行工作流...]                              │
  │                                               │
  │  4. 60 分钟后:                                │
  │     workflowCtx.Done()                        │
  │     ↳ context.DeadlineExceeded                │
  │                                               │
  │  5. pipeline runtime 停止执行                 │
  │     ↳ 所有 Step 被 cancel                     │
  │                                               │
  │  6. Done(workflowID, state) ────────────────▶│
  │     {                                         │
  │       Finished: now,                          │
  │       Error: "context deadline exceeded",     │
  │       Canceled: false                         │
  │     }                                         │
  │                                               │
  │                                               │  7. DB: workflow → failure
  │                                               │  8. PubSub 通知
  │                                               │  9. 更新 forge 状态
```

**Agent 端超时创建代码** (`agent/runner.go:79-104`)：
```go
// Compute workflow timeout
timeout := time.Hour  // 默认 1 小时
if minutes := workflow.Timeout; minutes != 0 {
    timeout = time.Duration(minutes) * time.Minute
}

// Workflow execution context
workflowCtx, _ := context.WithTimeout(ctxMeta, timeout)
workflowCtx, cancelWorkflowCtx := context.WithCancelCause(workflowCtx)
```

### 16.4 强制终止的四种触发方式

工作流可以被 **四种不同机制** 终止，每种都有不同的协议路径：

| 触发方式 | 发起方 | 协议链路 | 状态表现 |
|---------|--------|---------|---------|
| **业务超时** | Agent 本地 | `context.WithTimeout` 到期 → `Done(error=deadline)` | `Error: "context deadline exceeded"`, `Canceled: false` |
| **UI/API 取消** | Server 端 | `queue.Error(ErrCancel)` → `Wait` 返回 `canceled=true` → Agent 取消执行 → `Done(canceled=true)` | `Canceled: true`, `Error: "canceled"` |
| **Agent 崩溃** | Server 检测 | 心跳/Extend 停止 → `deadline` 过期 → `resubmitExpiredPipelines` → 重新入队 | 任务重新分配给其他 Agent |
| **SIGTERM 信号** | Agent 进程 | `utils.WithContextSigtermCallback` → `cancelWorkflowCtx(ErrCancel)` → `Done(canceled=true)` | `Canceled: true`, `Error: "canceled"` |

### 16.5 UI/API 取消的完整协议链路

这是最复杂的终止路径，涉及队列、Wait RPC、Agent 执行三层协作：

```
用户/API → 点击取消
    │
    ▼
Server API 层
    │
    ├── pipeline.CancelWorkflow(workflowID)
    │     ├── DB: workflow → status = "canceled"
    │     └── scheduler.Error(workflowID, ErrCancel)
    │           │
    │           ▼
    │     queue.Error()
    │         ├── entry.error = ErrCancel
    │         └── close(entry.done)  ← 关键操作
    │
    ▼
Wait RPC 阻塞在 entry.done 上的所有 goroutine 被释放
    │
    ├── Agent1 Wait() 收到 done
    │     └── err = ErrCancel → canceled=true
    │
    └── Agent2 Wait() 收到 done
          └── (如果是同一个 workflow 的并行 step)
    │
    ▼
Agent 端 Wait 协程返回
    │
    ├── cancelWorkflowCtx(pipeline_errors.ErrCancel)
    │     ↓
    ├── workflowCtx.Done()
    │     ↓
    ├── pipeline runtime 停止所有 Step
    │     ├── 运行中的 Step 收到 Kill 信号
    │     └── 未开始的 Step 标记为 Skipped
    │
    └── Done(workflowID, state)
          ├── Canceled: true
          ├── Error: "canceled"
          └── Finished: now
    │
    ▼
Server 端 Done()
    ├── scheduler.Done()  ← 幂等，entry 已删除
    ├── DB: workflow → killed
    ├── 更新 forge 状态
    └── PubSub 通知
```

**队列层取消实现** (`server/queue/fifo.go:133-160`)：
```go
func (q *fifo) Error(taskID string, err error) error {
    q.Lock()
    defer q.Unlock()

    taskState, ok := q.running[taskID]
    if !ok {
        // 任务不在 running，可能在 pending
        // 从 pending 中移除
        for e := q.pending.Front(); e != nil; e = e.Next() {
            if t, _ := e.Value.(*model.Task); t.ID == taskID {
                q.pending.Remove(e)
                return nil
            }
        }
        return nil
    }

    taskState.error = err
    close(taskState.done)  // 释放所有 Wait 阻塞
    return nil
}
```

### 16.6 Agent 崩溃后的任务恢复

当 Agent 崩溃或网络断开时，任务不会丢失，会被重新调度：

```
Agent 崩溃 → Extend/心跳停止
    │
    ▼
Server 队列 process() 循环中:
  resubmitExpiredPipelines()
    │
    ├── 遍历 running map
    ├── if now.After(taskState.deadline)
    │     ├── taskState.error = ErrTaskExpired
    │     ├── pending.PushFront(task)  ← 重新入队（队首）
    │     ├── delete(running, taskID)
    │     └── close(taskState.done)     ← 释放 Wait
    │
    ▼
下一轮 assignToWorker()
  → 任务分配给另一个健康的 Agent
```

**注意**：
- 重新入队的任务放在 **队首**（`PushFront`），优先重新执行
- 任务会从头开始执行，不会从断点恢复
- 崩溃前的日志和状态仍然保留在 DB 中（如果已上报）

### 16.7 多级超时的优先级

当多种超时同时存在时，**先到期的先生效**：

| 场景 | 先到期 | 结果 |
|------|--------|------|
| Workflow.Timeout = 30min, TaskTimeout = 1min | TaskTimeout (1min) 如果 Agent 不续期 | 任务被重新入队，不会被标记为超时失败 |
| Workflow.Timeout = 30min, Step 超时 = 10min | Step 超时 (10min) | 该 Step 失败，Workflow 继续或失败取决于配置 |
| UI 取消 + Workflow 超时 | 谁先发生谁生效 | 取消的状态是 `killed`，超时是 `failure` |

### 16.8 超时相关的关键常量

| 常量 | 值 | 位置 | 作用 |
|------|----|------|------|
| `TaskTimeout` | 1 分钟 | `shared/constant/constant.go:41` | 队列租约超时 |
| `shutdownTimeout` | 5 秒 | `agent/runner.go:36` | Done 上报的兜底超时 |
| 默认 Workflow Timeout | 可配置 | server flag `--default-pipeline-timeout` | 工作流默认超时 |
| 最大 Workflow Timeout | 可配置 | server flag `--max-pipeline-timeout` | 普通用户上限 |

---

## 17. Agent 监控与统计指标体系

本章节详解 Agent 端本地监控接口、Server 端 Prometheus 指标收集、以及两者之间的数据流向。

### 17.1 监控体系总览

Woodpecker 的监控分为 **Agent 侧本地监控** 和 **Server 侧全局监控** 两层，Agent 指标不会主动上报到 Server。

```
┌─────────────────────┐          ┌─────────────────────┐
│   Agent (每个)      │          │   Server            │
│                     │          │                     │
│  /healthz ────────────  HTTP   │  /metrics           │
│  /varz    ────────────  本地    │  (Prometheus)       │
│  /version ────────────  抓取    │    ├── 队列指标      │
│                     │          │    ├── 存储指标      │
│  State (内存)       │          │    └── 流水线指标    │
└─────────────────────┘          └─────────────────────┘
          ▲                                  ▲
          │                                  │
          │ Prometheus / VictoriaMetrics      │ Prometheus
          │ (外部监控系统主动拉取)              │ (外部监控系统主动拉取)
```

### 17.2 Agent 端本地监控接口

Agent 内置了轻量级 HTTP 监控服务器，遵循 Dockerflow 规范。

**启用条件**：`WOODPECKER_HEALTHCHECK=true`（默认开启）

**监听地址**：`WOODPECKER_HEALTHCHECK_ADDR`（默认 `:3000`）

#### 17.2.1 /healthz — 健康检查

**返回状态码**：
- `200 OK`：Agent 健康
- `500 Internal Server Error`：Agent 不健康

**健康判断逻辑** (`agent/state.go:62-73`)：
```go
func (s *State) Healthy() bool {
    s.Lock()
    defer s.Unlock()
    now := time.Now()
    buf := time.Hour // 1小时缓冲
    for _, item := range s.Metadata {
        if now.After(item.Started.Add(item.Timeout).Add(buf)) {
            return false  // 有工作流超时 + 1小时还没结束
        }
    }
    return true
}
```

> **判断标准**：任何运行中的工作流，如果 `开始时间 + 配置超时 + 1小时缓冲` < 当前时间，则认为 Agent 不健康。这是一个很宽松的标准，主要用于检测 Agent 卡死。

#### 17.2.2 /varz — 运行状态详情

返回 JSON 格式的详细运行状态：

```json
{
  "polling_count": 2,      // 正在轮询（等待任务）的 Runner 数
  "running_count": 1,      // 正在执行的工作流数
  "running": {
    "wf-12345": {
      "id": "wf-12345",
      "repository": "org/repo",
      "pipeline_number": "42",
      "pipeline_started": "2024-01-15T10:30:00Z",
      "pipeline_timeout": 3600000000000  // 纳秒
    }
  }
}
```

**State 数据结构** (`agent/state.go:25-38`)：
```go
type State struct {
    sync.Mutex
    Polling    int             // 轮询中的 Runner 数
    Running    int             // 运行中的工作流数
    Metadata   map[string]Info // 每个运行中工作流的详情
}
```

#### 17.2.3 /version — 版本信息

```json
{
  "version": "v2.5.0",
  "source": "https://github.com/woodpecker-ci/woodpecker"
}
```

### 17.3 State 计数器的生命周期

State 计数器在工作流的不同阶段更新：

```
Runner 启动
    │
    ├── 初始: Polling = capacity, Running = 0
    │
    ├── Next() 阻塞等待 → Polling 不变
    │
    ├── 收到任务 → r.counter.Add()
    │     ├── Polling--
    │     ├── Running++
    │     └── Metadata[id] = Info{...}
    │
    ├── [执行工作流...]
    │
    └── 工作流结束 → r.counter.Done()
          ├── Polling++
          ├── Running--
          └── delete(Metadata, id)
```

**关键代码**：
- Add: `agent/state.go:40-52`
- Done: `agent/state.go:54-60`
- 初始化: `cmd/agent/core/health.go:79-81`

### 17.4 Server 端 Prometheus 指标

Server 提供标准 Prometheus 指标端点，地址为 `/metrics`（需配置 `--metrics-server-addr` 独立端口）。

**指标分类**：

| 类别 | 指标名 | 类型 | 标签 | 说明 |
|------|--------|------|------|------|
| **队列指标** | `woodpecker_pending_steps` | Gauge | - | 等待执行的任务数 |
| | `woodpecker_waiting_steps` | Gauge | - | 等待依赖的任务数 |
| | `woodpecker_running_steps` | Gauge | - | 正在运行的任务数 |
| | `woodpecker_worker_count` | Gauge | - | 在线 Worker 数 |
| **流水线指标** | `woodpecker_pipeline_time` | GaugeVec | repo, branch, status, pipeline | 流水线执行时间 |
| | `woodpecker_pipeline_count` | CounterVec | repo, branch, status, pipeline | 流水线完成计数 |
| **存储指标** | `woodpecker_pipeline_total_count` | Gauge | - | 流水线总数 |
| | `woodpecker_user_count` | Gauge | - | 用户总数 |
| | `woodpecker_repo_count` | Gauge | - | 仓库总数 |

### 17.5 队列指标收集器

队列指标通过独立的 goroutine 定期采集：

**采集周期**：`queueInfoRefreshInterval`（代码中未显式定义，默认值需查证）

**采集逻辑** (`cmd/server/metrics_server.go:67-84`)：
```go
go func() {
    for {
        stats := server.Config.Services.Scheduler.Info(ctx)
        pendingSteps.Set(float64(stats.Stats.Pending))
        waitingSteps.Set(float64(stats.Stats.WaitingOnDeps))
        runningSteps.Set(float64(stats.Stats.Running))
        workers.Set(float64(stats.Stats.Workers))

        select {
        case <-ctx.Done():
            return
        case <-time.After(queueInfoRefreshInterval):
        }
    }
}()
```

**队列 Info 结构** (`server/queue/fifo.go:202-226`)：
```go
type InfoT struct {
    Stats         struct {
        Workers      int  // 在线 Worker 数
        Pending      int  // 等待执行的任务数
        WaitingOnDeps int // 等待依赖的任务数
        Running      int  // 正在运行的任务数
    }
    Pending      []*model.Task  // 等待中的任务列表
    WaitingOnDeps []*model.Task
    Running      []*model.Task
    Paused       bool
}
```

### 17.6 流水线指标埋点

流水线完成时（Done RPC 中），会更新时序指标：

**指标定义** (`server/rpc/server.go:49-58`)：
```go
pipelineTime := factory.NewGaugeVec(prometheus.GaugeOpts{
    Namespace: "woodpecker",
    Name:      "pipeline_time",
    Help:      "Pipeline time.",
}, []string{"repo", "branch", "status", "pipeline"})

pipelineCount := factory.NewCounterVec(prometheus.CounterOpts{
    Namespace: "woodpecker",
    Name:      "pipeline_count",
    Help:      "Pipeline count.",
}, []string{"repo", "branch", "status", "pipeline"})
```

**更新时机**：`Done()` RPC 处理完成后，工作流状态落库时更新。

### 17.7 存储指标收集器

存储指标定期从 DB 中统计：

**采集周期**：`storeInfoRefreshInterval`

**采集指标** (`cmd/server/metrics_server.go:85-107`)：
- 仓库总数 (`repo_count`)
- 用户总数 (`user_count`)
- 流水线总数 (`pipeline_total_count`)

### 17.8 监控部署最佳实践

#### Agent 监控

- **每个 Agent 单独抓取**：Prometheus 配置 `static_configs` 指向每个 Agent 的 `:3000`
- **或使用服务发现**：在 K8s 环境中通过 service discovery 自动发现 Agent Pod
- **告警规则**：`healthz != 200` 持续 5 分钟 → 告警

#### Server 监控

- **独立 metrics 端口**：使用 `--metrics-server-addr` 单独暴露，不与业务端口混用
- **认证保护**：配置 `--prometheus-auth-token` 防止未授权访问
- **关键告警**：
  - `woodpecker_pending_steps` 持续增长 → Agent 不足
  - `woodpecker_worker_count` 突降 → 大量 Agent 掉线
  - `woodpecker_running_steps` 长时间无变化 → 队列卡死

### 17.9 监控数据流向总结

| 数据流 | 方向 | 协议 | 频率 |
|-------|------|------|------|
| Agent 状态 → Agent HTTP | 本地 | HTTP | 按需（请求时计算） |
| 队列状态 → Prometheus 指标 | Server 内部 | 内存读取 | 定期刷新 |
| DB 统计 → Prometheus 指标 | Server 内部 | SQL 查询 | 定期刷新 |
| 流水线完成 → 指标更新 | 事件驱动 | 内存写入 | 每次 Done RPC |
| Agent → Server 指标上报 | ❌ 不存在 | - | - |

> **重要**：Agent 端的运行状态 **不会主动上报** 到 Server。Server 只能通过 `LastContact` 间接推断 Agent 是否存活，无法知道 Agent 当前跑了多少个工作流、负载如何。如果需要集中监控所有 Agent 的状态，需要外部 Prometheus 分别抓取每个 Agent 的 `/varz` 端点。

---

## 18. Agent 升级与协议向后兼容

本章节详解 Agent/Server 版本升级时的协议兼容策略、版本协商机制、以及升级过程中的任务处理。

### 18.1 协议版本号机制

Woodpecker 使用 **严格的整数版本号** 来标识 gRPC 协议版本。

**版本定义** (`rpc/proto/version.go:19`)：
```go
const Version int32 = 16
```

**Proto 文件中的约定** (`rpc/proto/woodpecker.proto:21-23`)：
```protobuf
// !IMPORTANT!
// Increased Version in version.go by 1 if you change something here!
// !IMPORTANT!
```

每次修改 `.proto` 文件，**必须** 将 `version.go` 中的 `Version` 递增 1。当前版本为 **16**。

### 18.2 版本协商流程

Agent 在启动时执行一次版本协商，**不兼容则直接退出**，不会尝试降级运行。

```
Agent                                          Server
  │                                               │
  │  Version() ─────────────────────────────────▶│
  │                                               │
  │  ◀──── VersionResponse {                      │
  │           grpc_version: 16,                   │
  │           server_version: "v2.7.0"            │
  │         }                                     │
  │                                               │
  │  比较:                                         │
  │    ClientGrpcVersion == resp.GrpcVersion ?    │
  │                                               │
  │    相等 ✓ → 继续                               │
  │    不等 ✗ → 日志 + 退出                        │
```

**代码实现** (`cmd/agent/core/agent.go:146-158`)：
```go
grpcServerVersion, err := client.Version(grpcCtx)
if err != nil {
    log.Error().Err(err).Msg("could not get grpc server version")
    return err
}
if grpcServerVersion.GrpcVersion != agent_rpc.ClientGrpcVersion {
    err := errors.New("GRPC version mismatch")
    log.Error().Err(err).Msgf(
        "server version %s does report grpc version %d but we only understand %d",
        grpcServerVersion.ServerVersion,
        grpcServerVersion.GrpcVersion,
        agent_rpc.ClientGrpcVersion)
    return err
}
```

**Agent 端版本号** 在编译时绑定 (`agent/rpc/client_grpc.go:44`)：
```go
const ClientGrpcVersion int32 = proto.Version
```

### 18.3 版本兼容策略

Woodpecker 采用 **严格匹配** 策略，而非语义化版本兼容：

| 策略 | Woodpecker | 典型的 gRPC 项目 |
|------|-----------|-----------------|
| 兼容判断 | `grpc_version == ClientGrpcVersion` | 支持版本范围（如 `>= min_version`） |
| 不兼容行为 | Agent 直接退出 | 降级使用旧字段 |
| 新增字段 | protobuf 默认值（零值） | 同上 |
| 删除字段 | 编译时错误 | 运行时忽略 |

**这意味着**：
- Server 从 v2.6 升级到 v2.7（grpc_version 从 15 → 16），**所有旧 Agent 必须同时升级**
- 没有"Server 先升级、Agent 后升级"的滚动升级窗口
- proto 新增字段时，旧 Agent 虽然能反序列化（protobuf 向后兼容），但版本号不同仍会被拒绝

### 18.4 Protobuf 字段演化的实际影响

尽管版本号采用严格匹配，protobuf3 本身的字段编号机制提供了隐式兼容：

| 变更类型 | protobuf 兼容性 | Woodpecker 兼容性 | 实际效果 |
|---------|----------------|-------------------|---------|
| 新增字段（新编号） | ✓ 旧代码忽略未知字段 | ✗ 版本号不同被拒绝 | 需同时升级 |
| 删除字段 | ✓ 旧代码返回零值 | ✗ 版本号不同被拒绝 | 需同时升级 |
| 重命名字段 | ✗ 二进制不兼容 | ✗ 版本号不同被拒绝 | 需同时升级 |
| 修改字段类型 | ✗ 二进制不兼容 | ✗ 版本号不同被拒绝 | 需同时升级 |

### 18.5 升级策略与运行中任务处理

#### 场景 1：Server 先升级（不推荐）

```
1. Server 停止 → gRPC 连接断开
2. 所有 Agent 的 retryRPC 开始重试
3. Server 升级完成，grpc_version 从 15 → 16
4. Server 启动，Agent 重连成功
5. Agent 调用 Version() → grpc_version=16 ≠ ClientGrpcVersion=15
6. Agent 退出

问题: 所有 Agent 同时退出，运行中的工作流丢失
```

#### 场景 2：Agent 先升级（不推荐）

```
1. Agent 停止 → 运行中的工作流被 Server 检测为超时（1分钟内）
2. Agent 升级完成，ClientGrpcVersion=16
3. Agent 启动 → Version() → grpc_version=15 ≠ 16
4. Agent 退出

问题: Agent 连不上旧 Server，无法工作
```

#### 场景 3：同时升级（推荐）

```
1. 暂停队列调度: scheduler.Pause()  ← 防止新任务分配
2. 等待运行中的工作流完成（或超时）
3. 停止所有 Agent
4. 停止 Server
5. 升级 Server → grpc_version 从 15 → 16
6. 升级 Agent → ClientGrpcVersion=16
7. 启动 Server
8. 启动 Agent → Version() 匹配 ✓
9. 恢复队列调度: scheduler.Resume()
```

#### 场景 4：滚动升级（受限支持）

由于严格版本匹配，Woodpecker **不支持** Agent/Server 版本不同的滚动升级。唯一的滚动升级方式是：

1. 启动新版本 Agent（新端口/新部署），旧 Agent 仍运行
2. 旧 Agent 完成任务后自然退出（`single-workflow` 模式）或手动停止
3. 升级 Server
4. 此时旧 Agent 已全部退出，新 Agent 可以连接

### 18.6 Server 端版本返回

Server 端在 `Version()` RPC 中返回两个版本信息：

```go
func (s *WoodpeckerServer) Version(_ context.Context, _ *proto.Empty) (*proto.VersionResponse, error) {
    return &proto.VersionResponse{
        GrpcVersion:   proto.Version,       // 协议版本（当前=16）
        ServerVersion: version.String(),     // 应用版本（如 "v2.7.0"）
    }, nil
}
```

- `GrpcVersion`：用于兼容性检查，**必须严格相等**
- `ServerVersion`：仅用于信息展示和日志记录，不参与兼容判断

### 18.7 Unimplemented RPC 的处理

Server 端嵌入了 `proto.UnimplementedWoodpeckerServer`，如果 Agent 调用了 Server 未实现的新 RPC，会返回 `Unimplemented` 错误码：

```go
type WoodpeckerServer struct {
    proto.UnimplementedWoodpeckerServer
    peer RPC
}
```

这提供了**向前兼容**的安全网：如果 Agent 版本比 Server 新，调用了 Server 没有的 RPC，不会崩溃，而是收到明确的错误。但由于版本号严格匹配在前，这种情况实际上不会发生。

---

## 19. Server 重启时 In-Flight 任务恢复路径

本章节详解 Server 重启（计划内/崩溃恢复）时，处于不同状态的任务如何恢复。

### 19.1 Server 重启对内存状态的影响

Server 重启时，所有内存中的状态全部丢失：

| 数据结构 | 存储位置 | 重启后状态 | 恢复方式 |
|---------|---------|-----------|---------|
| `workers` map | 内存 | 丢失（空） | Agent 重新调用 Next() 注册 |
| `running` map | 内存 | 丢失（空） | 任务从 DB 恢复 |
| `pending` list | 内存 | 丢失（空） | 任务从 DB 恢复 |
| `waitingOnDeps` list | 内存 | 丢失（空） | 依赖检查后重新分类 |
| `entry.done` channel | 内存 | 丢失 | 重建 |
| `entry.deadline` | 内存 | 丢失 | 重建为 `now + TaskTimeout` |
| JWT secret | 配置 | 不变 | 重启后重新加载 |
| Agent token | 配置 | 不变 | 重启后重新加载 |
| Task 记录 | DB | 保留 | `TaskList()` 恢复 |
| Workflow/Step 状态 | DB | 保留 | 查询 DB 恢复 |

### 19.2 任务恢复的完整流程

Server 启动时，`persistentQueue` 从 DB 加载所有未完成的任务：

```
Server 启动
    │
    ├── 1. 初始化内存队列 (NewMemoryQueue)
    │     ├── workers = {}
    │     ├── running = {}
    │     ├── pending = list.New()
    │     └── waitingOnDeps = list.New()
    │
    ├── 2. 从 DB 恢复任务 (WithTaskStore)
    │     ├── store.TaskList() → 所有 Task 记录
    │     └── q.PushAtOnce(tasks) → 写入 pending
    │
    ├── 3. 启动调度循环 (process goroutine)
    │     ├── resubmitExpiredPipelines() ← running 为空，无操作
    │     ├── filterWaiting() ← 检查依赖关系
    │     └── assignToWorker() ← 等待 Agent 连接
    │
    ├── 4. Agent 重新连接
    │     ├── Auth() → 获取新 JWT
    │     ├── Version() → 版本检查
    │     ├── RegisterAgent() → 注册
    │     └── Next() → 注册 worker
    │
    └── 5. 任务重新分配
          └── assignToWorker() → 匹配 worker → 执行
```

**代码实现** (`server/queue/persistent.go:32-38`)：
```go
func WithTaskStore(ctx context.Context, q Queue, s store.Store) Queue {
    tasks, _ := s.TaskList()
    if err := q.PushAtOnce(ctx, tasks); err != nil {
        log.Error().Err(err).Msg("PushAtOnce failed")
    }
    return &persistentQueue{q, s}
}
```

### 19.3 In-Flight 任务的状态分类与恢复

Server 重启时，DB 中可能有以下状态的工作流，恢复策略各不同：

#### 类型 A：等待调度（pending）

```
DB 状态: workflow.State = pending
队列状态: Task 在 pending list 中
恢复: 正常，直接入 pending 队列
影响: 无，等 Agent 连接后正常调度
```

#### 类型 B：等待依赖（waitingOnDeps）

```
DB 状态: workflow.State = pending, Dependencies 未完成
队列状态: Task 在 pending list 中
恢复: filterWaiting() 会重新检查依赖
      → 依赖已完成 → 移入 pending
      → 依赖未完成 → 移入 waitingOnDeps
影响: 依赖关系在 DB 中保留，正确恢复
```

#### 类型 C：运行中（running）— 最复杂

```
DB 状态: workflow.State = running
队列状态: Task 在 DB 中，但 running map 为空

问题: Agent 还在执行，但 Server 不知道
```

**恢复路径**：

```
Server 重启
    │
    ├── 所有 running 任务从 DB 加载到 pending
    │   (因为 running map 为空，它们被视为"新任务")
    │
    ├── Agent 正在执行的工作流:
    │     ├── Extend RPC 失败 → 只打日志，不影响执行
    │     ├── Update RPC 失败 → retryRPC 重试
    │     ├── Wait RPC 失败 → retryRPC 重试
    │     └── Done RPC → 新 Server 接收
    │         ├── checkAgentPermissionByWorkflow() → 验证 agentID
    │         ├── checkWorkflowState() → 校验 workflow 状态
    │         └── 正常更新 DB
    │
    └── 任务被重新调度给另一个 Agent:
          ├── 同一任务可能被两个 Agent 同时执行
          └── 这是已知的竞态问题（下文分析）
```

#### 类型 D：已完成但 Done 未上报

```
DB 状态: workflow.State = running (Agent 完成但 Done 未到达)
队列状态: Task 在 pending 中

恢复:
    Agent Done RPC 到达 → checkWorkflowState()
      → workflow 已在 DB 中标记为 running
      → Done 成功更新为 finished
      → 同时 pending 中的副本被忽略（DB 幂等）
```

### 19.4 任务重复执行的竞态问题

**核心问题**：Server 重启后，running 中的任务被重新放入 pending，可能被另一个 Agent 拿到并开始执行。同时，原 Agent 还在执行同一个任务。

```
时间线:
  T0: Agent1 执行 Task-A，Server 崩溃
  T1: Server 重启，Task-A 从 DB 加载到 pending
  T2: Agent2 通过 Next() 拿到 Task-A，开始执行
  T3: Agent1 执行完毕，调用 Done() → 更新 DB
  T4: Agent2 也执行完毕，调用 Done() → checkWorkflowState() 报错

  或者更糟:
  T3: Agent2 先完成 → Done() 成功
  T4: Agent1 后完成 → Done() → checkWorkflowState() 报错
```

**现有缓解措施**：
1. `checkWorkflowState()` 防止已完成的工作流被再次更新
2. `checkAgentPermissionByWorkflow()` 验证 agentID（但新 Agent 也获得了合法权限）

**未解决**：没有全局锁或分布式去重机制。Server 重启后的短暂窗口期内，任务可能被重复执行。

### 19.5 持久化队列的 DB 写入时机

| 操作 | DB 写入 | DB 删除 |
|------|---------|---------|
| `PushAtOnce` | `store.TaskInsert(task)` | 失败时回滚删除 |
| `Poll` | - | `store.TaskDelete(task.ID)` |
| `Error` | - | `store.TaskDelete(id)` |
| `ErrorAtOnce` | - | `store.TaskDelete(id)` |

**代码** (`server/queue/persistent.go:65-76`)：
```go
func (q *persistentQueue) Poll(c context.Context, agentID int64, f FilterFn) (*model.Task, error) {
    task, err := q.Queue.Poll(c, agentID, f)
    if task != nil {
        // 任务被消费，从 DB 删除
        if deleteErr := q.store.TaskDelete(task.ID); deleteErr != nil {
            log.Error().Err(deleteErr).Msgf("pull queue item: %s: failed to remove from backup", task.ID)
        }
    }
    return task, err
}
```

> ⚠️ **注意**：`Poll` 成功后才从 DB 删除。如果 Server 在 Poll 和 TaskDelete 之间崩溃，任务会同时存在于内存队列和 DB 中。重启后 `TaskList()` 会再次加载它，导致 pending 中出现重复任务。

### 19.6 Server 优雅关闭

Server 收到停止信号后，执行优雅关闭：

**代码** (`server/rpc/serve.go:74-82`)：
```go
grpcCtx, cancel := context.WithCancelCause(ctx)
defer cancel(nil)

go func() {
    <-grpcCtx.Done()
    log.Info().Msg("terminating grpc service gracefully")
    grpcServer.GracefulStop()  // 等待所有进行中的 RPC 完成
    log.Info().Msg("grpc service stopped")
}()
```

**GracefulStop 的行为**：
1. 停止接受新 RPC 请求
2. 等待所有进行中的 RPC 完成
3. 关闭连接

**对 Agent 的影响**：
- 进行中的 Next/Wait 调用会收到 `Unimplemented` 或 `Canceled` 错误
- Agent 的 `retryRPC` 会尝试重连，直到 `connectionRetryTimeout` 耗尽
- 如果 Server 很快重启，Agent 可以无缝恢复

### 19.7 恢复流程总结表

| 任务状态 | DB 状态 | 恢复后队列位置 | 风险 |
|---------|--------|--------------|------|
| 等待调度 | pending | pending | 无 |
| 等待依赖 | pending | pending → filterWaiting 分类 | 无 |
| 运行中（Agent 仍在执行） | running | pending（重复入队） | 可能重复执行 |
| 运行中（Agent 已崩溃） | running | pending（正确入队） | 延迟恢复，最多 1 分钟 |
| 已完成但 Done 未上报 | running | pending | Done 到达时幂等更新 |
| 已取消 | canceled | 不在 DB | 无 |

---

## 20. 网络分区下的 Agent/Server 状态收敛

本章节分析网络分区（Network Partition）场景下 Agent 和 Server 各自的状态演化，以及分区恢复后的状态收敛行为。

### 20.1 网络分区的分类

根据分区方向和持续时间，分为四种场景：

```
┌─────────────────────────────────────────────────────────┐
│                    网络分区场景                           │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ 单向:       │  │ 双向:       │  │ 部分分区:   │    │
│  │ Agent→Server│  │ 完全断开    │  │ 部分Agent   │    │
│  │ 不通        │  │             │  │ 不通        │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ┌─────────────┐                                       │
│  │ 短暂分区:   │  ← 最常见，通常 < 30s                 │
│  │ 自动恢复    │                                       │
│  └─────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### 20.2 分区期间的 Agent 端状态演化

分区发生时，Agent 端各个 RPC 的行为：

| RPC | 分区期间行为 | Agent 端状态 |
|-----|------------|-------------|
| **Next** (阻塞等待) | `retryRPC` 重试 | Runner 阻塞，等待重连 |
| **Wait** (阻塞等待) | `retryRPC` 重试 | Wait goroutine 阻塞 |
| **Extend** (定期续期) | `retryRPC` 重试 | 续期失败，只打日志 |
| **Update** (状态上报) | `retryRPC` 重试 | 状态暂存在 Agent 内存 |
| **Log** (日志上报) | 缓冲 → 丢弃 | 日志丢失 |
| **ReportHealth** (心跳) | `retryRPC` 重试 | Server 端 LastContact 过期 |
| **Done** (工作流完成) | `retryRPC` 重试 | Agent 已完成但 Server 不知道 |

**Agent 端关键状态**：
```go
// agent/runner.go 中的核心 context
workflowCtx    → WithTimeout(workflow.Timeout) → 不受分区影响，继续倒计时
cancelWorkflowCtx → Wait() 返回 canceled=true 时触发 → 分区时不会触发
runnerCtx      → Agent 进程生命周期 → 不受分区影响
```

**重要**：Agent 的工作流执行 **不依赖** 与 Server 的实时通信。分区期间：
- 工作流继续执行（受本地 workflowCtx 超时控制）
- Step 继续运行
- 日志写入本地容器 stdout（如果 logs channel 满则阻塞容器）

### 20.3 分区期间的 Server 端状态演化

| 状态 | 行为 | 影响 |
|------|------|------|
| **队列** | `resubmitExpiredPipelines()` 检测 deadline 过期 | 1 分钟后任务重新入队 |
| **Agent 记录** | `LastContact` 不更新 | UI 显示 Agent 离线 |
| **Wait 阻塞** | `entry.done` 不会被关闭 | 等待中的 Wait 持续阻塞 |
| **PubSub** | 无新事件推送 | 前端页面无更新 |

**Server 端对分区的感知延迟**：

```
T+0s:  网络分区发生
T+10s: ReportHealth 失败（但 Server 不主动检测）
T+60s: Extend 失败 → deadline 过期 → resubmitExpiredPipelines()
       → 任务重新入队 → 分配给其他 Agent
T+??:  新 Agent 开始执行同一任务（如果有的话）
```

### 20.4 双向完全分区的完整时序

```
Agent                                    Server
  │                                         │
  │  [分区发生]                              │
  │                                         │
  │  Extend → 失败 (retryRPC 重试)           │
  │  Update → 失败 (retryRPC 重试)           │
  │  Log → 失败 (缓冲，最终丢弃)             │
  │  ReportHealth → 失败 (retryRPC 重试)     │
  │                                         │  T+60s: deadline 过期
  │                                         │  resubmitExpiredPipelines()
  │                                         │  → Task-A 重新入 pending
  │                                         │
  │  [工作流继续执行]                        │  → Task-A 分配给 Agent2
  │  Agent1 仍在执行 Task-A                  │  Agent2 开始执行 Task-A
  │                                         │
  │  [工作流完成]                            │
  │  Done() → retryRPC 重试                  │  Agent2 也在执行 Task-A
  │  (如果分区仍未恢复，继续重试)              │
  │                                         │
  │  [分区恢复]                              │
  │  Done() 成功                             │
  │                                         │  Agent2 Done() → 冲突
  │                                         │  checkWorkflowState() 报错
```

### 20.5 分区恢复后的状态收敛

分区恢复后，Agent 和 Server 需要从不一致状态收敛到一致。收敛行为取决于分区期间发生了什么：

#### 场景 1：分区短于 TaskTimeout（< 1 分钟）

```
Agent                                    Server
  │                                         │
  │  [分区恢复]                              │
  │                                         │
  │  retryRPC 重连成功                       │
  │  Extend → 成功，续期 deadline            │  deadline 更新，任务仍在 running
  │  Update → 成功，状态更新到 DB            │  Step 状态更新
  │  Log → 发送缓冲的日志                    │  日志追加
  │                                         │
  │  [收敛结果: 无损恢复]                     │
  │  所有状态一致，无副作用                   │
```

**这是最理想的场景**。`retryRPC` 的指数退避保证 Agent 在分区期间持续重试，恢复后立即成功。

#### 场景 2：分区长于 TaskTimeout，但 Agent 工作流未完成

```
Agent                                    Server
  │                                         │
  │  [分区恢复]                              │  Task-A 已被 Agent2 接管
  │                                         │
  │  retryRPC 重连成功                       │
  │  Extend → 返回错误（Agent2 在续期）       │  AgentID 不匹配
  │  Wait → 可能返回 canceled=true            │  entry.done 已关闭
  │                                         │
  │  cancelWorkflowCtx(ErrCancel)            │
  │  → 工作流被取消                          │
  │  Done(canceled=true)                     │
  │                                         │  checkWorkflowState()
  │                                         │  → workflow 可能已被 Agent2 更新
  │                                         │  → 幂等或报错
  │
  │  [收敛结果: Agent1 工作流被取消]
  │  Agent2 继续执行，最终 Done()
```

#### 场景 3：分区长于 TaskTimeout，Agent 工作流已完成

```
Agent                                    Server
  │                                         │
  │  [分区恢复]                              │  Task-A 已被 Agent2 完成
  │  Done() → retryRPC 重连后发送            │
  │                                         │  checkWorkflowState()
  │                                         │  → workflow.State = finished (Agent2 已 Done)
  │                                         │  → 返回错误 "workflow already finished"
  │                                         │
  │  [收敛结果: Agent1 Done 被忽略]
  │  Task-A 以 Agent2 的结果为准
  │  Agent1 的中间状态更新（Update/Log）可能丢失
```

### 20.6 分区期间的数据丢失

| 数据类型 | 是否丢失 | 原因 |
|---------|---------|------|
| **Step 状态 (Update)** | 可能丢失 | retryRPC 有重试，但 Permanent error 会放弃 |
| **日志 (Log)** | 可能丢失 | 缓冲满后丢弃，不无限重试 |
| **工作流结果 (Done)** | 不丢失 | retryRPC + shutdownCtx 兜底 |
| **Agent 心跳 (ReportHealth)** | 不影响 | Server 端仅更新 LastContact |
| **取消信号 (Wait)** | 可能丢失 | 如果 Server 端 entry.done 已关闭，Wait 会返回 canceled=true |

### 20.7 状态收敛的关键保证

| 保证 | 机制 | 限制 |
|------|------|------|
| **工作流最终状态不会丢失** | Done 的 retryRPC + shutdownCtx(5s) | 如果 Agent 进程崩溃则丢失 |
| **任务最终会被执行** | resubmitExpiredPipelines 重新入队 | 可能重复执行 |
| **Agent 不会永远阻塞** | retryRPC 的 connectionRetryTimeout | 超时后 Agent 退出 |
| **Server 不会永远占用任务** | deadline 过期后重新入队 | 延迟最多 TaskTimeout |
| **已完成的工作流不会被覆盖** | checkWorkflowState() 校验 | 重复的 Done 只会报错不会覆盖 |

### 20.8 不保证的场景

| 场景 | 不保证的行为 | 原因 |
|------|------------|------|
| **恰好一次执行** | 任务可能被执行两次 | 没有全局去重或分布式锁 |
| **日志完整性** | 分区期间的日志可能丢失 | Agent 端内存有限 |
| **状态更新顺序** | Agent1 和 Agent2 的 Update 可能交错 | 没有全局序列号 |
| **取消信号的实时性** | 分区期间无法传达取消信号 | Wait 依赖 gRPC 连接 |
| **Agent 负载均衡** | 所有 Agent 可能同时重试 | 无抖动（jitter）机制 |

### 20.9 减少分区影响的运维建议

| 建议 | 实施方式 | 效果 |
|------|---------|------|
| **缩短 TaskTimeout** | 修改 `shared/constant/constant.go` 中的值 | 加速死 Agent 检测，但增加 Extend 频率 |
| **增加 connectionRetryTimeout** | Agent flag `--retry-timeout=0`（无限重试） | Agent 不会因短暂分区退出 |
| **部署多个 Agent** | 横向扩展 Agent 数量 | 分区影响部分 Agent 时，其他 Agent 仍可工作 |
| **使用持久化队列** | `WithTaskStore` 默认启用 | Server 重启不丢失 pending 任务 |
| **配置 Workflow.Timeout** | 合理设置超时时间 | 避免工作流无限运行 |
| **监控 Agent 连接状态** | Prometheus 抓取 `/healthz` | 及时发现分区 |




