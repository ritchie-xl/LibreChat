# @librechat/agents: Tool Execution & Built-in Tools

> Reference chapter for the [@librechat/agents SDK architecture](../README.md). Produced by static reading of
> [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) at `v3.9.3` (`2d5653d`). Paths are relative to
> that repository's root. Line numbers are approximate; check the code when a detail matters.

Scope: `ToolNode`, the host tool-execution contract, the tool-call lifecycle, eager execution, every built-in tool, the Code API, tool search, programmatic tool calling (PTC), web search, and subagents. All paths are relative to the SDK repo root.

---

## 1. ToolNode architecture and the host contract

### 1.1 What ToolNode is
`src/tools/ToolNode.ts` (6.3k lines) contains `class ToolNode<T> extends RunnableCallable<T, T>`. It is a custom replacement for LangGraph's prebuilt ToolNode.

**Accepted inputs:**
- `BaseMessage[]`
- `{ messages: BaseMessage[] }` (messages state)
- A LangGraph `Send` payload `{ lg_tool_call, ...state }`

**Returned outputs:**
- The same shape it received, containing `ToolMessage`s plus any injected `HumanMessage`s, or
- `Command` objects (for handoffs).

**Constructor inputs** are `t.ToolNodeConstructorParams`, defined in `src/types/tools.ts` as `ToolRefs & ToolNodeOptions`. The fields that matter most:
- `tools` / `toolMap`
- `eventDrivenMode`, `toolDefinitions: Map<string, LCTool>`, `directToolNames`
- `toolRegistry: Map<string, LCTool>` (used for PTC and tool search)
- `sessions: ToolSessionMap` and `codeSessionKey`, the shared code-session store (default key `execute_code`)
- `toolCallStepIds: Map<toolCallId, stepId>`, filled by the stream handler; used to route completion events
- `errorHandler(data, metadata) => Promise<boolean|void>`
- `handleToolErrors` (default true)
- `hookRegistry`, `humanInTheLoop`
- `maxToolResultChars` / `maxContextTokens`
- `toolOutputRegistry` / `toolOutputReferences`
- `toolExecution` (engine `sandbox | local | cloudflare-sandbox`)
- `eagerEventToolExecution*`, which are shared maps owned by the Graph
- `interruptingToolNames`
- `getBreakerSignal` / `getRunScope`
- `preparedSubagents`
- `executionContext` (subagent lineage)

**How the Graph builds it** (`src/graphs/Graph.ts` ~2837-2930):
- Event-driven mode is on when `agentContext.toolDefinitions` is non-empty.
- The Graph builds schema-only stubs with `createSchemaOnlyTools` (`src/tools/schema.ts`). Invoking a stub throws "should not be invoked directly in event-driven mode". The stub's `responseFormat` defaults to `content_and_artifact`.
- Every in-process "graph tool" is added to the tool map and auto-marked **direct**: handoff tools `lc_transfer_to_*`, the `subagent` tool, and similar.
- `applyToolExecutionOverrides()` (`src/tools/local/resolveLocalExecutionTools.ts`) swaps Code-API tools for local or Cloudflare implementations when `toolExecution.engine` is `local` or `cloudflare-sandbox`. Those names are also added to `directToolNames`, so they run in-process even in event-driven mode.

### 1.2 Two execution paths
1. **Direct path.** `runDirectBatchInterruptSafe` → `runDirectToolWithLifecycleHooks` → `runTool` → `invokeWithRuntime` → `tool.invoke(invokeParams, runtime)`.
   - `invokeParams` is `{...call, args, type:'tool_call', stepId, turn: usageCount}` plus special injections.
   - LangChain moves the non-schema fields into `config.toolCall`, which is how tools read injected context.
   - The runtime mirrors LangGraph 1.4's `ToolRuntime`: `{...config, state, toolCallId, config, context, store, writer}`.
2. **Event path.** `dispatchToolEvents` builds one `ToolExecuteBatchRequest` and emits it as a LangChain custom event `on_tool_execute` (`GraphEvents.ON_TOOL_EXECUTE`) through `safeDispatchCustomEvent` (`src/utils/events.ts`). The host's handler (registered on the Run's `HandlerRegistry`; `ON_TOOL_EXECUTE` is in `CUSTOM_GRAPH_EVENTS` in `src/run.ts`) runs the tools and calls `resolve(results)`.

**Mixed batches** (see `run()` at ~5332):
- Calls are split into direct entries and event entries. Direct means the name is in `directToolNames`, or it is an unknown `lc_transfer_to_*` handoff while handoff tools are registered.
- Event args are placeholder-resolved **synchronously, before any await**, against a frozen snapshot of the tool-output registry.
- The directs run first. This gives fail-fast behaviour: a thrown error or `GraphInterrupt` aborts before any host dispatch. The events are dispatched afterwards.
- Output order is fixed:
  `[promotedAiMessage?] + directOutputs + eventToolMessages + invalidCallResults + directInjected(HumanMessage) + eventInjected`

### 1.3 `ON_TOOL_EXECUTE` request: `ToolExecuteBatchRequest` (`src/types/tools.ts`)

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

`ToolCallRequest`:
```ts
{ id: string; name: string; args: Record<string,unknown>;   // schema-coerced
  stepId?: string; turn?: number;                            // per-tool usage index
  codeSessionContext?: { session_id: string; files?: CodeEnvFile[] }; // code tools + skill + read_file
  codeSessionBaselineId?: string; retainCodeSessionInputs?: true;
  runtimeSessionHint?: string }                              // only when sandbox.statefulSessions
```

`CodeEnvFile` is `{ id, resource_id, name, storage_session_id, kind: 'skill'|'agent'|'user', version? }`. `version` is required when `kind` is `skill`.

**Promise semantics.** The promise resolves only after **both** of these have happened:
- the dispatch settled (the custom event returned), and
- `resolve` was called.

If dispatch throws, the promise rejects. `safeDispatchCustomEvent` wraps `resolve` so it can record Langfuse host-result traces first.

### 1.4 Response: `ToolExecuteResult`
```ts
{ toolCallId: string; content: string | unknown[]; artifact?: unknown;
  status: 'success' | 'error'; errorMessage?: string;
  outcome?: string; outcome_patch?: { from: string; to: string };   // settled UI label
  injectedMessages?: { role:'user'|'system'; content: string|MessageContentComplex[];
                       isMeta?: boolean; source?: 'skill'|'hook'|'system'|'steer'; skillName?: string }[] }
```

