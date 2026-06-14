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

---

## 六、DSL 校验失败的错误反馈与定位机制

### 6.1 错误类型体系

Woodpecker 将编译期所有错误统一抽象为 `PipelineError`（`pipeline/errors/pipeline.go:33-42`），包含 5 种类型：

```go
type PipelineErrorType string

const (
    PipelineErrorTypeLinter      PipelineErrorType = "linter"      // 配置语法错误
    PipelineErrorTypeDeprecation PipelineErrorType = "deprecation" // 使用了废弃特性
    PipelineErrorTypeCompiler    PipelineErrorType = "compiler"    // 配置语义错误
    PipelineErrorTypeGeneric     PipelineErrorType = "generic"     // 通用错误
    PipelineErrorTypeBadHabit    PipelineErrorType = "bad_habit"   // 不良实践
)

type PipelineError struct {
    Type      PipelineErrorType `json:"type"`       // 错误分类
    Message   string            `json:"message"`    // 人类可读的错误描述
    IsWarning bool              `json:"is_warning"` // 是否为警告（不阻塞）
    Data      any               `json:"data"`       // 结构化错误详情
}
```

### 6.2 错误严重级别与阻断逻辑

**核心判断函数**（`pipeline/errors/linter.go:69-82`）：

```go
func HasBlockingErrors(err error) bool {
    errs := GetPipelineErrors(err)
    for _, err := range errs {
        if !err.IsWarning {
            return true  // 只要有一个 IsWarning=false，整个编译中止
        }
    }
    return false
}
```

| 严重级别 | `IsWarning` | 行为 | 举例 |
|---------|-------------|------|------|
| **阻塞错误** | `false` | 编译中止，流水线标记为 `error` | 缺少 `steps` 段、循环依赖、缺少必填镜像 |
| **警告** | `true` | 编译继续，错误记录到 `pipeline.Errors` 供 UI 展示 | Schema 校验失败、`runs_on` 弃用、缺少 event filter |

### 6.3 错误定位机制 — 精确到文件+字段

每种错误类型携带结构化的 `Data` 字段，实现精确定位：

#### 6.3.1 Linter 错误定位

**错误创建**（`pipeline/frontend/yaml/linter/error.go:21-28`）：

```go
func newLinterError(message, file, field string, isWarning bool) *PipelineError {
    return &PipelineError{
        Type:    PipelineErrorTypeLinter,
        Message: message,
        Data:    &LinterErrorData{File: file, Field: field},
        IsWarning: isWarning,
    }
}
```

`LinterErrorData` 结构（`pipeline/errors/linter.go:23-26`）：

```go
type LinterErrorData struct {
    File  string `json:"file"`  // YAML 文件名（如 "build" 或 "deploy"）
    Field string `json:"field"` // 精确到字段路径（如 "steps.build.image"）
}
```

示例错误输出：
```
[linter] Invalid or missing image
  File: "build"
  Field: "steps.my-step"
```

#### 6.3.2 Deprecation 错误定位

```go
type DeprecationErrorData struct {
    File  string `json:"file"`   // 文件名
    Field string `json:"field"`  // 废弃字段路径
    Docs  string `json:"docs"`   // 迁移文档 URL
}
```

示例：使用 `runs_on` 时：
```
[deprecation] Usage of `runs_on` is deprecated, use `when.status`
  File: "deploy"
  Field: "deploy.runs_on"
  Docs:  "https://woodpecker-ci.org/docs/usage/workflow-syntax#status"
```

#### 6.3.3 BadHabit 错误定位

```go
type BadHabitErrorData struct {
    File  string `json:"file"`
    Field string `json:"field"`  // 如 "steps.build" 或 "steps.build.when[0]"
    Docs  string `json:"docs"`
}
```

示例：步骤缺少 event filter 时：
```
[bad_habit] Consider adding a `when` block with an `event` filter to this step or the entire workflow
  File: "test"
  Field: "steps.build"
  Docs:  "https://woodpecker-ci.org/docs/usage/linter#event-filter-for-all-steps"
```

#### 6.3.4 Compiler 错误 — 4 种细粒度类型

**代码位置**：`pipeline/frontend/yaml/compiler/errors.go`

| 错误类型 | 触发场景 | 错误消息示例 |
|---------|---------|-------------|
| `ErrExtraHostFormat` | `extra_hosts` 格式错误 | `extra host "bad_host" is in wrong format` |
| `ErrStepMissingDependency` | depends_on 引用不存在的步骤 | `step 'deploy' depends on unknown step 'buildx'` |
| `ErrStepFilteredDependency` | 依赖步骤存在但被 When 过滤掉 | `step 'deploy' depends on step 'build' which is filtered out by its conditions` |
| `ErrStepDependencyCycle` | 步骤间存在循环依赖 | `cycle detected: [build, test, build]` |

**编译器错误的特殊处理**（`compiler/compiler.go:270-278`）：

```go
var missingDepErr *ErrStepMissingDependency
if errors.As(err, &missingDepErr) {
    if _, inConfig := stepNames[missingDepErr.dep]; inConfig {
        // 依赖存在但被过滤 → 提供更精确的错误提示
        return nil, &ErrStepFilteredDependency{name: missingDepErr.name, dep: missingDepErr.dep}
    }
}
```

#### 6.3.5 JSON Schema 校验 — YAML 结构合规

**代码位置**：`pipeline/frontend/yaml/linter/schema/schema.go`

流程：
1. YAML 文本 → `xyaml` 解析为 `yaml.Node`
2. `yaml2json.ConvertNode()` 转为 JSON
3. `gojsonschema.Validate()` 使用嵌入的 `schema.json` 校验
4. 过滤冗余组合错误（`filterRedundantCompositionErrors`）

Schema 校验结果中每个 `ResultError` 包含：
- `Field()`：出错字段的 JSON Path（如 `(root).steps.0.image`）
- `Description()`：人类可读的校验失败描述

### 6.4 错误生命周期 — 从产生到用户可见

```
Linter/Compiler 产生错误
       │
       ▼
multierr.Append() 聚合多个错误
       │
       ▼
genItemForWorkflow() 返回 errorsAndWarnings
       │
       ├── HasBlockingErrors() == true → 返回 nil, err
       │     │
       │     ▼
       │   createPipelineItems() 检测到阻塞错误
       │     │
       │     ▼
       │   handleParseErrors() → 返回 blocking=true
       │     │
       │     ▼
       │   UpdateToStatusError() → pipeline.Status = "error"
       │     │                     pipeline.Errors = GetPipelineErrors(err)
       │     ▼
       │   存入数据库 + 通知 Forge（如 PR 状态检查失败）
       │
       └── HasBlockingErrors() == false → 继续编译
             │
             ▼
           pipeline.Errors = GetPipelineErrors(parseErr) → 仅记录警告
             │
             ▼
           流水线正常运行，UI 显示警告信息
```

**关键函数**（`server/pipeline/pipeline_status.go:65-71`）：

```go
func UpdateToStatusError(store store.Store, pipeline model.Pipeline, err error) (*model.Pipeline, error) {
    pipeline.Errors = errors.GetPipelineErrors(err)  // 结构化错误写入 pipeline 记录
    pipeline.Status = model.StatusError
    pipeline.Started = time.Now().Unix()
    pipeline.Finished = pipeline.Started
    return &pipeline, store.UpdatePipeline(&pipeline)
}
```

`GetPipelineErrors()` 的聚合逻辑（`pipeline/errors/linter.go:52-67`）：遍历 `multierr` 展开的所有错误，将原生 `PipelineError` 直接保留，其他错误包装为 `PipelineErrorTypeGeneric`。

### 6.5 错误反馈设计要点总结

| 设计要点 | 实现方式 |
|---------|---------|
| **精确文件定位** | 每个错误携带 `File` 字段（来自 `YamlFile.Name`，经 `SanitizePath` 清洗） |
| **精确字段定位** | `Field` 使用点分路径（如 `steps.build.depends_on`） |
| **错误/警告二分** | `IsWarning` 区分，阻塞错误中止编译，警告仅记录 |
| **多错误聚合** | `multierr.Append()` 不会因首个错误丢弃后续错误 |
| **Schema 兼容** | JSON Schema 校验当前仅产生警告，不阻塞 |
| **迁移指引** | Deprecation/BadHabit 错误附带 `Docs` URL 指向迁移文档 |
| **运行时掩码** | `config.Secrets` 收集所有密钥值用于日志脱敏 |
| **UI 可见** | `pipeline.Errors` 序列化为 JSON 返回前端，在流水线页面展示 |

---

## 七、编译产物的缓存与复用策略

### 7.1 内存层 — 无缓存，每次全量编译

**Woodpecker 的编译管线在进程内没有任何缓存机制**。每次调用 `PipelineBuilder.Build()` 都会完整执行以下流程：

```
YAML 原文 → 环境变量替换 → YAML 反序列化 → Linter 校验 → Compiler 编译 → DAG 生成
```

原因分析：
1. **上下文敏感性**：同一段 YAML 在不同事件（push/tag/PR）、不同分支、不同 Matrix 轴下编译结果不同。环境变量替换（`EnvVarSubst`）和 `When.Match()` 过滤都依赖运行时元数据。
2. **无状态设计**：`Compiler` 和 `PipelineBuilder` 每次调用 `New()` 创建新实例，不保留历史状态。
3. **编译耗时可控**：纯 CPU 运算，无 I/O 阻塞，单文件编译通常在毫秒级完成。

### 7.2 持久化层 — 配置文件哈希去重

虽然编译结果不缓存，但**原始 YAML 配置文件**在持久化时有去重机制。

**代码位置**：`server/store/datastore/config.go:48-70`

```go
func (s storage) ConfigPersist(conf *model.Config) (*model.Config, error) {
    // 1. 计算内容 SHA-256 哈希
    conf.Hash = fmt.Sprintf("%x", sha256.Sum256(conf.Data))

    // 2. 事务内查找相同 repoID + hash + name 的已有记录
    existingConfig, err := s.configFindIdentical(sess, conf.RepoID, conf.Hash, conf.Name)
    if existingConfig != nil {
        return existingConfig, nil  // 3. 命中则直接返回已有记录，不重复插入
    }

    // 4. 未命中则创建新记录
    s.configCreate(sess, conf)
    return conf, sess.Commit()
}
```

去重维度：`(repo_id, hash, name)` 三元组。相同仓库、相同内容、相同文件名的配置只存储一份。

**调用位置**（`server/pipeline/create.go:117-125`）：

```go
for _, forgeYamlConfig := range forgeYamlConfigs {
    config, err := findOrPersistPipelineConfig(_store, pipeline, forgeYamlConfig)
    // config 已存在则返回已有记录
}
```

