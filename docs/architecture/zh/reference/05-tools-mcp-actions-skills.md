# 工具、MCP、Actions、技能、代码执行、网页搜索与记忆

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

这里涉及两个包：
- **`@librechat/agents`** 是基于 LangGraph 的智能体 SDK。它没有内置（vendored）在本仓库中。它提供 `createCodeExecutionTool`、`createSearchTool`、`createToolSearch`、`createBashExecutionTool`、`Calculator` 以及 `ON_TOOL_EXECUTE` 事件循环。本文根据 LibreChat 对它的调用方式来描述其契约。
- **`librechat-data-provider`** 是共享的 schema 与常量包，位于 `packages/data-provider/src`。

---

## 1. 工具解析管线（逐步说明）

智能体以 **“事件驱动”（仅定义）模式** 运行。初始化时，后端只解析出每个工具的 JSON-schema *定义*。真正可执行的实例是惰性创建的，只有在模型实际发出工具调用时才会创建。

### 1a. 初始化阶段：`loadAgentTools({definitionsOnly: true})` → `loadToolDefinitionsWrapper`
文件：`api/server/services/ToolService.js:800-1575`。它由 `packages/api/src/agents/initialize.ts` 中的 `initializeAgent` 调用，并以 `loadTools` 的形式注入。

1. **短路返回。** 如果 `agent.tools` 为空，或只包含旧版的 `context`/`ocr` 条目，则直接返回空。
2. **解析能力**（`resolveAgentCapabilities`，第 322 行）。能力集合来自 `endpoints.agents.capabilities`；临时智能体回退到默认值。`AgentCapabilities` 枚举（`packages/data-provider/src/config.ts:644`）为：`execute_code, stateful_code_sessions, file_search, web_search, artifacts, subagents, actions, context, skills, memory, ask_user_question, tools, chain, ocr, run_in_background, tool_intents, deferred_tools, programmatic_tools, end_after_tools, hide_sequential_outputs`。
3. **角色权限**（`resolveAgentToolPermissions`，由 `packages/api/src/tools/rolePermissions.ts` 支撑）。`file_search`→FILE_SEARCH、`execute_code`→RUN_CODE、`web_search`→WEB_SEARCH，按用户角色检查。
4. **过滤 `agent.tools`**（第 869-895 行）。每个工具的准入条件如下：

   | 工具 | 保留条件 |
   |---|---|
   | `file_search`、`execute_code`、`web_search` | 对应能力 **且** 角色权限 |
   | `memory` | `memory` 能力 |
   | `ask_user_question` | `ask_user_question` 能力 |
   | action 工具（`isActionTool` = 名称包含 `_action_`） | `actions` 能力 |
   | MCP 工具（包含 `_mcp_`） | `tools` 能力 **且** MCP USE 权限 |
   | 其他所有工具 | `tools` 能力 |

5. **解析代码执行上下文**（`resolveCodeExecutionContext` 加 `resolveCodeExecutionWorkspaceContext`）。见 §6。
6. **内容过滤预检。**
   - `assertToolResourcesAllowed` 检查智能体挂载的文件。
   - `prepareActionSnapshotForTools` 从 Mongo 一次性加载智能体的 Action 文档。该快照会被复用，以避免 TOCTOU 漂移；PII 过滤器会在建立任何 MCP 连接之前检查它。
7. **MCP 上下文。**
   - `resolveMcpServerContext(req)` 返回 `configServers`（管理员覆盖）、`serverNames`（规范化后）和 `rawServerNames`。它位于 `api/server/services/MCP.js:208`。
   - 冲突审计：`resolveCollisionAuditNames`、`findShadowedServerNames`、`buildServerNameAliases`。规范化后的服务器名与其他服务器冲突的工具会被丢弃；审计不完整时按失败关闭（fail closed）处理。
   - `getUserMCPAuthMap` 从 `pluginAuth` 集合加载每个用户的 `customUserVars`。键的形式为 `mcp_<serverName>`。
8. **`loadToolDefinitions(params, deps)`**（`packages/api/src/tools/definitions.ts`）把每个工具名归入以下三条路径之一：
   - **内置：** `isBuiltInTool`，然后从静态注册表 `packages/api/src/tools/registry/definitions.ts` 调用 `getToolDefinition(name)`，其中保存了 google、dalle、flux、open_weather、wolfram 等工具的 JSON schema。工具包（toolkit）在这里展开：`image_gen_oai` 会同时带入 `image_edit_oai`（`tools/toolkits/mapping.ts`）。
   - **MCP：** 键为 `<tool>_mcp_<normalizedServer>`，或 `sys__all__sys_mcp_<server>`，后者表示“该服务器的全部工具”。
     - `deps.getOrFetchMCPServerTools` 先查工具缓存（`getMCPServerTools`）。未命中时运行 `reinitMCPServer`：建立连接、列出工具，并可能启动 OAuth。
     - 如果所选工具在过期的目录中缺失，`refreshMCPServerTools` 会强制建立新连接。
     - Schema 通过 `resolveJsonSchemaRefs` + `normalizeJsonSchema` 规范化。对 Gemini/Vertex 提供商，还会由 `sanitizeGeminiSchema` 展平。
   - **Actions：** `deps.getActionToolDefinitions` 用 `openapiToFunction` 重新解析每份存储的 OpenAPI 规范，并输出名称与智能体工具列表匹配的函数签名。
9. **构建注册表。** `buildToolClassification` / `buildToolRegistryFromAgentOptions`（`packages/api/src/tools/classification.ts`）生成一个 `LCToolRegistry`，即 `Map<name, {name, description, parameters, allowed_callers, defer_loading, toolType, serverName}>`。每个工具的 `agent.tool_options[name] = {defer_loading, allowed_callers: ['direct'|'code_execution']}` 驱动两项功能：
   - **延迟工具 / 工具发现。** 当启用 `deferred_tools` 且有任一工具设置了 `defer_loading` 时，会加入一个本地 `tool_search` 工具（`createToolSearch({mode:'local', toolRegistry})`）。延迟工具对模型隐藏，直到模型通过搜索找到它们。
   - **程序化工具调用（Programmatic Tool Calling，PTC）。** 当启用 `programmatic_tools` 和 `execute_code`，且某个工具允许 `code_execution` 调用方时，会加入一个 `bash_programmatic_tool_calling` 工具（`createContextProgrammaticBashTool`，`packages/api/src/code/command.ts`）。此后沙箱中的代码可以调用其他工具。`packages/api/src/agents/ptc.ts` 用 Proxy 包装 PTC 工具映射，使内部调用能流式输出进度（`instrumentPtcToolMap`）。
