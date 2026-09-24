# 后台处理、可观测性与部署

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

整体设计只有一种形态。**MongoDB 是所有持久化异步工作的事实来源**：队列、租约、重试历史和死信都在这里。**Redis 是可选的。** 它用于领导者选举、共享的生成任务存储（可恢复的流）、子智能体控制消息的跨副本路由，以及限流和流程（flow）状态存储。每个后台循环都是单个 API 进程内的 `setTimeout`/`setInterval`。**没有独立的工作进程容器**：每个 API 副本都运行所有循环。有两种不同的机制防止副本之间冲突：
- **Mongo 比较并交换（CAS）租约**（仅当某字段仍为预期值时才修改它，以此认领文档）保护定时任务、触发器投递和排队轮次。
- **Redis 领导者锁**保护文件清理、代码环境协调器（reconciler）和 MCP 初始化。

---

## 1. 后台循环与工作进程清单

启动装配（wiring）位于 `api/server/index.js`。各步骤按以下顺序执行：
1. `startServer` → `connectDb`
2. `startCodeEnvironmentLifecycleReconciler`
3. `indexSync()`
4. `sweepOrphanedPreviews`
5. `startExpiredFileSweep`
6. `configureGenerationStreams()`（`GenerationJobManager.initialize`）
7. 在 `listen` 回调中：`initializeMCPs` → `initializeOAuthReconnectManager` → `checkMigrations` → `initializeAgentTriggerService` → `initializeScheduleEngine`。

之后设置 `serverReady=true`。在此之前，聊天 POST 请求返回 503 并带 `Retry-After: 1`，定时任务的写操作由 `createScheduleWriteGate` 把关（`packages/api/src/schedules/readiness.ts`，状态为 `starting|armed|unavailable`）。

关闭流程通过 `registerShutdownTask(name, fn, {phase:'pre-drain'|default, priority})`（`packages/api/src/app/shutdown`）完成。pre-drain 阶段的任务（定时任务引擎、触发器引擎、生成管理器的准备工作）会在 HTTP 监听器排空之前停止认领新工作。

| # | 循环 | 周期 / 时机 | 协调方式 | 作用 | 文件 |
|---|---|---|---|---|---|
| 1 | **定时任务引擎 tick** | `setTimeout` 链，每 30 s 一次，外加 0–2 s 随机抖动 | 在 `Schedule` 上做 Mongo 租约 CAS（租约 5 min，时钟偏差余量 30 s） | 最多认领 `admissionConcurrency`（默认 20）个到期的定时任务，并逐个触发（`fireSchedule`）。每第 4 次 tick（以及启动时一次）运行 `reconcile()` | `packages/api/src/schedules/engine.ts` |
| 2 | 定时任务协调轮次 | 约每 2 min 一次（每第 4 次 tick），批量 100 | 幂等写入、按行隔离、`reconciledAt` 轮换 | 依据任务存储状态结算处于 `started` / `requires_action` 的运行。重放缺失的记账（`getUnbookkeptRuns`），擦除已排空的软删除定时任务，为没有 `nextRunAt` 的已启用定时任务重新装定（re-arm） | 同上 |
| 3 | 定时任务擦除兜底清理 | 5 min + 0–30 s 抖动，批量 100 | 幂等 | 仅在引擎无法装定（arm）时，或在集群模式的 `experimental.js` 入口中运行。擦除 `deleting` 状态的定时任务，并结算滞留的运行 | `packages/api/src/schedules/erasure.ts`、`service.ts:startErasureFallback` |
| 4 | **智能体触发器投递引擎** | 基础轮询 1 s，空闲时指数退避至 15 s。入队或完成时立即唤醒。定时器还会缩短到已知最早的 `availableAt`（最多跟踪 64 个截止时间） | Mongo CAS 认领，使用新的 `claimToken`，租约 2 min | 最多认领 `concurrency=4` 个投递，并通过回环（loopback）HTTP POST 分发。处理重试、延后和死信 | `packages/api/src/agents/triggers/engine.ts` |
| 5 | 触发器维护（`recoverPurges`） | `setInterval` 30 s，每步上限 25 | 幂等 | 并行执行五个步骤：恢复孤立的用户清除（purge）、完成被放弃的通道（lane）发布、恢复批次回执、使旧版 actor 回执过期、回收检查点删除凭据。之后，如果批次恢复成功，再回收不活跃的通道 | `packages/api/src/agents/triggers/service.ts:recoverPurges` |
| 6 | 排队轮次恢复 | `setInterval` 30 s，上限 100 | Mongo 认领租约 2 min；协调租约 2 min，退避 5 s → 5 min | 为排队轮次重新发布触发器投递，并协调准入（admission）的不确定状态 | `packages/api/src/agents/queuedTurns.ts`（`initialize`） |
| 7 | GenerationJobManager 清理 | 60 s | 按存储各自处理 | 回收已完成和已过期的生成任务。Redis TTL：已完成 300 s，运行中 1200 s，requiresAction 86400 s。另有按订阅者的租约定时器 | `packages/api/src/stream/GenerationJobManager.ts:865`、`implementations/RedisJobStore.ts:1897`、`InMemoryJobStore.ts:233` |
| 8 | 过期文件保留期清理 | `FILE_RETENTION_SWEEP_INTERVAL_MS`，默认 1 h（0 表示禁用） | **仅领导者**（`isLeader()`） | 删除 `expiredAt` 已过的文件。每个文件的重试退避为 1 min → 24 h，最多 10 次，之后“搁置（park）”30 天 | `packages/api/src/files/sweep.ts`，由 `api/server/services/Files/process.js:startExpiredFileSweep` 调用 |
| 9 | 代码环境生命周期协调器 | 60 s | **仅领导者** | 协调已挂载的代码环境工作进程的吊销租约 | `packages/api/src/code/lifecycle.ts:383` |
| 10 | GitHub 技能同步调度器 | `setTimeout` 链，`skillSync.github.intervalMinutes`（默认 60 min，有上下限） | Mongo 锁租约 30 min，由 `setInterval` 刷新 | 从 GitHub 源拉取技能 | `packages/api/src/skills/sync/scheduler.ts`、`github.ts:2131` |
| 11 | MCP 授权栅栏（fence）重试工作进程 | 30 s | 幂等 | 重试发布 MCP 凭据的“代际栅栏（generation fence）” | `packages/api/src/mcp/authorizationRetry.ts` |
| 12 | 领导者选举续期 | 每 10 s 续期；租约 25 s；续期尝试 3 次，间隔 0.5 s | Redis `SET key uuid NX EX 25`；卸任时用 Lua 比较后删除 | 为仅领导者任务选出唯一领导者。`USE_REDIS` 关闭时始终是领导者 | `packages/api/src/cluster/LeaderElection.ts`、`cluster/config.ts`（环境变量 `LEADER_LEASE_DURATION` 等） |
| 13 | Redis 心跳 | 可配置 | – | 对 Redis 连接发送 PING，超时则强制重连 | `packages/api/src/cache/heartbeat.ts` |
| 14 | 子智能体线程租约心跳 | 心跳 10 s，TTL 30 s | 子 Conversation 上的 Mongo 租约 | 让脱离运行（detached）的子运行对删除协议保持可见 | `packages/api/src/agents/subagentThreads.ts:2426/2645` |
| 15 | Redis 子智能体任务路由注册 | 心跳 10 s | Redis 属主目录 | 公告哪个副本拥有哪个存活的子智能体任务 | `packages/api/src/agents/subagentTaskRouting.ts:1310` |
| 16 | 子智能体活动流 | 心跳 15 s；按需心跳 10 s，TTL 30 s | Redis pub/sub | 为打开的侧边栏提供实时进度投影 | `packages/api/src/agents/subagentActivity.ts` |
| 17 | 事件子生成租约 | 心跳 10 s，TTL 30 s | Mongo 租约 | 让事件驱动的子生成对删除保持可见 | `packages/api/src/agents/triggers/lease.ts` |
| 18 | 流程状态管理器轮询 | 按流程而定 | Keyv/Redis | 轮询 OAuth 和 MEILI_SYNC 的流程状态 | `packages/api/src/flow/manager.ts:759` |
| 19 | 文件处理漏桶 | 1000/60 ms（60 req/s） | 进程内 | 对文件处理调用限流 | `api/server/utils/queue.js` |
| 20 | 代码环境引用租约续期 | `renewIntervalMs` | Mongo | 在操作期间保持智能体对某代码环境的引用有效 | `packages/data-schemas/src/methods/codeEnvironment.ts:157` |
| 21 | 内存诊断 | 60 s（仅在 `--inspect` 或 `MEM_DIAG` 时） | – | 堆快照与统计 | `packages/api/src/utils/memory.ts` |
| 22 | 启动时一次性任务 | 一次 | Meilisearch 同步使用 `FlowStateManager` 流程锁（`MEILI_SYNC`） | `indexSync`（Mongo → Meilisearch，`MEILI_SYNC_THRESHOLD`）、`sweepOrphanedPreviews`、`seedDatabase`、`checkMigrations`、`initializeMCPs`（由领导者写入注册表） | `api/db/indexSync.js`、`api/server/index.js` |
| 23 | 被动的 Mongo TTL 索引 | Mongo TTL 监视器（约 60 s） | – | 见 §5 | 各 schema |

