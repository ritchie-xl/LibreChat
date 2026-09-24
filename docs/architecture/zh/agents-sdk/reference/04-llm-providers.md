# @librechat/agents：LLM 与提供商层

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

所有路径均相对于 SDK 仓库根目录。

---

## 0. 本层所处的位置

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

锁定的依赖版本（来自 `package.json`）：`@langchain/core 1.2.8`、`@langchain/openai 1.5.8`、`@langchain/anthropic 1.5.2`、`@langchain/aws ^1.4.2`、`@langchain/google-genai/-vertexai/-gauth/-common 2.2.0`、`@langchain/mistralai ^1.2.0`、`@langchain/deepseek ^1.1.3`、`@langchain/xai ^1.4.3`、`openai ^6.46`、`@anthropic-ai/sdk ^0.115`、`@aws-sdk/client-bedrock-runtime ^3.1075`。

---

## 1. 提供商矩阵

`Providers` 枚举定义在 `src/common/enum.ts:89`。每个提供商都在 `src/llm/providers.ts` 中注册，带三个特性：`family`、`manualToolStream` 和 `strictAlternation`。

| 枚举（线上字符串） | SDK 类（文件） | 上游 LangChain 基类（包） | 家族 / 特性 | 主要定制 |
|---|---|---|---|---|
| `OPENAI`（`openAI`） | `ChatOpenAI`（`src/llm/openai/index.ts:2807`，lc_name `LibreChatOpenAI`） | `ChatOpenAI`（@langchain/openai）。它委托给 `LibreChatOpenAICompletions`（1854）和 `LibreChatOpenAIResponses`（2291）。 | openai | `CustomOpenAIClient`，带可中止的 `fetchWithTimeout`，OpenAI SDK 设为 `maxRetries: 0`。通过 `_lc_stream_delay` 做流平滑。标量元数据去重。显式提示缓存断点（`promptCacheExplicit`）。`safety_identifier`。GPT‑6 Astra 请求整形。为 gpt‑5.6/Astra 加入加密推理的 `include`。透传 `reasoning`/`reasoning_details`/`provider_specific_fields` 增量。顺序工具调用的“封存”（seal）标记，仅限 api.openai.com。从 strict 工具中剥离 `intent` 参数。Responses API 的回放位置、注解（annotations）和缓存写入用量。 |
| `AZURE`（`azureOpenAI`） | `AzureChatOpenAI`（2912，`LibreChatAzureOpenAI`），带 Azure Completions/Responses 委托（2450/2605） | `AzureChatOpenAI`（@langchain/openai） | openai | 与 OpenAI 相同。`CustomAzureOpenAIClient`。封存标记只作用于第一方 Azure 主机（`*.openai.azure.com`、`*.cognitiveservices.azure.com`、`*.api.cognitive.microsoft.com`）。 |
| `DEEPSEEK` | `ChatDeepSeek`（3038） | `ChatDeepSeek`（@langchain/deepseek） | openai | 在工具调用轮次回放 `reasoning_content`（`includeReasoningContent: true`）。自定义非流式 `_generate`。流式解析 `<think>…</think>` 标签并写入 `reasoning_content`，处理标签被拆分到多个片段的情况。 |
| `XAI`（`xai`） | `ChatXAI`（3559） | `ChatXAI`（@langchain/xai） | openai | 通过重置客户端来支持自定义 `configuration.baseURL`/`clientConfig.baseURL`。流平滑。`maxRetries: 0`。 |
| `MOONSHOT` | `ChatMoonshot`（3544） | 继承 SDK 的 `ChatOpenAI` | **generic** | 强制 `includeReasoningContent: true`（Kimi 思考 + 工具）。 |
| `OPENROUTER` | `ChatOpenRouter`（`src/llm/openrouter/index.ts`） | 继承 SDK 的 `ChatOpenAI` | openai | 用 `reasoning` 对象代替 `reasoning_effort`。`includeReasoningDetails`。上游为 Claude 时，把 `reasoning_details` 转成 `thinking`/`redacted_thinking` 内容。流式过程中累积 reasoning_details，只在带 `finish_reason` 的片段上刷出。`preserveToolCacheControl`。工具级 `cache_control`。Responses 模式下的顶层 `cache_control`。 |
| `ANTHROPIC` | `CustomAnthropic`（`src/llm/anthropic/index.ts:452`） | `ChatAnthropicMessages`（@langchain/anthropic） | anthropic，**manualToolStream** | 内置（vendored）消息转换（`utils/message_inputs.ts`）。工具 id 规范化。思考参数校验（Opus 4.7）。抑制采样参数。根据工具类型自动推导 betas。`output_config`、`inference_geo`、`context_management`（压缩（compaction））。对 Claude ≥4.6 剥离助手预填充（prefill）并重新锚定缓存。在 `message_delta` 上按增量计算用量。流平滑。 |
| `BEDROCK` | `CustomChatBedrockConverse`（`src/llm/bedrock/index.ts:153`） | `ChatBedrockConverse`（@langchain/aws） | bedrock，**manualToolStream**，**strictAlternation** | 直接掌控 `ConverseStream`。把 `contentBlockIndex` 提升为内容块上的 `index`，使合并能正常工作。工具缓存点。应用推理配置文件（application inference profile）ARN 替换。`serviceTier`。Guardrails。在 `contentBlockStop` 处显式封存 tool-use。内置 `message_inputs.ts`（用户轮次合并、cachePoint 上提、推理过滤）。 |
| `GOOGLE` | `CustomChatGoogleGenerativeAI`（`src/llm/google/index.ts`） | `ChatGoogleGenerativeAI`（@langchain/google-genai） | google | 重建 `GenerativeAI` 客户端（baseUrl、apiVersion、customHeaders）。注入 `thinkingConfig`。`includeServerSideToolInvocations`（functionCalling 模式 `VALIDATED`）。内置 `utils/common.ts`（思考签名、gemini‑3 的占位签名、服务端工具片段）。对 Flash ≥3.6 丢弃预填充。用量包含 thoughts token 和 `over_200k` 分桶。流平滑。 |
| `VERTEXAI` | `ChatVertexAI`（`src/llm/vertexai/index.ts:477`，`LibreChatVertexAI`） | `ChatGoogle`（@langchain/google-gauth）加 `ChatConnection`（@langchain/google-common） | google | `CustomChatConnection.formatData`：`thinkingBudget:-1` 变为动态；`thinkingLevel` 覆盖 `thinkingBudget`；`fixThoughtSignatures`；`role:function` 改写为 `user`。`repairStreamUsageMetadata`（thoughts token）。封存完整的工具调用。流平滑。 |
| `MISTRALAI` / `MISTRAL` | `CustomChatMistralAI`（`src/llm/mistral/index.ts`） | `ChatMistralAI`（@langchain/mistralai） | mistral，**strictAlternation** | 只有流平滑。 |

注册表特性的作用：
- **`family`** 驱动与提供商无关的判断函数：`src/utils/llm.ts` 中的 `isOpenAILike`、`isGoogleLike` 和 `isAnthropicLike`；`src/llm/request.ts` 中的 `isThinkingEnabled`；附件投影；截断。
- **`manualToolStream`** 让 `attemptInvoke` 在累积完成后用 `modifyDeltaProperties`（`src/messages/core.ts:265`）对片段做后置规范化。该函数：
  - 把 `*_delta` 块类型映射为基础类型；
  - 依据 `allowedTypesByProvider` 把不允许的类型强制转为 `text`；
  - 把空的 `tool_use.input: ''` 改为 `'{}'`；
  - 对 Bedrock，合并连续的 `reasoning_content`/`text` 块并保留签名（`reduceBlocks`）。
