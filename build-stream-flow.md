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
| `web/src/components/repo/pipeline/PipelineStepDuration.vue` | Step/Workflow 计时器 |
| `web/src/compositions/useElapsedTime.ts` | 实时耗时计算：setInterval + 自动启停 |
| `web/src/compositions/useUserConfig.ts` | 用户配置持久化（含 collapseLogGroupsByDefault） |
| `web/src/compositions/useConfig.ts` | 全局配置（含 maxPipelineLogLineCount） |
| `web/src/style/console.css` | ANSI 颜色 CSS（light/dark 主题） |
| `web/src/lib/utils/index.ts` | 工具函数：debounce、escapeHtml |
| `web/src/main.ts` | 应用入口：初始化 useEvents 全局 SSE |

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

---

## 7. SSE 流式日志重连与断线恢复

### 7.1 前端 EventSource 生命周期管理

`PipelineLog.vue` 通过 `stream` ref 持有当前日志 EventSource 实例：

```
stream: ref<EventSource | undefined>
```

生命周期关键节点：

| 时机 | 操作 | 代码位置 |
|------|------|----------|
| Step 切换 / 初始加载 | `stream.value?.close()` → 重建 | `PipelineLog.vue:481,494` |
| Step 状态变为终态 | `watch(step)` → `loadLogs()` → `stream.value?.close()` + REST 拉取 | `PipelineLog.vue:555-564` |
| 组件销毁 | `onBeforeUnmount` → `stream.value?.close()` | `PipelineLog.vue:547-549` |

**关键**：前端不直接处理 EventSource 的 `onerror` 回调。
当 `opts.reconnect` 为 `true` 时（默认值），`ApiClient._subscribe()` 不注册 `onerror` handler，
完全依赖浏览器内置的 EventSource 自动重连机制。

```typescript
// client.ts:94-117
_subscribe<T>(path, callback, opts = { reconnect: true }) {
  const events = new EventSource(_path);
  events.onmessage = (event) => {
    callback(JSON.parse(event.data));
  };

  if (!opts.reconnect) {
    // 仅当 reconnect=false 时才注册 onerror
    events.onerror = (err) => {
      if (err.data === 'eof') events.close();
    };
  }
  return events;
}
```

`streamLogs()` 和 `on()` 都使用默认 `reconnect: true`，因此浏览器的 EventSource
在连接断开后会自动指数退避重连，前端代码无需介入。

### 7.2 服务端 Last-Event-ID 断点续传

`LogStreamSSE` (`server/api/stream.go:244-277`) 实现了基于 SSE 标准 `id` 字段的重连续传：

```
┌────────────────────────────────────────────────────────┐
│ 首次连接                                               │
│                                                        │
│ Server: id: 1\ndata: {...}\n\n                         │
│ Server: id: 2\ndata: {...}\n\n                         │
│ Server: id: 3\ndata: {...}\n\n                         │
│   ↕ 网络断开                                           │
│                                                        │
│ 浏览器自动重连，请求头带 Last-Event-ID: 3               │
│                                                        │
│ Server: 读取 Last-Event-ID → last = 3                  │
│ Server: id: 4\ndata: {...}\n\n  (id > last 才发送)     │
│ Server: id: 5\ndata: {...}\n\n                         │
└────────────────────────────────────────────────────────┘
```

服务端核心逻辑（`stream.go:244-277`）：

```go
id := 1
last, _ := strconv.Atoi(c.Request.Header.Get("Last-Event-ID"))

for {
    select {
    case buf := <-logChan:
        if id > last {                    // 只发送 id > last 的消息
            write("id: " + strconv.Itoa(id))
            write("data: " + buf)
            flush()
        }
        id++                              // id 始终递增，不论是否跳过
    }
}
```

**重要限制**：

1. **id 是 SSE 连接内的递增序号，不是日志行号** — 每条 `LogEntry` 的 JSON 中有自己的 `line` 字段，
   而 SSE `id` 仅用于断点续传。重连后跳过的是 SSE 层面的消息序号，不是日志行号。

