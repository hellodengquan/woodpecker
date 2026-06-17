# Webhook 调度全链路解析

> 基于 Woodpecker CI (v3) 源码，梳理从代码托管平台推送 push / PR / tag 事件到流水线执行完毕的完整代码走向，
> 聚焦签名校验、并发拦截、死信回收等关键节点。

---

## 1 全局概览

```
┌──────────────────────────────────────────────────────────────────────────┐
│  代码托管平台 (GitHub / GitLab / Gitea / Forgejo / Bitbucket / BDC)     │
│    │ push / PR / tag / release webhook                                   │
└────┬─────────────────────────────────────────────────────────────────────┘
     │ HTTP POST /api/hook
     ▼
┌─────────────────────────────────────┐
│  PostHook (server/api/hook.go:69)   │  ← JWT HookToken 签名校验
│    ① token.ParseRequest → repo      │
│    ② ForgeFromRepo → forge 驱动     │
│    ③ forge.Hook → 解析载荷          │
│    ④ repo 匹配 & 活跃校验           │
│    ⑤ pipeline.Create → 创建流水线   │
└────────────┬────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│  pipeline.Create (server/pipeline/create.go:35)                  │
│    ① skip-ci 检查                                                │
│    ② forge.Refresh → OAuth token 刷新 (singleflight 去重)        │
│    ③ ConfigService.Fetch → 拉取 .woodpecker.yaml                │
│    ④ parsePipeline → 编译 YAML → builder.Item                   │
│    ⑤ setApprovalState → 门控拦截                                 │
│    ⑥ start → cancelPreviousPipelines + queuePipeline            │
└────────────┬─────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│  FIFO 队列 (server/queue/fifo.go)                               │
│    pending → waitingOnDeps → running                             │
│    process() 每 100ms 轮询:                                      │
│      · resubmitExpiredPipelines → 过期任务重新入队               │
│      · filterWaiting → 依赖过滤                                  │
│      · assignToWorker → 匹配 Agent                               │
└────────────┬─────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────┐
│  Agent 通过 gRPC Poll 拉取 Task 并执行        │
│  完成后 Done / Error → 更新 DB & PubSub       │
└──────────────────────────────────────────────┘
```

---

## 2 签名校验（Signature Verification）

Woodpecker 采用 **两层校验**：入口层 JWT HookToken + Forge 层可选的 Payload 签名。

### 2.1 入口层：JWT HookToken 校验

**入口**：`server/api/hook.go:78`

```go
_, err := token.ParseRequest([]token.Type{token.HookToken}, c.Request, func(t *token.Token) (string, error) {
    repo, err = getRepoFromToken(_store, t)
    return repo.Hash, nil  // repo.Hash 即签名密钥
})
```

**流程**：

1. 从 HTTP 请求中提取 token，按优先级依次尝试：
   - `Authorization: Bearer <token>` 头
   - `X-Gitlab-Token` 头（GitLab 专用）
   - `access_token` URL 查询参数
   - `user_sess` Cookie
2. 使用 `jwt.Parse()` 解析，在 `keyFunc` 中：
   - 校验算法必须为 `HS256`（`shared/token/token.go:168`），防止算法混淆攻击
   - 从 token claims 提取 `type` 字段，必须为 `HookToken`
   - 通过回调 `SecretFunc` 获取 `repo.Hash` 作为 HMAC 密钥验证签名
3. 从 token claims 中提取 `forge-id` + `repo-forge-remote-id`（或兼容的 `repo-id`），在数据库中定位仓库

**安全设计要点**：

| 要点 | 位置 | 说明 |
|------|------|------|
| 算法锁定 | `token.go:168` | `t.Method.Alg() != SignerAlgo` 直接拒绝，防止 `alg: none` 攻击 |
| Token 类型白名单 | `token.go:59` | `slices.Contains(allowedTypes, token.Type)`，只允许 `HookToken` |
| 密钥绑定仓库 | `hook.go:79-85` | 每个仓库有独立 Hash，签名密钥互不干扰 |
| 仓库 ID 双重验证 | `hook.go:144` | token 中的 `ForgeRemoteID` 必须与 forge 返回的匹配 |

### 2.2 Forge 层：Payload 签名校验

