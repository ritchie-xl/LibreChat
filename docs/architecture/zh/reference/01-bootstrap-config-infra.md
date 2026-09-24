# 启动、中间件、配置、缓存与基础设施

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

主入口是 `api/server/index.js`，通过 `npm run backend` 启动，实际执行 `NODE_ENV=production node api/server/index.js`（根目录 `package.json:45`）。

---

## 1. 启动顺序（`api/server/index.js`）

**模块加载（在 `startServer` 之前）**
1. `require('../config/credentials')` 加载 `api/config/credentials.js`。它先运行 `dotenv`，再调用 `bootstrapCredentials()`（`packages/api/src/credentials.ts`）。该函数确保 `CREDS_KEY`、`CREDS_IV`、`JWT_SECRET` 和 `JWT_REFRESH_SECRET` 存在。每个值依次取自：环境变量、临时凭据文件（`LIBRECHAT_TEMP_CREDENTIALS_PATH`），或新生成的值。已知的旧默认值会按指纹拒绝。指纹记录保存在 Mongo 集合 `librechatCredentialMetadata` 中。
2. `require('./telemetry')` 加载 `api/server/telemetry.js`。只有当 `OTEL_TRACING_ENABLED` 或 `OTEL_LOGS_ENABLED` 为 true 且 `OTEL_SDK_DISABLED` 未开启时，才会启动 OpenTelemetry NodeSDK。这一步必须在加载 express、http 和 mongoose 之前完成，自动埋点才能对它们打补丁。
3. `module-alias` 把 `~` 映射到 `api/`。
4. require `~/db`（`api/db/index.js`）时调用 `createModels(mongoose)`，注册所有 Mongoose 模型。之后才 require `indexSync`，因为它在加载时就捕获了 `mongoose.models.Message/Conversation`。require `~/models`（`api/models/index.js`）时调用 `createMethods(mongoose, {matchModelName, findMatchingPattern, isExternalSkillId, getCache: getLogStores})`，返回整个数据访问层的函数集合。
5. require `~/cache/getLogStores` 时构建所有具名的 Keyv 命名空间（见 §5）。Redis 关闭时，还会启动一个 30 s 的内存 TTL 清理器。
6. 当 `USE_REDIS=true` 时，`packages/api/src/cache/redisClients.ts` 创建 `ioredis` 客户端和 `@keyv/redis`（node-redis）客户端。
7. 模块加载时配置两个 ReDoS 防护：`configureFileConfigRegexEngine()` 和 `configureMessageFilterRegexValidator()`（RE2）。
8. 读取环境变量：`PORT`（默认 3080；允许为 0）、`HOST`（默认 `localhost`）、`TRUST_PROXY`（默认 1）、`ALLOW_SOCIAL_LOGIN`、`DISABLE_COMPRESSION`。
9. 创建 Express 应用。设置 `app.locals.codeApiUploadRegistry`。初始化就绪状态 `serverReady=false` 和 `scheduleEngineState='starting'`。

**`startServer()`（async）**
1. `await waitForKeyvRedisClient()`：阻塞直到 node-redis 客户端连接成功（没有 Redis 时为空操作）。
2. `await configureSubagentTaskRouting()`。
3. `createMetrics(...)` 构建 Prometheus 注册表、`metricsMiddleware` 和 `metricsRouter`。如果未设置 `METRICS_SECRET` 会输出警告。
4. `await connectDb()`（`api/db/connect.js`）让 mongoose 连接到 `MONGO_URI`。
5. `startCodeEnvironmentLifecycleReconciler({mongoose})` 启动一个后台协调器。它通过 `packages/api/src/code/lifecycle.ts` 中的 `isLeader()` 做领导者门控。
6. `indexSync()` 以即发即弃（fire-and-forget）方式执行 Meilisearch 同步（§4.4）。
7. `app.disable('x-powered-by')` 和 `app.set('trust proxy', n)`。
8. 安全响应头（helmet）在所有路由之前注册，包括健康检查。
9. 针对 `TRUST_TENANT_HEADER` 与 `TENANT_ISOLATION_STRICT` 的组合输出警告。
10. `await runAsSystem(seedDatabase)` 运行 `initializeRoles`、`seedDefaultRoles`、`ensureDefaultCategories` 和 `seedSystemGrants`（`api/models/index.js`）。
11. `runAsSystem(sweepOrphanedPreviews)` 在后台运行。
12. `appConfig = await getAppConfig({ baseOnly: true })` 加载由 YAML 生成的基础配置（§4）。
13. 根据 appConfig 配置运行时：
    - `configureAgentEventRuntime(appConfig.endpoints.agents.eventDriven)`
    - `warnOnUnreachableDeliveryPaths`
    - `initializeFileStorage(appConfig)`（`packages/api/src/app/cdn.ts`）：为 `fileStrategy` 以及 `fileStrategies` 中的每一项初始化 S3、Firebase、Azure Blob 或 CloudFront。
14. 初始化部署插件和技能：
    - `initializeDeploymentPlugins`
    - `setPluginHookSource`
    - `initializeDeploymentSkills`
    - `initializeGitHubSkillSync(appConfig)`
    - `startExpiredFileSweep`
    - `loadToolApprovalHooks(endpoints.agents.toolApproval.hooks)`，仅在 `toolApproval.enabled` 时执行。
15. 在 `runAsSystem` 内：
    - `performStartupChecks(appConfig)`（`packages/api/src/app/checks.ts`）：环境变量弃用检查、凭据数据库检查、Azure 变量、界面配置、配置 `version`、网页搜索配置，以及 `handleRateLimits`（把 YAML 中的 `rateLimits` 复制到 `*_IP_MAX` 这类环境变量中）。它还会 ping `RAG_API_URL/health`。
    - `updateInterfacePermissions(...)`（`packages/api/src/app/permissions.ts`）：把 YAML 中的 `interface.*` 开关同步到 Role 文档。
16. 读取并预处理 `client/dist/index.html`：
    - 根据 `DOMAIN_CLIENT` 的路径改写 `<base href>`。
    - 注入来自 `CUSTOM_FOOTER` 的页脚引导脚本。
    - 设置 CSP 策略和页面外壳（shell）的缓存头。
    - `sendIndexHtml` 添加 `lang` 属性、query-devtools 引导脚本和 CSP nonce。
17. 健康检查路由：`/health` 和 `/livez` 总是返回 200。`/readyz` 在 `serverReady` 之前返回 503。
18. 挂载全局中间件管线和路由（§2、§3）。
19. `configureGenerationStreams()`：
    - `createStreamServices()` 根据 `USE_REDIS_STREAMS` 选择 Redis 任务存储加 pub/sub 传输，或者内存实现。
    - `GenerationJobManager.configure/initialize` 设置 `cleanupOnComplete = !STREAM_KEEP_COMPLETED_JOBS`。
    - 注册两个关闭任务：排空前的 `prepareForShutdown` 和排空后的 `destroy`，后者保留 10 s 余量。
20. `app.listen(port, host, cb)`。监听后的回调执行以下步骤，任何一步失败都会调用 `process.exit(1)`：
    - `runAsSystem`：`initializeMCPs()` 和 `initializeOAuthReconnectManager()`
    - `checkMigrations()`
    - 可选的 `memoryDiagnostics.start()`（`--inspect` 或 `MEM_DIAG`）
    - `initializeAgentTriggerService({address, completionResultBatchSize})`
    - `initializeScheduleEngine()`，把 `scheduleEngineState` 设为 `armed` 或 `unavailable`（后者在进程生命周期内不会改变）
    - `serverReady = true`
21. `configureServerTimeouts(server)`（`packages/api/src/app/server.ts`）应用 `HTTP_KEEP_ALIVE_TIMEOUT_MS`、`HTTP_KEEP_ALIVE_TIMEOUT_BUFFER_MS`、`HTTP_HEADERS_TIMEOUT_MS` 和 `HTTP_REQUEST_TIMEOUT_MS`，并把 headers 超时限制为 ≤ 请求超时。
22. `setupGracefulShutdown(server)`（`packages/api/src/app/shutdown.ts`）接管 SIGTERM、SIGINT、SIGQUIT 和 SIGHUP。收到信号后：
    - 开始 `server.close()`。
    - 运行**排空前（pre-drain）**任务，按优先级降序、再按注册顺序排序。
    - 等待 HTTP 连接排空。
    - 运行**排空后（post-drain）**任务。
    - `process.exit(0)`，任何失败则退出码为 1。
    - 60 s 后触发强制退出。`getRemainingShutdownMs()` 暴露剩余的时间预算。
    - 所有清理都必须通过 `registerShutdownTask(name, fn, {phase, priority})` 注册，不要直接写信号处理器。

