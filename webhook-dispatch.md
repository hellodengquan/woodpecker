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

**Token 签发位置**：`server/api/repo.go:155-165`

```go
t := token.New(token.HookToken)
t.Set("repo-forge-remote-id", string(forgeRemoteID))
t.Set("forge-id", strconv.FormatInt(repo.ForgeID, 10))
sig, err := t.Sign(repo.Hash)  // Sign() = SignExpires(secret, 0)
```

> **关键观察：无 jti 防重放**
>
> - `Sign()` 调用 `SignExpires(secret, 0)`，`exp=0` 表示**永不过期**（`token.go:140-142`）
> - 签发时未设置 `jti` (JWT ID) claim，`keyFunc` 中仅**跳过**了标准 claim (`iss/sub/aud/exp/nbf/iat/jti`) 而不做任何校验（`token.go:182`）
> - 没有 nonce 存储或时间窗校验：HookToken 一旦签发就是长期静态凭证，捕获后可无限重放
> - 实际防护依赖于 HTTPS 传输加密 + 每仓库独立 `repo.Hash`（密钥轮换通过重新激活仓库触发）

**安全设计要点**：

| 要点 | 位置 | 说明 |
|------|------|------|
| 算法锁定 | `token.go:168` | `t.Method.Alg() != SignerAlgo` 直接拒绝，防止 `alg: none` 攻击 |
| Token 类型白名单 | `token.go:59` | `slices.Contains(allowedTypes, token.Type)`，只允许 `HookToken` |
| 密钥绑定仓库 | `hook.go:79-85` | 每个仓库有独立 Hash，签名密钥互不干扰 |
| 仓库 ID 双重验证 | `hook.go:144` | token 中的 `ForgeRemoteID` 必须与 forge 返回的匹配 |
| **无 jti / 无过期** | `token.go:123` | `Sign()` = `SignExpires(secret, 0)`，永不过期无重放防护 |

### 2.2 Forge 层：Payload 签名校验 —— 兜底分析

Forge 接口契约在 `server/forge/forge.go:163` 明确要求：**Must verify webhook signature to prevent spoofing**。
但实际只有 Bitbucket DataCenter 严格兑现了契约，其余 Forge 均**退回依赖入口层 JWT**：

| Forge | Forge 层签名校验 | 实现方式 | 兜底机制 |
|-------|----------------|---------|---------|
| **Bitbucket DataCenter** | ✅ 完整 HMAC 校验 | `bitbucket.ValidateSignature(r, hook.Payload, []byte(repo.Hash))` | 入口层 JWT（双保险） |
| **GitHub** | ❌ 未实现 | `github.ParseWebHook(github.WebHookType(r), raw)` 无 secret 参数 | **依赖 JWT** + 官方 SDK 格式校验 |
| **GitLab** | ⚠️ 间接实现 | `X-Gitlab-Token` 头的值=JWT，在 `ParseRequest` 统一解析 | JWT 本身=签名校验 |
| **Gitea** | ❌ 未实现 | Webhook URL 直接嵌入 `access_token=<jwt>` | **纯依赖 JWT** |
| **Forgejo** | ❌ 未实现 | 同 Gitea，URL 携带 `access_token=<jwt>` | **纯依赖 JWT** |
| **Bitbucket Cloud** | ❌ 未实现 | 无签名校验代码分支 | **纯依赖 JWT** |

**Bitbucket DataCenter 完整签名流程**（`bitbucketdatacenter.go:506-514`）：
1. `bitbucket.ParsePayloadWithoutSignature(r)` 先用非校验路径解析
2. `bitbucket.ValidateSignature(r, hook.Payload, []byte(repo.Hash))` 对原始 HTTP payload 做 HMAC
3. 不匹配直接返回错误，拒绝处理

