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

---

## 10. 大日志量场景：内存占用与（未实现的）虚拟滚动

### 10.1 当前的内存控制策略

Woodpecker 前端对大日志量没有实现虚拟滚动（virtual scrolling），
采用的是**多重阈值截断 + 批量刷新**的保守策略：

**1. 服务端前置限制**（`server/api/stream.go` 的注释中未显式限制，但由 Log Mux 的背压机制兜底）

Log Mux 的 `Write` 方法对慢消费者采用**丢包**策略：

```go
// server/logging/log.go:100-104
select {
case sub.receiver <- entries:
default:
    log.Info().Msgf("subscriber channel is full -- dropping logs")
}
```

每个 subscriber 的 channel 容量固定（默认 100，`log.go:114-115`），
填满后新日志直接丢弃，防止服务端 OOM。

**2. SSE 层面的客户端缓冲限制**

`LogStreamSSE` 中每客户端有 `maxQueuedBatchesPerClient = 30` 的缓冲上限
（`server/api/stream.go:177`），SSE flush 失败时入队，超过 30 批则丢弃。

**3. 前端行数硬截断**（`PipelineLog.vue:245,398-412`）

```typescript
const maxLineCount = config.maxPipelineLogLineCount; // 默认 5000

// flushLogs() 中：
let buffer = logBuffer.value.slice(-maxLineCount);     // 新日志截断
if (buffer.length < maxLineCount && log.value) {       // 与旧日志合并后再截断
  buffer = [...log.value.slice(-(maxLineCount - buffer.length)), ...buffer];
}
log.value = buffer;
```

关键行为：
- `logBuffer` 是临时缓冲，`flushLogs()` 执行时 `slice(-maxLineCount)` 取最后 N 行
- 与 `log.value` 合并时再次确保总长度 ≤ `maxLineCount`
- 超出部分直接截断，**用户无法看到 5000 行之前的日志**
- 代码中有明确 TODO：`// TODO(2653): implement lazy-loading support`

**4. debounce 批量刷新**（`PipelineLog.vue:398`）

```typescript
const flushLogs = debounce((scroll: boolean) => { ... }, 500);
```

每 500ms 最多刷新一次 DOM，避免高频日志输出导致的页面卡顿。

### 10.2 DOM 结构与内存开销分析

当前 `PipelineLog.vue` 的模板采用 CSS Grid 三列布局：

```html
<div ref="consoleElement"
     class="grid w-full ... grid-cols-[min-content_minmax(0,1fr)_min-content]">
  <div v-for="group in groupedLogs" :key="group.id" class="contents">
    <!-- 命令标题行（sticky） -->
    <div v-if="group.isActualCommand" class="col-span-3 sticky -top-4 ...">...</div>
    <!-- 日志内容行（三列） -->
    <template v-if="!collapsedCommands.has(group.id)">
      <div v-for="line in group.lines" :key="line.index" class="contents font-mono">
        <a>行号</a>        <!-- 列 1 -->
        <span>内容</span>  <!-- 列 2，含 v-html -->
        <span>时间</span>  <!-- 列 3 -->
      </div>
    </template>
  </div>
</div>
```

每条日志行的 DOM 节点数：
- 行号列：`<a>`（含文本节点）
- 内容列：`<span>`（内部是 ANSI 转 HTML 后的多层 `<span>`）
- 时间列：`<span>`（含文本节点）

按每条日志平均 3-5 个 DOM 节点估算，5000 行日志约 1.5 万 - 2.5 万个 DOM 节点。
配合 CSS Grid + `font-mono`，Chrome 在 M1 Mac 上可勉强保持 60fps 滚动，
但在低端设备上会有明显卡顿。

### 10.3 虚拟滚动的缺失（TODO 2653）

代码中明确标注了虚拟滚动/懒加载未实现：

```typescript
// PipelineLog.vue:245
const maxLineCount = config.maxPipelineLogLineCount; // TODO(2653): implement lazy-loading support
```

潜在的实现方向（基于现有代码结构）：

1. **服务端分页接口**：`getLogs()` REST API 需支持 `offset` / `limit` / `since_line` 参数
2. **前端 IntersectionObserver**：监听滚动容器顶部/底部哨兵元素触发加载
3. **保留行号锚点**：`#L{lineNumber}` 需能精确跳转到虚拟滚动外的行
4. **分组折叠兼容**：`groupedLogs` 的折叠状态与虚拟滚动索引需同步

目前 **logBuffer → log → groupedLogs 三层响应式数据** 的结构意味着
虚拟滚动需要至少改造 `groupedLogs` computed，将其从全量计算改为
基于视口范围的增量计算。

### 10.4 浏览器端内存占用的其他来源

除了 DOM 节点，以下数据也常驻内存：

| 数据 | 来源 | 估算（5000 行日志） |
|------|------|---------------------|
| `log.value` (`LogLine[]`) | `flushLogs()` 写入 | ~2-5 MB（含 ANSI 转 HTML 字符串） |
| `logBuffer.value` | `writeLog()` 累积（flush 前） | < 1 MB |
| `groupedLogs` computed | `log.value` 派生 | ~5-10 MB（含 LogBlock 对象） |
| AnsiUp 实例状态 | ANSI 颜色状态机 | < 100 KB |
| collapsedCommands Set | 用户折叠状态 | < 10 KB |
| EventSource 缓冲 | SSE 浏览器内部缓冲 | 视浏览器而定，通常 < 1 MB |

**总计约 8-16 MB / Step**，多 Step 切换时由于每次 `loadLogs()` 都会重置
`log.value = undefined` + `logBuffer.value = []`，理论上旧 Step 的日志数据
可被 GC 回收。但实际测试中 `groupedLogs` computed 的缓存可能延迟释放。

---

## 11. Build 取消 / 重跑：前后端状态同步路径

### 11.1 取消 Pipeline：前端触发

`PipelineWrapper.vue:201-208` 实现取消按钮逻辑：

```typescript
const { doSubmit: cancelPipeline, isLoading: isCancelingPipeline } = useAsyncAction(async () => {
  if (!pipeline.value?.number) {
    throw new Error('Unexpected: Pipeline number not found');
  }
  await apiClient.cancelPipeline(repo.value.id, pipeline.value.number);
  notifications.notify({ title: i18n.t('repo.pipeline.actions.cancel_success'), type: 'success' });
});
```

前端仅发送一个 `POST /api/repos/{id}/pipelines/{number}/cancel` 请求，
**不直接修改本地 Store**，依赖服务端通过 SSE 推送状态变更来更新 UI。

`apiClient.cancelPipeline()` 定义在 `web/src/lib/api/index.ts:137-139`：

```typescript
async cancelPipeline(repoId: number, pipelineNumber: number): Promise<unknown> {
  return this._post(`/api/repos/${repoId}/pipelines/${pipelineNumber}/cancel`);
}
```

响应为 `204 No Content`，不含任何数据。

### 11.2 取消 Pipeline：服务端处理

`server/api/pipeline.go:475` 的 `CancelPipeline` → `server/pipeline/cancel.go:32` 的 `Cancel()`：

```
POST /cancel
    │
    ▼
CancelPipeline (api handler)
    │
    ├── 鉴权：用户 + repo 权限校验
    │
    ▼
pipeline.Cancel()  (cancel.go)
    │
    ├── 前置校验：仅允许 running / pending / blocked 状态取消
    │
    ├── Step 1：队列批量驱逐
    │     Scheduler.ErrorAtOnce([workflowIDs...], queue.ErrCancel)
    │       → pending 任务直接从队列移除
    │       → running 任务通过 Wait() 返回 ErrCancel 通知 Agent
    │
    ├── Step 2：DB 状态更新（两分支）
    │     ├── pending Workflow / Step → UpdateToStatusSkipped / Canceled
    │     └── running Workflow / Step → 不修改 DB（等 Agent 上报 cancel 信号）
    │
    ├── Step 3：Pipeline 终态判定
    │     ├── 全部为 pending → canceled（被用户取消）
    │     └── 存在 running → killed（被强制杀死，可能有副作用）
    │
    ├── Step 4：Forge commit status 更新
    │     updatePipelineStatus() → GitHub/GitLab 等 commit 状态同步
    │
    └── Step 5：PubSub 广播
          publishToTopic(killedPipeline, repo)
            → EventStreamSSE → 前端 PipelineStore 更新
```

**关键状态区分**：

| Pipeline 状态 | 含义 | 触发条件 |
|---------------|------|----------|
| `canceled` | 用户主动取消，且所有子任务尚未开始 | 全部 Workflow/Step 为 pending |
| `killed` | 被强制终止，可能有正在运行的任务 | 存在 running 状态的 Workflow/Step |

两种状态在前端 UI 上表现类似（`PipelineStatusIcon` 都用灰色图标），
但语义不同：`canceled` 可重跑无副作用，`killed` 需用户确认残留资源。

### 11.3 取消 Pipeline：Agent 侧响应

服务端 `Scheduler.ErrorAtOnce()` 注入 `queue.ErrCancel` 后，
Agent 侧通过 `Wait()` 感知到取消信号：

```
Agent poll loop:
  Poll() 获取任务
  Work() 执行 Step
    ├── 内部持续调用 Wait(workflowID)
    ├── Wait() 返回 ErrCancel → 触发终止流程
    └── 终止所有正在执行的容器 / 进程
  RPC.Done(workflowID, state: killed) → Server
```

Agent 上报 `RPC.Done()` 后，服务端：
1. `RPC.Done()` 更新 DB 中的 Workflow/Step 状态为终态
2. 调用 `logger.Close()` 关闭日志流（SSE 端收到 `event: eof`）
3. `notify()` 推送最终 Pipeline 状态