2. **重连窗口有限** — 断线期间日志仍通过 `logger.Write()` 写入 Log Mux 的 `stream.list` 缓冲。
   但如果 Step 在断线期间完成，`logger.Close()` 会删除 stream，重连时 Log Mux 的 `Tail()` 返回 `ErrNotFound`，
   SSE 连接发送 `event: error` 后关闭。此时前端 EventSource 重连也只会再次失败。

3. **Step 已完成的重连回退** — 如果重连时 Step 已经不再是 `pending/running`，
   `LogStreamSSE` 在校验 Step 状态时直接返回 `event: error\ndata: step not running (anymore)\n\n`，
   不进入 Tail 循环。

### 7.3 重连失败的兜底：Step 状态变更触发 REST 回退

当前端通过 `EventStreamSSE` 收到 Pipeline 状态更新（Step 从 running → success/failure），
`PipelineLog.vue:555-564` 的 `watch(step)` 被触发：

```typescript
watch(step, async (newStep, oldStep) => {
  if (oldStep?.name === newStep?.name) {
    if (oldStep?.state !== newStep?.state) {
      await loadLogs();  // 重新加载 → step 不再 running → 走 REST 路径
    }
  }
});
```

这条路径形成了 SSE 重连失败后的**最终兜底**：

```
SSE 断线
  → 浏览器自动重连
  → 重连失败（Step 已完成）
  → EventSource 收到 error event
  → 同时状态通道推送 Step 终态
  → watch(step) 触发 loadLogs()
  → loadLogs() 检测 step.state !== 'running'
  → 关闭 EventSource，切换 REST API 获取完整日志
```

### 7.4 事件流重连的特殊性

`EventStreamSSE` 不发送 `id` 字段，因此 **EventStream 不支持基于 Last-Event-ID 的断点续传**。
断线重连后，前端只能收到重连时刻之后的新事件。

但这在实践中影响有限：
- `useEvents` 回调将事件 merge 到 `PipelineStore.setPipeline()`
- `PipelineStore.setPipeline()` 是浅合并（`{...old, ...new}`）
- 丢失的是中间瞬态，最终态一定会被下一次事件推送覆盖

---

## 8. 多 Step / 多 Workflow 并发执行的渲染调度

### 8.1 数据模型：Pipeline → Workflow[] → Step[]

Woodpecker CI 的 Pipeline 由多个 Workflow 组成，每个 Workflow 下有多个 Step：

```typescript
interface Pipeline {
  workflows?: PipelineWorkflow[];   // 多 Workflow（对应 YAML 中的多 pipeline）
}

interface PipelineWorkflow {
  id: number;
  pid: number;
  name: string;
  state: PipelineStatus;
  children: PipelineStep[];          // 多 Step
}

interface PipelineStep {
  pid: number;
  state: PipelineStatus;
  type?: StepType;                   // Clone | Service | Plugin | Commands | Cache
}
```

Workflow 之间可以并行执行（由 `depends_on` 控制依赖），
同一 Workflow 下的 Step 也可以并行执行（如 `services` 和 `commands` 类型的 Step）。

### 8.2 SSE 事件通道的状态合并机制

当多个 Workflow/Step 并发执行时，Agent 会交错调用 `RPC.Update()`，
每次调用都触发 `notify()` 推送完整 Pipeline 状态快照：

```
Agent 并发执行 Step A 和 Step B:

  RPC.Update(stepA: running)  → notify({Pipeline 包含 stepA=running, stepB=pending})
  RPC.Update(stepB: running)  → notify({Pipeline 包含 stepA=running, stepB=running})
  RPC.Update(stepA: success)  → notify({Pipeline 包含 stepA=success, stepB=running})
  RPC.Update(stepB: success)  → notify({Pipeline 包含 stepA=success, stepB=success})
```

前端 `PipelineStore.setPipeline()` (`store/pipelines.ts:39-54`) 使用**浅合并**策略：

```typescript
function setPipeline(repoId: number, pipeline: Pipeline) {
  const repoPipelines = pipelines.get(repoId) ?? new Map();
  repoPipelines.set(pipeline.number, {
    ...repoPipelines.get(pipeline.number),  // 保留旧字段
    ...pipeline,                            // 覆盖新字段
  });
  pipelines.set(repoId, repoPipelines);
}
```