### 7.3 配置获取层 — 重启时的配置复用

**代码位置**：`server/services/config/forge.go:49-53`

```go
func (f *forgeFetcher) Fetch(..., oldConfigData []*types.FileMeta, restart bool) {
    // 重启时直接复用旧配置，不再从 Forge 拉取
    if restart && len(oldConfigData) > 0 {
        return oldConfigData, nil
    }
    // ... 正常从 Forge 获取
}
```

Pipeline 重启时（`restart=true`），跳过从 Forge 拉取配置文件，直接复用已存储的配置数据。

### 7.4 外部配置服务 — HTTP 扩展的回退机制

**代码位置**：`server/services/config/http.go:57-102`

```
Forge 获取配置 → HTTP 扩展服务处理
                       │
                       ├─ 200 OK     → 使用扩展返回的新配置
                       ├─ 204 No Content → 回退到 Forge 原始配置（oldConfigData）
                       └─ 其他错误   → 返回 oldConfigData + 错误
```

`config.NewCombined(forgeFetcher, httpFetcher)` 的链式处理（`server/services/config/combined.go:33-39`）：

```go
func (c *combined) Fetch(...) (files []*types.FileMeta, err error) {
    files = oldConfigData
    for _, s := range c.services {
        files, err = s.Fetch(ctx, forge, user, repo, pipeline, files, restart)
    }
    return files, err
}
```

Forge 获取的结果作为 HTTP 服务的输入，HTTP 服务可修改/替换/原样返回配置。204 表示"不做修改"，实现优雅回退。

### 7.5 缓存策略总结

| 层级 | 缓存策略 | 理由 |
|------|---------|------|
| **内存编译** | 无缓存，每次全量编译 | 上下文敏感（事件/分支/矩阵轴），编译快 |
| **DB 配置存储** | SHA-256 哈希去重 | 避免相同配置重复存储，节省空间 |
| **Pipeline 重启** | 复用已存储配置 | 无需再从 Forge 拉取，保证配置一致性 |
| **HTTP 配置扩展** | 204 回退机制 | 扩展服务不可用时自动降级到原始配置 |
| **Forge 配置拉取** | 可配置重试次数 | `forgeFetcher.retryCount` 应对网络抖动 |

---

## 八、DSL 演进与版本兼容性

### 8.1 版本演进时间线

Woodpecker CI 从 Drone v0.8 分叉后，DSL 经历了多次重大变更：

```
Drone v0.8  ──fork──►  Woodpecker v0.14  ──►  v1.0  ──►  v2.0  ──►  v3.0 (当前)
    │                      │                  │         │          │
    │                      │                  │         │          │
  .drone.yml          .woodpecker.yml    关键重构     大清理     语法严化
  pipeline:           pipeline:          移除command:  移除CI_BUILD_*  移除secrets:
  build:              steps:             API 用 ID    移除SSH后端    移除pipeline:
    ...                  ...             ed25519签名  platform→labels  branches→when.branch
```

### 8.2 各版本关键语法差异

#### 8.2.1 Drone → Woodpecker v0.x：基础迁移

| Drone 语法 | Woodpecker v0.x 语法 | 变更原因 |
|-----------|---------------------|---------|
| `.drone.yml` | `.woodpecker.yml` / `.woodpecker/` | 品牌独立 |
| `build:` 段 | `pipeline:` 段 | 术语统一（build→pipeline） |
| `DRONE_*` 环境变量 | `CI_*` 环境变量 | 通用 CI 前缀 |
| 工作空间 `/drone` | 工作空间 `/woodpecker` | 品牌独立 |

#### 8.2.2 v0.x → v1.x：API 重构与安全升级

| 变更项 | v0.x | v1.x | 影响 |
|--------|------|------|------|
| `command:` | 支持 | **移除** | 必须用 `commands:`（数组） |
| 配置扩展签名 | HMAC 共享密钥 | ed25519 密钥对 | 安全升级 |
| API 路由 | `owner/repo` | `repo-id`（数字 ID） | 路由重构 |
| 日志系统 | 旧格式 | 新格式 | 需迁移脚本 |

#### 8.2.3 v1.x → v2.x：环境变量大清理

| 变更项 | v1.x | v2.x | 影响 |
|--------|------|------|------|
| `CI_BUILD_*` 变量 | 存在（已标记废弃） | **移除** | 必须用 `CI_PIPELINE_*` |
| `platform:` 过滤器 | 存在（已标记废弃） | **移除** | 必须用 `labels:` |
| SSH 后端 | 支持 | **移除** | 改用 `local` 后端或 ssh 命令 |
| Secret 的 `plugin_only` | 支持 | **移除** | 改用 `image` 过滤器 |
| `build` CLI 命令 | 存在（已标记废弃） | **移除** | 必须用 `pipeline` 命令 |

#### 8.2.4 v2.x → v3.x（当前版本）：语法严化

| 变更项 | v2.x | v3.x | 影响 |
|--------|------|------|------|
| `secrets: [token]` | 支持（有警告） | **移除** | 必须用 `environment: { TOKEN: { from_secret: token } }` |
| `pipeline:` 段名 | 支持（有警告） | **报错** | 必须用 `steps:` |
| `branches:` | 支持（有警告） | **报错** | 必须用 `when.branch` |
| `platform:` | 支持（有警告） | **报错** | 必须用 `labels:` |
| `environment:` 列表语法 | 支持 | **移除** | 必须用映射语法 |
| 环境变量自动大写 | 默认行为 | **移除** | 保留原始大小写 |
| `steps.[name].group` | 支持 | **废弃** | 改用 `depends_on` |
| `includes/excludes` 事件过滤 | 支持 | **移除** | 改用 `when.event` |
| `gated` 设置 | 支持 | **移除** | 改用 `require-approval` |

### 8.3 当前代码中的兼容性处理

#### 8.3.1 `runs_on` 废弃兼容

**YAML 解析层**仍接受 `runs_on`（`pipeline/frontend/yaml/types/workflow.go:33`）：

```go
// Deprecated: use when.status. TODO remove in next major.
RunsOn []string `yaml:"runs_on,omitempty"`
```

**Linter 层**发出废弃警告（`pipeline/frontend/yaml/linter/linter.go:318-332`）：

```go
if len(parsed.RunsOn) > 0 { //nolint:staticcheck
    err = multierr.Append(err, &PipelineError{
        Type:      PipelineErrorTypeDeprecation,
        IsWarning: true,
        Message:   "Usage of `runs_on` is deprecated, use `when.status`",
        Data: DeprecationErrorData{...},
    })
}
```

**Builder 层**自动将 `runs_on` 转换为等价的 `RunsOn` 字段（`pipeline/frontend/builder/builder.go:178-183`）：

```go
if !slices.Contains(item.RunsOn, "failure") && parsed.When.IncludesStatusFailure(...) {
    item.RunsOn = append(item.RunsOn, "failure")
}
if !slices.Contains(item.RunsOn, "success") && parsed.When.IncludesStatusFailure(...) {
    item.RunsOn = append(item.RunsOn, "success")
}
```

#### 8.3.2 `forceIgnoreServiceFailure` 兼容

整个调用链保留此选项，计划在 4.x 移除：

```
server/config.go:88     ForceIgnoreServiceFailure bool
    ↓
server/pipeline/items.go:151-152  compiler.WithForceIgnoreServiceFailure()
    ↓
pipeline/frontend/yaml/compiler/option.go:203-208  WithForceIgnoreServiceFailure()
    ↓
pipeline/frontend/yaml/compiler/compiler.go:98-99   forceIgnoreServiceFailure bool
    ↓
pipeline/frontend/yaml/compiler/convert.go:159-162  if c.forceIgnoreServiceFailure && detached { failure = ignore }
```

#### 8.3.3 Drone CI 插件兼容

**代码位置**：`pipeline/frontend/metadata/drone_compatibility.go:19-72`

在**运行时**为 Plugin 类型步骤注入 `DRONE_*` 环境变量（`pipeline/runtime/step.go:95-96`）：

```go
if step.Type == backend_types.StepTypePlugin {
    metadata.SetDroneEnviron(step.Environment)
}
```

映射关系（部分）：

| Woodpecker 变量 | Drone 兼容变量 |
|-----------------|---------------|
| `CI_COMMIT_BRANCH` | `DRONE_BRANCH` |
| `CI_COMMIT_SHA` | `DRONE_COMMIT_SHA` |
| `CI_PIPELINE_NUMBER` | `DRONE_BUILD_NUMBER` |
| `CI_REPO` | `DRONE_REPO` |
| `CI_REPO_CLONE_URL` | `DRONE_REMOTE_URL` |

#### 8.3.4 环境变量废弃兼容

`pipeline/frontend/metadata/environment.go` 中多处标记了待移除的环境变量：

```go
// Deprecated remove in 4.x
setNonEmptyEnvVar(params, "CI_REPO_TRUSTED", ...)          // 合并值 → 应分别用 CI_REPO_TRUSTED_NETWORK/VOLUMES/SECURITY

// TODO Deprecated, remove in next major
setNonEmptyEnvVar(params, "CI_COMMIT_AUTHOR_AVATAR", ...)   // → 应使用 CI_PIPELINE_AVATAR
setNonEmptyEnvVar(params, "CI_PREV_COMMIT_AUTHOR_AVATAR", ...) // → 应使用 CI_PREV_PIPELINE_AVATAR
```

#### 8.3.5 ContainerList 双语法兼容

**代码位置**：`pipeline/frontend/yaml/types/container_list.go:29-77`

`steps`、`clone`、`services` 段同时支持**映射语法**和**列表语法**：

```yaml
# 映射语法（key 作为步骤名）
steps:
  build:
    image: golang
    commands: [go build]

# 列表语法（name 字段指定步骤名）
steps:
  - name: build
    image: golang
    commands: [go build]
```

实现原理：自定义 `UnmarshalYAML` 根据 `yaml.Kind` 分派：
- `yaml.MappingNode` → 遍历 `Content` 取奇数索引值，key 作为 `Name`
- `yaml.SequenceNode` → 遍历 `Content`，缺省 `Name` 为 `step-{i}`

### 8.4 废弃策略的执行流程

