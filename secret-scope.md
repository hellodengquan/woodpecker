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

#### 继承覆盖机制
Secret 采用**层级继承 + 同名覆盖**策略。低作用域的 Secret 可以**覆盖**高作用域的同名 Secret，也可以**继承**未被覆盖的上层 Secret。

例如：
- 全局定义了 `DOCKER_TOKEN=global-value` 和 `NPM_TOKEN=global-value`
- 组织级定义了 `DOCKER_TOKEN=org-value`
- 仓库级定义了 `NPM_TOKEN=repo-value`

最终仓库实际可用：
- `DOCKER_TOKEN=org-value`（组织覆盖全局）
- `NPM_TOKEN=repo-value`（仓库覆盖全局）

#### DB 层去重逻辑
定义位置：`server/services/secret/db.go:41-72`

`SecretListPipeline` 方法按以下优先级去重（从高到低）：
1. **Repository 级**（最高优先级）
2. **Organization 级**
3. **Global 级**（最低优先级）

```go
// 按作用域优先级覆盖
for _, s := range allSecrets {
    if _, exists := mergedSecrets[s.Name]; !exists {
        mergedSecrets[s.Name] = s  // 先出现的（高优先级）保留
    }
}
```

同名 Secret 只保留高优先级的。

#### Combined 层合并逻辑
定义位置：`server/services/secret/combined.go:37-74`

```
baseSecrets (DB 层返回的已去重 secrets)
    +
extensionSecrets (HTTP extension 返回的 secrets)
    ↓ 同名时 extension 优先（extension 优先级最高）
merged secrets
```

### 2.3 HTTP Extension 健康检查与错误处理

定义位置：`server/services/secret/http.go:47-69`

HTTP Extension **无独立健康检查机制**，采用请求时即时验证模式：

```go
func (h *httpExtension) SecretListPipeline(ctx context.Context, repo *model.Repo, pipeline *model.Pipeline, netrc *model.Netrc) ([]*model.Secret, error) {
    status, err := h.client.Send(ctx, http.MethodPost, h.endpoint, body, response)
    if err != nil && status != http.StatusNoContent {
        return nil, fmt.Errorf("failed to fetch secrets via http (%d) %w", status, err)
    }
    if status != http.StatusOK {
        return nil, nil  // 204 No Content 视为无额外 secrets，不报错
    }
    return response.Secrets, nil
}
```

行为分析：
| 状态码 | 行为 |
|--------|------|
| 200 OK | 正常返回 secrets |
| 204 No Content | 返回空切片 `nil`，表示无额外 secrets |
| 其他状态码（4xx/5xx）或网络错误 | **直接返回 error，Pipeline 解析失败终止** |

> ⚠️ **风险**：HTTP Extension 无超时熔断、重试或降级机制。Extension 服务不可用会导致整个 Pipeline 无法启动。若配置了仓库级 extension，优先级还会覆盖全局 extension，两者同时不可用时直接失败。

### 2.4 Pipeline 解析时获取 Secret

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

#### 匹配规则
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

#### 无 tag 处理边界情况

当 pipeline 中使用的镜像不带 tag 时，`expandImage()` 会自动补全 `:latest`，但匹配策略仍根据限制列表是否带 tag 来决定：

| Pipeline 使用的镜像 | Secret.Images 限制 | 匹配结果 | 说明 |
|-------------------|-------------------|---------|------|
| `plugins/docker` | `plugins/docker` | ✅ 匹配 | 均无 tag，走 trimImage 匹配 |
| `plugins/docker` | `plugins/docker:latest` | ✅ 匹配 | 限制带 tag，from 补全为 latest 后精确匹配 |
| `plugins/docker` | `plugins/docker:20` | ❌ 不匹配 | 限制指定 tag=20，from 补全为 latest，不一致 |
| `plugins/docker:20` | `plugins/docker` | ✅ 匹配 | 限制无 tag，走 trimImage 忽略 tag 匹配 |
| `plugins/docker:20` | `plugins/docker:20` | ✅ 匹配 | 均带 tag，精确匹配 |
| `plugins/docker:20` | `plugins/docker:latest` | ❌ 不匹配 | 都带 tag，精确匹配失败 |

> ⚠️ **注意**：`imageHasTag()` 判断依据是镜像字符串中是否包含冒号 `:`（且冒号后不是端口号）。因此像 `docker.io/myrepo/plugin:v1` 这种带自定义 registry 和 tag 的场景也能正确识别。`trimImage()` 会把完整镜像路径（含 registry）保留，只去掉末尾 `:tag`。

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

#### 前缀子串误伤风险

由于底层使用 `strings.NewReplacer`，该实现采用**最长匹配优先**但仍然是**子串匹配**。这带来以下潜在问题：

