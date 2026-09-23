# @librechat/agents: Public API, Run Lifecycle & Event Contract

> Reference chapter for the [@librechat/agents SDK architecture](../README.md). Produced by static reading of
> [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) at `v3.9.3` (`2d5653d`). Paths are relative to
> that repository's root. Line numbers are approximate; check the code when a detail matters.

All paths below are relative to the SDK repo root. Every claim comes from reading the code.

---

## 1. Public API surface

### 1.1 Package subpath exports (`package.json`)

| Subpath | Source | What it exposes |
|---|---|---|
| `.` (root) | `src/index.ts` | The main runtime. Details in 1.2. |
| `./provider-registration` | `src/provider-registration.ts` | `registerProvider()`, plus the types `ProviderFamily` (`'openai'｜'anthropic'｜'bedrock'｜'google'｜'mistral'｜'generic'`), `ProviderRegistrationOptions {provider, model: ctor, family?, manualToolStream?, strictAlternation?}` and `CustomProviderOptionsMap` (a type you extend by declaration merging). The implementation is in `src/llm/providerRegistry.ts`. It is a global registry keyed by `Symbol.for('@librechat/agents:providerRegistry:v1')`. It throws on a duplicate registration and returns a disposer. `family` defaults to `generic`; the two booleans default to false. |
| `./openai` | `src/openai/index.ts` | Adapter that turns graph events into OpenAI Chat Completions SSE (section 6.1). |
| `./responses` | `src/responses/index.ts` | Adapter that turns graph events into OpenAI Responses API SSE (section 6.2). |
| `./langchain`, `./langchain/messages`, `/messages/tool`, `/prompts`, `/runnables`, `/tools`, `/google-common`, `/openai`, `/utils/env`, `/language_models/chat_models` | `src/langchain/*` | Pass-through re-exports of `@langchain/core` and related classes (`AIMessage`, `HumanMessage`, `ToolMessage`, `mapStoredMessagesToChatMessages`, …). They exist so the host uses the same LangChain instance as the SDK. |
| `./llm/openai`, `./llm/anthropic`, `./llm/google`, `./llm/bedrock`, `./llm/vertexai`, `./llm/mistral`, `./llm/openrouter` | `src/llm/<provider>/index.ts` | The SDK's patched provider chat-model classes, plus provider helpers such as the OpenAI cache-breakpoint utilities. They were moved off the root barrel so importing the root does not load every provider SDK. |

### 1.2 Root barrel (`src/index.ts`)

