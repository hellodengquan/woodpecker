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
>
> **集中式 lock 改造方案评估**：
>
> | 方案 | 实现复杂度 | 一致性 | 性能开销 | 侵入性 |
> |------|-----------|--------|---------|--------|
> | **Redis SETNX + TTL** | 中 | 强（CP 倾向） | 每次刷新 1 次 RTT + 1 轮询 | 需引入 Redis 依赖，改造 `refresh.go` |
> | **DB 乐观锁（version 字段）** | 低 | 最终一致 | 每次刷新 1 次 SELECT + 1 次 UPDATE | 改 `users` 表加 `token_version`，零新依赖 |
> | **etcd / Consul lease** | 高 | 强 | 每次刷新 1 次 RTT + keepalive | 引入 etcd，架构负担重 |
> | **用户级分片 + sticky session** | 低 | 弱 | 零额外开销 | 仅在 LB 层配置，不改代码 |
>
> **推荐路径：DB 乐观锁兜底 + 内存 singleflight 双层**
>
> 改动集中在 `server/forge/refresh.go`，伪代码：
> ```go
> // 先过内存 singleflight（快路径）
> result, err, _ := refreshGroup.Do(key, func() (any, error) {
>     // 再走 DB 乐观锁（慢路径，跨实例兜底）
>     for i := 0; i < 3; i++ {
>         u, _ := _store.GetUser(user.ID)  // 重读最新 token
>         if time.Now().UTC().Unix() < u.Expiry - tokenMinTTL {
>             return &refreshResult{...}, nil  // 已有其他实例刷新过
>         }
>         newTok, err := refresher.Refresh(ctx, u)
>         if err != nil { return nil, err }
>         // UPDATE users SET token=?, expiry=? WHERE id=? AND expiry=?
>         // 通过旧 expiry 做 CAS，若影响行数=0 说明并发冲突，重试
>         rows, _ := _store.UpdateUserCAS(u, oldExpiry)
>         if rows == 1 { return &refreshResult{...}, nil }
>     }
> })
> ```
>
> 权衡：
> - 优点：零新组件依赖，利用现有数据库，失败回退到「可能重复刷新但不崩溃」
> - 缺点：数据库写压力增加，Forgejo 一次性 token 场景仍可能有竞态窗口
> - 进一步加固：可在 `repo_user` 表加 `last_refresh_at` 做简单速率限制

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
>
> **in-flight done channel 补偿任务分析**：
>
> 当前 `Cancel()` 对 Running task 的处理链路：
> ```
> Cancel() → ErrorAtOnce(ids, ErrCancel)
>   → finished()
>     → 从 running map 找 entry
>     → 设 entry.error = ErrExternal(ErrCancel)
>     → close(entry.done)    ← 仅发信号，不等待 Agent 响应
>     → 删除 running 中的 entry
> ```
>
> **补偿缺口**：
> 1. **状态同步延迟**：DB 中 workflow 仍为 `Running`，要等 Agent 回传 `Done(canceled=true)` 才更新。这段窗口内 `pipeline.Status` 与实际执行状态不一致
> 2. **Agent 无响应风险**：若 Agent 崩溃或网络断开，`Wait()` 永远收不到，DB 状态永远停在 Running
> 3. **done channel 单向**：`close(done)` 是 server→Agent 的单向通知，无 ack 机制，server 不知道 Agent 收到没有
>
> **补偿方案**：
>
> | 方案 | 实现方式 | 代码改动点 |
> |------|---------|-----------|
> | **超时兜底** | Cancel 后启动一个 timer（如 30s），到期若 workflow 仍为 Running 则强制写 StatusKilled | `pipeline/cancel.go` 加 goroutine 定时器 |
> | **ack 通道** | Agent 收到 cancel 后先发 ack 再执行清理，server 等 ack 后才从 running map 删 | `queue.Queue` 接口加 `AckCancel(taskID)` 方法 |
> | **补偿任务** | 利用持久化队列 + 死信语义，写一条「cancel_compensation」task 到延迟队列，到期检查并补偿 | `queue/` 新增延迟队列接口 |
>
> **最小可行补偿**：在 `pipeline/cancel.go` 的 `Cancel()` 末尾加一个 30 秒的 goroutine，到期后检查 DB 状态，若仍是 Running 则强制写 StatusKilled 并记录告警日志。利用 `UpdateToStatusKilled` 已有的幂等性保证安全性。
>
> **30s timer 兜底 jitter 设计**：
>
> 若大量流水线同时被 Cancel（如 100 条流水线同分支 push 触发 cancelPreviousPipelines），不加 jitter 会导致 30s 后 100 个 goroutine 同时写 DB，形成「惊群效应」。
>
> **jitter 方案**（使用 Go 标准库 math/rand）：
> ```go
> // cancel.go 补偿 goroutine
> baseDelay := 30 * time.Second
> jitter := time.Duration(rand.Int63n(int64(10 * time.Second)))  // ±5s 抖动
> totalDelay := baseDelay + jitter
>
> timer := time.NewTimer(totalDelay)
> defer timer.Stop()
> ```
>
> 与项目现有实践对齐：`github.com/cenkalti/backoff/v5` 已在 `server/store/datastore/pipeline.go:140` 使用指数退避，可复用 `backoff.NewExponentialBackOff()` 的 jitter 逻辑。
>
> **参数建议**：
> | 参数 | 值 | 理由 |
> |------|---|------|
> | 基础延迟 | 30s | 给 Agent 正常回传 Done(canceled=true) 留足时间窗口 |
> | Jitter 范围 | ±5s | 避免同秒 DB 尖峰 |
> | 最大并发补偿 goroutine | 可配置（默认 100） | 防止内存泄漏（cancel 风暴） |
>
> **三补偿方案幂等键设计**：
>
> 三种补偿方案各自的幂等键，确保重复执行补偿不破坏状态：
>
> | 方案 | 幂等键 | 幂等保障 |
> |------|--------|---------|
> | **超时兜底（goroutine） | `workflow_id`（DB 行级锁 |
> | **ack 通道 | `cancel_ack:{workflow_id}`（内存 map + CAS） |
> | **延迟补偿队列** | `compensation:{workflow_id}:{cancel_trigger_ts}`（DB 唯一索引 |
>
> **超时兜底幂等实现**（30s timer 方案）：
> 利用 `UpdateToStatusKilled` 的幂等性——它只在当前状态属于 `Pending/Running/Blocked` 时才执行变更，否则直接返回。
> ```go
> // pipeline/pipeline_status.go: UpdateToStatusKilled 已有状态前置检查
> func UpdateToStatusKilled(s store.Store, p model.Pipeline, finished int64) (*model.Pipeline, error) {
>     // 只有 Pending/Running/Blocked → Killed，其余直接返回
> }
> ```
> 即便多个补偿 goroutine 同时到期写，只有第一个真正执行 UPDATE，后续都是快速返回。
>
> **ack 通道幂等**：
> 内存中 `atomic.Bool` 标记 `cancelAcked`，Agent 回传 ack 后 CAS 设置为 true，补偿 goroutine 退出前检查该标记。
>
> **延迟补偿队列幂等**：
> 新增表 `pipeline_compensations`，唯一键 `(workflow_id, cancel_trigger_timestamp)`，补偿执行前先 INSERT，唯一键冲突说明已补偿则跳过。

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
>
> **FIFO 链表升级到堆的成本评估**：
>
> | 维度 | 当前链表方案 | 堆（heap）方案 |
> |------|-------------|---------------|
> | **入队 Push** | O(1) PushBack | O(log n) 上浮 |
> | **出队 Pop（最高优先级）** | O(n) 遍历查找（当前 `assignToWorker` 从头扫到第一个匹配的） | O(log n) 下沉 |
> | **随机删除（removeFromPendingAndWaiting）** | O(n) 遍历找元素 → O(1) 删 | O(n) 查找 → O(log n) 调整 |
> | **依赖过滤（filterWaiting）** | O(n) 整体重建链表 | O(n) 扫描 + O(k log n) 重新入堆 |
> | **内存占用** | `list.Element` 每个约 3 指针开销 | 切片存储，更紧凑 |
> | **实现复杂度** | 低（标准库开箱即用） | 中（需实现 `heap.Interface` 的 5 个方法） |
> | **优先级粒度** | 2 级（PushFront / PushBack） | 任意整数优先级，可多维度（仓库权重 + 重试次数 + 排队时间） |
>
> **关键性能瓶颈识别**：
> 当前 `assignToWorker` 是 `O(n * m)`（n = pending 数, m = worker 数），因为每个 task 都要遍历所有 worker 打分。
> 即使换成堆，瓶颈仍在 worker 匹配（filter 函数调用），不在数据结构本身。
>
> **改造建议**：
> 1. 先把 `assignToWorker` 的「每个 task 遍历所有 worker」改为「每个 worker 取最高优先级 task」，减少 O 常数
> 2. 若仍需多优先级，再引入 `container/heap`，`Less()` 按 `[优先级级别, 入队时间戳]` 排序
> 3. `waitingOnDeps` 保持链表不变（依赖满足靠事件驱动，不是轮询）
> 4. 新增 `priority int` 字段到 `model.Task`，默认 0（普通），重试+1，阻塞-1
>
> **代码改动量估算**：约 150-200 行（堆实现 + queue 接口适配 + 单测），属中等改造规模。

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
>
> **时钟漂移的守护进程改造方案**：
>
> **方案一：time.Ticker 替换 time.After（低风险，快速见效）**
>
> ```go
> // 当前：每次循环后重新 time.After，相对延迟累积漂移
> case <-time.After(processTimeInterval):
>
> // 改为：Ticker 基于单调时钟，自动纠正漂移
> ticker := time.NewTicker(processTimeInterval)
> defer ticker.Stop()
> case <-ticker.C:
> ```
>
> 改动仅 3 行，消除「每次 tick 包含 process() 执行时间」导致的漂移。但注意 `time.Ticker` 仍受 Go 调度器影响，极端 GC 下还是会漏 tick。
>
> **方案二：单调时钟 + 漂移校准（中风险，更精确）**
>
> 记录上次 tick 的 `monotonic time`，每次触发计算实际间隔，若 > 1.5x 阈值则记告警日志：
> ```go
> lastTick := time.Now()
> for {
>     select {
>     case <-ticker.C:
>         now := time.Now()
>         elapsed := now.Sub(lastTick)
>         if elapsed > processTimeInterval * 3 / 2 {
>             log.Warn().Dur("elapsed", elapsed).Msg("queue process tick drift detected")
>         }
>         lastTick = now
>         // ... 业务逻辑
>     }
> }
> ```
>
> **方案三：事件驱动 + 轮询双触发（高收益，中复杂度）**
>
> 新 task 入队时通过 channel 立即唤醒 process()，不必等下一个 tick：
> ```go
> type fifo struct {
>     // ...
>     wakeupCh chan struct{}  // 新 task 入队时发信号
> }
>
> func (q *fifo) PushAtOnce(...) {
>     // ... 入队后
>     select {
>     case q.wakeupCh <- struct{}{}:
>     default:  // 非阻塞，避免堆积
>     }
> }
>
> // process() 里：
> case <-ticker.C:
> case <-q.wakeupCh:  // 立即响应新 task
> ```
>
> 这样平均调度延迟从 ~50ms（等待半个 tick）降到 <1ms，100ms tick 仅作兜底。
>
> **监控指标建议**：
> - `queue_process_tick_duration_seconds`：每次 process() 执行耗时的直方图
> - `queue_process_tick_drift_total`：漂移超阈值的计数
> - `queue_scheduling_latency_seconds`：task 入队到分配的延迟直方图

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
>
> **TaskList 全表 SELECT 的向后兼容改造**：
>
> **问题本质**：`TaskList()` 是 `store.Store` 接口的一部分（`server/store/store.go:152`），被 `queue.WithTaskStore` 唯一使用。接口签名是 `TaskList() ([]*model.Task, error)`，无参数，无法加过滤条件。
>
> **改造路径（保持向后兼容）**：
>
> | 阶段 | 方案 | 兼容性 | 说明 |
> |------|------|--------|------|
> | 阶段 1 | 新增 `TaskListFiltered(ctx, status, page, size int)` 方法到 `Store` 接口 | 新增方法不破坏 | 现有 `TaskList()` 保留，内部可委托给新方法 |
> | 阶段 2 | `WithTaskStore` 启动时分页批量加载，首批高优 task 先入队 | 接口不变，内部优化 | 按 `pipeline_id` 排序，先加载最近的 N 条 |
> | 阶段 3 | 后台 goroutine 异步加载剩余 task | 对外接口不变 | 避免启动阻塞 |
>
> **最小改动方案（阶段 1 + 阶段 2 合并）**：
> ```go
> // store.Store 接口新增可选方法（或用接口扩展模式）
> type TaskListOptions struct {
>     Status  string  // "" = 全部
>     Page    int
>     PerPage int
> }
> type TaskListFilteredStore interface {
>     TaskListFiltered(TaskListOptions) ([]*model.Task, int64, error)
> }
>
> // WithTaskStore 中使用类型断言
> if fs, ok := s.(TaskListFilteredStore); ok {
>     // 分页加载
> } else {
>     tasks, _ = s.TaskList()  // 兜底：旧实现
> }
> ```
>
> **向后兼容要点**：
> 1. 现有 `mock_store` 实现（测试用）无需修改，类型断言不匹配自动走兜底
> 2. `e2e` 测试和 `datastore` 实现可逐步迁移
> 3. 不引入 breaking change，小版本可发布

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
>
> **KickAgentWorkers 1 分钟 deadline 告警**：
>
> 被踢 Agent 的 running task 有长达 1 分钟的「悬空期」，在此期间：
> - 监控面板上 pipeline 状态仍为 Running，用户无法感知已断开
> - 若同时踢出大量 Agent（如 K8s 节点缩容），会有大量 task 滞留在 running map
> - 没有任何告警机制提示「task 无心跳但仍标记为 running」
>
> **告警改造方案**：
>
> 在 `resubmitExpiredPipelines` 中增加 Prometheus 指标和日志告警：
> ```go
> // fifo.go 中新增指标（借助已有的 prometheus Registerer 模式）
> var taskResubmittedTotal = promauto.NewCounterVec(
>     prometheus.CounterOpts{Name: "woodpecker_queue_task_resubmitted_total"},
>     []string{"reason"},
> )
>
> func (q *fifo) resubmitExpiredPipelines() {
>     for taskID, taskState := range q.running {
>         if time.Now().After(taskState.deadline) {
>             taskResubmittedTotal.WithLabelValues("expired").Inc()
>             if time.Now().After(taskState.deadline.Add(30 * time.Second)) {
>                 log.Warn().Str("task_id", taskID).Msg("task expired and resubmitted after 30s grace period")
>             }
>             // ... 原有重入队逻辑
>         }
>     }
> }
> ```
>
> 配合 Prometheus 告警规则：
> ```yaml
> - alert: TaskExpiredResubmitted
>   expr: rate(woodpecker_queue_task_resubmitted_total{reason="expired"}[5m]) > 0
>   for: 2m
>   annotations:
>     summary: "Tasks are expiring and being resubmitted"
>     description: "Agents may be disconnecting frequently"
> ```
>
> **ctx.Err cause 丢失的修复 PR 思路**：
>
> **问题根因**：`context.WithCancelCause` 设置的 cause 必须通过 `context.Cause(ctx)` 读取，`ctx.Err()` 永远返回 `context.Canceled`。当前 `Poll()` 返回 `ctx.Err()`，丢掉了 `ErrWorkerKicked` 这个关键信息。
>
> **修复代码**：
> ```go
> // fifo.go Poll 函数中
> case <-ctx.Done():
>     q.Lock(); delete(q.workers, w); q.Unlock()
>     // 修复：返回 cause 而非通用 Canceled
>     if cause := context.Cause(ctx); cause != nil && !errors.Is(cause, context.Canceled) {
>         return nil, cause
>     }
>     return nil, ctx.Err()
> ```
>
> **调用链影响面**：
>
> | 调用方 | 当前行为 | 修复后行为 | 风险 |
> |--------|---------|-----------|------|
> | `rpc.Next()` | 收到 `codes.Canceled` → `backoff.Permanent` 不重试 | 收到 `ErrWorkerKicked` → `classifyRPCErr` 按默认 case 也判为 Permanent，不重试 | 行为一致 |
> | `rpc.Wait()` | 收到 `context.Canceled` | 收到具体错误（`ErrWorkerKicked` / `ErrCancel` / `ErrTaskExpired`） | 需确认上层是否依赖具体错误类型 |
> | Agent runner | 仅判断 `err != nil` 触发停止 | 同样判断 `err != nil` 停止 | 无影响 |
> | `persistentQueue.Error` | 区分 `ErrCancel` / `ErrTaskExpired` | 新增 `ErrWorkerKicked` 也需被识别为非外部错误 | 需同步更新 |
>
> **验收标准**：
> - Poll 返回 `ErrWorkerKicked`（当被 Kick 时）
> - Poll 返回 `ErrCancel`（当 task 被 Cancel 时）
> - Poll 返回 `context.Canceled`（当调用方 context 取消时）
> - gRPC 层将 cause 正确映射为 gRPC Status（可复用 `codes.Canceled`，但 message 带 cause 信息）
> - Agent 侧日志可明确区分「被踢出」「任务取消」「网络断开」三种场景
>
> **ctx.Cause 修复的单元测试矩阵**：
>
> 对应现有测试文件 `server/queue/fifo_test.go`，需新增/修正以下用例：
>
> | # | 测试场景 | 触发方式 | 预期返回错误 | 现有断言 | 修复后断言 |
> |---|---------|---------|-------------|---------|-----------|
> | 1 | Poll 调用方 context 取消 | `pollCancel(nil)` | `context.Canceled` | `assert.ErrorIs(err, context.Canceled)` ✅ | 不变 |
> | 2 | Poll 中 Agent 被 KickAgentWorkers | `q.KickAgentWorkers(42)` | `queue.ErrWorkerKicked` | `assert.ErrorIs(err, context.Canceled)` ❌ 需改 | `assert.ErrorIs(err, queue.ErrWorkerKicked)` |
> | 3 | Poll 中调用方 Cancel 带 cause | `pollCancel(customErr)` | `customErr` | 未覆盖 | `assert.ErrorIs(err, customErr)` |
> | 4 | Wait 中 task 被 Cancel | `q.ErrorAtOnce(ids, queue.ErrCancel)` | `queue.ErrCancel` | 已有覆盖 ✅ | 不变 |
> | 5 | Wait 中 task 过期重入队 | deadline 超时 | `queue.ErrTaskExpired` | 未覆盖 | `assert.ErrorIs(err, queue.ErrTaskExpired)` |
> | 6 | Wait 中 Agent worker 被 Kick | `KickAgentWorkers` + Wait 阻塞 | Wait 仍用 task 级别 cause，不触发 worker 级 cause | 未覆盖 | 验证 Wait 不受 worker kick 影响（Wait 绑 task 不绑 worker） |
> | 7 | cause 为 `context.Canceled` 本身（兼容） | `pollCancel(context.Canceled)` | 返回 `context.Canceled` | 未覆盖 | 防死循环：cause == context.Canceled 时不走 cause 分支 |
> | 8 | wrapped cause | `pollCancel(fmt.Errorf("wrap: %w", queue.ErrWorkerKicked))` | unwrap 后仍能 `errors.Is(err, ErrWorkerKicked)` | 未覆盖 | `assert.ErrorIs(err, queue.ErrWorkerKicked)` |
>
> **测试代码骨架**（对应用例 #2）：
> ```go
> t.Run("poll returns ErrWorkerKicked on kick", func(t *testing.T) {
>     pollResults := make(chan error, 1)
>     go func() {
>         _, err := q.Poll(ctx, 42, filterFnTrue)
>         pollResults <- err
>     }()
>     time.Sleep(50 * time.Millisecond)
>     q.KickAgentWorkers(42)
>     select {
>     case err := <-pollResults:
>         assert.ErrorIs(t, err, queue.ErrWorkerKicked)  // 修复前是 context.Canceled
>     case <-time.After(time.Second):
>         t.Fatal("Poll should return when worker is kicked")
>     }
> })
> ```

