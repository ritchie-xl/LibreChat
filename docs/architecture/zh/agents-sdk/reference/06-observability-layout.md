# @librechat/agents：可观测性、提示词、标题、仓库结构与移植顺序

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

所有路径均相对于 SDK 仓库根目录。检出的提交为 `2d5653d v3.9.3 (#559)`。

---

## 1. 仓库结构

LOC（代码行数）只统计非测试的 `.ts` 文件，不含 `*.test.ts`、`*.spec.ts`、`__tests__/`、`src/specs`、`src/scripts` 和 `src/test`。非测试源码合计约 **123k LOC**。

| 路径 | 职责 | 非测试 LOC | 备注 |
|---|---|---|---|
| `src/run.ts` | `Run` 类。`Run.create`、`processStream`、`resume`（HITL）、`generateTitle`、`generateActivityLabel`、`generateReasoningLabel`、`generateActivityPhaseLabel`。负责装配 Langfuse、流前钩子与中断。 | 3,471 | LibreChat 调用的公共门面 |
| `src/stream.ts` | 聊天模型流处理器、内容聚合器（`createContentAggregator`）、运行步骤（run step）簿记 | 2,886 | |
| `src/events.ts` | `HandlerRegistry`、`ToolEndHandler`、`ModelEndHandler` | 236 | |
| `src/langfuse*.ts`（7 个文件）+ `src/instrumentation.ts` | Langfuse/OTel 链路追踪 | 3,719 + 214 | 详见第 4 节 |
| `src/lazyRequire.ts` | 通过 `createRequire` 同步懒加载模型提供商 SDK。它是唯一触碰 `import.meta` 的模块。 | 67 | |
| `src/provider-registration.ts` | 公共入口 `registerProvider`，以及用于声明合并的 `CustomProviderOptionsMap` 类型 | 29 | ADR 0003 |
| `src/index.ts` | 根 barrel 导出文件 | 121 | |
| `src/llm/` | 模型提供商封装与注册表。各子目录 LOC：openai 4.9k、anthropic 3.4k、bedrock 2.6k、google 1.8k、vertexai 0.6k、openrouter 0.4k、mistral、`stream/`（平滑器）0.9k。另有 `init.ts`、`invoke.ts`（`attemptInvoke`、回退）、`prepareProviderRequest.ts`、`preempt.ts`、`streamLimits.ts`、`contextOverflowRecovery.ts`、`contextPressureMeter.ts`、`fake.ts`。 | 20,434 | 含 spec 共 112 个文件 |
| `src/tools/` | ToolNode、工具处理器、代码/bash 执行器、程序化工具调用（PTC）、ToolSearch（BM25）、Calculator、SkillTool、子智能体（7.5k）、网页搜索（7.0k）、本地执行引擎（5.7k）、cloudflare（2.9k）、intent 参数、流封存（seal） | 38,201 | 最大的模块 |
| `src/messages/` | `formatAgentMessages`（LibreChat 负载转 LangChain 消息）、内容处理、提示缓存标记、裁剪、角色交替、reducer、来源追溯（provenance）、工具历史投影、移交提示 | 17,577 | |
| `src/graphs/` | `Graph.ts`（StandardGraph）、`MultiAgentGraph.ts`（移交工具、并行扇出）、`handoff.ts`、工厂 | 8,373 | |
| `src/utils/` | token、工具内容、截断、错误、标题链、回调、代理、杂项 | 6,309 | 见下文 |
| `src/types/` | 共享 TS 类型（`graph.ts` 含 `LangfuseConfig` 和 `AgentInputs`；另有 `run.ts`、`llm.ts`、`stream.ts`、`tools.ts`、…） | 4,983 | |
| `src/session/` | `AgentSession`、`JsonlSessionStore`、会话投影 | 3,646 | 程序化会话 |
| `src/summarization/` | 摘要节点、共享提示词与承载消息（carrier）、语义索引 | 2,892 | ADR 0005–0008 |
| `src/hooks/` | `HookRegistry`、`executeHooks`、工具与工作区策略钩子、匹配器 | 2,723 | |
| `src/eventActor/` | `EventActorExecutor`：持久化、事件驱动的子线程 | 2,506 | 叶子模块：只导入 LangChain/LangGraph 及自身 |
| `src/agents/` | `AgentContext`（系统提示词组装、token 缓存、移交前言）、`projection.ts` | 2,360 | |
| `src/responses/` | 兼容 OpenAI Responses API 的 SSE 写出器 | 677 | |
| `src/prompts/` | 标签提示词，以及遗留的 supervisor 和 task-manager 提示词 | 603 | 未从根 barrel 重新导出 |
| `src/hitl/` | 审批复核、`askUserQuestion(s)` 中断 | 548 | |
| `src/openai/` | 兼容 OpenAI Chat-Completions 的 SSE 写出器 | 404 | |
| `src/common/` | 枚举（`GraphEvents`、`Providers`、`ContentTypes`、`Constants`、`TitleMethod`、…）与常量（运行名称、限制值） | 370 | 叶子模块 |
| `src/langchain/` | `@langchain/core` 部分组件的重新导出门面，让宿主共享同一套类图 | 57 | |
| `src/specs/` | 68 个行为/集成测试文件 + `spec.utils.ts` | （38.9k 测试 LOC） | |
| `src/scripts/` | 92 个手动运行脚本、基准测试和标签评测工具 | （23.1k） | |
| `src/__tests__/`、`src/test/mockTools.ts` | 流测试、模拟工具 | | |
| `config/` | `package-entries.mjs`（构建入口）、`circular-deps.mjs`（及其测试）、`clean.js` | | |
| `test/stubs/` | Jest 桩：`lazyRequire.ts`、`mistralai.ts` | | |
| `docs/`、`docs/adr/` | 14 篇设计/基准文档和 9 篇 ADR | | |
| `scripts/sort-imports.ts` | 强制执行导入排序约定 | | 由 lint-staged 使用 |