**进程级处理器**
- `startServer().catch` 以退出码 1 退出（快速失败）。
- `uncaughtException` 会吞掉一组已知情况：abort、`GoogleGenerativeAI`、`fetch failed`（视为 Meili 不可用）、OpenAIError，以及堆栈位于 `@librechat/agents` 中的错误。其他错误都会导致退出，除非设置了 `CONTINUE_ON_UNCAUGHT_EXCEPTION`。
- `unhandledRejection` 记录日志后继续运行。

**就绪门控**
- 服务就绪之前，`rejectChatStartsUntilReady` 对 `POST /api/agents/chat/*`（`/abort` 除外）返回 503，带 `Retry-After: 1` 和错误码 `SERVER_NOT_READY`。
- `rejectScheduleWritesUntilReady`（`createScheduleWriteGate`）对 `/api/schedules` 的写操作做同样处理。

**`api/server/experimental.js`**（`npm run backend:experimental`）在 Node `cluster` 下运行同样的启动流程：
- 主进程先对 Redis 执行 `FLUSHALL`（单节点或 `Redis.Cluster`），然后 fork `CLUSTER_WORKERS` 个工作进程（默认 4 个）来模拟多个 pod。
- 它通过 IPC `{type:'file-retention-sweep-worker'}` 选出一个正在监听的工作进程执行文件保留期清理，使用 `createClusteredFileSweep`。
- 它向下传递关闭截止时间，并在收到信号 10 s 后强制退出整个集群。
- 它落后于 `index.js`：没有 `/metrics`、`/api/rum`、`/api/traces`，缺少大部分 `/api/admin/*` 路由，没有遥测，也没有就绪门控。只应把它当作开发用的测试工具。

**其他启动相关文件**
- **`api/server/cleanup.js`**：智能体客户端的内存清理。`disposeClient(client)` 把大型 graph/run 属性置空（清单见 `graphPropsToClean` 和 `graphRunnablePropsToClean`）。`FinalizationRegistry` 用于调试日志，`requestDataMap` 是一个 WeakMap。`processReqData` 合并每个请求的流数据。在 Python 中基本不需要这些，直接释放引用即可。
- **`api/server/socialLogins.js`**：仅在 `ALLOW_SOCIAL_LOGIN=true` 时运行。
  - 根据环境变量是否存在注册 passport 策略：Google、Facebook、GitHub、Discord 和 Apple，每个都有一个用户策略和一个 `*Admin` 变体。OAuth state 用 `JWT_SECRET` 签名，其 TTL 来自 `registration.oauthStateTtlMs`。
  - 当设置了 `OPENID_CLIENT_ID`、`OPENID_ISSUER`、`OPENID_SCOPE` 和 `OPENID_SESSION_SECRET`，并且设置了 `OPENID_CLIENT_SECRET` 或 `OPENID_USE_PKCE` 之一时，启用 OpenID。它挂载 `express-session`（使用 `getLogStores` 中的 `OPENID_SESSION` 存储）以及 `passport.session()`。`registerOpenIdWithRetry` 注册 `openidJwt` 策略，重试参数来自 `OPENID_DISCOVERY_RETRY_*` 或 `registration.openidDiscovery`。开启 `OPENID_REUSE_TOKENS` 时，会话最长有效期至少为 `OPENID_REUSE_MAX_SESSION_AGE_MS`（默认 15 分钟）。
  - 当设置了 `SAML_ENTRY_POINT`、`SAML_ISSUER`、`SAML_CERT` 和 `SAML_SESSION_SECRET` 时启用 SAML，并使用 `SAML_SESSION` 存储。
  - 会话 cookie 的最长有效期为 `SESSION_EXPIRY`；`secure` 标志由 `shouldUseSecureCookie()` 决定。

---

## 2. 路由挂载表（`api/server/index.js:321-464`；路由器来自 `api/server/routes/index.js`）

以下路由在全局中间件（§3.1）之后按此顺序挂载。路由器文件位于 `api/server/routes/`。

| # | 前缀 | 挂载级别的额外中间件 | 路由器文件 |
|---|---|---|---|
| – | `GET /health`、`/livez`、`/readyz` | 无（在管线之前） | 内联 |
| – | `GET /index.html` | – | 内联 `sendIndexHtml` |
| – | 静态资源 `dist`、`fonts`、`assets` | `utils/staticCache` | `api/server/utils/staticCache.js` |
| 1 | `/oauth` | `preAuthTenantMiddleware` | `oauth.js` |
| 2 | `/api/auth` | `preAuthTenantMiddleware` | `auth.js` |
| 3 | `/api/insights` | – | `insights.js` |
| 4 | `/api/admin` | – | `admin/auth.js` |
| 5 | `/api/admin/config` | （路由器内部执行 `requireJwtAuth` + `requireCapability(ACCESS_ADMIN)`） | `admin/config.js` |
| 6 | `/api/admin/code-environments` | – | `admin/code.js` |
| 7 | `/api/code-environments` | – | `code-environments.js` |
| 8 | `/api/admin/langfuse` | – | `admin/langfuse.js` |
| 9 | `/api/admin/grants` | – | `admin/grants.js` |
| 10 | `/api/admin/groups` | – | `admin/groups.js` |
| 11 | `/api/admin/roles` | – | `admin/roles.js` |
| 12 | `/api/admin/skills` | – | `admin/skills.js` |
| 13 | `/api/admin/users` | – | `admin/users.js` |
| 14 | `/api/admin/audit-log` | – | `admin/audit.js` |
| 15 | `/api/actions` | – | `actions.js` |
| 16 | `/api/keys` | – | `keys.js` |
| 17 | `/api/api-keys` | – | `apiKeys.js` |
| 18 | `/api/user` | – | `user.js` |
| 19 | `/api/search` | – | `search.js` |
| 20 | `/api/messages` | – | `messages.js` |
| 21 | `/api/convos` | – | `convos.js` |
| 22 | `/api/traces` | – | `traces.js` |
| 23 | `/api/presets` | – | `presets.js` |
| 24 | `/api/projects` | – | `projects.js` |
| 25 | `/api/prompts` | – | `prompts.js` |
| 26 | `/api/skills` | – | `skills.js` |
| 27 | `/api/categories` | – | `categories.js` |
| 28 | `/api/endpoints` | – | `endpoints.js` |
| 29 | `/api/balance` | – | `balance.js` |
| 30 | `/api/models` | – | `models.js` |
| 31 | `/api/config` | `preAuthTenantMiddleware`、`optionalJwtAuth` | `config.js` |
| 32 | `/api/assistants` | – | `assistants/` |
| 33 | `/api/files` | –（async `routes.files.initialize()`） | `files/` |
| 34 | `/images/` | `createValidateImageRequest({secureImageLinks})` | `static.js` |
| 35 | `/api/share` | `preAuthTenantMiddleware` | `share.js` |
| 36 | `/api/roles` | – | `roles.js` |
| 37 | `/api/agents/chat` | `rejectChatStartsUntilReady`（仅门控） | – |
| 38 | `/api/agents` | – | `agents/index.js`（子挂载：`/v1/responses` → `responses.js`，`/v1/agents` → `management.js`，`/v1/skills` → `skills.js`，`/v1` → `openai.js`（API 密钥认证）；然后是 `requireJwtAuth`；`/chat/*` → `chat.js`；`/` → `v1.js`） |
| 39 | `/api/banner` | – | `banner.js` |
| 40 | `/api/memories` | – | `memories.js` |
| 41 | `/api/schedules` | `rejectScheduleWritesUntilReady` | `schedules.js` |
| 42 | `/api/permissions` | – | `accessPermissions.js` |
| 43 | `/api/tags` | – | `tags.js` |
| 44 | `/api/mcp` | – | `mcp.js` |
| 45 | `/api/rum` | – | `rum.js` |
| 46 | `/metrics` | `metricsRouter`（Bearer `METRICS_SECRET`，`timingSafeEqual`，否则返回 401） | `packages/api/src/app/metrics.ts` |
| 47 | `/api` | – | `openapi.js`（仅当 YAML 中 `openapi.enabled` 时提供） |
| 48 | `/api` | `apiNotFound`（JSON 404） | `@librechat/api` |
| 49 | `*` | SPA 兜底 → `sendIndexHtml` | `api/server/utils/fallback.js` |
| 50 | 错误处理 | `telemetryErrorMiddleware`（启用 OTel 时），然后是 `ErrorController` | `packages/api/src/middleware/error.ts` |

