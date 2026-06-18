# Cron 与手动触发协作流程分析

## 一、整体架构概览

Woodpecker CI 存在四种触发路径，它们在不同入口分离，但在 Pipeline 创建层汇合：

```
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│  ① Cron 定时轮询  │ │ ② 手动触发 Cron   │ │ ③ 手动触发Pipeline │ │   ④ Webhook       │
│ server/cron/cron  │ │ server/api/cron   │ │ server/api/pipe   │ │ server/api/hook   │
└────────┬──────────┘ └────────┬──────────┘ └────────┬──────────┘ └────────┬──────────┘
         │ Cron.CreatePipeline()        │ Cron.CreatePipeline()  │ createTmpPipeline()│ Forge 构造
         ▼                             ▼                        ▼                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                            pipeline.Create() —— 公共汇合点                                │
│                           server/pipeline/create.go:35                                    │
└──────────────────────────────────────┬───────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
            ⑤ 状态落库 Created   ⑥ 拉取配置解析    ⑦ 取消重叠 Pipeline
            store.CreatePipeline  configService.Fetch   cancelPreviousPipelines
                                       │                  start.go:31
                                       ▼
                            ⑧ 状态流转 → Pending → Running
                               UpdateToStatusPending
                                       │
                                       ▼
                            ⑨ 入队执行 queuePipeline
```

---

## 二、三种触发入口详解

### 2.1 ① Cron 定时调度（自动触发）

**入口**：`server/cron/cron.go:41` — `Run()`

```go
func Run(ctx context.Context, store store.Store) error {
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-time.After(checkTime): // checkTime = 1 分钟
            go func() {
                now := time.Now()
                // 批量拉取到期的 cron（最多 10 条）
                crons, err := store.CronListNextExecute(now.Unix(), checkItems)
                for _, cron := range crons {
                    runCron(ctx, store, cron, now)
                }
            }()
        }
    }
}
```

**核心特征**：
- **轮询间隔**：每分钟检查一次 (`checkTime = time.Minute`)
- **批量大小**：每次最多处理 10 条 (`checkItems = 10`)
- **并发处理**：每条到期 cron 在 goroutine 中异步执行
- **查询条件**：`CronListNextExecute` JOIN repos 表，要求 `repos.active = true` 且 `crons.enabled = true`

---

### 2.2 ② 手动触发 Cron Job

**入口**：`server/api/cron.go:72` — `RunCron()`

```go
func RunCron(c *gin.Context) {
    // POST /repos/{repo_id}/cron/{cron_id}
    cron, err := _store.CronFind(repo, id)
    // ① 复用 cron.CreatePipeline() 构造 Pipeline 对象
    repo, newPipeline, err := cron_scheduler.CreatePipeline(c, _store, cron)
    // ② 走公共创建流程
    pl, err := pipeline.Create(c, _store, repo, newPipeline)
}
```

**关键差异**：
- **跳过锁机制**：不经过 `CronGetLock`，直接创建
- **不更新 NextExec**：cron 的下次执行时间不变，不会因此影响定时调度
- **复用构造逻辑**：与自动 Cron 共用 `CreatePipeline()` 构造 Pipeline 对象

---

### 2.3 ③ 手动触发 Pipeline（通用手动）

**入口**：`server/api/pipeline.go:50` — `CreatePipeline()`

```go
func CreatePipeline(c *gin.Context) {
    // POST /repos/{repo_id}/pipelines
    var opts model.PipelineOptions // {Branch, Variables}
    lastCommit, err := _forge.BranchHead(c, user, repo, opts.Branch)
    // ① 独立构造 Pipeline 对象（Event = manual）
    tmpPipeline := createTmpPipeline(model.EventManual, lastCommit, user, &opts)
    // ② 走公共创建流程
    pl, err := pipeline.Create(c, _store, repo, tmpPipeline)
}
```

**Pipeline 构造对比**：

| 字段 | Cron 触发 (EventCron) | 手动触发 (EventManual) |
|------|----------------------|----------------------|
| `Event` | `cron` | `manual` |
| `Message` | — (ToAPIModel 时赋值 Cron 名) | `MANUAL PIPELINE @ {branch}` |
| `Sender` | — (ToAPIModel 时赋值 Cron 名) | 用户名 |
| `Author` | — | 用户登录名 |
| `Cron` | cron 名称 | — |
| `Timestamp` | `cron.NextExec` | `time.Now()` |
| `Ref` | `refs/heads/{branch}` | `{branch}` (无前缀) |

---

## 三、公共汇合点：`pipeline.Create()`

**位置**：`server/pipeline/create.go:35`

所有触发路径最终汇聚至此，执行以下标准流程：

```
pipeline.Create()
  │
  ├─ 1. 获取 Repo Owner（用于 Forge 认证）
  ├─ 2. 检查 Skip CI（commit message 匹配 [CI SKIP] 等）
  ├─ 3. 刷新 Forge Token（forge.Refresh）
  │
  ├─ 4. 状态落库：StatusCreated ──────────────────────┐
  │    pipeline.Status = StatusCreated                 │
  │    store.CreatePipeline(pipeline)                  │
  │    - 事务内分配 Number (MAX(number)+1)             │ 数据库写入点 ①
  │    - 指数退避重试唯一键冲突（最多 3 次）            │
  │                                                    │
  ├─ 5. 拉取 Pipeline 配置（configService.Fetch）      │
  │    - 从 Forge 拉取 .woodpecker.yml                  │
  │    - 失败则标记 StatusError 并落库 ──────────────────┤
  │                                                    │
  ├─ 6. 解析配置 createPipelineItems()                 │
  │    - 校验 yaml、生成 workflow/step 结构             │
  │    - 解析失败 → StatusError ────────────────────────┤
  │    - 全部被条件过滤 → 删除记录并返回 ErrFiltered    │
  │                                                    │
  ├─ 7. 持久化配置 Config + 关联 PipelineConfig        │ 数据库写入点 ②
  │                                                    │
  ├─ 8. 发布状态变更（PubSub + Forge 通知）             │
  │                                                    │
  ├─ 9. 检查 Gated（需要审批则标记 StatusBlocked）      │ 数据库写入点 ③
  │    - 是 → 返回，不入队                              │
  │                                                    │
  ├─ 10. 状态流转：StatusCreated → StatusPending ──────┤
  │     UpdateToStatusPending()                        │
  │                                                    │
  └─ 11. start() → 取消重叠 + 入队执行
        │
        ├─ cancelPreviousPipelines()  ←─── 重叠保护
        ├─ publishPipeline()
        └─ queuePipeline()  ←─── 提交到调度队列
```

---

## 四、重叠保护机制（两层防护）

### 4.1 第一层：Cron 级 CAS 乐观锁

**位置**：`server/cron/cron.go:97` + `server/store/datastore/cron.go:64`

