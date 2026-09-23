# Bootstrap, Middleware, Configuration, Cache & Infrastructure

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

The main entry point is `api/server/index.js`, started with `npm run backend`, which runs `NODE_ENV=production node api/server/index.js` (root `package.json:45`).

---

## 1. Startup sequence (`api/server/index.js`)

**Module load (before `startServer`)**
1. `require('../config/credentials')` loads `api/config/credentials.js`. It runs `dotenv`, then `bootstrapCredentials()` (`packages/api/src/credentials.ts`). That function ensures `CREDS_KEY`, `CREDS_IV`, `JWT_SECRET` and `JWT_REFRESH_SECRET` exist. Each value comes from, in order: the environment, a temp credentials file (`LIBRECHAT_TEMP_CREDENTIALS_PATH`), or a newly generated value. Known legacy default values are rejected by fingerprint. A fingerprint record is stored in the Mongo collection `librechatCredentialMetadata`.
2. `require('./telemetry')` loads `api/server/telemetry.js`. It starts the OpenTelemetry NodeSDK only if `OTEL_TRACING_ENABLED` or `OTEL_LOGS_ENABLED` is true and `OTEL_SDK_DISABLED` is not. This has to happen before express, http and mongoose are loaded so auto-instrumentation can patch them.
3. `module-alias` maps `~` to `api/`.
4. Requiring `~/db` (`api/db/index.js`) calls `createModels(mongoose)` to register every Mongoose model. Only then is `indexSync` required, because it captures `mongoose.models.Message/Conversation` at load time. Requiring `~/models` (`api/models/index.js`) calls `createMethods(mongoose, {matchModelName, findMatchingPattern, isExternalSkillId, getCache: getLogStores})`, which returns the whole data-access-layer function bag.
5. Requiring `~/cache/getLogStores` builds every named Keyv namespace (see §5). When Redis is off, it also starts a 30 s in-memory TTL sweeper.
6. `packages/api/src/cache/redisClients.ts` creates the `ioredis` client and the `@keyv/redis` (node-redis) client when `USE_REDIS=true`.
7. Two ReDoS guards are configured at module load: `configureFileConfigRegexEngine()` and `configureMessageFilterRegexValidator()` (RE2).
8. Env vars are read: `PORT` (default 3080; 0 is allowed), `HOST` (default `localhost`), `TRUST_PROXY` (default 1), `ALLOW_SOCIAL_LOGIN`, `DISABLE_COMPRESSION`.
9. The Express app is created. `app.locals.codeApiUploadRegistry` is set. Readiness state `serverReady=false` and `scheduleEngineState='starting'` are initialised.

**`startServer()` (async)**
1. `await waitForKeyvRedisClient()`: blocks until the node-redis client is connected (no-op without Redis).
2. `await configureSubagentTaskRouting()`.
3. `createMetrics(...)` builds the Prometheus registry, `metricsMiddleware` and `metricsRouter`. It warns if `METRICS_SECRET` is unset.
4. `await connectDb()` (`api/db/connect.js`) connects mongoose to `MONGO_URI`.
5. `startCodeEnvironmentLifecycleReconciler({mongoose})` starts a background reconciler. It is leader-gated via `isLeader()` in `packages/api/src/code/lifecycle.ts`.
6. `indexSync()` runs as fire-and-forget Meilisearch sync (§4.4).
7. `app.disable('x-powered-by')` and `app.set('trust proxy', n)`.
8. Security headers (helmet) are registered before every route, including health checks.
9. Warnings are logged for the `TRUST_TENANT_HEADER` and `TENANT_ISOLATION_STRICT` combinations.
10. `await runAsSystem(seedDatabase)` runs `initializeRoles`, `seedDefaultRoles`, `ensureDefaultCategories` and `seedSystemGrants` (`api/models/index.js`).
11. `runAsSystem(sweepOrphanedPreviews)` runs in the background.
12. `appConfig = await getAppConfig({ baseOnly: true })` loads the YAML-derived base config (§4).
13. Runtime is configured from appConfig:
    - `configureAgentEventRuntime(appConfig.endpoints.agents.eventDriven)`
    - `warnOnUnreachableDeliveryPaths`
    - `initializeFileStorage(appConfig)` (`packages/api/src/app/cdn.ts`): initialises S3, Firebase, Azure Blob or CloudFront for `fileStrategy` plus every entry in `fileStrategies`.
14. Deployment plugins and skills are initialised:
    - `initializeDeploymentPlugins`
    - `setPluginHookSource`
    - `initializeDeploymentSkills`
    - `initializeGitHubSkillSync(appConfig)`
    - `startExpiredFileSweep`
    - `loadToolApprovalHooks(endpoints.agents.toolApproval.hooks)`, only if `toolApproval.enabled`.
15. Inside `runAsSystem`:
    - `performStartupChecks(appConfig)` (`packages/api/src/app/checks.ts`): env var deprecation checks, the credential DB check, Azure variables, interface config, config `version`, web-search config, and `handleRateLimits`, which copies `rateLimits` from YAML into `*_IP_MAX`-style env vars. It also pings `RAG_API_URL/health`.
    - `updateInterfacePermissions(...)` (`packages/api/src/app/permissions.ts`): syncs the `interface.*` YAML flags into Role documents.
16. `client/dist/index.html` is read and prepared:
    - `<base href>` is rewritten from the `DOMAIN_CLIENT` path.
    - The footer bootstrap from `CUSTOM_FOOTER` is injected.
    - The CSP policy and shell cache headers are set.
    - `sendIndexHtml` adds the `lang` attribute, the query-devtools bootstrap and the CSP nonce.
17. Health routes: `/health` and `/livez` always return 200. `/readyz` returns 503 until `serverReady`.
18. The global middleware pipeline and routes are mounted (§2, §3).
19. `configureGenerationStreams()`:
    - `createStreamServices()` picks Redis job store plus pub/sub transport, or in-memory, based on `USE_REDIS_STREAMS`.
    - `GenerationJobManager.configure/initialize` sets `cleanupOnComplete = !STREAM_KEEP_COMPLETED_JOBS`.
    - Two shutdown tasks are registered: a pre-drain `prepareForShutdown` and a post-drain `destroy`, which keeps a 10 s reserve.
20. `app.listen(port, host, cb)`. The post-listen callback does the following; any failure calls `process.exit(1)`:
    - `runAsSystem`: `initializeMCPs()` and `initializeOAuthReconnectManager()`
    - `checkMigrations()`
    - optional `memoryDiagnostics.start()` (`--inspect` or `MEM_DIAG`)
    - `initializeAgentTriggerService({address, completionResultBatchSize})`
    - `initializeScheduleEngine()`, which sets `scheduleEngineState` to `armed` or `unavailable` (the latter is permanent for the process)
    - `serverReady = true`
21. `configureServerTimeouts(server)` (`packages/api/src/app/server.ts`) applies `HTTP_KEEP_ALIVE_TIMEOUT_MS`, `HTTP_KEEP_ALIVE_TIMEOUT_BUFFER_MS`, `HTTP_HEADERS_TIMEOUT_MS` and `HTTP_REQUEST_TIMEOUT_MS`, clamping headers timeout to be ≤ request timeout.
22. `setupGracefulShutdown(server)` (`packages/api/src/app/shutdown.ts`) wires SIGTERM, SIGINT, SIGQUIT and SIGHUP. On a signal:
    - `server.close()` starts.
    - **pre-drain** tasks run, sorted by priority descending then registration order.
    - It awaits the HTTP drain.
    - **post-drain** tasks run.
    - `process.exit(0)`, or 1 on any failure.
    - A hard force-exit fires after 60 s. `getRemainingShutdownMs()` exposes the remaining budget.
    - All cleanup must go through `registerShutdownTask(name, fn, {phase, priority})`, never raw signal handlers.