- **`strictAlternation`** 触发 `coalesceAdjacentUserTurns`（`src/messages/alternation.ts:187`）。

`initializeModel`（`src/llm/init.ts`）：
- 除非传入了 `override` 实例，否则构造 `new Class(clientOptions)`。
- 对 OpenAI 类提供商，它从选项中重新赋值 `temperature/topP/frequencyPenalty/presencePenalty/n`，以绕开 LangChain 构造函数的默认值。对上游 Vertex，重新赋值 `temperature/topP/topK/...`。
- 然后调用 `bindTools(tools)`；如果模型没有 `bindTools` 则抛错。

`src/llm/request.ts` 中的辅助函数：
- `isThinkingEnabled`：Anthropic 的 `thinking`；Bedrock 的 `additionalModelRequestFields.thinking`；OpenAI 兼容接口的 `modelKwargs.thinking.type==='enabled'`（LiteLLM 风格的 Claude）。
- `resolveClientOptionsModel`：读取 `model` 或 `modelName`。
- `getMaxOutputTokensKey`：Google/Vertex 用 `maxOutputTokens`，其他都用 `maxTokens`。

---

## 2. 模型调用路径（逐步说明）

### 2.1 调用之前（Graph.createCallModel）

1. **为缓存准备工具。** `getPreparedToolsForBinding` → `prepareToolsForPromptCache`，见 §3。
2. **构建模型。** `initializeModel({tools, provider, clientOptions})`，然后接在 `agentContext.systemRunnable` 之后。系统消息（及其缓存标记）和摘要载体消息都在 system runnable 中加入。模型被包装为 `RunnableSequence(systemRunnable → RunnableBinding(chatModel))`。
3. **线上转换（wire transforms）。** 每一步都经过 `contextPressure.trackProjection(before, after)`，使 token 统计能跟随每条消息的来源。
   - 若 `useLegacyContent`，执行 `formatContentStrings`。
   - `projectToolMessagesForProvider`：把工具调用输入限制在 `calculateMaxToolCallInputChars(maxContextTokens)` 以内。
   - **Bedrock 特殊处理：** 如果最后两条消息是“内容为字符串的 AI 消息 + ToolMessage”，则把 AI 文本去除首尾空白后包装为 `[{type:'text'}]`；若为空则替换为 `''`。Bedrock 拒绝结尾空白和空白文本。
   - `projectServingArtifacts`，仅在最后一条消息是 ToolMessage 时应用：
     - Anthropic 类：`projectAnthropicArtifactContent`，把图片等工具产物放进工具结果；
     - OpenAI 类（DeepSeek 除外）和 Google：`projectArtifactPayload`。
     - 仅当测量后的载荷仍然放得下时才应用。
   - `applyProviderMessageTransforms`：
     - 开启思考时执行 `ensureThinkingBlockInMessages`（`src/messages/format.ts:4271`）。最后一条人类消息之后、缺少 thinking/reasoning 块的 AI+工具链，会被折叠进一条 `[Previous agent context]` HumanMessage，因为 Anthropic 在思考模式下要求 tool_use 之前有带签名的思考块。本次运行产生的消息（索引 ≥ `startIndex`）不受影响。
     - 未绑定工具时执行 `foldToolBlocksForToollessAgent`，因为没有工具 schema 时提供商会拒绝工具块。
     - `appendInstructionlessHandoffCue`。
     - `appendPredecessorHandoffCue`，仅 Anthropic 类（见 §2.4）。
   - `compactSyntheticProviderContext`：二分查找（12 次迭代），收缩合成的上下文 HumanMessage，直到载荷放得下。
   - 对严格交替的提供商执行 `coalesceAdjacentUserTurns`。它在压缩之后、缓存标记之前运行。
   - `sanitizeOrphanToolBlocks`（Anthropic 类）：在裁剪器未运行、消息有变化或开启提示缓存时执行。
   - `applyServingTailCache`，最后放置唯一的尾部缓存标记（§3）。
4. **`prepareProviderRequest`**（`src/llm/prepareProviderRequest.ts:528`）：
   - **投影模式：** 若 `usesNativeOpenAIResponses(model, provider, config)` 为真则为 `openai-responses`。该函数沿 `bound/last` 包装链查找 `_useResponsesApi()` 或类名包含 `Responses` 的类。否则为 `chat-messages`。
   - **`projectMessagesForProviderMode`：**
     - `projectToolStreamContentForProvider`（原生与回退两种流式工具内容）；
     - `projectAttachmentsForProvider`：带原始字节的 Bedrock `document` 块，对其他提供商转成标准 base64 `file` 块。Anthropic 只保留 pdf/jpeg/png/gif/webp，OpenAI 类只保留 pdf，被丢弃的附件替换为文本 `[Attachment omitted because its binary format is unsupported by this provider.]`；
     - 然后是按提供商的工具消息投影和缓存标记剥离：

       | 提供商 | 投影 | 剥离的缓存标记 |
       |---|---|---|
       | Responses | `projectOpenAIResponsesToolMessageContent` | Anthropic + Bedrock |
       | OpenRouter | `projectOpenRouterToolMessageContent` + computer-call 输出 → 文本 | 只剥离 Bedrock；保留 Anthropic `cache_control` |
       | OpenAI 类 | `projectOpenAIChatToolMessageContent` | 两者都剥离 |
       | Anthropic | `projectSingleTextToolOutputsToText` | Bedrock |
       | Bedrock | `projectCacheControlledToolOutputsToText` | Anthropic |
       | generic | 结构化 + 单文本 → 文本 | 两者都剥离 |
   - `annotateMessagesForLLM`：工具输出引用注册表的注解。
   - 移交提示语（handoff cue）（无指令提示语对所有提供商生效；前驱提示语对 Anthropic 类添加，否则移除）。
   - 若严格交替，执行 `coalesceAdjacentUserTurns`。
   - `measure(preparedMessages)`，然后冻结结果，并用私有 symbol 打上标记。`assertPreparedProviderRequestFor` 检查模型身份、提供商和投影模式是否一致。
5. **调用前溢出守卫。** 如果 `measurement.fits === false`，会抛出一个本地 `ContextOverflowError`（`final_context_overflow`），进入与提供商溢出相同的 catch 块。测量得到的预算记录在一个 WeakMap 中。
6. **等待 `ON_CONTEXT_USAGE` 事件**完成。然后构建调用配置：
   - 与运行级 `attemptBreaker` 组合的 signal；
   - Langfuse 元数据和处理器；
   - `agentId`；
   - `STREAM_LIMIT_EPOCH_KEY`。

### 2.2 `attemptInvoke`（src/llm/invoke.ts:691）

这是主调用、回退调用和摘要调用共用的唯一入口。

1. **打元数据标记：**
   - `__invoked_provider`、`provider`、`resolvedProvider`；
   - `__invoked_model` 和 `model`（通过 `resolveServingModelId`，它沿包装链查找 `.model`）；
   - `lc_stream_limit_attempt`：进程内单调递增序号，使每次尝试都有独立的流限制预算。