**How each result becomes a message:**
- `injectedMessages` are converted to `HumanMessage`s by `convertInjectedMessages`; the original role is kept in `additional_kwargs.role`. They are appended **after** all `ToolMessage`s so each tool_call stays adjacent to its tool_result.
- If the host returns code-session artifacts (`artifact.session_id`, `files`, `deleted_files`) for code tools or `skill`, they are merged into `sessions` (see §4.4).
- **Error:** `ToolMessage{status:'error', content: "Error: <errorMessage>\n Please fix your mistakes."}`, truncated.
- **Success:** content is serialized within `maxToolResultChars`; `artifact` is passed through unchanged.

### 1.5 Completion events
For each result, ToolNode emits `ON_RUN_STEP_COMPLETED`:

```ts
{ result: { id: stepId, index: turn, type:'tool_call',
            tool_call: { args: <bounded serialized args>, name, id, output: <content string>,
                         progress: 1, outcome? },
            completed_at: Date.now(), eager?: true } }
```

**Deduplication.** The event is skipped if it was already emitted by:
- the early `onResult` fast path,
- the stream's eager dispatcher (`record.completionDispatched`), or
- the `errorHandler` (tracked in `ToolErrorOwnership`).

`handleRunToolCompletions` does the same job for direct outputs and additionally updates code sessions from `ToolMessage.artifact`.

---

## 2. Tool-call lifecycle

### 2.1 Batch entry (`run()`)
1. Compose the signal: `config.signal` plus the graph breaker signal. If it is already aborted with `StreamLimitExceededError` or `PreparedSubagentError`, throw immediately.
2. Mint `batchScopeId`: the run_id, or `\0anon-N` when there is none. Claim `turn` from the tool-output registry (`nextTurn`).
3. Find the last AIMessage (`findAssistantBatch`). Filter out:
   - calls that already have a ToolMessage (resume/replay), and
   - Anthropic server tool calls (id prefix `srvtoolu_`).
4. **Invalid tool calls.** `aiMessage.invalid_tool_calls` holds calls whose streamed args never parsed.
   - Each gets a synthesized error ToolMessage: "Error: Malformed args. The tool call input could not be parsed as a JSON object; the tool was not run."
   - A **replacement AIMessage with the same id** is emitted, with those calls promoted into `tool_calls` with `args:{}`. The reducer upserts it by id.
   - The broken tool_use blocks in its content are sanitized (`sanitizeInvalidToolUseBlocks`).
   - This happens only for messages-state input with an id-bearing AI message, so call and result stay paired for every provider converter.
   - The promotion is also patched into any handoff `Command.update` (`patchCommandUpdateForPromotedInvalidCalls`).
5. `loadRuntimeTools(tool_calls)` can rebuild the tool map on each batch.

### 2.2 Argument validation and coercion
There is **no full schema validation in the SDK** on the event path. The host validates.

`buildToolExecutionRequestPlan` (`src/tools/eagerEventExecution.ts`) does three things:
- `coerceRecordArgs`: string args are JSON-parsed. Anything that is not an object becomes a rejected result, "Invalid tool call arguments: expected a JSON object." (`invalidArgsBehavior:'error-result'`).
- `coerceArgsForSchema`: lossless, schema-directed repairs only.
  - Integer strings become ints; number strings become numbers only if they round-trip exactly.
  - `"true"`/`"false"` become booleans.
  - The coercion recurses into arrays, properties, and `additionalProperties`.
- Per-tool `turn` reservation: `usageCount[name]++`. The same count is mirrored into the eager usage counter.

On the direct path, LangChain's `tool()` Zod/JSON-schema validation runs inside `tool.invoke`. Validation errors surface through the catch path.

### 2.3 Hooks and HITL (both paths)
- **PreToolUse** hooks run per call in parallel.
  - `deny` produces a blocked `ToolMessage{status:'error', content:"Blocked: <reason>"}` and fires the `PermissionDenied` hook.
  - `updatedInput` rewrites args, and placeholders are re-resolved against the pre-batch snapshot.
  - `ask`: when `humanInTheLoop.enabled` is false, it collapses to deny. When true, a single LangGraph `interrupt()` is raised with this payload:

    ```ts
    { type:'tool_approval', hook_session_id?, action_requests:[{tool_call_id,name,arguments,description?}],
      review_configs:[{action_name, tool_call_id, allowed_decisions: ['approve','reject','edit','respond']}] }
    ```

    Resume accepts either an array (in order) or a map keyed by tool_call_id.
    - `approve`: execute.
    - `reject`: block.
    - `edit`: requires an object in `updatedInput`.
    - `respond`: requires a string `responseText`; it becomes a synthetic success result and the tool is never run.
    - Unknown or missing decisions, or decisions outside `allowed_decisions`, fail closed.
  - Blocked-call side effects (completion event and `PermissionDenied`) are deferred until after `interrupt()`, so they fire exactly once.
- **PostToolUse** may replace output (`updatedOutput`). **PostToolUseFailure** is observational. **PostToolBatch** receives entries in model order.
- All `additionalContext` strings are gathered into **one** `HumanMessage{additional_kwargs:{role:'system', source:'hook'}}`, appended after the tool messages. Hook `injectedMessages` go after that.
- Resume safety: `toolBatchReplay.ts` and `SubagentReplay.ts` persist settled direct results per batch key: `sha256(stableStringify(tool_calls))` + thread + AI message id. Replays reuse them instead of re-executing. A changed proposal fails closed.

### 2.4 Parallelism
- **Event path:** the whole approved set goes to the host in one request; the host decides concurrency. Eager results already in flight are awaited with `Promise.allSettled([eager, dispatch])`.
- **Direct path:** `Promise.allSettled` over all calls, then the breaker is rechecked.
  - If any call is in `interruptingToolNames` (e.g. `ask_user_question`), those run first as their own awaited group. The non-interrupting (possibly non-idempotent) siblings start only after no interrupt was raised.
  - A `GraphInterrupt` from the group is rethrown with a subagent resume manifest attached.

### 2.5 Errors
- Unknown tool: `Tool "x" not found.` For handoffs it adds `Did you mean "lc_transfer_to_y"? Handoff tool names must match exactly.`
- Direct-path exceptions (with `handleToolErrors`, which defaults to true):
  - `GraphInterrupt`, `StreamLimitExceededError`, and `PreparedSubagentError` are rethrown.
  - Otherwise `errorHandler` is called. It returns `false` if it could not dispatch the completion.
  - The error becomes `ToolMessage{status:'error', content:"Error: <msg>\n Please fix your mistakes."}`.
- Placeholder references that did not resolve are stamped into `additional_kwargs._unresolvedRefs`.

### 2.6 Truncation (`src/utils/truncation.ts`, `src/utils/toolContent.ts`)
**Limits:**
- `maxToolResultChars` = the explicit value, otherwise `min(floor(maxContextTokens*0.3)*4, 400_000)`. The default is `HARD_MAX_TOOL_RESULT_CHARS = 400_000`.

