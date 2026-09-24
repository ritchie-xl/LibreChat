# @librechat/agents：公共 API、运行生命周期与事件契约

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

下文所有路径均相对于 SDK 仓库根目录。每一条结论都来自对代码的阅读。

---

## 1. 公共 API 面

### 1.1 包的子路径导出（`package.json`）

| 子路径 | 源文件 | 暴露的内容 |
|---|---|---|
| `.`（根） | `src/index.ts` | 主运行时。详见 1.2。 |
| `./provider-registration` | `src/provider-registration.ts` | `registerProvider()`，以及类型 `ProviderFamily`（`'openai'｜'anthropic'｜'bedrock'｜'google'｜'mistral'｜'generic'`）、`ProviderRegistrationOptions {provider, model: ctor, family?, manualToolStream?, strictAlternation?}` 和 `CustomProviderOptionsMap`（通过声明合并来扩展的类型）。实现位于 `src/llm/providerRegistry.ts`。它是一个以 `Symbol.for('@librechat/agents:providerRegistry:v1')` 为键的全局注册表。重复注册会抛出异常，注册成功返回一个注销函数（disposer）。`family` 默认为 `generic`；两个布尔值默认为 false。 |
| `./openai` | `src/openai/index.ts` | 把图事件转换为 OpenAI Chat Completions SSE 的适配器（第 6.1 节）。 |
| `./responses` | `src/responses/index.ts` | 把图事件转换为 OpenAI Responses API SSE 的适配器（第 6.2 节）。 |
| `./langchain`、`./langchain/messages`、`/messages/tool`、`/prompts`、`/runnables`、`/tools`、`/google-common`、`/openai`、`/utils/env`、`/language_models/chat_models` | `src/langchain/*` | 对 `@langchain/core` 及相关类（`AIMessage`、`HumanMessage`、`ToolMessage`、`mapStoredMessagesToChatMessages` 等）的透传再导出。它们的作用是让宿主使用与 SDK 相同的 LangChain 实例。 |
| `./llm/openai`、`./llm/anthropic`、`./llm/google`、`./llm/bedrock`、`./llm/vertexai`、`./llm/mistral`、`./llm/openrouter` | `src/llm/<provider>/index.ts` | SDK 打过补丁的各提供商聊天模型类，以及提供商辅助函数，例如 OpenAI 缓存断点工具函数。它们被移出了根入口（barrel），这样导入根模块时不会加载所有提供商 SDK。 |

### 1.2 根入口（`src/index.ts`）

| 导出分组 | 关键符号 | 用途 |
|---|---|---|
| `./run` | `Run`、`defaultOmitOptions` | 运行编排器（第 2 节）。 |
| `./stream` | `ChatModelStreamHandler`、`createContentAggregator`、`getChunkContent`、`SDK_STREAM_DISPATCH`、`dispatchesChatModelStream` | 把 LLM 块转换为步骤事件，并把步骤事件折叠回内容片段。 |
| `./events` | `HandlerRegistry`、`composeEventHandlers`、`ModelEndHandler`、`ToolEndHandler`、`LLMStreamHandler`、`TestLLMStreamHandler`、`TestChatStreamHandler`、`createMetadataAggregator` | 处理器注册表和内置处理器。 |
| `./messages` | `formatAgentMessages`、`convertMessagesToContent`、`getMessageId`、裁剪、缓存、交替、淡化、助手阶段辅助函数等 | 消息格式化，以及消息与内容之间的转换。 |
| `./graphs` | `StandardGraph`、`MultiAgentGraph`、`createGraph`、`HandoffLimitError` | 图构建器。 |
| `./agents/projection` | `projectAgentContextUsage` | 宿主在发送前估算上下文用量。 |
| `./summarization` | `shouldTriggerSummarization` 等 | 摘要触发条件。 |
| `./tools/*` | Calculator、CodeExecutor、BashExecutor、ProgrammaticToolCalling（+Bash）、SkillTool、SubagentTool、`subagent/*`（`InMemorySubagentTaskStore`）、ReadFile、skillCatalog、ToolSearch、`ToolNode`、intentArg、schema、`handlers`（`handleToolCalls`、`handleToolCallChunks`、`handleServerToolResult`）、本地与 cloudflare 引擎、search | 内置工具和 ToolNode。 |
| `./common` | 全部枚举和常量（第 3.1 节） | |
| `./utils` | `createHandlers`（`src/utils/handlers.ts`）、`RunnableCallable`、`joinKeys`、`resetIfNotEmpty`、token 工具函数、截断、错误 | |
| `./hooks` | `HookRegistry`、`executeHooks`、`createToolPolicyHook`、`createWorkspacePolicyHook`、`HOOK_EVENTS` | 生命周期钩子：`RunStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PostToolBatch`、`PreemptBoundary`、`PermissionDenied`、`SubagentStart`、`SubagentStop`、`Stop`、`StopFinalize`、`StopFailure`、`PreCompact`、`PostCompact`。 |
| `./session` | `AgentSession`、`createAgentSession`、`createRunHandlers`、`JsonlSessionStore`、`SessionManager`、`deriveMessages`、`serializeMessage`/`deserializeMessage` | 程序化会话封装（第 6.3 节）。 |
| `./eventActor` | `EventActorExecutor`、`createEventActorExecutor` | 在检查点分叉上进行持久化的 actor 式调用（prepare、commit、discard、suspend、resume）。 |
| `./hitl` | `askUserQuestion`、`askUserQuestions`、审批复核辅助函数 | 人机协同（HITL）中断辅助函数。 |
| `export type * from './types'` | `src/types/*` 中的所有类型 | |
| LangGraph 再导出 | `Command`、`INTERRUPT`、`interrupt`、`MemorySaver`、`BaseCheckpointSaver`、`isInterrupted`、`Interrupt` | 让宿主与 SDK 使用同一个 LangGraph 版本。 |
| LLM | `getChatModelClass`、`registerProvider`、`initializeModel`、`attemptInvoke`、`tryFallbackProviders`、`prepareProviderRequest`、`canSealPreempt`、`isThinkingEnabled`、`getMaxOutputTokensKey`、`smoothStream`、`resolveStreamDelay`、`DEFAULT_STREAM_DELAY`、`computeAdaptivePieceSize`、`FakeChatModel`、`createFakeStreamingLLM`、`DEFAULT_MAX_TOOL_CALL_ARG_BYTES`、`StreamLimitExceededError`、`resolveStreamLimits`、`markTokenCounterCacheCompatible`、`LANGFUSE_OBSERVATION_METADATA_ARTIFACT_KEY` | |

