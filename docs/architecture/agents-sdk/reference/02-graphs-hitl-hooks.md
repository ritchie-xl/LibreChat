# @librechat/agents: Graphs, AgentContext, HITL, Hooks, Preemption & Event Actors

> Reference chapter for the [@librechat/agents SDK architecture](../README.md). Produced by static reading of
> [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) at `v3.9.3` (`2d5653d`). Paths are relative to
> that repository's root. Line numbers are approximate; check the code when a detail matters.

This report covers graph orchestration, AgentContext, HITL, hooks, preemption/steering, checkpoints and event actors in the TypeScript SDK, and maps each onto LangGraph Python. All paths are relative to the SDK repo root.

**Scope and one finding up front:**
- The report is based on reading `src/graphs/*`, `src/agents/*`, `src/hitl/*`, `src/hooks/*`, `src/llm/preempt.ts`, `src/eventActor/*`, the relevant parts of `src/run.ts`, `src/tools/ToolNode.ts` and `src/llm/invoke.ts`, the docs and ADRs, and `CONTEXT.md`.
- **`hide_sequential_outputs` does not appear anywhere in this SDK.** Output hiding is a LibreChat host concern. The SDK only supplies the raw material for it: `agentId`/`groupId` on run steps, `getContentPartAgentMap()` and `labelContentByAgent()`.

---

## 1. Single-agent graph (`StandardGraph`, `src/graphs/Graph.ts`)

### 1.1 Class shape and lifecycle

- **Classes.** `abstract class Graph` (Graph.ts:722) holds the per-run mutable state that sits beside the graph: run-step bookkeeping, handler and hook registries, HITL config, eager tool execution state, the tool-output reference registry and tool sessions. `StandardGraph extends Graph` (Graph.ts:1303) is the concrete implementation.
- **Construction.** `createGraph({kind:'standard'|'multi-agent', input})` (`src/graphs/createGraph.ts`) is the polymorphic factory. It injects `GraphFactoryDependencies` (`src/graphs/graphFactory.ts`) so that subagents can build child graphs of either kind.
- **Runtime config.** `applyGraphRuntimeConfig` (`src/graphs/applyGraphRuntimeConfig.ts`) copies seven run-scoped fields onto a graph: `hookRegistry`, `humanInTheLoop`, `toolOutputReferences`, `eagerEventToolExecution`, `codeSessionToolNames`, `interruptingToolNames`, `toolExecution`. `Run` uses it for top-level graphs and `createAgentNode` uses it for child graphs.
- **Agent contexts.** The constructor (Graph.ts:1527) builds one `AgentContext.fromConfig(...)` per `AgentInputs` into `agentContexts: Map<agentId, AgentContext>`. It seeds `calibrationRatio` and fading tiers. `defaultAgentId` is `summarizeOnlyAgentId ?? agents[0].agentId`.
- **Legacy/standard run path.** `Run.createLegacyGraph` (`src/run.ts:614`) only ever passes `agents[0]` into a standard graph.

### 1.2 State schema (two levels)

**Outer workflow** (`createWorkflow`, Graph.ts:5716), built with `Annotation.Root`:

| channel | reducer | notes |
|---|---|---|
| `messages: BaseMessage[]` | `messagesStateReducer` (`src/messages/reducer.ts`, upsert-by-id plus a `RemoveMessage(id='__remove_all__')` sentinel). It also mirrors the result into `this.messages` and records `this.startIndex = a.length + b.length` on the first write. | `startIndex` is the size of the host-supplied input. It drives `{results}` slicing, prune start and "run-produced" detection. |
| `manualSummary?: string` | last write wins | Summarize-only runs. |
| `runStepState: RunStepResumeState` | `update.revision >= current.revision ? update : current` (Graph.ts:714) | Checkpointed run-step and content-index sidecar used for HITL resume (`src/types/stream.ts:115`). |
| `handoffState?: HandoffState` | `mergeHandoffState` (`src/graphs/handoff.ts:41`): union by transition id, throws on conflict | Only meaningful for multi-agent graphs. |

**Inner agent subgraph** (`createAgentNode`, Graph.ts:5156):

| channel | reducer | notes |
|---|---|---|
| `messages` | Calls `agentContext.invalidateProviderProjectionForMessageUpdates(b)`, then `messagesStateReducer` | |
| `summarizationRequest?: {remainingContextTokens, agentId}` | last write wins | Set by the model node to detour to summarize. |
| `manualSummary`, `runStepState`, `handoffState` | same as outer | |

### 1.3 Node and edge topology (mermaid-ready)

Node names come from `GraphNodeKeys` (`src/common/enum.ts:104`): `AGENT='agent='`, `TOOLS='tools='`, `SUMMARIZE='summarize='`.

```
%% Outer (single-agent)
START --> <defaultAgentId>          %% node = compiled agent subgraph, ends:[END]
<defaultAgentId> --> END

%% Inner agent subgraph for agentId X
START --> agent=X
agent=X -- routeMessage --> agent=X        %% preempt self-loop (pendingPreemptReturn)
agent=X -- routeMessage --> summarize=X    %% state.summarizationRequest != null
agent=X -- routeMessage --> tools=X        %% toolsCondition: un-invoked tool_calls / attributable invalid calls
agent=X -- routeMessage --> END            %% no tool calls, or summarize-only run
summarize=X --> agent=X
tools=X --> agent=X                        %% or END when agentContext.toolEnd
```

In summarize-only mode the outer node is wrapped by `compactingAgentNode`. That wrapper prefixes `createRemoveAllMessage()` onto the result (`propagateManualCompaction`, Graph.ts:1623).

### 1.4 Routing (`routeMessage`, Graph.ts:5457)

Checks run in this order:
1. **Preempt self-loop.** `pendingPreemptReturn.delete(agentId)` returns `agentNode`. The sealed turn injected messages, so the model continues in the same Pregel run.
2. **Summarize detour.** `state.summarizationRequest != null` returns `summarizeNode`.
3. **Summarize-only run.** `summarizeOnlyAgentId != null` returns `END`.
4. **Tools.** `toolsCondition(state, toolNode, invokedToolIds)` (`src/tools/ToolNode.ts:6285`):
   - Routes to tools when the last AI message has `tool_calls` that are not all in `invokedToolIds`.
   - Also routes to tools when there are attributable `invalid_tool_calls` and every valid call is a provider-server call (`srvtoolu_` prefix).
   - Otherwise returns `END`.
5. **Truncation flag.** When the decision is `END` and `getTruncationStopReason(lastMessage)` is set, it sets `outputTruncatedIncomplete`. `Run` later reports this as `output_truncated`.

### 1.5 The agent node (`createCallModel`, Graph.ts:3225-5027)

The node is wrapped by `invokeWithRunStepState` (Graph.ts:5435). The wrapper:
- sets `this.config`;
- restores `runStepState` and `handoffRouting` from state;
- calls the model node;
- returns `runStepState: this.createRunStepResumeState()`, and clears `manualSummary` on ordinary runs.

