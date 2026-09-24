# 聊天 / 智能体生成管线与流式传输

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

**范围与说明。** 本章只覆盖聊天与智能体生成的后端路径。`@librechat/agents` SDK 是外部 npm 包，没有内置（vendor）在本仓库中。第 7 节对 SDK 的描述来自本仓库对它的导入和使用方式，以及它在 `package-lock.json` 中的依赖列表（`@librechat/agents@3.9.1`，依赖 `@langchain/core 1.2.8`、`@langchain/langgraph 1.4.8`、`@langchain/openai`/`anthropic`/`google-*`/`aws`/`mistralai`/`deepseek`/`xai`、`openai`、`@anthropic-ai/sdk`、`ai-tokenizer`、`@langfuse/*`）。

整体设计：
- **所有聊天都是一次智能体运行。** 普通的 “OpenAI/Anthropic/…” 聊天会变成一个临时智能体。
- **HTTP POST 只负责启动一个后台任务。** 它立即返回 JSON。
- **流式传输是单独的 SSE GET**，订阅某个任务的事件总线。总线可以在内存中，也可以基于 Redis。
- **任务层**（`GenerationJobManager`）提供跨副本的可恢复性、中止、HITL 暂停、插话引导和幂等性。

代码库的防御性很强：纪元（epoch）栅栏、CAS、“终态认领（terminal claim）”。Python 移植可以保留这些概念，并大幅简化混合版本与滚动部署相关的代码。

---

## 1. 聊天 POST 的端到端时序

**挂载。** 在 `api/server/index.js` 中：
- 用 `createStreamServices()` 构建流服务，传给 `GenerationJobManager.configure({...})`，然后调用 `.initialize()`。这就是周期性的清理/回收器（约第 132–143 行）。
- 将 `/api/agents` 挂载到 `api/server/routes/agents/index.js`，并在 `/api/agents/chat` 上挂载启动遥测中间件和“就绪前拒绝”中间件。

**`api/server/routes/agents/index.js` 中的路由顺序：**
1. `/v1/responses`（Open Responses API）、`/v1/agents`（管理）、`/v1/skills` 和 `/v1`（OpenAI 兼容）。这些使用 API 密钥认证（`requireRemoteAgentAuth`），挂载在 JWT 之前。
2. `requireJwtAuth`、`checkBan`、`uaParser`。
3. 流/控制端点，它们跳过请求体中间件链：
   - `GET /chat/stream/:streamId`
   - `GET /chat/active`
   - `GET /chat/status/:conversationId`
   - `POST /chat/abort`
   - `POST /chat/steer`、`/chat/steer/deliver`、`/chat/steer/cancel`、`/chat/steer/arm`
   - `POST|GET /chat/queued-turns`、`DELETE /chat/queued-turns/:id`
4. `router.use('/', v1)`：智能体 CRUD、工具、actions。
5. 挂载在 `/chat` 的 `chatRouter`：`configMiddleware`、重试探测限流器、`detectGenerationRetry`、IP/用户消息限流器，然后是 `routes/agents/chat.js`。

**`POST /api/agents/chat[/:endpoint]` 的编号流程：**

1. **`routes/agents/chat.js` 中间件链。** 按以下顺序执行：
   1. `restoreResumeContext`：仅用于 `/resume`。它把暂停轮次的配置从 `job.metadata.pendingAction.resumeContext` 回放到请求体上。
   2. `createMessageFilterPii`：PII 过滤，在内容审核之前执行。
   3. `moderateText`。
   4. `checkAgentAccess`：角色权限 `AGENTS.USE`。
   5. `canAccessAgentFromBody`：对 `agent_id` 的 ACL VIEW 检查。
   6. `validateConvoAccess`。
   7. `guardSubagentThreadTurn`：阻止人工写入子智能体的子线程。
   8. `buildEndpointOption`。

   路由：`POST /resume` → `ResumeController`；`POST /` 和 `POST /:endpoint` → `AgentController(req,res,next, initializeClient, addTitle)`。`/:endpoint` 形式用于临时智能体。

2. **`buildEndpointOption`**（`api/server/middleware/buildEndpointOption.js`）：
   - 用 `parseCompactConvo` 解析请求体，并强制执行模型规格。
   - 调用 `api/server/services/Endpoints/agents/build.js` 中的 `buildOptions`。它启动一个 **promise**：`agent: loadAgent({ agent_id: isAgentsEndpoint ? agent_id : EPHEMERAL_AGENT_ID, ... })`。
   - `loadAgent` / `loadEphemeralAgent`（`packages/api/src/agents/load.ts`）为非智能体端点合成一个智能体：
     - id = `encodeEphemeralAgentId({endpoint, model, sender})`，provider = endpoint，instructions = `promptPrefix`。
     - 工具来自 UI 开关和模型规格：`execute_code`、`file_search`、`web_search`、`memory`、`ask_user_question`、MCP 服务器（`mcp_all` 哨兵值或显式的工具名）、后台/意图（intent）工具选项、`subagents`、`skills`。
   - 结果存放在 `req.body.endpointOption`。

3. **`ResumableAgentController`**（`api/server/controllers/agents/request.js`，第 711 行起）：
   - 通过请求头 `x-librechat-generation-protocol`（1 或 2；`controllers/agents/protocol.js`）协商生成协议。
   - 校验 `clientRequestId`（幂等键，1–128 个字符）、`overrideUserMessageId`/`overrideConvoId`、`expectedPredecessorCreatedAt`、插话恢复 id，以及 `compact`（手动压缩）。
   - **conversationId 分配。** 新对话时：`uuidv5(userId:clientRequestId)`（幂等）或 `randomUUID()`。**始终 `streamId === conversationId`。** 每个对话只有一个活跃的生成。
   - 拒绝仅为预分配（未保存）id 的 `parentMessageId`。
   - **幂等性。** `GenerationJobManager.claimGeneration(userId, clientRequestId, streamId, …)`。如果该认领已有活跃任务，则返回 `{streamId, status:'resumed'}`（去重）。过期的认领通过 `takeoverGeneration` 接管。
   - **并发限流器。** `checkAndIncrementPendingRequest(userId)` → 429，并记录一次违规。
   - 预分配 `userMessageId` / `responseMessageId`，使 MCP 请求体占位符保持稳定。

4. **任务创建。** `GenerationJobManager.createJob(streamId, userId, conversationId, { initialMetadata: {...} })`：
   - 元数据包括 endpoint、iconURL、model、`agent_id`、`isTemporary`、`retentionExpiresAt`、`responseMessageId`、预分配的 `userMessage`、`mcpRequestBody`、`isRegenerate`、定时任务字段，以及能力标志（`preemptCapable`、`steerQuotesCapable`）。
   - 创建一个由任务持有的 `AbortController`，并记录 `jobCreatedAt`，即**生成纪元**，用来为之后的每次写入设置栅栏。

5. **提前返回 HTTP 响应。** `sendGenerationStarted()` 响应 `200 {streamId, conversationId, generationCreatedAt, status:'started', generationProtocolVersion}`。之后生成在后台继续。排队轮次的回环（loopback）请求会把这一步推迟到提供商开始输出之后。