**其余 Forge 的隐性兜底**：
- 入口层 JWT 已做 HMAC-SHA256 签名校验，token 伪造不通过
- 但这意味着 **payload 与 token 的绑定不强制**：若攻击者拿到合法 JWT，可替换 payload body 任意内容（除了 Bitbucket DC 有二次校验）
- `hook.go:144` 的 `repo.ForgeRemoteID != repoFromForge.ForgeRemoteID` 交叉校验提供了部分防护——payload 中伪造的仓库 ID 必须与 token 中的一致

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
  └─ ⑩ start → cancelPreviousPipelines + queuePipeline
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

**代码**：`server/forge/refresh.go:23-101`

```go
var refreshGroup singleflight.Group  // 包级变量，进程内单例
```

**问题**：多个并发 webhook 同时触发 `forge.Refresh`，可能导致：
- 同一用户的 OAuth token 被并发刷新多次
- Forgejo 等平台使用一次性 refresh token（`InvalidateRefreshTokens=true`），后到的刷新请求会失败

**解决方案**：

- 使用 `golang.org/x/sync/singleflight`，以 `refresh-{userID}` 为 key
- 第一个请求执行刷新，后续请求等待共享结果
- 刷新结果通过 `refreshResult` 结构体传递给等待的 goroutine，后者将新鲜 token 拷贝到自己的 `*model.User` 副本

```go
key := fmt.Sprintf("refresh-%d", user.ID)
result, err, _ := refreshGroup.Do(key, func() (any, error) {
    userUpdated, err := refresher.Refresh(ctx, user)
    if userUpdated {
        _store.UpdateUser(user)  // 写入 DB
    }
    return &refreshResult{AccessToken, RefreshToken, Expiry}, nil
})
// 等待方：把结果写入自己的 user 对象副本
user.AccessToken = r.AccessToken
```

**TTL 策略**：仅在 token 过期前 30 分钟（`tokenMinTTL = 1800s`）才触发刷新。

> **跨 instance 限制**：
>
> `singleflight.Group` 是 **纯内存结构**，无分布式协调。在多副本部署（HA / K8s replicas > 1）下：
> - 每个 server 实例有独立的 `refreshGroup`，实例间不共享去重状态
> - 同一用户的并发请求若落到不同实例，仍会触发**并发刷新**
> - 对 Forgejo（一次性 refresh token）尤其危险：实例 A 刷新成功后 token 作废，实例 B 的刷新会拿到 401
> - 无 Redis / 数据库级分布式锁兜底，依赖请求层负载均衡的粘性（sticky session）缓解

### 5.2 取消先前流水线（Cancel Previous Pipelines）

**代码**：`server/pipeline/cancel.go:32-159`

**触发条件**：仓库配置了 `CancelPreviousPipelineEvents` 且当前事件匹配

**去重逻辑**：

```go
pipelineNeedsCancel := func(active *model.Pipeline) bool {
    if active.Event != pipeline.Event { return false }
    switch pipeline.Event {
    case model.EventPush:
        return pipeline.Branch == active.Branch   // 同分支 push 取消同分支
    default:
        return pipeline.Refspec == active.Refspec  // 同 refspec 互斥
    }
}
```

**执行路径（Cancel 函数）**：

```
Cancel()
  │
  ├─ ① 校验：pipeline.Status 必须是 Running/Pending/Blocked
  ├─ ② store.WorkflowGetTree → 拉取 workflow + step 完整树
  ├─ ③ ErrorAtOnce → 从队列中**批量驱逐** in-flight task
  │     (workflowsToCancel = 所有 Running + Pending 的 workflow ID)
  │
  ├─ ④ 更新 DB 中 Pending workflow → StatusSkipped
  ├─ ⑤ 更新 DB 中 Pending step → StatusSkipped (StatusCanceled)
  │
  ├─ ⑥ 判断 hasPendingOnly：
  │     · 全部 Pending → pipeline.Status = StatusCanceled
  │     · 含 Running → pipeline.Status = StatusKilled
  │
  ├─ ⑦ UpdateToStatusKilled → 写 pipeline 最终状态
  ├─ ⑧ updatePipelineStatus → 推送给 Forge （commit status）
  └─ ⑨ publishToTopic → PubSub 广播
```

