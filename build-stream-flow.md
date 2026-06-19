# Build Stream Flow：构建状态、实时输出与页面展示的联动分析

本文梳理 Woodpecker CI 前端 build 视图和日志流从产生到渲染的完整路径，
覆盖 **Agent → Server RPC → 内存多路复用 → SSE 推送 → Vue Store/组件** 全链路。

---

## 1. 全局架构总览

```
┌──────────┐   gRPC: Log/Update/Init/Done    ┌───────────────────────┐
│  Agent   │ ─────────────────────────────────▶│  Server RPC (rpc.go)  │
│ (runner) │                                  └───────┬───────────────┘
└──────────┘                                          │
                                         ┌────────────┼────────────┐
                                         ▼            ▼            ▼
                               ┌─────────────┐ ┌──────────┐ ┌──────────┐
                               │  Log Mux    │ │  PubSub  │ │  Store   │
                               │ (logging/)  │ │ (pubsub/)│ │ (DB/File)│
                               └──────┬──────┘ └────┬─────┘ └──────────┘
                                      │             │
                                      ▼             ▼
                               ┌────────────┐ ┌──────────────────┐
                               │LogStreamSSE│ │ EventStreamSSE   │
                               │ /stream/logs│ │ /stream/events   │
                               └──────┬─────┘ └──────┬───────────┘
                                      │               │
                                      ▼               ▼
                               ┌──────────────────────────────────┐
                               │      Frontend (Vue 3 + Pinia)   │
                               │  ApiClient._subscribe(EventSource)│
                               │  ┌────────────┐ ┌──────────────┐ │
                               │  │PipelineLog │ │PipelineStore │ │
                               │  │(log stream)│ │(state update)│ │
                               │  └──────┬─────┘ └──────┬───────┘ │
                               │         ▼               ▼        │
                               │  ┌─────────────────────────────┐ │
                               │  │  PipelineStepList + UI      │ │
                               │  └─────────────────────────────┘ │
                               └──────────────────────────────────┘
```

系统存在 **两条独立的 SSE 通道**，分别承载日志数据和状态事件，
前端通过各自的 `EventSource` 消费，实现构建状态与日志输出的解耦推送。

---

## 2. 构建状态的数据流

### 2.1 状态生命周期（Pipeline/Workflow/Step 三级）

状态值定义见 `web/src/lib/api/types/pipeline.ts:102-113`：

```typescript
type PipelineStatus =
  | 'blocked' | 'declined' | 'error' | 'failure'
  | 'killed'  | 'pending'  | 'running' | 'skipped'
  | 'started' | 'success'  | 'canceled';
```

状态在三级结构上逐级汇聚：

| 层级 | 模型 | 关键字段 |
|------|------|----------|
| Pipeline | `model.Pipeline` | `status`, `started`, `finished` |
| Workflow | `model.Workflow` / `PipelineWorkflow` | `state`, `started`, `finished` |
| Step | `model.Step` / `PipelineStep` | `state`, `started`, `finished`, `exit_code`, `error` |

**Pipeline 的最终状态由所有 Workflow 状态归并决定**（`server/pipeline/pipeline_status.go:27` `PipelineStatus()`），
Workflow 状态由其下 Step 状态归并，任一 failure 则上层为 failure。

### 2.2 Agent 上报状态 → Server RPC

Agent 通过 gRPC 调用 Server 的四个核心方法：

```
Init(stepUUID, WorkflowState)   → Workflow 开始运行
Update(stepUUID, StepState)     → Step 状态变更
Done(stepUUID, WorkflowState)   → Workflow 完成
Log(stepUUID, []LogEntry)       → 日志写入（见第 3 节）
```

**关键调用路径：**

1. **`RPC.Init()`** — `server/rpc/rpc.go:229`
   - 将 Pipeline 从 `pending` 转为 `running`
   - 将 Workflow 转为 `running`
   - 更新 Forge commit status
   - 调用 `notify()` 推送事件到 PubSub

2. **`RPC.Update()`** — `server/rpc/rpc.go:158`
   - 通过 `CalcStepStatus()` 计算新 Step 状态
   - Step 状态机：`pending → running → success/failure/killed`
   - 若 `state.Exited`，通知 `LogStore.StepFinished()`
   - 重建 Workflow 树并调用 `notify()` 推送

