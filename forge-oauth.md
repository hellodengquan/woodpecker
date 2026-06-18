# Woodpecker CI 对接 Forge 的 OAuth 代码路径梳理

## 1. 整体架构概览

Woodpecker CI 通过 **Forge 抽象层** 对接多种代码托管平台（GitHub、GitLab、Gitea、Forgejo、Bitbucket、Bitbucket DataCenter），实现统一的 OAuth 认证、仓库管理和 Webhook 处理。核心设计为：

- 一个 `Forge` 接口定义所有平台行为
- 一个可选的 `Refresher` 接口处理 Token 刷新
- 一个 `ForgeManager` 管理多 Forge 实例的生命周期与缓存
- 每个用户和仓库都绑定一个 `ForgeID`，实现多 Forge 共存

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Frontend   │────▶│  Router/API  │────▶│  ForgeManager   │
│  (Login.vue)│     │  (/authorize)│     │  (cache+lookup) │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                      │
                    ┌──────▼───────┐       ┌──────▼──────┐
                    │  HandleAuth  │       │ Forge Interface│
                    │  (login.go)  │       │  (forge.go)   │
                    └──────┬───────┘       └──────┬──────┘
                           │                      │
              ┌────────────┼────────────┬─────────┼──────────┐
              ▼            ▼            ▼         ▼          ▼
         ┌────────┐  ┌────────┐  ┌────────┐ ┌────────┐ ┌────────────┐
         │ GitHub │  │ GitLab │  │ Gitea  │ │ Forgejo│ │ Bitbucket  │
         └────────┘  └────────┘  └────────┘ └────────┘ │ + BDC      │
                                                      └────────────┘
