# @librechat/agents: LLM & Provider Layer

> Reference chapter for the [@librechat/agents SDK architecture](../README.md). Produced by static reading of
> [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) at `v3.9.3` (`2d5653d`). Paths are relative to
> that repository's root. Line numbers are approximate; check the code when a detail matters.

All paths are relative to the SDK repo root.

---

## 0. Where this layer sits

```
Graph.createCallModel (src/graphs/Graph.ts ~3480-4760)
  ├─ getPreparedToolsForBinding → prepareToolsForPromptCache      (src/llm/promptCacheTools.ts)
  ├─ initializeModel(provider, clientOptions, tools)             (src/llm/init.ts → src/llm/providers.ts → providerRegistry.ts)
  ├─ agentContext.systemRunnable.pipe(model)                     (system prompt + system/summary cache markers, src/agents/AgentContext.ts)
  ├─ wire transforms: legacy content, tool-input bounding, thinking-block repair, tool-less folding,
  │   handoff cues, orphan sanitize, strict-alternation coalescing, tail cache marker
  ├─ prepareProviderRequest(...)                                 (src/llm/prepareProviderRequest.ts)
  ├─ contextPressure.measure → pre-invoke overflow guard        (src/llm/contextPressureMeter.ts)
  ├─ attemptInvoke({request, context: graph})                    (src/llm/invoke.ts)
  │     └─ model.stream(...) → provider subclass _streamResponseChunks → smoother → chunks
  └─ on error: overflow recovery (contextOverflowRecovery.ts + utils/errors.ts)
                else tryFallbackProviders (invoke.ts)
```

The pinned dependencies (from `package.json`) are: `@langchain/core 1.2.8`, `@langchain/openai 1.5.8`, `@langchain/anthropic 1.5.2`, `@langchain/aws ^1.4.2`, `@langchain/google-genai/-vertexai/-gauth/-common 2.2.0`, `@langchain/mistralai ^1.2.0`, `@langchain/deepseek ^1.1.3`, `@langchain/xai ^1.4.3`, `openai ^6.46`, `@anthropic-ai/sdk ^0.115`, `@aws-sdk/client-bedrock-runtime ^3.1075`.

---

## 1. Provider matrix

The `Providers` enum is defined in `src/common/enum.ts:89`. Each provider is registered in `src/llm/providers.ts` with three traits: `family`, `manualToolStream` and `strictAlternation`.

| Enum (wire string) | SDK class (file) | Upstream LangChain base (package) | Family / traits | Key customizations |
|---|---|---|---|---|
| `OPENAI` (`openAI`) | `ChatOpenAI` (`src/llm/openai/index.ts:2807`, lc_name `LibreChatOpenAI`) | `ChatOpenAI` (@langchain/openai). It delegates to `LibreChatOpenAICompletions` (1854) and `LibreChatOpenAIResponses` (2291). | openai | `CustomOpenAIClient` with abortable `fetchWithTimeout` and OpenAI SDK `maxRetries: 0`. Stream smoothing via `_lc_stream_delay`. Scalar-metadata dedup. Explicit prompt-cache breakpoints (`promptCacheExplicit`). `safety_identifier`. GPT‑6 Astra shaping. Encrypted reasoning `include` for gpt‑5.6/Astra. `reasoning`/`reasoning_details`/`provider_specific_fields` delta passthrough. Sequential tool-call "seal" stamping, only for api.openai.com. `intent` arg stripped from strict tools. Responses-API replay positions, annotations, cache-write usage. |
| `AZURE` (`azureOpenAI`) | `AzureChatOpenAI` (2912, `LibreChatAzureOpenAI`), with Azure Completions/Responses delegates (2450/2605) | `AzureChatOpenAI` (@langchain/openai) | openai | Same as OpenAI. `CustomAzureOpenAIClient`. The seal stamp applies only to first-party Azure hosts (`*.openai.azure.com`, `*.cognitiveservices.azure.com`, `*.api.cognitive.microsoft.com`). |
| `DEEPSEEK` | `ChatDeepSeek` (3038) | `ChatDeepSeek` (@langchain/deepseek) | openai | Replays `reasoning_content` on tool-call turns (`includeReasoningContent: true`). Custom non-stream `_generate`. Streams `<think>…</think>` tag parsing into `reasoning_content`, with partial-tag split handling. |
| `XAI` (`xai`) | `ChatXAI` (3559) | `ChatXAI` (@langchain/xai) | openai | Honors a custom `configuration.baseURL`/`clientConfig.baseURL` by resetting the client. Smoothing. `maxRetries: 0`. |
| `MOONSHOT` | `ChatMoonshot` (3544) | extends the SDK `ChatOpenAI` | **generic** | Forces `includeReasoningContent: true` (Kimi thinking + tools). |
| `OPENROUTER` | `ChatOpenRouter` (`src/llm/openrouter/index.ts`) | extends the SDK `ChatOpenAI` | openai | `reasoning` object instead of `reasoning_effort`. `includeReasoningDetails`. Claude upstreams get `reasoning_details` converted to `thinking`/`redacted_thinking` content. Stream-time reasoning_details accumulation, flushed only on the `finish_reason` chunk. `preserveToolCacheControl`. Tool-level `cache_control`. Responses-mode top-level `cache_control`. |
| `ANTHROPIC` | `CustomAnthropic` (`src/llm/anthropic/index.ts:452`) | `ChatAnthropicMessages` (@langchain/anthropic) | anthropic, **manualToolStream** | Vendored message conversion (`utils/message_inputs.ts`). Tool-id normalization. Thinking validation (Opus 4.7). Sampling param suppression. Betas auto-derived from tool types. `output_config`, `inference_geo`, `context_management` (compaction). Assistant-prefill stripping for Claude ≥4.6 with cache re-anchoring. Incremental usage on `message_delta`. Smoothing. |
| `BEDROCK` | `CustomChatBedrockConverse` (`src/llm/bedrock/index.ts:153`) | `ChatBedrockConverse` (@langchain/aws) | bedrock, **manualToolStream**, **strictAlternation** | Owns `ConverseStream` directly. Promotes `contentBlockIndex` to `index` on content blocks so merging works. Tool cache point. Application inference profile ARN swap. `serviceTier`. Guardrails. Explicit tool-use seal at `contentBlockStop`. Vendored `message_inputs.ts` (user-turn coalescing, cachePoint hoisting, reasoning filtering). |
| `GOOGLE` | `CustomChatGoogleGenerativeAI` (`src/llm/google/index.ts`) | `ChatGoogleGenerativeAI` (@langchain/google-genai) | google | Rebuilds the `GenerativeAI` client (baseUrl, apiVersion, customHeaders). `thinkingConfig` injection. `includeServerSideToolInvocations` (functionCalling mode `VALIDATED`). Vendored `utils/common.ts` (thought signatures, dummy signature for gemini‑3, server-side tool parts). Prefill dropping for Flash ≥3.6. Usage including thoughts tokens and `over_200k` buckets. Smoothing. |
| `VERTEXAI` | `ChatVertexAI` (`src/llm/vertexai/index.ts:477`, `LibreChatVertexAI`) | `ChatGoogle` (@langchain/google-gauth) plus `ChatConnection` (@langchain/google-common) | google | `CustomChatConnection.formatData`: `thinkingBudget:-1` becomes dynamic, `thinkingLevel` overrides `thinkingBudget`, `fixThoughtSignatures`, `role:function` rewritten to `user`. `repairStreamUsageMetadata` (thoughts tokens). Seals complete tool calls. Smoothing. |
| `MISTRALAI` / `MISTRAL` | `CustomChatMistralAI` (`src/llm/mistral/index.ts`) | `ChatMistralAI` (@langchain/mistralai) | mistral, **strictAlternation** | Smoothing only. |