**`truncateToolResultContent` (head/tail):**
- Head gets 70% of the budget, tail 30%.
- The cut marker is `\n\n… [truncated: N chars exceeded M limit] …\n\n`.
- It snaps to a newline if one is within 200 chars.
- If fewer than 200 chars are available it keeps the head only.
- It never splits UTF-16 surrogate pairs.

**Other functions:**
- `serializeStructuredValueBounded`: bounded JSON serialization for non-string outputs. It never calls `toJSON` or getters, enforces depth 200 and work caps of 1M, and handles cycles and bigint. It also returns an exact prefix for the output-reference registry.
- `compactToolContent`: used when a tool returns its own ToolMessage. Pure text or structured arrays become one bounded string. In mixed-media arrays, atomic blocks (image/file/audio...) stay intact and text is compacted.
- Computer-use screenshot outputs pass through untouched.
- Tool-call args in completion events are bounded with `serializeToolContentBounded`.

### 2.7 Artifacts and `content_and_artifact`
- Tools declared with `responseFormat:'content_and_artifact'` return `[content, artifact]`. LangChain builds a `ToolMessage` with `.artifact`, and ToolNode passes it through (after compaction).
- Artifacts carry:
  - code sessions: `{session_id, files, deleted_files, artifact_delivery, runtime_session_id, runtime_status}`
  - web search: `{web_search: data, outcome?}`
  - tool search: `{tool_references, metadata}`
  - settled labels: `outcome` / `outcome_patch`, read by `readOutcomeFields`
- Output references (optional, `toolOutputReferences.enabled`):
  - Each successful output is stored raw under `tool<idx>turn<turn>`: at most 400 KB per output and 5 MB total, FIFO eviction.
  - `{{toolXturnY}}` inside string args is substituted before invocation.
  - The LLM sees a `[ref: key]` prefix or a `_ref` JSON key through a lazy annotation.
  - The code-session "Generated files:" summary is stripped before storage.

### 2.8 Intent labels (`src/tools/intentArg.ts`)
Most built-in schemas put an optional `intent` string first (`INTENT_PROPERTY`; its description starts with `ALWAYS write this field FIRST`). Because it is first, it streams first and becomes the live UI label.

Tools strip it before use. `resolveToolOutcome` computes the settled label from `outcome` or `outcome_patch`, bounded to 256 chars on a single line.

### 2.9 Eager (speculative) execution
Note: `src/llm/providers.eager.ts` is unrelated. It is just source-mode lazy provider module registration. The eager-execution logic is in `src/stream.ts` plus ToolNode.

**When it is allowed** (`isEagerToolExecutionEnabledForBatch`), all of the following must hold:
- `eagerEventToolExecution.enabled`
- the agent has `toolDefinitions` (event mode)
- no result-altering hooks
- HITL is off
- the call is not inside PTC metadata
- an `ON_TOOL_EXECUTE` handler exists

**Per-call exclusions:**
- `excludeToolNames`
- the run-scoped suppression set
- host `codeSessionToolNames`
- code tools when `statefulSessions` is on, or when a non-default `codeSessionKey` is used

**Batch-level fallbacks:** eager is skipped if any direct/graph tool is in the batch, or if any `{{tool…}}` reference appears.

**When a call starts:**
- when the final chunk carries a finish-reason signal,
- when a call is sealed on arrival (Google), or
- when a streamed call is **sealed** mid-stream, which happens when:
  - a later index begins (Anthropic, OpenAI chat-sequential adapters), or
  - an adapter emits a seal in `response_metadata.lc_streamed_tool_call_seal` (OpenAI Responses `.done`, Bedrock `contentBlockStop`).

A sealed call starts only if its accumulated args text is provably canonical: its length equals the raw concatenation length, or it equals the adapter's authoritative restatement. `docs/eager-tool-readiness-benchmark.md` benchmarks this "seal-first" readiness check: 28–51% lower handler time on large args.

