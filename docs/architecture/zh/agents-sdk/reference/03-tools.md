# @librechat/agents：工具执行与内置工具

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

范围：`ToolNode`、宿主工具执行契约、工具调用生命周期、急切执行、所有内置工具、Code API、工具搜索、程序化工具调用（PTC）、网页搜索以及子智能体。所有路径均相对于 SDK 仓库根目录。

---

## 1. ToolNode 架构与宿主契约

### 1.1 ToolNode 是什么
`src/tools/ToolNode.ts`（6.3k 行）包含 `class ToolNode<T> extends RunnableCallable<T, T>`。它是 LangGraph 预置 ToolNode 的自定义替代品。

**接受的输入：**
- `BaseMessage[]`
- `{ messages: BaseMessage[] }`（消息状态）
- LangGraph `Send` 负载 `{ lg_tool_call, ...state }`

**返回的输出：**
- 与收到的输入形状相同，包含若干 `ToolMessage` 以及所有注入的 `HumanMessage`，或者
- `Command` 对象（用于移交）。

**构造函数输入** 为 `t.ToolNodeConstructorParams`，在 `src/types/tools.ts` 中定义为 `ToolRefs & ToolNodeOptions`。最重要的字段：
- `tools` / `toolMap`
- `eventDrivenMode`、`toolDefinitions: Map<string, LCTool>`、`directToolNames`
- `toolRegistry: Map<string, LCTool>`（用于 PTC 和工具搜索）
- `sessions: ToolSessionMap` 和 `codeSessionKey`，即共享的代码会话存储（默认键为 `execute_code`）
- `toolCallStepIds: Map<toolCallId, stepId>`，由流处理器填充；用于路由完成事件
- `errorHandler(data, metadata) => Promise<boolean|void>`
- `handleToolErrors`（默认 true）
- `hookRegistry`、`humanInTheLoop`
- `maxToolResultChars` / `maxContextTokens`
- `toolOutputRegistry` / `toolOutputReferences`
- `toolExecution`（引擎 `sandbox | local | cloudflare-sandbox`）
- `eagerEventToolExecution*`，由 Graph 持有的共享映射
- `interruptingToolNames`
- `getBreakerSignal` / `getRunScope`
- `preparedSubagents`
- `executionContext`（子智能体谱系）

**Graph 如何构建它**（`src/graphs/Graph.ts` 约 2837-2930）：
- 当 `agentContext.toolDefinitions` 非空时，启用事件驱动模式。
- Graph 用 `createSchemaOnlyTools`（`src/tools/schema.ts`）构建仅含模式的桩工具。调用桩工具会抛出 "should not be invoked directly in event-driven mode"。桩工具的 `responseFormat` 默认为 `content_and_artifact`。
- 每个进程内的“图工具”都会加入工具映射，并自动标记为 **direct（直接执行）**：移交工具 `lc_transfer_to_*`、`subagent` 工具等。
- 当 `toolExecution.engine` 为 `local` 或 `cloudflare-sandbox` 时，`applyToolExecutionOverrides()`（`src/tools/local/resolveLocalExecutionTools.ts`）把 Code API 工具替换为本地或 Cloudflare 实现。这些名称也会加入 `directToolNames`，因此即使在事件驱动模式下也在进程内运行。

### 1.2 两条执行路径
1. **直接路径。** `runDirectBatchInterruptSafe` → `runDirectToolWithLifecycleHooks` → `runTool` → `invokeWithRuntime` → `tool.invoke(invokeParams, runtime)`。
   - `invokeParams` 为 `{...call, args, type:'tool_call', stepId, turn: usageCount}`，外加一些特殊注入字段。
   - LangChain 把不属于模式的字段移到 `config.toolCall` 中，工具正是通过它读取注入的上下文。
   - runtime 对应 LangGraph 1.4 的 `ToolRuntime`：`{...config, state, toolCallId, config, context, store, writer}`。
2. **事件路径。** `dispatchToolEvents` 构建一个 `ToolExecuteBatchRequest`，并通过 `safeDispatchCustomEvent`（`src/utils/events.ts`）把它作为 LangChain 自定义事件 `on_tool_execute`（`GraphEvents.ON_TOOL_EXECUTE`）发出。宿主的处理器（注册在 Run 的 `HandlerRegistry` 上；`ON_TOOL_EXECUTE` 位于 `src/run.ts` 的 `CUSTOM_GRAPH_EVENTS` 中）运行这些工具并调用 `resolve(results)`。

**混合批次**（见约 5332 行的 `run()`）：
- 调用被拆分为直接条目和事件条目。直接指名称在 `directToolNames` 中，或者在注册了移交工具时，它是一个未知的 `lc_transfer_to_*` 移交。
- 事件参数中的占位符**在任何 await 之前同步**解析，依据的是工具输出注册表的一份冻结快照。
- 直接条目先运行。这带来快速失败的行为：抛出的错误或 `GraphInterrupt` 会在任何宿主分发之前中止。事件在之后分发。
- 输出顺序是固定的：
  `[promotedAiMessage?] + directOutputs + eventToolMessages + invalidCallResults + directInjected(HumanMessage) + eventInjected`

### 1.3 `ON_TOOL_EXECUTE` 请求：`ToolExecuteBatchRequest`（`src/types/tools.ts`）

```ts
{
  executionContext?: SubagentExecutionContext; // child lineage; unset for root graph
  toolCalls: ToolCallRequest[];
  userId?: string;              // config.configurable.user_id
  agentId?: string;             // executingAgentId (owning agent; host keys tool/cred lookup on it)
  callerCapabilityProjection?: { version:1; directToolNames[]; codeExecutionToolNames[];
                                 directOnlyToolNames[]; codeExecutionOnlyToolNames[] };
  configurable?: Record<string,unknown>; // RunnableConfig.configurable minus SDK-private keys
                                         // (approval review, TOOL_BATCH_REPLAY_KEY, RUN_BREAKER_SCOPE)
  metadata?: Record<string,unknown>;     // thread_id, run_id, provider...
  signal?: AbortSignal;                  // run cancel + breaker composed
  resolve(results: ToolExecuteResult[]): void; // REQUIRED: all results
  reject(error: Error): void;                   // fatal
  onResult?(result: ToolExecuteResult): void;   // optional per-call early completion;
                                                // only offered when no result-altering hooks and HITL off
}
```

`ToolCallRequest`：
```ts
{ id: string; name: string; args: Record<string,unknown>;   // schema-coerced
  stepId?: string; turn?: number;                            // per-tool usage index
  codeSessionContext?: { session_id: string; files?: CodeEnvFile[] }; // code tools + skill + read_file
  codeSessionBaselineId?: string; retainCodeSessionInputs?: true;
  runtimeSessionHint?: string }                              // only when sandbox.statefulSessions
```

`CodeEnvFile` 为 `{ id, resource_id, name, storage_session_id, kind: 'skill'|'agent'|'user', version? }`。当 `kind` 为 `skill` 时 `version` 为必填。

**Promise 语义。** 只有在以下**两件事**都发生之后，promise 才会 resolve：
- 分发已结束（自定义事件已返回），并且
- `resolve` 已被调用。

如果分发抛出异常，promise 会 reject。`safeDispatchCustomEvent` 包装了 `resolve`，以便先记录 Langfuse 宿主结果追踪。