> **in-flight task 清理的语义分层**：
>
> | 状态 | 队列侧（ErrorAtOnce） | 数据库侧 | 实际执行侧 |
> |------|---------------------|---------|-----------|
> | **Pending**（在 pending/waitingOnDeps 队列中） | `removeFromPendingAndWaiting` 直接移除 | 立即写 `StatusSkipped` | 不涉及 |
> | **Running**（在 running map 中，已分配给 Agent） | 设置 `entry.error = ErrCancel`，关闭 `done` channel | **DB 不立即更新** | Agent 侧 gRPC `Wait()` 检测到 `ErrCancel` → 停止执行 → 回调 `Done(canceled=true)` |
> | **已执行完毕的 workflow** | 不在队列，ErrorAtOnce 无操作 | 不更新 | —— |
>
> **关键延迟窗口**：Running workflow 在队列中已标记 cancel，但 Agent 必须等到下一次 `Wait()` 调用才检测到信号。这期间步骤可能仍在执行容器中运行，属于 **best-effort 取消**而非硬终止。

### 5.3 队列互斥（FIFO Lock）

`fifo` 结构体内嵌 `sync.Mutex`（`queue/fifo.go:46`），所有队列操作（入队、出队、状态更新、KickAgentWorkers）都在锁保护下进行，保证并发安全。

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

## 6 FIFO 优先级队列实现细节

**代码**：`server/queue/fifo.go`

Woodpecker 的队列名虽为 FIFO，但实际是**带优先级的双向链表**，底层数据结构为 Go 标准库 `container/list`：

```go
type fifo struct {
    sync.Mutex
    workers       map[*worker]struct{}
    running       map[string]*entry       // taskID → 运行中条目
    pending       *list.List               // 待执行，双向链表
    waitingOnDeps *list.List               // 等待依赖满足
    extension     time.Duration            // TaskTimeout
}
```

### 6.1 优先级语义

| 操作 | 入队位置 | 含义 |
|------|---------|------|
| 新 task `PushAtOnce` | `pending.PushBack(task)` | 队尾，FIFO 语义 |
| 过期 task 重入队 | `pending.PushFront(task)` | **队首**，重试优先于新任务 |
| `filterWaiting` 依赖满足 | `pending.PushBack(task)` | 队尾，重新参与调度 |

> **不是真正的优先级队列**：
> - 没有最小堆/最大堆，只有链表头尾两个插入点
> - 同一优先级内严格 FIFO
> - 唯一优先级区分：「重试 task」>「新 task」（通过 PushFront vs PushBack 实现）
> - 无法按 pipeline 紧急程度 / 仓库权重 / 调度公平性做细粒度优先级
> - `assignToWorker` 从头遍历 `pending` 队列，先到先匹配 worker

### 6.2 100ms 轮询时钟与漂移

**代码**：`server/queue/fifo.go:57-73, 257-287`

```go
const processTimeInterval = 100 * time.Millisecond

func (q *fifo) process() {
    for {
        select {
        case <-time.After(processTimeInterval):  // 相对延迟，非绝对时钟
        case <-q.ctx.Done():
            return
        }
        q.Lock()
        if q.paused { q.Unlock(); continue }
        q.resubmitExpiredPipelines()   // deadline 检查用 time.Now()
        q.filterWaiting()
        for pending, worker := q.assignToWorker(); ... { ... }
        q.Unlock()
    }
}
```

**漂移来源分析**：

| 因素 | 影响 | 代码位置 |
|------|------|---------|
| `time.After` 是**相对**延迟 | 若 `process()` 本体执行 30ms，则下一轮在 130ms 后，而非整点 100ms。高负载下 GC 或锁竞争导致单次 200ms，累计漂移严重 | `fifo.go:260` |
| `sync.Mutex` 等待时间不计入 tick | 高并发时 Lock() 可能阻塞几十 ms，tick 从拿到锁后才开始算 | `fifo.go:265` |
| `time.Now()` 墙钟依赖 | `resubmitExpiredPipelines` 用墙钟比较 deadline，NTP 校时跳变会误判过期/不过期 | `fifo.go:340` |
| Agent `Extend` 心跳间隔 1 分钟 | `updateAgentLastWorkDelay = 1m`，若 100ms tick 漏跑几轮（如 GC STW），task 可能被判过期误回收 | `rpc.go:51` |