各 Forge 驱动的 `Hook()` 方法可自行做载荷签名验证：

| Forge | 签名校验方式 | 代码位置 |
|-------|-------------|---------|
| **Bitbucket DataCenter** | `bitbucket.ValidateSignature(r, hook.Payload, []byte(repo.Hash))` | `bitbucketdatacenter.go:510` |
| **GitLab** | 通过 `X-Gitlab-Token` 头传递的 token（在入口层 `ParseRequest` 中已解析为 JWT） | `token.go:81-83` |
| **GitHub** | 依赖入口层 JWT 校验；`github.ParseWebHook` 使用官方 SDK 解析 | `github/parse.go:71` |
| **Gitea / Forgejo** | 依赖入口层 JWT 校验；Webhook URL 中嵌入 `access_token` 参数 | `gitea/gitea.go:370-374` |
| **Bitbucket Cloud** | 依赖入口层 JWT 校验 | `bitbucket/bitbucket.go` |

**Bitbucket DataCenter 的签名验证**是 Forge 层最完整的实现：
- 先用 `ParsePayloadWithoutSignature` 解析载荷（`parse.go:35`）
- 再用 `ValidateSignature` 对原始 payload + `repo.Hash` 做 HMAC 校验
- 签名不匹配直接返回错误，拒绝处理

---

## 3 载荷解析与仓库匹配

### 3.1 Forge 驱动分发

`PostHook` 通过 `server.Config.Services.Manager.ForgeFromRepo(repo)` 获取 forge 实例（`hook.go:101`），然后调用 `forge.Hook(ctx, c.Request)` 解析载荷。

### 3.2 各平台载荷解析一览

| Forge | 事件类型 | 解析入口 | 关键代码 |
|-------|---------|---------|---------|
| GitHub | push / deployment / pull_request / release | `github/parse.go:59 parseHook()` | `github.ParseWebHook(github.WebHookType(r), raw)` |
| GitLab | push / tag / merge_request / release | `gitlab/gitlab.go:624 Hook()` | `gitlab.ParseWebhook(eventType, payload)` |
| Gitea | push / create / pull_request / release | `gitea/parse.go:75 parseHook()` | 根据 `X-Gitea-Event` 头分发 |
| Forgejo | push / create / pull_request / release | `forgejo/parse.go:73 parseHook()` | 根据 `X-Forgejo-Event` 头分发 |
| Bitbucket Cloud | repo:push / pullrequest:* | `bitbucket/parse.go:40 parseHook()` | 根据 `X-Event-Key` 头分发 |
| Bitbucket DC | repo:ref_changed / pr:* | `bitbucketdatacenter/parse.go:34 parseHook()` | `bitbucket.ParsePayloadWithoutSignature(r)` |

**事件到 Pipeline 字段的映射**（以 GitHub 为例，`github/parse.go`）：

- **Push** → `EventPush`，提取 commit SHA、ref、branch、message
- **Tag** → 通过 push 事件中 `refs/tags/` 前缀识别，修正为 `EventTag`
- **Pull Request** → 根据 action 映射为 `EventPull` / `EventPullClosed` / `EventPullMetadata`
- **Release** → `EventRelease`，仅处理 `released` action
- **Deploy** → `EventDeploy`

### 3.3 仓库匹配

载荷解析返回 `repoFromForge` 后，`PostHook` 做二次校验（`hook.go:144`）：

```go
if repo.ForgeRemoteID != repoFromForge.ForgeRemoteID {
    // 拒绝：token 中的仓库与载荷中的仓库不匹配
}
```

随后校验仓库状态：
- `repo.IsActive` 必须为 true（`hook.go:154`）
- `repo.UserID` 必须有值（即仓库有 owner，`hook.go:160`）
- PR 事件需 `repo.AllowPull` 为 true（`hook.go:197`）

### 3.4 忽略事件机制

Forge 驱动可返回 `types.ErrIgnoreEvent` 表示事件应被忽略（非错误），`PostHook` 会返回 200 OK（`hook.go:114-118`）。

---

## 4 流水线编译与投递

### 4.1 pipeline.Create 全流程

**代码**：`server/pipeline/create.go:35`