### 7.5 rpc.go 6038 graceful shutdown 影响半径

**TODO 原文**（`server/rpc/rpc.go:62`）：
```go
// TODO (6038): Server does not release waiting agents on graceful shutdown.
```

**影响半径分析**：

```
Server 优雅关闭
  │
  ├─ cmd/server/server.go: stopServerFunc() → ctxCancel()
  │
  ├─ ① HTTP/TLS/Metrics Server: Shutdown(shutdownCtx)
  │     └─ 新连接被拒，存量请求 5s 内完成（shutdownTimeout=5s）
  │
  ├─ ② gRPC Server: grpcServer.GracefulStop()
  │     └─ 新 RPC 被拒，存量流式 RPC 继续
  │
  ├─ ③ Cron Service: ctx.Done() → 停止调度
  │
  └─ ④ Queue fifo.process(): 循环中 <-q.ctx.Done() → return
        └─ 队列停止调度，但 waiting agent 不释放 ★ 问题点
```

**受影响模块清单**：

| 模块 | 影响方式 | 严重度 | 说明 |
|------|---------|--------|------|
| **Agent gRPC `Next()` 长连接** | 阻塞在 `Poll()` 的 Agent 直到 gRPC `GracefulStop` 超时才被断 | 中 | 需等 gRPC 强制关闭连接，非优雅通知 |
| **Agent gRPC `Wait()` 长连接** | 阻塞在 `Wait()` 的 workflow 同样等连接断开 | 中 | Agent 侧 `status.Convert(err)` 拿到 `Unavailable` |
| **队列 running task** | Server 退出时无回写，task 仍在 running map 和 DB | 高 | 重启后靠 `WithTaskStore` 恢复，可能重复执行 |
| **队列 pending task** | 留在内存队列不落地 | 中 | 若未开持久化队列，pending task 全部丢失 |
| **PubSub 订阅者** | `api/stream.go` 中 stream 被连接断开带走 | 低 | 客户端重连即可 |
| **Logger 流式日志** | 日志流断连，Agent 端缓冲丢失 | 低 | 不影响流水线结果 |