| Export group | Key symbols | Purpose |
|---|---|---|
| `./run` | `Run`, `defaultOmitOptions` | Run orchestrator (section 2). |
| `./stream` | `ChatModelStreamHandler`, `createContentAggregator`, `getChunkContent`, `SDK_STREAM_DISPATCH`, `dispatchesChatModelStream` | Turns LLM chunks into step events, and folds step events back into content parts. |
| `./events` | `HandlerRegistry`, `composeEventHandlers`, `ModelEndHandler`, `ToolEndHandler`, `LLMStreamHandler`, `TestLLMStreamHandler`, `TestChatStreamHandler`, `createMetadataAggregator` | Handler registry and stock handlers. |
| `./messages` | `formatAgentMessages`, `convertMessagesToContent`, `getMessageId`, pruning, caching, alternation, fading, assistant-phase helpers, … | Message formatting and conversion between messages and content. |
| `./graphs` | `StandardGraph`, `MultiAgentGraph`, `createGraph`, `HandoffLimitError` | Graph builders. |
| `./agents/projection` | `projectAgentContextUsage` | Host-side estimate of context usage before a send. |
| `./summarization` | `shouldTriggerSummarization`, … | Summarization triggers. |
| `./tools/*` | Calculator, CodeExecutor, BashExecutor, ProgrammaticToolCalling (+Bash), SkillTool, SubagentTool, `subagent/*` (`InMemorySubagentTaskStore`), ReadFile, skillCatalog, ToolSearch, `ToolNode`, intentArg, schema, `handlers` (`handleToolCalls`, `handleToolCallChunks`, `handleServerToolResult`), local and cloudflare engines, search | Built-in tools and the ToolNode. |
| `./common` | All enums and constants (section 3.1) | |
| `./utils` | `createHandlers` (`src/utils/handlers.ts`), `RunnableCallable`, `joinKeys`, `resetIfNotEmpty`, token utilities, truncation, errors | |
| `./hooks` | `HookRegistry`, `executeHooks`, `createToolPolicyHook`, `createWorkspacePolicyHook`, `HOOK_EVENTS` | Lifecycle hooks: `RunStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `PreemptBoundary`, `PermissionDenied`, `SubagentStart`, `SubagentStop`, `Stop`, `StopFinalize`, `StopFailure`, `PreCompact`, `PostCompact`. |
| `./session` | `AgentSession`, `createAgentSession`, `createRunHandlers`, `JsonlSessionStore`, `SessionManager`, `deriveMessages`, `serializeMessage`/`deserializeMessage` | Programmatic session wrapper (section 6.3). |
| `./eventActor` | `EventActorExecutor`, `createEventActorExecutor` | Durable actor-style invocation on checkpoint forks (prepare, commit, discard, suspend, resume). |
| `./hitl` | `askUserQuestion`, `askUserQuestions`, approval-review helpers | Human-in-the-loop (HITL) interrupt helpers. |
| `export type * from './types'` | Every type in `src/types/*` | |
| LangGraph re-exports | `Command`, `INTERRUPT`, `interrupt`, `MemorySaver`, `BaseCheckpointSaver`, `isInterrupted`, `Interrupt` | Keeps the host on the same LangGraph version as the SDK. |
| LLM | `getChatModelClass`, `registerProvider`, `initializeModel`, `attemptInvoke`, `tryFallbackProviders`, `prepareProviderRequest`, `canSealPreempt`, `isThinkingEnabled`, `getMaxOutputTokensKey`, `smoothStream`, `resolveStreamDelay`, `DEFAULT_STREAM_DELAY`, `computeAdaptivePieceSize`, `FakeChatModel`, `createFakeStreamingLLM`, `DEFAULT_MAX_TOOL_CALL_ARG_BYTES`, `StreamLimitExceededError`, `resolveStreamLimits`, `markTokenCounterCacheCompatible`, `LANGFUSE_OBSERVATION_METADATA_ARTIFACT_KEY` | |

---

## 2. Run lifecycle (`src/run.ts`)

### 2.1 `RunConfig` (`src/types/run.ts`)

| Field | Meaning |
|---|---|
| `runId` (required) | Used as the run id, as `configurable.run_id`, in step keys and on `RunStep.runId`. In LibreChat this is the response message id. |
| `graphConfig` | One of three shapes: `LegacyGraphConfig` (`type?:'standard'`, `llmConfig`, `tools`, `instructions`, …), `StandardGraphConfig` (`{type?:'standard', agents: AgentInputs[], signal?, compileOptions?}`, where only `agents[0]` is used), or `MultiAgentGraphConfig` (`{type:'multi-agent', agents, edges: GraphEdge[], entryAgentId?, maxHandoffs?, compileOptions?}`). `compileOptions = {checkpointer?, interruptBefore?, interruptAfter?}`. |
| `customHandlers?: Record<eventName, EventHandler>` | **The host's event sink.** Each entry is registered into the `HandlerRegistry`. |
| `returnContent?` | If true, `processStream` returns `Graph.getContentParts()`, which comes from `convertMessagesToContent(messages.slice(startIndex))` and **not** from the aggregator. |
| `tokenCounter?`, `indexTokenCountMap?` | Used for pruning. If a map is given without a counter, `Run.create` builds a tiktoken counter from the model name. |
| `calibrationRatio?`, `fadingTier?`, `fadingTiers?` | Context-management state carried between runs. Read back with `getCalibrationRatio()` and `getFadingTier(s)()`. |
| `hooks?: HookRegistry`, `maxStopContinuations?` (default 8) | Lifecycle hooks. A Stop hook with `decision:'block'` can re-enter the graph with injected messages. |
| `preemption?: StreamPreemption` | Cooperative stop. `{shouldPreempt(): boolean, subscribe?(wake), restartGraceMs? (default 2000), maxSeals? (default 8)}`. A **seal** keeps the partial turn and self-loops. A **restart** discards a turn that holds only reasoning. |
| `streamLimits?` | `{maxToolCallArgBytes? (64 KiB default), maxToolCallArgBytesByTool?, maxDeltaEventsPerTurn?}`. A breach throws `StreamLimitExceededError`. |
| `humanInTheLoop?: {enabled}` | Off by default. When on, a `MemorySaver` is installed if no checkpointer was given, and `ask` tool decisions call `interrupt()`. |
| `langfuse?`, `subagentUsageSink?`, `subagentTasks?`, `subagentContext?`, `initialSessions?`, `toolOutputReferences?`, `eagerEventToolExecution?`, `codeSessionToolNames?`, `interruptingToolNames?`, `toolExecution?`, `skipCleanup?` | Tracing, subagents and tool execution. |

The per-call `RunStreamConfig` is `Partial<RunnableConfig> & {version:'v1'｜'v2', run_id?, durability?:'async'｜'sync'｜'exit'}`. Important `configurable` keys:

- `thread_id`: the checkpoint thread and the Langfuse session.
- `user_id`
- `requestBody.parentMessageId`
- `checkpoint_id` / `checkpoint_ns`

The `signal` field on this config is the per-call abort signal.

### 2.2 Step by step

1. **`Run.create(config)`** (static, async)
   - Loads the providers in source mode.
   - Builds a token counter if needed.
   - Calls `new Run()`. The constructor:
     - creates a `HandlerRegistry` and registers every `customHandlers` entry;
     - saves the hooks, HITL, preemption, limits and other config;
     - builds the graph with `createLegacyGraph` (`createGraph({kind:'standard'})`, where legacy `llmConfig` becomes `AgentInputs{agentId:'default', provider, clientOptions}`) or with `createMultiAgentGraph`;
     - applies the HITL checkpointer fallback, then `applyGraphRuntimeConfig` (hooks, HITL, output references, eager execution, …);
     - calls `graph.createWorkflow()` to compile LangGraph;
     - sets `Graph.handlerRegistry = registry`;
     - seeds `initialSessions`.
2. **`processStream(inputs: IState｜Command, callerConfig, streamOptions?)`**
   - `isResume = inputs instanceof Command`. On resume it runs `restoreInterruptFromCheckpoint` (`getState({subgraphs:true})`), restores the run-step resume state and checkpoint messages, and restores the tool-replay config.
   - `recursionLimit` = (caller value or 50) + `maxSeals` when preemption is on.
   - Strips internal configurable keys.
   - On a fresh run: `graph.resetValues(keepContent, checkpointScope)` clears `contentData`, `stepKeyIds`, `toolCallStepIds` and the other maps; handoff routing starts; a new stop-continuation execution id is set.
   - Callbacks:
     - a `BaseCallbackHandler.fromMethods({handleCustomEvent: customEventCallback})` with `awaitHandlers = true`;
     - optional `ClientCallbacks` (`handleToolError`, `handleToolStart`, `handleToolEnd`, each called as `(graph, ...args)`);
     - the Langfuse handler from `createLangfuseHandler(...)`, which carries trace metadata (`messageId=runId`, `parentMessageId`, agent id and name), tags `['librechat','agent']`, and a deterministic trace id seeded from `runId` when `deterministicTraceId` is set.
   - Sets `config.runId = config.run_id = configurable.run_id = this.id`.
   - `durability` defaults to `'exit'` when a checkpointer is present.
   - **Pre-stream hooks** (fresh runs only): `RunStart`, then `UserPromptSubmit` on the last human message.
     - `preventContinuation`, `deny` or `ask` → `_haltedReason` is set and the method returns `undefined` without any model call.
     - `additionalContext` strings are appended as one `HumanMessage{additional_kwargs:{role:'system', source:'hook'}}`.
   - **`consumeStream` loop.** Each iteration calls `graphRunnable.streamEvents(inputs, {...config, runName}, {raiseError:true, ignoreCustomEvent:true})` and handles every event:
     - Custom graph events (names in `CUSTOM_GRAPH_EVENTS`) are **skipped**. They reach handlers through the callback path instead (section 5).
     - The first `data.chunk[INTERRUPT]` it sees is captured as `_interrupt = {interruptId, threadId, payload}`.
     - `handlerRegistry.getHandler(event.event)?.handle(eventName, data, metadata, graph)`. This covers the native events: `on_chat_model_stream`, `on_chat_model_end`, `on_tool_end`, `on_chain_*`, …
     - On `on_chat_model_end` it then calls `graph.closeOpenMessageStep(metadata, modelEndAt)`, which emits `ON_RUN_STEP_CLOSED` for that agent lane's open message step.
     - A hook halt signal breaks the loop with `_haltedReason`.
   - When the stream ends:
     - If there is an interrupt, it resolves the checkpoint id for resume and returns.
     - If there is a halt, it returns.
     - Otherwise it computes `stopReason` (`preemptHaltReason`, `'preempt_incomplete'` or `'output_truncated'`) and runs the `Stop` and `StopFinalize` hooks. A Stop hook that blocks and injects messages starts another loop iteration with `messages: injected` (checkpointer present) or `messages: [...graph.messages, ...injected]` (no checkpointer), and advances `streamSegment`.
   - **Error path:** sets `streamThrew`; `HandoffLimitError` sets the halt reason `'handoff_limit'`; Langfuse `handleChainError`; `streamAborted` is set from `signal.aborted`; the `StopFailure` hook runs; the error is rethrown.
   - **`finally` block:**
     - clears the hook session unless the run is awaiting resume;
     - disposes the Langfuse handler;
     - **terminal sweep:** `closeUnfinishedRunSteps(status, terminalAt)`, where status is `cancelled` (abort or halt), `failed` (error) or `completed`. The sweep is skipped when the run paused for HITL;
     - clears configurable and callbacks;
     - reads `getContentParts()` if `returnContent` is set;
     - saves the calibration ratio and fading tiers;
     - computes the handoff outcome;
     - calls `clearHeavyState()` unless the run is awaiting resume.
3. **Completion inspectors:**
   - `getInterrupt<T>()` returns `RunInterruptResult {interruptId, threadId?, checkpointId?, checkpointNs?, payload}`. The payload is `ToolApprovalInterruptPayload {type:'tool_approval', action_requests, review_configs, hook_session_id?, subagent?}` or an ask-user-question payload.
   - `getHaltReason()`, `getOutputTruncated()`, `getPreemptStats() {seals, restarts, emptyBoundaries}`, `getHandoffOutcome()`, `getRunMessages()`, `getDiscoveredTools()`, `getToolCount()`, `getChildCheckpointThreadIds()`, `getFileCheckpointer()`, `rewindFiles()`.
   - `run.Graph.getRunSteps(agentId?)`, `getRunStepsByAgent()`, `getContentPartAgentMap()`, `getActiveAgentIds()`.
4. **`resume(resumeValue, callerConfig, streamOptions?, {update?, goto?})`**
   - Resolves the checkpoint id: from the interrupt, or by scanning `getStateHistory` for the matching interrupt id.
   - Scopes the value as `{[interruptId]: value}` unless it is already a map keyed by that id.
   - Calls `processStream(new Command({resume, update?, goto?}), resumeConfig)`.
   - The host must rebuild the Run with the same `thread_id` and checkpointer. The default resume value is `ToolApprovalDecision[]` or a map keyed by `tool_call_id`.
5. **Abort and preempt**
   - Abort: `callerConfig.signal` or `graphConfig.signal`. The stream throws; open steps are closed as `cancelled`.
   - Preempt: `RunConfig.preemption` (see 2.1). A sealed turn ends early; the `PreemptBoundary` hook supplies the injected messages. Discarded turns emit a synthetic `on_chat_model_end` with `response_metadata.preempted/preemptDiscarded`.
6. **Auxiliary generators.** Each makes its own model call with Langfuse wiring and emits **no** graph events.
   - `generateTitle({provider, inputText, contentParts, titlePrompt?, clientOptions?, chainOptions?, skipLanguage?, titleMethod: 'completion'｜'structured'｜'functions', titlePromptTemplate?})` → `{language?, title?}`.
   - `generateActivityLabel(...)` → `{label?}`.
   - `generateReasoningLabel(...)` → `{label?, usage?}`.
   - `generateActivityPhaseLabel(...)` → `{label?}`.
7. **Token usage.** There is no built-in accumulator on `Run`. The host registers `new ModelEndHandler(collectedUsage: UsageMetadata[])` on `on_chat_model_end`; it pushes `data.output.usage_metadata`. Subagent child usage never reaches that handler; it arrives through `RunConfig.subagentUsageSink(SubagentUsageEvent {usage, model?, provider?, subagentType, subagentRunId, subagentAgentId, runId (root), parentRunId?, depth?, ancestry?, memberAgentId?, subagentKind?})`. `ON_CONTEXT_USAGE` carries the per-call budget.

### 2.3 How LLM chunks become step events

There are two paths:

- **Host path.** The host registers `ChatModelStreamHandler` (or a wrapper branded with `SDK_STREAM_DISPATCH`) on `on_chat_model_stream`. Run's `streamEvents` loop passes each chunk to it, and it dispatches the step events. Runs on this path cannot seal.
- **In-graph path.** When no branded handler is registered, `attemptInvoke` in `src/llm/invoke.ts` runs a private `ChatModelStreamHandler` inside the model node for each chunk. This is the path that supports preemption.

---

## 3. The event contract

### 3.1 Enums (`src/common/enum.ts`), complete

**GraphEvents.** Custom events:

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

Official LangChain events:

- `ON_CUSTOM_EVENT='on_custom_event'`
- `CHAT_MODEL_START='on_chat_model_start'`, `CHAT_MODEL_STREAM='on_chat_model_stream'`, `CHAT_MODEL_END='on_chat_model_end'`
- `LLM_START='on_llm_start'`, `LLM_STREAM='on_llm_stream'`, `LLM_END='on_llm_end'`
- `CHAIN_START='on_chain_start'`, `CHAIN_STREAM='on_chain_stream'`, `CHAIN_END='on_chain_end'`
- `TOOL_START='on_tool_start'`, `TOOL_END='on_tool_end'`
- `RETRIEVER_START='on_retriever_start'`, `RETRIEVER_END='on_retriever_end'`
- `PROMPT_START='on_prompt_start'`, `PROMPT_END='on_prompt_end'`

**Providers:**

- `OPENAI='openAI'`, `VERTEXAI='vertexai'`, `BEDROCK='bedrock'`, `ANTHROPIC='anthropic'`
- `MISTRALAI='mistralai'`, `MISTRAL='mistral'`, `GOOGLE='google'`, `AZURE='azureOpenAI'`
- `DEEPSEEK='deepseek'`, `OPENROUTER='openrouter'`, `XAI='xai'`, `MOONSHOT='moonshot'`

**GraphNodeKeys:** `TOOLS='tools='`, `AGENT='agent='`, `SUMMARIZE='summarize='`, `ROUTER='router'`, `PRE_TOOLS='pre_tools'`, `POST_TOOLS='post_tools'`. Node names are `agent=<agentId>` and so on. `getAgentContext` parses `metadata.langgraph_node` to find the agent.

**GraphNodeActions:** `TOOL_NODE='tool_node'`, `CALL_MODEL='call_model'`, `ROUTE_MESSAGE='route_message'`.

**CommonEvents:** `LANGGRAPH='LangGraph'`.

**StepTypes:** `TOOL_CALLS='tool_calls'`, `MESSAGE_CREATION='message_creation'`.

**ContentTypes:**

- `TEXT='text'`, `ERROR='error'`, `THINK='think'`, `TOOL_CALL='tool_call'`
- `IMAGE_URL='image_url'`, `IMAGE_FILE='image_file'`
- `THINKING='thinking'` (Anthropic), `REASONING='reasoning'` (Vertex/Google), `REASONING_CONTENT='reasoning_content'` (Bedrock)
- `AGENT_UPDATE='agent_update'`, `SUMMARY='summary'`, `STEER='steer'`, `ACTIVITY_LABEL='activity_label'`

**ToolCallTypes:** `FUNCTION='function'`, `RETRIEVAL='retrieval'`, `FILE_SEARCH='file_search'`, `CODE_INTERPRETER='code_interpreter'`, `TOOL_CALL='tool_call'`.

**Callback:** `TOOL_ERROR='handleToolError'`, `TOOL_START='handleToolStart'`, `TOOL_END='handleToolEnd'`, `CUSTOM_EVENT='handleCustomEvent'`. The other LangChain callback names are commented out in the source.

**Constants:**

- `OFFICIAL_CODE_BASEURL='https://api.librechat.ai/v1'`
- `EXECUTE_CODE='execute_code'`, `TOOL_SEARCH='tool_search'`, `PROGRAMMATIC_TOOL_CALLING='run_tools_with_code'`, `WEB_SEARCH='web_search'`
- `CONTENT_AND_ARTIFACT='content_and_artifact'`
- `LC_TRANSFER_TO_='lc_transfer_to_'`, `HANDOFF_PARALLEL_BATCH='__handoff_parallel_batch'`, `HANDOFF_GROUP_ID='__handoff_group_id'`
- `MCP_DELIMITER='_mcp_'`, `ANTHROPIC_SERVER_TOOL_PREFIX='srvtoolu_'`
- `SKILL_TOOL='skill'`, `INVOKED_PROVIDER='__invoked_provider'`, `INVOKED_MODEL='__invoked_model'`
- `READ_FILE='read_file'`, `BASH_TOOL='bash_tool'`, `BASH_PROGRAMMATIC_TOOL_CALLING='run_tools_with_bash'`, `SUBAGENT='subagent'`
- `WRITE_FILE`, `EDIT_FILE`, `GREP_SEARCH`, `GLOB_SEARCH`, `LIST_DIRECTORY`, `COMPILE_CHECK` (snake_case string values)

**TitleMethod:** `STRUCTURED`, `FUNCTIONS`, `COMPLETION`.

**EnvVar:** `CODE_BASEURL='LIBRECHAT_CODE_BASEURL'`, `CODE_API_RUN_TIMEOUT_MS`.

`src/common/constants.ts` adds:

- `DEFAULT_RECURSION_LIMIT=50`, `DEFAULT_MAX_SEALS=8`, `DEFAULT_PREEMPT_RESTART_GRACE_MS=2000`, `DEFAULT_MAX_STOP_CONTINUATIONS=8`, `PREEMPT_BOUNDARY_HOOK_TIMEOUT_MS=120000`
- `ANTHROPIC_TOOL_TOKEN_MULTIPLIER=2.6`, `DEFAULT_TOOL_TOKEN_MULTIPLIER=1.4`
- Run names: `AgentGraph`, `MultiAgentGraph`, `AgentModelCall`, `StepLabel`, `ReasoningLabel`, `MultiStepLabel`, `MultiStepLabelGeneration`
- `SUBAGENT_CONTEXT_VERSION=1`, `COMPACTION_SEMANTIC_INDEX_LIMITS`

### 3.2 Handler signature

Every event goes through the same interface:

```ts
interface EventHandler { handle(event: string, data, metadata?: Record<string,unknown>, graph?: StandardGraph|MultiAgentGraph): void|Promise<void> }
```

`metadata` is the LangGraph callback metadata: `langgraph_node`, `langgraph_step`, `checkpoint_ns`, `run_id`, `thread_id`, `ls_provider`, `ls_model_name`, and the SDK keys (`__invoked_provider`, `__handoff_group_id`, …).

### 3.3 Core shapes (`src/types/stream.ts`)

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

In `ToolCompleteEvent`, `index` is the **tool usage turn** (`toolUsageCount` or `request.turn`), not the content index. The aggregator ignores it and matches on `tool_call.id`.

### 3.4 Every event: when it fires and what it carries

| Event | Emitted by and when | Payload |
|---|---|---|
| `on_run_step` | `Graph.dispatchRunStep`. Fires when a new MESSAGE_CREATION step begins: the first non-empty content for a new `stepKey`, a phase split, a switch from reasoning to text, or a Google server-tool part. Also fires for a TOOL_CALLS step: the first tool-call chunk carrying id and name (`handleToolCallChunks`), or each complete `tool_call` (`handleToolCalls`). Summarization dispatches a MESSAGE_CREATION step with `summary` set to a placeholder `{type:'summary', model, provider}`. | `RunStep` |
| `on_run_step_delta` | `handleToolCallChunks`, once per streamed chunk that has `tool_call_chunks`. The host may also emit `auth`/`expires_at` deltas (MCP OAuth). | `RunStepDeltaEvent` |
| `on_message_delta` | `dispatchMessageDelta` for text chunks. | `MessageDeltaEvent`. Parts are `{type:'text', text, citations?, phase?, tool_call_ids?}`; Google server-tool chunks can also carry `toolCall`/`toolResponse` parts. |
| `on_reasoning_delta` | `dispatchReasoningDelta` for reasoning chunks. Sources: `additional_kwargs[reasoningKey]`, OpenAI `reasoning.summary[0].text`, Anthropic `thinking`, Bedrock `reasoning_content`, Google `reasoning`, `<think>` tags. All are normalized to `{type:'think', think}`. | `ReasoningDeltaEvent` |
| `on_run_step_completed` | Tool calls: `ToolNode` (normal path and the early per-result path) or `dispatchEagerToolCompletions` (`eager:true`), sent as a custom event; `handleToolCallErrorStatic` sends it directly to the handler. Summary: the summarize node's `dispatchRunStepCompleted` (direct handler call; note that `metadata` receives `configurable`, a quirk). | `{result: ToolCompleteEvent ｜ SummaryCompleted}` |
| `on_run_step_closed` | `Graph.closeRunStep`, exactly once per step. The first close wins and terminal states never change. Triggers: (a) a successor step in the same agent lane closes the previous message step; (b) `on_chat_model_end` closes the lane's open message step; (c) `recordStepCompletion` once every pending tool call of a TOOL_CALLS step has completed; (d) a preempt discard closes it as `cancelled`; (e) an empty summary closes it as `failed`; (f) the end-of-run sweep. | `RunStepClosedEvent` |
| `on_tool_execute` | `ToolNode` in event-driven mode (`AgentInputs.toolDefinitions` non-empty), or the eager streaming path. **Host-handled RPC.** | `ToolExecuteBatchRequest {toolCalls: ToolCallRequest[] {id, name, args, stepId?, turn?, codeSessionContext?, runtimeSessionHint?}, userId?, agentId?, executionContext?, callerCapabilityProjection?, configurable?, metadata?, signal?, resolve(results: ToolExecuteResult[]), reject(err), onResult?(result)}`. `ToolExecuteResult = {toolCallId, content: string｜unknown[], artifact?, status:'success'｜'error', errorMessage?, outcome?, outcome_patch?:{from,to}, injectedMessages?: InjectedMessage[]}`. |
| `on_summarize_start` | Summarize node, right after the summary `on_run_step`. | `{agentId, provider, model?, messagesToRefineCount, summaryVersion, semanticIndexEntryCount?, semanticIndexCharCount?}` |
| `on_summarize_delta` | Each summarizer LLM chunk. | `{id: stepId, delta:{summary:{type:'summary', content: MessageContentComplex[], provider, model}}}` |
| `on_summarize_complete` | After `on_run_step_completed{summary}` on success, or with `error` on failure or empty output. | `{id: stepId, agentId, summary?: SummaryContentBlock, error?: string}`. `SummaryContentBlock = {type:'summary', content?, tokenCount?, coverage?:{retainedFromMessageId}, boundary?:{messageId, contentIndex}, summaryVersion?, model?, provider?, createdAt?}` |
| `on_subagent_update` | `SubagentExecutor`, calling the **parent's** registry handler directly (never through `streamEvents`). Child events are wrapped: `on_run_step`→`run_step`, `on_run_step_delta`→`run_step_delta`, `on_run_step_completed`→`run_step_completed`, `on_run_step_closed`→`run_step_closed`, `on_message_delta`→`message_delta`, `on_reasoning_delta`→`reasoning_delta`, `on_tool_execute`→`run_step` (the tool request is also forwarded to the parent's `on_tool_execute`). Phases `start`/`stop`/`error`/`control` are emitted around the child run. Updates go through a bounded ordered queue; low-priority phases are dropped under pressure. | `SubagentUpdateEvent {runId (root), parentRunId?, subagentRunId, parentToolCallId?, subagentType, subagentKind?, subagentAgentId, memberAgentId?, depth?, ancestry?, parentAgentId?, phase, data?, label?, timestamp (ISO)}` |
| `on_agent_log` | `emitAgentLog`, fire-and-forget. `debug` level only when `AGENT_DEBUG_LOGGING=true`. | `{level:'debug'｜'info'｜'warn'｜'error', scope:'prune'｜'summarize'｜'graph'｜'sanitize'｜string, message, data?, runId?, agentId?}` |
| `on_context_usage` | Once per model call, after pruning and **before** the model is invoked (awaited). Also sent at the end of a summarize-only run. | `ContextUsageEvent {runId?, agentId?, breakdown: TokenBudgetBreakdown, contextBudget?, effectiveInstructionTokens?, prePruneContextTokens?, remainingContextTokens?, calibrationRatio?}` |
| `on_agent_update` | **No SDK code emits it.** It is legacy; `createContentAggregator` still consumes it. | `AgentUpdate` |
| `on_chat_model_stream` | Native LangChain event reaching the Run loop, plus inline dispatch in `attemptInvoke`. | `{chunk: AIMessageChunk}` |
| `on_chat_model_end` | Native event, or a synthetic custom event after a seal or discard. | `{output: AIMessageChunk}`, with `usage_metadata` = `{input_tokens, output_tokens, total_tokens, input_token_details?, output_token_details?}` |
| `on_tool_end` | Native event for directly executed tools, plus a synthetic call for Anthropic server web-search results (`handleAnthropicSearchResults`). | `{input, output: ToolMessage｜Command}` |

### 3.5 Ordering guarantees

1. Every `on_run_step_delta`, `on_message_delta` and `on_reasoning_delta` for a step comes after that step's `on_run_step`.
2. Within one agent lane (`agentId` or `''`), the previous message step's `on_run_step_closed` is emitted **before** the next step's `on_run_step` (`trackDispatchedRunStep`). The content index is reserved before that await, so parallel lanes never collide.
3. `created_at` is stamped after the predecessor has closed and immediately before `on_run_step` is published.
4. A message step closes (`completed`) right after the host's `on_chat_model_end` handler returns. A TOOL_CALLS step closes after its **last** pending tool call's `on_run_step_completed`. Handler errors never prevent the close.
5. `on_context_usage` for a model call precedes that call's deltas.
6. Summary sequence: `on_run_step(summary placeholder)` → `on_summarize_start` → `on_summarize_delta*` → `on_run_step_completed{summary}` → `on_summarize_complete` → `on_run_step_closed`.
7. At the end of the run every step still `in_progress` receives a terminal `on_run_step_closed`, except when the run paused for HITL; those steps continue on resume through the persisted `RunStepResumeState`.
8. **Two delivery channels.** Step events (`RUN_STEP`, `RUN_STEP_DELTA`, `RUN_STEP_CLOSED`, `MESSAGE_DELTA`, `REASONING_DELTA`) are sent straight to the registry handler **and** echoed as LangChain custom events for tracing. The Run's custom-event callback drops an echo when `graph.hasHandlerDispatchedEvent(name, id)` is true, so handlers see each event exactly once. Every other custom event (`COMPLETED`, `TOOL_EXECUTE`, `SUMMARIZE_*`, `CONTEXT_USAGE`, `AGENT_LOG`) reaches handlers **only** through that callback, and the callback awaits handlers (`awaitHandlers = true`).

### 3.6 Step keys and message ids

- **Step key:** `stepKey = join('_', [run_id, thread_id, langgraph_node, langgraph_step, checkpoint_ns, streamSegment, (invokedToolIds.size)?, ('reasoning' | 'post-reasoning-N')?])`.
- **Message id:** `getMessageId(stepKey)` returns a new `msg_<nanoid>`, or the provider's `chunk.id` taken from an empty first chunk. It returns `undefined` if an id already exists for that key, which is what stops a second MESSAGE_CREATION step for the same key.
- **Reasoning state machine:** `agentContext.currentTokenType` (`'text'｜'think'｜'think_and_text'`) and `tokenTypeSwitch` (`'reasoning'｜'content'`) drive key changes. Leaving reasoning increments `reasoningTransitionCount`, which creates a new step.
- **Tool calls:** usually one TOOL_CALLS step per tool call. The message step that came before is marked `messageStepHasToolCalls`, and no empty text part is emitted.

---

## 4. Content aggregation (`createContentAggregator`, `src/stream.ts`)

`createContentAggregator()` returns `{contentParts: (MessageContentComplex|undefined)[], stepMap: Map<stepId, RunStep>, aggregateContent({event, data})}`.

**State it keeps:**

- `stepMap`
- `toolCallContentIndexMap` (tool_call_id → content index)
- `sourceContentIndexMap` (the event's `runStep.index` → physical index)
- `toolStepContentMap` (per tool step: `indices`, `chunkIndices` (chunk.index → content index), `unclaimedIndices`, `unboundIndices`, `callIdsByIndex`)
- `nextContentIndex` (a physical append cursor)
- `contentMetaMap` (index → `{agentId, groupId}`)

`syncSeededContent()` allows the host to pre-seed `contentParts`, for example with content from before a pause.

**Handling per event:**

- **`on_run_step`**
  - TOOL_CALLS step: each tool call gets its content index from `toolCallContentIndexMap` if already known; otherwise the first call uses `resolveSourceContentIndex(step.index)` and later calls get a newly allocated index.
  - Other steps: `resolveSourceContentIndex(step.index)`, which uses `max(nextContentIndex, sourceIndex)`.
  - The step is stored with its remapped `index`, and agentId/groupId metadata is recorded.
  - If `summary` is set, it runs `updateContent(index, summary)`.
  - TOOL_CALLS steps also register each call and write `{type:'tool_call', tool_call:{args, name, id}}`.
- **`on_message_delta` / `on_reasoning_delta`:** each part goes through `updateContent(step.index, part)`.
- **`on_run_step_delta` (tool_calls):** for each chunk, the content index is resolved in this order:
  1. explicit `id` → the known index;
  2. `chunk.index` → `chunkIndices`;
  3. the only index of the step;
  4. the first unclaimed or unbound index;
  5. a new allocation.

  It then writes `{type:'tool_call', tool_call:{args: chunk.args ?? '', name, id, auth, expires_at}}`.
- **`on_run_step_completed`:**
  - Summary results replace the slot outright.
  - Tool results find their slot by `tool_call.id` (falling back to the step's only index), then run `updateContent(idx, {type:'tool_call', tool_call}, finalUpdate=true)`.
- **`on_summarize_delta`:** `updateContent(step.index, delta.summary)` appends to `content`.
- **`on_summarize_complete`:** looks up the step by `summary.boundary.messageId` and **replaces** the part with the final summary.
- **`on_agent_update`:** places the part at the resolved index.

**`updateContent(index, part, final)` merge rules:**

- A placeholder `{type}` is created, except for `tool_call`. A type mismatch is ignored with a warning.
- **text:** `text` is concatenated; `tool_call_ids` are replaced when the incoming part has them and kept for citation-only deltas; `citations` are appended.
- **think:** `think` is concatenated.
- **agent_update, `toolCall`/`toolResponse` (Google):** replaced.
- **summary:** `{...incoming, content: [...old.content, ...incoming.content]}`.
- **image_url:** kept.
- **tool_call:**
  - Incoming parts without a name are ignored unless this is the final update.
  - Once `progress === 1`, non-final updates are ignored.
  - `args` are concatenated as strings, except that object args, or a final update, replace them.
  - `id` and `name` keep the first non-empty value.
  - `auth`/`expires_at` are carried forward.
  - A final update sets `progress: 1`, `output` and `outcome`.
  - The result is `{type:'tool_call', tool_call:{id, name, args, type:'tool_call', progress?, output?, outcome?, auth?, expires_at?}}`.
- After every update the index's `agentId`/`groupId` are applied to the part.

Content-part types that end up in `contentParts`:

- `text {text, citations?, tool_call_ids?, phase?}`
- `think {think}`
- `tool_call {tool_call}`
- `summary`
- `agent_update`
- `image_url`
- Google `toolCall`/`toolResponse`

The array can contain `undefined` holes, for example when a message step never received text; consumers filter them out.

---

## 5. Handler registry and callback model

- `HandlerRegistry` is a `Map<string, EventHandler>` with **one handler per event**. `composeEventHandlers(...sets)` chains handlers that share a key, runs them sequentially, and keeps the `SDK_STREAM_DISPATCH` brand.
- **How a host plugs in:** pass `customHandlers` to `RunConfig`. The standard LibreChat set (see `src/utils/handlers.ts:createHandlers` and `src/session/handlers.ts:createRunHandlers`) is:
  - `on_chat_model_stream: new ChatModelStreamHandler()`
  - `on_chat_model_end: new ModelEndHandler(collectedUsage)`
  - `on_tool_end: new ToolEndHandler(cb)`, which skips calls flagged as programmatic tool calls
  - `on_run_step`, `on_run_step_delta`, `on_run_step_completed`, `on_message_delta`, `on_reasoning_delta`, `on_summarize_delta`, `on_summarize_complete`: each forwards to SSE and calls `aggregateContent`
  - `on_run_step_closed`
  - `on_tool_execute`: the host runs the tools and calls `resolve`
  - `on_subagent_update`, `on_context_usage`, `on_agent_log` as needed
- `processStream(..., {callbacks: ClientCallbacks})` adds LangChain callback methods, each called as `(graph, ...args)`.
- `createMetadataAggregator()` is an `handleLLMEnd` callback that collects `response_metadata`.
- A child graph inherits only the parent's `on_tool_execute` and `on_subagent_update` handlers (`src/graphs/Graph.ts` ~L218). Everything else from the child is wrapped into `on_subagent_update`.

---

## 6. Adapters and AgentSession

### 6.1 `./openai`: Chat Completions SSE

- Setup: `createOpenAIStreamTracker()` and `createOpenAIHandlers({writer, context:{requestId, model, created}, tracker})`.
- `on_message_delta` → `data: {object:'chat.completion.chunk', choices:[{index:0, delta:{content}, finish_reason:null}]}`. The first chunk is always `{role:'assistant'}`.
- `on_reasoning_delta` → `delta.reasoning`.
- `on_run_step_delta` (tool_calls) → `delta.tool_calls[{index, id?, type:'function', function:{name?, arguments}}]`, with argument deltas tracked per step.
- `on_run_step` (tool_calls) → sends the complete call, computing only the argument suffix not yet sent.
- `on_chat_model_end` → accumulates usage.
- `sendOpenAIFinalChunk(config, finishReason?)` sends the `finish_reason` chunk (`tool_calls` if the last chunk was a tool call, otherwise `stop`), then a usage chunk with `completion_tokens_details.reasoning_tokens`, then `data: [DONE]`.
- Helpers: `createChatCompletionChunk`, `createChatCompletionUsageChunk`, `writeOpenAISSE`.

### 6.2 `./responses`: Responses API SSE

- Setup: `createResponseTracker()` and `createResponsesEventHandlers({writer, context:{responseId, model, createdAt, previousResponseId?, instructions?}, tracker})`.
- Every event is written as `event: <type>\ndata: {…, sequence_number}`.
- The first event is `response.created`.
- `on_message_delta` → one `message` item (`output_item.added`, then `response.output_text.delta`).
- `on_reasoning_delta` → one `reasoning` item (`response.reasoning_text.delta`).
- `on_run_step` / `on_run_step_delta` (tool_calls) → `function_call` items (`output_item.added`, `response.function_call_arguments.delta`).
- `on_run_step_completed` → `function_call_arguments.done` and `output_item.done`.
- `on_chat_model_end` → usage.
- `emitResponseCompleted()` closes every open item (`output_text.done`, `reasoning_text.done`, `output_item.done`), then sends `response.completed` with the full `ResponseObject` and usage, then `[DONE]`.
- `buildResponse()` is exported.

### 6.3 `AgentSession` (`src/session/AgentSession.ts`)

`AgentSession` is a high-level programmatic wrapper around `Run`, similar to an agent SDK session. It is not used by the LibreChat server.

- `AgentSession.create(config)` or `createAgentSession(config)`. `config` is a `RunConfig` plus `cwd`, `sessionPath`, `sessionId`, `name`, `ephemeral` and `checkpointing` (boolean or `{enabled, checkpointer}`).
- Persistence is a JSONL tree store (`JsonlSessionStore`). Entry types: message, summary, compaction, checkpoint, label, run_event, session_state. Each entry has a `parentId`, which allows branching.
- `run(input)` returns `AgentSessionRunResult {text, content, messages, usage{inputTokens, outputTokens, totalTokens}, steps, interrupt, haltedReason, runId, threadId}`.
- `stream(input)` returns an `AgentSessionStream`: an async iterable of `AgentSessionStreamEvent {type, sequence, runId, threadId, timestamp, data?}`, plus `toTextStream()` and `finalResult()`. Event types:
  - `run.started`, `message.delta`, `reasoning.delta`
  - `tool.started` (from a tool_calls `on_run_step`), `tool.delta`, `tool.completed`
  - `step.finished` (from `on_run_step_closed`), `usage.updated`
  - `run.completed`, `run.failed`, `run.interrupted`, `run.halted`
- Other methods: `resumeInterrupt()`, `resumeSession()`, `clone()`, `fork(entryId)`, `branch(entryId, {summarizeAbandoned})`, `compact({instructions, retainRecentTurns})`, `getLatestCheckpoint()`.
- Internally each run creates a fresh `Run` with `returnContent: true` and `createRunHandlers()`; user handlers are merged in. The calibration ratio and fading tiers are carried between runs.
- `createRunHandlers` (`src/session/handlers.ts`) is the reference implementation of a complete consumer: aggregator, step list, usage totals and a normalized event feed.

---

## 7. Python re-implementation guidance

### 7.1 Reproducing the event contract on LangGraph Python

- **Do not rely on `astream_events` custom-event echoes.** Reproduce the TypeScript *primary* path instead: a `Graph` object holds `handler_registry`, and `dispatch_run_step`, `dispatch_message_delta`, `dispatch_reasoning_delta`, `dispatch_run_step_delta` and `close_run_step` call `await handler.handle(event, data, metadata, graph)` directly. Optionally echo through `adispatch_custom_event` for LangSmith/Langfuse, and dedupe the echo with the same `(event, step_id)` counter.
- **Driving chunks:** in the agent node, call `model.astream(messages, config)` and pass every `AIMessageChunk` to a Python `ChatModelStreamHandler.handle(chunk, metadata, graph)`. This is the TypeScript in-graph path, and the one that supports preemption. `metadata` comes from `config["metadata"]`, which LangGraph fills with `langgraph_node`, `langgraph_step`, `langgraph_checkpoint_ns` and `thread_id`.
- **Outer loop:** `async for ev in graph.astream_events(inputs, config, version="v2")`, then route `on_chat_model_end`, `on_tool_end` and `on_chain_stream` to the registry. Detect interrupts with `"__interrupt__"` in a chain-stream chunk, or `stream_mode=["updates","custom","messages"]` and check the `__interrupt__` key in updates. Alternatively, use `graph.astream(..., stream_mode=["custom","updates"])` and write step events through `get_stream_writer()`. That removes the dual channel entirely; closing message steps then hooks into `on_chat_model_end` or your own call to `closeOpenMessageStep` after `ainvoke`.
- **Tool completion:** the ToolNode calls `graph.handler_registry[ON_RUN_STEP_COMPLETED]`, then `graph.record_step_completion(step_id, tool_call_id, at)`, which closes the step when none of its tool calls are still pending.
- **`on_tool_execute`:** pass an `asyncio.Future` in the request (`resolve` = `fut.set_result`) and await it inside the ToolNode.
- **State to keep on the graph:**
  - `content_data: list[RunStep]`
  - `content_index_map`
  - `step_key_ids: dict[str, list[str]]`
  - `message_ids_by_step_key`, `prelim_message_ids`
  - `tool_call_step_ids`, `pending_tool_calls_by_step`
  - `open_message_step_by_agent`
  - `message_step_has_tool_calls`
  - `next_content_index`
  - `stream_segment`
  - per-agent `current_token_type`, `token_type_switch`, `reasoning_transition_count`, `last_token`
- **Reproduce exactly:** the step-key formula, the ordering rules in 3.5, and the end-of-run sweep in a `finally` block.

### 7.2 Pydantic models (sketch)

```python
class StepType(str, Enum): TOOL_CALLS="tool_calls"; MESSAGE_CREATION="message_creation"
class MessageCreation(BaseModel): message_id: str; content_type: Literal["text","think"]|None=None; phase: Literal["commentary","final_answer"]|None=None
class MessageCreationDetails(BaseModel): type: Literal["message_creation"]; message_creation: MessageCreation
class ToolCall(BaseModel): id: str; name: str; args: dict|str; type: Literal["tool_call"]="tool_call"
class ToolCallsDetails(BaseModel): type: Literal["tool_calls"]; tool_calls: list[ToolCall]=[]
class RunStep(BaseModel):
    id: str; type: StepType; index: int; stepIndex: int|None=None
    stepDetails: MessageCreationDetails|ToolCallsDetails = Field(discriminator="type")
    runId: str|None=None; agentId: str|None=None; groupId: int|None=None
    created_at: int|None=None; status: Literal["in_progress","completed","cancelled","failed"]|None=None
    completed_at: int|None=None; cancelled_at: int|None=None; failed_at: int|None=None
    summary: "SummaryContentBlock|None"=None; usage: dict|None=None
class ToolCallChunk(BaseModel): name: str|None=None; args: str|None=None; id: str|None=None; index: int|None=None; type: str|None="tool_call_chunk"
class ToolCallDelta(BaseModel): type: StepType; tool_calls: list[ToolCallChunk]|None=None; summary: dict|None=None; auth: str|None=None; expires_at: int|None=None
class RunStepDeltaEvent(BaseModel): id: str; delta: ToolCallDelta
class TextPart(BaseModel): type: Literal["text"]; text: str; citations: list|None=None; tool_call_ids: list[str]|None=None; phase: str|None=None
class ThinkPart(BaseModel): type: Literal["think"]; think: str
class MessageDeltaEvent(BaseModel): id: str; delta: dict   # {"content": [TextPart...], "tool_call_ids": [...]}
class ReasoningDeltaEvent(BaseModel): id: str; delta: dict  # {"content": [ThinkPart...]}
class ProcessedToolCall(BaseModel): id: str; name: str; args: str; output: str; progress: float=1; outcome: str|None=None
class ToolCompleteResult(BaseModel): id: str; index: int; type: Literal["tool_call"]="tool_call"; tool_call: ProcessedToolCall; completed_at: int|None=None; eager: bool|None=None
class RunStepCompletedEvent(BaseModel): result: ToolCompleteResult|"SummaryCompletedResult"
class RunStepClosedEvent(BaseModel): id: str; index: int; type: StepType; status: Literal["completed","cancelled","failed"]; closed_at: int; created_at: int|None=None; runId: str|None=None; agentId: str|None=None; groupId: int|None=None; stepIndex: int|None=None
# plus SummaryContentBlock, SummarizeStart/Delta/Complete, SubagentUpdateEvent, ContextUsageEvent, AgentLogEvent, ToolExecuteBatchRequest (resolve/reject excluded from serialization), ToolExecuteResult
```

Serialize with `model_dump(exclude_none=True, by_alias=True)`. The field names must stay camelCase or snake_case exactly as in the TypeScript types (`stepDetails`, `runId`, `agentId`, `groupId`, `stepIndex`, but `created_at`, `closed_at`, `tool_call_ids`), because the LibreChat frontend depends on them.

### 7.3 Suggested module layout

```
librechat_agents/
  common/enums.py        # GraphEvents, Providers, StepTypes, ContentTypes, ToolCallTypes, Constants
  types/{run_step,events,content,tools,hitl,run_config}.py   # pydantic models
  events/registry.py     # HandlerRegistry, compose_event_handlers, ModelEndHandler, ToolEndHandler
  stream/chat_model_stream_handler.py  # chunk→step/delta state machine, reasoning detection, <think> parsing, get_chunk_content
  stream/aggregator.py   # create_content_aggregator (port 1:1, incl. tool index resolution)
  graphs/base.py         # step bookkeeping: dispatch_*, close_run_step, record_step_completion, sweep, step keys
  graphs/standard.py, graphs/multi_agent.py
  tools/tool_node.py     # direct + event-driven (on_tool_execute futures), completion dispatch
  tools/handlers.py      # handle_tool_calls / handle_tool_call_chunks
  summarization/node.py
  run.py                 # Run.create/process_stream/resume/get_interrupt/generate_title, hooks, langfuse
  adapters/openai_chat.py, adapters/responses.py
  session/agent_session.py (optional)
```

Port `createContentAggregator` and the reasoning and tool-call state machine in `ChatModelStreamHandler` line for line. Those are where frontend compatibility is most likely to break.
