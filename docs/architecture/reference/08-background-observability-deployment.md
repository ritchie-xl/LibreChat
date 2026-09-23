# Background Processing, Observability & Deployment

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

The design has one overall shape. **MongoDB is the source of truth for all durable async work**: queues, leases, retry history and dead letters. **Redis is optional.** It is used for leader election, the shared generation job store (resumable streams), cross-replica routing of subagent control messages, and rate-limit and flow stores. Every background loop is a `setTimeout`/`setInterval` inside the single API process. There is **no separate worker container**: every API replica runs every loop. Two different mechanisms keep replicas from colliding:
- **Mongo compare-and-swap (CAS) leases** (claim a document by changing a field only if it still has an expected value) guard schedules, trigger deliveries and queued turns.
- **The Redis leader lock** guards file sweeps, the code-environment reconciler and MCP init.

---

## 1. Catalog of background loops and workers

Startup wiring is in `api/server/index.js`. The steps run in this order:
1. `startServer` → `connectDb`
2. `startCodeEnvironmentLifecycleReconciler`
3. `indexSync()`
4. `sweepOrphanedPreviews`
5. `startExpiredFileSweep`
6. `configureGenerationStreams()` (`GenerationJobManager.initialize`)
7. Inside the `listen` callback: `initializeMCPs` → `initializeOAuthReconnectManager` → `checkMigrations` → `initializeAgentTriggerService` → `initializeScheduleEngine`.

After that, `serverReady=true`. Until then, chat POSTs get a 503 with `Retry-After: 1`, and schedule writes are gated by `createScheduleWriteGate` (`packages/api/src/schedules/readiness.ts`, states `starting|armed|unavailable`).

Shutdown goes through `registerShutdownTask(name, fn, {phase:'pre-drain'|default, priority})` (`packages/api/src/app/shutdown`). The pre-drain tasks (schedule engine, trigger engine, generation manager prepare) stop claiming new work before the HTTP listener drains.

| # | Loop | Period / timing | Coordination | What it does | File |
|---|---|---|---|---|---|
| 1 | **Schedule engine tick** | `setTimeout` chain every 30 s + 0–2 s random jitter | Mongo lease CAS on `Schedule` (5 min lease, 30 s clock-skew margin) | Claims up to `admissionConcurrency` (default 20) due schedules and fires each one (`fireSchedule`). Every 4th tick (and once at start) it runs `reconcile()` | `packages/api/src/schedules/engine.ts` |
| 2 | Schedule reconcile pass | ~every 2 min (every 4th tick), batch 100 | Idempotent writes, per-row isolation, `reconciledAt` rotation | Settles `started` / `requires_action` runs against job-store state. Replays missing bookkeeping (`getUnbookkeptRuns`), erases soft-deleted schedules that have drained, re-arms enabled schedules with no `nextRunAt` | same |
| 3 | Schedule erasure fallback sweep | 5 min + 0–30 s jitter, batch 100 | idempotent | Runs only when the engine could not arm, or in the clustered `experimental.js` entrypoint. Erases `deleting` schedules and settles stranded runs | `packages/api/src/schedules/erasure.ts`, `service.ts:startErasureFallback` |
| 4 | **Agent trigger delivery engine** | Base poll 1 s, exponential idle backoff to 15 s. Woken immediately on enqueue or completion. Timer also shortened to the earliest known `availableAt` (up to 64 deadlines tracked) | Mongo CAS claim with a fresh `claimToken` and 2 min lease | Claims up to `concurrency=4` deliveries and dispatches them via a loopback HTTP POST. Handles retry, defer and dead-letter | `packages/api/src/agents/triggers/engine.ts` |
| 5 | Trigger maintenance (`recoverPurges`) | `setInterval` 30 s, limit 25 per step | idempotent | Five steps run in parallel: recover orphaned user purges, finish abandoned lane publications, recover batch receipts, expire legacy actor receipts, and reclaim checkpoint-deletion evidence. After that, if batch recovery succeeded, it reclaims inactive lanes | `packages/api/src/agents/triggers/service.ts:recoverPurges` |
| 6 | Queued-turn recovery | `setInterval` 30 s, limit 100 | Mongo claim lease 2 min; reconciliation lease 2 min with backoff 5 s → 5 min | Re-publishes the trigger delivery for queued turns and reconciles admission ambiguity | `packages/api/src/agents/queuedTurns.ts` (`initialize`) |
| 7 | GenerationJobManager cleanup | 60 s | per store | Reaps finished and expired generation jobs. Redis TTLs: completed 300 s, running 1200 s, requiresAction 86400 s. Also a per-subscriber lease timer | `packages/api/src/stream/GenerationJobManager.ts:865`, `implementations/RedisJobStore.ts:1897`, `InMemoryJobStore.ts:233` |
| 8 | Expired-file retention sweep | `FILE_RETENTION_SWEEP_INTERVAL_MS`, default 1 h (0 disables) | **leader only** (`isLeader()`) | Deletes files whose `expiredAt` has passed. Per-file retry backoff 1 min → 24 h, max 10 attempts, then "park" for 30 days | `packages/api/src/files/sweep.ts`, called by `api/server/services/Files/process.js:startExpiredFileSweep` |
| 9 | Code-environment lifecycle reconciler | 60 s | **leader only** | Reconciles attached code-environment worker revocation leases | `packages/api/src/code/lifecycle.ts:383` |
| 10 | GitHub skill sync scheduler | `setTimeout` chain, `skillSync.github.intervalMinutes` (default 60 min, clamped) | Mongo lock lease 30 min, refreshed by `setInterval` | Pulls skills from GitHub sources | `packages/api/src/skills/sync/scheduler.ts`, `github.ts:2131` |
| 11 | MCP authorization-fence retry worker | 30 s | idempotent | Retries publishing MCP credential "generation fences" | `packages/api/src/mcp/authorizationRetry.ts` |
| 12 | Leader election refresh | Renew every 10 s; lease 25 s; 3 renew attempts, 0.5 s apart | Redis `SET key uuid NX EX 25`; Lua compare-and-delete on resign | Single leader for the leader-only jobs. Always leader when `USE_REDIS` is off | `packages/api/src/cluster/LeaderElection.ts`, `cluster/config.ts` (env `LEADER_LEASE_DURATION` etc.) |
| 13 | Redis heartbeat | configurable | – | PINGs Redis connections and forces a reconnect on timeout | `packages/api/src/cache/heartbeat.ts` |
| 14 | Subagent thread lease heartbeat | 10 s heartbeat, 30 s TTL | Mongo lease on the child Conversation | Keeps a detached child run visible to the deletion protocol | `packages/api/src/agents/subagentThreads.ts:2426/2645` |
| 15 | Redis subagent task-routing registration | 10 s heartbeat | Redis owner directory | Advertises which replica owns which live subagent task | `packages/api/src/agents/subagentTaskRouting.ts:1310` |
| 16 | Subagent activity stream | heartbeat 15 s; demand heartbeat 10 s, TTL 30 s | Redis pub/sub | Live progress projection for the open side panel | `packages/api/src/agents/subagentActivity.ts` |
| 17 | Event-child generation lease | heartbeat 10 s, TTL 30 s | Mongo lease | Makes event-driven child generations visible to deletion | `packages/api/src/agents/triggers/lease.ts` |
| 18 | Flow state manager polling | per flow | Keyv/Redis | Polls OAuth and MEILI_SYNC flow state | `packages/api/src/flow/manager.ts:759` |
| 19 | File-processing leaky bucket | 1000/60 ms (60 req/s) | in-process | Throttles file processing calls | `api/server/utils/queue.js` |
| 20 | Code-env reference lease renewal | `renewIntervalMs` | Mongo | Keeps an agent's reference to a code environment alive during an operation | `packages/data-schemas/src/methods/codeEnvironment.ts:157` |
| 21 | Memory diagnostics | 60 s (only with `--inspect` or `MEM_DIAG`) | – | Heap snapshots and statistics | `packages/api/src/utils/memory.ts` |
| 22 | One-shot startup jobs | once | Meilisearch sync uses a `FlowStateManager` flow lock (`MEILI_SYNC`) | `indexSync` (Mongo → Meilisearch, `MEILI_SYNC_THRESHOLD`), `sweepOrphanedPreviews`, `seedDatabase`, `checkMigrations`, `initializeMCPs` (leader writes the registry) | `api/db/indexSync.js`, `api/server/index.js` |
| 23 | Passive Mongo TTL indexes | Mongo TTL monitor (~60 s) | – | See §5 | schemas |