---

## 2. 运行生命周期（`src/run.ts`）

### 2.1 `RunConfig`（`src/types/run.ts`）

| 字段 | 含义 |
|---|---|
| `runId`（必填） | 用作运行 id、`configurable.run_id`，并出现在步骤键和 `RunStep.runId` 中。在 LibreChat 中它是响应消息的 id。 |
| `graphConfig` | 三种形态之一：`LegacyGraphConfig`（`type?:'standard'`、`llmConfig`、`tools`、`instructions` 等）、`StandardGraphConfig`（`{type?:'standard', agents: AgentInputs[], signal?, compileOptions?}`，其中只使用 `agents[0]`），或 `MultiAgentGraphConfig`（`{type:'multi-agent', agents, edges: GraphEdge[], entryAgentId?, maxHandoffs?, compileOptions?}`）。`compileOptions = {checkpointer?, interruptBefore?, interruptAfter?}`。 |
| `customHandlers?: Record<eventName, EventHandler>` | **宿主的事件接收端。** 每一项都会注册到 `HandlerRegistry` 中。 |
| `returnContent?` | 为 true 时，`processStream` 返回 `Graph.getContentParts()`，它来自 `convertMessagesToContent(messages.slice(startIndex))`，**而不是**来自聚合器。 |
| `tokenCounter?`、`indexTokenCountMap?` | 用于裁剪。如果只给了映射没给计数器，`Run.create` 会根据模型名构建一个 tiktoken 计数器。 |
| `calibrationRatio?`、`fadingTier?`、`fadingTiers?` | 在运行之间延续的上下文管理状态。通过 `getCalibrationRatio()` 和 `getFadingTier(s)()` 读回。 |
| `hooks?: HookRegistry`、`maxStopContinuations?`（默认 8） | 生命周期钩子。返回 `decision:'block'` 的 Stop 钩子可以带着注入的消息重新进入图。 |
| `preemption?: StreamPreemption` | 协作式停止。`{shouldPreempt(): boolean, subscribe?(wake), restartGraceMs? (default 2000), maxSeals? (default 8)}`。**封存（seal）**保留部分轮次并自循环。**重启（restart）**丢弃只含推理内容的轮次。 |
| `streamLimits?` | `{maxToolCallArgBytes? (64 KiB default), maxToolCallArgBytesByTool?, maxDeltaEventsPerTurn?}`。超限时抛出 `StreamLimitExceededError`。 |
| `humanInTheLoop?: {enabled}` | 默认关闭。开启后，如果没有提供检查点存储，会安装一个 `MemorySaver`，并且 `ask` 工具决策会调用 `interrupt()`。 |
| `langfuse?`、`subagentUsageSink?`、`subagentTasks?`、`subagentContext?`、`initialSessions?`、`toolOutputReferences?`、`eagerEventToolExecution?`、`codeSessionToolNames?`、`interruptingToolNames?`、`toolExecution?`、`skipCleanup?` | 链路追踪、子智能体和工具执行。 |

每次调用的 `RunStreamConfig` 为 `Partial<RunnableConfig> & {version:'v1'｜'v2', run_id?, durability?:'async'｜'sync'｜'exit'}`。重要的 `configurable` 键：

- `thread_id`：检查点线程，同时也是 Langfuse 会话。
- `user_id`
- `requestBody.parentMessageId`
- `checkpoint_id` / `checkpoint_ns`

该配置上的 `signal` 字段是本次调用的中止信号。

### 2.2 逐步说明

1. **`Run.create(config)`**（静态、异步）
   - 在源码模式下加载各提供商。
   - 需要时构建 token 计数器。
   - 调用 `new Run()`。构造函数：
     - 创建一个 `HandlerRegistry`，并注册每一个 `customHandlers` 项；
     - 保存钩子、HITL、抢占、限制及其他配置；
     - 用 `createLegacyGraph`（`createGraph({kind:'standard'})`，其中旧版 `llmConfig` 转换为 `AgentInputs{agentId:'default', provider, clientOptions}`）或 `createMultiAgentGraph` 构建图；
     - 应用 HITL 检查点存储的兜底逻辑，然后执行 `applyGraphRuntimeConfig`（钩子、HITL、输出引用、提前执行等）；
     - 调用 `graph.createWorkflow()` 编译 LangGraph；
     - 设置 `Graph.handlerRegistry = registry`；
     - 预置 `initialSessions`。
