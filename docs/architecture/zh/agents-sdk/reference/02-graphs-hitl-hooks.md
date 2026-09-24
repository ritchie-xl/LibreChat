# @librechat/agents：图、AgentContext、人机协同、钩子、抢占与事件 Actor

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

本报告涵盖 TypeScript SDK 中的图编排、AgentContext、人机协同（HITL）、钩子、抢占 / 插话引导、检查点和事件 Actor，并把每一部分映射到 LangGraph Python。所有路径均相对于 SDK 仓库根目录。

**范围，以及一个先说在前面的发现：**
- 本报告基于阅读 `src/graphs/*`、`src/agents/*`、`src/hitl/*`、`src/hooks/*`、`src/llm/preempt.ts`、`src/eventActor/*`，`src/run.ts`、`src/tools/ToolNode.ts` 和 `src/llm/invoke.ts` 的相关部分，以及文档、ADR 和 `CONTEXT.md`。
- **`hide_sequential_outputs` 在这个 SDK 中完全不存在。** 输出隐藏是 LibreChat 宿主层的职责。SDK 只提供实现它所需的原始材料：运行步骤上的 `agentId`/`groupId`、`getContentPartAgentMap()` 和 `labelContentByAgent()`。

---

## 1. 单智能体图（`StandardGraph`，`src/graphs/Graph.ts`）

### 1.1 类结构与生命周期

- **类。** `abstract class Graph`（Graph.ts:722）保存与图并列的每次运行可变状态：运行步骤簿记、处理器和钩子注册表、HITL 配置、工具急切执行状态、工具输出引用注册表以及工具会话。`StandardGraph extends Graph`（Graph.ts:1303）是具体实现。
- **构造。** `createGraph({kind:'standard'|'multi-agent', input})`（`src/graphs/createGraph.ts`）是多态工厂。它注入 `GraphFactoryDependencies`（`src/graphs/graphFactory.ts`），使子智能体可以构建任一种类的子图。
- **运行时配置。** `applyGraphRuntimeConfig`（`src/graphs/applyGraphRuntimeConfig.ts`）把七个运行级字段复制到图上：`hookRegistry`、`humanInTheLoop`、`toolOutputReferences`、`eagerEventToolExecution`、`codeSessionToolNames`、`interruptingToolNames`、`toolExecution`。`Run` 对顶层图使用它，`createAgentNode` 对子图使用它。
- **智能体上下文。** 构造函数（Graph.ts:1527）为每个 `AgentInputs` 构建一个 `AgentContext.fromConfig(...)`，放入 `agentContexts: Map<agentId, AgentContext>`。它会初始化 `calibrationRatio` 和淡化（fading）层级。`defaultAgentId` 为 `summarizeOnlyAgentId ?? agents[0].agentId`。
- **旧版 / 标准运行路径。** `Run.createLegacyGraph`（`src/run.ts:614`）只会把 `agents[0]` 传入标准图。

### 1.2 状态模式（两层）

**外层工作流**（`createWorkflow`，Graph.ts:5716），用 `Annotation.Root` 构建：

| 通道 | 归约器 | 说明 |
|---|---|---|
| `messages: BaseMessage[]` | `messagesStateReducer`（`src/messages/reducer.ts`，按 id 更新插入，外加 `RemoveMessage(id='__remove_all__')` 哨兵）。它还会把结果镜像到 `this.messages`，并在第一次写入时记录 `this.startIndex = a.length + b.length`。 | `startIndex` 是宿主提供的输入的大小。它决定 `{results}` 切片、裁剪起点以及“本次运行产生”的判定。 |
| `manualSummary?: string` | 最后一次写入生效 | 仅摘要运行。 |
| `runStepState: RunStepResumeState` | `update.revision >= current.revision ? update : current`（Graph.ts:714） | 写入检查点的运行步骤与内容索引附属状态，用于 HITL 恢复（`src/types/stream.ts:115`）。 |
| `handoffState?: HandoffState` | `mergeHandoffState`（`src/graphs/handoff.ts:41`）：按转移 id 取并集，冲突时抛出异常 | 仅对多智能体图有意义。 |

**内层智能体子图**（`createAgentNode`，Graph.ts:5156）：

| 通道 | 归约器 | 说明 |
|---|---|---|
| `messages` | 先调用 `agentContext.invalidateProviderProjectionForMessageUpdates(b)`，再调用 `messagesStateReducer` | |
| `summarizationRequest?: {remainingContextTokens, agentId}` | 最后一次写入生效 | 由模型节点设置，用于绕道进入摘要。 |
| `manualSummary`、`runStepState`、`handoffState` | 与外层相同 | |

### 1.3 节点与边拓扑（可直接转为 mermaid）

节点名来自 `GraphNodeKeys`（`src/common/enum.ts:104`）：`AGENT='agent='`、`TOOLS='tools='`、`SUMMARIZE='summarize='`。

```
%% 外层（单智能体）
START --> <defaultAgentId>          %% 节点 = 编译后的智能体子图，ends:[END]
<defaultAgentId> --> END

%% 智能体 X 的内层子图
START --> agent=X
agent=X -- routeMessage --> agent=X        %% 抢占自循环（pendingPreemptReturn）
agent=X -- routeMessage --> summarize=X    %% state.summarizationRequest != null
agent=X -- routeMessage --> tools=X        %% toolsCondition：存在未调用的 tool_calls / 可归属的无效调用
agent=X -- routeMessage --> END            %% 没有工具调用，或仅摘要运行
summarize=X --> agent=X
tools=X --> agent=X                        %% agentContext.toolEnd 时为 END
```

在仅摘要模式下，外层节点由 `compactingAgentNode` 包装。该包装器会把 `createRemoveAllMessage()` 加到结果前面（`propagateManualCompaction`，Graph.ts:1623）。

### 1.4 路由（`routeMessage`，Graph.ts:5457）

检查按以下顺序进行：
1. **抢占自循环。** `pendingPreemptReturn.delete(agentId)` 返回 `agentNode`。封存的轮次注入了消息，因此模型在同一次 Pregel 运行中继续。
2. **摘要绕道。** `state.summarizationRequest != null` 返回 `summarizeNode`。
3. **仅摘要运行。** `summarizeOnlyAgentId != null` 返回 `END`。
4. **工具。** `toolsCondition(state, toolNode, invokedToolIds)`（`src/tools/ToolNode.ts:6285`）：
   - 当最后一条 AI 消息的 `tool_calls` 并非全部在 `invokedToolIds` 中时，路由到工具。
   - 当存在可归属的 `invalid_tool_calls`，且每个有效调用都是提供商服务端调用（`srvtoolu_` 前缀）时，也路由到工具。
   - 否则返回 `END`。
5. **截断标志。** 当决定为 `END` 且 `getTruncationStopReason(lastMessage)` 有值时，设置 `outputTruncatedIncomplete`。`Run` 随后将其报告为 `output_truncated`。

### 1.5 智能体节点（`createCallModel`，Graph.ts:3225-5027）

该节点由 `invokeWithRunStepState`（Graph.ts:5435）包装。包装器：
- 设置 `this.config`；
- 从状态中恢复 `runStepState` 和 `handoffRouting`；
- 调用模型节点；
- 返回 `runStepState: this.createRunStepResumeState()`，并在普通运行中清除 `manualSummary`。

节点内部依次执行：
1. **熔断器。** 在入口捕获 `breakerAbort`/`breakerEpoch`。如果同级已经触发熔断，立即重新抛出。
2. **抢占停止。** 如果 `preemptHaltReason != null`（`PreemptBoundary` 钩子要求停止），返回 `{messages: []}`。这会让所有后继节点变成空操作。
3. **工具发现。** `extractToolDiscoveries(messages)` 的结果传给 `agentContext.markToolsAsDiscovered`（通过工具搜索找到的延迟加载工具）。然后等待 `tokenCalculationPromise`。
4. **裁剪与上下文预算。**
   - 用 `createPruneMessages` 惰性构建裁剪器（需要 `tokenCounter` 和 `maxContextTokens`）。
   - 在 `agentContext.getProviderProjectedMessages(messages)` 上运行它。
   - 得到裁剪后的 `context`、校准比率、淡化层级和一份上下文使用快照（作为 `ON_CONTEXT_USAGE` 事件发出）。
   - 如果启用了摘要且 `shouldTriggerSummarization(...)` 触发，返回 `{summarizationRequest}`。路由器随后绕道到摘要节点。
   - 仅摘要运行也在这里返回它的请求。