---

## 2. 定时任务子系统（“定时聊天”）

**文件：**
- 引擎与准入：`packages/api/src/schedules/{engine,fire,cadence,capacity,readiness,trigger,handlers,service,erasure,mcp,types}.ts`
- 宿主适配器：`api/server/services/Schedules/index.js`（惰性 `createSchedulesService`，装配 `enqueueAgentTrigger`、余额和 MCP 预检）
- 路由：`api/server/routes/schedules.js`
- Schema：`packages/data-schemas/src/schema/schedule.ts`、`scheduleRun.ts`
- 方法：`packages/data-schemas/src/methods/schedule.ts`

该功能属于实验性功能，默认关闭（`FOLLOWUPS.md`）。

**API**（挂载在 `/api/schedules`，位于写入闸门之后）：
- `GET /`、`GET /:id`（权限 SCHEDULES USE）
- `POST /`、`PATCH /:id`、`DELETE /:id`（CREATE）
- `POST /:id/run`（手动“立即运行”；可选 IP 限流器）

创建操作通过 `clientRequestId` 加 `clientRequestDigest` 实现幂等。附件会获得一个可续期的 TTL 保留（`SCHEDULE_FILE_HOLD`：续期 14 天，最长 90 天）。

**限制**（`types.ts DEFAULT_SCHEDULE_LIMITS`，来自 `interface.schedules`，可按角色/用户/租户解析）：

| 设置 | 默认值 |
|---|---|
| `maxPerUser` | 10 |
| `minIntervalMinutes` | 60 |
| `autoDisableAfterFailures` | 5 |
| `admissionConcurrency` | 20 |
| `fireConcurrency`（全局生成槽位） | 5 |
| `mcpPreflightConcurrency` | 3 |
| `mcpPreflightTimeoutMs` | 5 min |
| `requireProject` / `projectId` 固定 | – |

全局总开关是环境变量 `SCHEDULES_DISABLED`，或基础配置中的 `interface.schedules:false`。它们只停止认领；协调始终继续运行。

**拓扑规则**（`service.ts:isTopologySafeToArm`）：只有当 `GenerationJobManager.isRedis`（Redis 流）或 `SCHEDULES_SINGLE_PROCESS=true` 时，引擎才会装定。它还会先运行 `ensureScheduleIndexes()`，如果索引创建失败就拒绝装定，因为唯一索引正是防止重复触发的守卫。

**节奏**（`cadence.ts`）：
- 结构化的 `hourly|daily|weekdays|weekly`，或 `cron`（croner 库），基于 IANA 时区。
- 每个定时任务有确定性的抖动：id 的 djb2 哈希对 120 s 取模，这样“所有人都在 9:00”会被分散开。
- 夏令时遵循 croner 的语义。
- `nextRunAt = cronOccurrence + jitter`，严格晚于 `after`。

**认领 / 租约**（`methods/schedule.ts:claimDueSchedule`）：`findOneAndUpdate({enabled:true, deleting≠true, nextRunAt ≤ now, leaseUntil missing|null|< now−30s}, {$set:{leaseUntil:now+5min, leaseBy:instanceId, claimToken:uuid}}, sort nextRunAt asc)`。
- 使用工作进程自身的时钟；为兼容 DocumentDB，禁止使用 `$$NOW` 和管道式更新。
- `claimToken` 在每次认领和属主编辑时轮换，因此工作进程之后的每次写入都以它为栅栏。
- `configRevision` 只在属主编辑时递增。运行会捕获该值，这样记账和自动禁用就不会基于运行从未使用过的配置采取行动。
- 手动“立即运行”使用 `acquireManualRunLease`（持有者为 `manual:<token>`）。审批后恢复使用 `acquireResumeLease`（60 s）+ `consumeResumeLease`。