2. **`processStream(inputs: IState｜Command, callerConfig, streamOptions?)`**
   - `isResume = inputs instanceof Command`。恢复时它执行 `restoreInterruptFromCheckpoint`（`getState({subgraphs:true})`），恢复运行步骤的恢复状态和检查点消息，并恢复工具重放配置。
   - `recursionLimit` =（调用方给定的值，或 50）+ 抢占开启时的 `maxSeals`。
   - 去掉内部使用的 configurable 键。
   - 全新运行时：`graph.resetValues(keepContent, checkpointScope)` 清空 `contentData`、`stepKeyIds`、`toolCallStepIds` 及其他映射；移交路由开始；设置新的 Stop 续跑执行 id。
   - 回调：
     - 一个 `BaseCallbackHandler.fromMethods({handleCustomEvent: customEventCallback})`，并设置 `awaitHandlers = true`；
     - 可选的 `ClientCallbacks`（`handleToolError`、`handleToolStart`、`handleToolEnd`，每个都以 `(graph, ...args)` 调用）；
     - 来自 `createLangfuseHandler(...)` 的 Langfuse 处理器，携带 trace 元数据（`messageId=runId`、`parentMessageId`、智能体 id 和名称）、标签 `['librechat','agent']`，并在设置了 `deterministicTraceId` 时使用以 `runId` 为种子的确定性 trace id。
   - 设置 `config.runId = config.run_id = configurable.run_id = this.id`。
   - 存在检查点存储时，`durability` 默认为 `'exit'`。
   - **流开始前的钩子**（仅全新运行）：先 `RunStart`，再对最后一条人类消息执行 `UserPromptSubmit`。
     - `preventContinuation`、`deny` 或 `ask` → 设置 `_haltedReason`，方法返回 `undefined`，不发起任何模型调用。
     - `additionalContext` 字符串被合并追加为一条 `HumanMessage{additional_kwargs:{role:'system', source:'hook'}}`。
   - **`consumeStream` 循环。** 每次迭代调用 `graphRunnable.streamEvents(inputs, {...config, runName}, {raiseError:true, ignoreCustomEvent:true})`，并处理每一个事件：
     - 自定义图事件（名称在 `CUSTOM_GRAPH_EVENTS` 中）会被**跳过**。它们改为通过回调路径到达处理器（第 5 节）。
     - 遇到的第一个 `data.chunk[INTERRUPT]` 被捕获为 `_interrupt = {interruptId, threadId, payload}`。
     - `handlerRegistry.getHandler(event.event)?.handle(eventName, data, metadata, graph)`。这覆盖了原生事件：`on_chat_model_stream`、`on_chat_model_end`、`on_tool_end`、`on_chain_*` 等。
     - 遇到 `on_chat_model_end` 时，随后调用 `graph.closeOpenMessageStep(metadata, modelEndAt)`，为该智能体通道中未关闭的消息步骤发出 `ON_RUN_STEP_CLOSED`。
     - 钩子的叫停信号会以 `_haltedReason` 跳出循环。
   - 流结束时：
     - 如果有中断，它解析出用于恢复的检查点 id 并返回。
     - 如果被叫停，直接返回。
     - 否则计算 `stopReason`（`preemptHaltReason`、`'preempt_incomplete'` 或 `'output_truncated'`），并运行 `Stop` 和 `StopFinalize` 钩子。阻断并注入消息的 Stop 钩子会开始新一轮循环迭代，输入为 `messages: injected`（有检查点存储时）或 `messages: [...graph.messages, ...injected]`（无检查点存储时），并推进 `streamSegment`。
   - **错误路径：** 设置 `streamThrew`；`HandoffLimitError` 把叫停原因设为 `'handoff_limit'`；调用 Langfuse 的 `handleChainError`；根据 `signal.aborted` 设置 `streamAborted`；运行 `StopFailure` 钩子；重新抛出错误。
   - **`finally` 块：**
     - 除非运行正在等待恢复，否则清除钩子会话；
     - 释放 Langfuse 处理器；
     - **终结清扫：** `closeUnfinishedRunSteps(status, terminalAt)`，其中 status 为 `cancelled`（中止或叫停）、`failed`（出错）或 `completed`。运行因 HITL 暂停时跳过清扫；
     - 清除 configurable 和回调；
     - 如果设置了 `returnContent`，读取 `getContentParts()`；
     - 保存校准比值和淡化层级；
     - 计算移交结果；
     - 除非运行正在等待恢复，否则调用 `clearHeavyState()`。
3. **完成后的查询方法：**
   - `getInterrupt<T>()` 返回 `RunInterruptResult {interruptId, threadId?, checkpointId?, checkpointNs?, payload}`。载荷为 `ToolApprovalInterruptPayload {type:'tool_approval', action_requests, review_configs, hook_session_id?, subagent?}` 或询问用户问题的载荷。
   - `getHaltReason()`、`getOutputTruncated()`、`getPreemptStats() {seals, restarts, emptyBoundaries}`、`getHandoffOutcome()`、`getRunMessages()`、`getDiscoveredTools()`、`getToolCount()`、`getChildCheckpointThreadIds()`、`getFileCheckpointer()`、`rewindFiles()`。
   - `run.Graph.getRunSteps(agentId?)`、`getRunStepsByAgent()`、`getContentPartAgentMap()`、`getActiveAgentIds()`。