**Process-level handlers**
- `startServer().catch` exits with code 1 (fail fast).
- `uncaughtException` swallows a set of known cases: abort, `GoogleGenerativeAI`, `fetch failed` (treated as Meili unavailable), OpenAIError, and errors whose stack is in `@librechat/agents`. Everything else exits unless `CONTINUE_ON_UNCAUGHT_EXCEPTION` is set.
- `unhandledRejection` logs and continues.

**Readiness gates**
- `rejectChatStartsUntilReady` returns 503 with `Retry-After: 1` and code `SERVER_NOT_READY` for `POST /api/agents/chat/*` (except `/abort`) until the server is ready.
- `rejectScheduleWritesUntilReady` (`createScheduleWriteGate`) does the same for `/api/schedules` writes.

**`api/server/experimental.js`** (`npm run backend:experimental`) runs the same bootstrap under Node `cluster`:
- The primary process runs `FLUSHALL` on Redis (single node or `Redis.Cluster`), then forks `CLUSTER_WORKERS` workers (default 4) to simulate multiple pods.
- It picks one listening worker for the file-retention sweep via IPC `{type:'file-retention-sweep-worker'}`, using `createClusteredFileSweep`.
- It propagates a shutdown deadline and force-exits the cluster 10 s after a signal.
- It lags `index.js`: there is no `/metrics`, `/api/rum`, `/api/traces`, most `/api/admin/*` routes, no telemetry, and no readiness gates. Treat it as a dev harness only.

**Other bootstrap files**
- **`api/server/cleanup.js`**: memory hygiene for agent clients. `disposeClient(client)` nulls large graph/run properties (lists `graphPropsToClean` and `graphRunnablePropsToClean`). A `FinalizationRegistry` does debug logging, and `requestDataMap` is a WeakMap. `processReqData` merges per-request stream data. In Python this is largely unnecessary; just drop references.
- **`api/server/socialLogins.js`**: runs only when `ALLOW_SOCIAL_LOGIN=true`.
  - Registers passport strategies based on env var presence: Google, Facebook, GitHub, Discord and Apple, each with a user strategy plus a `*Admin` variant. OAuth state is signed with `JWT_SECRET`, and its TTL comes from `registration.oauthStateTtlMs`.
  - OpenID is enabled when `OPENID_CLIENT_ID`, `OPENID_ISSUER`, `OPENID_SCOPE` and `OPENID_SESSION_SECRET` are set, plus either `OPENID_CLIENT_SECRET` or `OPENID_USE_PKCE`. It mounts `express-session`, using the `OPENID_SESSION` store from `getLogStores`, plus `passport.session()`. `registerOpenIdWithRetry` registers the `openidJwt` strategy, with retries from `OPENID_DISCOVERY_RETRY_*` or `registration.openidDiscovery`. With `OPENID_REUSE_TOKENS`, session max age is at least `OPENID_REUSE_MAX_SESSION_AGE_MS` (default 15 min).
  - SAML is enabled when `SAML_ENTRY_POINT`, `SAML_ISSUER`, `SAML_CERT` and `SAML_SESSION_SECRET` are set, and uses the `SAML_SESSION` store.
  - Session cookie max age is `SESSION_EXPIRY`; the `secure` flag comes from `shouldUseSecureCookie()`.

---

## 2. Route mounting table (`api/server/index.js:321-464`; routers from `api/server/routes/index.js`)

These are mounted in this order after the global middleware (§3.1). Router files are in `api/server/routes/`.

| # | Prefix | Extra mount-level middleware | Router file |
|---|---|---|---|
| – | `GET /health`, `/livez`, `/readyz` | none (before the pipeline) | inline |
| – | `GET /index.html` | – | inline `sendIndexHtml` |
| – | static `dist`, `fonts`, `assets` | `utils/staticCache` | `api/server/utils/staticCache.js` |
| 1 | `/oauth` | `preAuthTenantMiddleware` | `oauth.js` |
| 2 | `/api/auth` | `preAuthTenantMiddleware` | `auth.js` |
| 3 | `/api/insights` | – | `insights.js` |
| 4 | `/api/admin` | – | `admin/auth.js` |
| 5 | `/api/admin/config` | (router does `requireJwtAuth` + `requireCapability(ACCESS_ADMIN)`) | `admin/config.js` |
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
| 31 | `/api/config` | `preAuthTenantMiddleware`, `optionalJwtAuth` | `config.js` |
| 32 | `/api/assistants` | – | `assistants/` |
| 33 | `/api/files` | – (async `routes.files.initialize()`) | `files/` |
| 34 | `/images/` | `createValidateImageRequest({secureImageLinks})` | `static.js` |
| 35 | `/api/share` | `preAuthTenantMiddleware` | `share.js` |
| 36 | `/api/roles` | – | `roles.js` |
| 37 | `/api/agents/chat` | `rejectChatStartsUntilReady` (gate only) | – |
| 38 | `/api/agents` | – | `agents/index.js` (sub-mounts: `/v1/responses` → `responses.js`, `/v1/agents` → `management.js`, `/v1/skills` → `skills.js`, `/v1` → `openai.js` (API-key auth); then `requireJwtAuth`; `/chat/*` → `chat.js`; `/` → `v1.js`) |
| 39 | `/api/banner` | – | `banner.js` |
| 40 | `/api/memories` | – | `memories.js` |
| 41 | `/api/schedules` | `rejectScheduleWritesUntilReady` | `schedules.js` |
| 42 | `/api/permissions` | – | `accessPermissions.js` |
| 43 | `/api/tags` | – | `tags.js` |
| 44 | `/api/mcp` | – | `mcp.js` |
| 45 | `/api/rum` | – | `rum.js` |
| 46 | `/metrics` | `metricsRouter` (Bearer `METRICS_SECRET`, `timingSafeEqual`, 401 otherwise) | `packages/api/src/app/metrics.ts` |
| 47 | `/api` | – | `openapi.js` (served only if YAML `openapi.enabled`) |
| 48 | `/api` | `apiNotFound` (JSON 404) | `@librechat/api` |
| 49 | `*` | SPA fallback → `sendIndexHtml` | `api/server/utils/fallback.js` |
| 50 | error | `telemetryErrorMiddleware` (if OTel), then `ErrorController` | `packages/api/src/middleware/error.ts` |

Most routers apply `requireJwtAuth` themselves, either at router level or per route. Authentication is not global.

---

## 3. Middleware catalog