### 1.4 响应：`ToolExecuteResult`
```ts
{ toolCallId: string; content: string | unknown[]; artifact?: unknown;
  status: 'success' | 'error'; errorMessage?: string;
  outcome?: string; outcome_patch?: { from: string; to: string };   // settled UI label
  injectedMessages?: { role:'user'|'system'; content: string|MessageContentComplex[];
                       isMeta?: boolean; source?: 'skill'|'hook'|'system'|'steer'; skillName?: string }[] }
```

**每个结果如何变成消息：**
- `injectedMessages` 由 `convertInjectedMessages` 转换为 `HumanMessage`；原始角色保存在 `additional_kwargs.role` 中。它们追加在所有 `ToolMessage` **之后**，从而使每个 tool_call 与其 tool_result 保持相邻。
- 如果宿主为代码工具或 `skill` 返回了代码会话产物（`artifact.session_id`、`files`、`deleted_files`），它们会合并到 `sessions` 中（见 §4.4）。
- **错误：** `ToolMessage{status:'error', content: "Error: <errorMessage>\n Please fix your mistakes."}`，会被截断。
- **成功：** 内容在 `maxToolResultChars` 范围内序列化；`artifact` 原样透传。

### 1.5 完成事件
对每个结果，ToolNode 发出 `ON_RUN_STEP_COMPLETED`：

```ts
{ result: { id: stepId, index: turn, type:'tool_call',
            tool_call: { args: <bounded serialized args>, name, id, output: <content string>,
                         progress: 1, outcome? },
            completed_at: Date.now(), eager?: true } }
```

**去重。** 如果该事件已经由以下任一方发出，则跳过：
- 提前的 `onResult` 快速路径，
- 流的急切分发器（`record.completionDispatched`），或
- `errorHandler`（在 `ToolErrorOwnership` 中跟踪）。

`handleRunToolCompletions` 对直接输出做同样的事，并额外根据 `ToolMessage.artifact` 更新代码会话。

---

## 2. 工具调用生命周期

### 2.1 批次入口（`run()`）
1. 组合信号：`config.signal` 加上图的熔断信号。如果它已因 `StreamLimitExceededError` 或 `PreparedSubagentError` 被中止，立即抛出。
2. 生成 `batchScopeId`：即 run_id，没有时为 `\0anon-N`。从工具输出注册表认领 `turn`（`nextTurn`）。
3. 找到最后一条 AIMessage（`findAssistantBatch`）。过滤掉：
   - 已经有 ToolMessage 的调用（恢复 / 重放），以及
   - Anthropic 服务端工具调用（id 前缀 `srvtoolu_`）。
4. **无效工具调用。** `aiMessage.invalid_tool_calls` 保存流式参数始终无法解析的调用。
   - 每个调用都会得到一条合成的错误 ToolMessage："Error: Malformed args. The tool call input could not be parsed as a JSON object; the tool was not run."
   - 发出一条**具有相同 id 的替换 AIMessage**，把这些调用以 `args:{}` 提升到 `tool_calls` 中。归约器按 id 更新插入它。
   - 其内容中损坏的 tool_use 块会被清理（`sanitizeInvalidToolUseBlocks`）。
   - 这只针对带 id 的 AI 消息的消息状态输入进行，从而使每种提供商转换器看到的调用和结果都保持成对。
   - 该提升也会被修补进任何移交 `Command.update`（`patchCommandUpdateForPromotedInvalidCalls`）。
5. `loadRuntimeTools(tool_calls)` 可以在每个批次重建工具映射。

### 2.2 参数校验与强制转换
在事件路径上，**SDK 不做完整的模式校验**，由宿主负责校验。

`buildToolExecutionRequestPlan`（`src/tools/eagerEventExecution.ts`）做三件事：
- `coerceRecordArgs`：字符串参数会被 JSON 解析。任何不是对象的结果都变成被拒绝的结果 "Invalid tool call arguments: expected a JSON object."（`invalidArgsBehavior:'error-result'`）。
- `coerceArgsForSchema`：只做无损的、由模式引导的修复。
  - 整数字符串变为整数；数字字符串只有在能精确往返时才变为数字。
  - `"true"`/`"false"` 变为布尔值。
  - 强制转换会递归进入数组、属性和 `additionalProperties`。
- 按工具预留 `turn`：`usageCount[name]++`。同一计数会镜像到急切执行的用量计数器中。

在直接路径上，LangChain `tool()` 的 Zod/JSON 模式校验在 `tool.invoke` 内部运行。校验错误通过 catch 路径暴露。

### 2.3 钩子与 HITL（两条路径都适用）
- **PreToolUse** 钩子对每个调用并行运行。
  - `deny` 产生一条被阻止的 `ToolMessage{status:'error', content:"Blocked: <reason>"}`，并触发 `PermissionDenied` 钩子。
  - `updatedInput` 改写参数，占位符会对照批次前的快照重新解析。
  - `ask`：当 `humanInTheLoop.enabled` 为 false 时，退化为 deny。为 true 时，发起一次 LangGraph `interrupt()`，负载如下：

    ```ts
    { type:'tool_approval', hook_session_id?, action_requests:[{tool_call_id,name,arguments,description?}],
      review_configs:[{action_name, tool_call_id, allowed_decisions: ['approve','reject','edit','respond']}] }
    ```

    恢复时接受数组（按顺序）或以 tool_call_id 为键的映射。
    - `approve`：执行。
    - `reject`：阻止。
    - `edit`：要求 `updatedInput` 是对象。
    - `respond`：要求字符串 `responseText`；它成为一个合成的成功结果，工具永远不会运行。
    - 未知或缺失的决策，或不在 `allowed_decisions` 内的决策，默认拒绝。
  - 被阻止调用的副作用（完成事件和 `PermissionDenied`）会推迟到 `interrupt()` 之后，因此只触发一次。
- **PostToolUse** 可以替换输出（`updatedOutput`）。**PostToolUseFailure** 仅观察。**PostToolBatch** 按模型给出的顺序接收条目。
- 所有 `additionalContext` 字符串汇集到**一条** `HumanMessage{additional_kwargs:{role:'system', source:'hook'}}` 中，追加在工具消息之后。钩子的 `injectedMessages` 放在它之后。
- 恢复安全：`toolBatchReplay.ts` 和 `SubagentReplay.ts` 按批次键持久化已完成的直接结果，批次键为 `sha256(stableStringify(tool_calls))` + 线程 + AI 消息 id。重放时复用它们而不是重新执行。提议发生变化时默认拒绝。

### 2.4 并行
- **事件路径：** 整个已批准的集合在一次请求中发给宿主；并发由宿主决定。已在进行中的急切结果通过 `Promise.allSettled([eager, dispatch])` 等待。
- **直接路径：** 对所有调用执行 `Promise.allSettled`，然后重新检查熔断器。
  - 如果有调用在 `interruptingToolNames` 中（例如 `ask_user_question`），它们作为单独一组先运行并被等待。非中断型（可能非幂等）的同级工具只有在确认没有发起中断之后才开始。
  - 该组抛出的 `GraphInterrupt` 会附带一份子智能体恢复清单后重新抛出。

### 2.5 错误
- 未知工具：`Tool "x" not found.`。对于移交，还会追加 `Did you mean "lc_transfer_to_y"? Handoff tool names must match exactly.`
- 直接路径异常（启用 `handleToolErrors` 时，默认为 true）：
  - `GraphInterrupt`、`StreamLimitExceededError` 和 `PreparedSubagentError` 会被重新抛出。
  - 否则调用 `errorHandler`。如果它无法分发完成事件，返回 `false`。
  - 错误变为 `ToolMessage{status:'error', content:"Error: <msg>\n Please fix your mistakes."}`。
