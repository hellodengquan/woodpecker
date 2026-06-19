# Woodpecker 权限模型分析

## 概述

Woodpecker 的权限系统由 **身份来源**、**成员关系** 和 **访问控制** 三个核心层次共同作用，形成了一个基于 OAuth2 认证、组织成员校验、仓库细粒度权限的完整权限体系。

---

## 一、身份来源（用户认证层）

### 1.1 核心数据模型

**User 模型** (`server/model/user.go:31-76`)

```go
type User struct {
    ID             int64         // 本地用户ID
    ForgeID        int64         // 关联的Forge实例ID
    ForgeRemoteID  ForgeRemoteID // Forge端的用户唯一标识
    Login          string        // 用户名
    AccessToken    string        // OAuth2访问令牌
    RefreshToken   string        // OAuth2刷新令牌
    Expiry         int64         // 令牌过期时间
    Email          string
    Avatar         string
    Admin          bool          // 是否系统管理员
    Hash           string        // 用于签名token的随机哈希
    OrgID          int64         // 关联的个人组织ID
}
```

### 1.2 认证流程

**登录入口**：`server/api/login.go:47-323` 的 `HandleAuth` 函数

```
用户访问 /authorize
    │
    ├─ 首次访问（无code参数）
    │   ├─ 生成 OAuth state token（含forge-id）
    │   └─ 重定向到 Forge 授权页面
    │
    └─ 回调阶段（带code和state参数）
        ├─ 验证 state token 有效性
        ├─ 调用 Forge.Login() 换取用户信息
        ├─ 检查组织白名单（如配置）
        ├─ 查找或创建本地用户
        ├─ 创建/关联个人组织
        ├─ 更新用户令牌和元数据
        ├─ 同步仓库权限（同步或异步）
        └─ 签发会话 cookie
```

### 1.2.1 OAuth Token 申请完整代码路径

**统一入口**：`server/api/login.go:47-323` `HandleAuth`

```
用户访问 GET /authorize （Handler: HandleAuth）
    │
    ├─ Step1: 解析 ForgeID 和 code 参数
    │   └─ 无 code → 进入「授权前阶段」，有 code → 进入「回调阶段」
    │
    ├─ 授权前阶段（req.Code 为空）：
    │   ├─ 构造 state token = JWT(kind=CsrfToken, forge_id=forgeID)
    │   │   └─ 有效期：stateTokenDuration = 5 分钟
    │   │
    │   ├─ 调用 Forge.Login(ctx, &OAuthRequest{State: stateToken})
    │   │   │
    │   │   ├─ **Gitea/Forgejo** (`server/forge/gitea/gitea.go:112-147`)：
    │   │   │   ├─ config.AuthCodeURL(state) 生成授权 URL
    │   │   │   └─ 带 scope = read:org, read:user, write:repository, read:repository
    │   │   │
    │   │   ├─ **GitHub** (`server/forge/github/github.go:107-154`)：
    │   │   │   └─ config.AuthCodeURL(state) + scope=read:org, repo
    │   │   │
    │   │   ├─ **GitLab** (`server/forge/gitlab/gitlab.go:115-156`)：
    │   │   │   └─ config.AuthCodeURL(state) + scope=api, read_user
    │   │   │
    │   │   └─ **Bitbucket**：scope=project, team, repository
    │   │
    │   └─ Forge 返回 (nil, redirectURL, nil)
    │       └─ HTTP 303 → redirectURL（跳转到 Forge 授权页
    │
    └─ 回调阶段（req.Code 非空）：
        ├─ 解析 state token，校验 forge_id
        │   └─ 失败 → 重定向 /login?error=...
        │
        ├─ 第二次调用 Forge.Login(ctx, &OAuthRequest{Code: code, State: state})
        │   │
        │   ├─ 所有 Provider 共同流程：
        │   │   ├─ config.Exchange(oauth2Ctx, req.Code)
        │   │   │   └─ 换取 AccessToken/RefreshToken/Expiry
        │   │   │
        │   │   ├─ 用新 AccessToken 构建 HTTP Client
        │   │   │
        │   │   └─ 查询 Forge 用户资料：
        │   │       ├─ Gitea: client.GetMyUserInfo() → ID/UserName/Email/AvatarURL
        │   │       ├─ GitHub: client.Users.Get(ctx, "") + ListEmails 匹配已验证邮箱
        │   │       ├─ GitLab: client.Users.CurrentUser() → Username/Email/AvatarURL
        │   │       └─ Bitbucket: client.User.Current()
        │   │
        │   └─ 返回 User 对象填充:
        │       {Login, Email, Avatar, AccessToken, RefreshToken, Expiry, ForgeRemoteID}
        │
        ├─ 组织白名单检查 Permissions.Orgs.IsMember
        │
        ├─ DB GetUserByRemoteID(ForgeID, ForgeRemoteID) 查找本地用户
        │   ├─ 存在 → 更新字段 + 创建/关联个人组织
        │   └─ 不存在 → CreateUser(事务内: 先建/找同名Org IsUser=true, 再插入users表)
        │
        ├─ updateRepoPermissions 同步仓库权限（见 3.2）
        │
        └─ 生成 SessToken (kind=SessToken, user-id=id)
            └─ 签名 user.Hash, 有效期 SessionExpires
            └─ Set-Cookie 写入浏览器
```

### 1.3 会话管理

**会话中间件**：`server/router/middleware/session/user.go:42-75` 的 `SetUser`

- 支持两种 token 类型：`SessToken`（浏览器会话）和 `UserToken`（API访问）
- 从请求中解析 JWT token，提取 `user-id`
- 从数据库加载用户，使用 `user.Hash` 验证 token 签名
- 会话 token 额外验证 CSRF token

### 1.4 系统管理员

系统管理员权限来源有二：

1. **环境变量配置**：`WOODPECKER_ADMIN` 环境变量中的用户名，登录时自动设置 `AdminEnv=true`
2. **数据库字段**：`users.admin` 字段直接标记