3. **`RPC.Done()`** — `server/rpc/rpc.go:296`
   - 完成 Workflow 下仍在运行的子 Step
   - 将 Workflow 状态转为终态
   - 若所有 Workflow 完成，将 Pipeline 转为终态
   - 关闭该 Workflow 下所有 Step 的日志流 (`logger.Close()`)
   - 调用 `notify()` 推送最终状态

### 2.3 Server → 前端：EventStreamSSE

`server/api/stream.go:56` 的 `EventStreamSSE` 端点 (`GET /api/stream/events`)：

```
┌─────────┐  Subscribe   ┌───────────┐  Publish   ┌──────────┐
│ SSE conn │◀────────────│ Scheduler │◀───────────│ RPC      │
│ (per user)│  (topics)   │ (PubSub)  │  (topics)  │ .notify()│
└─────────┘              └───────────┘            └──────────┘
```

- 用户连接时订阅 `public` topic + 所有自有 repo 的 `repo.id.{id}` topic
- `RPC.notify()` 将 `model.Event{Repo, Pipeline}` 序列化为 JSON，通过 PubSub 发布
- SSE 格式：`data: <json>\n\n`，带 30s idle ping 保活
- 前端 `ApiClient._subscribe()` 创建 `EventSource`，自动 reconnect

### 2.4 前端 Store 状态更新

`web/src/compositions/useEvents.ts` 是状态事件入口：

```
ApiClient.on(data) → PipelineStore.setPipeline(repoId, pipeline)
                   → RepoStore.setRepo(repo)
```

`PipelineStore` (`web/src/store/pipelines.ts`) 是核心状态仓库：

- **数据结构**：`Map<repoId, Map<pipelineNumber, Pipeline>>` — 二级 Map
- **`setPipeline()`**：merge 更新（浅合并），保留未变更字段
- **`setWorkflow()`**：替换指定 Workflow 并回写 Pipeline
- **`getPipeline()`**：返回 computed ref，驱动视图响应式更新

### 2.5 Pipeline 页面状态联动

`PipelineWrapper.vue` (`web/src/views/repo/pipeline/PipelineWrapper.vue`)：

```
onMounted → pipelineStore.loadPipeline()  // REST API 首次加载
watch(pipeline) → favicon.updateStatus()  // Favicon 状态图标联动
```

关键 inject/provide 链：
```
PipelineWrapper  provide('pipeline')     ← pipelineStore.getPipeline()
                 provide('pipeline-configs')
    ↓
Pipeline.vue     inject('pipeline')      ← 选择 stepId
    ↓
PipelineStepList  inject('pipeline')     ← 渲染 Workflow/Step 树
PipelineLog       inject('pipeline')     ← 渲染日志流
```

---

## 3. 实时日志流的处理路径

### 3.1 Agent 写入日志

Agent 执行 Step 命令时，通过 gRPC 调用 `RPC.Log()` (`server/rpc/rpc.go:415`)：

```
Agent → gRPC Log(stepUUID, []*rpc.LogEntry)
              │
              ▼
        RPC.Log()
              │
              ├── 鉴权 & 校验（agent 权限、pipeline 状态允许追加日志）
              │
              ├── goroutine: logger.Write(ctx, step.ID, logEntries)
              │     → 写入内存 Log Mux（实时推送给 SSE 客户端）
              │
              └── LogStore.LogAppend(step, logEntries)
                    → 持久化到文件/数据库（历史日志读取）
```

**两条写入路径并行执行**：
- **内存 Log Mux**：实时推送，面向 SSE 连接
- **LogStore 持久化**：面向历史查询 REST API

### 3.2 内存日志多路复用器（Log Mux）

`server/logging/log.go` 实现了基于 stepID 的日志多路复用：

```go
type logger struct {
    streams map[int64]*stream  // stepID → stream
}

type stream struct {
    stepID int64
    list   []*model.LogEntry   // 历史缓冲
    subs   map[*subscriber]struct{}  // 订阅者集合
    done   chan struct{}             // 关闭信号
}
```

