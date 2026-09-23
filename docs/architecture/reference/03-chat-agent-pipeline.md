# Chat / Agent Generation Pipeline & Streaming

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

**Scope and caveats.** This covers the backend path for chat and agent generation only. The `@librechat/agents` SDK is an external npm package that is not vendored in this repository. Section 7 describes the SDK from how this repo imports and uses it, plus its dependency list in `package-lock.json` (`@librechat/agents@3.9.1`, which depends on `@langchain/core 1.2.8`, `@langchain/langgraph 1.4.8`, `@langchain/openai`/`anthropic`/`google-*`/`aws`/`mistralai`/`deepseek`/`xai`, `openai`, `@anthropic-ai/sdk`, `ai-tokenizer`, `@langfuse/*`).

The overall design:
- **All chat is an agent run.** A plain "OpenAI/Anthropic/…" chat becomes an ephemeral agent.
- **The HTTP POST only starts a background job.** It returns JSON immediately.
- **Streaming is a separate SSE GET** that subscribes to a job's event bus. The bus is in-memory or Redis.
- **The job layer** (`GenerationJobManager`) provides resumability, abort, HITL pauses, steering and idempotency across replicas.

The codebase is very defensive: epoch fencing, CAS, "terminal claims". A Python port can keep the concepts and simplify a lot of the mixed-version and rolling-deploy code.

---

## 1. End-to-end sequence of a chat POST

**Mounting.** In `api/server/index.js`:
- It builds stream services with `createStreamServices()` and passes them to `GenerationJobManager.configure({...})`, then calls `.initialize()`. That is the periodic cleanup/reaper (lines ~132–143).
- It mounts `/api/agents` → `api/server/routes/agents/index.js`, with startup-telemetry and "reject until ready" middleware on `/api/agents/chat`.

**Route ordering in `api/server/routes/agents/index.js`:**
1. `/v1/responses` (Open Responses API), `/v1/agents` (management), `/v1/skills`, and `/v1` (OpenAI-compatible). These use API-key auth (`requireRemoteAgentAuth`) and are mounted before JWT.
2. `requireJwtAuth`, `checkBan`, `uaParser`.
3. Stream/control endpoints, which skip the body middleware chain:
   - `GET /chat/stream/:streamId`
   - `GET /chat/active`
   - `GET /chat/status/:conversationId`
   - `POST /chat/abort`
   - `POST /chat/steer`, `/chat/steer/deliver`, `/chat/steer/cancel`, `/chat/steer/arm`
   - `POST|GET /chat/queued-turns`, `DELETE /chat/queued-turns/:id`
4. `router.use('/', v1)`: agent CRUD, tools, actions.
5. `chatRouter` at `/chat`: `configMiddleware`, retry-probe limiter, `detectGenerationRetry`, IP/user message limiters, then `routes/agents/chat.js`.

**Numbered flow for `POST /api/agents/chat[/:endpoint]`:**

1. **`routes/agents/chat.js` middleware chain.** Runs in this order:
   1. `restoreResumeContext`: only for `/resume`. It replays the paused turn's config from `job.metadata.pendingAction.resumeContext` onto the body.
   2. `createMessageFilterPii`: PII filter, runs before moderation.
   3. `moderateText`.
   4. `checkAgentAccess`: role permission `AGENTS.USE`.
   5. `canAccessAgentFromBody`: ACL VIEW on `agent_id`.
   6. `validateConvoAccess`.
   7. `guardSubagentThreadTurn`: blocks human writes into subagent child threads.
   8. `buildEndpointOption`.

   Routes: `POST /resume` → `ResumeController`; `POST /` and `POST /:endpoint` → `AgentController(req,res,next, initializeClient, addTitle)`. The `/:endpoint` form is for ephemeral agents.

2. **`buildEndpointOption`** (`api/server/middleware/buildEndpointOption.js`):
   - Parses the body with `parseCompactConvo` and enforces model specs.
   - Calls `buildOptions` in `api/server/services/Endpoints/agents/build.js`. This starts a **promise** `agent: loadAgent({ agent_id: isAgentsEndpoint ? agent_id : EPHEMERAL_AGENT_ID, ... })`.
   - `loadAgent` / `loadEphemeralAgent` (`packages/api/src/agents/load.ts`) synthesizes an agent for non-agent endpoints:
     - id = `encodeEphemeralAgentId({endpoint, model, sender})`, provider = endpoint, instructions = `promptPrefix`.
     - Tools come from UI toggles and the model spec: `execute_code`, `file_search`, `web_search`, `memory`, `ask_user_question`, MCP servers (`mcp_all` sentinel or explicit tool names), background/intent tool options, `subagents`, `skills`.
   - Result stored at `req.body.endpointOption`.

3. **`ResumableAgentController`** (`api/server/controllers/agents/request.js`, lines 711+):
   - Negotiates the generation protocol from header `x-librechat-generation-protocol` (1 or 2; `controllers/agents/protocol.js`).
   - Validates `clientRequestId` (idempotency key, 1–128 chars), `overrideUserMessageId`/`overrideConvoId`, `expectedPredecessorCreatedAt`, steer-recovery ids, and `compact` (manual compaction).
   - **conversationId allocation.** If new: `uuidv5(userId:clientRequestId)` (idempotent) or `randomUUID()`. **`streamId === conversationId` always.** One live generation per conversation.
   - Rejects a `parentMessageId` that is only a preliminary (unsaved) id.
   - **Idempotency.** `GenerationJobManager.claimGeneration(userId, clientRequestId, streamId, …)`. If a live job already exists for the claim it returns `{streamId, status:'resumed'}` (dedup). A stale claim is taken over with `takeoverGeneration`.
   - **Concurrency limiter.** `checkAndIncrementPendingRequest(userId)` → 429 with a logged violation.
   - Preallocates `userMessageId` / `responseMessageId` so MCP request-body placeholders are stable.

4. **Job creation.** `GenerationJobManager.createJob(streamId, userId, conversationId, { initialMetadata: {...} })`:
   - Metadata includes endpoint, iconURL, model, `agent_id`, `isTemporary`, `retentionExpiresAt`, `responseMessageId`, preliminary `userMessage`, `mcpRequestBody`, `isRegenerate`, schedule fields, and capability flags (`preemptCapable`, `steerQuotesCapable`).
   - Creates a job-owned `AbortController` and captures `jobCreatedAt`, the **generation epoch** used to fence every later write.