2. **流限制租约。** `registerActiveStreamLimitGeneration(graph, generationKey)`，其中 `generationKey = checkpoint_ns|node|step|attempt`。在 `finally` 中释放。
3. **预备子智能体的 begin/finish**，用于工具的提前执行（在 LLM 作用域之外）。
4. **`attemptInvokeBody`：**
   - 可选地安装一个 `handleChatModelStart` 回调，它 (a) 捕获真实的 LLM `runId`（关闭已封存的运行时需要），(b) 执行提供商消息投影不变量检查（`off|warn|assert`）。
   - **流式路径**（存在 `model.stream`）。使用以下三种消费循环之一：
     1. **提供了 `onChunk`**（摘要、公共消费者）。每个片段：
        - `throwIfBreakerTripped`；
        - `enforceStreamLimitsForWireChunk`；
        - `await onChunk(chunk, metadata)`；
        - 用 `concat` 累积。
     2. **没有注册默认流处理器**（本地处理器）。每个片段：
        - 新建 `ChatModelStreamHandler().handle(CHAT_MODEL_STREAM, {chunk})`；
        - 累积；
        - 轮询抢占。这是唯一可以**封存**的循环。
     3. **已注册处理器。** 图的处理器通过 `streamEvents` 单独接收线上片段，所以这个循环只：
        - 同步计入流限制（生产者一侧）；
        - 用 `STREAM_LIMIT_REDISPATCH_KEY` 重新分发经 OpenRouter 转换的片段；
        - 累积。
   - **累积：** `@langchain/core/utils/stream.concat`（按 `index` 合并 AIMessageChunk）。
   - **OpenRouter 特殊处理：** 携带 `reasoning_details` 的最后一个片段还会回放全部累计内容。`removeOpenRouterFinalReasoningReplayContent` 切掉已经见过的前缀，`getStreamHandlingChunk` 从分发出去的副本中剥掉 `reasoning_details`。
   - 流结束后：
     - 若 `manualToolStream`，执行 `modifyDeltaProperties`；
     - 处理抢占结果（见 §4.2）；
     - 只保留 `name` 非空的 `tool_calls`；
     - `assertNotTruncatedToolCall(finalChunk, provider)`（`src/llm/truncation.ts`）。当停止原因是 max-tokens/length 且存在任何 tool_call、tool_call_chunk 或无效工具调用时，抛出 `OutputTruncationError`。它检查 `stopReason`、`stop_reason`、`finish_reason`、`finishReason`、`messageStop.stopReason`、`incomplete_details.reason` 和 `additional_kwargs.stop_reason`。Google/Vertex 跳过此检查，因为它们以原子方式发出工具调用。
   - **非流式路径：** `model.invoke`，然后做同样的过滤和截断检查。
   - 返回 `{messages:[finalChunk]}`。

### 2.3 重试与回退

- **SDK 层面没有重试循环。**
  - 瞬时错误的重试来自 LangChain 的 `AsyncCaller`：各提供商基类中的 `this.caller.call`、`callWithOptions` 以及 `completionWithRetry`/`createStreamWithRetry`。它使用 `clientOptions.maxRetries`（LangChain 默认 6），带指数退避。
  - OpenAI 家族的类以 `maxRetries: 0` 构造 `CustomOpenAIClient`，避免与 LangChain 的重试叠加。
  - `llmConfig.ts` 中只有一个被注释掉的 `maxRetries`。
- **溢出恢复先于回退**（§4.3）。触发的流限制或 `PreparedSubagentError` 会立即重新抛出（`attemptBreaker.abort(err)`），永远不进入恢复或回退。
- **`tryFallbackProviders`**（invoke.ts:1536）：
  - 对每个 `t.FallbackConfig {provider, clientOptions, maxContextTokens}`：
    - 用相同的工具执行 `initializeModel`，但**不带** system runnable（系统提示不会重新接入）；
    - 打上 `__invoked_model` 标记；
    - 可选地准备请求。Graph 会传入一条按回退模型窗口大小完整重做的投影管线：产物、转换、孤立块清理、尾部缓存、测量、合成上下文压缩、溢出错误；
    - 再次检查断路器；
    - 调用 `attemptInvoke`。
  - 失败时记录 `lastError`。溢出错误（按回退模型自己的 `maxContextTokens` 和估算的提示长度判断）会在一个 WeakMap 中附带 `FallbackErrorContext`。
  - 最终抛出的优先级：先是第一个回退的溢出，其次是主调用的溢出，最后是最后一个错误。完整的候选列表可通过 `getFallbackOverflowCandidates` 取得，这样 Graph 可以按回退模型的窗口规划恢复，并在不同校准比例之间换算预算。

### 2.4 移交提示语与交替

**`INSTRUCTIONLESS_HANDOFF_CUE`**（`src/messages/handoffCue.ts`）：
- 文本："Continue as the receiving agent using the preceding user request and context."（译：以接收方智能体的身份，基于前面的用户请求和上下文继续。）
- 当载荷末尾恰好是接收方进入时所带的那条 AI 消息时，它作为一条元（meta）HumanMessage（`isMeta`，`source:'routing'`）追加。那条消息的 id 由 `withInstructionlessHandoffCue` 记录在 `config.metadata.__handoff_cue_message_id` 中。
- 它在所有传输方式上都生效。

**`PREDECESSOR_HANDOFF_CUE`**：
- 只对 Anthropic 类提供商追加，条件是载荷以**本次运行产生的** AI 消息结尾（用 `graph.isRunProducedMessage` 检查）。
- 原因：Claude 会把结尾的助手轮次当作预填充并接着写，或者返回空内容。
- 对非 Claude 的回退模型，`removePredecessorHandoffCue` 会把它剥掉。

**`coalesceAdjacentUserTurns`**：
- 把相邻的、非工具结果的人类消息合并为一条。字符串用 `\n\n` 连接，块则直接拼接，并保留来源信息。
- 与前一条 AI 消息的工具调用配对的工具结果 HumanMessage 保持不变。
- 只对 Bedrock 和 Mistral 生效。

### 2.5 用量统计

目标是规范化的 LangChain `usage_metadata {input_tokens, output_tokens, total_tokens, input_token_details{cache_read, cache_creation}, output_token_details{reasoning}}`。

- **Anthropic**（`getAnthropicUsageMetadata`，`utils/message_outputs.ts`）：
  - `input_tokens = input + cache_creation + cache_read`。
  - 流式时，`message_start` 给出初始计数。在 `message_delta` 上，`withIncrementalMessageDeltaUsage` 把累计计数换算成增量，这样 `concat` 求和正确，不会重复计数。
- **OpenAI Chat：** `prompt_tokens_details.cached_tokens` 映射为 `cache_read`，`cache_write_tokens` 映射为 `cache_creation`，`reasoning_tokens` 原样带出。用量片段最后产出。
- **OpenAI Responses：** `input_tokens_details.cache_write_tokens` 先借道 `response.metadata.__librechat_cache_write_tokens` 传递，再移入 `usage_metadata`（`attachCacheWriteUsage`）。
- **Bedrock**（`handleConverseStreamMetadata`）：`cacheReadInputTokens`/`cacheWriteInputTokens` 成为 details。这里的 `input_tokens` 只是新增部分；缓存分桶单独计。
- **Google GenAI：** `output = candidates + thoughts`；`cachedContentTokenCount` 成为 `cache_read`；`gemini-3-pro-preview` 额外有 `over_200k` 和 `cache_read_over_200k`。用量在流结束后发出一次。
- **Vertex：** `repairStreamUsageMetadata` 用最后一个 `generationInfo.usage_metadata` 替换末尾片段中有缺陷的 `output_tokens`（漏掉了 thoughts）。
- **没有提供商用量的已封存或已重启轮次：** `synthesizeSealedUsage` 估算用量：输入 = 宿主 token 计数器对提示的计数加上指令开销，输出 = 计数器对部分片段的计数。它标记 `response_metadata.estimated_usage: true`，让校准忽略这条数据。
- **`endSealedModelRun`** 手动触发 `handleLLMEnd`。消费者从 `for await` 中跳出时，LangChain 不会触发 end/error 回调。它用配置加上沿模型包装链（`bound`/`last`/`steps`）找到的所有 `callbacks` 重建回调管理器。若失败，则改为分发一个自定义 `CHAT_MODEL_END` 事件。