核心操作：

| 方法 | 作用 | 代码位置 |
|------|------|----------|
| `Open(stepID)` | 创建 stream（若不存在） | `log.go:68` |
| `Write(stepID, entries)` | 追加日志 → 广播给所有 subscriber | `log.go:86` |
| `Tail(stepID, receiver)` | 订阅：先回放 `list`，再持续接收 | `log.go:111` |
| `Close(stepID)` | 关闭 stream → 通知所有 subscriber 退出 | `log.go:140` |

**Tail 的回放机制**（`log.go:123-124`）：
```go
if len(s.list) != 0 {
    sub.receiver <- s.list  // 一次性发送已缓冲的全部日志
}
```
后连接的 SSE 客户端能获取到连接前的日志。

**背压处理**（`log.go:100-104`）：
```go
select {
case sub.receiver <- entries:
default:
    log.Info().Msgf("subscriber channel is full -- dropping logs")
}
```
慢客户端导致 channel 满时直接丢弃日志，不阻塞写入。

### 3.3 LogStreamSSE → 前端

`server/api/stream.go:140` 的 `LogStreamSSE` 端点
(`GET /api/stream/logs/{repo_id}/{pipeline}/{step_id}`)：

```
┌────────────┐  Open + Tail   ┌──────────┐  Write   ┌─────────┐
│ SSE conn    │──────────────▶│ Log Mux  │◀─────────│ RPC.Log │
│ (per step)  │               │ (stream) │          │         │
└──────┬─────┘               └──────────┘          └─────────┘
       │
       │ SSE: id: N\ndata: <json>\n\n
       │ SSE: event: eof\ndata: eof\n\n  (step 完成)
       ▼
┌──────────────┐
│ EventSource  │  ApiClient._subscribe(path, cb, {reconnect: true})
│ (PipelineLog)│
└──────────────┘
```

**SSE 数据格式**：
- 每条日志：`id: <seq>\ndata: <LogEntry JSON>\n\n`
- 步骤完成：`event: eof\ndata: eof\n\n`
- 保活：30s 间隔发送 `: ping\n\n`
- 支持 `Last-Event-ID` 断线重连（`stream.go:246-249`）
- 每客户端最多缓存 30 批日志（`maxQueuedBatchesPerClient`）

**前提条件**：Step 状态必须是 `pending` 或 `running`，否则返回 `event: error`。

### 3.4 前端日志加载策略

`PipelineLog.vue` 的 `loadLogs()` (`web/src/components/repo/pipeline/PipelineLog.vue:471`)：

```typescript
if (step.state !== 'running' && step.state !== 'pending') {
    // 已完成 → REST API 一次性拉取
    logs = await apiClient.getLogs(repoId, pipelineNumber, stepId);
    logs.forEach(line => writeLog({index: line.line, text: line.data, time: line.time}));
    flushLogs(false);
} else {
    // 运行中 → SSE 实时流
    stream = apiClient.streamLogs(repoId, pipelineNumber, stepId, (line) => {
        writeLog({index: line.line, text: line.data, time: line.time});
        flushLogs(true);
    });
}
```

| Step 状态 | 加载方式 | API | 自动滚动 |
|-----------|----------|-----|----------|
| `running` / `pending` | SSE 实时流 | `/api/stream/logs/...` | ✅ |
| 其他（已完成） | REST 一次性拉取 | `/api/repos/{id}/logs/...` | ❌ |

### 3.5 日志渲染管线

```
SSE/REST 日志行
    │
    ▼
writeLog()                — Base64 解码 + AnsiUp ANSI→HTML 转换
    │
    ▼
logBuffer (缓冲区)       — debounce 500ms 批量刷新
    │
    ▼
flushLogs()               — 截取 maxLineCount 行 + 去重时间戳
    │
    ▼
log (ref<LogLine[]>)      — 响应式数据
    │
    ▼
groupedLogs (computed)    — 按 pipeline config 命令正则分组
    │
    ▼
<template> v-for渲染      — 行号、内容(HTML)、耗时三列布局
```

**关键细节**：