10. **MCP OAuth 等待。**
    - 任何需要 OAuth 的服务器（`pendingOAuthServers`）都会向客户端发出一个合成的运行步骤：`buildMCPAuthRunStepEvent`，外加一个携带 `authURL` 的 delta 事件。
    - 随后调用在 `reinitMCPServer({returnOnOAuth:false, oauthStart, oauthEnd})` 中阻塞，最长 2 分钟。
    - 成功后再次调用 `loadToolDefinitions`。
    - 如果智能体期望有 MCP 工具但一个都没解析出来，则抛出 `ExpectedMCPToolsUnavailableError`。
11. **上下文字符串。**
    - `toolContextMap` 保存静态说明，例如网页搜索的引用格式。
    - `dynamicToolContextMap` 保存每轮的上下文：网页搜索日期，以及由 `primeCodeFiles` / `primeSearchFiles` 生成的文件列表（这些文件也会上传到 Code API）。
12. **`initializeAgent` 中其余的注册**（`packages/api/src/agents/initialize.ts`）：
    - 记忆工具：`registerMemoryTools`（第 2110 行）。
    - 代码工具：`registerCodeExecutionTools` → `bash_tool`、`read_file`、`search_workspace`、`list_workspace_files`。
    - 文件编写：`registerFileAuthoringTools` → `create_file`、`edit_file`。
    - 技能：`injectSkillCatalog` → `skill` 工具加一个目录文本块。
    - 后台执行与意图参数：`backgroundToolNames` / `intentToolNames`。
    - 子智能体发现：`packages/api/src/agents/discovery.ts` 在智能体图的边上做 BFS，检查 VIEW 权限，并用同一条管线初始化每个子智能体。

**返回给运行的内容：** `{toolDefinitions, toolRegistry, mcpAvailableTools, userMCPAuthMap, requestScopedConnections, toolContextMap, dynamicToolContextMap, codeExecutionContext, primedCodeFiles, actionsEnabled, oauthActionToolNames, mcpToolAliases, repositoryInstructionSource}`。运行在 `agentToolContexts` 中为每个智能体保留一个条目。

### 1b. 执行阶段：`ON_TOOL_EXECUTE` → `loadToolsForExecution`
SDK 发出一批工具调用。`createToolExecuteHandler`（`packages/api/src/agents/handlers.ts:5488`）执行以下操作：
- 过滤被屏蔽的参数内容。
- 自行处理宿主原生工具：`skill`、`read_file`、`create_file`/`edit_file`、后台工具、子智能体任务。
- 调用宿主的 `loadTools` 回调（`api/server/services/Endpoints/agents/initialize.js:447`），即 `loadToolsForExecution`（`ToolService.js:2078`）。

`loadToolsForExecution` 接着：
- 针对*正在执行*的智能体重新检查能力和 RUN_CODE 权限。
- 构建 `tool_search`、PTC 工具和 `bash_tool`。`bash_tool` 有两种变体：托管环境用 `createBashExecutionTool`，附加环境用 `createAttachedWorkspaceBashTool`。
- 通过 `api/app/clients/tools/util/handleTools.js` 中的 `loadTools` 实例化常规工具。
- 通过 `loadActionToolsForExecution` 实例化 action 工具。
- 返回 `{loadedTools, configurable}`。`configurable` 携带 `userMCPAuthMap`、`requestBody`、`requestScopedConnections`、`codeExecutionContext`、`toolRegistry`、`ptcToolMap` 和 `backgroundToolNames`。
- **`callerCapabilities.ts`** 只接受带版本号（v1）的 SDK 快照，描述哪些工具可直接调用、哪些可由代码调用。它会与受信任的注册表取交集，且从不被当作授权依据。
- **`toolValidation.ts`** 把 schema 校验错误转换为结构化、保护隐私的诊断信息。

### 1c. `handleTools.js` 中的 `loadTools`（实例工厂）
对每个工具，它先重新检查角色门槛，然后分派：

| 工具 | 构建的内容 |
|---|---|
| `execute_code` | `primeCodeFiles`，然后 `createCodeExecutionTool({user_id, files, authHeaders, ...codeExecutionContext})` |
| `file_search` | `primeSearchFiles` 加 FILE_CITATIONS 权限检查，然后 `createFileSearchTool`（`util/fileSearch.js`）。它以 `{file_id, query, k:5, entity_id?}` 调用 RAG API 的 `/query` |
| `web_search` | `loadWebSearchAuth`，然后 `createSearchTool({...authResult, httpAgent, httpsAgent (SSRF-safe), onSearchResults, onGetHighlights})` |
| `ask_user_question` | `createAskUserQuestionTool`（一个 LangGraph 中断） |
| `set_memory` / `delete_memory` | `buildInlineMemoryTool` |
| MCP 键 | 按服务器分组，然后调用 `createMCPTools`（`all` 占位符）或逐个工具调用 `createMCPTool`。服务器按顺序处理；失败的服务器被跳过 |
| 自定义构造器 | `image_gen_oai`、`gemini_image_gen` |
| Manifest 类 | `loadToolWithAuth(user, authFields, Ctor)` |

**凭据解析** 由 `loadAuthValues`（`api/server/services/Tools/credentials.js`）完成。对每个认证字段（备选项用 `||` 分隔），如果设置了 `process.env[field]` 且其值不等于 `user_provided`，就使用它。否则读取该用户的 `pluginAuth` 行，并用 `PluginService.js` 中的 `getUserPluginAuthValue` 解密。

**`pluginAuth` schema**（`packages/data-schemas/src/schema/pluginAuth.ts`）：`{authField, value (AES-CBC encrypted), userId, pluginKey, tenantId}`，索引为 `(userId, pluginKey, authField, tenantId)`。
- 由 `POST /api/user/plugins` 写入，请求体 `{pluginKey, action: install|uninstall, auth}`（`UserController.updateUserPluginsController`）。
- 同一存储还保存 MCP `customUserVars`（`pluginKey = mcp_<server>`）和网页搜索的用户密钥。

**工具列表与直接调用。**
- `GET /api/agents/tools` → `PluginController.getAvailableTools`。它把 `manifest.json` 与缓存的工具定义合并，并将工具标记为 `authenticated`。
- `GET /api/agents/tools/:toolId/auth` → `verifyToolAuth`。
- `POST /api/agents/tools/:toolId/call` → `controllers/tools.js callTool`。这是从 UI 直接重新运行 `execute_code`。结果持久化到 ToolCall 集合，代码输出会被处理成文件。

**启动。** `api/server/services/start/tools.js` 加载结构化工具类，并把它们格式化为 OpenAI 函数工具。应用 yaml 中的 `includedTools`/`filteredTools`。结果存入工具缓存（`CacheKeys.TOOL_CACHE`）。

