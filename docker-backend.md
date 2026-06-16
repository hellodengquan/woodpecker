# Docker 后端：从拉起容器到回收退出码的完整链路

本文档按 **代码执行顺序** 分析 Docker 后端如何完成「拉起容器 → 执行命令 → 回收退出码」的完整流程，重点关注 **镜像选择**、**卷挂载** 和 **退出码回收** 三个环节的衔接。

---

## 一、整体架构与调用链入口

### 1.1 接口契约

Docker 后端实现 `Backend` 接口（`pipeline/backend/types/backend.go:60`），核心生命周期方法：

```go
type Backend interface {
    Load(ctx) (*BackendInfo, error)              // 初始化后端
    SetupWorkflow(ctx, conf, taskUUID) error     // 创建 workflow 环境（卷+网络）
    StartStep(ctx, step, taskUUID) error         // 启动 step 容器
    TailStep(ctx, step, taskUUID) (io.ReadCloser, error)  // 流式日志
    WaitStep(ctx, step, taskUUID) (*State, error)         // 等待退出
    DestroyStep(ctx, step, taskUUID) error       // 销毁 step 容器
    DestroyWorkflow(ctx, conf, taskUUID) error   // 销毁 workflow 环境
}
```

### 1.2 Runtime 层编排

`Runtime`（`pipeline/runtime/runtime.go:33`）按如下顺序编排：

```
Runtime.Run()
  ├─ SetupWorkflow()          ── 创建卷和网络
  ├─ 循环 stage
  │   └─ runStage()           ── 并发执行 stage 内所有 step
  │       └─ executeStep()
  │           ├─ startStep()  ── StartStep + TailStep
  │           └─ completeStep() ── 等待日志 + WaitStep + DestroyStep
  └─ DestroyWorkflow()        ── 销毁所有容器 + 删除卷 + 删除网络
```

单个 step 的执行在 `pipeline/runtime/step.go` 中分为两个阶段：
- `startStep`（`step.go:122`）：启动容器 + 日志流
- `completeStep`（`step.go:151`）：等待退出 + 回收结果 + 销毁容器

---

## 二、代码执行顺序详解

### 2.1 阶段 1：SetupWorkflow —— 创建共享卷和网络

**入口**：`pipeline/runtime/workflow.go:59` → `pipeline/backend/docker/docker.go:156`

这是第一个衔接点：**卷的创建**。

```go
func (e *docker) SetupWorkflow(ctx context.Context, conf *backend_types.Config, taskUUID string) error {
    // 创建共享卷，所有 step 的 workspace 通过此卷共享数据
    _, err := e.client.VolumeCreate(ctx, client.VolumeCreateOptions{
        Name:   conf.Volume,    // 如 "wp_default_xxxx"，由前端生成
        Driver: volumeDriver,   // "local"
    })

    // 创建隔离网络
    networkDriver := networkDriverBridge
    if e.info.OSType == "windows" {
        networkDriver = networkDriverNAT
    }
    _, err = e.client.NetworkCreate(ctx, conf.Network, client.NetworkCreateOptions{
        Driver:     networkDriver,
        EnableIPv6: &e.config.enableIPv6,
    })
    return err
}
```

**关键点**：
- `conf.Volume` 是 workflow 级共享卷的名称，由上游前端生成
- 此卷会在后续每个 step 中挂载到 workspace 目录
- 网络确保同一 workflow 的 step 之间可以互相通信

---

### 2.2 阶段 2：executeStep —— 前置检查

**入口**：`pipeline/runtime/step.go:34`

```go
func (r *Runtime) executeStep(runnerCtx context.Context, step *backend_types.Step) error {
    if r.shouldSkipStep(step) { ... }       // 检查是否跳过
    r.traceStep(nil, nil, step)             // 标记 step 开始
    r.setStepEnv(step)                      // 设置环境变量

    if step.Detached {
        return r.runDetachedStep(runnerCtx, step)
    }
    return r.runBlockingStep(runnerCtx, step)
}
```

`shouldSkipStep`（`step.go:65`）是 **退出码影响后续 step** 的衔接点：
- 读取 `r.err.Get()`（之前 step 的错误）
- `已有错误 + step.OnFailure == false` → 跳过
- `无错误 + step.OnSuccess == false` → 跳过

---

### 2.3 阶段 3：startStep —— 镜像选择 + 容器创建 + 卷挂载

**入口**：`pipeline/runtime/step.go:122`

```go
func (r *Runtime) startStep(step *backend_types.Step) (func(), int64, error) {
    if err := r.engine.StartStep(r.ctx, step, r.taskUUID); err != nil {
        return nil, 0, err
    }
    startTime := time.Now().Unix()

    rc, err := r.engine.TailStep(r.ctx, step, r.taskUUID)
    // ... 启动日志流式 goroutine
    return wg.Wait, startTime, nil
}
```

#### 2.3.1 StartStep —— 核心逻辑

**入口**：`pipeline/backend/docker/docker.go:178`

这是 **镜像选择** 和 **卷挂载** 的核心衔接点。

```go
func (e *docker) StartStep(ctx context.Context, step *backend_types.Step, taskUUID string) error {
    options, _ := parseBackendOptions(step)

    // ──────────────────────────────────────────
    // 衔接点 1: 镜像选择 → toConfig
    // ──────────────────────────────────────────
    config := e.toConfig(step, options)                   // convert.go:36
    hostConfig, err := toHostConfig(step, &e.config)      // convert.go:77

    containerName := toContainerName(step)                // "wp_" + step.UUID

    // 鉴权信息
    pullOpts := client.ImagePullOptions{}
    if step.AuthConfig.Username != "" && step.AuthConfig.Password != "" {
        pullOpts.RegistryAuth, _ = encodeAuthToBase64(step.AuthConfig)  // convert.go:199
    }

    // ──────────────────────────────────────────
    // 衔接点 2: 镜像拉取策略（第一级）
    // ──────────────────────────────────────────
    if step.Pull {
        responseBody, pErr := e.client.ImagePull(ctx, config.Image, pullOpts)
        if pErr == nil {
            // 显示拉取进度
            responseBody.Close()
        }
        // 有密码时拉取失败直接报错；匿名拉取失败降级到本地镜像
        if pErr != nil && step.AuthConfig.Password != "" {
            return pErr
        }
    }

    // ──────────────────────────────────────────
    // 衔接点 3: 卷挂载叠加（全局默认卷）
    // ──────────────────────────────────────────
    hostConfig.Binds = utils.DeduplicateStrings(append(hostConfig.Binds, e.config.volumes...))

    // ──────────────────────────────────────────
    // 衔接点 4: 容器创建 + 镜像拉取（第二级）
    // ──────────────────────────────────────────
    _, err = e.client.ContainerCreate(ctx, client.ContainerCreateOptions{
        Config:     config,
        HostConfig: hostConfig,
        Name:       containerName,
    })
    if errdefs.IsNotFound(err) {
        // 镜像不存在，自动拉取后重试
        responseBody, pErr := e.client.ImagePull(ctx, config.Image, pullOpts)
        if pErr != nil {
            return pErr
        }
        responseBody.Close()
        _, err = e.client.ContainerCreate(ctx, client.ContainerCreateOptions{
            Config:     config,
            HostConfig: hostConfig,
            Name:       containerName,
        })
    }
    if err != nil {
        return err
    }

    // ──────────────────────────────────────────
    // 衔接点 5: 网络连接
    // ──────────────────────────────────────────
    if len(step.NetworkMode) == 0 {
        for _, net := range step.Networks {
            _, err = e.client.NetworkConnect(ctx, net.Name, client.NetworkConnectOptions{
                EndpointConfig: &network.EndpointSettings{Aliases: net.Aliases},
                Container: containerName,
            })
        }
        if e.config.network != "" {
            _, err = e.client.NetworkConnect(ctx, e.config.network, client.NetworkConnectOptions{
                Container: containerName,
            })
        }
    }

    // ──────────────────────────────────────────
    // 衔接点 6: 启动容器
    // ──────────────────────────────────────────
    _, err = e.client.ContainerStart(ctx, containerName, client.ContainerStartOptions{})
    return err
}
```

##### 2.3.1.1 toConfig —— 镜像名 + 命令编码

**入口**：`pipeline/backend/docker/convert.go:36`

```go
func (e *docker) toConfig(step *types.Step, options BackendOptions) *container.Config {
    e.windowsPathPatch(step)

    config := &container.Config{
        Image:        step.Image,           // 镜像名直接来自前端
        Labels:       map[string]string{"wp_uuid": step.UUID, "wp_step": step.Name},
        WorkingDir:   step.WorkingDir,
        AttachStdout: true,
        AttachStderr: true,
        Volumes:      toVol(step.Volumes),  // 声明容器内挂载点
        User:         options.User,
    }

    // ──────────────────────────────────────────
    // 命令编码为 base64 脚本注入容器
    // ──────────────────────────────────────────
    if len(step.Commands) > 0 {
        // common/script.go:21 —— 核心命令注入逻辑
        env, entry := common.GenerateContainerConf(step.Commands, e.info.OSType, step.WorkingDir)
        maps.Copy(configEnv, env)
        config.Entrypoint = entry
        config.WorkingDir = step.WorkspaceBase  // 脚本内部会 cd 到 WorkingDir
    }

    return config
}
```