| 场景 | Secret 值 | 日志内容 | 脱敏结果 | 是否误伤 |
|------|----------|---------|---------|---------|
| 前缀重叠 | `mysecretpass123` 和 `mysecret` | `mysecretpass123 was used` | `******** was used` | ✅ 正确（长的优先匹配） |
| Secret 是通用词 | `build` | `build started, build finished` | `******** started, ******** finished` | ⚠️ 误伤 |
| Secret 是路径片段 | `/app/config` | `Reading /app/config/prod.yaml` | `Reading ********/prod.yaml` | ⚠️ 误伤 |
| Secret 含特殊字符 | `pass$word` | `pass$word` vs `passUSDword` | 取决于正则匹配能力 | 可能误伤 |
| Base64 前缀 | `cGFzc3dvcmQ=` (Base64) | 含相同前缀的其他 Base64 | 相同前缀被替换 | ⚠️ 误伤 |

测试用例参考：`pipeline/shared/replace_secrets_test.go:23-63`

> ⚠️ **风险点**：
> 1. 长度 ≤ 3 的 Secret 不脱敏（如 `abc`、`123`），可能直接泄露
> 2. 多行 Secret 按行拆分后分别脱敏，若某一行内容恰好是通用词，会造成日志中该词全部被替换
> 3. `strings.Replacer` 基于简单子串替换，不做词边界检测，可能误替换包含相同字符序列的合法内容
> 4. 脱敏替换符固定为 `********`（8 个星号），无法区分泄露的是哪个 Secret

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

### 5.5 数据库加密（可选）与密钥轮换

定义位置：`server/services/encryption/wrapper/store/secret_store_wrapper.go`、`server/services/encryption/wrapper/store/secret_store.go`、`server/services/encryption/tink_state.go`

`EncryptedSecretStore` 作为 `model.SecretStore` 的包装器：
- **写入前加密**：`encrypt()` 使用 secret.ID 作为关联数据（AAD），通过 EncryptionService 加密 Value
- **读取后解密**：`decrypt()` 解密 Value
- 支持加密启用、禁用和密钥迁移

#### 密钥轮换流程

Woodpecker 支持两种加密后端（AES 和 Tink），均实现了完整的密钥轮换：

定义位置：`server/services/encryption/tink_state.go:123-131`（Tink 后端示例）

```
密钥文件变更检测（fsnotify watcher）
    ↓
validateKeyset() 检测到 errEncryptionKeyRotated
    ↓
updateCiphertextSample()  更新密钥验证样本
    ↓
callbackOnRotation()  对所有已注册 client 执行 MigrateEncryption()
    ↓
EncryptedSecretStore.MigrateEncryption(newEncryptionService)
    ↓
SecretListAll() 取出所有 secrets
    ↓
旧密钥解密 → 新密钥加密
    ↓
SecretUpdate() 逐条写回
```

定义位置：`server/services/encryption/wrapper/store/secret_store_wrapper.go:67-88`

```go
func (wrapper *EncryptedSecretStore) MigrateEncryption(newEncryptionService types.EncryptionService) error {
    secrets, err := wrapper.store.SecretListAll()
    if err != nil {
        return fmt.Errorf(errMessageTemplateFailedToMigrate, err)
    }
    for _, secret := range secrets {
        err := wrapper.decrypt(secret)            // 旧密钥解密
        if err != nil { return err }
        wrapper.encryption = newEncryptionService  // 切换到新密钥
        err = wrapper.encrypt(secret)              // 新密钥加密
        if err != nil { return err }
        _, err = wrapper.store.SecretUpdate(secret) // 写回
        if err != nil { return err }
    }
    return nil
}
```

> **注意**：当前代码中此功能被注释禁用（`server/services/setup.go:58-64`），标注为 TODO。

#### Secret 改动回滚

定义位置：`server/services/encryption/wrapper/store/secret_store.go:52-71`

在启用加密存储时，`SecretCreate` 操作采用**先落库再加密再更新**的两阶段写入，若加密或更新失败则执行回滚：

```go
// 阶段 1: 先以明文创建（临时记录）
newSecret, err := wrapper.store.SecretCreate(secret)
// 阶段 2: 加密
err = wrapper.encrypt(secret)
if err != nil {
    // 加密失败 → 回滚：删除刚创建的临时记录
    deleteErr := wrapper.store.SecretDelete(newSecret)
    if deleteErr != nil {
        return fmt.Errorf("failed creating secret: %w. Also failed deleting temporary secret record from store: %s", err, deleteErr.Error())
    }
    return err
}
// 阶段 3: 更新为加密后的密文
err = wrapper.store.SecretUpdate(secret)
if err != nil {
    deleteErr := wrapper.store.SecretDelete(newSecret)  // 同样回滚
    ...
}
```

> ⚠️ **局限**：回滚只覆盖 `SecretCreate`，`SecretUpdate` 和 `SecretDelete` 操作**无审计日志、无版本历史、无回滚能力**。Secret 被修改或删除后原始值不可逆。

---

## 6. Plugin 子镜像 Trust 链

### 6.1 Trusted Clone Plugins

定义位置：`shared/constant/constant.go:33-38`