### 3.1 Global pipeline (in order, `api/server/index.js:204-391`)
1. `createSecurityHeaders()` (`packages/api/src/security/headers.ts`) applies helmet with CSP disabled. Env vars: `SECURITY_HEADERS`, `HSTS_ENABLED`, `HSTS_MAX_AGE` (default 31536000), `HSTS_INCLUDE_SUBDOMAINS`, `HSTS_PRELOAD`, `X_FRAME_OPTIONS`, `CROSS_ORIGIN_OPENER_POLICY`, `CROSS_ORIGIN_RESOURCE_POLICY`, `REFERRER_POLICY`.
2. `requestContextMiddleware` (`packages/api/src/middleware/tenant.ts:91`) generates a `requestId` (UUID) and runs the rest of the request inside `tenantStorage.run({requestId, requestMethod, requestPath})`, an AsyncLocalStorage.
3. `agentStartupIngressMiddleware`, on `/api/agents/chat` only: agent startup latency milestones.
4. `metricsMiddleware`: Prometheus HTTP, SSE and upload metrics.
5. `noIndex`: sets `X-Robots-Tag: noindex` unless `NO_INDEX=false`.
6. `express.json({limit:'3mb'})` and `express.urlencoded({limit:'3mb'})`, then `handleJsonParseError`.
7. A shim that makes `req.query` writable (Express 5), then `express-mongo-sanitize`, which strips `$` and `.` keys.
8. `cors()` (wide open), `cookieParser()`, and `compression()` unless `DISABLE_COMPRESSION`.
9. Static file serving (§2).
10. `telemetryMiddleware` (OTel span enrichment), and `agentStartupTelemetryMiddleware` on `/api/agents/chat`.
11. `passport.initialize()` with strategies `jwt` (`jwtLogin`), `local` (`passportLogin`), `ldap` (if `LDAP_URL` and `LDAP_USER_SEARCH_BASE` are set), and the social strategies plus session middleware.
12. `capabilityContextMiddleware` (`packages/api/src/middleware/capabilities.ts:90`) sets up a per-request AsyncLocalStorage `capabilityStore` that memoises principals and capability results.

### 3.2 `api/server/middleware/*`

**Authentication**
| Middleware | Purpose |
|---|---|
| `requireJwtAuth.js` | Passport `jwt` (Bearer access token). If the cookie `token_provider=openid` and `OPENID_REUSE_TOKENS` is on, tries `openidJwt` first and falls back to `jwt`; checks the `openid_user_id` cookie matches. Sets `req.user` and `req.authStrategy`, chains `tenantContextMiddleware`, then refreshes CloudFront signed cookies. Returns 401 with JSON `{message, code?:'ACCOUNT_DELETION_IN_PROGRESS'}`. Also exports `requireRumProxyAuth`, which silently answers 204 on failure. |
| `optionalJwtAuth.js` | Same strategies, but no failure; sets `req.user` and tenant context when a token is valid. |
| `requireLocalAuth.js` / `requireLdapAuth.js` | Passport `local` / `ldapauth` for login. |
| `setTwoFactorTempUser.js` | Decodes the temporary 2FA token into `req.user`. |
| `optionalShareFileAuth.js` | For shared-file `<img>` requests: resolves the viewer from the `refreshToken` cookie or an OpenID session plus the signed `openid_user_id` cookie. |
| `requireSameOrigin.js` | `createSameOriginGuard` with trusted origins `DOMAIN_CLIENT`, `DOMAIN_SERVER`, `ADMIN_PANEL_URL` (CSRF defence for cookie-authenticated routes). |
| `oauthNavigation.js` | `markOAuthNavigation` flags OAuth browser navigations so rejections redirect to login instead of returning JSON. |

**Abuse and rate limiting**
| Middleware | Purpose |
|---|---|
| `checkBan.js` | If `BAN_VIOLATIONS` is set: looks up the IP and user (or the user found by `req.body.email`) first in a Mongo-backed Keyv ban cache (keys `ban_cache:ip:<ip>` / `ban_cache:user:<id>` with Redis, otherwise raw values), then in `getLogStores(BAN)`. Expired bans are deleted. Banned requests get 403 JSON, a `denyRequest` SSE for interactive agent chat, or an OAuth redirect. |
| `uaParser.js` | Rejects non-browser User-Agents and logs a `NON_BROWSER` violation (score `NON_BROWSER_VIOLATION_SCORE`, default 20). |
| `limiters/*` | See §3.3. |
| `moderateText.js` | If `OPENAI_MODERATION` is set: calls `OPENAI_MODERATION_REVERSE_PROXY` or OpenAI `/v1/moderations` with `OPENAI_MODERATION_API_KEY` on the text, quotes and merged text. Flagged content is rejected through `denyRequest`. |

**Chat request validation**
| Middleware | Purpose |
|---|---|
| `validateModel.js` | Checks `req.body.model` against `getModelsConfig(req)`. A mismatch logs an `ILLEGAL_MODEL_REQUEST` violation (score `ILLEGAL_MODEL_REQ_SCORE`). It is currently commented out in the agents chat chain. |
| `buildEndpointOption.js` | Parses the body with `parseCompactConvo`, applies the model-spec preset (`applyModelSpecPreset` / `resolveModelSpecForEndpoint`), runs a content-filter check on `{{current_user}}` prompt substitutions, calls the endpoint's `buildOptions` (agents / assistants / azureAssistants), sets `req.body.endpointOption`, and updates file usage. |
| `validate/convoAccess.js` | Confirms the user owns `conversationId`. Results are cached in the `CONVO_ACCESS` violation cache; violations use `CONVO_ACCESS_VIOLATION_SCORE`. |
| `validate/subagentThreadTurn.js` | `createSubagentThreadTurnGuard`: guards turns on subagent threads. |
| `messageValidation.js` / `validateMessageReq.js` | `createMessageRequestMiddleware`: validates message routes (`canReadActiveJobConversation`, `prepareMessageRequestValidation`, `sendValidationResponse`). |

**Streaming, abort and errors**
| Middleware | Purpose |
|---|---|
| `abortMiddleware.js` | `abortMessage`: aborts a generation via `GenerationJobManager` (streamId equals conversationId) and persists partial content plus metadata. |
| `abortRun.js` | Aborts OpenAI Assistants runs. |
| `setHeaders.js` | SSE headers: `text/event-stream`, `Cache-Control: no-cache, no-transform`, `Connection: keep-alive`, `X-Accel-Buffering: no`, `ACAO: *`. |
| `denyRequest.js` | Sends an SSE error or final message, optionally saving the user message (used for ban and moderation). |
| `error.js` | `sendError` / `handleAbortError`: saves the error message and sends it as SSE. |

**Configuration and roles**
| Middleware | Purpose |
|---|---|
| `config/app.js` (`configMiddleware`) | `req.config = await getAppConfig(getAppConfigOptionsFromUser(req.user))` for per-user, role, group and tenant merged config. On failure it falls back to `{tenantId}` only. |
| `roles/admin.js` (`checkAdmin`) | 403 unless `req.user.role === ADMIN`. |
| `roles/capabilities.js` | `generateCapabilityCheck`: `hasCapability`, `requireCapability(cap)`, `hasConfigCapability`, `getHeldCapabilities`, `getReadableConfigSections`. These use the SystemGrant model and principals (user, role, groups). Import directly to avoid a circular require. |

**Resource access (ACL, `accessResources/*`)**
| Middleware | Purpose |
|---|---|
| `canAccessResource` | Generic ACL-bit check (`PermissionBits` VIEW/EDIT/DELETE/SHARE) on AclEntry for resource type and id. |
| `canAccessAgentResource`, `canAccessAgentFromBody` | Agent ACL from a URL param or `req.body.agent_id`. |
| `canAccessPromptGroupResource`, `canAccessPromptViaGroup` | Prompt and prompt-group ACL. |
| `canAccessMCPServerResource`, `canAccessSkillResource` | MCP server and skill ACL. |
| `fileAccess` | File ACL. |

**Sharing and people picker**
| Middleware | Purpose |
|---|---|
| `canAccessSharedLink.js` | `createSharedLinkAccessMiddleware({mongoose})`. |
| `checkSharePublicAccess.js` | `createSharePolicyMiddleware({getRoleByName, hasCapability})`. |
| `checkPeoplePickerAccess.js` | `createPeoplePickerAccess({getRoleByName})`. |