5. **构造模型。**
   - `getPreparedToolsForBinding` 返回工具列表，并在最后一个静态工具上放置提示缓存断点（`src/llm/promptCacheTools.ts`）。
   - 调用 `initializeModel({tools, provider, clientOptions})`，除非设置了 `overrideModel`。
   - 如果存在系统提示，`model = agentContext.systemRunnable.pipe(model)`。
6. **提供商投影**（只作用于线上请求；从不修改图状态），依次为：
   1. 旧版内容格式化；
   2. `projectToolMessagesForProvider`（限制工具调用输入的大小）；
   3. Bedrock 尾部空白修复；
   4. `ensureThinkingBlockInMessages`；
   5. `foldToolBlocksForToollessAgent`；
   6. `appendInstructionlessHandoffCue`，以及针对类 Anthropic 提供商的 `appendPredecessorHandoffCue`（`src/messages/handoffCue.ts`）；
   7. `annotateMessagesForLLM`（工具输出引用标签）；
   8. 合成上下文压缩，以适配预算；
   9. 针对严格交替提供商的 `coalesceAdjacentUserTurns`；
   10. 缓存标记，包括 `addBedrockTailCacheControl`；
   11. `sanitizeOrphanToolBlocks`；
   12. 最后是 `prepareProviderRequest(...)`。

   每一步都由 `contextPressure.trackProjection` 包装，以保留 token 归属。
7. **调用。** `attemptInvoke({request, context: this, preemptAgentId}, config)`（`src/llm/invoke.ts`）。它通过流处理器流式传递数据块，流处理器在数据到达时分发 `ON_RUN_STEP`、`ON_MESSAGE_DELTA`、`ON_REASONING_DELTA` 以及工具调用运行步骤。失败时：
   - 流限制错误或 `PreparedSubagentError` 触发熔断并重新抛出。
   - 上下文溢出时，如果摘要能缩小提示，就执行溢出恢复（摘要绕道）。
   - 其他错误执行 `tryFallbackProviders`。
8. **调用后的簿记。**
   - 如果响应没有 id，为其分配一个 v4 id，并记录到 `runProducedAiMessageIds`。
   - 流式不可用时，回退分发推理 / 文本 / 工具调用运行步骤（`handleToolCalls`）。
   - 更新 `currentUsage`/`lastCallUsage`（估算的用量不参与校准）。
9. **抢占边界**（见 §6），然后返回 `{messages: [aiMessage, ...injected]}`。

### 1.6 工具节点装配（`initializeTools`，Graph.ts:2827）

每个智能体都有一个 `CustomToolNode`（`src/tools/ToolNode.ts`）。
- **事件驱动模式**（`agentContext.toolDefinitions` 非空）：
  - 模型绑定的是仅含模式的工具（`createSchemaOnlyTools`）。
  - 执行通过 `ON_TOOL_EXECUTE` 事件分发给宿主。
  - `graphTools` 中由图管理的工具（移交、子智能体、宿主进程内工具）成为 `directToolNames`，在进程内运行。
- **旧版模式：** 直接调用 `tools` 加上 `graphTools`。
- **传给每个 ToolNode 的选项：** `handoffRouting`、`hookRegistry`、`humanInTheLoop`、`interruptingToolNames`、`toolOutputRegistry`（每次运行一个）、`fileCheckpointer`、熔断器访问器，以及 `restoreRunStepResumeState`/`createRunStepResumeState` 回调。
- **中断型工具。** `interruptingToolNames` 是一个调度提示：这些工具先于其直接执行的同级工具运行。启用 HITL 时，`subagent` 会被自动加入该集合。

当存在 `subagentConfigs` 且深度 > 0 时，子智能体工具在 `createAgentNode`（Graph.ts:5173-5424）中创建。它的模式是 `{description, subagent_type, subagent_thread_id?, run_in_background?}`，由 `SubagentExecutor` 支撑，并被放入 `graphTools`，这会触发 token 重新计数。

### 1.7 递归限制与结束条件

- **递归限制。** 配置了抢占时，`Run.processStream`（run.ts:1273）设置 `recursionLimit = (caller.recursionLimit ?? DEFAULT_RECURSION_LIMIT=50) + resolveMaxSeals(preemption.maxSeals)`。额外的余量为每次封存预留一个超步（superstep）。`MultiAgentGraph` 还可以在每次调用成员子图时设置 `memberRecursionLimit`。
- **结束条件：** 没有工具调用；`toolEnd`；仅摘要；抢占停止；钩子停止（`processStream` 在流事件之间通过 `hookRegistry.getHaltSignal(runId)` 轮询，然后跳出）；中断；`HandoffLimitError`；以及流限制熔断。

### 1.8 运行步骤与内容状态

- `contentData: RunStep[]`、`stepKeyIds`、`contentIndexMap`、`toolCallStepIds`、`pendingToolCallsByStep`、`openMessageStepByAgent` 等（Graph.ts:784-878）。
- **步骤键** 由 `[run_id, thread_id, langgraph_node, langgraph_step, checkpoint_ns, streamSegment]`（Graph.ts:2535）构成，再加上已调用工具数，再加上 `reasoning` 或 `post-reasoning-N`。
- **智能体解析。** `getAgentContext(metadata)` 解析 `metadata.langgraph_node`，去掉 `agent=`/`tools=`/`summarize=` 前缀（Graph.ts:2425）。
- **访问器：** `getRunSteps(agentId?)`、`getRunStepsByAgent()`、`getContentParts()`、`getContentPartAgentMap()`。
- **为恢复而序列化。** `createRunStepResumeState()`/`restoreRunStepResumeState()` 把这些结构序列化到 `runStepState` 通道。这样重建后的 `Run` 可以用相同的步骤 id 恢复。

### 1.9 检查点的使用

- **检查点存储。** `compileOptions.checkpointer` 由宿主提供。启用 HITL 且未提供检查点存储时，`Run.applyHITLCheckpointerFallback` 会安装一个 `MemorySaver`（run.ts:745）。
- **持久性。** 只要存在检查点存储，默认 `config.durability = 'exit'`（run.ts:1454），因此状态只在退出或中断边界持久化。
- **在已有检查点的线程上提交新输入** 时，对 `handoffState` 和 `runStepState` 使用 `Overwrite(...)`（run.ts:1500），因此旧的附属状态被替换而不是合并。
- **恢复** 时读取 `getState(config, {subgraphs:true})`，恢复 `handoffRouting`、`runStepState` 和消息，把 `checkpoint_id`/`checkpoint_ns` 固定到中断所在的检查点，然后流式提交 `new Command({resume: {[interruptId]: value}, update?, goto?})`（run.ts:2033-2231）。

---

## 2. 多智能体图（`src/graphs/MultiAgentGraph.ts`）

### 2.1 输入与校验

`MultiAgentGraphInput`（`src/types/graph.ts`）在标准输入之上增加以下字段：
- `edges: GraphEdge[]`
- `entryAgentId?`
- `maxHandoffs?`
- `resultAgentId?`（把该成员最后一条新的 AI 消息捕获到 `subagentResult`）
- `memberRecursionLimit?`

`GraphEdge` 为 `{from: string|string[], to: string|string[], description?, condition?(state) => bool|string|string[], edgeType?: 'handoff'|'direct', handoffScope?: 'turn'|'conversation', prompt?: string | (messages, runStartIndex) => string|Promise<string>|undefined, excludeResults?, promptKey?}`。