Inside the node, in order:
1. **Circuit breaker.** Capture `breakerAbort`/`breakerEpoch` at entry. Rethrow immediately if a sibling already tripped the breaker.
2. **Preempt halt.** If `preemptHaltReason != null` (a `PreemptBoundary` hook halted), return `{messages: []}`. This turns every successor into a no-op.
3. **Tool discovery.** `extractToolDiscoveries(messages)` feeds `agentContext.markToolsAsDiscovered` (deferred tools found via tool search). Then await `tokenCalculationPromise`.
4. **Pruning and context budget.**
   - Build the pruner lazily with `createPruneMessages` (needs a `tokenCounter` and `maxContextTokens`).
   - Run it on `agentContext.getProviderProjectedMessages(messages)`.
   - This yields the pruned `context`, calibration ratio, fading tier and a context-usage snapshot (emitted as the `ON_CONTEXT_USAGE` event).
   - If summarization is enabled and `shouldTriggerSummarization(...)` fires, return `{summarizationRequest}`. The router then detours to the summarize node.
   - A summarize-only run returns its request here as well.
5. **Model construction.**
   - `getPreparedToolsForBinding` returns tools with a prompt-cache breakpoint on the last static tool (`src/llm/promptCacheTools.ts`).
   - `initializeModel({tools, provider, clientOptions})`, unless `overrideModel` is set.
   - If a system prompt exists, `model = agentContext.systemRunnable.pipe(model)`.
6. **Provider projections** (wire only; graph state is never mutated), in order:
   1. legacy content formatting;
   2. `projectToolMessagesForProvider` (bounds tool-call input size);
   3. Bedrock trailing-whitespace fix;
   4. `ensureThinkingBlockInMessages`;
   5. `foldToolBlocksForToollessAgent`;
   6. `appendInstructionlessHandoffCue`, and `appendPredecessorHandoffCue` for Anthropic-like providers (`src/messages/handoffCue.ts`);
   7. `annotateMessagesForLLM` (tool-output reference tags);
   8. synthetic-context compaction to fit the budget;
   9. `coalesceAdjacentUserTurns` for strict-alternation providers;
   10. cache markers, including `addBedrockTailCacheControl`;
   11. `sanitizeOrphanToolBlocks`;
   12. finally `prepareProviderRequest(...)`.

   Every step is wrapped by `contextPressure.trackProjection` so token attribution survives.
7. **Invoke.** `attemptInvoke({request, context: this, preemptAgentId}, config)` (`src/llm/invoke.ts`). It streams chunks through the stream handler, which dispatches `ON_RUN_STEP`, `ON_MESSAGE_DELTA`, `ON_REASONING_DELTA` and tool-call run steps as they arrive. On failure:
   - A stream-limit error or `PreparedSubagentError` trips the breaker and rethrows.
   - A context overflow runs overflow recovery (summarize detour) when summarization can shrink the prompt.
   - Anything else runs `tryFallbackProviders`.
8. **Post-invoke bookkeeping.**
   - Assign a v4 id to the response if it has none, and record it in `runProducedAiMessageIds`.
   - Fallback dispatch of reasoning/text/tool-call run steps when streaming was unavailable (`handleToolCalls`).
   - Update `currentUsage`/`lastCallUsage` (estimated usage is excluded from calibration).
9. **Preempt boundary** (see §6), then return `{messages: [aiMessage, ...injected]}`.

### 1.6 Tool node wiring (`initializeTools`, Graph.ts:2827)

Every agent gets a `CustomToolNode` (`src/tools/ToolNode.ts`).
- **Event-driven mode** (`agentContext.toolDefinitions` is non-empty):
  - The model is bound to schema-only tools (`createSchemaOnlyTools`).
  - Execution is dispatched to the host via the `ON_TOOL_EXECUTE` event.
  - Graph-managed tools in `graphTools` (handoff, subagent, host in-process tools) become `directToolNames` and run in process.
- **Legacy mode:** `tools` plus `graphTools` are invoked directly.
- **Options passed to every ToolNode:** `handoffRouting`, `hookRegistry`, `humanInTheLoop`, `interruptingToolNames`, `toolOutputRegistry` (one per run), `fileCheckpointer`, breaker accessors, and `restoreRunStepResumeState`/`createRunStepResumeState` callbacks.
- **Interrupting tools.** `interruptingToolNames` is a scheduling hint: those tools run before their direct siblings. When HITL is enabled, `subagent` is added to the set automatically.

The subagent tool is created in `createAgentNode` (Graph.ts:5173-5424) when `subagentConfigs` exist and depth > 0. Its schema is `{description, subagent_type, subagent_thread_id?, run_in_background?}`, it is backed by `SubagentExecutor`, and it is pushed into `graphTools`, which triggers a token recount.

### 1.7 Recursion limit and end conditions

- **Recursion limit.** `Run.processStream` (run.ts:1273) sets `recursionLimit = (caller.recursionLimit ?? DEFAULT_RECURSION_LIMIT=50) + resolveMaxSeals(preemption.maxSeals)` when preemption is configured. The extra headroom covers one superstep per seal. `MultiAgentGraph` can also set `memberRecursionLimit` on each member subgraph invocation.
- **End conditions:** no tool calls; `toolEnd`; summarize-only; preempt halt; a hook halt, which `processStream` polls between stream events via `hookRegistry.getHaltSignal(runId)` and then breaks; an interrupt; `HandoffLimitError`; and stream-limit breakers.

### 1.8 Run-step and content state

- `contentData: RunStep[]`, `stepKeyIds`, `contentIndexMap`, `toolCallStepIds`, `pendingToolCallsByStep`, `openMessageStepByAgent`, and more (Graph.ts:784-878).
- **Step keys** are built from `[run_id, thread_id, langgraph_node, langgraph_step, checkpoint_ns, streamSegment]` (Graph.ts:2535), plus the invoked-tool count, plus `reasoning` or `post-reasoning-N`.
- **Agent resolution.** `getAgentContext(metadata)` parses `metadata.langgraph_node` by stripping the `agent=`/`tools=`/`summarize=` prefix (Graph.ts:2425).
- **Accessors:** `getRunSteps(agentId?)`, `getRunStepsByAgent()`, `getContentParts()`, `getContentPartAgentMap()`.
- **Serialization for resume.** `createRunStepResumeState()`/`restoreRunStepResumeState()` serialize these structures into the `runStepState` channel. This lets a rebuilt `Run` resume with the same step ids.

### 1.9 Checkpoint usage

- **Checkpointer.** `compileOptions.checkpointer` comes from the host. With HITL enabled and no checkpointer supplied, `Run.applyHITLCheckpointerFallback` installs a `MemorySaver` (run.ts:745).
- **Durability.** `config.durability = 'exit'` by default whenever a checkpointer exists (run.ts:1454), so state is persisted only at the exit or interrupt boundary.
- **Fresh input on a checkpointed thread** uses `Overwrite(...)` for `handoffState` and `runStepState` (run.ts:1500), so old sidecars are replaced rather than merged.
- **Resume** reads `getState(config, {subgraphs:true})`, restores `handoffRouting`, `runStepState` and messages, pins `checkpoint_id`/`checkpoint_ns` to the interrupt's checkpoint, and streams `new Command({resume: {[interruptId]: value}, update?, goto?})` (run.ts:2033-2231).

---

## 2. Multi-agent graph (`src/graphs/MultiAgentGraph.ts`)

### 2.1 Inputs and validation