```go
// runCron() 中
gotLock, err := store.CronGetLock(cron, newNext.Unix())
if !gotLock {
    return nil // 另一个 goroutine 已经抢到了
}
```

**实现原理**（`CronGetLock`）：
```sql
UPDATE crons
SET next_exec = ?
WHERE id = ? AND next_exec = ?  -- 关键：使用旧值作为条件
```

- **原子性**：依赖数据库 UPDATE 的原子性
- **抢占式**：多个 goroutine 同时命中时，只有第一个 UPDATE 影响行数 != 0
- **副作用**：抢到锁的同时，已将 `NextExec` 更新为下次执行时间
- **手动触发 Cron**：**绕过此锁**，直接执行，不修改 NextExec

### 4.2 第二层：Pipeline 级主动取消

**位置**：`server/pipeline/start.go:31` → `cancel.go:100`

```go
func cancelPreviousPipelines(ctx, _forge, _store, pipeline, repo, user) error {
    // ① 检查该事件类型是否在取消名单中
    eventIncluded := slices.Contains(repo.CancelPreviousPipelineEvents, pipeline.Event)
    if !eventIncluded {
        return nil
    }

    // ② 获取所有活跃 Pipeline
    activeBuilds, err := _store.GetActivePipelineList(repo)

    // ③ 匹配规则（按事件类型区分）
    pipelineNeedsCancel := func(active *model.Pipeline) bool {
        if active.Event != pipeline.Event { return false } // 必须同类型
        switch pipeline.Event {
        case model.EventPush:
            return pipeline.Branch == active.Branch        // 同分支
        case model.EventCron:
            return pipeline.Cron == active.Cron            // 同 Cron 名
        case model.EventManual:
            return pipeline.Branch == active.Branch        // 同分支
        // ... 其他事件类型
        }
    }

    // ④ 逐个调用 Cancel() 取消匹配项
    for _, active := range activeBuilds {
        if pipelineNeedsCancel(active) { Cancel(...) }
    }
}
```

**关键要点**：
- 取消与否由 **Repo 配置**决定（`repo.CancelPreviousPipelineEvents`）
- Cron 之间：按 `Cron 名称` 匹配，同一名 Cron 的新执行会取消旧执行
- 手动触发之间：按 `Branch` 匹配，同分支新手动会取消旧手动
- Cron 与手动：**互不影响**，因为 `Event` 不同

---

## 五、状态落库全流程

### 5.1 Pipeline 状态枚举

**位置**：`server/model/const.go:57`

| 状态 | 含义 | 落库时机 |
|------|------|---------|
| `StatusCreated` | 已创建（内部） | `CreatePipeline` 首次写入 |
| `StatusBlocked` | 待审批 | Gate 检查不通过时 |
| `StatusPending` | 待执行 | `UpdateToStatusPending` |
| `StatusRunning` | 执行中 | Agent 开始执行时 |
| `StatusSuccess` | 成功 | 全部 Step 成功 |
| `StatusFailure` | 失败 | Step 退出码非 0 |
| `StatusError` | 错误 | 配置拉取/解析失败 |
| `StatusKilled` | 被终止 | 用户或系统取消已运行的 |
| `StatusCanceled` | 未启动取消 | 取消尚未开始执行的 |
| `StatusDeclined` | 审批被拒 | Blocked 后被拒绝 |
| `StatusSkipped` | 条件跳过 | 工作流条件不满足 |

### 5.2 关键落库函数

**位置**：`server/pipeline/pipeline_status.go`

| 函数 | 变更状态 | 附加字段 |
|------|---------|---------|
| `UpdateToStatusPending` | → `Pending` | `Reviewer`, `Reviewed` |
| `UpdateToStatusRunning` | → `Running` | `Started` |
| `UpdateStatusToDone` | → `Success/Failure` | `Finished` |
| `UpdateToStatusError` | → `Error` | `Errors`, `Started`, `Finished` |
| `UpdateToStatusKilled` | → `Killed/Canceled` | `Finished`, `CancelInfo` |
| `UpdateToStatusDeclined` | → `Declined` | `Reviewer`, `Reviewed` |

所有函数底层调用 `store.UpdatePipeline()` → `xorm.ID(id).AllCols().Update()`

### 5.3 数据写入时序

```
时间轴 →
  │
  ├─ [T1] store.CreatePipeline()
  │     ┌─────────────────────────────────┐
  │     │ pipelines 表 INSERT             │
  │     │   id, repo_id, number           │
  │     │   status = 'created'            │
  │     │   created = now()               │
  │     │   event, commit, branch, ...    │
  │     └─────────────────────────────────┘
  │
  ├─ [T2] 配置解析成功
  │     ┌─────────────────────────────────┐
  │     │ configs 表 INSERT               │
  │     │ pipeline_configs 表 INSERT      │
  │     └─────────────────────────────────┘
  │
  ├─ [T3] UpdateToStatusPending
  │     ┌─────────────────────────────────┐
  │     │ pipelines 表 UPDATE             │
  │     │   status = 'pending'            │
  │     │   updated = now()               │
  │     └─────────────────────────────────┘
  │
  ├─ [T4] Agent 拉取执行 → Running
  │
  └─ [T5] 完成 → Success/Failure/Killed
```

---

## 六、Cron 与手动触发的协作关系

### 6.1 调用关系图

```
cron_scheduler.Run()
    │
    ├─ store.CronListNextExecute()  ←── 从 DB 查到期 cron
    │
    └─ runCron()
         ├─ CalcNewNext()           ←── 算出下次执行时间
         ├─ store.CronGetLock()     ←── CAS 抢锁（原子 UPDATE next_exec）
         │    └─ ✅ 成功 → 更新内存中 cron.NextExec
         │    └─ ❌ 失败 → return（已被其他 goroutine 处理）
         │
         └─ cron_scheduler.CreatePipeline()
              │   构造 Event = 'cron' 的 Pipeline
              │
api/RunCron() ──────────────────────────┘
    │   POST /repos/{id}/cron/{id}
    │   └─ cron_scheduler.CreatePipeline()  ←── 复用相同构造逻辑
    │         （绕过 CronGetLock）
    │
api/CreatePipeline()
    │   POST /repos/{id}/pipelines
    └─ createTmpPipeline()  ←── 独立构造 Event='manual' 的 Pipeline
              │
              ▼
      ┌─────────────────────────────────────────────────┐
      │          pipeline.Create() —— 汇合点             │
      │   统一的创建、状态流转、入队逻辑                   │
      └─────────────────────────────────────────────────┘
```

### 6.2 独立性与交互总结