- 未能解析的占位符引用会被写入 `additional_kwargs._unresolvedRefs`。

### 2.6 截断（`src/utils/truncation.ts`、`src/utils/toolContent.ts`）
**限制：**
- `maxToolResultChars` = 显式指定的值，否则为 `min(floor(maxContextTokens*0.3)*4, 400_000)`。默认值为 `HARD_MAX_TOOL_RESULT_CHARS = 400_000`。

**`truncateToolResultContent`（头部 / 尾部）：**
- 头部占预算的 70%，尾部占 30%。
- 截断标记为 `\n\n… [truncated: N chars exceeded M limit] …\n\n`。
- 如果 200 个字符以内有换行符，就对齐到换行符。
- 如果可用字符少于 200 个，只保留头部。
- 它从不拆分 UTF-16 代理对。

**其他函数：**
- `serializeStructuredValueBounded`：对非字符串输出做有界的 JSON 序列化。它从不调用 `toJSON` 或 getter，强制深度上限 200 和 1M 的工作量上限，并处理循环引用和 bigint。它还会为输出引用注册表返回一个精确的前缀。
- `compactToolContent`：在工具返回自己的 ToolMessage 时使用。纯文本或结构化数组变成一个有界字符串。在混合媒体数组中，原子块（image/file/audio...）保持完整，文本被压缩。
- 计算机使用（computer-use）的截图输出原样透传。
- 完成事件中的工具调用参数用 `serializeToolContentBounded` 限制长度。

### 2.7 产物与 `content_and_artifact`
- 声明了 `responseFormat:'content_and_artifact'` 的工具返回 `[content, artifact]`。LangChain 构建一个带 `.artifact` 的 `ToolMessage`，ToolNode 将其透传（压缩之后）。
- 产物携带：
  - 代码会话：`{session_id, files, deleted_files, artifact_delivery, runtime_session_id, runtime_status}`
  - 网页搜索：`{web_search: data, outcome?}`
  - 工具搜索：`{tool_references, metadata}`
  - 最终标签：`outcome` / `outcome_patch`，由 `readOutcomeFields` 读取
- 输出引用（可选，`toolOutputReferences.enabled`）：
  - 每个成功的输出以原始形式存储在 `tool<idx>turn<turn>` 下：每个输出最多 400 KB，总计 5 MB，先进先出淘汰。
  - 字符串参数中的 `{{toolXturnY}}` 会在调用前被替换。
  - LLM 通过惰性标注看到 `[ref: key]` 前缀或 `_ref` JSON 键。
  - 存储前会去掉代码会话的 "Generated files:" 摘要。

### 2.8 意图标签（`src/tools/intentArg.ts`）
大多数内置模式把一个可选的 `intent` 字符串放在第一位（`INTENT_PROPERTY`；其描述以 `ALWAYS write this field FIRST` 开头）。因为它排在第一，所以最先流式输出，并成为实时的 UI 标签。

工具在使用前会去掉它。`resolveToolOutcome` 根据 `outcome` 或 `outcome_patch` 计算最终标签，限制为单行 256 个字符。

### 2.9 急切（推测）执行
注意：`src/llm/providers.eager.ts` 与此无关。它只是源码模式下惰性注册提供商模块。急切执行逻辑位于 `src/stream.ts` 和 ToolNode 中。

**允许的条件**（`isEagerToolExecutionEnabledForBatch`），以下各项必须全部满足：
- `eagerEventToolExecution.enabled`
- 智能体有 `toolDefinitions`（事件模式）
- 没有改变结果的钩子
- HITL 关闭
- 调用不在 PTC 元数据中
- 存在 `ON_TOOL_EXECUTE` 处理器

**按调用排除：**
- `excludeToolNames`
- 运行级的抑制集合
- 宿主的 `codeSessionToolNames`
- 当 `statefulSessions` 开启或使用了非默认的 `codeSessionKey` 时的代码工具

**批次级回退：** 如果批次中有任何直接 / 图工具，或者出现任何 `{{tool…}}` 引用，就跳过急切执行。

**调用何时开始：**
- 当最后一个数据块带有结束原因信号时，
- 当调用到达时就已封存（Google），或者
- 当流式调用在流中途被**封存**时，这发生在：
  - 后一个索引开始（Anthropic、OpenAI chat 的顺序适配器），或者
  - 适配器在 `response_metadata.lc_streamed_tool_call_seal` 中发出封存信号（OpenAI Responses `.done`、Bedrock `contentBlockStop`）。

只有当累积的参数文本可证明是规范形式时，被封存的调用才会开始：其长度等于原始拼接长度，或者等于适配器给出的权威复述。`docs/eager-tool-readiness-benchmark.md` 对这种“先封存”的就绪检查做了基准测试：在大参数下处理器耗时降低 28–51%。

**机制：**
- 急切分发发送一个形状相同的独立 `ToolExecuteBatchRequest`，并把记录存到 `graph.eagerEventToolExecutions[callId] = {toolName, args, request, promise}`。
- 完成时，流发出带 `eager:true` 的 `ON_RUN_STEP_COMPLETED`。
- ToolNode 随后消费每条记录（`takeMatchingEagerEventExecution`）。它比较名称加 `stableStringify(args)`。
- **不匹配时：** 返回错误 "Tool call changed after eager execution started; refusing to re-run the tool to avoid duplicate side effects."。它还会把两个名称都加入抑制集合，使它们在本次运行中不再被预启动。这个熔断机制防止了重试循环（LibreChat#14371）。
- PreToolUse 的 deny 会移除急切记录，从而不会泄漏成功完成事件。
- 前台子智能体有自己的预启动：`prestartSubagent` + `PreparedSubagents`，`maxPendingSubagents` 默认为 4。

---

## 3. 内置工具目录

除非另有说明，几乎所有内置模式都有 `intent` 字段。