**状态同步的最终确认来自 Agent，而非 Cancel API**。前端点击取消后，
Pipeline 状态先在服务端变为 `killed` / `canceled` 并通过 SSE 推送，
但正在执行的 Step 状态仍为 `running`，直到 Agent 上报 `Done`。

### 11.4 重跑 Pipeline：前端触发

`PipelineWrapper.vue:210-219` 实现重跑按钮：

```typescript
const { doSubmit: restartPipeline, isLoading: isRestartingPipeline } = useAsyncAction(async () => {
  const newPipeline = await apiClient.restartPipeline(repo.value.id, pipelineId.value, {
    fork: true,
  });
  notifications.notify({ title: i18n.t('repo.pipeline.actions.restart_success'), type: 'success' });
  await router.push({
    name: 'repo-pipeline',
    params: { pipelineId: newPipeline.number },
  });
});
```

**关键差异**：重跑不是修改原 Pipeline，而是**创建新 Pipeline**。
`router.push()` 立即跳转到新 Pipeline 页面。

`apiClient.restartPipeline()` (`web/src/lib/api/index.ts:149-156`)：

```typescript
async restartPipeline(repoId, pipeline, opts?: { event?; deploy_to?; fork? }) {
  const query = encodeQueryString(opts);
  return this._post(`/api/repos/${repoId}/pipelines/${pipeline}?${query}`);
}
```

响应 `200 OK`，返回新 Pipeline 对象（含新的 `number`）。

### 11.5 重跑 Pipeline：服务端处理

`server/pipeline/restart.go:32` 的 `Restart()`：

```
POST /pipelines/{number}?fork=true
    │
    ▼
Restart()  (restart.go)
    │
    ├── 前置校验：blocked 状态不可重跑
    │
    ├── Step 1：获取旧 Pipeline 的 config（YAML）
    │     store.ConfigsForPipeline(lastPipeline.ID)
    │     → 若有 ConfigService，重新 fetch（可能变更）
    │
    ├── Step 2：创建新 Pipeline 对象
    │     createNewOutOfOld(lastPipeline)
    │       → ID/Number 重置为 0
    │       → Status = pending
    │       → Started/Finished = 0
    │       → Parent = lastPipeline.Number（父子关联）
    │       → RerunCount++
    │
    ├── Step 3：持久化 + 关联 configs
    │     store.CreatePipeline(newPipeline)
    │     linkPipelineConfigs(configs, newPipeline.ID)
    │
    ├── Step 4：解析 Pipeline → 生成 Workflow/Step
    │     createPipelineItems()
    │
    ├── Step 5：启动 Pipeline
    │     publishPipeline() → Forge commit status + 通知
    │     start() → 入队 Scheduler
    │
    └── 返回新 Pipeline（含新 number）
```

**新旧 Pipeline 的关联**：
- 新 Pipeline 的 `parent` 字段指向旧 Pipeline 的 `number`
- 旧 Pipeline 状态不变（仍为 success/failure/killed/canceled）
- 前端跳转后旧 Pipeline 的 SSE 连接在组件卸载时由 `onBeforeUnmount` 关闭

### 11.6 自动取消旧 Pipeline：`cancelPreviousPipelines`

`server/pipeline/cancel.go:100-158` 的 `cancelPreviousPipelines()` 在新 Pipeline
启动时自动取消同分支/同 ref 的旧 Pipeline：

```go
// cancel.go:109
eventIncluded := slices.Contains(repo.CancelPreviousPipelineEvents, pipeline.Event)
if !eventIncluded { return nil }

// cancel.go:120-133
pipelineNeedsCancel := func(active *model.Pipeline) bool {
  if active.Event != pipeline.Event { return false }
  switch pipeline.Event {
  case model.EventPush:
    return pipeline.Branch == active.Branch   // 同分支的 push 事件
  default:
    return pipeline.Refspec == active.Refspec  // 同 refspec 的其他事件
  }
}
```

这是 repo 级配置 `cancel_previous_pipeline_events` 控制的功能，
默认包含 `push`、`pull_request` 等高频事件。被自动取消的 Pipeline
其 `cancel_info.superseded_by` 字段指向新 Pipeline 的 `number`。

### 11.7 Approve / Decline：blocked Pipeline 的状态流转

`blocked` 状态 Pipeline 需要人工审核后才能执行：

| 操作 | API | 状态流转 |
|------|-----|----------|
| Approve | `POST /pipelines/{n}/approve` | `blocked → pending → running` |
| Decline | `POST /pipelines/{n}/decline` | `blocked → declined` |

**Approve 流程** (`server/pipeline/approve.go:31`)：
1. 校验 Pipeline 必须为 `blocked`
2. 将 Status 设为 `pending`（这一步必须在创建 Workflow 前完成，
   因为 Workflow 初始状态由 Pipeline Status 派生）
3. 解析 config → 创建 Workflow/Step
4. `UpdateToStatusPending()` 正式更新 DB
5. `publishPipeline()` + `start()` 入队执行

**Decline 流程** (`server/pipeline/decline.go:30`)：
1. 校验 Pipeline 必须为 `blocked`
2. `UpdateToStatusDeclined()` → DB 状态变更
3. `updatePipelineStatus()` → Forge commit status 同步
4. `publishToTopic()` → SSE 推送前端

两种操作的前端实现与 Cancel 相同：发 POST 请求，不直接修改 Store，
等待 SSE 推送更新。

---

## 12. 页面离开与连接释放：EventSource 生命周期与泄漏防护

### 12.1 连接类型总览

Woodpecker 前端有两类 EventSource 连接，生命周期不同：

| 连接类型 | 创建位置 | 生命周期 | 释放时机 |
|----------|----------|----------|----------|
| 状态事件流 (`/api/stream/events`) | `useEvents.ts` 调用 `apiClient.on()` | **应用级**：`main.ts` 中 `useEvents()` 启动后常驻 | 页面刷新 / 标签关闭时浏览器自动释放 |
| 日志流 (`/api/stream/logs/...`) | `PipelineLog.vue` 中 `loadLogs()` | **组件级**：随 PipelineLog 组件实例生命周期 | 组件卸载 / Step 切换 / Step 完成 |

### 12.2 状态事件流：全局常驻，无显式释放

`web/src/compositions/useEvents.ts` 的实现：

```typescript
let initialized = false;

export default () => {
  if (initialized) return;   // 单例保护，只初始化一次
  const repoStore = useRepoStore();
  const pipelineStore = usePipelineStore();

  initialized = true;

  apiClient.on((data) => {
    repoStore.setRepo(repo);
    pipelineStore.setPipeline(repo.id, pipeline);
  });
};
```

`web/src/main.ts:24` 中全局调用：

```typescript
useEvents();  // 应用启动时建立连接，永不关闭
```

**这个连接没有 `onBeforeUnmount` 或路由守卫释放**。
理由：用户在 SPA 内任何页面都可能看到 Pipeline 状态更新（Repo 列表、侧边栏 Feed、详情页），
因此需要持续接收事件。

**潜在泄漏点**：EventSource 对象存储在 `ApiClient.on()` 的闭包中，
外部没有引用。每次 `apiClient.on()` 调用都会**新建一个 EventSource**，
但由于 `initialized` 单例保护，`useEvents()` 实际只调用一次 `on()`。
如果有其他地方直接调用 `apiClient.on()` 而不做单例保护，会产生泄漏。

`apiClient.on()` (`web/src/lib/api/index.ts`) 内部：

```typescript
on(callback) {
  return this._subscribe('/api/stream/events', callback);  // 返回 EventSource
}
```

返回值未被 `useEvents()` 保存，意味着**无法通过代码主动关闭全局状态流**。
只能依赖浏览器 `beforeunload` 事件。

### 12.3 日志流：组件级精细释放

`PipelineLog.vue` 中有 **4 条释放路径**，覆盖各种场景：

**路径 1：组件卸载** (`PipelineLog.vue:547-549`)

```typescript
onBeforeUnmount(() => {
  stream.value?.close();
});
```

用户导航离开 Pipeline 详情页（如点击 Repo 列表、关闭标签）时，
Vue 生命周期钩子确保 EventSource 被 `close()`。

**路径 2：Step 切换** (`PipelineLog.vue:481`)

```typescript
async function loadLogs() {
  stream.value?.close();  // 建新连接前先关旧的
  // ... 建立新连接
}
```

`loadLogs()` 是日志加载的入口，无论触发来源如何（URL stepId 参数变化、
Step 状态变化、初次加载），都先关闭旧连接。

**路径 3：Step 状态变为终态** (`PipelineLog.vue:555-564`)

```typescript
watch(step, async (newStep, oldStep) => {
  if (oldStep?.state !== newStep?.state) {
    await loadLogs();  // loadLogs 内部会关闭 SSE + 切换 REST
  }
});
```

Step 完成（running → success/failure/killed/canceled）时，
`loadLogs()` 检测到 `step.state !== 'running'`，走 REST 路径，
SSE 连接被关闭。

**路径 4：SSE 收到 `event: eof`**

服务端在 Step 完成时调用 `logger.Close()` → Log Mux 关闭 stream →
LogStreamSSE handler 的 `logChan` 收到 nil/关闭 → SSE 发送：

```
event: eof
data: eof
```

前端 `ApiClient._subscribe()` 中 `reconnect: true` 模式下**不注册 onerror**，
因此浏览器 EventSource 默认行为是尝试重连。但由于：