| 维度 | Cron 自动调度 | 手动触发 Cron | 手动触发 Pipeline |
|------|-------------|-------------|-----------------|
| **入口** | 定时轮询 | HTTP POST | HTTP POST |
| **CronGetLock** | ✅ 经过 | ❌ 绕过 | — |
| **NextExec 更新** | ✅ 抢占时更新 | ❌ 不更新 | — |
| **Event 类型** | `cron` | `cron` | `manual` |
| **Cron 字段** | 填充名称 | 填充名称 | 空 |
| **重叠取消匹配键** | `Cron 名` | `Cron 名` | `Branch` |
| **Cron 间重叠保护** | ✅ | ✅ 与自动 cron 互相取消 | ❌ 不影响 |
| **手动间重叠保护** | ❌ 不影响 | ❌ 不影响 | ✅ 同分支互相取消 |

### 6.3 协作场景举例

**场景 A**：Cron "nightly" 每分钟触发，上次执行还在跑
1. T1：Run() 轮询发现 nightly 到期，CronGetLock 成功 → 创建 Pipeline #101 (Event=cron, Cron=nightly)
2. T1 + 30s：Pipeline #101 仍在 Running
3. T2（下一分钟）：Run() 再次发现 nightly（NextExec 已推进，不会再命中 T1 那条）
   - 若手动调整了 NextExec 导致再次命中 → CronGetLock 中旧值不匹配 → 抢锁失败
4. 新 Pipeline #102 创建前 → `cancelPreviousPipelines` → 找到 #101 (同 Cron=nightly) → Cancel #101

**场景 B**：用户手动执行 Cron "nightly"，此时自动调度也正好触发
1. T0：用户点击 POST /repos/1/cron/5 → 走 RunCron → CreatePipeline → #201 (Running)
   - **不经过 CronGetLock**，NextExec 不变
2. T0 + 5s：Run() 轮询发现 NextExec <= now → 走 runCron
   - CronGetLock → 更新 NextExec 成功 → 创建 #202
   - start() → cancelPreviousPipelines → 匹配到 #201 (同 Cron=nightly, Event=cron) → Cancel #201

**场景 C**：用户同时触发手动 Pipeline + Cron 执行同一分支
1. 手动 Pipeline POST /repos/1/pipelines → #301 (Event=manual, Branch=main)
2. 紧接着 Cron 触发 → #302 (Event=cron, Cron=nightly, Branch=main)
3. cancelPreviousPipelines 检查：
   - #302 的 Event=cron，匹配 #301 的 Event=manual？→ **不匹配**
   - 结果：两个 Pipeline **并行执行，互不影响**

---

## 七、Cron 表达式解析与时区处理

### 7.1 第三方库选型

Woodpecker 使用 `github.com/gdgvda/cron v0.7.0`，这是 `robfig/cron` 的独立 fork，由 Google Developers Group Valle d'Aosta 维护。

**与 robfig/cron 的关键区别**：
- 新增 `Clock` 接口抽象，可注入自定义时钟（便于测试）
- `NewDefaultClock(location, nopTimer)` 支持在时钟层设置时区
- `DefaultSchedule.WithLocation(loc)` 可程序化修改已解析 Schedule 的时区
- 支持 `CRON_TZ=Asia/Tokyo` 前缀语法在单条表达式内覆盖时区
- 保留 `StandardOptions`（5 字段标准格式，不含秒）

### 7.2 解析挂载点

Cron 表达式的解析仅在 **两个地方** 触发，均调用同一个 `CalcNewNext()` 函数：

```
┌──────────────────────────────────────────────────────────────────┐
│  CalcNewNext(schedule, tzLoc, now)  ←  唯一解析入口              │
│  位置: server/cron/cron.go:68                                    │
│                                                                  │
│  内部流程:                                                       │
│    1. time.LoadLocation(tzLoc)     → 时区对象（失败则报错）       │
│    2. now = now.In(zone)           → 将基准时间转到目标时区       │
│    3. cron.NewDefaultParser(       → 创建标准 5 字段解析器        │
│         cron.StandardOptions)                                    │
│    4. parser.Parse(schedule)       → 解析表达式 → Schedule 对象   │
│    5. schedule.Next(now)           → 计算下一次触发时刻           │
│                                                                  │
│  返回: time.Time (带时区信息)                                     │
└──────────────────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
    ┌────┴────┐                    ┌────┴────┐
    │ 调用点 1 │                    │ 调用点 2 │
    │ runCron  │                    │ API CRUD │
    └─────────┘                    └─────────┘
```

#### 调用点 1：`runCron()` — 自动调度时

```go
// server/cron/cron.go:91
newNext, err := CalcNewNext(cron.Schedule, cron.Timezone, now)
```

- `cron.Schedule`：用户在创建时指定的表达式（如 `0 3 * * *`）
- `cron.Timezone`：用户指定的时区（如 `Europe/Bucharest`），默认 `UTC`
- `now`：当前服务器时间 `time.Now()`
- 用途：**计算 next_exec 并推进 CronGetLock**

#### 调用点 2：API CRUD — 创建/更新 Cron 时

```go
// server/api/cron.go:146 — PostCron (创建)
nextExec, err := cron_scheduler.CalcNewNext(cron.Schedule, cron.Timezone, time.Now())
cron.NextExec = nextExec.Unix()

// server/api/cron.go:226 — PatchCron (更新时区)
nextExec, err := cron_scheduler.CalcNewNext(cron.Schedule, tz, time.Now())
cron.NextExec = nextExec.Unix()

// server/api/cron.go:237 — PatchCron (更新 schedule)
nextExec, err := cron_scheduler.CalcNewNext(schedule, cron.Timezone, time.Now())
cron.NextExec = nextExec.Unix()

// server/api/cron.go:255 — PatchCron (重新启用)
nextExec, err := cron_scheduler.CalcNewNext(cron.Schedule, cron.Timezone, time.Now())
cron.NextExec = nextExec.Unix()
```

- 用途：**校验表达式合法性** + **计算首次 NextExec 存入数据库**
- 如果 `CalcNewNext` 返回错误，API 直接返回 400/422，Cron 不会被持久化

### 7.3 时区处理详解

#### 数据模型

```go
// server/model/cron.go
type Cron struct {
    Schedule  string  `xorm:"schedule NOT NULL"`               // 表达式
    Timezone  string  `xorm:"timezone NOT NULL DEFAULT 'UTC'"` // 时区名
    NextExec  int64   `xorm:"next_exec"`                       // Unix 时间戳（无时区）
}
```

- `Timezone` 存 IANA 时区名（如 `UTC`、`Asia/Shanghai`、`Europe/Bucharest`）
- `NextExec` 存 Unix 时间戳，**与时区无关**（绝对时刻）
- 创建时若 `Timezone` 为空，默认设为 `"UTC"`

#### 时区生效路径