**管理员权限检查**：`server/router/middleware/session/user.go:77-91` 的 `MustAdmin`

---

## 二、成员关系（组织/团队层）

### 2.1 核心数据模型

**Org 模型** (`server/model/org.go:18-25`)

```go
type Org struct {
    ID       int64
    ForgeID  int64
    Name     string  // 组织名称
    IsUser   bool    // 是否为用户个人组织
    Private  bool    // 是否私有组织
}
```

**OrgPerm 模型** (`server/model/perm.go:36-39`)

```go
type OrgPerm struct {
    Member bool  // 是否组织成员
    Admin  bool  // 是否组织管理员
}
```

### 2.2 组织创建逻辑

**用户登录时自动创建个人组织** (`server/api/login.go:221-268`)

```
用户登录成功后
    │
    ├─ 检查 user.OrgID 是否为0
    │   ├─ 是：查找同名组织，不存在则创建
    │   │   └─ 设置 IsUser=true, Private=false
    │   └─ 否：更新组织名称（如用户名变更）
    └─ 关联 user.OrgID = org.ID
```

**仓库激活时自动创建组织** (`server/api/repo.go:126-151`)

```
激活仓库时
    │
    ├─ 根据 repo.Owner 查找组织
    ├─ 不存在则调用 Forge.Org() 获取组织信息
    └─ 创建组织并关联 repo.OrgID
```

### 2.3 成员关系校验

**成员关系服务**：`server/cache/membership.go`

```go
type MembershipService interface {
    Get(ctx context.Context, forge Forge, u *User, org string) (*OrgPerm, error)
}
```

**校验流程** (`server/cache/membership.go:54-70`)：

```
请求组织权限
    │
    ├─ 检查缓存（key: ForgeRemoteID-org，TTL: 10分钟）
    │   ├─ 命中且未过期：直接返回
    │   └─ 未命中或过期：
    │       ├─ 调用 Forge.OrgMembership() 查询 Forge 端
    │       ├─ 写入缓存
    │       └─ 返回结果
    └─ 结果：{Member: bool, Admin: bool}
```

**组织访问控制**：`server/router/middleware/session/user.go:123-168` 的 `MustOrgMember`

```
访问组织资源时
    │
    ├─ 检查用户是否已认证
    ├─ 检查组织是否已加载
    ├─ 特殊放行：
    │   ├─ 个人组织（org.Name == user.Login）
    │   └─ 系统管理员（user.Admin == true）
    ├─ 调用 MembershipService.Get() 检查成员关系
    └─ 校验：
        ├─ 普通成员：perm.Member == true
        └─ 管理员：perm.Admin == true
```

---

## 三、访问控制（仓库权限层）

### 3.1 核心数据模型

**Perm 模型** (`server/model/perm.go:19-28`)

```go
type Perm struct {
    UserID  int64  // 联合唯一索引
    RepoID  int64  // 联合唯一索引
    Pull    bool   // 可读权限（查看流水线、日志等）
    Push    bool   // 可写权限（触发流水线、管理密钥等）
    Admin   bool   // 管理权限（仓库设置、删除等）
    Synced  int64  // 上次同步时间戳
    Created int64
    Updated int64
}
```

**Repo 模型** (`server/model/repo.go:45-93`) 中的可见性：

```go
type RepoVisibility string
const (
    VisibilityPublic   RepoVisibility = "public"   // 公开
    VisibilityPrivate  RepoVisibility = "private"  // 私有
    VisibilityInternal RepoVisibility = "internal" // 内部（登录用户可见）
)
```

### 3.2 权限来源与同步

**登录时全量同步** (`server/api/login.go:325-373` 的 `updateRepoPermissions`)

```
用户登录时
    │
    ├─ 调用 Forge.Repos() 获取用户在 Forge 端的所有仓库
    ├─ 遍历每个仓库：
    │   ├─ 仅处理已在 Woodpecker 激活的仓库
    │   ├─ 使用 Forge 返回的 Repo.Perm（含 Pull/Push/Admin）
    │   ├─ 调用 PermUpsert() 写入/更新数据库
    │   └─ 记录 repoID 用于后续清理
    └─ 调用 PermPrune() 删除用户不再有权限的仓库记录
```

**请求时增量同步** (`server/router/middleware/session/repo.go:104-166` 的 `SetPerm`)

```
每次请求仓库资源时
    │
    ├─ 从数据库读取 Perm 记录
    ├─ 检查 Synced 时间：
    │   └─ 超过1小时：调用 Forge.Repo() 重新获取权限并更新
    └─ 应用权限提升规则（见3.3节）
```

### 3.3 权限计算规则

**SetPerm 中间件** (`server/router/middleware/session/repo.go:104-166`) 中的权限合并逻辑：

```
初始权限 = 数据库存储的 Perm
    │
    ├─ 规则1：系统管理员自动获得全权限
    │   if user.Admin: Pull=true, Push=true, Admin=true
    │
    └─ 规则2：可见性决定读权限
        ├─ 公开仓库 (VisibilityPublic): Pull=true（对所有人）
        └─ 内部仓库 (VisibilityInternal): Pull=true（对已登录用户）
```

### 3.4 权限检查中间件

| 中间件 | 检查逻辑 | 适用场景 |
|--------|----------|----------|
| `MustUser()` | 检查用户已认证 | 用户个人资源 |
| `MustAdmin()` | 检查 `user.Admin == true` | 系统级管理 |
| `MustOrgMember(false)` | 检查组织成员身份 | 组织只读操作 |
| `MustOrgMember(true)` | 检查组织管理员身份 | 组织管理操作 |
| `MustPull` | 检查 `perm.Pull == true` | 仓库只读操作 |
| `MustPush` | 检查 `perm.Push == true` | 仓库写操作 |
| `MustRepoAdmin()` | 检查 `perm.Admin == true` | 仓库管理操作 |