What the registry traits drive:
- **`family`** drives provider-agnostic predicates: `isOpenAILike`, `isGoogleLike` and `isAnthropicLike` in `src/utils/llm.ts`; `isThinkingEnabled` in `src/llm/request.ts`; attachment projection; truncation.
- **`manualToolStream`** makes `attemptInvoke` post-normalize the accumulated chunk with `modifyDeltaProperties` (`src/messages/core.ts:265`). That function:
  - maps `*_delta` block types to base types;
  - coerces disallowed types to `text` using `allowedTypesByProvider`;
  - turns an empty `tool_use.input: ''` into `'{}'`;
  - for Bedrock, merges consecutive `reasoning_content`/`text` blocks and keeps the signature (`reduceBlocks`).
- **`strictAlternation`** triggers `coalesceAdjacentUserTurns` (`src/messages/alternation.ts:187`).

`initializeModel` (`src/llm/init.ts`):
- It builds `new Class(clientOptions)` unless an `override` instance is passed.
- For OpenAI-like providers it re-assigns `temperature/topP/frequencyPenalty/presencePenalty/n` from the options, working around LangChain constructor defaults. For upstream Vertex it re-assigns `temperature/topP/topK/...`.
- It then calls `bindTools(tools)` and throws if the model has no `bindTools`.

Helper functions in `src/llm/request.ts`:
- `isThinkingEnabled`: Anthropic `thinking`; Bedrock `additionalModelRequestFields.thinking`; OpenAI-compatible `modelKwargs.thinking.type==='enabled'` (LiteLLM-style Claude).
- `resolveClientOptionsModel`: reads `model` or `modelName`.
- `getMaxOutputTokensKey`: `maxOutputTokens` for Google/Vertex, `maxTokens` for everyone else.

---

## 2. Model invocation path, step by step

### 2.1 Before the call (Graph.createCallModel)

1. **Tools prepared for caching.** `getPreparedToolsForBinding` → `prepareToolsForPromptCache`, covered in §3.
2. **Model built.** `initializeModel({tools, provider, clientOptions})`, then piped behind `agentContext.systemRunnable`. The system runnable is where the system message (and its cache markers) and the summary carrier message are added. The model is wrapped as `RunnableSequence(systemRunnable → RunnableBinding(chatModel))`.
3. **Wire transforms.** Every step goes through `contextPressure.trackProjection(before, after)` so token accounting follows each message's origin.
   - `formatContentStrings` if `useLegacyContent`.
   - `projectToolMessagesForProvider`: bounds tool-call inputs to `calculateMaxToolCallInputChars(maxContextTokens)`.
   - **Bedrock quirk:** if the last two messages are an AI message with string content followed by a ToolMessage, the AI text is trimmed and wrapped as `[{type:'text'}]`, or replaced with `''` when empty. Bedrock rejects trailing whitespace and blank text.
   - `projectServingArtifacts`, applied only when the last message is a ToolMessage:
     - Anthropic-like: `projectAnthropicArtifactContent`, which puts tool artifacts such as images into the tool result;
     - OpenAI-like (except DeepSeek) and Google: `projectArtifactPayload`.
     - It is applied only if the measured payload still fits.
   - `applyProviderMessageTransforms`:
     - `ensureThinkingBlockInMessages` when thinking is enabled (`src/messages/format.ts:4271`). An AI+tool chain after the last human message that lacks a thinking/reasoning block is folded into a `[Previous agent context]` HumanMessage, because Anthropic requires signed thinking before tool_use in thinking mode. Messages produced by the current run (index ≥ `startIndex`) are exempt.
     - `foldToolBlocksForToollessAgent` when no tools are bound, because the provider rejects tool blocks without tool schemas.
     - `appendInstructionlessHandoffCue`.
     - `appendPredecessorHandoffCue`, Anthropic-like only (see §2.4).
   - `compactSyntheticProviderContext`: a binary search (12 iterations) that shrinks synthetic context HumanMessages until the payload fits.
   - `coalesceAdjacentUserTurns` for strict-alternation providers. This runs after compaction and before the cache marker.
   - `sanitizeOrphanToolBlocks` (Anthropic-like) when the pruner didn't run, messages changed, or prompt caching is on.
   - `applyServingTailCache`, which places the single tail cache marker last (§3).
4. **`prepareProviderRequest`** (`src/llm/prepareProviderRequest.ts:528`):
   - **Projection mode:** `openai-responses` if `usesNativeOpenAIResponses(model, provider, config)`, which walks `bound/last` wrappers looking for `_useResponsesApi()` or a class name containing `Responses`. Otherwise `chat-messages`.
   - **`projectMessagesForProviderMode`:**
     - `projectToolStreamContentForProvider` (native vs fallback streamed tool content);
     - `projectAttachmentsForProvider`: Bedrock `document` blocks with raw bytes become standard base64 `file` blocks for other providers. Anthropic keeps only pdf/jpeg/png/gif/webp, OpenAI-like keeps only pdf, and dropped attachments are replaced with the text `[Attachment omitted because its binary format is unsupported by this provider.]`;
     - then provider-specific tool-message projection and cache-marker stripping:

       | Provider | Projection | Cache markers stripped |
       |---|---|---|
       | Responses | `projectOpenAIResponsesToolMessageContent` | Anthropic + Bedrock |
       | OpenRouter | `projectOpenRouterToolMessageContent` + computer-call outputs → text | Bedrock only; Anthropic `cache_control` is kept |
       | OpenAI-like | `projectOpenAIChatToolMessageContent` | both |
       | Anthropic | `projectSingleTextToolOutputsToText` | Bedrock |
       | Bedrock | `projectCacheControlledToolOutputsToText` | Anthropic |
       | generic | structured + single-text → text | both |
   - `annotateMessagesForLLM`: tool-output reference registry annotations.
   - Handoff cues (instructionless for all providers; the predecessor cue added for Anthropic-like or removed otherwise).
   - `coalesceAdjacentUserTurns` if strict alternation.
   - `measure(preparedMessages)`, then the result is frozen and branded with a private symbol. `assertPreparedProviderRequestFor` checks that model identity, provider and projection mode match.
5. **Pre-invoke overflow guard.** If `measurement.fits === false`, a local `ContextOverflowError` (`final_context_overflow`) is thrown into the same catch block as a provider overflow. The measured budget is recorded in a WeakMap.
6. **`ON_CONTEXT_USAGE` event** is awaited. Then the invoke config is built:
   - signal composed with the run-scoped `attemptBreaker`;
   - Langfuse metadata and handler;
   - `agentId`;
   - `STREAM_LIMIT_EPOCH_KEY`.

### 2.2 `attemptInvoke` (src/llm/invoke.ts:691)

