# Woodpecker Secret 注入与作用域判定处理路径分析

## 1. 数据模型与存储结构

### 1.1 Secret 数据模型

定义位置：`server/model/secret.go:47-56`

```go
type Secret struct {
    ID     int64          `json:"id"`
    OrgID  int64          `json:"org_id"`    // 组织 ID，0 表示非组织级
    RepoID int64          `json:"repo_id"`   // 仓库 ID，0 表示非仓库级
    Name   string         `json:"name"`      // Secret 名称（唯一约束：org_id + repo_id + name）
    Value  string         `json:"value"`     // Secret 值（敏感字段）
    Images []string       `json:"images"`    // 允许使用的插件镜像列表
    Events []WebhookEvent `json:"events"`    // 允许触发的事件类型
    Note   string         `json:"note"`      // 备注
}
```

### 1.2 作用域判定方法

定义位置：`server/model/secret.go:69-81`

| 作用域       | OrgID | RepoID | 判定方法                |
|-------------|-------|--------|------------------------|
| Global（全局）| 0     | 0      | `IsGlobal()`           |
| Org（组织级） | ≠0    | 0      | `IsOrganization()`     |
| Repo（仓库级）| 0     | ≠0     | `IsRepository()`       |

### 1.3 存储层查询

定义位置：`server/store/datastore/secret.go:32-40`

`SecretList(repo, includeGlobalAndOrgSecrets, p)` 方法根据 `includeGlobalAndOrgSecrets` 参数决定查询范围：
- `false`：仅查询 `repo_id = repo.ID` 的仓库级 Secret
- `true`：查询 `repo_id = repo.ID` OR `org_id = repo.OrgID` OR (`org_id = 0` AND `repo_id = 0`)，即仓库级 + 所属组织级 + 全局

---

## 2. 分层取值与优先级

### 2.1 Secret Service 构建链

#### 服务初始化
定义位置：`server/services/setup.go:57-67`

```
setupSecretService(store, endpoint, client, includeNetrc)
    ↓
若配置了全局 secret-extension-endpoint
    ↓
secret.NewCombined(secret.NewDB(store), secret.NewHTTP(endpoint, client, includeNetrc))
    ↓
否则
    ↓
secret.NewDB(store)
```

#### 仓库级 Service 选择
定义位置：`server/services/manager.go:104-110`

```
SecretServiceFromRepo(repo)
    ↓
若 repo.SecretExtensionEndpoint != ""
    ↓
返回 combined(base + repo 级 HTTP extension)
    ↓
否则
    ↓
返回全局 SecretService()
```

### 2.2 Pipeline 执行时的 Secret 聚合

#### DB 层去重逻辑
定义位置：`server/services/secret/db.go:41-72`

`SecretListPipeline` 方法按以下优先级去重：
1. **Repository 级**（最高优先级）
2. **Organization 级**
3. **Global 级**（最低优先级）

同名 Secret 只保留高优先级的。

#### Combined 层合并逻辑
定义位置：`server/services/secret/combined.go:37-74`

```
baseSecrets (DB 层返回的已去重 secrets)
    +
extensionSecrets (HTTP extension 返回的 secrets)
    ↓ 同名时 extension 优先
merged secrets
```

### 2.3 Pipeline 解析时获取 Secret

调用位置：`server/pipeline/items.go:51-70`

```go
secretService := server.Config.Services.Manager.SecretServiceFromRepo(repo)
secs, err := secretService.SecretListPipeline(ctx, repo, currentPipeline, netrc)
```

获取到的 `[]*model.Secret` 被转换为 `[]compiler.Secret`，其中：
- `sec.Name` → `compiler.Secret.Name`
- `sec.Value` → `compiler.Secret.Value`
- `sec.Images` → `compiler.Secret.AllowedPlugins`
- `sec.Events` → `compiler.Secret.Events`

---

## 3. Secret 注入流程

### 3.1 Compiler 选项注入

定义位置：`pipeline/frontend/yaml/compiler/option.go:62-68`

```go
func WithSecret(secrets ...Secret) Option {
    return func(compiler *Compiler) {
        for _, secret := range secrets {
            compiler.secrets[strings.ToLower(secret.Name)] = secret
        }
    }
}
```

Secret 名称被转为小写存储在 `compiler.secrets` map 中。

### 3.2 Step 创建时的 Secret 检查与注入

定义位置：`pipeline/frontend/yaml/compiler/convert.go:101-125`

每个 Step 创建时都会初始化一个 `getSecretValue` 闭包：