Woodpecker 遵循严格的废弃策略（参考官方文档 [Deprecation Policy](https://woodpecker-ci.org/docs/next/development/deprecations)）：

```
版本 N.x（小版本）           版本 (N+1).0（大版本）       版本 (N+1).x（小版本）
─────────────────         ──────────────────         ──────────────────
• Linter 添加警告           • 警告变为错误               • 废弃代码路径移除
• 旧语法仍可正常工作         • 旧语法不再支持              • 解析器简化
• 文档更新为新语法           • 迁移指南记录破坏性变更       • 不再识别旧语法
• 警告消息包含迁移指引       • 用户必须更新配置
```

**实际案例**：`secrets: [token]` 的废弃时间线：
- **v2.5.0**：Linter 添加废弃警告，`secrets` 语法仍可工作
- **v2.6-2.9**：警告持续，两种语法并存
- **v3.0.0**：Linter 报错，`secrets` 语法不再支持（破坏性变更）
- **v3.1.0**：废弃代码路径移除，解析器简化

### 8.5 Schema 校验与语法演进

**代码位置**：`pipeline/frontend/yaml/linter/schema/schema.json`

JSON Schema 是 DSL 语法的权威定义，随版本同步更新：
- `runs_on` 字段标记 `"description": "Deprecated: use when.status instead"` 但仍允许（兼容期）
- `depends_on` 支持 string 和 object 两种格式（新增 `depends_on_item` 定义支持 `name` + `optional` 字段）
- 新增字段（如 `backend_options`）通过 Schema 扩展而非修改核心类型

Schema 校验当前仅产生**警告**（`IsWarning: true`），不阻塞编译，这为语法演进提供了缓冲期。

---

## 九、动态变量与 Secret 注入的编译期处理

### 9.1 环境变量分层注入模型

Woodpecker 的环境变量注入分为 **编译期** 和 **运行期** 两个阶段，三层来源：

```
编译期 (Compile-time)
    ├── 层 1：CI 元数据环境变量（Metadata.Environ()）
    │       - CI_REPO, CI_PIPELINE_NUMBER, CI_COMMIT_SHA 等
    │       - 来自当前触发事件的静态信息
    │
    ├── 层 2：Matrix 轴变量（Axis）
    │       - GO_VERSION, DATABASE 等矩阵维度变量
    │       - 每个矩阵轴独立一组
    │
    └── 层 3：用户自定义环境变量（container.Environment）
            - YAML 中 environment 段定义的变量
            - 支持 from_secret 语法注入密钥

运行期 (Runtime)
    └── 层 4：运行时补充变量
            - CI_PIPELINE_STATUS（success/failure）
            - CI_PIPELINE_STARTED（运行起始时间戳）
            - CI_STEP_NAME, CI_STEP_TYPE, CI_STEP_STARTED
            - 步骤级别的动态信息
```

### 9.2 编译期变量替换 — EnvVarSubst

**代码位置**：`pipeline/frontend/metadata/substitution.go:24-31`

发生在 YAML 解析之前，对整段 YAML 文本做变量插值：

```go
func EnvVarSubst(data string, envs map[string]string) (string, error) {
    tmpl, err := envsubst.Parse(data)
    if err != nil {
        return "", &pipeline_errors.PipelineError{...}
    }
    return tmpl.Execute(envs)
}
```

**替换时机**：`PipelineBuilder.genItemForWorkflow()` → `yaml.ParseString()` 之前。

**支持的语法**（来自 `github.com/drone/envsubst`）：
- `${VAR}` / `$VAR` — 直接替换
- `${VAR:-default}` — 变量为空时使用默认值
- `${VAR:=default}` — 变量为空时使用默认值**并赋值**
- `${VAR:?error}` — 变量为空时报错
- 等等（共 15+ 种 bash 风格语法）

**特殊处理**：含换行符的值会被自动加引号转义，避免破坏 YAML 结构。

### 9.3 Secret 注入机制 — from_secret

#### 9.3.1 注入路径

Secret 注入发生在 **编译期** 的 `createProcess()` 阶段，有两条独立的注入通道：

```go
// settings → PLUGIN_* 前缀环境变量
settings.ParamsToEnv(container.Settings, environment, "PLUGIN_", true, getSecretValue, secretMapping)

// environment → 无前缀环境变量
settings.ParamsToEnv(container.Environment, environment, "", false, getSecretValue, secretMapping)
```

两条通道都通过 `getSecretValue` 回调函数访问密钥池。

#### 9.3.2 Secret 查找与权限校验

**代码位置**：`pipeline/frontend/yaml/compiler/compiler.go:42-79`

`getSecretValue` 闭包实现了两层权限校验：

```go
getSecretValue := func(name string) (string, error) {
    name = strings.ToLower(name)
    secret, ok := c.secrets[name]   // 从编译期密钥池查找
    if !ok {
        return "", fmt.Errorf("secret %q not found", name)
    }

    event := c.metadata.Curr.Event
    err := secret.Available(event, container)  // 权限校验
    if err != nil {
        return "", err
    }
    return secret.Value, nil
}
```

`Secret.Available()` 校验顺序（`compiler.go:49-64`）：

1. **Plugin 限制**：如果设置了 `AllowedPlugins`
   - 步骤必须是 plugin 类型（`container.IsPlugin()`）
   - 步骤镜像必须匹配白名单（`MatchImageDynamic`）
2. **事件限制**：如果设置了 `Events`
   - 当前事件必须在允许列表中
   - Pull Request 相关事件统一归一化为 `EventPull` 再匹配

#### 10.3.3 Secret 递归注入算法

**代码位置**：`pipeline/frontend/yaml/compiler/settings/params.go`

`from_secret` 语法支持**递归注入**，即可以出现在任意深度的嵌套结构中：

```yaml
steps:
  deploy:
    settings:
      ssh:
        host: example.com
        key:
          from_secret: ssh_key   # 深层嵌套中的 from_secret
        port: 22
```

算法流程（`injectSecretRecursive`）：

```
sanitizeParamValue(v)
  ├── 基本类型（bool/string/int/float）→ 直接字符串化
  ├── map[string]any
  │     ├── 检查是否为 {from_secret: "xxx"} 结构
  │     │     └── 是 → 调用 getSecretValue() 返回密钥值
  │     └── 否 → 递归处理每个 value → 最终 YAML→JSON 序列化
  └── slice/array
        ├── 全为简单类型 → strings.Join 逗号分隔
        └── 含复杂类型 → 递归每个元素 → YAML→JSON 序列化
```

#### 9.3.4 SecretMapping — 密钥使用追踪

**代码位置**：`pipeline/backend/types/step.go:30`

```go
type Step struct {
    SecretMapping map[string]string `json:"secret_mapping,omitempty"`
    ...
}
```

用途：
1. **日志脱敏**：运行时根据 `SecretMapping` 中的值对日志进行掩码替换
2. **审计追踪**：记录哪些环境变量实际来源于密钥，以及密钥名与环境变量名的对应关系
3. **密钥统计**：可以通过 mapping 统计每个 pipeline 使用了多少密钥

填充时机：`ParamsToEnv` 中每次调用 `getSecretValue` 成功后，将 `sanitizedParamKey → secretValue` 加入 mapping。

### 10.4 编译期 vs 运行期变量边界

| 特性 | 编译期确定 | 运行期确定 |
|------|-----------|-----------|
| CI 元数据（提交/分支/仓库） | ✅ | - |
| Matrix 轴变量 | ✅ | - |
| 用户 environment 变量 | ✅ | - |
| Settings（PLUGIN_*） | ✅ | - |
| Secret 值 | ✅ | - |
| 流水线状态（success/failure） | - | ✅ |
| 步骤开始时间戳 | - | ✅ |
| 步骤名称/类型 | - | ✅ |

> **重要**：Woodpecker **不支持步骤间传递环境变量**。每个步骤的环境变量在编译期确定（加运行时少量补充），步骤 A 设置的环境变量不会传递给步骤 B。如果需要步骤间数据传递，必须通过**共享工作区卷**写文件实现。

### 9.5 动态变量的替代方案

由于编译期变量都是静态确定的，Woodpecker 没有原生的"动态变量"机制。常见替代方案：

1. **共享卷文件**：步骤 A 写入 `$CI_WORKSPACE/.env`，步骤 B `source` 读取
2. **插件输出**：某些插件会输出文件到工作区供后续步骤使用
3. **配置扩展服务**：通过 HTTP 配置扩展服务动态生成/修改 YAML
4. **Matrix 矩阵**：在编译期就展开为多个独立 workflow，属于编译时展开而非运行时动态

---

## 十、流水线步骤间数据传递 — 共享卷与插件协议

### 10.1 共享工作区卷 — 数据传递的核心机制

Woodpecker 的步骤间数据传递完全依赖**共享数据卷**，这是编译期就规划好的基础设施。

#### 10.1.1 卷的创建与命名

**代码位置**：`pipeline/frontend/yaml/compiler/compiler.go:124,140`

```go
// 全局共享卷名
config.Volume = fmt.Sprintf("%s_default", c.prefix)
```

命名规则：`<前缀>_default`
- 前缀由 `WithPrefix()` 设置（`compiler/option.go:139-143`）
- 服务端通常设为 `wp-<pipeline_id>` 或类似唯一标识
- 目的：多个并发流水线之间的资源隔离

#### 10.1.2 卷的挂载路径

每个步骤都自动挂载共享卷：

```go
// convert.go:52-56,80-83
workspaceBase := c.workspaceBase
if container.IsPlugin() {
    workspaceBase = pluginWorkspaceBase  // "/woodpecker"
}
workspaceVolume := fmt.Sprintf("%s_default:%s", c.prefix, workspaceBase)

// 非本地模式下自动挂载
if !c.local {
    volumes = append(volumes, workspaceVolume)
}
```

**普通步骤**：工作区基目录 = 用户配置的 `workspace.base`（默认 `/woodpecker`）
**插件步骤**：工作区基目录固定为 `/woodpecker`（防止污染插件入口点）

工作目录计算：
```
普通步骤：<workspace.base>/<workspace.path>/<container.directory>
插件步骤：/woodpecker/<workspace.path>/<container.directory>
```

`CI_WORKSPACE` 环境变量指向完整工作目录路径。

#### 11.1.3 共享卷的生命周期

```
SetupWorkflow()          步骤执行期          DestroyWorkflow()
      │                       │                       │
      ▼                       ▼                       ▼
  创建卷               所有步骤读写              删除卷
  （Docker volume /    （同一挂载点）          （连同所有数据）
   Kubernetes PVC /
   本地目录）
```

整个 workflow 内所有步骤共享同一个卷，workflow 结束后卷被销毁。

### 10.2 Artifact 传递 — 基于文件系统的约定

Woodpecker **没有内建的 Artifact 系统**（没有 artifact DSL 关键字）。Artifact 传递完全通过共享工作区卷的文件系统约定实现。

#### 10.2.1 常见模式

```yaml
steps:
  build:
    image: golang
    commands:
      - go build -o output/myapp .   # 构建产物写入工作区

  test:
    image: golang
    commands:
      - ./output/myapp --test        # 后续步骤直接读取

  deploy:
    image: woodpeckerci/plugin-s3
    settings:
      source: output/myapp          # 插件读取工作区文件上传
      target: /release/
```

**关键要点**：
- 所有步骤在同一工作目录下操作
- 前序步骤写入的文件，后序步骤可以直接读取
- 插件也是通过挂载共享卷来访问工作区文件
- 没有"上传 artifact"/"下载 artifact"的显式概念

#### 10.2.2 跨 Workflow 传递

跨 workflow 的 artifact 传递**不支持**（每个 workflow 有独立的共享卷）。
如果需要跨 workflow 传递数据，通常的做法：
- 使用外部存储（S3/对象存储）作为中转站
- 合并到同一个 workflow 中
- 使用 `depends_on` 保证执行顺序但数据仍不共享

### 10.3 Cache 机制 — StepTypeCache 与插件实现

#### 10.3.1 StepTypeCache 类型定义

**代码位置**：`pipeline/backend/types/step.go:58`

```go
const (
    StepTypeClone    StepType = "clone"
    StepTypeService  StepType = "service"
    StepTypePlugin   StepType = "plugin"
    StepTypeCommands StepType = "commands"
    StepTypeCache    StepType = "cache"    // Cache 步骤类型
)
```

#### 10.3.2 编译器中的 Cache 处理

**重要发现**：在 `pipeline/frontend/yaml/compiler/compiler.go` 中，**没有任何步骤被标记为 `StepTypeCache`**。

Cache 步骤类型仅存在于：
- 后端类型定义（`backend/types/step.go`）
- 后端实现（`kubernetes/pod.go` 等，用于命名前缀区分）
- API 模型（`server/model/step.go`、`woodpecker-go/woodpecker/const.go`）

YAML DSL 中没有 `cache:` 段，编译器也不会生成 `StepTypeCache` 类型的步骤。

#### 10.3.3 Cache 的实际实现方式

Woodpecker 的缓存功能通过**插件**实现，而非 DSL 内置：

```yaml
steps:
  restore-cache:
    image: woodpeckerci/plugin-cache-s3
    settings:
      endpoint: s3.amazonaws.com
      bucket: my-cache
      restore: true          # 恢复模式
      key: "{{ .Checksum }}"
      mount:
        - node_modules

  build:
    image: node
    commands:
      - npm install

  save-cache:
    image: woodpeckerci/plugin-cache-s3
    settings:
      endpoint: s3.amazonaws.com
      bucket: my-cache
      rebuild: true          # 重建模式
      key: "{{ .Checksum }}"
      mount:
        - node_modules
```

Cache 插件协议：
1. **恢复阶段**（restore）：从外部存储下载缓存 → 解压到工作区指定路径
2. **构建阶段**：正常步骤读写工作区
3. **保存阶段**（rebuild）：将指定路径打包 → 上传到外部存储

缓存的判断逻辑在插件内部实现（如基于文件 checksum 生成 key，命中则恢复），与编译器无关。

### 11.4 插件协议 — Settings 与 PLUGIN_* 环境变量

插件（Plugin）是 Woodpecker 扩展能力的核心机制，其内部协议基于环境变量约定。

#### 10.4.1 插件类型判定

**代码位置**：`pipeline/frontend/yaml/types/container.go:60-64`

```go
func (c *Container) IsPlugin() bool {
    return len(c.Commands) == 0 &&
           len(c.Entrypoint) == 0 &&
           len(c.Environment) == 0
}
```

三个条件同时满足才被视为插件：
- 没有自定义 `commands`
- 没有自定义 `entrypoint`
- 没有自定义 `environment` 段

#### 10.4.2 Settings → PLUGIN_* 转换规则

| YAML settings | 环境变量（PLUGIN_ 前缀 + 大写 + 点/横杠转下划线） |
|--------------|--------------------------------------------------|
| `source: output/` | `PLUGIN_SOURCE=output/` |
| `bucket-name: my-bucket` | `PLUGIN_BUCKET_NAME=my-bucket` |
| `ssh.port: 22` | `PLUGIN_SSH_PORT=22` |
| `tags: [v1, v2]` | `PLUGIN_TAGS=v1,v2` |
| `config: {a: 1, b: 2}` | `PLUGIN_CONFIG={"a":1,"b":2}`（JSON 序列化） |
| `password: {from_secret: pwd}` | `PLUGIN_PASSWORD=<实际密钥值>` |

**嵌套处理逻辑**：
- **简单值**（bool/string/int/float）→ 直接字符串化
- **简单数组** → 逗号拼接（`a,b,c`）
- **复杂 map/嵌套数组** → YAML 序列化再转 JSON
- **`from_secret` 映射** → 递归替换为实际密钥值

#### 10.4.3 特权插件自动升级

**代码位置**：`pipeline/frontend/yaml/compiler/convert.go:127-129`

```go
if utils.MatchImageDynamic(container.Image, c.escalated...) && container.IsPlugin() {
    privileged = true
}
```

匹配 `escalated`（特权插件白名单）的插件自动获得 `--privileged` 权限。
白名单通过 `WithEscalated()` Option 注入（`compiler/option.go:55-58`）。

### 10.5 步骤间数据传递总结

| 传递方式 | 范围 | 机制 | 速度 | 持久化 |
|---------|------|------|------|--------|
| **共享工作区卷** | 同 Workflow 内 | 本地文件系统 | 快 | 临时（Workflow 结束销毁） |
| **Cache 插件** | 跨 Pipeline | 外部存储（S3等） | 中（网络传输） | 持久（按 key 保留） |
| **环境变量** | 单步骤内 | 进程环境 | 极快 | 不传递 |
| **Service 网络** | 同 Stage 内 | Docker/K8s 网络 | 快 | 临时 |
| **HTTP 扩展** | 编译期 | 配置生成服务 | N/A | 一次性 |

---

## 十一、Matrix Build 与 Fan-Out 扩展机制

### 11.1 Matrix 构建的本质 — 编译期展开

Woodpecker 的 Matrix（矩阵构建）不是运行时动态扩展，而是**编译期全量展开**为多个独立的 Workflow。

```
一份 YAML + Matrix 定义
        │
        ▼ 编译期展开
  ┌─────┼─────┐
  ▼     ▼     ▼
轴1   轴2   轴3   ... 轴N
  │     │     │          │
  ▼     ▼     ▼          ▼
WF1   WF2   WF3   ...  WFN
（每个都是完整独立的 workflow）
```

### 11.2 Matrix 的两种定义方式

#### 12.2.1 笛卡尔积模式（自动展开）

**代码位置**：`pipeline/frontend/yaml/matrix/matrix.go:70-112`

```yaml
matrix:
  GO_VERSION:
    - "1.21"
    - "1.22"
  DATABASE:
    - mysql
    - postgres
```

计算结果：2 × 2 = **4 个矩阵轴**

排列算法（`calc()` 函数）：
```go
func calc(matrix Matrix) []Axis {
    // 1. 计算总排列数 perm = Π len(v)  for each k,v
    // 2. 提取所有维度标签 tags
    // 3. 对 p in [0, perm):
    //    - 对每个维度 tag：
    //      decrease = perm / len(elems)
    //      elem_idx = p / decrease % len(elems)
    //      axis[tag] = elems[elem_idx]
    //    - 加入 axisList
}
```

特点：每个维度索引独立计算，保证所有组合都出现且仅出现一次。

#### 11.2.2 Include 列表模式（手动指定）

**代码位置**：`pipeline/frontend/yaml/matrix/matrix.go:124-134`

```yaml
matrix:
  include:
    - GO_VERSION: "1.21"
      DATABASE: mysql
    - GO_VERSION: "1.22"
      DATABASE: postgres
```

直接使用用户指定的组合，不做笛卡尔积计算。

解析优先级：先尝试 `parseList()`，成功且非空则直接返回；否则回退到 `parse()` + `calc()` 笛卡尔积模式。

### 11.3 Matrix 的限制与保护

**代码位置**：`pipeline/frontend/yaml/matrix/matrix.go:26-28`

```go
const (
    limitTags = 10   // 最多 10 个矩阵维度
    limitAxis = 25   // 最多 25 个排列组合
)
```

限制逻辑：
- 维度限制（`limitTags`）：`calc()` 中内层循环到第 10 个维度后 `break`，多余维度被忽略
- 组合限制（`limitAxis`）：`calc()` 中生成到第 25 个轴后 `break`，停止生成

> **注意**：超出限制时不会报错，只是静默截断。这是为了防止恶意或意外的超大矩阵导致系统过载。

### 12.4 Matrix 轴如何成为独立 Workflow

**代码位置**：`pipeline/frontend/builder/builder.go:67-86`

```go
for _, y := range b.Yamls {
    axes, err := matrix.ParseString(string(y.Data))
    if len(axes) == 0 {
        axes = append(axes, matrix.Axis{})  // 无 matrix 也有一个空轴
    }

    for i, axis := range axes {
        workflow := &Workflow{
            PID:     pidSequence,  // 全局递增 PID
            Environ: axis,         // 轴变量作为 workflow 级环境变量
            Name:    SanitizePath(y.Name),
            AxisID:  i + 1,        // 轴编号（多轴时设置）
        }
        // 每个轴独立编译
        item, err := b.genItemForWorkflow(workflow, axis, string(y.Data))
        items = append(items, item)
        pidSequence++
    }
}
```

关键特性：
1. **独立 PID**：每个矩阵轴分配独立的全局进程号
2. **独立环境**：轴变量注入 workflow 环境，参与 `EnvVarSubst` 和 `When.Match`
3. **独立编译**：每个轴调用一次 `genItemForWorkflow()`，产出独立的 `*Item`
4. **独立执行**：运行时每个 workflow 独立调度，可并行执行

### 11.5 Fan-Out 模式 — 多 Workflow 并行

Woodpecker 没有专门的 "fan-out" 关键字，但可以通过以下方式实现类似效果。

#### 11.5.1 方式一：多文件 Workflow

在 `.woodpecker/` 目录下放多个 YAML 文件：

```
.woodpecker/
├── build.yml     → workflow "build"
├── test.yml      → workflow "test"
├── lint.yml      → workflow "lint"
└── deploy.yml    → workflow "deploy"
```

每个文件独立编译为一个 workflow，默认并行执行。通过 `depends_on` 控制依赖关系：

```yaml
# deploy.yml
depends_on:
  - build
  - test
```

#### 12.5.2 方式二：Matrix 展开

单个文件内通过 matrix 展开为多个并行执行的 workflow 变体。

```yaml
matrix:
  GO_VERSION: ["1.20", "1.21", "1.22", "1.23"]

steps:
  build:
    image: golang:${GO_VERSION}
    commands: [go build]
```

展开为 4 个并行 workflow，每个使用不同的 Go 版本。

#### 11.5.3 方式三：DAG 步骤并行

在单个 workflow 内，通过 `depends_on` 实现步骤级别的 fan-out/fan-in：

```yaml
steps:
  setup:
    image: alpine
    commands: [./setup.sh]

  test-unit:
    image: golang
    commands: [go test ./unit/...]
    depends_on: [setup]      # fan-out

  test-integration:
    image: golang
    commands: [go test ./integration/...]
    depends_on: [setup]      # fan-out

  test-e2e:
    image: golang
    commands: [go test ./e2e/...]
    depends_on: [setup]      # fan-out

  deploy:
    image: alpine
    commands: [./deploy.sh]
    depends_on:              # fan-in
      - test-unit
      - test-integration
      - test-e2e
```

DAG 编译结果：
```
Stage 1: setup
Stage 2: test-unit | test-integration | test-e2e  (并行)
Stage 3: deploy
```

### 11.6 三种扩展方式对比

| 特性 | Matrix 构建 | 多文件 Workflow | DAG 步骤并行 |
|------|-----------|----------------|-------------|
| **展开时机** | 编译期 | 编译期 | 编译期 |
| **粒度** | Workflow 级 | Workflow 级 | Step 级 |
| **隔离性** | 高（独立容器/卷） | 高（独立容器/卷） | 低（共享工作区卷） |
| **并行度** | 取决于 agent 容量 | 取决于 agent 容量 | 单 agent 内并行 |
| **变量共享** | 仅 Matrix 轴变量 | 无共享 | 共享所有环境/文件 |
| **依赖控制** | 无（同文件矩阵轴相互独立） | `depends_on`（跨文件） | `depends_on`（同文件） |
| **适用场景** | 版本矩阵测试 | 异构任务分解 | 单构建多测试 |

### 11.7 Workflow 间依赖处理

**代码位置**：`pipeline/frontend/builder/utils.go:35-74`

多文件 workflow 之间的依赖通过 `depends_on` 声明，在编译完成后统一处理：

```go
func filterMissingDependencies(items []*Item) []*Item {
    // 循环直到稳定
    for {
        changed := false
        for i, item := range items {
            for _, dep := range item.DependsOn {
                if dependencyExists(dep, items) {
                    continue
                }
                if dep.Optional {
                    // 可选依赖：从列表中移除
                    item.DependsOn.Remove(dep.Name)
                    changed = true
                } else {
                    // 必需依赖：删除整个 workflow
                    items = append(items[:i], items[i+1:]...)
                    changed = true
                    break
                }
            }
        }
        if !changed {
            break
        }
    }
    // 最后将所有幸存的依赖标记为非可选
    for _, item := range items {
        for _, dep := range item.DependsOn {
            dep.Optional = false
        }
    }
    return items
}
```

处理逻辑：
1. 检查每个 workflow 的每个依赖是否存在
2. 必需依赖不存在 → 整个 workflow 被移除（连锁反应）
3. 可选依赖不存在 → 仅移除该依赖声明
4. 循环直到没有新变化
5. 最终所有保留的依赖都标记为必需（`optional=false`）

### 12.8 Matrix + Fan-Out 的组合使用

实际项目中经常组合使用：

```
.woodpecker/
├── test.yml          # matrix: {go: [1.21, 1.22], os: [linux, darwin]} → 4 个 workflow
├── build.yml         # 1 个 workflow
└── deploy.yml        # depends_on: [build, test] → 等所有 test 矩阵轴完成
```

执行顺序：
```
build (1个) ──┐
             ├──► deploy (1个)
test (4个)  ──┘
```

即：deploy workflow 会等待 build 完成 + 所有 4 个 test 矩阵轴都完成。

---

---

## 十二、执行端 Agent 的资源调度与 Step 隔离

### 12.1 整体调度架构

Woodpecker 采用 **Server 调度 + Agent 拉取** 的混合模型，而非传统的中央推模式：

```
┌─────────────────────────────────────────────────────────────────┐
│  Server 端                                                      │
│                                                                 │
│  ┌─────────────────┐    ┌───────────────────┐                  │
│  │  Scheduler      │    │  FIFO Queue       │                  │
│  │  (Queue+PubSub) │    │  (pending/running  │                  │
│  │                 │    │   /waitingOnDeps)  │                  │
│  └────────┬────────┘    └─────────┬─────────┘                  │
│           │                       │                            │
│           └───────────┬───────────┘                            │
│                       │  assignToWorker()                       │
│                       ▼                                        │
│              RPC: Next(filter)  ←── Agent 长轮询拉取任务        │
└───────────────────────┬─────────────────────────────────────────┘
                        │ gRPC
    ┌───────────────────┼─────────────────────┐
    │                   │                     │
┌───▼────┐        ┌─────▼─────┐        ┌─────▼─────┐
│ Agent  │        │   Agent   │        │   Agent   │
│ (Docker)│        │(Kubernetes)│       │  (Local)  │
└────────┘        └───────────┘        └───────────┘
```

核心组件：
- **Queue（FIFO）**：任务排队、依赖管理、租约超时
- **Scheduler**：Queue + PubSub 的组合接口
- **Agent Runner**：拉取任务、启动 Runtime、上报状态

### 12.2 服务端队列 — FIFO Queue 的内部机制

**代码位置**：`server/queue/fifo.go`

#### 12.2.1 队列的三个状态列表

```go
type fifo struct {
    pending       *list.List          // 等待调度的任务（可立即执行）
    waitingOnDeps *list.List          // 等待依赖完成的任务
    running       map[string]*entry   // 正在执行的任务（taskID → entry）
    workers       map[*worker]struct{} // 空闲的 Agent worker
}
```

每个任务在生命周期中会在这三个列表间流转：
```
push → pending → 依赖满足？
                ├── 是 → assignToWorker() → running → Done() → 出队
                └── 否 → waitingOnDeps → 依赖完成后重新回到 pending
```

#### 12.2.2 调度主循环 — process()

**代码位置**：`server/queue/fifo.go:257-287`

每 100ms 执行一次调度：

```go
func (q *fifo) process() {
    for {
        select {
        case <-time.After(processTimeInterval): // 100ms
        case <-q.ctx.Done():
            return
        }
        q.Lock()
        if q.paused { ... continue }

        q.resubmitExpiredPipelines()   // 1. 超时任务重新入队
        q.filterWaiting()              // 2. 重新评估 waitingOnDeps
        for pending, worker := q.assignToWorker(); ... { // 3. 任务-工人工人匹配
            // 分配给 worker，移入 running
        }
        q.Unlock()
    }
}
```

#### 12.2.3 任务-工人匹配算法 — assignToWorker()

**代码位置**：`server/queue/fifo.go:314-336`

```go
func (q *fifo) assignToWorker() (*list.Element, *worker) {
    for element := q.pending.Front(); element != nil; element = element.Next() {
        task := element.Value.(*model.Task)
        var bestWorker *worker
        var bestScore int

        for worker := range q.workers {
            matched, score := worker.filter(task)  // 标签匹配打分
            if matched && score > bestScore {
                bestWorker = worker
                bestScore = score
            }
        }
        if bestWorker != nil {
            return element, bestWorker   // 找到第一个可匹配的任务
        }
    }
    return nil, nil
}
```

调度策略：
- **任务优先遍历**：按 pending 队列顺序（FIFO），每个任务找最优 worker
- **评分制匹配**：标签匹配度越高分数越高，优先分配
- **首个命中即返回**：找到第一个可匹配的任务就分配，不做全局最优

#### 12.2.4 依赖管理 — filterWaiting() / depsInQueue()

```go
func (q *fifo) depsInQueue(task *model.Task) bool {
    // 检查 pending 列表中是否有依赖
    for element := q.pending.Front(); element != nil; element = element.Next() {
        possibleDep := element.Value.(*model.Task)
        for _, dep := range task.Dependencies {
            if possibleDep.ID == dep { return true }
        }
    }
    // 检查 running 列表中是否有依赖
    for possibleDepID := range q.running {
        if slices.Contains(task.Dependencies, possibleDepID) { return true }
    }
    return false
}
```

每轮调度中：所有 `waitingOnDeps` 的任务先重新进入 `pending`，再重新评估依赖。未满足的移回 `waitingOnDeps`。

#### 12.2.5 租约超时与自动重试

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            // deadline 过期：重新放入 pending 队首
            taskState.error = ErrTaskExpired
            q.pending.PushFront(taskState.item)
            delete(q.running, taskID)
            close(taskState.done)
        }
    }
}
```

Agent 必须定期（每 `TaskTimeout/3`）调用 `Extend()` 续约，否则任务会被重新调度给其他 Agent。

### 12.3 Agent 标签匹配机制

**代码位置**：`server/rpc/filter.go:27-86`

这是调度的核心过滤逻辑，决定一个任务可以分配给哪些 Agent。

#### 12.3.1 标签来源

任务的 Labels 来自三层合并：
1. **用户 YAML 定义**：`workflow.labels` 字段
2. **Builder 注入的内部标签**：`pipeline.InternalLabelPrefix`（`woodpecker-ci.org/`）命名空间
   - `woodpecker-ci.org/forge`：Forge 类型
   - `woodpecker-ci.org/repo-id`：仓库 ID
   - `woodpecker-ci.org/org-id`：组织 ID
   - 等等
3. **Repo 级标签**：`task.ApplyLabelsFromRepo()` 注入 `repo`、`org`

Agent 启动时声明自己的 Labels（如 `platform=linux/amd64`、`region=us-east`）。

#### 12.3.2 匹配算法

```go
func createFilterFunc(agentFilter rpc.Filter) queue.FilterFn {
    return func(task *model.Task) (bool, int) {
        labels := maps.Clone(task.Labels)

        // 1. 检查 Agent 要求的"!xxx"必需标签（Agent 必须**没有**该标签值）
        if requiredLabelsMissing(labels, agentFilter.Labels) {
            return false, 0
        }

        // 2. 忽略内部标签（不参与调度匹配）
        for k := range labels {
            if strings.HasPrefix(k, pipeline.InternalLabelPrefix) {
                delete(labels, k)
            }
        }

        // 3. 逐标签匹配评分
        score := 0
        for taskLabel, taskLabelValue := range labels {
            if taskLabelValue == "" { continue }

            agentLabelValue, ok := agentFilter.Labels[taskLabel]
            if !ok {
                // 反向匹配：检查是否 "!label" 要求排除
                agentLabelValue, ok = agentFilter.Labels["!"+taskLabel]
                if !ok { return false, 0 }   // 任务要求的标签 Agent 没有 → 不匹配
            }

            switch agentLabelValue {
            case "*":           score++      // 通配符匹配：+1 分
            case taskLabelValue: score += 10 // 精确匹配：+10 分
            default:            return false, 0 // 值不匹配 → 拒绝
            }
        }
        return true, score
    }
}
```

评分规则：
| 匹配情况 | 得分 |
|---------|------|
| Agent 标签值为 `*`（通配符） | +1 |
| Agent 标签值与任务标签值**精确相等** | +10 |
| 任务有标签要求但 Agent 无此标签 | 不匹配（0，拒绝） |
| 标签值不相等 | 不匹配（0，拒绝） |

特殊语法：
- Agent Label `!key=value`：**必需排除**，任务的该标签值不能是 value
- Agent Label `key=*`：任务有该标签即可，值不限

#### 12.3.3 内部标签

前缀 `woodpecker-ci.org/` 的标签会在匹配时被过滤掉，不参与调度决策，仅用于内部路由和追踪。

### 12.4 Agent 端执行 — Runner 工作流

**代码位置**：`agent/runner.go:63-217`

```
Runner.Run()
    │
    ├── 1. r.client.Next(filter)        长轮询 gRPC 拉取下一个 workflow
    │
    ├── 2. r.counter.Add()              注册到 Agent 运行状态计数器
    │
    ├── 3. context.WithTimeout()        workflow 超时控制（默认 1h）
    │
    ├── 4. utils.WithContextSigtermCallback()   SIGTERM 处理
    │
    ├── 5. 启动 goroutine: r.client.Wait()    监听服务端取消信号
    │
    ├── 6. 启动 goroutine: r.client.Extend()  每 TaskTimeout/3 续约一次
    │
    ├── 7. r.client.Init()              上报 workflow 启动状态
    │
    ├── 8. 注入 Agent 信息到所有步骤: CI_MACHINE, CI_SYSTEM_PLATFORM
    │
    ├── 9. pipeline_runtime.New().Run() 启动 Runtime 执行
    │     └── SetupWorkflow → 逐 Stage 执行 → DestroyWorkflow
    │
    └── 10. r.client.Done()             上报 workflow 结束状态