---

## 2. Schedules subsystem ("scheduled chats")

**Files:**
- Engine and admission: `packages/api/src/schedules/{engine,fire,cadence,capacity,readiness,trigger,handlers,service,erasure,mcp,types}.ts`
- Host adapter: `api/server/services/Schedules/index.js` (lazy `createSchedulesService`, wires `enqueueAgentTrigger`, balance, MCP preflight)
- Routes: `api/server/routes/schedules.js`
- Schemas: `packages/data-schemas/src/schema/schedule.ts`, `scheduleRun.ts`
- Methods: `packages/data-schemas/src/methods/schedule.ts`

The feature is experimental and off by default (`FOLLOWUPS.md`).

**API** (mounted at `/api/schedules` behind the write gate):
- `GET /`, `GET /:id` (permission SCHEDULES USE)
- `POST /`, `PATCH /:id`, `DELETE /:id` (CREATE)
- `POST /:id/run` (manual Run Now; optional IP limiter)

Create is idempotent via `clientRequestId` plus `clientRequestDigest`. Attachments get a renewable TTL hold (`SCHEDULE_FILE_HOLD`: renew 14 days, max 90 days).

**Limits** (`types.ts DEFAULT_SCHEDULE_LIMITS`, from `interface.schedules`, resolvable per role/user/tenant):

| Setting | Default |
|---|---|
| `maxPerUser` | 10 |
| `minIntervalMinutes` | 60 |
| `autoDisableAfterFailures` | 5 |
| `admissionConcurrency` | 20 |
| `fireConcurrency` (global generation slots) | 5 |
| `mcpPreflightConcurrency` | 3 |
| `mcpPreflightTimeoutMs` | 5 min |
| `requireProject` / `projectId` pin | – |

Global kill switches are the `SCHEDULES_DISABLED` env var or `interface.schedules:false` in the base config. These stop claims only; reconciliation always keeps running.

**Topology rule** (`service.ts:isTopologySafeToArm`): the engine arms only if `GenerationJobManager.isRedis` (Redis streams) or `SCHEDULES_SINGLE_PROCESS=true`. It also runs `ensureScheduleIndexes()` first and refuses to arm if index creation fails, because the unique indexes are the duplicate-fire guard.

**Cadence** (`cadence.ts`):
- Structured `hourly|daily|weekdays|weekly`, or `cron` (croner library), in an IANA timezone.
- Deterministic per-schedule jitter: a djb2 hash of the id mod 120 s, so "everyone at 9:00" spreads out.
- DST follows croner semantics.
- `nextRunAt = cronOccurrence + jitter`, strictly after `after`.

**Claim / lease** (`methods/schedule.ts:claimDueSchedule`): `findOneAndUpdate({enabled:true, deleting≠true, nextRunAt ≤ now, leaseUntil missing|null|< now−30s}, {$set:{leaseUntil:now+5min, leaseBy:instanceId, claimToken:uuid}}, sort nextRunAt asc)`.
- The worker's own clock is used; DocumentDB compatibility forbids `$$NOW` and pipeline updates.
- `claimToken` is rotated on every claim and on owner edits, so every later worker write is fenced on it.
- `configRevision` is bumped only by owner edits. Runs capture it so bookkeeping and auto-disable can't act on config the run never ran under.
- Manual Run Now uses `acquireManualRunLease` (holder `manual:<token>`). Resume after approval uses `acquireResumeLease` (60 s) + `consumeResumeLease`.