**Registration and account**
| Middleware | Purpose |
|---|---|
| `checkDomainAllowed.js` | Enforces `registration.allowedDomains` for social logins. |
| `checkInviteUser.js` | Validates the invite token on registration. |
| `validateRegistration.js` | Gates registration on `ALLOW_REGISTRATION`. |
| `validatePasswordReset.js` | Gates password reset on `ALLOW_PASSWORD_RESET`. |
| `validateEmailLogin.js` | Gates email login on `ALLOW_EMAIL_LOGIN`. |
| `canDeleteAccount.js` | Account-deletion permission check. |

**Other**
| Middleware | Purpose |
|---|---|
| `validateImageRequest.js` | Authorises `/images/*` using the JWT or `secureImageLinks`. |
| `assistants/validate.js`, `assistants/validateAuthor.js` | Assistant allow/deny lists and ownership. |
| `logHeaders.js` | Debug logging of forwarded headers. |
| `noIndex.js` | Above. |

### 3.3 Rate limiters (`api/server/middleware/limiters/*`, express-rate-limit)

The store is `limiterCache(prefix)`: a `rate-limit-redis` `RedisStore` using ioredis when `USE_REDIS=true`, otherwise the in-process default. On-wire keys look like `{REDIS_KEY_PREFIX}::{prefix}:{key}`. Windows are in minutes unless noted. Exceeding a limit calls `logViolation`.

| Limiter | Keying | Env vars (defaults) | Violation type |
|---|---|---|---|
| `loginLimiter` | IP (`removePorts`) | `LOGIN_WINDOW`=5, `LOGIN_MAX`=7, `LOGIN_VIOLATION_SCORE` | `logins` |
| `registerLimiter` | IP | `REGISTER_WINDOW`=60, `REGISTER_MAX`=5 | `registrations` |
| `resetPasswordLimiter` / `...SubmissionLimiter` | IP | `RESET_PASSWORD_WINDOW`=2, `_MAX`=2 (the submission limiter has its own `_SUBMISSION_` vars) | `reset_password_limit` |
| `verifyEmailLimiter` / `...SubmissionLimiter` | IP | `VERIFY_EMAIL_WINDOW`=2, `_MAX`=2 | `verify_email_limit` |
| `twoFactorTempLimiter` (IP + user) | IP; user id from the temp JWT | `TWO_FACTOR_TEMP_WINDOW`/`MAX` (default to the LOGIN_* values) | `logins` |
| `messageIpLimiter` / `messageUserLimiter` | IP / `req.user.id` | `MESSAGE_IP_MAX`=40, `MESSAGE_IP_WINDOW`=1, `MESSAGE_USER_MAX`=40, `MESSAGE_USER_WINDOW`=1; enabled by `LIMIT_MESSAGE_IP` / `LIMIT_MESSAGE_USER` | `message_limit` |
| `agentEventUserLimiter` | `req.apiKeyId ?? user.id` | `AGENT_EVENT_USER_MAX`=40, `_WINDOW`=1; returns 429 with `Retry-After` | – |
| `generationRetryProbeLimiter` / `generationRetryLimiter` | (packages/api) | active when message limits are on | – |
| file upload IP / user | IP / user | `FILE_UPLOAD_IP_MAX`=100/15 min, `FILE_UPLOAD_USER_MAX`=50/15 min; plus file usage `FILE_USAGE_USER_MAX`=120/15 min | `file_upload_limit` |
| import IP / user | IP / user | `IMPORT_IP_MAX`=100/15, `IMPORT_USER_MAX`=50/15 | `file_upload_limit` |
| fork IP / user | IP / user | `FORK_IP_MAX`=30/1, `FORK_USER_MAX`=7/1 | `file_upload_limit` |
| share IP / user | IP / user (the user limiter is skipped when anonymous) | `SHARE_IP_MAX`=100/1, `SHARE_USER_MAX`=60/1, `SHARE_VIOLATION_SCORE`=0 | `share_limit` |
| TTS / STT IP and user | IP / user | `TTS_IP_MAX`=100/1, `TTS_USER_MAX`=50/1 (STT the same) | `tts_limit` / `stt_limit` |
| `toolCallLimiter` | user | 1 request per 1000 ms | `tool_call_limit` |
| `promptUsageLimiter` | user | 30 per minute, fixed | – |

YAML `rateLimits.{fileUploads, conversationsImport, tts, stt, agentEvents}` overwrite the matching env vars at startup (`packages/api/src/app/limits.ts`).

The concurrent-message limit (`LIMIT_CONCURRENT_MESSAGES`, `CONCURRENT_MESSAGE_MAX`) uses the `PENDING_REQ` cache and is cleaned up by `api/cache/clearPendingReq.js`; it lives in `packages/api/src/middleware/concurrency.ts`.

### 3.4 Typical chat request chain: `POST /api/agents/chat/:endpoint?`
1. Global pipeline (§3.1). `rejectChatStartsUntilReady`.
2. `agents/index.js` router: `requireJwtAuth` (which includes `tenantContextMiddleware`), then trigger and schedule context capture (`isAgentTriggerRequest`, `captureScheduleFireContext`), then `checkBan`, then `uaParser`.
3. `chatRouter` (`agents/index.js:1138-1173`): `configMiddleware` (sets `req.config`). If message limits are enabled: `generationRetryProbeLimiter`, `detectGenerationRetry`, `generationRetryLimiter`, `messageIpLimiter` and `messageUserLimiter`, each with exemptions for triggers, schedules and confirmed retries.
4. `chat.js` router: `restoreResumeContext` (for `/resume`), `createMessageFilterPii`, `moderateText`, `checkAgentAccess` (role permission `AGENTS.USE`), `canAccessAgentFromBody(VIEW)`, `validateConvoAccess`, `guardSubagentThreadTurn`, `buildEndpointOption`.
5. `AgentController(req,res,next, initializeClient, addTitle)` starts a `GenerationJobManager` job. The client then subscribes to SSE at `GET /api/agents/chat/stream/:streamId`, which has no rate limiter. Abort is `POST /api/agents/chat/abort`.

---

## 4. Configuration system

### 4.1 Loading `librechat.yaml`
File: `api/server/services/Config/loadCustomConfig.js`.
1. The path is `CONFIG_PATH`, defaulting to `<root>/librechat.yaml`. An `http(s)` URL is fetched with axios.
2. The file is parsed with `loadYaml` / `js-yaml`.
3. `setMaxSubagents(endpoints.agents.maxSubagents)` runs before validation.
4. `configSchema.strict().safeParse(...)` validates it (zod, `packages/data-provider/src/config.ts:2957`). On failure the process exits, unless `CONFIG_BYPASS_VALIDATION=true`, in which case it returns `null` and runs on defaults.
5. The config is logged with secrets redacted by `redactConfigSecretMaps`.
6. Post-processing: OpenRouter defaults (`promptCache` param) and `customParams` validation against `paramSettings`.

Then `loadBaseConfig` (`api/server/services/Config/app.js`) calls `loadAndFormatTools` (filtered by `filteredTools` / `includedTools`), then `AppService({config, paths, systemTools})` (`packages/data-schemas/src/app/service.ts`). AppService normalises the raw config into an **AppConfig**:
- `ocr`, `webSearch`, `memory`, `summarization`, `skillSync`, `langfuse` and `filters` are validated.
- `interfaceConfig` comes from `loadDefaultInterface`, `turnstileConfig` from `loadTurnstileConfig`, `mcpConfig` from `mcpServers`, and endpoints from `loadEndpoints`.
- `fileStrategy` is set, and `process.env.CDN_PROVIDER` is set from it.
- `balance` falls back to `CHECK_BALANCE` / `START_BALANCE`.
- Other fields: `transactions`, `registration`, `imageOutputType`, `secureImageLinks` (default true), `availableTools`, `paths`, and the raw `config`.