```

关键隔离机制：
- **每个 workflow 独立 context**：取消互不影响
- **每个 workflow 独立 Runtime 实例**：状态不共享
- **taskUUID 隔离**：后端（Docker/K8s）用 UUID 作为资源前缀避免冲突
- **日志/trace 独立 logger 实例**：每个 workflow 独立日志流

### 12.5 Runtime 的 Step 级并行执行

**代码位置**：`pipeline/runtime/workflow.go`

```
Runtime.Run()
    │
    ├── SetupWorkflow()              Backend 创建网络/卷等资源
    │
    └── 对每个 Stage 顺序执行：
          │
          └── runStage(steps)         errgroup.Group 并行启动所有 step
                │
                ├── executeStep(step)
                │     ├── shouldSkipStep()    OnSuccess/OnFailure 判断
                │     ├── traceStep(start)    上报步骤开始
                │     ├── setStepEnv()        注入运行时变量
                │     └── runBlockingStep() / runDetachedStep()
                │           ├── StartStep()    启动容器/进程
                │           ├── TailStep()     日志流式传输
                │           ├── WaitStep()     阻塞等待完成
                │           └── DestroyStep()  清理资源
                │
                └── g.Wait()              等待该 stage 所有 step 完成
```

隔离细节：
- **Stage 串行，Step 并行**：Stage 间严格顺序，Stage 内步骤完全并行
- **Blocking vs Detached**：普通步骤阻塞等待，Service/Detached 步骤后台运行
- **每个 Step 独立 UUID**：资源名带 UUID 前缀，避免命名冲突
- **`Protected[error]`**：线程安全的 workflow 级错误状态，任意 step 失败即记录

### 12.6 各 Backend 的隔离实现

| Backend | 工作区隔离 | 网络隔离 | 资源隔离 |
|---------|----------|---------|---------|
| **Docker** | 独立 Docker Volume（`<prefix>_default`） | 独立 Docker Network（`<prefix>_default`） | 容器级 cgroup 隔离 |
| **Kubernetes** | PVC 或 EmptyDir Volume | Pod 内共享网络，taskUUID 作为 Pod 名前缀 | Pod/容器级隔离 |
| **Local** | 独立临时目录（`<prefix>`） | 无隔离，共享主机网络 | 进程级隔离（无沙箱） |
| **Dummy** | 无（测试用） | 无 | 无 |

---

## 十三、步骤失败重试与人工 Retry 链路

### 13.1 重试机制分层

Woodpecker 的重试分为三个层级：

```
┌───────────────────────────────────────────────────────┐
│  层 1：Agent 租约超时自动重试（Queue 层）                │
│  Agent 崩溃/失联 → 任务自动重新入队调度                 │
├───────────────────────────────────────────────────────┤
│  层 2：Pipeline 级人工 Retry（用户发起）                 │
│  UI/API 点击 "Restart" → 创建全新 Pipeline             │
├───────────────────────────────────────────────────────┤
│  层 3：同一次 Pipeline 内的步骤级失败处理                │
│  failure: ignore → 继续；failure: fail → 中止（无自动重试）│
└───────────────────────────────────────────────────────┘
```

> **注意**：Woodpecker **没有单个步骤的自动重试**（如 `retry: 3`）。失败就是失败，除非手动重启整个 Pipeline。

### 13.2 层 1：Agent 租约超时自动重试

**代码位置**：`server/queue/fifo.go:338-348` + `agent/runner.go:133-148`

#### 13.2.1 工作机制

```
Agent 端                                  Server 端
   │                                         │
   ├── StartStep()                           │
   │                                         │
   ├── 每 TaskTimeout/3 调用 Extend() ──────►│
   │     (TaskTimeout 默认 15 分钟)            │  重置 deadline
   │                                         │
   │  Agent 崩溃/网络中断 → Extend() 停止    │
   │                                         │  deadline 过期
   │                                         │  resubmitExpiredPipelines()
   │                                         │  task 重新进入 pending
   │                                         │
   │  Agent 恢复后重新 Next() ◄──────────────┤  分配给任意可用 Agent
   │                                         │