**触发 / 准入管线**（`fire.ts:fireSchedule`），按顺序：
1. **错过触发守卫**（引擎）：如果某次触发时间已逾期超过 15 min（`MISFIRE_GRACE_MS`），则不触发，直接跳到下一个未来的触发时间。
2. **重新解析属主上下文。** 加载用户（不存在则跳过，`user_missing`）和按属主的限制（`disabled` 则跳过）。重新检查间隔下限、账户删除栅栏（`user_deleting`）和 SCHEDULES 权限（`permission_revoked`）。检查智能体访问权限；`agent_deleted` 或无权访问会禁用该定时任务。解析项目（`project_required` / `project_deleted` 会禁用它）。
3. **余额检查。** 如果属主额度不足，插入一条 `skipped_balance` 运行。连续出现一定次数后会以 `insufficient_balance` 自动禁用（`BALANCE_SKIP_DISABLE_THRESHOLD`）。
4. **解析文件。** 暂时性失败会保留租约作为退避，不推进。
5. **重新验证认领**，然后检查是否正在关闭。
6. **MCP 就绪预检**（`mcp.ts`，无人值守连接，独立的有界池）。`mcp_reauth_required|configuration_missing|permission_denied` 会立即禁用该定时任务。`mcp_unavailable` 计入失败阈值。
7. 在预留之前**构建触发器信封**：`mode:'fire'`、`event.type:'schedule.occurrence'`、`deliveryId = sched:<id>:<iso>`。由此得到确定性的 `deliveryKey`。
8. **预留运行行。** 一次插入同时认领一个**全局容量槽位**（`capacity.ts:withCapacitySlot`；`capacitySlot` 上 `status:'started'` 条件下的唯一部分索引在数据库层面强制 `fireConcurrency`），并充当**重叠守卫**（`status:'started'` 条件下的唯一部分索引 `{scheduleId}`，因此每个定时任务只有一个活跃运行）。
   - `{scheduleId, scheduledFor}` 上的重复键表示 `duplicate`。
   - 槽位冲突时重试下一个槽位。
   - 槽位饱和表示 `capacity`：保留租约作为退避，不推进。
9. **最终重新验证**（令牌、租约、未在删除、已启用）。被取代的触发会回滚预留。
10. **持久化入队：**`enqueueAgentTrigger(envelope, {orderingKey: schedule.id})`。存储层返回不确定的错误时，运行保持 `started`，交由协调来判定。
11. **推进 `nextRunAt`**，以 `claimToken` 为栅栏。手动运行从不推进；它只释放自己的租约。

生成本身稍后在触发器引擎（§3）中通过回环 POST 到 `/api/agents/chat/agents` 执行。该请求携带一个 60 s 的触发器 JWT。`trigger.ts:captureScheduleFireContext` 只有在设置了 `req._isAgentTrigger` 且信封为 `schedule.occurrence` 时才信任定时任务身份。自动触发不受按用户的消息和并发限流器约束；手动触发则受约束。

生成结束时，会调用 `recordScheduleOutcome` → `methods.recordRunOutcome`，并传入 `autoDisableAfterFailures`：
- 它会等待任何交互式 Stop 的持久化确认。
- 它把 `scheduleOutcome` 写到保留的任务上（`preserveForScheduleReconcile`），这样即使崩溃也会留下证据。
- 计数器按每次触发幂等（`countedFor`，有界），并且有序（`countersAsOf`）。
- `failureCount >= threshold` 会设置 `disabledReason:'too_many_failures'`。

过期的审批通过 `GenerationJobManager.setApprovalExpiredHandler(recordExpiredScheduleApproval)` 结算，状态为 `interrupted`。

### ScheduleRun 状态机

以 `(scheduleId, scheduledFor)` 唯一标识：

```
            (no row)
   ┌───────────┼──────────────────────┬───────────────────────┐
   │ skip at admission                │ admission failure     │ reserve (slot + overlap index)
   ▼                                  ▼                       ▼
skipped_overlap / skipped_balance   started(admissionOnly)   started ──enqueue──► [trigger delivery → generation]
 (terminal, no slot)                   │ reconcile               │
                                       ▼                         ├─ generation pauses for tool approval ─► requires_action
                                     error                       │        (single-active index no longer applies;
                                                                 │         later occurrences may fire)
                                                                 │   requires_action ─ claimScheduleResume
                                                                 │        (resumeClaimedAt, re-claims capacity) ─► started
                                                                 │   requires_action ─ approval expired / abandoned >25h ─► interrupted
                                                                 ├─ complete ─► success | skipped_balance (mid-run refusal)
                                                                 ├─ error ────► error
                                                                 ├─ aborted/stop/delete ─► interrupted   (abortRequestedAt fence 30 min)
                                                                 └─ jobless & delivery dead ─► error ; jobless & age>30 min ─► interrupted
```

- 终态写入会设置 `settledAt`（驱动 TTL）和 `bookkept:false`。随后定时任务层面的记账（`lastRun`、`runCount`、连续计数、自动禁用）会把它翻转为 `bookkept:true`。协调器会重放停留在 `bookkept:false` 的行。
- 协调规则（`engine.ts:reconcile`）：
  - 存在不足 2 min 的运行会被跳过。
  - 任务处于 `running` → 不处理。
  - `requires_action` → 投影暂停状态。
  - `complete`/`error` → 根据保留的预期结果完成结算，然后清除保留的任务。
  - `aborted` → 遵守 30 min 的“中止进行中”栅栏。
  - 没有任务 → 查询持久化的投递：`staging/batched/pending/leased` = 仍然存活；`dead` = 立即判为 error；否则 30 min 后视为孤儿。
- 定时任务行：`enabled/disabledReason`（10 种原因）、`deleting`（软删除，会释放按用户的 `slot` 唯一索引）、`deletionSuspension{token, enabled, nextRunAt}`（账户删除期间的可逆暂停）、`erased` 墓碑（`erasedAt` 上的 TTL 为 24 h）。

---

## 3. 智能体触发器与事件 actor

**文件：**
- 核心：`packages/api/src/agents/triggers/*`。请先阅读 `README.md`。
- 装配：`api/server/services/Agents/triggers.js`
- Schema：`packages/data-schemas/src/schema/{triggerDelivery,triggerLaneSequence,triggerUserPurge}.ts`
- 持久化：`packages/data-schemas/src/methods/triggerDelivery.ts`（4k 行）

### 概念

触发器是**面向智能体轮次的、持久化且与来源无关的任务队列**。每个异步来源都会构建一个带版本的 `AgentTriggerEnvelope`（`envelope.ts`）并调用 `enqueueAgentTrigger`；没有任何来源直接调用智能体运行时。来源包括：
- 定时任务
- 通过 `POST /api/agents/v1/events` 的远程 webhook（Remote Agents API 密钥 + `Idempotency-Key` 请求头，返回 202 + `Location`，可用 `GET /v1/events/:id` 查询状态）
- 事件绑定（`POST /v1/events/bindings`）
- 子智能体完成唤醒
- 后台工具完成唤醒
- 排队轮次