5. **Early HTTP response.** `sendGenerationStarted()` responds `200 {streamId, conversationId, generationCreatedAt, status:'started', generationProtocolVersion}`. Generation continues in the background after this. Queued-turn loopback requests delay this until the provider starts.

6. **Disconnect hook.** `job.emitter.on('allSubscribersLeft', …)` saves a partial `unfinished:true` response to Mongo if every SSE client disconnects mid-run.

7. **`initializeClient`** (`api/server/services/Endpoints/agents/initialize.js`), called with `signal: job.abortController.signal`:
   - `createContentAggregator()` from the SDK → `{contentParts, aggregateContent, stepMap}`.
   - `createToolEndCallback` for artifacts (code outputs, file citations, web search results, memory artifacts).
   - `initializeAgent(...)` (`packages/api/src/agents/initialize.ts`) for the primary agent:
     - Resolves the provider via `getProviderConfig` → `providerConfigMap` (`packages/api/src/endpoints/config/providers.ts`): `openAI`/`azureOpenAI` → `initializeOpenAI`; `anthropic`, `google`/`vertexai`, `bedrock`; `xai`/`deepseek`/`moonshot`/`openrouter` → `initializeCustom`. Then `getOptions()` produces the `llmConfig`.
     - Primes files and resources (`primeResources`: attachments, code-env provisioning, vector DB).
     - Loads tools (`loadAgentTools` / `loadToolsForExecution`) and injects the skills catalog.
     - Computes `maxContextTokens = agentMax − maxOutputTokens` (default 32000).
   - `discoverConnectedAgents` (`packages/api/src/agents/discovery.ts`) walks graph **edges** (handoff/direct) and the legacy `agent_ids` chain, and initializes each connected agent into `agentConfigs`.
   - `processAddedConvo` (`addedConvo.js`, `packages/api/src/agents/added.ts`): the "multi-convo" side-by-side agent (`ADDED_AGENT_ID`) is added as an extra start node with no incoming edges, so it runs **in parallel**.
   - Resolves subagent trees (lazy subagent descriptors, `lazySubagents.ts`).
   - `getDefaultHandlers({ res, aggregateContent, contentParts, stepMap, toolEndCallback, collectedUsage, streamId, jobCreatedAt, toolExecuteOptions, summarizationOptions, … })` (`api/server/controllers/agents/callbacks.js`).
   - `new AgentClient({... agentConfigs, eventHandlers ...})` (`api/server/controllers/agents/client.js`, extends `api/app/clients/BaseClient.js`).

8. **Back in the controller:**
   - If already aborted, `completeJob`.
   - `resolveAgentTurnExecutionPlan` (`packages/api/src/agents/plan.ts`) picks the state-loading strategy: `fresh` (new convo), `history` (default: rebuild from Mongo messages) or `checkpoint` (event-actor turns only).
   - `resolveTitleTiming` returns `immediate` (default) or `final`.
   - Stores `sender`, then `GenerationJobManager.setContentParts(streamId, client.contentParts)`.

9. **`startGeneration()`** builds `messageOptions` with:
   - `onStart(userMsg, responseMessageId)`: updates job metadata `{userMessage, responseMessageId}` and **emits the `created` event**.
   - `getReqData`: observes the user-message write promise.
   - `beforeResponsePersistence: claimBeforeResponsePersistence`: claims the terminal state via `claimTerminalJob(... 'complete'|'aborted', {persistencePending:true})`.
   - `abortController: job.abortController`.

   It then calls `client.sendMessage(text, messageOptions)`, wrapped by the event-actor turn adapter for trigger-bound turns. If eligible (first turn, not temporary) and timing is `immediate`, it also starts `addTitle(req, {immediate:true, convoReady, onTitleGenerated: emitTitleEvent})` **in parallel**.

10. **`BaseClient.sendMessage` → `sendReservedMessage`** (`api/app/clients/BaseClient.js` ~741–1255):
    - `handleStartMethods` → `setMessageOptions`, `createUserMessage`, and `loadHistory(conversationId, parentMessageId)`. `loadHistory` reads all conversation messages and walks the parent chain (`getMessagesForConversation`). With summarization it starts from the latest usable summary part (`findCheckpointSummaryPart`).
    - `onStart` → `created` event.
    - `AgentClient.buildMessages` (client.js ~2285):
      - Per-message token counts → `indexTokenCountMap`.
      - Attachments: images, documents, text files, and file_search "augmented prompt" context.
      - Memories (`useMemory`, `getRequestMemories`).
      - Shared run context and MCP server instructions merged into each agent's system prompt via `applyContextToAgent` (`packages/api/src/agents/context.ts`).
    - **Save user message:** `saveMessageToDatabase(userMessage)` → `db.saveMessage` + `saveTurnConversation` (conversation upsert, `packages/api/src/conversations/save.ts`). Fired async; its promise is reported via `getReqData`.
    - `checkBalance` (balance reservation).
    - **`sendCompletion` → `AgentClient.chatCompletion`** (client.js 4404+):
      - Builds the LangGraph config: `thread_id=conversationId`, private checkpoint namespace keys, `user_id`, `requestBody`, `hide_sequential_outputs`, `last_agent_index`, `recursionLimit`, `signal`, `streamMode:'values'`, `version:'v2'`.
      - Converts history with the SDK's `formatAgentMessages` (TMessage content parts → LangChain `BaseMessage[]`, plus the token map and summary).
      - Starts memory extraction in parallel (`runMemory`).
      - Calls `createRun({...})` (`packages/api/src/agents/run.ts`; §5), then `GenerationJobManager.setGraph(streamId, run.Graph)`, then `run.processStream({messages}, config, {callbacks})`.
      - `handleRunInterrupt(run, streamId)` handles HITL pauses.
      - Afterwards: settles activity labels, awaits memory, and calls `recordCollectedUsage` (token transactions and balance).
    - Builds `responseMessage`: `messageId=responseMessageId`, `parentMessageId=userMessageId`, `content=contentParts`, `model`, `sender`, `tokenCount`, `attachments` from awaited artifact promises, `contextMeta`, and usage metadata.
    - Calls `opts.beforeResponsePersistence(responseMessage)` (terminal claim), then `saveMessageToDatabase(responseMessage)` (`databasePromise`).