6. **断连钩子。** `job.emitter.on('allSubscribersLeft', …)`：如果所有 SSE 客户端在运行中途断开，就把部分响应以 `unfinished:true` 保存到 Mongo。

7. **`initializeClient`**（`api/server/services/Endpoints/agents/initialize.js`），调用时传入 `signal: job.abortController.signal`：
   - 调用 SDK 的 `createContentAggregator()` → `{contentParts, aggregateContent, stepMap}`。
   - 为产物（artifact）调用 `createToolEndCallback`（代码输出、文件引用、网页搜索结果、记忆产物）。
   - 为主智能体调用 `initializeAgent(...)`（`packages/api/src/agents/initialize.ts`）：
     - 通过 `getProviderConfig` → `providerConfigMap`（`packages/api/src/endpoints/config/providers.ts`）解析提供商：`openAI`/`azureOpenAI` → `initializeOpenAI`；`anthropic`、`google`/`vertexai`、`bedrock`；`xai`/`deepseek`/`moonshot`/`openrouter` → `initializeCustom`。然后由 `getOptions()` 生成 `llmConfig`。
     - 预备文件和资源（`primeResources`：附件、代码环境供给、向量数据库）。
     - 加载工具（`loadAgentTools` / `loadToolsForExecution`），并注入技能目录。
     - 计算 `maxContextTokens = agentMax − maxOutputTokens`（默认 32000）。
   - `discoverConnectedAgents`（`packages/api/src/agents/discovery.ts`）遍历图的**边**（handoff/direct）和旧版 `agent_ids` 链，并把每个相连的智能体初始化到 `agentConfigs` 中。
   - `processAddedConvo`（`addedConvo.js`、`packages/api/src/agents/added.ts`）：“多对话”并排显示的智能体（`ADDED_AGENT_ID`）作为一个没有入边的额外起始节点加入，因此它**并行**运行。
   - 解析子智能体树（延迟加载的子智能体描述符，`lazySubagents.ts`）。
   - `getDefaultHandlers({ res, aggregateContent, contentParts, stepMap, toolEndCallback, collectedUsage, streamId, jobCreatedAt, toolExecuteOptions, summarizationOptions, … })`（`api/server/controllers/agents/callbacks.js`）。
   - `new AgentClient({... agentConfigs, eventHandlers ...})`（`api/server/controllers/agents/client.js`，继承 `api/app/clients/BaseClient.js`）。

8. **回到控制器：**
   - 如果已被中止，调用 `completeJob`。
   - `resolveAgentTurnExecutionPlan`（`packages/api/src/agents/plan.ts`）选择状态加载策略：`fresh`（新对话）、`history`（默认：从 Mongo 消息重建）或 `checkpoint`（仅用于事件执行者（event-actor）轮次）。
   - `resolveTitleTiming` 返回 `immediate`（默认）或 `final`。
   - 存储 `sender`，然后调用 `GenerationJobManager.setContentParts(streamId, client.contentParts)`。

9. **`startGeneration()`** 构建 `messageOptions`，包含：
   - `onStart(userMsg, responseMessageId)`：更新任务元数据 `{userMessage, responseMessageId}` 并**发出 `created` 事件**。
   - `getReqData`：观察用户消息写入的 promise。
   - `beforeResponsePersistence: claimBeforeResponsePersistence`：通过 `claimTerminalJob(... 'complete'|'aborted', {persistencePending:true})` 认领终态。
   - `abortController: job.abortController`。

   然后调用 `client.sendMessage(text, messageOptions)`；对于绑定触发器的轮次，由事件执行者轮次适配器包裹。如果符合条件（首轮、非临时对话）且时机为 `immediate`，它还会**并行**启动 `addTitle(req, {immediate:true, convoReady, onTitleGenerated: emitTitleEvent})`。

10. **`BaseClient.sendMessage` → `sendReservedMessage`**（`api/app/clients/BaseClient.js` 约 741–1255 行）：
    - `handleStartMethods` → `setMessageOptions`、`createUserMessage` 和 `loadHistory(conversationId, parentMessageId)`。`loadHistory` 读取对话的全部消息并沿父链遍历（`getMessagesForConversation`）。启用摘要时，它从最新的可用摘要片段开始（`findCheckpointSummaryPart`）。
    - `onStart` → `created` 事件。
    - `AgentClient.buildMessages`（client.js 约 2285 行）：
      - 每条消息的 token 数 → `indexTokenCountMap`。
      - 附件：图片、文档、文本文件，以及 file_search 的“增强提示（augmented prompt）”上下文。
      - 记忆（`useMemory`、`getRequestMemories`）。
      - 共享的运行上下文和 MCP 服务器指令通过 `applyContextToAgent`（`packages/api/src/agents/context.ts`）合并进每个智能体的系统提示。
    - **保存用户消息：** `saveMessageToDatabase(userMessage)` → `db.saveMessage` + `saveTurnConversation`（对话 upsert，`packages/api/src/conversations/save.ts`）。异步触发；其 promise 通过 `getReqData` 上报。
    - `checkBalance`（余额预留）。
    - **`sendCompletion` → `AgentClient.chatCompletion`**（client.js 第 4404 行起）：
      - 构建 LangGraph 配置：`thread_id=conversationId`、私有检查点命名空间键、`user_id`、`requestBody`、`hide_sequential_outputs`、`last_agent_index`、`recursionLimit`、`signal`、`streamMode:'values'`、`version:'v2'`。
      - 用 SDK 的 `formatAgentMessages` 转换历史（TMessage 内容片段 → LangChain `BaseMessage[]`，外加 token 映射和摘要）。
      - 并行启动记忆提取（`runMemory`）。
      - 调用 `createRun({...})`（`packages/api/src/agents/run.ts`；§5），然后 `GenerationJobManager.setGraph(streamId, run.Graph)`，然后 `run.processStream({messages}, config, {callbacks})`。
      - `handleRunInterrupt(run, streamId)` 处理 HITL 暂停。
      - 之后：结算活动标签，等待记忆完成，并调用 `recordCollectedUsage`（token 交易和余额）。
    - 构建 `responseMessage`：`messageId=responseMessageId`、`parentMessageId=userMessageId`、`content=contentParts`、`model`、`sender`、`tokenCount`、来自已 await 的产物 promise 的 `attachments`、`contextMeta` 和用量元数据。
    - 调用 `opts.beforeResponsePersistence(responseMessage)`（终态认领），然后 `saveMessageToDatabase(responseMessage)`（`databasePromise`）。

11. **控制器成功收尾**（request.js 约 2940–3240 行）：
    - 幂等地重新保存用户消息和响应（中止、抢占未完成或达到步数上限时标记 `unfinished`；步数预算耗尽时为 `finish_reason: TOOL_CALL_LIMIT`）。
    - 修复对话的消息引用，并 resolve `convoReady`（解除标题持久化的阻塞）。
    - 结算定时运行。
    - 构建**最终事件** `{final:true, conversation, title, requestMessage, responseMessage, pendingSteers?}`，并通过 `GenerationJobManager.publishTerminalClaim(terminalClaim, finalEvent)` 发布（任务 → `complete`/`aborted`）。
    - `finishResumableRequest` 递减并发计数器。
    - 在 `final` 标题模式下，此时运行 `addTitle`。然后 `disposeClient`。