```

---

## 2. OAuth 入口

### 2.1 前端发起

**文件**: `web/src/views/Login.vue` + `web/src/compositions/useAuthentication.ts`

登录页面加载时，前端调用 `getForges()` API 获取已配置的 Forge 列表，为每个 Forge 渲染一个登录按钮。用户点击按钮后：

```typescript
// useAuthentication.ts:9
authenticate(forgeId?: number) {
  window.location.href = `${useConfig().rootPath}/authorize?${forgeId !== undefined ? `forge_id=${forgeId}` : ''}`;
}
```

浏览器跳转到 `/authorize?forge_id=<id>`，进入后端 OAuth 流程。

### 2.2 后端路由注册

**文件**: `server/router/router.go:61-65`

```go
auth := base.Group("/authorize")
{
    auth.GET("", api.HandleAuth)
    auth.POST("", api.HandleAuth)
}
```

GET 和 POST 均由 `HandleAuth` 处理——首次请求（无 code）走 GET，回调（带 code）可能走 GET 或 POST。

---

## 3. OAuth 回调处理

**文件**: `server/api/login.go`

`HandleAuth` 函数是整个 OAuth 流程的核心，同时处理"发起授权"和"回调确认"两个阶段：

### 3.1 阶段一：发起授权（无 code）

1. 从请求中读取 `forge_id` 参数（默认为 1，即主 Forge）
2. 生成一个 JWT **state token**，类型为 `OAuthStateToken`，有效期 5 分钟，内含 `forge-id` 字段
3. 通过 `ForgeManager.ForgeByID(forgeID)` 获取 Forge 实例
4. 调用 `forge.Login(ctx, &OAuthRequest{Code: "", State: state})`
5. 各 Forge 实现的 `Login` 方法在 `Code` 为空时，返回 `(nil, redirectURL, nil)`——即 OAuth 授权页 URL
6. 后端将用户 303 重定向到该 URL

### 3.2 阶段二：回调确认（有 code + state）

1. 从请求中读取 `code` 和 `state`
2. **验证 state token**：解析 JWT，提取 `forge-id`，确保请求合法且未过期
3. 通过 `ForgeManager.ForgeByID(forgeID)` 获取对应 Forge 实例
4. 调用 `forge.Login(ctx, &OAuthRequest{Code: code, State: state})`
5. 各 Forge 实现的 `Login` 方法在 `Code` 非空时：
   - 用 `config.Exchange(ctx, code)` 将 authorization code 换取 access token
   - 用 access token 调用 Forge API 获取用户信息（登录名、邮箱、头像等）
   - 返回 `(*model.User, redirectURL, nil)`
6. **组织过滤**：如果配置了 `WOODPECKER_ORGS`，检查用户是否属于允许的组织
7. **用户查找/创建**：
   - 先按 `ForgeRemoteID` 查找，再按 `Login` 查找
   - 新用户：检查是否允许自注册，创建用户记录及关联的 Org
   - 已有用户：更新 AccessToken、RefreshToken、Expiry、Email、Avatar 等
8. **生成会话 token**：创建 `SessToken` 类型的 JWT，用 `user.Hash` 签名
9. **同步仓库权限**：调用 `updateRepoPermissions` 从 Forge 拉取用户的所有仓库并更新权限
10. 设置 `user_sess` Cookie，重定向到首页

### 3.3 错误处理

回调过程中若 Forge 返回错误（如 `error=access_denied`），后端会将错误参数转发到 `/login?error=xxx`，前端 Login.vue 会显示对应的错误信息。

---

## 4. Forge 抽象层

### 4.1 Forge 接口

**文件**: `server/forge/forge.go`

```go
type Forge interface {
    Name() string
    URL() string
    Login(ctx context.Context, r *types.OAuthRequest) (*model.User, string, error)
    Teams(ctx context.Context, u *model.User, p *model.ListOptions) ([]*model.Team, error)
    Repo(ctx context.Context, u *model.User, remoteID model.ForgeRemoteID, owner, name string) (*model.Repo, error)
    Repos(ctx context.Context, u *model.User, p *model.ListOptions) ([]*model.Repo, error)
    File(ctx context.Context, u *model.User, r *model.Repo, b *model.Pipeline, fileName string) ([]byte, error)
    Dir(ctx context.Context, u *model.User, r *model.Repo, b *model.Pipeline, dirName string) ([]*types.FileMeta, error)
    Status(ctx context.Context, u *model.User, r *model.Repo, b *model.Pipeline, p *model.Workflow) error
    Netrc(u *model.User, r *model.Repo) (*model.Netrc, error)
    Activate(ctx context.Context, u *model.User, r *model.Repo, link string) error
    Deactivate(ctx context.Context, u *model.User, r *model.Repo, link string) error
    Branches(ctx context.Context, u *model.User, r *model.Repo, p *model.ListOptions) ([]string, error)
    BranchHead(ctx context.Context, u *model.User, r *model.Repo, branch string) (*model.Commit, error)
    PullRequests(ctx context.Context, u *model.User, r *model.Repo, p *model.ListOptions) ([]*model.PullRequest, error)
    Hook(ctx context.Context, r *http.Request) (*model.Repo, *model.Pipeline, error)
    OrgMembership(ctx context.Context, u *model.User, org string) (*model.OrgPerm, error)
    Org(ctx context.Context, u *model.User, org string) (*model.Org, error)
}
```

其中与 OAuth 直接相关的是 `Login` 方法，其契约如下：
- 第一次调用（`Code` 为空）：返回 `(nil, redirectURL, nil)`，让调用方重定向用户
- 第二次调用（`Code` 非空）：返回 `(user, redirectURL, nil)`，user 中包含 OAuth token

### 4.2 Refresher 接口

**文件**: `server/forge/refresh.go`

```go
type Refresher interface {
    Refresh(ctx context.Context, u *model.User) (bool, error)
}
```

这是可选接口，只有支持 token 过期和刷新的 Forge 才需要实现。当前实现情况：

| Forge | 实现 Refresher | 原因 |
|-------|---------------|------|
| GitHub | ✅ | GitHub OAuth App 可能发放 refresh token |
| GitLab | ✅ | GitLab token 有过期时间 |
| Gitea | ✅ | Gitea token 可能过期 |
| Forgejo | ✅ | Forgejo token 可能过期 |
| Bitbucket | ✅ | Bitbucket token 有过期时间 |
| Bitbucket DC | ❌ | 使用 Git 机器账号 + OAuth，不实现 Refresher |

### 4.3 OAuth 请求类型

**文件**: `server/forge/types/oauth.go`

```go
type OAuthRequest struct {
    Code  string
    State string
}
```

### 4.4 Forge 数据模型

**文件**: `server/model/forge.go`

```go
type Forge struct {
    ID                int64          `xorm:"pk autoincr 'id'"`
    Type              ForgeType      `xorm:"VARCHAR(250)"`        // github/gitlab/gitea/forgejo/bitbucket/bitbucket-dc/addon
    URL               string         `xorm:"VARCHAR(500) 'url'"`  // Forge API URL
    OAuthClientID     string         `xorm:"VARCHAR(250) 'oauth_client_id'"`
    OAuthClientSecret string         `xorm:"VARCHAR(250) 'oauth_client_secret'"`
    SkipVerify        bool           // 是否跳过 TLS 验证
    OAuthHost         string         `xorm:"VARCHAR(250) 'oauth_host'"` // 用户侧 OAuth URL（可不同于 URL）
    AdditionalOptions map[string]any // 扩展配置（如 GitHub 的 merge-ref、Bitbucket DC 的 git-username 等）
}
```

`OAuthHost` 字段允许将 OAuth 授权页面指向与 API 不同的域名，适用于 Forge API 通过内网访问但 OAuth 需要公网访问的场景。

### 4.5 Forge 工厂与注册

**文件**: `server/forge/setup/setup.go`

`Forge(forge *model.Forge)` 函数根据 `forge.Type` 分发到各 Forge 的 `New()` 构造函数，将数据库中的 `model.Forge` 记录转换为 `forge.Forge` 接口实例。

### 4.6 ForgeManager

**文件**: `server/services/manager.go`

`ForgeManager` 负责管理所有 Forge 实例，提供按 ID、Repo、User 查找 Forge 的能力：

```go
type Manager interface {
    ForgeFromRepo(repo *model.Repo) (forge.Forge, error)
    ForgeFromUser(user *model.User) (forge.Forge, error)
    ForgeByID(forgeID int64) (forge.Forge, error)
    // ...其他服务方法
}
```

内部使用 **TTL 缓存**（10 分钟过期，不自动续期）避免反复从数据库加载和构造 Forge 实例。

查找流程：
1. 检查缓存是否命中且未过期
2. 缓存未命中时，从数据库加载 `model.Forge`
3. 调用 `setupForge(forgeModel)` 构造 `forge.Forge` 实例
4. 存入缓存

---

## 5. 各 Forge 的 OAuth 实现

所有 Forge 的 `Login` 方法遵循同一模式，差异在于 OAuth 端点和 API 调用：

### 5.1 GitHub

**文件**: `server/forge/github/github.go`

- OAuth 端点：
  - AuthURL: `{oAuthHost}/login/oauth/authorize`
  - TokenURL: `{url}/login/oauth/access_token`
- Scopes: `user:email`, `read:org`，若非 `OnlyPublic` 则附加 `repo`
- 回调后用 `go-github` SDK 获取用户信息和邮箱列表
- `Netrc` 凭据：login=AccessToken, password=`x-oauth-basic`

### 5.2 GitLab

**文件**: `server/forge/gitlab/gitlab.go`

- OAuth 端点：
  - AuthURL: `{oAuthHost}/oauth/authorize`
  - TokenURL: `{url}/oauth/token`
- Scopes: `api`
- 回调后用 `gitlab-sdk` 获取当前用户
- `Netrc` 凭据：login=`oauth2`, password=AccessToken

### 5.3 Gitea

**文件**: `server/forge/gitea/gitea.go`

- OAuth 端点：
  - AuthURL: `{oAuthHost}/login/oauth/authorize`
  - TokenURL: `{url}/login/oauth/access_token`
- 无 Scope 声明
- 回调后用 `gitea-sdk` 获取用户信息
- `Netrc` 凭据：login=用户名, password=AccessToken

### 5.4 Forgejo

**文件**: `server/forge/forgejo/forgejo.go`

- OAuth 端点：
  - AuthURL: `{oauth2URL}/login/oauth/authorize`
  - TokenURL: `{oauth2URL}/login/oauth/access_token`
- 无 Scope 声明
- 回调后用 `forgejo-sdk` 获取用户信息
- `Netrc` 凭据：login=用户名, password=AccessToken
- 与 Gitea 高度相似，但使用独立的 SDK 和 OAuth2 URL 配置

### 5.5 Bitbucket Cloud

**文件**: `server/forge/bitbucket/bitbucket.go`

- OAuth 端点：
  - AuthURL: `{url}/site/oauth2/authorize`
  - TokenURL: `{url}/site/oauth2/access_token`
- 无 Scope 声明
- 回调后用内部 HTTP 客户端获取用户信息和邮箱
- `Netrc` 凭据：login=`x-token-auth`, password=AccessToken

### 5.6 Bitbucket DataCenter

**文件**: `server/forge/bitbucketdatacenter/bitbucketdatacenter.go`

- 使用 OAuth 2.0 + Git 机器账号混合模式
- 除 OAuth Client ID/Secret 外，还需配置 `git-username` 和 `git-password` 用于克隆
- 不实现 `Refresher` 接口

---

## 6. Access Token 的复用

Access Token 存储在 `model.User` 中，是 Woodpecker 与 Forge 交互的核心凭据，在多个场景中被复用。

### 6.1 User 模型中的 Token 字段

**文件**: `server/model/user.go`

```go
type User struct {
    ID            int64          `xorm:"pk autoincr 'id'"`
    ForgeID       int64          `xorm:"forge_id UNIQUE(forge_id)"`
    ForgeRemoteID ForgeRemoteID  `xorm:"forge_remote_id UNIQUE(forge_id)"`
    AccessToken   string         `xorm:"TEXT 'access_token'"`
    RefreshToken  string         `xorm:"TEXT 'refresh_token'"`
    Expiry        int64          `xorm:"expiry"`          // Unix 时间戳
    // ...其他字段
}
```

### 6.2 Token 复用场景

#### 场景 A：用户发起的 API 请求（中间件自动刷新）

**文件**: `server/router/middleware/token/token.go`

每个 HTTP 请求经过 `token.Refresh` 中间件：

```go
func Refresh(c *gin.Context) {
    user := session.User(c)
    if user != nil {
        _forge, _ := server.Config.Services.Manager.ForgeFromUser(user)
        forge.Refresh(c, _forge, store.FromContext(c), user)
    }
    c.Next()
}
```

如果用户已登录，中间件会在处理请求前检查并刷新 Token。

#### 场景 B：Webhook 处理（Hook 中刷新）

**文件**: `server/forge/github/github.go`、`server/forge/gitlab/gitlab.go` 等

当 Webhook 到来时，系统需要用仓库关联用户的 Token 调用 Forge API（如获取 PR 变更文件）。流程：

1. 从 context 获取 store
2. 通过 `store.GetRepoNameFallback()` 获取仓库
3. 通过 `store.GetUser(repo.UserID)` 获取仓库关联的用户
4. 调用 `forge.Refresh(ctx, forge, store, user)` 刷新 Token
5. 使用刷新后的 `user.AccessToken` 创建 Forge API 客户端

示例（GitHub `loadChangedFilesFromPullRequest`）：

```go
user, _ := _store.GetUser(repo.UserID)
forge.Refresh(ctx, c, _store, user)
gh, _ := c.newClientToken(ctx, user.AccessToken)
files, _, _ := gh.PullRequests.ListFiles(ctx, repo.Owner, repo.Name, pull.GetNumber(), opts)
```

#### 场景 C：Netrc 凭据生成（克隆私有仓库）

**文件**: 各 Forge 的 `Netrc()` 方法

Agent 克隆私有仓库时使用 OAuth Access Token 作为凭据：

| Forge | Netrc Login | Netrc Password |
|-------|-------------|----------------|
| GitHub | AccessToken | `x-oauth-basic` |
| GitLab | `oauth2` | AccessToken |
| Gitea | 用户名 | AccessToken |
| Forgejo | 用户名 | AccessToken |
| Bitbucket | `x-token-auth` | AccessToken |

#### 场景 D：无直接用户时的回退查找

**文件**: `server/forge/common/utils.go`

`UserToken()` 函数在当前请求无用户时，从仓库的 `UserID` 回退查找：

```go
func UserToken(ctx context.Context, r *model.Repo, u *model.User) string {
    if u != nil {
        return u.AccessToken
    }
    user, _ := RepoUser(ctx, r)
    return user.AccessToken
}
```

这在 `Branches`、`BranchHead`、`PullRequests` 等操作中被使用。

### 6.3 Token 刷新机制

**文件**: `server/forge/refresh.go`

```go
func Refresh(ctx context.Context, forge Forge, _store store.Store, user *model.User)
```

核心逻辑：

1. **类型断言**：检查 Forge 是否实现了 `Refresher` 接口
2. **过期判断**：如果 Token 距过期超过 30 分钟（`tokenMinTTL = 1800`），不做任何操作
3. **并发去重**：使用 `singleflight.Group` 确保同一用户的并发刷新请求只执行一次
   - 防止多 goroutine 同时刷新导致的 race condition
   - 特别重要：Forgejo 的 `InvalidateRefreshTokens=true` 模式下 refresh token 只能用一次
4. **刷新结果传播**：执行刷新的 goroutine 将新 Token 通过 `refreshResult` 结构体传播给等待的 goroutine
5. **持久化**：刷新成功后调用 `store.UpdateUser()` 将新 Token 写入数据库

```
Goroutine 1 ─┐
Goroutine 2 ─┤──▶ singleflight ──▶ Refresher.Refresh() ──▶ store.UpdateUser()
Goroutine 3 ─┘         │                                        │
                        ▼                                        ▼
              refreshResult{AccessToken,              DB: users.access_token
                RefreshToken, Expiry}                   .refresh_token
                                                        .expiry