Woodpecker 维护了一个内置的可信克隆插件列表：

```go
var TrustedClonePlugins = []string{
    DefaultClonePlugin,          // docker.io/woodpeckerci/plugin-git:2.9.2
    "docker.io/woodpeckerci/plugin-git",
    "quay.io/woodpeckerci/plugin-git",
}
```

可通过环境变量 `WOODPECKER_PLUGINS_TRUSTED_CLONE` 运行时追加，也可通过 `repo.NetrcTrustedPlugins` 追加仓库级可信镜像。

### 6.2 Clone 步骤信任校验

定义位置：`pipeline/frontend/yaml/types/container.go:66-68` 和 `pipeline/frontend/yaml/linter/linter.go:115-166`

```go
func (c *Container) IsTrustedCloneImage(trustedClonePlugins []string) bool {
    return c.IsPlugin() && utils.MatchImageDynamic(c.Image, trustedClonePlugins...)
}
```

Linter 在 `lintCloneSteps()` 中对每个 clone 步骤进行检查：
- Clone 步骤必须满足 `IsTrustedCloneImage()`
- 只有通过信任校验的 clone 插件才能被注入 Netrc 凭证

### 6.3 Privileged Plugins 限制

定义位置：`pipeline/frontend/yaml/linter/linter.go:212-222`

出于兼容性考虑，Linter 对 Docker 系列插件（`plugins/docker`、`plugins/gcr`、`plugins/ecr`、`woodpeckerci/plugin-docker-buildx`）做了特殊检查：
- 这些插件曾经默认以 privileged 模式运行
- 现在若用户未将其加入 `WOODPECKER_PLUGINS_PRIVILEGED`，则 Linter 会报错或警告
- 检查同样使用 `MatchImageDynamic()` 进行镜像匹配

> ⚠️ **Trust 链局限**：Woodpecker 仅做**镜像名称字符串匹配**，不做镜像签名校验（如 cosign/notary）、不校验镜像摘要 digest、也不验证层完整性。攻击者若能污染 registry 或发动 DNS 劫持，仍可能推送恶意镜像冒充受信任插件。

---

## 7. Webhook 重放防护

### 7.1 Webhook Token 签名验证

定义位置：`server/api/hook.go:69-92`

所有 Webhook 请求必须经过 Woodpecker 的 `HookToken` JWT 验证：

```go
_, err := token.ParseRequest([]token.Type{token.HookToken}, c.Request, func(t *token.Token) (string, error) {
    repo, err = getRepoFromToken(_store, t)
    if err != nil { return "", err }
    return repo.Hash, nil  // 使用 repo.Hash 作为 JWT 签名密钥
})
```

Token 生成（激活仓库时）：`server/api/repo.go:730-737`

```go
t := token.New(token.HookToken)
t.Set("repo-forge-remote-id", string(repo.ForgeRemoteID))
t.Set("forge-id", strconv.FormatInt(repo.ForgeID, 10))
sig, err := t.Sign(repo.Hash)  // 用 repo.Hash 签名
```

- `repo.Hash` 是仓库激活时生成的 32 字节随机数（Base32 编码）：`server/api/repo.go:119-122`
- Token 作为 Webhook URL 的 query 参数传递给 Forge（GitHub/Gitea 等）
- Forge 每次触发 Webhook 时会原样带回该 Token

### 7.2 Forge 级签名验证

除了 Woodpecker 自身的 JWT Token，各 Forge 驱动还会校验 Forge 自带的签名：

| Forge | 校验实现 |
|-------|---------|
| Bitbucket DC | `bitbucket.ValidateSignature(r, hook.Payload, []byte(repo.Hash))` — `server/forge/bitbucketdatacenter/bitbucketdatacenter.go:510` |
| GitHub | 使用标准 `X-Hub-Signature-256` HMAC-SHA256 签名（由 Forge SDK 内部校验） |
| Gitea/GitLab | 各 Forge 均有对应的签名校验逻辑 |

### 7.3 二次校验

定义位置：`server/api/hook.go:141-148`

```go
// Check the repo from the token is matching the repo returned by the forge
if repo.ForgeRemoteID != repoFromForge.ForgeRemoteID {
    log.Warn().Msgf("ignoring hook: repo %s does not match the repo from the token", repo.FullName)
    c.String(http.StatusBadRequest, "failure to parse token from hook")
    return
}
```

即使 Token 校验通过，还会比对 Token 中的 repo ID 与 Forge Webhook Payload 解析出的 repo ID 是否一致，防止 Token 被用于其他仓库。

### 7.4 重放防护局限

> ⚠️ **重要**：Woodpecker **无显式的 nonce/timestamp 重放防护**。
> - JWT Token 无过期时间校验（`HookToken` 类型未设置 `exp` claim）
> - 无 nonce 缓存防止同一个请求被重复提交
> - 同一 Webhook Payload + 签名可被无限次重放触发 Pipeline
> - 防重放完全依赖 Forge 自身（如 GitHub 的 `X-GitHub-Delivery` 去重），Woodpecker 侧不做二次校验