`MultiAgentGraphInput` (`src/types/graph.ts`) adds these fields to the standard input:
- `edges: GraphEdge[]`
- `entryAgentId?`
- `maxHandoffs?`
- `resultAgentId?` (captures that member's last new AI message into `subagentResult`)
- `memberRecursionLimit?`

`GraphEdge` is `{from: string|string[], to: string|string[], description?, condition?(state) => bool|string|string[], edgeType?: 'handoff'|'direct', handoffScope?: 'turn'|'conversation', prompt?: string | (messages, runStartIndex) => string|Promise<string>|undefined, excludeResults?, promptKey?}`.

The constructor (lines 355-424) runs, in order:
- `validateEdgeAgents`: unknown agent ids throw early.
- `categorizeEdges` (line 534): an edge is **direct** when `edgeType==='direct'`, or when it has no `edgeType`, no condition, a single source and more than one destination. **Everything else is handoff**, including multi-source edges without an explicit `edgeType`.
- `validateCommandRoutedDirectEdges`:
  - A grouped (all-of) direct edge cannot include a handoff source.
  - A prompted direct edge cannot have a Command-routed source.
- `analyzeGraph`: starting nodes are agents with no incoming edge (fallback: the first agent), then `computeParallelCapability`.
- `entryAgentId` overrides the starting nodes, and `resolveEntryReachability` restricts compilation to reachable agents. It rejects unsatisfiable grouped joins.
- `maxHandoffs` validation, then `HandoffRouting(entry, maxHandoffs, parallel)`.
  - The graph is marked parallel if there are several starting nodes or any direct fan-out.
- `validateHandoffScopes`, then `createHandoffTools`.

**Parallel group ids** (`computeParallelCapability`, line 635): a BFS assigns incrementing `groupId`s to fan-out destinations; several starting nodes form group 1. They are exposed via `getParallelGroupIdForAgent` and stamped on run steps.

### 2.2 Handoff tools (`createHandoffToolsForEdge`, line 831)

- **Reserved names.** Existing `lc_transfer_to_*` and `conditional_transfer` tools are stripped from `graphTools` first.
- **Per destination:** a tool `lc_transfer_to_<dest>` (`Constants.LC_TRANSFER_TO_`).
  - Schema: `{[promptKey ?? 'instructions']: string}` when `edge.prompt` is a string (the prompt becomes the parameter description); otherwise empty.
  - The body builds `ToolMessage("Successfully transferred to X[\n\nInstructions: ...]")` with `additional_kwargs.handoff_source_name` and `handoff_instructions`.
  - It returns `new Command({goto: dest, graph: Command.PARENT, update: {messages, handoffRequest: {sourceAgentId, targetAgentId, toolCallId, scope}}})`.
  - When the originating AI message has several tool calls, `messages` is filtered to *only this call's* AI message plus its ToolMessage, keeping provenance. This keeps parallel handoffs provider-valid.
- **Conditional edges:** a single tool `conditional_transfer`. It evaluates `edge.condition(runtime.state)`:
  - `false` means no transfer (returns `null`);
  - a string or array selects the destination, which must be declared;
  - it stores `handoff_destination` in `additional_kwargs`.

### 2.3 ToolNode side of handoffs (ToolNode.ts ~5960-6090)

- **Aggregation.** Commands with `graph === PARENT` and a single destination are collected.
  - One handoff passes through unchanged.
  - Several handoffs in one batch become **`Send(dest, cmd.update)` fan-out**. Each handoff ToolMessage gets `handoff_parallel_siblings`, `__handoff_parallel_batch` and `__handoff_group_id` (a runtime group id).
  - Send-only Commands are merged.
- **Finalize.** Then `handoffRouting.finalize(commands, input, config)` (`handoff.ts:166`) runs:
  - Each `handoffRequest` becomes a `HandoffTransition` with a deterministic id `JSON.stringify([executionId, checkpoint_ns, sourceAgentId, lastMessage.id, toolCallId])` and a `depth` (so replay is idempotent).
  - It enforces `maxHandoffs` (`HandoffLimitError`, which `Run` reports as `handoff_limit`).
  - It deletes `handoffRequest` and stamps `handoffState` snapshots into every Command and Send.
- **Outcome.** `HandoffRouting.outcome()` gives `candidate | unchanged | ambiguous | incomplete`. It is exposed through `Run.getHandoffOutcome()` (docs/multi-agent-patterns.md). Only `conversation`-scoped transitions, in non-parallel graphs, can produce a candidate.

### 2.4 Outer workflow (`createWorkflow`, line 1376)

The state adds these channels to the base:
- `agentMessages: BaseMessage[]` (replace reducer), used by `excludeResults`;
- `subagentResult` (last write wins);
- `manualSummary`, `runStepState`, `handoffState`.

The `messages` reducer again sets `startIndex` on the first write.

```
%% per reachable agent A: node A = agentWrapper(A) around createAgentNode(A) subgraph
%% ends(A) = handoffDests(A) ∪ directDests(A) ∪ {END if hasHandoff || !hasDirect}
START --> s                       %% for each starting node s (or summarizeOnly agent only)
%% direct edge without prompt (static sources only; sources that also have handoff edges are skipped):
A --> B                           %% single source
[A1,A2] --> B                     %% grouped all-of join (addEdge(array, dest))
%% direct edges into B where any edge has a prompt:
A1 --> fan_in_B_prompt ; A2 --> fan_in_B_prompt ; fan_in_B_prompt --> B
%% handoffs: dynamic, via Command(graph=PARENT, goto=B) or Send(B, update) from the tools=A node
%% agents with BOTH handoff and direct edges: the wrapper returns Command(goto=handoffDest) if the last
%%   message is a lc_transfer_to_* ToolMessage, else Command(goto=directDests)  (exclusive routing)
```

### 2.5 `agentWrapper` (line 1484): per-agent input shaping

1. Restore `handoffRouting` from state. Apply `memberRecursionLimit`.
2. **`processHandoffReception(messages, agentId)`** (line 1129):
   - A handoff is "active" only if the most recent AI turn contained transfer calls and a transfer ToolMessage in the trailing tool block targets this agent.
   - It extracts instructions (structured `handoff_instructions`, else legacy `Label:` parsing), `sourceAgentName`, sibling names and the group id.
   - It strips **all** transfer ToolMessages, transfer `tool_calls` and transfer `tool_use` content blocks, so the recipient never sees handoff noise.
3. **Config metadata:**
   - `withActiveAgentMetadata`: `metadata.activeAgentId` / `activeAgentName`;
   - `withInstructionlessHandoffCue`: records the incoming AI message id, so that when the handoff carried no instructions a user cue `"Continue as the receiving agent…"` is appended to the provider payload only;
   - `withHandoffGroupMetadata` (`__handoff_group_id`).
4. **Handoff context.** `agentContext.setHandoffContext(source, siblings)` marks the system prompt stale so it rebuilds with a "Multi-Agent Workflow" identity preamble. Otherwise `clearHandoffContext()`.
5. **If a handoff was received with instructions:**
   - append a routing `HumanMessage` (`additional_kwargs {role:'user', isMeta:true, source:'routing'}`, stamped synthetic), bounded by the agent's remaining budget (`resolveMaxRoutingPromptChars`);
   - if the last filtered message is a ToolMessage, first insert the synthetic bridge `AIMessage("[Processed tool result and transferring to X]")`;
   - rebuild the token map;
   - invoke the subgraph with the transformed messages.
6. **Else if `state.agentMessages` is non-empty** (excludeResults path): invoke with `messages = agentMessages`, then clear `agentMessages`.
7. **Else** invoke with the full state.
8. `resultAgentId` captures `getLastNewAiMessage(result.messages, inputMessages)` into `subagentResult`.

**Important reducer semantics.** The subgraph returns its *entire* messages array, and the outer reducer upserts by id. The filtered, transfer-free view the recipient saw does **not** delete the transfer messages from the outer state, because they are still present with their original ids. A Python port must rely on the same id-based merge in `add_messages`.

### 2.6 Direct-edge prompts and `{results}` (line 1756)

The `fan_in_<dest>_prompt` node behaves as follows:
- **Prompt as a function:** `await prompt(state.messages, startIndex)`, bounded.
- **String containing `{results}`:**
  - `results = getBufferString(state.messages.slice(startIndex))`, bounded;
  - rendered with `PromptTemplate.fromTemplate(prompt)`;
  - `effectiveExcludeResults = excludeResults !== false && promptText !== ''`, so results are excluded by default.
- **Plain string:** used as is.
- **Output:**
  - Not excluding: `{messages: [routingPrompt]}`.
  - Excluding: `{messages: [routingPrompt], agentMessages: messagesStateReducer(messages[0:startIndex], [routingPrompt])}`. The destination sees only the original input plus the prompt, while the global transcript keeps everything.

### 2.7 Agent identity in messages and content

- **Run steps:** `runStep.agentId` and `runStep.groupId` are set only when `isMultiAgentGraph()` (Graph.ts ~5886). The agent is resolved from `langgraph_node`. The group id is resolved from the runtime `__handoff_group_id` metadata, or the static parallel groups.
- **Open message steps** are tracked per agent lane (`openMessageStepByAgent`).
- **Tool messages:** `handoff_source_name`, `handoff_parallel_siblings`, `handoff_destination`.
- **System prompt:** the identity preamble (§3).
- **Host-side replay helper:** `labelContentByAgent(contentParts, agentIdMap, agentNames, {labelNonTransferContent})` (`src/messages/format.ts:2330`). It folds a transferred agent's content into the transfer tool call's `output` as `--- Transfer to X --- … --- End of X response ---`. `ContentTypes.AGENT_UPDATE` content parts (`on_agent_update`) are aggregated by `src/stream.ts`, but the event is emitted by the host.
- **Prefill protection:** `appendPredecessorHandoffCue` adds a wire-only user cue when the payload ends on an assistant turn that *this run* produced (a direct-edge successor).

---

## 3. AgentContext (`src/agents/AgentContext.ts`) and system-prompt assembly

### 3.1 Responsibilities (one mutable object per agent per graph)

- **Identity and provider:** `agentId`, `name`, `endpoint`, `provider`, `clientOptions`, `langfuse`, `reasoningKey`, `toolEnd`, `useLegacyContent`.
- **Tools:**
  - `tools`/`toolMap` are host instances;
  - `graphTools` are SDK-managed tools (handoff, subagent) and are copied, never aliased;
  - `toolDefinitions` (event-driven schemas) and `toolRegistry` (per-tool `defer_loading` and `allowed_callers`);
  - `discoveredToolNames` holds deferred tools found via tool search;
  - `getToolsForBinding()` applies the caller-capability projection: bind only definitions that allow the `'direct'` caller and are active (non-deferred or discovered).
- **Token budget:**
  - `maxContextTokens`, `indexTokenCountMap`/`baseIndexTokenCountMap`, `systemMessageTokens`, `dynamicInstructionTokens`, `toolSchemaTokens`, `toolTokenCounts`, `instructionTokens` (a getter), `calibrationRatio` (an EMA), `fadingTier`, `pruneMessages`, `currentUsage`, `lastCallUsage`;
  - `calculateInstructionTokens` (line 1384) JSON-schemas each bound tool and counts it with a multiplier (Anthropic vs default), apportioned with the largest-remainder method;
  - `getTokenBudgetBreakdown`, `projectContextUsage` (also used by `src/agents/projection.ts::projectAgentContextUsage`, which gives a host a pre-send estimate without a model call).
- **Summaries:**
  - `setInitialSummary` is a cross-run summary placed in the system prompt;
  - `setSummary` is a mid-run summary placed as a user-message carrier;
  - it also tracks summarization triggers, failures, overflow recovery and fading-tier resets.
- **Handoff context:** `setHandoffContext` / `clearHandoffContext`.
- **Programmatic tool-calling guidance:** built from the caller-capability projection.
- **`reset()`** (line 1255) is a per-run reset. It restores the durable summary, clears discovered tools and handoff context, and recomputes tokens.

### 3.2 System-prompt assembly (`systemRunnable`, lines 799-1250)

The prompt is cached until marked stale. `systemRunnableStale` is set by handoff-context changes, new tool discoveries and summary changes.

- **Stable part** (`buildStableInstructionsString`), joined with `\n\n`:
  1. the identity preamble, present only with a handoff context: `## Multi-Agent Workflow\nYou are "<name>", transferred from "<source>".\n[Running in parallel with: …]\nExecute only tasks relevant to your role…`;
  2. `instructions`;
  3. the `## Programmatic Tool Calling` section.
- **Dynamic part** (`buildDynamicInstructionsString`):
  1. `additional_instructions`;
  2. `## Conversation Summary\n\n<summary>` when the summary location is `system_prompt` (the cross-run initial summary).
- **SystemMessage shape** (`buildSystemMessage`):
  - **Anthropic with `promptCache`:** content blocks `[{text: stable, cache_control: ephemeral(ttl default 1h)}, {text: dynamic}]`. When both stable and dynamic are non-empty and a cache provider is active, the dynamic text is *moved out* of the system message (`shouldMoveDynamicInstructions`).
  - **OpenRouter:** the same pattern with `cache_control`.
  - **Bedrock with `promptCache`:** `[{text: stable}, {cachePoint}, {text: dynamic}]`.
  - **Otherwise:** a plain string `stable\n\ndynamic`.
- **Runnable output** `[SystemMessage?, ...body]`, where the body is:
  - the mid-run summary as a `HumanMessage` carrier (`buildSummaryCarrierText`), prepended when there is no cache provider;
  - with a cache provider, a **dynamic tail** (`HumanMessage(dynamicInstructions)`, plus the summary carrier) inserted before the last human message, or at 0 when the summary precedes the messages;
  - stable-prefix messages get cache markers (the opening message is never an anchor), and the trailing messages get `addTailCacheControl`.

This keeps up to four cache breakpoints: tools, system, stable prefix and conversation tail. `CONTEXT.md` and ADR 0006 describe the cache-prefix alignment for compaction.

---

## 4. Human-in-the-loop (`src/hitl/*`, `src/types/hitl.ts`, `src/tools/ToolNode.ts`, `src/run.ts`)

### 4.1 Enabling HITL

`RunConfig.humanInTheLoop = {enabled: true}` is off by default. When enabled:
- a PreToolUse `ask` raises `interrupt()`;
- a `MemorySaver` is installed if no checkpointer was supplied.

When disabled, `ask` becomes a fail-closed deny: an error ToolMessage plus a `PermissionDenied` hook.

### 4.2 Interrupt payloads

- **Tool approval** (built by `buildToolApprovalInterruptPayload`, ToolNode.ts:514). One interrupt per ToolNode batch, bundling every `ask` call:
```ts
{ type: 'tool_approval',
  hook_session_id?: string,
  action_requests: [{ tool_call_id, name, arguments /*resolved + hook-rewritten*/, description? }],
  review_configs:  [{ action_name, tool_call_id, allowed_decisions: ('approve'|'reject'|'edit'|'respond')[] }],
  subagent?: { run_id, agent_id, subagent_type, parent_tool_call_id? } }   // bridged from child graphs
```
- **Ask user a question.** `askUserQuestion(q, {toolCallId})` (`src/hitl/askUserQuestion.ts`) raises `{type:'ask_user_question', question:{question, description?, options?:[{label,value}], multiSelect?}, tool_call_id?}`. The resume value is `{answer: string}`; multiSelect answers are values joined with `", "`.
- **Batched questions.** `askUserQuestions({questions})` (`src/hitl/askUserQuestions.ts`):
  - up to `MAX_ASK_USER_QUESTIONS=4` questions;
  - ids match `^[A-Za-z][A-Za-z0-9_-]{0,63}$` and are unique;
  - `question` carries the first question as a legacy fallback;
  - the resume value `{answers: {id: string}}` is validated strictly.
- **Type guards:** `isToolApprovalInterrupt`, `isAskUserQuestionInterrupt`, `isAskUserQuestionsInterrupt`.
- **What the host sees.** `run.getInterrupt()` returns `{interruptId, threadId?, checkpointId?, checkpointNs?, payload}` with private wrappers stripped (`getPublicToolInterruptPayload`).

### 4.3 Decisions (resume value)

The resume value is either `ToolApprovalDecision[]` (in `action_requests` order) or `Record<tool_call_id, ToolApprovalDecision>` (`normalizeApprovalDecisions`, ToolNode.ts:555).

| decision | effect (event path, ToolNode.ts ~3930-4125; direct path ~2614-2845) |
|---|---|
| `{type:'approve'}` | Execute with the reviewed args. |
| `{type:'reject', reason?}` | `blockEntry`: error ToolMessage with the reason, plus `PermissionDenied`. |
| `{type:'edit', updatedInput}` | `updatedInput` must be a plain object, or the call fails closed. It is applied through `applyInputOverride`, which re-resolves `{{tool<i>turn<n>}}` placeholders, and the tool runs. |
| `{type:'respond', responseText}` | `responseText` must be a string. No execution: a successful ToolMessage is created from the truncated text, `ON_RUN_STEP_COMPLETED` is dispatched, **PostToolUse does not fire**, and the call is included in the `PostToolBatch` entries. |

Fail-closed cases: a missing decision is treated as a reject; an unknown type, or a decision outside `allowed_decisions`, blocks with a diagnostic. If the current proposal (id, name, stable-stringified args and allowed decisions, in batch order) no longer matches the reviewed payload, `toolApprovalPayloadMatches` fails and the call is blocked with "Reviewed tool proposal changed…" (`src/hitl/approvalReview.ts`).

### 4.4 Resume flow and replay semantics

1. **Pausing.**
   - The ToolNode calls `interrupt(payload)` inside `AsyncLocalStorageProviderSingleton.runWithConfig(config, …)`. This is needed because ToolNode runs with tracing off.
   - The ToolNode wrapper catches `GraphInterrupt` (ToolNode.ts:1160) and **rewraps each interrupt value** with private state: `attachToolBatchReplayState` stores settled sibling results (serialized with the LangGraph serializer), the subagent turn state and the output-reference state; `attachRunStepResumeState` stores the run-step state.
   - It then rethrows. These records live in the checkpoint as part of the interrupt value.
2. **Capture.** `Run.processStream` captures `__interrupt__` chunks into `_interrupt`, then calls `resolveInterruptResumeConfig`. **`clearHeavyState` is skipped** while awaiting resume, and session hooks are preserved.
3. **Resuming.** `Run.resume(value, config, streamOptions?, {update?, goto?})`:
   - `restoreInterruptFromCheckpoint` works even on a rebuilt `Run`: it calls `getState(config, {subgraphs:true})` to find the first persisted interrupt and restores handoff state, run-step state and messages.
   - It sets private configurable keys: `TOOL_APPROVAL_REVIEW_CONFIG_KEY` (reviewed evidence) and `TOOL_BATCH_REPLAY_KEY` (from `restoreToolReplayConfig`), plus a fresh `SUBAGENT_RESUME_ATTEMPT_CONFIG_KEY` and a subagent resume manifest.
   - It copies the hook session named by `hook_session_id` into the new run id.
   - It pins `checkpoint_id`/`checkpoint_ns`, then streams `Command({resume: {[interruptId]: value}})`.
4. **Re-execution.** LangGraph re-runs the interrupted ToolNode from the start.
   - Completed siblings are restored from the replay record and are **not** re-executed.
   - PreToolUse hooks fire again. Consumed `once` hook contributions are replayed from `HookRegistry.pendingToolApprovals` (keyed by `{executionScope, agentId, toolUseId}`), so a one-shot `ask` hook still yields `ask`.
   - A current `deny` still wins.
   - Evidence is bound to an owner (execution scope from `TOOL_APPROVAL_EXECUTION_SCOPE_CONFIG_KEY` or the thread, plus agent, principal and conversation) and to the interrupt id, checked against LangGraph's resume map and scratchpad (docs/tool-approval-replay.md).
5. **Guarantees and limits.**
   - The interrupting tool's body runs once.
   - Siblings are protected by the replay records; without them they would run twice.
   - `interruptingToolNames` orders body-interrupting tools first.
   - This is **not** exactly-once for external side effects across crashes.
6. **Checkpointer requirements.**
   - HITL requires a checkpointer; `MemorySaver` works in-process only.
   - Durable cross-process resume requires the host's own saver, the same `thread_id`, the same graph configuration and the same checkpoint namespace.
   - Paused runs must not be routed between SDK versions.

---

## 5. Hook system (`src/hooks/*`)

### 5.1 Registry, matchers and execution

- **`HookRegistry`** (`src/hooks/HookRegistry.ts`):
  - global matchers plus per-session buckets (`Map<sessionId, bucket>`, where the session id is the run id);
  - `register`/`registerSession` return an unregister function;
  - `getMatchers(event, sessionId)` returns global matchers first, then session matchers;
  - `removeMatcher`, `clearSession`, `copySession` (for resume), and `forkSession` (an isolated snapshot for detached subagents);
  - halt signals: `haltRun`, first write wins per session, via `getHaltSignal`/`clearHaltSignal`;
  - `pendingToolApprovals` supports one-shot approval replay;
  - `hasResultAlteringHooks` is true when PreToolUse, PostToolUse or PostToolUseFailure are registered, and disables eager tool execution;
  - `hasHookFor` and `hasDispatchableHookFor` (a wildcard matcher with non-empty hooks).
- **`HookMatcher`:** `{pattern?, hooks: HookCallback[], timeout?, once?, internal?}`. `matchesQuery(pattern, query)` (`src/hooks/matchers.ts`) works as follows:
  - an empty pattern always matches;
  - a query-less event only fires wildcard matchers;
  - patterns are capped at 512 characters, rejected if they contain nested quantifiers, and compiled into a 256-entry LRU cache; an invalid pattern never matches.
- **`executeHooks({registry, input, sessionId, matchQuery, onceReplayKey, onceReplaySessionId, signal, timeoutMs=30_000, logger})`** (`src/hooks/executeHooks.ts`):
  - in a synchronous prefix, removes `once` matchers atomically;
  - runs every matching hook in parallel (`Promise.all`); each matcher shares one timeout plus parent abort signal, and each hook is raced against that signal;
  - folds results in registration order:
    - `decision` uses `deny > ask > allow`;
    - `stopDecision` is `block` if any hook blocks;
    - `updatedInput`, `updatedOutput` and `allowedDecisions` are last-writer-wins;
    - `additionalContexts[]` and `injectedMessages[]` accumulate;
    - `preventContinuation` is set if any hook sets it; the first `stopReason` wins;
    - `errors[]` and `hasHookFailures` are collected;
    - outputs with `async:true` are ignored.
  - If `preventContinuation` is set and a `sessionId` was given, it calls `registry.haltRun(sessionId, reason, event)`.
  - `mergeAggregatedHookResults` folds serialized phases (Stop, then StopFinalize).

### 5.2 Hook catalog

`BaseHookInput` is `{runId, threadId?, agentId? (set only in subagent scope), executingAgentId?, executionContext?}`. `BaseHookOutput` is `{additionalContext?, injectedMessages?, preventContinuation?, stopReason?, async?, asyncTimeout?}`.

| Event | Extra input | Extra output | Fires at | Effect |
|---|---|---|---|---|
| `RunStart` | `messages` | – | run.ts:817, before streaming (not on resume) | `additionalContext` is appended to the input as `HumanMessage{role:'system'}`. `preventContinuation` stops the run before the graph starts. |
| `UserPromptSubmit` | `prompt`, `attachments?` | `decision`, `reason` | run.ts:848, on the last human message | `deny` gives halt reason `prompt_denied`; `ask` gives `prompt_requires_approval` (halt, not interrupt); context is added. |
| `PreToolUse` (matchQuery = tool name) | `toolName`, `toolInput`, `toolUseId`, `stepId?`, `turn?` | `decision`, `reason`, `updatedInput`, `allowedDecisions` | ToolNode.ts:2519 (direct), 3605 (event); `src/tools/local/LocalProgrammaticToolCalling.ts:213` | `deny` blocks and fires PermissionDenied. `ask` interrupts (with HITL) or denies. `updatedInput` rewrites args before review. Context goes to the batch context. |
| `PostToolUse` (matchQuery) | `+toolOutput` | `updatedOutput` | ToolNode.ts:2921, 4547 | Replaces the tool output; context is added. |
| `PostToolUseFailure` (matchQuery) | `error` | – | ToolNode.ts:2883, 4494 | Observational; context is added. |
| `PostToolBatch` | `entries: [{toolName, toolInput, toolUseId, stepId?, turn?, status, toolOutput?, error?}]` | – | ToolNode.ts:4832, after all per-tool hooks and before the next model call | All batch `additionalContexts` go into **one** `HumanMessage{role:'system', source:'hook'}`, followed by `convertInjectedMessages(injectedMessages)`, one HumanMessage each. **This is the tool-boundary steering channel.** |
| `PreemptBoundary` | `sealCount` | – | Graph.ts:5040 after a seal or restart (timeout 120 s, `PREEMPT_BOUNDARY_HOOK_TIMEOUT_MS`) | Injected context and messages are appended and the node self-loops. Its own halt is cleared and stored as `preemptHaltReason`. |
| `PermissionDenied` | `toolName`, `toolInput`, `toolUseId`, `reason` | – | ToolNode.ts:3038, 3708 | Observational. |
| `SubagentStart` (matchQuery = subagent type) | `parentAgentId?`, `agentId`, `agentType`, `inputs` | `decision`, `reason` | `src/tools/subagent/SubagentExecutor.ts:2876` | `deny` or `ask` returns `"Blocked: …"` as the tool result. |
| `SubagentStop` | `agentId`, `agentType`, `messages` | – | SubagentExecutor.ts:3083 | Observational. |
| `Stop` | `messages`, `stopReason?`, `stopHookActive`, `continuationCount`, `continuationBudgetRemaining` | `decision: 'continue'\|'block'`, `reason` | run.ts:1606 after natural completion (not on interrupt or halt), hooks in parallel | `block` plus non-empty injected messages or context triggers a **terminal run continuation** (another graph segment in the same `processStream`). With a checkpointer only the delta is submitted; without one the full transcript is sent. Bounded by `maxStopContinuations` (default 8). |
| `StopFinalize` | Stop fields plus `continuationPlanned`, `continuationPrevented` | same as Stop | run.ts:1635, serialized after Stop is folded | A durable host makes one claim-or-seal decision. Failures are asserted (`assertFinalAdmissionSucceeded`). |
| `StopFailure` | `error`, `lastAssistantMessage?` | – | run.ts:1756 when the stream throws | Observational; the error is rethrown. |
| `PreCompact` | `messagesBeforeCount`, `trigger` | – | `src/summarization/node.ts:1332` | Observational. |
| `PostCompact` | `summary`, `messagesAfterCount` | – | `src/summarization/node.ts:923` | Observational. |

Capability flags exported from `src/hooks/index.ts`: `HOOK_INJECTED_MESSAGES_CAPABLE`, `HOOK_PREEMPT_BOUNDARY_CAPABLE`, `HOOK_STOP_CONTINUATION_CAPABLE`, `HOOK_PREEMPT_RESTART_CAPABLE`.

`InjectedMessage` (`src/types/tools.ts:655`) is `{role:'user'|'system', content, isMeta?, source?:'skill'|'hook'|'system'|'steer', skillName?}`. `convertInjectedMessages` (`src/messages/injected.ts`):
- always produces a `HumanMessage`, with `additional_kwargs {role, injected:true, isMeta?, source?, skillName?}`;
- drops empty or whitespace-only entries;
- stamps provenance `user` for steers and `synthetic` otherwise.

### 5.3 Policy factories

- **`createToolPolicyHook({mode='default'|'dontAsk'|'bypass', allow[], deny[], ask[], reason})`** (`src/hooks/createToolPolicyHook.ts`). Globs use anchored `*`. Precedence: deny, then ask, then allow; then `bypass` allows, `dontAsk` denies, and the fallthrough is `ask`. `{tool}` is substituted in the reason.
- **`createWorkspacePolicyHook({root, additionalRoots, outsideRead='ask', outsideWrite='ask', reason, pathExtractors})`** (`src/hooks/createWorkspacePolicyHook.ts`):
  - per-tool path extractors (defaults cover the local coding tools);
  - containment is checked by realpath against the resolved roots;
  - read tools use `outsideRead`; write tools and unknown tools use `outsideWrite`;
  - `ask` limits the reviewer to `allowedDecisions: ['approve','reject']`.

---

## 6. Preemption and steering

### 6.1 Host contract

`RunConfig.preemption: StreamPreemption` (`src/types/run.ts:161`) has four fields:
- `shouldPreempt()`: polled once per chunk. It must be synchronous, O(1) and **level-triggered**: keep returning true until the `PreemptBoundary` hook drains the queue.
- `subscribe?(wake)`: a wake hint for silent or reasoning-only windows. Required for a restart.
- `restartGraceMs?`
- `maxSeals?` (normalized by `resolveMaxSeals`).

Preemption also requires a dispatchable `PreemptBoundary` matcher, and a `tokenCounter` so a sealed turn still reports usage.

### 6.2 Graph gate (Graph.ts:1440-1960)

- `canClaimPreemptSeal()` requires: preemption configured, no seal in flight, remaining budget, a run id, and `hasDispatchableHookFor('PreemptBoundary')`.
- `shouldPreemptStream()` is `canClaimPreemptSeal() && preemption.shouldPreempt()`.
- `claimPreemptSeal()` and `claimPreemptRestart()` take the same slot and budget in one synchronous step. This makes them safe for parallel multi-agent lanes: only one lane wins.
- Per-agent sets `pendingPreemptReturn` and `preemptRestartPending`.
- Counters: `preemptSealCount`, `preemptRestartCount`, `preemptEmptyBoundaries`, exposed via `getPreemptStats()`. Flags: `preemptIncomplete` and `preemptHaltReason`.

### 6.3 Stream decision (`src/llm/preempt.ts`)

- **`canSealPreempt(chunk)`**, true only when the accumulated chunk is safe to cut:
  - it has non-whitespace text;
  - no open `tool_calls` or `tool_call_chunks` (Anthropic server-tool calls that already have a result block count as settled);
  - no `invalid_tool_calls`;
  - no open Gemini server `toolCall`.
- **`canRestartPreempt(chunk)`**, true when there is nothing worth keeping:
  - no chunk at all, or only reasoning and blank-text blocks, checked against a whitelist;
  - no tool machinery of any kind;
  - no OpenAI Responses provider outputs.
- **`resolvePreemptAction({chunk, requestAgeMs, graceMs})`** prefers a seal. It returns a restart only once `requestAgeMs >= graceMs`.

`src/llm/invoke.ts` (~960-1440) applies the decision:
- **Seal:** break the stream, mark `response_metadata.preempted = true`, and synthesize the model-end event and usage (`endSealedModelRun`).
- **Restart:** tear the provider stream down, cancel the open message step, emit a model-end with `preemptDiscarded`, call `notePreemptRestart(agentId)`, and return `{messages: []}`.

### 6.4 Node boundary (Graph.ts:4929-5010)

After the invoke returns, if `consumePreemptRestart(agentId)` is true or the response has `preempted`:
- call `dispatchPreemptBoundary`, which converts `additionalContexts` into a `HumanMessage{role:'system', isMeta, source:'hook'}` and `injectedMessages` through `convertInjectedMessages`;
- then `releasePreemptSeal()`.

The four outcomes:

| outcome | behavior |
|---|---|
| `preventContinuation` | Commit the sealed turn plus anything injected, with no self-loop. Sets `preemptIncomplete`; counts an empty boundary only for a seal with nothing injected. |
| Injected messages | `pendingPreemptReturn.add(agentId)`, then `routeMessage` loops back to the agent node, so the model continues from the steer. |
| Nothing injected after a restart | Loop back and reissue the same call. |
| Nothing injected after a seal | `preemptEmptyBoundaries += 1`, `preemptIncomplete = true`, route to END. `Run` reports `preempt_incomplete`. |

### 6.5 Steering channels in summary

1. **Tool boundary:** `PostToolBatch` returns `injectedMessages` with `source:'steer'`.
2. **Mid-generation:** `PreemptBoundary` (seal or restart).
3. **Terminal:** a `Stop` hook with `block` plus injected messages.

Persisted steers are `ContentTypes.STEER` content parts inside assistant messages. `formatAgentMessages` (`src/messages/format.ts` ~2025) replays each one as a `HumanMessage{source:'steer'}`, splitting the assistant message and adding an anchor when a steer comes last. At the wire, `coalesceAdjacentUserTurns` merges consecutive user turns for strict-alternation providers (Mistral, Bedrock).

---

## 7. Event actor executor (`src/eventActor/*`, `CONTEXT.md` "Event Actors")

### 7.1 Purpose

Run a stable logical child thread (an **Event Actor**) that handles a sequence of authoritative host events without keeping a live executor between them.
- Each event runs on an isolated **Invocation Fork**, a checkpoint namespace per attempt: `event-actor/<sha256(actorThreadId\0invocationId\0attemptId)[:32]>`.
- The actor's **Head** `{actorThreadId, generation, checkpoint?}` advances only through a host atomic compare-and-swap on generation plus checkpoint identity.

### 7.2 Flow (`EventActorExecutor.execute`, EventActorExecutor.ts:1847)

1. **Snapshot the event.** `snapshotEvent` makes a deep-frozen JSON copy: finite numbers only, `-0` becomes `0`, no cycles, holes or symbols, and keys are sorted.
2. **Resolve depth** from the ambient `configurable.event_actor_depth`, bounded by `maxDepth` (default 1).
3. **Prepare.** `prepare()` calls `adapter.prepare`. It returns either `ready` (a warm fork on the committed checkpoint) or `checkpoint_unavailable`, in which case `coldContinue` asks the host to rebuild the fork from transcript and summary.
   - Prepared invocations carry an HMAC `preparationDigest` (key from `preparationSigningKey`, time-bounded by `dormantCheckpointTtlMs`, default 24 h).
4. **Invoke.** `#invokeWithConfig` builds a scrubbed `RunnableConfig`:
   - it removes the parent's `__pregel_*`, `__librechat_*`, run, thread and checkpoint keys;
   - it sets `thread_id` and `checkpoint_ns` from the fork, plus the `event_actor_*` keys;
   - it runs `adapter.invoke(invocation, {signal, config})` under `runWithConfig`.
5. **Interpret the result:**
   - A thrown `GraphInterrupt` or parent `Command` propagates unchanged.
   - Any other throw is a definite no-action failure: `discard(fork)` and return `failed` or `cancelled`.
   - `completed_no_action`: discard the fork.
   - `suspended`: authenticated `EventActorSuspension` evidence (version 1, `suspensionDigest`, expiry, up to 64 KB) published via `adapter.suspend`. Later calls are `resume(...)` (a claim with `resumeAttemptId`; re-pause atomically replaces it), `cancelSuspension` and `settleSuspension`.
   - `applied`: `#issueSettlement`, then `commit` via the host CAS. The outcome is `applied` with a new head (generation + 1), `commit_conflict` (a stale head; the fork is retained for reconciliation) or `commit_indeterminate`.

The host adapter interface is `EventActorHostAdapter` (`src/eventActor/types.ts`): `prepare`, `coldContinue`, `invoke`, `commit`, `discard`, plus optional `suspend`, `resume`, `cancelSuspension`, `settleSuspension`. This is host infrastructure layered around graph runs; it does not change graph topology.

---

## 8. Re-implementing on LangGraph Python

### 8.1 Primitive mapping

| TS | Python |
|---|---|
| `Annotation.Root({...})` | `TypedDict` with `Annotated[T, reducer]` |
| `messagesStateReducer` plus the `__remove_all__` sentinel | `langgraph.graph.message.add_messages` plus `RemoveMessage(id=REMOVE_ALL_MESSAGES)` (natively `"__remove_all__"`). Add the null-chunk filtering and the pre-assigned uuid on the AI message. |
| `runStepState` revision reducer | `lambda cur, upd: upd if upd["revision"] >= cur["revision"] else cur` |
| `handoffState` merge reducer | Port `merge_handoff_state` (union by id, raise on conflict). |
| `Overwrite(...)` on fresh input | `langgraph.types.Overwrite` in recent versions; otherwise a sentinel wrapper handled in your reducers. |
| Compiled subgraph as a node | `builder.add_node("agent_id", compiled_subgraph_or_wrapper, destinations=(...))`. `ends` corresponds to `destinations` for Command-routed nodes. |
| `addConditionalEdges(agent, routeMessage)` | `add_conditional_edges("agent=X", route_message, [...])` |
| `addEdge([a,b], c)` all-of join | `add_edge(["a","b"], "c")` (waits for all) |
| `Command({goto, update, graph: Command.PARENT})` | `Command(goto=..., update=..., graph=Command.PARENT)` |
| `Send(node, args)` | `langgraph.types.Send` |
| `interrupt(payload)` / `Command({resume})` | `langgraph.types.interrupt`, `Command(resume={interrupt_id: value})` (map keyed by interrupt id is supported) |
| `MemorySaver` / durable saver | `InMemorySaver`, `AsyncPostgresSaver` / `AsyncSqliteSaver` / Redis |
| `durability: 'exit'` | `graph.astream(..., durability="exit")` |
| `getState(cfg, {subgraphs:true})`, `getStateHistory` | `aget_state(cfg, subgraphs=True)`, `aget_state_history(cfg)` |
| `streamEvents` plus custom events | `astream_events(version="v2")` and `adispatch_custom_event`, or `astream(stream_mode=["messages","custom","updates"])` with `get_stream_writer()` |
| `AsyncLocalStorageProviderSingleton.runWithConfig` | `contextvars` (the runnable config var); Python `interrupt()` reads config from the context automatically inside nodes and tools. |
| `AbortSignal` / `AbortSignal.any` | `asyncio` cancellation plus an `asyncio.Event`-based composite signal; hook timeouts via `asyncio.wait_for` or `asyncio.timeout` |
| `PromptTemplate.fromTemplate('{results}')` | `langchain_core.prompts.PromptTemplate` (identical) |
| `getBufferString` | `langchain_core.messages.get_buffer_string` |
| `GraphRecursionError` / `recursionLimit` | `config["recursion_limit"]` |

### 8.2 Suggested module layout

```
lc_agents/
  run.py                    # Run: create/process_stream/resume/get_interrupt/get_halt_reason/
                            #   get_handoff_outcome; pre-stream hooks; Stop/StopFinalize loop; halt polling
  graphs/
    base.py                 # Graph: run-step/content sidecars, step keys, dispatch_run_step*
    standard.py             # StandardGraph: create_agent_node(), create_workflow(), route_message, call_model
    multi_agent.py          # MultiAgentGraph: edge categorization, reachability, parallel groups,
                            #   handoff tools, agent_wrapper, fan-in prompt nodes
    handoff.py              # HandoffRouting, merge_handoff_state, HandoffLimitError
    factory.py              # create_graph(kind, input), apply_graph_runtime_config
  agents/
    context.py              # AgentContext (tools binding, discovery, budgets, summaries, handoff ctx)
    system_prompt.py        # stable/dynamic builders, cache-aware SystemMessage, dynamic tail
    projection.py           # project_agent_context_usage
  tools/
    tool_node.py            # batch execution, hooks, HITL interrupt, Command aggregation (Send fan-out)
    conditions.py           # tools_condition
    replay.py               # batch replay records, run-step resume wrapping, approval evidence
  hitl/
    types.py                # TypedDicts for payloads/decisions; type guards
    approval.py             # evidence, proposal matching, normalize_decisions
    ask_user.py             # ask_user_question(s)
  hooks/
    types.py registry.py execute.py matchers.py
    policies/tool_policy.py policies/workspace_policy.py
  llm/
    invoke.py               # streaming attempt, fallbacks, overflow recovery
    preempt.py              # can_seal/can_restart/resolve_action, restart registry
  messages/
    reducer.py injected.py handoff_cue.py provenance.py alternation.py
  event_actor/
    types.py executor.py signing.py
```

### 8.3 Pitfalls when porting

1. **Node re-execution on resume.** Python `interrupt()` also restarts the node from its beginning. Port the replay records: settled siblings attached to the interrupt value, and one-shot hook `pendingToolApprovals`. Otherwise side-effecting siblings run twice. Keep `interrupting_tool_names` ordering. Python's prebuilt `ToolNode` has none of this, so write a custom ToolNode.
2. **Wrapping and stripping interrupt values.** Store private replay and run-step state *inside* the interrupt value (a wrapper dict with sentinel keys). Strip it for `get_interrupt()`, and restore it into private `configurable` keys on resume. Never trust host-supplied values under those keys: delete them on fresh input (run.ts:1280).
3. **Handoff Commands from nested subgraphs.** Handoff tools run inside `tools=X` in the agent subgraph and return `Command(graph=Command.PARENT, …)`. Python supports this, but:
   - the outer node must declare `destinations`;
   - several parent Commands from one batch must be converted to `Send` fan-out, as ToolNode does;
   - `handoff_request` must be removed from the update after `HandoffRouting.finalize` records it.
4. **Message merging.** The recipient's filtered view is local to its invocation, and the subgraph returns the full list. Rely on id-based upsert. Make sure every message has an id *before* it leaves the node; the SDK assigns a v4 id itself (Graph.ts:4826).
5. **`start_index` semantics.** It is computed in the outer reducer on the first write, as the size of the host input. `{results}`, `exclude_results` slicing, pruning and "run-produced" detection all depend on it. The Python reducers are pure, so capture it in the `Run` or the graph object before streaming instead of inside the reducer.
6. **Mutable per-run graph object.** JS mutates `self.config`, `AgentContext` and many maps from inside nodes, and it relies on single-threaded sync sections for `claim_preempt_slot` and `once` matcher removal. In Python, keep everything on one asyncio loop and never await inside those critical sections. Avoid thread executors for nodes.
7. **Hook halts and stream cancellation.** `preventContinuation` raises a registry halt that the stream loop polls. `PreemptBoundary` must clear its own halt, or breaking the stream destroys the sealed turn before the reducer commits it.
8. **Terminal continuation.** With a checkpointer, submit only the injected delta and advance the stream segment and step keys. Without one, reseed the full transcript. Never run RunStart or UserPromptSubmit for continuations or resumes.
9. **Recursion headroom.** Add `max_seals` to `recursion_limit`. Apply `member_recursion_limit` per member invocation.
10. **Mid-conversation system messages.** Anthropic and Google reject them. All injected context and routing prompts are `HumanMessage` objects with `additional_kwargs.role='system'`/`source`, and adjacent user turns are coalesced for strict providers.
11. **Prompt caching.** Reproduce the stable/dynamic split and the dynamic tail placed before the last human message, or cache hit rates collapse. Keep tool-cache breakpoints on the last static tool so discovered deferred tools do not invalidate the prefix.
12. **Parallel preemption lanes.** The pending-return and restart sets are keyed by agent id; only one lane can hold the seal slot.
13. **Approval fail-closed rules.** Validate decision shapes strictly. Enforce `allowed_decisions`. Compare the current proposal (stable-JSON args) with the reviewed one. Treat missing decisions as rejections.
14. **Event actor.** Use `hmac` and `hashlib`, and constant-time comparison (`hmac.compare_digest`). Freeze events into canonical JSON. Scrub `__pregel_*` keys from the inherited config. Let `GraphInterrupt` and `ParentCommand` propagate (in Python, `GraphInterrupt` and `ParentCommand` live in `langgraph.errors`).