`api/config/paths.js` supplies `paths`: root, uploads, clientPath, dist, publicPath, fonts, assets, imageOutput, structuredTools, pluginManifest.

### 4.2 Top-level `librechat.yaml` keys (configSchema)
- **General:** `version` (required; checked against `Constants.CONFIG_VERSION`), `cache` (bool), `permissions.maxWriteAttempts`, `openapi.enabled`.
- **Feature sections:** `ocr`, `webSearch`, `langfuse`, `memory`, `summarization`, `skillSync`.
- **Tools:** `includedTools`, `filteredTools`.
- **Images:** `secureImageLinks`, `imageOutputType` (png, jpeg, webp).
- **MCP:** `mcpServers`, and `mcpSettings.{allowedDomains, allowedAddresses, catalogRecovery{discoveryBackoffMs, discoveryTimeoutMs, reauthRetryMs, authorizationFence*...}}`.
- **UI and auth:** `interface` (feature toggles, which also drive role permissions), `turnstile`, `registration.{socialLogins, allowedDomains, oauthStateTtlMs, openidDiscovery}`.
- **File storage:** `fileStrategy` (local, s3, firebase, azure_blob, cloudfront), `fileStrategies` (per file type), `cloudfront`, `fileConfig`.
- **Actions:** `actions.{allowedDomains, allowedAddresses}`.
- **Billing:** `balance`, `transactions`.
- **Speech:** `speech.{tts, stt, speechTab}`.
- **Limits:** `rateLimits`.
- **Model specs:** `modelSpecs`.
- **Content filtering:** `filters` (base-only security boundary), `messageFilter` (PII regex, RE2).
- **Endpoints:** `endpoints.{allowedAddresses, all, openAI, google, anthropic, azureOpenAI, azureAssistants, assistants, agents, custom[], bedrock}`, strict, with at least one key. Resolution order is `all`, then the named endpoint, then a custom endpoint's own config.

### 4.3 Per-tenant, role, group and user overrides
- **DB model:** `Config` (`packages/data-schemas/src/schema/config.ts`), tenant-isolated. Fields: `principalType` (user, role, group…), `principalId`, `principalModel`, `priority`, `overrides` (a Mixed subset of YAML using YAML key names), `tombstones` (a list of dotted paths to delete), `isActive`, `configVersion`, `tenantId`. There is a unique index on (principalType, principalId, tenantId).
- **Special principal:** the tenant base is `{principalType: ROLE, principalId: '__base__'}` (`BASE_CONFIG_PRINCIPAL_ID`).
- **Service:** `createAppConfigService` (`packages/api/src/app/service.ts`):
  - `ensureBaseConfig()` caches the YAML AppConfig under key `_BASE_` in the `APP_CONFIG` cache, with no TTL, and pushes `availableTools` into the tool cache.
  - `getAppConfig({role, userId, idOnTheSource, tenantId, refresh, baseOnly, failClosed, resolvedPrincipals, skipRuntimeAugmentation})`:
    - `baseOnly` returns the base config and is used at startup and in auth strategies.
    - Otherwise it resolves principals with `getUserPrincipals` (user, role, groups).
    - Cache key: `_OVERRIDE_:{tenant}:{role}:{userId}` (variants for role-only, user-only, `__base__`). The tenant falls back to the ALS tenant, then `__default__`.
    - On a miss it calls `getApplicableConfigs(principals)`, a Mongo query for `__base__` plus the principals where `isActive`, sorted by priority ascending.
    - `mergeConfigOverrides` (`packages/data-schemas/src/app/resolution.ts:284`) applies each document in priority order: tombstones first, then a deep merge of `overrides`.
      - YAML keys are remapped: `mcpServers` → `mcpConfig`, `interface` → `interfaceConfig`, `turnstile` → `turnstileConfig`.
      - `endpoints.custom` arrays are merged by `name`. MCP server overrides are filtered.
      - `filters` is base-only and cannot be overridden. `langfuse` can only be set by the `__base__` principal.
      - Merge depth is limited, and the `__proto__` family of keys is rejected.
    - The merged result is cached with a 60 s TTL (`DEFAULT_OVERRIDE_CACHE_TTL`).
    - `augmentConfig` then merges accessible code environments per user.
    - In strict tenant mode with no tenant available, it returns the base config uncached.
  - `clearAppConfigCache()` deletes `_BASE_`. `clearOverrideCache(tenantId?)` enumerates keys only for the in-memory store; with Redis it relies on the TTL, deliberately avoiding SCAN.
  - `invalidateConfigCaches(tenantId)` (`api/server/services/Config/app.js:77`) clears the base, override, tool and MCP-config caches after admin mutations.
- **Auth-flow helper:** `resolveAppConfigForUser` (`packages/api/src/app/resolve.ts`) runs `getAppConfig({role, tenantId})` inside `tenantStorage.run({tenantId})` when the user has a tenant, otherwise returns the base config.
- **Admin API:** `/api/admin/config` (`api/server/routes/admin/config.js`, handlers in `createAdminConfigHandlers`), requiring the `ACCESS_ADMIN` capability.
  - `GET /` and `GET /base`
  - `GET|PUT|DELETE /:principalType/:principalId`
  - `PATCH /:principalType/:principalId/fields`
  - `POST /:principalType/:principalId/fields/tombstone`
  - `DELETE /:principalType/:principalId/fields`
  - `PATCH /:principalType/:principalId/active`
  - Section-level read and write capabilities are enforced (`hasConfigCapability`, `getReadableConfigSections`).
- **Helpers in `packages/api/src/app/config.ts`:** `getBalanceConfig`, `getTransactionsConfig` (forces transactions on when balance is on), `getCustomEndpointConfig` (normalised name match plus secret resolution), `getEndpointsDropParamsMap`.
- **Build info:** `packages/api/src/app/build.ts` provides `resolveBuildInfo()` from `BUILD_COMMIT`, `BUILD_BRANCH` and `BUILD_DATE`, falling back to git. It is exposed through `/api/config` when `interface.buildInfo` is set.

### 4.4 Database layer
- **`api/db/connect.js`:** mongoose connection cached on `global.mongoose`, with `bufferCommands:false` and `strictQuery`. Env vars: `MONGO_URI` (required), `MONGO_MAX_POOL_SIZE`, `MONGO_MIN_POOL_SIZE`, `MONGO_MAX_CONNECTING`, `MONGO_MAX_IDLE_TIME_MS`, `MONGO_WAIT_QUEUE_TIMEOUT_MS`, `MONGO_AUTO_INDEX`, `MONGO_AUTO_CREATE`. Query metrics come from `instrumentMongooseQueryMetrics`.
- **Models:** `packages/data-schemas/src/models/index.ts` `createModels(mongoose)` registers about 45 models, including:
  - User, Token, Session, Balance, Conversation, Message
  - Agent, AgentApiKey, AgentCategory, MCPServer, Role, Action, Assistant
  - File, Banner, Key, PluginAuth, Transaction, Preset, Prompt, PromptGroup
  - Skill, SkillFile, SkillSync*, ConversationTag, Config, AccessRole, AclEntry, Group, SystemGrant
  - SharedLink, ToolCall, Memory, AuditLog, Schedule / ScheduleRun, QueuedTurn, trigger delivery, lane and purge models
  - OpenIDRefreshFlight, RefreshTokenBridge, CodeEnvironment, ChatProject, Favorite

  Each `createXModel` applies `applyTenantIsolation(schema)`. Message and Conversation also get the `mongoMeili` plugin (`models/plugins/mongoMeili.ts`), which syncs them to Meilisearch.
