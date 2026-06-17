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

## 8. 关键代码路径索引

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