> **实际影响**：
> - 正常情况下调度延迟约 **100ms 量级**（task 入队后最多等一个 tick 才被分配）
> - 注释 `as the agent pull in 10 milliseconds we should also give them work asap` 与实际 `100ms` 不符（Agent gRPC `Poll` 是阻塞长连接，不是 10ms 轮询）
> - 死信回收的延迟：task 过期后最多再等一个 100ms tick 才能被 `resubmitExpiredPipelines` 捡回

---

## 7 重试与死信回收

### 7.1 过期任务自动重入队（Resubmit Expired Pipelines）

**代码**：`server/queue/fifo.go:338-348`

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            taskState.error = ErrTaskExpired
            q.pending.PushFront(taskState.item)  // 重新放回队首（优先级最高）
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

### 7.2 持久化恢复（Task Store Recovery）

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

> **TaskList 大量加载延迟问题**：
>
> 实现位于 `server/store/datastore/task.go:21-24`：
> ```go
> func (s storage) TaskList() ([]*model.Task, error) {
>     tasks := make([]*model.Task, 0, perPage)  // perPage=50 仅预分配容量
>     return tasks, s.engine.Find(&tasks)       // SELECT * FROM tasks，全量加载
> }
> ```
>
> - 无分页、无 WHERE 过滤、无状态筛选（`store.go:152` TODO 注释：`// TaskList TODO: paginate & opt filter`）
> - 数据库中若有数千条积压 task（例如 server 长时间宕机），**启动时全量加载会阻塞**：
>   - SQL 查询传输延迟
>   - ORM 反序列化大量对象
>   - `PushAtOnce` 逐个 `PushBack` 到链表
> - `persistentQueue.PushAtOnce` 也是逐条 `TaskInsert`（无事务、无 batch），见 `persistent.go:47-57`
> - **无去重**：重启后可能与正常 Agent 回传的 `Done/Error` 竞争，同 task 既在 DB 中又被 Poll 过一次，可能产生重复执行（Agent 侧 gRPC `checkWorkflowState` 做二次校验兜底）

### 7.3 任务错误处理

**代码**：`server/queue/fifo.go:133-160`

`finished()` 方法处理任务完成/错误：

1. 先从 `running` 中移除 task，关闭 `done` channel
2. 若不在 `running` 中，尝试从 `pending` / `waitingOnDeps` 中移除
3. 更新依赖此 task 的其他 task 的 `DepStatus`
4. 错误被包装为 `ErrExternal`，`Wait()` 时会过滤掉外部错误

### 7.4 Agent 踢出（KickAgentWorkers）与优雅退出

**代码**：`server/queue/fifo.go:242-253, 87-111`

四个触发场景（`server/api/agent.go`）：

| 场景 | 代码位置 | 说明 |
|------|---------|------|
| 管理员设置 `NoSchedule=true` | `agent.go:145` | 阻止该 Agent 接收新任务 |
| 删除 Agent | `agent.go:225` | 删除前先踢 worker（但先检查无 running task） |
| 组织 Agent 设置 NoSchedule | `agent.go:348` | 同场景 1 |
| 用户 Agent 设置 NoSchedule | `agent.go:400` | 同场景 1 |

**执行流程**：

```go
func (q *fifo) KickAgentWorkers(agentID int64) {
    q.Lock()
    defer q.Unlock()
    for worker := range q.workers {
        if worker.agentID == agentID {
            worker.stop(ErrWorkerKicked)   // context.WithCancelCause
            delete(q.workers, worker)
        }
    }
}
```

Worker 侧 Poll 检测到取消：