构造函数（第 355-424 行）依次执行：
- `validateEdgeAgents`：未知的智能体 id 会提前抛出异常。
- `categorizeEdges`（第 534 行）：当 `edgeType==='direct'`，或者边没有 `edgeType`、没有条件、只有一个源且目标多于一个时，该边为 **direct（直连）**。**其余一律视为移交**，包括没有显式 `edgeType` 的多源边。
- `validateCommandRoutedDirectEdges`：
  - 分组（全部满足）直连边不能包含移交源。
  - 带提示的直连边不能有经 Command 路由的源。
- `analyzeGraph`：起始节点是没有入边的智能体（回退：第一个智能体），然后执行 `computeParallelCapability`。
- `entryAgentId` 会覆盖起始节点，`resolveEntryReachability` 把编译范围限制在可达的智能体上。它会拒绝无法满足的分组汇合。
- 校验 `maxHandoffs`，然后创建 `HandoffRouting(entry, maxHandoffs, parallel)`。
  - 如果有多个起始节点或存在任何直连扇出，图会被标记为并行。
- `validateHandoffScopes`，然后 `createHandoffTools`。

**并行组 id**（`computeParallelCapability`，第 635 行）：BFS 为扇出目标分配递增的 `groupId`；多个起始节点构成第 1 组。它们通过 `getParallelGroupIdForAgent` 暴露，并被标记到运行步骤上。

### 2.2 移交工具（`createHandoffToolsForEdge`，第 831 行）

- **保留名称。** 先从 `graphTools` 中移除已有的 `lc_transfer_to_*` 和 `conditional_transfer` 工具。
- **每个目标一个：** 工具 `lc_transfer_to_<dest>`（`Constants.LC_TRANSFER_TO_`）。
  - 模式：当 `edge.prompt` 是字符串时为 `{[promptKey ?? 'instructions']: string}`（提示成为该参数的描述）；否则为空。
  - 函数体构建 `ToolMessage("Successfully transferred to X[\n\nInstructions: ...]")`，带有 `additional_kwargs.handoff_source_name` 和 `handoff_instructions`。
  - 它返回 `new Command({goto: dest, graph: Command.PARENT, update: {messages, handoffRequest: {sourceAgentId, targetAgentId, toolCallId, scope}}})`。
  - 当发起的 AI 消息含有多个工具调用时，`messages` 会被过滤为*仅包含本次调用*的 AI 消息及其 ToolMessage，同时保留来源信息。这使并行移交对提供商而言仍然合法。
- **条件边：** 单个工具 `conditional_transfer`。它对 `edge.condition(runtime.state)` 求值：
  - `false` 表示不转移（返回 `null`）；
  - 字符串或数组选择目标，目标必须已声明；
  - 它把 `handoff_destination` 存入 `additional_kwargs`。

### 2.3 移交在 ToolNode 一侧的处理（ToolNode.ts 约 5960-6090）

- **聚合。** 收集 `graph === PARENT` 且只有单个目标的 Command。
  - 单个移交原样透传。
  - 同一批次中的多个移交会变成 **`Send(dest, cmd.update)` 扇出**。每个移交 ToolMessage 获得 `handoff_parallel_siblings`、`__handoff_parallel_batch` 和 `__handoff_group_id`（运行时组 id）。
  - 仅含 Send 的 Command 会被合并。
- **收尾。** 然后运行 `handoffRouting.finalize(commands, input, config)`（`handoff.ts:166`）：
  - 每个 `handoffRequest` 变为一个 `HandoffTransition`，带有确定性 id `JSON.stringify([executionId, checkpoint_ns, sourceAgentId, lastMessage.id, toolCallId])` 和 `depth`（从而使重放幂等）。
  - 它强制执行 `maxHandoffs`（`HandoffLimitError`，`Run` 将其报告为 `handoff_limit`）。
  - 它删除 `handoffRequest`，并把 `handoffState` 快照写入每个 Command 和 Send。
- **结果。** `HandoffRouting.outcome()` 给出 `candidate | unchanged | ambiguous | incomplete`。它通过 `Run.getHandoffOutcome()` 暴露（docs/multi-agent-patterns.md）。只有非并行图中 `conversation` 作用域的转移才能产生 candidate。

### 2.4 外层工作流（`createWorkflow`，第 1376 行）

状态在基础通道之外增加以下通道：
- `agentMessages: BaseMessage[]`（替换型归约器），供 `excludeResults` 使用；
- `subagentResult`（最后一次写入生效）；
- `manualSummary`、`runStepState`、`handoffState`。

`messages` 归约器同样在第一次写入时设置 `startIndex`。

```
%% 对每个可达智能体 A：节点 A = 包裹 createAgentNode(A) 子图的 agentWrapper(A)
%% ends(A) = handoffDests(A) ∪ directDests(A) ∪ {END if hasHandoff || !hasDirect}
START --> s                       %% 对每个起始节点 s（或仅对 summarizeOnly 智能体）
%% 不带提示的直连边（仅静态源；同时拥有移交边的源会被跳过）：
A --> B                           %% 单源
[A1,A2] --> B                     %% 分组全部满足汇合（addEdge(array, dest)）
%% 指向 B 且任一边带提示的直连边：
A1 --> fan_in_B_prompt ; A2 --> fan_in_B_prompt ; fan_in_B_prompt --> B
%% 移交：动态路由，由 tools=A 节点通过 Command(graph=PARENT, goto=B) 或 Send(B, update) 实现
%% 同时拥有移交边和直连边的智能体：如果最后一条消息是 lc_transfer_to_* ToolMessage，
%%   包装器返回 Command(goto=handoffDest)，否则返回 Command(goto=directDests)（互斥路由）
```

### 2.5 `agentWrapper`（第 1484 行）：按智能体整形输入

1. 从状态中恢复 `handoffRouting`。应用 `memberRecursionLimit`。
2. **`processHandoffReception(messages, agentId)`**（第 1129 行）：
   - 只有当最近一个 AI 轮次包含转移调用，且尾部工具块中有一条转移 ToolMessage 以本智能体为目标时，移交才算“活跃”。
   - 它提取指令（结构化的 `handoff_instructions`，否则按旧版 `Label:` 解析）、`sourceAgentName`、同级名称和组 id。
   - 它移除**所有**转移 ToolMessage、转移 `tool_calls` 和转移 `tool_use` 内容块，因此接收方永远看不到移交噪声。
3. **配置元数据：**
   - `withActiveAgentMetadata`：`metadata.activeAgentId` / `activeAgentName`；
   - `withInstructionlessHandoffCue`：记录传入 AI 消息的 id，这样当移交不带指令时，只在发给提供商的负载中追加一条用户提示 `"Continue as the receiving agent…"`；
   - `withHandoffGroupMetadata`（`__handoff_group_id`）。
4. **移交上下文。** `agentContext.setHandoffContext(source, siblings)` 把系统提示标记为过期，使其带着“Multi-Agent Workflow”身份前言重建。否则调用 `clearHandoffContext()`。
5. **如果收到了带指令的移交：**
   - 追加一条路由 `HumanMessage`（`additional_kwargs {role:'user', isMeta:true, source:'routing'}`，标记为合成），长度受该智能体剩余预算限制（`resolveMaxRoutingPromptChars`）；
   - 如果过滤后的最后一条消息是 ToolMessage，先插入合成的桥接消息 `AIMessage("[Processed tool result and transferring to X]")`；
   - 重建 token 映射；
   - 用变换后的消息调用子图。
6. **否则，如果 `state.agentMessages` 非空**（excludeResults 路径）：以 `messages = agentMessages` 调用，然后清空 `agentMessages`。
7. **否则** 用完整状态调用。
8. `resultAgentId` 把 `getLastNewAiMessage(result.messages, inputMessages)` 捕获到 `subagentResult`。

**重要的归约器语义。** 子图返回它的*整个*消息数组，外层归约器按 id 更新插入。接收方看到的经过过滤、不含转移消息的视图**不会**从外层状态中删除转移消息，因为它们仍以原始 id 存在。Python 移植必须依赖 `add_messages` 中同样的按 id 合并。