**命令编码细节**（`pipeline/backend/common/script.go:21`）：

```go
func GenerateContainerConf(commands []string, osType, workDir string) (env map[string]string, entry []string) {
    if osType == "windows" {
        env["CI_SCRIPT"] = base64.StdEncoding.EncodeToString([]byte(generateScriptWindows(commands, workDir)))
        entry = []string{"powershell", "-noprofile", "-noninteractive", "-command",
            "[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($Env:CI_SCRIPT)) | iex"}
    } else {
        env["CI_SCRIPT"] = base64.StdEncoding.EncodeToString([]byte(generateScriptPosix(commands, workDir)))
        entry = []string{"/bin/sh", "-c", "echo $CI_SCRIPT | base64 -d | /bin/sh -e"}
    }
    return env, entry
}
```

**镜像选择总结**：
- `step.Image` 在 YAML 前端解析阶段就已写入 `Step` 结构体
- Docker 后端不参与镜像选择，只负责「拿到名字后怎么用」
- 两级拉取策略提供容错：显式 `pull: true` → `ContainerCreate` 失败自动重试

##### 2.3.1.2 toHostConfig —— 卷挂载 + 资源限制

**入口**：`pipeline/backend/docker/convert.go:77`

```go
func toHostConfig(step *types.Step, conf *config) (*container.HostConfig, error) {
    config := &container.HostConfig{
        Resources: container.Resources{
            CPUQuota:   conf.resourceLimit.CPUQuota,
            CPUShares:  conf.resourceLimit.CPUShares,
            CpusetCpus: conf.resourceLimit.CPUSet,
            Memory:     conf.resourceLimit.MemLimit,
            MemorySwap: conf.resourceLimit.MemSwapLimit,
        },
        ShmSize:      conf.resourceLimit.ShmSize,
        LogConfig:    container.LogConfig{Type: "json-file"},
        Privileged:   step.Privileged,
    }

    // ──────────────────────────────────────────
    // Step 级卷挂载
    // ──────────────────────────────────────────
    if len(step.Volumes) != 0 {
        config.Binds = step.Volumes   // 如 ["wp_default_abc:/src", "/cache:/cache"]
    }

    // ──────────────────────────────────────────
    // 设备挂载
    // ──────────────────────────────────────────
    if len(step.Devices) != 0 {
        config.Devices = toDev(step.Devices)  // convert.go:174
    }

    // ──────────────────────────────────────────
    // Tmpfs
    // ──────────────────────────────────────────
    config.Tmpfs = map[string]string{}
    for _, path := range step.Tmpfs {
        if !strings.Contains(path, ":") {
            config.Tmpfs[path] = ""
        } else {
            parts, _ := splitVolumeParts(path)
            config.Tmpfs[parts[0]] = parts[1]
        }
    }

    return config, nil
}
```

**卷挂载总结（三层叠加）**：

| 层次 | 位置 | 说明 |
|------|------|------|
| 1 | `SetupWorkflow` | 创建 workflow 级共享卷 `conf.Volume` |
| 2 | `toHostConfig` | `step.Volumes` → `hostConfig.Binds` |
| 3 | `StartStep:219` | 追加全局默认卷 `e.config.volumes`（来自 `WOODPECKER_BACKEND_DOCKER_VOLUMES`） |

**注意**：`utils.DeduplicateStrings(append(hostConfig.Binds, e.config.volumes...))` 确保 step 级声明优先（先出现的保留）。

##### 2.3.1.3 toVol vs Binds 的分工

- `config.Volumes`（`convert.go:141`）：声明容器内需要挂载的目录路径（`map[string]struct{}`），不指定来源
- `hostConfig.Binds`：实际的绑定挂载映射，格式 `src:dst[:mode]`

两者必须对应，Docker daemon 才能正确挂载。

##### 2.3.1.4 卷字符串解析

`splitVolumeParts`（`convert.go:216`）使用正则分解卷字符串：
```go
pattern := `^((?:[\w]\:)?[^\:]*)\:((?:[\w]\:)?[^\:]*)(?:\:([rwom]*))?`
```
兼容 Windows 盘符路径（如 `C:\src:C:\src:ro`），分解为 `[hostPath, containerPath, mode]`。

---

### 2.4 阶段 4：TailStep —— 日志流式输出

**入口**：`pipeline/backend/docker/docker.go:316`

```go
func (e *docker) TailStep(ctx context.Context, step *backend_types.Step, taskUUID string) (io.ReadCloser, error) {
    logs, err := e.client.ContainerLogs(ctx, toContainerName(step), client.ContainerLogsOptions{
        Follow:     true,
        ShowStdout: true,
        ShowStderr: true,
    })

    rc, wc := io.Pipe()

    // 后台 goroutine 解复用 stdout/stderr
    go func() {
        _, _ = stdcopy.StdCopy(wc, wc, logs)
        _ = logs.Close()
        _ = wc.Close()
    }()
    return rc, nil
}
```

Docker 的 `ContainerLogs` 返回的流包含多路复用的 stdout/stderr，通过 `stdcopy.StdCopy` 解复用后写入管道。

---

### 2.5 阶段 5：completeStep —— 退出码回收 + 错误映射

**入口**：`pipeline/runtime/step.go:151`

这是 **退出码回收** 的核心衔接点。

```go
func (r *Runtime) completeStep(runnerCtx context.Context, step *backend_types.Step, waitForLogs func(), startTime int64) (*backend_types.State, error) {
    // ──────────────────────────────────────────
    // 衔接点 1: 等待日志流排空（必须在 WaitStep 之前）
    // ──────────────────────────────────────────
    waitForLogs()

    // ──────────────────────────────────────────
    // 衔接点 2: WaitStep —— 阻塞等待容器退出
    // ──────────────────────────────────────────
    waitState, err := r.engine.WaitStep(r.ctx, step, r.taskUUID)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            if waitState == nil {
                waitState = &backend_types.State{}
            }
            waitState.Error = pipeline_errors.ErrCancel
        } else {
            return nil, err
        }
    }

    // ──────────────────────────────────────────
    // 衔接点 3: DestroyStep —— 销毁容器（使用 runnerCtx，不受 workflow 取消影响）
    // ──────────────────────────────────────────
    if err := r.engine.DestroyStep(runnerCtx, step, r.taskUUID); err != nil {
        return nil, err
    }

    waitState.Started = startTime

    // ──────────────────────────────────────────
    // 衔接点 4: 重新检查上下文取消（可能与 WaitStep 竞态）
    // ──────────────────────────────────────────
    if ctxErr := r.ctx.Err(); ctxErr != nil && errors.Is(ctxErr, context.Canceled) {
        waitState.Error = pipeline_errors.ErrCancel
    }

    // ──────────────────────────────────────────
    // 衔接点 5: 退出码语义映射
    // ──────────────────────────────────────────
    if waitState.OOMKilled {
        return waitState, &pipeline_errors.OomError{
            UUID: step.UUID,
            Code: waitState.ExitCode,
        }
    }
    if waitState.ExitCode != 0 {
        return waitState, &pipeline_errors.ExitError{
            UUID: step.UUID,
            Code: waitState.ExitCode,
        }
    }

    return waitState, nil
}
```

#### 2.5.1 WaitStep —— 退出码提取

**入口**：`pipeline/backend/docker/docker.go:278`

```go
func (e *docker) WaitStep(ctx context.Context, step *backend_types.Step, taskUUID string) (*backend_types.State, error) {
    containerName := toContainerName(step)

    // 阻塞等待容器退出
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

    // 获取完整容器状态
    info, err := e.client.ContainerInspect(ctx, containerName, client.ContainerInspectOptions{})
    if err != nil {
        return nil, err
    }

    exitCode := info.Container.State.ExitCode

    // ──────────────────────────────────────────
    // Windows 异常退出码修正
    // Windows Docker 可能返回 4294967295（uint32 max = int32(-1)）
    // ──────────────────────────────────────────
    if int64(exitCode) == int64(4294967295) {
        exitCode = -1
    }

    return &backend_types.State{
        Exited:    true,
        ExitCode:  exitCode,
        OOMKilled: info.Container.State.OOMKilled,
    }, nil
}
```

#### 2.5.2 DestroyStep —— 容器清理

**入口**：`pipeline/backend/docker/docker.go:340`

```go
func (e *docker) DestroyStep(ctx context.Context, step *backend_types.Step, taskUUID string) error {
    containerName := toContainerName(step)
    var stopErr error

    // 优雅停止
    if _, err := e.client.ContainerStop(ctx, containerName, client.ContainerStopOptions{
        Timeout: toRef(int(e.config.stopTimeout)),
    }); err != nil && !isErrContainerNotFoundOrNotRunning(err) {
        stopErr = fmt.Errorf("could not stop container '%s': %w", step.Name, err)
    }

    // 优雅停止失败则强制杀死
    if _, err := e.client.ContainerKill(ctx, containerName, client.ContainerKillOptions{
        Signal: "9",
    }); err != nil && !isErrContainerNotFoundOrNotRunning(err) {
        return errors.Join(stopErr, fmt.Errorf("could not kill container '%s': %w", step.Name, err))
    }

    // 移除容器
    if _, err := e.client.ContainerRemove(ctx, containerName, removeOpts); err != nil && !isErrContainerNotFoundOrNotRunning(err) {
        return fmt.Errorf("could not remove container '%s': %w", step.Name, err)
    }

    return nil
}
```

