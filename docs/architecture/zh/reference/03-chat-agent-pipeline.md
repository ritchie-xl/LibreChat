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
