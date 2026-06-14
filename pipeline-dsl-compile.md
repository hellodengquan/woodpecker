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

## 九、补充文件清单

| 文件路径 | 补充内容相关 |
|---------|------------|
| `pipeline/errors/pipeline.go` | PipelineError 类型体系与 5 种错误分类 |
| `pipeline/errors/linter.go` | 错误聚合、阻断判断、结构化 Data 提取 |
| `pipeline/errors/runtime.go` | 运行时错误类型（ExitError、OomError） |
| `pipeline/frontend/yaml/linter/error.go` | Linter 错误创建（File + Field 定位） |
| `pipeline/frontend/yaml/compiler/errors.go` | 编译器 4 种细粒度错误类型 |
| `pipeline/frontend/yaml/linter/schema/schema.go` | JSON Schema 校验实现 |
| `pipeline/frontend/yaml/linter/schema/schema.json` | DSL Schema 权威定义 |
| `server/store/datastore/config.go` | 配置 SHA-256 哈希去重存储 |
| `server/pipeline/config.go` | 配置持久化入口 |
| `server/pipeline/pipeline_status.go` | 错误状态更新（UpdateToStatusError） |
| `server/pipeline/create.go` | Pipeline 创建与错误处理主流程 |
| `server/services/config/forge.go` | Forge 配置获取与重启复用 |
| `server/services/config/http.go` | HTTP 配置扩展与 204 回退机制 |
| `server/services/config/combined.go` | 配置服务链式组合 |
| `pipeline/frontend/metadata/drone_compatibility.go` | Drone CI 环境变量兼容层 |
| `pipeline/frontend/yaml/types/container_list.go` | 步骤列表双语法兼容（Map/Sequence） |
| `pipeline/frontend/yaml/types/workflow.go` | `runs_on` 废弃字段定义 |
| `pipeline/frontend/yaml/constraint/constraint.go` | `when` 条件约束实现 |
| `shared/constant/constant.go` | 默认配置路径、克隆插件常量 |
