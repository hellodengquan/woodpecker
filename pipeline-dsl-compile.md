# Pipeline DSL 解析与编译过程分析

本文件详细分析 Woodpecker CI 流水线配置文件（YAML）从原始文本到可执行单元（`backend_types.Config`）的完整解析与组装过程。

---

## 一、整体架构概览

### 1.1 核心模块分层

```
┌───────────────────────────────────────────────────────────────┐
│  入口层 (Entry Points)                                        │
│  - server/pipeline/items.go  (服务端调度)                     │
│  - cli/exec/exec.go           (CLI 本地执行)                  │
└─────────────────────┬─────────────────────────────────────────┘
                      │ 构造 PipelineBuilder 并调用 .Build()
┌─────────────────────▼─────────────────────────────────────────┐
│  编排层 (Orchestration)                                        │
│  - pipeline/frontend/builder/builder.go                       │
│    PipelineBuilder: 多文件 + Matrix 扩展 + Workflow 过滤      │
└─────────────────────┬─────────────────────────────────────────┘
                      │
     ┌────────────────┼────────────────┐
     │                │                │
┌────▼────┐    ┌──────▼──────┐   ┌────▼─────┐
│ Matrix  │    │  YAML Parse │   │  Linter  │
│ 解析    │    │  (xyaml)    │   │  校验    │
└────┬────┘    └──────┬──────┘   └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      │ 产出 yaml_types.Workflow
┌─────────────────────▼─────────────────────────────────────────┐
│  编译层 (Compiler)                                             │
│  - pipeline/frontend/yaml/compiler/compiler.go                │
│    Compiler.Compile(): Workflow → backend_types.Config        │
└─────────────────────┬─────────────────────────────────────────┘
                      │
     ┌────────────────┼───────────────────┐
     │                │                   │
┌────▼────┐    ┌──────▼───────┐    ┌──────▼───────┐
│ Step    │    │  DAG 编译    │    │  Settings    │
│ 创建    │    │  (拓扑排序)  │    │  → Env 转换  │
└────┬────┘    └──────┬───────┘    └──────┬───────┘
     │                │                   │
     └────────────────┼───────────────────┘
                      │ 产出 backend_types.Config
┌─────────────────────▼─────────────────────────────────────────┐
│  后端可执行单元 (Backend IR)                                   │
│  - pipeline/backend/types/config.go                           │
│    Config { Stages[] → Stage { Steps[] → Step } }             │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据结构流转

| 阶段 | 数据结构 | 文件位置 |
|------|---------|----------|
| 原始输入 | `[]byte` (YAML 文本) | - |
| 矩阵轴 | `matrix.Axis` | `pipeline/frontend/yaml/matrix/matrix.go:34` |
| YAML 语义模型 | `yaml_types.Workflow` | `pipeline/frontend/yaml/types/workflow.go:23` |
| 容器定义 | `yaml_types.Container` | `pipeline/frontend/yaml/types/container.go:24` |
| 编译中间件 | `dagCompilerStep` | `pipeline/frontend/yaml/compiler/dag.go:24` |
| 后端 IR | `backend_types.Config` | `pipeline/backend/types/config.go:18` |
| 后端步骤 | `backend_types.Step` | `pipeline/backend/types/step.go:18` |
| 最终编排 | `builder.Item` | `pipeline/frontend/builder/types.go:24` |

---

## 二、执行顺序详解

### 2.1 阶段一：入口初始化 — 构造 PipelineBuilder

**调用位置**：
- 服务端：`server/pipeline/items.go:38` → `parsePipeline()`
- CLI 本地：`cli/exec/exec.go:65` → `runExec()`

**操作流程**：

```go
// server/pipeline/items.go:114-148
b := builder.PipelineBuilder{
    GetWorkflowMetadata: serverMetadata.GetWorkflowMetadata, // Workflow 元数据生成器
    Envs:                envs,                               // 全局环境变量
    Yamls:               yamls,                              // YAML 文件列表 []*YamlFile
    TrustedClonePlugins: ...,                                // 可信克隆插件白名单
    PrivilegedPlugins:   ...,                                // 特权插件列表
    RepoTrusted:         ...,                                // 仓库信任配置
    DefaultLabels:       ...,                                // 默认标签
    CompilerOptions: []compiler.Option{                      // 编译器选项
        compiler.WithRegistry(registries...),                // - 镜像仓库凭证
        compiler.WithSecret(secrets...),                     // - 密钥列表
        compiler.WithNetrc(...),                             // - Netrc 认证
        compiler.WithWorkspaceFromURL(...),                  // - 工作目录
        compiler.WithVolumes(...),                           // - 挂载卷
        compiler.WithNetworks(...),                          // - 网络
        compiler.WithProxy(...),                             // - 代理
        compiler.WithDefaultClonePlugin(...),                // - 默认克隆插件
        compiler.WithLocal(false),                           // - 本地模式标记
    },
}
items, err := b.Build()
```

---

### 2.2 阶段二：PipelineBuilder.Build() — 多文件编排

**代码位置**：`pipeline/frontend/builder/builder.go:52-97`

**执行步骤**：

#### 2.2.1 步骤 1：YAML 文件排序
```go
b.Yamls = SortYamlFilesByName(b.Yamls)
```
对多个配置文件按文件名排序，确保流水线依赖（`depends_on`）处理顺序一致。  
代码位置：`pipeline/frontend/builder/types.go:51-55`

#### 2.2.2 步骤 2：遍历每个 YAML 文件并解析 Matrix

```go
for _, y := range b.Yamls {
    // Matrix 解析
    axes, err := matrix.ParseString(string(y.Data))
    if len(axes) == 0 {
        axes = append(axes, matrix.Axis{}) // 无 matrix 则使用空轴
    }

    for i, axis := range axes {
        workflow := &Workflow{
            PID:     pidSequence,     // 全局递增 PID
            Environ: axis,            // 当前矩阵轴的环境变量
            Name:    SanitizePath(y.Name), // 清洗后的文件名
            AxisID:  i + 1,           // 矩阵轴编号（多轴时设置）
        }
        // 为每个矩阵轴生成独立的 workflow
        item, err := b.genItemForWorkflow(workflow, axis, string(y.Data))
        items = append(items, item)
        pidSequence++
    }
}
```

#### 2.2.3 步骤 3：Matrix 扩展算法

**代码位置**：`pipeline/frontend/yaml/matrix/matrix.go`

支持两种 matrix 语法：
1. **笛卡尔积模式**：自动计算所有组合
   ```go
   // matrix.go:70-112 calc()
   // 例: {go: [1.21, 1.22], db: [mysql, pg]} → 2×2=4 个轴
   ```
2. **Include 显式列表**：用户手动指定组合
   ```go
   // matrix.go:124-134 parseList()
   // 例: matrix.include: [{go: 1.21, db: mysql}, ...]
   ```

限制：最多 10 个标签维度，最多 25 个排列组合（`limitTags=10, limitAxis=25`）。

#### 2.2.4 步骤 4：过滤缺失的 Workflow 依赖

```go
// pipeline/frontend/builder/utils.go:35-74
items = filterMissingDependencies(items)
```
循环直到稳定：
- 删除缺少**必需依赖**（`optional: false`）的 workflow
- 清理不再存在的**可选依赖**（`optional: true`）
- 最终将所有幸存依赖标记为 `optional=false`

---

### 2.3 阶段三：genItemForWorkflow() — 单 Workflow 处理

**代码位置**：`pipeline/frontend/builder/builder.go:99-205`

这是最核心的处理函数，包含 7 个子步骤：

#### 2.3.1 子步骤 1：构建环境变量集合

```go
workflowMetadata := b.GetWorkflowMetadata(workflow)
environ := b.environmentVariables(workflowMetadata, axis)
// 合并全局 Envs（不覆盖已有值）
for k, v := range b.Envs {
    if _, exists := environ[k]; !exists {
        environ[k] = v
    }
}
```
`environmentVariables()` 合并：CI 元数据环境变量（仓库、分支、提交等）+ Matrix 轴变量。

#### 2.3.2 子步骤 2：环境变量替换（Substitution）

```go
// pipeline/frontend/metadata/substitution.go:24-31
substituted, err := metadata.EnvVarSubst(data, environ)
```
使用 `github.com/drone/envsubst` 库，将 YAML 文本中的 `${VAR}` / `$VAR` 替换为实际值。  
特殊处理：含换行符的值会被自动加引号转义。

#### 2.3.3 子步骤 3：YAML 文本 → 语义模型

```go
// pipeline/frontend/yaml/parse.go:24-31
parsed, err := yaml.ParseString(substituted)
```
使用自定义的 `codeberg.org/6543/xyaml/v2` 反序列化为 `yaml_types.Workflow`：

```go
type Workflow struct {
    When      constraint.When      // 执行条件过滤
    Workspace Workspace            // 工作目录
    Clone     ContainerList        // 克隆步骤列表
    Steps     ContainerList        // 主步骤列表（核心）
    Services  ContainerList        // 服务步骤列表
    Labels    map[string]string    // 标签
    DependsOn constraint.DependsOn // Workflow 间依赖
    SkipClone bool                 // 是否跳过克隆
    RunsOn    []string             // （已废弃）何时运行
}
```

#### 2.3.4 子步骤 4：Lint 静态检查

```go
// pipeline/frontend/yaml/linter/linter.go:69-79
errorsAndWarnings = linter.New(...).Lint([]*linter.WorkflowConfig{{...}})
```

Linter 执行顺序（`linter.go:81-112` `lintFile()`）：

| 检查项 | 代码位置 | 作用 |
|--------|---------|------|
| 必需 `steps` 段 | linter.go:84 | 报错（阻塞） |
| 克隆步骤镜像校验 | linter.go:88-90, 115-137 | 白名单校验（警告） |
| 容器基础校验 | linter.go:92-100 + 140-176 | 镜像、settings、信任级别、特权、depends_on |
| JSON Schema 校验 | linter.go:102-103, 295-308 | YAML 结构合规性（警告） |
| 废弃语法检测 | linter.go:105-106, 311-332 | `runs_on` 弃用提示 |
| 坏习惯检查 | linter.go:108-109, 334-385 | 缺少 event filter 提示 |

错误分为两类：
- **阻塞错误**（`IsWarning: false`）：中止编译
- **警告错误**（`IsWarning: true`）：继续执行但记录

#### 2.3.5 子步骤 5：Workflow 级别 When 过滤

```go
if match, err := parsed.When.Match(workflowMetadata, true, environ); !match && err == nil {
    return nil, nil // 整个 workflow 被跳过
}
```
基于当前事件类型、分支、标签等元数据，评估是否应该执行此 workflow。

#### 2.3.6 子步骤 6：编译为后端 IR — toInternalRepresentation()

```go
ir, err := b.toInternalRepresentation(parsed, environ, workflowMetadata, workflow.ID)
```
见 **2.4 节 Compiler.Compile()** 详细说明。

#### 2.3.7 子步骤 7：组装 Item 并打标签

```go
item = &Item{
    Workflow:  workflow,
    Config:    ir,                    // 编译好的 backend_types.Config
    Labels:    parsed.Labels,
    DependsOn: parsed.DependsOn,
    RunsOn:    parsed.RunsOn,
}
```

后处理：
- 默认标签注入：用户未定义 Labels 时使用 `DefaultLabels`
- 内部标签打戳：添加 forge 类型、仓库 ID、分支等路由标签（`woodpecker-ci.org/*` 命名空间）
- 删除用户自定义的保留前缀标签（防冲突）
- `RunsOn` 自动补全：根据 `when.status` 自动添加 `success`/`failure`

---

### 2.4 阶段四：Compiler.Compile() — 编译为后端 IR

**代码位置**：`pipeline/frontend/yaml/compiler/compiler.go:119-283`

Compiler 选项通过 Functional Options 模式注入（见 `compiler/option.go`）。

#### 2.4.1 初始化

```go
config := new(backend_types.Config)

// 再次做 When 检查（防御性）
if match, err := conf.When.Match(c.metadata, true, c.env); !match && err == nil {
    return config, nil
}

// 生成全局资源名（带前缀避免容器/卷/网络冲突）
config.Volume  = fmt.Sprintf("%s_default", c.prefix) // 共享数据卷
config.Network = fmt.Sprintf("%s_default", c.prefix) // 共享网络

// 收集所有密钥（用于日志掩码）
for _, sec := range c.secrets {
    config.Secrets = append(config.Secrets, &backend_types.Secret{...})
}

// 覆盖工作目录配置
c.workspaceBase = conf.Workspace.Base
c.workspacePath = conf.Workspace.Path
```

#### 2.4.2 阶段 A：注入 Clone 阶段

```go
// compiler.go:153-201
if !c.local && len(conf.Clone.ContainerList) == 0 && !conf.SkipClone && len(c.defaultClonePlugin) != 0 {
    // 默认克隆步骤：使用默认插件
    container := &yaml_types.Container{
        Name:     "clone",
        Image:    c.defaultClonePlugin, // 例如: woodpeckerci/plugin-git
        Settings: map[string]any{"depth": "0"},
    }
    if c.metadata.Curr.Event == metadata.EventTag {
        cloneSettings["tags"] = "true"
    }
    step, _ := c.createProcess(container, conf, StepTypeClone)
    stage := &Stage{Steps: []*Step{step}}
    config.Stages = append(config.Stages, stage)
} else if !c.local && !conf.SkipClone {
    // 用户自定义克隆步骤
    for _, container := range conf.Clone.ContainerList {
        if !container.When.Match(...) { continue } // 步骤级 When 过滤
        step, _ := c.createProcess(container, conf, StepTypeClone)
        // 可信仓库或可信克隆镜像才注入 netrc 凭证
        if c.securityTrustedPipeline || container.IsTrustedCloneImage(...) {
            maps.Copy(step.Environment, c.cloneEnv)
        }
        stage := &Stage{Steps: []*Step{step}}
        config.Stages = append(config.Stages, stage)
    }
}
```

每个克隆步骤独占一个 Stage（顺序串行执行）。

#### 2.4.3 阶段 B：注入 Services 阶段

```go
// compiler.go:203-222
if len(conf.Services.ContainerList) != 0 {
    stage := new(backend_types.Stage) // 所有 services 合并到一个 stage 并行启动
    for _, container := range conf.Services.ContainerList {
        if !container.When.Match(...) { continue }
        step, _ := c.createProcess(container, conf, StepTypeService)
        // 标记 detached=true（服务不退出）
        stage.Steps = append(stage.Steps, step)
    }
    config.Stages = append(config.Stages, stage)
}
```
所有服务容器**在一个 Stage 内并行**启动。

#### 2.4.4 阶段 C：处理 Steps（核心）

```go
// compiler.go:224-263
stepNames := make(map[string]struct{}, len(conf.Steps.ContainerList))
for _, container := range conf.Steps.ContainerList {
    stepNames[container.Name] = struct{}{}
}

steps := make([]*dagCompilerStep, 0, len(conf.Steps.ContainerList))
for pos, container := range conf.Steps.ContainerList {
    // 本地模式过滤
    if c.local && !container.When.IsLocal() { continue }
    // When 条件过滤
    if !container.When.Match(...) { continue }

    // 判断步骤类型
    stepType := StepTypeCommands
    if container.IsPlugin() { // 无 Commands、Entrypoint、Environment 的纯插件
        stepType = StepTypePlugin
    }
    step, err := c.createProcess(container, conf, stepType)

    // 注入 netrc（与 clone 相同规则）
    if c.securityTrustedPipeline || container.IsTrustedCloneImage(...) {
        maps.Copy(step.Environment, c.cloneEnv)
    }

    // 包装为 DAG 节点，保留原始位置和 depends_on 信息
    steps = append(steps, &dagCompilerStep{
        step:      step,
        position:  pos,       // 原始定义顺序
        name:      container.Name,
        dependsOn: container.DependsOn,
    })
}
```

**关键概念 — 步骤类型判定**（`pipeline/frontend/yaml/types/container.go:60-64`）：
```go
func (c *Container) IsPlugin() bool {
    return len(c.Commands) == 0 && len(c.Entrypoint) == 0 && len(c.Environment) == 0
}
```

#### 2.4.5 阶段 D：DAG 编译 — 拓扑排序生成 Stages

```go
// compiler.go:265-280
stepStages, err := newDAGCompiler(steps).compile()
```
详见 **2.5 节 DAG 编译**。

最后将生成的 Stages 追加到 `config.Stages`。

---

### 2.5 阶段五：DAG 编译（步骤 → 阶段）

**代码位置**：`pipeline/frontend/yaml/compiler/dag.go`

#### 2.5.1 判定模式

```go
func (c dagCompiler) isDAG() bool {
    for _, v := range c.steps {
        if !v.dependsOn.IsZero() {
            return true
        }
    }
    return false
}
```

- **无任何 depends_on** → **顺序模式**（Sequence）：一步一 Stage
- **至少一个 depends_on** → **DAG 模式**：拓扑排序分层

#### 2.5.2 顺序模式 compileSequence()

```go
// dag.go:57-67
func (c dagCompiler) compileSequence() ([]*Stage, error) {
    stages := make([]*Stage, 0, len(c.steps))
    for _, s := range c.steps {
        stages = append(stages, &Stage{Steps: []*Step{s.step}})
    }
    return stages, nil
}
```
每个步骤独占一个 Stage → 严格按定义顺序串行执行。

#### 2.5.3 DAG 模式 compileByDependsOn()

```go
func convertDAGToStages(steps map[string]*dagCompilerStep) ([]*Stage, error) {
    // 第 1 步：解析依赖关系
    for name, step := range steps {
        for _, dep := range step.dependsOn {
            if _, ok := steps[dep.Name]; !ok {
                if dep.Optional { continue }
                return nil, &ErrStepMissingDependency{name, dep.Name}
            }
        }
        // DFS 检测循环依赖
        if err := dfsVisit(steps, name, visited, []string{}); err != nil {
            return nil, err // ErrStepDependencyCycle
        }
    }

    // 第 2 步：Kahn 式拓扑分层
    for len(steps) > 0 {
        addedNodesThisLevel := make(map[string]struct{})
        stage := new(Stage)
        var stepsToAdd []*dagCompilerStep

        // 找出所有依赖已满足的节点
        for name, step := range steps {
            if allDependenciesSatisfied(step, addedSteps) {
                stepsToAdd = append(stepsToAdd, step)
                addedNodesThisLevel[name] = struct{}{}
                delete(steps, name)
            }
        }

        // 按原始定义位置排序（同层步骤保证确定性）
        sort.Slice(stepsToAdd, func(i, j int) bool {
            return stepsToAdd[i].position < stepsToAdd[j].position
        })

        for _, s := range stepsToAdd {
            stage.Steps = append(stage.Steps, s.step)
        }
        stages = append(stages, stage)
    }
    return stages, nil
}
```

**DAG 编译结果特征**：
- 每层（Stage）内的 Steps 可以**并行执行**
- 层与层之间严格**串行依赖**
- 同层步骤按原始 YAML 定义位置排序（确定性）
- 可选依赖（`optional: true`）缺失时不报错，直接跳过

---

### 2.6 阶段六：createProcess() — 单个 Step 组装

**代码位置**：`pipeline/frontend/yaml/compiler/convert.go:40-194`

这是将 `yaml_types.Container` 转换为 `backend_types.Step` 的核心函数。

#### 2.6.1 工作目录计算

```go
// 插件使用固定基础目录防止污染入口点
workspaceBase := c.workspaceBase
if container.IsPlugin() {
    workspaceBase = "/woodpecker" // pluginWorkspaceBase
}

// 卷挂载格式：<前缀>_default:<workspaceBase>
workspaceVolume := fmt.Sprintf("%s_default:%s", c.prefix, workspaceBase)

// 实际工作目录计算 (convert.go:196-205)
func (c *Compiler) stepWorkingDir(container *yaml_types.Container) string {
    if path.IsAbs(container.Directory) { return container.Directory }
    base := c.workspaceBase
    if container.IsPlugin() { base = pluginWorkspaceBase }
    return path.Join(base, c.workspacePath, container.Directory)
}
```

#### 2.6.2 网络与卷配置

```go
// 加入默认网络（容器名作为 DNS 别名）
networks := []Conn{{Name: fmt.Sprintf("%s_default", c.prefix), Aliases: []string{container.Name}}}
// 附加用户自定义网络
for _, network := range c.networks {
    networks = append(networks, Conn{Name: network})
}

// 非本地模式挂载共享卷
if !c.local { volumes = append(volumes, workspaceVolume) }
// 附加全局卷
volumes = append(volumes, c.volumes...)
// 附加容器级卷
for _, volume := range container.Volumes.Volumes {
    volumes = append(volumes, volume.String())
}
```

#### 2.6.3 Settings / Environment → 环境变量转换

```go
// 1. Settings → PLUGIN_ 前缀环境变量（深度处理 from_secret）
settings.ParamsToEnv(
    container.Settings,    // 用户配置的 settings
    environment,           // 写入目标 map
    "PLUGIN_",             // 前缀
    true,                  // 转大写
    getSecretValue,        // 密钥获取回调
    secretMapping,         // 密钥使用记录
)

// 2. Environment → 无前缀环境变量
settings.ParamsToEnv(
    container.Environment,
    environment,
    "",                    // 无前缀
    false,                 // 不转大写
    getSecretValue,
    secretMapping,
)
```

#### 2.6.4 ParamsToEnv 深度算法

**代码位置**：`pipeline/frontend/yaml/compiler/settings/params.go`

转换规则：

| Go 类型 | 转换方式 | 示例 |
|---------|---------|------|
| `bool` | `strconv.FormatBool` | `true` → `"true"` |
| `string` | 原样传递 | `echo hi` → `"echo hi"` |
| `int/float` | `fmt.Sprintf("%v")` | `42` → `"42"` |
| 简单 slice | `strings.Join` 逗号分隔 | `[a,b,c]` → `"a,b,c"` |
| `map[string]any` 含 `from_secret` | 密钥查找替换 | `{from_secret: PWD}` → 实际密码值 |
| 复杂 map/slice | YAML → JSON 序列化 | `{k:v}` → `"{\"k\":\"v\"}"` |

`from_secret` 处理逻辑（递归）：
```go
// params.go:178-191
func injectSecret(v map[string]any, getSecretValue func...) {
    if secretNameI, ok := v["from_secret"]; ok {
        // 执行 Secret 权限检查（事件类型、插件白名单等）
        return getSecretValue(secretName)
    }
    // 否则正常递归处理子元素
}
```

密钥可用性检查（`compiler/compiler.go:49-79`）：
1. `AllowedPlugins` 过滤：设置了则只能用于匹配镜像
2. `Events` 过滤：匹配当前触发事件（tag/pull/push 等）
3. Pull Request 事件归一化：`pull/*` 统一按 `EventPull` 匹配

#### 2.6.5 特权与认证

```go
// 自动特权升级：匹配特权插件白名单的插件
if utils.MatchImageDynamic(container.Image, c.escalated...) && container.IsPlugin() {
    privileged = true
}

// 镜像仓库认证匹配
for _, registry := range c.registries {
    if utils.MatchHostname(container.Image, registry.Hostname) {
        authConfig.Username = registry.Username
        authConfig.Password = registry.Password
        break
    }
}
```

#### 2.6.6 执行条件与失败策略

```go
// 根据 When 条件推导 OnSuccess / OnFailure 标志
onSuccess := container.When.IncludesStatusSuccess(c.metadata, false, c.env)
onFailure := container.When.IncludesStatusFailure(c.metadata, false, c.env)

// 失败策略：fail（默认）/ ignore
failure := container.Failure
if container.Failure == "" {
    failure = string(metadata.FailureFail)
}
// 兼容模式：强制忽略服务失败
if c.forceIgnoreServiceFailure && detached {
    failure = string(metadata.FailureIgnore)
}
```

---

## 三、完整流程图（时序）

```
入口: parsePipeline() / execFile()
  │
  ├─► 构造 PipelineBuilder
  │      ├─ GetWorkflowMetadata
  │      ├─ Envs (全局 + 仓库)
  │      ├─ Yamls ([]*YamlFile)
  │      ├─ CompilerOptions
  │      └─ ...
  │
  ▼
PipelineBuilder.Build()
  │
  ├─► SortYamlFilesByName()        按文件名排序
  │
  └─► 对每个 YamlFile:
        │
        ├─► matrix.ParseString()   解析 matrix → []Axis
        │     ├─ try parseList()   matrix.include 模式
        │     └─ try parse() calc() 笛卡尔积
        │
        └─► 对每个 Axis:
              │
              └─► genItemForWorkflow()
                    │
                    ├─► 1. 合并环境变量 (metadata + axis + envs)
                    ├─► 2. EnvVarSubst()                YAML 内 $VAR 替换
                    ├─► 3. yaml.ParseString()           YAML → Workflow 结构体
                    ├─► 4. linter.Lint()                静态校验
                    ├─► 5. When.Match() → skip?         Workflow 级过滤
                    │
                    ├─► 6. toInternalRepresentation()
                    │     │
                    │     └─► Compiler.Compile()
                    │           │
                    │           ├─► 创建 Volume/Network 名
                    │           ├─► Clone 阶段注入        StepTypeClone
                    │           ├─► Services 阶段注入     StepTypeService
                    │           │
                    │           └─► Steps 处理:
                    │                 │
                    │                 ├─► When 过滤
                    │                 ├─► createProcess():
                    │                 │     ├─ UUID 生成
                    │                 │     ├─ 工作目录/卷/网络计算
                    │                 │     ├─► ParamsToEnv() settings→env
                    │                 │     │     └─► injectSecret() 密钥替换
                    │                 │     ├─ 特权/认证计算
                    │                 │     ├─ OnSuccess/OnFailure
                    │                 │     └─ 组装 backend_types.Step
                    │                 │
                    │                 └─► 包装 dagCompilerStep{position, dependsOn}
                    │
                    └─► 7. DAG 编译:
                          │
                          ├─► isDAG()?
                          │     ├─ YES → compileByDependsOn()
                          │     │     ├─ 缺失依赖检查
                          │     │     ├─ DFS 循环检测
                          │     │     └─ 拓扑分层 (Kahn 算法)
                          │     └─ NO  → compileSequence() 一步一 Stage
                          │
                    └─► 8. 组装 Item + 打内部 Labels

filterMissingDependencies()   Workflow 级依赖清理
  │
  ▼
返回 []*Item (每个含 Workflow + Config{Stages})
```

---

## 四、关键文件清单

| 文件路径 | 职责 |
|---------|------|
| `server/pipeline/items.go` | 服务端入口：组装 PipelineBuilder、保存结果到 DB |
| `cli/exec/exec.go` | CLI 本地执行入口 |
| `pipeline/frontend/builder/builder.go` | 多文件/Matrix 编排主流程 |
| `pipeline/frontend/builder/types.go` | Item / Workflow / YamlFile 类型定义 |
| `pipeline/frontend/builder/utils.go` | 依赖过滤、路径清洗工具 |
| `pipeline/frontend/yaml/parse.go` | YAML 文本 → Workflow 结构体 |
| `pipeline/frontend/yaml/matrix/matrix.go` | Matrix 笛卡尔积计算 |
| `pipeline/frontend/yaml/linter/linter.go` | 静态语法与策略校验 |
| `pipeline/frontend/yaml/compiler/compiler.go` | Compiler 核心：阶段注入 + 编排 |
| `pipeline/frontend/yaml/compiler/convert.go` | Container → Step 转换 |
| `pipeline/frontend/yaml/compiler/dag.go` | DAG 拓扑排序 |
| `pipeline/frontend/yaml/compiler/option.go` | Compiler Option 定义 |
| `pipeline/frontend/yaml/compiler/settings/params.go` | Settings → Env 转换 & 密钥注入 |
| `pipeline/frontend/yaml/types/workflow.go` | Workflow YAML 类型定义 |
| `pipeline/frontend/yaml/types/container.go` | Container YAML 类型定义 |
| `pipeline/backend/types/config.go` | 后端 Config IR |
| `pipeline/backend/types/stage.go` | 后端 Stage IR |
| `pipeline/backend/types/step.go` | 后端 Step IR |
| `pipeline/frontend/metadata/substitution.go` | 环境变量替换 |

---

## 五、编译产出物结构

最终的 `[]*builder.Item` 结构：

```
[]*builder.Item
└── Item
    ├── Workflow          # Workflow 元信息
    │   ├── PID           # 全局进程号 (1,2,3...)
    │   ├── Name          # 配置文件名 (已清洗)
    │   ├── Environ       # 此 workflow 的矩阵环境变量
    │   └── AxisID        # 矩阵轴编号
    │
    ├── Config            # backend_types.Config (核心可执行单元)
    │   ├── Network       # 共享网络名
    │   ├── Volume        # 共享卷名
    │   ├── Secrets       # 密钥列表（日志掩码）
    │   └── Stages[]      # 执行阶段（串行）
    │       └── Stage
    │           └── Steps[]      # 同阶段步骤（并行）
    │               └── Step
    │                   ├── UUID         # 唯一标识
    │                   ├── Type         # clone/service/plugin/commands
    │                   ├── Image        # 镜像
    │                   ├── Commands     # 命令列表
    │                   ├── Entrypoint   # 入口点
    │                   ├── Environment  # 环境变量（含 PLUGIN_*）
    │                   ├── SecretMapping # 使用的密钥映射
    │                   ├── Volumes      # 挂载列表
    │                   ├── Networks     # 网络列表
    │                   ├── AuthConfig   # 镜像仓库认证
    │                   ├── Privileged   # 特权模式
    │                   ├── Detached     # 后台模式
    │                   ├── OnSuccess    # 成功时执行?
    │                   ├── OnFailure    # 失败时执行?
    │                   └── Failure      # fail/ignore
    │
    ├── Labels           # Workflow 标签 (含内部标签)
    ├── DependsOn        # 依赖的其他 Workflow
    └── RunsOn           # [success, failure] 执行条件
```

这就是整个流水线 DSL 解析、编译、组装的完整过程。
