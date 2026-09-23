# Tools, MCP, Actions, Skills, Code Execution, Web Search & Memory

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

Two packages matter here:
- **`@librechat/agents`** is the LangGraph agent SDK. It is not vendored in this repository. It supplies `createCodeExecutionTool`, `createSearchTool`, `createToolSearch`, `createBashExecutionTool`, `Calculator` and the `ON_TOOL_EXECUTE` event loop. Its contracts are described from how LibreChat calls it.
- **`librechat-data-provider`** is the shared schemas and constants package, at `packages/data-provider/src`.

---

## 1. Tool resolution pipeline (step by step)

Agents run in **"event-driven" (definitions-only) mode**. At initialization the backend works out only the JSON-schema *definitions* of each tool. It creates real executable instances lazily, when the model actually emits tool calls.

### 1a. Initialization phase: `loadAgentTools({definitionsOnly: true})` → `loadToolDefinitionsWrapper`
File: `api/server/services/ToolService.js:800-1575`. It is called from `initializeAgent` in `packages/api/src/agents/initialize.ts`, where it is injected as `loadTools`.

1. **Short-circuits.** Return nothing if `agent.tools` is empty, or holds only the legacy `context`/`ocr` entry.
2. **Resolve capabilities** (`resolveAgentCapabilities`, line 322). The set comes from `endpoints.agents.capabilities`; ephemeral agents fall back to the defaults. The `AgentCapabilities` enum (`packages/data-provider/src/config.ts:644`) is: `execute_code, stateful_code_sessions, file_search, web_search, artifacts, subagents, actions, context, skills, memory, ask_user_question, tools, chain, ocr, run_in_background, tool_intents, deferred_tools, programmatic_tools, end_after_tools, hide_sequential_outputs`.
3. **Role permissions** (`resolveAgentToolPermissions`, backed by `packages/api/src/tools/rolePermissions.ts`). `file_search`→FILE_SEARCH, `execute_code`→RUN_CODE, `web_search`→WEB_SEARCH, checked against the user's role.
4. **Filter `agent.tools`** (lines 869-895). Each tool is gated as follows:

   | Tool | Condition to keep it |
   |---|---|
   | `file_search`, `execute_code`, `web_search` | its capability **and** the role permission |
   | `memory` | `memory` capability |
   | `ask_user_question` | `ask_user_question` capability |
   | action tools (`isActionTool` = name contains `_action_`) | `actions` capability |
   | MCP tools (contain `_mcp_`) | `tools` capability **and** MCP USE permission |
   | everything else | `tools` capability |

5. **Resolve the code-execution context** (`resolveCodeExecutionContext` plus `resolveCodeExecutionWorkspaceContext`). See §6.
6. **Content-filter preflight.**
   - `assertToolResourcesAllowed` checks the agent's attached files.
   - `prepareActionSnapshotForTools` loads the agent's Action documents from Mongo once. The snapshot is reused to avoid TOCTOU drift, and PII filters inspect it before any MCP connection is made.
7. **MCP context.**
   - `resolveMcpServerContext(req)` returns `configServers` (admin overrides), `serverNames` (normalized) and `rawServerNames`. It lives in `api/server/services/MCP.js:208`.
   - Collision audit: `resolveCollisionAuditNames`, `findShadowedServerNames`, `buildServerNameAliases`. It drops tools whose normalized server name collides with another server's; it fails closed when the audit is incomplete.
   - `getUserMCPAuthMap` loads per-user `customUserVars` from the `pluginAuth` collection. Keys look like `mcp_<serverName>`.
8. **`loadToolDefinitions(params, deps)`** (`packages/api/src/tools/definitions.ts`) sorts each tool name into one of three paths:
   - **Built-in:** `isBuiltInTool`, then `getToolDefinition(name)` from the static registry `packages/api/src/tools/registry/definitions.ts`, which holds JSON schemas for google, dalle, flux, open_weather, wolfram, etc. Toolkits expand here: `image_gen_oai` also brings in `image_edit_oai` (`tools/toolkits/mapping.ts`).
   - **MCP:** the key is `<tool>_mcp_<normalizedServer>`, or `sys__all__sys_mcp_<server>`, which means "all tools of this server".
     - `deps.getOrFetchMCPServerTools` checks the tool cache first (`getMCPServerTools`). On a miss it runs `reinitMCPServer`: it connects, lists tools, and possibly starts OAuth.
     - If the selected tool is missing from a stale catalog, `refreshMCPServerTools` forces a new connection.
     - Schemas are normalized with `resolveJsonSchemaRefs` + `normalizeJsonSchema`. For Gemini/Vertex providers they are also flattened by `sanitizeGeminiSchema`.
   - **Actions:** `deps.getActionToolDefinitions` re-parses each stored OpenAPI spec with `openapiToFunction` and emits the signatures whose names match the agent's tool list.
9. **Build the registry.** `buildToolClassification` / `buildToolRegistryFromAgentOptions` (`packages/api/src/tools/classification.ts`) produce an `LCToolRegistry`, a `Map<name, {name, description, parameters, allowed_callers, defer_loading, toolType, serverName}>`. Per-tool `agent.tool_options[name] = {defer_loading, allowed_callers: ['direct'|'code_execution']}` drives two features:
   - **Deferred tools / tool discovery.** When `deferred_tools` is enabled and any tool has `defer_loading`, a local `tool_search` tool is added (`createToolSearch({mode:'local', toolRegistry})`). Deferred tools are hidden from the model until it finds them through search.
   - **Programmatic Tool Calling (PTC).** When `programmatic_tools` and `execute_code` are enabled and some tool allows `code_execution` callers, a `bash_programmatic_tool_calling` tool is added (`createContextProgrammaticBashTool`, `packages/api/src/code/command.ts`). Sandboxed code can then call the other tools. `packages/api/src/agents/ptc.ts` wraps the PTC tool map in a Proxy so inner calls stream progress (`instrumentPtcToolMap`).
