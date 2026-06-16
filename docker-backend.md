# Docker 后端：从拉起容器到回收退出码的完整链路

## 全局视图

Docker 后端实现了 `Backend` 接口（`pipeline/backend/types/backend.go:60`），核心生命周期为：

```
Load → SetupWorkflow → [StartStep → TailStep → WaitStep → DestroyStep]×N → DestroyWorkflow
```

运行时 `Runtime`（`pipeline/runtime/runtime.go:33`）按 stage 顺序编排，每个 stage 内的 step 并发执行（`pipeline/runtime/workflow.go:143`）。单个 step 的完整流程在 `pipeline/runtime/step.go` 中，分为 `startStep` → `completeStep` 两个阶段。

---

## 一、镜像选择与拉取

### 1.1 镜像名从哪来

`Step.Image`（`pipeline/backend/types/step.go:23`）在 YAML 前端解析阶段就已写入 `Step` 结构体，到达 Docker 后端时是一个只读的已决值。Docker 后端在 `StartStep`（`pipeline/backend/docker/docker.go:178`）中直接使用：

```go
config := e.toConfig(step, options)   // docker.go:186
// toConfig 内部: config.Image = step.Image   (convert.go:40)
```

镜像名的选取完全由上游前端决定，Docker 后端不参与选择逻辑，只负责「拿到名字后怎么用」。

### 1.2 拉取策略：两级 fallback

`StartStep` 实现了一个两级镜像拉取策略（`docker.go:199-245`）：

**第一级：显式 `step.Pull == true`**

当 YAML 中声明了 `pull: true`，后端主动调用 `ImagePull`：

```go
if step.Pull {
    responseBody, pErr := e.client.ImagePull(ctx, config.Image, pullOpts)
    // ...
    if pErr != nil && step.AuthConfig.Password != "" {
        return pErr  // 有密码时拉取失败直接报错
    }
}
```

注意：即使拉取失败但 `step.AuthConfig.Password` 为空（匿名拉取），也不返回错误——降级到本地镜像继续运行。

**第二级：`ContainerCreate` 返回 `NotFound`**

如果第一步没有拉取（或拉取失败被吞掉），代码继续执行 `ContainerCreate`。当 Docker daemon 报告镜像不存在（`errdefs.IsNotFound`），自动触发拉取并重试创建：

```go
_, err = e.client.ContainerCreate(ctx, ...)
if errdefs.IsNotFound(err) {
    responseBody, pErr := e.client.ImagePull(ctx, config.Image, pullOpts)
    // 拉取后重试 ContainerCreate
    _, err = e.client.ContainerCreate(ctx, ...)
}
```

**鉴权流程**：拉取前通过 `encodeAuthToBase64(step.AuthConfig)`（`convert.go:199`）将 `AuthConfig` 序列化为 base64 JSON 放入 `RegistryAuth` 头。

### 1.3 镜像名 → 容器 Config 的映射

`toConfig`（`convert.go:36`）将 `Step` 转换为 Docker `container.Config`：

| Step 字段 | container.Config 字段 | 说明 |
|---|---|---|
| `step.Image` | `config.Image` | 镜像全名 |
| `step.WorkingDir` | `config.WorkingDir` | 仅 plugin 类型使用原始值 |
| `step.WorkspaceBase` | `config.WorkingDir` | commands 类型覆盖为此值 |
| `step.Commands` | `config.Entrypoint` + `config.Env["CI_SCRIPT"]` | 命令编码为 base64 脚本 |
| `step.Volumes` | `config.Volumes` | 容器内挂载点声明（`toVol`） |
| `step.Environment` | `config.Env` | 环境变量（`toEnv`） |

**命令执行的特殊处理**（`convert.go:54-61`）：

当 `step.Commands` 非空时，调用 `common.GenerateContainerConf`（`pipeline/backend/common/script.go:21`）将命令列表编码为 base64 脚本注入 `CI_SCRIPT` 环境变量，并设置 entrypoint 为：

- Linux: `/bin/sh -c "echo $CI_SCRIPT | base64 -d | /bin/sh -e"`
- Windows: `powershell -noprofile -noninteractive -command "..." `

此时 `WorkingDir` 被覆盖为 `step.WorkspaceBase`（脚本内部会 `cd` 到 `step.WorkingDir`）。

---

## 二、卷挂载

### 2.1 两层来源：workflow 级 + step 级 + 全局默认

卷挂载由三个层次叠加：

**层次 1：Workflow 级共享卷**（`SetupWorkflow`，`docker.go:156`）

每个 workflow 启动时创建一个命名卷：

