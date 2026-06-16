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

## 五、关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
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