10. **MCP OAuth waits.**
    - Any server that needs OAuth (`pendingOAuthServers`) emits a synthetic run step to the client: `buildMCPAuthRunStepEvent` plus a delta event carrying the `authURL`.
    - The call then blocks, up to 2 minutes, in `reinitMCPServer({returnOnOAuth:false, oauthStart, oauthEnd})`.
    - On success it calls `loadToolDefinitions` again.
    - If an agent expects MCP tools and none resolve, it throws `ExpectedMCPToolsUnavailableError`.
11. **Context strings.**
    - `toolContextMap` holds static instructions such as the web-search citation format.
    - `dynamicToolContextMap` holds per-turn context: the web-search date, and the file lists produced by `primeCodeFiles` / `primeSearchFiles`, which are also uploaded to the Code API.
12. **Remaining registrations in `initializeAgent`** (`packages/api/src/agents/initialize.ts`):
    - Memory tools: `registerMemoryTools` (line 2110).
    - Code tools: `registerCodeExecutionTools` → `bash_tool`, `read_file`, `search_workspace`, `list_workspace_files`.
    - File authoring: `registerFileAuthoringTools` → `create_file`, `edit_file`.
    - Skills: `injectSkillCatalog` → the `skill` tool plus a catalog text block.
    - Background-execution and intent parameters: `backgroundToolNames` / `intentToolNames`.
    - Subagent discovery: `packages/api/src/agents/discovery.ts` runs a BFS over agent graph edges, checks VIEW permission, and initializes each subagent with the same pipeline.

**Returned to the run:** `{toolDefinitions, toolRegistry, mcpAvailableTools, userMCPAuthMap, requestScopedConnections, toolContextMap, dynamicToolContextMap, codeExecutionContext, primedCodeFiles, actionsEnabled, oauthActionToolNames, mcpToolAliases, repositoryInstructionSource}`. The run keeps one entry per agent in `agentToolContexts`.

### 1b. Execution phase: `ON_TOOL_EXECUTE` → `loadToolsForExecution`
The SDK emits a batch of tool calls. `createToolExecuteHandler` (`packages/api/src/agents/handlers.ts:5488`) does the following:
- Filters blocked argument content.
- Handles host-native tools itself: `skill`, `read_file`, `create_file`/`edit_file`, background tools, subagent tasks.
- Calls the host's `loadTools` callback (`api/server/services/Endpoints/agents/initialize.js:447`), which is `loadToolsForExecution` (`ToolService.js:2078`).

`loadToolsForExecution` then:
- Re-checks capabilities and the RUN_CODE permission for the *executing* agent.
- Builds `tool_search`, PTC tools and `bash_tool`. There are two `bash_tool` variants: `createBashExecutionTool` for managed environments, `createAttachedWorkspaceBashTool` for attached ones.
- Instantiates regular tools through `loadTools` in `api/app/clients/tools/util/handleTools.js`.
- Instantiates action tools through `loadActionToolsForExecution`.
- Returns `{loadedTools, configurable}`. `configurable` carries `userMCPAuthMap`, `requestBody`, `requestScopedConnections`, `codeExecutionContext`, `toolRegistry`, `ptcToolMap` and `backgroundToolNames`.
- **`callerCapabilities.ts`** accepts only a versioned (v1) SDK snapshot of which tools are directly callable and which are code-callable. It is intersected with the trusted registry and is never treated as authorization.
- **`toolValidation.ts`** turns schema-validation errors into structured, privacy-safe diagnostics.

### 1c. `loadTools` in `handleTools.js` (the instance factory)
For every tool it first re-checks the role gate. Then it dispatches:

| Tool | What gets built |
|---|---|
| `execute_code` | `primeCodeFiles`, then `createCodeExecutionTool({user_id, files, authHeaders, ...codeExecutionContext})` |
| `file_search` | `primeSearchFiles` plus the FILE_CITATIONS permission check, then `createFileSearchTool` (`util/fileSearch.js`). It calls the RAG API `/query` with `{file_id, query, k:5, entity_id?}` |
| `web_search` | `loadWebSearchAuth`, then `createSearchTool({...authResult, httpAgent, httpsAgent (SSRF-safe), onSearchResults, onGetHighlights})` |
| `ask_user_question` | `createAskUserQuestionTool` (a LangGraph interrupt) |
| `set_memory` / `delete_memory` | `buildInlineMemoryTool` |
| MCP keys | Grouped by server, then `createMCPTools` (the `all` placeholder) or `createMCPTool` per tool. Servers are processed sequentially; a failed server is skipped |
| Custom constructors | `image_gen_oai`, `gemini_image_gen` |
| Manifest classes | `loadToolWithAuth(user, authFields, Ctor)` |

**Credential resolution** is `loadAuthValues` (`api/server/services/Tools/credentials.js`). For each auth field (alternatives separated by `||`) it uses `process.env[field]` if set and not equal to `user_provided`. Otherwise it reads the user's `pluginAuth` row, decrypted with `getUserPluginAuthValue` in `PluginService.js`.

**`pluginAuth` schema** (`packages/data-schemas/src/schema/pluginAuth.ts`): `{authField, value (AES-CBC encrypted), userId, pluginKey, tenantId}`, indexed on `(userId, pluginKey, authField, tenantId)`.
- Written by `POST /api/user/plugins`, `{pluginKey, action: install|uninstall, auth}` (`UserController.updateUserPluginsController`).
- The same store holds MCP `customUserVars` (`pluginKey = mcp_<server>`) and web-search user keys.

**Tool listing and direct calls.**
- `GET /api/agents/tools` → `PluginController.getAvailableTools`. It joins `manifest.json` with the cached tool definitions and marks tools `authenticated`.
- `GET /api/agents/tools/:toolId/auth` → `verifyToolAuth`.
- `POST /api/agents/tools/:toolId/call` → `controllers/tools.js callTool`. This is a direct re-run of `execute_code` from the UI. The result is persisted to the ToolCall collection, and code outputs are processed into files.

**Startup.** `api/server/services/start/tools.js` loads structured tool classes and formats them as OpenAI function tools. `includedTools`/`filteredTools` from yaml are applied. The result is stored in the tool cache (`CacheKeys.TOOL_CACHE`).