信封包含：
- **模式：**`fire`（新对话）、`continue`（已有对话 + 精确的 `parentMessageId`）、`steer`（注入到正在运行的生成中）。
- **身份：**主体 `{userId, role, tenantId}`、事件 `{id, type, occurredAt, source{id,type}, payload}`、目标 `{agentId, conversationId?, bindingId?, sourceKeyId?}`，另外还有 `input`，对于 `fire` 还有一个 `run` 上下文。
- **`expectedAction`**（可选）：一个工具名加一个参数子集。

**身份与幂等**（`delivery.ts`）：
- `deliveryKey` 由 `getAgentTriggerIdempotencyKey` 派生（对稳定字段的规范化 JSON 做 sha256）。`fingerprint` 是信封的摘要。
- `deliveryKey` 上的唯一索引使入队幂等。以不同的 fingerprint 重放会抛出 `AgentTriggerDeliveryConflictError`。
- **排序通道** `orderingKey`：`trigger_lane_<sha256(tenant, user, lane)>`。通道要么是显式的键，要么是默认的 `(source.type, source.id, mode, agentId, conversationId)`。
- **合并**（仅限已绑定子智能体的 `continue`）：在 750 ms 内，最多 8 个事件或 512 KiB 会被合并为一个 `librechat.agent_event_batch` 轮次。每个成员保留各自的回执。
- 信封上限为 1 MiB。

### 投递信箱与排序（通俗解释）

每个排序通道都像一个严格的 FIFO 信箱：
1. 入队时先插入一行 **`staging`**（`laneSequence: 0`）。该行在获得位置之前就已持久化。
2. 由**通道发布者栅栏**分配位置。该栅栏是 `TriggerLaneSequence` 文档 `{_id: orderingKey, value, publisherDeliveryId, publisherStartedAt}`：工作进程原子地对 `value` 执行 `$inc`，并把自己记录为发布者。然后它把该通道中**最旧的** staging 行发布为 `pending`，并设置 `laneSequence = value`。
   - 任何副本都可以完成被放弃的发布（30 s 维护循环中的 `recoverAgentTriggerLanePublications`）。
   - 因此，后来的事件不会越过卡在间隙中的较早事件。
3. 认领时，`findEarlierUnsettled` 检查同一通道中是否有更早的未结算投递。如果有，已认领的投递会被**释放**，250 ms 后再检查；如果阻塞属于“正在处理（active handling）”，则 5 s 后再检查。
4. **Actor 信箱**（已绑定的事件 actor）：`awaitTerminalHandling:true` 会让*下一个*投递保持阻塞，即使当前投递在传输层面已“成功”。它会一直阻塞，直到子生成记录一个终态 `handling.status`（`applied | completed_no_action | failed | cancelled`）。不同的绑定并行运行。
5. 死信不会阻塞通道。`requeue` 会把死信作为新的通道尾部重新接纳（回到 staging）。不活跃的通道计数器会被回收（`reclaimInactiveAgentTriggerLanes`）。

### 认领、租约与重试

引擎位于 `engine.ts`；`claimNextAgentTriggerDelivery` 位于 `methods/triggerDelivery.ts:1332`。
- **认领：**读取最多 8 个最旧的候选（`status:pending, availableAt ≤ now`，或 `leased` 且 `leaseUntil ≤ now−30s`），然后按 `_id` + status 对其中一个做 CAS。设置 `status:leased, leaseBy:workerId, leaseUntil:now+2min, claimToken:uuid`。最多尝试 16 次 CAS。
- 之后的每次写入（`beginAttempt`、`complete`、`retry`、`defer`、`dead`、`release`）都以 `(id, workerId, claimToken)` 为栅栏。即使是同一进程重新认领，也会拿到新的令牌。
- **分发**（`host.ts`）是对已绑定监听器（或 `AGENT_TRIGGERS_SELF_URL`）的回环 `fetch`：
  - fire/continue → `POST /api/agents/chat/agents`
  - steer → `POST /api/agents/chat/steer/deliver`
  - Bearer 是用 `generateAgentTriggerToken(userId, '60s')` 签发的 JWT；超时 30 s；响应体上限 64 KiB。
  - 错误类型为 `AgentTriggerExecutionError{certainty: definite|ambiguous, retryable, retryAfter, deferWithoutAttempt}`。
- **重试：**最多 8 次尝试。退避为 `min(1s·2^(n−1), 5min)`，并在 `[d/2, d]` 内取随机延迟。遵守 `Retry-After`，上限 24 h。
- 无效信封（`AgentTriggerDispatchError`）和不可重试的错误直接进入 dead。
- `AgentTriggerDeliveryDeferredError`、账户删除导致的延后以及运行时未就绪，都会**延后且不消耗尝试次数**（默认 5 s）。
- 在进入死信之前，`settleSourceBeforeDeadLetter` 让来源先自行进入终态（排队轮次使用此机制）。
- `history[]` 最多 64 条，每条为 `{attempt, outcome, at, workerId, error}`。

### TriggerDelivery 状态机

```
enqueue ─► staging ──(lane publisher: $inc laneSequence)──► pending ◄───────────────┐
              │ coalesce member                              │ claimNext CAS          │ release (ordering block / user cancelled
              ▼                                              ▼                        │  / stopping) ; defer (no attempt) ;
           batched (merged into batch root)               leased ─────────────────────┘ retry (attempts++, backoff)
                                                             │ lease expired (+30s skew) → reclaimable
                                    success ┌────────────────┼──────────────── non-retryable / attempts≥8
                                            ▼                                   ▼
                                       succeeded                              dead ──requeue──► staging (new tail)
                      (expiresAt=+90d unless awaitTerminalHandling;       (kept until requeued/removed)
                       handling: started → applied|completed_no_action|failed|cancelled,
                       then retention window starts)
```

- 成功的 fire 结果携带 `conversationId`、`streamId` 和 `generationCreatedAt`，后续的 `steer` 事件需要这些字段。
- `handling` 是受生成栅栏保护的证据。`applied` 要求宿主观察到的工具证据与 `expectedAction` 匹配（`outcome.ts`、`expectedAction.ts`）；模型的文字描述从不算数。

### 能力屏蔽（通俗解释）

在滚动部署期间，有些内部工作只能在较新的工作进程上运行：
- 脱离运行的动作（`event_actor_detached_action_v1`）
- 后台完成（v1 和回执 v2）
- 排队轮次（`agent_queued_turn_v1`）

因此每一行有两套生命周期：
- **旧版可见的状态：**`capability_staging`，或带有伪造属主且 `leaseUntil = 9999-12-31` 的 `leased`，或 `capability_dead`。旧工作进程无法认领它，但它仍计入通道排序和账户删除安全判断。
- **私有生命周期：**`capabilityStatus: publishing|pending|leased|dead`，配合 `capabilityLeaseBy/Until/ClaimToken` 和 `claimAvailableAt`。只有公告了该能力（`workerCapabilities`）的工作进程才会通过它认领。`producerLeaseUntil` 是生产进程的私有存活租约。