---

## 8. Secret 改动审计与回滚

### 8.1 审计现状

通过代码审计，Woodpecker 在 Secret 变更方面**缺乏完整的审计链路**：

| 操作 | 是否有审计日志 | 是否保留历史版本 | 是否可回滚 |
|-----|--------------|----------------|-----------|
| Secret Create | ❌ 仅通用 HTTP 访问日志 | ❌ | N/A |
| Secret Update | ❌ 仅通用 HTTP 访问日志 | ❌ | ❌ 原值直接覆盖 |
| Secret Delete | ❌ 仅通用 HTTP 访问日志 | ❌ | ❌ 物理删除 |
| 启用加密/密钥轮换 | ✅ `log.Warn()` 记录开始和结束 | ❌ | ⚠️ 仅支持 `SecretCreate` 两阶段回滚（见 5.5 节） |

相关 API 处理：
- `server/api/repo_secret.go`：`PostSecret`（创建）、`PatchSecret`（更新）、`DeleteSecret`（删除）
- `server/api/org_secret.go`：组织级对应操作
- `server/api/global_secret.go`：全局级对应操作

所有 API 只调用 `secret.Service.SecretCreate/Update/Delete`，无审计 hook。

### 8.2 现有回滚能力

唯一的回滚机制存在于 `EncryptedSecretStore.SecretCreate()` 中（见 5.5 节），仅限：
- 场景：启用数据库加密后，加密或更新步骤失败
- 方式：删除刚插入的临时明文记录
- 范围：仅 `SecretCreate`，不覆盖 `SecretUpdate`/`SecretDelete`

### 8.3 风险

1. **误删无法恢复**：Secret 被删除后数据库中永久丢失，无回收站机制
2. **误改无法回退**：更新 Secret Value 后原值被覆盖，无法恢复
3. **无法追溯变更者**：无操作人、变更时间、变更前后 diff 的审计记录
4. **合规风险**：对于金融/医疗等受监管行业，缺少 Secret 变更审计链可能违反合规要求

---

## 9. 删除事件继承传播

### 9.1 当前行为

定义位置：`server/services/secret/db.go:82-88`、`server/api/repo_secret.go:180-189`

Secret 删除操作是**纯本作用域物理删除**，不触发任何事件向下游传播：

```go
func (d *db) SecretDelete(repo *model.Repo, name string) error {
    secret, err := d.store.SecretFind(repo, name)
    if err != nil { return err }
    return d.store.SecretDelete(secret)  // 直接物理删除该行
}
```

### 9.2 继承断裂场景

当高层级 Secret 被删除后，依赖继承的下层仓库会出现**静默断裂**：

```
场景：删除组织级 Secret "DOCKER_TOKEN"
──────────────────────────────────────
T0: 全局 DOCKER_TOKEN=global-value     ← 仓库 A 继承此值
    组织 DOCKER_TOKEN=org-value         ← 仓库 B 继承此值（覆盖全局）
    仓库 DOCKER_TOKEN=repo-value        ← 仓库 C 有自己的值

T1: 管理员删除组织级 DOCKER_TOKEN
    → store.SecretDelete() 直接删除
    → 无通知、无事件、无传播

T2: 仓库 B 下次 Pipeline 运行
    → SecretListPipeline() 聚合时发现组织级已消失
    → 回退到继承全局 DOCKER_TOKEN=global-value
    → Pipeline 使用了不同的 Secret 值！静默切换！
```

| 场景 | 预期行为 | 实际行为 | 风险 |
|------|---------|---------|------|
| 删除组织级 Secret | 下属仓库收到通知/阻断 | 下次 Pipeline 自动回退继承全局值 | **静默切换到错误凭证** |
| 删除全局 Secret | 所有继承仓库收到通知 | 下次 Pipeline 该 Secret 缺失 | **Pipeline 静默失败** |
| 仓库有同名覆盖 | 无影响 | 仓库级值不受影响 | ✅ 安全 |
| 正在运行的 Pipeline | 使用新 Secret 列表 | 已注入的 Secret 不变，下次运行生效 | ✅ 安全（惰性生效） |

> ⚠️ **关键风险**：Woodpecker **无事件总线、无 webhook 通知、无依赖图追踪**。删除高层级 Secret 后，依赖继承的下层仓库无法感知变更，直到下次 Pipeline 运行时才会发现 Secret 值变化或缺失。

---

## 10. HTTP Extension 缓存与降级窗口

### 10.1 请求级缓存

定义位置：`server/services/secret/http.go:47-69`

HTTP Extension **无缓存机制**。每次 Pipeline 触发都会发起 HTTP 请求：

```go
func (h *httpExtension) SecretListPipeline(...) ([]*model.Secret, error) {
    status, err := h.client.Send(ctx, http.MethodPost, h.endpoint, body, response)
    // 无缓存，每次都是实时请求
}
```

### 10.2 重试机制