```go
func (e *docker) SetupWorkflow(ctx context.Context, conf *backend_types.Config, ...) error {
    _, err := e.client.VolumeCreate(ctx, client.VolumeCreateOptions{
        Name:   conf.Volume,    // 如 "wp_default_xxxx"
        Driver: volumeDriver,   // "local"
    })
    // 同时创建 network ...
}
```

`conf.Volume` 由上游前端生成，所有 step 的 workspace 通过此卷共享数据。

**层次 2：Step 级卷声明**（`toHostConfig`，`convert.go:77`）

`step.Volumes` 直接赋给 `hostConfig.Binds`：

```go
if len(step.Volumes) != 0 {
    config.Binds = step.Volumes   // 如 ["wp_default_abc:/src", "/cache:/cache"]
}
```

**层次 3：全局默认卷**（`StartStep`，`docker.go:219`）

管理员通过 `WOODPECKER_BACKEND_DOCKER_VOLUMES` 配置的默认卷，在 `StartStep` 中追加到 `hostConfig.Binds`：

```go
hostConfig.Binds = utils.DeduplicateStrings(append(hostConfig.Binds, e.config.volumes...))
```

`e.config.volumes` 在 `configFromCli`（`config.go:57`）中从逗号分隔的配置字符串解析而来，每个卷定义经过 `splitVolumeParts` 验证。

### 2.2 卷字符串的解析

`splitVolumeParts`（`convert.go:216`）使用正则 `^((?:[\w]\:)?[^\:]*)\:((?:[\w]\:)?[^\:]*)(?:\:([rwom]*))?` 分解为 `[hostPath, containerPath, mode]`，兼容 Windows 盘符路径（如 `C:\src:C:\src:ro`）。

### 2.3 Config.Volumes vs HostConfig.Binds 的分工

Docker API 中两者职责不同：

- `config.Volumes`（`toVol` 函数，`convert.go:141`）：声明容器内需要挂载的目录路径（仅 `map[string]struct{}`），不指定来源。从 `step.Volumes` 解析出容器侧路径。
- `hostConfig.Binds`：实际的绑定挂载映射，格式 `src:dst[:mode]`，同时包含了层次 2 和层次 3。

两者必须对应：`config.Volumes` 中声明的路径需要在 `hostConfig.Binds` 中有对应的绑定，Docker daemon 才能正确挂载。

### 2.4 Tmpfs 与设备

`toHostConfig` 还处理：

- `step.Tmpfs`（`convert.go:123-134`）：解析为 `hostConfig.Tmpfs` map，支持 `path:opts` 格式。
- `step.Devices`（`convert.go:117-119` → `toDev`，`convert.go:174`）：解析为 `container.DeviceMapping`，自动剥离 `:ro`/`:rw` 后缀，默认 `rwm` 权限。

### 2.5 清理：卷的删除

`DestroyWorkflow`（`docker.go:369`）在销毁所有容器后删除卷，使用指数退避重试（最多 3 次），处理「卷仍在使用」的竞态：

```go
_, _ = backoff.Retry(ctx, func() (any, error) {
    _, err = e.client.VolumeRemove(ctx, conf.Volume, ...)
    if err == nil || !isErrVolumeInUse(err) {
        return nil, nil
    }
    return nil, err
}, backoff.WithMaxTries(maxRetry), ...)
```

---

## 三、退出码回收

### 3.1 WaitStep：阻塞等待 + 检查容器状态

`WaitStep`（`docker.go:278`）是退出码回收的入口：

```go
func (e *docker) WaitStep(ctx context.Context, step *backend_types.Step, ...) (*backend_types.State, error) {
    wait := e.client.ContainerWait(ctx, containerName, client.ContainerWaitOptions{})
    select {
    case resp := <-wait.Result:
        if resp.Error != nil {
            return nil, fmt.Errorf("ContainerWait error: %s", resp.Error.Message)
        }
    case err := <-wait.Error:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err()
    }

    info, err := e.client.ContainerInspect(ctx, containerName, ...)
    // ...
    exitCode := info.Container.State.ExitCode
}
```

流程：
1. `ContainerWait` 阻塞直到容器退出（或出错 / 上下文取消）
2. `ContainerInspect` 获取完整容器状态
3. 从 `info.Container.State.ExitCode` 提取退出码

### 3.2 Windows 异常退出码修正

Docker 后端对 Windows 的特殊处理（`docker.go:304-307`）：

```go
if int64(exitCode) == int64(4294967295) {  // uint32 max → int32(-1)
    exitCode = -1
}
```

Windows Docker 在容器异常退出时可能返回 `4294967295`（uint32 溢出），此修正将其规范化为 `-1`。

### 3.3 State 结构体

`WaitStep` 返回 `backend_types.State`（`pipeline/backend/types/state.go:18`）：