**Fire / admission pipeline** (`fire.ts:fireSchedule`), in order:
1. **Misfire guard** (engine): if the occurrence is overdue by more than 15 min (`MISFIRE_GRACE_MS`), skip forward to the next future occurrence without firing.
2. **Re-resolve the owner's context.** Load the user (skip `user_missing`) and per-owner limits (skip `disabled`). Re-check the interval floor, the account-deletion fence (`user_deleting`), and the SCHEDULES permission (`permission_revoked`). Check agent access; `agent_deleted` or forbidden disables the schedule. Resolve the project (`project_required` / `project_deleted` disable it).
3. **Balance check.** If the owner is out of credits, insert a `skipped_balance` run. A streak of these auto-disables with `insufficient_balance` (`BALANCE_SKIP_DISABLE_THRESHOLD`).
4. **Resolve files.** A transient failure keeps the lease as backoff and does not advance.
5. **Revalidate the claim**, then check for shutdown.
6. **MCP readiness preflight** (`mcp.ts`, unattended connections, separate bounded pool). `mcp_reauth_required|configuration_missing|permission_denied` disables the schedule immediately. `mcp_unavailable` counts toward the failure threshold.
7. **Build the trigger envelope** before reserving: `mode:'fire'`, `event.type:'schedule.occurrence'`, `deliveryId = sched:<id>:<iso>`. This yields a deterministic `deliveryKey`.
8. **Reserve the run row.** One insert both claims a **global capacity slot** (`capacity.ts:withCapacitySlot`; the unique partial index on `capacitySlot` where `status:'started'` enforces `fireConcurrency` in the DB) and serves as the **overlap guard** (unique partial index `{scheduleId}` where `status:'started'`, so only one active run per schedule).
   - Duplicate key on `{scheduleId, scheduledFor}` means `duplicate`.
   - A slot collision retries the next slot.
   - Saturation means `capacity`: keep the lease as backoff and don't advance.
9. **Final revalidation** (token, lease, not deleting, enabled). A superseded fire rolls back the reservation.
10. **Durable enqueue:** `enqueueAgentTrigger(envelope, {orderingKey: schedule.id})`. An ambiguous storage error leaves the run `started` so reconciliation can decide.
11. **Advance `nextRunAt`**, fenced on `claimToken`. A manual run never advances; it only releases its lease.

The generation itself runs later in the trigger engine (§3) via a loopback POST to `/api/agents/chat/agents`. The request carries a 60 s trigger JWT. `trigger.ts:captureScheduleFireContext` trusts schedule identity only when `req._isAgentTrigger` is set and the envelope is `schedule.occurrence`. Automatic fires are exempt from the per-user message and concurrency limiters; manual fires are not.

When the generation finishes, `recordScheduleOutcome` → `methods.recordRunOutcome` is called with `autoDisableAfterFailures`:
- It waits for any interactive Stop's persistence ack.
- It stamps `scheduleOutcome` onto the retained job (`preserveForScheduleReconcile`), so a crash still leaves evidence.
- Counters are idempotent per occurrence (`countedFor`, bounded) and ordered (`countersAsOf`).
- `failureCount >= threshold` sets `disabledReason:'too_many_failures'`.

An expired approval is settled through `GenerationJobManager.setApprovalExpiredHandler(recordExpiredScheduleApproval)` with status `interrupted`.

### ScheduleRun state machine

Keyed uniquely by `(scheduleId, scheduledFor)`:

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

- A terminal write sets `settledAt` (drives the TTL) and `bookkept:false`. Schedule-level bookkeeping (`lastRun`, `runCount`, streaks, auto-disable) then flips it to `bookkept:true`. The reconciler replays rows left at `bookkept:false`.
- Reconciliation rules (`engine.ts:reconcile`):
  - Runs younger than 2 min are skipped.
  - Job `running` → leave alone.
  - `requires_action` → project the pause.
  - `complete`/`error` → finalize from the retained intended outcome, then clear the retained job.
  - `aborted` → honor the 30 min abort-in-flight fence.
  - Jobless → consult the durable delivery: `staging/batched/pending/leased` = still live; `dead` = error now; otherwise orphan after 30 min.
- Schedule rows: `enabled/disabledReason` (10 reasons), `deleting` (soft delete, which frees the per-user `slot` unique index), `deletionSuspension{token, enabled, nextRunAt}` (reversible suspension during account deletion), `erased` tombstone (TTL 24 h on `erasedAt`).

---

## 3. Agent triggers and event actors

**Files:**
- Core: `packages/api/src/agents/triggers/*`. Read `README.md` first.
- Wiring: `api/server/services/Agents/triggers.js`
- Schemas: `packages/data-schemas/src/schema/{triggerDelivery,triggerLaneSequence,triggerUserPurge}.ts`
- Persistence: `packages/data-schemas/src/methods/triggerDelivery.ts` (4k lines)

### Concept

A trigger is a **durable, source-neutral job queue for agent turns**. Every async source builds a versioned `AgentTriggerEnvelope` (`envelope.ts`) and calls `enqueueAgentTrigger`; no source invokes the agent runtime directly. Sources include:
- schedules
- remote webhooks via `POST /api/agents/v1/events` (Remote Agents API key + `Idempotency-Key` header, returns 202 + `Location`, with `GET /v1/events/:id` for status)
- event bindings (`POST /v1/events/bindings`)
- subagent completion wakeups
- background-tool completion wakeups
- queued turns

The envelope contains:
- **Modes:** `fire` (new conversation), `continue` (existing conversation + exact `parentMessageId`), `steer` (inject into a running generation).
- **Identity:** principal `{userId, role, tenantId}`, event `{id, type, occurredAt, source{id,type}, payload}`, target `{agentId, conversationId?, bindingId?, sourceKeyId?}`, plus `input`, and for `fire` a `run` context.
- **`expectedAction`** (optional): a tool name plus an argument subset.