```
Create()
  │
  ├─ ① 获取 repoUser，刷新 OAuth token (forge.Refresh)
  ├─ ② skip-ci 检查 (constraint.IsSkipCommitMessage)
  ├─ ③ 持久化 pipeline 记录 (store.CreatePipeline)
  ├─ ④ ConfigService.Fetch → 拉取 .woodpecker.yaml
  │     ├─ ErrConfigNotFound → 删除 pipeline, 返回 ErrFiltered
  │     └─ 有 config 但获取出错 → 旧 config 降级兜底
  ├─ ⑤ parsePipeline → 编译 YAML → []*builder.Item
  ├─ ⑥ createPipelineItems → 持久化 workflow & step
  ├─ ⑦ 持久化 config (findOrPersistPipelineConfig + linkPipelineConfigs)
  ├─ ⑧ publishPipeline → 推送状态到 PubSub + Forge
  ├─ ⑨ setApprovalState → 门控检查 (gated)
  │     └─ StatusBlocked → 直接返回, 等待审批
  └─ ⑩ start → 投递队列
```

### 4.2 YAML 编译流水线

`parsePipeline` (`items.go:38`) 核心步骤：

1. **获取 netrc 凭证** → 用于克隆私有仓库
2. **获取 secrets** → 通过 `SecretService.SecretListPipeline`，按事件类型过滤
3. **获取 registry 凭证** → 拉取私有镜像
4. **构建 PipelineBuilder**：
   - 注入环境变量（全局 + 仓库级 + pipeline 级）
   - 注入 secrets、registries、volumes、networks
   - 设置 trusted 权限
   - 设置代理配置
5. `b.Build()` → 编译为 `[]*builder.Item`，每个 Item 包含一个 workflow 的完整配置

### 4.3 投递到队列

`queuePipeline` (`pipeline/queue.go:29`)：

1. 每个 `builder.Item` 转换为 `model.Task`
2. 设置 task labels（repo full name, org ID）
3. 解析 `depends_on` 依赖关系为 task ID 列表
4. 序列化 `rpc.Workflow` 到 `task.Data`
5. 调用 `server.Config.Services.Scheduler.PushAtOnce(ctx, tasks)`

### 4.4 持久化队列

`WithTaskStore` (`queue/persistent.go:32`) 包装内存队列：

- **PushAtOnce**：先写入数据库（`TaskInsert`），再入内存队列；入队失败则从数据库删除
- **Poll**：Agent 拉取后从数据库删除 task
- **Error / ErrorAtOnce**：任务失败后清理数据库记录
- **启动恢复**：构造时从数据库 `TaskList()` 恢复所有未完成任务

---

## 5 并发拦截与去重

### 5.1 单飞刷新（OAuth Token singleflight）

**代码**：`server/forge/refresh.go:43-48`

```go
var refreshGroup singleflight.Group
```

**问题**：多个并发 webhook 同时触发 `forge.Refresh`，可能导致：
- 同一用户的 OAuth token 被并发刷新多次
- Forgejo 等平台使用一次性 refresh token（`InvalidateRefreshTokens=true`），后到的刷新请求会失败

**解决方案**：

- 使用 `golang.org/x/sync/singleflight`，以 `refresh-{userID}` 为 key
- 第一个请求执行刷新，后续请求等待共享结果
- 刷新结果通过 `refreshResult` 结构体传递给等待的 goroutine

```go
result, err, _ := refreshGroup.Do(key, func() (any, error) {
    userUpdated, err := refresher.Refresh(ctx, user)
    // ... 更新 DB
    return &refreshResult{...}, nil
})
// 等待的 goroutine 从 result 拷贝 token 到自己的 user 对象
```

**TTL 策略**：仅在 token 过期前 30 分钟（`tokenMinTTL = 1800s`）才触发刷新。

### 5.2 取消先前流水线（Cancel Previous Pipelines）

**代码**：`server/pipeline/cancel.go:100-158`

**触发条件**：仓库配置了 `CancelPreviousPipelineEvents` 且当前事件匹配

**去重逻辑**：