12. **错误路径。** `completeJob(streamId, errorMessage, jobCreatedAt, { beforeErrorPublication: () => saveErrorTurn(...) })` 保存用户消息、一条 `error:true` 的响应消息和对话记录，然后发布 SSE `error` 事件。

13. **客户端流。** 客户端另外打开 `GET /api/agents/chat/stream/:streamId?generationCreatedAt=…`（重连时带 `&resume=true`），接收下面列出的事件（§2、§3）。

**HITL 恢复流程。** `POST /api/agents/chat/resume` `{conversationId, actionId, generationCreatedAt, decisions[] | answer(s)}` → `api/server/controllers/agents/resume.js`：
1. 校验所有者/租户，并对照待处理动作检查智能体和请求指纹。
2. `resolveResumeValue`：`tool_approval` → `mapToolApprovalResolutions`；`ask_user_question` → `mapAskUserAnswers`（`packages/api/src/agents/hitl/resume.ts`）。
3. `GenerationJobManager.approvals.resolve(...)`：原子的单一胜者认领，`requires_action` → `running`。
4. 发送 ACK，然后用 `initializeClient` 重建客户端。
5. `client.resumeCompletion` → 使用相同的 `thread_id` 和检查点存储调用 `createRun` → `run.resume(resumeValue)`。事件通过**同一个 streamId** 流出，因此现有的 SSE 连接会继续。
6. 收尾（保存响应、发布最终事件、删除检查点），或再次暂停。

---

## 2. SSE 事件类型与负载结构

**帧格式**（`routes/agents/index.js` 中的 `writeEvent`）：每一帧都是 `event: message\ndata: <json>\n\n`，错误除外，错误使用 `event: error\ndata: {"error": "...", "generationProtocolVersion": n}`。
- 响应头：`Content-Type: text/event-stream`、`Cache-Control: no-cache, no-transform`、`X-Accel-Buffering: no`、`Content-Encoding: identity`、`x-librechat-generation-protocol`。
- JSON 是以下三种结构之一：(a) “控制”对象（`created`、`sync`、`final`）；(b) 用于图事件和宿主事件的 `{event: <name>, data: <payload>}`；(c) 原始错误。

| 事件 | 发出方 | 负载 |
|---|---|---|
| `{created:true, message:<userMessage + files/manualSkills>, streamId}` | request.js `onStart` | 带 id 的用户消息；客户端据此渲染用户气泡和预分配的响应 |
| `{sync:true, resumeState, pendingEvents}` | `resume=true` 时的流路由 | `ResumeState`（`stream/interfaces/IJobStore.ts` 约 863 行）：`runSteps[]`、`aggregatedContent[]`、`userMessage`、`responseMessageId`、`conversationId`、`sender`、`iconURL`、`model`、`titleEvent`、`replayEvents[]`、`pendingOAuthPrompts`、待处理动作/插话、用量 |
| `on_run_step` | SDK `GraphEvents.ON_RUN_STEP`，经由 `getDefaultHandlers` | `RunStep {id, runId, agentId, index, stepIndex, groupId?, type: 'message_creation'|'tool_calls', stepDetails:{message_creation:{message_id}} | {tool_calls:[{id,name,args,mcpServerName?}]}, created_at, status}`（`packages/data-provider/src/types/agents.ts` 约 231 行） |
| `on_run_step_delta` | SDK | `{id, delta:{type:'tool_calls', tool_calls:[ToolCallChunk], auth?, expires_at?, approval?}}`（工具参数流式输出和 MCP OAuth 提示） |
| `on_run_step_completed` | SDK | `{result:{id, index, tool_call:{id,name,args,output,inputValidationError?}}}` |
| `on_run_step_closed` | SDK | `{id, status, created_at, completed_at}`；宿主把 `runStepStatus`/`runStepDurationMs` 写到对应片段上 |
| `on_message_delta` | SDK | `{id: stepId, delta:{content:[{type:'text', text}]}}` |
| `on_reasoning_delta` | SDK | `{id, delta:{content:[{type:'think', think}]}}` |
| `on_agent_update` | 宿主，启用 `hide_sequential_outputs` 时 | `{runId, message:"<Agent> is thinking..."}` |
| `on_summarize_start` / `_delta` / `_complete` | SDK 摘要节点 | 摘要内容片段的进度（压缩） |
| `on_context_usage` / `on_token_usage` | `UsageEvents`；`ModelEndHandler` → `emitTokenUsage` | 每次调用的用量 `{input_tokens, output_tokens, cache…, model, cost?}`；上下文窗口仪表 |
| `on_pending_action` | `AgentClient.exposePendingApproval` 中的 `ApprovalEvents` | `PendingAction {actionId, streamId, conversationId, runId, responseMessageId, payload: ToolApprovalInterruptPayload | AskUserQuestionInterruptPayload, createdAt, expiresAt}` 的客户端投影 |
| `on_steer_applied` / `on_steer_updated` | `SteerEvents` | 在工具边界注入的插话（内容片段 `type:'steer'`）；能力更新（仅 v2） |
| `on_subagent_update` | SDK `ON_SUBAGENT_UPDATE` | 子智能体进度信封 `{runId, parentRunId, subagentRunId, subagentType, phase, data, label, …}` |
| `on_activity_label`、`on_reasoning_label`（+ 内部的 `_attempt`） | 宿主的快速模型标注器 | 工具批次或推理片段的标签 `{index, stepId, label…}` |
| `on_sandbox_starting`、`on_ptc_tool_call` | `StepEvents` | 代码沙箱冷启动通知；程序化工具调用的进度 |
| `attachment` | callbacks.js 中的 `writeAttachment` | `TAttachment`（文件元数据、`toolCallId`、`messageId`、类型，例如 `web_search`、`memory`、代码文件） |
| `title` | request.js `emitTitleEvent` | `{conversationId, title}` |
| `{final:true, conversation, title, requestMessage, responseMessage, pendingSteers?}` | `publishTerminalClaim` | 终态。中止变体：`{final:true, aborted:true, conversation:{conversationId}|null, requestMessage, responseMessage:{messageId, content, unfinished:true…}}`（GenerationJobManager 约 4700 行）。对账（reconcile）变体 `{final:true, reconcile:true, reconcileReason}` 通知客户端重新拉取 |
| `event: error` | `completeJob`/`emitError` | `{error: string}` |

枚举定义在 `packages/data-provider/src/types/runs.ts`（`StepEvents`、`UsageEvents`、`ApprovalEvents`、`SteerEvents`、`ActivityLabelEvents`、`ReasoningLabelEvents`）。

**可见性规则**（`checkIfLastAgent`）。在启用 `hide_sequential_outputs` 的多智能体图中，只转发最后一个智能体的消息/推理增量。工具调用步骤总是会转发。