**工具命名约定**（`packages/data-provider/src/config.ts:4001-4019`、`types/tools.ts:120`）：
- MCP：`<tool>_mcp_<server>`；前缀 `mcp_`；“全部”占位符 `sys__all__sys`。
- Actions：`<operationId>_action_<encodedDomain>`，其中域名分隔符 `---` 被规范化为 `_`。如果域名编码后不超过 10 个字符，就直接编码（`.`→`---`）；否则取主机名的 base64 前缀，并通过 `ENCODED_DOMAINS` 缓存映射回原域名。

---

## 2. 工具类别

| 类别 | 标识符 | 定义来源 | 执行器 | 认证 / 凭据 |
|---|---|---|---|---|
| 内置 / manifest 插件 | `google, dalle, flux, wolfram, open_weather, tavily_search_results_json, traversaal_search, stable-diffusion, azure-ai-search, calculator, image_gen_oai(+image_edit_oai), gemini_image_gen, ask_user_question` | `api/app/clients/tools/manifest.json`、`packages/api/src/tools/registry/definitions.ts` | `api/app/clients/tools/structured/*` 中的 LangChain `Tool` 类 | 环境变量，或每用户的 `pluginAuth`（`authConfig[].authField`，备选项用 `||`） |
| MCP | `<tool>_mcp_<server>` | 来自 MCP 服务器的实时 `tools/list`，带缓存 | 经 `services/MCP.js createToolInstance` 调用 `MCPManager.callTool` | 服务器级（管理员 apiKey、headers、OAuth、OBO、`customUserVars`） |
| Actions（OpenAPI） | `<opId>_action_<domain>` | `Action.metadata.raw_spec` → `openapiToFunction` | `ActionRequest` 执行器（`data-provider/src/actions.ts`） | 无 / API 密钥（basic、bearer、自定义 header）/ OAuth2 授权码；密钥以 AES 加密 |
| 技能 | `skill`、`read_file`、`create_file`、`edit_file` | Skill 集合，加上部署目录和插件目录 | `packages/api/src/agents/handlers.ts` 中的宿主处理器 | Skill 资源上的 ACL |
| 代码执行 | `execute_code`（旧版）、`bash_tool`、`read_file`、`search_workspace`、`list_workspace_files`、`bash_programmatic_tool_calling` | SDK 定义加 `agents/tools.ts` | 外部 Code API（`/exec`、`/upload`、`/download`、`/workspace-tools/execute`） | Code API 密钥或签发的 JWT；RUN_CODE 权限 |
| 网页搜索 | `web_search` | SDK `WebSearchToolDefinition` | SDK `createSearchTool`（搜索 → 抓取 → 重排序） | 环境变量 / 用户密钥（`webSearch` yaml） |
| 文件搜索 | `file_search` | 内置 | RAG API `/query` | FILE_SEARCH 权限，访问 RAG API 的 JWT |
| 记忆 | `set_memory`、`delete_memory`（内联）；后台记忆智能体 | `agents/memory.ts getMemoryToolDefinitions` | Mongo `memoryEntries` | MEMORIES 权限 |
| 工具发现 | `tool_search` | SDK `ToolSearchToolDefinition` | 本地注册表搜索 | `deferred_tools` 能力 |
| 子智能体 | task 工具 / 移交边 | 智能体图（`discovery.ts`、`edges.ts`、`lazySubagents.ts`） | 嵌套的 LangGraph 运行 | 子智能体上的 VIEW ACL，`subagents` 能力 |

---

## 3. MCP 架构

### 组件（`packages/api/src/mcp/`）
- **`registry/MCPServersRegistry.ts`**：服务器配置的唯一事实来源，分为三层。
  - YAML 缓存：`librechat.yaml` 中的 `mcpServers` 块，加上 Agent Plugins 的服务器。
  - Config 层缓存：存储在数据库配置中的管理员覆盖。
  - 数据库存储库（`registry/db/ServerConfigsDB.ts`）：用户在 `mcpServer` 集合中创建的服务器，带 ACL。
  - `getServerConfig` 的查找顺序：读穿缓存 → YAML → 数据库，然后叠加 config 层的候选项。用户来源和进程承载（stdio）的条目永远不会被叠加。
  - 缓存可以放在 Redis 中。存储的值用 `encryptV2` 加密。
  - 它还负责解析允许列表（`mcpSettings.allowedDomains/allowedAddresses`）。
- **`registry/MCPServerInspector.ts`**：在启动/检查时建立连接。它先在连接前强制执行域名允许列表，然后探测 OAuth（`oauth/detectOAuth.ts`）、能力、服务器说明（server instructions）和工具。结果是一个 `ParsedServerConfig`。失败时写入一个 `inspectionFailed` 占位记录，5 分钟后重试。
- **`registry/MCPServersInitializer.ts`**：启动时检查所有 YAML 服务器。
- **`connection.ts`（`MCPConnection`）**：封装 MCP TypeScript SDK 的 `Client`。传输层在 `constructTransport`（第 1473 行）中构建：
  - `stdio`：`StdioClientTransport({command, args, env: {...defaultEnv, ...env}, cwd})`。
  - `websocket`：`WebSocketClientTransport`，带 SSRF DNS 预检。
  - `sse`：`SSEClientTransport`，使用自定义的 undici fetch/dispatcher，具备 SSRF 安全的连接、可选代理、`sseReadTimeout`，并从 OAuth 令牌注入 Bearer。
  - `streamable-http` / `http`：`StreamableHTTPClientTransport`，使用相同的 fetch 包装，并在关闭时显式发送会话 DELETE。
  - 重连：最多 3 次，带退避。遇到 429 或需要 OAuth 时停止。熔断器（`CB_*` 环境变量设置）限制连接/断开的循环次数。
  - `tools/list` 分页有上限：50 页、1000 个工具、5 MiB、30 秒。
  - 通过 `ToolListChangedNotificationSchema` 处理 `notifications/tools/list_changed`。
  - 健康检查使用 `ping`，服务器未实现时采用回退方案。
- **`MCPConnectionFactory.ts`**：创建连接，并负责 OAuth、OBO 和令牌刷新逻辑：`handleOAuthRequired`、`attemptSilentTokenRefresh`、未认证时的工具列表回退，以及 `discoverTools`。
- **`ConnectionsRepository.ts`**：按作用域划分的连接池（`ownerId` 为 undefined 表示应用作用域）。生命周期状态转换按服务器串行化，连接并发数为 3。
- **`UserConnectionManager.ts` → `MCPManager.ts`**（单例）：
  - `appConnections` 是一个 `ConnectionsRepository`。
  - `userConnections` 是一个 `Map<userId, Map<server, MCPConnection>>`，配合 `userLastActivity` 做空闲驱逐。默认空闲超时为 15 分钟（`MCP_USER_CONNECTION_IDLE_TIMEOUT`）。
  - 借用方租约和延迟释放可以防止空闲清扫杀掉仍在使用中的连接。
  - 请求作用域的临时连接存放在 `RequestScopedMCPConnectionStore`（`api/server/services/MCPRequestContext.js`）中，键为 `${userId}:${server}`。
  - `callTool` 会在 OAuth 和 OBO 恢复过程中重试。