| 名称（`Constants`） | 文件 | 输入模式 | 行为 / 外部调用 |
|---|---|---|---|
| `calculator` | `src/tools/Calculator.ts` | `{input: string}`（旧版 `Tool`，没有 intent） | `mathjs.evaluate(input).toString()`，惰性加载。出错时返回 "I don't know how to do that." |
| `execute_code` | `src/tools/CodeExecutor.ts` | `{lang: enum[py,js,ts,c,cpp,java,php,rs,go,d,f90,r,bash], code: string, args?: string[]}`；lang 和 code 必填 | POST `{base}/exec`；返回 content_and_artifact（§4） |
| `bash_tool` | `src/tools/BashExecutor.ts` | `{command: string, args?: string[]}` | POST `/exec`，带 `lang:'bash'`。附加了工作区时，POST `/exec/programmatic`，带 `tools:[]` 和请求头 `X-LibreChat-Code-Workspace-ID`。可选地在描述后追加 `BashToolOutputReferencesGuide`。 |
| `run_tools_with_code`（PTC） | `src/tools/ProgrammaticToolCalling.ts` | `{code: string(minLength 1), tool_manifest?: string[] unique, timeout?: integer ms [1000..max(cap,300000)], default cap}` | 在 Code API 沙箱中运行 Python，往返循环（§5.2） |
| `run_tools_with_bash` | `src/tools/BashProgrammaticToolCalling.ts` | 形状相同；code 为 bash | 同样的循环，带 `lang:'bash'`。工具变成接收 JSON 字符串的 bash 函数。代码前缀为 `: &\nwait "$!"`。结果做 JSON 规范化。 |
| `tool_search` | `src/tools/ToolSearch.ts` | `{query?: string(≤200, default ''), fields?: ('name'|'description'|'parameters')[] default [name,description], max_results?: 1..50 default 5, mcp_server?: string|string[]}` | 本地 BM25，或在 Code API `/exec` 上运行的 JS 正则脚本（§5.1） |
| `web_search` | `src/tools/search/*` | `{query, date?: 'h'|'d'|'w'|'m'|'y', country?: string (serper/tavily only), images?, videos?, news?: boolean}` | §6 |
| `read_file`（远程定义） | `src/tools/ReadFile.ts` | `{path}` | 仅有定义（`responseFormat: content_and_artifact`），由**宿主**执行。技能文件使用 `{skillName}/{path}`；代码输出使用其返回的路径。 |
| `skill` | `src/tools/SkillTool.ts` | `{skillName: string, args?: string}` | 仅有定义，由宿主执行。宿主返回 `injectedMessages`（来源 `skill`）。`skillCatalog.ts` 格式化 "Available Skills" 提示。 |
| `subagent` | `src/tools/SubagentTool.ts` + `Graph.ts` | `{description, subagent_type: enum(types), run_in_background?: boolean, subagent_thread_id?: string}` | §7 |
| `lc_transfer_to_<agent>` | graphs | 移交 | 返回 `Command(graph: PARENT, goto)`。同一批次中的多个移交会变成并行的 `Send`，并带有 `handoff_parallel_siblings`、`__handoff_parallel_batch`、`__handoff_group_id` 标记。 |
| 本地编码套件：`read_file`、`write_file`、`edit_file`、`grep_search`、`glob_search`、`list_directory`、`compile_check`，外加本地的 `bash_tool`、`execute_code` 和两个 PTC 运行器 | `src/tools/local/*` | read `{path, offset?, limit?}`；write `{path, content}`；edit `{path, old_text?, new_text?, edits?: [{old_text,new_text}]}`（每个 old_text 必须唯一）；grep `{pattern, path?, glob?, max_results?}`；glob `{pattern, path?, max_results?}`；list `{path?}` | 在 `workspace.root` 中以子进程或 fs 方式运行。见 §4.5。 |
| Cloudflare 套件（名称相同） | `src/tools/cloudflare/*` | 相同 | 通过结构化运行时调用 `sandbox.exec` / `readFile` / `writeFile`（§4.6） |

所有代码工具共享 `CODE_EXECUTION_TOOLS = {execute_code, bash_tool, run_tools_with_code, run_tools_with_bash}`（`src/common/enum.ts`）。这些工具参与共享的代码会话。

---

## 4. 代码执行：Code API 契约

### 4.1 配置
- **基础 URL：** `baseUrl` 参数，否则为环境变量 `LIBRECHAT_CODE_BASEURL`，否则为 `https://api.librechat.ai/v1`。端点通过 `buildCodeApiEndpoint(base, route)` 构建。
- **超时：** 环境变量 `CODE_API_RUN_TIMEOUT_MS`，即 PTC 每次运行的上限。默认 15000，最小 1000。
- **代理：** 环境变量 `PROXY`，遵守 `NO_PROXY`；根据协议选择 `https-proxy-agent` 或 SOCKS agent。
- **其他环境变量：** `PTC_DEBUG=true`。
- **认证：** SDK **没有内置的 API 密钥环境变量**。认证来自 `authHeaders`：一个静态映射，或者每次请求时解析的同步 / 异步函数。请求头解析失败会产生 "Code execution is not authorized…"。

**通用请求头：**
- `Content-Type: application/json`
- `User-Agent: LibreChat/1.0`
- `X-CodeAPI-Expected-Profile: default|stateful`（设置了 `executionProfile` 时）
- `X-LibreChat-Code-Request-ID: <uuid>`（PTC，存在信号时）
- `X-LibreChat-Code-Workspace-ID`（附加了工作区时）

### 4.2 端点
1. **`POST {base}/exec`**，请求体：
   ```json
   { "lang": "py", "code": "...", "args": ["..."]?, "files": [CodeEnvFile]?, "session_id"?: "...",
     "runtime_session_hint"?: "...", "timeout"?: 15000, "user_id"?: "...", "workspace_instance_id"?: "<64 hex>" }
   ```
   - 工厂参数（`session_id`、`user_id`、`files`）会展开进请求体。
   - `intent` 以及任何由模型提供的 `runtime_session_hint` 或 `workspace_instance_id` 都会被**去除**。
   - 只有当 `statefulSessions === true` 且 `executionProfile !== 'default'` 时才发送 `runtime_session_hint`。
   - 该提示来自工厂；否则来自 `config.toolCall._runtime_session_hint`，后者由 ToolNode 从 `toolExecution.sandbox.runtimeSessionHint` 注入，没有时使用 `thread_id`。

   响应（`ExecuteResult`）：
   ```json
   { "session_id": "exec-session", "stdout": "", "stderr": "", "files": [{"id","name","path"?,"storage_session_id"?,"resource_id"?,"kind"?,"version"?,"inherited"?:true}],
     "deleted_files": ["path"], "artifact_delivery": {"code":"artifact_delivery_failed","status":"partial|failed","attempted","delivered","failed"},
     "runtime_session_id"?: "...", "runtime_status"?: "new|reused" }
   ```
2. **`POST {base}/exec/programmatic`**，PTC 启动请求：
   ```json
   { "code": "...", "lang"?: "bash", "tools": [LCTool{name,description,parameters}], "session_id"?, "timeout": ms,
     "files"?: [...], "runtime_session_hint"?, "workspace_instance_id"? }
   ```
   续接请求：`{ "continuation_token": "...", "tool_results": [{call_id, result, is_error, error_message?}] }`。
   响应（`ProgrammaticExecutionResponse`）：
   `{status:'tool_call_required', continuation_token, tool_calls:[{id, name, input}]}`，或 `{status:'completed', stdout, stderr, files, deleted_files, artifact_delivery, session_id, runtime_*}`，或 `{status:'error', error, stderr}`。
3. **`POST {endpoint}/cancel`**，带 `{request_id}`：尽力而为，超时 2 秒，在信号中止时发送。
4. **`GET {base}/files/{session_id}?detail=full[&kind=&id=&version=]`**：`fetchSessionFiles`。返回 `[{id, name, metadata:{'original-filename'}, resource_id, storage_session_id}]`。执行器原先回退到它的旧逻辑已被移除；现在执行器依赖注入的文件。

**错误映射**（`buildCodeApiHttpErrorMessage`）：
- 读取错误响应体时有上限：64 KB、1 秒。
- 能力类错误码（`bridge_worker_mismatch`、`execution_profile_mismatch`，...）产生永久性的 "not supported… Do not retry" 消息。
- 429 产生限流消息，使用 `retry_after_seconds`（上限 3600）。
- 401/403：未授权。400/422：请求被拒绝。其他情况："temporarily unavailable"。
- 工具错误以 `CodeApiRequestError("Execution error:\n\n…")` 抛出。ToolNode 把它们转换为错误 ToolMessage。

### 4.3 输出格式
- 内容：`stdout:\n…\n`（或 "stdout: Empty. Ensure you're writing output explicitly."）加上 `stderr:\n…`。
- 会追加以下提醒：
  - 当代码引用 `/tmp` 时，提醒 `/tmp` 只是临时空间。
  - 失败时，写入 `/mnt/data` 的文件没有被登记。
  - 存在产物投递警告时，附上该警告。