**内容聚合。** `aggregateContent({event, data})`（SDK `createContentAggregator`）把每个事件折叠进 `contentParts[]`，按 `runStep.index` 索引。片段类型包括 `text`、`think`、`tool_call`、`image_file`、`summary`、`steer`、`agent_update` 以及标签片段。`contentParts` 最终成为 `responseMessage.content`。

---

## 3. 流任务的生命周期、可恢复性与中止

**文件：** `packages/api/src/stream/GenerationJobManager.ts`（约 9.5k 行）、`interfaces/IJobStore.ts`、`implementations/{InMemoryJobStore,RedisJobStore,InMemoryEventTransport,RedisEventTransport}.ts`、`createStreamServices.ts`、`ApprovalLifecycle.ts`、`SteeringLifecycle.ts`、`internal/coalescing.ts`。

**组成。** `GenerationJobManager` 组合了两个服务：
- **jobStore**：任务元数据、内容状态和插话队列。
- **eventTransport**：发布/订阅。

当 `USE_REDIS_STREAMS`（默认取 `USE_REDIS`）开启且有可用客户端时，`createStreamServices()` 使用 Redis。它为订阅者复制一条连接；否则回退到内存实现。运行时状态（`RuntimeJobState`：中止控制器、早期事件缓冲、订阅者处理器、序号计数器、抢占状态）始终是进程本地的。

**任务记录**（`SerializableJobData`）：
- 身份与状态：`streamId`、`userId`、`tenantId`、`status: running|requires_action|complete|error|aborted`、`createdAt`（纪元）、`conversationId`、`generationProtocolVersion`。
- 轮次数据：`userMessage`、`responseMessageId`、`sender`、`endpoint`、`iconURL`、`model`、`mcpRequestBody`。
- 序列化负载：`finalEvent`、`titleEvent`、`replayEvents`、`tokenUsage`、`contextUsage`、`contextMeta`。
- HITL：`pendingAction`、`resolvedAskUserQuestions`。
- 幂等性与提供商执行：幂等认领令牌、`providerExecutionId` 和排空（drain）标志。
- 标志：`terminalPersistencePending`、`steersClosed`、`syncSent`、`createdEventEmitted`、订阅者租约，以及定时任务/事件执行者字段。

**Redis 键**（为 Cluster 加了哈希标签）：
- `stream:{id}:job`：hash。
- `stream:{id}:chunks`：一个 Redis **Stream**（XADD 追加日志，记录每个发出的事件）。
- `stream:{id}:runsteps`、`stream:{id}:seq`（序号计数器）、`stream:{id}:generation-epoch`。
- `stream:{id}:steers`、`:steers-claimed`、`:parked`、`:steer-receipts`。
- `stream:{id}:subscriber-leases:<createdAt>`。
- 全局集合：`stream:running`、`stream:requires_action`。
- `stream:user:{tenant:user}:jobs`、`stream:idem:<user:clientRequestId>`。
- 发布/订阅频道 `stream:{id}:events` 承载消息类型 `chunk | chunk_batch | done | error | abort | abort_ack | preempt | subscription_frontier`，每条都带 `seq` 和 `generationId`（= createdAt）。订阅者按 `seq` 重新排序，使用基于超时的重排缓冲区。
- 大多数状态变更是 Lua 脚本（对 `createdAt`/status 做 CAS）。

**发出事件。** `emitChunk(streamId, event, {expectedCreatedAt})` 受生成纪元栅栏保护；如果运行时已被替换，它会丢弃事件。
- Redis：先 `jobStore.appendChunk`（XADD），再 `eventTransport.emitChunk`（PUBLISH）。把增量合并成 `chunk_batch` 是可选的。
- 内存：在第一个订阅者接入之前，事件进入**早期事件缓冲区**。内容保存在指向 SDK 图的 WeakRef 和宿主的 `contentParts` 上。
- `created` 事件总是排在所有其他事件之前序列化。

**订阅。** `GET /chat/stream/:streamId`：
- 检查所有权和租户，以及纪元（查询参数 `generationCreatedAt` → 409 `GENERATION_REPLACED`）。
- 首次接入使用 `subscribe(streamId, onChunk, onDone, onError)`：回放早期缓冲区，然后切换为实时。
- `resume=true` 时使用 `subscribeWithResume`：对 `getResumeState()` 做快照，写入一个 `sync` 帧（运行步骤 + 聚合内容 + 快照与接入之间捕获的待处理事件），然后调用 `subscription.activate()`。
- 在 Redis 模式下，`getResumeState` **通过把 XADD 分块日志回放进一个新的 `createContentAggregator` 来重建聚合内容**，除非同一进程持有实时图（`RedisJobStore.getContentParts`）。
- 如果因为运行已结束或已被替换而订阅失败，它会发送一个 `reconcile` 最终帧。
- `req.on('close')` 取消订阅。当最后一个本地订阅者离开时，`allSubscribersLeft` 保存部分响应。

**状态与活跃端点：**
- `GET /chat/status/:conversationId` → `{active, status, streamId, aggregatedContent, createdAt, elapsedMs, resumeState, pendingAction, unrecoveredSteers}`。客户端在重新加载后用它决定是否重新接入。
- `GET /chat/active` → 该用户的活跃任务 id。

**终态化：**
- `claimTerminalJob` 是一次 CAS：`running → complete|aborted`，带 `persistencePending`。
- `publishTerminalClaim(claim, finalEvent)` 把 `finalEvent` 存进任务，以便迟到的订阅者回放，然后发布 `done`。
- `finishTerminalJob` 负责清理和 TTL（内存中为 `ttlAfterComplete`；Redis 为 EXPIRE）。
- `completeJob(streamId, error)` 处理错误。
- 周期性的 `cleanup()` 回收过期任务：`staleJobTimeout` 在内存模式下默认 20 分钟；过期的审批也会被回收。

**跨副本中止。** `POST /chat/abort` `{streamId|conversationId|abortKey, generationCreatedAt}`（`routes/agents/index.js` 约 569 行）：
1. 解析任务（对于 `'new'` 占位符，可以找到唯一的活跃任务）。
2. 检查所有者和纪元。
3. `GenerationJobManager.abortJob(streamId, { expectedCreatedAt, transformAbortContent, beforePublish })`：
   - 用 CAS 把状态改为 `aborted`。
   - Redis：在 `stream:{id}:events` 上发布 `abort`。**持有该任务的副本**上的 `onAbort` 处理器触发 `job.abortController.abort()`，它作为 LangGraph 运行的 `signal` 向下传播，并写入一个 `abort-ack` 键。请求方可以等待该 ack 或等待提供商排空（`awaitProviderDrain`）。
   - 重建内容（Redis 模式下来自分块日志）和文本，并收集用量。
   - `beforePublish`：**路由把用户消息和 `unfinished:true` 的部分响应持久化到 Mongo**，并删除 HITL 检查点。
   - 发布中止的最终事件。
   - 响应：视情况返回 409 `RUN_REPLACED` / `RUN_STILL_ACTIVE`。