- **`tools.ts`**：`formatMCPServerTools` 把 MCP 的 `Tool[]` 转换为 OpenAI 风格的函数映射。键为 `<strippedTool>_mcp_<normalizedServer>`，`serverToolName` 保留上游原始名称。`createMCPToolCacheService` 实现工具缓存。
- **`parsers.ts`（`formatToolContent`）**：把 MCP `CallToolResult` 的内容转换为 `[text, artifacts]`。图片变为 artifact；资源 blob 被解码或摘要。
- **`toolsChanged.ts`**：收到 `list_changed` 时，重新获取列表，并通过 `initializeMCPs.js` 中的 `refreshChangedServerTools` 覆盖缓存条目。它使用一个 generation/revision 栅栏（fence）。
- **`catalog/`**：有界的后台目录恢复，带退避（`mcpSettings.catalogRecovery`）。
- **`authority/`**：用于权威证明（authority proof）的底层设施，默认关闭。
- **`oauth/`**：
  - `handler.ts`：发现、DCR、PKCE、令牌交换、刷新、吊销。
  - `tokens.ts`：`MCPTokenStorage`。
  - `OAuthReconnectionManager.ts`：登录时，重连该用户已存有令牌的服务器，每个间隔 500 ms 错开。
  - `obo.ts`：Entra On-Behalf-Of jwt-bearer 交换。
  - `detectOAuth.ts`：探测 401/`WWW-Authenticate` 以及受保护资源元数据。
- **应用胶水代码**：
  - `api/server/services/MCP.js`：`createMCPTool(s)`、`reconnectServer`、权限上下文、冲突审计、OAuth 运行步骤发送器。
  - `api/server/services/initializeMCPs.js`：启动时创建注册表和管理器，合并插件服务器，并注册 toolsChanged 处理器。
  - 路由：`api/server/routes/mcp.js`，控制器在 `api/server/controllers/mcp.js`。

### 连接作用域（`mcp/utils.ts`）
- **`canUseAppConnection(config)`** 决定一个由运维方拥有的连接能否服务所有用户。它要求同时满足：
  - `startup !== false`；
  - 不是用户来源（来自数据库），也不是插件来源；
  - 不是 `requiresUserScopedConnection`，即没有 `requiresOAuth`、没有 `obo`、没有 `customUserVars`，也没有运行时占位符（`{{LIBRECHAT_USER_*}}`、`{{LIBRECHAT_OPENID_*}}`、`{{LIBRECHAT_BODY_*}}`）；
  - 没有 `requestHeaders`。
- 其他情况都使用 **每用户连接**。
- 带有 `{{LIBRECHAT_BODY_conversationId|parentMessageId|messageId}}` 占位符的配置是 **请求作用域（临时）** 的。它们在请求结束时被拆除，永远不会转入后台。

### 占位符处理（`packages/api/src/utils/env.ts processMCPEnv`）
处理顺序：
1. 对管理员模板做 `${ENV}` 展开。这一步最先执行，以防止二阶注入。
2. `{{customUserVar}}`。
3. `{{LIBRECHAT_USER_<FIELD>}}`，仅限白名单中的用户字段。
4. `{{LIBRECHAT_OPENID_*}}`（令牌）。令牌过期时抛出 `OpenIDReauthRequiredError`。
5. `{{LIBRECHAT_BODY_*}}`。

例外情况：
- 数据库（用户）服务器 **只** 解析 customUserVars。
- 插件来源的配置原样返回。
- 管理员 `apiKey` 会转成一个 header。

`{{LIBRECHAT_GRAPH_ACCESS_TOKEN}}` 通过 OBO 单独解析。

### 配置字段（`packages/data-provider/src/mcp.ts`）
- **基础选项：** `title, description, startup, iconPath, timeout, initTimeout, sseReadTimeout, chatMenu, serverInstructions (bool|string), requiresOAuth, oauth, oauth_headers, apiKey{key, source: admin|user, authorization_type: basic|bearer|custom, custom_header}, customUserVars{name:{title, description, sensitive}}, oauthRefreshWaitTimeout, oauthRefreshCoordination, oauthPersistenceWaitTimeout`。
- **`oauth`：** `authorization_url, token_url, client_id, client_secret, scope, redirect_uri, token_exchange_method, grant_types_supported, token_endpoint_auth_methods_supported, code_challenge_methods_supported, skip_code_challenge_check, audience, forward_audience_on_refresh, send_resource_parameter, revocation_endpoint...`。
- **传输相关：**
  - `stdio{command, args, env, stderr, cwd}`
  - `websocket{url}`
  - `sse` / `streamable-http`/`http` `{url, headers, requestHeaders, obo{scopes}, proxy}`
- **全局：** `mcpSettings{allowedDomains, allowedAddresses, catalogRecovery{...}}`。
- **环境变量：** `MCP_OAUTH_HANDLING_TIMEOUT`（默认 10 分钟）、`MCP_OAUTH_FLOW_TTL`（15 分钟）、`MCP_CONNECTION_CHECK_TTL`、`MCP_TOOLS_LIST_*`、`MCP_CB_*`（`mcp/mcpConfig.ts`）。

数据库中的 `mcpServer` schema 为 `{serverName, normalizedServerName (unique per tenant), config: Mixed, author, tenantId}`。CRUD 接口位于 `/api/mcp/servers`，通过 ACL 共享。

### OAuth 时序（MCP）
1. 一次连接尝试收到 401（或配置了 `requiresOAuth`）。运行 `MCPConnectionFactory.handleOAuthRequired`。
2. 它在 **`FlowStateManager`**（`packages/api/src/flow/manager.ts`）中查找已有的流程。该管理器以 Redis 上或内存中的 Keyv 为后端，流程类型为 `mcp_oauth`，flowId 为 `userId:serverName`，有租户时加前缀 `tenant:<t>:`。
   - 若存在新鲜的 PENDING 流程，则加入它，并重新发送其存储的认证 URL。
   - 若存在新鲜的 COMPLETED 流程，则复用其令牌。
   - 否则删除旧流程。
3. `MCPOAuthHandler.initiateOAuthFlow`。如果域名允许列表拒绝该服务器，这一步会失败。
   - 如果预先配置了 `client_id` + `authorization_url` + `token_url`，就使用这些值，只有令牌端点匹配时才去发现能力。
   - 否则自动发现：先是 RFC 9728 受保护资源元数据（其中 `resource` 必须与服务器 URL 匹配），然后是 RFC 8414 授权服务器元数据，再是动态客户端注册（Dynamic Client Registration）。已存储的客户端可以复用。
   - PKCE 使用 `startAuthorization`。它会附加 `resource`（RFC 8707）或 `audience`，以及一个随机的 32 字节 `state`。
   - 重定向 URI 为 `${DOMAIN_SERVER}/api/mcp/<server>/oauth/callback`。