**为什么是问题**：
1. **Agent 侧体验差**：Agent 收到 `Unavailable` 错误而非明确的「server shutdown」信号，无法区分「服务维护」和「网络故障」
2. **恢复慢**：Agent 的指数退避重连从 0 开始，多等好几秒
3. **重复执行风险**：running task 没有在 shutdown 时主动回写状态，重启后 `TaskList` 重新加载，可能与 Agent 回传的 `Done` 竞争
4. **不符合 k8s 滚动更新最佳实践**：滚动更新期间 Agent 连接频繁断连，影响流水线稳定性

**6 受影响模块的依赖图**：

```
cmd/server/server.go (stopServerFunc)
  │
  ├─ ctxCancel()  ──── 根 context 取消
  │     │
  │     ├─ cron_scheduler.Run(ctx)   ← 独立 goroutine
  │     ├─ runGrpcServer(ctx)        ← 独立 goroutine
  │     │     │
  │     │     └─ server/rpc/serve.go (Serve)
  │     │           │
  │     │           ├─ grpcCtx, cancel := WithCancelCause(ctx)
  │     │           │     │
  │     │           │     └─ <-grpcCtx.Done() → GracefulStop()
  │     │           │
  │     │           └─ RPC (WoodpeckerServer)
  │     │                 ├─ Next() → scheduler.Poll() → fifo.Poll()
  │     │                 ├─ Wait() → scheduler.Wait() → fifo.Wait()
  │     │                 ├─ Extend() → scheduler.Extend() → fifo.Extend()
  │     │                 └─ Done/Update/Init/Log/ReportHealth/RegisterAgent/UnregisterAgent
  │     │
  │     ├─ TLS/HTTP/Redirect/Metrics Server
  │     │     └─ <-ctx.Done() → Shutdown(shutdownCtx)
  │     │
  │     └─ (fifo.process goroutine, 由 queue.NewMemoryQueue 启动)
  │           └─ <-q.ctx.Done() → return   ★ 此 ctx 独立创建，不随根 ctx 走！
  │
  └─ shutdownCtx, shutdownCancelFunc = WithTimeout(Background, 5s)
        └─ HTTP Server Shutdown 参数

依赖断裂点：
  ★ fifo.process 的 ctx 由 NewMemoryQueue 内部创建，不挂在根 ctx 上
  ★ Poll/Wait 阻塞在内部 worker channel/done channel 上，根 ctx 取消无法穿透
  ★ Logger stream 的 ctx 是 stream-specific，不受根 ctx 控制
```