### 2.6 直连边提示与 `{results}`（第 1756 行）

`fan_in_<dest>_prompt` 节点的行为如下：
- **提示为函数：** `await prompt(state.messages, startIndex)`，有长度上限。
- **包含 `{results}` 的字符串：**
  - `results = getBufferString(state.messages.slice(startIndex))`，有长度上限；
  - 用 `PromptTemplate.fromTemplate(prompt)` 渲染；
  - `effectiveExcludeResults = excludeResults !== false && promptText !== ''`，因此默认排除结果。
- **普通字符串：** 原样使用。
- **输出：**
  - 不排除：`{messages: [routingPrompt]}`。
  - 排除：`{messages: [routingPrompt], agentMessages: messagesStateReducer(messages[0:startIndex], [routingPrompt])}`。目标智能体只看到原始输入加上提示，而全局记录保留全部内容。

### 2.7 消息与内容中的智能体身份

- **运行步骤：** 只有在 `isMultiAgentGraph()` 时才设置 `runStep.agentId` 和 `runStep.groupId`（Graph.ts 约 5886）。智能体从 `langgraph_node` 解析。组 id 从运行时的 `__handoff_group_id` 元数据或静态并行组解析。
- **打开的消息步骤** 按智能体通道（lane）跟踪（`openMessageStepByAgent`）。
- **工具消息：** `handoff_source_name`、`handoff_parallel_siblings`、`handoff_destination`。
- **系统提示：** 身份前言（§3）。
- **宿主侧重放辅助函数：** `labelContentByAgent(contentParts, agentIdMap, agentNames, {labelNonTransferContent})`（`src/messages/format.ts:2330`）。它把被转移智能体的内容折叠进转移工具调用的 `output`，格式为 `--- Transfer to X --- … --- End of X response ---`。`ContentTypes.AGENT_UPDATE` 内容片段（`on_agent_update`）由 `src/stream.ts` 聚合，但该事件由宿主发出。
- **预填充保护：** 当负载以*本次运行*产生的助手轮次结尾时（直连边的后继），`appendPredecessorHandoffCue` 会添加一条只作用于线上请求的用户提示。

---

## 3. AgentContext（`src/agents/AgentContext.ts`）与系统提示组装

### 3.1 职责（每个图中每个智能体一个可变对象）

- **身份与提供商：** `agentId`、`name`、`endpoint`、`provider`、`clientOptions`、`langfuse`、`reasoningKey`、`toolEnd`、`useLegacyContent`。
- **工具：**
  - `tools`/`toolMap` 是宿主实例；
  - `graphTools` 是 SDK 管理的工具（移交、子智能体），总是复制而不是共享引用；
  - `toolDefinitions`（事件驱动的模式）和 `toolRegistry`（每个工具的 `defer_loading` 和 `allowed_callers`）；
  - `discoveredToolNames` 保存通过工具搜索找到的延迟加载工具；
  - `getToolsForBinding()` 应用调用方能力投影：只绑定允许 `'direct'` 调用方且处于活跃状态（非延迟加载或已发现）的定义。
- **token 预算：**
  - `maxContextTokens`、`indexTokenCountMap`/`baseIndexTokenCountMap`、`systemMessageTokens`、`dynamicInstructionTokens`、`toolSchemaTokens`、`toolTokenCounts`、`instructionTokens`（getter）、`calibrationRatio`（EMA）、`fadingTier`、`pruneMessages`、`currentUsage`、`lastCallUsage`；
  - `calculateInstructionTokens`（第 1384 行）把每个绑定工具转成 JSON 模式并计数，乘以一个系数（Anthropic 与默认不同），用最大余数法分摊；
  - `getTokenBudgetBreakdown`、`projectContextUsage`（也被 `src/agents/projection.ts::projectAgentContextUsage` 使用，后者让宿主无需调用模型即可得到发送前的估算）。
- **摘要：**
  - `setInitialSummary` 是放在系统提示中的跨运行摘要；
  - `setSummary` 是以用户消息为载体的运行中摘要；
  - 它还跟踪摘要触发、失败、溢出恢复和淡化层级重置。
- **移交上下文：** `setHandoffContext` / `clearHandoffContext`。
- **程序化工具调用指引：** 根据调用方能力投影构建。
- **`reset()`**（第 1255 行）是每次运行的重置。它恢复持久摘要，清除已发现的工具和移交上下文，并重新计算 token。

### 3.2 系统提示组装（`systemRunnable`，第 799-1250 行）

提示会被缓存，直到被标记为过期。移交上下文变化、新的工具发现和摘要变化都会设置 `systemRunnableStale`。

- **稳定部分**（`buildStableInstructionsString`），用 `\n\n` 连接：
  1. 身份前言，仅在有移交上下文时出现：`## Multi-Agent Workflow\nYou are "<name>", transferred from "<source>".\n[Running in parallel with: …]\nExecute only tasks relevant to your role…`；
  2. `instructions`；
  3. `## Programmatic Tool Calling` 小节。
- **动态部分**（`buildDynamicInstructionsString`）：
  1. `additional_instructions`；
  2. 当摘要位置为 `system_prompt` 时的 `## Conversation Summary\n\n<summary>`（跨运行的初始摘要）。
- **SystemMessage 的形状**（`buildSystemMessage`）：
  - **Anthropic 且启用 `promptCache`：** 内容块 `[{text: stable, cache_control: ephemeral(ttl default 1h)}, {text: dynamic}]`。当稳定和动态部分都非空且缓存提供商处于活跃状态时，动态文本会被*移出*系统消息（`shouldMoveDynamicInstructions`）。
  - **OpenRouter：** 同样的模式，使用 `cache_control`。
  - **Bedrock 且启用 `promptCache`：** `[{text: stable}, {cachePoint}, {text: dynamic}]`。
  - **其他情况：** 普通字符串 `stable\n\ndynamic`。
- **Runnable 输出** 为 `[SystemMessage?, ...body]`，其中 body 为：
  - 运行中摘要以 `HumanMessage` 为载体（`buildSummaryCarrierText`），在没有缓存提供商时置于前面；
  - 有缓存提供商时，一段**动态尾部**（`HumanMessage(dynamicInstructions)`，加上摘要载体）插入到最后一条人类消息之前；如果摘要在消息之前，则插入到位置 0；
  - 稳定前缀消息获得缓存标记（开头的消息从不作为锚点），尾部消息获得 `addTailCacheControl`。

这样最多保留四个缓存断点：工具、系统、稳定前缀和对话尾部。`CONTEXT.md` 和 ADR 0006 描述了压缩时缓存前缀的对齐方式。

---

## 4. 人机协同（`src/hitl/*`、`src/types/hitl.ts`、`src/tools/ToolNode.ts`、`src/run.ts`）

### 4.1 启用 HITL

`RunConfig.humanInTheLoop = {enabled: true}` 默认关闭。启用后：
- PreToolUse 返回 `ask` 时会调用 `interrupt()`；
- 如果没有提供检查点存储，会安装一个 `MemorySaver`。

关闭时，`ask` 会变成默认拒绝（fail-closed deny）：一条错误 ToolMessage 加上一次 `PermissionDenied` 钩子。

### 4.2 中断负载