对于没有混合版本集群的 Python 重写，这可以简化为单个 `required_capability` 列（或完全去掉）。

### 事件 actor 与绑定

- `bindings.ts` 注册一个**事件 actor 绑定**：API 密钥 + 父对话 + 父消息 + 目标子智能体。这会创建一个隐藏的只读子对话，并返回一个不透明的 `evtbind_…` id 和 `threadId`。
- 对于按 `bindingId` 寻址的 `continue` 投递，`bindingResolver.ts`（`prepareContinue` 适配器）会在分发前一刻解析智能体、子对话和最新的分支叶节点。
- `continuation.ts` 恰好选择一个适配器：
  - 绑定 → 事件 actor
  - `source.type:'internal'` → `subagent-completion` | `background-tool-completion` | `agent-queued-turn`
- `actor.ts`/`turn.ts` 实现 actor 轮次，状态加载方式为 `fresh|history|checkpoint`：
  - LangGraph 检查点的 “head” 指针只在动作被应用时通过 CAS 推进。
  - 每个投递使用一个调用分叉（invocation fork）命名空间。
  - 出现不确定的提交后，协调日志会阻塞之后的轮次。
- `lease.ts` 为子生成提供 Mongo 租约（TTL 30 s，心跳 10 s），以便删除流程能找到并中止它。
- `detachedAction.ts` 处理工具以脱离方式运行的动作；恢复通过一个受能力屏蔽的内部投递进行。
- Actor 回执（投递行上的 `actorReceipt`）是终态重放证明。它不存储提示词或工具输出。

### 主体删除栅栏（触发器）

- `User.agentTriggerDeletionStartedAt` 是准入栅栏（`methods/user.ts:beginAgentTriggerUserDeletion` / `isAgentTriggerPrincipalActive`）。入队在插入前后都会检查它；分发时会再检查一次。
- `drainUser` 取消该用户在进程内的投递，并轮询直到活跃投递数降为 0（超时 35 s）。
- 在删除用户之前，会写入一个 `TriggerUserPurge{_id:userId, fenceStartedAt}` 标记。用户删除提交后清除载荷，30 s 维护循环会重试孤立的清除标记。

---

## 4. 排队轮次与子智能体

### 排队轮次

**文件：**`packages/api/src/agents/queuedTurns.ts`、`queuedTurnHttp.ts`，schema `data-schemas/src/schema/queuedTurn.ts`，方法 `methods/queuedTurn.ts`。

**路由**（`api/server/routes/agents/index.js:1115-1135`）：
- `POST /api/agents/chat/queued-turns`：插话限流器 + PII 过滤 + 内容审核。
- `GET /api/agents/chat/queued-turns`：只读轮询。
- `DELETE /api/agents/chat/queued-turns/:queuedTurnId`。

**行为：**
- 生成进行中发送的后续消息会存为一行 `AgentQueuedTurn`。该行是 **FIFO 顺序、载荷和生命周期的唯一权威**。它保存文本（≤32 KiB）、文件、引用、手动技能、`parentMessageId` 和 `expectedPredecessorCreatedAt`。
- 行状态：`reserving → queued → claimed → admitted | cancelled | dead`。
- 投递状态：`pending → publishing → published → retiring → retired`。
- 唯一索引强制保证：
  - `(tenant, user, conversation, clientRequestId)`
  - 每个对话内的序号
  - `activeSlot` 容量（≤100）
  - 每个通道只有一个 `claimed`
  - 每个通道只有一个已开始的准入
- 触发器投递（来源 `agent-queued-turn`，能力 `agent_queued_turn_v1`）只是一个可重放的**唤醒**。分发时，`prepareContinue` 认领该行（认领租约 2 min）。只有当捕获的分支有一个干净的、已持久化的前驱结果时才准入；否则延后。
- 生成回执（`terminalReceipt{outcome, admissionId, generationId, generationCreatedAt…}`）只在提供商调用完成登记（enroll）之后才提交（`settleAgentQueuedTurnExecutionAdmission`）。如果进程在登记和回执之间挂掉，该行会保留明确的“准入不确定”证据。之后由带租约的协调（退避 5 s → 5 min）借助 `GenerationJobManager.getGenerationAdmissionEvidence` 解决。
- 取消会退役（retire）对应的投递（`retireAgentTriggerDelivery`，失败时退回到“仅当已死”，再退回到缺失投递栅栏）。
- 一个 30 s 的恢复循环会重新发布投递并协调不确定状态。

### 子智能体

**文件：**`packages/api/src/agents/subagent*.ts`、`lazySubagents.ts`、`background*.ts`、`remote/*`。

- **子智能体线程**（`subagentThreads.ts`、`SubagentThreadTaskStore`）：一个持久化、仅供查看的子对话，以父对话 + 子智能体身份为键。可以通过 `threadId` 继续。
  - 每次继续都会在该对话上获取一个 Mongo 租约（`acquire/renew/releaseSubagentThreadLease`，TTL 30 s，心跳 10 s）。
  - 默认最大线程深度为 1。对话记录上限为 12 MiB。
  - 一个有界的控制队列最多容纳 4096 次调用，持久化的控制回执每 5 s 重试一次。
  - 属主排空的超时为 45 s。
- **存活任务的属主：**恰好有一个 API 进程持有执行器及其中止控制器。
  - `RedisSubagentTaskControlTransport`（`subagentTaskRouting.ts`）维护一个 Redis 属主目录（心跳 10 s），以及带 2 s 超时和重放缓存的请求/应答信封。它把轮询/控制命令路由到属主副本；执行器从不迁移。
  - 只有逻辑线程及其栅栏持久化在 Mongo 中。
- **完成唤醒**（`subagentCompletionWakeup.ts`）：在子智能体脱离执行*之前*，先注册一个持久化的内部 `continue` 触发器（来源 `subagent-completion`），这样崩溃也不会丢失它。
  - 投递会延后，直到子智能体的终态对话记录持久化（最多等待 35 min，因为 SDK 任务 30 min 超时）并且父生成结算完毕。
  - 然后它启动一个收集结果的父轮次。
- **活动流**（`subagentActivity.ts`）：基于 Redis stream 的实时进度（前缀 `subagent-activity:`），仅用于观察。
- **后台工具调用**（`background.ts`）：通过 `AgentCapabilities.run_in_background` 选择启用。
  - 工具返回一个合成的句柄；真正的调用作为进程内的浮动 promise 运行；模型通过 `check_background_task` 轮询。
  - 终态结果可以持久化到投递行上（`backgroundToolResult`，可认领）。
  - 完成唤醒使用来源 `background-tool-completion`（`backgroundCompletionWakeup.ts`），批大小为 `endpoints.agents.backgroundTasks.completionResultBatchSize`（默认 8）。