```
用户创建 Cron (Schedule="0 3 * * *", Timezone="Asia/Shanghai")
    │
    ├─ API 调用 CalcNewNext("0 3 * * *", "Asia/Shanghai", time.Now())
    │     │
    │     ├─ time.LoadLocation("Asia/Shanghai") → *time.Location
    │     ├─ now = now.In(loc)                   → 转为上海时间
    │     ├─ parser.Parse("0 3 * * *")           → Schedule 对象
    │     └─ schedule.Next(now)                  → 上海时间 03:00 对应的 UTC 时刻
    │
    └─ NextExec = result.Unix()  ← 存入数据库的是绝对时间戳
                                           (如 1661979600)

运行时 runCron():
    │
    ├─ CronListNextExecute(now.Unix())  ← 用 UTC 时间戳比较
    │     → 找到 NextExec <= now 的 Cron
    │
    └─ CalcNewNext(cron.Schedule, cron.Timezone, now)
          → 基于上海时区计算下一个 03:00 → 新的绝对时间戳
          → CronGetLock 更新 NextExec
```

#### 时区边界行为

| 场景 | 行为 |
|------|------|
| DST 向前跳（春季） | 跳过不存在的时间点，直接到下一个有效时刻 |
| DST 向后跳（秋季） | gdgvda/cron 会选择第一次出现的时刻 |
| 无效时区名 | `time.LoadLocation` 返回错误，Cron 创建/更新被拒绝 |
| `@every` 间隔格式 | 不受时区影响，始终从当前时刻加固定间隔 |

#### 迁移：秒字段移除

`migration/011_cron_without_sec.go` 将旧版 6 字段（含秒）表达式截断为 5 字段：

```go
// 对非预定义表达式，去掉第一个空格前的秒字段
if !strings.HasPrefix(schedule, "@") {
    cron.Schedule = strings.SplitN(strings.TrimSpace(cron.Schedule), " ", 2)[1]
}
```

这确保 Woodpecker 统一使用 `StandardOptions`（5 字段），不包含秒级精度。

---

## 八、分布式多 Server 部署下的重叠保护

### 8.1 Woodpecker 的部署假设

**关键发现：Woodpecker 不包含任何 leader 选举或分布式协调机制。**

代码中搜索 `leader`、`election`、`raft`、`consul`、`etcd`、`SELECT FOR UPDATE` 等关键词，均未找到相关实现。`cron_scheduler.Run()` 在每个 Server 进程中无条件启动：

```go
// cmd/server/server.go:127
serviceWaitingGroup.Go(func() error {
    log.Info().Msg("starting cron service ...")
    if err := cron_scheduler.Run(ctx, _store); err != nil {
        go stopServerFunc(err)
        return err
    }
    return nil
})
```

**这意味着：多 Server 实例部署时，每个实例都会运行独立的 Cron 轮询循环。**

### 8.2 多 Server 下的竞争时序

```
Server A                          Server B
  │                                 │
  ├─ Run() 每分钟轮询               ├─ Run() 每分钟轮询
  │  CronListNextExecute()          │  CronListNextExecute()
  │  → 返回 cron{ID:5, NextExec:T} │  → 返回 cron{ID:5, NextExec:T}
  │                                 │
  ├─ runCron(cron, now)             ├─ runCron(cron, now)
  │  ├─ CalcNewNext → newNext       │  ├─ CalcNewNext → newNext
  │  ├─ CronGetLock(ID:5, T, newNext)│  ├─ CronGetLock(ID:5, T, newNext)
  │  │                              │  │
  │  │  ┌──────── RACE ────────┐   │  │
  │  │  │ UPDATE crons         │   │  │
  │  │  │ SET next_exec=newNext│   │  │
  │  │  │ WHERE id=5           │   │  │
  │  │  │   AND next_exec=T    │   │  │
  │  │  │                      │   │  │
  │  │  │ 只有1个能影响0行≠0    │   │  │
  │  │  └──────────────────────┘   │  │
  │  │                              │  │
  │  ├─ ✅ cols=1, gotLock=true     │  ├─ ❌ cols=0, gotLock=false
  │  ├─ CreatePipeline()            │  ├─ return nil ← 放弃
  │  └─ pipeline.Create()           │  └─ (另一个goroutine处理了)
  │                                 │
```

### 8.3 CronGetLock 的跨数据库原子性分析

`CronGetLock` 通过 xorm 生成如下 SQL：

```go
// server/store/datastore/cron.go:64
cols, err := s.engine.ID(cron.ID).Where(builder.Eq{"next_exec": cron.NextExec}).
    Cols("next_exec").Update(&model.Cron{NextExec: newNextExec})
```

等效 SQL：
```sql
UPDATE crons SET next_exec = ? WHERE id = ? AND next_exec = ?
```

**各数据库引擎的原子性保证**：

| 数据库 | UPDATE 原子性 | 并发安全性 | 备注 |
|--------|-------------|-----------|------|
| **PostgreSQL** | ✅ MVCC 行级锁 | ✅ 安全 | UPDATE 对同一行串行化；WHERE 条件在最新快照求值 |
| **MySQL (InnoDB)** | ✅ 行级排他锁 | ✅ 安全 | UPDATE 获取 X 锁；第二个事务阻塞直到第一个提交，然后 WHERE 不匹配 |
| **SQLite** | ⚠️ 文件级锁 | ✅ 安全（单写者） | WAL 模式下读不阻塞写，但写仍串行；CAS 语义成立 |

**核心机制**：无论哪种引擎，`UPDATE ... WHERE next_exec = old_value` 都保证了：
1. **原子性**：读-改-写作为单条 SQL 原子执行
2. **互斥**：只有一个 UPDATE 能匹配到旧值并返回 `affected rows = 1`
3. **无害失败**：其他竞争者得到 `affected rows = 0`，安全退出

### 8.4 缺失的保护：多 Server 部署的隐患

虽然 `CronGetLock` 解决了"同一 Cron 任务不会被多个 Server 同时执行"的问题，但多 Server 场景下仍存在以下隐患：

#### 隐患 1：Pipeline 创建非幂等

`CronGetLock` 成功后，`CreatePipeline()` → `pipeline.Create()` 链路中：

```
CronGetLock 成功
    → cron_scheduler.CreatePipeline()  ← 构造 Pipeline 对象
    → pipeline.Create()                ← 状态落库 + 入队
```

如果 `CronGetLock` 成功但 `pipeline.Create()` 因网络/Forge 故障失败：
- `NextExec` 已被推进到下次时间 ✅
- 但本次 Pipeline 未创建 ❌
- **结果：跳过一次执行，无法自动补偿**

#### 隐患 2：CronListNextExecute 的批量竞争

```go
// 每分钟每 Server 都执行此查询
crons, err := store.CronListNextExecute(now.Unix(), checkItems)
```

如果到期 Cron 超过 `checkItems`（10 条）：
- 多个 Server 可能拿到**不同子集**的到期 Cron
- 某些 Cron 可能被**所有 Server 忽略**（恰好不在任何子集中）
- 但下一轮轮询会重新拾取，**最多延迟 1 分钟**