This is the single funnel for primary, fallback and summarization calls.

1. **Stamp metadata:**
   - `__invoked_provider`, `provider`, `resolvedProvider`;
   - `__invoked_model` and `model` (via `resolveServingModelId`, which walks wrappers to find `.model`);
   - `lc_stream_limit_attempt`: a per-process monotonic sequence, so each attempt gets its own stream-limit budget.
2. **Stream-limit lease.** `registerActiveStreamLimitGeneration(graph, generationKey)` where `generationKey = checkpoint_ns|node|step|attempt`. It is released in `finally`.
3. **Prepared-subagent begin/finish** for eager tool execution (outside the LLM scope).
4. **`attemptInvokeBody`:**
   - Optionally installs a `handleChatModelStart` callback that (a) captures the real LLM `runId`, needed to close a sealed run, and (b) runs the provider message-projection invariant (`off|warn|assert`).
   - **Streaming path** (`model.stream` exists). It uses one of three consumer loops:
     1. **`onChunk` supplied** (summarization, public consumers). Per chunk:
        - `throwIfBreakerTripped`;
        - `enforceStreamLimitsForWireChunk`;
        - `await onChunk(chunk, metadata)`;
        - accumulate with `concat`.
     2. **No registered default stream handler** (local handler). Per chunk:
        - a new `ChatModelStreamHandler().handle(CHAT_MODEL_STREAM, {chunk})`;
        - accumulate;
        - poll preemption. This is the only loop that can **seal**.
     3. **Registered handler.** The graph's handler receives the wire chunks through `streamEvents` separately, so this loop only:
        - charges stream limits synchronously (producer side);
        - re-dispatches OpenRouter-transformed chunks with `STREAM_LIMIT_REDISPATCH_KEY`;
        - accumulates.
   - **Accumulation:** `@langchain/core/utils/stream.concat` (AIMessageChunk merging by `index`).
   - **OpenRouter quirk:** the final chunk that carries `reasoning_details` also replays the full cumulative content. `removeOpenRouterFinalReasoningReplayContent` slices off the already-seen prefix, and `getStreamHandlingChunk` strips `reasoning_details` from the dispatched copy.
   - After the stream:
     - if `manualToolStream`, run `modifyDeltaProperties`;
     - handle the preemption outcome (see §4.2);
     - filter `tool_calls` to those with a non-empty `name`;
     - `assertNotTruncatedToolCall(finalChunk, provider)` (`src/llm/truncation.ts`). This throws `OutputTruncationError` when the stop reason is max-tokens/length and any tool_call, tool_call_chunk or invalid tool call exists. It checks `stopReason`, `stop_reason`, `finish_reason`, `finishReason`, `messageStop.stopReason`, `incomplete_details.reason` and `additional_kwargs.stop_reason`. It is skipped for Google/Vertex because they emit tool calls atomically.
   - **Non-streaming path:** `model.invoke`, then the same filter and truncation check.
   - Returns `{messages:[finalChunk]}`.

### 2.3 Retry and fallback

- **No SDK-level retry loop exists.**
  - Transient retries come from LangChain's `AsyncCaller`: `this.caller.call`, `callWithOptions`, and `completionWithRetry`/`createStreamWithRetry` in each provider base. It uses `clientOptions.maxRetries` (LangChain default 6) with exponential backoff.
  - The OpenAI-family classes construct `CustomOpenAIClient` with `maxRetries: 0` so retries don't stack with LangChain's.
  - `llmConfig.ts` only has a commented-out `maxRetries`.