- **工具审批**（由 `buildToolApprovalInterruptPayload` 构建，ToolNode.ts:514）。每个 ToolNode 批次一个中断，打包所有 `ask` 调用：
```ts
{ type: 'tool_approval',
  hook_session_id?: string,
  action_requests: [{ tool_call_id, name, arguments /*resolved + hook-rewritten*/, description? }],
  review_configs:  [{ action_name, tool_call_id, allowed_decisions: ('approve'|'reject'|'edit'|'respond')[] }],
  subagent?: { run_id, agent_id, subagent_type, parent_tool_call_id? } }   // bridged from child graphs
```
- **向用户提问。** `askUserQuestion(q, {toolCallId})`（`src/hitl/askUserQuestion.ts`）发起 `{type:'ask_user_question', question:{question, description?, options?:[{label,value}], multiSelect?}, tool_call_id?}`。恢复值为 `{answer: string}`；多选答案是用 `", "` 连接的值。
- **批量提问。** `askUserQuestions({questions})`（`src/hitl/askUserQuestions.ts`）：
  - 最多 `MAX_ASK_USER_QUESTIONS=4` 个问题；
  - id 匹配 `^[A-Za-z][A-Za-z0-9_-]{0,63}$` 且唯一；
  - `question` 携带第一个问题，作为旧版回退；
  - 恢复值 `{answers: {id: string}}` 会被严格校验。
- **类型守卫：** `isToolApprovalInterrupt`、`isAskUserQuestionInterrupt`、`isAskUserQuestionsInterrupt`。
- **宿主看到的内容。** `run.getInterrupt()` 返回 `{interruptId, threadId?, checkpointId?, checkpointNs?, payload}`，其中私有包装已被去除（`getPublicToolInterruptPayload`）。

### 4.3 决策（恢复值）

恢复值为 `ToolApprovalDecision[]`（按 `action_requests` 顺序）或 `Record<tool_call_id, ToolApprovalDecision>`（`normalizeApprovalDecisions`，ToolNode.ts:555）。

| 决策 | 效果（事件路径 ToolNode.ts 约 3930-4125；直接路径约 2614-2845） |
|---|---|
| `{type:'approve'}` | 用审阅过的参数执行。 |
| `{type:'reject', reason?}` | `blockEntry`：带原因的错误 ToolMessage，加上 `PermissionDenied`。 |
| `{type:'edit', updatedInput}` | `updatedInput` 必须是普通对象，否则调用默认拒绝。它通过 `applyInputOverride` 应用（会重新解析 `{{tool<i>turn<n>}}` 占位符），然后工具运行。 |
| `{type:'respond', responseText}` | `responseText` 必须是字符串。不执行工具：用截断后的文本创建一条成功的 ToolMessage，分发 `ON_RUN_STEP_COMPLETED`，**PostToolUse 不触发**，该调用会包含在 `PostToolBatch` 的条目中。 |

默认拒绝的情形：缺失的决策视为拒绝；未知类型或不在 `allowed_decisions` 内的决策会被阻止并附带诊断信息。如果当前提议（id、名称、稳定序列化的参数和允许的决策，按批次顺序）与审阅过的负载不再一致，`toolApprovalPayloadMatches` 失败，调用被阻止并提示 "Reviewed tool proposal changed…"（`src/hitl/approvalReview.ts`）。

### 4.4 恢复流程与重放语义

1. **暂停。**
   - ToolNode 在 `AsyncLocalStorageProviderSingleton.runWithConfig(config, …)` 内部调用 `interrupt(payload)`。这是必要的，因为 ToolNode 在关闭链路追踪的情况下运行。
   - ToolNode 包装器捕获 `GraphInterrupt`（ToolNode.ts:1160），并用私有状态**重新包装每个中断值**：`attachToolBatchReplayState` 保存已完成的同级结果（用 LangGraph 序列化器序列化）、子智能体轮次状态和输出引用状态；`attachRunStepResumeState` 保存运行步骤状态。
   - 然后重新抛出。这些记录作为中断值的一部分存在于检查点中。
2. **捕获。** `Run.processStream` 把 `__interrupt__` 数据块捕获到 `_interrupt`，然后调用 `resolveInterruptResumeConfig`。等待恢复期间**跳过 `clearHeavyState`**，并保留会话钩子。
3. **恢复。** `Run.resume(value, config, streamOptions?, {update?, goto?})`：
   - `restoreInterruptFromCheckpoint` 即使在重建的 `Run` 上也能工作：它调用 `getState(config, {subgraphs:true})` 找到第一个已持久化的中断，并恢复移交状态、运行步骤状态和消息。
   - 它设置私有的 configurable 键：`TOOL_APPROVAL_REVIEW_CONFIG_KEY`（审阅证据）和 `TOOL_BATCH_REPLAY_KEY`（来自 `restoreToolReplayConfig`），外加一个新的 `SUBAGENT_RESUME_ATTEMPT_CONFIG_KEY` 和一份子智能体恢复清单。
   - 它把 `hook_session_id` 指定的钩子会话复制到新的运行 id 下。
   - 它固定 `checkpoint_id`/`checkpoint_ns`，然后流式提交 `Command({resume: {[interruptId]: value}})`。
4. **重新执行。** LangGraph 从头重新运行被中断的 ToolNode。
   - 已完成的同级工具从重放记录中恢复，**不会**重新执行。
   - PreToolUse 钩子会再次触发。已消耗的 `once` 钩子结果从 `HookRegistry.pendingToolApprovals`（以 `{executionScope, agentId, toolUseId}` 为键）重放，因此一次性的 `ask` 钩子仍然返回 `ask`。
   - 当前的 `deny` 仍然优先。
   - 证据绑定到一个所有者（来自 `TOOL_APPROVAL_EXECUTION_SCOPE_CONFIG_KEY` 或线程的执行作用域，加上智能体、主体和对话）以及中断 id，并对照 LangGraph 的恢复映射和暂存区进行检查（docs/tool-approval-replay.md）。
5. **保证与限制。**
   - 触发中断的工具的函数体只运行一次。
   - 同级工具受重放记录保护；没有这些记录，它们会运行两次。
   - `interruptingToolNames` 让会在函数体中触发中断的工具先运行。
   - 对于跨崩溃的外部副作用，这**不是**恰好一次（exactly-once）语义。
6. **对检查点存储的要求。**
   - HITL 需要检查点存储；`MemorySaver` 只能在同一进程内工作。
   - 持久的跨进程恢复需要宿主自己的 saver、相同的 `thread_id`、相同的图配置和相同的检查点命名空间。
   - 已暂停的运行不能在不同 SDK 版本之间路由。

---

## 5. 钩子系统（`src/hooks/*`）

### 5.1 注册表、匹配器与执行

- **`HookRegistry`**（`src/hooks/HookRegistry.ts`）：
  - 全局匹配器加上按会话划分的桶（`Map<sessionId, bucket>`，会话 id 即运行 id）；
  - `register`/`registerSession` 返回一个注销函数；
  - `getMatchers(event, sessionId)` 先返回全局匹配器，再返回会话匹配器；
  - `removeMatcher`、`clearSession`、`copySession`（用于恢复），以及 `forkSession`（为分离运行的子智能体提供隔离快照）；
  - 停止信号：`haltRun`，每个会话首次写入生效，通过 `getHaltSignal`/`clearHaltSignal` 访问；
  - `pendingToolApprovals` 支持一次性审批的重放；
  - 当注册了 PreToolUse、PostToolUse 或 PostToolUseFailure 时，`hasResultAlteringHooks` 为 true，并会禁用工具急切执行；
  - `hasHookFor` 和 `hasDispatchableHookFor`（带有非空钩子的通配匹配器）。
- **`HookMatcher`：** `{pattern?, hooks: HookCallback[], timeout?, once?, internal?}`。`matchesQuery(pattern, query)`（`src/hooks/matchers.ts`）的规则如下：
  - 空模式总是匹配；
  - 没有查询值的事件只触发通配匹配器；
  - 模式最长 512 个字符，包含嵌套量词时被拒绝，编译结果放入 256 项的 LRU 缓存；无效模式永不匹配。