#### 2.5.3 退出码回收总结

退出码从容器到 Runtime 的完整路径：

```
Container.ExitCode
    ↓ (ContainerInspect)
info.Container.State.ExitCode
    ↓ (Windows 修正)
waitState.ExitCode
    ↓ (completeStep 语义映射)
    ├─ OOMKilled → pipeline_errors.OomError
    ├─ ExitCode!=0 → pipeline_errors.ExitError
    └─ context.Canceled → pipeline_errors.ErrCancel
        ↓ (runStage errgroup)
        r.err.Set(err)
            ↓ (shouldSkipStep)
            后续 step 检查 r.err 决定是否跳过
```

---

### 2.6 阶段 6：DestroyWorkflow —— 环境清理

**入口**：`pipeline/backend/docker/docker.go:369`

```go
func (e *docker) DestroyWorkflow(ctx context.Context, conf *backend_types.Config, taskUUID string) error {
    // 并发销毁所有 step 容器
    errWG := errgroup.Group{}
    for _, stage := range conf.Stages {
        for _, step := range stage.Steps {
            errWG.Go(func() error {
                return e.DestroyStep(ctx, step, taskUUID)
            })
        }
    }
    errWG.Wait()

    // ──────────────────────────────────────────
    // 删除共享卷（指数退避重试，处理"卷仍在使用"竞态）
    // ──────────────────────────────────────────
    var err error
    _, _ = backoff.Retry(ctx, func() (any, error) {
        _, err = e.client.VolumeRemove(ctx, conf.Volume, client.VolumeRemoveOptions{Force: true})
        if err == nil || !isErrVolumeInUse(err) {
            return nil, nil
        }
        return nil, err
    }, backoff.WithMaxTries(maxRetry), backoff.WithBackOff(&backoff.ExponentialBackOff{
        InitialInterval: volumeRetryWait,  // 1s
        Multiplier:      2,                // 每次翻倍
    }))

    // 删除网络
    e.client.NetworkRemove(ctx, conf.Network, client.NetworkRemoveOptions{})

    return nil
}
```

---

## 三、完整时序图（按代码顺序）

```
Runtime.Run(runnerCtx)
  │
  ├─ validateConfig()
  ├─ destroyWorkflowFunc = sync.OnceFunc(...)
  ├─ defer destroyWorkflowFunc()
  ├─ r.started = time.Now().Unix()
  │
  ├─ engine.SetupWorkflow(r.ctx, conf, taskUUID)
  │   ├─ VolumeCreate(conf.Volume)      ←── 卷创建
  │   └─ NetworkCreate(conf.Network)    ←── 网络创建
  │
  └─ 循环 stage in conf.Stages
      │
      ├─ runStage(runnerCtx, stage.Steps)
      │   ├─ errgroup.Group
      │   └─ 并发执行每个 step:
      │       └─ executeStep(runnerCtx, step)
      │           ├─ shouldSkipStep()   ←── 检查 r.err
      │           │   └─ r.err.Get()
      │           ├─ traceStep(started)
      │           ├─ setStepEnv(step)
      │           │
      │           └─ runBlockingStep(runnerCtx, step)
      │               │
      │               ├─ startStep(step)
      │               │   ├─ engine.StartStep(r.ctx, step, taskUUID)
      │               │   │   ├─ toConfig(step)            ←── 镜像选择 + 命令编码
      │               │   │   ├─ toHostConfig(step, conf)  ←── 卷挂载(step级)
      │               │   │   ├─ [ImagePull if step.Pull]  ←── 拉取策略1
      │               │   │   ├─ hostConfig.Binds += volumes ←── 卷挂载(全局)
      │               │   │   ├─ ContainerCreate
      │               │   │   │   └─ [ImagePull if NotFound] ←── 拉取策略2
      │               │   │   ├─ NetworkConnect×N
      │               │   │   └─ ContainerStart
      │               │   ├─ startTime = time.Now().Unix()
      │               │   └─ engine.TailStep(r.ctx, step, taskUUID)
      │               │       └─ ContainerLogs (follow) → io.Pipe
      │               │
      │               └─ completeStep(runnerCtx, step, waitForLogs, startTime)
      │                   ├─ waitForLogs()                 ←── 日志流排空
      │                   ├─ engine.WaitStep(r.ctx, step, taskUUID)
      │                   │   ├─ ContainerWait            ←── 阻塞等待退出
      │                   │   ├─ ContainerInspect         ←── 提取 ExitCode/OOMKilled
      │                   │   └─ [Windows exitCode 修正]
      │                   ├─ engine.DestroyStep(runnerCtx, step, taskUUID)
      │                   │   ├─ ContainerStop
      │                   │   ├─ ContainerKill (if stop failed)
      │                   │   └─ ContainerRemove
      │                   ├─ [context.Canceled 二次检查]
      │                   ├─ [OOM → OomError]             ←── 语义映射
      │                   ├─ [ExitCode!=0 → ExitError]    ←── 语义映射
      │                   └─ traceStep(processState, err, step)
      │
      ├─ select {
      │   case <-r.ctx.Done(): return ErrCancel
      │   case err := <-stageChan: r.err.Set(err)         ←── 错误累积
      │   }
      │
  ├─ destroyWorkflowFunc()
  │   └─ engine.DestroyWorkflow(ctx, conf, taskUUID)
  │       ├─ 并发 DestroyStep × N
  │       ├─ VolumeRemove (重试3次)  ←── 卷删除
  │       └─ NetworkRemove           ←── 网络删除
  │
  └─ r.uploadWait.Wait()
  └─ return r.err.Get()
```

---

## 四、关键设计要点

### 4.1 上下文的双重角色

`Runtime` 使用两个不同的 context：
- `r.ctx`：workflow 执行上下文，用于正常操作（`StartStep`、`WaitStep`），可被取消
- `runnerCtx`：runner 生命周期上下文，用于清理操作（`DestroyStep`、`DestroyWorkflow`），必须比 `r.ctx` 存活更久

这确保即使 workflow 被取消，容器清理仍能正常完成。

### 4.2 镜像拉取的宽容策略

两级 fallback 设计：
- 显式 `pull: true` 拉取失败时，匿名拉取降级到本地镜像，私有仓库则直接报错
- `ContainerCreate` 返回 `NotFound` 时自动拉取重试

既保证了网络抖动时的容错性，又避免了静默使用错误的私有镜像。

### 4.3 卷挂载的叠加语义

`append(step.Volumes, e.config.volumes...)` + 去重意味着：
- 管理员可以通过全局配置为所有容器注入额外卷（如证书目录）
- step 级声明优先（去重时保留先出现的）
- 顺序是先 step 后全局，step 的声明不会被覆盖

### 4.4 退出码的语义丰富化

Docker 后端只返回原始 `ExitCode` + `OOMKilled`，但 Runtime 层将其丰富为三种语义：
- `OomError`：容器因内存不足被杀（环境问题）
- `ExitError`：命令执行失败（用户代码问题）
- `ErrCancel`：workflow 被外部取消（用户操作）

这种分层让上层逻辑能区分「自己的锅」vs「环境的锅」vs「用户主动取消」。

### 4.5 日志流与退出等待的顺序

`waitForLogs()` 必须在 `WaitStep()` 之前调用，因为：
- 某些后端在 `WaitStep` 返回时会关闭日志流
- 如果先 `WaitStep` 再等待日志，可能会丢失最后几行日志
- Docker 后端虽然没有这个问题，但 Runtime 层统一了这个模式

### 4.6 容器命名的隔离性

容器名 `wp_${step.UUID}`（`convert.go:72`）确保全局唯一，使得同一 Docker daemon 上可以安全并发执行多个 workflow。

---

## 五、补充细节分析

### 5.1 Matrix Build：Step 展开多 Container 的对应关系

**展开发生在前端构建阶段**（后端 Docker 层不感知 matrix），完整链路在 `pipeline/frontend/builder/builder.go:52-97`：

```
PipelineBuilder.Build()
  ├─ matrix.ParseString(y.Data)                  ←── 解析 matrix 定义
  │   └─ calc(matrix) → []Axis                   ←── 笛卡尔积生成排列组合
  │       限制: limitTags=10, limitAxis=25       (matrix.go:26-28)
  │
  ├─ 空 matrix 时: axes = [Axis{}]               ←── 非 matrix pipeline 退化为单个空轴
  │
  └─ for i, axis := range axes                   ←── 每轴 = 一个独立 workflow
      ├─ workflow.PID = pidSequence++
      ├─ workflow.Environ = axis                 ←── matrix 变量注入环境
      ├─ workflow.AxisID = i+1                   ←── 多轴时编号
      └─ genItemForWorkflow(workflow, axis, data)
          ├─ environmentVariables(metadata, axis) ←── axis 作为环境变量用于模板替换
          ├─ metadata.EnvVarSubst(data, environ)  ←── ${GO_VERSION} 等占位符替换
          ├─ yaml.ParseString(substituted)        ←── 解析替换后的 YAML
          └─ toInternalRepresentation()           ←── 转为 backend_types.Config
```