4. 在持有任务的副本上，`sendMessage` 提前返回。由于中止已经胜出，终态认领失败，因此不会重复发布。

**幂等性。** 基于 `stream:idem:*` 键（带认领令牌）的 `claimGeneration` / `resumeClaimedGeneration` / `takeoverGeneration` / `releaseGeneration`，防止响应丢失后出现重复 POST。

**Python 草图：**
- `JobStore` 协议（Redis hash + Redis Stream + Lua CAS，或内存字典）。
- `EventBus`（带 seq 的 Redis 发布/订阅，或基于 `asyncio.Queue` 的扇出）。
- `JobManager`：create、emit、subscribe(resume)、abort、complete、审批、插话引导。
- 保留 `streamId == conversationId`、纪元栅栏 `createdAt`、可回放的分块日志和 `sync` 快照。大部分混合版本和 protocol-v1 兼容代码可以去掉。

---

## 4. 持久化时机（通过 `~/models` / `@librechat/data-schemas` 写入 Mongo）

1. **任务创建。** 只写 Redis/内存（预分配的用户消息放在元数据中）。不写 Mongo。
2. **用户消息。** 在 `sendMessage` 期间，与生成并行执行 `BaseClient.saveMessageToDatabase(userMessage)` → `db.saveMessage` 加 `saveTurnConversation`（对话 upsert：endpoint、model、agent_id、标题 “New Chat”、…）。在成功收尾以及中止/错误路径中会幂等地重新保存。
3. **标题（immediate 模式）。** `addTitle`（`services/Endpoints/agents/title.js`）：
   - `client.titleConvo` → `run.generateTitle(...)`，这是一次独立的小型 LLM 调用，使用标题专用的模型/端点配置。
   - 标题在 `getLogStores(TITLE)` 中缓存 120 秒，然后发出 `title` SSE 事件。
   - 在 `convoReady` 之后，以仅写元数据的方式执行 `saveConvo({title}, {noUpsert:true})`。
   - 仅适用于首轮（`parentMessageId === NO_PARENT`）、新对话且非临时对话。
4. **响应消息。** 在终态认领之后执行 `saveMessageToDatabase(responseMessage)`，然后控制器带上 `unfinished` / `finish_reason` 再保存一次。消息字段（schema `packages/data-schemas/src/schema/message.ts`）：
   - 身份与谱系：`messageId`、`conversationId`、`parentMessageId`、`user`、`sender`、`isCreatedByUser`。
   - 内容与模型：`text`、`content[]`（内容片段）、`model`、`endpoint`、`iconURL`、`tokenCount`。
   - 状态：`unfinished`、`error`、`finish_reason`。
   - 其他：`attachments`、`files`、`metadata`（用量汇总）、`contextMeta`（校准/淡化层级）、`manualSkills`、`quotes`、`addedConvo`、`expiredAt`（临时聊天）、子智能体投影。
5. **部分响应。**
   - 所有订阅者离开时：`unfinished:true`。
   - 中止时：在 `beforePublish` 中。
   - HITL 暂停时：保存未完成的记录，并保留对话栅栏。
6. **错误轮次。** `saveErrorTurn` 写入用户消息、一条 `error:true` 的助手消息和对话。
7. **token 用量。** `recordCollectedUsage`（`packages/api/src/agents/usage.ts` 约 813 行）：
   - 针对 `collectedUsage` 中的每次模型调用（由 SDK 的 `ModelEndHandler` 填充；包括并行智能体、子智能体和摘要），通过 `prepareTokenSpend` / `prepareStructuredTokenSpend`（`transactions.ts`）构建交易文档。
   - 拆分为 prompt/completion/cache-read/cache-write，按端点的 token 配置计价。
   - 通过 `bulkWriteTransactions` 写入（insertMany + 一次余额更新）。
   - 余额在 `sendMessage` 中预先检查（`checkBalance`）。
8. **记忆。**
   - `processMemory` / `createMemoryProcessor`（`packages/api/src/agents/memory.ts`）在最近的消息上并行执行一次**独立的记忆智能体 LLM 运行**，带 `set_memory`/`delete_memory` 工具 → `db.setMemory` / `deleteMemory`。
   - 主智能体也可以使用内联的记忆工具。
   - 记忆产物以 `attachment` 事件发出。
9. **工具调用集合**（`schema/toolCall.ts`：`conversationId, messageId, toolId, user, result, attachments, blockIndex, partIndex`）。用于直接工具调用（`POST /api/agents/tools/:toolId/call`）和迟到的后台代码执行结果（`updateToolCallResult`）。运行中的普通工具调用以 `tool_call` 片段的形式持久化在 **`message.content` 内部**，不在这个集合中。
10. **LangGraph 检查点。** 仅在 HITL/ask-user/事件执行者生效时写入。存储在 Mongo 集合 `agent_checkpoints` 和 `agent_checkpoint_writes` 中，带 TTL（配置 `endpoints.agents.checkpointer {type:'mongo'|'memory', ttl, …}`）。见 §5。
11. **文件。** `updateFilesUsage` 增加请求文件的使用计数。产物（代码输出等）通过 `toolEndCallback` 创建 File 文档。

---

## 5. 智能体运行的构建（`packages/api/src/agents/run.ts: createRun`）

**输入：**
- `agents[]`：主智能体加上相连/附加的智能体，每个都是 `InitializedAgent`，包含 provider、`llmConfig`、instructions、`toolDefinitions`、`toolRegistry`、`maxContextTokens`、`edges`。
- `messages`、`indexTokenCountMap`、`initialSummary`、`calibrationRatio`、`fadingTier(s)`、`tokenCounter`、`customHandlers`（上文的事件处理器，外面包了插话偏移 / 活动 / 推理标签包装器）、`signal`、`requestBody`、`user`、`tenantId`。
- 功能标志：`hitlCapable`、`steering`、`summarizationConfig`、`summarizeOnly`、`subagentTasks`、`runFiles`。

**每个智能体的 `AgentInputs`：**
- 模型与端点：`provider`、`endpoint`、`clientOptions`（`llmConfig`）、`reasoningKey`。
- 提示：`instructions`（系统）、`additional_instructions`。
- 工具：`tools`（实例）、`toolDefinitions`（仅 schema，用于事件驱动执行）、`toolRegistry`、`discoveredTools`（工具搜索）、`graphTools`。
- 上下文：`maxContextTokens`、`summarizationEnabled/Config`、`contextPruningConfig`、`initialSummary`、`compactionSemanticIndex`、`maxToolResultChars`、`initialSessions`（代码沙箱会话）。
- 当调用方无法暂停时，会移除 ask-user 工具；后台/意图参数注入也在这里应用。

**`graphConfig`：**
- `{signal, agents: agentInputs, edges}`。
- 多于一个智能体或存在任何边时 `type: 'multi-agent'`，否则为 `'standard'`。
- 仅当适用 HITL、ask-user 或事件执行者检查点时，才设置 `compileOptions.checkpointer = getAgentCheckpointer(...)`。