11. **Controller success tail** (request.js ~2940–3240):
    - Idempotently re-saves the user message and the response (`unfinished` if aborted, preempt-incomplete or step-limit reached; `finish_reason: TOOL_CALL_LIMIT` when the step budget ran out).
    - Repairs conversation message references and resolves `convoReady` (unblocks title persistence).
    - Settles the schedule run.
    - Builds the **final event** `{final:true, conversation, title, requestMessage, responseMessage, pendingSteers?}` and publishes it with `GenerationJobManager.publishTerminalClaim(terminalClaim, finalEvent)` (job → `complete`/`aborted`).
    - `finishResumableRequest` decrements the concurrency counter.
    - In `final` title mode, runs `addTitle` now. Then `disposeClient`.

12. **Error path.** `completeJob(streamId, errorMessage, jobCreatedAt, { beforeErrorPublication: () => saveErrorTurn(...) })` saves the user message plus an `error:true` response message and a conversation row, then publishes an SSE `error` event.

13. **Client stream.** Separately, the client opens `GET /api/agents/chat/stream/:streamId?generationCreatedAt=…` (`&resume=true` on reconnect) and receives the events below (§2, §3).

**HITL resume flow.** `POST /api/agents/chat/resume` `{conversationId, actionId, generationCreatedAt, decisions[] | answer(s)}` → `api/server/controllers/agents/resume.js`:
1. Authorize the owner/tenant, and check the agent and request fingerprint against the pending action.
2. `resolveResumeValue`: `tool_approval` → `mapToolApprovalResolutions`; `ask_user_question` → `mapAskUserAnswers` (`packages/api/src/agents/hitl/resume.ts`).
3. `GenerationJobManager.approvals.resolve(...)`: an atomic single-winner claim, `requires_action` → `running`.
4. Send the ACK, then rebuild the client with `initializeClient`.
5. `client.resumeCompletion` → `createRun` with the same `thread_id` and checkpointer → `run.resume(resumeValue)`. Events stream over the **same streamId**, so the existing SSE connection continues.
6. Finalize (save the response, publish the final event, delete the checkpoint), or re-pause.

---

## 2. SSE event types and payload shapes