4. 存储流程：`flowManager.initFlow(flowId,'mcp_oauth',{...metadata, codeVerifier, clientInfo, authorizationUrl})`，外加一个 state→flowId 的映射。
5. `oauthStart(authURL)` 向 SSE 流 / GenerationJobManager 发出一个带 `auth` 和 `expires_at` 的工具调用运行步骤。随后服务器通过 `waitForSharedOAuthFlow` **等待**：轮询流程状态，直到 COMPLETED/FAILED 或超时。
6. 浏览器访问 `GET /api/mcp/:server/oauth/initiate?userId&flowId`，该接口设置 CSRF cookie 并重定向。随后 IdP 重定向到 `GET /api/mcp/:server/oauth/callback?code&state`。
7. 回调处理：
   - 解析 state → flowId；
   - 校验 CSRF cookie、会话 cookie，或存在活跃的 PENDING 流程；
   - 检查 state 与当前这次尝试匹配；
   - 调用 `completeOAuthFlow`（用 code + verifier 交换令牌）；
   - 调用 `MCPTokenStorage.storeTokens`；
   - 调用 `flowManager.completeFlow`；
   - 重连该用户连接；
   - 重定向到 `/oauth/success`。
8. **令牌存储。** 令牌存入 `token` 集合：`{userId, type, identifier, token (encryptV2), expiresAt, metadata}`，带 TTL 索引。共三条记录：
   - `type:'mcp_oauth', identifier:'mcp:<server>'`：访问令牌；
   - `mcp_oauth_refresh` / `mcp:<server>:refresh`；
   - `mcp_oauth_client` / `mcp:<server>:client`：DCR 客户端信息。
9. **刷新。** 连接前会静默刷新令牌。刷新通过流程租约在副本之间协调（`refreshOAuthTokens`，“refresh flights”）。

其他路由：
- `GET /oauth/tokens/:flowId`
- `GET /oauth/status/:flowId`
- `POST /oauth/cancel/:server`
- `POST /:server/oauth/bind`
- `POST /:server/reinitialize`
- `GET /connection/status[/:server]`
- `GET /:server/auth-values`（哪些 customUserVars 已设置）
- `GET /api/mcp/tools`

---

## 4. Actions（OpenAPI → 工具）

- **Schema**（`packages/data-schemas/src/schema/action.ts`）：`{user, action_id, type, agent_id|assistant_id, metadata{domain, raw_spec, privacy_policy_url, api_key*, oauth_client_id*, oauth_client_secret*, auth{type: service_http|oauth|none, authorization_type (basic|bearer|custom), custom_auth_header, authorization_url, client_url (token URL), scope, token_exchange_method: default_post|basic_auth_header}}, tenantId}`。标有 `*` 的字段用 `encryptV2(encodeURIComponent(v))` 加密（`packages/api/src/actions/crypto.ts`）。`agent.actions[]` 保存 `"<domain>_action_<action_id>"`，`agent.tools` 保存函数名。
- **创建/更新：** `POST /api/agents/actions/:agent_id`（`api/server/routes/agents/actions.js`）。请求体为 `{functions, action_id?, metadata}`。处理器依次：
  1. 运行内容过滤检查；
  2. 加密元数据；
  3. 运行 `validateAndParseOpenAPISpec(raw_spec)`；
  4. 调用 **`validateActionDomain(metadata.domain, spec.servers[0].url)`**，这是一个防 SSRF 检查，确认客户端提供的域名与规范中的服务器一致；
  5. 调用 `isActionDomainAllowed(domain, actions.allowedDomains, actions.allowedAddresses)`；
  6. 对域名编码；
  7. 运行 `planAgentActionUpdate`（`packages/api/src/actions/update.ts`），它合并工具和 actions，并在目标变化时删除 OAuth 令牌；
  8. 调用 `validateActionOAuthMetadata`；
  9. 用 `forceVersion` 更新智能体，然后 upsert 该 Action。响应中会剔除密钥。

  `DELETE /api/agents/actions/:agent_id/:action_id` 用于删除。
- **规范 → 函数**（`openapiToFunction`，`packages/data-provider/src/actions.ts:455`）：
  - 每个 路径×方法 生成一个函数。名称取 `operationId`，或经过清洗的 `method_path`；描述取 `summary || description`。
  - 参数（query/path/header）与 requestBody 的顶层属性合并为一个扁平的对象 schema。`paramLocations` 记录每个字段应放在哪里，并生成一个 Zod schema。
  - 每个函数得到一个 `ActionRequest(baseUrl, path, method, operationId, isConsequential, contentType, paramLocations)`。
  - 支持 `x-strict` 和 `x-openai-isConsequential`。
- **执行**（`ActionService.createActionTool`，`api/server/services/ActionService.js:185`）：一个 LangChain `tool()`，其 `_call` 执行以下操作。
  - `executor.setParams(input)`。
  - 认证：
    - **API 密钥：** `Authorization: Basic base64(key)`、`Bearer key`，或自定义 header。
    - **OAuth：** 查找 `token{type:'oauth', identifier:'userId:action_id'}` 和 `oauth_refresh`。
      - 如果没有令牌，`requestLogin` 构建认证 URL（`client_id, scope, redirect_uri=${DOMAIN_CLIENT}/api/actions/:id/oauth/callback, response_type=code, access_type=offline, state=JWT{nonce,user,action_id}`，用 JWT_SECRET 签名，有效期 10 分钟）。它发出一个带 `auth` 和 `expires_at`（2 分钟）的运行步骤 delta，然后在流程类型 `oauth` 上等待。
      - `GET /api/actions/:action_id/oauth/callback` 校验 state JWT，检查 CSRF/会话，交换 code（`getAccessToken`），存储令牌并完成流程。
      - 如果存在刷新令牌，由一个共享的 `oauth_refresh` 流程刷新访问令牌。
  - `executor.execute(ssrfAgents)`：一次 axios 调用。当 `allowedDomains` 为空时使用 SSRF 安全的 agent。
  - 对象类型的响应会被 JSON 字符串化。
- 运行时每次加载都会重新检查域名允许列表、规范有效性和域名匹配。旧版域名编码也会被注册，因此较早的智能体仍能解析。OAuth actions 不参与后台分派（`oauthActionToolNames`）。

---

## 5. 技能