**ServerStatus 探活频率设计**：

| 参数 | 建议值 | 理由 |
|------|--------|------|
| 初始探活间隔 | **15s** | Agent 启动时检查 server 版本兼容性，无需太频繁 |
| 稳态探活间隔 | **30s** | 平衡探活开销（gRPC RTT ~ms 级）和发现延迟 |
| 连续失败阈值 | **2 次** | 避免网络抖动误判 |
| 失败后退避 | 指数退避，最大 2min | 与 Agent gRPC 重连退避一致（`retry-timeout` 默认 15min） |
| shutdown 信号后间隔 | **5s** | 即将关机时提高频率，让 Agent 尽快感知 |

对齐现有参数：
- `updateAgentLastWorkDelay = 1m`（Agent 心跳）
- `grpc-keepalive-time`（默认 2h，gRPC 保活）
- `shutdownTimeout = 5s`（HTTP 优雅关闭超时）

探活 gRPC 方法设计：
```protobuf
message ServerStatusRequest {}
message ServerStatusResponse {
  bool healthy = 1;
  bool shutting_down = 2;   // true = 正在 graceful shutdown
  string message = 3;
  int64 estimated_shutdown_in_seconds = 4;  // 预估剩余关机窗口
  string grpc_version = 5;
  string server_version = 6;
}
rpc ServerStatus(ServerStatusRequest) returns (ServerStatusResponse);
```