```go
pipelineNeedsCancel := func(active *model.Pipeline) bool {
    if active.Event != pipeline.Event { return false }
    switch pipeline.Event {
    case model.EventPush:
        return pipeline.Branch == active.Branch  // 同分支 push 取消同分支
    default:
        return pipeline.Refspec == active.Refspec // 同 refspec 互斥
    }
}
```

**执行**：
1. `store.GetActivePipelineList(repo)` 获取所有活跃流水线
2. 对匹配的旧流水线调用 `Cancel()`
3. `Cancel()` 先通过 `Scheduler.ErrorAtOnce` 批量驱逐队列中的 workflow
4. 再更新 DB 中 pending workflow 为 skipped

### 5.3 队列互斥（FIFO Lock）

`fifo` 结构体内嵌 `sync.Mutex`（`queue/fifo.go:46`），所有队列操作（入队、出队、状态更新）都在锁保护下进行，保证并发安全。

### 5.4 Agent 过滤匹配（Worker Filter）

Agent 通过 `Poll` 注册 worker 和 filter 函数，队列 `assignToWorker` 使用 filter 做匹配打分（`fifo.go:314-336`）：

```go
matched, score := worker.filter(task)
if matched && score > bestScore {
    bestWorker = worker
    bestScore = score
}
```

这确保同一 task 只被分配给一个 worker，且选择匹配度最高的。

---

## 6 重试与死信回收

### 6.1 过期任务自动重入队（Resubmit Expired Pipelines）