- **数据模型：**
  - `skill`（`packages/data-schemas/src/schema/skill.ts`）：`name`（kebab-case，≤64，禁止保留前缀/保留词）、`displayTitle, description (≤1024), body (≤100k, the SKILL.md content), frontmatter (Mixed), disableModelInvocation, userInvocable, allowedTools[], category, author, authorName, version (monotonic, bumped on every edit/file change), source: inline|github|notion, sourceMetadata, fileCount, alwaysApply, tenantId`。
  - `skillFile`：`{skillId, relativePath (safe relative, not SKILL.md), file_id, filename, filepath, storageKey, source, mimeType, bytes, category: script|reference|asset|other, isExecutable, content?, isBinary?, codeEnvRef, codeEnvRefs}`。
  - `codeEnvRef`（`schema/codeEnvRef.ts`）：`{kind: skill|agent|user, id, storage_session_id, file_id, version, executionProfile, executionRouteKey, provisionedAt, sandboxFilename}`。它缓存文件在 Code API 中的上传位置；条目在 23 小时内视为新鲜。
  - `skillSyncCredential`（加密的 GitHub 令牌）和 `skillSyncStatus`（每次同步运行的状态）。
- **来源：**
  - 内联（UI/REST）。
  - 导入 `.md`/`.zip`/`.skill`（`skills/import.ts`）。
  - 部署目录 `DEPLOYMENT_SKILLS_DIR`（默认 `skill/`），由内存注册表提供（`skills/deployment.ts`）。
  - Agent Plugins 包：`plugin.json` + `skills/` + `mcp.json` + `hooks`（`packages/api/src/plugins/*`）。
  - GitHub 镜像（`skills/sync/github.ts`、`orchestrator.ts`、`scheduler.ts`），由管理员触发运行，或在 `GET /api/skills` 时顺带触发，限流为每 5 分钟一次。
- **REST：**
  - `/api/skills`（`api/server/routes/skills.js`）：列表、创建、按 `:id` 获取/修改/删除，`/:id/files` 列表与上传，`/:id/files/*relativePath` 下载与删除，`/import`。访问受 ACL 保护（VIEW/EDIT/DELETE 位）。
  - `/api/admin/skills`：同步凭据与同步运行。
  - 一个机器认证的管理 API，位于 `/api/agents/v1/skills`（`docs/skills-management-api.md`）。它通过 `expectedVersion` 实现乐观并发，冲突时返回 409。
  - 每用户的激活覆盖存放在 `skills/skillStates.ts`。
- **运行时**（`packages/api/src/agents/skills.ts`、`skillFiles.ts`）：
  - `injectSkillCatalog` 列出可访问且处于激活状态的技能。默认目录中最多出现 100 个，描述会被截断。注入目录文本，并注册 `skill` 工具加 `read_file`（启用代码时还有 `bash_tool`）。
  - **`skill` 工具调用**（`handleSkillToolCall`，`handlers.ts:5051`）在 `accessibleSkillIds` 范围内按名称解析技能。它拒绝 `disableModelInvocation` 技能，把技能正文作为标记为 `source:'skill'` 的注入消息返回，并把随附文件预置到代码沙箱的 `/mnt/data/skills/<name>/...`（`primeSkillFiles`，按技能版本单飞（single-flight），只读批量上传）。
  - **手动**（UI 中的 `$skill`）和 **始终应用**（always-apply）的技能在轮次开始时以 HumanMessage 形式预置。上限：手动 10 个、始终应用 20 个、合计 30 个。
  - 已预置技能的 `allowedTools` 会并入工具集合。
  - 模型可以在运行中编写技能：在 `skills/<name>/SKILL.md` 处 `create_file`，即可让该技能在同一次运行中可被调用。

---

## 6. 代码执行与环境

- **执行配置（profile）**（`packages/api/src/agents/execution.ts`）：
  - **`default`**：无状态的托管 Code API。基础 URL 来自 SDK 的 `getCodeBaseURL()`，环境变量为 `LIBRECHAT_CODE_BASEURL`；SDK 的默认值在此无法核实。
  - **`stateful`**：持久会话。URL 为 `LIBRECHAT_CODE_BASEURL_STATEFUL` 或所配置环境的 `baseURL`。作用域为 `user | agent-user | conversation`（`STATEFUL_CODE_ENVIRONMENTS`）。`runtimeSessionHint` 是作用域的哈希指纹，`codeSessionKey = execute_code:<routeKey>:<hint>`。
- **环境配置：** `endpoints.agents.statefulCodeSessions{allowedEnvironments, principalWorkers{enabled, maxPerUser}, conversationMoves, environments[{id, name, type: managed|attached, baseURL, default, workerId, owner, configSchema, settings, pairing{workerId, allowPrincipalWorkers, tokenEnv}}]}`。
  - 数据库的 `codeEnvironment` 集合保存用户注册的（“主体”）附加环境：`{environmentId, name, type, baseURL, controlPlaneId, createdBy, workerId, workerPrincipal, settings{permissions{fileWrite, commandExecution: allow|ask|deny}}, plus lease/revocation fields}`。
  - 路由：`/api/code-environments`（列表、配对、创建、设置、删除）和 `/api/admin/code-environments/:id/pairings|revoke`。
  - `code/lifecycle.ts` 在后台对注册租约和吊销租约进行对账。
- **附加环境**（CONTEXT.md）：用户自己机器上的 `librechat-code` 工作进程主动向外连接 Code API 桥。每个对话保存一个不可变的工作区决定（`codeWorkspaces`）。工作区操作：`read_file, search_text, list_files, execute_command, write_file, edit_file, preview_edit`。
- **访问 Code API 的认证**（`packages/api/src/auth/codeapi.ts`）：
  - 要么是静态 API 密钥（通过 `loadAuthValues` 取得的 `CODE_API_KEY` 凭据），要么在开启 `CODEAPI_JWT_ENABLED`/托管模式时，使用签发的 RS256/ES JWT。
  - JWT 声明：`iss, aud, sub=userId, tenant_id, role, principal_source, org_id, service_id, plan_id, code_worker_id, auth_context_hash, jti, exp`。令牌会被缓存。
  - 额外的 header：`X-CodeAPI-Expected-Profile: default|stateful`、`X-LibreChat-Code-Worker-ID`、`User-Id`、`User-Agent: LibreChat/1.0`。