定义位置：`server/services/utils/http.go:97-207`

HTTP Client 实现了基于指数退避的重试：

```go
const maxRetries = 3

// 重试条件：
// - 网络级可重试错误：超时、连接拒绝、连接重置、DNS 失败、TLS 握手超时
// - 5xx 服务端错误
// - 不可重试：4xx 客户端错误、JSON 解析失败
```

| 错误类型 | 是否重试 | 最大次数 | 退避策略 |
|---------|---------|---------|---------|
| 网络超时 | ✅ | 3 | 指数退避 |
| 连接拒绝/重置 | ✅ | 3 | 指数退避 |
| 5xx 服务端错误 | ✅ | 3 | 指数退避 |
| 4xx 客户端错误 | ❌ | 0 | N/A |
| JSON 解析失败 | ❌ | 0 | N/A |

### 10.3 降级窗口分析

重试 3 次的指数退避窗口估算：

```
第 1 次：立即执行
第 2 次：~500ms 后
第 3 次：~1s 后
─────────────────
总降级窗口：约 1.5-3 秒
```

> ⚠️ **局限**：
> 1. **无降级策略**：3 次重试全部失败后直接报错，不回退到 base secrets 或空列表
> 2. **无缓存降级**：不像 Vault Agent 有 last-good-response 缓存，Extension 宕机 = 立即不可用
> 3. **10 秒超时**（`server/services/utils/http.go:46`）：每次 HTTP 请求 10s 超时，3 次重试最坏情况约 30s+ 阻塞 Pipeline 解析
> 4. **SSRF 防护**：`hostmatcher.NewDialContext()` 限制 Extension 只能访问 `WOODPECKER_EXTENSIONS_ALLOWED_HOSTS` 白名单中的主机，默认仅允许外部地址

---

## 11. MatchImageDynamic 多 tag 通配

### 11.1 通配能力分析

定义位置：`pipeline/frontend/yaml/utils/image.go:64-85`

`MatchImageDynamic` **不支持任何通配符语法**（`*`、`?`、`**` 等），只做精确字符串匹配：

```go
func imageHasTag(name string) bool {
    return strings.Contains(name, ":")  // 仅检测冒号，无通配逻辑
}
```

| 用户期望 | Secret.Images 配置 | 实际效果 | 原因 |
|---------|-------------------|---------|------|
| 允许所有 tag | `plugins/docker:*` | ❌ 不匹配 | `*` 被当作字面 tag 值 |
| 允许 v2.x 系列 | `plugins/docker:v2.*` | ❌ 不匹配 | 不支持 glob |
| 允许多个 tag | `plugins/docker:v1,plugins/docker:v2` | ✅ 分别匹配 | 需显式列出 |
| 允许所有版本 | `plugins/docker` | ✅ 忽略 tag | 不带 tag 时忽略 tag 匹配 |

### 11.2 多 tag 变通方案

要允许 Secret 被多个特定 tag 版本的插件使用，必须在 `Images` 列表中**逐一列出**：

```yaml
# Secret 定义
images:
  - plugins/docker:v1
  - plugins/docker:v2
  - plugins/docker:v3
```

### 11.3 `imageHasTag` 误判风险

```go
func imageHasTag(name string) bool {
    return strings.Contains(name, ":")
}
```

以下场景可能导致误判：

| 镜像字符串 | `imageHasTag` 结果 | 实际含义 | 是否误判 |
|-----------|-------------------|---------|---------|
| `plugins/docker:v1` | `true` | tag=v1 | ✅ 正确 |
| `localhost:5000/myplugin` | `true` | 端口号，非 tag | ⚠️ 误判为有 tag |
| `docker.io/library/alpine` | `false` | 无 tag | ✅ 正确 |

> 误判后果：当 `Images` 限制列表中包含 `localhost:5000/myplugin` 时，`imageHasTag` 返回 `true`，走精确匹配路径。而 Pipeline 中的 `localhost:5000/myplugin` 经 `expandImage()` 补全后为 `localhost:5000/myplugin:latest`，与限制列表中的原始字符串不一致，导致**匹配失败**。这是已知边界问题，实际使用中应避免在 `Images` 列表中使用带端口的镜像地址，或显式带 tag。

---

## 12. NewSecretsReplacer UTF-8 切片误伤

### 12.1 长度判断基于字节而非字符

定义位置：`pipeline/shared/replace_secrets.go:29-33`

```go
const minStringLength = 3

for _, old := range secrets {
    old = strings.TrimSpace(old)
    if len(old) <= minStringLength {  // len() 返回字节数，非字符数
        continue
    }
```

`len(string)` 在 Go 中返回**字节数**，非 Unicode 字符数。这导致：

