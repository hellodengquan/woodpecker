# Cron 与手动触发协作流程分析

## 一、整体架构概览

Woodpecker CI 存在三种触发路径，它们在不同入口分离，但在 Pipeline 创建层汇合：

```
┌───────────────────────┐     ┌───────────────────────┐     ┌───────────────────────┐
│   ① Cron 定时轮询     │     │  ② 手动触发 Cron Job  │     │  ③ 手动触发 Pipeline  │
│ server/cron/cron.go   │     │   server/api/cron.go  │     │ server/api/pipeline.go│
└───────────┬───────────┘     └───────────┬───────────┘     └───────────┬───────────┘
            │ Cron.CreatePipeline()       │ Cron.CreatePipeline()       │ createTmpPipeline()
            ▼                             ▼                             ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          ④ pipeline.Create() —— 公共汇合点                             │
│                         server/pipeline/create.go:35                                  │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
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

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| `server/cron/cron.go` | Cron 调度主循环、runCron、CreatePipeline 构造 |
| `server/api/cron.go` | Cron REST API：RunCron（手动触发 cron）、CRUD |
| `server/api/pipeline.go` | Pipeline REST API：CreatePipeline（手动触发） |
| `server/pipeline/create.go` | Pipeline 创建主流程（三方汇合点） |
| `server/pipeline/start.go` | start()、cancelPreviousPipelines 调用入口 |
| `server/pipeline/cancel.go` | cancelPreviousPipelines 具体实现 |
| `server/pipeline/pipeline_status.go` | 状态更新函数 |
| `server/store/datastore/cron.go` | CronListNextExecute、CronGetLock 乐观锁 |
| `server/store/datastore/pipeline.go` | CreatePipeline（事务+重试）、UpdatePipeline |
| `server/model/cron.go` | Cron 数据结构 |
| `server/model/pipeline.go` | Pipeline 数据结构（含 Cron 字段） |
| `server/model/const.go` | WebhookEvent、StatusValue 枚举 |