- **Methods:** `createMethods(mongoose, deps)` in `packages/data-schemas/src/methods/*`, one file per domain (config, aclEntry, role, user, message, conversation…). It is flattened into `api/models/index.js`.
- **`api/db/indexSync.js`:** enabled by `SEARCH=true`; `MEILI_NO_SYNC` skips indexing. Needs `MEILI_HOST` and `MEILI_MASTER_KEY`; `MEILI_SYNC_THRESHOLD` defaults to 1000.
  - Uses `FlowStateManager` over the `FLOWS` cache with flow id `meili-index-sync` as a distributed single-runner lock (10 min TTL, Redis Lua executor).
  - Health-checks Meili, ensures filterable attributes (resetting `_meiliIndex` flags to force a re-sync when settings change), deletes docs without a `user` field, and syncs unindexed messages and convos past the threshold.

### 4.5 Important env vars (bootstrap and infrastructure only)
- **Server:** `PORT`, `HOST`, `TRUST_PROXY`, `DOMAIN_CLIENT`, `DOMAIN_SERVER`, `NODE_ENV`, `DISABLE_COMPRESSION`, `NO_INDEX`, `CUSTOM_FOOTER`, `HTTP_*_TIMEOUT_MS`, `CONTINUE_ON_UNCAUGHT_EXCEPTION`, `MEM_DIAG`, `CONFIG_PATH`, `CONFIG_BYPASS_VALIDATION`.
- **Secrets:** `CREDS_KEY`, `CREDS_IV`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `SESSION_EXPIRY`, `REFRESH_TOKEN_EXPIRY`, `OPENID_*`, `SAML_*`, and each provider's `*_CLIENT_ID` / `*_CLIENT_SECRET`, `ALLOW_SOCIAL_LOGIN`, `ALLOW_REGISTRATION`, `ALLOW_EMAIL_LOGIN`, `ALLOW_PASSWORD_RESET`, `LDAP_URL`, `LDAP_USER_SEARCH_BASE`.
- **Database and search:** `MONGO_*`, `SEARCH`, `MEILI_*`, `RAG_API_URL`.
- **Redis:** `USE_REDIS`, `REDIS_URI` (comma-separated for a cluster), `USE_REDIS_CLUSTER`, `REDIS_USERNAME`, `REDIS_PASSWORD`, `REDIS_CA`, `REDIS_KEY_PREFIX` or `REDIS_KEY_PREFIX_VAR`, `USE_REDIS_STREAMS`, `FORCED_IN_MEMORY_CACHE_NAMESPACES`, `REDIS_PING_INTERVAL` and the other tuning vars listed in §5.
- **Abuse:** `BAN_VIOLATIONS`, `BAN_DURATION` (default 2 h), `BAN_INTERVAL` (default 20), `VIOLATION_SCORE_TTL` (default 1 h), `*_VIOLATION_SCORE`, `LIMIT_*`, the limiter vars in §3.3, `OPENAI_MODERATION*`.
- **Tenancy:** `TENANT_ISOLATION_STRICT`, `TRUST_TENANT_HEADER`.
- **Cluster:** `LEADER_LEASE_DURATION` (25 s), `LEADER_RENEW_INTERVAL` (10 s), `LEADER_RENEW_ATTEMPTS` (3), `LEADER_RENEW_RETRY_DELAY` (0.5 s), `CLUSTER_WORKERS`.
- **Observability:** `METRICS_SECRET`, `OTEL_TRACING_ENABLED`, `OTEL_LOGS_ENABLED`, `OTEL_LOGS_LEVEL`, `OTEL_SDK_DISABLED`, `OTEL_SERVICE_NAME`, `OTEL_SERVICE_VERSION`, `OTEL_IOREDIS_TRACING_ENABLED` plus the standard `OTEL_EXPORTER_*`, `DEBUG_LOGGING`, `DEBUG_CONSOLE`, `CONSOLE_JSON`, `CONSOLE_LOG_LEVEL`, `LOG_TO_FILE`.
- **Security headers:** `SECURITY_HEADERS`, `HSTS_*`, `X_FRAME_OPTIONS`, `REFERRER_POLICY`, `CROSS_ORIGIN_*`.
- **Balance:** `CHECK_BALANCE`, `START_BALANCE`.

---

## 5. Cache and Redis design

**Configuration** (`packages/api/src/cache/cacheConfig.ts`)
- `USE_REDIS` requires `REDIS_URI`.
- `USE_REDIS_STREAMS` defaults to the value of `USE_REDIS`.
- `FORCED_IN_MEMORY_CACHE_NAMESPACES` defaults to `CONFIG_STORE,APP_CONFIG`, so YAML-derived config stays per container. Names are validated against `CacheKeys`.
- The key prefix is `REDIS_KEY_PREFIX`, or the value of the env var named by `REDIS_KEY_PREFIX_VAR`, with separator `::`.
- Tuning vars: `REDIS_MAX_LISTENERS` (40), `REDIS_PING_INTERVAL`, `REDIS_PING_TIMEOUT` (5000), `REDIS_SUBSCRIBER_PING_INTERVAL` (15), `REDIS_KEEP_ALIVE`, `REDIS_RETRY_MAX_DELAY` (3000), `REDIS_RETRY_MAX_ATTEMPTS` (10), `REDIS_CONNECT_TIMEOUT`, `REDIS_READONLY_RECOVERY_INTERVAL`, `REDIS_ENABLE_OFFLINE_QUEUE`, `REDIS_USE_ALTERNATIVE_DNS_LOOKUP` (ElastiCache TLS), `REDIS_CLUSTER_SAFE_DELETE`, `REDIS_DELETE_CHUNK_SIZE`, `REDIS_UPDATE_CHUNK_SIZE`, `REDIS_SCAN_COUNT` (1000), `MCP_REGISTRY_CACHE_TTL` (5000).

**Clients** (`redisClients.ts`)
- There are two client libraries:
  1. **ioredis** (`ioredisClient`), used for rate limits, sessions, leader election, principal locks and stream pub/sub. It is `Redis` or `Redis.Cluster`, with `keyPrefix = PREFIX::`, exponential backoff with jitter, reconnect on `READONLY`, and TLS with the CA.
  2. **node-redis via @keyv/redis** (`keyvRedisClient`), used for Keyv caches. It is `createClient` or `createCluster`; a `scanIterator` shim is added for clusters.
- `heartbeat.ts` sends deadline-bounded PINGs per node and destroys sockets that do not answer.
- `recovery.ts` forces a reconnect after a failover `READONLY` reply.
- `redisUtils.ts` provides cluster-safe batch delete, SCAN, and a duplicated subscriber.
- `redisScript.ts` provides EVALSHA with an EVAL fallback.
- `redisTelemetry.ts` records `redis_operations_total` and duration by use case.

**Factories** (`cacheFactory.ts`)
- `standardCache(ns, ttl?, fallback?)` returns a Keyv. With Redis it is Keyv over KeyvRedis, memoised per (namespace, ttl), with a custom namespace-aware `clear()` implemented as SCAN plus DEL. Without Redis it is an in-memory Keyv memoised per namespace, using a JSON serialiser.
- `violationCache(ns)` is `standardCache('violations:'+ns, VIOLATION_SCORE_TTL, violationFile)`. The fallback is the `keyv-file` store `./data/violations.json`.
- `sessionCache(ns)` is `connect-redis` over ioredis with prefix `ns:`, or `memorystore` without Redis.
- `limiterCache(prefix)` is a `rate-limit-redis` store, or undefined without Redis.
- `keyvMongo` (`keyvMongo.ts`) is a Keyv store in the Mongo collection `keyv`.
- `logFile` is `./data/logs.json`.
- `userPrincipalsCache()` (`principals.ts`) is `USER_PRINCIPALS` with TTL `USER_PRINCIPALS_CACHE_TTL_MS` (5 min). When Redis-backed it adds a cross-process build lock: `SET key token PX lockTtl NX`, released by a Lua compare-and-delete (`USER_PRINCIPALS_LOCK_TTL_MS`, `_LOCK_WAIT_MS`).