#### 隐患 3：手动触发 Cron 无锁

`api/RunCron()` 不经过 `CronGetLock`，这意味着：
- 用户通过 API 手动触发 Cron Job 时，不检查 NextExec
- 手动触发的 Pipeline 与自动触发的 Pipeline 使用 `cancelPreviousPipelines` 互斥
- 但如果用户在短时间内多次点击"手动触发"，**可能创建多个并行的 Cron Pipeline**
- 并行 Pipeline 的去重依赖 `cancelPreviousPipelines`（需 Repo 配置 `CancelPreviousPipelineEvents` 包含 `cron` 事件）

### 8.5 为什么不需要 Leader 选举

Woodpecker 的设计选择是 **数据库 CAS 锁** 而非 **Leader 选举**，理由如下：

| 维度 | Leader 选举 | 数据库 CAS（当前实现） |
|------|-----------|---------------------|
| 部署复杂度 | 需要 etcd/Consul 或自定义协议 | 零额外依赖 |
| 故障恢复 | 需要心跳 + 选举超时 | 天然容错：任何 Server 可抢占 |
| 一致性 | 强一致（同一时刻只有1个调度器） | 最终一致（最多重复1轮，由 CAS 去重） |
| 适用场景 | 对"恰好一次"语义要求极高 | 允许偶发跳过/延迟 |
| 代码复杂度 | 高（状态机、选举协议） | 低（1条 UPDATE 语句） |

**Woodpecker 的定位是 CI/CD 工具**，Cron 调度容忍分钟级延迟，无需强一致调度。数据库 CAS 是最简方案。

### 8.6 完整的多 Server 并发时序图

```
时间轴 →

Server A (轮询周期 T)                Server B (轮询周期 T)
    │                                    │
    ├─ CronListNextExecute(T)            ├─ CronListNextExecute(T)
    │  → [cron_1, cron_2, ...]           │  → [cron_1, cron_2, ...]
    │                                    │
    ├─ runCron(cron_1) ──────┐           ├─ runCron(cron_1) ──────┐
    │  ├─ CalcNewNext()      │           │  ├─ CalcNewNext()      │
    │  ├─ CronGetLock ───────┼─── RACE ──┤  ├─ CronGetLock ───────┤
    │  │  UPDATE next_exec   │   ON DB   │  │  UPDATE next_exec   │
    │  │  ✅ rows=1          │           │  │  ❌ rows=0          │
    │  ├─ CreatePipeline()   │           │  └─ return nil         │
    │  └─ pipeline.Create()  │           │                        │
    │                         │           │                        │
    ├─ runCron(cron_2) ──────┤           ├─ runCron(cron_2) ──────┤
    │  ├─ CronGetLock ───────┼─── RACE ──┤  ├─ CronGetLock ───────┤
    │  │  ❌ rows=0          │           │  │  ✅ rows=1          │
    │  └─ return nil         │           │  ├─ CreatePipeline()   │
    │                         │           │  └─ pipeline.Create()  │
    │                                    │
    └─ 等待下一轮 (T+1min)               └─ 等待下一轮 (T+1min)
```

**核心保证**：同一个 Cron Job 在同一个执行周期内，**只有一个 Server 能成功创建 Pipeline**。

---

## 十、Cron 触发失败的重试与退避策略

### 10.1 总览：三层失败点，两种重试策略

```
runCron() 调用链
    │
    ├─ [F1] CalcNewNext()        ← 失败点 1：表达式/时区错误
    │   └─ 无重试，直接返回错误
    │
    ├─ [F2] CronGetLock()        ← 失败点 2：抢锁失败
    │   └─ 无需重试，正常退出（已被其他 goroutine 处理）
    │
    ├─ [F3] cron_scheduler.CreatePipeline()  ← 失败点 3a：Forge 网络错误
    │   ├─ store.GetRepo()       → 无重试
    │   ├─ ForgeFromRepo()       → 无重试
    │   ├─ store.GetUser()       → 无重试
    │   ├─ forge.Refresh()       → 无重试
    │   └─ _forge.BranchHead()   → 无重试
    │
    └─ [F4] pipeline.Create()    ← 失败点 3b：Pipeline 创建流程
        ├─ store.CreatePipeline() → ✅ 指数退避重试（最多 3 次）
        ├─ configService.Fetch()  → 无重试（失败 → StatusError）
        ├─ createPipelineItems()  → 无重试（失败 → StatusError）
        └─ start()/queuePipeline() → 无重试
```

### 10.2 Cron 调度层：无重试，仅记日志

```go
// server/cron/cron.go:58
for _, cron := range crons {
    if err := runCron(ctx, store, cron, now); err != nil {
        log.Error().Err(err).Int64("cronID", cron.ID).Msg("run cron failed")
        // ← 仅记录日志，不重试，不回滚 NextExec
    }
}
```

**各失败场景的后果**：

| 失败点 | NextExec 状态 | Pipeline 状态 | 后果 | 是否自动恢复 |
|--------|-------------|-------------|------|------------|
| `CalcNewNext` 失败 | 未修改（仍为旧值） | 不存在 | 下分钟重新命中 | ✅ 1 分钟后重试 |
| `CronGetLock` 抢锁失败 | 已被别人推进 | 由别人创建 | 正常，无影响 | — |
| `CronGetLock` 本身报错 | 未修改 | 不存在 | 下分钟重新命中 | ✅ 1 分钟后重试 |
| `CreatePipeline` 中 Forge 失败 | **已推进** | 不存在 | **永久跳过一次执行** | ❌ 不会补偿 |
| `pipeline.Create` 配置拉取失败 | **已推进** | StatusError（已落库） | 算作一次执行，有记录 | ✅ 可人工 Restart |
| `pipeline.Create` 入队失败 | **已推进** | StatusPending | Pipeline 卡在 Pending | ⚠️ 需人工干预 |

### 10.3 唯一的重试点：`store.CreatePipeline` 的指数退避

**位置**：`server/store/datastore/pipeline.go:135`

这是整条链路中**唯一带有重试机制**的环节，使用 `cenkalti/backoff/v5`：