### 2.6 推理 / 思考内容的提取

各提供商的推理内容形态如下：

| 来源 | 形态 |
|---|---|
| Anthropic | 内容块 `thinking`（带 `signature`，来自 `thinking_delta`/`signature_delta`）和 `redacted_thinking` |
| Bedrock | `reasoning_content` 块 `{reasoningText:{text, signature}}` |
| Google GenAI | `additional_kwargs.reasoning`（拼接后的 `thought:true` 片段）；另有函数调用的思考签名，位于 `additional_kwargs.__gemini_function_call_thought_signatures__`（id → 签名） |
| Vertex | `additional_kwargs.signatures[]`，以及推理内容块 |
| OpenAI Chat 兼容 | `additional_kwargs.reasoning_content`（DeepSeek、Moonshot、vLLM，以及部分网关的 `delta.reasoning`）；`reasoning_details`；`provider_specific_fields` |
| OpenAI Responses | `additional_kwargs.reasoning` `{id, summary[], encrypted_content}`；回放位置 |
| OpenRouter | `reasoning`（文本，逐片段）和 `reasoning_details`（仅最后一个片段） |
| DeepSeek 回退 | 内容中的 `<think>` 标签，解析进 `reasoning_content` |

- `isReasoningContentBlock`（`src/messages/reasoningTypes.ts`）是权威集合：`think, thinking, thinking_delta, redacted_thinking, reasoning, reasoning-delta, reasoning_content`。
- 面向 UI 的提取在 `src/stream.ts` 中（`getChunkContent`，约第 1400 行；推理检测约第 2137 行）：
  - OpenAI/Azure 优先使用 `reasoning.summary[0].text`；
  - OpenRouter 优先使用内容而非推理；
  - 其他提供商使用带键的 `reasoning_content`/`reasoning`。

---

## 3. 各提供商的提示缓存策略

默认 TTL 为 `'1h'`（`DEFAULT_PROMPT_CACHE_TTL`，`src/messages/cache.ts`）。`'5m'` 会完全省略 `ttl` 字段，使载荷与旧版标记逐字节一致。缓存需通过 `clientOptions.promptCache: true` 和 `promptCacheTtl` 显式开启。

**核心原则：单一尾部断点。** 每次调用都会剥掉所有过期标记，再在最后一条非合成消息的最后一个可缓存块上放一个标记。理由见 `docs/prompt-cache-benchmark.md`：在智能体工具循环中，旧的“最后 2 条用户消息”放置方式会让后续追加的每个助手/工具轮次都得不到缓存。实测采用尾部策略后有效成本降低 26–41%。旧函数 `addCacheControl`/`addBedrockCacheControl` 仍然导出。

### 3.1 Anthropic（直连）

最多预留 4 个断点：工具、系统、稳定前缀和尾部。

- **工具**（`src/messages/anthropicToolCache.ts`，`partitionAndMarkAnthropicToolCache`）：
  - 把工具稳定地分区为 `[static…, deferred…]`，“deferred”指工具定义中的 `defer_loading`。
  - 在**最后一个静态工具**上打 `cache_control`，这样后来发现的工具不会破坏前缀缓存。
  - LangChain 工具把标记放在 `tool.extras.cache_control` 中，由适配器提升。提供商原生形态的工具直接在块上携带标记：包括类型前缀为 `text_editor_`、`computer_`、`bash_`、`web_search_`、`web_fetch_`、`code_execution_`、`memory_`、`tool_search_`、`mcp_toolset` 的内置工具，或带 `input_schema` 的原始对象。
  - 从其他所有工具上剥掉过期标记，因为较长的 TTL 必须位于较短的 TTL 之前。包装器保留原型，从不修改原始工具。
- **系统**（`AgentContext.buildSystemMessage`）：
  - 系统消息为 `[{text: stableInstructions, cache_control}, {text: dynamicInstructions}]`。
  - 当稳定指令和动态指令都存在且开启缓存时，动态指令会被**移出系统消息**，放进一条 HumanMessage“动态尾部”。摘要也一样。
  - 随后 `buildBodyWithPromptCacheDynamicTail`：
    - 用 `addCacheControlToStablePrefixMessages(…, 1)` 标记到最近一条人类消息为止的稳定前缀（优先最后一条助手消息，否则任意对话消息；开头的消息永远不作为锚点）；
    - 插入动态尾部；
    - 在末尾消息上放尾部标记。
  - 没有动态尾部时，使用 `addTailCacheControl(body)`。
  - 摘要载体有自己的 `cache_control`。
- **尾部**（`addTailCacheControl`）：
  - 从后往前遍历，剥掉所有消息上的 `cache_control` 和 `cachePoint`。
  - 锚定在最后一个既不是推理、也不是 `input_json_delta`、不是空文本、也不是 cachePoint 的块上。
  - 跳过合成的元消息（`additional_kwargs.isMeta` 或 `source==='skill'`）和 computer-call 输出。
  - 当 system runnable 负责系统提示时，Graph 不放自己的尾部标记，改由 AgentContext 放置。
- **转换器层面的修正：**
  - `hoistToolResultCacheControl` 把标记从工具结果内部内容上提到 `tool_result` 块上。
  - `stripUnsupportedAssistantPrefill` 对 Claude 4.6+（`modelDisallowsAssistantPrefill`）移除结尾的助手轮次；如果尾部标记原本在被剥掉的预填充上，则**重新锚定**，并保留 `1h` TTL。
  - 顶层 `cache_control` 调用选项（自动推进，SDK 1.5.x）以叠加方式透传。

### 3.2 Bedrock Converse

- **系统：** `[{text: stable}, {cachePoint:{type:'default', ttl?}}, {text: dynamic}]`。已有的系统 cachePoint 会被规范化为解析后的 TTL（`sanitizeBedrockSystemMessage`）。
- **尾部：** `addBedrockTailCacheControl` 在最后一条非系统、非合成消息的最后一个非空文本块之后，插入一个**独立的** `{cachePoint}` 块。即使存在 system runnable，Bedrock 也总是在 Graph 中打标记。
- **工具结果：** 嵌在 `toolResult.content` 内的 cachePoint 会被 Bedrock 静默忽略，所以 `convertToolMessageToConverseMessage` 把它提出来，作为紧跟 `toolResult` 的同级块。
- **工具**（`src/llm/bedrock/toolCache.ts`）：
  - 工具先转换为 `toolSpec`，再分为静态/延迟两组。
  - 最后一个静态工具获得私有标记 `__lc_bedrock_cache_point_after`。在 `invocationParams` 阶段，`insertBedrockToolCachePoint` 把该标记替换为 `toolConfig.tools` 中一个真正的 `{cachePoint}` 条目。
  - 如果所有工具都是延迟加载的，则改为设置禁用标记 `__lc_bedrock_skip_tool_cache`。
- **按模型限制：**
  - 工具 cachePoint **仅限 Claude**（`supportsBedrockToolCache` = `/claude|anthropic/i`）；Nova 拒绝工具中的 `cachePoint`。
  - 消息和系统 cachePoint 在 Nova 上可用。
  - 对非 Claude 模型，`1h` TTL 被压到 `5m`（`resolveBedrockPromptCacheTtl`）。
- **内置的上游 `applyCachePointsToConversePayload`**（`cachePoints.ts`）处理 `options.cache_control` 路径：系统、最后一条消息和工具，并对 Nova 排除工具块和 TTL。