大多数路由器自己应用 `requireJwtAuth`，要么在路由器级别，要么逐个路由。认证不是全局的。

---

## 3. 中间件目录

### 3.1 全局管线（按顺序，`api/server/index.js:204-391`）
1. `createSecurityHeaders()`（`packages/api/src/security/headers.ts`）应用 helmet，并关闭 CSP。环境变量：`SECURITY_HEADERS`、`HSTS_ENABLED`、`HSTS_MAX_AGE`（默认 31536000）、`HSTS_INCLUDE_SUBDOMAINS`、`HSTS_PRELOAD`、`X_FRAME_OPTIONS`、`CROSS_ORIGIN_OPENER_POLICY`、`CROSS_ORIGIN_RESOURCE_POLICY`、`REFERRER_POLICY`。
2. `requestContextMiddleware`（`packages/api/src/middleware/tenant.ts:91`）生成 `requestId`（UUID），并在 `tenantStorage.run({requestId, requestMethod, requestPath})`（一个 AsyncLocalStorage）中执行请求的后续部分。
3. `agentStartupIngressMiddleware`，仅作用于 `/api/agents/chat`：记录智能体启动延迟的里程碑。
4. `metricsMiddleware`：Prometheus 的 HTTP、SSE 和上传指标。
5. `noIndex`：设置 `X-Robots-Tag: noindex`，除非 `NO_INDEX=false`。
6. `express.json({limit:'3mb'})` 和 `express.urlencoded({limit:'3mb'})`，然后是 `handleJsonParseError`。
7. 一个让 `req.query` 可写的垫片（Express 5），然后是 `express-mongo-sanitize`，它会剔除以 `$` 开头和包含 `.` 的键。
8. `cors()`（完全放开）、`cookieParser()`，以及 `compression()`（除非设置了 `DISABLE_COMPRESSION`）。
9. 静态文件服务（§2）。
10. `telemetryMiddleware`（OTel span 增强），以及作用于 `/api/agents/chat` 的 `agentStartupTelemetryMiddleware`。
11. `passport.initialize()`，策略包括 `jwt`（`jwtLogin`）、`local`（`passportLogin`）、`ldap`（设置了 `LDAP_URL` 和 `LDAP_USER_SEARCH_BASE` 时），以及社交登录策略和会话中间件。
12. `capabilityContextMiddleware`（`packages/api/src/middleware/capabilities.ts:90`）建立一个按请求的 AsyncLocalStorage `capabilityStore`，用来记忆主体和能力检查结果。

### 3.2 `api/server/middleware/*`

**认证**
| 中间件 | 用途 |
|---|---|
| `requireJwtAuth.js` | Passport `jwt`（Bearer 访问令牌）。如果 cookie `token_provider=openid` 且开启了 `OPENID_REUSE_TOKENS`，先尝试 `openidJwt`，失败再回退到 `jwt`；同时检查 `openid_user_id` cookie 是否匹配。设置 `req.user` 和 `req.authStrategy`，串联 `tenantContextMiddleware`，然后刷新 CloudFront 签名 cookie。返回 401，JSON 为 `{message, code?:'ACCOUNT_DELETION_IN_PROGRESS'}`。它还导出 `requireRumProxyAuth`，失败时静默返回 204。 |
| `optionalJwtAuth.js` | 策略相同，但不会失败；令牌有效时设置 `req.user` 和租户上下文。 |
| `requireLocalAuth.js` / `requireLdapAuth.js` | 用于登录的 Passport `local` / `ldapauth`。 |
| `setTwoFactorTempUser.js` | 把临时 2FA 令牌解码到 `req.user`。 |
| `optionalShareFileAuth.js` | 用于共享文件的 `<img>` 请求：从 `refreshToken` cookie，或 OpenID 会话加签名的 `openid_user_id` cookie 中解析出查看者。 |
| `requireSameOrigin.js` | `createSameOriginGuard`，受信任来源为 `DOMAIN_CLIENT`、`DOMAIN_SERVER`、`ADMIN_PANEL_URL`（用于基于 cookie 认证的路由的 CSRF 防护）。 |
| `oauthNavigation.js` | `markOAuthNavigation` 标记 OAuth 浏览器导航，使被拒绝的请求重定向到登录页，而不是返回 JSON。 |

**滥用防护与限流**
| 中间件 | 用途 |
|---|---|
| `checkBan.js` | 若设置了 `BAN_VIOLATIONS`：先在基于 Mongo 的 Keyv 封禁缓存中查询 IP 和用户（或由 `req.body.email` 找到的用户）（有 Redis 时键为 `ban_cache:ip:<ip>` / `ban_cache:user:<id>`，否则为原始值），再查询 `getLogStores(BAN)`。过期的封禁会被删除。被封禁的请求会收到 403 JSON；交互式智能体聊天收到 `denyRequest` SSE；OAuth 请求则被重定向。 |
| `uaParser.js` | 拒绝非浏览器的 User-Agent，并记录 `NON_BROWSER` 违规（分值 `NON_BROWSER_VIOLATION_SCORE`，默认 20）。 |
| `limiters/*` | 见 §3.3。 |
| `moderateText.js` | 若设置了 `OPENAI_MODERATION`：使用 `OPENAI_MODERATION_API_KEY` 对文本、引用内容和合并后的文本调用 `OPENAI_MODERATION_REVERSE_PROXY` 或 OpenAI `/v1/moderations`。被标记的内容通过 `denyRequest` 拒绝。 |

**聊天请求校验**
| 中间件 | 用途 |
|---|---|
| `validateModel.js` | 用 `getModelsConfig(req)` 校验 `req.body.model`。不匹配时记录 `ILLEGAL_MODEL_REQUEST` 违规（分值 `ILLEGAL_MODEL_REQ_SCORE`）。目前在智能体聊天链中被注释掉了。 |
| `buildEndpointOption.js` | 用 `parseCompactConvo` 解析请求体，应用模型规格预设（`applyModelSpecPreset` / `resolveModelSpecForEndpoint`），对 `{{current_user}}` 提示词替换执行内容过滤检查，调用端点的 `buildOptions`（agents / assistants / azureAssistants），设置 `req.body.endpointOption`，并更新文件使用情况。 |
| `validate/convoAccess.js` | 确认用户拥有 `conversationId`。结果缓存在 `CONVO_ACCESS` 违规缓存中；违规分值为 `CONVO_ACCESS_VIOLATION_SCORE`。 |
| `validate/subagentThreadTurn.js` | `createSubagentThreadTurnGuard`：保护子智能体线程上的轮次。 |
| `messageValidation.js` / `validateMessageReq.js` | `createMessageRequestMiddleware`：校验消息相关路由（`canReadActiveJobConversation`、`prepareMessageRequestValidation`、`sendValidationResponse`）。 |

**流式、中止与错误**
| 中间件 | 用途 |
|---|---|
| `abortMiddleware.js` | `abortMessage`：通过 `GenerationJobManager` 中止一次生成（streamId 等于 conversationId），并持久化已生成的部分内容和元数据。 |
| `abortRun.js` | 中止 OpenAI Assistants 的运行。 |
| `setHeaders.js` | SSE 响应头：`text/event-stream`、`Cache-Control: no-cache, no-transform`、`Connection: keep-alive`、`X-Accel-Buffering: no`、`ACAO: *`。 |
| `denyRequest.js` | 发送 SSE 错误或最终消息，可选择保存用户消息（用于封禁和内容审核）。 |
| `error.js` | `sendError` / `handleAbortError`：保存错误消息并以 SSE 发送。 |

**配置与角色**
| 中间件 | 用途 |
|---|---|
| `config/app.js`（`configMiddleware`） | `req.config = await getAppConfig(getAppConfigOptionsFromUser(req.user))`，得到按用户、角色、组和租户合并后的配置。失败时回退为只含 `{tenantId}`。 |
| `roles/admin.js`（`checkAdmin`） | 除非 `req.user.role === ADMIN`，否则返回 403。 |
| `roles/capabilities.js` | `generateCapabilityCheck`：`hasCapability`、`requireCapability(cap)`、`hasConfigCapability`、`getHeldCapabilities`、`getReadableConfigSections`。它们使用 SystemGrant 模型和主体（用户、角色、组）。请直接导入，以避免循环 require。 |