**4 路 graceful shutdown 回滚预案**：

| 回滚预案 | 触发条件 | 执行动作 | 验证方式 |
|----------|---------|---------|---------|
| **P0：特性开关回滚** | KickAgentWorkers(-1) 引发 Agent 大量重连风暴 | 环境变量 `WOODPECKER_GRACEFUL_KICK=false` 关闭新逻辑 | 观察 Agent 连接数恢复到基线 |
| **P1：gRPC 版本降级** | ServerStatus 探活协议引发 Agent 旧版本兼容问题 | Server 端卸载 `ServerStatus` RPC 注册，保留老方法 | Agent 无法再调用新方法，回退到被动断连 |
| **P2：配置降级** | 30s timer 补偿引发 DB 写压力骤增 | 调大补偿超时 `WOODPECKER_CANCEL_GRACE_PERIOD=120s` 或设 0 禁用 | DB 写 QPS 回落，补偿 goroutine 数量为 0 |
| **P3：二进制回滚** | shutdown 时数据不一致或进程 hang | K8s 回滚 deployment 到上一版本镜像 | 所有 Pod 恢复旧代码，task 从 DB 重新加载 |

**回滚依赖链**：P0 → P1 → P2 → P3，逐级升级，每步留 5 分钟观察窗口。

**修复路线图**：
1. **短期（最小改动）**：在 `Serve()` 的 `<-grpcCtx.Done()` 分支，先调用 `scheduler.Queue.KickAgentWorkers(-1)`（特殊 ID = 全部踢掉），再 `GracefulStop()`
2. **中期**：新增 gRPC 方法 `ServerStatus()` 让 Agent 主动探活，或在 `Next()` 响应中带 shutdown hint
3. **长期**：实现 task 迁移协议，shutdown 时将 running task 优雅转移给其他 Agent