**Framing** (`routes/agents/index.js` `writeEvent`): every frame is `event: message\ndata: <json>\n\n`, except errors, which use `event: error\ndata: {"error": "...", "generationProtocolVersion": n}`.
- Headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache, no-transform`, `X-Accel-Buffering: no`, `Content-Encoding: identity`, `x-librechat-generation-protocol`.
- The JSON is one of three shapes: (a) a "control" object (`created`, `sync`, `final`); (b) `{event: <name>, data: <payload>}` for graph and host events; (c) a raw error.

| Event | Emitted by | Payload |
|---|---|---|
| `{created:true, message:<userMessage + files/manualSkills>, streamId}` | request.js `onStart` | User message with ids; the client renders the user bubble and preliminary response |
| `{sync:true, resumeState, pendingEvents}` | stream route on `resume=true` | `ResumeState` (`stream/interfaces/IJobStore.ts` ~863): `runSteps[]`, `aggregatedContent[]`, `userMessage`, `responseMessageId`, `conversationId`, `sender`, `iconURL`, `model`, `titleEvent`, `replayEvents[]`, `pendingOAuthPrompts`, pending action/steers, usage |
| `on_run_step` | SDK `GraphEvents.ON_RUN_STEP` via `getDefaultHandlers` | `RunStep {id, runId, agentId, index, stepIndex, groupId?, type: 'message_creation'|'tool_calls', stepDetails:{message_creation:{message_id}} | {tool_calls:[{id,name,args,mcpServerName?}]}, created_at, status}` (`packages/data-provider/src/types/agents.ts` ~231) |
| `on_run_step_delta` | SDK | `{id, delta:{type:'tool_calls', tool_calls:[ToolCallChunk], auth?, expires_at?, approval?}}` (tool-arg streaming and MCP OAuth prompts) |
| `on_run_step_completed` | SDK | `{result:{id, index, tool_call:{id,name,args,output,inputValidationError?}}}` |
| `on_run_step_closed` | SDK | `{id, status, created_at, completed_at}`; the host stamps `runStepStatus`/`runStepDurationMs` onto the part |
| `on_message_delta` | SDK | `{id: stepId, delta:{content:[{type:'text', text}]}}` |
| `on_reasoning_delta` | SDK | `{id, delta:{content:[{type:'think', think}]}}` |
| `on_agent_update` | host, when `hide_sequential_outputs` | `{runId, message:"<Agent> is thinking..."}` |
| `on_summarize_start` / `_delta` / `_complete` | SDK summarization node | Summary content-part progress (compaction) |
| `on_context_usage` / `on_token_usage` | `UsageEvents`; `ModelEndHandler` → `emitTokenUsage` | Per-call usage `{input_tokens, output_tokens, cache…, model, cost?}`; context-window gauge |
| `on_pending_action` | `ApprovalEvents` in `AgentClient.exposePendingApproval` | Client projection of `PendingAction {actionId, streamId, conversationId, runId, responseMessageId, payload: ToolApprovalInterruptPayload | AskUserQuestionInterruptPayload, createdAt, expiresAt}` |
| `on_steer_applied` / `on_steer_updated` | `SteerEvents` | Steer injected at a tool boundary (content part `type:'steer'`); capability update (v2 only) |
| `on_subagent_update` | SDK `ON_SUBAGENT_UPDATE` | Subagent progress envelope `{runId, parentRunId, subagentRunId, subagentType, phase, data, label, …}` |
| `on_activity_label`, `on_reasoning_label` (+ internal `_attempt`) | host fast-model labelers | Label for a tool batch or reasoning part `{index, stepId, label…}` |
| `on_sandbox_starting`, `on_ptc_tool_call` | `StepEvents` | Code sandbox cold-boot notice; programmatic-tool-calling progress |
| `attachment` | `writeAttachment` in callbacks.js | `TAttachment` (file metadata, `toolCallId`, `messageId`, type, e.g. `web_search`, `memory`, code files) |
| `title` | request.js `emitTitleEvent` | `{conversationId, title}` |
| `{final:true, conversation, title, requestMessage, responseMessage, pendingSteers?}` | `publishTerminalClaim` | Terminal. The abort variant: `{final:true, aborted:true, conversation:{conversationId}|null, requestMessage, responseMessage:{messageId, content, unfinished:true…}}` (GenerationJobManager ~4700). The reconcile variant `{final:true, reconcile:true, reconcileReason}` tells the client to refetch |
| `event: error` | `completeJob`/`emitError` | `{error: string}` |

Enums are in `packages/data-provider/src/types/runs.ts` (`StepEvents`, `UsageEvents`, `ApprovalEvents`, `SteerEvents`, `ActivityLabelEvents`, `ReasoningLabelEvents`).

**Visibility rule** (`checkIfLastAgent`). In multi-agent graphs with `hide_sequential_outputs`, only the last agent's message/reasoning deltas are forwarded. Tool-call steps are always forwarded.

**Content aggregation.** `aggregateContent({event, data})` (SDK `createContentAggregator`) folds each event into `contentParts[]`, indexed by `runStep.index`. Part types include `text`, `think`, `tool_call`, `image_file`, `summary`, `steer`, `agent_update` and label parts. `contentParts` becomes `responseMessage.content`.

---

## 3. Stream job lifecycle, resumability and abort

**Files:** `packages/api/src/stream/GenerationJobManager.ts` (~9.5k lines), `interfaces/IJobStore.ts`, `implementations/{InMemoryJobStore,RedisJobStore,InMemoryEventTransport,RedisEventTransport}.ts`, `createStreamServices.ts`, `ApprovalLifecycle.ts`, `SteeringLifecycle.ts`, `internal/coalescing.ts`.

**Composition.** `GenerationJobManager` combines two services:
- **jobStore**: job metadata, content state and the steer queue.
- **eventTransport**: pub/sub.

`createStreamServices()` uses Redis when `USE_REDIS_STREAMS` (defaults to `USE_REDIS`) and a client are available. It duplicates the connection for the subscriber and otherwise falls back to in-memory. Runtime state (`RuntimeJobState`: abort controller, early event buffer, subscriber handlers, sequence counters, preempt state) is always process-local.

**Job record** (`SerializableJobData`):
- Identity and status: `streamId`, `userId`, `tenantId`, `status: running|requires_action|complete|error|aborted`, `createdAt` (epoch), `conversationId`, `generationProtocolVersion`.
- Turn data: `userMessage`, `responseMessageId`, `sender`, `endpoint`, `iconURL`, `model`, `mcpRequestBody`.
- Serialized payloads: `finalEvent`, `titleEvent`, `replayEvents`, `tokenUsage`, `contextUsage`, `contextMeta`.
- HITL: `pendingAction`, `resolvedAskUserQuestions`.
- Idempotency and provider execution: idempotency claim token, `providerExecutionId` and drain flags.
- Flags: `terminalPersistencePending`, `steersClosed`, `syncSent`, `createdEventEmitted`, subscriber leases, and schedule/event-actor fields.

**Redis keys** (hash-tagged for Cluster):
- `stream:{id}:job`: hash.
- `stream:{id}:chunks`: a Redis **Stream** (XADD append log of every emitted event).
- `stream:{id}:runsteps`, `stream:{id}:seq` (sequence counter), `stream:{id}:generation-epoch`.
- `stream:{id}:steers`, `:steers-claimed`, `:parked`, `:steer-receipts`.
- `stream:{id}:subscriber-leases:<createdAt>`.
- Global sets: `stream:running`, `stream:requires_action`.
- `stream:user:{tenant:user}:jobs`, `stream:idem:<user:clientRequestId>`.
- Pub/sub channel `stream:{id}:events` carries message types `chunk | chunk_batch | done | error | abort | abort_ack | preempt | subscription_frontier`, each with `seq` and `generationId` (= createdAt). Subscribers reorder by `seq`, with a timeout-based reorder buffer.
- Most state changes are Lua scripts (CAS on `createdAt`/status).

**Emitting.** `emitChunk(streamId, event, {expectedCreatedAt})` is fenced by the generation epoch; it drops events if the runtime was replaced.
- Redis: `jobStore.appendChunk` (XADD) then `eventTransport.emitChunk` (PUBLISH). Delta coalescing into `chunk_batch` is optional.
- In-memory: events go to an **early event buffer** until the first subscriber attaches. Content lives on a WeakRef to the SDK graph and the host `contentParts`.
- The `created` event is serialized ahead of everything else.

**Subscribing.** `GET /chat/stream/:streamId`:
- Checks ownership and tenant, and the epoch (`generationCreatedAt` query → 409 `GENERATION_REPLACED`).
- `subscribe(streamId, onChunk, onDone, onError)` for a first attach: replays the early buffer, then goes live.
- `subscribeWithResume` for `resume=true`: snapshots `getResumeState()`, writes one `sync` frame (run steps + aggregated content + pending events captured between snapshot and attach), then `subscription.activate()`.
- In Redis mode `getResumeState` **reconstructs aggregated content by replaying the XADD chunk log through a fresh `createContentAggregator`**, unless the same process has the live graph (`RedisJobStore.getContentParts`).
- If subscription fails because the run finished or was replaced, it sends a `reconcile` final frame.
- `req.on('close')` unsubscribes. When the last local subscriber leaves, `allSubscribersLeft` saves a partial response.

**Status and active endpoints:**
- `GET /chat/status/:conversationId` → `{active, status, streamId, aggregatedContent, createdAt, elapsedMs, resumeState, pendingAction, unrecoveredSteers}`. Used by the client after a reload to decide whether to reattach.
- `GET /chat/active` → the user's active job ids.

**Terminalization:**
- `claimTerminalJob` is a CAS `running → complete|aborted` with `persistencePending`.
- `publishTerminalClaim(claim, finalEvent)` stores `finalEvent` in the job so late subscribers replay it, then publishes `done`.
- `finishTerminalJob` handles cleanup and the TTL (in-memory `ttlAfterComplete`; Redis EXPIRE).
- `completeJob(streamId, error)` handles errors.
- The periodic `cleanup()` reaps stale jobs: `staleJobTimeout` defaults to 20 min in memory, and expired approvals are reaped.

**Abort across replicas.** `POST /chat/abort` `{streamId|conversationId|abortKey, generationCreatedAt}` (`routes/agents/index.js` ~569):
1. Resolves the job (can find the single active job for `'new'` placeholders).
2. Checks owner and epoch.
3. `GenerationJobManager.abortJob(streamId, { expectedCreatedAt, transformAbortContent, beforePublish })`:
   - CAS the status to `aborted`.
   - Redis: publish `abort` on `stream:{id}:events`. The **owning replica's** `onAbort` handler fires `job.abortController.abort()`, which propagates as the LangGraph run `signal`, and writes an `abort-ack` key. The requester can await the ack or provider drain (`awaitProviderDrain`).
   - Reconstruct content (from the chunk log in Redis) and text, and collect usage.
   - `beforePublish`: **the route persists the user message and the partial `unfinished:true` response to Mongo** and deletes the HITL checkpoint.
   - Publish the abort final event.
   - Responses: 409 `RUN_REPLACED` / `RUN_STILL_ACTIVE` when appropriate.
4. On the owning replica, `sendMessage` returns early. The terminal claim fails because the abort won, so it does not double-publish.

**Idempotency.** `claimGeneration` / `resumeClaimedGeneration` / `takeoverGeneration` / `releaseGeneration` over the `stream:idem:*` key (with a claim token) protect against duplicate POSTs after a lost response.

**Python sketch:**
- `JobStore` protocol (Redis hash + Redis Stream + Lua CAS, or an in-memory dict).
- `EventBus` (Redis pub/sub with a seq, or an `asyncio.Queue` fan-out).
- `JobManager`: create, emit, subscribe(resume), abort, complete, approvals, steering.
- Keep `streamId == conversationId`, the epoch fence `createdAt`, the replayable chunk log and the `sync` snapshot. You can drop most of the mixed-version and protocol-v1 compatibility.

---

## 4. Persistence points (Mongo via `~/models` / `@librechat/data-schemas`)

1. **Job creation.** Redis/in-memory only (preliminary user message in metadata). No Mongo writes.
2. **User message.** `BaseClient.saveMessageToDatabase(userMessage)` → `db.saveMessage` plus `saveTurnConversation` (conversation upsert: endpoint, model, agent_id, title "New Chat", …) during `sendMessage`, in parallel with generation. It is re-saved idempotently in the success tail and in the abort/error paths.
3. **Title (immediate mode).** `addTitle` (`services/Endpoints/agents/title.js`):
   - `client.titleConvo` → `run.generateTitle(...)`, a separate small LLM call with a title-specific model/endpoint config.
   - Title cached in `getLogStores(TITLE)` for 120 s, then the `title` SSE event.
   - After `convoReady`, `saveConvo({title}, {noUpsert:true})` as a metadata-only write.
   - Only for the first turn (`parentMessageId === NO_PARENT`), a new convo, and not temporary.
4. **Response message.** `saveMessageToDatabase(responseMessage)` after the terminal claim, then a controller re-save with `unfinished` / `finish_reason`. Message fields (schema `packages/data-schemas/src/schema/message.ts`):
   - Identity and lineage: `messageId`, `conversationId`, `parentMessageId`, `user`, `sender`, `isCreatedByUser`.
   - Content and model: `text`, `content[]` (content parts), `model`, `endpoint`, `iconURL`, `tokenCount`.
   - Status: `unfinished`, `error`, `finish_reason`.
   - Extras: `attachments`, `files`, `metadata` (usage rollup), `contextMeta` (calibration/fading tiers), `manualSkills`, `quotes`, `addedConvo`, `expiredAt` (temporary chats), subagent projections.
5. **Partial response.**
   - On all-subscribers-left: `unfinished:true`.
   - On abort: in `beforePublish`.
   - On HITL pause: the unfinished row is saved and the conversation fence is kept.
6. **Error turn.** `saveErrorTurn` writes the user message, an `error:true` assistant message and the conversation.
7. **Token usage.** `recordCollectedUsage` (`packages/api/src/agents/usage.ts` ~813):
   - Per model call in `collectedUsage` (filled by the SDK `ModelEndHandler`; includes parallel agents, subagents and summarization), builds transaction docs via `prepareTokenSpend` / `prepareStructuredTokenSpend` (`transactions.ts`).
   - Prompt/completion/cache-read/cache-write split, priced via endpoint token config.
   - Written with `bulkWriteTransactions` (insertMany + one balance update).
   - Balance is pre-checked in `sendMessage` (`checkBalance`).
8. **Memory.**
   - `processMemory` / `createMemoryProcessor` (`packages/api/src/agents/memory.ts`) runs a **separate memory-agent LLM run** in parallel over recent messages, with `set_memory`/`delete_memory` tools → `db.setMemory` / `deleteMemory`.
   - Inline memory tools are also available to the main agent.
   - Memory artifacts are emitted as `attachment` events.
9. **Tool calls collection** (`schema/toolCall.ts`: `conversationId, messageId, toolId, user, result, attachments, blockIndex, partIndex`). Used for direct tool invocations (`POST /api/agents/tools/:toolId/call`) and late background code-execution results (`updateToolCallResult`). Normal in-run tool calls are persisted **inside `message.content`** as `tool_call` parts, not in this collection.
10. **LangGraph checkpoints.** Only when HITL/ask-user/event-actor is active. Stored in Mongo collections `agent_checkpoints` and `agent_checkpoint_writes` with a TTL (config `endpoints.agents.checkpointer {type:'mongo'|'memory', ttl, …}`). See §5.
11. **Files.** `updateFilesUsage` bumps usage on request files. Artifacts (code outputs, etc.) create File docs via `toolEndCallback`.

---

## 5. Agent run construction (`packages/api/src/agents/run.ts: createRun`)

**Inputs:**
- `agents[]`: primary plus connected/added agents, each an `InitializedAgent` with provider, `llmConfig`, instructions, `toolDefinitions`, `toolRegistry`, `maxContextTokens`, `edges`.
- `messages`, `indexTokenCountMap`, `initialSummary`, `calibrationRatio`, `fadingTier(s)`, `tokenCounter`, `customHandlers` (the event handlers above, wrapped by steer-offset / activity / reasoning-label wrappers), `signal`, `requestBody`, `user`, `tenantId`.
- Feature flags: `hitlCapable`, `steering`, `summarizationConfig`, `summarizeOnly`, `subagentTasks`, `runFiles`.

**Per-agent `AgentInputs`:**
- Model and endpoint: `provider`, `endpoint`, `clientOptions` (`llmConfig`), `reasoningKey`.
- Prompts: `instructions` (system), `additional_instructions`.
- Tools: `tools` (instances), `toolDefinitions` (schema-only, for event-driven execution), `toolRegistry`, `discoveredTools` (tool search), `graphTools`.
- Context: `maxContextTokens`, `summarizationEnabled/Config`, `contextPruningConfig`, `initialSummary`, `compactionSemanticIndex`, `maxToolResultChars`, `initialSessions` (code sandbox sessions).
- The ask-user tool is stripped when the caller cannot pause; background/intent arg injection is applied here.

**`graphConfig`:**
- `{signal, agents: agentInputs, edges}`.
- `type: 'multi-agent'` if more than one agent or any edges, else `'standard'`.
- `compileOptions.checkpointer = getAgentCheckpointer(...)` only when HITL, ask-user or event-actor checkpointing applies.

**`runConfig`:**
- `runId` (= responseMessageId), `graphConfig`, `tokenCounter`, `customHandlers`, `indexTokenCountMap`, `subagentUsageSink`, `subagentTasks`.
- `eagerEventToolExecution` (speculative tool execution while args stream, excluding file-write/code/ask tools), `codeSessionToolNames`.
- `langfuse` tracing, `toolOutputReferences`.
- `humanInTheLoop`, `hooks` (`HookRegistry`), `preemption`, `streamLimits`.
- Then `Run.create(runConfig)`.

**Tools (event-driven execution).** The graph holds **schema-only tool definitions**. When the model calls tools, the SDK emits `GraphEvents.ON_TOOL_EXECUTE`, handled by `createToolExecuteHandler` (`packages/api/src/agents/handlers.ts` ~5488). This handler:
- lazily loads the named tools (`loadTools(toolNames, agentId, configurable, …)`; MCP connections, actions/OpenAPI, built-ins such as web_search, file_search, execute_code/bash, read_file/create_file/edit_file, skills, memory);
- provisions files to the sandbox;
- executes with the run's abort signal;
- returns ToolMessages;
- supports **background execution** (`run_in_background` arg, `check_background_task` poll tool; `background.ts`, `backgroundCompletion.ts`).

`TOOL_END` results go through `toolEndCallback` to create attachments. Tool input validation errors are tracked. "Intent" labels (`intent.ts`) inject an optional first `intent` property into tool schemas for live status labels.

**Providers.** `initializeModel` / `getChatModelClass` from the SDK wrap LangChain chat models (ChatOpenAI, ChatAnthropic, ChatGoogleGenerativeAI/Vertex, ChatBedrockConverse, DeepSeek, xAI, Mistral, OpenRouter, custom OpenAI-compatible). LibreChat builds `llmConfig` per provider in `packages/api/src/endpoints/{openai,anthropic,google,bedrock,custom}`: API keys (user-provided or env), baseURL, headers, proxies, reasoning params, Responses-API toggle.

**Memory.** Two mechanisms:
1. A background memory-agent run per turn (above).
2. Stored memories injected into the system context (`getRequestMemories` / `buildInlineMemoryContext`), plus optional inline `set_memory`/`delete_memory` tools with a strict usage guard.

**Context window, pruning and compaction:**
- **Token counting.** The SDK `createTokenCounter` (ai-tokenizer / tiktoken) with cached counts per message (`indexTokenCountMap`) and a `calibrationRatio` learned from real provider usage (persisted in `contextMeta`).
- **Pruning.** The SDK pruner (`createPruneMessages` / `getMessagesWithinTokenLimit`) drops oldest messages to fit `maxContextTokens`.
- **Fading** (`fading.ts`): tiered masking/truncation of older tool results (`contextPruningConfig`). The chosen tier is "latched" and persisted in the message's `contextMeta`, so the next turn produces the same projection and keeps the provider prompt-cache prefix stable.
- **Summarization / compaction** (`compaction.ts`):
  - When enabled (`appConfig.summarization`, per-agent override; the provider/model may differ from the agent's), the SDK summarizes overflow into a `summary` content part (streamed via `on_summarize_*`).
  - The next turn's `loadHistory` starts from the latest usable summary part (`findCheckpointSummaryPart`) and passes it as `initialSummary`.
  - A manual `compact:true` request runs `summarizeOnly` (no user message, response parented to an existing message).
  - Summarization is disabled below a minimum viable context budget.
- **Step budget.** `recursionLimit` plus the `createStepBudgetHook` PostToolBatch hook. Exhaustion yields `unfinished` with `finish_reason = tool-call-limit`.

**HITL approvals** (`packages/api/src/agents/hitl/*`, `stream/ApprovalLifecycle.ts`):
- **Policy.** `endpoints.agents.toolApproval` (`enabled`, `mode`, allow/deny/ask globs), resolved by `resolveToolApprovalPolicy`. It becomes an SDK `PreToolUse` hook (`buildHITLRunWiring`), plus programmatic hooks (`registerToolApprovalHook`) and attached-code-environment hooks.
- **Pause.** An `ask` decision makes the SDK raise a LangGraph `interrupt()`, so it requires a checkpointer. `ask_user_question` is a tool whose body calls the SDK interrupt helper.
- **After the pause.** `processStream` returns and `run.getInterrupt()` has a payload. `AgentClient.handleRunInterrupt` builds a `PendingAction` with `actionId`, `interruptId`, `threadId`, `expiresAt` (approval TTL = checkpoint TTL), `requestFingerprint` (so a resume cannot swap agent/tools) and `resumeContext` (the ephemeral agent config and model params). Then:
  1. `approvals.pause` CAS: job `running → requires_action`.
  2. Emit `on_pending_action`.
  3. Persist the partial response as unfinished.
  4. The controller exits without a final event.
- **Resume.** Rebuilds a new `Run` on the same `thread_id` and namespace and calls `run.resume(decisionMap | answers)`. Decisions are `approve | reject | edit(editedArguments) | respond`.
- **Expiry.** The reaper `expireApproval` aborts the job; the TTL index cleans up checkpoints.
- **Checkpointer** (`checkpointer.ts`, `checkpoints/saver.ts` `OwnedMongoSaver extends MongoDBSaver`): a namespace per generation (private configurable key `__librechat_checkpoint_ns`, because LangGraph resets `checkpoint_ns`) and owner-scoped storage. `deleteAgentCheckpoint` runs on terminal transitions.

**Steering** (mid-run user messages; `packages/api/src/agents/steering/*`, `stream/SteeringLifecycle.ts`, `controllers/agents/steer.js`):
- `POST /chat/steer {conversationId, generationCreatedAt, text, files, quotes, clientSteerId, preempt}` enqueues into a per-stream FIFO (max depth 10, durable receipts).
- A `PostToolBatch` hook drains the queue at each tool-batch boundary. It appends a `steer` content part, emits `on_steer_applied` and returns `injectedMessages`; the SDK converts these to HumanMessages in graph state.
- `preempt:true` (or `/chat/steer/arm`) asks the owning replica (via a Redis `preempt` message) to seal the live model stream at the next `PreemptBoundary`.
- A `StopFinalize` terminal hook lets a steer that arrives just before the end continue the same run ("warm terminal steer continuation").
- Steers that never apply ride the final event as `pendingSteers`, or are parked and recovered as a new turn (`recoverySteerId`).
- Offset wrappers shift SDK content indices past host-spliced parts.

**Queued turns** (`queuedTurns.ts`, `queuedTurnHttp.ts`, schema `queuedTurn.ts`):
- A follow-up message sent while a generation is active is stored as a Mongo row (the FIFO authority), with a trigger delivery as the wakeup.
- Once the predecessor generation settles cleanly, it is admitted as an ordinary new turn: an internal loopback POST to the chat route with `agentContinuationAdmission`.
- Rows can be listed and cancelled via `/chat/queued-turns`.

**Subagents:**
- Config: `agent.subagents {enabled, allowSelf, shareFiles, agent_ids, graphs}`.
- The SDK exposes a `subagent` spawn tool (`Constants.SUBAGENT`) that runs a child graph with its own context, bounded depth and recursion multiplier. Child events surface as `on_subagent_update` and are aggregated per tool-call id; child usage flows through `subagentUsageSink` into billing.
- Detached (background) subagents: task store (`InMemorySubagentTaskStore` / `subagentTaskRouting.ts` with Redis routing to the owning process), durable **subagent threads** (child conversations, `subagentThreads.ts`, `services/Endpoints/agents/subagentThreadStore.js`) and **completion wakeups** (a durable `continue` trigger that starts a parent turn when the child finishes; `subagentCompletionWakeup.ts`).
- Lazy subagents resolve configs on first spawn.

**Handoffs and graphs** (`edges.ts`, `chain.ts`, `discovery.ts`):
- `GraphEdge {from, to, edgeType:'handoff'|'direct', prompt, excludeResults, description}`.
- `handoff` gives the agent transfer tools (dynamic routing). `direct` gives static edges that allow parallel fan-out/fan-in, with an optional prompt template using `{results}`.
- The legacy `agent_ids` chain becomes sequential direct edges (`createSequentialChainEdges`).
- Added-convo agents are extra start nodes and run in parallel; parallel parts carry `groupId` for column display.

**Other host features, briefly:**
- Activity/phase/reasoning labels: a fast model labels tool batches and reasoning.
- Skills: catalog injection and a `skill` tool (`skills.ts`, `skillFiles.ts`).
- Run files shared with subagents.
- Code execution contexts / attached environments (`execution.ts`) and sandbox prewarm.
- Event actors / triggers (`agents/triggers/*`, README): a durable, Mongo-queued source-neutral delivery of webhook/schedule/event work into agent turns, optionally continuing from a LangGraph checkpoint. That is the `checkpoint` strategy in `plan.ts`.

---

## 6. OpenAI-compatible and Responses API adapters

Both are mounted before JWT, with API-key auth (`requireRemoteAgentAuth`, schema `agentApiKey`), `checkRemoteAgentsFeature` and per-agent remote ACLs. Both wrap the request in a versioned, transport-safe `AgentRunEnvelope` (`packages/api/src/agents/envelope.ts`: `protocol`, `requestId`, `receivedAt`, `principal`, `payload`). They call `createRun` + `run.processStream` directly, **without** GenerationJobManager, resumability or HITL (`hitlCapable` is off, so approval-gated tools cannot pause).

**`POST /api/agents/v1/chat/completions`** (`controllers/agents/openai.js`, `packages/api/src/agents/openai/*`):
- `model` = agent id. Also accepts `messages`, `stream`, and optional `conversation_id` and `parent_message_id`.
- Handlers turn `on_message_delta` into `chat.completion.chunk` with `delta.content`, and `on_reasoning_delta` into `delta.reasoning`, written as `data: {...}\n\n` and ended with `data: [DONE]`.
- Tool calls are projected and **published only at successful completion** with complete id/name/args (README in `agents/openai/`).
- Non-streaming requests build a `chat.completion`.
- Usage is recorded with `recordCollectedUsage`. Messages are **not** persisted.
- `GET /v1/models` and `/v1/models/:model` list accessible agents.
- Also here: `POST /v1/events`, `/v1/events/bindings` (trigger ingress).

**`POST /api/agents/v1/responses`** (`controllers/agents/responses.js`, `packages/api/src/agents/responses/*`):
- Implements the Open Responses spec. Streaming events: `response.created`, `response.in_progress`, `response.output_item.added/done`, `response.content_part.added/done`, `response.output_text.delta/done`, `response.reasoning.delta/done`, `response.function_call_arguments.delta/done`, `response.completed/failed`, plus attachment events.
- `previous_response_id` = conversationId, loading prior stored messages.
- With `store:true`, it persists the conversation, input messages (as user text) and output text (`saveConversation`, `saveInputMessages`, `saveResponseOutput`).
- **Client-side function tools** (`clientTools.ts`): the model's call to a caller-declared tool ends the run and returns a `function_call` item. The caller continues statelessly by replaying the `function_call` + `function_call_output` items.
- `GET /v1/responses/:id` reconstructs a stored response; `GET /v1/responses/models` lists models.

---

## 7. What `@librechat/agents` provides, and Python substitutes

**What the repo uses it for:**
- **`Run`**: `Run.create(config)`, `processStream(input, config, {callbacks})`, `resume(value)`, `getInterrupt()`, `generateTitle()`, `Graph`/`graphRunnable.getState()`, `getRunSteps()`, `getFadingTier(s)`, preempt stats and halt reason.
- **Graphs**: `StandardGraph` (single agent: agent node ↔ tool node loop, `toolsCondition`) and a multi-agent graph (edges, handoff tools, parallel fan-out), both compiled LangGraph StateGraphs using `messagesStateReducer`.
- **Provider layer**: `Providers` enum, `initializeModel`, `getChatModelClass`, `isOpenAILike` over LangChain chat models, provider-specific reasoning keys, Anthropic prompt-cache handling and Google thought signatures.
- **Streaming abstraction**: LangChain stream callbacks turned into normalized `GraphEvents` (`ON_RUN_STEP`, `ON_RUN_STEP_DELTA`, `ON_RUN_STEP_COMPLETED`, `ON_RUN_STEP_CLOSED`, `ON_MESSAGE_DELTA`, `ON_REASONING_DELTA`, `CHAT_MODEL_END`, `TOOL_END`, `ON_TOOL_EXECUTE`, `ON_SUMMARIZE_*`, `ON_SUBAGENT_UPDATE`, `ON_CONTEXT_USAGE`, `ON_AGENT_LOG`), with the `RunStep` model (OpenAI-Assistants-like). Handlers: `ModelEndHandler`, `ToolEndHandler`, `ChatModelStreamHandler`, `HandlerRegistry`.
- **Content**: `createContentAggregator` (events → content parts), `formatAgentMessages` (stored content parts → LangChain messages, including tool-call/result pairing, steer and summary replay), `labelContentByAgent`, `createMetadataAggregator`.
- **Context management**: `createTokenCounter`, `getTokenCountForMessage`, `createPruneMessages`, `getMessagesWithinTokenLimit`, summarization node, fading tiers, image token estimators.
- **Tools**: event-driven tool execution with schema-only definitions, eager execution, `createToolSearch`, `Calculator`, web search, `ReadFile`/`Bash`/code-execution definitions, programmatic tool calling, `askUserQuestion(s)` interrupt helper, tool output references.
- **Hooks**: `HookRegistry` with `PreToolUse` (approval: allow/ask/deny), `PostToolUse`, `PostToolBatch` (steer injection, labels, step budget), `PreemptBoundary`, `Stop`/`StopFinalize`; `executeHooks`, `createToolPolicyHook`.
- **HITL**: `humanInTheLoop` config, LangGraph `interrupt`/`Command(resume)`, checkpointer integration.
- **Subagents**: spawn tool, `InMemorySubagentTaskStore`, `buildChildInputs`, subagent usage sink.
- **Event actor executor**, and Langfuse/OpenTelemetry tracing.

**Python substitution plan:**
- **Graph runtime.** `langgraph` (Python) `StateGraph` with `MessagesState` / `add_messages`, `ToolNode`/`tools_condition` (or a custom tool node that calls a host executor), `create_react_agent` as a starting point. Multi-agent via `langgraph-supervisor`/`langgraph-swarm`-style handoff tools (`Command(goto=...)`) or explicit direct edges and `Send` for parallel fan-out.
- **Checkpoints.** `langgraph-checkpoint-mongodb` (`MongoDBSaver`) or `langgraph-checkpoint-redis`/postgres. HITL uses `interrupt()` + `Command(resume=...)`, with `thread_id = conversationId` and a per-generation `checkpoint_ns`/thread suffix. TTL via a Mongo TTL index.
- **Providers.** `langchain-openai`, `langchain-anthropic`, `langchain-google-genai`/`langchain-google-vertexai`, `langchain-aws` (`ChatBedrockConverse`), `langchain-mistralai`, `langchain-deepseek`, `langchain-xai`; OpenAI-compatible custom endpoints via `ChatOpenAI(base_url=…)`. Alternatively `litellm` (or `langchain-litellm`'s `ChatLiteLLM`) as a single adapter with per-provider params and cost tables. That is useful for `recordCollectedUsage` pricing, but keep your own token-config table for parity.
- **Streaming events.** Use `graph.astream_events(version="v2")` or `astream(stream_mode=["messages","updates","custom"])`, then write a **ContentAggregator** that maps `on_chat_model_stream` chunks (text, reasoning, tool_call_chunks) and tool start/end to LibreChat's `RunStep` / `on_message_delta` / `on_run_step_delta` / `on_run_step_completed` schema. Reproducing the SDK's event contract is the main porting effort, because the frontend depends on it.
- **Token counting and pruning.** `tiktoken` / Anthropic `count_tokens`, `langchain_core.messages.trim_messages`, `langmem` or a custom summarization node for compaction.
- **Hooks.** Implement a small hook registry: PreToolUse (policy → interrupt), PostToolBatch (drain steer queue → return `Command(update={"messages":[HumanMessage...]})`), step budget via `recursion_limit`.
- **Tools.** LangChain `StructuredTool`, `langchain-mcp-adapters` for MCP, custom code-exec/file tools.
- **Tracing.** Langfuse Python `CallbackHandler`.

---

## 8. Suggested Python module layout

```
app/
  api/routes/agents/
    chat.py            # POST /api/agents/chat[/{endpoint}], /resume (dependency chain = middlewares)
    stream.py          # GET /chat/stream/{id} (SSE via sse-starlette), /chat/status, /chat/active, /chat/abort
    steer.py           # /chat/steer, /cancel, /arm ; queued_turns.py
    openai_compat.py   # /v1/chat/completions, /v1/models
    responses.py       # /v1/responses
  agents/
    request_controller.py  # ResumableAgentController: validation, idempotency, job create, background task
    resume_controller.py   # HITL resume
    client.py              # AgentClient: history load, build_messages, chat_completion, finalize, title
    initialize.py          # initialize_client / initialize_agent: provider config, tools, files, discovery
    load.py                # load_agent / load_ephemeral_agent
    discovery.py edges.py added.py   # graph edges, handoffs, parallel added convo
    run_factory.py         # create_run → LangGraph graph + config (checkpointer, hooks, limits)
    graph/                 # standard_graph.py, multi_agent_graph.py, tool_node.py (event-driven exec)
    events.py              # GraphEvents enum + RunStep models (pydantic) + ContentAggregator
    format_messages.py     # stored content parts <-> LangChain messages
    handlers.py            # event → SSE emit (default handlers), tool-end artifact callbacks
    context/               # token_counter.py, pruning.py, fading.py, compaction.py (summaries)
    hitl/                  # policy.py, pending_action.py, resume_mapping.py, ask_user_tool.py
    steering/              # queue drain hook, injection, preempt
    subagents/             # spawn tool, task store, threads, completion wakeup
    memory.py usage.py transactions.py title.py
  providers/               # openai.py anthropic.py google.py bedrock.py custom.py (llm_config builders) or litellm
  stream/
    job_manager.py         # GenerationJobManager (create/emit/subscribe/resume/abort/complete/approvals)
    job_store.py           # Protocol + memory_store.py + redis_store.py (hash + XADD log + Lua CAS)
    event_bus.py           # memory_bus.py + redis_bus.py (pub/sub channel stream:{id}:events, seq ordering)
    resume_state.py        # snapshot/replay (re-aggregate chunk log)
  checkpoint/              # mongo saver with owner/namespace scoping + TTL + delete
  db/                      # messages, conversations, transactions, memories, tool_calls, files (motor/beanie)
```

**Key invariants to preserve:**
- `streamId == conversationId`.
- The generation epoch (`createdAt`) fences all writes, emits and aborts.
- The POST returns early and SSE subscribes separately.
- Every emitted event is appended to a replayable log so reconnects get a `sync` snapshot.
- Terminal state is decided by a single CAS winner (complete / abort / error / pause), which alone publishes the `final`/`error` frame after persisting messages.
- User and response messages are idempotent upserts keyed by `messageId`, with `unfinished` / `error` flags.
- The conversation is upserted with the user message; the title is written metadata-only afterwards.