- **外部 Code API 契约**，根据 `api/server/services/Files/Code/*` 和 `packages/api/src/code/*` 中的调用整理：

  | 端点 | 请求 | 响应 / 备注 |
  |---|---|---|
  | `POST {base}/exec` | JSON `{lang, code, args?, session_id?, runtime_session_hint?, files?: [{id, session_id, name}]}` | `{stdout, stderr, session_id, files:[{id, name}]}`。输出文件作为工具 artifact 返回。 |
  | `POST {base}/upload` | multipart：file、`kind`、`id`、`version` | `{message:'success', storage_session_id, files:[{fileId, filename}]}` |
  | `POST {base}/upload/batch` | 同上，另加 `read_only` | |
  | `GET {base}/download/{session_id}/{fileId}` | | 文件流 |
  | `DELETE {base}/files/{session_id}/{fileId}` | | |
  | 文件信息 / lastModified | | 用于新鲜度检查 |
  | `POST {base}/workspace-tools/execute` | `{protocolVersion:1, operation, workspaceId, workspaceInstanceId, ...}` | 有界 JSON；遇到准入超时会重试，并遵守 `Retry-After` |
  | `POST {base}/bridge/pairings` | Bearer 配对令牌，`{workerId, binding}` | |
  | `GET /bridge/workers/:id/status` | | |
  | `POST /bridge/workers/:id/revoke` | | |

  SDK 的 `execute_code` 工具输入 schema 为 `{lang, code, args}`。宿主的 `read_file` 通过 `/exec` 执行 `cat`。`bash_tool` 发送 bash 代码。
- **文件预置**（`Files/Code/process.js primeFiles`）：
  - `tool_resources.execute_code` 中的智能体文件和用户文件会被上传，或通过新鲜的 `codeEnvRef` 复用。
  - 模型看到的是一个 `/mnt/data/<file>` 路径列表。
  - `primedCodeFiles` 为图的代码会话提供初始数据，使第一次调用就能看到之前的 artifact。
  - 输出文件被下载，经过内容过滤器，通过文件存储策略保存（`processCodeOutput`），并作为附件流式发送。

---

## 7. 网页搜索管线

- **配置**（yaml 中的 `webSearch`；`webSearchSchema`，`packages/data-provider/src/config.ts:2529`）：
  - **搜索提供商：** `serper | searxng | tavily | keenable`。
  - **抓取器：** `firecrawl | serper | tavily | keenable`。
  - **重排序器：** `jina | cohere | none`。
  - 密钥和 URL 默认取 `${SERPER_API_KEY}`、`${SEARXNG_INSTANCE_URL}`、`${FIRECRAWL_API_KEY}`/`URL`/`VERSION`、`${JINA_API_KEY}`/`URL`、`${COHERE_API_KEY}`、`${TAVILY_*}`、`${KEENABLE_*}`。
  - 其他设置：`scraperTimeout`、`safeSearch`（0/1/2）、`firecrawlOptions`、`searxngSearchOptions`、`allowedAddresses`。
- **认证解析**（`packages/api/src/web/web.ts loadWebSearchAuth`）。对每个类别（提供商、抓取器、重排序器）：
  - 选出第一个所需字段都能解析的提供商，字段来自环境变量或用户的 `pluginAuth`（设为 `user_provided` 的值需要用户提供）。用户可以自行选择提供商。
  - 用户提供的 URL 要经过 SSRF 预检。
  - 管理员密钥永远不会转发到用户提供的 URL。
  - `authenticated=false` 时整个工具被丢弃。
- **SSRF agent**（`web/agent.ts`）是在连接时检查 IP 的 http/https agent，对已配置的代理有豁免。
- **执行：** SDK 的 `createSearchTool` 依次执行：
  1. 搜索（organic、topStories、images、news）；
  2. 抓取排名靠前的链接；
  3. 对高亮片段分块并重排序；
  4. 返回带 `turn` 索引的格式化来源。
- **宿主回调**（`api/server/services/Tools/search.js`）：`onSearchResults` 构建一个 `web_search` 附件 `{turn, organic, topStories, ...}`，以 `event: attachment` 流式发送；`onGetHighlights` 在高亮片段到达时更新它。
- **提示上下文**（`tools/toolkits/web.ts`）：引用语法使用私用区 unicode 锚点（`turn{N}search{i}`、分组 `…`、高亮 `…`），外加一个动态日期上下文。
- REST：`GET /api/agents/tools/web_search/auth` 校验网页搜索认证。密钥通过 `POST /api/user/plugins`（`pluginKey: web_search`）安装。

---

## 8. 记忆

- **Schema** `memoryEntries`（`schema/memory.ts`）：`{userId, key (/^[a-z_]+$/), value, agentId? (partition; null means the shared personal pool), tokenCount, updated_at, tenantId}`，索引为 `(userId, agentId, key)`。
- **配置** `memory`（`memorySchema`）：
  - `disabled, validKeys[], tokenLimit, charLimit (10000), maxInputTokens, personalize, messageWindowSize (5)`；
  - `agent: {enabled, id} | {enabled, provider, model, instructions, model_parameters}`。
  - 自动提取需要通过 `agent.enabled: true` 显式开启（`packages/data-schemas/src/app/memory.ts`）。
- **注入**（`AgentClient.useMemory`，`api/server/controllers/agents/client.js:3214`）：
  - 检查 MEMORIES USE 权限和 `personalize`。
  - `getFormattedMemories` 返回 `withKeys`/`withoutKeys` 字符串及 token 计数。
  - `withoutKeys` 文本以 `"# Existing memory about the user:\n..."` 的形式放入系统提示，前面是 `memoryInstructions`。
  - 一个按请求划分的 `WeakMap` 缓存让多个智能体共享同一次读取（`getRequestMemories`）。
- **提取（后台记忆智能体）：**
  - `createMemoryProcessor` → `processMemory`（`packages/api/src/agents/memory.ts:745`）。响应完成后，`runMemory(messages)` 取最后 `messageWindowSize` 条聊天消息（去掉技能预置消息），并截断到 `maxInputTokens`。
  - 它构建一个独立的 `Run`，系统指令为记忆指令加上 `# Existing memory:` 和带键的记忆。回退模型为 `openai/gpt-4.1-mini`；对 Bedrock，指令改为放在用户消息中。
  - 该运行只有两个工具：`set_memory({key, value})` 和 `delete_memory({key})`，它们强制校验有效键和 token 上限，并通过 `setMemory`/`deleteMemory` 写入。
  - 结果以 `memory` artifact/附件的形式流式发送。
- **内联记忆工具：** 当智能体包含 `memory` 工具（`memory` 能力）时，`set_memory`/`delete_memory` 会直接注册到该智能体上（`registerMemoryTools`、`buildInlineMemoryTool`），并按 `agentId` 分区。提示中的使用约束限定它们只用于明确的“记住/忘记”请求。
- **REST** `/api/memories`（`routes/memories.js`）：GET（列表，含用量）、POST（创建）、`PATCH /preferences`（退出）、`PATCH/DELETE /:key`，通过 `?agent_id` 按分区限定（`memory/authorization.ts`），全部位于 PII/内容过滤器之后（`memory/protection.ts`）。

---

## 9. Artifacts 与 OpenAPI 导出