| Secret 值 | `len()` | 是否脱敏 | 说明 |
|----------|---------|---------|------|
| `abc` | 3 | ❌ 跳过 | 3 字节 = 3 字符 |
| `中文` | 6 | ✅ 脱敏 | 2 个字符 = 6 字节（UTF-8 每汉字 3 字节） |
| `你` | 3 | ❌ 跳过 | 1 个汉字 = 3 字节，恰好等于阈值 |
| `ab` | 2 | ❌ 跳过 | 正确 |
| `日本語` | 9 | ✅ 脱敏 | 3 字符 = 9 字节 |
| `αβ` | 4 | ✅ 脱敏 | 2 希腊字母各 2 字节 |

### 12.2 多行 Secret 的 UTF-8 切片风险

```go
for _, part := range strings.Split(old, "\n") {
    if len(part) == 0 { continue }
    oldNew = append(oldNew, part)
    oldNew = append(oldNew, "********")
}
```

`strings.Split` 按 `\n` 分割是字节级操作，对 UTF-8 是安全的（`\n` 是单字节 ASCII 字符，不会出现在 UTF-8 多字节序列内部）。**但**以下场景仍有风险：

| 场景 | Secret 值 | 风险 |
|------|----------|------|
| 某行恰好是 3 字节 UTF-8 单字符 | `line1\n你\nline3` | `你` 被跳过不脱敏，但作为单行恰好是完整 Secret 片段 |
| CRLF 行尾 | `line1\r\nline2` | `\r` 残留在行尾，`strings.TrimSpace` 仅在整体上做了 trim，但 Split 后的 part 末尾可能残留 `\r` |
| 零宽字符 | `pass\u200bword` | 零宽空格（3 字节）可能导致 replacer 无法匹配日志中的可见文本 |

### 12.3 `strings.Replacer` 的 UTF-8 行为

`strings.Replacer` 内部基于字节级匹配，对 UTF-8 字符串的子串替换是**字节精确匹配**。这意味着：

- ✅ 完整的 UTF-8 字符不会被截断匹配
- ⚠️ 但若日志输出系统对多字节字符做了截断（如按字节限制行宽），截断后的残余字节可能恰好与 Secret 前缀匹配，造成误替换或漏替换

---

## 13. fsnotify 在 NFS 与共享卷下的兼容

### 13.1 当前实现

定义位置：`server/services/encryption/tink_keyset_watcher.go:25-61`

```go
func (svc *tinkEncryptionService) initFileWatcher() error {
    watcher, err := fsnotify.NewWatcher()
    err = watcher.Add(svc.keysetFilePath)
    go svc.handleFileEvents()
    return nil
}

func (svc *tinkEncryptionService) handleFileEvents() {
    for {
        select {
        case event, ok := <-svc.keysetFileWatcher.Events:
            if (event.Op == fsnotify.Write) || (event.Op == fsnotify.Create) {
                err := svc.rotate()  // 自动触发密钥轮换
                if err != nil {
                    log.Fatal().Err(err).Msg(...)  // 轮换失败 → 进程自杀！
                }
                return
            }
        case err, ok := <-svc.keysetFileWatcher.Errors:
            log.Fatal().Err(err).Msg(...)  // watcher 错误 → 进程自杀！
        }
    }
}
```

### 13.2 NFS 兼容性

fsnotify 底层使用操作系统的文件事件通知机制：
- **Linux**：`inotify`（inode 级别监控）
- **macOS**：`kqueue`（文件描述符监控）

在 NFS/CIFS 等网络文件系统上：

| 场景 | inotify 行为 | 影响 |
|------|-------------|------|
| NFS 客户端缓存 | `inotify` **无法监控 NFS 远程写入** | 密钥文件更新后 watcher 不触发，**密钥轮换静默失败** |
| NFS 服务器推送 | 某些 NFS 实现会触发 `IN_ATTRIB` 事件 | 可能误触发轮换 |
| CIFS/SMB | 完全不支持 `inotify` | `watcher.Add()` 可能成功但永远不触发事件 |
| NFS hard mount + 服务器宕机 | 进程挂起在 NFS I/O | `handleFileEvents` goroutine 永久阻塞 |

### 13.3 共享卷场景（Kubernetes ConfigMap/Secret）

Kubernetes 中密钥文件通常以 ConfigMap 或 Secret 挂载：

| 挂载方式 | 文件更新机制 | fsnotify 能否检测 | 影响 |
|---------|------------|-------------------|------|
| `subPath` 挂载 | 原地更新文件内容 | ✅ 可检测 Write 事件 | 正常工作 |
| 目录级挂载 | 创建新目录 + 符号链接 | ⚠️ 仅检测 Create 事件 | 需依赖 `fsnotify.Create` 分支 |
| ConfigMap 更新 | 原子性：新临时目录 → 符号链接切换 | ❌ 符号链接变更不触发 | **密钥轮换静默失败** |

### 13.4 进程自杀风险

当 watcher 出错或轮换失败时，代码使用 `log.Fatal()` 直接终止进程：

```go
log.Fatal().Msg(errMessageTinkKeysetFileWatchFailed)  // os.Exit(1)
log.Fatal().Err(err).Msg(errMessageFailedRotatingEncryption)  // os.Exit(1)
```