**Tool naming conventions** (`packages/data-provider/src/config.ts:4001-4019`, `types/tools.ts:120`):
- MCP: `<tool>_mcp_<server>`; prefix `mcp_`; "all" placeholder `sys__all__sys`.
- Actions: `<operationId>_action_<encodedDomain>`, where the domain separator `---` is normalized to `_`. The domain is encoded if it fits in 10 characters (`.`→`---`); otherwise it is the base64 prefix of the hostname, mapped back through the `ENCODED_DOMAINS` cache.

---

## 2. Tool categories

| Category | Identifiers | Definition source | Executor | Auth / credentials |
|---|---|---|---|---|
| Built-in / manifest plugins | `google, dalle, flux, wolfram, open_weather, tavily_search_results_json, traversaal_search, stable-diffusion, azure-ai-search, calculator, image_gen_oai(+image_edit_oai), gemini_image_gen, ask_user_question` | `api/app/clients/tools/manifest.json`, `packages/api/src/tools/registry/definitions.ts` | LangChain `Tool` classes in `api/app/clients/tools/structured/*` | env var, or per-user `pluginAuth` (`authConfig[].authField`, `||` for alternates) |
| MCP | `<tool>_mcp_<server>` | Live `tools/list` from the MCP server, cached | `MCPManager.callTool` via `services/MCP.js createToolInstance` | server-level (admin apiKey, headers, OAuth, OBO, `customUserVars`) |
| Actions (OpenAPI) | `<opId>_action_<domain>` | `Action.metadata.raw_spec` → `openapiToFunction` | `ActionRequest` executor (`data-provider/src/actions.ts`) | none / API key (basic, bearer, custom header) / OAuth2 auth-code; secrets AES-encrypted |
| Skills | `skill`, `read_file`, `create_file`, `edit_file` | Skill collection plus deployment and plugin dirs | host handlers in `packages/api/src/agents/handlers.ts` | ACL on the Skill resource |
| Code execution | `execute_code` (legacy), `bash_tool`, `read_file`, `search_workspace`, `list_workspace_files`, `bash_programmatic_tool_calling` | SDK definitions plus `agents/tools.ts` | external Code API (`/exec`, `/upload`, `/download`, `/workspace-tools/execute`) | Code API key or minted JWT; RUN_CODE permission |
| Web search | `web_search` | SDK `WebSearchToolDefinition` | SDK `createSearchTool` (search → scrape → rerank) | env / user keys (`webSearch` yaml) |
| File search | `file_search` | built-in | RAG API `/query` | FILE_SEARCH permission, JWT to RAG API |
| Memory | `set_memory`, `delete_memory` (inline); background memory agent | `agents/memory.ts getMemoryToolDefinitions` | Mongo `memoryEntries` | MEMORIES permission |
| Tool discovery | `tool_search` | SDK `ToolSearchToolDefinition` | local registry search | `deferred_tools` capability |
| Subagents | task tools / handoff edges | agent graph (`discovery.ts`, `edges.ts`, `lazySubagents.ts`) | nested LangGraph runs | VIEW ACL on sub-agent, `subagents` capability |

---

## 3. MCP architecture

### Components (`packages/api/src/mcp/`)
- **`registry/MCPServersRegistry.ts`**: the single source of truth for server configs, in three tiers.
  - The YAML cache: the `mcpServers` block of `librechat.yaml` plus Agent Plugins servers.
  - The Config-tier cache: admin overrides stored in DB config.
  - The DB repository (`registry/db/ServerConfigsDB.ts`): user-created servers in the `mcpServer` collection, with ACL.
  - Lookup order in `getServerConfig`: read-through cache → YAML → DB, then overlay the config-tier candidate. User-sourced and process-backed (stdio) entries are never overlaid.
  - Caches can live in Redis. Stored values are encrypted with `encryptV2`.
  - It also resolves allowlists (`mcpSettings.allowedDomains/allowedAddresses`).
- **`registry/MCPServerInspector.ts`**: connects at startup/inspection time. It enforces the domain allowlist before connecting, then detects OAuth (`oauth/detectOAuth.ts`), capabilities, server instructions and tools. The result is a `ParsedServerConfig`. A failure writes an `inspectionFailed` stub, retried after 5 minutes.
- **`registry/MCPServersInitializer.ts`**: inspects all YAML servers at boot.
- **`connection.ts` (`MCPConnection`)**: wraps the MCP TypeScript SDK `Client`. Transports are built in `constructTransport` (line 1473):
  - `stdio`: `StdioClientTransport({command, args, env: {...defaultEnv, ...env}, cwd})`.
  - `websocket`: `WebSocketClientTransport`, with an SSRF DNS pre-check.
  - `sse`: `SSEClientTransport`, using a custom undici fetch/dispatcher with SSRF-safe connect, optional proxy, `sseReadTimeout`, and Bearer injection from OAuth tokens.
  - `streamable-http` / `http`: `StreamableHTTPClientTransport`, with the same fetch wrapper and an explicit session DELETE on close.
  - Reconnection: up to 3 attempts with backoff. It stops on a 429 or when OAuth is required. A circuit breaker (`CB_*` env settings) limits connect/disconnect cycles.
  - `tools/list` pagination is bounded: 50 pages, 1000 tools, 5 MiB, 30 s.
  - It handles `notifications/tools/list_changed` via `ToolListChangedNotificationSchema`.
  - Health checks use `ping`, falling back when the server does not implement it.
- **`MCPConnectionFactory.ts`**: creates connections and owns the OAuth, OBO and token-refresh logic: `handleOAuthRequired`, `attemptSilentTokenRefresh`, unauthenticated tool-list fallback, and `discoverTools`.
- **`ConnectionsRepository.ts`**: a per-scope pool (`ownerId` undefined = app scope). Lifecycle transitions are serialized per server, with a connect concurrency of 3.
- **`UserConnectionManager.ts` → `MCPManager.ts`** (singleton):
  - `appConnections` is a `ConnectionsRepository`.
  - `userConnections` is a `Map<userId, Map<server, MCPConnection>>`, with `userLastActivity` for idle eviction. The default idle timeout is 15 minutes (`MCP_USER_CONNECTION_IDLE_TIMEOUT`).
  - Borrower leases and deferred disposal prevent an idle sweep from killing a connection that is still in use.
  - Request-scoped ephemeral connections live in `RequestScopedMCPConnectionStore` (`api/server/services/MCPRequestContext.js`), keyed `${userId}:${server}`.
  - `callTool` retries across OAuth and OBO recovery.