- **Artifacts** *不是* 工具。它们以内联方式生成在模型文本中，位于 `:::artifact{identifier,type,title}` 和 `:::` 之间。
  - 提示来自 `api/app/clients/prompts/artifacts.js generateArtifactsPrompt({endpoint, artifacts: default|shadcnui|custom})`，由 `artifacts` 能力控制。
  - `packages/api/src/artifacts/update.ts` 提供 `findAllArtifacts`（在 `message.content[].text` 或 `message.text` 上做能识别代码围栏的边界解析）和 `replaceArtifactContent`。
  - `POST /api/messages/artifact/:messageId {index, original, updated}` 就地编辑单个 artifact 并重新保存消息。
- **OpenAPI 导出：** `api/server/routes/openapi.js` 把 `createOpenApiRouter`（`packages/api/src/openapi/router.ts`）挂载到 `/api`。
  - 仅当 `openapi.enabled: true` 时，它提供 `GET /api/openapi.json`（预生成的 `agents.openapi.json`）以及位于 `/api/docs` 的 Swagger UI。
  - 规范由 Zod 契约生成（`openapi/agents.ts`、`skills.ts`、`registry.ts`、`adapter.ts`、`generate.ts`），使用 `oidcBearer` 安全方案，覆盖智能体和技能的管理 API。

---

## 10. Python 重新实现映射

| 关注点 | Node 实现 | Python 对应方案 |
|---|---|---|
| 智能体图 / 工具循环 | `@librechat/agents`（LangGraph JS），`ON_TOOL_EXECUTE` 批次 | `langgraph`（`StateGraph`、`ToolNode` 或自定义的异步工具执行节点），`langchain_core.tools.StructuredTool` / `BaseTool`，配合 `response_format="content_and_artifact"` |
| 工具注册表 / 延迟工具 | `LCToolRegistry`、`createToolSearch` | 一个 名称 → `{schema, allowed_callers, defer_loading}` 的 dict，加一个本地 BM25/正则 `tool_search` 工具，返回 schema 供模型在下一轮绑定 |
| JSON-schema → 工具参数 | zod、`normalizeJsonSchema`、`sanitizeGeminiSchema` | `pydantic.create_model` 或 `jsonschema`；`langchain_core.utils.function_calling.convert_to_openai_tool`；用 `jsonref` 解析 `$ref` |
| MCP 客户端与传输 | `@modelcontextprotocol/sdk` | 官方 `mcp` SDK：`mcp.ClientSession`；`mcp.client.stdio.stdio_client(StdioServerParameters)`、`mcp.client.sse.sse_client`、`mcp.client.streamable_http.streamablehttp_client`、`mcp.client.websocket.websocket_client`；`session.list_tools()`（游标分页）、`session.call_tool()`，以及 `tools/list_changed` 的通知处理器 |
| MCP OAuth | 自定义 `MCPOAuthHandler` 加 SDK 认证辅助函数 | `mcp.client.auth.OAuthClientProvider`（PKCE、DCR、RFC 9728/8414 发现），配合以 Mongo 为后端的自定义 `TokenStorage`；或 `authlib` / `httpx-oauth` |
| MCP → LangChain 工具 | `services/MCP.js createToolInstance` | 以 `langchain-mcp-adapters`（`load_mcp_tools`、`MultiServerMCPClient`）为基础，或写一个薄的 `StructuredTool` 包装，调用池化的 `ClientSession` |
| 连接池 / 空闲驱逐 | `MCPManager`、`ConnectionsRepository` | 一个 asyncio 管理器：`dict[(scope, user, server)] → session`，由 `contextlib.AsyncExitStack` 持有，每个键一把 `asyncio.Lock`，一个后台任务负责空闲清扫，用 `tenacity` 做退避和熔断 |
| FlowStateManager（OAuth 等待） | Keyv + Redis 轮询 | `redis.asyncio`（`SET NX PX`、pub/sub 或轮询），状态为 PENDING/COMPLETED/FAILED，配合 TTL 租约 |
| OpenAPI actions | `openapiToFunction`、`ActionRequest` | 用 `openapi-core` 或 `prance` 解析和校验，配合 `jsonref`；为每个操作构建 pydantic 模型；用 `httpx.AsyncClient` 执行（自定义传输层做 SSRF IP 检查） |
| 加密 | AES-CBC `encryptV2`（`hex iv:hex ct`，`CREDS_KEY`） | `cryptography.hazmat` AES-CBC + PKCS7，采用相同格式以保证数据兼容 |
| JWT（action state、Code API） | `jsonwebtoken`、自定义签名器 | `PyJWT`（state 用 HS256，Code API 令牌用 RS256/ES256） |
| Code API 客户端 | axios/fetch | `httpx.AsyncClient`（通过 `files=` 发送 multipart），流式下载 |
| 网页搜索 | SDK `createSearchTool` | 直接用 `httpx` 调用 Serper/SearXNG/Tavily；Firecrawl（`firecrawl-py`）；通过 Jina HTTP 或 `cohere` SDK 重排序；用 `tiktoken` 做分块预算 |
| SSRF 防护 | `createSSRFSafeAgents`、`resolveHostnameSSRF` | 在 `socket.getaddrinfo` / `anyio` DNS 之后做 `ipaddress` 检查，在自定义的 `httpx.AsyncHTTPTransport` 中强制执行 |
| 记忆智能体 | 带 set/delete 工具的独立 `Run` | 第二次 LangGraph/LLM 调用，带两个 `StructuredTool`，在后台运行（`asyncio.create_task` / Celery / arq） |
| Schema / 数据库 | Mongoose | `motor`/`beanie` 或 `pydantic` + `pymongo`，保持相同的集合与索引（`pluginAuth`、带 TTL 的 `token`、`action`、`mcpServer`、`skill`、`skillFile`、`memoryEntries`、`codeEnvironment`） |
| 配置校验 | zod（`librechat.yaml`） | `pydantic` v2 模型，对 MCP 传输在 `type` 上使用可辨识联合；`${VAR}` 环境变量展开在用户占位符之前完成 |
| OpenAPI 导出 | zod-to-openapi + Swagger UI | FastAPI 内置的 OpenAPI 与 Swagger（`/openapi.json`、`/docs`） |

**需要保持的关键不变量：**
- 应用级、每用户、请求作用域三种 MCP 连接，由 `canUseAppConnection` / `requiresUserScopedConnection` 决定。
- 占位符顺序：先环境变量，再用户变量。数据库服务器只解析 customUserVars；插件配置永不解析。
- 每次对外连接前都检查域名与 SSRF 允许列表，适用于 actions、MCP 和网页搜索。
- action 的域名必须与规范中的服务器 URL 一致。
- 工具名经过规范化，冲突时按失败关闭处理。
- 工具实例在每次调用时惰性创建，凭据在调用时解析。
- OAuth 是一个阻塞的“流程”，聊天流把它呈现为一个带认证 URL 的合成工具调用步骤。