- **`executeHooks({registry, input, sessionId, matchQuery, onceReplayKey, onceReplaySessionId, signal, timeoutMs=30_000, logger})`**（`src/hooks/executeHooks.ts`）：
  - 在一段同步前缀中原子地移除 `once` 匹配器；
  - 并行运行所有匹配的钩子（`Promise.all`）；每个匹配器共享一个超时加父级中止信号，每个钩子都与该信号竞速；
  - 按注册顺序折叠结果：
    - `decision` 取 `deny > ask > allow`；
    - 只要有一个钩子阻止，`stopDecision` 就是 `block`；
    - `updatedInput`、`updatedOutput` 和 `allowedDecisions` 最后写入者生效；
    - `additionalContexts[]` 和 `injectedMessages[]` 累加；
    - 任一钩子设置了 `preventContinuation` 即设置；第一个 `stopReason` 生效；
    - 收集 `errors[]` 和 `hasHookFailures`；
    - 带 `async:true` 的输出被忽略。
  - 如果设置了 `preventContinuation` 且提供了 `sessionId`，它会调用 `registry.haltRun(sessionId, reason, event)`。
  - `mergeAggregatedHookResults` 折叠顺序执行的各阶段（先 Stop，再 StopFinalize）。

### 5.2 钩子目录

`BaseHookInput` 为 `{runId, threadId?, agentId?（仅在子智能体作用域中设置）, executingAgentId?, executionContext?}`。`BaseHookOutput` 为 `{additionalContext?, injectedMessages?, preventContinuation?, stopReason?, async?, asyncTimeout?}`。

| 事件 | 额外输入 | 额外输出 | 触发位置 | 效果 |
|---|---|---|---|---|
| `RunStart` | `messages` | – | run.ts:817，流式开始之前（恢复时不触发） | `additionalContext` 以 `HumanMessage{role:'system'}` 追加到输入中。`preventContinuation` 在图启动前停止运行。 |
| `UserPromptSubmit` | `prompt`、`attachments?` | `decision`、`reason` | run.ts:848，针对最后一条人类消息 | `deny` 产生停止原因 `prompt_denied`；`ask` 产生 `prompt_requires_approval`（停止，而不是中断）；会添加上下文。 |
| `PreToolUse`（matchQuery = 工具名） | `toolName`、`toolInput`、`toolUseId`、`stepId?`、`turn?` | `decision`、`reason`、`updatedInput`、`allowedDecisions` | ToolNode.ts:2519（直接）、3605（事件）；`src/tools/local/LocalProgrammaticToolCalling.ts:213` | `deny` 阻止调用并触发 PermissionDenied。`ask` 触发中断（启用 HITL 时）或拒绝。`updatedInput` 在审阅前改写参数。上下文进入批次上下文。 |
| `PostToolUse`（matchQuery） | `+toolOutput` | `updatedOutput` | ToolNode.ts:2921、4547 | 替换工具输出；会添加上下文。 |
| `PostToolUseFailure`（matchQuery） | `error` | – | ToolNode.ts:2883、4494 | 仅观察；会添加上下文。 |
| `PostToolBatch` | `entries: [{toolName, toolInput, toolUseId, stepId?, turn?, status, toolOutput?, error?}]` | – | ToolNode.ts:4832，在所有单工具钩子之后、下一次模型调用之前 | 批次中所有 `additionalContexts` 放入**一条** `HumanMessage{role:'system', source:'hook'}`，随后是 `convertInjectedMessages(injectedMessages)`，每条一个 HumanMessage。**这是工具边界的插话通道。** |
| `PreemptBoundary` | `sealCount` | – | Graph.ts:5040，在封存或重启之后（超时 120 秒，`PREEMPT_BOUNDARY_HOOK_TIMEOUT_MS`） | 注入的上下文和消息被追加，节点自循环。它自身的停止信号会被清除并保存为 `preemptHaltReason`。 |
| `PermissionDenied` | `toolName`、`toolInput`、`toolUseId`、`reason` | – | ToolNode.ts:3038、3708 | 仅观察。 |
| `SubagentStart`（matchQuery = 子智能体类型） | `parentAgentId?`、`agentId`、`agentType`、`inputs` | `decision`、`reason` | `src/tools/subagent/SubagentExecutor.ts:2876` | `deny` 或 `ask` 时以 `"Blocked: …"` 作为工具结果返回。 |
| `SubagentStop` | `agentId`、`agentType`、`messages` | – | SubagentExecutor.ts:3083 | 仅观察。 |
| `Stop` | `messages`、`stopReason?`、`stopHookActive`、`continuationCount`、`continuationBudgetRemaining` | `decision: 'continue'\|'block'`、`reason` | run.ts:1606，在自然完成后（中断或停止时不触发），钩子并行执行 | `block` 加上非空的注入消息或上下文会触发**终态运行续接**（在同一次 `processStream` 中再运行一段图）。有检查点存储时只提交增量；没有时发送完整记录。受 `maxStopContinuations`（默认 8）限制。 |
| `StopFinalize` | Stop 的字段，加上 `continuationPlanned`、`continuationPrevented` | 与 Stop 相同 | run.ts:1635，在 Stop 折叠之后顺序执行 | 持久化宿主做出一次“认领或封存”决定。失败会被断言（`assertFinalAdmissionSucceeded`）。 |
| `StopFailure` | `error`、`lastAssistantMessage?` | – | run.ts:1756，流抛出异常时 | 仅观察；错误会被重新抛出。 |
| `PreCompact` | `messagesBeforeCount`、`trigger` | – | `src/summarization/node.ts:1332` | 仅观察。 |
| `PostCompact` | `summary`、`messagesAfterCount` | – | `src/summarization/node.ts:923` | 仅观察。 |

从 `src/hooks/index.ts` 导出的能力标志：`HOOK_INJECTED_MESSAGES_CAPABLE`、`HOOK_PREEMPT_BOUNDARY_CAPABLE`、`HOOK_STOP_CONTINUATION_CAPABLE`、`HOOK_PREEMPT_RESTART_CAPABLE`。

`InjectedMessage`（`src/types/tools.ts:655`）为 `{role:'user'|'system', content, isMeta?, source?:'skill'|'hook'|'system'|'steer', skillName?}`。`convertInjectedMessages`（`src/messages/injected.ts`）：
- 总是生成 `HumanMessage`，带有 `additional_kwargs {role, injected:true, isMeta?, source?, skillName?}`；
- 丢弃空的或只含空白的条目；
- 对插话标记来源为 `user`，其余标记为 `synthetic`。

### 5.3 策略工厂

- **`createToolPolicyHook({mode='default'|'dontAsk'|'bypass', allow[], deny[], ask[], reason})`**（`src/hooks/createToolPolicyHook.ts`）。通配符使用锚定的 `*`。优先级：先 deny，再 ask，再 allow；然后 `bypass` 允许，`dontAsk` 拒绝，兜底为 `ask`。原因文本中的 `{tool}` 会被替换。
- **`createWorkspacePolicyHook({root, additionalRoots, outsideRead='ask', outsideWrite='ask', reason, pathExtractors})`**（`src/hooks/createWorkspacePolicyHook.ts`）：
  - 每个工具有自己的路径提取器（默认覆盖本地编码工具）；
  - 通过 realpath 对照解析后的根目录检查包含关系；
  - 读取类工具使用 `outsideRead`；写入类工具和未知工具使用 `outsideWrite`；
  - `ask` 会把审阅者的选项限制为 `allowedDecisions: ['approve','reject']`。

---

## 6. 抢占与插话引导

### 6.1 宿主契约

`RunConfig.preemption: StreamPreemption`（`src/types/run.ts:161`）有四个字段：
- `shouldPreempt()`：每个数据块轮询一次。它必须是同步的、O(1) 的，并且是**电平触发**的：在 `PreemptBoundary` 钩子清空队列之前一直返回 true。
- `subscribe?(wake)`：在静默或只有推理输出的窗口期提供唤醒提示。重启时必需。
- `restartGraceMs?`
- `maxSeals?`（由 `resolveMaxSeals` 规范化）。

抢占还需要一个可分发的 `PreemptBoundary` 匹配器，以及一个 `tokenCounter`，使被封存的轮次仍能报告用量。

### 6.2 图的闸门（Graph.ts:1440-1960）