**Mechanics:**
- Eager dispatch sends a separate `ToolExecuteBatchRequest` of the same shape and stores records in `graph.eagerEventToolExecutions[callId] = {toolName, args, request, promise}`.
- On completion the stream emits `ON_RUN_STEP_COMPLETED` with `eager:true`.
- ToolNode later consumes each record (`takeMatchingEagerEventExecution`). It compares name plus `stableStringify(args)`.
- **On mismatch:** it returns the error "Tool call changed after eager execution started; refusing to re-run the tool to avoid duplicate side effects." It also adds both names to the suppression set so they are never prestarted again in the run. This circuit breaker prevents retry loops (LibreChat#14371).
- PreToolUse deny removes the eager record so no success completion leaks.
- Foreground subagents have their own prestart: `prestartSubagent` + `PreparedSubagents`, with `maxPendingSubagents` defaulting to 4.

---

## 3. Built-in tool catalog

The `intent` field is present on almost all built-in schemas unless noted.

| Name (`Constants`) | File | Input schema | Behavior / external calls |
|---|---|---|---|
| `calculator` | `src/tools/Calculator.ts` | `{input: string}` (legacy `Tool`, no intent) | `mathjs.evaluate(input).toString()`, loaded lazily. On error returns "I don't know how to do that." |
| `execute_code` | `src/tools/CodeExecutor.ts` | `{lang: enum[py,js,ts,c,cpp,java,php,rs,go,d,f90,r,bash], code: string, args?: string[]}`; required lang, code | POST `{base}/exec`; returns content_and_artifact (§4) |
| `bash_tool` | `src/tools/BashExecutor.ts` | `{command: string, args?: string[]}` | POST `/exec` with `lang:'bash'`. With an attached workspace it POSTs `/exec/programmatic` with `tools:[]` and the header `X-LibreChat-Code-Workspace-ID`. Optional `BashToolOutputReferencesGuide` appended to the description. |
| `run_tools_with_code` (PTC) | `src/tools/ProgrammaticToolCalling.ts` | `{code: string(minLength 1), tool_manifest?: string[] unique, timeout?: integer ms [1000..max(cap,300000)], default cap}` | Python in the Code API sandbox, round-trip loop (§5.2) |
| `run_tools_with_bash` | `src/tools/BashProgrammaticToolCalling.ts` | same shape; code is bash | Same loop with `lang:'bash'`. Tools become bash functions that take a JSON string. Code is prefixed with `: &\nwait "$!"`. Results are JSON-normalized. |
| `tool_search` | `src/tools/ToolSearch.ts` | `{query?: string(≤200, default ''), fields?: ('name'|'description'|'parameters')[] default [name,description], max_results?: 1..50 default 5, mcp_server?: string|string[]}` | Local BM25, or a JS regex script run on the Code API `/exec` (§5.1) |
| `web_search` | `src/tools/search/*` | `{query, date?: 'h'|'d'|'w'|'m'|'y', country?: string (serper/tavily only), images?, videos?, news?: boolean}` | §6 |
| `read_file` (remote def) | `src/tools/ReadFile.ts` | `{path}` | Definition only (`responseFormat: content_and_artifact`), executed by the **host**. Skill files use `{skillName}/{path}`; code outputs use their returned paths. |
| `skill` | `src/tools/SkillTool.ts` | `{skillName: string, args?: string}` | Definition only, host-executed. The host returns `injectedMessages` (source `skill`). `skillCatalog.ts` formats the "Available Skills" prompt. |
| `subagent` | `src/tools/SubagentTool.ts` + `Graph.ts` | `{description, subagent_type: enum(types), run_in_background?: boolean, subagent_thread_id?: string}` | §7 |
| `lc_transfer_to_<agent>` | graphs | handoff | Returns `Command(graph: PARENT, goto)`. Several handoffs in one batch become parallel `Send`s tagged with `handoff_parallel_siblings`, `__handoff_parallel_batch`, `__handoff_group_id`. |
| Local coding suite: `read_file`, `write_file`, `edit_file`, `grep_search`, `glob_search`, `list_directory`, `compile_check`, plus local `bash_tool`, `execute_code`, and both PTC runners | `src/tools/local/*` | read `{path, offset?, limit?}`; write `{path, content}`; edit `{path, old_text?, new_text?, edits?: [{old_text,new_text}]}` (each old_text must be unique); grep `{pattern, path?, glob?, max_results?}`; glob `{pattern, path?, max_results?}`; list `{path?}` | Runs as child processes or fs in `workspace.root`. See §4.5. |
| Cloudflare suite (same names) | `src/tools/cloudflare/*` | same | `sandbox.exec` / `readFile` / `writeFile` through a structural runtime (§4.6) |

All code tools share `CODE_EXECUTION_TOOLS = {execute_code, bash_tool, run_tools_with_code, run_tools_with_bash}` (`src/common/enum.ts`). These tools take part in the shared code session.

---

## 4. Code execution: Code API contract

### 4.1 Configuration
- **Base URL:** the `baseUrl` param, otherwise env `LIBRECHAT_CODE_BASEURL`, otherwise `https://api.librechat.ai/v1`. Endpoints are built with `buildCodeApiEndpoint(base, route)`.
- **Timeout:** env `CODE_API_RUN_TIMEOUT_MS`, the PTC per-run cap. Default 15000, minimum 1000.
- **Proxy:** env `PROXY`, honouring `NO_PROXY`; an `https-proxy-agent` or SOCKS agent is picked by scheme.
- **Other env:** `PTC_DEBUG=true`.
- **Auth:** the SDK has **no built-in API-key env**. Auth comes from `authHeaders`: a static map, or a sync/async function resolved per request. Header resolution failure produces "Code execution is not authorized…".

**Common headers:**
- `Content-Type: application/json`
- `User-Agent: LibreChat/1.0`
- `X-CodeAPI-Expected-Profile: default|stateful` (when `executionProfile` is set)
- `X-LibreChat-Code-Request-ID: <uuid>` (PTC, when a signal exists)
- `X-LibreChat-Code-Workspace-ID` (attached workspace)

### 4.2 Endpoints
1. **`POST {base}/exec`**, request body:
   ```json
   { "lang": "py", "code": "...", "args": ["..."]?, "files": [CodeEnvFile]?, "session_id"?: "...",
     "runtime_session_hint"?: "...", "timeout"?: 15000, "user_id"?: "...", "workspace_instance_id"?: "<64 hex>" }
   ```
   - Factory params (`session_id`, `user_id`, `files`) are spread in.
   - `intent` and any model-supplied `runtime_session_hint` or `workspace_instance_id` are **stripped**.
   - `runtime_session_hint` is sent only when `statefulSessions === true` and `executionProfile !== 'default'`.
   - The hint comes from the factory; otherwise from `config.toolCall._runtime_session_hint`, which ToolNode injects from `toolExecution.sandbox.runtimeSessionHint` or else `thread_id`.

   Response (`ExecuteResult`):
   ```json
   { "session_id": "exec-session", "stdout": "", "stderr": "", "files": [{"id","name","path"?,"storage_session_id"?,"resource_id"?,"kind"?,"version"?,"inherited"?:true}],
     "deleted_files": ["path"], "artifact_delivery": {"code":"artifact_delivery_failed","status":"partial|failed","attempted","delivered","failed"},
     "runtime_session_id"?: "...", "runtime_status"?: "new|reused" }
   ```
2. **`POST {base}/exec/programmatic`**, the PTC start request:
   ```json
   { "code": "...", "lang"?: "bash", "tools": [LCTool{name,description,parameters}], "session_id"?, "timeout": ms,
     "files"?: [...], "runtime_session_hint"?, "workspace_instance_id"? }
   ```
   Continuation request: `{ "continuation_token": "...", "tool_results": [{call_id, result, is_error, error_message?}] }`.
   Response (`ProgrammaticExecutionResponse`):
   `{status:'tool_call_required', continuation_token, tool_calls:[{id, name, input}]}`, or `{status:'completed', stdout, stderr, files, deleted_files, artifact_delivery, session_id, runtime_*}`, or `{status:'error', error, stderr}`.
3. **`POST {endpoint}/cancel`** with `{request_id}`: best effort, 2 s timeout, sent when the signal aborts.
4. **`GET {base}/files/{session_id}?detail=full[&kind=&id=&version=]`**: `fetchSessionFiles`. Returns `[{id, name, metadata:{'original-filename'}, resource_id, storage_session_id}]`. The legacy executor fallback to it was removed; executors now rely on injected files.

**Error mapping** (`buildCodeApiHttpErrorMessage`):
- The error body is read with bounds: 64 KB, 1 s.
- Capability codes (`bridge_worker_mismatch`, `execution_profile_mismatch`, ...) produce a permanent "not supported… Do not retry" message.
- 429 produces a rate-limited message, using `retry_after_seconds` (capped at 3600).
- 401/403: not authorized. 400/422: rejected request. Everything else: "temporarily unavailable".
- Tool errors are thrown as `CodeApiRequestError("Execution error:\n\n…")`. ToolNode converts them to error ToolMessages.

### 4.3 Output formatting
- Content: `stdout:\n…\n` (or "stdout: Empty. Ensure you're writing output explicitly.") plus `stderr:\n…`.
- Reminders are appended:
  - `/tmp` is scratch only, when the code references `/tmp`.
  - On failure, files written to `/mnt/data` were not registered.
  - An artifact delivery warning, when present.
- A "Generated files:\nSession files: N persisted file(s)… including K image(s)…" summary is appended for non-inherited files (`src/tools/CodeSessionFileSummary.ts`).
- Artifact: `{session_id, files?, artifact_delivery?, deleted_files?, runtime_session_id?, runtime_status?}`.

### 4.4 Session model
There are two separate ids:
- The **exec `session_id`**: transient, one per sandbox run.
- The **per-file `storage_session_id`**: the object-store bucket where the file lives.

Mixing them up causes 404s on later calls.

**Where session state lives.** `Graph.sessions` is a `Map`. Under `codeSessionKey` it holds `{session_id, files: FileRef[], lastUpdated}`.

**Before a call:**
- Direct path: ToolNode injects `session_id` and `_injected_files` (a `CodeEnvFile[]` built by `toInjectedFileRef`: kind defaults to `user`, `resource_id` defaults to `id`, `storage_session_id` defaults to the exec id) into `invokeParams`.
- Event path: the same data goes into `codeSessionContext`.

**After a call** (`updateCodeSession`):
- Files are merged by identity `(storage_session_id, id)`.
- A new file replaces an existing one with the same name.
- A file is removed **only** if it is listed in `deleted_files` **and** its identity matches the baseline injected into that request. A file missing from `files` is not treated as a deletion.
- `inherited` echoes never resurrect a deleted file.
- `retainCodeSessionInputs` keeps host-refreshed inputs even when execution returned no artifact.

### 4.5 Local engine (`src/tools/local/*`)
- **Execution world:** `ExecutionWorld{spawn, fs, sandboxed}`. Defaults to Node `child_process` plus `fs/promises`, overridable through `local.exec`.
- **Limits and paths:** default timeout 60 s; max output 200k chars; hard kill at 50 MiB streamed. Paths are clamped to `workspace.root` and `additionalRoots` after realpath.
- **`execute_code` runtimes:** writes the code to a temp file, then runs `python3`, `node`, `npx --no-install tsx`, `php`, `go run`, `rustc`, `gcc`/`g++`, `javac && java`, `Rscript`, `ldc`, or `gfortran`.
- **Bash validation** (`validateBashCommand`):
  - regex blocks destructive patterns and nested-shell tricks,
  - `bash -n` syntax check,
  - optional tree-sitter AST check (`bashAst: off|auto|strict`),
  - `readOnly` mode blocks mutations,
  - optional `@anthropic-ai/sandbox-runtime` wrapping.
- **Output** adds `exit_code`, `timed_out`, `killed`, `full_output_path`, and `working_directory` lines. The artifact is `{session_id:'local', files:[]}`.
- **`edit_file` / `write_file`:** optional `FileCheckpointer` (enables `Run.rewindFiles()`) and a post-edit syntax check (`off|auto|strict`: `node --check`, `py_compile`, `JSON.parse`, `bash -n`).
- **`grep_search`:** ripgrep, with a regex fallback that has ReDoS guards and a 5 s budget.
- **`read_file`:** optional image/PDF attachments (`attachReadAttachments`).

**Local PTC** (`LocalProgrammaticToolCalling.ts`):
- Starts an HTTP bridge on `127.0.0.1:<random>/tool` with a 32-byte token in the header `x-librechat-bridge-token` (compared in constant time).
- Generates a Python program with one `async def <normalized_name>(**kwargs)` stub per tool. Each stub POSTs `{name, input}` and raises on `is_error`. Bash stubs use curl with `?mode=text`.
- Each bridge request runs PreToolUse hooks (deny/updatedInput) and then `executeTools`.

### 4.6 Cloudflare (`src/tools/cloudflare/*`, `docs/cloudflare-sandbox-tools.md`)
- **Runtime interface:** `exec`, `readFile`, `writeFile`, `mkdir`, `listFiles`, `deleteFile`.
- **Settings:** `workspaceRoot` defaults to `/workspace`. `codingToolNames` can restrict the suite, e.g. `CLOUDFLARE_BASH_CODING_TOOL_NAMES` has no `execute_code` and no `run_tools_with_code`.
- **HTTP bridge adapter** (`createCloudflareBridgeRuntime({baseURL, apiKey, sandboxId})`):
  - `Authorization: Bearer`
  - `POST /v1/sandbox` → `{id}`
  - `POST /v1/sandbox/:id/exec` with `{argv:[shell,'-lc',cmd], cwd, timeout_ms}` (streamed response)
  - `GET/PUT /v1/sandbox/:id/file/<path>`
  - `mkdir`, `list`, and `delete` are implemented with bash.
- **PTC** generates Python or JS "native" implementations of the coding tools that run **inside** the sandbox. Only the built-in coding tools can be called; there is no host bridge.

---

## 5. Tool search and programmatic tool calling

### 5.1 Deferred tools and `tool_search`
- **Tool metadata** (`LCTool`): `defer_loading: true` hides a tool from the initial binding. `allowed_callers: ('direct'|'code_execution')[]` defaults to `['direct']`.
- **Active binding:** `AgentContext.getToolsForBinding()` binds tools that allow the `direct` caller and are active, where active means `!defer_loading || discoveredToolNames.has(name)`. In event mode these are schema-only tools built from `getActiveToolDefinitions()`, plus graph tools.
- **Registry injection:** ToolNode injects `toolRegistry` into `invokeParams` for `tool_search`.

**Search modes** (`createToolSearch({mode})`):
- `local`: BM25 (`okapibm25`, k1=1.5, b=0.75).
  - Documents are the name twice, the description, and optionally parameter names.
  - Tokenization splits on non-alphanumerics and camelCase and merges single-letter acronym fragments.
  - Identifier priority overrides scores: exact full name (4), exact base name without `_mcp_server` (3), exact joined-token identifier (2), prefix (1, score ≥0.95).
  - An empty query returns tools sorted alphabetically.
- `code_interpreter` (default): a regex search.
  - The pattern is sanitized: nested quantifiers, group depth over 5, or other dangerous constructs, or an invalid regex, cause it to be escaped as a literal (with a note).
  - A generated JS script runs through `POST /exec {lang:'js', code, timeout:5000}` **with no auth headers**.
  - Scores: name 0.95, description 0.75, parameters 0.60.

**Filters and naming:**
- `onlyDeferred` defaults to true.
- `mcp_server` resolution: exact match, then canonical (lowercase, NFC, `[-_. ]+` becomes `-`), then with a `mcp-` prefix stripped. Ambiguous matches return all candidates.
- MCP tool names use the form `tool_mcp_server` (`Constants.MCP_DELIMITER='_mcp_'`).
- The description embeds a listing of deferred tools grouped by server (`name(…)` marks tools with parameters).

**Output:**
- Content (JSON): `{found, tools:[{name, score, matched_in, snippet}], total_searched, query, notes?}`.
- Artifact: `{tool_references:[{tool_name, match_score, matched_field, snippet}], metadata:{total_searched, pattern, error?, available/unmatched/idle_mcp_servers?}}`.
- Server listing mode (a server filter with an empty query) is a preview only and discovers nothing.

**How discovery becomes binding:**
1. Before each model call, `Graph` runs `extractToolDiscoveries(messages)` (`src/messages/tools.ts`). This scans the ToolMessages named `tool_search` in the current turn and collects `artifact.tool_references[].tool_name`.
2. `agentContext.markToolsAsDiscovered` adds the names, marks the system runnable stale, and recomputes tool-token accounting.
3. The next model call binds those tools.
4. Hosts can pre-seed `discoveredTools` from history.
5. The event-mode snapshot `callerCapabilityProjection` sends the live discovery state to the host.

### 5.2 PTC design
ToolNode injects the following into `invokeParams` (and therefore `config.toolCall`) for `run_tools_with_code` and `run_tools_with_bash`:
- `toolMap` and `toolDefs`: the active tools whose `allowed_callers` include `code_execution`. In event-driven mode only direct or local tools are included, because host tools cannot run in-process.
- `disallowedToolDefs`: the direct-only tools.
- `programmaticToolName`
- `hookContext`

**Flow:**
1. `selectProgrammaticTools` applies these rules:
   - If direct-only tools exist, `tool_manifest` is required.
   - Requesting a disallowed tool throws: "cannot be called … not marked for code_execution".
   - Unknown names throw.
2. `assertUnambiguousIdentifiers`: two tools that normalize to the same identifier (hyphens become `_`; Python keywords get a `_tool` suffix) cause an error.
3. If the code needs no tools, it runs through plain `/exec`, with Python wrapped in an `async def __user_main__()` shim.
4. Otherwise `POST /exec/programmatic` with the tool defs. While the status is `tool_call_required` (at most `maxRoundTrips`, default 20):
   - `executeTools` runs each call in parallel with `tool.invoke(normalizedInput, {metadata:{[ptcName]: true}})`.
   - MCP tuple or content-block outputs are unwrapped to text or parsed JSON.
   - Errors become `{is_error:true, error_message}`.
   - The results are POSTed back with the `continuation_token`.
5. `completed`: formatted like the code tools, with a session artifact. `error`: a sanitized message (`buildCodeApiExecutionErrorMessage` keeps only a whitelist of details).

The runtime session hint is sent on the first request only. PTC keeps a stateless prompt.

---

## 6. Web search pipeline (`src/tools/search/*`)

`createSearchTool(config)` assembles four parts:

**1. Search provider** (`searchProvider`, default `serper`):
- **serper**: `POST https://google.serper.dev/{search|images|videos|news}` with header `X-API-KEY` (`SERPER_API_KEY`).
  - Body: `{q, safe: off|moderate|active[safeSearch 0..2, default 1], num: 1..10 (default 8), type?, tbs:'qdr:<date>', gl: country}`.
- **searxng**: `GET <SEARXNG_INSTANCE_URL>/search` with `format=json, categories, language(all), safesearch, engines(default google,bing,duckduckgo), time_range`, optional header `X-API-Key` (`SEARXNG_API_KEY`).
  - Results are mapped to Serper shape. News is guessed from title keywords or URL paths. At most 6 images. `topStories` = the first 5 news items.
- **tavily**: `POST https://api.tavily.com/search`, Bearer auth.
  - Body: `{query, search_depth:basic, topic: news|general, max_results ≤20, time_range, country, include_*…}`.
- Also **keenable** (`https://api.keenable.ai/v1/search`, keyless fallback) and **crw** (`https://api.fastcrw.com/v1/search`).

**2. `executeParallelSearches`:** runs the main web search plus optional images, videos, and news sub-searches in parallel.
- A failed main search is fatal. Failed sub-searches are dropped.
- News is merged into `topStories` and deduplicated by link.
- Provider support flags: keenable is organic-only; tavily, keenable, and crw have no videos.

**3. `createSourceProcessor.processSources`:**
- `topStories` is capped at `numElements` (default 5).
- `proMode` (default true): scrape the organic links (and news links when `news`), skipping duplicates. When off, only the first Wikipedia link is scraped.
- The **scraper** (`scraperProvider`, default `firecrawl`) runs per link. Firecrawl request:
  - `POST {FIRECRAWL_BASE_URL|https://api.firecrawl.dev}/v2/scrape`, Bearer `FIRECRAWL_API_KEY`.
  - Body: `{url, formats:['markdown','rawHtml'], timeout:7500, onlyMainContent?, …}`.

  Other scrapers:
  - Serper: `https://scrape.serper.dev`, `X-API-KEY`.
  - Tavily: `POST https://api.tavily.com/extract`, batched.
  - Also crw and keenable.
- **Content processing** (`content.ts`): cheerio collects `<a>`, `<img>`, `<video>`, and iframes, and rewrites markdown links to markers `(link#N "title")`, `(image#N)`, `(video#N)`. References are kept as `{links, images, videos}`.
- Text is cleaned and truncated to `SEARCH_MAX_CONTENT_LENGTH` (default 50000).
- **Chunking:** `RecursiveCharacterTextSplitter` with separators `\n\n`, `\n`; chunkSize 150 and overlap 50, overridable via `SEARCH_CHUNK_SIZE` / `SEARCH_CHUNK_OVERLAP`. Overlap is clamped below size.

**4. Reranker** (`rerankerType`, default `cohere`):
- **jina:** `POST https://api.jina.ai/v1/rerank` (env `JINA_API_URL`), Bearer.
  - Body: `{model:'jina-reranker-v2-base-multilingual', query, top_n, documents, return_documents:true}`.
- **cohere:** `POST https://api.cohere.com/v2/rerank` (env `COHERE_API_URL`).
  - Body: `{model:'rerank-v3.5', query, top_n, documents}`.
- **rag-api:** `POST {RAG_API_URL}/v1/rerank` with a token supplier.
  - Body: `{profile:'fast-v1', query, candidates:[{id,text,base_score:0}] (≤50), top_n ≤25}`.
- **infinity** and **none**: no scoring. Chunks are passed through in order via `getDefaultRanking` with score 0.
- Timeout 10 s. `topResults` (default 5) gives the number of highlights per source.
- Every failure falls back to default ranking and is recorded in `metrics`, which flushes one summary log line per search.

**`expandHighlights(mainExpandBy=300, separatorExpandBy=150)`:**
- Each highlight is located in the content and widened by up to ±300 chars, then snapped to a natural boundary within a further 150-char window. Boundary priority: paragraph/newline, then sentence/semicolon/colon, then comma/space.
- The reference markers found in the highlight are tracked.
- **The raw content is always removed** from each source.

**Output formatting** (`format.ts` `formatResultsForLLM(turn, data, maxOutputChars)`):
- Highlight budget: `SEARCH_MAX_LLM_OUTPUT_CHARS`, default 50000. Highlights are kept in relevance order. The highlight at the boundary is truncated if at least 200 chars of budget remain, with `…[truncated]`. The rest are dropped, and an "N additional highlights omitted" note is added.
- Sections: `=== Web Results, Turn X ===`, `=== News Results ===`, Knowledge Graph, Answer Box, People Also Ask.
- Each source is rendered as:
  ```
  # Search 0: "Title"
  Anchor: turn{turn}search{i}
  URL / Summary / Date / Source
  ## Highlights
  ### Highlight 1 [Relevance: 0.93]
  ```text ... ```
  Core References:
  - link#3: <url>
      - Anchor: turn{turn}ref{k}
  ```
  Only link-type references are listed; mailto links and file extensions are skipped. They populate `references[]`.
- **Citation anchors** (in the tool description):
  - `turn{X}{search|news|image|ref}{Y}`
  - highlight span: `…`
  - group: `…`
- `turn` comes from `config.toolCall.turn`, the per-tool usage count injected by ToolNode.
- Tool return value: `[output, {web_search: {turn, ...data, references}, outcome?}]`, where the outcome is `Found N results for "q"` or `Search failed for "q"`.
- Provider errors are returned as data with `error`, not thrown.

---

## 7. Subagent system

### 7.1 Spawn tool and configs
- **Builder:** `buildSubagentToolParams(configs, {background, threadContinuation})` (`src/tools/SubagentTool.ts`) produces:
  - schema: `{intent?, description: string, subagent_type: enum<types>, run_in_background?: boolean (only when taskConfig is set), subagent_thread_id?: string (only when the store supports continuation)}`, with required `description` and `subagent_type`
  - a description that lists `- "type" (name): description`.
- **Config types** (`src/types/graph.ts`), a `SubagentConfig` in one of four forms:
  - eager `agentInputs`,
  - `self: true` (reuses the parent's source inputs),
  - lazy `resolveAgentInputs(ctx)` with a required `configId`,
  - `GraphSubagentConfig{kind:'graph', agents, edges(direct DAG), entryAgentId, resultAgentId}`.

  Shared options: `maxTurns` (default 25) and `allowNested` (default false).
- **Wiring:** `Graph.createAgentNode` (`src/graphs/Graph.ts` ~5173-5420) adds the tool as a **graph tool**, which makes it direct: it always runs in the SDK. It captures `toolCallId`, `thread_id`, the batch breaker scope, and the parent `configurable`.

### 7.2 Depth and child context rules (`childGraphConfig.ts`)
**Depth:**
- `maxSubagentDepth` defaults to 1. The tool is registered only if the depth is greater than 0.
- A child keeps `subagentConfigs` only if `allowNested`, and then gets `maxSubagentDepth = parentMax - 1`. Otherwise both are cleared.
- Graph subagents never nest.

**What a child starts without:**
- `initialSummary`, `discoveredTools`
- for `self`: `graphTools` and `compactionSemanticIndex`
- `toolDefinitions`, unless the parent has an `ON_TOOL_EXECUTE` handler (otherwise the child's event batch would hang)

**Child agent id:** `agentInputs.agentId`, or else `${parentAgentId}_sub_${suffix}`. For graphs: `${parent}_subgraph_${suffix}`.

**Child input:**
- `[...initialMessages, HumanMessage(description, {subagent hook-session and run-id markers}), ...prepared.messages]`
- The child never sees the parent's history.

**Configurable:**
- The parent `configurable` is inherited with `__pregel_*` and checkpoint keys removed. Host context merges on top.
- `executionContext = {rootRunId, hookSessionId, depth+1, ancestry:[...,{subagentRunId, subagentType, subagentKind, subagentAgentId, parentRunId, parentAgentId, parentToolCallId}]}`.
- `thread_id` = the child thread id when HITL is on, otherwise the inherited value, otherwise `childRunId`.

**Invocation:**
- `recursionLimit = maxTurns*3*(members) + (background ? 32 : 0)`
- callbacks: a forwarder plus a usage capture handler. These **replace** the inherited callback chain, so child streaming does not leak into the parent stream.
- `runName: subagent:<type>`

**Result:**
- Only the last non-empty AI text is returned (`filterSubagentResult`). For graphs, it is the result agent's final turn.
- Failures produce `Subagent error: <truncated msg>`.
- Stream-limit breaches abort the shared breaker and propagate.

**Hooks and lifecycle events:**
- `SubagentStart` hook: `deny`/`ask` produce `Blocked: …`.
- `SubagentStop` hook: observational.
- `ON_SUBAGENT_UPDATE` events: `{runId, parentRunId, subagentRunId, parentToolCallId, subagentType, subagentKind, subagentAgentId, memberAgentId, depth, ancestry, phase: start|run_step|…|stop|error, data (sanitized), label, timestamp}`.
- The child's `ON_TOOL_EXECUTE` is routed to the **parent's** handler, carrying the child `executionContext`. Hosts should treat that batch field as authoritative.

**Host context adapter** (`docs/subagent-context.md`, `SUBAGENT_CONTEXT_VERSION=1`):
- `prepare(input)` runs before construction and on retry or rebuild. It can supply messages, `configurable`, and per-member `agentSessions` (`codeSessionKey` plus `initialSessions`, which partition the code-session map). Unknown members are rejected.
- `complete(input, result)` projects the returned text. Delivery failure keeps the completed result, which can be retried without re-execution (`retryableDelivery`).

### 7.3 `SubagentExecutionRegistry`
The registry keys executions by an **address**:
- `identity = [threadId ?? parentRunId, checkpoint_id ?? 'root', parentAgentId, parentToolCallId, parentBatchKey]`
- `childThreadId = 'subagent:' + base64url(JSON(identity))` (plus the resume attempt id when durable)
- `childRunId = ${parentRunId}_sub_<encoded>`

`SubagentExecutionRecord` responsibilities:
- deduplicates concurrent `execute()` calls through a single pending promise,
- binds the definition, invocation, and settlement fingerprints; a later change is rejected with "config changed" or "invocation changed",
- tracks phases (registered, active, interrupted, failed, …),
- holds preparation leases that roll back created checkpoints or approval scopes on failure,
- stores a settled output so a parent replay returns the persisted ToolMessage (`getSettledToolOutput` / `persistSettledToolOutput`, exposed through `SUBAGENT_REPLAY_CONTROLLER` on the tool).

It is durable only when HITL is on and a checkpointer exists. HITL inside a child raises a scoped `GraphInterrupt` with a resume manifest.

### 7.4 Background (detached) tasks and usage sink
**`run_in_background:true` → `executeInBackground`.** It requires:
- `taskConfig{store, scopeId}`
- depth greater than 0
- HITL off
- a parent tool call id

**Setup:**
- It forks the hook session and copies the `ON_TOOL_EXECUTE` handler into a detached registry.
- It calls `store.start({scopeId, idempotencyKey: JSON([parentRunId,parentAgentId,parentToolCallId]), requestFingerprint, threadId?, input, subagentKind, subagentType, run(runtime, initialMessages)})`.

**Immediate return:** a JSON string `{background_task_id, subagent_thread_id?, tool:'subagent', subagent_type, status, message}`. Rejections return `{status:'rejected', message}` for capacity, conflict, or thread_unavailable.

**`SubagentTaskRuntime`:**
- `signal`
- `shouldPreempt()`, which drives cooperative seals (up to 32)
- `drain(boundary)` for steer, queue, or interrupt messages
- `closeTurn()`: a queued follow-up re-invokes the child with `messages + injected`
- `reportProgress`

**`InMemorySubagentTaskStore` defaults:**
- 30 min task timeout
- 1 h TTL after completion
- 10 running per scope, 100 in total
- results capped at 100k chars
- 32 controls per task

It supports `get`, `list`, `claim` (the result can be claimed once), and `control` (`steer|queue|interrupt|cancel|cancel_message`). Host tools do the polling.

**Usage sink.** `usageSink(event)` receives `{usage, model, provider (INVOKED_PROVIDER), subagentType, subagentKind, subagentRunId, subagentAgentId, memberAgentId, parentRunId, depth, ancestry, runId: ROOT run}` for every model call a child makes. Calls are awaited and errors swallowed. Nested levels forward upward.

---

## 8. Python re-implementation guidance

### 8.1 Node type
Do **not** use `langgraph.prebuilt.ToolNode`. Write a custom async node (`async def tool_node(state, config)`) and return `list[BaseMessage] | Command | list[Command|dict]`. The prebuilt node lacks:
- event dispatch,
- mixed direct/event ordering,
- the invalid-call promotion,
- HITL batching,
- handoff aggregation into `Send`s.

Keep `tools_condition` semantics.

### 8.2 Host contract
Model `ToolExecuteBatchRequest` as a pydantic model plus an `asyncio.Future`. Replace the resolve/reject callbacks with this API:

```python
class ToolExecutor(Protocol):
    async def execute_batch(self, req: ToolExecuteBatchRequest,
                            on_result: Callable[[ToolExecuteResult], Awaitable[None]] | None) -> list[ToolExecuteResult]: ...
```

Also emit the event through LangChain `adispatch_custom_event("on_tool_execute", …)` for host parity. Keep the field names exactly (`toolCallId`, `codeSessionContext`, `storage_session_id`, …) so the TypeScript LibreChat host stays compatible, and pin them with pydantic aliases.

### 8.3 Concurrency
- Direct path: `asyncio.gather(..., return_exceptions=True)`, then rethrow with interrupts ranked below other errors. Run the interrupting group first.
- Freeze the output-reference snapshot before any `await`.
- Compose cancellation with an `asyncio.Event` or a task-group cancel scope standing in for `AbortSignal`. Check the breaker at each stage: entry, before dispatch, after the direct group.
- `interrupt()` in Python LangGraph must be called inside the node context.

### 8.4 HTTP
- Use one shared `httpx.AsyncClient` per base URL, with `follow_redirects=False`, timeouts per provider (serper/searxng 10 s, firecrawl 7.5 s, rerank 10 s, error body 1 s / 64 KB), and proxy from `PROXY`/`NO_PROXY`.
- PTC cancel: on cancellation, send the fire-and-forget `POST {endpoint}/cancel` with the request id (2 s timeout).
- Stream error bodies with bounds.

### 8.5 Schemas
- Keep the tool schemas as **raw JSON Schema dicts** (the `intent`-first property order matters; Python dicts preserve insertion order). Bind them through `StructuredTool.from_function(args_schema=dict)` or `convert_to_openai_tool`.
- Use pydantic for internal wire models: `CodeEnvFile` as a discriminated union on `kind`, `ExecuteResult`, `ProgrammaticExecutionResponse`, `ToolExecuteResult`.
- Implement `coerce_args_for_schema` exactly: lossless coercion only.

### 8.6 Suggested module layout
```
agents/tools/node.py            # ToolNode run(), mixed batching, invalid-call promotion, handoffs
agents/tools/contract.py        # pydantic ToolCallRequest/BatchRequest/Result, CallerCapabilitySnapshot
agents/tools/lifecycle.py       # hooks, HITL approval payloads, decision normalization
agents/tools/eager.py           # plan builder, arg coercion, stable_stringify, eager registry
agents/tools/truncation.py      # head/tail, bounded structured serialization, compact content
agents/tools/code_session.py    # update/retain session merge by (storage_session_id,id)
agents/tools/output_refs.py     # {{toolNturnM}} registry
agents/tools/code_api/{client,execute_code,bash,ptc,bash_ptc,errors}.py
agents/tools/local/{engine,coding_tools,ptc_bridge}.py
agents/tools/tool_search.py     # BM25 (rank_bm25), regex-sandbox mode, MCP server resolution
agents/tools/web_search/{providers,scrapers,content,chunking,rerankers,highlights,format,tool}.py
agents/tools/subagent/{tool,executor,registry,child_config,task_store,usage}.py
```

### 8.7 Pitfalls to preserve
1. **Message order:** keep tool_call → tool_result adjacency, and put injected or hook messages after all results. Stamp synthetic messages with `additional_kwargs.role='system'`.
2. **Invalid tool calls:** promote them with a same-id replacement AIMessage, or Anthropic returns a 400 on the next turn (and on HITL resume).
3. **Exactly-once completion events** across the early `onResult` emission, eager emission, `errorHandler` ownership, deferred blocked calls, and replay.
4. **Eager mismatch:** return an error and suppress eager execution for that tool name. Never re-run the tool.
5. **Two session ids:** never overwrite a file's `storage_session_id` with the exec `session_id`. Delete files only on an explicit `deleted_files` entry that matches the baseline identity.
6. **Host-controlled fields:** strip model-supplied `runtime_session_hint`, `workspace_instance_id`, and `intent` from Code API bodies. Strip SDK-private configurable keys before sending to the host.
7. **Turn counters** are per tool name, and web search anchors depend on them. Keep direct and eager counters consistent.
8. **Web search** must remove raw scraped content before output. Budget the highlights, not the snippets.
9. **Subagent children** replace the callback chain. Usage must go through the sink (with the root runId), and child `ON_TOOL_EXECUTE` goes to the parent handler.
10. **Truncation boundaries:** truncate by code units without splitting surrogates. In Python, slice by `str` code points, which is naturally safe, but keep the 70/30 head/tail split and the newline snapping.
11. **Tool search regex mode:** it posts to the Code API **without auth headers**. Treat this as a known gap. A Python port should forward auth, or default to `local` mode.