**`runConfig`：**
- `runId`（= responseMessageId）、`graphConfig`、`tokenCounter`、`customHandlers`、`indexTokenCountMap`、`subagentUsageSink`、`subagentTasks`。
- `eagerEventToolExecution`（在参数流式输出时推测性地执行工具，不包括文件写入/代码/询问类工具）、`codeSessionToolNames`。
- `langfuse` 链路追踪、`toolOutputReferences`。
- `humanInTheLoop`、`hooks`（`HookRegistry`）、`preemption`、`streamLimits`。
- 然后调用 `Run.create(runConfig)`。

**工具（事件驱动执行）。** 图中只保存**仅 schema 的工具定义**。模型调用工具时，SDK 发出 `GraphEvents.ON_TOOL_EXECUTE`，由 `createToolExecuteHandler`（`packages/api/src/agents/handlers.ts` 约 5488 行）处理。这个处理器：
- 延迟加载指定名称的工具（`loadTools(toolNames, agentId, configurable, …)`；MCP 连接、actions/OpenAPI、内置工具，如 web_search、file_search、execute_code/bash、read_file/create_file/edit_file、技能、记忆）；
- 把文件供给到沙箱；
- 使用运行的中止信号执行；
- 返回 ToolMessage；
- 支持**后台执行**（`run_in_background` 参数、`check_background_task` 轮询工具；`background.ts`、`backgroundCompletion.ts`）。

`TOOL_END` 结果经过 `toolEndCallback` 生成附件。工具输入校验错误会被记录。“意图（intent）”标签（`intent.ts`）会在工具 schema 中注入一个可选的首个 `intent` 属性，用于实时状态标签。

**提供商。** SDK 中的 `initializeModel` / `getChatModelClass` 封装了 LangChain 聊天模型（ChatOpenAI、ChatAnthropic、ChatGoogleGenerativeAI/Vertex、ChatBedrockConverse、DeepSeek、xAI、Mistral、OpenRouter、自定义 OpenAI 兼容）。LibreChat 在 `packages/api/src/endpoints/{openai,anthropic,google,bedrock,custom}` 中为每个提供商构建 `llmConfig`：API 密钥（用户提供或来自环境变量）、baseURL、请求头、代理、推理参数、Responses-API 开关。

**记忆。** 两种机制：
1. 每轮一次的后台记忆智能体运行（见上文）。
2. 已存储的记忆注入到系统上下文中（`getRequestMemories` / `buildInlineMemoryContext`），另有可选的内联 `set_memory`/`delete_memory` 工具，带严格的使用保护。

**上下文窗口、裁剪与压缩：**
- **token 计数。** SDK 的 `createTokenCounter`（ai-tokenizer / tiktoken），每条消息的计数有缓存（`indexTokenCountMap`），并使用从真实提供商用量中学习得到的 `calibrationRatio`（持久化在 `contextMeta` 中）。
- **裁剪。** SDK 裁剪器（`createPruneMessages` / `getMessagesWithinTokenLimit`）丢弃最旧的消息以适配 `maxContextTokens`。
- **淡化（fading）**（`fading.ts`）：对较早的工具结果做分层遮蔽/截断（`contextPruningConfig`）。选定的层级会被“锁定”并持久化到消息的 `contextMeta` 中，这样下一轮会产生相同的投影，保持提供商提示缓存前缀稳定。
- **摘要 / 压缩（compaction）**（`compaction.ts`）：
  - 启用时（`appConfig.summarization`，可按智能体覆盖；提供商/模型可以与智能体的不同），SDK 把溢出部分摘要成一个 `summary` 内容片段（通过 `on_summarize_*` 流式输出）。
  - 下一轮的 `loadHistory` 从最新的可用摘要片段（`findCheckpointSummaryPart`）开始，并把它作为 `initialSummary` 传入。
  - 手动的 `compact:true` 请求执行 `summarizeOnly`（没有用户消息，响应挂在一条已有消息之下）。
  - 低于最小可用上下文预算时，摘要被禁用。
- **步数预算。** `recursionLimit` 加上 `createStepBudgetHook` 这个 PostToolBatch 钩子。耗尽时产生 `unfinished`，且 `finish_reason = tool-call-limit`。

**HITL 审批**（`packages/api/src/agents/hitl/*`、`stream/ApprovalLifecycle.ts`）：
- **策略。** `endpoints.agents.toolApproval`（`enabled`、`mode`、allow/deny/ask glob），由 `resolveToolApprovalPolicy` 解析。它成为一个 SDK `PreToolUse` 钩子（`buildHITLRunWiring`），另有程序化钩子（`registerToolApprovalHook`）和附加代码环境的钩子。
- **暂停。** `ask` 决策使 SDK 抛出 LangGraph 的 `interrupt()`，因此需要检查点存储。`ask_user_question` 是一个工具，其实现调用 SDK 的中断辅助函数。
- **暂停之后。** `processStream` 返回，且 `run.getInterrupt()` 带有负载。`AgentClient.handleRunInterrupt` 构建一个 `PendingAction`，包含 `actionId`、`interruptId`、`threadId`、`expiresAt`（审批 TTL = 检查点 TTL）、`requestFingerprint`（防止恢复时替换智能体/工具）和 `resumeContext`（临时智能体配置和模型参数）。然后：
  1. `approvals.pause` CAS：任务 `running → requires_action`。
  2. 发出 `on_pending_action`。
  3. 把部分响应持久化为未完成。
  4. 控制器退出，不发送最终事件。
- **恢复。** 在相同的 `thread_id` 和命名空间上重建一个新的 `Run`，并调用 `run.resume(decisionMap | answers)`。决策为 `approve | reject | edit(editedArguments) | respond`。
- **过期。** 回收器 `expireApproval` 中止任务；TTL 索引清理检查点。
- **检查点存储**（`checkpointer.ts`、`checkpoints/saver.ts` `OwnedMongoSaver extends MongoDBSaver`）：每次生成一个命名空间（私有 configurable 键 `__librechat_checkpoint_ns`，因为 LangGraph 会重置 `checkpoint_ns`），并按所有者隔离存储。`deleteAgentCheckpoint` 在终态转换时执行。

**插话引导**（运行中途的用户消息；`packages/api/src/agents/steering/*`、`stream/SteeringLifecycle.ts`、`controllers/agents/steer.js`）：
- `POST /chat/steer {conversationId, generationCreatedAt, text, files, quotes, clientSteerId, preempt}` 入队到每个流一个的 FIFO（最大深度 10，持久回执）。
- 一个 `PostToolBatch` 钩子在每个工具批次边界排空队列。它追加一个 `steer` 内容片段，发出 `on_steer_applied`，并返回 `injectedMessages`；SDK 把它们转换为图状态中的 HumanMessage。
- `preempt:true`（或 `/chat/steer/arm`）请求持有任务的副本（通过 Redis `preempt` 消息）在下一个 `PreemptBoundary` 封存实时模型流。
- `StopFinalize` 终态钩子让在结束前一刻到达的插话继续同一次运行（“热终态插话续跑”）。
- 始终未被应用的插话会随最终事件以 `pendingSteers` 带回，或者被暂存（park），之后作为新轮次恢复（`recoverySteerId`）。
- 偏移包装器把 SDK 的内容索引移到宿主插入的片段之后。