### 3.3 OpenRouter（OpenAI 线上格式，Anthropic 风格标记）

- 系统消息使用 Anthropic 的 `cache_control` 文本块格式。
- 尾部标记使用 Anthropic 格式的 `addTailCacheControl`。OpenRouter **不**应用 `stripAnthropicCacheControl`；只剥离 Bedrock 标记。
- 序列化工具消息时，`preserveToolCacheControl` 保留带缓存修饰的规范工具文本块。
- 工具：`partitionAndMarkOpenRouterToolCache` 转换为 OpenAI 工具格式，并在最后一个静态工具上放 `cache_control`。保留 `defer_loading`。
- Responses 模式把 `cache_control` 放在请求顶层，并从工具上剥掉。
- TTL 会被转发；上游为 Claude 时按需降级。

### 3.4 OpenAI / Azure（第一方）

OpenAI 的自动缓存不需要标记。SDK 另外通过 `promptCacheExplicit`（GPT‑5.6 托管请求透传）支持**显式缓存**：
- 请求带上 `prompt_cache_options: {mode:'explicit', ttl:'30m'}`。
- **Chat：** `addChatCacheBreakpoints` 在以下位置放 `prompt_cache_breakpoint:{mode:'explicit'}`：
  - **最后一条 system/developer 消息**的最后一个可缓存片段；
  - **最近一条用户消息之前**的最后一条可缓存消息。
- **Responses：** `addResponseCacheBreakpoints` 使用相同的选择规则，但只作用于输入角色（system/developer/user），且只作用于 `input_text`/`input_image`/`input_file` 片段。输出片段会被 400 拒绝。
- 对 OpenAI 类提供商，所有 Anthropic 和 Bedrock 标记都会被剥掉。
- `cache_write_tokens` 在用量中呈现为 `cache_creation`。

### 3.5 Google、Mistral、DeepSeek、xAI

没有显式缓存。标记由通用投影剥掉。会报告隐式缓存命中（Google 为 `cachedContentTokenCount`，OpenAI 兼容提供商为 `cached_tokens`）。

---

## 4. 流限制、抢占与上下文溢出

### 4.1 流限制（src/llm/streamLimits.ts）

这是针对失控生成的断路器。起因是一个案例：一条 149,923 字符的 SQL 参数流式输出了 26 分钟。

**配置**（`resolveStreamLimits`）：

| 设置 | 默认值 | 含义 |
|---|---|---|
| `maxToolCallArgBytes` | 65,536 | 单个流式工具调用的 UTF-8 字节数 |
| `maxToolCallArgBytesByTool[name]` | — | 按工具覆盖 |
| `maxDeltaEventsPerTurn` | 0（关闭，需显式开启） | 每次生成的流片段事件数 |

`0`、负值或 `Infinity` 会禁用限制；`NaN` 回退到默认值。

**计数键：**
- 生成键：`langgraph_checkpoint_ns|langgraph_node|langgraph_step|lc_stream_limit_attempt`。刻意*不*使用 step 键，因为它会在推理切换时分叉。
- 单次调用的计数键：片段的 `index`，没有则用 `id`（空字符串视为不存在），再没有则用在批次中的位置。

**工具参数字节计数：**
- 处理跨片段被拆开的 UTF-16 代理对。
- 感知封存：
  - `kind:'single'` 封存（OpenAI Responses 的 `…arguments.done` 重述，或 Bedrock 的空参数停止片段）**替换**计数而不是累加，然后释放该调用；
  - `kind:'all'` 封存（Google）对每个调用单独判断。
- `enforceCompleteToolCallArgLimit` 覆盖没有片段、直接以完整解析形式到达的调用。

**计费去重：**
- LangChain 会把同一个片段对象同时交给生产者循环和 `streamEvents` 消费者。`claimStreamLimitCharge` 按生成、按片段对象维护一个带符号的额度余额，使片段只计一次。
- `linkStreamLimitCanonical` 把 Bedrock 增强后的片段与其回调副本关联，使两者算作一次发出。

**生命周期：**
- 每次尝试一份租约；重置时 `sweepStaleStreamLimitEntries` 按纪元（epoch）清扫。
- 触发时抛出 `StreamLimitExceededError {kind, limit, observed, toolName}`。它会中止运行级断路器，让并行的智能体节点和子智能体也停下。该错误永远不会被恢复，也不会通过回退重试。
- 每个产出的片段都会检查 `throwIfBreakerTripped`，因为适配器可能忽略中止信号。

**流平滑**（`src/llm/stream/smoother.ts`、`chunkAdapters.ts`）：
- 这是节奏控制，不是限制。`_lc_stream_delay` 默认 25ms；`0` 表示关闭。
- 切片大小与积压量成正比，目标延迟约 250ms，在单词边界处切分（64 字符前瞻）。
- 队列上限：256 个片段、8192 个字符。
- 工具调用/用量/元数据片段零延迟直通。混合了文本与推理或工具增量的片段整体按节奏输出，从不拆分。
- 如果生产者忽略关闭信号，1s 后放弃它。

**`dropRepeatedScalarMetadata`**（`src/llm/openai/streamMetadata.ts`）：对每个 completion index，`model_name`/`finish_reason`/`service_tier`/`system_fingerprint` 只保留首次出现的值。没有它，`_mergeDicts` 会把重复值拼接成 `"stopstop"` 这样的值。

### 4.2 LLM 层中的抢占钩子（src/llm/preempt.ts + invoke.ts）

宿主设置一个电平触发的 `context.shouldPreemptStream()`，在插话（steer）或中断时触发。每个片段由 `resolvePreemptAction` 选择动作：
- **`seal`**：累积的轮次中有可见文本（不只是推理）且没有未完成的工具调用。这里的未完成包括未得到应答的 Google 服务端工具调用（`toolCall` 数 > `toolResponse` 数）和没有结果的 Anthropic 服务端工具使用。带内置工具输出的 OpenAI Responses 轮次也视为不可丢弃。
  - 封存保留部分轮次：`response_metadata.preempted=true`，流循环跳出，`endSealedModelRun` 关闭回调并合成用量。封存从共享预算中申领（`claimPreemptSeal`，`DEFAULT_MAX_SEALS`）。
- **`restart`**：只在轮次中只有推理内容、且请求已超过 `restartGraceMs` 时使用。
  - 组合进流 signal 的 `AbortController` 中止提供商请求。
  - 丢弃该轮次：`preemptDiscarded`、`cancelOpenMessageStep`、`notePreemptRestart(lane)`。
  - 返回 `{messages: []}`，使注入的用户轮次直接接在上一个用户轮次之后。这在所有提供商上都安全；严格交替的提供商会做合并。
  - 一个自我重新调度、已 unref 的定时器会在静默推理期间重新评估。
  - “重启路径”取值为 `aborted`、`broke` 或 `exhausted` 之一，它决定 LLM 运行如何关闭。
- **`none`**：其他情况。

### 4.3 上下文溢出的检测与恢复

**检测**（`src/utils/errors.ts`，`getContextOverflowInfo(error, {provider, estimatedPromptTokens, maxContextTokens})`）：
1. `collectErrorText` 把错误的 `message/code/type/status/reason` 以及嵌套的 `error/cause/body/data/response`（深度 ≤ 4）拍平，然后去掉 URL。
2. 先执行排除过滤：
   - `NON_RECOVERABLE_RE`：达到速率限制、rpm、配额、计费、认证、禁止访问；
   - `OUTPUT_LIMIT_RE`：max_tokens 过大；
   - `AMBIGUOUS_CONTEXT_OR_OUTPUT_RE`。