- `lazySubagents.ts` 计算智能体定义的稳定版本哈希，用于惰性解析子智能体。
- `remote/lifecycle.ts`（`AgentExecutionEnrollment`）是与协议无关的准入、中止和结算权威，供兼容 OpenAI 的 Chat Completions 和 Responses 入口（`remote/host.ts`）使用。它会重新检查主体删除栅栏，并返回 409 `ACCOUNT_DELETION_IN_PROGRESS` / `RUN_REPLACED`。

---

## 5. 清理、保留期与删除级联

### 通过 TTL 和清理实现的保留期

| 数据 | 保留机制 | 位置 |
|---|---|---|
| `ScheduleRun` | `settledAt` 上的 TTL 为 90 天。存活的行没有该字段，永不过期 | `scheduleRun.ts` |
| `Schedule` 擦除墓碑 | `erasedAt` 上的 TTL 为 24 h（部分索引 `erased:true`） | `schedule.ts` |
| `AgentTriggerDelivery` | `expiresAt` 上的 TTL，设为成功时间 + 90 天（`SUCCESS_RETENTION_MS`）。死信一直保留，直到被重新入队或移除。Actor 信箱行只在终态处理之后才开始计算保留窗口 | `triggerDelivery.ts` |
| 文件 | `expiredAt` 保留期由对话保留期设定（`api/server/services/Files/retention.js`；事件绑定继承绑定的截止时间，`eventRetention.ts`）。由仅领导者执行的每小时清理删除 | `packages/api/src/files/sweep.ts` |
| 生成任务（Redis） | TTL 300 s / 1200 s / 86400 s + 60 s 清理 | `RedisJobStore.ts` |
| 聊天过期 | 临时聊天使用 `createChatExpirationDate`（data-schemas） | – |

**需要避免混淆的同名文件：**
- `api/server/services/cleanup.js` 只是启动时的 `deleteNullOrEmptyConversations`。
- `api/server/cleanup.js` 是**内存清理**：它在请求结束后释放 client/graph 对象（`FinalizationRegistry`、将 graph 属性置空）。它不是数据任务。
- `packages/api/src/agents/cleanup.ts` 去除代码工具输出中的样板内容。
- `orphans.ts` 包含工具资源文件 id 的辅助函数。
- `agents/deletion.ts` 是智能体管理的 DELETE 处理器。

### 账户删除级联

`api/server/controllers/UserController.js:deleteUserController`，约第 396–600 行：
1. 如果启用了 2FA，先验证 2FA。
2. **属主栅栏：**`beginAgentTriggerUserDeletion` 设置 `User.agentTriggerDeletionStartedAt`，返回 `acquired|in_progress|missing`。然后 `prepareAgentTriggerUserPurge` 写入 `TriggerUserPurge` 标记。
3. `drainAgentTriggerDeliveriesForUser`。
4. `subagentThreadTaskStore.cancelAndDrainForOwner`。
5. `quiesceUserSchedules(userId, token)`：一个可逆的暂停快照。活跃运行会被中止；如果无法确认它们已停止，调用会失败。
6. 枚举 `GenerationJobManager.getAccountCleanupJobIdsForUser`。对每个任务调用 `abortJob(streamId, {expectedCreatedAt, awaitProviderDrain:true})`，然后调用 `waitForGenerationPersistence`。“任务状态从来不是持久化确认”（`schedules/types.ts`）。
7. `openCheckpointDeletion` → `deleteConvos`（记录 id）→ `checkpointDeletion.cleanup()` / `acknowledge()`。这会移除 LangGraph 检查点。
8. 依次删除：消息、会话、交易记录、密钥、余额、预设、插件认证、分享链接、文件（存储 + 数据库）、工具调用、智能体、智能体 API 密钥、助手、标签、记忆、提示词、技能、MCP 服务器、actions、令牌、组成员关系、ACL 条目、定时任务 → `deleteUserById`。
9. 吊销代码环境工作进程并使缓存失效。
10. 提交后执行 `purgeAgentTriggerDeliveriesForUser`。
11. 失败时：在释放栅栏*之前*执行 `restoreUserSchedulesFromDeletion(token)`，然后 `cancelAgentTriggerUserPurge` + `cancelAgentTriggerUserDeletion`。

被放弃的栅栏只能由运维人员通过 `config/delete-user.js`（`recoverStaleAgentTriggerUserDeletion`）恢复。

### 对话删除

`api/server/routes/convos.js` `DELETE /`：
- **单个对话：**
  - 打开检查点删除并记住 id。
  - 规划并取消子智能体任务（`planCancellationForConversations` / `cancelPlan`）。
  - 调用 `deleteConvos`，其 `beforeDelete` 钩子会调用 `confirmAgentGenerationsDrained`。
  - 为迟到的子任务重放取消（`retryPostDeleteCancellation`），然后 `drainDeletedAgentGenerations`。
  - 清理检查点，然后删除工具调用和分享链接，最后确认。
- **批量删除**在 `withAgentOwnerDeletionFence`（`deleteOwnerConversationPersistence`）内运行。它排空属主的运行、执行删除、再排空一次；如果栅栏失效，则运行一次幂等的恢复清理。
- 子智能体准入使用单独的按属主栅栏列表：`User.subagentAdmissionFences`（`fenceSubagentAdmission`，每个栅栏自行过期）。
- 根据 `CONTEXT.md`，对话删除还会取消排队轮次行、退役其投递并清除其载荷。
- `eraseAgentTriggerDeliveryConversationResults` 移除已存储的后台结果。

---

## 6. 可观测性

- **OpenTelemetry（服务端）。** `api/server/telemetry.js` 在 `index.js` 中被最先 require。当设置了 `OTEL_TRACING_ENABLED` 或 `OTEL_LOGS_ENABLED` 且未设置 `OTEL_SDK_DISABLED` 时启用。
  - `packages/api/src/telemetry/sdk.ts` 使用 `NodeSDK`，插桩包括 HTTP、Express、MongoDB、Mongoose、Undici，以及可选的 IORedis（`OTEL_IOREDIS_TRACING_ENABLED`）。
  - 资源属性：`OTEL_SERVICE_NAME` / `OTEL_SERVICE_VERSION`。
  - 日志导出桥接 winston 日志器（`telemetry/logs.ts`、`OTEL_LOGS_LEVEL`）。导出使用标准的 OTLP 环境变量；支持 gRPC（`grpc.spec.ts`）。
  - `telemetryMiddleware` 和 `telemetryErrorMiddleware` 挂载在 Express 中。关闭时最后刷新导出器。