- **`tools.ts`**: `formatMCPServerTools` converts MCP `Tool[]` into an OpenAI-style function map. Keys are `<strippedTool>_mcp_<normalizedServer>`, and `serverToolName` keeps the original upstream name. `createMCPToolCacheService` implements the tool cache.
- **`parsers.ts` (`formatToolContent`)**: converts MCP `CallToolResult` content into `[text, artifacts]`. Images become artifacts; resource blobs are decoded or summarized.
- **`toolsChanged.ts`**: on `list_changed`, re-fetches the list and overwrites the cache entry through `refreshChangedServerTools` in `initializeMCPs.js`. It uses a generation/revision fence.
- **`catalog/`**: bounded background catalog recovery with backoff (`mcpSettings.catalogRecovery`).
- **`authority/`**: a default-off substrate for authority proofs.
- **`oauth/`**:
  - `handler.ts`: discovery, DCR, PKCE, token exchange, refresh, revoke.
  - `tokens.ts`: `MCPTokenStorage`.
  - `OAuthReconnectionManager.ts`: at login, reconnects the user's servers that already have stored tokens, staggered 500 ms apart.
  - `obo.ts`: Entra On-Behalf-Of jwt-bearer exchange.
  - `detectOAuth.ts`: probes for 401/`WWW-Authenticate` and protected-resource metadata.
- **App glue**:
  - `api/server/services/MCP.js`: `createMCPTool(s)`, `reconnectServer`, permission context, collision audit, OAuth run-step emitters.
  - `api/server/services/initializeMCPs.js`: at boot, creates the registry and manager, merges plugin servers, and registers the toolsChanged handler.
  - Routes: `api/server/routes/mcp.js`, with controllers in `api/server/controllers/mcp.js`.

### Connection scoping (`mcp/utils.ts`)
- **`canUseAppConnection(config)`** decides whether one operator-owned connection can serve every user. It requires all of:
  - `startup !== false`;
  - not user-sourced (from the DB) and not plugin-sourced;
  - not `requiresUserScopedConnection`, meaning no `requiresOAuth`, no `obo`, no `customUserVars`, and no runtime placeholders (`{{LIBRECHAT_USER_*}}`, `{{LIBRECHAT_OPENID_*}}`, `{{LIBRECHAT_BODY_*}}`);
  - no `requestHeaders`.
- Anything else gets a **per-user connection**.
- Configs with `{{LIBRECHAT_BODY_conversationId|parentMessageId|messageId}}` placeholders are **request-scoped (ephemeral)**. They are torn down at the end of the request and never backgrounded.

### Placeholder processing (`packages/api/src/utils/env.ts processMCPEnv`)
Processing order:
1. `${ENV}` expansion on the admin template. This happens first, to prevent second-order injection.
2. `{{customUserVar}}`.
3. `{{LIBRECHAT_USER_<FIELD>}}`, limited to a whitelist of user fields.
4. `{{LIBRECHAT_OPENID_*}}` (tokens). It throws `OpenIDReauthRequiredError` if the token is expired.
5. `{{LIBRECHAT_BODY_*}}`.

Exceptions:
- DB (user) servers resolve **only** customUserVars.
- Plugin-sourced configs are returned verbatim.
- An admin `apiKey` becomes a header.

`{{LIBRECHAT_GRAPH_ACCESS_TOKEN}}` is resolved separately via OBO.

### Config fields (`packages/data-provider/src/mcp.ts`)
- **Base options:** `title, description, startup, iconPath, timeout, initTimeout, sseReadTimeout, chatMenu, serverInstructions (bool|string), requiresOAuth, oauth, oauth_headers, apiKey{key, source: admin|user, authorization_type: basic|bearer|custom, custom_header}, customUserVars{name:{title, description, sensitive}}, oauthRefreshWaitTimeout, oauthRefreshCoordination, oauthPersistenceWaitTimeout`.
- **`oauth`:** `authorization_url, token_url, client_id, client_secret, scope, redirect_uri, token_exchange_method, grant_types_supported, token_endpoint_auth_methods_supported, code_challenge_methods_supported, skip_code_challenge_check, audience, forward_audience_on_refresh, send_resource_parameter, revocation_endpoint...`.
- **Transport-specific:**
  - `stdio{command, args, env, stderr, cwd}`
  - `websocket{url}`
  - `sse` / `streamable-http`/`http` `{url, headers, requestHeaders, obo{scopes}, proxy}`
- **Global:** `mcpSettings{allowedDomains, allowedAddresses, catalogRecovery{...}}`.
- **Env:** `MCP_OAUTH_HANDLING_TIMEOUT` (default 10 min), `MCP_OAUTH_FLOW_TTL` (15 min), `MCP_CONNECTION_CHECK_TTL`, `MCP_TOOLS_LIST_*`, `MCP_CB_*` (`mcp/mcpConfig.ts`).

The DB `mcpServer` schema is `{serverName, normalizedServerName (unique per tenant), config: Mixed, author, tenantId}`. CRUD is at `/api/mcp/servers` and ACL-shared.

### OAuth sequence (MCP)
1. A connection attempt gets a 401 (or `requiresOAuth`). `MCPConnectionFactory.handleOAuthRequired` runs.
2. It looks up an existing flow in the **`FlowStateManager`** (`packages/api/src/flow/manager.ts`), backed by Keyv on Redis or in memory, with flow type `mcp_oauth` and flowId `userId:serverName`, prefixed `tenant:<t>:` when tenanted.
   - A fresh PENDING flow is joined, and its stored auth URL is re-emitted.
   - A fresh COMPLETED flow's tokens are reused.
   - Otherwise the old flow is deleted.
3. `MCPOAuthHandler.initiateOAuthFlow`. This step can fail if the domain allowlist rejects the server.
   - With a pre-configured `client_id` + `authorization_url` + `token_url`, it uses those, discovering capabilities only if the token endpoint matches.
   - Otherwise it auto-discovers: RFC 9728 protected-resource metadata (the `resource` must match the server URL), then RFC 8414 authorization-server metadata, then Dynamic Client Registration. A stored client may be reused.
   - PKCE uses `startAuthorization`. It appends `resource` (RFC 8707) or `audience`, and a random 32-byte `state`.
   - The redirect URI is `${DOMAIN_SERVER}/api/mcp/<server>/oauth/callback`.