**Identity and idempotency** (`delivery.ts`):
- `deliveryKey` is derived from `getAgentTriggerIdempotencyKey` (a sha256 of canonical JSON of the stable fields). `fingerprint` is the digest of the envelope.
- The unique index on `deliveryKey` makes enqueue idempotent. Replaying with a different fingerprint raises `AgentTriggerDeliveryConflictError`.
- **Ordering lane** `orderingKey`: `trigger_lane_<sha256(tenant, user, lane)>`. The lane is either an explicit key or a default of `(source.type, source.id, mode, agentId, conversationId)`.
- **Coalescing** (bound-child `continue` only): within 750 ms, up to 8 events or 512 KiB are merged into one `librechat.agent_event_batch` turn. Each member keeps its own receipt.
- Envelopes are capped at 1 MiB.

### Delivery mailbox and ordering (plain terms)

Each ordering lane behaves like a strict FIFO mailbox:
1. Enqueue first inserts a **`staging`** row (`laneSequence: 0`). The row is durable before it has a position.
2. A **lane publisher fence** assigns the position. The fence is the `TriggerLaneSequence` document `{_id: orderingKey, value, publisherDeliveryId, publisherStartedAt}`: a worker atomically `$inc`s `value` and records itself as publisher. It then publishes the **oldest** staging row in that lane to `pending` with `laneSequence = value`.
   - Any replica can finish an abandoned publication (`recoverAgentTriggerLanePublications` in the 30 s maintenance loop).
   - A later event therefore can't overtake an earlier one that is stuck in the gap.
3. At claim time, `findEarlierUnsettled` checks for an earlier unsettled delivery in the same lane. If there is one, the claimed delivery is **released** and rechecked after 250 ms, or 5 s when the block is "active handling".
4. **Actor mailbox** (bound event actors): `awaitTerminalHandling:true` keeps the *next* delivery blocked even after the current one "succeeded" at transport level. It stays blocked until the child generation records a terminal `handling.status` (`applied | completed_no_action | failed | cancelled`). Different bindings run in parallel.
5. Dead letters don't block the lane. `requeue` re-admits a dead letter as a new lane tail (back to staging). Inactive lane counters are reclaimed (`reclaimInactiveAgentTriggerLanes`).

### Claiming, leases, retries

The engine is in `engine.ts`; `claimNextAgentTriggerDelivery` is in `methods/triggerDelivery.ts:1332`.
- **Claim:** read up to 8 oldest candidates (`status:pending, availableAt ≤ now`, or `leased` with `leaseUntil ≤ now−30s`), then CAS one by `_id` + status. Set `status:leased, leaseBy:workerId, leaseUntil:now+2min, claimToken:uuid`. Up to 16 CAS attempts.
- Every later write (`beginAttempt`, `complete`, `retry`, `defer`, `dead`, `release`) is fenced on `(id, workerId, claimToken)`. A reclaim by the same process still gets a fresh token.
- **Dispatch** (`host.ts`) is a loopback `fetch` to the bound listener (or `AGENT_TRIGGERS_SELF_URL`):
  - fire/continue → `POST /api/agents/chat/agents`
  - steer → `POST /api/agents/chat/steer/deliver`
  - Bearer is a JWT minted with `generateAgentTriggerToken(userId, '60s')`; 30 s timeout; response body capped at 64 KiB.
  - Errors are typed `AgentTriggerExecutionError{certainty: definite|ambiguous, retryable, retryAfter, deferWithoutAttempt}`.
- **Retry:** max 8 attempts. Backoff `min(1s·2^(n−1), 5min)`, with a random delay in `[d/2, d]`. `Retry-After` is honored, capped at 24 h.
- Invalid envelopes (`AgentTriggerDispatchError`) and non-retryable errors go straight to dead.
- `AgentTriggerDeliveryDeferredError`, account-deletion deferral, and runtime-not-ready all **defer without consuming an attempt** (default 5 s).
- Before dead-lettering, `settleSourceBeforeDeadLetter` lets the source terminalize itself first (used by queued turns).
- `history[]` is bounded to 64 entries, each `{attempt, outcome, at, workerId, error}`.

### TriggerDelivery state machine

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

- A successful fire result carries `conversationId`, `streamId` and `generationCreatedAt`, which a later `steer` event needs.
- `handling` is generation-fenced evidence. `applied` requires host-observed tool evidence that matches `expectedAction` (`outcome.ts`, `expectedAction.ts`); model prose never counts.

### Capability shield (plain terms)

Some internal work must only run on newer workers during a rolling deploy:
- detached actions (`event_actor_detached_action_v1`)
- background completions (v1 and receipt v2)
- queued turns (`agent_queued_turn_v1`)

The row therefore has two lifecycles:
- **Legacy-visible status:** `capability_staging`, or `leased` with a fake owner and `leaseUntil = 9999-12-31`, or `capability_dead`. Old workers can't claim it, but it still counts for lane ordering and account-deletion safety.
- **Private lifecycle:** `capabilityStatus: publishing|pending|leased|dead` with `capabilityLeaseBy/Until/ClaimToken` and `claimAvailableAt`. Only workers advertising the capability (`workerCapabilities`) claim through it. `producerLeaseUntil` is a private liveness lease for the producing process.

For a Python rewrite with no mixed-version fleet, this can collapse to a single `required_capability` column (or be dropped entirely).

### Event actors and bindings

- `bindings.ts` registers an **Event Actor binding**: API key + parent conversation + parent message + target child agent. This creates a hidden, read-only child conversation and returns an opaque `evtbind_…` id and `threadId`.
- For `continue` deliveries addressed by `bindingId`, `bindingResolver.ts` (the `prepareContinue` adapter) resolves the agent, the child conversation and the latest branch leaf right before dispatch.
- `continuation.ts` picks exactly one adapter:
  - binding → event actor
  - `source.type:'internal'` → `subagent-completion` | `background-tool-completion` | `agent-queued-turn`
