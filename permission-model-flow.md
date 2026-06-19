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

---

## 六、设计特点总结

1. **权限源分离**：系统管理员权限、组织成员权限、仓库权限分别来自不同数据源
2. **以 Forge 为权威**：仓库权限最终以 Forge 端为准，本地仅做缓存
3. **多层缓存机制**：组织成员缓存10分钟，仓库权限缓存1小时
4. **权限提升规则**：系统管理员和可见性规则作为额外权限层叠加
5. **中间件链式校验**：通过 Gin 中间件实现声明式权限检查
6. **用户即组织**：每个用户自动拥有一个同名个人组织，统一资源归属模型