3. LangChain `ContextOverflowError` 通过实例类型、`name` 或 `lc_error_code: CONTEXT_OVERFLOW` 识别。
4. 有序的正则表。每一项设置 `kind`、`limitTokens`、`requestedTokens`，只有当数字仅指提示时才设置 `promptTokens`：
   - Anthropic/Bedrock-Claude：`prompt is too long: N tokens > M maximum`
   - OpenAI：`maximum context length is M tokens. However, your messages resulted in N`（仅提示）
   - OpenRouter/DeepSeek：`…you requested (about) N`。这个总数包含补全；`PROMPT_ONLY_BREAKDOWN_RE` 从 `(X of text input` / `(X in the messages` 中恢复仅提示部分。
   - xAI：`maximum prompt length is M but the request contains N tokens`
   - Mistral：`Prompt contains N tokens … too large for model with M maximum context length`
   - Gemini：`input token count exceeds the maximum number of tokens allowed (M)`
   - OpenAI **429** `Request too large … Limit L, Requested R` 归为 `request_too_large`，但仅当 `Requested > Limit`；否则是真正的限流。
   - Bedrock Llama/Nova/Sonnet 的措辞（Nova 和 Llama 以 HTTP 200 流中途 `Error` 的形式到达）。
   - 通用的 `context_length_exceeded` 及类似措辞。
   - **Vertex** 裸的 `Google request failed with status code 400`（不带 `:`）只在存在*上下文压力*时才计入，即 `estimatedPromptTokens / maxContextTokens ≥ 0.8`。

**恢复策略**（`src/llm/contextOverflowRecovery.ts`，`planContextOverflowRecovery`）：
- 每个智能体每次运行最多恢复 2 次（`DEFAULT_MAX_OVERFLOW_RECOVERIES`）。
- 目标预算：
  - 如果提供商给出了上限：`(limitTokens − reservedCompletion) × 0.95`，其中两者都已知时 reserved = `requested − prompt`，否则用配置的 `maxTokens`。若结果 ≤ 0，则拒绝恢复；
  - 否则 `estimatedPrompt × 0.7`；
  - 否则 `maxContextTokens × 0.7`。
- 如果目标 ≥ 当前上限，则改为盲目收缩。
- 下限：`instructionTokens + 2000`，否则 4000；如果可以做摘要则为 1。低于下限或预算没有缩小时拒绝恢复。
- `observedCalibrationRatio = (providerPrompt − instr) / (estimated − instr) × currentRatio` 单独返回，避免重复校正。
- `translateRecoveryBudget` 为来自回退模型的错误在不同校准空间之间换算。

在 Graph 中：
- 可恢复的溢出会触发 `beginOverflowRecovery`，它返回一个原因为 `overflow` 的 `summarizationRequest`，而不是抛错。
- 分阶段：先在更小的预算下重新裁剪（工具输出截断和遮蔽）；只有第二次溢出且已启用摘要时才做摘要。
- 如果没有什么可以收缩，则跳过恢复：既没有 token 计数器也没有摘要，或 `overflowRecoveryStalled`。
- 只有在恢复不适用时才走回退。

**上下文压力计**（`src/llm/contextPressureMeter.ts`）：
- 用精确的、经提供商投影后的载荷，对照 `contextBudget − effectiveInstructionTokens` 进行测量，包含 `REPLY_PRIMER_TOKENS`，并按校准比例缩放。
- 每个消息对象只做一次分词（WeakMap）。来自裁剪器的、以提供商实测为依据的基线，通过 `trackProjection` 和 `trackClone` 按来源消息归属，这样一个转换带来的缩减不会掩盖另一个转换带来的增长。
- 产出 `ProviderPayloadMeasurement {fits, projectedMessageTokens, availableMessageTokens, toolMessageTokens…}`。

---

## 5. 各提供商的消息转换特殊处理（Python 移植会遇到的问题）

### Anthropic（src/llm/anthropic/utils/message_inputs.ts，内置自 @langchain/anthropic）

1. **Tool-use id** 必须匹配 `^[a-zA-Z0-9_-]+$` 且 ≤ 64 个字符。`normalizeAnthropicToolCallId` 做清理，并追加*原始* id 的 SHA-256 前 10 个十六进制字符。它是确定性的，所以 `tool_use.id` 和 `tool_result.tool_use_id` 无需映射表就能保持配对。OpenAI Responses 的 `call_…`/`fc_…` id 和 Google id 需要这一步。
2. **连续的 ToolMessage** 合并为一条由 `tool_result` 块组成的用户消息（`_ensureMessageContents` 加 `mergeMessages`）。内容已经是单个 `tool_result` 信封的 ToolMessage 会被解包。`is_error` 会被传递。
3. **`tool_use.input` 必须是对象。** `coerceAnthropicToolUseInput` 在 JSON 字符串完整时解析它，否则用 `{}`。压缩之后历史中会出现 `null` 输入。
4. **空的助手内容**变为文本占位符 `'_'`。
5. **思考块：**
   - 助手轮次上未签名的 `thinking`（例如 Google 的输出）被丢弃；
   - 缺少文本的已签名块（Opus 4.7 `display:'omitted'`）发送 `thinking: ''`；
   - 带签名的标准 `reasoning` 块被转换回 `thinking`；
   - 外来推理（`reasoning_content`、`reasoning`、`think`）被丢弃；
   - Google 的 `toolCall`/`toolResponse` 服务端片段被丢弃。
6. **只存在于 `tool_calls`** 而不在内容中的工具调用（例如 Bedrock 的思考轮次）必须物化为 `tool_use` 块。Google 的 `functionCall` 片段转换为 `tool_use`，按 id 去重。
7. **服务端工具：** id 前缀为 `srvtoolu_` 的块被重建为干净的 `server_tool_use` 和 `web_search_tool_result` 块。`input_json_delta` 块被丢弃，拼装好的输入恢复到 `tool_use` 上。
8. **思考模式要求**当前尾部中每个 tool_use 轮次都有思考块 → `ensureThinkingBlockInMessages`（见 §2.1）。
9. **助手预填充：** Claude ≥ 4.6 拒绝结尾的助手消息。将其剥掉并重新锚定缓存标记。
10. **请求校验：**
    - Opus 4.7 拒绝 `thinking.type:'enabled'`（应使用 `adaptive`）、`budget_tokens`、`topK`，以及非默认的 `topP`/`temperature`；
    - 开启思考时，不能有 `top_k`/`top_p`，temperature 必须为 1；
    - `temperature/topP/topK === -1` 表示未设置；
    - 除非显式设置，否则省略 `thinking`。
11. **Betas 根据工具类型自动添加：** `tool_search_tool_*` → `advanced-tool-use-2025-11-20`，`memory_20250818` → `context-management-2025-06-27`，`web_fetch`、`code_execution`、`computer_*`、`mcp_toolset`。压缩（`compact_20260112`）和任务预算会添加各自的 betas。
12. **流式**内容只在请求中没有工具、文档、思考或压缩时才被强制转为字符串。

### Bedrock（src/llm/bedrock/utils/message_inputs.ts，内置）