```

### 6.4 Token 生命周期总结

```
┌─────────────────────────────────────────────────────────────┐
│                    Access Token 生命周期                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. OAuth Login                                             │
│     forge.Login() ──▶ 获取 AccessToken/RefreshToken/Expiry │
│     store.UpdateUser() ──▶ 持久化到数据库                     │
│                                                             │
│  2. 请求时自动刷新（中间件）                                   │
│     token.Refresh 中间件 ──▶ forge.Refresh()                │
│     ──▶ singleflight 去重 ──▶ Refresher.Refresh()          │
│     ──▶ store.UpdateUser()                                  │
│                                                             │
│  3. Webhook/Hook 时刷新                                      │
│     forge.Refresh() ──▶ 同上流程                              │
│                                                             │
│  4. 复用场景                                                 │
│     ├─ API 调用（u.AccessToken）                              │
│     ├─ Netrc 克隆（各 Forge 格式不同）                         │
│     ├─ Webhook 处理中的二次 API 调用                           │
│     └─ common.UserToken() 回退查找                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 深度分析：singleflight 调度、Token 继承与刷新风暴

> 本章节基于 `server/forge/refresh.go`、`server/forge/refresh_test.go`、`server/router/middleware/token/token.go`、`server/services/utils/http.go`、`server/pipeline/create.go` 及各下游 API handler 的实际代码逐行核对得出结论。

---

### 7.1 singleflight 如何调度等待者

**核心代码**: `server/forge/refresh.go:70-86`

```go
key := fmt.Sprintf("refresh-%d", user.ID)
result, err, _ := refreshGroup.Do(key, func() (any, error) {
    userUpdated, err := refresher.Refresh(ctx, user)
    // ...构造 refreshResult 并返回
})
```

#### 调度机制详解

使用 `golang.org/x/sync/singleflight` 包，按 **用户维度**（`key = "refresh-<userID>"`）做去重：

1. **Winner 选举**：当多个 goroutine（通常来自不同的并发 HTTP 请求 / Webhook / Pipeline 调度）在同一时刻以相同 key 调用 `Do` 时，**第一个到达的 goroutine 成为 Winner**，负责真正执行传入的闭包函数（即调用 `refresher.Refresh(ctx, user)` + `store.UpdateUser(user)`）。

2. **Waiter 阻塞**：其他 goroutine 在 `Do()` 调用内部被阻塞——挂在 singleflight 内部 `call` 结构的 `wg.Wait()` 上，直到 Winner 执行完毕。

3. **结果广播**：Winner 执行完闭包后，singleflight 将返回值 `(v interface{}, err error)` 通过 `call.val` / `call.err` 字段共享给 **所有 Waiter**，并唤醒所有阻塞的 Waiter，让它们从 `Do()` 调用中返回。

4. **共享标识被忽略**：`Do` 的第三个返回值 `shared bool` 被用 `_` 显式丢弃（第 71 行）。该标识本来可以区分"我是 Winner（false）"还是"我是 Waiter（true）"，但当前代码不需要此信息，因为 Winner 和 Waiter 都走完全相同的后续逻辑（`refreshResult → user.AccessToken/RefreshToken/Expiry` 字段复制）。

#### 调用时序图

```
t0 ─── Goroutine A (HTTP /repos) ───► refreshGroup.Do("refresh-42", fn)  ──► 成为 Winner，开始执行 fn
       Goroutine B (HTTP /pipeline) ─► refreshGroup.Do("refresh-42", fn)  ──► Waiter，阻塞在 wg.Wait()
       Goroutine C (Webhook PR)    ─► refreshGroup.Do("refresh-42", fn)  ──► Waiter，阻塞在 wg.Wait()

t1 ─── Winner A 完成: refresher.Refresh() → store.UpdateUser() → 构造 &refreshResult{...}
       singleflight 内部 wg.Done()

t2 ─── Goroutine A 从 Do() 返回
       Goroutine B 从 Do() 返回，拿到 A 的 result & err
       Goroutine C 从 Do() 返回，拿到 A 的 result & err

t3 ─── 所有 3 个 goroutine 都执行 95-99 行：refreshResult → 各自 user 对象字段赋值
```

**测试验证**: `server/forge/refresh_test.go:120-168` `TestRefresh_ConcurrentRefreshSerialized`
- 10 个 goroutine 并发调用 `forge.Refresh`（同 user.ID=42）
- `refreshCount` 原子计数器的值最终为 1（只执行了一次真正的刷新）
- 全部 10 个独立的 `*model.User` 对象最终都包含新 token

---

### 7.2 旧请求（Waiter）是否能继承新 token

**结论：✅ 完全可以继承，并且是显式设计的。**

#### 传播机制——两级拷贝

```
Level 1 (Winner 内部闭包内):
  refresher.Refresh(ctx, user_A)
    ↳ user_A.AccessToken = "new-access-token"   // 修改 Winner 自己的 user 对象
    ↳ user_A.RefreshToken = "new-refresh-token"
    ↳ user_A.Expiry = new_expiry

  store.UpdateUser(user_A)                      // 持久化 Winner 对象的新 token 到 DB

  return &refreshResult{                         // 构造"独立副本"放入 singleflight 结果通道
      AccessToken:  user_A.AccessToken,
      RefreshToken: user_A.RefreshToken,
      Expiry:       user_A.Expiry,
  }

Level 2 (Do() 返回后，所有 goroutine 共享):
  result, err, _ := refreshGroup.Do(...)         // A、B、C 都拿到同一个 *refreshResult 指针

  if r, ok := result.(*refreshResult); ok {
      user.AccessToken = r.AccessToken            // A: 从 refreshResult 拷回自己的 user（冗余）
      user_A.RefreshToken = r.RefreshToken        // B: Waiter，通过 refreshResult 继承新 token ✓
      user_B.Expiry       = r.Expiry              // C: Waiter，通过 refreshResult 继承新 token ✓
  }                                               // user_C...
```

#### 为什么需要 `refreshResult` 中间结构体？

**关键代码注释** (`refresh.go:92-94`):

> waiting goroutines have their own `*model.User` copies that weren't passed to `refresher.Refresh()`

每个调用方（goroutine）都持有自己独立的 `*model.User` 指针副本（来自不同的中间件调用路径，如 session.User() 返回、store.GetUser() 查询）。Winner 的闭包只会修改 **Winner 自己传入的那份** user 对象。如果没有 `refreshResult`，Waiter 从 `Do()` 返回后，自己持有的 user 对象仍是旧值。

`refreshResult` 起到了"跨 goroutine 广播"的作用：把刷新结果从 Winner 传播给所有 Waiter。

---

### 7.3 刷新失败时旧请求如何降级/返回 401

**结论：❌ 没有 401 返回，也没有中间件层面的降级；刷新失败是静默的，请求继续执行，由下游 Forge API 调用自然失败并被包装成业务错误（通常是 500）。**

#### 失败时的完整执行路径

**核心代码**: `server/forge/refresh.go:87-89`

```go
if err != nil {
    log.Error().Err(err).Msgf("refresh oauth token of user '%s' failed", user.Login)
    return  // ← 直接 return，不修改 user 对象，不 panic，不返回 error 给调用方
}
```

注意：`forge.Refresh()` 签名是 **无返回值**（`func Refresh(...)`），调用方无法知道刷新是否成功。失败只表现为一条 error log。

#### 各路径的具体表现

| 调用场景 | 中间件/入口 | 刷新失败后 | 最终用户可见结果 |
|----------|------------|-----------|----------------|
| HTTP API（用户浏览器请求） | `token.Refresh` 中间件 (`token/token.go:28-41`) | 中间件始终调用 `c.Next()`，不 `AbortWithError(401)` | 下游 handler 用过期 token 调 Forge API → Forge 返回 401/403 → handler 包装成 **500** 业务错误返回。例：`api/repo.go:81 "Could not fetch repository from forge."` |
| Pipeline 创建（Webhook 触发） | `pipeline.Create()` 内调用 (`pipeline/create.go:62`) | 刷新静默失败，继续执行后续代码 | ① `configService.Fetch()` 取配置时若拿到部分结果 → **唯一降级点**：使用旧 pipeline 配置（`pipeline/create.go:87-93`）<br>② 否则 → pipeline 被标为 error，错误信息写入 DB |
| Webhook 内部二次 API 调用 | 各 Forge Hook 内，如 GitHub `loadChangedFilesFromPullRequest` | 刷新静默失败，继续创建 Forge 客户端 | Forge API 返回 401 → PR 变更文件无法获取 → 可能导致 step condition 过滤失败或 pipeline 被错误触发 |