```go
func (s storage) CreatePipeline(pipeline *model.Pipeline, stepList ...*model.Step) error {
    const maxRetries = 3
    exponentialBackoff := backoff.NewExponentialBackOff()

    _, err := backoff.Retry(context.Background(), func() (struct{}, error) {
        sess := s.engine.NewSession()
        defer sess.Close()
        if err := sess.Begin(); err != nil {
            return struct{}{}, err
        }

        // 事务内：查 MAX(number) + INSERT pipeline
        var number int64
        if _, err := sess.Select("MAX(number)").
            Table(new(model.Pipeline)).
            Where("repo_id = ?", pipeline.RepoID).
            Get(&number); err != nil {
            return struct{}{}, err
        }
        pipeline.Number = number + 1

        if err := wrapInsert(sess.Insert(pipeline)); err != nil {
            if isUniqueConstraintError(err) {
                return struct{}{}, err       // ← 唯一键冲突：可重试
            }
            return struct{}{}, backoff.Permanent(err)  // ← 其他错误：永久失败
        }

        return struct{}{}, sess.Commit()
    }, backoff.WithBackOff(exponentialBackoff), backoff.WithMaxTries(maxRetries))
    return err
}
```

**退避参数**（`backoff.NewExponentialBackOff()` 默认值）：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `InitialInterval` | 500ms | 首次重试等待 |
| `RandomizationFactor` | 0.5 | ±50% 随机抖动 |
| `Multiplier` | 1.5 | 每次间隔 ×1.5 |
| `MaxInterval` | 60s | 单次最大等待 |
| `MaxElapsedTime` | 15min | 总最大等待 |

**实际重试间隔估算**：
- 第 1 次重试：~500ms（±250ms）
- 第 2 次重试：~750ms（±375ms）
- 第 3 次重试：~1125ms（±562ms）

**为什么需要这个重试**：

高并发下多个 Pipeline 同时创建时，`SELECT MAX(number) + INSERT` 事务之间会竞争 `(repo_id, number)` 唯一约束：

```
Webhook 触发 Pipeline A               Cron 触发 Pipeline B
    │                                      │
    ├─ BEGIN                               ├─ BEGIN
    ├─ SELECT MAX(number) → 100            ├─ SELECT MAX(number) → 100
    ├─ INSERT number=101                   ├─ INSERT number=101  ← 💥 唯一键冲突
    ├─ COMMIT                              ├─ ROLLBACK
    │                                      │
    │                                      ├─ backoff 等待 ~500ms
    │                                      ├─ BEGIN
    │                                      ├─ SELECT MAX(number) → 101  ← 已提交
    │                                      ├─ INSERT number=102  ✅
    │                                      └─ COMMIT
```

`isUniqueConstraintError` 检测 5 种数据库的唯一键错误模式：

```go
func isUniqueConstraintError(err error) bool {
    errStr := err.Error()
    return strings.Contains(errStr, "duplicate key")        // PostgreSQL
        || strings.Contains(errStr, "Duplicate entry")       // MySQL
        || strings.Contains(errStr, "UNIQUE constraint failed") // SQLite
        || strings.Contains(errStr, "unique constraint")     // 通用
        || strings.Contains(errStr, "UNIQUE violation")      // 通用
}
```

**关键**：只有唯一键冲突被判定为**可重试错误**，其他 INSERT 错误通过 `backoff.Permanent(err)` 标记为**永久失败**，不再重试。

### 10.4 pipeline.Create 内部的失败处理：StatusError 落库

当 `pipeline.Create()` 在配置拉取或解析阶段失败时，Pipeline 不会被静默丢弃，而是**标记为 StatusError 并持久化**：

```go
// server/pipeline/create.go:78-94 — 配置拉取失败
case configFetchErr != nil && forgeYamlConfigs != nil:
    // Forge 返回异常状态码但有旧配置 → 降级使用旧配置，仅 Warn
    log.Warn().Err(configFetchErr).Msg("will fallback to old config")

case configFetchErr != nil:
    // Forge 返回异常且无旧配置 → StatusError 落库
    return nil, updatePipelineWithErr(ctx, _forge, _store, pipeline, repo, repoUser, ...)
```

```go
// server/pipeline/create.go:96-101 — YAML 解析失败
if handleParseErrors(pipeline, parseErr) {
    return pipeline, updatePipelineWithErr(ctx, _forge, _store, pipeline, repo, repoUser, parseErr)
}
```

`updatePipelineWithErr` 调用 `UpdateToStatusError`：
```go
func UpdateToStatusError(store store.Store, pipeline model.Pipeline, err error) (*model.Pipeline, error) {
    pipeline.Errors = errors.GetPipelineErrors(err)
    pipeline.Status = model.StatusError
    pipeline.Started = time.Now().Unix()
    pipeline.Finished = pipeline.Started    // 瞬时完成
    return &pipeline, store.UpdatePipeline(&pipeline)
}
```

**效果**：失败的 Cron Pipeline 在 UI 中可见（Status=error），可被 `pipeline.Restart` 重新触发。

### 10.5 ErrFiltered：静默丢弃，无重试

以下场景返回 `ErrFiltered`，Pipeline 被删除或不创建，**不重试**：

| 场景 | 代码位置 | 行为 |
|------|---------|------|
| commit message 含 `[CI SKIP]` | `create.go:43` | 返回 `ErrFiltered`，Pipeline 未创建 |
| 配置文件不存在 | `create.go:82` | 先删除已创建记录，再返回 `ErrFiltered` |
| 全部 workflow 被 when 条件过滤 | `create.go:108` | 先删除已创建记录，再返回 `ErrFiltered` |

API 层对 `ErrFiltered` 的处理：
```go
// server/api/helper.go:38
case errors.Is(err, pipeline.ErrFiltered):
    c.Writer.Header().Add("Pipeline-Filtered", "true")
    c.Status(http.StatusNoContent)
```

Cron 层对此的处理：
```go
// server/cron/cron.go:111
_, err = pipeline.Create(ctx, store, repo, newPipeline)
return err  // ErrFiltered 会被 log.Error 记录，但不影响下次调度
```

### 10.6 完整的失败-恢复决策树

```
runCron() 失败
    │
    ├─ CalcNewNext 错误
    │   └─ NextExec 未变 → 下轮自动重试 ✅
    │
    ├─ CronGetLock 失败
    │   ├─ 抢锁失败 (gotLock=false) → 正常，别人已处理 ✅
    │   └─ DB 错误 → NextExec 未变 → 下轮自动重试 ✅
    │
    ├─ cron_scheduler.CreatePipeline 错误
    │   ├─ DB 错误 (GetRepo/GetUser) → NextExec 已推 → 跳过 ❌
    │   └─ Forge 错误 (BranchHead) → NextExec 已推 → 跳过 ❌
    │
    └─ pipeline.Create 错误
        ├─ CreatePipeline 唯一键冲突 → 自动退避重试 ✅
        ├─ CreatePipeline 其他错误 → Permanent 失败 → 跳过 ❌
        ├─ 配置拉取失败 (无旧配置) → StatusError 落库 → 可 Restart ✅
        ├─ 配置拉取失败 (有旧配置) → 降级使用旧配置 → 继续 ✅
        ├─ YAML 解析失败 → StatusError 落库 → 可 Restart ✅
        ├─ ErrFiltered → Pipeline 被删除 → 不可 Restart ❌
        └─ 入队失败 → Pipeline 卡在 Pending → 需人工 ⚠️
```