**值得关注的工具模块**（`src/utils/`）：
- `tokens.ts`（1.5k）：`createTokenCounter`、`TokenEncoderManager`、`getTokenCountForMessage`、图片与文档 token 估算器、`apportionTokenCounts`。
- `toolContent.ts`（2.7k）：工具内容的有界序列化与压缩、computer-call 截图。
- `truncation.ts`：`HARD_MAX_TOOL_RESULT_CHARS`、`truncateToolResultContent`、代理对（surrogate）安全的切片。
- `errors.ts`：上下文溢出检测、`getContextOverflowInfo`。
- `callbacks.ts`：`appendCallbacks`、`findCallback`、`filterCallbacks`。
- `title.ts`：标题链。
- `llmConfig.ts`：脚本用的模型配置。
- `llm.ts`：`isOpenAILike`、`isAnthropicLike`、…
- `proxy.ts`：HTTPS/SOCKS 代理 agent。
- `redactSecrets.ts`。
- `misc.ts`：`isPresent`、`parseBooleanEnv`、`composeAbortSignals`。
- `events.ts`：`safeDispatchCustomEvent`、`emitAgentLog`。
- `run.ts`：`RunnableCallable`、`sleep`。
- `schema.ts`：zod 转 JSON-schema。
- `toolSessions.ts`。
- `logging.ts`：把控制台输出同时写入文件；供脚本使用。

**`src/lazyRequire.ts`。**
- `requireInternalModule('llm/openai/index')` 加载构建产物旁边格式匹配的同级文件（`.mjs` 或 `.cjs`）。这样懒加载的提供商与调用方处于同一套 LangChain 类图中。
- 在源码模式（`tsx`）下，这些模块必须先通过 `registerSourceModeModules` 预注册。`src/llm/providers.eager.ts` 负责这件事，`run.ts` 在 `!isBuiltRuntime()` 时动态导入它。
- `requireLazyModule(spec)` 在首次使用时加载第三方包。
- Jest 将它映射到 `test/stubs/lazyRequire.ts`。
- 目的：宿主不必在启动时承担每个提供商 SDK 的初始化开销。`src/index.ts` 有意不再重新导出提供商类；它们放在子路径入口里。

**移植中需要关注的公共常量**（`src/common/constants.ts`）：
- 运行名称：`STANDARD_GRAPH_RUN_NAME='AgentGraph'`、`MULTI_AGENT_GRAPH_RUN_NAME='MultiAgentGraph'`、`AGENT_MODEL_CALL_RUN_NAME='AgentModelCall'`、`ACTIVITY_LABEL_RUN_NAME='StepLabel'`、`REASONING_LABEL_RUN_NAME='ReasoningLabel'`、`ACTIVITY_PHASE_RUN_NAME='MultiStepLabel'`、`ACTIVITY_PHASE_LABEL_RUN_NAME='MultiStepLabelGeneration'`。
- 限制值与乘数：`DEFAULT_RECURSION_LIMIT=50`、`ANTHROPIC_TOOL_TOKEN_MULTIPLIER=2.6`、`DEFAULT_TOOL_TOKEN_MULTIPLIER=1.4`、`DEFAULT_MAX_SEALS=8`。

**枚举**（`src/common/enum.ts`）：
- `GraphEvents`：`on_run_step*`、`on_message_delta`、`on_reasoning_delta`、`on_tool_execute`、`on_summarize_*`、`on_subagent_update`、`on_agent_log` 和 `on_context_usage` 事件，以及 LangChain 的 `on_chat_model_*` / `on_chain_*` / `on_tool_*` 事件。
- `Providers`：`openAI`、`vertexai`、`bedrock`、`anthropic`、`mistralai`、`mistral`、`google`、`azureOpenAI`、`deepseek`、`openrouter`、`xai`、`moonshot`。
- `GraphNodeKeys`：`tools=`、`agent=`、`summarize=`、`router`、`pre_tools`、`post_tools`。
- `Constants`：工具名，如 `execute_code`、`tool_search`、`run_tools_with_code`、`lc_transfer_to_`、`subagent`、`bash_tool`、`read_file`、…
- `TitleMethod`：`structured`、`functions`、`completion`。

---