#### 测试验证: `TestRefresh_ConcurrentRefreshError` (`refresh_test.go:170-205`)

- 模拟刷新返回错误 `fmt.Errorf("token was already used")`
- 全部 5 个 goroutine 的 user 对象 token 保持原值 `"old-access-token"`
- `mockStore.AssertNotCalled(t, "UpdateUser", mock.Anything)` 验证数据库持久化未被调用

#### 为什么不直接返回 401？

当前设计隐含的假设是：
1. **Token 过期 ≠ 会话失效**：用户的 Woodpecker 会话 Cookie（`user_sess`）仍然有效，只是需要一个新的 Forge Access Token
2. **刷新可能是临时故障**：如 Forge 暂时不可用，几秒后下一次请求可能成功
3. **部分操作不依赖 Forge**：如查看历史 pipeline 记录、查看用户信息等，根本不需要访问 Forge，强行 401 会误杀合法请求

**潜在问题**：刷新持续失败（如 refresh token 已被吊销/过期）时，用户不会被踢回登录页，而是持续看到 500 错误。

---

### 7.4 Forge 速率限制（429）下的刷新风暴与回退路径

**结论：⚠️ 不存在排队处理、熔断器或显式回退路径；遇到速率限制是直接报错，且串行请求可能形成刷新风暴，进一步加剧 Forge 侧的 429。**

#### 逐层检查现有机制

##### 第 1 层：`forge.Refresh()` 内部 — 完全无防护

```go
// server/forge/refresh.go:70-100
// ✗ 无指数退避（exponential backoff）
// ✗ 无重试（refresh 内部不 retry）
// ✗ 无熔断器（circuit breaker）—— 连续 N 次失败后停止 M 秒尝试
// ✗ 无失败冷却期标记（如 "该用户最近 30s 刷新失败，跳过"）
// ✓ 只有 singleflight：同一用户 **并发** 去重，但 **串行** 请求不受限
```

**问题场景**：
```
t=0s   Request-1 到达 → refresh 失败（Forge 429）→ 记日志，请求继续走下游（500）
t=0.5s Request-2 到达 → token 仍过期 → 再次进入 refreshGroup.Do → 再次刷新 → 再次 429
t=1s   Request-3 到达 → 同上 → 429
...（每次都 hammer Forge token endpoint）
```

因为 `Expiry` 字段从未被更新（刷新失败不写 DB），每次新请求进入时，`time.Now() > Expiry - 1800` 的条件都成立，从而无限次尝试刷新。

##### 第 2 层：通用 HTTP 重试 — **对 429 无效**

**文件**: `server/services/utils/http.go:236-240`

```go
func isRetryableStatusCode(statusCode int) bool {
    // Retry on server errors (5xx) only
    return statusCode >= http.StatusInternalServerError &&
           statusCode < http.StatusNetworkAuthenticationRequired
}
```

- 仅 retry **5xx**
- 429（`http.StatusTooManyRequests` = 429）是 4xx，在 `http.go:176-178` 被标记为 `backoff.Permanent(err)`，**不 retry**
- 且这条 retry 路径属于通用 HTTP 客户端，OAuth token endpoint 的刷新走的是各 Forge SDK（如 `oauth2.Config.Exchange`），根本不经过 utils/http.go 的 `Send()` 函数

##### 第 3 层：各 Forge 内部 Refresh 实现 — 无额外防护

如 GitLab Refresh (`gitlab.go`)、GitHub Refresh (`github.go`) 等直接调用各自 SDK 的 `TokenSource.Token()`，没有包装任何 rate limit 处理。

##### 第 4 层：配置文件拉取 — 唯一存在重试的降级路径

**文件**: `server/services/config/forge.go:65-73`

```go
for i := 0; i < int(f.retryCount); i++ {
    files, err = ffc.fetch(ctx, strings.TrimSpace(repo.Config))
    if err == nil { break }
}
```

这是 **Pipeline 配置文件获取**（而非 OAuth token 刷新），重试次数由 `WOODPECKER_FORGE_RETRY`（默认 3）控制。同样只针对 Forge API 文件读取端点，不涉及 OAuth token endpoint。

#### 回退路径总结表

| 故障场景 | 是否排队 | 是否重试 | 是否降级 | 实际表现 |
|---------|---------|---------|---------|---------|
| 同一用户 **并发** 请求同时刷新（如多个浏览器 tab） | ✅ singleflight 排队，只执行 1 次 | ✗ | ✗ （失败则全部失败） | 由 Winner 执行 + 结果广播 |
| 同一用户 **串行** 请求（间隔>刷新耗时） | ✗ 不排队，各自独立触发 | ✗ | ✗ | 反复执行 refresh，持续 429 |
| **不同用户** 同时到 TTL 期 | ✗ 完全不排队（key 含 userID） | ✗ | ✗ | 并发 N 个刷新，可能触发 Forge 全局 rate limit |
| Forge 返回 500/502/503（token endpoint） | ✗ （OAuth2.Exchange 本身通常不 retry） | 视 SDK 而定 | ✗ | 单次失败，请求继续 |
| Forge 返回 429（token endpoint） | ✗ | ✗ | ✗ | 立即失败，下一个请求继续踩坑 |
| 配置文件拉取失败（Pipelines） | ✗ | ✅ `retryCount` 次指数退避 | ✅ （部分结果时用旧配置） | `services/config/forge.go` 与 `pipeline/create.go:87-93` |

#### 应对刷新风暴的缺失机制清单（当前代码不存在）

1. **失败冷却期**：刷新失败后 N 秒内不再尝试（可通过 `user.ForgeID + user.ID` 在内存中记一个"失败直到"的时间戳）
2. **熔断器（Circuit Breaker）**：连续 M 次失败后自动打开，冷却期内所有刷新跳过
3. **429 Retry-After 尊重**：解析 Forge 响应的 `Retry-After` header，在该时间之前不再次尝试
4. **持久化最近刷新时间**：即使刷新失败，也把 `Expiry` 向后推移一小段时间（如 60s），避免下一次请求立即再次触发
5. **刷新失败时的前端反馈**：中间件检测到刷新持续失败时，将特定 header 写入响应，前端据此引导用户重新授权

---

## 8. 深度分析（续）：OAuth Token Endpoint 重试耗尽后的响应路径

> 本章节基于 `golang.org/x/oauth2 v0.36.0` 源码行为、各 Forge 的 `Refresh` 实现、`server/router/middleware/token/token.go`、`server/api/hook.go`、`server/pipeline/create.go`、`web/src/lib/api/client.ts`、`web/src/App.vue`、`web/src/router.ts` 的代码逐行核对。

---

### 8.1 OAuth TokenSource.Token() 内部：是否存在重试？

所有 Forge 的 `Refresh` 方法都遵循同一模式（以 GitLab 为例，`server/forge/gitlab/gitlab.go:160-179`）：

```go
func (g *GitLab) Refresh(ctx context.Context, user *model.User) (bool, error) {
    config, oauth2Ctx := g.oauth2Config(ctx)
    source := config.TokenSource(oauth2Ctx, &oauth2.Token{
        AccessToken:  user.AccessToken,
        RefreshToken: user.RefreshToken,
        Expiry:       time.Unix(user.Expiry, 0),
    })
    token, err := source.Token()   // ← 实际发起 HTTP POST 到 Forge token endpoint
    // ...
}
```

#### oauth2 库内部 `Token()` 的行为（`golang.org/x/oauth2 v0.36.0`）

`oauth2.TokenSource.Token()` 在 token 过期时会调用内部 `tokenRefresher.RefreshToken()`，底层走 `oauth2.RetrieveToken()`，其核心特征：

| 特征 | 行为 |
|------|------|
| **HTTP 客户端** | 使用 `context.Context` 关联的 `http.Client`（即 Forge 配置中带 TLS skipVerify 的那个），**不经过** `server/services/utils/http.go` 的 backoff 重试逻辑 |
| **重试次数** | **无任何重试**。HTTP 请求是单发的：一次 `http.Client.Do()`，成功返回 token，失败直接把 error 往上抛 |
| **超时控制** | 完全依赖传入的 `ctx`。在 Woodpecker 中：<br>• HTTP API 请求：由 gin 的 `c.Request.Context()` 传入，默认无显式超时，仅依赖 `http.Server.ReadTimeout`/`WriteTimeout`（未在 Woodpecker 中显式配置，默认无限制）<br>• Webhook / Pipeline：同上，无显式 `context.WithTimeout` 包裹 token 刷新<br>• **唯一有超时的是 config fetcher**（`services/config/forge.go:90`），但那是文件读取端点不涉及 token endpoint |
| **错误传播** | 原始错误原样返回，无封装、无降级标记。典型返回值如 `&oauth2.RetrieveError{Response: *http.Response, Body: []byte}`，内部包含 HTTP 状态码和响应体 |
| **429 处理** | oauth2 库本身不识别 429、不解析 `Retry-After`、不 backoff。429 与 400、401、500 同样对待：直接 `return nil, err` |