- **Overflow recovery comes before fallbacks** (§4.3). A tripped stream limit or `PreparedSubagentError` is rethrown immediately (`attemptBreaker.abort(err)`) and never enters recovery or fallback.
- **`tryFallbackProviders`** (invoke.ts:1536):
  - For each `t.FallbackConfig {provider, clientOptions, maxContextTokens}`:
    - `initializeModel` with the same tools, but **without** the system runnable (the system prompt is not re-piped);
    - stamps `__invoked_model`;
    - optionally prepares a request. Graph passes a full re-projection pipeline sized to the fallback's window: artifacts, transforms, orphan sanitize, tail cache, measure, synthetic compaction, overflow error;
    - rechecks the breaker;
    - calls `attemptInvoke`.
  - On failure it records `lastError`. Overflow errors (checked against the fallback's own `maxContextTokens` and the estimated prompt) get a `FallbackErrorContext` attached in a WeakMap.
  - Final throw preference: first fallback overflow, then primary overflow, then last error. The full candidate list is retrievable via `getFallbackOverflowCandidates`, so Graph can plan recovery against the fallback's window, translating the budget between calibration ratios.

### 2.4 Handoff cues and alternation

**`INSTRUCTIONLESS_HANDOFF_CUE`** (`src/messages/handoffCue.ts`):
- Text: "Continue as the receiving agent using the preceding user request and context."
- It is appended as a meta HumanMessage (`isMeta`, `source:'routing'`) when the payload tail is the exact AI message the recipient was entered with. That message's id is recorded in `config.metadata.__handoff_cue_message_id` by `withInstructionlessHandoffCue`.
- It applies on all transports.

**`PREDECESSOR_HANDOFF_CUE`**:
- Appended for Anthropic-like providers only, when the payload ends with an AI message **this run produced** (checked with `graph.isRunProducedMessage`).
- Reason: Claude treats a trailing assistant turn as a prefill and continues it, or returns empty.
- `removePredecessorHandoffCue` strips it for non-Claude fallbacks.

**`coalesceAdjacentUserTurns`**:
- Merges runs of adjacent non-tool-result human messages into one. It joins strings with `\n\n` or concatenates blocks, and preserves provenance.
- Tool-result HumanMessages that pair with the preceding AI message's tool calls are left intact.
- Applies to Bedrock and Mistral only.

### 2.5 Usage accounting

The target is normalized LangChain `usage_metadata {input_tokens, output_tokens, total_tokens, input_token_details{cache_read, cache_creation}, output_token_details{reasoning}}`.

- **Anthropic** (`getAnthropicUsageMetadata`, `utils/message_outputs.ts`):
  - `input_tokens = input + cache_creation + cache_read`.
  - In streaming, `message_start` seeds the counts. On `message_delta`, `withIncrementalMessageDeltaUsage` converts the cumulative counts to increments, so `concat` sums correctly and nothing is double-counted.
- **OpenAI Chat:** `prompt_tokens_details.cached_tokens` maps to `cache_read`, `cache_write_tokens` maps to `cache_creation`, and `reasoning_tokens` is carried through. The usage chunk is yielded last.
- **OpenAI Responses:** `input_tokens_details.cache_write_tokens` is smuggled via `response.metadata.__librechat_cache_write_tokens` and then moved into `usage_metadata` (`attachCacheWriteUsage`).
- **Bedrock** (`handleConverseStreamMetadata`): `cacheReadInputTokens`/`cacheWriteInputTokens` become the details. Here `input_tokens` is the fresh delta only; the cache buckets are separate.
- **Google GenAI:** `output = candidates + thoughts`; `cachedContentTokenCount` becomes `cache_read`; `gemini-3-pro-preview` gets `over_200k` and `cache_read_over_200k`. Usage is emitted once, after the stream.
- **Vertex:** `repairStreamUsageMetadata` replaces the trailing chunk's buggy `output_tokens` (which omits thoughts) with the last `generationInfo.usage_metadata`.
- **Sealed or restarted turns with no provider usage:** `synthesizeSealedUsage` estimates input as the host token counter over the prompt plus instruction overhead, and output as the counter over the partial chunk. It marks `response_metadata.estimated_usage: true` so calibration ignores it.
- **`endSealedModelRun`** manually fires `handleLLMEnd`. LangChain fires no end/error callback when a consumer breaks out of `for await`. It rebuilds the callback manager from the config plus every `callbacks` found by walking the model wrapper chain (`bound`/`last`/`steps`). If that fails it dispatches a custom `CHAT_MODEL_END` event instead.

### 2.6 Reasoning / thinking extraction

Reasoning arrives in these per-provider shapes:

| Source | Shape |
|---|---|
| Anthropic | content blocks `thinking` (with `signature`, from `thinking_delta`/`signature_delta`) and `redacted_thinking` |
| Bedrock | `reasoning_content` blocks `{reasoningText:{text, signature}}` |
| Google GenAI | `additional_kwargs.reasoning` (joined `thought:true` parts); also function-call thought signatures in `additional_kwargs.__gemini_function_call_thought_signatures__` (id → signature) |
| Vertex | `additional_kwargs.signatures[]`, plus reasoning content blocks |
| OpenAI Chat-compatible | `additional_kwargs.reasoning_content` (DeepSeek, Moonshot, vLLM, and `delta.reasoning` from some gateways); `reasoning_details`; `provider_specific_fields` |
| OpenAI Responses | `additional_kwargs.reasoning` `{id, summary[], encrypted_content}`; replay positions |
| OpenRouter | `reasoning` (text, per chunk) and `reasoning_details` (final chunk only) |
| DeepSeek fallback | `<think>` tags in content, parsed into `reasoning_content` |

- `isReasoningContentBlock` (`src/messages/reasoningTypes.ts`) is the canonical set: `think, thinking, thinking_delta, redacted_thinking, reasoning, reasoning-delta, reasoning_content`.
- The UI-facing extraction lives in `src/stream.ts` (`getChunkContent`, around line 1400; reasoning detection around line 2137):
  - OpenAI/Azure prefer `reasoning.summary[0].text`;
  - OpenRouter prefers content over reasoning;
  - the others use the keyed `reasoning_content`/`reasoning`.

---

## 3. Prompt caching strategy per provider

The default TTL is `'1h'` (`DEFAULT_PROMPT_CACHE_TTL`, `src/messages/cache.ts`). `'5m'` omits the `ttl` field entirely, keeping the payload byte-identical to the legacy marker. Caching is opt-in through `clientOptions.promptCache: true` and `promptCacheTtl`.

**Core principle: a single tail breakpoint.** Every call strips all stale markers, then places one marker on the last cacheable block of the final non-synthetic message. The rationale is in `docs/prompt-cache-benchmark.md`: in agent tool loops the legacy "last 2 user messages" placement leaves every appended assistant/tool turn uncached. Measured effective cost is 26–41% lower with the tail strategy. The legacy functions `addCacheControl`/`addBedrockCacheControl` are still exported.

### 3.1 Anthropic (direct)

Up to 4 breakpoints are budgeted: tools, system, stable-prefix and tail.

- **Tools** (`src/messages/anthropicToolCache.ts`, `partitionAndMarkAnthropicToolCache`):
  - Stable-partition the tools into `[static…, deferred…]`, where "deferred" means `defer_loading` in the tool definitions.
  - Stamp `cache_control` on the **last static tool**, so tools discovered later don't bust the prefix.
  - LangChain tools carry the marker in `tool.extras.cache_control`, which the adapter promotes. Provider-shaped tools carry it directly on the block: built-ins with type prefixes `text_editor_`, `computer_`, `bash_`, `web_search_`, `web_fetch_`, `code_execution_`, `memory_`, `tool_search_`, `mcp_toolset`, or raw objects with `input_schema`.
  - Stale markers are stripped from every other tool, because a longer TTL must precede a shorter one. Wrappers preserve the prototype and never mutate the original tool.
- **System** (`AgentContext.buildSystemMessage`):
  - The system message is `[{text: stableInstructions, cache_control}, {text: dynamicInstructions}]`.
  - When both stable and dynamic instructions exist and caching is on, the dynamic instructions are **moved out of the system message** into a HumanMessage "dynamic tail". So is the summary.
  - `buildBodyWithPromptCacheDynamicTail` then:
    - marks the stable prefix up to the latest human message with `addCacheControlToStablePrefixMessages(…, 1)` (last assistant message preferred, otherwise any conversation message; the opening message is never an anchor);
    - inserts the dynamic tail;
    - puts a tail marker on the trailing messages.
  - With no dynamic tail, `addTailCacheControl(body)` is used.
  - The summary carrier gets its own `cache_control`.
- **Tail** (`addTailCacheControl`):
  - Walks backwards and strips `cache_control` and `cachePoint` from all messages.
  - Anchors on the last block that is not reasoning, not `input_json_delta`, not empty text, and not a cachePoint.
  - Skips synthetic meta messages (`additional_kwargs.isMeta` or `source==='skill'`) and computer-call outputs.
  - When the system runnable owns the system prompt, Graph skips its own tail marker; AgentContext places it instead.
- **Converter-level fixes:**
  - `hoistToolResultCacheControl` lifts markers from inner tool-result content onto the `tool_result` block.
  - `stripUnsupportedAssistantPrefill` removes trailing assistant turns for Claude 4.6+ (`modelDisallowsAssistantPrefill`) and **re-anchors** the tail marker if it was on the stripped prefill, preserving the `1h` TTL.
  - A top-level `cache_control` call option (auto-advancing, SDK 1.5.x) is passed through additively.

### 3.2 Bedrock Converse

- **System:** `[{text: stable}, {cachePoint:{type:'default', ttl?}}, {text: dynamic}]`. Existing system cachePoints are normalized to the resolved TTL (`sanitizeBedrockSystemMessage`).
- **Tail:** `addBedrockTailCacheControl` inserts a **separate** `{cachePoint}` block right after the last non-empty text block of the last non-system, non-synthetic message. Bedrock always marks in Graph, even when a system runnable exists.
- **Tool results:** a cachePoint nested inside `toolResult.content` is silently ignored by Bedrock, so `convertToolMessageToConverseMessage` hoists it out as a sibling after the `toolResult`.
- **Tools** (`src/llm/bedrock/toolCache.ts`):
  - Tools are converted to `toolSpec`, then partitioned static/deferred.
  - The last static tool gets the private marker `__lc_bedrock_cache_point_after`. At `invocationParams` time, `insertBedrockToolCachePoint` replaces the marker with a real `{cachePoint}` entry in `toolConfig.tools`.
  - If all tools are deferred, the disabled marker `__lc_bedrock_skip_tool_cache` is set instead.
- **Model gating:**
  - The tool cachePoint is **Claude-only** (`supportsBedrockToolCache` = `/claude|anthropic/i`); Nova rejects `cachePoint` in tools.
  - Message and system cachePoints work on Nova.
  - The `1h` TTL is clamped to `5m` for non-Claude models (`resolveBedrockPromptCacheTtl`).
- **Vendored upstream `applyCachePointsToConversePayload`** (`cachePoints.ts`) handles the `options.cache_control` path: system, last message and tools, with Nova exclusions for tool blocks and TTL.

### 3.3 OpenRouter (OpenAI-wire, Anthropic-style markers)

- The system message uses the Anthropic `cache_control` text-block format.
- The tail marker uses `addTailCacheControl` in Anthropic format. `stripAnthropicCacheControl` is **not** applied for OpenRouter; only Bedrock markers are stripped.
- `preserveToolCacheControl` keeps the canonical cache-decorated tool text block when serializing tool messages.
- Tools: `partitionAndMarkOpenRouterToolCache` converts to OpenAI tool format and puts `cache_control` on the last static tool. `defer_loading` is preserved.
- Responses mode puts `cache_control` at the request top level and strips it from tools.
- TTL is forwarded; Claude upstreams downgrade if needed.

### 3.4 OpenAI / Azure (first-party)

OpenAI's automatic caching needs no markers. The SDK additionally supports **explicit caching** via `promptCacheExplicit` (the GPT‑5.6 managed-request passthrough):
- The request gets `prompt_cache_options: {mode:'explicit', ttl:'30m'}`.
- **Chat:** `addChatCacheBreakpoints` puts `prompt_cache_breakpoint:{mode:'explicit'}` on:
  - the last cacheable part of the **last system/developer message**;
  - the last cacheable message **before the latest user message**.
- **Responses:** `addResponseCacheBreakpoints` uses the same selection, but only on input roles (system/developer/user) and only on `input_text`/`input_image`/`input_file` parts. Output parts are rejected with a 400.
- All Anthropic and Bedrock markers are stripped for OpenAI-like providers.
- `cache_write_tokens` is surfaced as `cache_creation` in usage.

### 3.5 Google, Mistral, DeepSeek, xAI

There is no explicit caching. Markers are stripped by the generic projection. Implicit cache reads are reported (`cachedContentTokenCount` for Google, `cached_tokens` for OpenAI-compatible providers).

---

## 4. Stream limits, preemption, context overflow

### 4.1 Stream limits (src/llm/streamLimits.ts)

These are circuit breakers for runaway generations. The motivating case: one 149,923-char SQL argument streamed for 26 minutes.

**Configuration** (`resolveStreamLimits`):

| Setting | Default | Meaning |
|---|---|---|
| `maxToolCallArgBytes` | 65,536 | UTF-8 bytes per single streamed tool call |
| `maxToolCallArgBytesByTool[name]` | — | Per-tool override |
| `maxDeltaEventsPerTurn` | 0 (off, opt-in) | Stream chunk events per generation |

`0`, negative values or `Infinity` disable a limit; `NaN` falls back to the default.

**Accounting keys:**
- Generation key: `langgraph_checkpoint_ns|langgraph_node|langgraph_step|lc_stream_limit_attempt`. The step key is deliberately *not* used, because it forks on reasoning transitions.
- Per-call tally key: chunk `index`, falling back to `id` (empty string counts as absent), then to position in the batch.

**Tool-arg byte counting:**
- Handles split UTF-16 surrogate pairs across chunks.
- Seal-aware:
  - a `kind:'single'` seal (the OpenAI Responses `…arguments.done` restatement, or Bedrock's empty-args stop chunk) **replaces** the tally rather than adding to it, then releases the call;
  - a `kind:'all'` seal (Google) judges each call standalone.
- `enforceCompleteToolCallArgLimit` covers complete parsed calls that arrive with no chunks.

**Charge dedup:**
- LangChain delivers the same chunk object to both the producer loop and the `streamEvents` consumer. `claimStreamLimitCharge` keeps a signed credit balance per chunk object per generation, so the chunk is charged once.
- `linkStreamLimitCanonical` links a Bedrock enriched chunk to its callback copy so the two count as one emission.

**Lifecycle:**
- Leases per attempt; `sweepStaleStreamLimitEntries` sweeps by epoch on reset.
- A trip throws `StreamLimitExceededError {kind, limit, observed, toolName}`. This aborts the run-scoped breaker so parallel agent nodes and subagents stop too. The error is never recovered or retried via fallback.
- `throwIfBreakerTripped` is checked on every yielded chunk, because adapters may ignore abort.

**Stream smoothing** (`src/llm/stream/smoother.ts`, `chunkAdapters.ts`):
- This is pacing, not limiting. `_lc_stream_delay` defaults to 25ms; `0` disables it.
- Pieces are backlog-proportional, targeting about 250ms lag, cut at word boundaries (64-char lookahead).
- Queue caps: 256 chunks and 8192 chars.
- Tool-call/usage/metadata chunks pass through with zero delay. Chunks mixing text with reasoning or tool deltas are paced whole, never split.
- The producer is abandoned after 1s if it ignores the close signal.

**`dropRepeatedScalarMetadata`** (`src/llm/openai/streamMetadata.ts`): keeps only the first occurrence of `model_name`/`finish_reason`/`service_tier`/`system_fingerprint` per completion index. Without it, `_mergeDicts` would concatenate repeats into values like `"stopstop"`.

### 4.2 Preemption hooks in the LLM layer (src/llm/preempt.ts + invoke.ts)

The host sets a level-triggered `context.shouldPreemptStream()`, which fires for steering or interrupts. The action is chosen per chunk by `resolvePreemptAction`:
- **`seal`** if the accumulated turn has visible text (not reasoning-only) and no open tool call. This includes unanswered Google server tool calls (`toolCall` count > `toolResponse` count) and Anthropic server tool uses without results. OpenAI Responses turns with built-in tool output are also treated as not disposable.
  - Sealing keeps the partial turn: `response_metadata.preempted=true`, the stream loop breaks, and `endSealedModelRun` closes the callbacks and synthesizes usage. Seals are claimed from a shared budget (`claimPreemptSeal`, `DEFAULT_MAX_SEALS`).
- **`restart`** only when the turn holds nothing but reasoning and the request is older than `restartGraceMs`.
  - An `AbortController` composed into the stream signal aborts the provider.
  - The turn is discarded: `preemptDiscarded`, `cancelOpenMessageStep`, `notePreemptRestart(lane)`.
  - It returns `{messages: []}`, so the injected user turn lands directly after the previous user turn. That is safe everywhere; strict providers get coalescing.
  - A self-rescheduling unref'd timer re-evaluates during silent reasoning windows.
  - The "restart route" is one of `aborted`, `broke` or `exhausted`, and it decides how the LLM run is closed.
- **`none`** otherwise.

### 4.3 Context overflow detection & recovery

**Detection** (`src/utils/errors.ts`, `getContextOverflowInfo(error, {provider, estimatedPromptTokens, maxContextTokens})`):
1. `collectErrorText` flattens the error through `message/code/type/status/reason` and nested `error/cause/body/data/response` (depth ≤ 4), then strips URLs.
2. Negative filters run first:
   - `NON_RECOVERABLE_RE`: rate limit reached, rpm, quota, billing, auth, forbidden;
   - `OUTPUT_LIMIT_RE`: max_tokens too large;
   - `AMBIGUOUS_CONTEXT_OR_OUTPUT_RE`.
3. LangChain `ContextOverflowError` is recognized by instance, by `name`, or by `lc_error_code: CONTEXT_OVERFLOW`.
4. Ordered regex table. Each entry sets `kind`, `limitTokens`, `requestedTokens`, and `promptTokens` only when the figure is prompt-only:
   - Anthropic/Bedrock-Claude: `prompt is too long: N tokens > M maximum`
   - OpenAI: `maximum context length is M tokens. However, your messages resulted in N` (prompt-only)
   - OpenRouter/DeepSeek: `…you requested (about) N`. This total includes the completion; `PROMPT_ONLY_BREAKDOWN_RE` recovers `(X of text input` / `(X in the messages`.
   - xAI: `maximum prompt length is M but the request contains N tokens`
   - Mistral: `Prompt contains N tokens … too large for model with M maximum context length`
   - Gemini: `input token count exceeds the maximum number of tokens allowed (M)`
   - OpenAI **429** `Request too large … Limit L, Requested R` becomes `request_too_large`, but only when `Requested > Limit`; otherwise it is genuine throttling.
   - Bedrock Llama/Nova/Sonnet wording (Nova and Llama arrive as an HTTP 200 mid-stream `Error`).
   - Generic `context_length_exceeded` and similar.
   - **Vertex** bare `Google request failed with status code 400` (with no `:`) counts only with *context pressure*, i.e. `estimatedPromptTokens / maxContextTokens ≥ 0.8`.

**Recovery policy** (`src/llm/contextOverflowRecovery.ts`, `planContextOverflowRecovery`):
- Maximum 2 recoveries per agent per run (`DEFAULT_MAX_OVERFLOW_RECOVERIES`).
- Target budget:
  - if the provider named a limit: `(limitTokens − reservedCompletion) × 0.95`, where reserved = `requested − prompt` when both are known, else the configured `maxTokens`. If this is ≤ 0, recovery is declined;
  - else `estimatedPrompt × 0.7`;
  - else `maxContextTokens × 0.7`.
- If the target is ≥ the current max, a blind shrink is used instead.
- Floor: `instructionTokens + 2000`, else 4000, or 1 if summarization is possible. Recovery is declined below the floor or if the budget does not shrink.
- `observedCalibrationRatio = (providerPrompt − instr) / (estimated − instr) × currentRatio` is returned separately, to avoid correcting twice.
- `translateRecoveryBudget` converts between calibration spaces for fallback-sourced errors.

In Graph:
- A recoverable overflow triggers `beginOverflowRecovery`, which returns a `summarizationRequest` with reason `overflow` instead of throwing.
- Staging: first re-prune under the smaller budget (tool-output truncation and masking); summarize only on a second overflow, and only if enabled.
- Recovery is skipped if nothing can shrink: no token counter and no summarization, or `overflowRecoveryStalled`.
- Fallbacks run only when recovery is not applicable.

**Context pressure meter** (`src/llm/contextPressureMeter.ts`):
- Measures the exact provider-projected payload against `contextBudget − effectiveInstructionTokens`, including `REPLY_PRIMER_TOKENS`, scaled by the calibration ratio.
- Tokenizes each message object once (WeakMap). The provider-grounded baseline from the pruner is attributed per origin message through `trackProjection` and `trackClone`, so one transform's shrink cannot mask another's growth.
- Produces `ProviderPayloadMeasurement {fits, projectedMessageTokens, availableMessageTokens, toolMessageTokens…}`.

---

## 5. Per-provider message conversion quirks (what a Python port will hit)

### Anthropic (src/llm/anthropic/utils/message_inputs.ts, vendored from @langchain/anthropic)

1. **Tool-use ids** must match `^[a-zA-Z0-9_-]+$` and be ≤ 64 chars. `normalizeAnthropicToolCallId` sanitizes and appends the first 10 hex chars of the SHA-256 of the *original* id. It is deterministic, so `tool_use.id` and `tool_result.tool_use_id` stay paired without a map. Needed for OpenAI Responses `call_…`/`fc_…` ids and Google ids.
2. **Consecutive ToolMessages** merge into one user message of `tool_result` blocks (`_ensureMessageContents` plus `mergeMessages`). A ToolMessage whose content is already a single `tool_result` envelope is unwrapped. `is_error` is propagated.
3. **`tool_use.input` must be an object.** `coerceAnthropicToolUseInput` parses a JSON string when complete, else uses `{}`. `null` inputs appear in history after compaction.
4. **Empty assistant content** becomes the text placeholder `'_'`.
5. **Thinking blocks:**
   - unsigned `thinking` on an assistant turn (for example Google output) is dropped;
   - signed blocks with missing text (Opus 4.7 `display:'omitted'`) send `thinking: ''`;
   - standard `reasoning` blocks with a signature are converted back to `thinking`;
   - foreign reasoning (`reasoning_content`, `reasoning`, `think`) is dropped;
   - Google `toolCall`/`toolResponse` server parts are dropped.
6. **Tool calls present only in `tool_calls`** and not in content (for example a Bedrock thinking turn) must be materialized as `tool_use` blocks. Google `functionCall` parts are converted to `tool_use`, deduplicated by id.
7. **Server tools:** blocks with id prefix `srvtoolu_` are rebuilt as clean `server_tool_use` and `web_search_tool_result` blocks. `input_json_delta` blocks are dropped, and the assembled input is restored onto `tool_use`.
8. **Thinking mode requires** each tool_use turn in the current tail to have a thinking block → `ensureThinkingBlockInMessages` (see §2.1).
9. **Assistant prefill:** Claude ≥ 4.6 rejects a trailing assistant message. Strip it and re-anchor the cache marker.
10. **Request validation:**
    - Opus 4.7 rejects `thinking.type:'enabled'` (use `adaptive`), `budget_tokens`, `topK`, and non-default `topP`/`temperature`;
    - with thinking on, no `top_k`/`top_p` and temperature must be 1;
    - `temperature/topP/topK === -1` means unset;
    - `thinking` is omitted unless explicitly set.
11. **Betas are auto-added** from tool types: `tool_search_tool_*` → `advanced-tool-use-2025-11-20`, `memory_20250818` → `context-management-2025-06-27`, `web_fetch`, `code_execution`, `computer_*`, `mcp_toolset`. Compaction (`compact_20260112`) and task budgets add their own betas.
12. **Streaming** content is coerced to a string only when there are no tools, documents, thinking or compaction in the request.

### Bedrock (src/llm/bedrock/utils/message_inputs.ts, vendored)

1. **All consecutive user-role messages** (ToolMessage and HumanMessage both become `user`) are combined, with block order preserved (toolResult blocks, then text). An empty user message gets the `'_'` placeholder, applied only after the group is complete.
2. **cachePoint** inside a tool result is hoisted to a sibling position (see §3.2).
3. **Reasoning:** consecutive reasoning blocks are concatenated. `reasoningText` with empty text is dropped, because signature-only blocks fail. Foreign reasoning and Google server parts are dropped. An assistant turn left empty by filtering gets `'_'`.
4. **`toolUse.input`** gets the same object coercion as Anthropic.
5. **v1 standard-content messages** from other providers (`response_metadata.output_version==='v1'`) take a separate conversion path.
6. **Streaming:** the Converse SDK omits `index`, so `_mergeLists` appends instead of merging. The SDK:
   - reads `contentBlockIndex`;
   - promotes it to `index` on each block;
   - promotes text deltas to array form once more than one block index has been seen;
   - strips it from `response_metadata`.
7. **Tool-use seal** at `contentBlockStop`, only when no guardrails are configured, because guardrails can intervene at `messageStop`.
8. **Application inference profile:** `this.model` is temporarily swapped to the ARN, and the configured model id is kept for cache gating.
9. **Documents:** `document` blocks with raw bytes are projected to base64 `file` blocks for other providers (`prepareProviderRequest`).
10. **Blank text:** Bedrock rejects blank text blocks and trailing whitespace in the last AI text before a tool result, hence the trim in Graph.

### OpenAI / compatible (src/llm/openai/utils/index.ts)

1. The `system` role becomes `developer` for reasoning models (`/\b(o\d|gpt-[5-9])\b/`).
2. Tool-call arguments are bounded (`projectToolCallInputs`). Tool results are serialized with a bounded serializer (`HARD_MAX_TOOL_RESULT_CHARS`).
3. **`reasoning_content` replay** (DeepSeek/Moonshot): included on assistant turns with tool calls, and on every turn after the first tool interaction. DeepSeek errors if it is missing in thinking mode with tools.
4. **`reasoning_details`** (OpenRouter/Gemini): sent as a field for non-Claude models. For Claude via OpenRouter it is converted to `thinking`/`redacted_thinking` content blocks.
5. **Strict tools:** the optional `intent` property is removed from `strict:true` tool schemas, because strict mode requires every property to be in `required`.
6. **Sequential tool-call seal:** only official OpenAI and Azure hosts are stamped. Kimi/Moonshot revise earlier indices.
7. **Responses API:**
   - encrypted reasoning via `include: ['reasoning.encrypted_content']` for gpt-5.6/Astra;
   - replay-output items (`shell_call_output`, `apply_patch_call_output`, …) are kept;
   - text block indices are remapped;
   - `response.failed`/`error` events are turned into thrown `APIError`s.
8. **GPT‑6 Astra** (only with `firstPartyEndpoint: true`): drop `temperature`, `top_p`, `logprobs`, `top_logprobs` and the `message.output_text.logprobs` include; map effort `none`/`minimal` to `low`.

### Google GenAI / Vertex

1. **Thought signatures:**
   - gemini‑3 requires `thoughtSignature` on replayed `functionCall` parts. GenAI stores them in `__gemini_function_call_thought_signatures__` keyed by tool-call id and falls back to a hard-coded `DUMMY_SIGNATURE` for gemini-3.
   - Vertex (`fixThoughtSignatures`) matches AI messages to `model` contents by position and re-attaches non-empty signatures that google-common failed to apply. AI messages without signatures still occupy their position so later messages line up.
2. **ToolMessages need a function name.** It is inferred from previous AI tool_calls, or an error is thrown. The response is wrapped as `{result}` or `{error:{details}}` (it must be an object), and the `id` is passed when non-empty.
3. **Parsed tool calls** are not also sent as `functionCall` content parts; content mirrors are deduplicated by id or name.
4. The system message becomes the `systemInstruction`. The **`function` role is rewritten to `user`**; gemini-3.1+ rejects `function`.
5. **Prefill:** trailing `model` turns are dropped for Flash ≥ 3.6 and `gemini-3.5-flash-lite`.
6. **Schema sanitation:** `additionalProperties`, `$schema` and `strict` are removed recursively. A tool with empty `properties` is sent without `parameters`. A missing description defaults to `'A function available to call.'`.
7. **Thinking config:**
   - `thinkingBudget: -1` means dynamic: delete the budget and force `includeThoughts`;
   - `thinkingLevel` and `thinkingBudget` are mutually exclusive;
   - GenAI patches `client.generationConfig.thinkingConfig` directly.
8. **Server-side tools** (Search, URL context) arrive as `toolCall`/`toolResponse` content blocks, not as `tool_calls`. They need `includeServerSideToolInvocations` and functionCalling mode `VALIDATED`.
9. **Tool calls are atomic:** chunks are stamped with the seal `kind:'all'`, and the truncation guard skips Google.

### Mistral

Mistral needs strict alternation: coalesce adjacent user turns. It uses the upstream LangChain converter otherwise.

---

## 6. Custom provider registration API

The entry point is `src/provider-registration.ts`, exported as `@librechat/agents/provider-registration`. The implementation is in `src/llm/providerRegistry.ts`.

```ts
registerProvider({
  provider: 'myllm',             // non-empty, untrimmed-whitespace rejected
  model: MyChatModel,            // constructible BaseChatModel subclass
  family?: 'openai'|'anthropic'|'bedrock'|'google'|'mistral'|'generic',  // default 'generic'
  manualToolStream?: boolean,    // default false
  strictAlternation?: boolean,   // default false
}): () => void                   // disposer; only removes if still owned (Symbol owner token)
```

- **Storage.** The registry lives on `globalThis[Symbol.for('@librechat/agents:providerRegistry:v1')]`, so duplicate package copies (CJS/ESM) share it. Built-ins live in a separate module-local map and cannot be overridden: a duplicate registration throws `LLM provider already registered`.
- **Lazy built-ins.** Built-ins register via `registerBuiltInProviderLoader` with a `loadModel()` thunk that uses `requireInternalModule` (`src/lazyRequire.ts`). The provider SDK is imported on first use, then the class is validated and memoized. `src/llm/providers.eager.ts` registers source-mode modules for running TypeScript directly.
- **Typing.** Declaration-merge `CustomProviderOptionsMap` to type the options.
- **What `family` enables for a custom provider:**
  - thinking detection;
  - message projection branch (Anthropic/Bedrock/OpenAI);
  - attachment filtering;
  - Anthropic-like handoff cue;
  - `maxOutputTokens` vs `maxTokens` key;
  - truncation guard skip for google;
  - `manualToolStream` normalization, mapped to the Anthropic/Bedrock rules.
- **Lookups:** `getChatModelClass(provider)`, `getProviderFamily`, `providerUsesManualToolStream`, `providerRequiresStrictAlternation`.

---

## 7. Python re-implementation guidance

### 7.1 Package coverage

| Provider | Python base | Coverage | Must re-write |
|---|---|---|---|
| OpenAI / Azure | `langchain-openai` (`ChatOpenAI`, `AzureChatOpenAI`, `use_responses_api=True`) | chat, Responses, reasoning summary, `stream_usage` | explicit cache breakpoints (`prompt_cache_options`, `prompt_cache_breakpoint`); `safety_identifier`; Astra shaping; encrypted reasoning include; `reasoning`/`reasoning_details`/`provider_specific_fields` delta capture (subclass `_convert_chunk_to_generation_chunk`); `cache_write_tokens` into `cache_creation`; scalar metadata dedup; first-party seal stamping; strict-tool intent stripping; system→developer (partly native) |
| DeepSeek | `langchain-deepseek` | `reasoning_content` in output | `reasoning_content` **replay on input** for tool turns; `<think>` tag stream parser |
| xAI | `langchain-xai` | basic | custom base_url; overflow regex |
| Moonshot | `ChatOpenAI` with base_url | — | reasoning_content capture and replay (same as DeepSeek) |
| OpenRouter | `ChatOpenAI` with base_url, or `langchain-openrouter` if available | — | `reasoning` param object; `reasoning_details` accumulation and final-chunk flush; Claude reasoning_details → thinking blocks; cumulative-replay de-duplication; cache_control on system, tail and tools |
| Anthropic | `langchain-anthropic` `ChatAnthropic` | thinking, signatures, `cache_control` passthrough, betas | tool-id normalization; input coercion; foreign-reasoning dropping; tool_result hoisting and unwrapping; prefill stripping with cache re-anchor; Opus 4.7 validation; auto betas; incremental `message_delta` usage (verify Python aggregation); `'_'` placeholder |
| Bedrock | `langchain-aws` `ChatBedrockConverse` | Converse, reasoning, some cachePoint support | user-turn coalescing; cachePoint hoisting out of toolResult; tool cachePoint with Claude gating and Nova TTL clamp; inference-profile ARN; service tier; content-block-index merge fix (check whether Python `ChatBedrockConverse` preserves `index`); blank-text/placeholder handling; reasoning filtering |
| Google GenAI | `langchain-google-genai` | thought signatures (newer versions), thinking_budget, include_thoughts | dummy signature fallback; prefill dropping; server-side tool parts; schema sanitation (`additionalProperties`/`$schema`/`strict`); usage including thoughts and >200k buckets |
| Vertex | `langchain-google-vertexai` (`ChatVertexAI`), or `langchain-google-genai` with `vertexai=True` | — | signature re-attachment; `function`→`user` role; thinkingLevel vs budget; usage repair |
| Mistral | `langchain-mistralai` | — | strict alternation only |
| Fallback option | `litellm` (`ChatLiteLLM` / `litellm.acompletion`) | Broad wire coverage and its own `ContextWindowExceededError` mapping | Loses the fine-grained signature, cache and streaming control this SDK relies on. Use only for `generic`-family custom providers. |

Recommendations:
- Keep LangChain Python for parity with LangGraph, but **own the message converters**. The TS SDK vendors the Anthropic, Bedrock and Google converters precisely because the upstream ones break on cross-provider history.
- Set `max_retries` on each client explicitly. Python LangChain relies on the provider SDK's own retries (the `openai`/`anthropic` clients default to 2), whereas TS relies on AsyncCaller.

### 7.2 Must-port logic (provider-independent)

1. Provider registry with families and traits, lazy import, and a disposer.
2. `prepare_provider_request`: projection-mode detection, attachment projection, per-family tool-message projection and cache stripping, handoff cues, alternation coalescing, and a frozen request object carrying its measurement.
3. `attempt_invoke`:
   - metadata stamps;
   - `astream` accumulation with `+` (AIMessageChunk add);
   - OpenRouter replay fix;
   - manual-tool-stream normalization (`modify_delta_properties`);
   - empty-name tool-call filter;
   - truncation guard;
   - sealed-run closing via `run_manager.on_llm_end` (in Python, breaking out of `astream` similarly skips `on_llm_end`).
4. `try_fallback_providers` with overflow-preferred error selection.
5. Prompt-cache module: tail marker (Anthropic and Bedrock variants), tool partitioning (3 variants), system-message builders, TTL rules, OpenAI explicit breakpoints.
6. Stream limits: byte tallies per call key, seal semantics, per-attempt generation key, run-wide abort (an `asyncio.Event` or cancellation scope instead of AbortSignal).
7. Overflow regex table (keep the fixtures from `src/utils/__tests__/fixtures/contextOverflowSignatures.ts`) and the recovery planner (pure functions; port 1:1).
8. Context-pressure meter and preemption seal/restart (depends on the graph slice).
9. Stream smoother: an async generator with a queue and adaptive piece sizing. Optional, UI only.

### 7.3 Suggested module layout

```
librechat_agents/llm/
  __init__.py
  registry.py            # ProviderFamily, register_provider, get_chat_model_class, traits
  providers.py           # built-in lazy registrations (Providers enum → import path)
  init.py                # initialize_model (param re-assignment, bind_tools)
  request.py             # is_thinking_enabled, resolve_model, max_output_tokens_key
  prepare_request.py     # PreparedProviderRequest, projection modes, attachments
  invoke.py              # attempt_invoke, try_fallback_providers, sealed-run close, usage synth
  truncation.py          # OutputTruncationError, stop-reason normalization
  stream_limits.py
  preempt.py
  context_overflow/{signatures.py, recovery.py}
  context_pressure.py
  prompt_cache/{markers.py, anthropic_tools.py, bedrock_tools.py, openrouter_tools.py, openai_explicit.py, ttl.py}
  stream/{smoother.py, chunk_adapters.py, metadata_dedup.py}
  providers/
    openai/{chat.py, responses.py, convert.py, azure.py, deepseek.py, xai.py, moonshot.py, astra.py}
    openrouter/{chat.py}
    anthropic/{chat.py, convert_inputs.py, convert_outputs.py, ids.py, betas.py}
    bedrock/{chat.py, convert_inputs.py, convert_outputs.py, cache_points.py, tool_cache.py}
    google/{genai.py, vertex.py, convert.py, schema.py, signatures.py}
    mistral/{chat.py}
librechat_agents/messages/{reasoning_types.py, alternation.py, handoff_cue.py, cache.py, thinking.py, core.py}
```

### 7.4 Porting pitfalls checklist

- Chunk merging relies on `index` on content blocks. Any provider whose Python adapter omits it (Bedrock) will produce fragmented content arrays.
- Anthropic `usage_metadata` must not be double-counted across `message_start` and `message_delta`.
- Cache markers must be stripped when switching providers mid-conversation:
  - Anthropic `cache_control` into Bedrock/OpenAI;
  - Bedrock `{cachePoint}` blocks into everyone else. They have no `type` key, so filter them explicitly.
- A longer TTL must precede a shorter one (Anthropic and Bedrock), so strip stale 5m markers.
- Keep exactly one tail marker, and never anchor it on thinking, reasoning, empty text or `input_json_delta` blocks.
- Handle overflow errors wrapped by LangChain (`ContextOverflowError`), including HTTP 429 (OpenAI) and HTTP 200 mid-stream failures (Bedrock Nova/Llama).
- Gemini-3 without thought signatures on replayed function calls fails with a 400.
- Tool-call ids from OpenAI Responses (`fc_…`, long) break Anthropic id validation.
- Thinking-mode Claude turns with tool_use but no thinking block (history from another agent) must be folded into text.

Key files:
- `src/llm/providers.ts`, `src/llm/providerRegistry.ts`, `src/llm/init.ts`
- `src/llm/invoke.ts`, `src/llm/prepareProviderRequest.ts`
- `src/messages/cache.ts`, `src/messages/anthropicToolCache.ts`, `src/llm/bedrock/toolCache.ts`, `src/llm/openrouter/toolCache.ts`
- `src/llm/streamLimits.ts`, `src/llm/contextOverflowRecovery.ts`, `src/utils/errors.ts`
- `src/llm/{anthropic,bedrock,google}/utils/*`, `src/llm/vertexai/index.ts`, `src/llm/openai/index.ts`
- `src/graphs/Graph.ts` (lines ~3480–4760)