**对应关系**：
- 一个 matrix `axis`（如 `GO_VERSION=1.21, REDIS=7`）→ 一个 `Workflow` → 一个 `backend_types.Config`
- 每个 Config 有独立的 `conf.Volume`（`${prefix}_default`）和 `conf.Network`，互不共享
- 展开后多个 workflow 可能被调度到不同 agent 上执行，各自独立创建卷/网络/容器

**关键代码**：
- `pipeline/frontend/yaml/matrix/matrix.go:70-112` —— `calc()` 笛卡尔积算法（阶乘式展开）
- `pipeline/frontend/builder/builder.go:57-88` —— axis 循环生成 workflow
- `pipeline/frontend/builder/builder.go:207-211` —— `environmentVariables()` 将 axis 合并入环境

---

### 5.2 同 Workflow 拉多 Registry 镜像：AuthConfig 列表顺序

AuthConfig 匹配发生在 **编译阶段**（`createProcess`），不是后端 Docker 层。

**匹配逻辑**：`pipeline/frontend/yaml/compiler/convert.go:131-138`

```go
authConfig := backend_types.Auth{}
for _, registry := range c.registries {          // 按配置顺序遍历
    if utils.MatchHostname(container.Image, registry.Hostname) {
        authConfig.Username = registry.Username
        authConfig.Password = registry.Password
        break                                     // 第一个匹配即停止
    }
}
```

**`MatchHostname` 实现**：`pipeline/frontend/yaml/utils/image.go:98-107`

```go
func MatchHostname(image, hostname string) bool {
    named, err := ParseNamed(image)                // 解析 image 为 reference.Named
    if hostname == "index.docker.io" {
        hostname = "docker.io"                     // Docker Hub 别名统一
    }
    return reference.Domain(named) == hostname     // 比较 domain 部分
}
```

**列表顺序的影响**：
- 多个 registry 配置了相同 hostname 时，**第一个配置优先**，后续被忽略
- 如果 image 没有匹配到任何 registry，`authConfig` 为空，后端走匿名拉取
- `step.AuthConfig` 随 Step 结构体一路传递到 Docker 后端，编码为 `RegistryAuth` 头

**示例**：
```
registries: [
    {Hostname: "registry.example.com", Username: "user1", Password: "pass1"},
    {Hostname: "registry.example.com", Username: "user2", Password: "pass2"},
]
image: "registry.example.com/app:latest" → 匹配第一个（user1），第二个被丢弃
```

---

### 5.3 OOM 时 Tmpfs 是否被优先回收

**结论：Woodpecker 不管理 tmpfs 回收，由 Linux 内核 OOM Killer 统一决策。**

**代码事实**：
- Tmpfs 配置直接透传给 Docker：`convert.go:123-134` → `hostConfig.Tmpfs`
- 没有任何针对 tmpfs 的 OOM 专属处理逻辑
- OOM 检测只有一行：`WaitStep` 中读取 `info.Container.State.OOMKilled`

**内核视角下的行为**：
- tmpfs 内存计入容器 cgroup 的 `memory.limit_in_bytes`（v1）或 `memory.max`（v2）
- tmpfs 写操作触发 cgroup 内存超出时，内核会触发 cgroup 级 OOM Killer
- OOM Killer 根据 `oom_score_adj` 选择进程杀死，**不区分"普通内存" vs "tmpfs 占用"**
- tmpfs 的内容不会被 swap 出去（除非启用 `tmpfs` swap，但 Docker 默认不配置）
- tmpfs 数据随容器 `ContainerRemove` 清空，不是"回收"而是"丢弃"

**映射到 Woodpecker**：
```
cgroup memory.max 超限
    ↓
内核 OOM Killer → 杀死容器内进程
    ↓
容器退出，OOMKilled = true
    ↓
WaitStep → info.Container.State.OOMKilled = true
    ↓
completeStep → pipeline_errors.OomError{UUID, Code}
```

**tmpfs 与 OOM 的实际关系**：如果 step 配置了大量 tmpfs（如 `/dev/shm`），且写入大量数据，会更快触发 OOM。但 Woodpecker 侧无法"优先回收 tmpfs"——只能通过 `WOODPECKER_BACKEND_DOCKER_LIMIT_MEM` 和 `WOODPECKER_BACKEND_DOCKER_LIMIT_SHM_SIZE` 提前限制。

---

### 5.4 Logs 流 Hang 时的超时控制

**Woodpecker 没有为日志流单独设置超时，但有多层级联超时间接兜底。**

**层级 1：Workflow 全局超时**（`agent/runner.go:80-103`）

```go
timeout := time.Hour                                        // 默认 1 小时
if minutes := workflow.Timeout; minutes != 0 {
    timeout = time.Duration(minutes) * time.Minute
}
workflowCtx, _ := context.WithTimeout(ctxMeta, timeout)     // context 级超时
```

超时后 `workflowCtx.Done()` 触发，会：
1. 取消 `ContainerWait`（`WaitStep` 中 `<-ctx.Done()` 分支返回 `context.Canceled`）
2. 取消 `ContainerLogs`（`TailStep` 的 ctx 被取消，流返回错误）
3. 后续 `DestroyStep` 使用 `runnerCtx`（存活更久）清理容器

**层级 2：单行大小切割**（`pipeline/utils/copy_line_by_line.go:45-107`）

```go
func CopyLineByLine(dst io.Writer, src io.Reader, maxSize int) error {
    // 超过 maxSize 的行被强制分块写出，避免单行无限增长导致内存泄漏
    case len(buf) >= maxSize:
        dst.Write(buf[:maxSize])  // 即使没有 \n 也强制写出 maxSize 字节
        buf = buf[maxSize:]
}
```

**层级 3：Agent Healthy 检查**（`agent/state.go:62-73`）

```go
func (s *State) Healthy() bool {
    now := time.Now()
    buf := time.Hour                                      // 超出超时 1 小时缓冲
    for _, item := range s.Metadata {
        if now.After(item.Started.Add(item.Timeout).Add(buf)) {
            return false                                  // 标记 agent 不健康
        }
    }
    return true
}
```

**Hang 场景的实际流程**：
```
日志流挂住（ContainerLogs 无数据返回、无 EOF）
    ↓
waitForLogs() 阻塞
    ↓
workflowCtx timeout 到期 → Cancel
    ↓
ContainerLogs ctx 取消 → rc 返回错误，goroutine 退出
    ↓
waitForLogs() 解除阻塞
    ↓
WaitStep 返回 ctx.Canceled → 映射为 ErrCancel
    ↓
DestroyStep (runnerCtx) 停止+删除容器
```

**注意**：如果是 **goroutine 卡在 `stdcopy.StdCopy`** 且底层 TCP 连接完全静默（没有 RST、没有 FIN），需要靠 TCP keepalive 或 workflow 超时兜底。`WOODPECKER_KEEPALIVE_TIME`（`cmd/agent/core/flags.go:106-109`）配置 gRPC 层 keepalive，但 Docker daemon 连接的 keepalive 取决于系统配置。

---

### 5.5 多 Worker 同时崩的容器留存上限

**结论：没有"容器留存上限"机制，崩溃后残留 = 当时正在运行的容器数。**

**正常路径**（不崩溃）：`DestroyWorkflow` 保证容器一律被清理（`docker.go:369-408`）

```go
errWG := errgroup.Group{}
for _, stage := range conf.Stages {
    for _, step := range stage.Steps {
        errWG.Go(func() error { return e.DestroyStep(...) })  // 并发销毁所有 step
    }
}
```

**崩溃路径**（Agent 进程被杀）：
- 若 Agent 在 workflow 执行中崩溃：`defer destroyWorkflowFunc()` 未执行 → 所有 `wp_${UUID}` 容器 + `${prefix}_default` 卷 + 网络全部残留
- 容器名带 UUID，不会与后续 workflow 冲突，但会占用磁盘和资源

**残留容器的上限 = 并行度 × 单 workflow step 数**：

| 参数 | 配置项 | 默认值 | 说明 |
|------|--------|--------|------|
| 并行 workflow 数 | `WOODPECKER_MAX_WORKFLOWS` | 1 | `cmd/agent/core/flags.go:82-86` |
| step 名冲突防护 | UUID → `wp_${UUID}` | - | 不依赖上限，天然隔离 |
| 重启后清理 | ❌ 无自动扫描残留机制 | - | 重启不清理旧容器 |

**防护建议（代码中缺失，需运维侧）**：
- 定期执行 `docker container prune --filter "label=wp_step"`（容器都打了 `wp_uuid/wp_step` label）
- 或者基于 `wp_` 前缀的命名约定清理