#### 结论 1：OAuth Token 刷新是 **单次同步 HTTP POST，无重试、无内建超时、无 backoff**。

---

### 8.2 调用方最终拿到什么？—— 401 自动重登 / 5xx 透传 / 死循环？

按三条独立调用路径分别追踪：

#### 路径 A：用户浏览器发起的 HTTP API 请求（最常见场景）

完整调用链：

```
浏览器 fetch() ──► gin.Engine
  ├─ session.SetUser() 中间件         (session/user.go:42-75)
  │    └─ 解析 user_sess Cookie → 查 DB 取 *model.User
  │       若 session 本身失效（过期 JWT）→ c.Next() 继续（user=nil）
  │
  ├─ token.Refresh() 中间件           (token/token.go:28-41)
  │    └─ forge.Refresh(c, _forge, store, user)   (refresh.go:58-101)
  │         ├─ 类型断言 Refresher
  │         ├─ 过期判断（TTLX）
  │         ├─ singleflight.Do(key, fn)
  │         │    └─ refresher.Refresh() → oauth2.TokenSource.Token()
  │         │         └─ Forge 返回 401/429/500 → err != nil
  │         ├─ err != nil 分支:
  │         │    log.Error(...)         ← 仅写日志
  │         │    return                 ← 直接 return，无错误上报，无 c.Abort
  │         └─ user token 保持旧值（过期/无效）不变
  │
  ├─ MustUser / MustAdmin 中间件       (session/user.go:77-121)
  │    └─ user != nil → 通过（因为 session 本身仍有效）
  │
  └─ Handler（如 api/repo.go:49 PostRepo）
       ├─ forge.Refresh(c, _forge, _store, user)    ← 有些 handler 再调一次
       └─ _forge.Repo(c, user, ...) / _forge.Repos(c, user, ...)
            └─ go-github/gitlab-sdk/gitea-sdk 内部 http.Do()
                 └─ Forge API 返回 401 Unauthorized
                      └─ err != nil
                           └─ c.String(http.StatusInternalServerError, "...")
                                ← 实际返回 **500**，不是 401
```

**前端接收**（`web/src/lib/api/client.ts:54-68` + `web/src/App.vue:37-43`）：

```typescript
// client.ts:54-68
if (!res.ok) {
    const error: ApiError = { status: res.status, message: `${res.statusText}: ${resText}` };
    if (this.onerror) { this.onerror(error); }
    throw new Error(message);
}

// App.vue:37-43
apiClient.setErrorHandler((err) => {
  if (err.status === 404) {
    notify({ title: i18n.t('errors.not_found'), type: 'error' });
    return;
  }
  notify({ title: err.message || i18n.t('unknown_error'), type: 'error' });
});
```

关键：**onerror handler 对 401 没有特殊处理**（不跳登录页、不清 session），只做 toast 通知。router 的认证 guard 仅在页面跳转时检查 `isAuthenticated`（来自 localStorage 的 token），不会主动失效。

**最终用户可见表现**：
- 用户页面上弹出一个红底 toast，内容类似 "Internal Server Error: Could not fetch repository from forge."
- 用户仍显示为已登录状态（Navbar 头像、用户名仍在）
- 无自动跳转登录页
- 刷新页面、点击其他页面都会反复触发相同的 500 + toast

**结论 2A：HTTP API 路径下，调用方拿到的是 **业务层封装的 500**，前端仅弹 toast，**不会自动重登，也不会死循环**（每个请求独立触发一次刷新 → 一次失败 → 一次 500）。**

#### 路径 B：Webhook 触发的 Pipeline 创建

完整调用链（`server/api/hook.go:69-213`）：

```
Forge POST /hook ──► PostHook(c)
  ├─ 解析 HookToken → 获取 repo
  ├─ _forge.Hook(c, c.Request)         ← 解析 hook payload（不需要 OAuth token）
  ├─ _store.GetUser(repo.UserID)       ← 取仓库关联用户
  ├─ forge.Refresh(c, _forge, _store, user)
  │    └─ (同路径 A) 失败：仅记日志，静默返回
  ├─ _store.UpdateRepo(repo)
  └─ pipeline.Create(c, _store, repo, pipelineFromForge)   (pipeline/create.go:35)
       ├─ forge.Refresh(ctx, _forge, _store, repoUser)     ← 再调一次刷新
       ├─ configService.Fetch(ctx, _forge, repoUser, repo, pipeline, ...)
       │    └─ _forge.File(c, repoUser, repo, pipeline, ".woodpecker.yml")
       │         └─ Forge API 返回 401 → err != nil
       │              └─ retryCount 次重试（仅对文件读取端点，非 token endpoint）
       │
       ├─ 如果 fetch 返回 (部分结果, err):
       │    └─ 唯一降级点 → "will fallback to old config" (create.go:87-93)
       ├─ 如果 fetch 返回 (nil, err):
       │    └─ updatePipelineWithErr(...) → pipeline.Status = "error"，错误写 DB
       └─ 返回给 Forge: 200 OK（hook 本身处理完成）或 500（DB 失败等）
```

**最终表现**：
- hook 调用向 Forge 返回 200（hook 本身解析成功）
- Woodpecker 内部该 pipeline 被标为 error 状态
- 错误信息保存在 pipeline 记录中，用户在 Web UI 上看到红色失败标记
- 不影响其他 pipeline 和用户

**结论 2B：Webhook 路径下，失败被吸收进 pipeline error 状态，**不对外抛 401/5xx**，无死循环风险。**

#### 路径 C：后台定时任务（cron、membership sync 等）

这些任务直接调 `forge.Refresh()` + Forge API，失败仅记日志并跳过当前任务，由下一次 cron tick 再触发。属于"静默失败 + 延迟重试"模式，非用户可见。

---

### 8.3 上层 Middleware 如何把失败转成用户可见状态

#### 中间件层：token.Refresh 的设计缺陷

`server/router/middleware/token/token.go:28-41`：

```go
func Refresh(c *gin.Context) {
    user := session.User(c)
    if user != nil {
        _forge, err := server.Config.Services.Manager.ForgeFromUser(user)
        if err != nil {
            _ = c.AbortWithError(http.StatusInternalServerError, err)  // ← Forge 查不到才 Abort
            return
        }
        forge.Refresh(c, _forge, store.FromContext(c), user)
        // ← forge.Refresh() 返回后，无论成败都 c.Next()
    }
    c.Next()
}
```

关键观察：
1. **Forge 查不到** 会 `AbortWithError(500)` — 但这不是 token 过期导致的
2. **token 刷新失败**（Forge 返回 401/429/500）：`forge.Refresh()` **无 error 返回值**，中间件无法感知，直接 `c.Next()`
3. **无机制**：刷新失败后没有设置响应 header（如 `X-Woodpecker-Token-Refresh-Failed: true`）、没有写 context、没有调用 `c.Set("token_refresh_failed", true)`

因此 **middleware 层完全没有将刷新失败转成用户可见状态的任何机制**。

#### 用户可见状态的唯一转换路径：前端 router guard

`web/src/router.ts:376-399`：

```typescript
router.beforeEach(async (to, _, next) => {
  if (authenticationMode === 'required' && !isAuthenticated) {
    config.setUserConfig('redirectUrl', to.fullPath);
    next({ name: 'login' });   // ← 跳转登录页
    return;
  }
  // ...
});
```

`isAuthenticated` 来自 `useAuthentication.ts`，检查的是 localStorage 中的 **Woodpecker 会话 token**，与 Forge Access Token 是否有效完全解耦。因此：

| 情况 | isAuthenticated | 页面跳转行为 |
|------|-----------------|-------------|
| user_sess Cookie 有效 + Forge token 已过期但刷新失败 | `true` | 正常进入页面，后续 API 调返回 500 + toast |
| user_sess Cookie 失效 / 被清 | `false` | 任何 `authentication: 'required'` 路由跳 `/login` |

**结论 3：刷新失败没有中间件级别的用户可见状态转换。用户可见的"登出"仅发生在 Woodpecker 会话本身失效时，与 Forge token 是否有效完全无关。**

---

### 8.4 并发用户都撞 Forge 限速时 goroutine 是否被撑爆

#### 关键代码事实

**事实 1：无 bounded goroutine pool / semaphore**

全项目搜索 `worker.*pool`、`goroutine.*pool`、`semaphore`、`ants` 均无命中。Woodpecker server 端没有任何全局 goroutine 池限制。HTTP 请求由 `net/http` server 默认行为处理：**每请求一个 goroutine**，并发上限取决于 `net/http` 的 `MaxHeaderBytes`、TCP backlog、文件描述符限制，而非应用层限制。