1. 重连时服务端 `LogStreamSSE` 检测 Step 状态已非 running → 返回 `event: error`
2. 此时 `watch(step)` 已通过状态通道收到 Step 终态 → 触发 `loadLogs()` → `stream.value?.close()`

最终在路径 3 的掩护下连接会被释放。

### 12.4 路由跳转的连接释放分析

Vue Router 使用 `createWebHistory()` 模式，SPA 内跳转不触发页面刷新。
Pipeline 详情页路由为 `/repos/:repoId/pipeline/:pipelineId/:stepId?`。

**场景 A：在 Pipeline 详情页内切换 Step**
- 仅 URL 的 `:stepId` 参数变化
- `PipelineLog` 组件不卸载（相同路由匹配）
- `watch(stepSlug)` 触发 `loadLogs()` → 路径 2 释放旧日志流
- 全局状态流不受影响

**场景 B：从 Pipeline 详情页跳转到其他页面**
- `PipelineWrapper` → `Pipeline` → `PipelineLog` 三级组件依次卸载
- 每个 `PipelineLog` 实例的 `onBeforeUnmount` 被调用 → 路径 1 释放日志流
- 全局状态流保持连接（正确行为）

**场景 C：刷新 / 关闭标签**
- 浏览器 `beforeunload` 事件 → 自动关闭所有 EventSource / WebSocket
- 服务端侧通过 HTTP 连接断开检测 LogStreamSSE handler 的 `ctx.Writer.CloseNotify()`
  或 `<-c.Request.Context().Done()`，退出 goroutine

### 12.5 服务端连接泄漏防护

`server/api/stream.go` 中 SSE handler 使用 `context.Context` 做生命周期管理：

```go
// EventStreamSSE: stream.go:121-127
pub, sub, err := s.pubsub.Subscribe(c)
defer s.pubsub.Unsubscribe(pub, sub)   // 退出时取消订阅

// LogStreamSSE: stream.go:189-193
logger.Tail(step.ID, logChan)
defer func() {
  logger.CloseReceiver(step.ID, logChan)  // 退出时从 Log Mux 注销 subscriber
}()
```

两个 SSE handler 都有双重退出信号：

```go
for {
  select {
  case ev := <-sub:        // PubSub 事件 / Log Mux 日志
    // 处理并写入 SSE
  case <-c.Request.Context().Done():  // 客户端断开（HTTP 连接关闭）
    return                             // handler 退出，defer 清理资源
  case <-ticker.C:         // 30s ping 保活
    // 发送 ping
  }
}
```

客户端关闭 EventSource 后，TCP 连接断开，Go 的 `http.ResponseWriter` 检测到
`context.Canceled`，`c.Request.Context().Done()` channel 被触发，
handler goroutine 正常退出，`defer` 执行 PubSub 退订和 Log Mux 注销。

### 12.6 泄漏风险评估

| 场景 | 是否泄漏 | 防护机制 |
|------|----------|----------|
| SPA 内路由跳转 | ✅ 安全 | `onBeforeUnmount` + `loadLogs()` 前置 close |
| 页面刷新 / 关闭标签 | ✅ 安全 | 浏览器自动释放 + 服务端 context Done |
| Step 完成 | ✅ 安全 | `watch(step)` 触发切换 REST + SSE eof |
| 网络异常断开 | ✅ 安全 | 浏览器 EventSource 自动重连 / 服务端 30s ping 超时 |
| `apiClient.on()` 多次调用 | ⚠️ 风险 | `initialized` 单例保护仅覆盖 `useEvents()`，外部直接调用可能泄漏 |
| 用户长时间停留在 running Step 页面后睡眠唤醒 | ✅ 安全 | TCP 超时 + EventSource 自动重连（Last-Event-ID 续传） |
| PipelineFeed 侧边栏组件多次挂载卸载 | ✅ 安全 | Feed 不建立独立连接，只消费 `PipelineStore.pipelineFeed` computed |

### 12.7 其他需要释放的资源

除 EventSource 外，Pipeline 页面还有以下资源需要清理：

| 资源 | 释放位置 | 机制 |
|------|----------|------|
| 日志流 `setInterval` | `useElapsedTime.ts` → `onBeforeUnmount` | 每个 `PipelineStepDuration` 实例的独立计时器 |
| Favicon 状态 | `PipelineWrapper.vue:223-225` → `onBeforeUnmount` | `favicon.updateStatus('default')` 恢复默认图标 |
| `log.value` 大数组 | `loadLogs()` → `log.value = undefined` | 每次重加载时显式置空，配合 GC 回收 |
| Blob URL（下载日志） | `PipelineLog.vue:468` | `window.URL.revokeObjectURL(fileURL)` 下载完成后立即释放 |
| scroll 监听 / hash watch | Vue 响应式自动管理 | 组件卸载时 watcher 自动注销 |

**唯一的已知潜在泄漏**：全局状态事件流的 EventSource 没有显式 `close()` 方法，
仅依赖浏览器 `beforeunload`。如果未来需要支持"登出后断开连接"等场景，
需改造 `ApiClient` 暴露 `closeEventStream()` 方法。

---

## 13. i18n 多语言切换对 Build 输出文本的影响

### 13.1 翻译范围：UI 文本 vs Build 数据文本

Woodpecker 前端的 i18n 体系基于 `vue-i18n`，但翻译严格限定于 **UI 界面文本**。
Build 相关的数据（日志输出、Step 名称、状态 token、commit message、分支名等）
**一律不做翻译**，原文输出。

UI 文本 vs 数据文本的分界线：

| 类别 | 是否翻译 | 示例 | 代码位置 |
|------|----------|------|----------|
| 页面标题 | ✅ 翻译 | `repo.pipeline.log_title` → "Build log" | `PipelineLog.vue:21` |
| 按钮文案 | ✅ 翻译 | `cancel_success` → "Pipeline canceled" | `PipelineWrapper.vue:207` |
| 状态图标旁文本 | ✅ 翻译 | `time.not_started` → "not started yet" | `usePipeline.ts:73` |
| 退出代码提示 | ✅ 翻译 | `repo.pipeline.exit_code` → "Exit code {exitCode}" | `PipelineLog.vue:159` |
| 全屏/退出全屏 | ✅ 翻译 | `fullscreen` / `exit_fullscreen` | `PipelineLog.vue:27` |
| Step 名称 | ❌ 不翻译 | 用户 `.woodpecker.yml` 中定义的 `name:` | 原样展示 |
| Pipeline 状态值 | ❌ 不翻译 | `running` / `success` / `failure` 等 token | 仅用于图标切换 |
| 日志内容 | ❌ 不翻译 | Step 命令执行的 stdout/stderr | 原文 Base64 解码后展示 |
| 命令行前缀 `+` | ❌ 不翻译 | shell 执行命令（如 `+ npm run build`） | `substring(2)` 去掉前缀 |
| Commit message | ❌ 不翻译 | Git commit 原始 message | `escapeHtml()` 后原文展示 |
| Branch / Tag / PR 引用 | ❌ 不翻译 | `refs/heads/main` → `main`，`#123` | `prettyRef` 正则裁剪 |
| 初始化块标题 | ❌ 不翻译 | `"Initialization"` 硬编码字符串 | `PipelineLog.vue:300` |
| 日志行号 | ❌ 不翻译 | `1, 2, 3...` 数字 | `Intl.NumberFormat` 未应用 |

### 13.2 i18n 初始化与切换流程

`web/src/compositions/useI18n.ts` 实现了完整的 i18n 生命周期：

**初始化（模块加载时）**：

```typescript
const userLanguage = getUserLanguage();
// 1. navigator.language 检测浏览器语言
// 2. localStorage 'woodpecker:locale' 覆盖
// 3. 不在 SUPPORTED_LOCALES 列表中 → 取短码（如 zh-CN → zh）

const fallbackLocale = 'en';
export const i18n = createI18n({
  locale: userLanguage,
  legacy: false,        // Composition API 模式
  globalInjection: true,  // 注入 $t 全局方法
  fallbackLocale,       // 缺失翻译回退英语
});

// 预加载 fallback 和用户语言
loadLocaleMessages(fallbackLocale).catch(console.error);
loadLocaleMessages(userLanguage).catch(console.error);
```

**动态切换**（`setI18nLanguage()`）：

```typescript
export const setI18nLanguage = async (lang: string): Promise<void> => {
  if (!i18n.global.availableLocales.includes(lang)) {
    await loadLocaleMessages(lang);   // 懒加载：动态 import JSON
  }
  i18n.global.locale.value = lang;   // 切换 locale，触发响应式重渲染
  await setDateLocale(lang);         // 同步切换日期/时间格式化 locale
};
```

关键点：`loadLocaleMessages()` 使用 `await import(\`~/assets/locales/\${locale}.json\`)`
做代码分割，非默认语言的 JSON 按需加载。

### 13.3 语言切换对 Build 页面的实时影响

由于 `vue-i18n` 的 `locale` 是响应式 ref，切换语言后：

1. **所有 `$t()` / `t()` 调用自动重算** — 按钮、标题、提示文案实时刷新
2. **`useDate.currentLocale` 同步更新** — `timeAgo()`、`prettyDuration()`、`toLocaleString()`
   下次调用使用新 locale
3. **`PipelineLog` 的 `Initialization` 字符串不刷新** — 因为它是在 `groupedLogs`
   computed 生成时硬编码的：
   ```typescript
   // PipelineLog.vue:298-301
   if (currentBlock === null || currentBlock.command !== null) {
     currentBlock = {
       command: { text: 'Initialization', ... },  // 非 $t 翻译
       ...
     };
   }
   ```
   这是一个**小缺陷**：`Initialization` 分组标题没有走 `$t()`，语言切换后不更新。