- `canClaimPreemptSeal()` 要求：已配置抢占、没有正在进行的封存、仍有剩余预算、有运行 id，并且 `hasDispatchableHookFor('PreemptBoundary')`。
- `shouldPreemptStream()` 为 `canClaimPreemptSeal() && preemption.shouldPreempt()`。
- `claimPreemptSeal()` 和 `claimPreemptRestart()` 在一个同步步骤中占用同一个槽位和预算。这使它们对并行的多智能体通道是安全的：只有一个通道能获胜。
- 按智能体划分的集合 `pendingPreemptReturn` 和 `preemptRestartPending`。
- 计数器：`preemptSealCount`、`preemptRestartCount`、`preemptEmptyBoundaries`，通过 `getPreemptStats()` 暴露。标志：`preemptIncomplete` 和 `preemptHaltReason`。

### 6.3 流上的决策（`src/llm/preempt.ts`）

- **`canSealPreempt(chunk)`**，仅当累积的数据块可以安全截断时为 true：
  - 有非空白文本；
  - 没有未完成的 `tool_calls` 或 `tool_call_chunks`（已有结果块的 Anthropic 服务端工具调用视为已完成）；
  - 没有 `invalid_tool_calls`；
  - 没有未完成的 Gemini 服务端 `toolCall`。
- **`canRestartPreempt(chunk)`**，当没有值得保留的内容时为 true：
  - 完全没有数据块，或者只有推理块和空白文本块（按白名单检查）；
  - 没有任何类型的工具机制；
  - 没有 OpenAI Responses 提供商输出。
- **`resolvePreemptAction({chunk, requestAgeMs, graceMs})`** 优先选择封存。只有在 `requestAgeMs >= graceMs` 时才返回重启。

`src/llm/invoke.ts`（约 960-1440）执行该决策：
- **封存：** 中断流，标记 `response_metadata.preempted = true`，并合成模型结束事件和用量（`endSealedModelRun`）。
- **重启：** 拆除提供商流，取消打开的消息步骤，发出带 `preemptDiscarded` 的模型结束事件，调用 `notePreemptRestart(agentId)`，并返回 `{messages: []}`。

### 6.4 节点边界（Graph.ts:4929-5010）

调用返回后，如果 `consumePreemptRestart(agentId)` 为 true 或响应带有 `preempted`：
- 调用 `dispatchPreemptBoundary`，它把 `additionalContexts` 转换为 `HumanMessage{role:'system', isMeta, source:'hook'}`，把 `injectedMessages` 经 `convertInjectedMessages` 转换；
- 然后调用 `releasePreemptSeal()`。

四种结果：

| 结果 | 行为 |
|---|---|
| `preventContinuation` | 提交被封存的轮次及所有注入内容，不自循环。设置 `preemptIncomplete`；只有在封存且没有注入任何内容时才计为一次空边界。 |
| 有注入消息 | `pendingPreemptReturn.add(agentId)`，然后 `routeMessage` 回到智能体节点，模型从插话处继续。 |
| 重启后没有注入内容 | 回到循环并重新发出同一次调用。 |
| 封存后没有注入内容 | `preemptEmptyBoundaries += 1`，`preemptIncomplete = true`，路由到 END。`Run` 报告 `preempt_incomplete`。 |

### 6.5 插话通道小结

1. **工具边界：** `PostToolBatch` 返回带 `source:'steer'` 的 `injectedMessages`。
2. **生成过程中：** `PreemptBoundary`（封存或重启）。
3. **终态：** `Stop` 钩子返回 `block` 并带有注入消息。

持久化的插话是助手消息中的 `ContentTypes.STEER` 内容片段。`formatAgentMessages`（`src/messages/format.ts` 约 2025）把每一个重放为 `HumanMessage{source:'steer'}`，拆分助手消息，并在插话位于末尾时添加一个锚点。在线上请求层，`coalesceAdjacentUserTurns` 为严格交替的提供商（Mistral、Bedrock）合并相邻的用户轮次。

---

## 7. 事件 Actor 执行器（`src/eventActor/*`，`CONTEXT.md` 中的 “Event Actors”）

### 7.1 目的

运行一个稳定的逻辑子线程（**事件 Actor**），由它处理一系列权威的宿主事件，而无需在事件之间保持一个存活的执行器。
- 每个事件运行在一个隔离的**调用分叉（Invocation Fork）**上，即每次尝试一个检查点命名空间：`event-actor/<sha256(actorThreadId\0invocationId\0attemptId)[:32]>`。
- Actor 的**头部（Head）** `{actorThreadId, generation, checkpoint?}` 只能通过宿主对 generation 加检查点标识执行的原子比较并交换（CAS）来推进。

### 7.2 流程（`EventActorExecutor.execute`，EventActorExecutor.ts:1847）

1. **为事件做快照。** `snapshotEvent` 生成一份深度冻结的 JSON 副本：只允许有限数值，`-0` 变为 `0`，不允许循环、空洞或 symbol，键会排序。
2. **解析深度**，来自环境中的 `configurable.event_actor_depth`，受 `maxDepth`（默认 1）限制。
3. **准备。** `prepare()` 调用 `adapter.prepare`。它返回 `ready`（基于已提交检查点的热分叉）或 `checkpoint_unavailable`，后一种情况下 `coldContinue` 要求宿主根据记录和摘要重建分叉。
   - 已准备的调用带有 HMAC `preparationDigest`（密钥来自 `preparationSigningKey`，有效期受 `dormantCheckpointTtlMs` 限制，默认 24 小时）。
4. **调用。** `#invokeWithConfig` 构建一个清洗过的 `RunnableConfig`：
   - 移除父级的 `__pregel_*`、`__librechat_*`、运行、线程和检查点相关键；
   - 根据分叉设置 `thread_id` 和 `checkpoint_ns`，外加 `event_actor_*` 键；
   - 在 `runWithConfig` 下运行 `adapter.invoke(invocation, {signal, config})`。
5. **解读结果：**
   - 抛出的 `GraphInterrupt` 或父级 `Command` 原样向上传播。
   - 其他任何异常都是明确的“无动作”失败：`discard(fork)`，并返回 `failed` 或 `cancelled`。
   - `completed_no_action`：丢弃分叉。
   - `suspended`：经过认证的 `EventActorSuspension` 证据（版本 1、`suspensionDigest`、过期时间，最大 64 KB），通过 `adapter.suspend` 发布。后续调用为 `resume(...)`（使用 `resumeAttemptId` 的认领；再次暂停会原子地替换它）、`cancelSuspension` 和 `settleSuspension`。
   - `applied`：先 `#issueSettlement`，再通过宿主 CAS `commit`。结果为 `applied` 并带有新的头部（generation + 1）、`commit_conflict`（头部已过期；保留分叉以便对账）或 `commit_indeterminate`。

宿主适配器接口为 `EventActorHostAdapter`（`src/eventActor/types.ts`）：`prepare`、`coldContinue`、`invoke`、`commit`、`discard`，以及可选的 `suspend`、`resume`、`cancelSuspension`、`settleSuspension`。这是包裹在图运行外层的宿主基础设施；它不改变图拓扑。

---

## 8. 在 LangGraph Python 上重新实现

### 8.1 原语映射