**容器标签**（`convert.go:41-44`）是唯一残留清理线索：
```go
Labels: map[string]string{
    "wp_uuid": step.UUID,
    "wp_step": step.Name,
}
```

---

### 5.6 Cgroup v1 与 v2 兼容差异

**结论：Woodpecker 代码层完全不感知 cgroup 版本差异，全部委托给 Docker/Moby 客户端适配。**

**资源限制参数**（`config.go:33-40` → `convert.go:78-86`）：

```go
type resourceLimit struct {
    MemSwapLimit int64   // WOODPECKER_BACKEND_DOCKER_LIMIT_MEM_SWAP
    MemLimit     int64   // WOODPECKER_BACKEND_DOCKER_LIMIT_MEM
    ShmSize      int64   // WOODPECKER_BACKEND_DOCKER_LIMIT_SHM_SIZE
    CPUQuota     int64   // WOODPECKER_BACKEND_DOCKER_LIMIT_CPU_QUOTA
    CPUShares    int64   // WOODPECKER_BACKEND_DOCKER_LIMIT_CPU_SHARES
    CPUSet       string  // WOODPECKER_BACKEND_DOCKER_LIMIT_CPU_SET
}
```

透传为 Docker `container.Resources`，Docker daemon 根据宿主机 cgroup 版本翻译：

| 参数 | Cgroup v1 | Cgroup v2 | 备注 |
|------|-----------|-----------|------|
| `Memory` | `memory.limit_in_bytes` | `memory.max` | 语义一致 |
| `MemorySwap` | `memory.memsw.limit_in_bytes` | `memory.swap.max` | v2 需 kernel 开启 swapaccount |
| `CPUQuota` | `cpu.cfs_quota_us` + `cpu.cfs_period_us` | `cpu.max` | 语义一致，v2 简化为 single file |
| `CPUShares` | `cpu.shares` | `cpu.weight` | **值范围不同**：v1=[2,262144]，v2=[1,10000]，Docker 自动换算 |
| `CpusetCpus` | `cpuset.cpus` | `cpuset.cpus` | 语义一致 |
| `ShmSize` | 不经过 cgroup（tmpfs mount option） | 同左 | 不涉及 cgroup 版本 |

**CPUShares 换算陷阱**：
- Woodpecker 传入原始值，如 `CPUShares=1024`（v1 常见默认）
- Docker 在 cgroup v2 宿主机上自动按比例换算为 `cpu.weight`（1024 → ~5000）
- 但如果显式设置了 v2 风格值（如 `5000`），在 v1 宿主机上也会被反向换算
- Woodpecker 用户感知不到，但调优时需注意两个版本的权重语义不同

**设备 cgroup 权限**（`convert.go:174-195`）：
```go
container.DeviceMapping{
    PathOnHost:        parts[0],
    PathInContainer:   parts[1],
    CgroupPermissions: "rwm",     // 硬编码 rwm，不区分 cgroup 版本
}
```
`CgroupPermissions` 在 v1 写入 `devices.allow`，v2 写入 `device.allow`，Docker 适配。

---

### 5.7 IPv6 Only 环境：Alias DNS 回退

**结论：Woodpecker 不管理容器内 DNS 解析行为，完全依赖 Docker daemon 的网络 DNS 配置。**

**网络创建流程**（`docker.go:156-175`）：

```go
networkDriver := networkDriverBridge    // Linux: bridge
if e.info.OSType == "windows" {
    networkDriver = networkDriverNAT    // Windows: nat
}
_, err = e.client.NetworkCreate(ctx, conf.Network, client.NetworkCreateOptions{
    Driver:     networkDriver,
    EnableIPv6: &e.config.enableIPv6,   // WOODPECKER_BACKEND_DOCKER_ENABLE_IPV6
})
```

**Step 容器加入网络时的 Alias 设置**（`docker.go:250-271`）：

```go
for _, net := range step.Networks {
    _, err = e.client.NetworkConnect(ctx, net.Name, client.NetworkConnectOptions{
        EndpointConfig: &network.EndpointSettings{
            Aliases: net.Aliases,       // 如 ["build", "app"]，来自 convert.go:58-68
        },
        Container: containerName,
    })
}
```

**Alias 的解析链路**：
```
容器内进程访问 http://build:8080
    ↓
容器内 /etc/resolv.conf → 指向 Docker 内置 DNS resolver (127.0.0.11)
    ↓
Docker 内置 DNS 查找网络内的 alias "build"
    ↓
找到对应容器的 Endpoint → 返回其 IP 地址
    ↓
IPv4/IPv6 双栈环境: 返回 A + AAAA 记录
IPv6-only 环境: 只返回 AAAA 记录（如 enableIPv6=true 且容器分配了 v6 地址）
```

**Woodpecker 可配置项**：

| 配置 | 作用 | 对 IPv6-only 的影响 |
|------|------|---------------------|
| `WOODPECKER_BACKEND_DOCKER_ENABLE_IPV6` | 给 bridge 网络分配 IPv6 子网 | 必须 `true` 才能拿到 v6 地址 |
| `step.DNS` | 容器 `/etc/resolv.conf` 附加 nameserver | 可填 IPv6 DNS 地址，但 Docker 内置 resolver 优先级更高 |
| `step.DNSSearch` | search domain | 不影响 A/AAAA 选择 |
| `step.ExtraHosts` | `/etc/hosts` 硬编码 | 填 IPv6 地址（`host:[::1]`）即可绕过 DNS |

**无回退机制**：
- 如果 Docker 网络层只分配了 IPv4 地址而环境是 IPv6-only，alias 无法解析 → 连接失败
- Woodpecker 不会在 step 侧做 `getaddrinfo` 回退，需要 Docker daemon 层面正确配置 IPv6
- 补救：`step.ExtraHosts` 手动注入 IPv6 地址映射

---

### 5.8 401 Unauthorized 自动 Refresh Token 时机

**结论：Woodpecker 的 registry 鉴权使用静态 Username/Password 对，没有 token refresh 机制。401 直接失败。**

**AuthConfig 编码流程**（`docker.go:193-197` + `convert.go:199-205`）：

```go
// 编译阶段: 从 registries 列表匹配到 {Username, Password}
step.AuthConfig = backend_types.Auth{Username, Password}

// Docker 后端 StartStep:
pullOpts := client.ImagePullOptions{}
if step.AuthConfig.Username != "" && step.AuthConfig.Password != "" {
    pullOpts.RegistryAuth, _ = encodeAuthToBase64(step.AuthConfig)
    // encodeAuthToBase64 = base64(json.Marshal({Username, Password}))
}
e.client.ImagePull(ctx, config.Image, pullOpts)
```

**401 的两种场景**：

| 场景 | 代码行为 | 结果 |
|------|----------|------|
| `step.Pull=true` 时拉取 401 + 密码非空 | `docker.go:213-215`: `if pErr != nil && Password != "" { return pErr }` | 直接报错，不重试 |
| `step.Pull=true` 时拉取 401 + 密码为空（匿名） | 错误被吞，降级到本地镜像 | 静默继续（设计如此，见注释 drone#1917） |
| `ContainerCreate` 触发拉取时 401 | `docker.go:229-231`: `if pErr != nil { return pErr }` | 直接报错，不区分密码是否为空 |

**无 refresh 的原因**：
- `AuthConfig` 是静态的 `{Username, Password}` 对（`pipeline/backend/types/auth.go:15-21`），没有 `RefreshToken`、`ExpiresAt` 等字段
- `encodeAuthToBase64` 直接 JSON + base64，不获取短期 token
- Docker 客户端 `ImagePull` 内部虽然有 token 协商（Bearer auth 挑战），但那是 Docker daemon ↔ registry 之间的协议，Woodpecker 侧不可见
- 如果使用的是短期 password（如 AWS ECR 的 12h 密码），需要在编译 workflow 之前**外部提前刷新**，Woodpecker 不会帮你做

**OAuth token refresh 的区别**：搜索到的 `server/forge/refresh.go` 是 **用户 OAuth token 刷新**（用于 forge/GitHub API 访问），与 registry 镜像拉取无关：
```go
// 触发时机: 30 分钟内即将过期（refresh.go:59-60）
if time.Until(user.Expiry) < 30*time.Minute {
    // singleflight 防止并发刷新 → Refresh() → 写回 DB
}
```

**建议（若需自动 refresh registry token）**：在 workflow 编译前（如 API 层或 custom plugin）调用 registry API 获取新密码，然后通过 registries 列表传入最新凭证。

---

## 六、边界条件补充分析

### 6.1 Matrix Step 的 include 与 exclude 修饰符处理顺序

Woodpecker 的 matrix 机制在两个不同层面使用了 include/exclude，处理顺序完全不同。

#### 层面 1：Matrix 定义中的 include（无 exclude）

**入口**：`pipeline/frontend/yaml/matrix/matrix.go:47-63`

```go
func Parse(data []byte) ([]Axis, error) {
    axis, err := parseList(data)     // 尝试解析 matrix.include 语法
    if err == nil && len(axis) != 0 {
        return axis, nil             // include 语法成功则直接返回，跳过 calc()
    }

    matrix, err := parse(data)       // 回退到 map 语法
    return calc(matrix), nil         // 笛卡尔积展开
}
```