**Named caches** (`api/cache/getLogStores.js`; `CacheKeys` enum at `packages/data-provider/src/config.ts:~3525`)

| Key | Backing | TTL |
|---|---|---|
| `general` violations | Keyv on the `logs.json` file | – |
| `logins`, `concurrent`, `non_browser`, `message_limit`, `registrations`, `token_balance`, `tts_limit`, `stt_limit`, `convo_access`, `share_limit`, `tool_call_limit`, `file_upload_limit`, `verify_email_limit`, `reset_password_limit`, `illegal_model_request` | `violationCache` | `VIOLATION_SCORE_TTL` |
| `ban` | Keyv on keyvMongo, namespace `BANS` | `BAN_DURATION` |
| `OPENID_SESSION`, `SAML_SESSION` | `sessionCache` | – |
| `ROLES`, `APP_CONFIG`*, `CONFIG_STORE`*, `TOOL_CACHE`, `PENDING_REQ`, `MODEL_QUERIES`, `AUTH_USER_DOC` | `standardCache` (* in-memory by default) | – |
| `USER_PRINCIPALS` | `userPrincipalsCache`, or disabled when the TTL is 0 | 5 min |
| `PROMPT_GROUPS_ACCESS` | deliberately disabled no-op | – |
| `ENCODED_DOMAINS` | keyvMongo | – |
| `ABORT_KEYS` | standard | 10 min |
| `TOKEN_CONFIG` | standard | 30 min |
| `GEN_TITLE` | standard | 2 min |
| `S3_EXPIRY_INTERVAL` | standard | 30 min |
| `AUDIO_RUNS` | standard | 10 min |
| `MESSAGES` | standard | 1 min |
| `FLOWS` | standard | 10 min |
| `OPENID_EXCHANGED_TOKENS` | standard | 10 min |
| `ADMIN_OAUTH_EXCHANGE` | standard | 30 s |

Other `CacheKeys` values (`SANDBOX_PREWARM`, `TOOLS`, `MODELS_CONFIG`, `STARTUP_CONFIG`, `ENDPOINT_CONFIG`) are used via `standardCache` elsewhere. `CODE_ENVIRONMENT_CONFIG` is Redis-only (`Config/app.js`).

Without Redis, `getLogStores` runs a 30 s interval that purges expired in-memory entries, registered as a shutdown task.

**Violations and bans** (`api/cache/logViolation.js`, `banViolation.js`)
1. `logViolation(req,res,type,errorMessage,score)` increments the per-user score in the type's cache. The key is `type:userId` with Redis, otherwise `userId`.
2. It appends the event to the `general` log.
3. It calls `banViolation`. If `BAN_VIOLATIONS` is set and the score crosses a multiple of `BAN_INTERVAL`, it deletes all of the user's sessions, clears the auth cookies (`refreshToken`, `openid_*`, `token_provider`), and writes `{type, violation_count, duration, expiresAt}` to the `ban` store under both the userId and the IP.
4. `checkBan` then enforces the ban.

**Resumable streams** (`packages/api/src/stream/createStreamServices.ts`)
- With Redis: `RedisJobStore` plus `RedisEventTransport` (pub/sub on a duplicated ioredis subscriber). Otherwise in-memory.
- `GenerationJobManager` holds jobs keyed by streamId (which equals conversationId) and supports resume and replay across replicas.

---

## 6. Tenancy and clustering

**Tenancy** (`packages/data-schemas/src/config/tenantContext.ts`, `tenant/policy.ts`, `models/plugins/tenantIsolation.ts`)
- **Context:** AsyncLocalStorage `tenantStorage` holds `{tenantId, userId, requestId, requestMethod, requestPath}`. Accessors: `getTenantId()`, `getUserId()`, `getRequestId()`. `SYSTEM_TENANT_ID='__SYSTEM__'`, and `runAsSystem(fn)` runs cross-tenant system operations. `scopedCacheKey(base)` appends `:tenantId` to cache keys.
- **Where context is set:**
  1. `requestContextMiddleware`: request id only.
  2. `preAuthTenantMiddleware` on `/oauth`, `/api/auth`, `/api/config` and `/api/share`: reads the `X-Tenant-Id` header only if `TRUST_TENANT_HEADER=true`, validates it against `^[-a-zA-Z0-9_.]+$` with a maximum of 128 characters, and rejects `__SYSTEM__`.
  3. `tenantContextMiddleware`, chained by `requireJwtAuth` and `optionalJwtAuth`: takes `tenantId` from `req.tenantId` or `req.user.tenantId`. Returns 403 for `__SYSTEM__`, and 403 when there is no tenant and `TENANT_ISOLATION_STRICT=true`.
- **Mongoose plugin `applyTenantIsolation`:**
  - Injects `{tenantId}` into every find, update, delete and aggregate filter.
  - Stamps `tenantId` on saves and inserts.
  - Blocks mutations of `tenantId` unless running as system.
  - With no tenant in context, strict mode throws (fail closed) and non-strict mode passes the query through (legacy single-tenant behaviour).
  - The policy is engine-neutral in `tenant/policy.ts`, so a Python port can reuse the same rules.
- **Other tenant scoping:** the config override cache keys, and generation jobs, which store `metadata.tenantId` and are checked against `user.tenantId`.

**Clustering / multi-replica** (`packages/api/src/cluster/LeaderElection.ts`, `config.ts`)
- `isLeader()` returns true when Redis is off.
- Otherwise it reads the key `LeadingServerUUID` (prefixed). If this instance owns it, it is leader while its renew timer is still alive. If no one owns it, it waits a random 0–2 s, then `SET key uuid EX 25 NX`.
- Renewal runs every 10 s via a Lua check-and-`EXPIRE`, with 3 attempts and a 0.5 s delay. After that it gives up leadership to avoid split-brain.
- `resign()` on SIGTERM or SIGINT does a Lua compare-and-delete.
- Leader-only work includes MCP registry writes (`mcp/registry/cache/BaseRegistryCache.ts`, `MCPServersInitializer.ts`), the expired-file sweep (`files/sweep.ts`) and the code-environment lifecycle reconciler.
- Other cross-replica coordination:
  - `FlowStateManager` locks (Meili sync, OAuth flows).
  - Redis-backed resumable streams.
  - The user-principal build lock.
  - `APP_CONFIG` stays per container on purpose. Admin changes rely on the 60 s override TTL plus local invalidation.

---

## 7. Observability
- **Logging:** Winston in `packages/data-schemas/src/config/winston.ts`, exported as `logger`.
  - Levels: `NODE_ENV=development` logs at debug, otherwise warn.
  - Transports: a DailyRotateFile `error-%DATE%.log`, 20 MB, kept 14 days, unless `LOG_TO_FILE=false`. `debug-%DATE%.log` is added when `DEBUG_LOGGING` is set.
  - Console: colourised, or JSON with `CONSOLE_JSON`. `DEBUG_CONSOLE` enables verbose console output; `CONSOLE_LOG_LEVEL` sets the level.
  - Formats include redaction (`parsers.ts`: `redactFormat`, `redactMessage`), JSON truncation, and `attachRequestContext`, which adds request_id, tenant and user from ALS.
  - Meilisearch has its own logger (`meiliLogger.ts`).
  - Log directory: `getLogDirectory()` in `config/utils.ts`.