**资源访问（ACL，`accessResources/*`）**
| 中间件 | 用途 |
|---|---|
| `canAccessResource` | 通用的 ACL 权限位检查（`PermissionBits` VIEW/EDIT/DELETE/SHARE），基于资源类型和 id 在 AclEntry 上检查。 |
| `canAccessAgentResource`、`canAccessAgentFromBody` | 基于 URL 参数或 `req.body.agent_id` 的智能体 ACL。 |
| `canAccessPromptGroupResource`、`canAccessPromptViaGroup` | 提示词和提示词组 ACL。 |
| `canAccessMCPServerResource`、`canAccessSkillResource` | MCP 服务器和技能 ACL。 |
| `fileAccess` | 文件 ACL。 |

**共享与人员选择器**
| 中间件 | 用途 |
|---|---|
| `canAccessSharedLink.js` | `createSharedLinkAccessMiddleware({mongoose})`。 |
| `checkSharePublicAccess.js` | `createSharePolicyMiddleware({getRoleByName, hasCapability})`。 |
| `checkPeoplePickerAccess.js` | `createPeoplePickerAccess({getRoleByName})`。 |

**注册与账户**
| 中间件 | 用途 |
|---|---|
| `checkDomainAllowed.js` | 对社交登录强制执行 `registration.allowedDomains`。 |
| `checkInviteUser.js` | 注册时校验邀请令牌。 |
| `validateRegistration.js` | 根据 `ALLOW_REGISTRATION` 控制是否允许注册。 |
| `validatePasswordReset.js` | 根据 `ALLOW_PASSWORD_RESET` 控制是否允许重置密码。 |
| `validateEmailLogin.js` | 根据 `ALLOW_EMAIL_LOGIN` 控制是否允许邮箱登录。 |
| `canDeleteAccount.js` | 账户删除权限检查。 |

**其他**
| 中间件 | 用途 |
|---|---|
| `validateImageRequest.js` | 使用 JWT 或 `secureImageLinks` 对 `/images/*` 做授权。 |
| `assistants/validate.js`、`assistants/validateAuthor.js` | Assistant 的允许/拒绝列表以及所有权。 |
| `logHeaders.js` | 以调试级别记录转发相关的请求头。 |
| `noIndex.js` | 见上文。 |

### 3.3 限流器（`api/server/middleware/limiters/*`，express-rate-limit）

存储为 `limiterCache(prefix)`：当 `USE_REDIS=true` 时是使用 ioredis 的 `rate-limit-redis` `RedisStore`，否则是进程内默认存储。实际写入的键形如 `{REDIS_KEY_PREFIX}::{prefix}:{key}`。除非另有说明，时间窗口单位为分钟。超出限制时调用 `logViolation`。

| 限流器 | 键 | 环境变量（默认值） | 违规类型 |
|---|---|---|---|
| `loginLimiter` | IP（`removePorts`） | `LOGIN_WINDOW`=5，`LOGIN_MAX`=7，`LOGIN_VIOLATION_SCORE` | `logins` |
| `registerLimiter` | IP | `REGISTER_WINDOW`=60，`REGISTER_MAX`=5 | `registrations` |
| `resetPasswordLimiter` / `...SubmissionLimiter` | IP | `RESET_PASSWORD_WINDOW`=2，`_MAX`=2（提交限流器有自己的 `_SUBMISSION_` 变量） | `reset_password_limit` |
| `verifyEmailLimiter` / `...SubmissionLimiter` | IP | `VERIFY_EMAIL_WINDOW`=2，`_MAX`=2 | `verify_email_limit` |
| `twoFactorTempLimiter`（IP + 用户） | IP；用户 id 取自临时 JWT | `TWO_FACTOR_TEMP_WINDOW`/`MAX`（默认取 LOGIN_* 的值） | `logins` |
| `messageIpLimiter` / `messageUserLimiter` | IP / `req.user.id` | `MESSAGE_IP_MAX`=40，`MESSAGE_IP_WINDOW`=1，`MESSAGE_USER_MAX`=40，`MESSAGE_USER_WINDOW`=1；由 `LIMIT_MESSAGE_IP` / `LIMIT_MESSAGE_USER` 启用 | `message_limit` |
| `agentEventUserLimiter` | `req.apiKeyId ?? user.id` | `AGENT_EVENT_USER_MAX`=40，`_WINDOW`=1；返回 429 并带 `Retry-After` | – |
| `generationRetryProbeLimiter` / `generationRetryLimiter` | （packages/api） | 开启消息限流时生效 | – |
| 文件上传 IP / 用户 | IP / 用户 | `FILE_UPLOAD_IP_MAX`=100/15 分钟，`FILE_UPLOAD_USER_MAX`=50/15 分钟；另有文件使用量 `FILE_USAGE_USER_MAX`=120/15 分钟 | `file_upload_limit` |
| 导入 IP / 用户 | IP / 用户 | `IMPORT_IP_MAX`=100/15，`IMPORT_USER_MAX`=50/15 | `file_upload_limit` |
| 分叉（fork）IP / 用户 | IP / 用户 | `FORK_IP_MAX`=30/1，`FORK_USER_MAX`=7/1 | `file_upload_limit` |
| 共享 IP / 用户 | IP / 用户（匿名时跳过用户限流器） | `SHARE_IP_MAX`=100/1，`SHARE_USER_MAX`=60/1，`SHARE_VIOLATION_SCORE`=0 | `share_limit` |
| TTS / STT 的 IP 与用户 | IP / 用户 | `TTS_IP_MAX`=100/1，`TTS_USER_MAX`=50/1（STT 相同） | `tts_limit` / `stt_limit` |
| `toolCallLimiter` | 用户 | 每 1000 ms 1 个请求 | `tool_call_limit` |
| `promptUsageLimiter` | 用户 | 每分钟 30 次，固定值 | – |

YAML 中的 `rateLimits.{fileUploads, conversationsImport, tts, stt, agentEvents}` 会在启动时覆盖对应的环境变量（`packages/api/src/app/limits.ts`）。

并发消息限制（`LIMIT_CONCURRENT_MESSAGES`、`CONCURRENT_MESSAGE_MAX`）使用 `PENDING_REQ` 缓存，由 `api/cache/clearPendingReq.js` 清理；实现位于 `packages/api/src/middleware/concurrency.ts`。

### 3.4 典型的聊天请求链：`POST /api/agents/chat/:endpoint?`
1. 全局管线（§3.1）。`rejectChatStartsUntilReady`。
2. `agents/index.js` 路由器：`requireJwtAuth`（其中包含 `tenantContextMiddleware`），然后捕获触发器和定时任务上下文（`isAgentTriggerRequest`、`captureScheduleFireContext`），然后是 `checkBan`，再然后是 `uaParser`。
3. `chatRouter`（`agents/index.js:1138-1173`）：`configMiddleware`（设置 `req.config`）。如果启用了消息限流：`generationRetryProbeLimiter`、`detectGenerationRetry`、`generationRetryLimiter`、`messageIpLimiter` 和 `messageUserLimiter`，每一个都对触发器、定时任务和已确认的重试做了豁免。
4. `chat.js` 路由器：`restoreResumeContext`（用于 `/resume`）、`createMessageFilterPii`、`moderateText`、`checkAgentAccess`（角色权限 `AGENTS.USE`）、`canAccessAgentFromBody(VIEW)`、`validateConvoAccess`、`guardSubagentThreadTurn`、`buildEndpointOption`。
5. `AgentController(req,res,next, initializeClient, addTitle)` 启动一个 `GenerationJobManager` 任务。客户端随后在 `GET /api/agents/chat/stream/:streamId` 订阅 SSE，该路由没有限流器。中止接口为 `POST /api/agents/chat/abort`。

---

## 4. 配置系统

### 4.1 加载 `librechat.yaml`
文件：`api/server/services/Config/loadCustomConfig.js`。
1. 路径为 `CONFIG_PATH`，默认 `<root>/librechat.yaml`。如果是 `http(s)` URL，则用 axios 获取。
2. 用 `loadYaml` / `js-yaml` 解析文件。
3. 在校验之前执行 `setMaxSubagents(endpoints.agents.maxSubagents)`。
4. 用 `configSchema.strict().safeParse(...)` 校验（zod，`packages/data-provider/src/config.ts:2957`）。校验失败时进程退出；但如果 `CONFIG_BYPASS_VALIDATION=true`，则返回 `null` 并以默认值运行。
5. 记录配置日志，其中的密钥由 `redactConfigSecretMaps` 脱敏。
6. 后处理：OpenRouter 默认值（`promptCache` 参数），以及依据 `paramSettings` 校验 `customParams`。