- **Prometheus 指标。** `packages/api/src/app/metrics.ts:createMetrics` 提供 `/metrics`，由 `METRICS_SECRET` 保护（没有它则返回 401）。其中包含事件 actor 回执和协调存储的 gauge 指标（`index.js:startServer`）。
- **Langfuse**（`packages/api/src/langfuse/*`）：
  - `buildLangfuseConfig`（`config.ts`）生成每次运行的配置，由 `packages/api/src/agents/run.ts:2981` 中的 `@librechat/agents` 使用。SDK 通过 Langfuse 的 OTLP 摄取接口发出 trace。
  - trace id 是确定性的：`sha256(runId/messageId)[:32]`（`trace.ts`）。这样无需查找即可为反馈打分。
  - 采样使用 `LANGFUSE_SAMPLE_RATE`。凭据：`LANGFUSE_PUBLIC_KEY`、`LANGFUSE_SECRET_KEY`、`LANGFUSE_BASE_URL|HOST`。
  - 按租户的 Langfuse 设置存放在应用配置中（`appConfig.langfuse`：enabled、密钥、目的地、请求头、`identity.ts` 中的 `trace` 元数据允许列表）。管理员通过 `/api/admin/langfuse` 编辑（`GET/PUT /connection`、`POST /connection/test`、`GET /connection/session/:conversationId`）。密钥静态加密存储；变更时调用 `invalidateConfigCaches`。
  - 消息存储 `langfuseSampled`、`langfuseDestinationIds` 和 `langfuseRunId`。
  - **反馈**（`feedback.ts:sendFeedbackScore`，由 `api/server/routes/messages.js:720` 调用）调用 Langfuse REST API，分数 id 为 `feedback-<traceId>`，名称为 `user-feedback`，值为 BOOLEAN。清除评分会删除该分数。
- **Langfuse fanout**（`docker-compose.langfuse-fanout.yml`、`otel/langfuse-fanout/`、helm `langfuse-fanout-*.yaml`）：
  - API 把 OTLP 发送到一个 Go 网关（`:4318`）。租户 trace 使用按目的地划分的路径 `/tenant/<eu|us|jp>`。
  - 网关代理到 otel-collector-contrib（`:4319`）。collector 负责内存限制和批处理，按 span 属性 `librechat.langfuse.destination` 路由，按 `…tenant_export.enabled` / `…central_export.enabled` 过滤，并在导出前删除这些路由属性。
  - 导出目标是中心项目（由 collector 持有的 `LANGFUSE_FANOUT_CENTRAL_AUTH_HEADER`）和租户项目（透传租户的 `Authorization` 请求头）。
  - 网关还会扇出 Langfuse 媒体的 create/upload/patch，使用自己的 Redis（`langfuse-fanout-redis`）。
  - 总开关：`LANGFUSE_FANOUT_TENANT_EXPORT_DISABLED`、`LANGFUSE_FANOUT_CENTRAL_MEDIA_EXPORT_DISABLED`。
- **Trace 查看器**（`api/server/routes/traces.js`、`packages/api/src/traces/*`、`langfuse/reader.ts`）：
  - `GET /api/traces/:conversationId/availability`
  - `GET /api/traces/:conversationId/records`
  - `GET /api/traces/:conversationId/records/:recordId`
  - 处理器检查归属（`getConvoOwnership`）并读取被采样的消息引用（`getConversationTraceRefs`）。它在一个时间窗口内对每个可读的目的地查询 Langfuse `GET /api/public/v2/observations`，带超时和字节预算。
  - 按用户的限流器（`traces/limiter.ts`）使用 60 s 窗口、`interface.traceViewer.requestsPerMinute`，以及 Redis 存储 `trace_viewer_user_limiter`。
- **Insights**（`api/server/routes/insights.js`、`packages/api/src/insights/*`、`data-schemas/src/methods/insights.ts`）：
  - 通过 `ENABLE_INSIGHTS` 启用。
  - `GET /api/insights/access` 返回调用者可管理的智能体（ACL/能力解析器）。
  - `GET /api/insights?range=24h|7d|30d|custom&agentIds&search&page&timeZone` 在 Conversation 和 Message 上运行 Mongo 聚合。它返回汇总（用户、对话、消息、token）、每日序列、活跃用户排行、流失用户，以及分页的最新对话列表。这些都是实时计算的；没有后台任务。
- **RUM**（浏览器遥测）（`api/server/routes/rum.js`、`packages/api/src/rum/proxy.ts`）：
  - `POST /api/rum/v1/traces` 和 `/v1/logs` 接收原始的 OTLP protobuf 请求体。
  - 仅当设置了 `RUM_ENABLED`、`RUM_AUTH_MODE=proxy` 和 `RUM_PROXY_TARGET_URL` 时生效。认证通过 `requireRumProxyAuth`。
  - 请求携带 `RUM_PROXY_AUTHORIZATION` 转发。请求体上限和超时来自 `RUM_PROXY_BODY_LIMIT` / `RUM_PROXY_TIMEOUT_MS`。

---

## 7. 部署拓扑

**Dockerfile** 基于 `node:24.16.0-alpine` 构建，包含 jemalloc 和 python3/uv。它先构建前端，然后运行 `npm run backend` 并 `EXPOSE 3080`。一个镜像同时提供 API 和静态 SPA。

| 容器 | 镜像 | 端口 | 依赖 / 通信对象 |
|---|---|---|---|
| `api`（LibreChat） | `librechat-dev(-api)` | 3080（`PORT`） | MongoDB `mongodb://mongodb:27017/LibreChat`、Meilisearch `http://meilisearch:7700`、RAG API `http://rag_api:8000`、可选的 Redis（`USE_REDIS`、`REDIS_URI`；启用 Redis 时 `USE_REDIS_STREAMS` 默认开启）、可选的 Langfuse fanout（4318）、外部 LLM/MCP/Code API |
| `admin-panel` | `librechat-admin-panel` | 3000 | `API_SERVER_URL=http://api:3080` |
| `client`（仅 deploy-compose） | `nginx:1.27.0-alpine` | 80/443 | 反向代理到 api 和 admin-panel（`client/nginx.conf`） |
| `mongodb` | `mongo:8.0.20`（`--noauth`） | 27017（内部） | 卷 `./data-node` |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | 7700（内部） | `MEILI_MASTER_KEY`，卷 `./meili_data_v1.35.1` |
| `vectordb` | `pgvector/pgvector:0.8.0-pg15-trixie` | 5432（rag.yml 映射为 5433） | 卷 `pgdata2` |
| `rag_api` | `librechat-rag-api-dev(-lite)` | 8000（`RAG_PORT`） | `DB_HOST=vectordb` |
| Redis（helm / 可选） | bitnami redis 24.1.3 chart | 6379 | 多副本时必需：领导者选举、任务存储、子智能体路由、限流器。`redis-config/` 中有开发用集群（7001–7003）和 TLS（6380）配置 |
| `langfuse-fanout-collector`（可选） | Go 网关（`otel/langfuse-fanout/Dockerfile`） | 4318 | → `langfuse-fanout-otel:4319`、`langfuse-fanout-redis:6379`、Langfuse Cloud EU/US/JP |
| `langfuse-fanout-otel` | `otel/opentelemetry-collector-contrib:0.143.0` | 4319 | Langfuse OTLP 端点 |
| `langfuse-fanout-redis` | `redis:7.4-alpine` | 6379 | 网关的媒体状态 |