- 对非继承的文件，追加一段 "Generated files:\nSession files: N persisted file(s)… including K image(s)…" 摘要（`src/tools/CodeSessionFileSummary.ts`）。
- 产物：`{session_id, files?, artifact_delivery?, deleted_files?, runtime_session_id?, runtime_status?}`。

### 4.4 会话模型
有两个不同的 id：
- **执行 `session_id`**：临时的，每次沙箱运行一个。
- **每个文件的 `storage_session_id`**：文件所在的对象存储桶。

把两者混用会导致后续调用返回 404。

**会话状态存放位置。** `Graph.sessions` 是一个 `Map`。在 `codeSessionKey` 下，它保存 `{session_id, files: FileRef[], lastUpdated}`。

**调用之前：**
- 直接路径：ToolNode 把 `session_id` 和 `_injected_files`（由 `toInjectedFileRef` 构建的 `CodeEnvFile[]`：kind 默认为 `user`，`resource_id` 默认为 `id`，`storage_session_id` 默认为执行 id）注入 `invokeParams`。
- 事件路径：同样的数据放入 `codeSessionContext`。

**调用之后**（`updateCodeSession`）：
- 文件按标识 `(storage_session_id, id)` 合并。
- 新文件会替换同名的已有文件。
- **只有**当文件列在 `deleted_files` 中，**并且**其标识与注入该请求的基线一致时，才会被移除。`files` 中缺失某个文件不视为删除。
- `inherited` 回显永远不会让已删除的文件复活。
- `retainCodeSessionInputs` 即使在执行没有返回产物时，也会保留宿主刷新的输入。

### 4.5 本地引擎（`src/tools/local/*`）
- **执行环境：** `ExecutionWorld{spawn, fs, sandboxed}`。默认使用 Node `child_process` 加 `fs/promises`，可通过 `local.exec` 覆盖。
- **限制与路径：** 默认超时 60 秒；最大输出 200k 字符；流式输出达到 50 MiB 时强制终止。路径在 realpath 之后被限制在 `workspace.root` 和 `additionalRoots` 内。
- **`execute_code` 运行时：** 把代码写入临时文件，然后运行 `python3`、`node`、`npx --no-install tsx`、`php`、`go run`、`rustc`、`gcc`/`g++`、`javac && java`、`Rscript`、`ldc` 或 `gfortran`。
- **Bash 校验**（`validateBashCommand`）：
  - 用正则阻止破坏性模式和嵌套 shell 技巧，
  - `bash -n` 语法检查，
  - 可选的 tree-sitter AST 检查（`bashAst: off|auto|strict`），
  - `readOnly` 模式阻止修改操作，
  - 可选的 `@anthropic-ai/sandbox-runtime` 包装。
- **输出** 增加 `exit_code`、`timed_out`、`killed`、`full_output_path` 和 `working_directory` 行。产物为 `{session_id:'local', files:[]}`。
- **`edit_file` / `write_file`：** 可选的 `FileCheckpointer`（启用 `Run.rewindFiles()`）以及编辑后的语法检查（`off|auto|strict`：`node --check`、`py_compile`、`JSON.parse`、`bash -n`）。
- **`grep_search`：** ripgrep，回退到带 ReDoS 防护和 5 秒预算的正则实现。
- **`read_file`：** 可选的图片 / PDF 附件（`attachReadAttachments`）。

**本地 PTC**（`LocalProgrammaticToolCalling.ts`）：
- 在 `127.0.0.1:<random>/tool` 上启动一个 HTTP 桥，在请求头 `x-librechat-bridge-token` 中携带 32 字节令牌（以常量时间比较）。
- 生成一个 Python 程序，每个工具对应一个 `async def <normalized_name>(**kwargs)` 桩函数。每个桩函数 POST `{name, input}`，并在 `is_error` 时抛出异常。Bash 桩函数使用带 `?mode=text` 的 curl。
- 每个桥请求先运行 PreToolUse 钩子（deny/updatedInput），再运行 `executeTools`。

### 4.6 Cloudflare（`src/tools/cloudflare/*`、`docs/cloudflare-sandbox-tools.md`）
- **运行时接口：** `exec`、`readFile`、`writeFile`、`mkdir`、`listFiles`、`deleteFile`。
- **设置：** `workspaceRoot` 默认为 `/workspace`。`codingToolNames` 可以限制套件范围，例如 `CLOUDFLARE_BASH_CODING_TOOL_NAMES` 不含 `execute_code` 和 `run_tools_with_code`。
- **HTTP 桥适配器**（`createCloudflareBridgeRuntime({baseURL, apiKey, sandboxId})`）：
  - `Authorization: Bearer`
  - `POST /v1/sandbox` → `{id}`
  - `POST /v1/sandbox/:id/exec`，带 `{argv:[shell,'-lc',cmd], cwd, timeout_ms}`（流式响应）
  - `GET/PUT /v1/sandbox/:id/file/<path>`
  - `mkdir`、`list` 和 `delete` 用 bash 实现。
- **PTC** 生成编码工具的 Python 或 JS “原生”实现，在沙箱**内部**运行。只能调用内置的编码工具；没有宿主桥。

---

## 5. 工具搜索与程序化工具调用

### 5.1 延迟加载工具与 `tool_search`
- **工具元数据**（`LCTool`）：`defer_loading: true` 使工具不出现在初始绑定中。`allowed_callers: ('direct'|'code_execution')[]` 默认为 `['direct']`。
- **活跃绑定：** `AgentContext.getToolsForBinding()` 绑定允许 `direct` 调用方且处于活跃状态的工具，活跃指 `!defer_loading || discoveredToolNames.has(name)`。在事件模式下，这些是由 `getActiveToolDefinitions()` 构建的仅含模式的工具，外加图工具。
- **注册表注入：** ToolNode 为 `tool_search` 把 `toolRegistry` 注入 `invokeParams`。

**搜索模式**（`createToolSearch({mode})`）：
- `local`：BM25（`okapibm25`，k1=1.5，b=0.75）。
  - 文档由名称（重复两次）、描述以及可选的参数名组成。
  - 分词按非字母数字字符和 camelCase 切分，并合并单字母的缩写片段。
  - 标识符优先级会覆盖得分：完全匹配完整名称（4），完全匹配去掉 `_mcp_server` 的基础名称（3），完全匹配连接后的 token 标识符（2），前缀匹配（1，得分 ≥0.95）。
  - 空查询按字母顺序返回工具。
- `code_interpreter`（默认）：正则搜索。
  - 模式会被清理：嵌套量词、分组深度超过 5 或其他危险结构，或者无效的正则，都会导致它被转义为字面量（并附带说明）。
  - 生成的 JS 脚本通过 `POST /exec {lang:'js', code, timeout:5000}` 运行，**不带认证请求头**。
  - 得分：名称 0.95，描述 0.75，参数 0.60。

**过滤与命名：**
- `onlyDeferred` 默认为 true。
- `mcp_server` 解析：先精确匹配，再做规范化匹配（小写、NFC、`[-_. ]+` 变为 `-`），再去掉 `mcp-` 前缀匹配。有歧义时返回所有候选。
- MCP 工具名采用 `tool_mcp_server` 形式（`Constants.MCP_DELIMITER='_mcp_'`）。
- 描述中嵌入按服务器分组的延迟加载工具列表（`name(…)` 标记带参数的工具）。