1. **ANSI 转换**：使用 `AnsiUp` 库，`use_classes = true` 启用 CSS 类模式
2. **URL 自动链接**：`processText()` 中用正则将 URL 转为 `<a>` 标签
3. **行数限制**：`maxLineCount`（来自配置）防止内存溢出
4. **日志分组**：匹配 pipeline config 中的 `- command` 模式，将输出按命令分组折叠
5. **去重时间戳**：相邻行相同时间戳只显示一次
6. **debounce 刷新**：500ms 防抖，减少 DOM 更新频率

---

## 4. 两条通道的联动时序

下面展示一次完整构建过程中，状态通道和日志通道的交互时序：

```
 Agent                        Server                          Frontend
  │                             │                                │
  │─── Init(workflowID) ───────▶│                                │
  │                             │── publish Event ──────────────▶│ PipelineStore: pending→running
  │                             │                                │ PipelineWrapper: Favicon 变化
  │                             │                                │ PipelineStepList: Step 图标更新
  │                             │                                │
  │─── Update(stepState) ─────▶│                                │
  │    (started, running)       │── publish Event ──────────────▶│ PipelineStore: step→running
  │                             │                                │ PipelineLog: watch(step.state变化)
  │                             │                                │   → loadLogs() → 开启 SSE 日志流
  │                             │                                │
  │─── Log(stepUUID, entries) ▶│                                │
  │                             │── logger.Write() ─────────────│
  │                             │   → Log Mux 广播              │
  │                             │── LogStore.LogAppend()         │
  │                             │   → 文件持久化                 │
  │                             │                                │ SSE → writeLog() → flushLogs()
  │                             │                                │   → 实时日志渲染 + 自动滚动
  │                             │                                │
  │─── Log(stepUUID, entries) ▶│  ... (重复多轮) ...            │
  │                             │                                │
  │─── Update(stepState) ─────▶│                                │
  │    (exited, exit_code=0)    │── publish Event ──────────────▶│ PipelineStore: step→success
  │                             │── LogStore.StepFinished()      │
  │                             │                                │ PipelineLog: watch(step.state变化)
  │                             │                                │   → 重新 loadLogs()
  │                             │                                │   → step 非running → 切换为 REST 拉取
  │                             │                                │
  │─── Done(workflowID) ───────▶│                                │
  │                             │── logger.Close() ─────────────│ SSE 收到 event: eof
  │                             │── publish Event ──────────────▶│ PipelineStore: workflow→success
  │                             │                                │ 若所有 workflow 完成:
  │                             │                                │   PipelineStore: pipeline→success
  │                             │                                │   Favicon → 绿色
```

**关键联动点**：

1. **Step 状态变更触发日志加载策略切换** — `PipelineLog.vue:555` 的 `watch(step)` 监听 `state` 变化，
   从 running 变为终态时，关闭 SSE 流，改用 REST API 重新拉取完整日志

2. **Favicon 与 Pipeline 状态联动** — `PipelineWrapper.vue:181` 监听 `pipeline` 变化，
   通过 `useFavicon()` 更新浏览器标签图标

3. **Workflow 完成关闭日志流** — `RPC.Done()` 中遍历所有子 Step 调用 `logger.Close()`，
   通知 SSE 连接发送 `event: eof`，前端关闭 EventSource

---

## 5. 关键文件索引

### 后端

| 文件 | 作用 |
|------|------|
| `server/rpc/rpc.go` | Agent gRPC 入口：Init/Update/Done/Log |
| `server/logging/logging.go` | Log Mux 接口定义（Open/Write/Tail/Close） |
| `server/logging/log.go` | Log Mux 内存实现：stream + subscriber 多路复用 |
| `server/api/stream.go` | SSE 端点：EventStreamSSE + LogStreamSSE |
| `server/pubsub/pubsub.go` | PubSub 接口定义 |
| `server/pubsub/memory/pub.go` | 内存 PubSub 实现 |
| `server/pipeline/pipeline_status.go` | Pipeline 状态转换逻辑 |
| `server/pipeline/step_status.go` | Step 状态转换 + CalcStepStatus |
| `server/pipeline/topic.go` | publishToTopic：Pipeline 事件发布到 PubSub |
| `server/services/log/service.go` | LogStore 接口：LogFind/LogAppend/LogDelete |
| `server/services/log/file/file.go` | 文件系统 LogStore 实现 |
| `server/model/log.go` | LogEntry 数据模型 |
| `server/model/event.go` | Event 数据模型（Repo + Pipeline） |
| `server/scheduler/scheduler.go` | Scheduler = Queue + PubSub 组合 |