**代码**：`server/queue/fifo.go:338-348`

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            taskState.error = ErrTaskExpired
            q.pending.PushFront(taskState.item)  // 重新放回队首
            delete(q.running, taskID)
            close(taskState.done)
        }
    }
}
```

**机制**：

- 每个 task 出队时设置 `deadline = time.Now().Add(extension)`（默认 `TaskTimeout = 1 分钟`，见 `shared/constant/constant.go:41`）
- Agent 必须定期调用 `Extend` 续约（通过 gRPC heartbeat）
- 若 Agent 崩溃或网络中断，deadline 过期后 task 自动从 `running` 移回 `pending` 队首
- 等待的 `Wait()` 会收到 `ErrTaskExpired`

### 6.2 持久化恢复（Task Store Recovery）

**代码**：`server/queue/persistent.go:32-38`

```go
func WithTaskStore(ctx context.Context, q Queue, s store.Store) Queue {
    tasks, _ := s.TaskList()           // 启动时从 DB 加载所有未完成 task
    q.PushAtOnce(ctx, tasks)           // 重新入队
    return &persistentQueue{q, s}
}
```

- Server 重启后，数据库中残留的 task 会被重新加载到内存队列
- 相当于「死信回收」：本来应执行但 server crash 导致丢失的任务被恢复

### 6.3 任务错误处理

**代码**：`server/queue/fifo.go:133-160`

`finished()` 方法处理任务完成/错误：

1. 先从 `running` 中移除 task，关闭 `done` channel
2. 若不在 `running` 中，尝试从 `pending` / `waitingOnDeps` 中移除
3. 更新依赖此 task 的其他 task 的 `DepStatus`
4. 错误被包装为 `ErrExternal`，`Wait()` 时会过滤掉外部错误

### 6.4 Agent 踢出（Kick Agent Workers）

**代码**：`server/queue/fifo.go:243-253`

当 Agent 断连时，`KickAgentWorkers(agentID)` 被调用：
- 遍历所有 worker，踢出匹配 agentID 的 worker
- 被踢的 worker 的 `context.CancelCauseFunc` 被触发，Poll 返回 `ErrWorkerKicked`
- 该 Agent 正在运行的 task 会在 deadline 过期后被 `resubmitExpiredPipelines` 回收

### 6.5 流水线重启（Restart）

**代码**：`server/pipeline/restart.go:32`

- 从旧 pipeline 获取 config，可选重新 fetch
- 创建新 pipeline 记录（`createNewOutOfOld` 重置 ID/Number/Status）
- `RerunCount++` 记录重试次数
- 走正常 `createPipelineItems → start` 流程

---

## 7 门控拦截（Gated Pipeline）

**代码**：`server/pipeline/gated.go:23-30`

```go
func setApprovalState(repo *model.Repo, pipeline *model.Pipeline) {
    if !needsApproval(repo, pipeline) { return }
    pipeline.Status = model.StatusBlocked
}
```

**审批策略**（`repo.RequireApproval`）：

| 值 | 行为 |
|----|------|
| `RequireApprovalNone` | 不拦截 |
| `RequireApprovalForks` | 来自 fork 的 PR 需审批 |
| `RequireApprovalPullRequests` | 所有 PR 需审批 |
| `RequireApprovalAllEvents` | 所有事件需审批 |

**豁免**：
- `EventCron` / `EventManual` 不拦截
- `repo.ApprovalAllowedUsers` 中的作者不拦截

门控状态下 pipeline 状态为 `StatusBlocked`，workflow 同步为 `StatusBlocked`，
不进入队列，等待人工 `Approve` 后才走 `start` 流程。

---

## 8 关键代码索引

| 模块 | 文件 | 核心函数/结构体 |
|------|------|----------------|
| Webhook 入口 | `server/api/hook.go` | `PostHook()` |
| JWT 校验 | `shared/token/token.go` | `ParseRequest()`, `keyFunc()` |
| Forge 接口 | `server/forge/forge.go` | `Forge.Hook()` |
| GitHub 解析 | `server/forge/github/parse.go` | `parseHook()`, `parsePushHook()` |
| GitLab 解析 | `server/forge/gitlab/gitlab.go` | `Hook()` |
| Gitea 解析 | `server/forge/gitea/parse.go` | `parseHook()` |
| Forgejo 解析 | `server/forge/forgejo/parse.go` | `parseHook()` |
| BDC 解析+签名 | `server/forge/bitbucketdatacenter/bitbucketdatacenter.go` | `Hook()`, `ValidateSignature()` |
| Pipeline 创建 | `server/pipeline/create.go` | `Create()` |
| YAML 编译 | `server/pipeline/items.go` | `parsePipeline()`, `createPipelineItems()` |
| 入队 | `server/pipeline/queue.go` | `queuePipeline()` |
| FIFO 队列 | `server/queue/fifo.go` | `fifo`, `process()`, `resubmitExpiredPipelines()` |
| 持久化队列 | `server/queue/persistent.go` | `WithTaskStore()`, `persistentQueue` |
| Token 刷新 | `server/forge/refresh.go` | `Refresh()`, `refreshGroup` |
| 取消/去重 | `server/pipeline/cancel.go` | `Cancel()`, `cancelPreviousPipelines()` |
| 门控 | `server/pipeline/gated.go` | `setApprovalState()`, `needsApproval()` |
| 重启 | `server/pipeline/restart.go` | `Restart()` |
| 状态管理 | `server/pipeline/pipeline_status.go` | `UpdateToStatusError()`, `UpdateToStatusKilled()` |
| Task 模型 | `server/model/task.go` | `Task`, `ShouldRun()` |
| 超时常量 | `shared/constant/constant.go` | `TaskTimeout` |
| 调度器 | `server/scheduler/scheduler.go` | `Scheduler` = `Queue` + `PubSub` |

---

## 9 总结

Woodpecker 的 webhook 调度采用 **双层校验 + 单入口 + 内存队列 + 持久化兜底** 的架构：

1. **签名校验**：入口层 JWT（每个仓库独立密钥）+ Forge 层可选的 payload 签名（BDC 已实现），确保请求来源可信
2. **载荷解析**：统一通过 `Forge.Hook()` 接口分发，各驱动按平台格式解析 push/PR/tag/release 事件
3. **仓库匹配**：token 中的 `ForgeRemoteID` 与载荷返回的交叉校验，加上 IsActive/UserID/AllowPull 等状态检查
4. **流水线编译**：YAML → `PipelineBuilder.Build()` → `builder.Item` → `model.Task`，注入 secrets/registries/envs
5. **并发拦截**：`singleflight` 防 OAuth token 并发刷新；`cancelPreviousPipelines` 做同分支/同 refspec 去重；`sync.Mutex` 保护队列状态
6. **死信回收**：过期 task 自动重入队（`resubmitExpiredPipelines`）；server 重启从 DB 恢复（`WithTaskStore`）；Agent 断连踢出后 task 自然过期回收
7. **门控**：按仓库策略拦截需审批的事件，`StatusBlocked` 状态不入队