> ⚠️ **风险**：
> 1. NFS/CIFS 上 watcher 永不触发 → 密钥轮换永远不执行 → 用旧密钥加密的数据在新密钥环境下无法解密
> 2. K8s ConfigMap 更新不触发 → 同上
> 3. watcher 任何错误 → `log.Fatal()` → **Woodpecker Server 进程崩溃退出**
> 4. 无 fallback 轮询机制作为 fsnotify 的补充

---

## 14. Trusted Plugins 动态加载

### 14.1 加载时机

定义位置：`cmd/server/setup.go:198-199`、`cmd/server/flags.go:228-236`

Trusted Clone Plugins 和 Privileged Plugins 在 **Server 启动时一次性加载**到全局配置中：

```go
// setup.go:198-199
server.Config.Pipeline.TrustedClonePlugins = c.StringSlice("plugins-trusted-clone")
server.Config.Pipeline.PrivilegedPlugins = c.StringSlice("plugins-privileged")
```

环境变量：
- `WOODPECKER_PLUGINS_TRUSTED_CLONE`：可信克隆插件列表
- `WOODPECKER_PLUGINS_PRIVILEGED`：特权插件列表

### 14.2 运行时合并

定义位置：`server/pipeline/items.go:118`

每次 Pipeline 解析时，Trusted Clone Plugins 从两处合并：

```go
TrustedClonePlugins: append(repo.NetrcTrustedPlugins, server.Config.Pipeline.TrustedClonePlugins...)
```

- `repo.NetrcTrustedPlugins`：仓库级动态配置（数据库存储，API 可修改）
- `server.Config.Pipeline.TrustedClonePlugins`：全局静态配置（启动时加载，需重启生效）

### 14.3 动态更新能力

| 配置项 | 存储位置 | 可否运行时修改 | 修改后生效方式 |
|-------|---------|--------------|--------------|
| `WOODPECKER_PLUGINS_TRUSTED_CLONE` | 进程内存 | ❌ | 重启 Server |
| `WOODPECKER_PLUGINS_PRIVILEGED` | 进程内存 | ❌ | 重启 Server |
| `repo.NetrcTrustedPlugins` | 数据库 | ✅ (API) | 下次 Pipeline 立即生效 |
| `DefaultClonePlugin` | 进程内存 | ❌ | 重启 Server |

> ⚠️ **局限**：
> 1. 全局 Trusted/Privileged Plugins 列表**不支持热加载**，增删插件必须重启 Server
> 2. `repo.NetrcTrustedPlugins` 可通过 API 动态修改，但无审计日志
> 3. 内置的 `constant.TrustedClonePlugins` 是编译期硬编码的，仅可通过环境变量覆盖
> 4. 无插件版本约束机制（如 `plugins/docker:>2.0`），只能按完整镜像名逐一枚举

---

## 15. JWT 密钥泄露后的强制轮换

### 15.1 Webhook JWT 密钥体系

定义位置：`shared/token/token.go:122-145`、`server/api/repo.go:119-122`

每个仓库的 Webhook JWT 签名密钥是 `repo.Hash`：

```go
// 密钥生成
if repo.Hash == "" {
    repo.Hash = base32.StdEncoding.EncodeToString(random.GetRandomBytes(32))
}

// Token 签名
t := token.New(token.HookToken)
sig, err := t.Sign(repo.Hash)  // repo.Hash 作为 HS256 签名密钥
```

JWT 特性：
- 算法：`HS256`（`shared/token/token.go:40`）
- **无过期时间**：`HookToken` 使用 `Sign(secret)` → `SignExpires(secret, 0)`，`exp=0` 表示永不过期
- **无 `jti`/`nonce`**：claims 中无唯一标识，无法检测重放

### 15.2 泄露影响

若 `repo.Hash` 泄露，攻击者可以：
1. **伪造 Webhook 请求**：构造任意 Pipeline 触发请求
2. **注入恶意代码**：通过 `deployment` 事件触发生产环境部署
3. **绕过 Forge 签名**：因为 Woodpecker 先验证 JWT，再交给 Forge 解析

### 15.3 强制轮换流程

Woodpecker **无内置的 Hash 轮换 API**。泄露后需要手动操作：

```
1. 直接修改数据库：UPDATE repos SET hash = '<new_random_hash>' WHERE id = <repo_id>
2. 重新激活仓库 Webhook（调用 Activate API 或手动在 Forge 中更新 Webhook URL）
   → Activate API 会重新生成 JWT 并更新 Forge 的 Webhook URL：server/api/repo.go:730-737
3. 旧 Hash 立即失效（JWT 验证使用数据库中的新 Hash）
```

> ⚠️ **局限**：
> 1. **无自动轮换机制**：不像 Tink 加密密钥有 fsnotify 自动检测，`repo.Hash` 需要手动更新
> 2. **无泄露检测**：无日志异常检测、无 Token 使用频率监控
> 3. **轮换窗口风险**：Hash 更新到 Forge Webhook 更新之间存在短暂不一致，可能丢失 Webhook 事件
> 4. **全局影响**：若 Forge 的 Webhook Secret（非 `repo.Hash`）泄露，需要逐仓库重新激活