```go
getSecretValue := func(name string) (string, error) {
    name = strings.ToLower(name)
    secret, ok := c.secrets[name]
    if !ok {
        return "", fmt.Errorf("secret %q not found", name)
    }
    event := c.metadata.Curr.Event
    err := secret.Available(event, container)  // 关键：作用域检查
    if err != nil {
        return "", err
    }
    return secret.Value, nil
}
```

### 3.3 YAML 配置中的 Secret 引用解析

#### from_secret 语法
定义位置：`pipeline/frontend/yaml/compiler/settings/params.go:178-191`

```yaml
steps:
  - name: deploy
    image: plugins/docker
    settings:
      password:
        from_secret: docker_password  # 引用 Secret
```

解析过程：
1. `injectSecret()` 检测 map 中是否包含 `from_secret` key
2. 调用 `getSecretValue(name)` 获取值（会触发作用域检查）
3. 支持递归解析复杂嵌套结构（map/slice）：`injectSecretRecursive()`

#### Secret 注入位置
- **Plugin Settings**：`settings.ParamsToEnv(container.Settings, environment, "PLUGIN_", true, getSecretValue, secretMapping)`
- **Environment 变量**：`settings.ParamsToEnv(container.Environment, environment, "", false, getSecretValue, secretMapping)`

### 3.4 SecretMapping 记录

定义位置：`pipeline/frontend/yaml/compiler/settings/params.go:39-53`

每次 Secret 被成功解析时，会记录到 `secretMapping` 中：
- Key：最终注入的环境变量名（如 `PLUGIN_PASSWORD`）
- Value：Secret 的实际值

`secretMapping` 会被附加到 `backend_types.Step.SecretMapping` 字段。

---

## 4. Plugin 字段限制（AllowedPlugins/Images）

### 4.1 限制检查入口

定义位置：`pipeline/frontend/yaml/compiler/compiler.go:49-64`

```go
func (s *Secret) Available(event metadata.Event, container *yaml_types.Container) error {
    onlyAllowSecretForPlugins := len(s.AllowedPlugins) > 0
    
    // 检查 1：如果设置了 Images 限制，该 Step 必须是 Plugin
    if onlyAllowSecretForPlugins && !container.IsPlugin() {
        return fmt.Errorf("secret %q is only allowed to be used by plugins...", s.Name)
    }
    
    // 检查 2：镜像匹配
    if onlyAllowSecretForPlugins && !utils.MatchImageDynamic(container.Image, s.AllowedPlugins...) {
        return fmt.Errorf("secret %q is not allowed to be used with image %q...", s.Name, container.Image)
    }
    
    // 检查 3：事件类型匹配
    if !s.Match(event) {
        return fmt.Errorf("secret %q is not allowed to be used with pipeline event %q", s.Name, event)
    }
    
    return nil
}
```

### 4.2 镜像匹配逻辑 MatchImageDynamic

定义位置：`pipeline/frontend/yaml/utils/image.go:66-81`

匹配规则：
- **若限制列表中的镜像带 tag**（如 `plugins/docker:20`）：需要完全匹配（包括 tag）
- **若限制列表中的镜像不带 tag**（如 `plugins/docker`）：忽略 tag，只匹配镜像名

```go
func MatchImageDynamic(from string, to ...string) bool {
    fullFrom := expandImage(from)    // 补全 tag（默认 latest）
    trimFrom := trimImage(from)       // 去掉 tag
    for _, match := range to {
        if imageHasTag(match) {
            if fullFrom == expandImage(match) { return true }  // 精确匹配
        } else {
            if trimFrom == trimImage(match) { return true }    // 忽略 tag 匹配
        }
    }
    return false
}
```

### 4.3 事件类型匹配

定义位置：`pipeline/frontend/yaml/compiler/compiler.go:68-79`

```go
func (s *Secret) Match(event metadata.Event) bool {
    if len(s.Events) == 0 {
        return true  // 未设置则匹配所有事件
    }
    if event.IsPull() {
        event = metadata.EventPull  // 所有 pull_request 相关事件统一为 pull
    }
    return slices.Contains(s.Events, event)
}
```

---

## 5. 脱敏处理

### 5.1 脱敏规则 NewSecretsReplacer

定义位置：`pipeline/shared/replace_secrets.go:24-47`

```go
func NewSecretsReplacer(secrets []string) *strings.Replacer {
    const minStringLength = 3  // 长度 <=3 的字符串不脱敏
    for _, old := range secrets {
        old = strings.TrimSpace(old)
        if len(old) <= minStringLength {
            continue
        }
        // 多行 Secret 按行拆分处理
        for _, part := range strings.Split(old, "\n") {
            if len(part) == 0 { continue }
            oldNew = append(oldNew, part)
            oldNew = append(oldNew, "********")  // 替换为 8 个星号
        }
    }
    return strings.NewReplacer(oldNew...)
}
```