| TS | Python |
|---|---|
| `Annotation.Root({...})` | 使用 `Annotated[T, reducer]` 的 `TypedDict` |
| `messagesStateReducer` 加上 `__remove_all__` 哨兵 | `langgraph.graph.message.add_messages` 加上 `RemoveMessage(id=REMOVE_ALL_MESSAGES)`（原生即为 `"__remove_all__"`）。需补上空数据块过滤，以及 AI 消息上预先分配的 uuid。 |
| `runStepState` 的 revision 归约器 | `lambda cur, upd: upd if upd["revision"] >= cur["revision"] else cur` |
| `handoffState` 合并归约器 | 移植 `merge_handoff_state`（按 id 取并集，冲突时抛出异常）。 |
| 新输入上的 `Overwrite(...)` | 较新版本中的 `langgraph.types.Overwrite`；否则在自己的归约器中处理一个哨兵包装。 |
| 把编译后的子图作为节点 | `builder.add_node("agent_id", compiled_subgraph_or_wrapper, destinations=(...))`。对于经 Command 路由的节点，`ends` 对应 `destinations`。 |
| `addConditionalEdges(agent, routeMessage)` | `add_conditional_edges("agent=X", route_message, [...])` |
| `addEdge([a,b], c)` 全部满足汇合 | `add_edge(["a","b"], "c")`（等待全部完成） |
| `Command({goto, update, graph: Command.PARENT})` | `Command(goto=..., update=..., graph=Command.PARENT)` |
| `Send(node, args)` | `langgraph.types.Send` |
| `interrupt(payload)` / `Command({resume})` | `langgraph.types.interrupt`、`Command(resume={interrupt_id: value})`（支持以中断 id 为键的映射） |
| `MemorySaver` / 持久化 saver | `InMemorySaver`、`AsyncPostgresSaver` / `AsyncSqliteSaver` / Redis |
| `durability: 'exit'` | `graph.astream(..., durability="exit")` |
| `getState(cfg, {subgraphs:true})`、`getStateHistory` | `aget_state(cfg, subgraphs=True)`、`aget_state_history(cfg)` |
| `streamEvents` 加自定义事件 | `astream_events(version="v2")` 与 `adispatch_custom_event`，或 `astream(stream_mode=["messages","custom","updates"])` 配合 `get_stream_writer()` |
| `AsyncLocalStorageProviderSingleton.runWithConfig` | `contextvars`（runnable config 变量）；在节点和工具内部，Python 的 `interrupt()` 会自动从上下文读取配置。 |
| `AbortSignal` / `AbortSignal.any` | `asyncio` 取消，加上基于 `asyncio.Event` 的组合信号；钩子超时用 `asyncio.wait_for` 或 `asyncio.timeout` |
| `PromptTemplate.fromTemplate('{results}')` | `langchain_core.prompts.PromptTemplate`（完全一致） |
| `getBufferString` | `langchain_core.messages.get_buffer_string` |
| `GraphRecursionError` / `recursionLimit` | `config["recursion_limit"]` |

### 8.2 建议的模块布局

```
lc_agents/
  run.py                    # Run：create/process_stream/resume/get_interrupt/get_halt_reason/
                            #   get_handoff_outcome；流式前钩子；Stop/StopFinalize 循环；停止信号轮询
  graphs/
    base.py                 # Graph：运行步骤 / 内容附属状态、步骤键、dispatch_run_step*
    standard.py             # StandardGraph：create_agent_node()、create_workflow()、route_message、call_model
    multi_agent.py          # MultiAgentGraph：边分类、可达性、并行组、
                            #   移交工具、agent_wrapper、扇入提示节点
    handoff.py              # HandoffRouting、merge_handoff_state、HandoffLimitError
    factory.py              # create_graph(kind, input)、apply_graph_runtime_config
  agents/
    context.py              # AgentContext（工具绑定、发现、预算、摘要、移交上下文）
    system_prompt.py        # 稳定 / 动态构建器、感知缓存的 SystemMessage、动态尾部
    projection.py           # project_agent_context_usage
  tools/
    tool_node.py            # 批量执行、钩子、HITL 中断、Command 聚合（Send 扇出）
    conditions.py           # tools_condition
    replay.py               # 批次重放记录、运行步骤恢复包装、审批证据
  hitl/
    types.py                # 负载 / 决策的 TypedDict；类型守卫
    approval.py             # 证据、提议匹配、normalize_decisions
    ask_user.py             # ask_user_question(s)
  hooks/
    types.py registry.py execute.py matchers.py
    policies/tool_policy.py policies/workspace_policy.py
  llm/
    invoke.py               # 流式尝试、回退、溢出恢复
    preempt.py              # can_seal/can_restart/resolve_action、重启注册表
  messages/
    reducer.py injected.py handoff_cue.py provenance.py alternation.py
  event_actor/
    types.py executor.py signing.py
```

### 8.3 移植时的陷阱

1. **恢复时节点重新执行。** Python 的 `interrupt()` 同样会让节点从头重新开始。需要移植重放记录：附加在中断值上的已完成同级结果，以及一次性钩子的 `pendingToolApprovals`。否则有副作用的同级工具会运行两次。保留 `interrupting_tool_names` 的顺序。Python 预置的 `ToolNode` 没有这些功能，因此要自己写 ToolNode。
2. **包装与剥离中断值。** 把私有的重放状态和运行步骤状态存放在中断值*内部*（用带哨兵键的包装字典）。在 `get_interrupt()` 中剥离它，并在恢复时将其还原到私有的 `configurable` 键中。永远不要信任宿主在这些键下提供的值：在新输入时删除它们（run.ts:1280）。
3. **来自嵌套子图的移交 Command。** 移交工具在智能体子图的 `tools=X` 中运行，并返回 `Command(graph=Command.PARENT, …)`。Python 支持这一点，但：
   - 外层节点必须声明 `destinations`；
   - 同一批次中的多个父级 Command 必须像 ToolNode 那样转换为 `Send` 扇出；
   - `HandoffRouting.finalize` 记录之后，必须从 update 中移除 `handoff_request`。
4. **消息合并。** 接收方的过滤视图只存在于它自己的那次调用中，子图返回的是完整列表。依赖按 id 的更新插入。确保每条消息在离开节点*之前*都有 id；SDK 自己会分配 v4 id（Graph.ts:4826）。
5. **`start_index` 语义。** 它在外层归约器的第一次写入时计算，等于宿主输入的大小。`{results}`、`exclude_results` 切片、裁剪以及“本次运行产生”的判定都依赖它。Python 的归约器是纯函数，因此要在流式开始前把它记录在 `Run` 或图对象上，而不是在归约器内部计算。
6. **每次运行可变的图对象。** JS 在节点内部修改 `self.config`、`AgentContext` 和许多映射，并依赖单线程的同步区段来实现 `claim_preempt_slot` 和 `once` 匹配器的移除。在 Python 中，要把所有东西放在同一个 asyncio 事件循环上，并且绝不在这些临界区内 await。节点不要使用线程执行器。
7. **钩子停止与流取消。** `preventContinuation` 会在注册表上发出停止信号，由流循环轮询。`PreemptBoundary` 必须清除它自己的停止信号，否则中断流会在归约器提交之前毁掉被封存的轮次。
8. **终态续接。** 有检查点存储时，只提交注入的增量，并推进流分段和步骤键。没有时，重新提交完整记录。续接和恢复时绝不运行 RunStart 或 UserPromptSubmit。
9. **递归余量。** 把 `max_seals` 加到 `recursion_limit` 上。对每次成员调用应用 `member_recursion_limit`。
10. **对话中间的系统消息。** Anthropic 和 Google 会拒绝它们。所有注入的上下文和路由提示都是带有 `additional_kwargs.role='system'`/`source` 的 `HumanMessage` 对象，并且对严格的提供商合并相邻的用户轮次。
11. **提示缓存。** 复现稳定 / 动态的拆分，以及放在最后一条人类消息之前的动态尾部，否则缓存命中率会崩溃。把工具缓存断点保留在最后一个静态工具上，这样新发现的延迟加载工具不会使前缀失效。
12. **并行抢占通道。** 待返回集合和重启集合以智能体 id 为键；同一时间只有一个通道能持有封存槽位。
13. **审批的默认拒绝规则。** 严格校验决策的形状。强制执行 `allowed_decisions`。把当前提议（参数用稳定 JSON 序列化）与审阅过的提议进行比较。把缺失的决策视为拒绝。
14. **事件 Actor。** 使用 `hmac` 和 `hashlib`，以及常量时间比较（`hmac.compare_digest`）。把事件冻结为规范化 JSON。从继承的配置中清除 `__pregel_*` 键。让 `GraphInterrupt` 和 `ParentCommand` 向上传播（在 Python 中，`GraphInterrupt` 和 `ParentCommand` 位于 `langgraph.errors`）。