**parseList 只识别 include，不识别 exclude**（`matrix.go:124-135`）：

```go
func parseList(raw []byte) ([]Axis, error) {
    data := struct {
        Matrix struct {
            Include []Axis                    // 只有 Include，没有 Exclude
        }
    }{}
    xyaml.Unmarshal(raw, &data)
    return data.Matrix.Include, nil
}
```

这意味着 **matrix 定义层面不支持 exclude**，只支持两种写法：
- `matrix.include: [{go: "1.21"}, {go: "1.22"}]` → 显式列举
- `matrix: {go: ["1.21", "1.22"], redis: ["6", "7"]}` → 笛卡尔积自动展开

两种写法互斥：如果 `parseList` 解析到非空的 `include` 列表，`calc()` 不会被调用。

#### 层面 2：Step 约束中的 include/exclude（when 块）

这是 step 运行时条件判断，有两类约束对象：

**Path 约束**（`pipeline/frontend/yaml/constraint/path.go:100-120`）：

```go
func (c *Path) Match(v []string, message string) bool {
    // 1. ignore_message 优先级最高 → 直接返回 true
    if len(c.IgnoreMessage) > 0 && strings.Contains(...) { return true }

    // 2. 空文件列表 → on_empty 决定
    if len(v) == 0 { return c.OnEmpty.ValueOrDefault(true) }

    // 3. exclude 先于 include 检查
    if len(c.Exclude) > 0 && c.Excludes(v) { return false }  // 任何文件全匹配 exclude → 拒绝
    if len(c.Include) > 0 && !c.Includes(v) { return false }  // 没有文件匹配 include → 拒绝
    return true
}
```

**关键语义差异**：
- `Excludes(v)`: 所有文件都必须匹配某个 exclude 模式才返回 true（**全部命中才排除**）
- `Includes(v)`: 任意文件匹配某个 include 模式就返回 true（**任一命中即包含**）

这意味着 `include + exclude` 同时存在时，exclude 是 **全量排除**（所有文件都是被排除的文件），include 是 **存在性包含**（至少有一个文件在包含列表中）。

**Map 约束**（`pipeline/frontend/yaml/constraint/map.go:27-53`，用于 `when.matrix`）：

```go
func (c *Map) Match(params map[string]string) bool {
    // exclude 先检查
    if len(c.Exclude) != 0 {
        var matches int
        for key, val := range c.Exclude {
            if ok, _ := doublestar.Match(val, params[key]); ok {
                matches++
            }
        }
        if matches == len(c.Exclude) {  // 所有 exclude 键值都匹配 → 排除
            return false
        }
    }
    // 再检查 include
    for key, val := range c.Include {
        if ok, _ := doublestar.Match(val, params[key]); !ok {
            return false                 // 任一 include 键值不匹配 → 排除
        }
    }
    return true
}
```

**统一处理顺序**：无论 Path 还是 Map 约束，都是 **exclude 先于 include**，且两者都是 **AND 语义**（所有条件都满足才生效）。

#### 与 matrix 展开的衔接

matrix 展开后，每个 Axis 变成独立 Workflow。Step 的 `when.matrix` 约束在编译阶段逐个 Axis 检查，决定该 step 是否出现在此 Axis 的 workflow 中：

```
matrix: {go: ["1.21", "1.22"], redis: ["6", "7"]}
    ↓ calc() → 4 个 Axis
    ↓
Axis {go:1.21, redis:6} → 编译 → when.matrix: {include: {go: "1.*"}, exclude: {redis: "6"}}
    → exclude 全匹配(redis:6 匹配 "6") → step 不出现在此 workflow
Axis {go:1.21, redis:7} → 编译 → when.matrix: {include: {go: "1.*"}, exclude: {redis: "6"}}
    → exclude 不全匹配(redis:7 ≠ "6") → include 全匹配(go:1.21 匹配 "1.*") → step 出现
```

---

### 6.2 同 Image 多 Push 路径歧义解决

**结论：Woodpecker 不感知镜像推送（push），推送由 step 内部命令完成，Docker 后端只负责拉取（pull）。**

**代码证据**：
- Docker 后端只有 `ImagePull` 调用，没有任何 `ImagePush` 调用
- `plugins/docker`（社区常用推送插件）是一个普通 step，通过 `settings` 配置 repo/tags，在容器内部调用 Docker daemon 完成推送
- 编译阶段 `createProcess` 对 `container.Image` 只做两件事：写入 `step.Image` 和匹配 `authConfig`

**多 push 路径歧义场景**：
```yaml
steps:
  publish-ghcr:
    image: plugins/docker
    settings:
      repo: ghcr.io/org/app
      registry: ghcr.io
  publish-dockerhub:
    image: plugins/docker
    settings:
      repo: org/app
      registry: docker.io
```

**歧义解决方式**：每个 step 独立匹配 `authConfig`（`convert.go:131-138`），推送哪个 registry 由 step 的 `settings` 决定，鉴权信息通过 `MatchHostname` 分别匹配。不存在"同 image 多 push 路径"的歧义，因为推送路径是 step 内部逻辑而非 Woodpecker 编排。

**但存在 AuthConfig 匹配歧义**：如果 `plugins/docker` 的 image 需要从 `docker.io` 拉取，而 step 又要推送到 `ghcr.io`：
- `step.AuthConfig` 匹配的是 **镜像拉取** 的 registry（`plugins/docker` → `docker.io`）
- 推送目标 registry 的鉴权通过 `settings.registry` + `settings.username/password` 传入容器内部环境变量，不走 `step.AuthConfig`

**Docker config 文件的补充鉴权**（`server/services/registry/filesystem.go:39-88`）：

```go
func parseDockerConfig(path string) ([]*model.Registry, error) {
    configFile := configfile.ConfigFile{AuthConfigs: make(map[string]types.AuthConfig)}
    json.NewDecoder(f).Decode(&configFile)

    // CredentialHelpers 优先解析
    for registryHostname := range configFile.CredentialHelpers {
        newAuth, _ := configFile.GetAuthConfig(registryHostname)
        configFile.AuthConfigs[registryHostname] = newAuth
    }

    // AuthConfigs 解码 base64 auth 字段
    for addr, ac := range configFile.AuthConfigs { ... }

    // 转为 model.Registry 列表
    for key, auth := range configFile.AuthConfigs {
        registries = append(registries, &model.Registry{
            Address:  key,        // hostname 作为 Address
            Username: auth.Username,
            Password: auth.Password,
            ReadOnly: true,       // 标记为只读（来自文件系统）
        })
    }
}
```

此文件通过 `--docker-config` 全局加载，与 DB 中的 registry 合并后传给编译器。文件中的 `CredentialHelpers` 条目会被解析后合并到 `AuthConfigs` 中，确保 cred helper 支持的 registry 也能被匹配到。

---

### 6.3 Tmpfs 与 Page Cache 监控区分

**结论：Woodpecker 没有任何运行时监控机制区分 tmpfs 与 page cache，Docker API 也不提供这种区分。**

**代码中 tmpfs 的完整生命周期**：

1. **创建**：`convert.go:123-134` → `hostConfig.Tmpfs` 传给 Docker daemon
2. **运行时**：tmpfs 挂载在容器内存中，完全透明
3. **检测 OOM**：`WaitStep` 检查 `info.Container.State.OOMKilled`，不区分内存来源
4. **销毁**：`DestroyStep` → `ContainerRemove` 时 tmpfs 随容器消失

**Docker stats API 的局限**：
- `ContainerStats` 返回的 `memory_stats.usage` 包含 page cache + tmpfs + anon，是**混合值**
- Docker 不会在 stats 流中细分 tmpfs 占用
- `memory_stats.stats` 在 cgroup v2 中可能包含 `file`（page cache）和 `shmem`（shared memory），但 Woodpecker 不读取这些字段

**如果需要区分**：
- 容器内：`df -h /tmpfs-mount-point` 查看 tmpfs 使用量
- 宿主机 cgroup v2：`memory.current` vs `memory.stat.file` vs `memory.stat.shmem`
- 这都需要在 step 命令中手动执行，Woodpecker 不提供结构化报告

**page cache 与 tmpfs 的关系**：
- tmpfs 写入会同时增加 `memory.current` 和 `memory.stat.shmem`
- tmpfs 写入**不会**增加 `memory.stat.file`（file 是文件系统 page cache）
- 但 `memory.current` = `anon` + `file` + `shmem` + ... ，所以 tmpfs 计入总内存
- OOM Killer 看 `memory.current` > `memory.max`，不区分来源

---

### 6.4 Logs 流超时的动态调整

**结论：Woodpecker 不支持日志流超时的动态调整，超时值在 workflow 开始时确定，中途不可变。**

**超时确定的代码路径**（`agent/runner.go:79-102`）：

```go
// 超时值在 workflow 启动时一次性计算
timeout := time.Hour                              // 默认 1 小时
if minutes := workflow.Timeout; minutes != 0 {
    timeout = time.Duration(minutes) * time.Minute  // 来自 YAML timeout 字段
}

// 创建不可变更的 context
workflowCtx, _ := context.WithTimeout(ctxMeta, timeout)
workflowCtx, cancelWorkflowCtx := context.WithCancelCause(workflowCtx)
```