### 7.6 流水线重启（Restart）

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
| FIFO 队列核心 | `server/queue/fifo.go` | `fifo`, `process()`, `resubmitExpiredPipelines()`, `KickAgentWorkers()` |
| FIFO 链表调度 | `server/queue/fifo.go` | `PushAtOnce()`, `assignToWorker()`, `filterWaiting()`, `finished()` |
| 持久化队列 | `server/queue/persistent.go` | `WithTaskStore()`, `persistentQueue` |
| Task 全量加载 | `server/store/datastore/task.go` | `TaskList()`, `TaskInsert()`, `TaskDelete()` |
| Store 接口定义 | `server/store/store.go` | `Store.TaskList` (TODO: paginate & opt filter) |
| Token 刷新 singleflight | `server/forge/refresh.go` | `Refresh()`, `refreshGroup`, `refreshResult` |
| 取消/去重 + in-flight 清理 | `server/pipeline/cancel.go` | `Cancel()`, `cancelPreviousPipelines()`, `CancelPreviousPipelines()` |
| done channel 信号 | `server/queue/fifo.go` | `entry.done`, `entry.error`, `Wait()` |
| 门控 | `server/pipeline/gated.go` | `setApprovalState()`, `needsApproval()` |
| 重启 | `server/pipeline/restart.go` | `Restart()` |
| 状态管理 | `server/pipeline/pipeline_status.go` | `UpdateToStatusError()`, `UpdateToStatusKilled()` |
| Task 模型 | `server/model/task.go` | `Task`, `ShouldRun()` |
| 超时常量 | `shared/constant/constant.go` | `TaskTimeout` |
| Agent gRPC 协议 | `server/rpc/rpc.go` | `Next()`, `Wait()`, `Extend()`, TODO(6038) |
| gRPC 服务启动/关闭 | `server/rpc/serve.go` | `Serve()`, `GracefulStop()` |
| gRPC 错误分类 | `agent/rpc/client_grpc.go` | `classifyRPCErr()`, `retryRPC()` |
| context.Cause 读取 | `shared/utils/context.go` | `WithCancelCause` 模式 |
| Agent 踢出触发 | `server/api/agent.go` | `PatchAgent()`, `DeleteAgent()`, `NoSchedule` |
| Server 优雅关闭入口 | `cmd/server/server.go` | `stopServerFunc`, `shutdownCtx`, `shutdownTimeout` |
| Prometheus 指标 | `server/rpc/server.go` | `pipelineTime`, `pipelineCount`, `Registerer` |
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
| **in-flight done channel** | 单向通知无 ack，无补偿任务 | Agent 崩溃时 DB 状态永久停在 Running，状态不一致 |
| **FIFO 队列粒度** | 双向链表仅头尾两点插入 | 无真正优先级调度，无法按紧急度/权重做公平性控制 |
| **100ms 时钟漂移** | `time.After` 相对延迟 + 墙钟比较 deadline | 高负载下调度延迟放大，NTP 跳变可能误判 task 过期 |
| **TaskList 启动加载** | 无分页全量 `SELECT *` | 宕机恢复时大量积压 task 导致启动阻塞，无批量入库 |
| **KickAgentWorkers** | 只删 worker，不动 running task | 被踢 Agent 的 task 悬空 1 分钟靠超时回收，无告警、无优雅移交 |
| **ctx.Err cause 丢失** | `Poll()` 返回 `ctx.Err()` 而非 `context.Cause(ctx)` | 调用方无法区分「被踢」「被取消」「正常退出」 |
| **优雅关闭** | `rpc.go:62` TODO(6038) 明确标注未实现 | Server 关机时阻塞中的 Poll 不会被释放，gRPC 连接硬断 |