---

## 十一、Webhook 触发与 Cron 触发的并发路径

### 11.1 第四种触发入口：Webhook

**入口**：`server/api/hook.go:69` — `PostHook()`

```go
func PostHook(c *gin.Context) {
    // POST /hook

    // 1. 验证 webhook token
    _, err := token.ParseRequest([]token.Type{token.HookToken}, c.Request, ...)

    // 2. 解析 webhook 数据
    repoFromForge, pipelineFromForge, err := _forge.Hook(c, c.Request)

    // 3. 校验 repo + 用户
    if !repo.IsActive { return }
    forge.Refresh(c, _forge, _store, user)

    // 4. 更新 repo 信息
    repo.Update(repoFromForge)
    _store.UpdateRepo(repo)

    // 5. 创建 Pipeline
    pl, err := pipeline.Create(c, _store, repo, pipelineFromForge)
}
```

**Webhook 产生的 Event 类型**：

| Forge 事件 | Pipeline Event | Refspec 字段 |
|-----------|---------------|-------------|
| push 到分支 | `push` | 空（Branch 填充） |
| push 标签 | `tag` | 空 |
| Pull Request 创建/更新 | `pull_request` | `source:target` |
| PR 关闭 | `pull_request_closed` | `source:target` |
| Release | `release` | 空 |
| Deployment | `deployment` | 空 |

**与 Cron/手动触发的关键区别**：
- Webhook Pipeline 由 Forge 构造，**包含完整的 commit 信息**（message、author 等）
- Event 类型可以是 `push`、`tag`、`pull_request` 等，但**永远不会是 `cron` 或 `manual`**
- Webhook 是**外部驱动**的，频率不可控

### 11.2 四种触发路径的完整并发视图

```
┌──────────────┐ ┌───────────────┐ ┌───────────────┐ ┌──────────────┐
│  Webhook     │ │ Cron 自动调度  │ │ 手动触发 Cron  │ │ 手动触发     │
│  POST /hook  │ │ cron.Run()    │ │ POST /cron/{id}│ │ POST /pipes  │
│  Event: push │ │ Event: cron   │ │ Event: cron    │ │ Event: manual│
└──────┬───────┘ └──────┬────────┘ └──────┬────────┘ └──────┬───────┘
       │                │                 │                 │
       └────────────────┴─────────────────┴─────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │   pipeline.Create()      │  ← 四路汇合
                    │   公共创建流程            │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────┴─────────────┐
                    │  store.CreatePipeline()  │  ← 唯一键竞争点
                    │  (repo_id, number) UNIQUE│
                    │  指数退避重试 ×3          │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────┴─────────────┐
                    │ cancelPreviousPipelines()│  ← 事件隔离的取消
                    │ 匹配规则：同 Event +      │
                    │   push→同 Branch          │
                    │   cron→同 Cron 名         │
                    │   manual→同 Refspec       │
                    │   其他→同 Refspec         │
                    └──────────────────────────┘
```

### 11.3 并发竞争核心：`store.CreatePipeline` 的 number 分配

**四种触发源可能同时为同一 Repo 创建 Pipeline**，它们在 `(repo_id, number)` 唯一约束上竞争：

```
Webhook (push main)              Cron (nightly, main)           手动 Pipeline (main)
    │                                 │                              │
    ├─ pipeline.Create()              ├─ pipeline.Create()           ├─ pipeline.Create()
    │  ├─ CreatePipeline()            │  ├─ CreatePipeline()         │  ├─ CreatePipeline()
    │  │  ├─ BEGIN                    │  │  ├─ BEGIN                 │  │  ├─ BEGIN
    │  │  ├─ MAX(number)→100          │  │  ├─ MAX(number)→100       │  │  ├─ MAX(number)→100
    │  │  ├─ INSERT #101 ✅           │  │  ├─ INSERT #101 💥        │  │  ├─ INSERT #101 💥
    │  │  └─ COMMIT                   │  │  └─ ROLLBACK              │  │  └─ ROLLBACK
    │  │                              │  │                           │  │
    │  │                              │  │  ← backoff ~500ms →      │  │  ← backoff ~500ms →
    │  │                              │  │  ├─ BEGIN                 │  │  ├─ BEGIN
    │  │                              │  │  ├─ MAX(number)→101       │  │  ├─ MAX(number)→101 💥
    │  │                              │  │  ├─ INSERT #102 ✅        │  │  ├─ INSERT #102 💥
    │  │                              │  │  └─ COMMIT                │  │  └─ ROLLBACK
    │  │                              │  │                           │  │
    │  │                              │  │                           │  │  ← backoff ~750ms →
    │  │                              │  │                           │  │  ├─ BEGIN
    │  │                              │  │                           │  │  ├─ MAX(number)→102
    │  │                              │  │                           │  │  ├─ INSERT #103 ✅
    │  │                              │  │                           │  │  └─ COMMIT
```

**为什么用 `MAX(number)+1` 而非自增**：
- `number` 是**Repo 范围内**的流水号（非全局自增 ID `id`）
- 用户通过 `repo/repo-id/pipelines/103` 访问，需要连续且有意义的编号
- 全局 `id` 是数据库自增主键，对用户不可见

**退避重试的边界**：
- 最多 3 次重试，在高并发下仍有耗尽重试的可能
- 3 次重试仍冲突 → `CreatePipeline` 返回唯一键错误 → `pipeline.Create` 返回错误
- 对 Webhook：API 返回 500，Forge 可能重试 webhook（取决于 Forge 实现）
- 对 Cron：`runCron` 记录日志，**NextExec 已推，跳过执行**

### 11.4 `cancelPreviousPipelines` 的事件隔离机制

**关键代码**：`server/pipeline/cancel.go:120-133`

```go
pipelineNeedsCancel := func(active *model.Pipeline) bool {
    if active.Event != pipeline.Event {   // ← 必须同 Event 才可能取消
        return false
    }
    switch pipeline.Event {
    case model.EventPush:
        return pipeline.Branch == active.Branch   // push: 同分支
    default:
        return pipeline.Refspec == active.Refspec  // 其他: 同 Refspec
    }
}
```

**各 Event 的取消匹配规则**：

| 新 Pipeline Event | 匹配条件 | 会被取消的旧 Pipeline |
|------------------|---------|---------------------|
| `push` | 同 Branch | 同分支的 `push` Pipeline |
| `cron` | 同 Refspec（Cron 名存于 Refspec？） | 同 Refspec 的 `cron` Pipeline |
| `manual` | 同 Refspec | 同 Refspec 的 `manual` Pipeline |
| `pull_request` | 同 Refspec (`source:target`) | 同 PR 的 `pull_request` Pipeline |
| `tag` | 同 Refspec | 同 tag 的 `tag` Pipeline |