- **Metrics:** Prometheus via `prom-client` (`packages/api/src/app/metrics.ts`), served at `/metrics` with `Authorization: Bearer $METRICS_SECRET`.
  - Default process metrics.
  - HTTP and uploads: `http_requests_total`, `http_request_duration_seconds`, `http_requests_in_flight`, `http_request_body_bytes`, `upload_*`.
  - SSE and generation: `sse_streams_total`, `sse_streams_in_flight`, `sse_stream_duration_seconds`, `generation_jobs_total`, `generation_jobs_in_flight`, `generation_stream_*` (subscriptions, recoveries, attachments), `agent_startup_*`.
  - Backends: `mongoose_queries_total`, `mongoose_query_duration_seconds`, `redis_operations_total`, `redis_operation_duration_seconds`.
  - Auth, sharing and RUM: `openid_user_lookup_total`, `share_link_rejections_total`, `rum_proxy_requests_total`, `content_filter_locator_traversal_*`.
  - Agent events: `agent_event_actor_*` gauges, collected on scrape through `runAsSystem`.
  - Paths are normalised by `normalizePath`.
- **Tracing and log export:** OpenTelemetry (`packages/api/src/telemetry/*`).
  - `sdk.ts`: NodeSDK with HTTP, Undici, Express, MongoDB, Mongoose and (optionally) IORedis instrumentations.
  - `middleware.ts`: request span enrichment and error recording.
  - `stream.ts`: `createSseStreamTelemetry` for SSE spans.
  - `logs.ts`: `OpenTelemetryTransportV3` attaches a Winston-to-OTLP log transport.
  - Configured by the `OTEL_*` env vars; shutdown is registered with the shutdown coordinator.
- **Browser RUM:** `/api/rum` proxies OTLP traces and logs from the browser (`requireRumProxyAuth`).
- **Langfuse:** optional LLM tracing (`packages/api/src/langfuse`, YAML `langfuse`, `/api/admin/langfuse`).
- **Diagnostics:** `memoryDiagnostics` (with `MEM_DIAG`).

---

## 8. Suggested Python equivalents

| Node concern | Python replacement |
|---|---|
| Express app + router mounting | **FastAPI** app with `APIRouter`s using the same prefixes. Use `app.include_router(router, prefix=..., dependencies=[...])` for mount-level middleware such as `preAuthTenant`. |
| Global middleware | Starlette middleware, in order: `secure` or a custom helmet-like header middleware, request-id / ALS (`contextvars.ContextVar`), `prometheus-fastapi-instrumentator` or `prometheus_client`, `GZipMiddleware`, `CORSMiddleware`, a body-size limit, and a Mongo-sanitize equivalent (reject keys starting with `$` or containing `.`, or rely on typed pydantic bodies). |
| Per-route middleware chains | FastAPI `Depends` chains. For example, the chat route: `Depends(require_jwt_auth)` → `tenant_context` → `check_ban` → `ua_parser` → `load_app_config` → rate limiters → `pii_filter` → `moderate_text` → `check_agent_access` → `validate_convo_access` → `build_endpoint_option`. Use a `Request.state` object for `req.user`, `req.config` and so on. |
| passport strategies | `python-jose` / `PyJWT` for JWT; `authlib` for OAuth2/OIDC (Google, GitHub, Discord, Facebook, Apple); `python3-saml` or `pysaml2`; `ldap3`; `passlib[bcrypt]` for local login; `starlette.middleware.sessions` or `starsessions` with a Redis store for OIDC/SAML sessions. |
| AsyncLocalStorage (tenant, capabilities) | `contextvars.ContextVar`, which propagates through asyncio tasks. Implement `run_as_system()` as a context manager. |
| Mongoose + tenant plugin | **Motor/PyMongo** with **Beanie** or ODMantic. Implement tenant isolation in a repository base class that injects `tenantId` into every filter, stamps it on insert, forbids `$set: tenantId`, and fails closed in strict mode. Port `tenant/policy.ts` directly. Create indexes at startup (`MONGO_AUTO_INDEX`). |
| zod `configSchema` + `librechat.yaml` | **pydantic v2** models with `extra='forbid'` for strict mode, loaded with `ruamel.yaml` or `PyYAML` (or `httpx` for a remote `CONFIG_PATH`). Env settings via **pydantic-settings** `BaseSettings`, one class per concern (Server, Mongo, Redis, Auth, RateLimits, OTel). |
| AppConfig service + overrides | A `ConfigService` with an in-process base (`functools` cache, or `cachetools.TTLCache(ttl=60)` for per-principal merges) and the deep-merge, tombstone and key-remap rules ported from `resolution.ts`. |
| Keyv namespaces | A small `Cache` protocol (`get`/`set`/`delete`/`clear`, TTL) with backends `redis.asyncio` (namespaced `PREFIX::NS:key`), in-memory `cachetools.TTLCache`, and a Mongo collection with a TTL index. **aiocache** is an option. |
| ioredis / node-redis / cluster | **redis-py** `redis.asyncio.Redis` / `redis.asyncio.cluster.RedisCluster`, with `retry=Retry(ExponentialBackoff())`, `health_check_interval` (replaces the heartbeat), `socket_keepalive`, and SSL with the CA. Lua via `register_script`. |
| express-rate-limit + rate-limit-redis | **limits** / **slowapi** with the Redis storage, or a custom fixed-window `INCR`+`PEXPIRE` Lua script. Key functions per limiter (IP without port, user id, API key id). Call `log_violation` on exceed. |
| express-session / connect-redis | `starsessions` with a Redis store. |
| Leader election | The same algorithm with redis-py: `SET NX EX 25`, renewal via Lua every 10 s in an asyncio task, resignation on shutdown. Or `redis.lock.Lock`. |
| Resumable SSE streams | `sse-starlette` `EventSourceResponse`, with Redis Streams or pub/sub for cross-replica fan-out and an in-memory fallback. |
| Graceful shutdown registry | FastAPI `lifespan` context. Keep a priority-ordered task registry (pre-drain before uvicorn stops accepting connections, post-drain after), and use uvicorn's `--timeout-graceful-shutdown 60`. |
| Readiness gates | A global `AppState` with `ready` / `schedule_state`, plus a dependency that returns 503 with `Retry-After`. Routes `/health`, `/livez`, `/readyz`. |
| prom-client | **prometheus_client** with the same metric names; `/metrics` behind a bearer dependency using `hmac.compare_digest`. |
| OpenTelemetry NodeSDK | `opentelemetry-sdk` with `opentelemetry-instrumentation-fastapi`, `-httpx`, `-pymongo`, `-redis`, and the OTLP exporters. For logs, the OTel `LoggingHandler`. |
| Winston | `structlog` or stdlib `logging` with `TimedRotatingFileHandler` (error and debug files, 14-day retention), a JSON formatter, a redaction processor, and a `contextvars` processor for request_id / tenant / user. |
| Meilisearch sync | The `meilisearch` Python SDK (or `meilisearch-python-sdk` for async). Run the sync as a background task guarded by a Redis lock; hook message and conversation writes to upsert documents. |
| Node cluster (`experimental.js`) | Not needed: use `uvicorn --workers N` or `gunicorn -k uvicorn.workers.UvicornWorker`, with leader election for singleton jobs. |
| File-backed violation stores (`./data/*.json`) | Replace with the in-memory TTL cache, or skip; they exist only when Redis is off. |