```

#### 13.2.2 关键参数

| 参数 | 位置 | 默认值 | 说明 |
|------|------|--------|------|
| `TaskTimeout` | `shared/constant` | 15 分钟 | 任务租约有效期 |
| 续约间隔 | `agent/runner.go:141` | TaskTimeout / 3 = 5 分钟 | Agent 续约频率 |
| 调度检查间隔 | `queue/fifo.go:59` | 100ms | 过期检测频率 |

#### 13.2.3 幂等性保证

超时重试可能导致同一 workflow 在多个 Agent 上同时执行（僵尸任务）。Woodpecker 通过以下方式缓解：
1. **DB 状态乐观更新**：只有第一个上报 Done 的 Agent 能更新 workflow 状态
2. **taskUUID 资源隔离**：即使重复执行，资源名带 UUID 不会冲突
3. **服务端取消监听**：新 Agent 执行后，旧 Agent 的 Wait 会收到 cancel 信号

### 13.3 层 2：Pipeline 级人工 Retry — Restart

**代码位置**：`server/pipeline/restart.go:32-119`

#### 13.3.1 Restart 完整链路

```
用户点击 "Restart" / API POST /repos/:id/pipelines/:number
        │
        ▼
server/pipeline.Restart()
        │
        ├── 1. 检查：StatusBlocked 不能 Restart
        │
        ├── 2. store.ConfigsForPipeline()   从 DB 读取旧 Pipeline 的 YAML 配置
        │
        ├── 3. configService.Fetch(restart=true)
        │     │  注意：restart=true 时 Forge fetcher 会直接返回缓存的 oldConfigData
        │     └── 但 HTTP 扩展服务仍会被调用（可能重新生成配置）
        │
        ├── 4. createNewOutOfOld()          复制旧 Pipeline，重置状态
        │     ├── ID/Number = 0              ← 新 ID
        │     ├── Status = Pending
        │     ├── Started/Finished = 0
        │     ├── Errors = nil
        │     ├── Parent = old.Number
        │     └── RerunCount++
        │
        ├── 5. store.CreatePipeline()        持久化新 Pipeline
        │
        ├── 6. linkPipelineConfigs()         关联相同的 Config 记录（复用）
        │
        ├── 7. createPipelineItems()         重新解析 YAML → 编译 → 生成 Workflows/Steps
        │     │                                注意：每次 Restart 都完整重新编译
        │     └── handleParseErrors()         编译错误直接中止
        │
        ├── 8. publishPipeline()             发布 PubSub 事件 + 更新 Forge 状态
        │
        └── 9. start()                       入队调度
              ├── cancelPreviousPipelines()   可选：取消同分支旧 Pipeline
              └── queuePipeline()             Task 推入 Queue