**排队轮次**（`queuedTurns.ts`、`queuedTurnHttp.ts`，schema `queuedTurn.ts`）：
- 在生成进行中发送的后续消息存为一条 Mongo 记录（FIFO 的权威来源），并以一次触发器投递作为唤醒。
- 前一次生成干净结束后，它作为一个普通新轮次被接纳：向聊天路由发起一次带 `agentContinuationAdmission` 的内部回环 POST。
- 可以通过 `/chat/queued-turns` 列出和取消这些记录。

**子智能体：**
- 配置：`agent.subagents {enabled, allowSelf, shareFiles, agent_ids, graphs}`。
- SDK 暴露一个 `subagent` 派生工具（`Constants.SUBAGENT`），运行一个拥有独立上下文、深度有界、带递归乘数的子图。子事件以 `on_subagent_update` 呈现，并按工具调用 id 聚合；子智能体用量通过 `subagentUsageSink` 流入计费。
- 分离（后台）子智能体：任务存储（`InMemorySubagentTaskStore` / `subagentTaskRouting.ts`，通过 Redis 路由到持有任务的进程）、持久的**子智能体线程**（子对话，`subagentThreads.ts`、`services/Endpoints/agents/subagentThreadStore.js`）以及**完成唤醒**（一个持久的 `continue` 触发器，在子智能体完成时启动父轮次；`subagentCompletionWakeup.ts`）。
- 延迟子智能体在首次派生时解析配置。

**移交与图**（`edges.ts`、`chain.ts`、`discovery.ts`）：
- `GraphEdge {from, to, edgeType:'handoff'|'direct', prompt, excludeResults, description}`。
- `handoff` 给智能体提供转移工具（动态路由）。`direct` 提供静态边，允许并行扇出/扇入，并可带使用 `{results}` 的提示模板。
- 旧版 `agent_ids` 链变成顺序的 direct 边（`createSequentialChainEdges`）。
- 附加对话的智能体是额外的起始节点，并行运行；并行片段带有 `groupId`，用于分栏显示。

**其他宿主功能（简述）：**
- 活动/阶段/推理标签：由快速模型为工具批次和推理打标签。
- 技能：目录注入和一个 `skill` 工具（`skills.ts`、`skillFiles.ts`）。
- 与子智能体共享的运行文件。
- 代码执行上下文 / 附加环境（`execution.ts`）和沙箱预热。
- 事件执行者 / 触发器（`agents/triggers/*`，见其 README）：把 webhook/定时任务/事件工作以持久、Mongo 排队、与来源无关的方式投递为智能体轮次，可选地从 LangGraph 检查点继续。这就是 `plan.ts` 中的 `checkpoint` 策略。

---

## 6. OpenAI 兼容与 Responses API 适配器

两者都挂载在 JWT 之前，使用 API 密钥认证（`requireRemoteAgentAuth`，schema `agentApiKey`）、`checkRemoteAgentsFeature` 以及按智能体的远程 ACL。两者都把请求包装进一个带版本、可安全传输的 `AgentRunEnvelope`（`packages/api/src/agents/envelope.ts`：`protocol`、`requestId`、`receivedAt`、`principal`、`payload`）。它们直接调用 `createRun` + `run.processStream`，**不经过** GenerationJobManager、可恢复性或 HITL（`hitlCapable` 关闭，因此需要审批的工具无法暂停）。

**`POST /api/agents/v1/chat/completions`**（`controllers/agents/openai.js`、`packages/api/src/agents/openai/*`）：
- `model` = 智能体 id。也接受 `messages`、`stream`，以及可选的 `conversation_id` 和 `parent_message_id`。
- 处理器把 `on_message_delta` 转成带 `delta.content` 的 `chat.completion.chunk`，把 `on_reasoning_delta` 转成 `delta.reasoning`，以 `data: {...}\n\n` 写出，并以 `data: [DONE]` 结束。
- 工具调用会被投影，并且**只在成功完成时发布**，带完整的 id/name/args（见 `agents/openai/` 中的 README）。
- 非流式请求构建一个 `chat.completion`。
- 用量通过 `recordCollectedUsage` 记录。消息**不会**持久化。
- `GET /v1/models` 和 `/v1/models/:model` 列出可访问的智能体。
- 这里还有：`POST /v1/events`、`/v1/events/bindings`（触发器入口）。

**`POST /api/agents/v1/responses`**（`controllers/agents/responses.js`、`packages/api/src/agents/responses/*`）：
- 实现 Open Responses 规范。流式事件：`response.created`、`response.in_progress`、`response.output_item.added/done`、`response.content_part.added/done`、`response.output_text.delta/done`、`response.reasoning.delta/done`、`response.function_call_arguments.delta/done`、`response.completed/failed`，以及附件事件。
- `previous_response_id` = conversationId，用于加载之前存储的消息。
- `store:true` 时，持久化对话、输入消息（作为用户文本）和输出文本（`saveConversation`、`saveInputMessages`、`saveResponseOutput`）。
- **客户端函数工具**（`clientTools.ts`）：模型调用调用方声明的工具时，运行结束并返回一个 `function_call` 项。调用方通过回放 `function_call` + `function_call_output` 项以无状态方式继续。
- `GET /v1/responses/:id` 重建一个已存储的响应；`GET /v1/responses/models` 列出模型。

---

## 7. `@librechat/agents` 提供了什么，以及 Python 替代方案

**本仓库用它做什么：**
- **`Run`**：`Run.create(config)`、`processStream(input, config, {callbacks})`、`resume(value)`、`getInterrupt()`、`generateTitle()`、`Graph`/`graphRunnable.getState()`、`getRunSteps()`、`getFadingTier(s)`、抢占统计和停止原因。
- **图**：`StandardGraph`（单智能体：智能体节点 ↔ 工具节点循环，`toolsCondition`）和多智能体图（边、移交工具、并行扇出），两者都是使用 `messagesStateReducer` 编译的 LangGraph StateGraph。
- **提供商层**：`Providers` 枚举、`initializeModel`、`getChatModelClass`、基于 LangChain 聊天模型的 `isOpenAILike`、各提供商特定的推理键、Anthropic 提示缓存处理和 Google thought signature。
- **流式抽象**：把 LangChain 流回调转换为规范化的 `GraphEvents`（`ON_RUN_STEP`、`ON_RUN_STEP_DELTA`、`ON_RUN_STEP_COMPLETED`、`ON_RUN_STEP_CLOSED`、`ON_MESSAGE_DELTA`、`ON_REASONING_DELTA`、`CHAT_MODEL_END`、`TOOL_END`、`ON_TOOL_EXECUTE`、`ON_SUMMARIZE_*`、`ON_SUBAGENT_UPDATE`、`ON_CONTEXT_USAGE`、`ON_AGENT_LOG`），配合 `RunStep` 模型（类似 OpenAI Assistants）。处理器：`ModelEndHandler`、`ToolEndHandler`、`ChatModelStreamHandler`、`HandlerRegistry`。
- **内容**：`createContentAggregator`（事件 → 内容片段）、`formatAgentMessages`（存储的内容片段 → LangChain 消息，包括工具调用/结果配对、插话和摘要回放）、`labelContentByAgent`、`createMetadataAggregator`。
- **上下文管理**：`createTokenCounter`、`getTokenCountForMessage`、`createPruneMessages`、`getMessagesWithinTokenLimit`、摘要节点、淡化层级、图片 token 估算器。
- **工具**：使用仅 schema 定义的事件驱动工具执行、提前（eager）执行、`createToolSearch`、`Calculator`、网页搜索、`ReadFile`/`Bash`/代码执行定义、程序化工具调用、`askUserQuestion(s)` 中断辅助函数、工具输出引用。
- **钩子**：`HookRegistry`，包括 `PreToolUse`（审批：allow/ask/deny）、`PostToolUse`、`PostToolBatch`（插话注入、标签、步数预算）、`PreemptBoundary`、`Stop`/`StopFinalize`；`executeHooks`、`createToolPolicyHook`。
- **HITL**：`humanInTheLoop` 配置、LangGraph `interrupt`/`Command(resume)`、检查点存储集成。
- **子智能体**：派生工具、`InMemorySubagentTaskStore`、`buildChildInputs`、子智能体用量汇集器（usage sink）。
- **事件执行者执行器**，以及 Langfuse/OpenTelemetry 链路追踪。