### 前端

| 文件 | 作用 |
|------|------|
| `web/src/lib/api/client.ts` | ApiClient：REST + `_subscribe()` SSE 封装 |
| `web/src/lib/api/index.ts` | WoodpeckerClient：`on()` 状态事件 + `streamLogs()` 日志流 |
| `web/src/lib/api/types/pipeline.ts` | Pipeline/Workflow/Step/Log 类型定义 |
| `web/src/store/pipelines.ts` | Pinia PipelineStore：状态存储 + 响应式更新 |
| `web/src/compositions/useEvents.ts` | SSE 状态事件入口：回调 → PipelineStore |
| `web/src/compositions/usePipeline.ts` | Pipeline 计算：duration/since/message |
| `web/src/compositions/useFavicon.ts` | Favicon 状态图标联动 |
| `web/src/compositions/usePipelineFeed.ts` | 侧边栏 Pipeline Feed |
| `web/src/views/repo/pipeline/PipelineWrapper.vue` | Pipeline 页面容器：加载 + inject/provide |
| `web/src/views/repo/pipeline/Pipeline.vue` | Pipeline 主视图：StepList + Log 布局 |
| `web/src/components/repo/pipeline/PipelineStepList.vue` | Step 列表组件：Workflow/Step 树 |
| `web/src/components/repo/pipeline/PipelineLog.vue` | 日志组件：SSE/REST 加载 + 渲染 + 分组 |

---

## 6. 易混淆点辨析

### 6.1 为什么有两条 SSE 通道？

| 通道 | 端点 | 作用 | 数据粒度 |
|------|------|------|----------|
| 事件流 | `/api/stream/events` | Pipeline 状态变更通知 | Pipeline 级别（含完整 Pipeline+Repo） |
| 日志流 | `/api/stream/logs/{repo}/{pipeline}/{step}` | 单 Step 实时日志输出 | LogEntry 级别 |

分离的原因：
- 事件流是**全局性**的，一个连接覆盖用户所有 repo
- 日志流是**单 Step** 的，只在用户查看该 Step 时建立
- 两者生命周期不同：事件流长连接，日志流随 Step 选择而开关

### 6.2 Log Mux vs LogStore 的区别

| | Log Mux (`logging.Log`) | LogStore (`log.Service`) |
|---|---|---|
| 存储 | 内存 | 文件系统/数据库 |
| 用途 | 实时推送给 SSE 客户端 | 持久化，供 REST API 历史查询 |
| 生命周期 | Step 运行期间 | 永久（直到删除） |
| 写入时机 | goroutine 异步（不阻塞 Agent） | 同步（确保持久化成功） |

### 6.3 日志获取时机：SSE vs REST 的切换

`PipelineLog.vue:487-498` 的判断逻辑：

```typescript
if (step.state !== 'running' && step.state !== 'pending') {
    // REST: 已完成的 Step 直接从 LogStore 读取
} else {
    // SSE: 运行中的 Step 通过 Log Mux 实时订阅
}
```

注意：Step 从 `running` 变为终态时，`watch(step)` 触发 `loadLogs()`，
关闭 SSE 连接，改用 REST 重新获取完整日志。这确保了：
- 日志数据完整性（SSE 可能因网络丢包丢失）
- 避免在 Step 已完成后维持无意义的 SSE 连接

### 6.4 SSE 断线重连

- **事件流**：`ApiClient._subscribe()` 默认 `reconnect: true`，
  浏览器 EventSource 自动重连
- **日志流**：同样 `reconnect: true`，且服务端支持 `Last-Event-ID`，
  重连后从上次断点继续推送（`stream.go:246-249`）
- 日志流的 `event: eof` 表示 Step 日志流正常结束，
  前端收到后关闭 EventSource