```

#### 13.3.2 Restart 的数据一致性

| 数据项 | 是否复用 | 说明 |
|--------|---------|------|
| YAML 配置原文 | ✅ 复用 | 直接从 DB 读取，Forge 不重新拉取 |
| Config 数据库记录 | ✅ 复用 | `linkPipelineConfigs()` 复用相同 ID |
| Compile 产物（Workflow/Step） | ❌ 重新编译 | 每次 Restart 都重新 `createPipelineItems()` |
| Pipeline ID/Number | ❌ 全新 | 自增生成新 Number |
| Matrix 展开结果 | ❌ 重新计算 | 基于相同 YAML，结果通常相同但非缓存 |
| 环境变量（如时间戳） | ❌ 重新生成 | `CI_PIPELINE_CREATED` 等取当前时间 |

#### 13.3.3 被阻塞 Pipeline 的解锁 — Approve

**代码位置**：`server/pipeline/approve.go:31-88`

需要人工审批的 Pipeline（`require-approval` 启用）进入 `StatusBlocked` 状态：

```go
// Approve 流程
currentPipeline.Status = model.StatusPending   // 先解锁
createPipelineItems(...)                       // 重新编译+生成 Workflow
UpdateToStatusPending()                        // 标记为 Pending
start()                                        // 入队执行
```

`Approve` 本质上是一次"受限的 Restart"：只解除 Blocked 状态，不创建新 Pipeline 记录。

### 13.4 层 3：步骤级失败策略 — failure 字段

**代码位置**：`pipeline/frontend/yaml/compiler/convert.go:154-162` + `pipeline/runtime/step.go:216-218`

```yaml
steps:
  flaky-test:
    image: golang
    commands: [go test ./flaky/...]
    failure: ignore    # fail（默认）| ignore