- `actor.ts`/`turn.ts` implement the actor turn with state loading `fresh|history|checkpoint`:
  - A LangGraph checkpoint "head" pointer is advanced by CAS only on an applied action.
  - An invocation fork namespace is used per delivery.
  - A reconciliation journal blocks later turns after an ambiguous commit.
- `lease.ts` gives the child generation a Mongo lease (30 s TTL, 10 s heartbeat) so deletion can find and abort it.
- `detachedAction.ts` covers actions whose tool runs detached; the resume goes through a capability-shielded internal delivery.
- The actor receipt (`actorReceipt` on the delivery row) is terminal replay proof. It stores no prompts or tool output.

### Principal deletion fence (triggers)

- `User.agentTriggerDeletionStartedAt` is the admission fence (`methods/user.ts:beginAgentTriggerUserDeletion` / `isAgentTriggerPrincipalActive`). Enqueue checks it both before and after the insert; dispatch checks it again.
- `drainUser` cancels in-process deliveries for that user and polls until active deliveries reach 0 (35 s timeout).
- A `TriggerUserPurge{_id:userId, fenceStartedAt}` marker is written before the user is deleted. Payloads are purged after the user deletion commits, and the 30 s maintenance loop retries orphaned purge markers.

---

## 4. Queued turns and subagents

### Queued turns

**Files:** `packages/api/src/agents/queuedTurns.ts`, `queuedTurnHttp.ts`, schema `data-schemas/src/schema/queuedTurn.ts`, methods `methods/queuedTurn.ts`.

**Routes** (`api/server/routes/agents/index.js:1115-1135`):
- `POST /api/agents/chat/queued-turns`: steer limiters + PII filter + moderation.
- `GET /api/agents/chat/queued-turns`: read-only poll.
- `DELETE /api/agents/chat/queued-turns/:queuedTurnId`.

**Behavior:**
- A follow-up message sent while a generation is active is stored as an `AgentQueuedTurn` row. That row is the **sole FIFO, payload and lifecycle authority**. It holds text (≤32 KiB), files, quotes, manual skills, `parentMessageId` and `expectedPredecessorCreatedAt`.
- Row status: `reserving → queued → claimed → admitted | cancelled | dead`.
- Delivery state: `pending → publishing → published → retiring → retired`.
- Unique indexes enforce:
  - `(tenant, user, conversation, clientRequestId)`
  - sequence per conversation
  - `activeSlot` capacity (≤100)
  - one `claimed` per lane
  - one admission-started per lane
- The trigger delivery (source `agent-queued-turn`, capability `agent_queued_turn_v1`) is only a replayable **wakeup**. When dispatched, `prepareContinue` claims the row (2 min claim lease). It admits only after the captured branch has a clean durable predecessor outcome; otherwise it defers.
- The generation receipt (`terminalReceipt{outcome, admissionId, generationId, generationCreatedAt…}`) is committed only after provider invocation enrolls (`settleAgentQueuedTurnExecutionAdmission`). If the process dies between enrollment and receipt, the row keeps explicit "admission-indeterminate" evidence. It is resolved by leased reconciliation (backoff 5 s → 5 min) using `GenerationJobManager.getGenerationAdmissionEvidence`.
- Cancel retires the delivery (`retireAgentTriggerDelivery`, falling back to only-if-dead, then a missing-delivery fence).
- A 30 s recovery loop re-publishes deliveries and reconciles ambiguity.

### Subagents

**Files:** `packages/api/src/agents/subagent*.ts`, `lazySubagents.ts`, `background*.ts`, `remote/*`.

- **Subagent thread** (`subagentThreads.ts`, `SubagentThreadTaskStore`): a durable, view-only child conversation keyed by parent conversation + subagent identity. It can be continued by `threadId`.
  - Each continuation takes a Mongo lease on the conversation (`acquire/renew/releaseSubagentThreadLease`, TTL 30 s, heartbeat 10 s).
  - Maximum thread depth is 1 by default. Transcripts are capped at 12 MiB.
  - A bounded control queue holds up to 4096 invocations, with durable control receipts retried every 5 s.
  - Owner drain uses a 45 s timeout.
- **Live task owner:** exactly one API process holds the executor and its abort controller.
  - `RedisSubagentTaskControlTransport` (`subagentTaskRouting.ts`) keeps a Redis owner directory (10 s heartbeat) and request/reply envelopes with a 2 s timeout and replay caches. It routes poll/control commands to the owner replica; the executor never migrates.
  - Only the logical thread and its fence are persisted in Mongo.
- **Completion wakeup** (`subagentCompletionWakeup.ts`): a durable internal `continue` trigger (source `subagent-completion`) is registered *before* detached child execution, so a crash can't lose it.
  - Delivery defers until the child's terminal transcript persists (waiting up to 35 min, since SDK tasks time out at 30 min) and the parent generation settles.
  - It then starts a parent turn that collects the result.
- **Activity stream** (`subagentActivity.ts`): Redis-stream live progress (`subagent-activity:` prefix), observational only.
- **Background tool calls** (`background.ts`): opt-in via `AgentCapabilities.run_in_background`.
  - The tool returns a synthetic handle; the real call runs as an in-process floating promise; the model polls with `check_background_task`.
  - Terminal results may be persisted onto the delivery row (`backgroundToolResult`, claimable).
  - Completion wakeups use source `background-tool-completion` (`backgroundCompletionWakeup.ts`), with batch size `endpoints.agents.backgroundTasks.completionResultBatchSize` (default 8).
- `lazySubagents.ts` computes stable version hashes of agent definitions for lazy subagent resolution.
- `remote/lifecycle.ts` (`AgentExecutionEnrollment`) is the protocol-neutral admission, abort and settlement authority used by the OpenAI-compatible Chat Completions and Responses ingress (`remote/host.ts`). It rechecks the principal deletion fence and returns 409 `ACCOUNT_DELETION_IN_PROGRESS` / `RUN_REPLACED`.

---