**输出：**
- 内容（JSON）：`{found, tools:[{name, score, matched_in, snippet}], total_searched, query, notes?}`。
- 产物：`{tool_references:[{tool_name, match_score, matched_field, snippet}], metadata:{total_searched, pattern, error?, available/unmatched/idle_mcp_servers?}}`。
- 服务器列表模式（带服务器过滤、查询为空）只是预览，不会发现任何工具。

**发现如何变成绑定：**
1. 每次模型调用之前，`Graph` 运行 `extractToolDiscoveries(messages)`（`src/messages/tools.ts`）。它扫描当前轮次中名为 `tool_search` 的 ToolMessage，收集 `artifact.tool_references[].tool_name`。
2. `agentContext.markToolsAsDiscovered` 添加这些名称，把系统 runnable 标记为过期，并重新计算工具 token 统计。
3. 下一次模型调用会绑定这些工具。
4. 宿主可以根据历史预先填充 `discoveredTools`。
5. 事件模式下的快照 `callerCapabilityProjection` 把实时的发现状态发送给宿主。

### 5.2 PTC 设计
对于 `run_tools_with_code` 和 `run_tools_with_bash`，ToolNode 把以下内容注入 `invokeParams`（因而也在 `config.toolCall` 中）：
- `toolMap` 和 `toolDefs`：`allowed_callers` 包含 `code_execution` 的活跃工具。在事件驱动模式下只包含直接或本地工具，因为宿主工具无法在进程内运行。
- `disallowedToolDefs`：仅限直接调用的工具。
- `programmaticToolName`
- `hookContext`

**流程：**
1. `selectProgrammaticTools` 应用以下规则：
   - 如果存在仅限直接调用的工具，则 `tool_manifest` 必填。
   - 请求不允许的工具会抛出："cannot be called … not marked for code_execution"。
   - 未知名称会抛出异常。
2. `assertUnambiguousIdentifiers`：两个工具规范化为同一个标识符（连字符变为 `_`；Python 关键字加 `_tool` 后缀）时报错。
3. 如果代码不需要任何工具，就通过普通的 `/exec` 运行，Python 代码包在一个 `async def __user_main__()` 垫片中。
4. 否则带着工具定义 `POST /exec/programmatic`。只要状态为 `tool_call_required`（最多 `maxRoundTrips` 次，默认 20）：
   - `executeTools` 用 `tool.invoke(normalizedInput, {metadata:{[ptcName]: true}})` 并行运行每个调用。
   - MCP 元组或内容块输出被解包为文本或解析后的 JSON。
   - 错误变为 `{is_error:true, error_message}`。
   - 结果与 `continuation_token` 一起 POST 回去。
5. `completed`：按代码工具的方式格式化，并带会话产物。`error`：经过清理的消息（`buildCodeApiExecutionErrorMessage` 只保留白名单中的细节）。

运行时会话提示只在第一次请求中发送。PTC 保持无状态的提示。

---

## 6. 网页搜索管线（`src/tools/search/*`）

`createSearchTool(config)` 组装四个部分：

**1. 搜索提供商**（`searchProvider`，默认 `serper`）：
- **serper**：`POST https://google.serper.dev/{search|images|videos|news}`，请求头 `X-API-KEY`（`SERPER_API_KEY`）。
  - 请求体：`{q, safe: off|moderate|active[safeSearch 0..2, default 1], num: 1..10 (default 8), type?, tbs:'qdr:<date>', gl: country}`。
- **searxng**：`GET <SEARXNG_INSTANCE_URL>/search`，参数 `format=json, categories, language(all), safesearch, engines(default google,bing,duckduckgo), time_range`，可选请求头 `X-API-Key`（`SEARXNG_API_KEY`）。
  - 结果映射为 Serper 的形状。新闻根据标题关键词或 URL 路径推测。最多 6 张图片。`topStories` = 前 5 条新闻。
- **tavily**：`POST https://api.tavily.com/search`，Bearer 认证。
  - 请求体：`{query, search_depth:basic, topic: news|general, max_results ≤20, time_range, country, include_*…}`。
- 另外还有 **keenable**（`https://api.keenable.ai/v1/search`，无需密钥的回退）和 **crw**（`https://api.fastcrw.com/v1/search`）。

**2. `executeParallelSearches`：** 并行运行主网页搜索以及可选的图片、视频和新闻子搜索。
- 主搜索失败是致命的。失败的子搜索被丢弃。
- 新闻合并进 `topStories`，并按链接去重。
- 提供商支持标志：keenable 只支持普通结果；tavily、keenable 和 crw 不支持视频。

**3. `createSourceProcessor.processSources`：**
- `topStories` 上限为 `numElements`（默认 5）。
- `proMode`（默认 true）：抓取普通结果链接（启用 `news` 时也抓取新闻链接），跳过重复项。关闭时只抓取第一个 Wikipedia 链接。
- **抓取器**（`scraperProvider`，默认 `firecrawl`）对每个链接运行。Firecrawl 请求：
  - `POST {FIRECRAWL_BASE_URL|https://api.firecrawl.dev}/v2/scrape`，Bearer `FIRECRAWL_API_KEY`。
  - 请求体：`{url, formats:['markdown','rawHtml'], timeout:7500, onlyMainContent?, …}`。

  其他抓取器：
  - Serper：`https://scrape.serper.dev`，`X-API-KEY`。
  - Tavily：`POST https://api.tavily.com/extract`，批量。
  - 另外还有 crw 和 keenable。
- **内容处理**（`content.ts`）：cheerio 收集 `<a>`、`<img>`、`<video>` 和 iframe，并把 markdown 链接改写为标记 `(link#N "title")`、`(image#N)`、`(video#N)`。引用保存为 `{links, images, videos}`。
- 文本经过清理，并截断到 `SEARCH_MAX_CONTENT_LENGTH`（默认 50000）。
- **分块：** `RecursiveCharacterTextSplitter`，分隔符为 `\n\n`、`\n`；chunkSize 150，overlap 50，可通过 `SEARCH_CHUNK_SIZE` / `SEARCH_CHUNK_OVERLAP` 覆盖。overlap 会被限制在小于 size 的范围内。

**4. 重排序器**（`rerankerType`，默认 `cohere`）：
- **jina：** `POST https://api.jina.ai/v1/rerank`（环境变量 `JINA_API_URL`），Bearer。
  - 请求体：`{model:'jina-reranker-v2-base-multilingual', query, top_n, documents, return_documents:true}`。
- **cohere：** `POST https://api.cohere.com/v2/rerank`（环境变量 `COHERE_API_URL`）。
  - 请求体：`{model:'rerank-v3.5', query, top_n, documents}`。
- **rag-api：** `POST {RAG_API_URL}/v1/rerank`，带令牌提供函数。
  - 请求体：`{profile:'fast-v1', query, candidates:[{id,text,base_score:0}] (≤50), top_n ≤25}`。
- **infinity** 和 **none**：不打分。分块通过 `getDefaultRanking` 按原顺序透传，得分为 0。
- 超时 10 秒。`topResults`（默认 5）决定每个来源的高亮数量。
- 任何失败都会回退到默认排序，并记录到 `metrics` 中，每次搜索输出一行汇总日志。

**`expandHighlights(mainExpandBy=300, separatorExpandBy=150)`：**
- 在内容中定位每个高亮，最多向两侧各扩展 300 个字符，然后在额外 150 个字符的窗口内对齐到自然边界。边界优先级：段落 / 换行，然后句号 / 分号 / 冒号，然后逗号 / 空格。
- 跟踪高亮中出现的引用标记。
- **始终从每个来源中移除原始内容。**