### 3.5 路由中的权限应用

**示例：仓库路由配置** (`server/router/api.go:92-163`)

```go
repoBase := repo.Group("/:repo_id")
{
    repoBase.Use(session.SetRepo())    // 加载仓库
    repoBase.Use(session.SetPerm())    // 计算权限

    repoBase.GET("/permissions", api.GetRepoPermissions)

    repo := repoBase.Group("")
    {
        repo.Use(session.MustPull)     // 以下所有路由需要Pull权限

        repo.GET("", api.GetRepo)
        repo.GET("/branches", api.GetRepoBranches)

        // 需要Push权限的路由
        repo.POST("/pipelines", session.MustPush, api.CreatePipeline)
        repo.POST("/pipelines/:pipeline_number/cancel", session.MustPush, api.CancelPipeline)

        // 需要Admin权限的路由
        repo.PATCH("", session.MustRepoAdmin(), api.PatchRepo)
        repo.DELETE("", session.MustRepoAdmin(), api.DeleteRepo)
    }
}
```

### 3.6 Secret 资源的三级作用域隔离

Secret 采用三级作用域架构，实现全局/组织/仓库三层覆盖关系与优先级隔离。

**核心数据模型** (`server/model/secret.go:47-81`)

```go
type Secret struct {
    ID     int64
    OrgID  int64          // 组织ID（组织级secret非0）
    RepoID int64          // 仓库ID（仓库级secret非0）
    Name   string
    Value  string
    Images []string       // 限制哪些镜像可使用
    Events []WebhookEvent   // 限制哪些事件类型可使用
}

// 三级作用域判断
IsGlobal()       bool  // RepoID==0 && OrgID==0
IsOrganization() bool  // RepoID==0 && OrgID!=0
IsRepository()   bool  // RepoID!=0 && OrgID==0
```

**三级覆盖优先级** (`server/services/secret/db.go:41-72` `SecretListPipeline`)

```
流水线运行时收集 Secret 注入（按顺序，重名优先级：
    │
    ├─ 优先级1：仓库级 Secret（IsRepository()=true
    │   └─ 同名覆盖：仅对特定仓库
    │
    ├─ 优先级2：组织级 Secret（IsOrganization()=true
    │   └─ 同名覆盖：覆盖该组织所有仓库
    │
    └─ 优先级3：全局级 Secret（IsGlobal()=true
        └─ 同名覆盖：覆盖所有仓库
```

**事件与镜像细粒度限制** (`server/model/secret.go:94-122`)