## 5. Cleanup, retention and deletion cascade

### Retention via TTL and sweeps

| Data | Retention mechanism | Location |
|---|---|---|
| `ScheduleRun` | TTL 90 days on `settledAt`. Live rows lack the field and never expire | `scheduleRun.ts` |
| `Schedule` erased tombstone | TTL 24 h on `erasedAt` (partial `erased:true`) | `schedule.ts` |
| `AgentTriggerDelivery` | TTL on `expiresAt`, set to success + 90 days (`SUCCESS_RETENTION_MS`). Dead letters kept until requeued or removed. Actor-mailbox rows start the window only after terminal handling | `triggerDelivery.ts` |
| Files | `expiredAt` retention set from conversation retention (`api/server/services/Files/retention.js`; event bindings inherit the binding deadline, `eventRetention.ts`). Deleted by the leader-only hourly sweep | `packages/api/src/files/sweep.ts` |
| Generation jobs (Redis) | TTLs 300 s / 1200 s / 86400 s + 60 s cleanup | `RedisJobStore.ts` |
| Chat expiry | `createChatExpirationDate` (data-schemas) for temporary chats | – |

**Name collisions to avoid:**
- `api/server/services/cleanup.js` is only a startup `deleteNullOrEmptyConversations`.
- `api/server/cleanup.js` is **memory hygiene**: it disposes client/graph objects after a request (`FinalizationRegistry`, nulling graph properties). It is not a data job.
- `packages/api/src/agents/cleanup.ts` strips code-tool output boilerplate.
- `orphans.ts` has tool-resource file-id helpers.
- `agents/deletion.ts` is the agent-management DELETE handler.

### Account deletion cascade

`api/server/controllers/UserController.js:deleteUserController`, lines ~396–600:
1. Verify 2FA, if enabled.
2. **Owner fence:** `beginAgentTriggerUserDeletion` sets `User.agentTriggerDeletionStartedAt`, returning `acquired|in_progress|missing`. Then `prepareAgentTriggerUserPurge` writes the `TriggerUserPurge` marker.
3. `drainAgentTriggerDeliveriesForUser`.
4. `subagentThreadTaskStore.cancelAndDrainForOwner`.
5. `quiesceUserSchedules(userId, token)`: a reversible suspension snapshot. Active runs are aborted; the call fails if they can't be confirmed stopped.
6. Enumerate `GenerationJobManager.getAccountCleanupJobIdsForUser`. Call `abortJob(streamId, {expectedCreatedAt, awaitProviderDrain:true})` on each, then `waitForGenerationPersistence`. "Job state is never a persistence acknowledgement" (`schedules/types.ts`).
7. `openCheckpointDeletion` → `deleteConvos` (records ids) → `checkpointDeletion.cleanup()` / `acknowledge()`. This removes LangGraph checkpoints.
8. Delete in sequence: messages, sessions, transactions, keys, balances, presets, plugin auth, shared links, files (storage + DB), tool calls, agents, agent API keys, assistants, tags, memories, prompts, skills, MCP servers, actions, tokens, group memberships, ACL entries, schedules → `deleteUserById`.
9. Revoke code-environment workers and invalidate caches.
10. `purgeAgentTriggerDeliveriesForUser`, post-commit.
11. On failure: `restoreUserSchedulesFromDeletion(token)` *before* releasing the fence, then `cancelAgentTriggerUserPurge` + `cancelAgentTriggerUserDeletion`.

An abandoned fence is recovered only by an operator via `config/delete-user.js` (`recoverStaleAgentTriggerUserDeletion`).

### Conversation deletion

`api/server/routes/convos.js` `DELETE /`:
- **Single conversation:**
  - Open checkpoint deletion and remember the id.
  - Plan and cancel subagent tasks (`planCancellationForConversations` / `cancelPlan`).
  - `deleteConvos` with a `beforeDelete` hook that calls `confirmAgentGenerationsDrained`.
  - Replay cancellation for late children (`retryPostDeleteCancellation`), then `drainDeletedAgentGenerations`.
  - Checkpoint cleanup, then delete tool calls and shared links, then acknowledge.
- **Bulk delete** runs inside `withAgentOwnerDeletionFence` (`deleteOwnerConversationPersistence`). It drains owner runs, deletes, repeats the drain, and runs an idempotent recovery sweep if the fence lapsed.
- Subagent admission uses a separate per-owner fence list: `User.subagentAdmissionFences` (`fenceSubagentAdmission`, each fence self-expiring).
- Per `CONTEXT.md`, conversation deletion also cancels queued-turn rows, retires their deliveries and purges their payloads.
- `eraseAgentTriggerDeliveryConversationResults` removes stored background results.

---

## 6. Observability

- **OpenTelemetry (server).** `api/server/telemetry.js` is required first in `index.js`. It is enabled when `OTEL_TRACING_ENABLED` or `OTEL_LOGS_ENABLED` is set and `OTEL_SDK_DISABLED` is not.
  - `packages/api/src/telemetry/sdk.ts` uses `NodeSDK` with instrumentations HTTP, Express, MongoDB, Mongoose, Undici, and optionally IORedis (`OTEL_IOREDIS_TRACING_ENABLED`).
  - Resource attributes: `OTEL_SERVICE_NAME` / `OTEL_SERVICE_VERSION`.
  - Log export bridges the winston logger (`telemetry/logs.ts`, `OTEL_LOGS_LEVEL`). Export uses the standard OTLP env vars; gRPC is supported (`grpc.spec.ts`).
  - `telemetryMiddleware` and `telemetryErrorMiddleware` are mounted in Express. Shutdown flushes the exporters last.