### 13.4 Pipeline 状态值：Token 不翻译，只有图标变化

`PipelineStatus`（`running`、`success`、`failure` 等）是前后端通用的枚举 token，
**不在任何地方做翻译映射**。前端通过状态值驱动：

- **图标选择**：`PipelineStatusIcon` 组件根据 `status` 选择对应 SVG（check-circle /
  x-circle / spinner 等）
- **CSS 颜色**：Tailwind 的 `text-green-500` / `text-red-500` 等通过状态 token 判断
- **工具提示**：`title` 属性同样使用英文 token 或图标替代，没有 `$t('status.' + status)`
  映射表

用户感知状态的主要方式是**图标 + 颜色**，而非文本标签。

### 13.5 日期/时间格式化：i18n 联动但本地化

`useDate.ts` 维护独立的 `currentLocale` 变量，随 `setI18nLanguage()` 同步：

```typescript
let currentLocale = 'en';

function toLocaleString(date: Date, tz?: string) {
  return date.toLocaleString(currentLocale, { dateStyle: 'short', timeStyle: 'short', timeZone: tz });
}

function prettyDuration(durationMs: number) {
  return Intl.NumberFormat(currentLocale, { style: 'unit', unit: 'hour', unitDisplay: 'long' })
    .format(Math.round(t.totalHours));
}

function timeAgo(date: number) {
  const formatter = new Intl.RelativeTimeFormat(currentLocale);
  return formatter.format(-Math.round(interval), 'day'); // 等
}
```

**注意时区**：`toLocaleString()` 接受可选的 `tz` 参数，但 `usePipeline.ts` 调用时
未传入时区，使用浏览器本地时区。Pipeline 的 `created/started/finished` 是 Unix 秒级
UTC 时间戳，`new Date(start * 1000)` 转为本地时间后用 `toLocaleString()` 格式化，
因此用户看到的是**其浏览器本地时区**的时间，而非服务器时区。

### 13.6 静态 token 的翻译覆盖验证

翻译文件路径：`web/src/assets/locales/{locale}.json`（20+ 种语言），
关键 Build 相关 key 组织：

```
repo.pipeline.log_title        // 日志面板标题
repo.pipeline.exit_code        // 退出代码提示
repo.pipeline.actions.cancel_success
repo.pipeline.actions.restart_success
repo.pipeline.actions.approve_success
repo.pipeline.actions.decline_success
time.not_started               // 未开始
time.just_now                  // 刚刚
cancel / fullscreen / exit_fullscreen  // 通用按钮
```

**未覆盖/未翻译的硬编码英文**（搜索代码发现）：

| 硬编码文本 | 位置 | 影响 |
|------------|------|------|
| `"Initialization"` | `PipelineLog.vue:300` | 日志初始化块标题 |
| `"-"` (not started) | `usePipeline.ts:33` | 注释中原本计划用 `t('time.not_started')` 改为硬编码 `-` |
| `LogEntry.Type` 未接入 | `PipelineLog.vue:385` | `type: null` TODO，error/warning 行无法区分 |

---

## 14. 深色 / 浅色主题：日志高亮切换路径与样式注入

### 14.1 主题切换核心机制：CSS 类切换 + CSS 变量

Woodpecker 使用 **`@vueuse/core` 的 `useColorMode()`** 做主题状态管理，
配合 **Tailwind `dark:` 变体** + **CSS 自定义属性** 实现样式切换：

```typescript
// useTheme.ts:4-10
const {
  store: storeTheme,       // 用户手动选择：light / dark / auto
  state: resolvedTheme,    // 解析后结果：light 或 dark
  system: systemTheme,     // 系统 prefers-color-scheme
} = useColorMode({
  storageKey: 'woodpecker:theme',  // localStorage key
});
```

**应用主题** (`useTheme.ts:12-24`)：

```typescript
function updateTheme() {
  if (resolvedTheme.value === 'dark') {
    document.documentElement.classList.remove('light');
    document.documentElement.classList.add('dark');
    document.documentElement.setAttribute('data-theme', 'dark');
    document.querySelector('meta[name=theme-color]')?.setAttribute('content', '#2A2E3A');
  } else {
    document.documentElement.classList.remove('dark');
    document.documentElement.classList.add('light');
    document.documentElement.setAttribute('data-theme', 'light');
    document.querySelector('meta[name=theme-color]')?.setAttribute('content', '#369943');
  }
}
```

**三层联动**：
1. **`.dark` / `.light` CSS class** — 供 Tailwind `@custom-variant dark` 使用
2. **`[data-theme=dark]` / `[data-theme=light]` 属性选择器** — 供 CSS 变量切换使用
3. **`<meta name=theme-color>`** — 移动端浏览器地址栏颜色

### 14.2 Tailwind dark 变体的编译机制

`web/src/tailwind.css:7`：

```css
@custom-variant dark (&:is(.dark *));
```

这是 Tailwind v4 的自定义 variant 语法：**任何带 `dark:` 前缀的 class
只有在祖先元素有 `.dark` class 时才生效**。编译后：

```css
/* 源码：class="text-gray-700 dark:text-gray-200" */
.text-gray-700 { color: #374151; }
.dark .text-gray-200 { color: #e5e7eb; }   /* &:is(.dark *) 展开后 */
```

Pipeline 页面随处可见这种用法：
- `PipelineLog.vue:110`：`bg-red-600/40 dark:bg-red-800/50`
- `PipelineLog.vue:84`：`bg-blue-900`（选中行背景，dark 模式直接用深蓝色）
- 日志容器：`bg-wp-code-300 text-wp-code-text-100`（依赖 CSS 变量）

### 14.3 CSS 自定义属性（CSS Variables）：两套配色方案

`web/src/style.css:3-114` 定义了 `--wp-*` 系列 CSS 变量，通过
`[data-theme=light]` 和 `[data-theme=dark]` 属性选择器提供两套值：

```css
:root,
:root[data-theme='light'] {
  --wp-background-100: var(--color-white);
  --wp-text-200: var(--color-gray-700);
  --wp-code-100: var(--color-int-wp-secondary-300);   /* 命令行背景：深蓝灰 */
  --wp-code-300: var(--color-int-wp-secondary-600);   /* 日志容器背景：#2a2e3a */
  --wp-code-text-100: var(--color-gray-200);
  --wp-code-text-alt-100: var(--color-gray-300);
  --wp-link-100: var(--color-blue-600);
}

:root[data-theme='dark'] {
  --wp-background-100: var(--color-int-wp-secondary-200);
  --wp-text-200: var(--color-gray-200);
  --wp-code-100: var(--color-int-wp-secondary-700);   /* #222631 */
  --wp-code-300: var(--color-int-wp-secondary-800);   /* #1B1F28 */
  --wp-code-text-100: var(--color-gray-200);
  --wp-code-text-alt-100: var(--color-gray-400);
  --wp-link-100: var(--color-blue-400);
}
```

**关键观察**：代码块背景色在 light 模式下也使用深灰（`--wp-code-300: #2a2e3a`），
这是 Woodpecker 的设计选择——**日志面板永远使用深色背景**，无论全局主题如何。
因此 `PipelineLog` 容器本身的背景在两种主题下差异不大（只是深浅的区别）。

Tailwind class `bg-wp-code-300` 不是实际的 Tailwind 颜色，而是通过
`@source './**/*.css'` 扫描 CSS 变量中的 `--wp-code-300` 自动映射为 utility class：

```css
.bg-wp-code-300 {
  background-color: var(--wp-code-300);
}
```

### 14.4 ANSI 日志高亮：`.dark` 选择器覆盖两套颜色

`web/src/style/console.css` 是 ANSI 颜色专用样式表，在 `PipelineLog.vue:166`
通过模块级 `import '~/style/console.css'` 引入。它包含 **16 种前景色 + 16 种背景色**
的两套配色：

```css
/* ===== Light 模式默认配色 ===== */
.ansi-red-fg      { color: #cc0000; }    /* 标准红：较深 */
.ansi-green-fg    { color: #4e9a06; }    /* 标准绿 */
.ansi-yellow-fg   { color: #c4a000; }    /* 标准黄：棕黄色 */
/* ... 8 种标准色 + 8 种亮色 ... */

/* ===== Dark 模式覆盖配色 ===== */
.dark .ansi-red-fg      { color: #ff7070; }    /* 标准红：浅红，在深色背景可读 */
.dark .ansi-green-fg    { color: #b0f986; }    /* 标准绿：荧光绿 */
.dark .ansi-yellow-fg   { color: #c6c502; }    /* 标准黄：高饱和度 */
/* ... 同样 16 种，均提升亮度和饱和度以适应深色背景 ... */
```

**AnsiUp 库的配合**：

```typescript
// PipelineLog.vue:239-240
const ansiUp = ref(new AnsiUp());
ansiUp.value.use_classes = true;   // 关键：使用 CSS class 模式，而非内联 style
```

`use_classes = true` 使得 AnsiUp 在解析 ANSI 转义序列时输出：
```html
<span class="ansi-red-fg">error: something failed</span>
```
而非
```html
<span style="color: rgb(204, 0, 0);">error: something failed</span>
```

这样主题切换时，浏览器**自动重新匹配 `.dark .ansi-red-fg` 选择器**，
无需重新渲染日志行，**零成本切换 ANSI 配色**。

### 14.5 主题切换的样式注入时序

**首次加载**：