4. The flow is stored: `flowManager.initFlow(flowId,'mcp_oauth',{...metadata, codeVerifier, clientInfo, authorizationUrl})`, plus a state→flowId mapping.
5. `oauthStart(authURL)` emits a tool-call run step with `auth` and `expires_at` to the SSE stream / GenerationJobManager. The server then **waits**, via `waitForSharedOAuthFlow`, which polls flow state until COMPLETED/FAILED or the timeout.
6. The browser follows `GET /api/mcp/:server/oauth/initiate?userId&flowId`, which sets a CSRF cookie and redirects. The IdP then redirects to `GET /api/mcp/:server/oauth/callback?code&state`.
7. The callback:
   - resolves state → flowId;
   - validates the CSRF cookie, the session cookie, or an active PENDING flow;
   - checks that the state matches the current attempt;
   - calls `completeOAuthFlow` (exchange code + verifier);
   - calls `MCPTokenStorage.storeTokens`;
   - calls `flowManager.completeFlow`;
   - reconnects the user connection;
   - redirects to `/oauth/success`.
8. **Token storage.** Tokens go in the `token` collection: `{userId, type, identifier, token (encryptV2), expiresAt, metadata}` with a TTL index. Three records:
   - `type:'mcp_oauth', identifier:'mcp:<server>'`: the access token;
   - `mcp_oauth_refresh` / `mcp:<server>:refresh`;
   - `mcp_oauth_client` / `mcp:<server>:client`: the DCR client info.
9. **Refresh.** Tokens are refreshed silently before connecting. Refreshes are coordinated across replicas through flow leases (`refreshOAuthTokens`, "refresh flights").

Other routes:
- `GET /oauth/tokens/:flowId`
- `GET /oauth/status/:flowId`
- `POST /oauth/cancel/:server`
- `POST /:server/oauth/bind`
- `POST /:server/reinitialize`
- `GET /connection/status[/:server]`
- `GET /:server/auth-values` (which customUserVars are set)
- `GET /api/mcp/tools`

---

## 4. Actions (OpenAPI → tools)

- **Schema** (`packages/data-schemas/src/schema/action.ts`): `{user, action_id, type, agent_id|assistant_id, metadata{domain, raw_spec, privacy_policy_url, api_key*, oauth_client_id*, oauth_client_secret*, auth{type: service_http|oauth|none, authorization_type (basic|bearer|custom), custom_auth_header, authorization_url, client_url (token URL), scope, token_exchange_method: default_post|basic_auth_header}}, tenantId}`. Fields marked `*` are encrypted with `encryptV2(encodeURIComponent(v))` (`packages/api/src/actions/crypto.ts`). `agent.actions[]` holds `"<domain>_action_<action_id>"`, and `agent.tools` holds the function names.
- **Create/update:** `POST /api/agents/actions/:agent_id` (`api/server/routes/agents/actions.js`). The body is `{functions, action_id?, metadata}`. The handler:
  1. runs the content-filter check;
  2. encrypts the metadata;
  3. runs `validateAndParseOpenAPISpec(raw_spec)`;
  4. calls **`validateActionDomain(metadata.domain, spec.servers[0].url)`**, an anti-SSRF check that the client domain matches the spec's server;
  5. calls `isActionDomainAllowed(domain, actions.allowedDomains, actions.allowedAddresses)`;
  6. encodes the domain;
  7. runs `planAgentActionUpdate` (`packages/api/src/actions/update.ts`), which merges tools and actions and deletes OAuth tokens if the target changed;
  8. calls `validateActionOAuthMetadata`;
  9. updates the agent with `forceVersion`, then upserts the Action. Secrets are stripped from the response.

  `DELETE /api/agents/actions/:agent_id/:action_id` removes it.
- **Spec → functions** (`openapiToFunction`, `packages/data-provider/src/actions.ts:455`):
  - Each path×method becomes one function. The name is `operationId`, or the sanitized `method_path`; the description is `summary || description`.
  - Parameters (query/path/header) plus the requestBody's top-level properties merge into one flat object schema. `paramLocations` records where each field goes, and a Zod schema is generated.
  - Each function gets an `ActionRequest(baseUrl, path, method, operationId, isConsequential, contentType, paramLocations)`.
  - `x-strict` and `x-openai-isConsequential` are honored.
- **Execution** (`ActionService.createActionTool`, `api/server/services/ActionService.js:185`): a LangChain `tool()` whose `_call` does the following.
  - `executor.setParams(input)`.
  - Auth:
    - **API key:** `Authorization: Basic base64(key)`, `Bearer key`, or a custom header.
    - **OAuth:** look up `token{type:'oauth', identifier:'userId:action_id'}` and `oauth_refresh`.
      - If there is no token, `requestLogin` builds an auth URL (`client_id, scope, redirect_uri=${DOMAIN_CLIENT}/api/actions/:id/oauth/callback, response_type=code, access_type=offline, state=JWT{nonce,user,action_id}` signed with JWT_SECRET, valid 10 minutes). It emits a run-step delta with `auth` and `expires_at` (2 minutes), then waits on flow type `oauth`.
      - `GET /api/actions/:action_id/oauth/callback` verifies the state JWT, checks CSRF/session, exchanges the code (`getAccessToken`), stores the tokens, and completes the flow.
      - If a refresh token exists, a shared `oauth_refresh` flow refreshes the access token.
  - `executor.execute(ssrfAgents)`: an axios call. SSRF-safe agents are used when `allowedDomains` is empty.
  - Object responses are JSON-stringified.
- At runtime every load re-checks the domain allowlist, spec validity and domain match. Legacy domain encodings are registered too, so older agents still resolve. OAuth actions are excluded from background dispatch (`oauthActionToolNames`).

---

## 5. Skills