接着 `loadBaseConfig`（`api/server/services/Config/app.js`）调用 `loadAndFormatTools`（按 `filteredTools` / `includedTools` 过滤），然后调用 `AppService({config, paths, systemTools})`（`packages/data-schemas/src/app/service.ts`）。AppService 把原始配置规范化为 **AppConfig**：
- 校验 `ocr`、`webSearch`、`memory`、`summarization`、`skillSync`、`langfuse` 和 `filters`。
- `interfaceConfig` 来自 `loadDefaultInterface`，`turnstileConfig` 来自 `loadTurnstileConfig`，`mcpConfig` 来自 `mcpServers`，端点来自 `loadEndpoints`。
- 设置 `fileStrategy`，并据此设置 `process.env.CDN_PROVIDER`。
- `balance` 回退到 `CHECK_BALANCE` / `START_BALANCE`。
- 其他字段：`transactions`、`registration`、`imageOutputType`、`secureImageLinks`（默认 true）、`availableTools`、`paths`，以及原始 `config`。

`api/config/paths.js` 提供 `paths`：root、uploads、clientPath、dist、publicPath、fonts、assets、imageOutput、structuredTools、pluginManifest。

### 4.2 `librechat.yaml` 的顶层键（configSchema）
- **通用：** `version`（必填；与 `Constants.CONFIG_VERSION` 比对）、`cache`（bool）、`permissions.maxWriteAttempts`、`openapi.enabled`。
- **功能模块：** `ocr`、`webSearch`、`langfuse`、`memory`、`summarization`、`skillSync`。
- **工具：** `includedTools`、`filteredTools`。
- **图片：** `secureImageLinks`、`imageOutputType`（png、jpeg、webp）。
- **MCP：** `mcpServers`，以及 `mcpSettings.{allowedDomains, allowedAddresses, catalogRecovery{discoveryBackoffMs, discoveryTimeoutMs, reauthRetryMs, authorizationFence*...}}`。
- **界面与认证：** `interface`（功能开关，同时驱动角色权限）、`turnstile`、`registration.{socialLogins, allowedDomains, oauthStateTtlMs, openidDiscovery}`。
- **文件存储：** `fileStrategy`（local、s3、firebase、azure_blob、cloudfront）、`fileStrategies`（按文件类型）、`cloudfront`、`fileConfig`。
- **Actions：** `actions.{allowedDomains, allowedAddresses}`。
- **计费：** `balance`、`transactions`。
- **语音：** `speech.{tts, stt, speechTab}`。
- **限制：** `rateLimits`。
- **模型规格：** `modelSpecs`。
- **内容过滤：** `filters`（仅限基础配置的安全边界）、`messageFilter`（PII 正则，RE2）。
- **端点：** `endpoints.{allowedAddresses, all, openAI, google, anthropic, azureOpenAI, azureAssistants, assistants, agents, custom[], bedrock}`，严格模式，至少包含一个键。解析顺序为 `all`，然后是具名端点，最后是自定义端点自身的配置。

### 4.3 按租户、角色、组和用户的覆盖配置
- **数据库模型：** `Config`（`packages/data-schemas/src/schema/config.ts`），按租户隔离。字段：`principalType`（user、role、group…）、`principalId`、`principalModel`、`priority`、`overrides`（YAML 的一个 Mixed 子集，使用 YAML 键名）、`tombstones`（要删除的点分路径列表）、`isActive`、`configVersion`、`tenantId`。在 (principalType, principalId, tenantId) 上有唯一索引。
- **特殊主体：** 租户基础配置为 `{principalType: ROLE, principalId: '__base__'}`（`BASE_CONFIG_PRINCIPAL_ID`）。
- **服务：** `createAppConfigService`（`packages/api/src/app/service.ts`）：
  - `ensureBaseConfig()` 把 YAML 生成的 AppConfig 以键 `_BASE_` 缓存在 `APP_CONFIG` 缓存中，不设 TTL，并把 `availableTools` 推入工具缓存。
  - `getAppConfig({role, userId, idOnTheSource, tenantId, refresh, baseOnly, failClosed, resolvedPrincipals, skipRuntimeAugmentation})`：
    - `baseOnly` 返回基础配置，用于启动阶段和认证策略中。
    - 否则用 `getUserPrincipals` 解析主体（用户、角色、组）。
    - 缓存键：`_OVERRIDE_:{tenant}:{role}:{userId}`（另有仅角色、仅用户、`__base__` 的变体）。租户依次回退到 ALS 中的租户，再到 `__default__`。
    - 未命中时调用 `getApplicableConfigs(principals)`，这是一个 Mongo 查询，取 `__base__` 以及各主体中 `isActive` 的文档，按 priority 升序排序。
    - `mergeConfigOverrides`（`packages/data-schemas/src/app/resolution.ts:284`）按优先级顺序应用每个文档：先处理 tombstones，再深度合并 `overrides`。
      - YAML 键会被重新映射：`mcpServers` → `mcpConfig`，`interface` → `interfaceConfig`，`turnstile` → `turnstileConfig`。
      - `endpoints.custom` 数组按 `name` 合并。MCP 服务器的覆盖会被过滤。
      - `filters` 仅限基础配置，不能被覆盖。`langfuse` 只能由 `__base__` 主体设置。
      - 合并深度有限制，并且拒绝 `__proto__` 这一类键。
    - 合并结果以 60 s TTL 缓存（`DEFAULT_OVERRIDE_CACHE_TTL`）。
    - 然后 `augmentConfig` 按用户合并其可访问的代码环境。
    - 在严格租户模式下，如果拿不到租户，则返回不缓存的基础配置。
  - `clearAppConfigCache()` 删除 `_BASE_`。`clearOverrideCache(tenantId?)` 只对内存存储枚举键；使用 Redis 时依赖 TTL 过期，刻意避免 SCAN。
  - `invalidateConfigCaches(tenantId)`（`api/server/services/Config/app.js:77`）在管理员修改后清除基础配置、覆盖配置、工具和 MCP 配置缓存。
- **认证流程辅助函数：** `resolveAppConfigForUser`（`packages/api/src/app/resolve.ts`）在用户有租户时，于 `tenantStorage.run({tenantId})` 中执行 `getAppConfig({role, tenantId})`，否则返回基础配置。
- **管理 API：** `/api/admin/config`（`api/server/routes/admin/config.js`，处理器在 `createAdminConfigHandlers` 中），需要 `ACCESS_ADMIN` 能力。
  - `GET /` 和 `GET /base`
  - `GET|PUT|DELETE /:principalType/:principalId`
  - `PATCH /:principalType/:principalId/fields`
  - `POST /:principalType/:principalId/fields/tombstone`
  - `DELETE /:principalType/:principalId/fields`
  - `PATCH /:principalType/:principalId/active`
  - 按配置分区强制执行读写能力检查（`hasConfigCapability`、`getReadableConfigSections`）。
- **`packages/api/src/app/config.ts` 中的辅助函数：** `getBalanceConfig`、`getTransactionsConfig`（开启余额时强制开启交易记录）、`getCustomEndpointConfig`（规范化名称匹配加密钥解析）、`getEndpointsDropParamsMap`。
- **构建信息：** `packages/api/src/app/build.ts` 提供 `resolveBuildInfo()`，数据来自 `BUILD_COMMIT`、`BUILD_BRANCH` 和 `BUILD_DATE`，缺失时回退到 git。设置了 `interface.buildInfo` 时通过 `/api/config` 暴露。

### 4.4 数据库层
- **`api/db/connect.js`：** mongoose 连接缓存在 `global.mongoose` 上，使用 `bufferCommands:false` 和 `strictQuery`。环境变量：`MONGO_URI`（必填）、`MONGO_MAX_POOL_SIZE`、`MONGO_MIN_POOL_SIZE`、`MONGO_MAX_CONNECTING`、`MONGO_MAX_IDLE_TIME_MS`、`MONGO_WAIT_QUEUE_TIMEOUT_MS`、`MONGO_AUTO_INDEX`、`MONGO_AUTO_CREATE`。查询指标来自 `instrumentMongooseQueryMetrics`。
- **模型：** `packages/data-schemas/src/models/index.ts` 中的 `createModels(mongoose)` 注册约 45 个模型，包括：
  - User、Token、Session、Balance、Conversation、Message
  - Agent、AgentApiKey、AgentCategory、MCPServer、Role、Action、Assistant
  - File、Banner、Key、PluginAuth、Transaction、Preset、Prompt、PromptGroup
  - Skill、SkillFile、SkillSync*、ConversationTag、Config、AccessRole、AclEntry、Group、SystemGrant
  - SharedLink、ToolCall、Memory、AuditLog、Schedule / ScheduleRun、QueuedTurn，以及触发器投递、通道（lane）和清除（purge）相关模型
  - OpenIDRefreshFlight、RefreshTokenBridge、CodeEnvironment、ChatProject、Favorite

  每个 `createXModel` 都会应用 `applyTenantIsolation(schema)`。Message 和 Conversation 还挂载了 `mongoMeili` 插件（`models/plugins/mongoMeili.ts`），用于同步到 Meilisearch。