**Python 替代方案：**
- **图运行时。** `langgraph`（Python）的 `StateGraph`，配合 `MessagesState` / `add_messages`、`ToolNode`/`tools_condition`（或调用宿主执行器的自定义工具节点），以 `create_react_agent` 作为起点。多智能体可以用 `langgraph-supervisor`/`langgraph-swarm` 风格的移交工具（`Command(goto=...)`），或用显式 direct 边和 `Send` 做并行扇出。
- **检查点。** `langgraph-checkpoint-mongodb`（`MongoDBSaver`）或 `langgraph-checkpoint-redis`/postgres。HITL 使用 `interrupt()` + `Command(resume=...)`，`thread_id = conversationId`，并按生成使用 `checkpoint_ns`/线程后缀。TTL 通过 Mongo TTL 索引实现。
- **提供商。** `langchain-openai`、`langchain-anthropic`、`langchain-google-genai`/`langchain-google-vertexai`、`langchain-aws`（`ChatBedrockConverse`）、`langchain-mistralai`、`langchain-deepseek`、`langchain-xai`；OpenAI 兼容的自定义端点通过 `ChatOpenAI(base_url=…)` 实现。也可以用 `litellm`（或 `langchain-litellm` 的 `ChatLiteLLM`）作为单一适配器，带各提供商参数和成本表。这对 `recordCollectedUsage` 的计价有用，但为保持一致，仍应维护自己的 token 配置表。
- **流式事件。** 使用 `graph.astream_events(version="v2")` 或 `astream(stream_mode=["messages","updates","custom"])`，然后编写一个 **ContentAggregator**，把 `on_chat_model_stream` 分块（文本、推理、tool_call_chunks）和工具开始/结束映射为 LibreChat 的 `RunStep` / `on_message_delta` / `on_run_step_delta` / `on_run_step_completed` 结构。复现 SDK 的事件契约是移植的主要工作量，因为前端依赖它。
- **token 计数与裁剪。** `tiktoken` / Anthropic `count_tokens`、`langchain_core.messages.trim_messages`，压缩可用 `langmem` 或自定义摘要节点。
- **钩子。** 实现一个小型钩子注册表：PreToolUse（策略 → 中断）、PostToolBatch（排空插话队列 → 返回 `Command(update={"messages":[HumanMessage...]})`）、通过 `recursion_limit` 实现步数预算。
- **工具。** LangChain `StructuredTool`，MCP 用 `langchain-mcp-adapters`，自定义代码执行/文件工具。
- **链路追踪。** Langfuse Python `CallbackHandler`。

---

## 8. 建议的 Python 模块布局

```
app/
  api/routes/agents/
    chat.py            # POST /api/agents/chat[/{endpoint}]、/resume（依赖链 = 中间件）
    stream.py          # GET /chat/stream/{id}（通过 sse-starlette 实现 SSE）、/chat/status、/chat/active、/chat/abort
    steer.py           # /chat/steer、/cancel、/arm；queued_turns.py
    openai_compat.py   # /v1/chat/completions、/v1/models
    responses.py       # /v1/responses
  agents/
    request_controller.py  # ResumableAgentController：校验、幂等性、创建任务、后台任务
    resume_controller.py   # HITL 恢复
    client.py              # AgentClient：加载历史、build_messages、chat_completion、收尾、标题
    initialize.py          # initialize_client / initialize_agent：提供商配置、工具、文件、发现
    load.py                # load_agent / load_ephemeral_agent
    discovery.py edges.py added.py   # 图的边、移交、并行附加对话
    run_factory.py         # create_run → LangGraph 图 + 配置（检查点存储、钩子、限制）
    graph/                 # standard_graph.py、multi_agent_graph.py、tool_node.py（事件驱动执行）
    events.py              # GraphEvents 枚举 + RunStep 模型（pydantic）+ ContentAggregator
    format_messages.py     # 存储的内容片段 <-> LangChain 消息
    handlers.py            # 事件 → SSE 发出（默认处理器）、工具结束产物回调
    context/               # token_counter.py、pruning.py、fading.py、compaction.py（摘要）
    hitl/                  # policy.py、pending_action.py、resume_mapping.py、ask_user_tool.py
    steering/              # 队列排空钩子、注入、抢占
    subagents/             # 派生工具、任务存储、线程、完成唤醒
    memory.py usage.py transactions.py title.py
  providers/               # openai.py anthropic.py google.py bedrock.py custom.py（llm_config 构建器）或 litellm
  stream/
    job_manager.py         # GenerationJobManager（create/emit/subscribe/resume/abort/complete/approvals）
    job_store.py           # Protocol + memory_store.py + redis_store.py（hash + XADD 日志 + Lua CAS）
    event_bus.py           # memory_bus.py + redis_bus.py（发布/订阅频道 stream:{id}:events，按 seq 排序）
    resume_state.py        # 快照/回放（重新聚合分块日志）
  checkpoint/              # 按所有者/命名空间隔离的 mongo saver + TTL + 删除
  db/                      # messages、conversations、transactions、memories、tool_calls、files（motor/beanie）
```

**需要保持的关键不变式：**
- `streamId == conversationId`。
- 生成纪元（`createdAt`）为所有写入、事件发出和中止设置栅栏。
- POST 提前返回，SSE 单独订阅。
- 每个发出的事件都追加到可回放的日志中，因此重连时能拿到 `sync` 快照。
- 终态由唯一的 CAS 胜者决定（complete / abort / error / pause），只有它在持久化消息之后发布 `final`/`error` 帧。
- 用户消息和响应消息是以 `messageId` 为键的幂等 upsert，带 `unfinished` / `error` 标志。
- 对话随用户消息一起 upsert；标题之后以仅写元数据的方式写入。