- **Data model:**
  - `skill` (`packages/data-schemas/src/schema/skill.ts`): `name` (kebab-case, ≤64, reserved prefixes/words blocked), `displayTitle, description (≤1024), body (≤100k, the SKILL.md content), frontmatter (Mixed), disableModelInvocation, userInvocable, allowedTools[], category, author, authorName, version (monotonic, bumped on every edit/file change), source: inline|github|notion, sourceMetadata, fileCount, alwaysApply, tenantId`.
  - `skillFile`: `{skillId, relativePath (safe relative, not SKILL.md), file_id, filename, filepath, storageKey, source, mimeType, bytes, category: script|reference|asset|other, isExecutable, content?, isBinary?, codeEnvRef, codeEnvRefs}`.
  - `codeEnvRef` (`schema/codeEnvRef.ts`): `{kind: skill|agent|user, id, storage_session_id, file_id, version, executionProfile, executionRouteKey, provisionedAt, sandboxFilename}`. It caches where the file was uploaded in the Code API; entries are treated as fresh for 23 hours.
  - `skillSyncCredential` (encrypted GitHub token) and `skillSyncStatus` (status of each sync run).
- **Sources:**
  - Inline (UI/REST).
  - Import of `.md`/`.zip`/`.skill` (`skills/import.ts`).
  - Deployment directory `DEPLOYMENT_SKILLS_DIR` (default `skill/`), served from an in-memory registry (`skills/deployment.ts`).
  - Agent Plugins packages: `plugin.json` + `skills/` + `mcp.json` + `hooks` (`packages/api/src/plugins/*`).
  - GitHub mirroring (`skills/sync/github.ts`, `orchestrator.ts`, `scheduler.ts`), started by an admin run or opportunistically on `GET /api/skills`, rate-limited to one run per 5 minutes.
- **REST:**
  - `/api/skills` (`api/server/routes/skills.js`): list, create, get/patch/delete by `:id`, `/:id/files` list and upload, `/:id/files/*relativePath` download and delete, `/import`. Access is ACL-guarded (VIEW/EDIT/DELETE bits).
  - `/api/admin/skills`: sync credentials and runs.
  - A machine-authenticated management API at `/api/agents/v1/skills` (`docs/skills-management-api.md`). It uses optimistic concurrency via `expectedVersion`, returning 409 on conflict.
  - Per-user activation overrides live in `skills/skillStates.ts`.
- **Runtime** (`packages/api/src/agents/skills.ts`, `skillFiles.ts`):
  - `injectSkillCatalog` lists accessible, active skills. At most 100 appear in the catalog by default, and descriptions are truncated. The catalog text is injected, and the `skill` tool plus `read_file` (and `bash_tool` when code is enabled) are registered.
  - A **`skill` tool call** (`handleSkillToolCall`, `handlers.ts:5051`) resolves the skill by name within `accessibleSkillIds`. It rejects `disableModelInvocation` skills, returns the body as an injected message tagged `source:'skill'`, and primes bundled files into the code sandbox at `/mnt/data/skills/<name>/...` (`primeSkillFiles`, single-flight per skill version, read-only batch upload).
  - **Manual** (`$skill` in the UI) and **always-apply** skills are primed as HumanMessages at the start of the turn. Limits: 10 manual, 20 always-apply, 30 total.
  - `allowedTools` from primed skills is unioned into the tool set.
  - The model can author skills during the run: `create_file` at `skills/<name>/SKILL.md` makes that skill invocable in the same run.

---

## 6. Code execution and environments

- **Profiles** (`packages/api/src/agents/execution.ts`):
  - **`default`**: the stateless managed Code API. The base URL comes from the SDK `getCodeBaseURL()`, env `LIBRECHAT_CODE_BASEURL`; the SDK default is not verifiable here.
  - **`stateful`**: persistent sessions. The URL is `LIBRECHAT_CODE_BASEURL_STATEFUL` or the configured environment's `baseURL`. The scope is `user | agent-user | conversation` (`STATEFUL_CODE_ENVIRONMENTS`). `runtimeSessionHint` is a hashed scope fingerprint, and `codeSessionKey = execute_code:<routeKey>:<hint>`.
- **Environments config:** `endpoints.agents.statefulCodeSessions{allowedEnvironments, principalWorkers{enabled, maxPerUser}, conversationMoves, environments[{id, name, type: managed|attached, baseURL, default, workerId, owner, configSchema, settings, pairing{workerId, allowPrincipalWorkers, tokenEnv}}]}`.
  - The DB `codeEnvironment` collection stores user-registered ("principal") attached environments: `{environmentId, name, type, baseURL, controlPlaneId, createdBy, workerId, workerPrincipal, settings{permissions{fileWrite, commandExecution: allow|ask|deny}}, plus lease/revocation fields}`.
  - Routes: `/api/code-environments` (list, pairings, create, settings, delete) and `/api/admin/code-environments/:id/pairings|revoke`.
  - `code/lifecycle.ts` reconciles registration and revocation leases in the background.
- **Attached environments** (CONTEXT.md): a `librechat-code` worker on the user's own machine connects outbound to the Code API bridge. Each conversation stores an immutable workspace decision (`codeWorkspaces`). Workspace operations: `read_file, search_text, list_files, execute_command, write_file, edit_file, preview_edit`.
- **Auth to the Code API** (`packages/api/src/auth/codeapi.ts`):
  - Either a static API key (the `CODE_API_KEY` credential via `loadAuthValues`), or, when `CODEAPI_JWT_ENABLED`/managed mode is on, a minted RS256/ES JWT.
  - JWT claims: `iss, aud, sub=userId, tenant_id, role, principal_source, org_id, service_id, plan_id, code_worker_id, auth_context_hash, jti, exp`. Tokens are cached.
  - Extra headers: `X-CodeAPI-Expected-Profile: default|stateful`, `X-LibreChat-Code-Worker-ID`, `User-Id`, `User-Agent: LibreChat/1.0`.