### 10.3 改造路线图（按性价比排序）

| 优先级 | 改造项 | 预计代码量 | 收益 | 风险 | 依赖 |
|--------|--------|-----------|------|------|------|
| **P0** | `Poll()` 返回 `context.Cause(ctx)` 修复 cause 丢失 | ~20 行 | 错误可观测性大幅提升，Agent 端日志可区分原因 | 低，向后兼容 | 无 |
| **P0** | `time.Ticker` 替换 `time.After` 消除累积漂移 | ~5 行 | 调度时钟稳定，消除漂移累积 | 极低 | 无 |
| **P1** | graceful shutdown 时 `KickAgentWorkers(-1)` 全量踢出 | ~10 行 | Agent 收到明确断开信号，可优雅重连 | 低 | 需先修 cause 丢失 |
| **P1** | Cancel 补偿任务：30s 超时强制写 DB StatusKilled | ~40 行 | 消除状态不一致窗口，用户可见性提升 | 中，需幂等校验 | `UpdateToStatusKilled` 已有幂等性 |
| **P1** | Prometheus 指标：task 过期重入队计数 | ~15 行 + 告警规则 | 可监控 Agent 健康度，及时发现断连潮 | 低 | 已有 Prometheus 基础设施 |
| **P2** | TaskList 分页加载 + 类型断言兼容 | ~80 行 | 大积压场景启动不阻塞 | 中，需 mock 适配 | 接口扩展模式 |
| **P2** | 事件驱动唤醒 + 100ms 兜底双触发 | ~30 行 | 平均调度延迟从 ~50ms 降到 <1ms | 低 | 需新增 wakeup channel |
| **P2** | DB 乐观锁加固 singleflight 跨实例 | ~60 行 | HA 部署下并发刷新冲突率下降 | 中，需 DB schema 变更 | `users` 表加 version 字段 |
| **P3** | FIFO 链表 → 堆（heap）支持多优先级 | ~150-200 行 | 支持按仓库权重/紧急度调度 | 中，需全面回归测试 | 需配合 `model.Task` 加 priority 字段 |
| **P3** | KickAgentWorkers 同时回滚 running task | ~50 行 | task 悬空时间从 60s → <1s | 中，需处理 Agent 侧竞态 | 需 gRPC 侧配合 |
| **P4** | JWT HookToken 加 exp + jti 防重放 | ~100 行 + DB 表 | 凭证安全等级提升 | 高，涉及 token 格式变更 | 需 token 轮换机制 + 向下兼容 |
| **P4** | 全 Forge 平台 payload HMAC 签名校验 | ~200 行/平台 | 完全阻断 JWT + payload 伪造攻击 | 高，需各 Forge 适配测试 | 各平台 webhook 配置需同步 |

### 10.4 12 项 P0-P4 改造的依赖关系图