- **Prometheus metrics.** `packages/api/src/app/metrics.ts:createMetrics` serves `/metrics`, guarded by `METRICS_SECRET` (401 without it). It includes event-actor receipt and reconciliation storage gauges (`index.js:startServer`).
- **Langfuse** (`packages/api/src/langfuse/*`):
  - `buildLangfuseConfig` (`config.ts`) produces a per-run config consumed by `@librechat/agents` in `packages/api/src/agents/run.ts:2981`. The SDK emits traces through Langfuse OTLP ingestion.
  - The trace id is deterministic: `sha256(runId/messageId)[:32]` (`trace.ts`). This lets feedback be scored without a lookup.
  - Sampling uses `LANGFUSE_SAMPLE_RATE`. Credentials: `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_BASE_URL|HOST`.
  - Per-tenant Langfuse settings live in the app config (`appConfig.langfuse`: enabled, keys, destination, headers, `trace` metadata allowlist in `identity.ts`). Admins edit them via `/api/admin/langfuse` (`GET/PUT /connection`, `POST /connection/test`, `GET /connection/session/:conversationId`). The secret is encrypted at rest; changes call `invalidateConfigCaches`.
  - Messages store `langfuseSampled`, `langfuseDestinationIds` and `langfuseRunId`.
  - **Feedback** (`feedback.ts:sendFeedbackScore`, called from `api/server/routes/messages.js:720`) calls the Langfuse REST API with score id `feedback-<traceId>`, name `user-feedback`, and a BOOLEAN value. Clearing the rating deletes the score.
- **Langfuse fanout** (`docker-compose.langfuse-fanout.yml`, `otel/langfuse-fanout/`, helm `langfuse-fanout-*.yaml`):
  - The API sends OTLP to a Go gateway (`:4318`). Tenant traces use a destination-scoped path `/tenant/<eu|us|jp>`.
  - The gateway proxies to an otel-collector-contrib (`:4319`). The collector does memory limiting and batching, routes on the span attribute `librechat.langfuse.destination`, filters on `…tenant_export.enabled` / `…central_export.enabled`, and deletes the routing attributes before export.
  - Export goes to the central project (collector-held `LANGFUSE_FANOUT_CENTRAL_AUTH_HEADER`) and to tenant projects (pass-through of the tenant `Authorization` header).
  - The gateway also fans out Langfuse media create/upload/patch, using its own Redis (`langfuse-fanout-redis`).
  - Kill switches: `LANGFUSE_FANOUT_TENANT_EXPORT_DISABLED`, `LANGFUSE_FANOUT_CENTRAL_MEDIA_EXPORT_DISABLED`.
- **Trace viewer** (`api/server/routes/traces.js`, `packages/api/src/traces/*`, `langfuse/reader.ts`):
  - `GET /api/traces/:conversationId/availability`
  - `GET /api/traces/:conversationId/records`
  - `GET /api/traces/:conversationId/records/:recordId`
  - The handler checks ownership (`getConvoOwnership`) and reads sampled message refs (`getConversationTraceRefs`). It queries Langfuse `GET /api/public/v2/observations` on each readable destination within a time window, with a timeout and byte budget.
  - A per-user rate limiter (`traces/limiter.ts`) uses a 60 s window, `interface.traceViewer.requestsPerMinute`, and the Redis store `trace_viewer_user_limiter`.
- **Insights** (`api/server/routes/insights.js`, `packages/api/src/insights/*`, `data-schemas/src/methods/insights.ts`):
  - Enabled with `ENABLE_INSIGHTS`.
  - `GET /api/insights/access` returns the agents the caller can manage (ACL/capability resolver).
  - `GET /api/insights?range=24h|7d|30d|custom&agentIds&search&page&timeZone` runs Mongo aggregations over Conversation and Message. It returns a summary (users, conversations, messages, tokens), daily series, top users, churned users, and a paginated list of latest conversations. This is computed live; there are no background jobs.
- **RUM** (browser telemetry) (`api/server/routes/rum.js`, `packages/api/src/rum/proxy.ts`):
  - `POST /api/rum/v1/traces` and `/v1/logs` take a raw OTLP protobuf body.
  - Active only when `RUM_ENABLED`, `RUM_AUTH_MODE=proxy` and `RUM_PROXY_TARGET_URL` are set. Auth goes through `requireRumProxyAuth`.
  - Requests are forwarded with `RUM_PROXY_AUTHORIZATION`. Body limit and timeout come from `RUM_PROXY_BODY_LIMIT` / `RUM_PROXY_TIMEOUT_MS`.

---

## 7. Deployment topology

The **Dockerfile** builds on `node:24.16.0-alpine` with jemalloc and python3/uv. It builds the frontend, then runs `npm run backend` and `EXPOSE 3080`. A single image serves both the API and the static SPA.

| Container | Image | Port | Depends on / talks to |
|---|---|---|---|
| `api` (LibreChat) | `librechat-dev(-api)` | 3080 (`PORT`) | MongoDB `mongodb://mongodb:27017/LibreChat`, Meilisearch `http://meilisearch:7700`, RAG API `http://rag_api:8000`, optional Redis (`USE_REDIS`, `REDIS_URI`; `USE_REDIS_STREAMS` defaults on with Redis), optional Langfuse fanout at 4318, external LLM/MCP/Code API |
| `admin-panel` | `librechat-admin-panel` | 3000 | `API_SERVER_URL=http://api:3080` |
| `client` (deploy-compose only) | `nginx:1.27.0-alpine` | 80/443 | reverse proxy to api and admin-panel (`client/nginx.conf`) |
| `mongodb` | `mongo:8.0.20` (`--noauth`) | 27017 (internal) | volume `./data-node` |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | 7700 (internal) | `MEILI_MASTER_KEY`, volume `./meili_data_v1.35.1` |
| `vectordb` | `pgvector/pgvector:0.8.0-pg15-trixie` | 5432 (rag.yml maps 5433) | volume `pgdata2` |
| `rag_api` | `librechat-rag-api-dev(-lite)` | 8000 (`RAG_PORT`) | `DB_HOST=vectordb` |
| Redis (helm / optional) | bitnami redis 24.1.3 chart | 6379 | Required for multi-replica: leader election, job store, subagent routing, limiters. `redis-config/` has dev cluster (7001–7003) and TLS (6380) setups |
| `langfuse-fanout-collector` (optional) | Go gateway (`otel/langfuse-fanout/Dockerfile`) | 4318 | → `langfuse-fanout-otel:4319`, `langfuse-fanout-redis:6379`, Langfuse Cloud EU/US/JP |
| `langfuse-fanout-otel` | `otel/opentelemetry-collector-contrib:0.143.0` | 4319 | Langfuse OTLP endpoints |
| `langfuse-fanout-redis` | `redis:7.4-alpine` | 6379 | gateway media state |

