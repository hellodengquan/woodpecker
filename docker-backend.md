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

## 六、关键代码位置速查

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