每个 Secret 可以通过两个维度控制可用范围：
- **Events 过滤：`push/pull_request/tag/deploy/cron/manual 等事件类型白名单
- **Images 过滤**：限制仅当 pipeline 步骤使用的 Docker 镜像名匹配正则白名单（正则校验

**路由权限入口** (路由配置 `server/router/api.go`)

| API 端点 | 前置权限 |
|---------|----------|
| `GET /repos/:repo_id/secrets` | MustPull（管理员级） |
| `POST/PATCH/DELETE` 增删改 | MustRepoAdmin() |
| `GET/POST/PATCH /orgs/:org_id/secrets` | MustOrgMember(true) |
| `/secrets` 全局 | MustAdmin() |

**Secret 服务组合模式** (`server/services/manager.go:99-106`)

```
SecretServiceFromRepo(repo)
    │
    ├─ 仓库配置了 SecretExtensionEndpoint?
    │   ├─ 是：Combined(本地 + HTTP外部
    │   └─ 否：仅本地 DB
    │
    └─ 外部 Secret 服务 (HTTP)通过 Netrc 认证
```

### 3.7 Pipeline 资源的 Gated 审批隔离

**审批门控机制** (`server/pipeline/gated.go:23-67`)

```go
// needsApproval(repo, pipeline) 判断流程：
```

```
Webhook 触发 pipeline 时：
    │
    ├─ cron/manual 事件直接放行（无需审批
    │
    ├─ 白名单用户检查 approval_allowed_users
    │   └─ pipeline.Author 在白名单 → 放行
    │
    └─ 根据 repo.RequireApproval 策略：
        ├─ RequireApprovalNone：所有事件放行
        ├─ RequireApprovalForks：仅 fork 来源的 PR 需要审批
        ├─ RequireApprovalPullRequests：所有 PR 需要审批
        └─ RequireApprovalAllEvents：所有事件都需要审批
```

**审批权限入口**

审批操作权限：`server/router/api.go` 路由配置中对应 API 权限：

| 端点 | 权限 | 实现文件 |
|-----|------|---------|
| `POST /approve` 审批通过 | MustPush + StatusBlocked → StatusPending | `server/pipeline/approve.go:31-88` |
| `POST /decline` 审批拒绝 | MustPush + StatusBlocked → StatusDeclined | `server/pipeline/decline.go:30-72` |

**审批状态流转**

```
Webhook到达 → setApprovalState()
    │
    ├─ needsApproval() = true?
    │   ├─ 是：StatusBlocked
    │   │   ├─ PostApproval() → StatusPending → 运行
    │   │   └─ PostDecline() → StatusDeclined → 结束
    │   │
    │   └─ 否：StatusPending → 直接运行
```

---

## 四、三者协作完整流程

### 4.1 单次 API 请求的权限校验链

以 `GET /api/repos/:repo_id/pipelines` 为例：

```
HTTP 请求到达
    │
    ▼
[路由中间件层]
    ├─ session.SetUser() ── 解析token → 加载用户（身份来源）
    ├─ token.Refresh    ── 刷新OAuth令牌
    │
    ▼
[API路由匹配] → /api/repos/:repo_id/pipelines
    │
    ▼
[仓库权限计算层]
    ├─ session.SetRepo() ── 根据repo_id加载Repo
    ├─ session.SetPerm() ── 计算Perm：
    │   ├─ 从DB读取用户-仓库权限
    │   ├─ 过期则从Forge同步
    │   ├─ 系统管理员 → 全权限
    │   └─ 仓库可见性 → Pull权限
    │
    ▼
[权限检查层]
    ├─ session.MustPull ── 检查 perm.Pull == true
    │   ├─ 通过 → 继续执行
    │   └─ 失败 → 返回 401/404
    │
    ▼
[业务逻辑层]
    └─ api.GetPipelines ── 执行实际业务
```

### 4.2 权限数据流转图

```
┌─────────────────┐     OAuth2     ┌─────────────────┐
│   Forge (GitHub)│<──────────────>│  User (Identity)│
│  GitLab, Gitea  │                └─────────────────┘
└─────────────────┘                       │
       │                                  │
       │ OrgMembership()                  │ 系统管理员标记
       │ Repos() / Repo()                 │
       ▼                                  ▼
┌─────────────────┐     缓存10分钟    ┌─────────────────┐
│  Org Membership │<────────────────>│  Org (IsUser)   │
│  {Member,Admin} │                  │  Organization   │
└─────────────────┘                  └─────────────────┘
       │                                  │
       │                                  │
       ▼                                  ▼
┌─────────────────┐     合并规则     ┌─────────────────┐
│  Repo Perm      │<────────────────>│  Repo Visibility│
│  {Pull,Push,Admin}│                │  public/private │
└─────────────────┘                  └─────────────────┘
       │
       │ 权限检查
       ▼
┌─────────────────┐
│  API Handler    │
└─────────────────┘
```

### 4.3 关键优先级规则

权限判断遵循以下优先级（高→低）：

1. **系统管理员**：`user.Admin == true` → 所有仓库全权限，所有组织全权限
2. **个人组织**：`org.IsUser && org.Name == user.Login` → 组织全权限
3. **仓库可见性**：
   - `public` → 任何人 `Pull=true`
   - `internal` → 已登录用户 `Pull=true`
4. **Forge 同步权限**：数据库中存储的 `perms` 表记录
5. **组织成员关系**：通过 `MembershipService` 实时查询

---

## 五、关键代码位置速查

| 功能 | 文件 | 关键函数/类型 |
|------|------|--------------|
| 用户模型 | `server/model/user.go` | `User` struct |
| 组织模型 | `server/model/org.go` | `Org` struct |
| 权限模型 | `server/model/perm.go` | `Perm`, `OrgPerm` struct |
| 仓库模型 | `server/model/repo.go` | `Repo` struct, `RepoVisibility` |
| 用户登录 | `server/api/login.go` | `HandleAuth`, `updateRepoPermissions` |
| 用户会话 | `server/router/middleware/session/user.go` | `SetUser`, `MustAdmin`, `MustOrgMember` |
| 仓库权限 | `server/router/middleware/session/repo.go` | `SetPerm`, `MustPull`, `MustPush`, `MustRepoAdmin` |
| 组织会话 | `server/router/middleware/session/org.go` | `SetOrg` |
| 成员缓存 | `server/cache/membership.go` | `MembershipService`, `membershipCache` |
| 权限存储 | `server/store/datastore/permission.go` | `PermFind`, `PermUpsert`, `PermPrune` |
| Forge接口 | `server/forge/forge.go` | `Forge` interface (Login, OrgMembership, Repo, Repos) |
| 路由配置 | `server/router/api.go` | `apiRoutes()` |
| Secret模型/服务 | `server/model/secret.go` `server/services/secret/db.go` | `Secret`, `Service`, `SecretListPipeline` |
| 审批隔离 | `server/pipeline/gated.go` `server/pipeline/approve.go` | `needsApproval`, `Approve`, `Decline` |
| Token刷新 | `server/forge/refresh.go` | `Refresher` interface, `Refresh()` (singleflight) |
| Agent模型 | `server/model/agent.go` | `Agent`, `CanAccessRepo`, `IsSystemAgent` |
| Agent认证服务 | `server/rpc/auth_server.go` | `WoodpeckerAuthServer.Auth`, `getAgent` |
| Agent授权拦截器 | `server/rpc/authorizer.go` `server/rpc/jwt_manager.go` | `Authorizer`, `JWTManager.Generate/Verify` |
| Agent权限校验 | `server/rpc/rpc.go` `server/rpc/sanitize.go` | `checkAgentPermissionByWorkflow`, `Next` |
| Agent过滤 | `server/rpc/filter.go` | `createFilterFunc`, 任务标签匹配 |
| Agent客户端 | `agent/rpc/auth_interceptor.go` | `AuthInterceptor`, `scheduleRefreshToken` |
| OAuth申请代码路径 | `server/api/login.go` + `server/forge/github/gitlab/gitea` | `HandleAuth`, `Config.Exchange`, `Forge.Login` 各Provider实现 |
| 仓库重命名/迁移 | `server/api/hook.go` `server/model/repo.go` | `PostHook`, `Redirection`, `Repo.Update`, `ForgeRemoteID` 稳定主键 |
| 用户硬删除清理 | `server/store/datastore/user.go` `server/store/datastore/org.go` `server/store/datastore/repo.go` `server/store/datastore/pipeline.go` | `DeleteUser`, `orgDelete`, `deleteRepo`, `deletePipeline` 五级级联清理 |
| 用户管理API | `server/api/users.go` | `GetUsers/GetUser/PatchUser/PostUser/DeleteUser` |
| 组织查找/创建 | `server/store/datastore/org.go` | `orgFindByName(forgeID+name)`, `OrgLookup` |
| 多Forge隔离查询 | `server/store/datastore/user.go` | `GetUserByRemoteID(forgeID,remoteID)`, `GetUserByLogin(forgeID,login)` |
| GitLab刷新实现 | `server/forge/gitlab/gitlab.go:160-179` | `Refresh()` TokenSource自动刷新 |
| GitHub刷新实现 | `server/forge/github/github.go:158-181` | `Refresh()` 仅RefreshToken非空时触发 |

---

## 七、Forge 外部权限源同步策略与失败兜底机制

### 7.1 同步策略总览

Forge 作为唯一权威权限源，采用"三通道同步 + 多级缓存"架构：

| 同步通道 | 触发时机 | 同步范围 | 缓存策略 | 实现位置 |
|----------|---------|---------|---------|----------|
| 登录全量同步 | 用户登录时 | 全部仓库权限 | 立即写入DB | `server/api/login.go:325-373` |
| 请求增量同步 | 请求仓库资源时 | 单个仓库权限 | TTL=1小时（过期才刷新） | `server/router/middleware/session/repo.go:104-166` |
| 用户手动刷新 | `POST /user/repos/refresh` | 全部仓库权限 | 立即写入DB | `server/api/user.go:216-233` |
| 成员关系查询 | 访问组织资源时 | 单个组织成员身份 | TTL=10分钟 | `server/cache/membership.go:54-70` |

### 7.2 OAuth Token 刷新机制

**刷新流程** (`server/forge/refresh.go:58-101`)

```go
type Refresher interface {
    Refresh(ctx context.Context, u *model.User) (bool, error)
}
// 实现：GitLab, Bitbucket（GitHub/Gitea 永不过期的 token 不实现）
```

```
Refresh(ctx, forge, store, user) 调用链：
    │
    ├─ 检查 Forge 是否实现 Refresher 接口
    │   └─ 否：直接返回（不刷新
    │
    ├─ 检查过期阈值：user.Expiry - 1800秒（提前30分钟
    │   └─ 未过期：直接返回
    │
    ├─ singleflight 去重刷新（per user per user
    │   └─ key = "refresh-{userID}"
    │       ├─ 并发请求中只有第一个实际执行
    │       └─ 其他等待共享结果
    │
    ├─ 执行 Refresh 调用：
    │   ├─ 成功 → 更新 AccessToken/RefreshToken/Expiry
    │   │   ├─ 持久化到数据库 store.UpdateUser
    │   │   └─ 复制到所有等待 goroutine 的 user 对象
    │   │
    │   └─ 失败 → 记录 error log，**不中断**（使用旧 token）
    │
    └─ 兜底：刷新失败仅打日志，后续请求继续使用旧 token
```

### 7.3 仓库权限同步降级策略

**登录时全量同步兜底** (`server/api/login.go:346-373`)

```
updateRepoPermissions:
    │
    ├─ Paginate(Forge.Repos()) 遍历所有 Forge 端仓库
    │   ├─ 单页 Forge 调用失败 →
    │   │   ├─ 抛出错误，同步中止
    │   │   └─ 已同步的部分保留（不回滚）
    │   │
    │   └─ 权限只接受 Push 权限用户 & 白名单过滤
    │       ├─ r.Perm.Admin 才能激活新仓库
    │       └─ 已激活仓库即使 Admin 仍同步 Perm
    │
    └─ PermPrune() 清理：
        └─ 仅清理当前 Forge 的权限（不跨 Forge
```

**请求时增量同步兜底** (`server/router/middleware/session/repo.go:132-154`)

```
SetPerm:
    │
    ├─ 从 DB 读取 Perm（可能过期）
    │   └─ DB 无记录：Perm = zero value（Pull=false Push=false Admin=false
    │
    ├─ Synced 超过 1 小时？
    │   ├─ 是：调用 Forge.Repo() 拉取最新权限
    │   │   ├─ 成功 → PermUpsert 更新，重置 Synced
    │   │   └─ 失败：仅打 warn 日志 → **继续使用旧权限
    │   │
    │   └─ 否：使用 DB 中已有权限（即使过期
    │
    └─ 叠加管理员和可见性规则（见3.3
```

**成员关系缓存降级** (`server/cache/membership.go:54-70`)

```
MembershipService.Get:
    │
    ├─ 检查缓存（ttl = 10分钟
    │   ├─ 命中未过期 → 返回缓存
    │   └─ 未命中/过期：Forge.OrgMembership()
    │       ├─ 成功 → 更新缓存 → 返回结果
    │       └─ 失败：打日志，**返回 {Member:false, Admin:false}
    │
    └─ 失败兜底：无权限
```

### 7.4 手动刷新通道

用户可主动调用 `POST /user/repos/refresh` 强制全量刷新权限：
```go
// server/api/user.go:216-233
RefreshRepos → updateRepoPermissions → 同登录时全量同步逻辑
```

### 7.5 Token 刷新各 Provider 代码路径对比

**Refresher 接口声明** (`server/forge/forge.go`)：

```go
type Refresher interface {
    Refresh(ctx context.Context, u *model.User) (bool, error)
}
```

**各 Provider 实现对比**：

| Forge Provider | 实现 `Refresher`? | Token 类型 | 刷新机制 | 代码位置 |
|--------------|-------------------|------------|---------|----------|
| **GitHub** | ✅ 仅有 RefreshToken 时生效 | Personal access token / OAuth | `oauth2.Config.TokenSource().Token()` 自动检测并刷新 | `server/forge/github/github.go:158-181` |
| **GitLab** | ✅ 完整实现 | OAuth token（2小时过期） | `config.RedirectURL = ""` → `TokenSource().Token()` Go 标准库自动刷新 | `server/forge/gitlab/gitlab.go:160-179` |
| **Gitea** | ✅ 完整实现 | OAuth token | 同 GitLab 流程 | `server/forge/gitea/gitea.go:?` (通过 `oauth2Config` 初始化) |
| **Forgejo** | ✅ 完整实现 | OAuth token | 同 Gitea | `server/forge/forgejo/forgejo.go` |
| **Bitbucket Cloud** | ✅ 完整实现 | OAuth token | 标准 TokenSource 自动刷新 | `server/forge/bitbucket/bitbucket.go` |
| **Bitbucket DC** | ✅ 完整实现 | OAuth token | 标准 TokenSource 自动刷新 | `server/forge/bitbucketdatacenter/bitbucketdatacenter.go` |

**统一刷新调度点** (`server/forge/refresh.go:58-101`)：

```
每次调用 Forge 接口前注入：
    └─ router middleware token.Refresh
        └─ 检查过期阈值：Expiry - 1800秒
            └─ 实现 Refresher? → 是 → singleflight(Refresh)
                ├─ 成功 → user.{AccessToken, RefreshToken, Expiry} 更新后 persist
                └─ 失败 → log.Error 但返回旧 token 继续使用
```

---

## 八、Agent/Runner 通信鉴权机制（独立路径）

Agent 与服务端通过 gRPC 通信，采用与用户侧完全独立的双 Token 鉴权架构。

### 8.1 Agent 类型与权限边界

**Agent 模型** (`server/model/agent.go:26-88`)

```go
type Agent struct {
    ID           int64
    OwnerID      int64             // 创建者用户ID（-1=系统 Agent）
    OrgID        int64             // 所属组织（-1=全局无限制
    Token        string            // 鉴权 token
    CustomLabels map[string]string // 自定义标签过滤
}
```

**三种 Agent 权限范围**：

| 类型 | OwnerID | OrgID | 权限边界 | CanAccessRepo |
|-----|---------|-------|----------|---------------|
| 全局系统 Agent | -1 | -1 | 可处理所有仓库任务 | 返回 true |
| 组织系统 Agent | -1 | N | 仅处理该组织下的仓库任务 | 仅当 OrgID == repo.OrgID |
| 用户注册 Agent | UID | -1/N | 按 OrgID 限制 | 同组织规则 |

### 8.2 双 Token 认证流程

**服务端认证入口** (`server/rpc/auth_server.go:41-99`)

```
Agent 启动 → gRPC 握手
    │
    ▼
1. 初次鉴权：/proto.WoodpeckerAuth/Auth（绕过 JWT 拦截器
    │
    ├─ 模式1：Master Token（WOODPECKER_AGENT_SECRET
    │   ├─ agentID = -1 且 token == MasterToken
    │   │   └─ 自动注册新系统 Agent（OwnerID=-1 OrgID=-1）
    │   │
    │   ├─ agentID > 0 且 token == MasterToken
    │   │   ├─ DB 查 AgentID
    │   │   └─ 要求 agent.IsSystemAgent() == true
    │   │
    │   └─ 不匹配：拒绝
    │
    └─ 模式2：独立 Token（注册时分配
        └─ DB.AgentFindByToken(token)
            ├─ 存在且匹配：鉴权成功
            └─ 不存在：拒绝
    │
    ▼
2. 签发短期 JWT（HS256，有效期 1 小时
    │
    │   jwtManager.Generate(agentID):
    │   claims = {
    │     agent_id: agent.ID,
    │     exp: now + 1h
    │   }
    │
    ▼
3. 后续所有 gRPC 调用带 JWT
    │
    └─ metadata["token"] = accessToken
```

**JWT 拦截器校验** (`server/rpc/authorizer.go:95-146`)

```
gRPC Unary/Stream Interceptor
    │
    ├─ bypass 白名单：/proto.WoodpeckerAuth/Auth → 放行
    │
    ├─ 从 metadata["token"] 取 accessToken
    │   ├─ 缺失 → codes.Unauthenticated
    │   │
    │   └─ jwtManager.Verify(accessToken)
    │       ├─ 签名错误/过期 → codes.Unauthenticated
    │       └─ 成功 → claims.AgentID 注入 context（agentIDKey
    │
    └─ context 中带 agent_id → 业务处理层可查询 Agent 对象
```

**Agent 客户端自动续期** (`agent/rpc/auth_interceptor.go:26-115`)

```go
// agent 端 AuthInterceptor
scheduleRefreshToken(ctx, refreshInterval = 30分钟)
    │
    ├─ 立即 refreshToken() 首次获取
    │
    └─ goroutine 循环：
        ├─ 正常：每 refreshInterval 前刷新
        └─ 失败：退避 1 秒后重试
```

### 8.3 任务分配阶段权限校验（双保险）

**第一层：任务调度过滤** (`server/rpc/rpc.go:63-108` `Next`)

```
Agent 调用 Next(agentFilter)：
    │
    ├─ getAgentFromContext(ctx) 从 JWT 查 Agent
    │   └─ NoSchedule=true → 空转不拿任务
    │
    ├─ 强制注入服务端标签：agent.GetServerLabels()
    │   ├─ OrgID = -1 → {"org_id": "*"} 匹配所有组织
    │   └─ OrgID = N  → {"org_id": "N"} 仅匹配该组织
    │
    └─ scheduler.Poll(filterFn)
        └─ filterFn = createFilterFunc(agentFilter)
            ├─ 任务 label 必须被 agent labels 包含
            ├─ 支持 ! 前缀排除匹配
            └─ 支持 wildcard * 匹配
```

**第二层：Workflow 执行前二次校验** (`server/rpc/sanitize.go:32-67` `checkAgentPermissionByWorkflow`)

所有会修改状态的 RPC 方法 (Init/Update/Done/Log/Wait/Extend 均调用：

```
checkAgentPermissionByWorkflow(agent, workflowID, pipeline, repo)
    │
    ├─ 加载 workflow → pipeline → repo 完整链路
    │
    └─ agent.CanAccessRepo(repo)
        ├─ 全局 Agent (OrgID=-1): return true
        ├─ 组织 Agent (OrgID=N): match repo.OrgID == N
        └─ 不匹配: return ErrAgentIllegalRepo
```

### 8.4 Agent 状态管理权限

| RPC 方法 | Agent 范围权限 | 实现 |
|---------|--------------|------|
| RegisterAgent | 所有已鉴权 Agent，更新 agent.Backend/Platform/Capacity/CustomLabels | `rpc.RegisterAgent` |
| UnregisterAgent | **仅系统 Agent** (IsSystemAgent) 才从 DB 删除，用户注册不删除 | `rpc.UnregisterAgent` |
| ReportHealth | 所有 Agent 更新 LastContact | `rpc.ReportHealth` |

### 8.5 gRPC 权限边界总结

```
┌────────────────────────────────────────────────────────┐
│                    gRPC 服务端                             │
├────────────────────────────────────────────────────────┤
│  /proto.WoodpeckerAuth/Auth  [BYPASS]                   │
│    └─ MasterToken / 独立 Token 鉴权 → 签发 JWT          │
├────────────────────────────────────────────────────────┤
│  /proto.WoodpeckerServer/*  [JWT拦截]                   │
│    ├─ Next / Wait / Extend                              │
│    │   └─ 调度器过滤(标签+Org) + 二次校验(CanAccessRepo │
│    ├─ Init / Update / Done / Log                        │
│    │   └─ checkAgentPermissionByWorkflow 强制校验       │
│    └─ Register / Unregister / ReportHealth             │
│        └─ UnregisterAgent 限系统 Agent                 │
└────────────────────────────────────────────────────────┘
```

---

## 九、全系统权限体系全景图

```
                            ┌──────────────────────────┐
                            │   用户浏览器 / API 调用    │
                            └───────────┬──────────────┘
                                        │  HTTP / REST
                                        ▼
┌───────────────────────────────────────────────────────────────────────┐
│                        用户侧 HTTP 权限链                                │
│  SetUser (JWT会话) → token.Refresh (OAuth续期)                        │
│       │                                                               │
│       ├── SetOrg + MustOrgMember → Org Membership Cache (10min)      │
│       │                                                               │
│       └── SetRepo + SetPerm → Pull/Push/Admin (1h 增量同步)          │
│              │                                                        │
│              ├── 3.6 Secret 三级隔离 (Global/Org/Repo 优先级)        │
│              └── 3.7 Gated Pipeline 审批 (RequireApproval 策略)      │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
                            ┌──────────────────────────┐
                            │   Forge 外部权限源          │
                            │  (GitHub/GitLab/Gitea)    │
                            │                           │
                            │  OAuth Token 刷新机制       │
                            │  Repos() 全量同步          │
                            │  Repo() 增量同步           │
                            │  OrgMembership() 成员查询  │
                            └───────────┬──────────────┘
                                        │  gRPC (独立通道)
                                        ▼
┌───────────────────────────────────────────────────────────────────────┐
│                       Agent 侧 gRPC 权限链                               │
│                                                                       │
│  Phase1: 初始认证（绕过 JWT 拦截）                                     │
│  └─ WoodpeckerAuth/Auth → MasterToken / 独立 Token 鉴权              │
│         └→ 签发短期 JWT (HS256, 1h 有效期)                           │
│                                                                       │
│  Phase2: 后续所有调用（JWT 拦截器）                                    │
│  └─ Authorizer 拦截器 → 验证 metadata["token"] → 注入 agent_id       │
│         │                                                             │
│         ├── Next: 调度过滤（Org Label + 自定义标签）                  │
│         ├── Init/Update/Done/Log/Wait/Extend:                        │
│         │    └─ checkAgentPermissionByWorkflow → CanAccessRepo       │
│         └── UnregisterAgent: 限系统 Agent                            │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 十、组织/仓库重命名与迁移的权限映射，及用户清理残留机制

### 10.1 仓库重命名与跨组织迁移的权限重映射

**稳定主键设计原则**：

Woodpecker 以 `ForgeRemoteID`（Forge 端的仓库/用户/组织的不可变内部 ID）作为稳定关联键，而非依赖可变的 `Owner/Name/FullName` 字符串。

**重命名检测与处理** (`server/api/hook.go:177-191`)：

```
Webhook 触发 → PostHook handler：
    │
    ├─ 步骤 1：HookToken 解析出仓库后，比较：
    │   repo.FullName(当前) vs repoFromForge.FullName(来自webhook)
    │
    ├─ 步骤 2：不一致则记录重定向：
    │   _store.CreateRedirection({RepoID: repo.ID, FullName: repo.FullName})
    │   └─ 历史URL → 新URL 自动跳转（Redirection 表
    │
    ├─ 步骤 3：repo.Update(repoFromForge) 更新本地元数据：
    │   {ForgeRemoteID, Owner, Name, FullName, Avatar, ForgeURL,
    │    Clone, CloneSSH, Branch, Visibility, IsSCMPrivate}
    │   └─ Visibility 仅当 forge 返回非空才更新（防止GitLab等部分事件丢失
    │
    └─ 步骤 4：_store.UpdateRepo(repo) 持久化
```

**组织迁移自动映射** (`server/api/repo.go:125-151`)：

```
激活/重新同步仓库时 PostRepo：
    │
    ├─ 根据 repo.Owner + user.ForgeID → OrgFindByName
    │   ├─ 命中：orgID 关联
    │   └─ 未命中：
    │       ├─ 调用 Forge.Org(c, user, repo.Owner) 查 Forge 端组织信息
    │       └─ OrgCreate：新建组织，关联 ForgeID + Name
    │
    └─ 结果：repo.OrgID = org.ID（权限关联保持正确
```

**用户登录时个人组织重映射** (`server/api/login.go:221-268`)：

```
HandleAuth 回调阶段：
    │
    ├─ user.Login(来自Forge的新用户名) ≠ org.Name(数据库中旧名)?
    │   └─ 是：OrgUpdate → org.Name = user.Login（更新组织名匹配新用户名）
    │
    └─ 个人组织始终跟随用户名变化，保持 IsUser=true
```

### 10.2 用户硬删除的级联清理路径

**删除入口** (`server/api/users.go:198-224` `DeleteUser`)：

```
DELETE /users/:login → DeleteUser：
    │
    └─ _store.DeleteUser(user) （事务，全有或全无
        │
        ▼
        DeleteUser 事务 (`server/store/datastore/user.go:88-108`):
        ├─ BEGIN
        │
        ├─ [1] s.orgDelete(sess, user.OrgID) 级联删除个人组织：
        │   │
        │   ▼ orgDelete (`server/store/datastore/org.go:57-74`):
        │   ├─ 删组织级 Secret: DELETE secrets WHERE org_id=?
        │   │
        │   ├─ 列出所有组织下的仓库：SELECT repos WHERE org_id=?
        │   │   └─ 逐个 repo → 调用 deleteRepo(sess, repo)
        │   │       │
        │   │       ▼ deleteRepo (`server/store/datastore/repo.go:105-141`):
        │   │       ├─ DELETE configs     WHERE repo_id=?
        │   │       ├─ DELETE perms       WHERE repo_id=?
        │   │       ├─ DELETE registries  WHERE repo_id=?
        │   │       ├─ DELETE secrets     WHERE repo_id=?
        │   │       ├─ DELETE redirections WHERE repo_id=?
        │   │       │
        │   │       ├─ 分批（每次50）取 pipelineIDs
        │   │       │   └─ 逐个 pipeline → deletePipeline(sess, pipelineID)
        │   │       │       │
        │   │       │       ▼ deletePipeline (`server/store/datastore/pipeline.go:219-245`):
        │   │       │       ├─ workflowsDelete() → 级联 steps + logs
        │   │       │       ├─ 查 pipeline_configs 关联的 config_id
        │   │       │       │   └─ 若该 config 只被此 pipeline 使用 → 删 configs
        │   │       │       ├─ DELETE pipeline_configs WHERE pipeline_id=?
        │   │       │       └─ DELETE pipelines WHERE id=?
        │   │       │
        │   │       └─ DELETE repos WHERE id=?
        │   │
        │   └─ DELETE orgs WHERE id=?
        │
        ├─ [2] DELETE users WHERE id=?
        │
        ├─ [3] DELETE perms WHERE user_id=? （清理用户的所有仓库权限
        │
        └─ COMMIT / ROLLBACK
```

### 10.3 清理覆盖范围与残留分析

| 数据类型 | 清理方式 | 触发点 | 可能的残留 |
|---------|----------|-------|-----------|
| 用户账号 | 硬 DELETE | DeleteUser | 无 |
| 个人组织 | 硬 DELETE（级联）| DeleteUser | 若 OrgID=-1 或关联失败可能残留 |
| 用户-仓库权限表 perms | 硬 DELETE（按 user_id）| DeleteUser | 无（事务保证 |
| 仓库完整数据（configs/perm/secret/redirection/pipelines）| 硬 DELETE 级联 | orgDelete → deleteRepo → deletePipeline | 无（分批删除保证大量流水线完整清理）|
| 流水线运行日志 | workflowsDelete → stepsDelete → logsDelete | deletePipeline 级联 | 无 |
| 组织级 Secrets | 直接 DELETE | orgDelete | 无 |
| 组织下仓库级 Secrets | 随 deleteRepo 删除 | orgDelete → deleteRepo | 无 |
| Agent 注册数据 | 不随用户删除 | Agent 独立生命周期 | ✅ 可能残留（OwnerID 字段留空）|
| TaskLog 审计表（若存在）| 无关联清理 | 独立日志表 | ✅ 可能残留 |
| membership 缓存 | Redis/Memory TTL 自动过期 | TTL 10分钟 | ✅ 过期前有窗口 |

### 10.4 跨 Forge 实例迁移的权限隔离

多 Forge 场景的权限键组合：

```go
// 唯一性约束（数据库层面
// users 表唯一索引：(forge_id, forge_remote_id)
// orgs  表唯一索引：(forge_id, name)
// repos 表唯一索引：(forge_id, forge_remote_id)
```

```
同一用户名在不同 Forge 实例：
    │
    ├─ forge_id=1 (GitHub)  → userA（ForgeRemoteID=123）→ orgA(org_id=1)
    ├─ forge_id=2 (GitLab)  → userA（ForgeRemoteID=456）→ orgA(org_id=2)  ← 重名但不同Forge
    │
    └─ 权限查询：
        ├─ GetUserByRemoteID(forgeID, remoteID) → 精确匹配
        ├─ GetUserByLogin(forgeID, login)       → 带 forgeID 过滤
        └─ OrgFindByName(name, forgeID)         → forgeID 确保跨实例隔离
```

---

## 六、设计特点总结

1. **权限源分离**：系统管理员权限、组织成员权限、仓库权限分别来自不同数据源
2. **以 Forge 为权威**：仓库权限最终以 Forge 端为准，本地仅做缓存
3. **多层缓存机制**：组织成员缓存10分钟，仓库权限缓存1小时
4. **权限提升规则**：系统管理员和可见性规则作为额外权限层叠加
5. **中间件链式校验**：通过 Gin 中间件实现声明式权限检查
6. **用户即组织**：每个用户自动拥有一个同名个人组织，统一资源归属模型
7. **失败优先可用**：Forge 同步失败不中断，继续使用过期权限或降级为无权限，避免雪崩
8. **双轨鉴权架构**：用户侧 HTTP/JWT 会话与 Agent 侧 gRPC 双 Token 体系完全隔离
9. **任务分配双保险**：调度器标签过滤 + Workflow 执行前二次 Org 校验，防越权执行
10. **Secret 分层覆盖**：Global/Org/Repo 三级作用域，同名优先级从细到粗
11. **Pipeline 门控审批**：按事件类型（PR/fork/all）分级审批策略，白名单用户自动放行
12. **稳定主键迁移容忍**：以 ForgeRemoteID 为不可变关联键，重命名/跨组织迁移通过 Redirection 表保持访问连通性
13. **硬删除级联清理**：DeleteUser 单事务级联清理用户→组织→仓库→流水线→步骤日志五层结构，保证数据一致性
14. **多 Forge 实例隔离**：所有查询强制带 forge_id，同名用户/组织在不同 Forge 实例完全隔离不串权