```
1. index.html <link> 加载 style.css + tailwind.css
   → 所有 --wp-* 变量、.ansi-*-fg/bg 基础样式、dark 变体 class 注册
2. main.ts 执行 useTheme() 的 module 级初始化
   → useColorMode() 读取 localStorage 'woodpecker:theme'
   → updateTheme() → document.documentElement 加上 .dark / .light
   → 浏览器立即重绘：dark 变体 class 和 .dark .ansi-* 生效
3. PipelineLog 组件挂载时 import '~/style/console.css'
   → 因为是在 <script setup> 顶层 import，构建时已合并到主 CSS
   → 实际无额外网络请求
```

**运行时切换**：

```
用户点击主题切换按钮
  → storeTheme.value = 'dark' (写入 localStorage)
  → watch([storeTheme, systemTheme], updateTheme) 触发
  → updateTheme():
    1. document.documentElement.classList.toggle('dark')
    2. setAttribute('data-theme', 'dark')
    3. 更新 meta[theme-color]
  → 浏览器 Recalculate Style：
    - 所有 .dark :is(*) 变体重新匹配
    - [data-theme=dark] 的 --wp-* 变量重新赋值
    - .dark .ansi-*-fg/bg 选择器命中
  → 无需重新执行 JavaScript：
    - 不触发 Vue 组件重渲染
    - 不重新遍历日志行
    - 纯 CSS 级样式切换，性能零开销
```

### 14.6 URL 自动链接的主题适配

`PipelineLog.vue:370-374` 中 URL 自动转换使用了 Tailwind class `underline`，
而链接颜色通过 CSS 变量 `--wp-link-100` 间接适配主题：

```css
/* style.css */
:root[data-theme='light'] { --wp-link-100: var(--color-blue-600); }  /* 深蓝 */
:root[data-theme='dark']  { --wp-link-100: var(--color-blue-400); }  /* 浅蓝 */
```

Vue Router 的 `<router-link>` 也使用相同机制。由于全局链接样式通过 Tailwind 预设
和 CSS 变量处理，日志内的普通 `<a href>` 标签自动继承主题色。

### 14.7 选中行高亮：主题不感知的问题

`PipelineLog.vue:110`：

```html
:class="{ 'bg-blue-600/30': isSelected(line) }"
```

选中行使用 `bg-blue-600/30`（`rgba(37, 99, 235, 0.3)`），**未加 dark: 变体**。
在 light 模式和 dark 模式下都使用相同的蓝色半透明背景。由于日志面板背景在
两种主题下都是深色调，这基本合理，但 dark 模式下对比度略低。

### 14.8 样式注入总览

| 样式层 | 注入方式 | 主题切换机制 |
|--------|----------|--------------|
| Tailwind utilities (bg/text/border) | `@import 'tailwindcss'` + `@custom-variant dark` | `.dark` class → CSS cascade 选择器重匹配 |
| `--wp-*` 语义色变量 | `style.css` 中 `:root[data-theme=...]` | `data-theme` 属性切换 → CSS 变量重计算 |
| ANSI 颜色 (`.ansi-*-fg/bg`) | `console.css` 基础 + `.dark .ansi-*` 覆盖 | `.dark` class → 选择器级联覆盖（16 fg × 2 + 16 bg × 2 = 64 规则） |
| Error/Warning 行背景 | `bg-red-600/40 dark:bg-red-800/50` | Tailwind dark 变体 |
| URL 链接颜色 | `--wp-link-100` 变量 | data-theme 属性切换 |
| 命令行 sticky 背景 | `bg-wp-code-100` 变量 | data-theme 属性切换 |

---

## 15. Build 时间戳与服务端时钟漂移对账机制

### 15.1 时间戳来源分层：Agent / Server / Browser 三台机器

Woodpecker 是分布式系统，涉及**三种独立时钟源**的时间戳：

```
┌─────────────────────────────────────────────────────────────────┐
│  Agent 端（执行流水线的机器）                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  • LineWriter.startTime = time.Now().UTC()               │   │
│  │    → LogEntry.Time = int64(time.Since(startTime).Seconds)│   │
│  │    相对秒数，Step 级起点                                  │   │
│  │                                                          │   │
│  │  • StepState.Started = time.Now().Unix()                 │   │
│  │  • StepState.Finished = time.Now().Unix()                │   │
│  │    Unix 秒级 UTC 绝对时间戳                                │   │
│  │                                                          │   │
│  │  • WorkflowState.Started / Finished                      │   │
│  │    = state.Workflow.Started / Finished                   │   │
│  │    (Runtime 端 startTime 通过 traceStep 传递)             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                         ↕ gRPC                                   │
│  Server 端（Woodpecker Server 进程）                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  • Pipeline.Created = time.Now().Unix() (接收 webhook)   │   │
│  │  • Pipeline.Updated = 每次 Status 更新时 time.Now()      │   │
│  │  • Agent 上报的 Started/Finished 不转换，直接入库        │   │
│  │  • 兜底：若 Agent 上报的 Started=0 → 用 Server time.Now()│   │
│  │    兜底：若 Agent 上报的 Finished=0 → 用 Server time.Now()│   │
│  └──────────────────────────────────────────────────────────┘   │
│                         ↕ HTTP / SSE                             │
│  Browser 端（用户浏览器）                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  • since / duration: Date.now() - start * 1000           │   │
│  │    (running Pipeline 的实时计时器)                        │   │
│  │  • toLocaleString(): new Date(ts * 1000) + 本地时区       │   │
│  │  • timeAgo: Date.now() 做相对时间                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 日志行时间戳：Step 级相对秒数，非绝对时间

这是最容易误解的地方：**`LogEntry.Time` 不是 Unix 时间戳，而是从 Step 开始起算的
相对秒数**。

**Agent 端生成**（`agent/log/line_writer.go:41-70`）：

```go
func NewLineWriter(...) io.Writer {
    return &LineWriter{
        startTime: time.Now().UTC(),    // Step 启动时刻（Agent 本地时钟）
        // ...
    }
}

func (w *LineWriter) Write(p []byte) (n int, err error) {
    line := &rpc.LogEntry{
        Time: int64(time.Since(w.startTime).Seconds()),  // 0, 1, 2, ... 秒
        Line: w.num,
        Data: []byte(data),
    }
    w.num++
    w.peer.EnqueueLog(line)
}
```

**前端展示**（`PipelineLog.vue:342-344`）：

```typescript
function formatTime(time?: number): string {
  return time === undefined ? '' : `${time}s`;   // 直接显示 "5s"、"120s"
}
```

这带来的特性：
- **日志行时间戳不受 Agent 与 Server 时钟差异影响** — 它本质是 stopwatch 测量值
- **同一 Step 内的日志时间间隔精确** — `time.Since()` 单调递增
- **跨 Step 日志时间不可直接比较** — 每个 Step 有独立的 startTime 基点
- **时钟漂移不在此处发生** — 唯一的风险是 `time.Now()` 被 NTP 校时回拨，
  可能导致后续 `LogEntry.Time` 小于前面的（出现负数）

### 15.3 Step/Workflow/Pipeline 级绝对时间戳：Agent 主导，Server 兜底

绝对时间戳使用 Unix 秒级 UTC，遵循以下优先级：

#### Step 时间戳（`server/pipeline/step_status.go`）：

```go
// Agent 上报的 state.Started 优先
step.Started = state.Started

// Agent 上报为 0 时（setup 错误未进入 running 等情况），Server 本地兜底
if step.Started == 0 {
    step.Started = time.Now().Unix()
}

// Finished 同理
step.Finished = state.Finished
if step.Finished == 0 {
    step.Finished = time.Now().Unix()
}
```

#### Workflow 时间戳（`server/pipeline/workflow_status.go`）：

```go
// UpdateWorkflowStatusToRunning:
workflow.Started = state.Started    // 直接用 Agent 上报值

// UpdateWorkflowStatusToDone:
workflow.Finished = state.Finished  // 直接用 Agent 上报值
if state.Started == 0 {
    workflow.State = Skipped        // Started=0 视为从未执行 → skipped
}
```

#### Pipeline 时间戳（`server/pipeline/pipeline_status.go:28` 的 `PipelineStatus`
中不直接修改 Started/Finished，看 `UpdateStatusToDone`）：

```go
// UpdateToStatusRunning:
pipeline.Started = startedAt        // 来自首个 Workflow 的 Init RPC