4. **`resume(resumeValue, callerConfig, streamOptions?, {update?, goto?})`**
   - 解析检查点 id：从中断中获取，或扫描 `getStateHistory` 找到匹配的中断 id。
   - 把值限定为 `{[interruptId]: value}`，除非它本身已经是以该 id 为键的映射。
   - 调用 `processStream(new Command({resume, update?, goto?}), resumeConfig)`。
   - 宿主必须用同一个 `thread_id` 和检查点存储重建 Run。默认的恢复值为 `ToolApprovalDecision[]` 或以 `tool_call_id` 为键的映射。
5. **中止与抢占**
   - 中止：`callerConfig.signal` 或 `graphConfig.signal`。流会抛出异常；未关闭的步骤以 `cancelled` 关闭。
   - 抢占：`RunConfig.preemption`（见 2.1）。被封存的轮次提前结束；由 `PreemptBoundary` 钩子提供注入的消息。被丢弃的轮次会发出一个合成的 `on_chat_model_end`，带 `response_metadata.preempted/preemptDiscarded`。
6. **辅助生成器。** 每个都单独发起一次模型调用，接入 Langfuse，并且**不**发出任何图事件。
   - `generateTitle({provider, inputText, contentParts, titlePrompt?, clientOptions?, chainOptions?, skipLanguage?, titleMethod: 'completion'｜'structured'｜'functions', titlePromptTemplate?})` → `{language?, title?}`。
   - `generateActivityLabel(...)` → `{label?}`。
   - `generateReasoningLabel(...)` → `{label?, usage?}`。
   - `generateActivityPhaseLabel(...)` → `{label?}`。
7. **token 用量。** `Run` 上没有内置的累加器。宿主在 `on_chat_model_end` 上注册 `new ModelEndHandler(collectedUsage: UsageMetadata[])`；它会压入 `data.output.usage_metadata`。子智能体的用量永远不会到达该处理器；它通过 `RunConfig.subagentUsageSink(SubagentUsageEvent {usage, model?, provider?, subagentType, subagentRunId, subagentAgentId, runId (root), parentRunId?, depth?, ancestry?, memberAgentId?, subagentKind?})` 送达。`ON_CONTEXT_USAGE` 携带每次调用的预算。

### 2.3 LLM 块如何变成步骤事件

有两条路径：

- **宿主路径。** 宿主在 `on_chat_model_stream` 上注册 `ChatModelStreamHandler`（或带有 `SDK_STREAM_DISPATCH` 标记的包装器）。Run 的 `streamEvents` 循环把每个块交给它，由它分发步骤事件。走这条路径的运行不能封存。
- **图内路径。** 没有注册带标记的处理器时，`src/llm/invoke.ts` 中的 `attemptInvoke` 会在模型节点内为每个块运行一个私有的 `ChatModelStreamHandler`。这是支持抢占的路径。

---

## 3. 事件契约

### 3.1 枚举（`src/common/enum.ts`），完整列表

**GraphEvents。** 自定义事件：

- `ON_AGENT_UPDATE='on_agent_update'`
- `ON_RUN_STEP='on_run_step'`
- `ON_RUN_STEP_DELTA='on_run_step_delta'`
- `ON_RUN_STEP_COMPLETED='on_run_step_completed'`
- `ON_RUN_STEP_CLOSED='on_run_step_closed'`
- `ON_MESSAGE_DELTA='on_message_delta'`
- `ON_REASONING_DELTA='on_reasoning_delta'`
- `ON_TOOL_EXECUTE='on_tool_execute'`
- `ON_SUMMARIZE_START='on_summarize_start'`
- `ON_SUMMARIZE_DELTA='on_summarize_delta'`
- `ON_SUMMARIZE_COMPLETE='on_summarize_complete'`
- `ON_SUBAGENT_UPDATE='on_subagent_update'`
- `ON_AGENT_LOG='on_agent_log'`
- `ON_CONTEXT_USAGE='on_context_usage'`

LangChain 官方事件：

- `ON_CUSTOM_EVENT='on_custom_event'`
- `CHAT_MODEL_START='on_chat_model_start'`、`CHAT_MODEL_STREAM='on_chat_model_stream'`、`CHAT_MODEL_END='on_chat_model_end'`
- `LLM_START='on_llm_start'`、`LLM_STREAM='on_llm_stream'`、`LLM_END='on_llm_end'`
- `CHAIN_START='on_chain_start'`、`CHAIN_STREAM='on_chain_stream'`、`CHAIN_END='on_chain_end'`
- `TOOL_START='on_tool_start'`、`TOOL_END='on_tool_end'`
- `RETRIEVER_START='on_retriever_start'`、`RETRIEVER_END='on_retriever_end'`
- `PROMPT_START='on_prompt_start'`、`PROMPT_END='on_prompt_end'`

**Providers：**

- `OPENAI='openAI'`、`VERTEXAI='vertexai'`、`BEDROCK='bedrock'`、`ANTHROPIC='anthropic'`
- `MISTRALAI='mistralai'`、`MISTRAL='mistral'`、`GOOGLE='google'`、`AZURE='azureOpenAI'`
- `DEEPSEEK='deepseek'`、`OPENROUTER='openrouter'`、`XAI='xai'`、`MOONSHOT='moonshot'`

**GraphNodeKeys：** `TOOLS='tools='`、`AGENT='agent='`、`SUMMARIZE='summarize='`、`ROUTER='router'`、`PRE_TOOLS='pre_tools'`、`POST_TOOLS='post_tools'`。节点名形如 `agent=<agentId>` 等。`getAgentContext` 通过解析 `metadata.langgraph_node` 找到对应的智能体。