```

编译阶段：
```go
failure := container.Failure
if container.Failure == "" {
    failure = string(metadata.FailureFail)   // 默认 fail
}
if c.forceIgnoreServiceFailure && detached {
    failure = string(metadata.FailureIgnore) // 兼容模式：Service 强制 ignore
}
```

运行阶段（`runBlockingStep`）：
```go
err = r.traceStep(processState, err, step)
if err != nil && metadata.Failure(step.Failure) == metadata.FailureIgnore {
    return nil   // failure: ignore → 吞掉错误，不影响后续步骤
}
return err       // failure: fail → 记录错误，后续 OnSuccess 步骤被跳过
```

### 13.5 OnSuccess / OnFailure 步骤路由

**代码位置**：`pipeline/runtime/step.go:64-85`

每个步骤在执行前由 `shouldSkipStep()` 判断是否跳过：

```go
func (r *Runtime) shouldSkipStep(step *backend_types.Step) bool {
    currentErr := r.err.Get()

    // 当前有错误，但步骤没有设置 OnFailure → 跳过
    if currentErr != nil && !step.OnFailure {
        return true
    }
    // 当前无错误，但步骤没有设置 OnSuccess → 跳过
    if currentErr == nil && !step.OnSuccess {
        return true
    }
    return false
}
```

编译阶段如何设置这两个标志（`IncludesStatusSuccess` / `IncludesStatusFailure`）已在第二章 2.6.6 节说明。

### 13.6 Cancel 链路 — 主动中止

**代码位置**：`server/pipeline/cancel.go:32-98`

用户/系统发起 Cancel 时：

```
Cancel(pipeline)
    │
    ├── 1. 收集所有 Running/Pending 的 Workflow
    │
    ├── 2. Scheduler.ErrorAtOnce(workflowIDs, ErrCancel)
    │     │  Queue 层：
    │     ├── Pending 任务：直接从 pending/waitingOnDeps 移除
    │     └── Running 任务：设置 ErrCancel，关闭 done channel
    │
    ├── 3. DB 更新：Pending Workflow/Step → Skipped
    │
    └── 4. Pipeline Status 更新：
          ├── 全是 Pending → Canceled
          └── 有 Running → Killed
```

Agent 侧的取消传播：
```
Runner.Go r.client.Wait() 收到 canceled=true
    │
    └── cancelWorkflowCtx(ErrCancel)
          │
          ├── Runtime ctx 被取消
          │     └── runStage() 检测到 ctx.Done() → 返回 ErrCancel
          │
          └── executeStep() 中 StartStep/WaitStep 检测 ctx 取消
                └── completeStep() 标记 ErrCancel
```

### 13.7 取消前 Pipeline 的自动中止 — cancelPreviousPipelines

**代码位置**：`server/pipeline/cancel.go:100-159`

新 Pipeline 启动时可自动取消同上下文的旧 Pipeline：

```go
func cancelPreviousPipelines(...) {
    eventIncluded := slices.Contains(repo.CancelPreviousPipelineEvents, pipeline.Event)
    if !eventIncluded { return }

    pipelineNeedsCancel := func(active *model.Pipeline) bool {
        if active.Event != pipeline.Event { return false }
        switch pipeline.Event {
        case model.EventPush:
            return pipeline.Branch == active.Branch   // 同分支的 push 事件
        default:
            return pipeline.Refspec == active.Refspec  // 同 refspec（同一 PR/Tag）
        }
    }
    // 对所有符合条件的 active pipeline 调用 Cancel()
}
```

仓库配置 `cancel_previous_pipeline_events` 决定哪些事件类型触发自动取消。

---

## 十四、流水线触发条件 When / If 表达式的编译

### 14.1 When 表达式的 YAML 解析模型

**代码位置**：`pipeline/frontend/yaml/constraint/constraint.go:36-57`

```go
type When struct {
    Constraints []Constraint   // 多个约束是 OR 关系
}

type Constraint struct {
    Ref      List                          // refs/heads/main, refs/tags/v*
    Repo     List                          // owner/repo
    Instance List                          // woodpecker.example.com
    Platform List                          // linux/amd64
    Branch   List                          // main, develop, feature/*
    Cron     List                          // nightly, weekly
    Status   yaml_base_types.StringOrSlice // success, failure
    Matrix   Map                           // GO_VERSION=1.21, DATABASE=mysql
    Local    optional.Option[bool]         // true/false
    Path     Path                          // **/*.go, src/**
    Evaluate string                        // 自定义 expr 表达式
    Event    yaml_base_types.StringOrSlice // push, pull_request, tag...
}
```

支持两种 YAML 写法：

```yaml
# 单个约束（MappingNode）
when:
  branch: main
  event: push

# 多个约束（SequenceNode，OR 关系）
when:
  - branch: main
    event: push
  - branch: develop
    event: [push, pull_request]
```

自定义 `UnmarshalYAML` 自动归一化为 `[]Constraint` 结构。

### 14.2 When 的匹配模型 — OR 外包 AND 内

**代码位置**：`pipeline/frontend/yaml/constraint/constraint.go:64-81`

```go
func (when *When) Match(metadata metadata.Metadata, global bool, env map[string]string) (bool, error) {
    for _, c := range when.Constraints {
        match, err := c.Match(metadata, global, env)
        if err != nil { return false, err }
        if match { return true, nil }   // 任一 Constraint 满足 → 整体 true
    }

    if when.IsEmpty() {
        // 空 When 用默认空 Constraint 匹配
        return (&Constraint{}).Match(metadata, global, env)
    }
    return false
}
```

逻辑总结：
- **多个 Constraint 之间：OR**（任一满足即执行）
- **单个 Constraint 内的各字段：AND**（所有条件都满足才算该 Constraint 通过）
- **字段内部：List 是 Include/Exclude（见 14.3）**

### 14.3 List 类型 — Include/Exclude 双模式

**代码位置**：`pipeline/frontend/yaml/constraint/list.go`

`List` 是多数字段（Branch、Ref、Repo、Platform、Cron、Instance、Event）的底层类型：

```go
type List struct {
    Include []string
    Exclude []string
}
```

支持两种 YAML 语法：

```yaml
# 简写：直接写字符串或列表 → 归入 Include
branch: main
branch: [main, develop]