**事实 2：singleflight 仅做用户维度并发去重**

`refreshGroup.Do(key, fn)` 的 key 是 `"refresh-<userID>"`：
- 同一用户的 N 个并发请求 → 只产生 **1 个** 正在执行刷新的 goroutine + **N-1 个** 在 `singleflight.call.wg.Wait()` 上阻塞的 goroutine
- **不同用户** 同时触发刷新 → 每个用户各自一个 winner goroutine，完全并行

**事实 3：刷新 goroutine 的阻塞时间取决于 Forge token endpoint 的响应时间**

oauth2 `RetrieveToken()` 是同步 HTTP POST，无超时（8.1 节已验证）。当 Forge 对 token endpoint 做 429 限速且请求未被立即拒绝时（如 Forge 侧做了排队 + 长超时），winner goroutine 会长时间阻塞。

#### 撑爆路径推演

```
假设 Forge token endpoint 出现异常，响应时间从 100ms → 30s（或无限挂起）：

T=0s
  100 个活跃用户同时有请求到达，各自 token 都过期
  → 100 个 winner goroutine 并行执行 oauth2 HTTP POST
  → 每个 winner 阻塞 30s

T=0.5s
  第 2 批 100 个请求到达（同一批用户的新操作）
  → 100 个 waiter goroutine 挂在 singleflight.wg.Wait() 上
  → goroutine 总数 ≈ 200

T=1s
  第 3 批 100 个请求到达
  → goroutine 总数 ≈ 300

...
T=10s
  goroutine 总数 = 100 winner + 1900 waiter = 2000（仍在增长）

T=30s
  第一批 winner 返回（假设成功或失败）
  waiter 全部唤醒并退出等待
  但 30s 内新请求还在继续进入...
```

每 goroutine 初始栈 ~2KB，堆上 `*model.User`、`*refreshResult`、singleflight 内部 `call` 结构等 ~数百字节。**2000 goroutine 的内存占用约数 MB，本身不会直接 OOM**。真正的风险在于：

1. **文件描述符耗尽**：每个 winner goroutine 持有一个到 Forge token endpoint 的 TCP 连接。Go `http.Transport` 默认 `MaxIdleConnsPerHost=2`，但并发请求会创建新连接不回收。1000 并发 → 1000 个 FDs，超过 `ulimit -n` 后新连接全部失败（`"too many open files"`），连 DB 连接也受影响。

2. **上游反向代理（nginx）超时**：waiter goroutine 阻塞在 singleflight 上时，其对应的 HTTP 请求尚未写响应。nginx `proxy_read_timeout`（默认 60s）到期后会切断连接，客户端收到 504 Gateway Timeout，但 Woodpecker 端 goroutine 仍在跑，形成"连接已断但 goroutine  leaked"。

3. **singleflight 无超时泄漏**：`singleflight.Group` 的 `call` 结构在 `wg.Done()` 之前不会被回收。如果 winner goroutine 因网络问题永久挂起（对方不回 RST、`http.Client.Timeout=0`），则所有相关 waiter 永久阻塞，形成真正的 goroutine 泄漏。

#### 保护机制盘点

| 保护机制 | 是否存在 | 说明 |
|---------|---------|------|
| Goroutine 池 / Semaphore 限制 | ✗ | 无应用层限制 |
| oauth2 HTTP 请求超时 | ✗ | 未显式设置 `context.WithTimeout` 包裹 token 刷新 |
| 同一用户刷新失败冷却期 | ✗ | 失败后下一个请求立即再次尝试 |
| 熔断器（连续失败后暂停刷新） | ✗ | 无 |
| Gin HTTP Server 超时 | ? | Woodpecker 代码中未见显式 `srv.ReadTimeout` / `WriteTimeout` 设置，依赖部署环境 |
| singleflight 继承 context cancel | ⚠️ 部分 | `singleflight.Do` 内部 fn 使用的是 **winner 传入的 ctx**。如果 winner 的 HTTP 请求被取消（如客户端断开），fn 中 `refresher.Refresh(ctx, user)` 的 ctx 会被取消，oauth2 HTTP 请求会失败，fn 返回 error，所有 waiter 拿到同一个 error。但如果 winner 已进入 `http.Client.Do()` 且对方不处理 RST，cancel 不会立即释放 goroutine。 |

**结论 4：并发限速时存在 goroutine 堆积和 FD 耗尽的真实风险。** 当前代码依赖部署层（nginx 超时、ulimit、K8s livenessProbe）兜底，应用层无任何主动防护。大规模部署且 Forge 不稳定时，需额外补：
- `context.WithTimeout` 包裹 token 刷新（如 10s 上限）
- 失败后用户级冷却期
- HTTP server 显式超时设置
- 可选：全局 semaphore 限制并发刷新总数

---

## 9. 深度分析（续二）：部署层超时 vs 应用层刷新等待的匹配关系与最小代价补救