**这意味着**：每次 SSE 事件推送的是**完整 Pipeline 对象**（含所有 Workflow 和 Step），
而不是增量 diff。前端无需手动合并多个 Step 的状态更新，
因为服务端在 `RPC.Update()` 中每次都重建完整 Workflow 树：

```go
// server/rpc/rpc.go:220-225
if currentPipeline.Workflows, err = s.store.WorkflowGetTree(currentPipeline); err != nil {
    log.Error().Err(err).Msg("cannot build tree from step list")
    return err
}
return s.notify(c, repo, currentPipeline)
```

### 8.3 PipelineStepList 的渲染调度

`PipelineStepList.vue` 是左侧 Step 列表面板，渲染逻辑：

**1. Workflow 折叠策略** (`PipelineStepList.vue:153-165`)

```typescript
const workflowsCollapsed = ref<Record<PipelineStep['id'], boolean>>(
  pipeline.value.workflows.length > 1
    ? pipeline.value.workflows.reduce((collapsed, workflow) => ({
        ...collapsed,
        [workflow.id]:
          ['success', 'skipped', 'blocked'].includes(workflow.state) &&
          !workflow.children.some((child) => child.pid === selectedStepId.value),
      }), {})
    : {},
);
```

- 多 Workflow 时，已完成的 Workflow 默认折叠
- 包含当前选中 Step 的 Workflow 保持展开
- 单 Workflow 时全部展开

**2. 单配置检测** (`PipelineStepList.vue:167-169`)

```typescript
const singleConfig = computed(
  () => pipelineConfigs?.length === 1 && pipeline.workflows.length === 1,
);
```

单配置时隐藏 Workflow 名称行和折叠按钮，直接展示 Step 列表。

**3. Step 选中联动** (`Pipeline.vue:95-132`)

```typescript
const selectedStepId = computed({
  get() {
    // 1. URL 参数指定的 stepId 优先
    // 2. 桌面端默认选中第一个 Step
    // 3. 移动端默认不选中
  },
  set(_selectedStepId) {
    // 通过 router.replace 更新 URL 参数
  },
});
```

选中 Step 通过 URL 参数 (`stepId`) 持久化，支持直接链接分享。

**4. Step 滚动定位** (`PipelineStepList.vue:172-183`)

```typescript
watch(selectedStepId, async (newId, oldId) => {
  if (!oldId && newId) {
    await nextTick();
    const step = steps.value?.find(s => s.dataset.stepId === newId.toString());
    step?.scrollIntoView({ behavior: 'auto', block: 'start' });
  }
});
```

首次选中 Step 时自动滚动到可视区域。

### 8.4 并发 Step 的日志流隔离

每个 Step 的日志流是独立的 SSE 连接。用户同一时刻只能查看一个 Step 的日志
（`PipelineLog` 只渲染 `selectedStepId` 对应的 Step），因此：

- **不存在多 Step 日志同时流式推送** — 切换 Step 时关闭旧 SSE，建立新 SSE
- **Step 切换的幂等保护** — `loadLogs()` 通过 `loadedStepSlug` 防止重复加载

```typescript
const stepSlug = computed(() =>
  `${repo.owner} - ${repo.name} - ${pipeline.id} - ${stepId}`
);

async function loadLogs() {
  if (loadedStepSlug.value === stepSlug.value) return;  // 已加载，跳过
  // ...
}
```

- **非选中 Step 的日志静默累积在 LogStore** — 即使前端未查看，
  Agent 写入的日志仍通过 `LogStore.LogAppend()` 持久化到文件系统，
  用户后续切换到该 Step 时通过 REST API 一次性获取

### 8.5 并发 Step 的计时器调度

`PipelineStepDuration.vue` 为每个 Step/Workflow 显示实时耗时，
内部使用 `useElapsedTime` composition：

```typescript
// useElapsedTime.ts
const running = computed(() => step?.state === 'running');
const { time: durationElapsed } = useElapsedTime(running, durationRaw);
```

- 每个 `PipelineStepDuration` 实例独立持有 `setInterval`
- 仅当 Step 状态为 `running` 时启动定时器
- Step 完成时定时器自动停止
- 组件卸载时 (`onBeforeUnmount`) 清除定时器