// UpdateStatusToDone:
pipeline.Finished = finishedAt      // 来自最后完成的 Workflow Done RPC
pipeline.Updated = time.Now().Unix() // Server 本地时间，记录本次更新时间
```

#### 接收 webhook 时：

```go
// Pipeline.Created = time.Now().Unix()    // Server 接收到 webhook 的时刻
// Pipeline.Timestamp = forge webhook 上报的 commit 时间（来自 Git forge）
```

### 15.4 跨节点时钟漂移的实际影响

| 指标 | 影响评估 | 说明 |
|------|----------|------|
| Step 持续时长计算 | ✅ 精确 | `finished - started`，两个值都来自同一 Agent 时钟，漂移抵消 |
| Workflow 持续时长 | ✅ 精确 | 同上，两个值都来自同一 Agent |
| Pipeline 持续时长 | ⚠️ 轻微偏差 | `Started` 来自首个 Workflow Agent，`Finished` 来自最后一个 Workflow Agent，若在不同机器上，可能有毫秒级误差 |
| Pipeline.Created 到 Started 的等待时间 | ⚠️ 偏差明显 | `Created` 用 Server 时钟，`Started` 用 Agent 时钟，两台机器 NTP 不同步可能有 ±5s 误差 |
| 前端 since 显示 (timeAgo) | ⚠️ 三重偏差 | `Date.now()` (Browser) - `Created * 1000` (Server) + 浏览器时区转换，最大误差来源 |
| Cron 定时触发精度 | ⚠️ Server 时钟依赖 | Cron ticker 用 Server 本地 `time.Now()` 与 Cron 表达式匹配 |
| Forge commit status 超时判定 | ⚠️ 平台间依赖 | GitHub/GitLab 用自己的时钟，与 Server 偏差过大可能显示 outdated |

### 15.5 前端 duration 计算：已完成用差值，运行中用 Browser Date.now()

`usePipeline.ts:44-77` 的 `durationRaw` computed 是关键对账点：

```typescript
const durationRaw = computed(() => {
  const start = pipeline.value.started || 0;
  const end   = pipeline.value.finished || pipeline.value.updated || 0;

  if (start === 0 || end === 0) { return 0; }

  if (pipeline.value.status === 'running') {
    // 运行中：Browser 时钟做终点（实时更新）
    return Date.now() - start * 1000;
  }
  // 已完成：Server/Agent 提供的终点（差值恒定）
  return (end - start) * 1000;
});
```

**运行中 Pipeline 的时钟漂移暴露**：假设 Server 比 Browser 慢 10s，
`start * 1000` (Server) 比 Browser 的对应时刻小 10000ms，
`Date.now() - start * 1000` 就会**多出 10 秒**。用户看到的 Pipeline 持续时间
比实际多 10 秒。但由于 Step 级 duration 用 Agent 时钟差值，两者可能不匹配：

```
Pipeline 总时长（Browser 计时）：3m 10s
  ├── Step A 耗时（Agent 差值）：1m 00s
  └── Step B 耗时（Agent 差值）：2m 00s
合计应是 3m 00s，但 Pipeline 显示 3m 10s
```

这种 10-20s 级别的不一致是**正常现象**，用户一般不会察觉。

### 15.6 无 NTP 对账：Woodpecker 的设计假设

代码中**完全没有**以下机制：

| 对账机制 | 是否存在 | 说明 |
|----------|----------|------|
| Agent 注册时上报时钟偏移 | ❌ 不存在 | 没有 `timeOffset` 字段或 NTP 协商 |
| Server 向 Agent 校时 | ❌ 不存在 | gRPC 协议没有 `GetCurrentTime` / `AdjustClock` 方法 |
| 前端接收 Server 时间同步 | ❌ 不存在 | 没有 `X-Server-Time` HTTP 头或 `/api/time` 端点 |
| `Pipeline.Updated` 与 Agent 时钟差值检测 | ❌ 不存在 | 不检查 `Agent - Server` 时间差是否超过阈值 |
| Monotonic clock 使用 | ⚠️ 部分 | Go 中 `time.Since()` 自动使用 monotonic clock（Go 1.9+），防 NTP 回拨；但 JS 端 `Date.now()` 不是单调的 |

**设计假设**：运维环境中 Server 和所有 Agent 通过 NTP 保持时钟同步（±1s 内），
这是分布式 CI 系统的通用前置条件。Woodpecker 通过以下方式降低漂移影响：
1. **日志时间戳用相对秒数**（Section 15.2）— 与绝对时钟无关
2. **持续时长用差值**（`finished - started`）— 同节点时钟漂移抵消
3. **Server 端兜底** — Agent 未上报时用 Server 本地时间填充
4. **前端显示容忍** — `timeAgo` 显示的是模糊相对时间（"5 分钟前"），
   漂移 ±1 分钟不改变显示文本

### 15.7 `Timestamp` 字段的特殊地位

`Pipeline.Timestamp`（`model/pipeline.go:46`）是来自 Git Forge（GitHub/GitLab 等）
的 **commit 本身的时间戳**，与 Server/Agent 时钟无关。它通常用于：
- 按"提交实际时间"排序 Pipeline（而非创建时间）
- Cron 场景下对比 commit 新旧

这个字段完全不参与 drift 对账，因为它是 Git 自带的权威时间。

### 15.8 状态图：多节点时间戳如何协作完成一次 Build

```
T0 (Server):  收到 webhook → Pipeline.Created = Server time.Now().Unix()
              入队等待...

T1 (Agent):   Poll 获取任务 → Runtime.started = Agent time.Now().Unix()
              traceStep(start) → StepState.Started = T1
                                  WorkflowState.Started = T1
              gRPC Init() → Server: pipeline.Started = T1   （Agent 时间入库）
                                 UpdateWorkflowStatusToRunning: workflow.Started = T1

T1 + Δt (Agent): LineWriter 建立 → startTime = Agent time.Now().UTC()
                 命令执行 → LogEntry.Time = 0, 1, 2, ... （相对秒）

T2 (Agent):   Step 完成 → StepState.Finished = Agent time.Now().Unix()
              gRPC Update(step_state) → Server
                 step.Finished = T2   (差值 T2-T1 = Step 持续时长，精确)

T3 (Agent):   Workflow 完成 → WorkflowState.Finished = T3
              gRPC Done() → Server
                 workflow.Finished = T3
                 pipeline.Finished = T3   (若这是最后一个 Workflow)
                 pipeline.Updated = Server time.Now()   (记录本次更新时间)

前端显示（Browser 本地时钟 TB）：
  since (Created 距今):       TB - T0
  duration (running):         TB - T1
  duration (finished):        T3 - T1
  单 Step 耗时:               T2 - T1 (差值，无漂移)
  日志行时间戳:               Δ (相对秒，无漂移)
  创建时间本地化:             new Date(T0 * 1000) → toLocaleString()
```

**最容易出现用户感知不一致的场景**：
- Server 时区与用户浏览器不同 → `toLocaleString()` 显示的时间戳带不同时区
  （代码未强制 UTC 显示）
- 用户手动修改本地时钟 → running 计时器跳变
- Agent 与 Server 严重不同步（> 30s）→ `since` (T0:Server) 与 `duration` (T1:Agent)
  的基准不一致，可能出现"等待了 2 分钟，执行 1 分钟，但 since 显示 1 分钟"
  的反直觉现象

---

## 16. Build 列表分页加载性能策略

### 16.1 两套分页组件：`RepoPipelines.vue` vs `RepoBranches.vue`

Build 列表有两条独立的分页实现路径，分别服务不同的视图：

| 视图 | 分页实现 | 代码位置 |
|------|----------|----------|
| Repo 活动页（`RepoPipelines.vue`） | **手动点击 "Load More"** + Pinia Store 增量累加 | `pipelines.ts:85-96` |
| Repo 分支页（`RepoBranches.vue`）、PR 列表、Cron 列表等 | **`usePagination` composition** + 无限滚动 | `usePaginate.ts:24-96` |

TODO(4626) 标记了重构计划（`RepoPipelines.vue:19-20`），目标是统一为
`usePagination` + 服务端过滤，但目前仍各自为政。

### 16.2 RepoPipelines：Pinia Store 全量累加 + 手动加载

这是主列表的实现，核心在 `store/pipelines.ts:85-96`：

```typescript
const perPage = 50;
const hasMore = ref(false);

async function loadRepoPipelines(repoId: number, page?: number) {
  loading.value = true;
  const _pipelines = await apiClient.getPipelineList(repoId, { page, perPage });
  _pipelines.forEach((pipeline) => {
    setPipeline(repoId, pipeline);       // Map.set 去重合并
  });
  hasMore.value = _pipelines.length >= perPage;
  loading.value = false;
}
```

**前端 API 调用** (`web/src/lib/api/index.ts:113-119`)：

```typescript
getPipelineList(repoId, opts?: {
  page?: number;
  perPage?: number;
  before?: string;      // RFC3339 日期
  after?: string;       // RFC3339 日期
  ref?: string;         // ref 包含的字符串
  branch?: string;
  events?: string;      // 逗号分隔的事件列表
}): Promise<Pipeline[]>;
```

**关键行为**：
- `perPage = 50` 写死，无配置项
- 返回数组长度 === 50 → `hasMore = true`，否则为 `false`
- 每次加载 `forEach` 调用 `setPipeline()`，后者通过 `Map.set(pipeline.number)`
  做**去重合并**（同 pipeline.number 保留新数据）
- 数据存储在 `pipelines: Map<repoId, Map<number, Pipeline>>`，即 repo → 序号 → Pipeline

**前端渲染层** (`RepoPipelines.vue:25-30`):

```typescript
const page = ref(1);