```go
func (q *fifo) Poll(c context.Context, agentID int64, filter FilterFn) (*model.Task, error) {
    ctx, stop := context.WithCancelCause(c)
    w := &worker{agentID, filter, make(chan *model.Task, 1), stop}
    q.workers[w] = struct{}{}
    for {
        select {
        case <-ctx.Done():
            q.Lock(); delete(q.workers, w); q.Unlock()
            return nil, ctx.Err()          // 返回 context.Canceled
        case t := <-w.channel:
            return t, nil
        }
    }
}
```

> **优雅退出的不完整性**：
>
> 1. **Poll 中等待的 worker**：被踢后 `ctx.Done()` 触发，`Poll` 返回 `context.Canceled`。Agent 侧 gRPC `Next()` 检测到 error 后重连（`rpc.go:93`）
>
> 2. **已分配 task（running map 中）**：`KickAgentWorkers` **只删 worker，不动 running 条目**！
>    - 这些 task 继续留在 `q.running` 中，`deadline` 正常倒计时
>    - Agent 侧 gRPC `Wait()` 会最终收到断连错误，但 **Agent 端运行的容器不会被强杀**
>    - 等 `TaskTimeout`（默认 1 分钟）到期后，`resubmitExpiredPipelines` 才把 task 放回 `pending.PushFront`
>    - 这期间 task 被**悬空**：DB 里仍被标记为运行中，无任何 Agent 心跳续约
>
> 3. **TODO 注释佐证**：`rpc.go:62` 标注 `// TODO (6038): Server does not release waiting agents on graceful shutdown.`
>
> 4. **Cause 丢失**：`worker.stop(ErrWorkerKicked)` 设置了 cause，但 `ctx.Err()` 只返回通用 `context.Canceled`，调用方无法区分「被踢」和「普通取消」

### 7.5 流水线重启（Restart）

**代码**：`server/pipeline/restart.go:32`

- 从旧 pipeline 获取 config，可选重新 fetch
- 创建新 pipeline 记录（`createNewOutOfOld` 重置 ID/Number/Status）
- `RerunCount++` 记录重试次数
- 走正常 `createPipelineItems → start` 流程

---

## 8 门控拦截（Gated Pipeline）

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

## 9 关键代码索引

| 模块 | 文件 | 核心函数/结构体 |
|------|------|----------------|
| Webhook 入口 | `server/api/hook.go` | `PostHook()` |
| JWT 校验 | `shared/token/token.go` | `ParseRequest()`, `keyFunc()`, `Sign()`, `SignExpires()` |
| JWT HookToken 签发 | `server/api/repo.go` | `PostRepo()` 中 `token.New(HookToken) + t.Sign()` |
| Forge 接口 | `server/forge/forge.go` | `Forge.Hook()` 契约 |
| GitHub 解析 | `server/forge/github/parse.go` | `parseHook()`, `parsePushHook()` |
| GitLab 解析 | `server/forge/gitlab/gitlab.go` | `Hook()` |
| Gitea 解析 | `server/forge/gitea/parse.go` | `parseHook()` |
| Forgejo 解析 | `server/forge/forgejo/parse.go` | `parseHook()` |
| BDC 解析+签名 | `server/forge/bitbucketdatacenter/bitbucketdatacenter.go` | `Hook()`, `ValidateSignature()` |
| Pipeline 创建 | `server/pipeline/create.go` | `Create()` |
| YAML 编译 | `server/pipeline/items.go` | `parsePipeline()`, `createPipelineItems()` |
| 入队 | `server/pipeline/queue.go` | `queuePipeline()` |
| FIFO 队列 | `server/queue/fifo.go` | `fifo`, `process()`, `resubmitExpiredPipelines()`, `KickAgentWorkers()` |
| FIFO 链表调度 | `server/queue/fifo.go` | `PushAtOnce()`, `assignToWorker()`, `filterWaiting()` |
| 持久化队列 | `server/queue/persistent.go` | `WithTaskStore()`, `persistentQueue` |
| Task 全量加载 | `server/store/datastore/task.go` | `TaskList()` |
| Token 刷新 singleflight | `server/forge/refresh.go` | `Refresh()`, `refreshGroup` |
| 取消/去重 + in-flight 清理 | `server/pipeline/cancel.go` | `Cancel()`, `cancelPreviousPipelines()` |
| 门控 | `server/pipeline/gated.go` | `setApprovalState()`, `needsApproval()` |
| 重启 | `server/pipeline/restart.go` | `Restart()` |
| 状态管理 | `server/pipeline/pipeline_status.go` | `UpdateToStatusError()`, `UpdateToStatusKilled()` |
| Task 模型 | `server/model/task.go` | `Task`, `ShouldRun()` |
| 超时常量 | `shared/constant/constant.go` | `TaskTimeout` |
| Agent gRPC 协议 | `server/rpc/rpc.go` | `Next()`, `Wait()`, `Extend()` |
| Agent 踢出触发 | `server/api/agent.go` | `PatchAgent()`, `DeleteAgent()` |
| 调度器 | `server/scheduler/scheduler.go` | `Scheduler` = `Queue` + `PubSub` |