**GraphNodeActions：** `TOOL_NODE='tool_node'`、`CALL_MODEL='call_model'`、`ROUTE_MESSAGE='route_message'`。

**CommonEvents：** `LANGGRAPH='LangGraph'`。

**StepTypes：** `TOOL_CALLS='tool_calls'`、`MESSAGE_CREATION='message_creation'`。

**ContentTypes：**

- `TEXT='text'`、`ERROR='error'`、`THINK='think'`、`TOOL_CALL='tool_call'`
- `IMAGE_URL='image_url'`、`IMAGE_FILE='image_file'`
- `THINKING='thinking'`（Anthropic）、`REASONING='reasoning'`（Vertex/Google）、`REASONING_CONTENT='reasoning_content'`（Bedrock）
- `AGENT_UPDATE='agent_update'`、`SUMMARY='summary'`、`STEER='steer'`、`ACTIVITY_LABEL='activity_label'`

**ToolCallTypes：** `FUNCTION='function'`、`RETRIEVAL='retrieval'`、`FILE_SEARCH='file_search'`、`CODE_INTERPRETER='code_interpreter'`、`TOOL_CALL='tool_call'`。

**Callback：** `TOOL_ERROR='handleToolError'`、`TOOL_START='handleToolStart'`、`TOOL_END='handleToolEnd'`、`CUSTOM_EVENT='handleCustomEvent'`。其他 LangChain 回调名在源码中被注释掉了。

**Constants：**

- `OFFICIAL_CODE_BASEURL='https://api.librechat.ai/v1'`
- `EXECUTE_CODE='execute_code'`、`TOOL_SEARCH='tool_search'`、`PROGRAMMATIC_TOOL_CALLING='run_tools_with_code'`、`WEB_SEARCH='web_search'`
- `CONTENT_AND_ARTIFACT='content_and_artifact'`
- `LC_TRANSFER_TO_='lc_transfer_to_'`、`HANDOFF_PARALLEL_BATCH='__handoff_parallel_batch'`、`HANDOFF_GROUP_ID='__handoff_group_id'`
- `MCP_DELIMITER='_mcp_'`、`ANTHROPIC_SERVER_TOOL_PREFIX='srvtoolu_'`
- `SKILL_TOOL='skill'`、`INVOKED_PROVIDER='__invoked_provider'`、`INVOKED_MODEL='__invoked_model'`
- `READ_FILE='read_file'`、`BASH_TOOL='bash_tool'`、`BASH_PROGRAMMATIC_TOOL_CALLING='run_tools_with_bash'`、`SUBAGENT='subagent'`
- `WRITE_FILE`、`EDIT_FILE`、`GREP_SEARCH`、`GLOB_SEARCH`、`LIST_DIRECTORY`、`COMPILE_CHECK`（值为 snake_case 字符串）

**TitleMethod：** `STRUCTURED`、`FUNCTIONS`、`COMPLETION`。

**EnvVar：** `CODE_BASEURL='LIBRECHAT_CODE_BASEURL'`、`CODE_API_RUN_TIMEOUT_MS`。

`src/common/constants.ts` 另外定义了：

- `DEFAULT_RECURSION_LIMIT=50`、`DEFAULT_MAX_SEALS=8`、`DEFAULT_PREEMPT_RESTART_GRACE_MS=2000`、`DEFAULT_MAX_STOP_CONTINUATIONS=8`、`PREEMPT_BOUNDARY_HOOK_TIMEOUT_MS=120000`
- `ANTHROPIC_TOOL_TOKEN_MULTIPLIER=2.6`、`DEFAULT_TOOL_TOKEN_MULTIPLIER=1.4`
- 运行名称：`AgentGraph`、`MultiAgentGraph`、`AgentModelCall`、`StepLabel`、`ReasoningLabel`、`MultiStepLabel`、`MultiStepLabelGeneration`
- `SUBAGENT_CONTEXT_VERSION=1`、`COMPACTION_SEMANTIC_INDEX_LIMITS`

### 3.2 处理器签名

每个事件都经过同一个接口：

```ts
interface EventHandler { handle(event: string, data, metadata?: Record<string,unknown>, graph?: StandardGraph|MultiAgentGraph): void|Promise<void> }
```

`metadata` 是 LangGraph 的回调元数据：`langgraph_node`、`langgraph_step`、`checkpoint_ns`、`run_id`、`thread_id`、`ls_provider`、`ls_model_name`，以及 SDK 自己的键（`__invoked_provider`、`__handoff_group_id` 等）。

### 3.3 核心数据结构（`src/types/stream.ts`）