**不可动态调整的原因**：
- `context.WithTimeout` 创建的 timer 在创建时就固定了截止时间
- Go 标准库没有提供修改已创建 context 截止时间的 API
- Runtime 没有任何 Extend/restart context 的逻辑

**队列层的 Lease 延长**（`server/queue/fifo.go:185-200`）：

```go
func (q *fifo) Extend(_ context.Context, agentID int64, taskID string) error {
    state, ok := q.running[taskID]
    if ok {
        state.deadline = time.Now().Add(q.extension)  // 延长 queue 层 deadline
        return nil
    }
    return ErrNotFound
}
```

这是**队列层**的 lease 续约，防止队列认为 task 已死而重新调度。与 workflow 的日志流超时完全无关。队列层 `TaskTimeout` 默认仅 1 分钟（`shared/constant/constant.go:40-41`），agent 需要定期调用 `Extend` 续约。

**三层超时的关系**：

| 层次 | 超时值 | 可动态调整 | 控制对象 |
|------|--------|-----------|---------|
| 队列 FIFO | 1 分钟（`TaskTimeout`） | ✅ 通过 `Extend` 续约 | 队列认为 task 是否存活 |
| Workflow context | 默认 1 小时或 YAML `timeout` | ❌ 不可变 | 日志流 + WaitStep + 整个 workflow |
| Agent 健康 | 超时 + 1h 缓冲 | ❌ 不可变 | 标记 agent 是否健康 |

**如果需要动态调整**：必须在 `runner.go` 中引入新的 Extend 机制，将 queue 层的续约信号传递到 workflow context 层，当前代码无此路径。

---

### 6.5 容器滚动 Evict 策略

**结论：Woodpecker 没有容器级别的滚动 evict 策略，只有队列级别的 task 过期重提交。**

**队列层 Evict**（`server/queue/fifo.go:338-348`）：

```go
func (q *fifo) resubmitExpiredPipelines() {
    for taskID, taskState := range q.running {
        if time.Now().After(taskState.deadline) {
            log.Info().Msgf("queue: resubmitting expired task %s", taskID)
            taskState.error = ErrTaskExpired
            q.pending.PushFront(taskState.item)    // 重新入队（队首优先）
            delete(q.running, taskID)
            close(taskState.done)                  // 通知 Wait() 返回
        }
    }
}
```

**关键行为**：
- 过期 task 被重新放入 pending 队列**队首**（`PushFront`），优先重新调度
- 但**不会主动停止已运行的容器**——只是从队列视角认为它"死了"
- agent 侧的 workflow context 如果还没到期，容器会继续运行
- 这导致同一 task 可能被两个 agent 同时执行（队列认为已过期重新分配 + 原始 agent 仍在跑）

**Pipeline 取消时的批量 Evict**（`server/pipeline/cancel.go:42-53`）：

```go
// First cancel/evict workflows in the queue in one go
var workflowsToCancel []string
for _, w := range workflows {
    if w.State == model.StatusRunning || w.State == model.StatusPending {
        workflowsToCancel = append(workflowsToCancel, fmt.Sprint(w.ID))
    }
}
server.Config.Services.Scheduler.ErrorAtOnce(ctx, workflowsToCancel, queue.ErrCancel)
```

**批量 Evict 流程**：
1. `ErrorAtOnce` 批量标记所有 running + pending 的 workflow 为 `ErrCancel`
2. 队列层 `finished()` 从 `q.running` 删除 + 从 `q.pending` 移除 + 通知 `Wait()` 返回
3. Agent 侧收到取消信号 → workflow context cancel → `DestroyStep` 清理容器

**没有"滚动"策略**：
- 不存在 LRU/LFU 容器驱逐
- 不存在基于资源压力（如磁盘满）的容器驱逐
- 不存在优先级驱动的容器驱逐（所有 `wp_` 容器平等对待）
- 唯一的"evict"触发条件是：队列 lease 过期 或 用户手动取消

**Agent 断连时的处理**（`server/queue/fifo.go:243-253`）：

```go
func (q *fifo) KickAgentWorkers(agentID int64) {
    for worker := range q.workers {
        if worker.agentID == agentID {
            worker.stop(ErrWorkerKicked)    // 取消 worker 的 Poll context
            delete(q.workers, worker)       // 从 workers map 移除
        }
    }
}
```

Kick 后 agent 的 running tasks 不会被立即清理——需要等队列 lease 过期（1 分钟无 Extend），`resubmitExpiredPipelines` 才会将它们重新入队。

---

### 6.6 Systemd Cgroup Driver 与 Cgroupfs 混合场景

**结论：Woodpecker 代码完全不感知 cgroup driver 差异，但混合场景会导致容器创建失败。**

**代码中与 cgroup 相关的所有交互**：

| 代码位置 | 操作 | 是否涉及 driver |
|----------|------|----------------|
| `docker.go:156` VolumeCreate | 创建卷 | ❌ |
| `docker.go:171` NetworkCreate | 创建网络 | ❌ |
| `docker.go:178` StartStep → ContainerCreate | 创建容器 | ✅ 间接 |
| `convert.go:78-86` Resources | 设置资源限制 | ✅ 间接 |
| `convert.go:174-195` DeviceMapping | 设备权限 | ✅ 间接 |
| `docker.go:278` WaitStep → ContainerInspect | 查询状态 | ❌ |

所有 cgroup 交互都通过 Docker API 间接进行，Woodpecker 不直接操作 `/sys/fs/cgroup`。

**混合场景的问题根源**：

Docker daemon 的 cgroup driver 由 `/etc/docker/daemon.json` 的 `"exec-opts": ["native.cgroupdriver=systemd"]` 决定。当 Docker 使用 systemd driver 而宿主机某些路径仍按 cgroupfs 布局时：

| 现象 | 原因 | Woodpecker 感知 |
|------|------|-----------------|
| `ContainerCreate` 返回 500 | Docker 尝试在 `/sys/fs/cgroup/system.slice/docker-xxx.scope/` 创建 cgroup，但路径被 cgroupfs 模式的其他进程占用 | ❌ 只看到错误 |
| 资源限制不生效 | Docker 写入 `system.slice` 下的 cgroup 文件，但实际进程在 `cgroupfs` 树下 | ❌ 只看到 ExitCode=0 |
| OOMKilled 检测失败 | `ContainerInspect` 读取的 cgroup 统计来自错误的 cgroup 树 | ❌ 只看到 OOMKilled=false |

**Woodpecker 可以感知到的唯一信号**：
- `ContainerCreate` 返回错误 → `StartStep` 直接返回错误
- 但错误消息是 Docker 的内部错误，Woodpecker 不解析其含义

**代码中没有 cgroup driver 探测**：
- `Load()` 方法（`docker.go:131-155`）只获取 `e.info`（Docker 系统信息），不检查 cgroup driver
- `e.info` 包含 `OSType`、`Architecture` 等，但不包含 `CgroupDriver`
- 没有 cgroup driver 兼容性检查或降级逻辑

**运维侧必须保证**：Docker daemon 的 cgroup driver 与宿主机 init 系统使用同一个（systemd 系统 → systemd driver，其他 → cgroupfs driver），否则 Woodpecker 无法诊断问题。

---

### 6.7 IPv6 反向 DNS 查询

**结论：Woodpecker 不管理反向 DNS（PTR）查询，完全依赖 Docker 网络栈和容器内 glibc/musl 解析。**

**代码中 DNS 相关的所有配置**：

1. **网络级 IPv6**（`docker.go:171-175`）：
```go
_, err = e.client.NetworkCreate(ctx, conf.Network, client.NetworkCreateOptions{
    EnableIPv6: &e.config.enableIPv6,   // 分配 IPv6 子网
})
```

2. **容器级 DNS**（`convert.go:44` → `step.DNS` + `step.DNSSearch`）：
```go
config := &container.Config{...}
hostConfig := &container.HostConfig{...}
// step.DNS → hostConfig.DNS
// step.DNSSearch → hostConfig.DNSSearch
```

3. **ExtraHosts**（`convert.go:70-78` → `step.ExtraHosts` → `/etc/hosts`）：
```go
extraHosts[i] = backend_types.HostAlias{Name: name, IP: ip}
```

**反向 DNS 查询的完整路径**：

```
容器内进程执行 gethostbyaddr("fd00:db8::1")
    ↓
glibc/musl 调用 getnameinfo()
    ↓
查询 /etc/nsswitch.conf → hosts: files dns
    ↓
先查 /etc/hosts → 没有 IPv6 PTR 条目（Docker 不自动写入反向映射）
    ↓
查 DNS nameserver（/etc/resolv.conf → 127.0.0.11）
    ↓
Docker 内置 DNS resolver 不支持 PTR 查询
    ↓
转发到宿主机 /etc/resolv.conf 的 nameserver
    ↓
如果上游 DNS 支持 IPv6 反向解析区域 → 返回 PTR 记录
否则 → NXDOMAIN → 解析失败
```