**输出格式**（`format.ts` 中的 `formatResultsForLLM(turn, data, maxOutputChars)`）：
- 高亮预算：`SEARCH_MAX_LLM_OUTPUT_CHARS`，默认 50000。高亮按相关性顺序保留。位于边界处的高亮，如果剩余预算至少 200 个字符，会被截断并加上 `…[truncated]`。其余的被丢弃，并添加一条 "N additional highlights omitted" 说明。
- 分节：`=== Web Results, Turn X ===`、`=== News Results ===`、Knowledge Graph、Answer Box、People Also Ask。
- 每个来源渲染为：
  ```
  # Search 0: "Title"
  Anchor: turn{turn}search{i}
  URL / Summary / Date / Source
  ## Highlights
  ### Highlight 1 [Relevance: 0.93]
  ```text ... ```
  Core References:
  - link#3: <url>
      - Anchor: turn{turn}ref{k}
  ```
  只列出链接类型的引用；跳过 mailto 链接和文件扩展名。它们填充 `references[]`。
- **引用锚点**（在工具描述中）：
  - `turn{X}{search|news|image|ref}{Y}`
  - 高亮片段：`…`
  - 分组：`…`
- `turn` 来自 `config.toolCall.turn`，即 ToolNode 注入的按工具用量计数。
- 工具返回值：`[output, {web_search: {turn, ...data, references}, outcome?}]`，其中 outcome 为 `Found N results for "q"` 或 `Search failed for "q"`。
- 提供商错误以带 `error` 的数据返回，而不是抛出。

---

## 7. 子智能体系统

### 7.1 生成工具与配置
- **构建器：** `buildSubagentToolParams(configs, {background, threadContinuation})`（`src/tools/SubagentTool.ts`）生成：
  - 模式：`{intent?, description: string, subagent_type: enum<types>, run_in_background?: boolean (only when taskConfig is set), subagent_thread_id?: string (only when the store supports continuation)}`，`description` 和 `subagent_type` 必填
  - 一段列出 `- "type" (name): description` 的描述。
- **配置类型**（`src/types/graph.ts`），`SubagentConfig` 有四种形式之一：
  - 急切的 `agentInputs`，
  - `self: true`（复用父级的源输入），
  - 惰性的 `resolveAgentInputs(ctx)`，需要 `configId`，
  - `GraphSubagentConfig{kind:'graph', agents, edges(direct DAG), entryAgentId, resultAgentId}`。

  共享选项：`maxTurns`（默认 25）和 `allowNested`（默认 false）。
- **装配（wiring）：** `Graph.createAgentNode`（`src/graphs/Graph.ts` 约 5173-5420）把该工具作为**图工具**添加，这使它成为直接执行的工具：它总是在 SDK 中运行。它捕获 `toolCallId`、`thread_id`、批次熔断作用域以及父级 `configurable`。

### 7.2 深度与子上下文规则（`childGraphConfig.ts`）
**深度：**
- `maxSubagentDepth` 默认为 1。只有深度大于 0 时才注册该工具。
- 只有在 `allowNested` 时，子级才保留 `subagentConfigs`，并获得 `maxSubagentDepth = parentMax - 1`。否则两者都被清除。
- 图子智能体从不嵌套。

**子级启动时不带的内容：**
- `initialSummary`、`discoveredTools`
- 对于 `self`：`graphTools` 和 `compactionSemanticIndex`
- `toolDefinitions`，除非父级有 `ON_TOOL_EXECUTE` 处理器（否则子级的事件批次会挂起）

**子智能体 id：** `agentInputs.agentId`，否则为 `${parentAgentId}_sub_${suffix}`。对于图：`${parent}_subgraph_${suffix}`。

**子级输入：**
- `[...initialMessages, HumanMessage(description, {subagent hook-session and run-id markers}), ...prepared.messages]`
- 子级永远看不到父级的历史。

**Configurable：**
- 继承父级的 `configurable`，并移除 `__pregel_*` 和检查点相关键。宿主上下文合并在其上。
- `executionContext = {rootRunId, hookSessionId, depth+1, ancestry:[...,{subagentRunId, subagentType, subagentKind, subagentAgentId, parentRunId, parentAgentId, parentToolCallId}]}`。
- `thread_id` = HITL 开启时为子线程 id，否则为继承的值，再否则为 `childRunId`。

**调用：**
- `recursionLimit = maxTurns*3*(members) + (background ? 32 : 0)`
- 回调：一个转发器加一个用量捕获处理器。它们**替换**继承的回调链，因此子级的流式输出不会泄漏到父级流中。
- `runName: subagent:<type>`

**结果：**
- 只返回最后一段非空的 AI 文本（`filterSubagentResult`）。对于图，是结果智能体的最后一个轮次。
- 失败产生 `Subagent error: <truncated msg>`。
- 超出流限制会中止共享熔断器并向上传播。

**钩子与生命周期事件：**
- `SubagentStart` 钩子：`deny`/`ask` 产生 `Blocked: …`。
- `SubagentStop` 钩子：仅观察。
- `ON_SUBAGENT_UPDATE` 事件：`{runId, parentRunId, subagentRunId, parentToolCallId, subagentType, subagentKind, subagentAgentId, memberAgentId, depth, ancestry, phase: start|run_step|…|stop|error, data (sanitized), label, timestamp}`。
- 子级的 `ON_TOOL_EXECUTE` 被路由到**父级**的处理器，并携带子级的 `executionContext`。宿主应把该批次字段视为权威。

**宿主上下文适配器**（`docs/subagent-context.md`，`SUBAGENT_CONTEXT_VERSION=1`）：
- `prepare(input)` 在构造之前以及重试或重建时运行。它可以提供消息、`configurable` 以及每个成员的 `agentSessions`（`codeSessionKey` 加 `initialSessions`，用于划分代码会话映射）。未知成员会被拒绝。
- `complete(input, result)` 投影返回的文本。投递失败时保留已完成的结果，可以在不重新执行的情况下重试（`retryableDelivery`）。

### 7.3 `SubagentExecutionRegistry`
注册表以一个**地址**为键来索引执行：
- `identity = [threadId ?? parentRunId, checkpoint_id ?? 'root', parentAgentId, parentToolCallId, parentBatchKey]`
- `childThreadId = 'subagent:' + base64url(JSON(identity))`（持久化时再加上恢复尝试 id）
- `childRunId = ${parentRunId}_sub_<encoded>`

`SubagentExecutionRecord` 的职责：
- 通过单个待定 promise 对并发的 `execute()` 调用去重，
- 绑定定义、调用和结算指纹；之后的变化会以 "config changed" 或 "invocation changed" 被拒绝，
- 跟踪阶段（registered、active、interrupted、failed，…），
- 持有准备租约，在失败时回滚已创建的检查点或审批作用域，
- 保存已结算的输出，使父级重放时返回已持久化的 ToolMessage（`getSettledToolOutput` / `persistSettledToolOutput`，通过工具上的 `SUBAGENT_REPLAY_CONTROLLER` 暴露）。

只有在 HITL 开启且存在检查点存储时，它才是持久化的。子级内部的 HITL 会发起一个带恢复清单的作用域化 `GraphInterrupt`。

### 7.4 后台（分离）任务与用量汇
**`run_in_background:true` → `executeInBackground`。** 它要求：
- `taskConfig{store, scopeId}`
- 深度大于 0
- HITL 关闭
- 父级工具调用 id