**并发 Step 场景下**：多个 Step 同时 running → 多个 `setInterval` 同时运行，
每秒更新各自的 `durationRaw` → 触发 Vue 响应式更新 → 各自的 `<span>` 独立渲染。
由于每次更新只是简单的文本替换，性能影响可控。

---

## 9. 日志检索、关键字高亮与时间戳过滤

### 9.1 行级锚点：URL Hash 驱动的日志行定位

PipelineLog 通过 URL hash (`#L{lineNumber}`) 实现日志行精确定位：

**行锚点渲染** (`PipelineLog.vue:106-117`)

```html
<a :id="`L${line.number}`" :href="`#L${line.number}`">{{ line.number }}</a>
```

- 每行的行号是 `<a>` 标签，`id` 为 `L{number}`，`href` 为 `#L{number}`
- 点击行号 → URL hash 变更 → `isSelected()` 计算命中 → 高亮

**行选中判定** (`PipelineLog.vue:338-340`)

```typescript
function isSelected(line: LogLine): boolean {
  return route.hash === `#L${line.number}`;
}
```

选中行通过 CSS class `bg-blue-600/30` + `underline` 高亮显示。

**Hash 变更监听** (`PipelineLog.vue:594-600`)

```typescript
watch(() => route.hash, (newHash) => {
  expandLogGroupWithPageHash(newHash);
}, { immediate: true });
```

URL hash 变化时，自动展开包含目标行的日志分组。

**日志加载后滚动** (`PipelineLog.vue:431-435`)

```typescript
if (route.hash.length > 0) {
  nextTick(() => document.getElementById(route.hash.substring(1))?.scrollIntoView());
}
```

新日志加载后，如果 URL 有 hash，自动滚动到目标行。

### 9.2 日志分组折叠与命令高亮

PipelineLog 将日志按命令分组，每组可折叠展开：

**命令检测** (`PipelineLog.vue:254-271`)

```typescript
const knownCommandMatchers = computed(() => {
  if (!pipelineConfigs.value) return [];
  const patterns: RegExp[] = [];
  pipelineConfigs.value.forEach((config) => {
    const decoded = decode(config.data);               // Base64 解码 YAML
    const matches = decoded.matchAll(commandRegex);     // 匹配 "- xxx" 模式
    for (const match of matches) {
      const rawCommand = match[1].trim();
      const patternString = rawCommand
        .replace(specialCharsRegex, '\\$&')            // 转义特殊字符
        .replace(matrixVariableRegex, '.*');            // ${VAR} → 通配
      patterns.push(new RegExp(`^${patternString}$`));
    }
  });
  return patterns;
});
```

分组算法 (`PipelineLog.vue:273-322`)：
1. 解析 pipeline YAML 配置，提取所有 `- command` 行作为 `knownCommandMatchers`
2. 遍历日志行，`rawText` 以 `+ ` 开头且匹配已知命令 → 新建 LogBlock
3. 否则追加到当前 LogBlock
4. 未匹配任何命令的前导日志归入 "Initialization" 块

**命令行渲染** (`PipelineLog.vue:82-101`)

```html
<div v-if="group.isActualCommand" class="sticky -top-4 z-10 ..."
     @click="toggleGroup(group.id)">
  <Icon name="chevron-right"
        :class="{ 'rotate-90': !collapsedCommands.has(group.id) }" />
  <span v-html="group.command.text?.substring(2)" />