# 完整写法：显式声明 Include/Exclude
branch:
  include: [main, develop]
  exclude: [feature/*]
```

#### 14.3.1 匹配算法

```go
func (c *List) Match(v string) bool {
    if c.Excludes(v) { return false }   // 先判断排除：命中即拒绝
    if c.Includes(v) { return true }    // 再判断包含：命中即通过
    if len(c.Include) == 0 { return true }  // 无 Include → 默认通过
    return false
}
```

优先级：**Exclude > Include > 默认通过**。

#### 14.3.2 Glob 模式匹配

所有 `Include/Exclude` 字符串都通过 `github.com/bmatcuk/doublestar/v4` 进行 glob 匹配，支持：
- `*` 匹配单级路径
- `**` 匹配多级路径
- `?` 匹配单个字符
- `[abc]` 字符类
- `{a,b}` 多选项

示例：
```yaml
branch:
  include:
    - main
    - feature/*      # 匹配 feature/login, feature/cart，但不匹配 feature/user/login
    - release/**     # 匹配 release/1.0, release/1.0/rc1
  exclude:
    - feature/deprecated-*
```

### 14.4 Path 类型 — 变更文件过滤

**代码位置**：`pipeline/frontend/yaml/constraint/path.go`

```go
type Path struct {
    Include       []string
    Exclude       []string
    IgnoreMessage string                // 提交消息包含此字符串则跳过文件检查
    OnEmpty       optional.Option[bool] // 空变更列表（无文件变更）时的返回值，默认 true
}
```

YAML 写法：

```yaml
# 简写
path: [**/*.go, **/*.mod]

# 完整写法
path:
  include: [src/**/*.go, go.mod]
  exclude: [docs/**, README.md]
  ignore_message: "[skip ci]"
  on_empty: true
```

#### 14.4.1 匹配算法

```go
func (c *Path) Match(changedFiles []string, commitMessage string) bool {
    // 1. 提交消息豁免检查
    if len(c.IgnoreMessage) > 0 && strings.Contains(strings.ToLower(commitMessage), strings.ToLower(c.IgnoreMessage)) {
        return true
    }
    // 2. 空变更（空 commit 或无文件变更的事件）
    if len(v) == 0 { return c.OnEmpty.ValueOrDefault(true) }
    // 3. Exclude 优先：所有文件都被 Exclude → 拒绝
    if len(c.Exclude) > 0 && c.Excludes(changedFiles) { return false }
    // 4. Include：任一文件匹配 Include → 通过
    if len(c.Include) > 0 && !c.Includes(changedFiles) { return false }
    return true
}
```

注意：Path 过滤**仅对 push 和 pull_request 事件生效**（`constraint.go:184-186`）。

### 14.5 Map 类型 — Matrix 过滤

**代码位置**：`pipeline/frontend/yaml/constraint/map.go`

```go
type Map struct {
    Include map[string]string
    Exclude map[string]string
}
```

YAML 写法：

```yaml
steps:
  build-linux:
    when:
      matrix:
        OS: linux     # 仅在 matrix.OS=linux 时执行
    ...

  build-non-windows:
    when:
      matrix:
        exclude:
          OS: windows   # matrix.OS=windows 时跳过
    ...
```

匹配逻辑：Exclude 全匹配则拒绝，Include 全匹配才通过。

Matrix 约束仅在 **Step 级** 生效（`global=false` 时），Workflow 级 Match 会跳过 Matrix 检查。

### 14.6 Evaluate — 自定义 Expr 表达式

**代码位置**：`pipeline/frontend/yaml/constraint/constraint.go:196-215`

这是最灵活的条件表达方式，使用 `github.com/expr-lang/expr` 引擎。

```yaml
when:
  evaluate: 'CI_PIPELINE_NUMBER > 100 && CI_COMMIT_BRANCH == "main"'
```

#### 14.6.1 编译与执行

```go
if c.Evaluate != "" {
    if env == nil {
        env = m.Environ()
    } else {
        maps.Copy(env, m.Environ())   // 合并所有 CI_* 环境变量
    }
    // 编译表达式（expr AST）
    out, err := expr.Compile(
        c.Evaluate,
        expr.Env(env),                     // 注入所有环境变量
        expr.AllowUndefinedVariables(),     // 未定义变量不报错（视为零值）
        expr.AsBool(),                      // 强制返回 bool
    )
    // 执行表达式
    result, err := expr.Run(out, env)
    bResult := result.(bool)
    match = match && bResult   // 与其他约束是 AND 关系
}
```

#### 14.6.2 可用变量

表达式可访问全部 CI 环境变量：
- `CI_PIPELINE_NUMBER`、`CI_PIPELINE_EVENT`、`CI_COMMIT_BRANCH`...
- 所有 Matrix 变量（`GO_VERSION`、`DATABASE`...）
- 用户自定义 environment 变量

#### 14.6.3 Expr 引擎特性

`expr-lang/expr` 支持：
- 算术：`+ - * / %`
- 比较：`== != > < >= <=`
- 逻辑：`&& || ! not and or`
- 字符串：`contains()`, `startsWith()`, `endsWith()`, `matches()`（正则）
- 集合：`in` 操作符，`len()` 函数
- 三元：`condition ? a : b`

示例：
```yaml
evaluate: 'CI_COMMIT_BRANCH in ["main", "develop"] && len(CI_PIPELINE_FILES) > 0'
evaluate: 'CI_COMMIT_MESSAGE matches "^feat:.*" && CI_PIPELINE_EVENT == "push"'
evaluate: 'startsWith(CI_COMMIT_TAG, "v") || CI_COMMIT_BRANCH == "release"'
```

### 16.7 Status 特殊字段 — OnSuccess / OnFailure 来源

`Constraint.Status` 不参与常规的 Match 流程，而是被编译器单独提取：

```go
// compiler/convert.go:150-152
onSuccess := container.When.IncludesStatusSuccess(c.metadata, false, c.env)
onFailure := container.When.IncludesStatusFailure(c.metadata, false, c.env)
```

逻辑差异：
- `IncludesStatusSuccess`：无 status 约束时**默认返回 true**（步骤默认在成功时执行）
- `IncludesStatusFailure`：无 status 约束时**默认返回 false**（默认不在失败时执行）

### 14.8 When 表达式的编译位置

When 条件在编译流水线的**三个层级**分别被评估：

| 层级 | 位置 | 作用 |
|------|------|------|
| **Workflow 级** | `builder/builder.go:131` + `compiler/compiler.go` | 整个 workflow 是否应该执行 |
| **Step 级** | `compiler/compiler.go:231,238` | 单个步骤是否应该被编译进 Stage |
| **运行时** | `runtime/step.go:64-85` `shouldSkipStep()` | 基于前序步骤成败，决定 OnSuccess/OnFailure |

注意：
- Workflow/Step 级的 When（除了 status）在**编译期**一次性评估，基于触发时的元数据
- 只有 `status` 条件延迟到**运行时**动态判断（因为依赖前序步骤执行结果）
- `Evaluate` 表达式也在编译期求值（除非引用了运行时变量，但运行时变量 CI 元数据里不存在）

### 14.9 When 条件评估总流程

```
When.Match(metadata, global, env)
     │
     ├── [对每个 Constraint 逐一检查，任一通过即返回 true]
     │
     └── Constraint.Match()
           │
           ├── global==false? → Matrix.Match(workflow matrix)
           │
           ├── Platform.Match(sys.platform)
           ├── Event.Match(curr.event)
           ├── Repo.Match(owner/repo)
           ├── Ref.Match(commit.ref)
           ├── Instance.Match(sys.host)
           │
           ├── event in [push, pull_request]?
           │     └── Path.Match(changedFiles, commit.message)
           │
           ├── event != tag?
           │     └── Branch.Match(commit.branch)
           │
           ├── event == cron?
           │     └── Cron.Match(curr.cron)
           │
           └── Evaluate != ""?
                 ├── env ← merge(env, metadata.Environ())
                 ├── expr.Compile()
                 └── expr.Run() → bool
```

---

## 十五、附录：核心文件全清单

### 15.1 编译与前端模块

| 文件路径 | 职责 | 相关章节 |
|---------|------|---------|
| `pipeline/frontend/yaml/parse.go` | YAML 文本 → Workflow 结构体 | 第 2、3 章 |
| `pipeline/frontend/yaml/types/workflow.go` | Workflow YAML 类型定义 | 第 3、8 章 |
| `pipeline/frontend/yaml/types/container.go` | Container YAML 类型定义、IsPlugin() | 第 3、10 章 |
| `pipeline/frontend/yaml/types/container_list.go` | 步骤列表双语法兼容（Map/Sequence） | 第 8 章 |
| `pipeline/frontend/yaml/types/volume.go` | Volume 挂载类型定义 | 第 10 章 |
| `pipeline/frontend/yaml/compiler/compiler.go` | Compiler 核心、阶段注入、Secret 权限校验 | 第 2、4、9 章 |
| `pipeline/frontend/yaml/compiler/convert.go` | Container → Step 转换 | 第 2、4 章 |
| `pipeline/frontend/yaml/compiler/dag.go` | DAG 拓扑排序、步骤级并行 | 第 4、11 章 |
| `pipeline/frontend/yaml/compiler/option.go` | Compiler Functional Options | 第 4 章 |
| `pipeline/frontend/yaml/compiler/errors.go` | 编译器 4 种细粒度错误类型 | 第 6 章 |
| `pipeline/frontend/yaml/compiler/settings/params.go` | Settings→Env 转换、from_secret 递归注入 | 第 4、9 章 |
| `pipeline/frontend/yaml/linter/linter.go` | 静态语法与策略校验 | 第 3、6 章 |
| `pipeline/frontend/yaml/linter/error.go` | Linter 错误创建（File + Field 定位） | 第 6 章 |
| `pipeline/frontend/yaml/linter/option.go` | Linter 配置选项 | 第 6 章 |
| `pipeline/frontend/yaml/linter/schema/schema.go` | JSON Schema 校验实现 | 第 6 章 |
| `pipeline/frontend/yaml/linter/schema/schema.json` | DSL Schema 权威定义 | 第 6、8 章 |
| `pipeline/frontend/yaml/matrix/matrix.go` | Matrix 解析、笛卡尔积计算、限制保护 | 第 2、11 章 |
| `pipeline/frontend/yaml/constraint/constraint.go` | When/Constraint 核心匹配逻辑、Evaluate 表达式 | 第 8、14 章 |
| `pipeline/frontend/yaml/constraint/list.go` | List 类型：Include/Exclude + Glob 匹配 | 第 14 章 |
| `pipeline/frontend/yaml/constraint/path.go` | Path 类型：变更文件过滤 | 第 14 章 |
| `pipeline/frontend/yaml/constraint/map.go` | Map 类型：Matrix 维度过滤 | 第 14 章 |
| `pipeline/frontend/yaml/constraint/depends_on.go` | DependsOn 依赖类型 | 第 4、11 章 |
| `pipeline/frontend/builder/builder.go` | PipelineBuilder、多文件/Matrix 编排主流程 | 第 2、11 章 |
| `pipeline/frontend/builder/types.go` | Item / Workflow / YamlFile 类型定义 | 第 1、11 章 |
| `pipeline/frontend/builder/utils.go` | 依赖过滤、路径清洗工具 | 第 2、11 章 |
| `pipeline/frontend/metadata/substitution.go` | 环境变量替换（EnvVarSubst） | 第 3、9 章 |
| `pipeline/frontend/metadata/environment.go` | CI 元数据环境变量生成 | 第 9 章 |
| `pipeline/frontend/metadata/drone_compatibility.go` | Drone CI 环境变量兼容层 | 第 8 章 |

### 15.2 后端与运行时模块

| 文件路径 | 职责 | 相关章节 |
|---------|------|---------|
| `pipeline/backend/types/config.go` | 后端 Config IR | 第 1、10 章 |
| `pipeline/backend/types/stage.go` | 后端 Stage IR | 第 1 章 |
| `pipeline/backend/types/step.go` | 后端 Step IR、StepType 枚举、SecretMapping | 第 1、9、10 章 |
| `pipeline/backend/types/backend.go` | Backend 接口定义 | 第 10 章 |
| `pipeline/errors/pipeline.go` | PipelineError 类型体系与 5 种错误分类 | 第 6 章 |
| `pipeline/errors/linter.go` | 错误聚合、阻断判断、结构化 Data 提取 | 第 6 章 |
| `pipeline/errors/runtime.go` | 运行时错误类型（ExitError、OomError） | 第 6 章 |
| `pipeline/runtime/workflow.go` | Runtime：Stage 串行、Step 并行执行 | 第 12 章 |
| `pipeline/runtime/step.go` | 步骤执行、OnSuccess/OnFailure 跳过、运行时环境变量 | 第 9、13 章 |
| `pipeline/runtime/runtime.go` | Runtime 工作流执行器 | 第 9 章 |

### 15.3 服务端调度与 Agent 模块

| 文件路径 | 职责 | 相关章节 |
|---------|------|---------|
| `server/queue/fifo.go` | FIFO 队列实现：任务状态、依赖管理、调度匹配 | 第 12 章 |
| `server/queue/queue.go` | Queue 接口定义 | 第 12 章 |
| `server/scheduler/scheduler.go` | Scheduler（Queue + PubSub 组合） | 第 12 章 |
| `server/rpc/filter.go` | Agent-Task 标签匹配算法与评分 | 第 12 章 |
| `server/rpc/server.go` | RPC 服务端：Next/Wait/Init/Extend/Done | 第 12 章 |
| `server/model/task.go` | Task 模型：依赖、运行状态、ShouldRun 判断 | 第 12、13 章 |
| `agent/runner.go` | Agent Runner：拉取任务、续约、Runtime 启动 | 第 12、13 章 |

### 15.4 服务端 Pipeline 生命周期

| 文件路径 | 职责 | 相关章节 |
|---------|------|---------|
| `server/pipeline/items.go` | 服务端入口：组装 PipelineBuilder | 第 2 章 |
| `server/pipeline/config.go` | 配置持久化入口 | 第 7 章 |
| `server/pipeline/create.go` | Pipeline 创建与错误处理主流程 | 第 6、7 章 |
| `server/pipeline/pipeline_status.go` | 错误状态更新（UpdateToStatusError） | 第 6 章 |
| `server/pipeline/restart.go` | Pipeline Restart：人工 Retry 链路 | 第 13 章 |
| `server/pipeline/cancel.go` | Pipeline Cancel：主动中止与自动取消前序 | 第 13 章 |
| `server/pipeline/approve.go` | Pipeline Approve：阻塞流水线解锁 | 第 13 章 |
| `server/pipeline/start.go` | Pipeline 启动：入队、取消前序、发布 | 第 13 章 |
| `server/services/config/forge.go` | Forge 配置获取与重启复用 | 第 7 章 |
| `server/services/config/http.go` | HTTP 配置扩展与 204 回退机制 | 第 7 章 |
| `server/services/config/combined.go` | 配置服务链式组合 | 第 7 章 |
| `server/store/datastore/config.go` | 配置 SHA-256 哈希去重存储 | 第 7 章 |

### 15.5 CLI 与共享模块

| 文件路径 | 职责 | 相关章节 |
|---------|------|---------|
| `cli/exec/exec.go` | CLI 本地执行入口 | 第 2 章 |
| `shared/constant/constant.go` | 默认配置路径、克隆插件常量、TaskTimeout | 第 8、12 章 |