1. **所有连续的 user 角色消息**（ToolMessage 和 HumanMessage 都会变成 `user`）被合并，保持块顺序（先 toolResult 块，后文本）。空的用户消息获得 `'_'` 占位符，且只在整组合并完成后才应用。
2. 工具结果中的 **cachePoint** 被上提到同级位置（见 §3.2）。
3. **推理：** 连续的推理块被拼接。文本为空的 `reasoningText` 被丢弃，因为只有签名的块会失败。外来推理和 Google 服务端片段被丢弃。因过滤而变空的助手轮次获得 `'_'`。
4. **`toolUse.input`** 采用与 Anthropic 相同的对象强制转换。
5. 来自其他提供商的 **v1 标准内容消息**（`response_metadata.output_version==='v1'`）走单独的转换路径。
6. **流式：** Converse SDK 不提供 `index`，所以 `_mergeLists` 会追加而不是合并。SDK 的做法：
   - 读取 `contentBlockIndex`；
   - 把它提升为每个块上的 `index`；
   - 一旦见到不止一个块索引，就把文本增量提升为数组形式；
   - 从 `response_metadata` 中剥掉它。
7. 在 `contentBlockStop` 处**封存 tool-use**，仅在未配置 guardrails 时，因为 guardrails 可能在 `messageStop` 时介入。
8. **应用推理配置文件：** 临时把 `this.model` 换成 ARN，并保留配置的模型 id 用于缓存判断。
9. **文档：** 带原始字节的 `document` 块对其他提供商投影为 base64 `file` 块（`prepareProviderRequest`）。
10. **空白文本：** Bedrock 拒绝空白文本块，也拒绝工具结果之前最后一条 AI 文本末尾的空白，所以 Graph 中会做 trim。

### OpenAI / 兼容接口（src/llm/openai/utils/index.ts）

1. 对推理模型（`/\b(o\d|gpt-[5-9])\b/`），`system` 角色变为 `developer`。
2. 工具调用参数有长度上限（`projectToolCallInputs`）。工具结果用有上限的序列化器序列化（`HARD_MAX_TOOL_RESULT_CHARS`）。
3. **`reasoning_content` 回放**（DeepSeek/Moonshot）：在带工具调用的助手轮次上包含，并在第一次工具交互之后的每个轮次上包含。DeepSeek 在带工具的思考模式下缺少它会报错。
4. **`reasoning_details`**（OpenRouter/Gemini）：对非 Claude 模型作为字段发送。对经 OpenRouter 访问的 Claude，则转换为 `thinking`/`redacted_thinking` 内容块。
5. **Strict 工具：** 从 `strict:true` 工具 schema 中移除可选的 `intent` 属性，因为 strict 模式要求每个属性都出现在 `required` 中。
6. **顺序工具调用封存：** 只给官方 OpenAI 和 Azure 主机打标记。Kimi/Moonshot 会修改之前的索引。
7. **Responses API：**
   - 对 gpt-5.6/Astra 通过 `include: ['reasoning.encrypted_content']` 获取加密推理；
   - 保留回放输出项（`shell_call_output`、`apply_patch_call_output`、…）；
   - 重新映射文本块索引；
   - 把 `response.failed`/`error` 事件转成抛出的 `APIError`。
8. **GPT‑6 Astra**（仅在 `firstPartyEndpoint: true` 时）：去掉 `temperature`、`top_p`、`logprobs`、`top_logprobs` 和 `message.output_text.logprobs` include；把 effort `none`/`minimal` 映射为 `low`。

### Google GenAI / Vertex

1. **思考签名：**
   - gemini‑3 要求回放的 `functionCall` 片段带 `thoughtSignature`。GenAI 把它们存放在按工具调用 id 索引的 `__gemini_function_call_thought_signatures__` 中，对 gemini-3 回退到硬编码的 `DUMMY_SIGNATURE`。
   - Vertex（`fixThoughtSignatures`）按位置把 AI 消息与 `model` 内容对齐，重新附上 google-common 未能应用的非空签名。没有签名的 AI 消息仍然占据自己的位置，以便后面的消息对齐。
2. **ToolMessage 需要函数名。** 从之前 AI 的 tool_calls 推断，否则抛错。响应被包装为 `{result}` 或 `{error:{details}}`（必须是对象），`id` 非空时一并传递。
3. **已解析的工具调用**不会再作为 `functionCall` 内容片段发送；内容中的镜像按 id 或名称去重。
4. 系统消息变为 `systemInstruction`。**`function` 角色被改写为 `user`**；gemini-3.1+ 拒绝 `function`。
5. **预填充：** 对 Flash ≥ 3.6 和 `gemini-3.5-flash-lite` 丢弃结尾的 `model` 轮次。
6. **Schema 清理：** 递归移除 `additionalProperties`、`$schema` 和 `strict`。`properties` 为空的工具不带 `parameters` 发送。缺少描述时默认为 `'A function available to call.'`。
7. **思考配置：**
   - `thinkingBudget: -1` 表示动态：删除预算并强制 `includeThoughts`；
   - `thinkingLevel` 和 `thinkingBudget` 互斥；
   - GenAI 直接修补 `client.generationConfig.thinkingConfig`。
8. **服务端工具**（Search、URL context）以 `toolCall`/`toolResponse` 内容块到达，而不是 `tool_calls`。它们需要 `includeServerSideToolInvocations` 和 functionCalling 模式 `VALIDATED`。
9. **工具调用是原子的：** 片段被打上 `kind:'all'` 封存，截断守卫跳过 Google。

### Mistral

Mistral 需要严格交替：合并相邻的用户轮次。其余使用上游 LangChain 转换器。

---

## 6. 自定义提供商注册 API

入口是 `src/provider-registration.ts`，导出为 `@librechat/agents/provider-registration`。实现在 `src/llm/providerRegistry.ts`。

```ts
registerProvider({
  provider: 'myllm',             // non-empty, untrimmed-whitespace rejected
  model: MyChatModel,            // constructible BaseChatModel subclass
  family?: 'openai'|'anthropic'|'bedrock'|'google'|'mistral'|'generic',  // default 'generic'
  manualToolStream?: boolean,    // default false
  strictAlternation?: boolean,   // default false
}): () => void                   // disposer; only removes if still owned (Symbol owner token)
```

- **存储。** 注册表位于 `globalThis[Symbol.for('@librechat/agents:providerRegistry:v1')]`，这样重复的包副本（CJS/ESM）共享同一份。内置提供商放在另一个模块内部的 map 中，不能被覆盖：重复注册会抛出 `LLM provider already registered`。
- **延迟加载的内置提供商。** 内置提供商通过 `registerBuiltInProviderLoader` 注册，带一个使用 `requireInternalModule`（`src/lazyRequire.ts`）的 `loadModel()` thunk。提供商 SDK 在首次使用时导入，随后校验类并缓存。`src/llm/providers.eager.ts` 注册源码模式的模块，用于直接运行 TypeScript。
- **类型。** 通过声明合并 `CustomProviderOptionsMap` 来为选项提供类型。
- **`family` 为自定义提供商启用的能力：**
  - 思考检测；
  - 消息投影分支（Anthropic/Bedrock/OpenAI）；
  - 附件过滤；
  - Anthropic 类的移交提示语；
  - `maxOutputTokens` 与 `maxTokens` 键的选择；
  - google 跳过截断守卫；
  - `manualToolStream` 规范化，映射到 Anthropic/Bedrock 的规则。
- **查询函数：** `getChatModelClass(provider)`、`getProviderFamily`、`providerUsesManualToolStream`、`providerRequiresStrictAlternation`。

---

## 7. Python 重新实现指南

### 7.1 包覆盖情况