- **External Code API contract**, as seen from the calls in `api/server/services/Files/Code/*` and `packages/api/src/code/*`:

  | Endpoint | Request | Response / notes |
  |---|---|---|
  | `POST {base}/exec` | JSON `{lang, code, args?, session_id?, runtime_session_hint?, files?: [{id, session_id, name}]}` | `{stdout, stderr, session_id, files:[{id, name}]}`. Output files come back as tool artifacts. |
  | `POST {base}/upload` | multipart: file, `kind`, `id`, `version` | `{message:'success', storage_session_id, files:[{fileId, filename}]}` |
  | `POST {base}/upload/batch` | as above, plus `read_only` | |
  | `GET {base}/download/{session_id}/{fileId}` | | file stream |
  | `DELETE {base}/files/{session_id}/{fileId}` | | |
  | file info / lastModified | | used for freshness checks |
  | `POST {base}/workspace-tools/execute` | `{protocolVersion:1, operation, workspaceId, workspaceInstanceId, ...}` | bounded JSON; retried on admission timeouts, honoring `Retry-After` |
  | `POST {base}/bridge/pairings` | Bearer pairing token, `{workerId, binding}` | |
  | `GET /bridge/workers/:id/status` | | |
  | `POST /bridge/workers/:id/revoke` | | |

  The SDK `execute_code` tool's input schema is `{lang, code, args}`. The host `read_file` uses `/exec` with `cat`. `bash_tool` sends bash code.
- **File priming** (`Files/Code/process.js primeFiles`):
  - Agent and user files in `tool_resources.execute_code` are uploaded, or reused through a fresh `codeEnvRef`.
  - The model sees a list of `/mnt/data/<file>` paths.
  - `primedCodeFiles` seeds the graph's code session so the first call can see earlier artifacts.
  - Output files are downloaded, run through content filters, stored through the file strategy (`processCodeOutput`), and streamed as attachments.

---

## 7. Web search pipeline

- **Config** (`webSearch` in yaml; `webSearchSchema`, `packages/data-provider/src/config.ts:2529`):
  - **Search providers:** `serper | searxng | tavily | keenable`.
  - **Scrapers:** `firecrawl | serper | tavily | keenable`.
  - **Rerankers:** `jina | cohere | none`.
  - Keys and URLs default to `${SERPER_API_KEY}`, `${SEARXNG_INSTANCE_URL}`, `${FIRECRAWL_API_KEY}`/`URL`/`VERSION`, `${JINA_API_KEY}`/`URL`, `${COHERE_API_KEY}`, `${TAVILY_*}`, `${KEENABLE_*}`.
  - Other settings: `scraperTimeout`, `safeSearch` (0/1/2), `firecrawlOptions`, `searxngSearchOptions`, `allowedAddresses`.
- **Auth resolution** (`packages/api/src/web/web.ts loadWebSearchAuth`). For each category (providers, scrapers, rerankers):
  - It picks the first provider whose required fields resolve, from env or from user `pluginAuth` (values set to `user_provided` are asked of the user). The user may select a provider.
  - User-supplied URLs go through an SSRF preflight.
  - Admin keys are never forwarded to user-supplied URLs.
  - `authenticated=false` drops the tool entirely.
- **SSRF agents** (`web/agent.ts`) are http/https agents with connect-time IP checks, with exemptions for configured proxies.
- **Execution:** the SDK `createSearchTool` runs, in order:
  1. search (organic, topStories, images, news);
  2. scrape the top links;
  3. chunk and rerank highlights;
  4. return formatted sources with `turn` indices.
- **Host callbacks** (`api/server/services/Tools/search.js`): `onSearchResults` builds a `web_search` attachment `{turn, organic, topStories, ...}` streamed as `event: attachment`; `onGetHighlights` updates it as highlights arrive.
- **Prompt context** (`tools/toolkits/web.ts`): the citation grammar uses private-use unicode anchors (`turn{N}search{i}`, groups `…`, highlights `…`), plus a dynamic date context.
- REST: `GET /api/agents/tools/web_search/auth` verifies web-search auth. Keys are installed with `POST /api/user/plugins` (`pluginKey: web_search`).

---

## 8. Memory

- **Schema** `memoryEntries` (`schema/memory.ts`): `{userId, key (/^[a-z_]+$/), value, agentId? (partition; null means the shared personal pool), tokenCount, updated_at, tenantId}`, indexed on `(userId, agentId, key)`.
- **Config** `memory` (`memorySchema`):
  - `disabled, validKeys[], tokenLimit, charLimit (10000), maxInputTokens, personalize, messageWindowSize (5)`;
  - `agent: {enabled, id} | {enabled, provider, model, instructions, model_parameters}`.
  - Automatic extraction is opt-in via `agent.enabled: true` (`packages/data-schemas/src/app/memory.ts`).
- **Injection** (`AgentClient.useMemory`, `api/server/controllers/agents/client.js:3214`):
  - Checks the MEMORIES USE permission and `personalize`.
  - `getFormattedMemories` returns `withKeys`/`withoutKeys` strings and token counts.
  - The `withoutKeys` text goes into the system prompt as `"# Existing memory about the user:\n..."`, preceded by `memoryInstructions`.
  - A per-request `WeakMap` cache shares one read across multiple agents (`getRequestMemories`).
- **Extraction (background memory agent):**
  - `createMemoryProcessor` → `processMemory` (`packages/api/src/agents/memory.ts:745`). After the response, `runMemory(messages)` takes the last `messageWindowSize` chat messages (skill primes stripped), truncated to `maxInputTokens`.
  - It builds a separate `Run` whose system instructions are the memory instructions plus `# Existing memory:` and the keyed memories. The fallback model is `openai/gpt-4.1-mini`; Bedrock gets the instructions in the user message instead.
  - The run's only tools are `set_memory({key, value})` and `delete_memory({key})`, which enforce valid keys and the token limit and write through `setMemory`/`deleteMemory`.
  - Results are streamed as `memory` artifacts/attachments.
- **Inline memory tools:** when an agent includes the `memory` tool (capability `memory`), `set_memory`/`delete_memory` are registered directly on the agent (`registerMemoryTools`, `buildInlineMemoryTool`), partitioned by `agentId`. A usage guard in the prompt limits them to explicit "remember/forget" requests.
- **REST** `/api/memories` (`routes/memories.js`): GET (list with usage), POST (create), `PATCH /preferences` (opt-out), `PATCH/DELETE /:key`, partition-scoped via `?agent_id` (`memory/authorization.ts`), all behind PII/content filters (`memory/protection.ts`).

---