```
P0 cause 修复 ──────┐
                     ├─→ P1 graceful shutdown KickAgentWorkers(-1)
P0 Ticker 替换 ──────┤                     │
                                          │
P1 Cancel 补偿任务（30s timer + jitter）  │
                                          │
P1 Prometheus 指标                        │
                                          │
P2 TaskList 分页 ─────┐                    │
                       ├─→ P3 running task 回滚
P2 事件驱动唤醒 ──────┤                    │
                                          │
P2 DB 乐观锁 singleflight                 │
                                          │
P3 FIFO → heap ───────────────────────────┘
     │
     └─→ 调度公平性（多优先级后才有意义）

P3 running task 回滚
     │
     └─→ P4 task 迁移协议（长期演进）

P4 JWT exp + jti ─── 独立，与其他改造无依赖
P4 全 Forge 签名 ─── 独立，按平台逐一落地
```

**强依赖对（必须按顺序）**：

| 前置 | 依赖项 | 原因 |
|------|--------|------|
| P0 cause 修复 | P1 KickAgentWorkers(-1) | 修复后 Agent 才能区分「被踢走」和「网络断开」，否则重连逻辑无差别 |
| P3 running task 回滚 | P2 TaskList 分页 | 回滚逻辑需遍历 running map，需确保启动时 task 加载不阻塞 |
| P4 task 迁移协议 | P3 running task 回滚 | 优雅移交是强制回滚的超集，先实现强制再做优雅 |

**可并行**：
- P0 cause 修复 || P0 Ticker 替换（完全无关）
- P1 Cancel 补偿任务 || P1 Prometheus 指标（可并行开发）
- P2 DB 乐观锁 || P2 事件驱动唤醒（一个在 forge/refresh.go，一个在 queue/fifo.go）
- P4 JWT 防重放 || P4 全 Forge 签名（安全改造互不影响）

### 10.5 task 迁移协议兼容性设计

**目标**：shutdown 或 Agent 被踢时，将 running task 从源 Agent 转移到目标 Agent，无需重新拉取镜像、重新 clone 代码。

**兼容性分层（老 Agent 新 Server、新 Agent 老 Server、混合部署）**：

| 场景 | 兼容策略 | 实现方式 |
|------|---------|---------|
| **新 Server + 老 Agent** | 降级为「Kick + 超时重入队」 | gRPC `TransferTask` 方法未实现，老 Agent 收到 `Unimplemented`，Server 走旧逻辑 |
| **老 Server + 新 Agent** | Agent 忽略迁移信号，按断连处理 | Agent 检测不到 `TransferTask` 调用，维持现有行为 |
| **新 Server + 新 Agent** | 完整迁移流程 | 三步走握手 |
| **混合版本 Agent** | 逐 Agent 协商能力位 | `RegisterAgent` 时上报 `supports_task_transfer` 标志位 |

**三步走迁移握手**（gRPC protobuf 增量设计）：

```protobuf
// 版本 1：现有协议
rpc Next(NextRequest) returns (NextResponse);
rpc Wait(WaitRequest) returns (WaitResponse);

// 版本 2：新增迁移方法（向后兼容）
message TransferTaskRequest {
  string task_id = 1;
  string target_agent_id = 2;
  int64 deadline_extension_seconds = 3;  // 给迁移过程的额外时间
  bytes checkpoint = 4;  // 源 Agent 可选：当前 step 进度快照
}
message TransferTaskResponse {
  enum Status {
    ACCEPTED = 0;       // 目标 Agent 接受了任务
    REJECTED_BUSY = 1;  // 目标 Agent 已满，换一个
    REJECTED_MISMATCH = 2;  // 能力不匹配（如标签/平台不符）
  }
  Status status = 1;
  string reason = 2;
}
rpc TransferTask(TransferTaskRequest) returns (TransferTaskResponse);

// Agent 注册时上报能力位
message AgentInfo {
  // ... 已有字段
  bool supports_task_transfer = 15;  // 新增字段，老 Agent 默认为 false
}
```

**与现有机制的协同**：
1. 迁移失败（如目标 Agent 拒绝、超时未响应）——自动回退到「Kick + 60s deadline 重入队」旧逻辑
2. 迁移成功后，原 Agent 的 running entry 直接转给新 Agent ID，deadline 延长 60s 做交接
3. 迁移中的 task 持久化队列仍由 `WithTaskStore` 管理，双写保障

### 10.6 核心设计哲学总结

Woodpecker 的调度架构体现了三个明确的取舍：

1. **简单优先于完美**：用链表而非堆、用 singleflight 而非分布式锁、用 100ms 轮询而非事件驱动——在功能可用的前提下尽量减少复杂度
2. **持久化兜底而非强一致**：内存队列 + DB 持久化双层，崩溃恢复靠全量重放而非增量同步，牺牲一致性边界换取实现简单
3. **Best-effort 而非硬保证**：取消靠信号通知、踢出靠超时回收、关闭靠连接硬断——把「最终正确」放在「及时准确」之前

这些取舍在中小规模部署下表现良好，但在大规模 HA 部署（多副本 + 数千并发 task）场景下，需要按上述路线图逐步加固。