### 5.2 脱敏 Secret 来源

定义位置：`pipeline/frontend/yaml/compiler/compiler.go:136-142`

Compiler 编译阶段，**所有** Secret（无论是否被某个 Step 实际引用）都会被收集到 `config.Secrets`：

```go
for _, sec := range c.secrets {
    config.Secrets = append(config.Secrets, &backend_types.Secret{
        Name:  sec.Name,
        Value: sec.Value,
    })
}
```

### 5.3 Agent 侧日志脱敏

定义位置：`agent/logger.go:30-52` 和 `agent/log/line_writer.go:52-56`

```
Runner.createLogger()
    ↓
从 workflow.Config.Secrets 收集所有 Secret.Value
    ↓
log.NewLineWriter(peer, step.UUID, secrets...)
    ↓
内部用 shared.NewSecretsReplacer(secrets) 创建 replacer
    ↓
LineWriter.Write() 时：data = w.replacer.Replace(data)
    ↓
脱敏后的数据发送到 Server
```

### 5.4 API 返回脱敏

定义位置：`server/model/secret.go:125-135`

所有 API 响应（列表/单个）都调用 `Secret.Copy()` 方法，该方法**不包含 Value 字段**：

```go
func (s *Secret) Copy() *Secret {
    return &Secret{
        ID:     s.ID,
        OrgID:  s.OrgID,
        RepoID: s.RepoID,
        Name:   s.Name,
        Images: s.Images,
        Events: sortEvents(s.Events),
        Note:   s.Note,
        // 注意：Value 字段被省略
    }
}
```

涉及的 API 处理：
- `server/api/repo_secret.go`：仓库级 Secret API
- `server/api/org_secret.go`：组织级 Secret API
- `server/api/global_secret.go`：全局 Secret API

### 5.5 数据库加密（可选）

定义位置：`server/services/encryption/wrapper/store/secret_store_wrapper.go` 和 `server/services/encryption/wrapper/store/secret_store.go`

`EncryptedSecretStore` 作为 `model.SecretStore` 的包装器：
- **写入前加密**：`encrypt()` 使用 secret.ID 作为关联数据，通过 EncryptionService 加密 Value
- **读取后解密**：`decrypt()` 解密 Value
- 支持加密启用和密钥迁移

> **注意**：当前代码中此功能被注释禁用（`server/services/setup.go:58-64`），标注为 TODO。

---

## 6. 完整处理路径总结

```
┌──────────────────────────────────────────────────────────────────────┐
│  1. API 创建 Secret (Repo/Org/Global 三级)                            │
│     server/api/*_secret.go → secret.Service → store.SecretStore      │
│     写入前：Validate() 校验 name/value/images/events 格式             │
│     响应时：Copy() 脱敏 Value                                         │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│  2. Pipeline 触发，解析流水线配置                                      │
│     server/pipeline/items.go:parsePipeline()                         │
│     → Manager.SecretServiceFromRepo(repo)  选择 service 实现         │
│     → SecretListPipeline()  聚合分层 secrets 并按优先级去重           │
│       (Repository > Organization > Global > HTTP Extension)          │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│  3. Compiler 编译 YAML                                                │
│     pipeline/frontend/yaml/compiler/compiler.go:Compile()            │
│     → WithSecret() 存入 compiler.secrets (name 小写化)               │
│     → createProcess() 为每个 Step 创建 getSecretValue 闭包           │
│       闭包内调用 Secret.Available() 进行三重检查：                    │
│         ① Images 设置则必须是 Plugin Step                             │
│         ② MatchImageDynamic() 镜像匹配                                │
│         ③ Match() 事件类型匹配                                        │
│     → settings.ParamsToEnv() 递归解析 from_secret 引用               │
│       记录 secretMapping: env_var_name → secret_value                │
│     → 所有 secrets 收集到 config.Secrets 用于日志脱敏                │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│  4. Agent 执行 Step                                                   │
│     agent/logger.go → log.NewLineWriter(secrets...)                  │
│     → shared.NewSecretsReplacer() 创建替换器                         │
│       规则: len>3, 多行拆分, 替换为 "********"                       │
│     → LineWriter.Write() 每行日志脱敏后上报 Server                   │
└──────────────────────────────────────────────────────────────────────┘
```