> 本章基于 `cmd/server/server.go`、`docker/Dockerfile.server.alpine.multiarch.rootless`、`cmd/server/health.go`、`server/api/z.go`、`server/forge/refresh.go`、`server/forge/refresh_test.go`、`server/router/middleware/token/token.go`、`server/rpc/rpc.go:550-572`、`server/pipeline/create.go:35-62`、[Woodpecker Helm chart values.yaml](https://github.com/woodpecker-ci/helm) 的代码逐行核对。

---

### 9.1 仓库代码中的部署超时配置——逐层盘点

#### 第 1 层：Docker HEALTHCHECK（非 K8s 部署场景）

**文件**: `docker/Dockerfile.server.alpine.multiarch.rootless:20`

```dockerfile
HEALTHCHECK CMD ["/bin/woodpecker-server", "ping"]
```

`ping` 子命令的实现位于 `cmd/server/health.go:29-63`：

```go
const pingTimeout = 1 * time.Second

func pinger(_ context.Context, c *cli.Command) error {
    // ...
    client := http.Client{Timeout: pingTimeout}
    resp, err := client.Get(healthURL)
    // ...
}
```

Docker HEALTHCHECK 默认参数：`--interval=30s --timeout=30s --start-period=0s --retries=3`

| 参数 | Docker 默认 | 实际效果 |
|------|------------|---------|
| interval | 30s | 每 30s 执行一次 `woodpecker-server ping` |
| timeout | 30s | 但 ping 内部 `http.Client.Timeout = 1s`，所以实际超时 1s |
| retries | 3 | 连续 3 次失败后标记 unhealthy |
| **最大容忍时间** | | **30s × 3 = 90s**（从第一个失败到 unhealthy） |

**healthz 端点**（`server/api/z.go:37-43`）：只检查 `store.Ping()`（DB 连通性），不检查 Forge 连通性。即使 Forge 全部 429，`/healthz` 仍返回 204。

**结论**：Docker HEALTHCHECK **无法检测** Forge 刷新阻塞，因为 `/healthz` 不检查 Forge 状态。

#### 第 2 层：K8s Liveness / Readiness Probe（Helm Chart 部署场景）

Helm chart 位于独立仓库 `woodpecker-ci/helm`，仓库内不含 manifests。以下来自 [Helm values.yaml](https://github.com/woodpecker-ci/helm/blob/main/charts/woodpecker/values.yaml)：

```yaml
server:
  probes:
    liveness:
      timeoutSeconds: 10
      periodSeconds: 10
      successThreshold: 1
      failureThreshold: 3
    readiness:
      timeoutSeconds: 10
      periodSeconds: 10
      successThreshold: 1
      failureThreshold: 3
```

Helm chart 的 statefulset 模板对 probe 的实现使用 `httpGet` 指向 `/healthz` 端口 8000。

| 参数 | Liveness | Readiness |
|------|----------|-----------|
| httpGet.path | `/healthz` | `/healthz` |
| timeoutSeconds | 10 | 10 |
| periodSeconds | 10 | 10 |
| failureThreshold | 3 | 3 |
| **最大容忍时间** | **10s × 3 = 30s** | **10s × 3 = 30s** |

**关键问题**：Liveness 和 Readiness 都指向 `/healthz`，而 `/healthz` 只检查 DB 连通性。当 Forge 刷新阻塞导致 goroutine 堆积时：
- **Readiness**：不会 fail（DB 仍连通）→ Pod 仍接收流量 → 请求继续堆积
- **Liveness**：不会 fail（进程不死、DB 通）→ 不会重启 → goroutine 泄漏持续积累

**结论**：K8s probe 配置与 Forge 刷新阻塞**完全解耦**，无法通过现有探针触发自愈。

#### 第 3 层：Go HTTP Server 超时

**文件**: `cmd/server/server.go:183-189, 249-252`

```go
// TLS server
tlsServer := &http.Server{
    Addr:    server.Config.Server.PortTLS,
    Handler: handler,
    TLSConfig: &tls.Config{...},
    // ← 无 ReadTimeout / WriteTimeout / IdleTimeout 设置
}

// HTTP server
httpServer := &http.Server{
    Addr:    c.String("server-addr"),
    Handler: handler,
    // ← 同样无任何超时设置
}
```

**Go `http.Server` 零值行为**：`ReadTimeout=0` / `WriteTimeout=0` / `IdleTimeout=0`，意味着 **无超时**。当一个请求进入 `token.Refresh` 中间件后阻塞在 singleflight 上时，HTTP server 永远不会主动断开该连接。

**结论**：应用层 HTTP Server **无任何请求级超时**，goroutine 可无限期存活。

#### 第 4 层：Nginx 反向代理超时（典型部署拓扑）

仓库内无 nginx 配置。以下为典型 K8s nginx-ingress 默认值：

| 参数 | Nginx Ingress 默认 | 含义 |
|------|-------------------|------|
| `proxy_connect_timeout` | 5s | 与上游建立 TCP 连接的超时 |
| `proxy_send_timeout` | 60s | 向上游发送请求体的超时 |
| `proxy_read_timeout` | 60s | **等待上游响应的超时** |

**关键**：`proxy_read_timeout = 60s` 是最外层有实际约束力的超时。当 Woodpecker 的请求因 Forge 刷新阻塞超过 60s 时：
1. nginx 返回 504 Gateway Timeout 给客户端
2. **但 Woodpecker 端的 goroutine 仍在运行**——nginx 断开连接不会触发 `c.Request.Context()` 的 cancel（除非 Woodpecker 显式使用 `http.CloseNotifier` 或 `c.Request.Context()` 监听连接关闭，当前代码没有）
3. goroutine 泄漏直到 Forge 响应或永远

---

### 9.2 超时匹配关系总图

```
                  客户端              Nginx Ingress          Woodpecker Server          forge.Refresh
                  ┌──────┐          ┌───────────┐          ┌───────────────┐          ┌──────────────┐
  请求发出 ──────►│      │────────►│            │────────►│               │────────►│              │
                  │      │          │ proxy_read │          │ http.Server   │          │ oauth2 POST  │
  等待响应 ◄──────│      │◄────────│ _timeout=  │◄────────│ 无超时(0)     │◄────────│ 无超时(0)    │
                  └──────┘          │   60s      │          │               │          │              │
                                    └───────────┘          └───────────────┘          └──────────────┘
                                     ↑                        ↑                          ↑
                               60s 后返回 504            goroutine 不死             单次 HTTP POST
                               客户端看到 504             继续等 Forge              等 Forge 响应
                               但 Woodpecker             直到 Forge 响应            可能无限期
                               goroutine 仍在跑           或进程被 kill              阻塞

  ── K8s Probe ──────────────────────────────────────────────────────────────────────────────────────
  liveness/readiness → GET /healthz → 检查 DB → DB 通 → 204 → 不触发任何动作
  ──────────────────────────────────────────────────────────────────────────────────────────────────

  ── Docker HEALTHCHECK ──────────────────────────────────────────────────────────────────────────────
  woodpecker-server ping → GET /healthz → 1s 超时 → DB 通 → 204 → healthy
  ──────────────────────────────────────────────────────────────────────────────────────────────────
```

**核心矛盾**：

| 层级 | 超时值 | 覆盖范围 |
|------|--------|---------|
| nginx `proxy_read_timeout` | 60s | 客户端 ↔ nginx 连接 |
| K8s liveness probe timeout | 10s | 仅 DB 连通性 |
| K8s readiness probe timeout | 10s | 仅 DB 连通性 |
| Docker HEALTHCHECK | 1s (http.Client) | 仅 DB 连通性 |
| Go `http.Server` | ∞ (0) | 请求级无超时 |
| `forge.Refresh` → `oauth2.TokenSource.Token()` | ∞ (取决于 ctx) | Forge token endpoint 调用无超时 |

**结论**：Forge token endpoint 刷新的**假设等待时长是无上限**，而部署层唯一有约束力的超时是 nginx 的 60s（但只切断了客户端连接，不杀 goroutine）。两层之间**严重不匹配**。

---

### 9.3 最小代价补救方案：context cancel 透传 + 刷新超时上限

#### 设计目标

1. `forge.Refresh` 内部为 token 刷新设置**硬性超时上限**（如 15s）
2. 当 winner goroutine 的 ctx 被 cancel 时，oauth2 HTTP 请求能及时终止
3. waiter goroutine 不无限期阻塞——如果 winner 长时间无响应，waiter 也应能超时退出
4. 改动**最小化**，不改变 `forge.Refresh` 的函数签名，现有调用方无需修改

#### 方案详解

**修改文件 1**: `server/forge/refresh.go`

```go
// 新增常量：token 刷新最大等待时间
const tokenRefreshTimeout = 15 * time.Second

func Refresh(ctx context.Context, forge Forge, _store store.Store, user *model.User) {
    const tokenMinTTL = 1800

    if refresher, ok := forge.(Refresher); ok {
        if time.Now().UTC().Unix() < (user.Expiry - tokenMinTTL) {
            return
        }

        key := fmt.Sprintf("refresh-%d", user.ID)

        // ★ 改动点 1：为 winner 创建带超时的独立 ctx
        //   不用传入的 ctx，因为 winner 的刷新可能比调用方 ctx 的生命周期更长
        //   （调用方的 ctx 可能因为客户端断连而 cancel，但我们不希望因此中断正在进行的刷新）
        refreshCtx, refreshCancel := context.WithTimeout(context.WithoutCancel(ctx), tokenRefreshTimeout)
        defer refreshCancel()

        result, err, _ := refreshGroup.Do(key, func() (any, error) {
            // ★ 改动点 2：winner 使用 refreshCtx 而非原始 ctx
            userUpdated, err := refresher.Refresh(refreshCtx, user)
            if err != nil {
                return nil, err
            }
            if userUpdated {
                if err := _store.UpdateUser(user); err != nil {
                    log.Error().Err(err).Msg("fail to save user to store after refresh oauth token")
                }
            }
            return &refreshResult{
                AccessToken:  user.AccessToken,
                RefreshToken: user.RefreshToken,
                Expiry:       user.Expiry,
            }, nil
        })
        if err != nil {
            log.Error().Err(err).Msgf("refresh oauth token of user '%s' failed", user.Login)
            return
        }

        if r, ok := result.(*refreshResult); ok {
            user.AccessToken = r.AccessToken
            user.RefreshToken = r.RefreshToken
            user.Expiry = r.Expiry
        }
    }
}
```

**关键设计决策解释**：

1. **为什么用 `context.WithoutCancel(ctx)` 而非直接用传入的 `ctx`？**

   当前的 3 个调用方传入的 ctx 来源不同：
   - `token/token.go:37`：`c.Request.Context()`——客户端断连时 ctx 会被 cancel
   - `pipeline/create.go:62`：hook handler 的 `c.Request.Context()`——同上
   - `rpc/rpc.go:563`：gRPC stream 的 ctx——agent 断连时 ctx 会被 cancel

   如果 winner 使用调用方的 ctx，当第一个到达的请求的客户端断连时，刷新会立即中止，但其他 waiter 仍在等待结果——形成死锁（winner 被取消，singleflight 永远不会返回给 waiter）。

   使用 `context.WithoutCancel(ctx)` 继承值（trace ID、logger 等）但不受父 ctx cancel 影响，确保 winner 总是能完成或超时。

2. **为什么 15s？**

   - Forge token endpoint 正常响应时间 < 1s
   - 429 排队时 Forge 可能在 10-30s 后返回
   - 15s < nginx `proxy_read_timeout`（60s），给下游 handler 留 45s 余量
   - 15s 内至少可以尝试一次完整的 token exchange

3. **waiter 的超时如何保障？**

   `singleflight.Do()` 会阻塞直到 winner 的闭包返回。winner 闭包受 `refreshCtx` 的 15s 超时约束，因此 waiter 最多阻塞 15s + 几微秒（singleflight 内部唤醒开销）。不需要为 waiter 单独设置超时。

**修改文件 2**: `server/forge/refresh_test.go`

新增测试用例：

```go
func TestRefresh_ContextTimeout(t *testing.T) {
    mockForge := forge_mocks.NewMockForge(t)
    mockRefresher := forge_mocks.NewMockRefresher(t)
    mockStore := store_mocks.NewMockStore(t)

    f := &refresherForge{MockForge: mockForge, MockRefresher: mockRefresher}
    user := expiredUser(1)

    // 模拟 Forge 响应极慢（超过 tokenRefreshTimeout）
    mockRefresher.On("Refresh", mock.Anything, mock.Anything).Return(false, context.DeadlineExceeded).Run(func(_ mock.Arguments) {
        time.Sleep(20 * time.Second) // 超过 15s 上限
    })

    start := time.Now()
    forge.Refresh(context.Background(), f, mockStore, user)
    elapsed := time.Since(start)

    // 应在 ~15s 内返回，而非无限阻塞
    assert.Less(t, elapsed, 20*time.Second, "refresh should timeout within tokenRefreshTimeout")
    // Token 应保持不变
    assert.Equal(t, "old-access-token", user.AccessToken)
    mockStore.AssertNotCalled(t, "UpdateUser", mock.Anything)
}
```

**修改文件 3**: `cmd/server/server.go`

为 HTTP server 添加基础超时（与 tokenRefreshTimeout 配合，形成双层保护）：

```go
httpServer := &http.Server{
    Addr:    c.String("server-addr"),
    Handler: handler,
    // ★ 新增：请求级超时，兜底任何单个请求的无限阻塞
    ReadHeaderTimeout: 10 * time.Second,
    IdleTimeout:       120 * time.Second,
    // 不设 ReadTimeout/WriteTimeout，因为 webhook 和 SSE 等长连接场景需要
    // 由 refresh.go 内部的 tokenRefreshTimeout 精确控制刷新时长
}
```

同样对 `tlsServer` 做相同修改。

---

### 9.4 对现有调用方的影响评估

| 调用方 | 文件 | 传入 ctx | 当前行为 | 修改后行为 | 影响程度 |
|--------|------|---------|---------|-----------|---------|
| HTTP API 中间件 | `token/token.go:37` | `c.Request.Context()` | 刷新无超时，客户端断连后 ctx cancel 但 winner 继续跑 | winner 最多 15s，不受客户端断连影响 | **零**：中间件调用 `forge.Refresh()` 签名不变 |
| Pipeline 创建 | `pipeline/create.go:62` | hook handler ctx | 同上 | 同上 | **零** |
| gRPC Agent 回调 | `rpc/rpc.go:563` | gRPC stream ctx | agent 断连后 ctx cancel 但 winner 继续跑 | winner 最多 15s，不受 agent 断连影响 | **零** |
| 单元测试 | `refresh_test.go` | `context.Background()` | 所有测试通过 | 需新增 context timeout 测试 | **低**：仅新增测试，不改已有测试 |

**函数签名变更**：`forge.Refresh()` 签名完全不变（`func Refresh(ctx context.Context, forge Forge, _store store.Store, user *model.User)`），所有调用方无需修改。

**行为变更**：

| 行为 | 修改前 | 修改后 |
|------|--------|--------|
| Forge token endpoint 正常响应（<1s） | 刷新成功，token 更新 | **无变化** |
| Forge 返回 429/5xx（立即失败） | 记日志，请求继续用旧 token | **无变化** |
| Forge 挂起不响应（无限期） | goroutine 永久阻塞 | **15s 后超时返回，goroutine 释放** |
| 客户端断连但 winner 正在刷新 | winner 继续跑完（使用调用方 ctx 则会中断） | winner 仍继续跑完（用独立 ctx），最多 15s |
| 多个 waiter 等待同一个 winner | 等到 winner 完成（可能无限期） | 最多等 15s |
| 刷新失败后的下一个请求 | 立即再次尝试刷新 | **无变化**（仍无冷却期，但每次至少有 15s 上限） |

**剩余风险（本方案未解决）**：

1. **串行刷新风暴**：用户 A 的 token 过期，每 15s 就会有一个新请求触发刷新尝试（15s 失败 → 下一个请求进入 → 又 15s → 循环）。需要额外的"失败冷却期"机制，但代价更高（需改 `forge.Refresh` 签名或引入内存缓存标记）。
2. **不同用户并发刷新**：100 个用户同时到期 → 100 个独立 winner 并行执行，仍可能触发 Forge 全局 rate limit。需要全局 semaphore，但代价更高。
3. **`/healthz` 不反映 Forge 状态**：需要增加 `/healthz` 对 Forge 连通性的检查，但需谨慎——Forge 暂时不可用不应触发 liveness kill（会级联重启所有 Pod）。

---

### 9.5 方案代价总结

| 维度 | 代价 |
|------|------|
| 修改文件数 | 3（`refresh.go`、`refresh_test.go`、`cmd/server/server.go`） |
| 函数签名变更 | 0 |
| 调用方代码变更 | 0 |
| 新增依赖 | 0（`context.WithoutCancel` 是 Go 1.21+ 标准库） |
| 新增配置项 | 0（`tokenRefreshTimeout` 是硬编码常量，可后续改为 flag） |
| 回归风险 | 极低：正常路径行为不变，仅对"Forge 挂起"场景增加超时上限 |
| 测试覆盖 | 新增 1 个测试用例（`TestRefresh_ContextTimeout`） |

---

## 10. 关键代码路径索引

| 功能 | 文件 | 行号/函数 |
|------|------|-----------|
| 前端登录入口 | `web/src/views/Login.vue` | `authenticate(forge.id)` |
| 前端认证组合 | `web/src/compositions/useAuthentication.ts` | `authenticate()` |
| 路由注册 | `server/router/router.go:61-65` | `/authorize` GET/POST |
| OAuth 核心处理 | `server/api/login.go` | `HandleAuth()` |
| Forge 接口定义 | `server/forge/forge.go` | `Forge interface` |
| OAuth 请求类型 | `server/forge/types/oauth.go` | `OAuthRequest` |
| Token 刷新 | `server/forge/refresh.go` | `Refresh()`, `Refresher` |
| Token 刷新中间件 | `server/router/middleware/token/token.go` | `Refresh()` |
| Session 用户加载 | `server/router/middleware/session/user.go` | `SetUser()` |
| Forge 数据模型 | `server/model/forge.go` | `Forge struct` |
| 用户数据模型 | `server/model/user.go` | `User struct` |
| Forge 工厂 | `server/forge/setup/setup.go` | `Forge()` |
| Forge 管理器 | `server/services/manager.go` | `ForgeByID()` |
| GitHub 实现 | `server/forge/github/github.go` | `Login()`, `Refresh()` |
| GitLab 实现 | `server/forge/gitlab/gitlab.go` | `Login()`, `Refresh()` |
| Gitea 实现 | `server/forge/gitea/gitea.go` | `Login()`, `Refresh()` |
| Forgejo 实现 | `server/forge/forgejo/forgejo.go` | `Login()`, `Refresh()` |
| Bitbucket 实现 | `server/forge/bitbucket/bitbucket.go` | `Login()`, `Refresh()` |
| Bitbucket DC 实现 | `server/forge/bitbucketdatacenter/bitbucketdatacenter.go` | `Login()` |
| 通用 Token 查找 | `server/forge/common/utils.go` | `UserToken()`, `RepoUser()` |
| JWT Token 类型 | `shared/token/token.go` | `OAuthStateToken`, `SessToken` |
| Netrc 凭据 | `server/model/netrc.go` | `Netrc struct` |
| Token 刷新单元测试 | `server/forge/refresh_test.go` | `TestRefresh_ConcurrentRefreshSerialized`, `TestRefresh_ConcurrentRefreshError` |
| Pipeline 创建降级 | `server/pipeline/create.go:62,87-93` | `forge.Refresh()` + config fallback |
| 通用 HTTP 重试 | `server/services/utils/http.go:236-240` | `isRetryableStatusCode()` (仅 5xx) |
| Config Fetcher 重试 | `server/services/config/forge.go:65-73` | `retryCount` 次循环 |
| HTTP API 客户端 | `web/src/lib/api/client.ts` | `_request()`, ApiClient.onerror |
| 前端全局错误处理 | `web/src/App.vue:37-43` | `apiClient.setErrorHandler()`（仅 404 特殊处理） |
| 前端认证 Router Guard | `web/src/router.ts:376-399` | `router.beforeEach()` |
| Webhook 入口 | `server/api/hook.go:69-213` | `PostHook()` |
| Pipeline 创建 | `server/pipeline/create.go:35-93` | `Create()` |
| GitLab Refresh 实现 | `server/forge/gitlab/gitlab.go:160-179` | `Refresh()` |
| GitHub Refresh 实现 | `server/forge/github/github.go:158-181` | `Refresh()` |
| Gitea/Forgejo/Bitbucket Refresh 实现 | 各 forge 目录下的主文件 | `Refresh()`，模式与 GitLab 完全一致 |
| HTTP Server 启动 | `cmd/server/server.go:183-189, 249-252` | `http.Server{}`（无超时设置） |
| Docker HEALTHCHECK | `docker/Dockerfile.server.alpine.multiarch.rootless:20` | `HEALTHCHECK CMD ping` |
| Ping 子命令 | `cmd/server/health.go:29-63` | `pingTimeout = 1s` |
| /healthz 端点 | `server/api/z.go:37-43` | `Health()`（仅检查 DB） |