**Helm** (`helm/librechat`): Deployment (`replicaCount`, HPA), configmaps, ingress, PVC, and langfuse-fanout Deployment/Service/ConfigMap. Chart dependencies are mongodb 16.5.45, meilisearch 0.11.0, redis 24.1.3 (sets `USE_REDIS`/`REDIS_URI` automatically) and `librechat-rag-api` (pgvector). Running more than one replica **requires Redis**; without it, schedules refuse to arm unless `SCHEDULES_SINGLE_PROCESS` is set.

---

## 8. Python equivalents and recommendations

**Process model.** Keep the "every replica runs every loop" model: one FastAPI/Starlette app with a lifespan that starts `asyncio.create_task` loops. Correctness does not depend on a single worker because it rests on Mongo CAS. Optionally split loops into a separate worker entrypoint (same image, `--role=worker`).

**Loop primitive.**

```python
async def loop(name, period, fn, jitter=0):
    while not stop.is_set():
        try: await fn()
        except Exception: log.exception(name)
        try: await asyncio.wait_for(stop.wait(), period + random.uniform(0, jitter))
        except asyncio.TimeoutError: pass
```

For the trigger engine's adaptive poll, use an `asyncio.Event` "wake" plus a deadline heap (`heapq`) of `available_at` times, with an exponential idle backoff of 1 s → 15 s.

**Mongo.** Use `motor` or `pymongo` async (`AsyncMongoClient`) and `find_one_and_update(..., return_document=AFTER, sort=[...])` for leases and claims. Reproduce the partial unique indexes: `{scheduleId}` where `status:'started'`, `{capacitySlot}` where `status:'started'`, `{scheduleId, scheduledFor}` unique, and `deliveryKey` unique. Reproduce the TTL indexes: `settledAt` 90 days, `expiresAt` 0 s, `erasedAt` 24 h. The DB-enforced capacity-slot trick carries over directly: catch `DuplicateKeyError` (11000) and try the next slot. Create indexes at startup and refuse to arm schedules on failure.

**Scheduler.** Keep the hand-rolled Mongo-lease engine rather than APScheduler's job store; APScheduler 4 has a Mongo data store, but the fencing rules here are custom. Use `croniter` or `APScheduler`'s `CronTrigger` with `zoneinfo` for `computeNextRunAt`. Port the djb2 jitter, the 15 min misfire grace and the "claim clock = leaseUntil − LEASE_MS" rule.

**Trigger queue.** Keep it Mongo-backed so the mailbox, lane sequences and dead letters stay in one transactional store. arq, Celery or Dramatiq can't express per-lane FIFO with "block until terminal handling" plus dead-letter requeue as a new tail. Dispatch can stay a loopback HTTP call to the app itself (httpx + a short-lived JWT via PyJWT), or become a direct in-process call to the agent execution host. The loopback keeps auth, limits and PII middleware uniform.

**Leader election.** Use `redis.asyncio`: `SET key uuid NX EX 25`, renew every 10 s, and a Lua compare-and-delete on shutdown. `redis.lock.Lock` also works. Alternative: `pottery`/`aioredlock`.

**Distributed pieces.** Cross-replica subagent control maps to Redis pub/sub or streams via `redis.asyncio` (`XADD`/`XREAD`, or pub/sub with request ids and timeouts). Rate limits map to `limits`/`slowapi` with Redis storage.

**Shutdown.** Use a phased coordinator: a pre-drain phase stops claims and awaits in-flight passes, then the HTTP server drains, then a post-drain phase flushes exporters. In uvicorn, use the lifespan shutdown plus a global `shutting_down` flag checked at dispatch boundaries.

**OpenTelemetry.**
- `opentelemetry-sdk` + `opentelemetry-exporter-otlp` (HTTP/gRPC).
- Instrumentations: `opentelemetry-instrumentation-fastapi`/`asgi`, `-httpx`, `-pymongo`, `-redis`, `-logging`.
- The same env vars (`OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_*`) work unchanged.
- Prometheus: `prometheus-client`, with the `/metrics` endpoint behind a bearer secret.

**Langfuse.** Use the `langfuse` Python SDK v3, which is OTel-based. Create spans with `trace_id = sha256(run_id)[:32]` (in v3 either `Langfuse.create_trace_id(seed=run_id)` or explicit trace context) so feedback maps deterministically. Send feedback with `langfuse.create_score(id=f"feedback-{trace_id}", trace_id=..., name="user-feedback", value=0|1, data_type="BOOLEAN")`. For fanout, point the OTLP exporter or `LANGFUSE_HOST` at the gateway's `/tenant/<dest>` path and set span attributes `librechat.langfuse.*`. The Go gateway and collector are reusable unchanged. The trace viewer is a plain httpx client for `GET /api/public/v2/observations`.

**RUM proxy.** Use an httpx streaming passthrough of the raw protobuf body.

**Background tool calls and subagent executors.** Use `asyncio.Task` in a per-process registry keyed by task id, with `asyncio.timeout` deadlines. Keep "live executor never migrates; Redis routes control to the owner" as-is.

**Retries.** Use a hand-rolled helper (`delay = min(base*2**(n-1), cap); uniform(delay/2, delay)`) or `tenacity`. Honor `Retry-After` capped at 24 h, and never consume an attempt on a defer.

**What not to port:** the capability-shield dual lifecycle (a rolling-deploy compatibility layer), and the DocumentDB-specific workarounds, unless DocumentDB is a target.