---

## 10 总结

Woodpecker 的 webhook 调度采用 **双层校验 + 单入口 + 内存队列 + 持久化兜底** 的架构，关键节点的设计取舍归纳如下：

### 10.1 已实现的安全与可靠性

1. **签名校验**：入口层 JWT-HS256（每个仓库独立密钥）+ 算法白名单防 `alg:none`；Bitbucket DC 额外做 payload HMAC 双保险
2. **载荷解析**：统一通过 `Forge.Hook()` 接口分发，各驱动按平台格式解析 push/PR/tag/release 事件
3. **仓库匹配**：token 中的 `ForgeRemoteID` 与 forge 返回值交叉校验，防止 token 跨仓库滥用
4. **流水线编译**：YAML → `PipelineBuilder.Build()` → `builder.Item` → `model.Task`，注入 secrets/registries/envs
5. **并发拦截**：
   - `singleflight` 防同一实例内 OAuth token 并发刷新
   - `cancelPreviousPipelines` 做同分支/同 refspec 去重，Pending 立删、Running 发取消信号
   - `sync.Mutex` 保护所有队列状态变更
6. **死信回收**：
   - 过期 task 自动重入队（`resubmitExpiredPipelines`，PushFront 优先级最高）
   - server 重启从 DB `TaskList()` 全量恢复
   - Agent 断连踢出后 running task 靠 deadline 超时自然回收

### 10.2 已知风险 / 改进空间

| 节点 | 现状 | 风险 |
|------|------|------|
| **JWT 防重放** | HookToken `Sign()` 无 `exp`、无 `jti`、无 nonce | 凭证被捕获可无限重放，只能靠重新激活仓库轮换密钥 |
| **Forge 签名兜底** | 仅 BDC 做 payload HMAC，其余纯依赖 JWT | 攻击者持合法 JWT 可伪造 payload（仓库 ID 交叉校验提供部分防护） |
| **singleflight 跨实例** | 纯内存去重，无分布式协调 | HA 部署下同一用户并发刷新仍可能冲突，Forgejo 一次性 token 尤甚 |
| **Running task 取消** | 仅关闭 done channel，Agent 靠 Wait() 轮询检测 | 取消信号传播延迟，容器可能已执行大半，best-effort 而非硬终止 |
| **FIFO 队列粒度** | 双向链表仅头尾两点插入 | 无真正优先级调度，无法按紧急度/权重做公平性控制 |
| **100ms 时钟漂移** | `time.After` 相对延迟 + 墙钟比较 deadline | 高负载下调度延迟放大，NTP 跳变可能误判 task 过期 |
| **TaskList 启动加载** | 无分页全量 `SELECT *` | 宕机恢复时大量积压 task 导致启动阻塞，无批量入库 |
| **KickAgentWorkers** | 只删 worker，不动 running task | 被踢 Agent 的 task 悬空 1 分钟靠超时回收，无任务优雅移交 |
| **优雅关闭** | `rpc.go:62` TODO(6038) 明确标注未实现 | Server 关机时阻塞中的 Poll 不会被释放，gRPC 连接硬断 |