**准备：**
- 它分叉钩子会话，并把 `ON_TOOL_EXECUTE` 处理器复制到一个分离的注册表中。
- 它调用 `store.start({scopeId, idempotencyKey: JSON([parentRunId,parentAgentId,parentToolCallId]), requestFingerprint, threadId?, input, subagentKind, subagentType, run(runtime, initialMessages)})`。

**立即返回：** 一个 JSON 字符串 `{background_task_id, subagent_thread_id?, tool:'subagent', subagent_type, status, message}`。因容量、冲突或 thread_unavailable 被拒绝时返回 `{status:'rejected', message}`。

**`SubagentTaskRuntime`：**
- `signal`
- `shouldPreempt()`，驱动协作式封存（最多 32 次）
- `drain(boundary)`，用于插话、排队或中断消息
- `closeTurn()`：排队的后续消息会以 `messages + injected` 重新调用子级
- `reportProgress`

**`InMemorySubagentTaskStore` 默认值：**
- 任务超时 30 分钟
- 完成后 TTL 1 小时
- 每个作用域 10 个运行中任务，总计 100 个
- 结果上限 100k 字符
- 每个任务 32 个控制指令

它支持 `get`、`list`、`claim`（结果只能被认领一次）和 `control`（`steer|queue|interrupt|cancel|cancel_message`）。轮询由宿主工具完成。

**用量汇。** 对子级发起的每次模型调用，`usageSink(event)` 都会收到 `{usage, model, provider (INVOKED_PROVIDER), subagentType, subagentKind, subagentRunId, subagentAgentId, memberAgentId, parentRunId, depth, ancestry, runId: ROOT run}`。调用会被等待，错误被吞掉。嵌套层级逐级向上转发。

---

## 8. Python 重新实现指引

### 8.1 节点类型
**不要**使用 `langgraph.prebuilt.ToolNode`。编写一个自定义异步节点（`async def tool_node(state, config)`），返回 `list[BaseMessage] | Command | list[Command|dict]`。预置节点缺少：
- 事件分发，
- 直接 / 事件混合时的顺序，
- 无效调用的提升，
- HITL 批处理，
- 把移交聚合为 `Send`。

保留 `tools_condition` 的语义。

### 8.2 宿主契约
把 `ToolExecuteBatchRequest` 建模为 pydantic 模型加一个 `asyncio.Future`。用以下 API 替换 resolve/reject 回调：

```python
class ToolExecutor(Protocol):
    async def execute_batch(self, req: ToolExecuteBatchRequest,
                            on_result: Callable[[ToolExecuteResult], Awaitable[None]] | None) -> list[ToolExecuteResult]: ...
```

同时通过 LangChain `adispatch_custom_event("on_tool_execute", …)` 发出事件，以与宿主保持一致。字段名保持完全一致（`toolCallId`、`codeSessionContext`、`storage_session_id`，…），使 TypeScript 的 LibreChat 宿主保持兼容，并用 pydantic 别名固定它们。

### 8.3 并发
- 直接路径：`asyncio.gather(..., return_exceptions=True)`，然后重新抛出，其中中断的优先级低于其他错误。先运行中断型分组。
- 在任何 `await` 之前冻结输出引用快照。
- 用 `asyncio.Event` 或 task-group 的取消作用域组合取消信号，替代 `AbortSignal`。在每个阶段检查熔断器：入口、分发前、直接分组之后。
- Python LangGraph 中的 `interrupt()` 必须在节点上下文内调用。

### 8.4 HTTP
- 每个基础 URL 使用一个共享的 `httpx.AsyncClient`，设置 `follow_redirects=False`，按提供商设置超时（serper/searxng 10 秒，firecrawl 7.5 秒，rerank 10 秒，错误响应体 1 秒 / 64 KB），代理取自 `PROXY`/`NO_PROXY`。
- PTC 取消：取消时发送“发出即忘”的 `POST {endpoint}/cancel`，带上请求 id（超时 2 秒）。
- 以有上限的方式流式读取错误响应体。

### 8.5 模式
- 把工具模式保留为**原始 JSON Schema 字典**（`intent` 排在第一的属性顺序很重要；Python 字典保留插入顺序）。通过 `StructuredTool.from_function(args_schema=dict)` 或 `convert_to_openai_tool` 绑定。
- 内部线上模型使用 pydantic：`CodeEnvFile` 作为以 `kind` 为判别字段的联合类型、`ExecuteResult`、`ProgrammaticExecutionResponse`、`ToolExecuteResult`。
- 精确实现 `coerce_args_for_schema`：只做无损强制转换。

### 8.6 建议的模块布局
```
agents/tools/node.py            # ToolNode run()、混合批处理、无效调用提升、移交
agents/tools/contract.py        # pydantic ToolCallRequest/BatchRequest/Result、CallerCapabilitySnapshot
agents/tools/lifecycle.py       # 钩子、HITL 审批负载、决策规范化
agents/tools/eager.py           # 计划构建器、参数强制转换、stable_stringify、急切执行注册表
agents/tools/truncation.py      # 头部 / 尾部截断、有界结构化序列化、内容压缩
agents/tools/code_session.py    # 按 (storage_session_id,id) 更新 / 保留会话合并
agents/tools/output_refs.py     # {{toolNturnM}} 注册表
agents/tools/code_api/{client,execute_code,bash,ptc,bash_ptc,errors}.py
agents/tools/local/{engine,coding_tools,ptc_bridge}.py
agents/tools/tool_search.py     # BM25（rank_bm25）、正则沙箱模式、MCP 服务器解析
agents/tools/web_search/{providers,scrapers,content,chunking,rerankers,highlights,format,tool}.py
agents/tools/subagent/{tool,executor,registry,child_config,task_store,usage}.py
```

### 8.7 需要保留的陷阱处理
1. **消息顺序：** 保持 tool_call → tool_result 相邻，把注入的或钩子产生的消息放在所有结果之后。给合成消息标记 `additional_kwargs.role='system'`。
2. **无效工具调用：** 用相同 id 的替换 AIMessage 提升它们，否则 Anthropic 在下一个轮次（以及 HITL 恢复时）会返回 400。
3. **完成事件恰好一次**，覆盖提前的 `onResult` 发出、急切发出、`errorHandler` 归属、推迟的被阻止调用以及重放。
4. **急切执行不匹配：** 返回错误，并对该工具名禁止急切执行。绝不重新运行该工具。
5. **两个会话 id：** 绝不用执行 `session_id` 覆盖文件的 `storage_session_id`。只有在 `deleted_files` 中有明确条目且与基线标识一致时才删除文件。
6. **宿主控制的字段：** 从 Code API 请求体中去除模型提供的 `runtime_session_hint`、`workspace_instance_id` 和 `intent`。发送给宿主之前去除 SDK 私有的 configurable 键。
7. **轮次计数器** 按工具名计数，网页搜索的锚点依赖它们。保持直接计数器与急切计数器一致。
8. **网页搜索** 必须在输出前移除抓取到的原始内容。预算针对的是高亮，而不是摘要片段。
9. **子智能体子级** 替换回调链。用量必须经由汇（带根 runId）上报，子级的 `ON_TOOL_EXECUTE` 发往父级处理器。
10. **截断边界：** 按代码单元截断，但不拆分代理对。在 Python 中按 `str` 码点切片天然是安全的，但要保留 70/30 的头尾拆分和换行对齐。
11. **工具搜索的正则模式：** 它向 Code API 发送请求时**不带认证请求头**。把这视为已知缺陷。Python 移植应转发认证信息，或默认使用 `local` 模式。