```ts
type RunStepStatus = 'in_progress'|'completed'|'cancelled'|'failed';
type RunStep = {
  id: string;              // `step_${nanoid()}`
  type: StepTypes;
  index: number;           // content-part slot (monotonic nextContentIndex; reserved synchronously)
  stepIndex?: number;      // ordinal within its stepKey
  stepDetails: MessageCreationDetails | ToolCallsDetails;
  runId?: string; agentId?: string /* multi-agent only */; groupId?: number /* parallel group, positive int */;
  created_at?: number; status?: RunStepStatus;   // 'in_progress' at dispatch
  completed_at?: number; cancelled_at?: number; failed_at?: number;
  summary?: SummaryContentBlock;  // summarization steps only
  usage?: null | object;          // null, or {prompt_tokens, completion_tokens, total_tokens} on summary steps
};
MessageCreationDetails = { type:'message_creation', message_creation:{ message_id: string /* msg_… or provider chunk.id */, content_type?: 'text'|'think', phase?: 'commentary'|'final_answer' } };
ToolCallsDetails = { type:'tool_calls', tool_calls?: AgentToolCall[] }; // LangChain ToolCall {id,name,args,type:'tool_call'} or OpenAI {id,type:'function',function:{name,arguments}}
RunStepDeltaEvent = { id: stepId, delta: ToolCallDelta };
ToolCallDelta = { type: StepTypes, tool_calls?: ToolCallChunk[] /* {name?, args?: string, id?, index?, type?:'tool_call_chunk'} */, summary?, auth?: string, expires_at?: number };
MessageDeltaEvent = { id: stepId, delta: { content?: MessageContentComplex[], tool_call_ids?: string[] } };
ReasoningDeltaEvent = { id: stepId, delta: { content?: MessageContentComplex[] } };  // parts are {type:'think', think}
RunStepClosedEvent = { id, index, type, status: 'completed'|'cancelled'|'failed', closed_at: number, created_at?, runId?, agentId?, groupId?, stepIndex? };
ToolCompleteEvent/ToolEndEvent (inside {result}) = { id: stepId, index: number, type:'tool_call', tool_call: { id, name, args: string /*bounded JSON*/, output: string, progress: 1, outcome?: string }, completed_at?: number, eager?: true };
SummaryCompleted (inside {result}) = { type:'summary', summary: SummaryContentBlock, id, index, completed_at };
AgentUpdate = { type:'agent_update', agent_update:{ index, runId, agentId } };
```

在 `ToolCompleteEvent` 中，`index` 是**工具使用轮次**（`toolUsageCount` 或 `request.turn`），而不是内容索引。聚合器忽略它，按 `tool_call.id` 进行匹配。

### 3.4 每个事件：何时触发、携带什么

| 事件 | 发出方与时机 | 载荷 |
|---|---|---|
| `on_run_step` | `Graph.dispatchRunStep`。新的 MESSAGE_CREATION 步骤开始时触发：新 `stepKey` 的第一段非空内容、阶段拆分、从推理切换到正文，或一个 Google 服务端工具片段。TOOL_CALLS 步骤也会触发：第一个带 id 和 name 的工具调用块（`handleToolCallChunks`），或每一个完整的 `tool_call`（`handleToolCalls`）。摘要会分发一个 MESSAGE_CREATION 步骤，其 `summary` 设为占位值 `{type:'summary', model, provider}`。 | `RunStep` |
| `on_run_step_delta` | `handleToolCallChunks`，每个带 `tool_call_chunks` 的流式块触发一次。宿主也可能发出 `auth`/`expires_at` 增量（MCP OAuth）。 | `RunStepDeltaEvent` |
| `on_message_delta` | `dispatchMessageDelta`，用于文本块。 | `MessageDeltaEvent`。片段为 `{type:'text', text, citations?, phase?, tool_call_ids?}`；Google 服务端工具块还可能携带 `toolCall`/`toolResponse` 片段。 |
| `on_reasoning_delta` | `dispatchReasoningDelta`，用于推理块。来源：`additional_kwargs[reasoningKey]`、OpenAI `reasoning.summary[0].text`、Anthropic `thinking`、Bedrock `reasoning_content`、Google `reasoning`、`<think>` 标签。全部规范化为 `{type:'think', think}`。 | `ReasoningDeltaEvent` |
| `on_run_step_completed` | 工具调用：`ToolNode`（常规路径和提前的逐结果路径）或 `dispatchEagerToolCompletions`（`eager:true`），以自定义事件发送；`handleToolCallErrorStatic` 直接发给处理器。摘要：summarize 节点的 `dispatchRunStepCompleted`（直接调用处理器；注意 `metadata` 收到的是 `configurable`，这是一个怪异之处）。 | `{result: ToolCompleteEvent ｜ SummaryCompleted}` |
| `on_run_step_closed` | `Graph.closeRunStep`，每个步骤恰好一次。先到的关闭生效，终态不再改变。触发条件：(a) 同一智能体通道中的后继步骤关闭前一个消息步骤；(b) `on_chat_model_end` 关闭该通道中未关闭的消息步骤；(c) TOOL_CALLS 步骤的所有待完成工具调用都完成后，由 `recordStepCompletion` 关闭；(d) 抢占丢弃以 `cancelled` 关闭；(e) 空摘要以 `failed` 关闭；(f) 运行结束时的清扫。 | `RunStepClosedEvent` |
| `on_tool_execute` | 事件驱动模式下的 `ToolNode`（`AgentInputs.toolDefinitions` 非空），或提前执行的流式路径。**由宿主处理的 RPC。** | `ToolExecuteBatchRequest {toolCalls: ToolCallRequest[] {id, name, args, stepId?, turn?, codeSessionContext?, runtimeSessionHint?}, userId?, agentId?, executionContext?, callerCapabilityProjection?, configurable?, metadata?, signal?, resolve(results: ToolExecuteResult[]), reject(err), onResult?(result)}`。`ToolExecuteResult = {toolCallId, content: string｜unknown[], artifact?, status:'success'｜'error', errorMessage?, outcome?, outcome_patch?:{from,to}, injectedMessages?: InjectedMessage[]}`。 |
| `on_summarize_start` | summarize 节点，紧跟在摘要的 `on_run_step` 之后。 | `{agentId, provider, model?, messagesToRefineCount, summaryVersion, semanticIndexEntryCount?, semanticIndexCharCount?}` |
| `on_summarize_delta` | 摘要 LLM 的每一个块。 | `{id: stepId, delta:{summary:{type:'summary', content: MessageContentComplex[], provider, model}}}` |
| `on_summarize_complete` | 成功时在 `on_run_step_completed{summary}` 之后发出；失败或输出为空时带 `error` 发出。 | `{id: stepId, agentId, summary?: SummaryContentBlock, error?: string}`。`SummaryContentBlock = {type:'summary', content?, tokenCount?, coverage?:{retainedFromMessageId}, boundary?:{messageId, contentIndex}, summaryVersion?, model?, provider?, createdAt?}` |
| `on_subagent_update` | `SubagentExecutor`，直接调用**父级**注册表中的处理器（从不经过 `streamEvents`）。子级事件会被包装：`on_run_step`→`run_step`、`on_run_step_delta`→`run_step_delta`、`on_run_step_completed`→`run_step_completed`、`on_run_step_closed`→`run_step_closed`、`on_message_delta`→`message_delta`、`on_reasoning_delta`→`reasoning_delta`、`on_tool_execute`→`run_step`（工具请求同时转发给父级的 `on_tool_execute`）。在子运行前后发出 `start`/`stop`/`error`/`control` 阶段。更新经过一个有界的有序队列；压力大时会丢弃低优先级阶段。 | `SubagentUpdateEvent {runId (root), parentRunId?, subagentRunId, parentToolCallId?, subagentType, subagentKind?, subagentAgentId, memberAgentId?, depth?, ancestry?, parentAgentId?, phase, data?, label?, timestamp (ISO)}` |
| `on_agent_log` | `emitAgentLog`，发出后不等待（fire-and-forget）。只有 `AGENT_DEBUG_LOGGING=true` 时才输出 `debug` 级别。 | `{level:'debug'｜'info'｜'warn'｜'error', scope:'prune'｜'summarize'｜'graph'｜'sanitize'｜string, message, data?, runId?, agentId?}` |
| `on_context_usage` | 每次模型调用一次，在裁剪之后、模型被调用**之前**（会被 await）。仅做摘要的运行结束时也会发送。 | `ContextUsageEvent {runId?, agentId?, breakdown: TokenBudgetBreakdown, contextBudget?, effectiveInstructionTokens?, prePruneContextTokens?, remainingContextTokens?, calibrationRatio?}` |
| `on_agent_update` | **没有任何 SDK 代码发出它。** 它是遗留事件；`createContentAggregator` 仍会消费它。 | `AgentUpdate` |
| `on_chat_model_stream` | 到达 Run 循环的 LangChain 原生事件，以及 `attemptInvoke` 中的内联分发。 | `{chunk: AIMessageChunk}` |
| `on_chat_model_end` | 原生事件，或封存、丢弃之后发出的合成自定义事件。 | `{output: AIMessageChunk}`，其中 `usage_metadata` = `{input_tokens, output_tokens, total_tokens, input_token_details?, output_token_details?}` |
| `on_tool_end` | 直接执行的工具的原生事件，以及针对 Anthropic 服务端网页搜索结果的一次合成调用（`handleAnthropicSearchResults`）。 | `{input, output: ToolMessage｜Command}` |