| 提供商 | Python 基础 | 已覆盖 | 必须重写 |
|---|---|---|---|
| OpenAI / Azure | `langchain-openai`（`ChatOpenAI`、`AzureChatOpenAI`、`use_responses_api=True`） | chat、Responses、推理摘要、`stream_usage` | 显式缓存断点（`prompt_cache_options`、`prompt_cache_breakpoint`）；`safety_identifier`；Astra 请求整形；加密推理 include；`reasoning`/`reasoning_details`/`provider_specific_fields` 增量捕获（子类化 `_convert_chunk_to_generation_chunk`）；`cache_write_tokens` 写入 `cache_creation`；标量元数据去重；第一方封存标记；strict 工具的 intent 剥离；system→developer（部分已原生支持） |
| DeepSeek | `langchain-deepseek` | 输出中的 `reasoning_content` | 工具轮次**输入侧**的 `reasoning_content` **回放**；`<think>` 标签流式解析器 |
| xAI | `langchain-xai` | 基础功能 | 自定义 base_url；溢出正则 |
| Moonshot | 带 base_url 的 `ChatOpenAI` | — | reasoning_content 的捕获与回放（与 DeepSeek 相同） |
| OpenRouter | 带 base_url 的 `ChatOpenAI`，或可用时使用 `langchain-openrouter` | — | `reasoning` 参数对象；`reasoning_details` 累积与最终片段刷出；Claude 的 reasoning_details → 思考块；累计回放去重；系统、尾部和工具上的 cache_control |
| Anthropic | `langchain-anthropic` `ChatAnthropic` | 思考、签名、`cache_control` 透传、betas | 工具 id 规范化；输入强制转换；丢弃外来推理；tool_result 上提与解包；剥离预填充并重新锚定缓存；Opus 4.7 校验；自动 betas；`message_delta` 增量用量（需核实 Python 的聚合方式）；`'_'` 占位符 |
| Bedrock | `langchain-aws` `ChatBedrockConverse` | Converse、推理、部分 cachePoint 支持 | 用户轮次合并；把 cachePoint 从 toolResult 中上提；工具 cachePoint 的 Claude 限定与 Nova TTL 压缩；推理配置文件 ARN；service tier；内容块索引合并修复（检查 Python 的 `ChatBedrockConverse` 是否保留 `index`）；空白文本/占位符处理；推理过滤 |
| Google GenAI | `langchain-google-genai` | 思考签名（较新版本）、thinking_budget、include_thoughts | 占位签名回退；丢弃预填充；服务端工具片段；schema 清理（`additionalProperties`/`$schema`/`strict`）；包含 thoughts 和 >200k 分桶的用量 |
| Vertex | `langchain-google-vertexai`（`ChatVertexAI`），或带 `vertexai=True` 的 `langchain-google-genai` | — | 签名重新附加；`function`→`user` 角色；thinkingLevel 与 budget；用量修复 |
| Mistral | `langchain-mistralai` | — | 只有严格交替 |
| 备选方案 | `litellm`（`ChatLiteLLM` / `litellm.acompletion`） | 线上协议覆盖面广，并有自己的 `ContextWindowExceededError` 映射 | 失去本 SDK 依赖的细粒度签名、缓存和流式控制。只用于 `generic` 家族的自定义提供商。 |

建议：
- 为了与 LangGraph 保持一致，保留 LangChain Python，但**消息转换器要自己掌控**。TS SDK 之所以内置 Anthropic、Bedrock 和 Google 转换器，正是因为上游转换器在跨提供商的历史上会出错。
- 在每个客户端上显式设置 `max_retries`。Python LangChain 依赖提供商 SDK 自身的重试（`openai`/`anthropic` 客户端默认 2 次），而 TS 依赖 AsyncCaller。

### 7.2 必须移植的逻辑（与提供商无关）

1. 带家族与特性、延迟导入和释放函数（disposer）的提供商注册表。
2. `prepare_provider_request`：投影模式检测、附件投影、按家族的工具消息投影与缓存剥离、移交提示语、交替合并，以及携带测量结果的冻结请求对象。
3. `attempt_invoke`：
   - 元数据标记；
   - 用 `+`（AIMessageChunk 相加）累积 `astream`；
   - OpenRouter 回放修正；
   - 手动工具流规范化（`modify_delta_properties`）；
   - 空名称工具调用过滤；
   - 截断守卫；
   - 通过 `run_manager.on_llm_end` 关闭已封存的运行（在 Python 中，从 `astream` 跳出同样会跳过 `on_llm_end`）。
4. `try_fallback_providers`，优先选择溢出错误。
5. 提示缓存模块：尾部标记（Anthropic 和 Bedrock 两种）、工具分区（3 种）、系统消息构建器、TTL 规则、OpenAI 显式断点。
6. 流限制：按调用键的字节计数、封存语义、按尝试的生成键、全运行范围的中止（用 `asyncio.Event` 或取消作用域代替 AbortSignal）。
7. 溢出正则表（保留 `src/utils/__tests__/fixtures/contextOverflowSignatures.ts` 中的测试夹具）和恢复规划器（纯函数；1:1 移植）。
8. 上下文压力计和抢占的封存/重启（依赖图的切片）。
9. 流平滑器：带队列和自适应切片大小的异步生成器。可选，仅用于 UI。

### 7.3 建议的模块布局

```
librechat_agents/llm/
  __init__.py
  registry.py            # ProviderFamily、register_provider、get_chat_model_class、特性
  providers.py           # 内置提供商的延迟注册（Providers 枚举 → 导入路径）
  init.py                # initialize_model（参数重新赋值、bind_tools）
  request.py             # is_thinking_enabled、resolve_model、max_output_tokens_key
  prepare_request.py     # PreparedProviderRequest、投影模式、附件
  invoke.py              # attempt_invoke、try_fallback_providers、关闭已封存运行、合成用量
  truncation.py          # OutputTruncationError、停止原因规范化
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

### 7.4 移植陷阱清单

- 片段合并依赖内容块上的 `index`。任何 Python 适配器省略了它的提供商（Bedrock）都会产生碎片化的内容数组。
- Anthropic 的 `usage_metadata` 不能在 `message_start` 和 `message_delta` 之间重复计数。
- 对话中途切换提供商时，必须剥掉缓存标记：
  - 进入 Bedrock/OpenAI 的 Anthropic `cache_control`；
  - 进入其他所有提供商的 Bedrock `{cachePoint}` 块。它们没有 `type` 键，所以要显式过滤。
- 较长的 TTL 必须位于较短的 TTL 之前（Anthropic 和 Bedrock），所以要剥掉过期的 5m 标记。
- 保持恰好一个尾部标记，且永远不要锚定在 thinking、reasoning、空文本或 `input_json_delta` 块上。
- 处理被 LangChain 包装的溢出错误（`ContextOverflowError`），包括 HTTP 429（OpenAI）和 HTTP 200 流中途失败（Bedrock Nova/Llama）。
- Gemini-3 回放函数调用时若没有思考签名，会以 400 失败。
- 来自 OpenAI Responses 的工具调用 id（`fc_…`，较长）会破坏 Anthropic 的 id 校验。
- 思考模式下带 tool_use 但没有思考块的 Claude 轮次（来自另一个智能体的历史）必须折叠为文本。

关键文件：
- `src/llm/providers.ts`、`src/llm/providerRegistry.ts`、`src/llm/init.ts`
- `src/llm/invoke.ts`、`src/llm/prepareProviderRequest.ts`
- `src/messages/cache.ts`、`src/messages/anthropicToolCache.ts`、`src/llm/bedrock/toolCache.ts`、`src/llm/openrouter/toolCache.ts`
- `src/llm/streamLimits.ts`、`src/llm/contextOverflowRecovery.ts`、`src/utils/errors.ts`
- `src/llm/{anthropic,bedrock,google}/utils/*`、`src/llm/vertexai/index.ts`、`src/llm/openai/index.ts`
- `src/graphs/Graph.ts`（约第 3480–4760 行）