async function loadMore() {
  page.value += 1;
  await pipelineStore.loadRepoPipelines(repo.value.id, page.value);
}
```

点击 "Load More" → `page++` → 拉取下一页 → 合并到 Store → 触发响应式更新。

**Store 数据在组件间共享**：`RepoWrapper.vue:98` 通过 `provide('pipelines', pipelines)`
注入给所有子组件，`pipelines` 是 `getRepoPipelines(repositoryId)` 返回的 computed：

```typescript
// store/pipelines.ts:56-58
function getRepoPipelines(repoId: Ref<number>) {
  return computed(() =>
    [...(pipelines.get(repoId.value)?.values() ?? [])]
      .sort(comparePipelines)  // 按 created 降序
  );
}
```

**排序是每次访问时全量排序**，无缓存。对 1000 条数据，比较函数执行 ~5000 次，
性能可接受。

### 16.3 服务端分页：数据库索引 + 按 number 排序

`server/api/pipeline.go:129-189` 的 `GetPipelines` handler：

```go
func GetPipelines(c *gin.Context) {
    repo := session.Repo(c)
    filter := &model.PipelineFilter{
        Branch:      c.Query("branch"),
        RefContains: c.Query("ref"),
    }

    // 解析 before/after RFC3339 → Unix 秒
    if before := c.Query("before"); before != "" {
        filter.Before, _ = time.Parse(time.RFC3339, before).Unix()
    }
    if after := c.Query("after"); after != "" {
        filter.After, _ = time.Parse(time.RFC3339, after).Unix()
    }
    if events := c.Query("event"); events != "" {
        filter.Events = strings.Split(events, ",")  // 验证每个 event 值
    }
    if status := c.Query("status"); status != "" {
        filter.Status = status
    }

    pipelines, _ := store.GetPipelineList(repo, session.Pagination(c), filter)
    c.JSON(http.StatusOK, pipelines)
}
```

**数据库层** (`server/store/datastore/pipeline.go:68-102`)：

```go
func (s storage) GetPipelineList(repo *model.Repo, p *model.ListOptions, f *model.PipelineFilter) ([]*model.Pipeline, error) {
    cond := builder.NewCond().And(builder.Eq{"repo_id": repo.ID})

    if f != nil {
        if f.After != 0  { cond = cond.And(builder.Gt{"created": f.After}) }
        if f.Before != 0 { cond = cond.And(builder.Lt{"created": f.Before}) }
        if f.Branch != "" { cond = cond.And(builder.Eq{"branch": f.Branch}) }
        if f.Status != "" { cond = cond.And(builder.Eq{"status": f.Status}) }
        if len(f.Events) != 0 { cond = cond.And(builder.In("event", f.Events)) }
        if f.RefContains != "" { cond = cond.And(builder.Like{"ref", f.RefContains}) }
    }

    return pipelines, s.paginate(p).Where(cond).
        Desc("number").      // 按 pipeline number 降序（即新建在前）
        Find(&pipelines)
}
```

**排序键是 `number`（自增序号）**，不是 `created`。因为 number 是数据库自增字段，
有索引 (`UNIQUE(repo_id, number)`)，查询性能比 `created` 更好。

**分页使用 offset/limit**（`s.paginate(p)` 实现），不是 cursor 分页。
大页数（> 1000 条）会有性能衰减。

### 16.4 usePagination：无限滚动 + 支持"each"级联加载

`usePaginate.ts:24-96` 的 `usePagination` composition 是更通用的分页实现，
支持**无限滚动自动加载**，还支持 `each` 数组参数的级联加载（如分页拉取 A 分支 →
拉完自动切换 B 分支继续拉）：

```typescript
export function usePagination<T, S = unknown>(
  _loadData: (page: number, arg: S) => Promise<T[] | null>,
  isActive: () => boolean = () => true,
  {
    scrollElement: _scrollElement,  // 滚动容器，默认 #scroll-component
    each: _each,                     // 级联参数数组
    pageSize: _pageSize,             // 默认 50
  }: { ... } = {},
) {
  // 内部状态
  const page = ref(1);
  const hasMore = ref(true);
  const data = ref<T[]>([]);
  const loading = ref(false);
  const each = ref([...(_each ?? [])]);

  async function loadData() {
    if (loading.value || !hasMore.value) return;

    loading.value = true;
    const newData = await _loadData(page.value, each.value?.[0]) ?? [];
    hasMore.value = newData.length >= pageSize.value && newData.length > 0;

    if (newData.length > 0) {
      data.value.push(...newData);
    }

    // 当前 each 元素拉完，切换到下一个
    if (!hasMore.value && each.value.length > 0) {
      each.value.shift();
      page.value = 1;
      hasMore.value = each.value.length > 0;
      if (hasMore.value) {
        loading.value = false;
        await loadData();  // 递归拉取下一个
      }
    }
    loading.value = false;
  }

  if (scrollElement !== null) {
    useInfiniteScroll(scrollElement, nextPage, { distance: 10 });
  }
}
```

**无限滚动触发**：使用 `@vueuse/core` 的 `useInfiniteScroll`，
滚动到底部 `10px` 内自动调用 `nextPage()` → `page++` → `loadData()`。

**每个视图按需使用**：
- `RepoBranches.vue:42`：`usePagination(loadBranches)`，无 each 参数
- `RepoPullRequests.vue`：类似

**性能特点**：
- 无限滚动无 DOM 虚拟化，500 条数据以上有滚动卡顿
- `each` 级联是同步循环拉取，可能产生连续多次网络请求
- `loadData` 有 `loading` 标志防重入

### 16.5 分页性能瓶颈分析

| 瓶颈 | 影响 | 代码位置 |
|------|------|----------|
| 前端每次访问 `pipelines` computed 都全量 sort | O(n log n)，n > 2000 时感知明显 | `store/pipelines.ts:57` |
| 无限滚动无虚拟列表，DOM 随数据量线性增长 | 500 条 Pipeline → 约 5000+ DOM 节点 | `PipelineList.vue:3-11` + `PipelineItem.vue` |
| 服务端 offset/limit 分页，大 offset 全表扫描 | page > 20（> 1000 条）时查询变慢 | `datastore/pipeline.go:99` |
| `RepoPipelines` 每次 "Load More" 50 条，500 条需 10 次点击 | UX 较差，无无限滚动 | `RepoPipelines.vue:27-30` |
| `pipelineFeed` computed 每次访问跨 repo 合并 + 全量 sort | 多 repo 场景性能开销大 | `store/pipelines.ts:105-117` |

**`pipelineFeed` 跨 repo 聚合** (`store/pipelines.ts:105-117`) 是性能最吃紧的点：

```typescript
const pipelineFeed = computed(() =>
  [...pipelines.entries()]                            // Map<repoId, Map<number, Pipeline>>
    .reduce<PipelineFeed[]>((acc, [_repoId, repoPipelines]) => {
      const repoPipelinesArray = Array.from(repoPipelines.entries(),
        ([_pipelineNumber, pipeline]) => ({
          ...pipeline, repo_id: _repoId, number: _pipelineNumber,
        })
      );
      return [...acc, ...repoPipelinesArray];         // 展平为一维数组
    }, [])
    .sort(comparePipelinesWithStatus)                 // 全局排序：优先级 + created
    .filter(/* owned repos only */)
);
```

每次访问（包括 SSE 推送更新触发响应式）都执行：
1. 遍历所有 repo 的所有 Pipeline，创建新数组（展开）
2. 全量排序（running 优先 + created 降序）
3. 过滤不属于当前用户的 repo

用户有 10 个 repo，每个 100 条 Pipeline → 每次更新遍历 1000 条 + 1000 log n 次比较。

---

## 17. 过滤器持久化：URL 查询参数驱动，无本地存储

### 17.1 过滤体系：服务端过滤 vs 客户端过滤

Build 列表的过滤分两层：

| 过滤层 | 实现方式 | 持久化方式 |
|--------|----------|------------|
| **服务端过滤** | `GetPipelineList` 支持 `branch` / `status` / `event` / `before` / `after` / `ref` 等查询参数 | URL query string |
| **客户端过滤** | `RepoBranch.vue:24-31` 对已加载数据做 `filter()` | 路由参数（`:branch`） |

**关键发现**：用户保存的筛选条件（Saved Filters / Presets）**在代码中完全不存在**。
没有本地存储、没有 Pinia store、没有服务端数据库表，过滤仅由 URL 驱动。

### 17.2 RepoBranch：客户端分支过滤

`RepoBranch.vue` 是最典型的客户端过滤场景：

```typescript
// RepoBranch.vue:20-31
const props = defineProps<{ branch: string }>();
const branch = toRef(props, 'branch');

const allPipelines = requiredInject('pipelines');  // Store 全量数据
const pipelines = computed(() =>
  allPipelines.value.filter(
    (b) =>
      b.branch === branch.value &&
      b.event !== 'pull_request' &&
      b.event !== 'pull_request_closed' &&
      b.event !== 'pull_request_metadata',
  ),
);
```

- 分支名来自**路由参数** `:branch`（URL `/repos/{id}/branch/{branch}`）
- 过滤范围是 `pipelines` Store 中**已加载**的数据
- 切换分支时不触发网络请求（假设数据已加载）
- 如果数据未加载（用户直接访问分支 URL），走 `RepoWrapper.loadRepo()` 的
  `loadRepoPipelines(repoId, 1)`，只拉第一页（50 条），可能显示不全

**过滤与分页的交互缺陷**：如果某分支的 Pipeline 分散在多个分页页中，
只拉了第一页的用户可能看不到该分支的历史 Pipeline（即使存在）。

### 17.3 PR 列表：类似机制

`RepoPullRequest.vue` 采用相同模式，但通过 PR 索引页拉取：

```typescript
// RepoPullRequests.vue 中调用
apiClient.getRepoPullRequests(repoId, { page, perPage })
```

这是一个**独立的 API 端点**（`/api/repos/{id}/pull_requests`），服务端直接按
`event in (pull_request, pull_request_closed, pull_request_metadata)` 过滤，
不需要客户端过滤。

### 17.4 服务端过滤：API 参数与数据库查询对应

服务端 `GetPipelineList` 支持的过滤参数：

| URL 参数 | 数据库条件 | 类型 |
|----------|------------|------|
| `branch=main` | `branch = 'main'` | 等值 |
| `status=success` | `status = 'success'` | 等值 |
| `event=push,pull_request` | `event IN ('push','pull_request')` | 集合 |
| `ref=release/v1.0` | `ref LIKE '%release/v1.0%'` | 模糊 |
| `before=2025-06-20T00:00:00Z` | `created < 1750406400` | 范围（Unix 秒） |
| `after=2025-06-19T00:00:00Z` | `created > 1750320000` | 范围 |
| `page=3&perPage=25` | `LIMIT 25 OFFSET 50` | 分页 |

**前端未直接暴露 UI 控件**让用户输入这些参数（除了分页）。
目前 UI 上没有状态筛选下拉框、事件筛选 checkbox、日期范围选择器。
用户需要手动构造 URL 才能使用这些服务端过滤能力。

### 17.5 加载顺序与数据竞态

`RepoWrapper.vue` 的 `loadRepo()` 函数是数据加载入口：

```
onMounted / watch(repositoryId) → loadRepo():
  ├─ getRepoPermissions()           // 权限校验
  ├─ repoStore.loadRepo()           // 拉取 Repo 元数据
  ├─ pipelineStore.loadRepoPipelines(repoId)  // 拉取 page=1 的 Pipeline
  ├─ forgeStore.getForge()
  └─ updateLastAccess()