**关键缺失**：
- Docker 的 `NetworkConnect` 中 `Aliases` 只设置正向映射（name → IP），不设置反向映射
- `/etc/hosts` 中只有 `hostname → IP` 条目，没有 `IP → hostname` 条目
- 容器内 `gethostbyaddr()` 对 IPv6 地址几乎必然失败

**对 Woodpecker 的影响**：
- step 之间通过 alias（如 `http://build:8080`）通信 → 正向 DNS ✅
- 日志中 IP 地址反解为主机名 → 反向 DNS ❌
- 某些应用（如 PostgreSQL）启动时做反向 DNS 验证 → 可能失败

**绕过方案**：在 step 的 `extra_hosts` 中手动写入 IPv6 反向映射（但 `extra_hosts` 的格式 `name:ip` 只写正向 `/etc/hosts`，无法写反向条目）。实际解决方案是在应用层关闭反向 DNS 检查。

---

### 6.8 AuthConfig Refresh 失败时全新 Login 回退

**结论：Forge OAuth token refresh 失败后不会回退到全新 login，用户必须重新授权。Registry 鉴权 refresh 失败后不会重试任何操作。**

#### Forge OAuth Token（用于 Forge API 访问）

**Refresh 流程**（`server/forge/refresh.go:58-100`）：

```go
func Refresh(ctx context.Context, forge Forge, _store store.Store, user *model.User) {
    if refresher, ok := forge.(Refresher); ok {
        if time.Now().UTC().Unix() < (user.Expiry - tokenMinTTL) {
            return                              // 未过期 → 跳过
        }

        result, err, _ := refreshGroup.Do(key, func() (any, error) {
            userUpdated, err := refresher.Refresh(ctx, user)
            if err != nil {
                return nil, err                 // refresh 失败 → 返回错误
            }
            // ... 持久化到 DB
        })

        if err != nil {
            log.Error().Err(err).Msgf("refresh oauth token of user '%s' failed", user.Login)
            return                              // ← 仅打日志，不回退到 login
        }

        // 复制新 token 到调用者
        user.AccessToken = r.AccessToken
        user.RefreshToken = r.RefreshToken
        user.Expiry = r.Expiry
    }
}
```

**GitLab Refresh 实现**（`server/forge/gitlab/gitlab.go:160-175`）：

```go
func (g *GitLab) Refresh(ctx context.Context, user *model.User) (bool, error) {
    source := config.TokenSource(oauth2Ctx, &oauth2.Token{
        RefreshToken: user.RefreshToken,
    })
    token, err := source.Token()
    if err != nil || len(token.AccessToken) == 0 {
        return false, err                       // OAuth2 refresh 失败 → 直接返回错误
    }
    // 更新 user token 字段
}
```

**Bitbucket Refresh 实现**（`server/forge/bitbucket/bitbucket.go:124-133`）：

```go
func (c *config) Refresh(ctx context.Context, user *model.User) (bool, error) {
    source := config.TokenSource(ctx, &oauth2.Token{RefreshToken: user.RefreshToken})
    token, err := source.Token()
    if err != nil || len(token.AccessToken) == 0 {
        return false, err                       // 同样直接返回错误
    }
}
```

**Refresh 失败后的行为链**：

```
refresher.Refresh() 返回错误
    ↓
refreshGroup.Do() 返回 err
    ↓
log.Error() 打印日志
    ↓
Refresh() 函数 return（不更新 user）
    ↓
调用方（如 API handler）继续使用旧 token
    ↓
Forge API 调用使用过期/无效 token → 返回 401
    ↓
API handler 返回错误给前端
    ↓
用户必须重新登录授权
```

**没有"全新 login 回退"的原因**：
- 全新 OAuth login 需要用户交互（浏览器重定向 → 授权页面 → 回调）
- 服务端无法在后台自动触发用户的 OAuth 授权流程
- `Refresh()` 在服务端 goroutine 中执行，无用户浏览器上下文
- 代码中没有存储 OAuth client_secret 用于服务端发起授权的路径

#### Registry 鉴权（用于镜像拉取）

**不存在 refresh 机制**，更不存在"refresh 失败 → login 回退"的路径。

- `step.AuthConfig` 是静态 `{Username, Password}`，一次写入不再变更
- `ImagePull` 失败直接返回错误或降级到本地镜像（见 5.8 节分析）
- 没有任何重新获取 registry 凭证的逻辑

**唯一的外部刷新路径**：`server/services/registry/filesystem.go:39-88` 的 `parseDockerConfig` 在服务启动时读取 `--docker-config` 指向的文件。如果该文件内容更新（如外部 cron 刷新了 ECR 密码），需要**重启 Woodpecker server** 才能生效，没有热加载机制。

---

## 七、关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| **核心链路** | | |
| Backend 接口定义 | `pipeline/backend/types/backend.go` | 60 |
| Docker 后端主逻辑 | `pipeline/backend/docker/docker.go` | - |
| SetupWorkflow | `pipeline/backend/docker/docker.go` | 156 |
| StartStep | `pipeline/backend/docker/docker.go` | 178 |
| WaitStep | `pipeline/backend/docker/docker.go` | 278 |
| DestroyStep | `pipeline/backend/docker/docker.go` | 340 |
| DestroyWorkflow | `pipeline/backend/docker/docker.go` | 369 |
| toConfig (镜像+命令) | `pipeline/backend/docker/convert.go` | 36 |
| toHostConfig (卷挂载) | `pipeline/backend/docker/convert.go` | 77 |
| 命令编码注入 | `pipeline/backend/common/script.go` | 21 |
| Runtime 主循环 | `pipeline/runtime/workflow.go` | 35 |
| executeStep | `pipeline/runtime/step.go` | 34 |
| startStep | `pipeline/runtime/step.go` | 122 |
| completeStep | `pipeline/runtime/step.go` | 151 |
| shouldSkipStep | `pipeline/runtime/step.go` | 65 |
| 错误类型定义 | `pipeline/errors/runtime.go` | - |
| Step 结构体 | `pipeline/backend/types/step.go` | 18 |
| State 结构体 | `pipeline/backend/types/state.go` | 18 |
| **补充章节** | | |
| Matrix 笛卡尔积展开 | `pipeline/frontend/yaml/matrix/matrix.go` | 70 |
| Matrix Build 生成 Workflow | `pipeline/frontend/builder/builder.go` | 52-97 |
| AuthConfig 匹配 registry | `pipeline/frontend/yaml/compiler/convert.go` | 131 |
| Image hostname 匹配 | `pipeline/frontend/yaml/utils/image.go` | 98 |
| Tmpfs 配置 | `pipeline/backend/docker/convert.go` | 123 |
| 日志单行大小切割 | `pipeline/utils/copy_line_by_line.go` | 45 |
| Workflow 超时设置 | `agent/runner.go` | 80 |
| Agent 健康检查 | `agent/state.go` | 62 |
| Agent 配置 (MAX_WORKFLOWS) | `cmd/agent/core/flags.go` | 82 |
| Docker 资源限制配置 | `pipeline/backend/docker/config.go` | 33 |
| Docker 后端 Flags | `pipeline/backend/docker/flags.go` | - |
| Cgroup 设备权限 | `pipeline/backend/docker/convert.go` | 174 |
| IPv6 网络创建 | `pipeline/backend/docker/docker.go` | 171 |
| Network Alias 设置 | `pipeline/backend/docker/docker.go` | 250 |
| Registry Auth 编码 | `pipeline/backend/docker/convert.go` | 199 |
| 卷字符串解析正则 | `pipeline/backend/docker/convert.go` | 216 |
| **边界条件章节** | | |
| Matrix Parse (include 语法) | `pipeline/frontend/yaml/matrix/matrix.go` | 47 |
| Matrix parseList (仅 include) | `pipeline/frontend/yaml/matrix/matrix.go` | 124 |
| Path 约束 Match (exclude→include) | `pipeline/frontend/yaml/constraint/path.go` | 100 |
| Path Includes (任一命中) | `pipeline/frontend/yaml/constraint/path.go` | 123 |
| Path Excludes (全量命中) | `pipeline/frontend/yaml/constraint/path.go` | 135 |
| Map 约束 Match (when.matrix) | `pipeline/frontend/yaml/constraint/map.go` | 27 |
| Registry 文件系统解析 | `server/services/registry/filesystem.go` | 39 |
| 队列 FIFO Extend | `server/queue/fifo.go` | 185 |
| 队列 FIFO resubmitExpired | `server/queue/fifo.go` | 338 |
| 队列 FIFO finished | `server/queue/fifo.go` | 133 |
| 队列 FIFO KickAgentWorkers | `server/queue/fifo.go` | 243 |
| Pipeline 批量取消 | `server/pipeline/cancel.go` | 42 |
| TaskTimeout 常量 | `shared/constant/constant.go` | 40 |
| Forge Refresh | `server/forge/refresh.go` | 58 |
| GitLab Refresh 实现 | `server/forge/gitlab/gitlab.go` | 160 |
| Bitbucket Refresh 实现 | `server/forge/bitbucket/bitbucket.go` | 124 |