- **方法：** `packages/data-schemas/src/methods/*` 中的 `createMethods(mongoose, deps)`，每个领域一个文件（config、aclEntry、role、user、message、conversation…）。它在 `api/models/index.js` 中被展平导出。
- **`api/db/indexSync.js`：** 由 `SEARCH=true` 启用；`MEILI_NO_SYNC` 会跳过索引。需要 `MEILI_HOST` 和 `MEILI_MASTER_KEY`；`MEILI_SYNC_THRESHOLD` 默认为 1000。
  - 基于 `FLOWS` 缓存使用 `FlowStateManager`，以流程 id `meili-index-sync` 作为分布式单执行者锁（10 分钟 TTL，Redis Lua 执行器）。
  - 对 Meili 做健康检查，确保可过滤属性已设置（设置变化时重置 `_meiliIndex` 标志以强制重新同步），删除没有 `user` 字段的文档，并在超过阈值时同步尚未索引的消息和对话。

### 4.5 重要环境变量（仅限启动与基础设施）
- **服务器：** `PORT`、`HOST`、`TRUST_PROXY`、`DOMAIN_CLIENT`、`DOMAIN_SERVER`、`NODE_ENV`、`DISABLE_COMPRESSION`、`NO_INDEX`、`CUSTOM_FOOTER`、`HTTP_*_TIMEOUT_MS`、`CONTINUE_ON_UNCAUGHT_EXCEPTION`、`MEM_DIAG`、`CONFIG_PATH`、`CONFIG_BYPASS_VALIDATION`。
- **密钥：** `CREDS_KEY`、`CREDS_IV`、`JWT_SECRET`、`JWT_REFRESH_SECRET`、`SESSION_EXPIRY`、`REFRESH_TOKEN_EXPIRY`、`OPENID_*`、`SAML_*`，以及各提供商的 `*_CLIENT_ID` / `*_CLIENT_SECRET`、`ALLOW_SOCIAL_LOGIN`、`ALLOW_REGISTRATION`、`ALLOW_EMAIL_LOGIN`、`ALLOW_PASSWORD_RESET`、`LDAP_URL`、`LDAP_USER_SEARCH_BASE`。
- **数据库与搜索：** `MONGO_*`、`SEARCH`、`MEILI_*`、`RAG_API_URL`。
- **Redis：** `USE_REDIS`、`REDIS_URI`（集群时以逗号分隔）、`USE_REDIS_CLUSTER`、`REDIS_USERNAME`、`REDIS_PASSWORD`、`REDIS_CA`、`REDIS_KEY_PREFIX` 或 `REDIS_KEY_PREFIX_VAR`、`USE_REDIS_STREAMS`、`FORCED_IN_MEMORY_CACHE_NAMESPACES`、`REDIS_PING_INTERVAL`，以及 §5 中列出的其他调优变量。
- **滥用防护：** `BAN_VIOLATIONS`、`BAN_DURATION`（默认 2 小时）、`BAN_INTERVAL`（默认 20）、`VIOLATION_SCORE_TTL`（默认 1 小时）、`*_VIOLATION_SCORE`、`LIMIT_*`、§3.3 中的限流器变量、`OPENAI_MODERATION*`。
- **租户：** `TENANT_ISOLATION_STRICT`、`TRUST_TENANT_HEADER`。
- **集群：** `LEADER_LEASE_DURATION`（25 s）、`LEADER_RENEW_INTERVAL`（10 s）、`LEADER_RENEW_ATTEMPTS`（3）、`LEADER_RENEW_RETRY_DELAY`（0.5 s）、`CLUSTER_WORKERS`。
- **可观测性：** `METRICS_SECRET`、`OTEL_TRACING_ENABLED`、`OTEL_LOGS_ENABLED`、`OTEL_LOGS_LEVEL`、`OTEL_SDK_DISABLED`、`OTEL_SERVICE_NAME`、`OTEL_SERVICE_VERSION`、`OTEL_IOREDIS_TRACING_ENABLED`，以及标准的 `OTEL_EXPORTER_*`、`DEBUG_LOGGING`、`DEBUG_CONSOLE`、`CONSOLE_JSON`、`CONSOLE_LOG_LEVEL`、`LOG_TO_FILE`。
- **安全响应头：** `SECURITY_HEADERS`、`HSTS_*`、`X_FRAME_OPTIONS`、`REFERRER_POLICY`、`CROSS_ORIGIN_*`。
- **余额：** `CHECK_BALANCE`、`START_BALANCE`。

---

## 5. 缓存与 Redis 设计

**配置**（`packages/api/src/cache/cacheConfig.ts`）
- `USE_REDIS` 要求设置 `REDIS_URI`。
- `USE_REDIS_STREAMS` 默认取 `USE_REDIS` 的值。
- `FORCED_IN_MEMORY_CACHE_NAMESPACES` 默认为 `CONFIG_STORE,APP_CONFIG`，因此由 YAML 生成的配置保持在每个容器内。名称会与 `CacheKeys` 比对校验。
- 键前缀为 `REDIS_KEY_PREFIX`，或由 `REDIS_KEY_PREFIX_VAR` 所指定的环境变量的值，分隔符为 `::`。
- 调优变量：`REDIS_MAX_LISTENERS`（40）、`REDIS_PING_INTERVAL`、`REDIS_PING_TIMEOUT`（5000）、`REDIS_SUBSCRIBER_PING_INTERVAL`（15）、`REDIS_KEEP_ALIVE`、`REDIS_RETRY_MAX_DELAY`（3000）、`REDIS_RETRY_MAX_ATTEMPTS`（10）、`REDIS_CONNECT_TIMEOUT`、`REDIS_READONLY_RECOVERY_INTERVAL`、`REDIS_ENABLE_OFFLINE_QUEUE`、`REDIS_USE_ALTERNATIVE_DNS_LOOKUP`（ElastiCache TLS）、`REDIS_CLUSTER_SAFE_DELETE`、`REDIS_DELETE_CHUNK_SIZE`、`REDIS_UPDATE_CHUNK_SIZE`、`REDIS_SCAN_COUNT`（1000）、`MCP_REGISTRY_CACHE_TTL`（5000）。

**客户端**（`redisClients.ts`）
- 有两套客户端库：
  1. **ioredis**（`ioredisClient`），用于限流、会话、领导者选举、主体锁和流的 pub/sub。它是 `Redis` 或 `Redis.Cluster`，设置 `keyPrefix = PREFIX::`，带抖动的指数退避，遇到 `READONLY` 时重连，并使用 CA 做 TLS。
  2. **通过 @keyv/redis 使用的 node-redis**（`keyvRedisClient`），用于 Keyv 缓存。它是 `createClient` 或 `createCluster`；集群模式下额外加了一个 `scanIterator` 垫片。
- `heartbeat.ts` 对每个节点发送有截止时间的 PING，并销毁没有响应的套接字。
- `recovery.ts` 在故障切换后收到 `READONLY` 回复时强制重连。
- `redisUtils.ts` 提供集群安全的批量删除、SCAN，以及复制出的订阅者客户端。
- `redisScript.ts` 提供 EVALSHA，并在失败时回退到 EVAL。
- `redisTelemetry.ts` 按用途记录 `redis_operations_total` 和耗时。

**工厂函数**（`cacheFactory.ts`）
- `standardCache(ns, ttl?, fallback?)` 返回一个 Keyv。有 Redis 时是基于 KeyvRedis 的 Keyv，按（namespace, ttl）记忆化，并有一个自定义的、感知命名空间的 `clear()`，实现为 SCAN 加 DEL。没有 Redis 时是按命名空间记忆化的内存 Keyv，使用 JSON 序列化器。
- `violationCache(ns)` 即 `standardCache('violations:'+ns, VIOLATION_SCORE_TTL, violationFile)`。回退存储是 `keyv-file` 存储 `./data/violations.json`。
- `sessionCache(ns)` 在有 Redis 时是基于 ioredis、前缀为 `ns:` 的 `connect-redis`，没有 Redis 时是 `memorystore`。
- `limiterCache(prefix)` 是 `rate-limit-redis` 存储，没有 Redis 时为 undefined。
- `keyvMongo`（`keyvMongo.ts`）是存放在 Mongo 集合 `keyv` 中的 Keyv 存储。
- `logFile` 为 `./data/logs.json`。
- `userPrincipalsCache()`（`principals.ts`）即 `USER_PRINCIPALS`，TTL 为 `USER_PRINCIPALS_CACHE_TTL_MS`（5 分钟）。由 Redis 支撑时，会额外加一个跨进程的构建锁：`SET key token PX lockTtl NX`，并通过 Lua 的比较后删除来释放（`USER_PRINCIPALS_LOCK_TTL_MS`、`_LOCK_WAIT_MS`）。