</div>
```

- 命令行是 `sticky` 定位，滚动时固定在顶部
- 点击切换折叠/展开
- `substring(2)` 去掉 `+ ` 前缀

### 9.3 Error / Warning 行高亮

`PipelineLog.vue:110-113, 122-125, 132-135` 使用 CSS class 标记错误和警告行：

```html
:class="{
  'bg-red-600/40 dark:bg-red-800/50': line.type === 'error',
  'bg-yellow-600/40 dark:bg-yellow-800/50': line.type === 'warning',
}"
```

**但当前 `type` 字段始终为 `null`**（`PipelineLog.vue:385`）：

```typescript
function writeLog(line: Partial<LogLine>) {
  logBuffer.value.push({
    // ...
    type: null, // TODO: implement way to detect errors and warnings
  });
}
```

服务端 `LogEntry` 有 `Type` 字段（`LogEntryStdout=0, LogEntryStderr=1, LogEntryExitCode=2, ...`），
但前端 `writeLog()` 未使用该字段来设置 `line.type`，因此 **error/warning 高亮目前未生效**。

### 9.4 时间戳显示与去重

**时间戳格式化** (`PipelineLog.vue:342-344`)

```typescript
function formatTime(time?: number): string {
  return time === undefined ? '' : `${time}s`;
}
```

时间戳以秒为单位显示在日志行右侧。

**连续相同时间戳去重** (`PipelineLog.vue:414-427`)

```typescript
buffer = buffer.reduce((acc, line) => ({
  lastTime: line.time ?? 0,
  lines: [...acc.lines, {
    ...line,
    time: acc.lastTime === line.time ? undefined : line.time,  // 去重
  }],
}), { lastTime: -1, lines: [] as LogLine[] }).lines;
```

相邻行时间戳相同时，后续行的 `time` 设为 `undefined` → `formatTime()` 返回空字符串。
视觉上同一秒内的多行日志只在首行显示时间戳。

**注意**：这是前端渲染层的去重，不影响原始日志数据。下载日志时
（`download()` 函数）直接从 API 获取原始 `LogEntry`，不做去重。

### 9.5 ANSI 颜色与 URL 自动链接

**ANSI 转换** (`PipelineLog.vue:368-375`)

```typescript
function processText(text: string): string {
  let txt = ansiUp.value.ansi_to_html(`${decode(text)}\n`);
  txt = txt.replace(urlRegex, (url) =>
    `<a href="${url}" target="_blank" rel="noopener noreferrer" class="underline">${url}</a>`
  );
  return txt;
}
```

- `AnsiUp` 库将 ANSI 转义序列转为 HTML `<span>` + CSS class
- `use_classes = true` 启用 CSS 类模式而非内联样式
- ANSI 颜色样式定义在 `web/src/style/console.css`（支持 light/dark 主题）
- URL 正则自动将 HTTP URL 转为可点击链接

**CSS 类映射** (`console.css`):
- `.ansi-red-fg` → 红色文本（错误输出常见）
- `.ansi-green-fg` → 绿色文本（成功标记）
- `.ansi-yellow-fg` → 黄色文本（警告）
- dark 模式下有独立的高对比度配色

### 9.6 已完成 Step 的日志自动折叠

当用户打开一个已完成的 Step 日志时，`watch(loadedLogs)` (`PipelineLog.vue:580-591`)
触发自动折叠：

```typescript
watch(loadedLogs, async (isLoaded, wasLoaded) => {
  if (isLoaded && !wasLoaded && userConfig.value.collapseLogGroupsByDefault) {
    const isFinished = step.value && !['running','pending','started'].includes(step.value.state);
    if (isFinished) {
      await nextTick();
      collapseAll();
      expandLogGroupWithPageHash(route.hash);  // 保留 hash 指定的行展开
    }
  }
});
```

- 仅在 `collapseLogGroupsByDefault` 用户配置为 `true` 时生效（默认 `true`，`useUserConfig.ts:15`）
- 配置持久化在 localStorage (`woodpecker:user-config`)
- 运行中的 Step 不自动折叠，确保实时日志可见
- URL hash 指定的行所在分组会保持展开

### 9.7 当前不支持的功能

| 功能 | 状态 | 代码线索 |
|------|------|----------|
| 关键字搜索/过滤 | ❌ 不支持 | 无搜索输入框，`log` 数组全量渲染 |
| 正则过滤 | ❌ 不支持 | 无相关 UI 或逻辑 |
| 时间范围过滤 | ❌ 不支持 | 时间戳仅用于展示，无过滤交互 |
| Error/Warning 高亮 | ⚠️ 框架已建，数据未接入 | `LogLine.type` 始终为 `null`，CSS class 已定义 |
| 日志分页/懒加载 | ❌ 不支持 | `maxLineCount` 截断，代码中有 TODO(2653) |

`maxLineCount`（默认 5000，来自 `WOODPECKER_MAX_PIPELINE_LOG_LINE_COUNT`）
是当前唯一的"过滤"机制——超出部分直接截断，不渲染。
