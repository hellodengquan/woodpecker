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

## 7. 关键代码路径索引

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