**具名缓存**（`api/cache/getLogStores.js`；`CacheKeys` 枚举位于 `packages/data-provider/src/config.ts:~3525`）

| 键 | 底层存储 | TTL |
|---|---|---|
| `general` 违规 | 基于 `logs.json` 文件的 Keyv | – |
| `logins`、`concurrent`、`non_browser`、`message_limit`、`registrations`、`token_balance`、`tts_limit`、`stt_limit`、`convo_access`、`share_limit`、`tool_call_limit`、`file_upload_limit`、`verify_email_limit`、`reset_password_limit`、`illegal_model_request` | `violationCache` | `VIOLATION_SCORE_TTL` |
| `ban` | 基于 keyvMongo 的 Keyv，命名空间 `BANS` | `BAN_DURATION` |
| `OPENID_SESSION`、`SAML_SESSION` | `sessionCache` | – |
| `ROLES`、`APP_CONFIG`*、`CONFIG_STORE`*、`TOOL_CACHE`、`PENDING_REQ`、`MODEL_QUERIES`、`AUTH_USER_DOC` | `standardCache`（* 默认在内存中） | – |
| `USER_PRINCIPALS` | `userPrincipalsCache`，TTL 为 0 时禁用 | 5 分钟 |
| `PROMPT_GROUPS_ACCESS` | 刻意禁用的空操作 | – |
| `ENCODED_DOMAINS` | keyvMongo | – |
| `ABORT_KEYS` | standard | 10 分钟 |
| `TOKEN_CONFIG` | standard | 30 分钟 |
| `GEN_TITLE` | standard | 2 分钟 |
| `S3_EXPIRY_INTERVAL` | standard | 30 分钟 |
| `AUDIO_RUNS` | standard | 10 分钟 |
| `MESSAGES` | standard | 1 分钟 |
| `FLOWS` | standard | 10 分钟 |
| `OPENID_EXCHANGED_TOKENS` | standard | 10 分钟 |
| `ADMIN_OAUTH_EXCHANGE` | standard | 30 s |

其他 `CacheKeys` 值（`SANDBOX_PREWARM`、`TOOLS`、`MODELS_CONFIG`、`STARTUP_CONFIG`、`ENDPOINT_CONFIG`）在别处通过 `standardCache` 使用。`CODE_ENVIRONMENT_CONFIG` 仅在 Redis 下使用（`Config/app.js`）。

没有 Redis 时，`getLogStores` 以 30 s 的间隔清除已过期的内存条目，这个定时器注册为一个关闭任务。

**违规与封禁**（`api/cache/logViolation.js`、`banViolation.js`）
1. `logViolation(req,res,type,errorMessage,score)` 在该类型的缓存中累加用户的分值。有 Redis 时键为 `type:userId`，否则为 `userId`。
2. 把事件追加到 `general` 日志。
3. 调用 `banViolation`。如果设置了 `BAN_VIOLATIONS` 且分值越过 `BAN_INTERVAL` 的某个倍数，就删除该用户的所有会话，清除认证 cookie（`refreshToken`、`openid_*`、`token_provider`），并以 userId 和 IP 为键，把 `{type, violation_count, duration, expiresAt}` 写入 `ban` 存储。
4. 随后由 `checkBan` 执行封禁。

**可恢复的流**（`packages/api/src/stream/createStreamServices.ts`）
- 有 Redis 时：`RedisJobStore` 加 `RedisEventTransport`（在复制出的 ioredis 订阅者上做 pub/sub）。否则使用内存实现。
- `GenerationJobManager` 以 streamId（等于 conversationId）为键保存任务，支持跨副本的恢复和重放。

---

## 6. 租户与集群

**租户**（`packages/data-schemas/src/config/tenantContext.ts`、`tenant/policy.ts`、`models/plugins/tenantIsolation.ts`）
- **上下文：** AsyncLocalStorage `tenantStorage` 保存 `{tenantId, userId, requestId, requestMethod, requestPath}`。访问函数：`getTenantId()`、`getUserId()`、`getRequestId()`。`SYSTEM_TENANT_ID='__SYSTEM__'`，`runAsSystem(fn)` 用于执行跨租户的系统操作。`scopedCacheKey(base)` 在缓存键后追加 `:tenantId`。
- **上下文在哪里设置：**
  1. `requestContextMiddleware`：只设置请求 id。
  2. `preAuthTenantMiddleware`，作用于 `/oauth`、`/api/auth`、`/api/config` 和 `/api/share`：仅当 `TRUST_TENANT_HEADER=true` 时读取 `X-Tenant-Id` 请求头，按 `^[-a-zA-Z0-9_.]+$` 校验且最长 128 个字符，并拒绝 `__SYSTEM__`。
  3. `tenantContextMiddleware`，由 `requireJwtAuth` 和 `optionalJwtAuth` 串联调用：从 `req.tenantId` 或 `req.user.tenantId` 取得 `tenantId`。对 `__SYSTEM__` 返回 403；没有租户且 `TENANT_ISOLATION_STRICT=true` 时也返回 403。
- **Mongoose 插件 `applyTenantIsolation`：**
  - 在每个 find、update、delete 和 aggregate 的过滤条件中注入 `{tenantId}`。
  - 在 save 和 insert 时写入 `tenantId`。
  - 除非以系统身份运行，否则禁止修改 `tenantId`。
  - 上下文中没有租户时，严格模式抛出异常（失败即关闭），非严格模式让查询原样通过（旧的单租户行为）。
  - 策略在 `tenant/policy.ts` 中与存储引擎无关，因此 Python 移植可以复用同一套规则。
- **其他租户作用域：** 配置覆盖的缓存键，以及生成任务（它们保存 `metadata.tenantId`，并与 `user.tenantId` 比对）。

**集群 / 多副本**（`packages/api/src/cluster/LeaderElection.ts`、`config.ts`）
- Redis 关闭时，`isLeader()` 返回 true。
- 否则读取键 `LeadingServerUUID`（带前缀）。如果该键归本实例所有，只要续期定时器仍然存活，本实例就是领导者。如果没有实例持有该键，则随机等待 0–2 s，然后执行 `SET key uuid EX 25 NX`。
- 续期每 10 s 通过 Lua 的检查后 `EXPIRE` 执行一次，最多尝试 3 次，间隔 0.5 s。之后放弃领导权，以避免脑裂。
- 收到 SIGTERM 或 SIGINT 时，`resign()` 执行 Lua 的比较后删除。
- 仅由领导者执行的工作包括 MCP 注册表写入（`mcp/registry/cache/BaseRegistryCache.ts`、`MCPServersInitializer.ts`）、过期文件清理（`files/sweep.ts`）以及代码环境生命周期协调器。
- 其他跨副本协调：
  - `FlowStateManager` 锁（Meili 同步、OAuth 流程）。
  - 基于 Redis 的可恢复流。
  - 用户主体构建锁。
  - `APP_CONFIG` 有意保持在每个容器内。管理员的修改依赖 60 s 的覆盖 TTL 加本地失效。

---

## 7. 可观测性
- **日志：** Winston，位于 `packages/data-schemas/src/config/winston.ts`，以 `logger` 导出。
  - 级别：`NODE_ENV=development` 时为 debug，否则为 warn。
  - 传输：DailyRotateFile `error-%DATE%.log`，20 MB，保留 14 天，除非 `LOG_TO_FILE=false`。设置了 `DEBUG_LOGGING` 时额外添加 `debug-%DATE%.log`。
  - 控制台：彩色输出，或在 `CONSOLE_JSON` 下输出 JSON。`DEBUG_CONSOLE` 开启详细的控制台输出；`CONSOLE_LOG_LEVEL` 设置级别。
  - 格式处理包括脱敏（`parsers.ts`：`redactFormat`、`redactMessage`）、JSON 截断，以及 `attachRequestContext`（从 ALS 中添加 request_id、租户和用户）。
  - Meilisearch 有自己的日志器（`meiliLogger.ts`）。
  - 日志目录：`config/utils.ts` 中的 `getLogDirectory()`。