### 15.4 补救建议

| 措施 | 实现方式 | 优先级 |
|------|---------|--------|
| 添加 HookToken 过期时间 | `SignExpires(secret, time.Now().Add(24h).Unix())` | 高 |
| 添加 `jti` claim 防重放 | `claims["jti"] = uuid.New().String()` + Redis 去重 | 中 |
| 添加 Hash 轮换 API | 新增 `POST /repos/{id}/rotate-hash` 端点 | 高 |
| 泄露告警 | 监控同一 Token 的 IP 异常、频率异常 | 中 |

---

## 16. 审计四大风险的补救方案

### 16.1 风险 1：误删无法恢复

**现状**：`SecretDelete()` 执行物理删除，无回收站。

**补救方案**：

| 方案 | 实现路径 | 侵入性 |
|------|---------|--------|
| **软删除** | 添加 `deleted_at timestamp` 字段，`SecretDelete()` 改为设置 `deleted_at`，查询时过滤 | 中 |
| **回收站表** | 删除前将 Secret 备份到 `secrets_trash` 表，提供 `POST /secrets/{id}/restore` API | 低 |
| **定时快照** | 定期将 `secrets` 表导出至对象存储（S3/MinIO），保留 N 份 | 低 |

**推荐**：回收站表方案，实现简单、侵入最低、可精确恢复单条记录。

### 16.2 风险 2：误改无法回退

**现状**：`SecretUpdate()` 直接覆盖 Value 字段，无版本历史。

**补救方案**：

| 方案 | 实现路径 | 侵入性 |
|------|---------|--------|
| **版本表** | 新建 `secret_versions` 表，`SecretUpdate()` 前插入历史版本快照 | 中 |
| **审计日志表** | 新建 `audit_log` 表，记录 `{action, resource_type, resource_id, old_value_hash, new_value_hash, operator, timestamp}` | 中 |
| **GitOps 模式** | Secret 值只从外部 Vault/External Secrets 获取，Woodpecker 不持久化 Value | 高 |

**推荐**：版本表 + 审计日志组合。版本表存完整值用于回退，审计日志存操作痕迹用于追溯。注意版本表的 Value 也需加密存储。

### 16.3 风险 3：无法追溯变更者

**现状**：API 处理函数中无审计 hook，无法知道谁在何时修改了哪个 Secret。

**补救方案**：

```go
// 在 Secret Service 层添加审计中间件
type AuditedSecretService struct {
    inner  Service
    auditor Auditor
}

func (a *AuditedSecretService) SecretCreate(repo *model.Repo, secret *model.Secret) error {
    err := a.inner.SecretCreate(repo, secret)
    if err == nil {
        a.auditor.Record(AuditEntry{
            Action:     "secret.create",
            ResourceID: secret.ID,
            Operator:   session.UserFromContext(ctx).Login,
            Timestamp:  time.Now(),
            Metadata:   map[string]string{"name": secret.Name, "scope": scopeOf(secret)},
        })
    }
    return err
}
```

| 方案 | 实现路径 | 侵入性 |
|------|---------|--------|
| **Service 层审计装饰器** | 用装饰器模式包装 `secret.Service`，自动记录所有 CUD 操作 | 低 |
| **数据库触发器** | PostgreSQL 层面添加 `AFTER INSERT/UPDATE/DELETE ON secrets` 触发器 | 低（对应用透明） |
| **API 中间件** | 在 Gin 中间件层记录请求体 + 用户信息 | 中（需处理 Body 读取问题） |

**推荐**：Service 层审计装饰器，与现有 `EncryptedSecretStore` 包装器模式一致，不侵入业务逻辑。

### 16.4 风险 4：合规风险

**现状**：无审计链、无密钥轮换证明、无访问控制审计。

**补救方案**：

| 合规要求 | 对应补救 | 优先级 |
|---------|---------|--------|
| 变更审计 | 16.3 审计装饰器 | 高 |
| 数据加密 | 启用 `EncryptedSecretStore`（当前被注释） | 高 |
| 密钥轮换证明 | Tink 密钥轮换日志 + 审计记录 | 中 |
| 访问控制 | 记录 Secret 读取（`SecretFind`/`SecretList`）操作 | 中 |
| 数据留存 | Secret 值的哈希（非明文）写入审计日志，满足不可篡改 | 低 |
| 分离职责 | Secret 创建/更新/删除需要不同角色审批 | 低 |

**推荐分阶段实施**：
1. **Phase 1**：启用 `EncryptedSecretStore` + 添加审计装饰器
2. **Phase 2**：添加 Secret 版本表 + 软删除
3. **Phase 3**：添加 HookToken 过期时间 + Hash 轮换 API

---

## 17. 完整处理路径总结

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