## 9. Artifacts and OpenAPI export

- **Artifacts** are *not* a tool. They are generated inline in model text between `:::artifact{identifier,type,title}` and `:::`.
  - The prompt comes from `api/app/clients/prompts/artifacts.js generateArtifactsPrompt({endpoint, artifacts: default|shadcnui|custom})`, gated by the `artifacts` capability.
  - `packages/api/src/artifacts/update.ts` provides `findAllArtifacts` (fence-aware boundary parsing over `message.content[].text` or `message.text`) and `replaceArtifactContent`.
  - `POST /api/messages/artifact/:messageId {index, original, updated}` edits one artifact in place and re-saves the message.
- **OpenAPI export:** `api/server/routes/openapi.js` mounts `createOpenApiRouter` (`packages/api/src/openapi/router.ts`) at `/api`.
  - It serves `GET /api/openapi.json` (a pre-generated `agents.openapi.json`) and Swagger UI at `/api/docs`, only when `openapi.enabled: true`.
  - The spec is generated from Zod contracts (`openapi/agents.ts`, `skills.ts`, `registry.ts`, `adapter.ts`, `generate.ts`), with the `oidcBearer` security scheme, covering the Agent and Skill management APIs.

---

## 10. Python re-implementation mapping

| Concern | Node implementation | Python equivalent |
|---|---|---|
| Agent graph / tool loop | `@librechat/agents` (LangGraph JS), `ON_TOOL_EXECUTE` batches | `langgraph` (`StateGraph`, `ToolNode` or a custom async tool-executor node), `langchain_core.tools.StructuredTool` / `BaseTool` with `response_format="content_and_artifact"` |
| Tool registry / deferred tools | `LCToolRegistry`, `createToolSearch` | A dict of name → `{schema, allowed_callers, defer_loading}`, plus a local BM25/regex `tool_search` tool that returns schemas for the model to bind next turn |
| JSON-schema → tool args | zod, `normalizeJsonSchema`, `sanitizeGeminiSchema` | `pydantic.create_model` or `jsonschema`; `langchain_core.utils.function_calling.convert_to_openai_tool`; `jsonref` for `$ref` resolution |
| MCP client and transports | `@modelcontextprotocol/sdk` | Official `mcp` SDK: `mcp.ClientSession`; `mcp.client.stdio.stdio_client(StdioServerParameters)`, `mcp.client.sse.sse_client`, `mcp.client.streamable_http.streamablehttp_client`, `mcp.client.websocket.websocket_client`; `session.list_tools()` (cursor pagination), `session.call_tool()`, notification handler for `tools/list_changed` |
| MCP OAuth | custom `MCPOAuthHandler` plus SDK auth helpers | `mcp.client.auth.OAuthClientProvider` (PKCE, DCR, RFC 9728/8414 discovery) with a custom `TokenStorage` backed by Mongo; or `authlib` / `httpx-oauth` |
| MCP → LangChain tools | `services/MCP.js createToolInstance` | `langchain-mcp-adapters` (`load_mcp_tools`, `MultiServerMCPClient`) as a base, or a thin `StructuredTool` wrapper calling a pooled `ClientSession` |
| Connection pooling / idle eviction | `MCPManager`, `ConnectionsRepository` | An asyncio manager: `dict[(scope, user, server)] → session` held by `contextlib.AsyncExitStack`, `asyncio.Lock` per key, a background task for the idle sweep, `tenacity` for backoff and circuit breaking |
| FlowStateManager (OAuth waits) | Keyv + Redis polling | `redis.asyncio` (`SET NX PX`, pub/sub or polling), with status PENDING/COMPLETED/FAILED and TTL leases |
| OpenAPI actions | `openapiToFunction`, `ActionRequest` | `openapi-core` or `prance` to parse and validate, `jsonref`; build pydantic models per operation; execute with `httpx.AsyncClient` (custom transport for SSRF IP checks) |
| Encryption | AES-CBC `encryptV2` (`hex iv:hex ct`, `CREDS_KEY`) | `cryptography.hazmat` AES-CBC + PKCS7, same format for data compatibility |
| JWT (action state, Code API) | `jsonwebtoken`, custom signer | `PyJWT` (HS256 state, RS256/ES256 Code API tokens) |
| Code API client | axios/fetch | `httpx.AsyncClient` (multipart via `files=`), streaming downloads |
| Web search | SDK `createSearchTool` | Direct `httpx` calls to Serper/SearXNG/Tavily; Firecrawl (`firecrawl-py`); rerank via Jina HTTP or the `cohere` SDK; `tiktoken` for chunk budgeting |
| SSRF protection | `createSSRFSafeAgents`, `resolveHostnameSSRF` | `ipaddress` checks after `socket.getaddrinfo` / `anyio` DNS, enforced in a custom `httpx.AsyncHTTPTransport` |
| Memory agent | separate `Run` with set/delete tools | A second LangGraph/LLM call with two `StructuredTool`s, run in the background (`asyncio.create_task` / Celery / arq) |
| Schemas / DB | Mongoose | `motor`/`beanie` or `pydantic` + `pymongo`, keeping the same collections and indexes (`pluginAuth`, `token` with TTL, `action`, `mcpServer`, `skill`, `skillFile`, `memoryEntries`, `codeEnvironment`) |
| Config validation | zod (`librechat.yaml`) | `pydantic` v2 models with discriminated unions on `type` for MCP transports; env expansion `${VAR}` done before user placeholders |
| OpenAPI export | zod-to-openapi + Swagger UI | FastAPI's built-in OpenAPI and Swagger (`/openapi.json`, `/docs`) |

**Key invariants to keep:**
- App-level versus per-user versus request-scoped MCP connections, decided by `canUseAppConnection` / `requiresUserScopedConnection`.
- Placeholder ordering: env first, then user vars. DB servers resolve only customUserVars; plugin configs are never resolved.
- Domain and SSRF allowlists checked before every outbound connection, for actions, MCP and web search.
- The action domain must match the spec's server URL.
- Tool names are normalized and collisions fail closed.
- Tool instances are created lazily per call, with credentials resolved at call time.
- OAuth is a blocking "flow" that the chat stream surfaces as a synthetic tool-call step with an auth URL.