```

**执行顺序是串行**的：权限 → Repo → Pipelines。Pipeline 数据的加载
依赖前面的权限校验和 Repo 加载。

**数据竞态分析**：
- `provide('pipelines', pipelines)` 是同步执行的（`RepoWrapper.vue:101`），
  在 `onMounted` 之前执行
- `pipelines` 是 computed，初始值为空数组
- `onMounted` 触发 `loadRepo()` → 异步 `loadRepoPipelines()` → 数据填充 →
  computed 重新计算 → 子组件（`RepoPipelines.vue`）自动刷新

因此子组件看到的 `pipelines` 是响应式的，不会出现 undefined。

### 17.6 过滤器持久化现状总结

| 功能 | 状态 | 代码线索 |
|------|------|----------|
| 分支过滤 | ✅ 基于路由参数 | `RepoBranch.vue:24-31` |
| PR 事件过滤 | ✅ 独立 API 端点 | `/api/repos/{id}/pull_requests` |
| 状态筛选 UI | ❌ 不存在 | API 支持但无 UI |
| 事件多选 UI | ❌ 不存在 | API 支持但无 UI |
| 日期范围选择 | ❌ 不存在 | API 支持但无 UI |
| 保存筛选条件（"我的筛选"） | ❌ 不存在 | 无相关代码 |
| 筛选条件持久化（localStorage） | ❌ 不存在 | 无 `useStorage` 调用于 Pipeline 过滤 |
| 筛选条件分享（URL 复制） | ⚠️ 需手动构造 | API 参数可用但无 UI 生成 |

**当前唯一的"持久化"机制**是 Vue Router 的 `$route.query` / `$route.params`，
用户手动在 URL 中添加查询参数（如 `?status=success&branch=main`）时，可以分享链接。
但代码中没有任何 `watch(route.query)` 自动同步到 API 调用的逻辑。

TODO(4626) 的重构目标之一就是 "server-side filtering"，即让 URL query 参数
自动驱动 `getPipelineList` 调用。

---

## 18. Build 历史趋势图表：（几乎）完全不存在的聚合层

### 18.1 现状：没有图表组件，只有 badge 状态图标

代码搜索 `chart` / `graph` / `trend` / `sparkline` / `metrics` 等关键词，
仅在 `tailwind.css` 的 `font-family: graphik` 中出现了 "graphik" 字符串，
以及 `useRepoSearch.ts` 的一些无关引用。**没有任何 SVG chart、折线图、
柱状图、饼图等可视化组件**。

UI 中显示 Pipeline 状态的方式：
- 列表中每条 Pipeline 左侧的 `PipelineStatusIcon` 图标
- 徽章 SVG：`/api/badges/{repoId}/status.svg`
- 分支列表中的 `Badge` 组件（标记 default 分支）
- Repo 卡片右下角的最新状态图标

### 18.2 服务端没有预聚合接口

检查服务端 API 端点（`server/api/`），没有找到任何聚合类接口：
- ❌ `GET /api/repos/{id}/pipelines/aggregate`
- ❌ `GET /api/repos/{id}/metrics`
- ❌ `GET /api/repos/{id}/builds/stats`
- ❌ `GET /api/user/metrics`

唯一的计数 API 是 `GetPipelineCount()` (`datastore/pipeline.go:130-132`)，
用于数据库总 Pipeline 数统计（内部管理用），不暴露给前端。

```go
func (s storage) GetPipelineCount() (int64, error) {
    return s.engine.Count(new(model.Pipeline))
}
```

### 18.3 前端也没有聚合计算

检查前端 `store/pipelines.ts`，没有聚合 computed：
- ❌ `successRate = computed(...)`
- ❌ `dailyBuilds = computed(...)`
- ❌ `statusCounts = computed(...)`
- ❌ `avgDuration = computed(...)`

唯一的聚合类逻辑是 `pipelineFeed` 和 `activePipelines` computed，
它们做的是**跨 repo 合并和过滤**，而非统计聚合：

```typescript
// store/pipelines.ts:119-121
const activePipelines = computed(() =>
  pipelineFeed.value.filter(
    (pipeline) => ['pending', 'running', 'started'].includes(pipeline.status)
  )
);
```

这只是 `filter()`，不做 `reduce()` 统计。

### 18.4 唯一的"趋势"信息：Repo 卡片的 last pipeline

Repo 列表卡片（`RepoList.vue` 或类似组件）展示每个 Repo 的最后一次 Pipeline 状态，
数据来源是 `Repo.last_pipeline` 字段：

```typescript
// store/pipelines.ts:47-51
// setPipeline 中顺带更新 Repo.last_pipeline_number
const repo = repoStore.repos.get(repoId);
if (repo?.last_pipeline_number < pipeline.number) {
    repo.last_pipeline_number = pipeline.number;
    repoStore.setRepo(repo);
}
```

这是**单条最新状态**，不是历史趋势。

### 18.5 如果要实现：前端聚合的可行性分析

假设未来要添加趋势图表，前端现有数据结构可以支持：

```typescript
// 假设 N 天历史聚合（如果拉取了足够数据）
const dailyStats = computed(() => {
  const byDay = new Map<string, { success: number; failure: number; running: number; }>();

  for (const p of pipelines.value) {
    const day = new Date(p.created * 1000).toISOString().slice(0, 10);  // YYYY-MM-DD
    const stats = byDay.get(day) ?? { success: 0, failure: 0, running: 0 };
    stats[p.status as keyof typeof stats]++;
    byDay.set(day, stats);
  }

  return [...byDay.entries()].sort();  // 按日期升序
});
```

**但限制明显**：
1. 最多只能聚合 Store 中已加载的数据（默认只拉了第一页 50 条）
2. 要聚合 90 天历史，可能需要拉取上千条 Pipeline，内存和网络开销大
3. 聚合在每次 `setPipeline()` 触发响应式更新时全量重算

**更合理的实现路径**（服务端预聚合）：

```go
// 假设的聚合 API
type PipelineAggregation struct {
    Date     string `json:"date"`
    Success  int    `json:"success"`
    Failure  int    `json:"failure"`
    Canceled int    `json:"canceled"`
    Running  int    `json:"running"`
    AvgDuration int64 `json:"avg_duration"`
}

func GetPipelineAggregation(c *gin.Context) {
    repo := session.Repo(c)
    // SQL: SELECT DATE(FROM_UNIXTIME(created)) as date, status, COUNT(*), AVG(finished-started)
    //      FROM pipelines WHERE repo_id = ? AND created > ? GROUP BY date, status
}
```

### 18.6 聚合相关 TODO 和设计意图

`store/pipelines.ts` 的 `perPage = 50` + `hasMore` 设计意味着
Woodpecker 假设用户浏览 Pipeline 是**从新到旧、按需加载**，而非一次性
获取全量做聚合分析。Pipeline 列表的定位是**流水式事件流**，不是**数据仓库**。

如果要添加趋势图，需要：
1. 后端新增聚合 API（可能用单独的统计表或即时 SQL 聚合）
2. 前端新增图表组件（ECharts / Chart.js 或自定义 SVG）
3. 在 Repo 设置或独立视图中展示

当前没有相关 issue 或 TODO 标注（代码中未找到），因此这是**非核心功能**。

### 18.7 相关但不相同：Forge commit status

唯一接近"聚合"的概念是 `updatePipelineStatus()` 将 Pipeline 状态
同步到 Git forge（GitHub / GitLab 等）的 commit status。这是**外部系统**的
状态展示，不是 Woodpecker 自身的趋势图。`updatePipelineStatus()` 只同步
**最新状态**，不同步历史数据。

---

## 18. 关键未实现功能小结

将 16-18 节发现的缺失功能汇总：

| 功能 | 服务端能力 | 前端 UI | 持久化 |
|------|----------|----------|--------|
| 状态筛选下拉框 | ✅ API 支持 | ❌ 不存在 | - |
| 事件多选过滤 | ✅ API 支持 | ❌ 不存在 | - |
| 日期范围筛选 | ✅ API 支持 | ❌ 不存在 | - |
| 保存筛选条件（预设） | - | ❌ 不存在 | ❌ |
| URL query 自动驱动过滤 | ✅ API 支持 | ⚠️ TODO(4626) | ✅ URL 即持久化 |
| 无限滚动（RepoPipelines） | ✅ API 支持 | ❌ 手动 Load More | - |
| 虚拟滚动/分页性能 | - | ❌ 无虚拟化 | - |
| Build 成功率趋势图 | ❌ 无聚合 API | ❌ 无图表组件 | - |
| Build 时长趋势图 | ❌ 无聚合 API | ❌ 无图表组件 | - |
| 状态分布饼图 | ❌ 无聚合 API | ❌ 无图表组件 | - |

这些未实现功能解释了为什么之前的分析中"感觉有东西没追到"——这些功能本身不存在。