```go
type State struct {
    Started   int64  // 启动时间（由 runtime 填充，非后端）
    ExitCode  int    // 容器退出码
    Exited    bool   // 是否已退出
    Skipped   bool   // 是否被跳过
    OOMKilled bool   // 是否因 OOM 被杀
    Error     error  // 容器级错误
}
```

### 3.4 Runtime 层的退出码解读

`completeStep`（`pipeline/runtime/step.go:151`）接收 `State` 后做三层判断：

**OOM 判断**（`step.go:180-184`）：

```go
if waitState.OOMKilled {
    return waitState, &pipeline_errors.OomError{UUID: step.UUID, Code: waitState.ExitCode}
}
```

**非零退出码**（`step.go:186-189`）：

```go
if waitState.ExitCode != 0 {
    return waitState, &pipeline_errors.ExitError{UUID: step.UUID, Code: waitState.ExitCode}
}
```

**上下文取消**（`step.go:156-165`、`step.go:176-178`）：

两次检查 context cancellation——一次在 `WaitStep` 返回后，一次在 `DestroyStep` 后。如果 workflow 上下文被取消，错误被替换为 `pipeline_errors.ErrCancel`，确保取消不会被误判为普通失败。

### 3.5 退出码如何影响后续 step

`executeStep` 返回的 error 会被 `runStage` 的 `errgroup` 收集（`workflow.go:148-150`），设置到 `r.err`（`workflow.go:72`）。后续 step 通过 `shouldSkipStep`（`step.go:65`）检查 `r.err.Get()`：

- 已有错误 + `step.OnFailure == false` → 跳过
- 无错误 + `step.OnSuccess == false` → 跳过

这就是退出码最终影响流水线行为的路径。

---

## 四、完整链路时序图

以一个 `commands` 类型 step 为例，完整调用链如下：

```
Runtime.Run()
  ├─ engine.SetupWorkflow()          ─→ VolumeCreate + NetworkCreate
  │
  ├─ runStage()  (per stage, parallel steps)
  │   └─ executeStep(step)
  │       ├─ shouldSkipStep()        ─→ 检查 r.err 判断是否跳过
  │       ├─ traceStep(started)      ─→ 通知 tracer step 开始
  │       ├─ startStep(step)
  │       │   ├─ engine.StartStep()  ─→ toConfig + toHostConfig
  │       │   │   ├─ [ImagePull if step.Pull]
  │       │   │   ├─ ContainerCreate ─→ [ImagePull if NotFound, retry]
  │       │   │   ├─ NetworkConnect
  │       │   │   └─ ContainerStart
  │       │   └─ engine.TailStep()   ─→ ContainerLogs (goroutine 流式输出)
  │       │
  │       └─ completeStep(step)
  │           ├─ waitForLogs()       ─→ 等待日志流排空
  │           ├─ engine.WaitStep()   ─→ ContainerWait + ContainerInspect
  │           │                       ─→ 提取 ExitCode / OOMKilled
  │           ├─ engine.DestroyStep() ─→ ContainerStop + ContainerKill + ContainerRemove
  │           ├─ [Windows exitCode 修正]
  │           └─ [OOM / ExitCode!=0 / Cancel 错误映射]
  │
  └─ engine.DestroyWorkflow()        ─→ DestroyStep×N + VolumeRemove(重试) + NetworkRemove
```

---

## 五、关键设计要点

### 5.1 镜像拉取的宽容策略

两级 fallback 设计让匿名拉取的镜像即使网络抖动也不会立即失败——只要本地有缓存就能继续。但配置了密码的私有仓库拉取失败会直接报错，避免静默降级到错误镜像。

### 5.2 卷挂载的叠加语义

`step.Volumes` + `e.config.volumes` 的 append + 去重设计意味着：
- 管理员可以通过全局配置为所有容器注入额外卷（如证书目录）
- step 级声明优先（去重时保留先出现的，即 step 自身的）
- 但顺序是先 step 后全局，所以实际上 step 的声明不会被覆盖

### 5.3 退出码的语义丰富化

Docker 后端只返回原始 `ExitCode` + `OOMKilled`，但 Runtime 层将其丰富为三种语义：
- `OomError`：容器因内存不足被杀
- `ExitError`：命令执行失败
- `ErrCancel`：workflow 被外部取消

这种分层让上层逻辑（如 UI 展示、重试决策）能区分「自己的锅」vs「环境的锅」vs「用户主动取消」。

### 5.4 容器命名的隔离性

容器名 `wp_${step.UUID}`（`convert.go:72`）确保全局唯一，使得同一 Docker daemon 上可以安全并发执行多个 workflow。