### 3.5 顺序保证

1. 一个步骤的所有 `on_run_step_delta`、`on_message_delta` 和 `on_reasoning_delta` 都在该步骤的 `on_run_step` 之后。
2. 在同一个智能体通道内（`agentId` 或 `''`），前一个消息步骤的 `on_run_step_closed` 在下一个步骤的 `on_run_step` **之前**发出（`trackDispatchedRunStep`）。内容索引在这次 await 之前就已预留，因此并行通道永远不会冲突。
3. `created_at` 在前驱步骤关闭之后、`on_run_step` 发布之前的那一刻打上。
4. 消息步骤在宿主的 `on_chat_model_end` 处理器返回后立即关闭（`completed`）。TOOL_CALLS 步骤在其**最后一个**待完成工具调用的 `on_run_step_completed` 之后关闭。处理器报错不会阻止关闭。
5. 某次模型调用的 `on_context_usage` 先于该调用的增量。
6. 摘要序列：`on_run_step(summary placeholder)` → `on_summarize_start` → `on_summarize_delta*` → `on_run_step_completed{summary}` → `on_summarize_complete` → `on_run_step_closed`。
7. 运行结束时，每个仍处于 `in_progress` 的步骤都会收到一个终态的 `on_run_step_closed`，除非运行因 HITL 暂停；这些步骤在恢复时通过持久化的 `RunStepResumeState` 继续。
8. **两条投递通道。** 步骤事件（`RUN_STEP`、`RUN_STEP_DELTA`、`RUN_STEP_CLOSED`、`MESSAGE_DELTA`、`REASONING_DELTA`）直接发给注册表中的处理器，**同时**作为 LangChain 自定义事件回送一份用于链路追踪。当 `graph.hasHandlerDispatchedEvent(name, id)` 为 true 时，Run 的自定义事件回调会丢弃这份回送，因此处理器对每个事件恰好只看到一次。其他所有自定义事件（`COMPLETED`、`TOOL_EXECUTE`、`SUMMARIZE_*`、`CONTEXT_USAGE`、`AGENT_LOG`）**只**通过该回调到达处理器，并且回调会 await 处理器（`awaitHandlers = true`）。