**Helm**（`helm/librechat`）：Deployment（`replicaCount`、HPA）、configmap、ingress、PVC，以及 langfuse-fanout 的 Deployment/Service/ConfigMap。Chart 依赖为 mongodb 16.5.45、meilisearch 0.11.0、redis 24.1.3（自动设置 `USE_REDIS`/`REDIS_URI`）和 `librechat-rag-api`（pgvector）。运行多于一个副本**需要 Redis**；没有 Redis 时，除非设置了 `SCHEDULES_SINGLE_PROCESS`，否则定时任务拒绝装定。

---

## 8. Python 对应方案与建议

**进程模型。** 保持“每个副本运行所有循环”的模型：一个 FastAPI/Starlette 应用，在 lifespan 中用 `asyncio.create_task` 启动各个循环。正确性依赖 Mongo CAS，而不依赖单一工作进程。也可以把循环拆分到单独的工作进程入口（同一镜像，`--role=worker`）。

**循环原语。**

```python
async def loop(name, period, fn, jitter=0):
    while not stop.is_set():
        try: await fn()
        except Exception: log.exception(name)
        try: await asyncio.wait_for(stop.wait(), period + random.uniform(0, jitter))
        except asyncio.TimeoutError: pass
```

对于触发器引擎的自适应轮询，使用一个 `asyncio.Event` 作为“唤醒”，再加一个存放 `available_at` 时间的截止时间堆（`heapq`），空闲时指数退避 1 s → 15 s。

**Mongo。** 使用 `motor` 或 `pymongo` 异步版（`AsyncMongoClient`），并用 `find_one_and_update(..., return_document=AFTER, sort=[...])` 实现租约和认领。复刻这些唯一部分索引：`status:'started'` 条件下的 `{scheduleId}`、`status:'started'` 条件下的 `{capacitySlot}`、唯一的 `{scheduleId, scheduledFor}`，以及唯一的 `deliveryKey`。复刻这些 TTL 索引：`settledAt` 90 天、`expiresAt` 0 s、`erasedAt` 24 h。由数据库强制的容量槽位技巧可以直接沿用：捕获 `DuplicateKeyError`（11000）并尝试下一个槽位。在启动时创建索引，失败时拒绝装定定时任务。

**调度器。** 保留手写的基于 Mongo 租约的引擎，而不是使用 APScheduler 的任务存储；APScheduler 4 虽有 Mongo 数据存储，但这里的栅栏规则是定制的。使用 `croniter` 或 `APScheduler` 的 `CronTrigger` 配合 `zoneinfo` 实现 `computeNextRunAt`。移植 djb2 抖动、15 min 的错过触发宽限期，以及“认领时钟 = leaseUntil − LEASE_MS”规则。

**触发器队列。** 保持基于 Mongo，让信箱、通道序号和死信都留在同一个事务性存储中。arq、Celery 或 Dramatiq 无法表达按通道的 FIFO、“阻塞直到终态处理”以及把死信作为新尾部重新入队这些语义。分发可以继续是对应用自身的回环 HTTP 调用（httpx + 通过 PyJWT 签发的短期 JWT），也可以改为对智能体执行宿主的进程内直接调用。回环方式能让认证、限流和 PII 中间件保持一致。

**领导者选举。** 使用 `redis.asyncio`：`SET key uuid NX EX 25`，每 10 s 续期，关闭时用 Lua 比较后删除。`redis.lock.Lock` 也可以。替代方案：`pottery`/`aioredlock`。

**分布式部分。** 跨副本的子智能体控制对应到通过 `redis.asyncio` 使用的 Redis pub/sub 或 streams（`XADD`/`XREAD`，或带请求 id 和超时的 pub/sub）。限流对应到使用 Redis 存储的 `limits`/`slowapi`。

**关闭。** 使用分阶段的协调器：pre-drain 阶段停止认领并等待进行中的轮次完成，然后 HTTP 服务器排空，最后 post-drain 阶段刷新导出器。在 uvicorn 中，使用 lifespan 关闭流程，外加一个在分发边界检查的全局 `shutting_down` 标志。

**OpenTelemetry。**
- `opentelemetry-sdk` + `opentelemetry-exporter-otlp`（HTTP/gRPC）。
- 插桩：`opentelemetry-instrumentation-fastapi`/`asgi`、`-httpx`、`-pymongo`、`-redis`、`-logging`。
- 同样的环境变量（`OTEL_SERVICE_NAME`、`OTEL_EXPORTER_OTLP_*`）无需修改即可使用。
- Prometheus：`prometheus-client`，`/metrics` 端点放在 bearer 密钥之后。

**Langfuse。** 使用基于 OTel 的 `langfuse` Python SDK v3。用 `trace_id = sha256(run_id)[:32]` 创建 span（在 v3 中可用 `Langfuse.create_trace_id(seed=run_id)` 或显式的 trace 上下文），使反馈能确定性地映射。用 `langfuse.create_score(id=f"feedback-{trace_id}", trace_id=..., name="user-feedback", value=0|1, data_type="BOOLEAN")` 发送反馈。对于 fanout，把 OTLP 导出器或 `LANGFUSE_HOST` 指向网关的 `/tenant/<dest>` 路径，并设置 span 属性 `librechat.langfuse.*`。Go 网关和 collector 可以原样复用。Trace 查看器就是一个调用 `GET /api/public/v2/observations` 的普通 httpx 客户端。

**RUM 代理。** 使用 httpx 流式透传原始 protobuf 请求体。

**后台工具调用与子智能体执行器。** 使用 `asyncio.Task`，放在以任务 id 为键的进程内注册表中，并用 `asyncio.timeout` 设置截止时间。“存活的执行器从不迁移；由 Redis 把控制消息路由到属主”这一点保持不变。

**重试。** 使用手写的辅助函数（`delay = min(base*2**(n-1), cap); uniform(delay/2, delay)`）或 `tenacity`。遵守 `Retry-After`，上限 24 h，并且延后时绝不消耗尝试次数。

**不需要移植的部分：**能力屏蔽的双生命周期（一个滚动部署兼容层），以及 DocumentDB 专用的变通做法，除非 DocumentDB 是目标平台。