- **指标：** 通过 `prom-client` 使用 Prometheus（`packages/api/src/app/metrics.ts`），在 `/metrics` 提供，需要 `Authorization: Bearer $METRICS_SECRET`。
  - 默认进程指标。
  - HTTP 与上传：`http_requests_total`、`http_request_duration_seconds`、`http_requests_in_flight`、`http_request_body_bytes`、`upload_*`。
  - SSE 与生成：`sse_streams_total`、`sse_streams_in_flight`、`sse_stream_duration_seconds`、`generation_jobs_total`、`generation_jobs_in_flight`、`generation_stream_*`（订阅、恢复、附加）、`agent_startup_*`。
  - 后端存储：`mongoose_queries_total`、`mongoose_query_duration_seconds`、`redis_operations_total`、`redis_operation_duration_seconds`。
  - 认证、共享与 RUM：`openid_user_lookup_total`、`share_link_rejections_total`、`rum_proxy_requests_total`、`content_filter_locator_traversal_*`。
  - 智能体事件：`agent_event_actor_*` 仪表（gauge），在抓取时通过 `runAsSystem` 采集。
  - 路径由 `normalizePath` 规范化。
- **链路追踪与日志导出：** OpenTelemetry（`packages/api/src/telemetry/*`）。
  - `sdk.ts`：NodeSDK，带 HTTP、Undici、Express、MongoDB、Mongoose 以及（可选的）IORedis 埋点。
  - `middleware.ts`：请求 span 增强和错误记录。
  - `stream.ts`：`createSseStreamTelemetry`，用于 SSE 的 span。
  - `logs.ts`：`OpenTelemetryTransportV3` 挂载一个把 Winston 日志转为 OTLP 的传输。
  - 由 `OTEL_*` 环境变量配置；关闭逻辑注册到关闭协调器。
- **浏览器 RUM：** `/api/rum` 代理来自浏览器的 OTLP 链路和日志（`requireRumProxyAuth`）。
- **Langfuse：** 可选的 LLM 链路追踪（`packages/api/src/langfuse`、YAML `langfuse`、`/api/admin/langfuse`）。
- **诊断：** `memoryDiagnostics`（配合 `MEM_DIAG`）。

---

## 8. 建议的 Python 对应方案

| Node 中的关注点 | Python 替代方案 |
|---|---|
| Express 应用 + 路由挂载 | 使用 **FastAPI** 应用，配合前缀相同的 `APIRouter`。对 `preAuthTenant` 这类挂载级中间件，使用 `app.include_router(router, prefix=..., dependencies=[...])`。 |
| 全局中间件 | 按顺序使用 Starlette 中间件：`secure` 或自定义的类 helmet 响应头中间件、请求 id / ALS（`contextvars.ContextVar`）、`prometheus-fastapi-instrumentator` 或 `prometheus_client`、`GZipMiddleware`、`CORSMiddleware`、请求体大小限制，以及一个等价于 Mongo-sanitize 的处理（拒绝以 `$` 开头或包含 `.` 的键，或者依赖有类型的 pydantic 请求体）。 |
| 按路由的中间件链 | FastAPI `Depends` 链。例如聊天路由：`Depends(require_jwt_auth)` → `tenant_context` → `check_ban` → `ua_parser` → `load_app_config` → 限流器 → `pii_filter` → `moderate_text` → `check_agent_access` → `validate_convo_access` → `build_endpoint_option`。用一个 `Request.state` 对象承载 `req.user`、`req.config` 等。 |
| passport 策略 | JWT 用 `python-jose` / `PyJWT`；OAuth2/OIDC（Google、GitHub、Discord、Facebook、Apple）用 `authlib`；`python3-saml` 或 `pysaml2`；`ldap3`；本地登录用 `passlib[bcrypt]`；OIDC/SAML 会话用 `starlette.middleware.sessions` 或带 Redis 存储的 `starsessions`。 |
| AsyncLocalStorage（租户、能力） | `contextvars.ContextVar`，它会在 asyncio 任务之间传播。把 `run_as_system()` 实现为上下文管理器。 |
| Mongoose + 租户插件 | **Motor/PyMongo** 配合 **Beanie** 或 ODMantic。在仓储基类中实现租户隔离：在每个过滤条件中注入 `tenantId`，插入时写入它，禁止 `$set: tenantId`，严格模式下失败即关闭。直接移植 `tenant/policy.ts`。在启动时创建索引（`MONGO_AUTO_INDEX`）。 |
| zod `configSchema` + `librechat.yaml` | 使用 **pydantic v2** 模型，严格模式用 `extra='forbid'`，用 `ruamel.yaml` 或 `PyYAML` 加载（远程 `CONFIG_PATH` 用 `httpx`）。环境变量设置用 **pydantic-settings** 的 `BaseSettings`，每个关注点一个类（Server、Mongo、Redis、Auth、RateLimits、OTel）。 |
| AppConfig 服务 + 覆盖配置 | 一个 `ConfigService`，基础配置放在进程内（`functools` 缓存，或对按主体的合并结果使用 `cachetools.TTLCache(ttl=60)`），并从 `resolution.ts` 移植深度合并、tombstone 和键重映射规则。 |
| Keyv 命名空间 | 一个小型 `Cache` 协议（`get`/`set`/`delete`/`clear`，带 TTL），后端包括 `redis.asyncio`（命名空间形如 `PREFIX::NS:key`）、内存 `cachetools.TTLCache`，以及带 TTL 索引的 Mongo 集合。也可以考虑 **aiocache**。 |
| ioredis / node-redis / 集群 | **redis-py** 的 `redis.asyncio.Redis` / `redis.asyncio.cluster.RedisCluster`，配合 `retry=Retry(ExponentialBackoff())`、`health_check_interval`（替代心跳）、`socket_keepalive`，以及带 CA 的 SSL。Lua 用 `register_script`。 |
| express-rate-limit + rate-limit-redis | **limits** / **slowapi** 配合 Redis 存储，或自定义的固定窗口 `INCR`+`PEXPIRE` Lua 脚本。每个限流器一个键函数（去掉端口的 IP、用户 id、API 密钥 id）。超限时调用 `log_violation`。 |
| express-session / connect-redis | 带 Redis 存储的 `starsessions`。 |
| 领导者选举 | 用 redis-py 实现同样的算法：`SET NX EX 25`，在 asyncio 任务中每 10 s 通过 Lua 续期，关闭时放弃领导权。或者使用 `redis.lock.Lock`。 |
| 可恢复的 SSE 流 | `sse-starlette` 的 `EventSourceResponse`，用 Redis Streams 或 pub/sub 做跨副本扇出，并提供内存回退。 |
| 优雅关闭注册表 | FastAPI 的 `lifespan` 上下文。维护一个按优先级排序的任务注册表（排空前任务在 uvicorn 停止接受连接之前运行，排空后任务在其之后运行），并使用 uvicorn 的 `--timeout-graceful-shutdown 60`。 |
| 就绪门控 | 一个全局 `AppState`，含 `ready` / `schedule_state`，再加一个返回 503 和 `Retry-After` 的依赖。路由 `/health`、`/livez`、`/readyz`。 |
| prom-client | **prometheus_client**，使用相同的指标名；`/metrics` 放在一个用 `hmac.compare_digest` 校验的 bearer 依赖后面。 |
| OpenTelemetry NodeSDK | `opentelemetry-sdk` 配合 `opentelemetry-instrumentation-fastapi`、`-httpx`、`-pymongo`、`-redis` 以及 OTLP 导出器。日志使用 OTel 的 `LoggingHandler`。 |
| Winston | `structlog` 或标准库 `logging` 配合 `TimedRotatingFileHandler`（error 和 debug 文件，保留 14 天）、JSON 格式化器、脱敏处理器，以及一个提供 request_id / 租户 / 用户的 `contextvars` 处理器。 |
| Meilisearch 同步 | `meilisearch` Python SDK（异步可用 `meilisearch-python-sdk`）。把同步作为受 Redis 锁保护的后台任务运行；在消息和对话写入时挂钩子以 upsert 文档。 |
| Node cluster（`experimental.js`） | 不需要：使用 `uvicorn --workers N` 或 `gunicorn -k uvicorn.workers.UvicornWorker`，单例任务用领导者选举。 |
| 基于文件的违规存储（`./data/*.json`） | 用内存 TTL 缓存替代，或直接省略；它们只在 Redis 关闭时存在。 |