### 3.6 步骤键与消息 id

- **步骤键：** `stepKey = join('_', [run_id, thread_id, langgraph_node, langgraph_step, checkpoint_ns, streamSegment, (invokedToolIds.size)?, ('reasoning' | 'post-reasoning-N')?])`。
- **消息 id：** `getMessageId(stepKey)` 返回一个新的 `msg_<nanoid>`，或从空的首个块中取提供商的 `chunk.id`。如果该键已经有 id，则返回 `undefined`，正是这一点阻止了同一个键出现第二个 MESSAGE_CREATION 步骤。
- **推理状态机：** `agentContext.currentTokenType`（`'text'｜'think'｜'think_and_text'`）和 `tokenTypeSwitch`（`'reasoning'｜'content'`）驱动键的变化。离开推理状态时 `reasoningTransitionCount` 递增，从而产生一个新步骤。
- **工具调用：** 通常每个工具调用对应一个 TOOL_CALLS 步骤。在它之前的消息步骤会被标记为 `messageStepHasToolCalls`，并且不会发出空的文本片段。

---

## 4. 内容聚合（`createContentAggregator`，`src/stream.ts`）

`createContentAggregator()` 返回 `{contentParts: (MessageContentComplex|undefined)[], stepMap: Map<stepId, RunStep>, aggregateContent({event, data})}`。

**它维护的状态：**

- `stepMap`
- `toolCallContentIndexMap`（tool_call_id → 内容索引）
- `sourceContentIndexMap`（事件中的 `runStep.index` → 物理索引）
- `toolStepContentMap`（每个工具步骤一项：`indices`、`chunkIndices`（chunk.index → 内容索引）、`unclaimedIndices`、`unboundIndices`、`callIdsByIndex`）
- `nextContentIndex`（物理追加游标）
- `contentMetaMap`（索引 → `{agentId, groupId}`）

`syncSeededContent()` 允许宿主预先填充 `contentParts`，例如填入暂停之前的内容。

**逐事件的处理：**

- **`on_run_step`**
  - TOOL_CALLS 步骤：每个工具调用如果在 `toolCallContentIndexMap` 中已有内容索引就直接使用；否则第一个调用使用 `resolveSourceContentIndex(step.index)`，后续调用分配新的索引。
  - 其他步骤：`resolveSourceContentIndex(step.index)`，取 `max(nextContentIndex, sourceIndex)`。
  - 步骤以重映射后的 `index` 保存，并记录 agentId/groupId 元数据。
  - 如果设置了 `summary`，执行 `updateContent(index, summary)`。
  - TOOL_CALLS 步骤还会登记每个调用，并写入 `{type:'tool_call', tool_call:{args, name, id}}`。
- **`on_message_delta` / `on_reasoning_delta`：** 每个片段都经过 `updateContent(step.index, part)`。
- **`on_run_step_delta`（tool_calls）：** 对每个块，按以下顺序解析内容索引：
  1. 显式的 `id` → 已知的索引；
  2. `chunk.index` → `chunkIndices`；
  3. 该步骤唯一的索引；
  4. 第一个未认领或未绑定的索引；
  5. 新分配一个。

  然后写入 `{type:'tool_call', tool_call:{args: chunk.args ?? '', name, id, auth, expires_at}}`。
- **`on_run_step_completed`：**
  - 摘要结果直接替换槽位。
  - 工具结果按 `tool_call.id` 找到槽位（找不到时回退到该步骤唯一的索引），然后执行 `updateContent(idx, {type:'tool_call', tool_call}, finalUpdate=true)`。
- **`on_summarize_delta`：** `updateContent(step.index, delta.summary)` 追加到 `content`。
- **`on_summarize_complete`：** 按 `summary.boundary.messageId` 查找步骤，并用最终摘要**替换**该片段。
- **`on_agent_update`：** 把片段放到解析出的索引处。

**`updateContent(index, part, final)` 的合并规则：**

- 会创建一个占位的 `{type}`，`tool_call` 除外。类型不匹配时忽略并给出警告。
- **text：** `text` 拼接；传入片段带有 `tool_call_ids` 时替换，仅含引用的增量则保留原值；`citations` 追加。
- **think：** `think` 拼接。
- **agent_update、`toolCall`/`toolResponse`（Google）：** 替换。
- **summary：** `{...incoming, content: [...old.content, ...incoming.content]}`。
- **image_url：** 保留。
- **tool_call：**
  - 没有 name 的传入片段会被忽略，除非这是最终更新。
  - 一旦 `progress === 1`，非最终更新都会被忽略。
  - `args` 按字符串拼接，但对象形式的 args 或最终更新会直接替换。
  - `id` 和 `name` 保留第一个非空值。
  - `auth`/`expires_at` 会向后沿用。
  - 最终更新设置 `progress: 1`、`output` 和 `outcome`。
  - 结果为 `{type:'tool_call', tool_call:{id, name, args, type:'tool_call', progress?, output?, outcome?, auth?, expires_at?}}`。
- 每次更新后，都会把该索引的 `agentId`/`groupId` 应用到片段上。

最终出现在 `contentParts` 中的内容片段类型：

- `text {text, citations?, tool_call_ids?, phase?}`
- `think {think}`
- `tool_call {tool_call}`
- `summary`
- `agent_update`
- `image_url`
- Google `toolCall`/`toolResponse`

数组中可能有 `undefined` 空洞，例如某个消息步骤从未收到文本时；消费方需要把它们过滤掉。

---