**注意**：Cron Event 走的是 `default` 分支，用 `Refspec` 匹配。Cron Pipeline 构造时 `Refspec` 为空字符串（只有 `Branch` 和 `Ref` 字段被填充），所以**同一 Repo 的所有 Cron Pipeline 会互相取消**（因为 Refspec 都是空）。这与之前文档中"按 Cron 名匹配"的描述有差异——实际代码行为取决于 `Refspec` 字段值。

### 11.5 Webhook 与 Cron 并发的四个典型场景

#### 场景 A：同一分支的 push 和 Cron 同时触发

```
T0: Git push 到 main → Webhook → pipeline.Create() → Pipeline #50 (Event=push, Branch=main)
T0+1s: Cron "nightly" 到期 → runCron → pipeline.Create() → Pipeline #51 (Event=cron, Branch=main)

cancelPreviousPipelines:
  - #51 是 Event=cron，查找活跃 Pipeline 中 Event=cron 且 Refspec 相同的
  - #50 是 Event=push → 不匹配 → 不会被取消

结果: #50 (push) 和 #51 (cron) 并行执行 ✅
```

#### 场景 B：快速连续 push 触发

```
T0: push main → Webhook → Pipeline #60 (Event=push, Branch=main, Status=Pending)
T0+2s: 另一次 push main → Webhook → Pipeline #61 (Event=push, Branch=main)

cancelPreviousPipelines (Repo 配置 push 在取消名单中):
  - #61 是 Event=push，查找活跃 Pipeline 中 Event=push 且 Branch=main 的
  - #60 匹配 → Cancel #60 (SupersededBy: 61)

结果: #60 被取消，只有 #61 执行 ✅
```

#### 场景 C：Cron 和 push 争抢同一 Branch 的 Pipeline number

```
T0: push main + Cron "nightly" 同时触发
    两者几乎同时调用 store.CreatePipeline()

    Webhook goroutine:                Cron goroutine:
    BEGIN                              BEGIN
    MAX(number) → 70                   MAX(number) → 70
    INSERT #71 ✅                      INSERT #71 💥 unique conflict
    COMMIT                             ROLLBACK
                                       ← backoff ~500ms →
                                       BEGIN
                                       MAX(number) → 71
                                       INSERT #72 ✅
                                       COMMIT

    Pipeline #71: Event=push (先创建)
    Pipeline #72: Event=cron (后退避创建)

    cancelPreviousPipelines:
      #72 (cron) 不会取消 #71 (push) → 并行执行 ✅
```

#### 场景 D：PR Webhook 和 Cron 交错

```
T0: PR #42 opened → Webhook → Pipeline #80 (Event=pull_request, Refspec="feature:main")
T0+5s: Cron "nightly" 触发 → Pipeline #81 (Event=cron, Refspec="")

cancelPreviousPipelines:
  #81 (cron) 查找 Event=cron 且 Refspec="" 的 → 无匹配（#80 是 pull_request）
  #80 (pull_request) 不会被取消

结果: PR Pipeline 和 Cron Pipeline 并行 ✅
```

### 11.6 并发安全总结

| 并发对 | 是否互斥 | 保护机制 | 潜在问题 |
|--------|---------|---------|---------|
| Webhook vs Webhook | 按事件+分支 | `cancelPreviousPipelines` + 唯一键退避 | 无 |
| Webhook vs Cron | **不互斥** | 事件类型不同，不会互相取消 | 同分支并行执行可能冲突 |
| Webhook vs Manual | **不互斥** | 事件类型不同 | 同分支并行 |
| Cron vs Cron | 按事件+Refspec | `CronGetLock` + `cancelPreviousPipelines` | Refspec 为空可能误杀其他 Cron |
| Manual vs Manual | 按事件+Refspec | `cancelPreviousPipelines` + 唯一键退避 | 无 |
| Cron vs Manual | **不互斥** | 事件类型不同 | 同分支并行 |

---

## 十二、关键文件索引

| 文件 | 作用 |
|------|------|
| `server/cron/cron.go` | Cron 调度主循环、`CalcNewNext`、`runCron`、`CreatePipeline` 构造 |
| `server/cron/cron_test.go` | `TestCalcNewNext` — 时区解析测试（UTC vs Europe/Bucharest） |
| `server/api/cron.go` | Cron REST API：RunCron（手动触发 cron）、CRUD（含 CalcNewNext 调用点） |
| `server/api/hook.go` | Webhook 入口：PostHook — 解析 Forge webhook 并调用 pipeline.Create |
| `server/api/pipeline.go` | Pipeline REST API：CreatePipeline（手动触发） |
| `server/api/helper.go` | `handlePipelineErr` — API 层错误分类（ErrFiltered/ErrNotFound/ErrBadRequest） |
| `server/pipeline/create.go` | Pipeline 创建主流程（四方汇合点），含 StatusError 落库 |
| `server/pipeline/start.go` | start()、cancelPreviousPipelines 调用入口 |
| `server/pipeline/cancel.go` | cancelPreviousPipelines 具体实现（事件隔离 + Cancel 逻辑） |
| `server/pipeline/pipeline_status.go` | 状态更新函数（含 UpdateToStatusError） |
| `server/pipeline/errors.go` | `ErrFiltered` — Pipeline 被条件过滤时的错误类型 |
| `server/pipeline/restart.go` | Restart — 重启 StatusError 的 Pipeline |
| `server/store/datastore/cron.go` | CronListNextExecute、CronGetLock（CAS 乐观锁） |
| `server/store/datastore/cron_test.go` | CronGetLock 测试（抢锁成功/失败场景） |
| `server/store/datastore/pipeline.go` | CreatePipeline（事务+指数退避重试）、isUniqueConstraintError |
| `server/store/datastore/init_cgo.go` | 支持的数据库驱动（sqlite3 + mysql + postgres） |
| `server/store/datastore/init.go` | 非 cgo 构建的驱动（mysql + postgres only） |
| `server/store/datastore/migration/011_cron_without_sec.go` | 秒字段迁移：6 字段 → 5 字段 |
| `server/store/datastore/migration/027_add_cron_field.go` | Pipeline 增加 Cron 字段迁移 |
| `server/model/cron.go` | Cron 数据结构 + Validate（内含 cron 表达式校验） |
| `server/model/pipeline.go` | Pipeline 数据结构（含 Cron、Refspec 字段） |
| `server/model/const.go` | WebhookEvent、StatusValue 枚举 |
| `server/model/repo.go` | Repo 数据结构（含 CancelPreviousPipelineEvents 配置） |
| `cmd/server/server.go` | Server 启动入口，Cron 调度在 errgroup 中无条件启动 |
| `go.mod` | `github.com/gdgvda/cron v0.7.0`、`github.com/cenkalti/backoff/v5` |
