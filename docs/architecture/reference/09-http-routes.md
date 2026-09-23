# HTTP API Inventory

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

This covers every route in every router file under `api/server/routes/**`, plus the routes defined in `api/server/index.js` itself.

**Paths.** `A` = `api/server`. `P` = `packages/api/src`. Inside the code, `~/` resolves to `api/`.

**Abbreviations used in the tables:**
- **JWT** = `requireJwtAuth` (`A/middleware/requireJwtAuth.js`). It checks a passport `jwt` Bearer token, or `openidJwt` when the `token_provider=openid` cookie is set and `OPENID_REUSE_TOKENS` is on. It then runs `tenantContextMiddleware`.
- **optJWT** = `optionalJwtAuth`.
- **cfg** = `configMiddleware` (`A/middleware/config/app`). It sets `req.config` to the per-user/role/tenant app config.
- **gCA(X:[perms])** = `generateCheckAccess({permissionType: PermissionTypes.X, permissions: [...]})` from `P/middleware/access.ts`. It is a role-permission check.
- **cap(X)** = `requireCapability(SystemCapabilities.X)` (`A/middleware/roles/capabilities`).
- **ADMIN** = `JWT + cap(ACCESS_ADMIN)`.
- **preTenant** = `preAuthTenantMiddleware` (reads `X-Tenant-Id`).
- Responses are JSON unless a row says otherwise.

---

## 1. Summary: prefix → router file → auth → number of routes

| Mount prefix | Router file | Auth | # routes |
|---|---|---|---|
| `/health`, `/livez`, `/readyz` | inline in `A/index.js` | public | 3 |
| `/oauth` | `routes/oauth.js` | public (passport social) | 15 |
| `/api/auth` | `routes/auth.js` | mixed: public / JWT | 14 |
| `/api/insights` | `routes/insights.js` | JWT | 2 |
| `/api/admin` | `routes/admin/auth.js` | mixed: public login and OAuth / ADMIN | 19 |
| `/api/admin/config` | `routes/admin/config.js` | ADMIN | 9 |
| `/api/admin/code-environments` | `routes/admin/code.js` | ADMIN + cap(MANAGE_CODE_ENVIRONMENTS, platformOnly) | 2 |
| `/api/code-environments` | `routes/code-environments.js` | JWT | 7 |
| `/api/admin/langfuse` | `routes/admin/langfuse.js` | ADMIN + langfuse config capability | 4 |
| `/api/admin/grants` | `routes/admin/grants.js` | ADMIN | 5 |
| `/api/admin/groups` | `routes/admin/groups.js` | ADMIN + READ/MANAGE_GROUPS | 8 |
| `/api/admin/roles` | `routes/admin/roles.js` | ADMIN + READ/MANAGE_ROLES | 9 |
| `/api/admin/skills` | `routes/admin/skills.js` | ADMIN + skill-sync caps | 4 |
| `/api/admin/users` | `routes/admin/users.js` | ADMIN + READ_USERS | 2 |
| `/api/admin/audit-log` | `routes/admin/audit.js` | ADMIN + READ_AUDIT_LOG | 4 |
| `/api/actions` | `routes/actions.js` | mixed: JWT / public OAuth callback | 2 |
| `/api/keys` | `routes/keys.js` | JWT | 4 |
| `/api/api-keys` | `routes/apiKeys.js` | JWT + gCA(REMOTE_AGENTS:USE) | 4 |
| `/api/user` (+ `/api/user/settings`) | `routes/user.js`, `routes/settings.js` | JWT; verify-email routes are public | 8 + 9 |
| `/api/search` | `routes/search.js` | JWT | 1 |
| `/api/messages` | `routes/messages.js` | JWT | 9 |
| `/api/convos` | `routes/convos.js` | JWT | 16 |
| `/api/traces` | `routes/traces.js` | JWT | 3 |
| `/api/presets` | `routes/presets.js` | JWT | 3 |
| `/api/projects` | `routes/projects.js` | JWT | 6 |
| `/api/prompts` | `routes/prompts.js` | JWT + gCA(PROMPTS:USE) | 12 |
| `/api/skills` | `routes/skills.js` | JWT + gCA(SKILLS:USE) | 10 |
| `/api/categories` | `routes/categories.js` | JWT | 1 |
| `/api/endpoints` | `routes/endpoints.js` | JWT | 2 |
| `/api/balance` | `routes/balance.js` | JWT | 1 |
| `/api/models` | `routes/models.js` | JWT | 1 |
| `/api/config` | `routes/config.js` | preTenant + optJWT | 1 |
| `/api/assistants` | `routes/assistants/*` | JWT | 23 |
| `/api/files` | `routes/files/*` (built by async `initialize()`) | JWT | 19 |
| `/images/` | `routes/static.js` | `createValidateImageRequest` (secureImageLinks cookie/JWT) | static |
| `/api/share` | `routes/share.js` | preTenant; mixed optJWT / JWT | 11 (6 only when `ALLOW_SHARED_LINKS` is on) |
| `/api/roles` | `routes/roles.js` | JWT | 9 |
| `/api/agents` | `routes/agents/*` | mixed: remote-agent API key/OIDC, M2M OIDC, JWT | 54 |
| `/api/banner` | `routes/banner.js` | optJWT | 1 |
| `/api/memories` | `routes/memories.js` | JWT | 7 |
| `/api/schedules` | `routes/schedules.js` | JWT (+ startup write gate) | 6 |
| `/api/permissions` | `routes/accessPermissions.js` | JWT + checkBan + uaParser | 6 |
| `/api/tags` | `routes/tags.js` | JWT + gCA(BOOKMARKS:USE) | 5 |
| `/api/mcp` | `routes/mcp.js` | JWT; OAuth callback is public | 16 |
| `/api/rum` | `routes/rum.js` | `requireRumProxyAuth` | 2 |
| `/metrics` | `metricsRouter` from `createMetrics()` (`P/app/metrics.ts`) | Bearer `METRICS_SECRET` | 1 |
| `/api` (openapi) | `routes/openapi.js` → `createOpenApiRouter` (`P/openapi/router.ts`) | public (404 unless `config.openapi.enabled`) | 3 |
| `/api/*` fallthrough | `apiNotFound` (`P/middleware/notFound.ts`) | — | 404 JSON `{message:'Endpoint not found'}` |
| `*` | `createSpaFallback(sendIndexHtml)` (`A/utils/fallback`) | — | serves the SPA `index.html` |

Two files are not mounted directly:
- `routes/settings.js` is only reached through `/api/user/settings`.
- `routes/types/assistants.js` contains only JSDoc types, no routes.

`A/experimental.js` is a separate cluster entrypoint (`npm run backend:experimental`). It mounts a subset of the same routers and leaves out adminConfig, langfuse, grants, groups, roles, users, audit, traces, rum, metrics and the agent-chat startup gates.

---

## 2. Global Express middleware, in order (`A/index.js` ~L204–464)

1. `app.disable('x-powered-by')`, then `app.set('trust proxy', Number(TRUST_PROXY) || 1)`
2. `createSecurityHeaders()`, only if it is configured
3. **Routes registered before any other middleware:** `GET /health` → `"OK"`, `GET /livez` → `"OK"`, `GET /readyz` → `"OK"` or 503 `"NOT_READY"` until `serverReady`. All three are plain text.
4. `requestContextMiddleware`
5. `app.use('/api/agents/chat', agentStartupIngressMiddleware)`
6. `metricsMiddleware`
7. `noIndex`
8. `express.json({limit:'3mb'})`
9. `express.urlencoded({extended:true, limit:'3mb'})`
10. `handleJsonParseError`
11. A shim that makes `req.query` writable (Express 5)
12. `mongoSanitize()`
13. `cors()`
14. `cookieParser()`
15. `compression()`, unless `DISABLE_COMPRESSION` is set
16. `GET /index.html` → `sendIndexHtml` (injects lang, CSP nonce, footer bootstrap)
17. `staticCache(dist)`, `staticCache(fonts)`, `staticCache(assets)`
18. `telemetry.telemetryMiddleware`, if telemetry is enabled
19. `app.use('/api/agents/chat', agentStartupTelemetryMiddleware)`
20. `passport.initialize()` with strategies `jwt`, `local`, and `ldap` (when `LDAP_URL` and `LDAP_USER_SEARCH_BASE` are set)
21. If `ALLOW_SOCIAL_LOGIN`: `configureSocialLogins(app)` (`A/socialLogins.js`). This adds `express-session` + `passport.session()` for OpenID (and SAML) and registers the social and admin strategies.
22. `capabilityContextMiddleware`
23. Route mounts, in the order of the table in section 1. Two gates are applied at mount time:
    - `app.use('/api/agents/chat', rejectChatStartsUntilReady)` is placed just before `/api/agents`. It returns 503 `{code:'SERVER_NOT_READY'}` with `Retry-After: 1` for any POST except `/abort` until the server is ready.
    - `/api/schedules` is mounted behind `rejectScheduleWritesUntilReady` (`createScheduleWriteGate`).
24. `/metrics`, then `/api` openapi, then `/api` `apiNotFound`, then the SPA fallback
25. `telemetry.telemetryErrorMiddleware` (if enabled), then `ErrorController`

---

## 3. Route tables

### `/oauth`: `routes/oauth.js`
Router-level middleware: `logHeaders`, `markOAuthNavigation`, `loginLimiter`. Mounted with `preTenant`.
- The callbacks run `passport.authenticate(<strategy>)`, then `setBalanceConfig` (`createSetBalanceConfig`), then `checkDomainAllowed`, then `oauthHandler`.
- `oauthHandler` is `createOAuthHandler()` from `A/controllers/auth/oauth.js`. It sets the auth cookies and redirects to the client.
- Every route here returns a redirect.

| Method | Path | Middleware | Handler (file) | Notes |
|---|---|---|---|---|
| GET | /oauth/error | – | inline; `redirectToAuthFailure` (P) | redirect |
| GET | /oauth/google | passport `google` | passport | redirect to the IdP |
| GET | /oauth/google/callback | passport google, setBalanceConfig, checkDomainAllowed | oauthHandler | |
| GET | /oauth/facebook | passport `facebook` | passport | |
| GET | /oauth/facebook/callback | same chain as google | oauthHandler | |
| GET | /oauth/openid | passport `openid` (randomState) | inline | |
| GET | /oauth/openid/callback | `createOpenIDCallbackAuthenticator` (P), setBalanceConfig, checkDomainAllowed | oauthHandler | |
| GET | /oauth/github | passport `github` | | |
| GET | /oauth/github/callback | same chain as google | oauthHandler | |
| GET | /oauth/discord | passport `discord` | | |
| GET | /oauth/discord/callback | same chain as google | oauthHandler | |
| GET | /oauth/apple | passport `apple` | | |
| POST | /oauth/apple/callback | same chain as google | oauthHandler | form POST |
| GET | /oauth/saml | passport `saml` | | |
| POST | /oauth/saml/callback | passport saml (failureMessage) | oauthHandler | no balance or domain check |

### `/api/auth`: `routes/auth.js`
Mounted with `preTenant`. Controllers live in:
- `A/controllers/AuthController.js`
- `A/controllers/TwoFactorController.js`
- `A/controllers/auth/{LoginController, LogoutController, TwoFactorAuthController}.js`

| Method | Path | Middleware | Handler (file) | Notes |
|---|---|---|---|---|
| POST | /api/auth/logout | JWT | logoutController | |
| POST | /api/auth/login | logHeaders, requireSameOrigin, loginLimiter, checkBan, validateEmailLogin, `requireLdapAuth` or `requireLocalAuth`, setBalanceConfig | loginController | public; sets refresh cookies and returns `{token,user}` |
| POST | /api/auth/refresh | – | refreshController | uses the refresh cookie |
| POST | /api/auth/cloudfront/refresh | JWT | inline (`forceRefreshCloudFrontAuthCookies`, P) | 404 if CloudFront is disabled |
| POST | /api/auth/register | registerLimiter, checkBan, checkInviteUser, validateRegistration | registrationController | public |
| POST | /api/auth/requestPasswordReset | resetPasswordLimiter, checkBan, validatePasswordReset | resetPasswordRequestController | public |
| POST | /api/auth/resetPassword | resetPasswordSubmissionLimiter, checkBan, validatePasswordReset | resetPasswordController | public |
| POST | /api/auth/2fa/enable | JWT | enable2FA | |
| POST | /api/auth/2fa/verify | JWT | verify2FA | |
| POST | /api/auth/2fa/verify-temp | requireSameOrigin, setTwoFactorTempUser, twoFactorTempLimiter, checkBan | verify2FAWithTempToken | temp-token auth |
| POST | /api/auth/2fa/confirm | JWT | confirm2FA | |
| POST | /api/auth/2fa/disable | JWT | disable2FA | |
| POST | /api/auth/2fa/backup/regenerate | JWT | regenerateBackupCodes | |
| GET | /api/auth/graph-token | JWT | graphTokenController | |

### `/api/insights`: `routes/insights.js`
Router-level middleware: JWT. Handlers are in `P/insights/handlers.ts`. Both routes are gated by `ENABLE_INSIGHTS`.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| GET | /api/insights/access | JWT | `createInsightsAccessHandler` | |
| GET | /api/insights | JWT | `createInsightsHandler` | |

### `/api/admin`: `routes/admin/auth.js` (admin panel login and OAuth)
The admin OAuth callbacks run: `passport.authenticate(<x>Admin)`, then `tenantContextMiddleware`, then `retrievePkceChallenge`, then cap(ACCESS_ADMIN), then setBalanceConfig, then checkDomainAllowed, then `createOAuthHandler(<adminPanelUrl>/auth/<p>/callback)`. The callback redirects to the admin panel with a one-time exchange code.

Each start route (`GET /oauth/<p>`) is guarded by `requireAdminStrategy('<p>Admin')`, or `requireOpenIdConfig` for OpenID; either returns a 404 JSON when the strategy is not configured. The start route stores a PKCE challenge in the `ADMIN_OAUTH_EXCHANGE` cache and redirects to the IdP.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| POST | /api/admin/login/local | logHeaders, requireSameOrigin, loginLimiter, checkBan, validateEmailLogin, requireLocalAuth, tenantContextMiddleware, cap(ACCESS_ADMIN), setBalanceConfig | loginController | |
| GET | /api/admin/verify | JWT, cap(ACCESS_ADMIN) | inline | returns `{user}` |
| GET | /api/admin/oauth/openid/check | – | inline | 404 if OpenID is not configured |
| GET | /api/admin/oauth/openid | requireOpenIdConfig | inline → passport `openidAdmin` | redirect |
| GET | /api/admin/oauth/openid/callback | admin callback chain | createOAuthHandler | redirect |
| GET | /api/admin/oauth/saml | requireAdminStrategy(samlAdmin) | passport | |
| POST | /api/admin/oauth/saml/callback | chain (state taken from `RelayState`) | createOAuthHandler | |
| GET / GET | /api/admin/oauth/google, /google/callback | googleAdmin | same pattern | |
| GET / GET | /api/admin/oauth/github, /github/callback | githubAdmin | same pattern | |
| GET / GET | /api/admin/oauth/discord, /discord/callback | discordAdmin | same pattern | |
| GET / GET | /api/admin/oauth/facebook, /facebook/callback | facebookAdmin | same pattern | |
| GET / POST | /api/admin/oauth/apple, /apple/callback (POST) | appleAdmin | same pattern | |
| POST | /api/admin/oauth/exchange | loginLimiter | inline; `exchangeAdminCode` (P) | body `{code (64 hex), code_verifier?}`; returns `{token, refreshToken, user}` |
| POST | /api/admin/oauth/refresh | loginLimiter, checkBan, preTenant | inline; `applyAdminRefresh` / `applyGoogleAdminRefresh` (P) | body `{refresh_token, user_id?, provider: openid\|google}` |

### `/api/admin/config`: `routes/admin/config.js`
Router-level: ADMIN. Handlers come from `createAdminConfigHandlers` in `P/admin/config.ts`.

| Method | Path | Handler |
|---|---|---|
| GET | / | listConfigs |
| GET | /base | getBaseConfig |
| GET | /:principalType/:principalId | getConfig |
| PUT | /:principalType/:principalId | upsertConfigOverrides |
| PATCH | /:principalType/:principalId/fields | patchConfigField |
| POST | /:principalType/:principalId/fields/tombstone | tombstoneConfigField |
| DELETE | /:principalType/:principalId/fields | deleteConfigField |
| DELETE | /:principalType/:principalId | deleteConfigOverrides |
| PATCH | /:principalType/:principalId/active | toggleConfig |

### `/api/admin/code-environments`: `routes/admin/code.js`
Router-level: ADMIN + cap(MANAGE_CODE_ENVIRONMENTS, platformOnly). Handlers come from `createAdminCodeEnvironmentHandlers` in `P/admin/code.ts`.

| Method | Path | Handler |
|---|---|---|
| POST | /:environmentId/pairings | createPairing |
| POST | /:environmentId/revoke | revokeWorker |

### `/api/code-environments`: `routes/code-environments.js`
Router-level: JWT. Handlers come from `createCodeEnvironmentHttpHandlers` in `P/code/http.ts` and are created lazily.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | / | – | list |
| POST | /pairings | codeEnvironmentPairingLimiter | pair |
| POST | / | cap(MANAGE_CODE_ENVIRONMENTS) | register |
| GET | /:environmentId/status | codeEnvironmentStatusIpLimiter, codeEnvironmentStatusLimiter | status |
| PATCH | /:environmentId/settings | – | updateSettings |
| DELETE | /:environmentId | – | remove |
| PATCH | /conversations/:conversationId/decision | codeEnvironmentStatusIpLimiter, codeEnvironmentStatusLimiter | moveConversationDecision |

### `/api/admin/langfuse`: `routes/admin/langfuse.js`
Router-level: JWT, cap(ACCESS_ADMIN), `requireLangfuseManage` (runs `hasConfigCapability(user,'langfuse')`), cfg. Handlers are in `P/admin/langfuse.ts`.

| Method | Path | Handler |
|---|---|---|
| GET | /connection | getConnection |
| GET | /connection/session/:conversationId | getSessionLink |
| PUT | /connection | updateConnection |
| POST | /connection/test | testConnection |

### `/api/admin/grants`: `routes/admin/grants.js`
Router-level: ADMIN. Handlers are in `P/admin/grants.ts`.

| Method | Path | Handler |
|---|---|---|
| GET | / | listGrants |
| GET | /effective | getEffectiveCapabilities |
| GET | /:principalType/:principalId | getPrincipalGrants |
| POST | / | assignGrant |
| DELETE | /:principalType/:principalId/:capability | revokeGrant (the capability is URI-encoded) |

### `/api/admin/groups`: `routes/admin/groups.js`
Router-level: ADMIN. Handlers are in `P/admin/groups.ts`. Per route, R = cap(READ_GROUPS) and M = cap(MANAGE_GROUPS).

| Method | Path | Cap | Handler |
|---|---|---|---|
| GET | / | R | listGroups |
| POST | / | M | createGroup |
| GET | /:id | R | getGroup |
| PATCH | /:id | M | updateGroup |
| DELETE | /:id | M | deleteGroup |
| GET | /:id/members | R | getGroupMembers |
| POST | /:id/members | M | addGroupMember |
| DELETE | /:id/members/:userId | M | removeGroupMember |

### `/api/admin/roles`: `routes/admin/roles.js`
Router-level: ADMIN. Handlers are in `P/admin/roles.ts`. R = READ_ROLES, M = MANAGE_ROLES.

| Method | Path | Cap | Handler |
|---|---|---|---|
| GET | / | R | listRoles |
| POST | / | M | createRole |
| GET | /:name | R | getRole |
| PATCH | /:name | M | updateRole |
| DELETE | /:name | M | deleteRole |
| PATCH | /:name/permissions | M | updateRolePermissions |
| GET | /:name/members | R | getRoleMembers |
| POST | /:name/members | M | addRoleMember |
| DELETE | /:name/members/:userId | M | removeRoleMember |

### `/api/admin/skills`: `routes/admin/skills.js`
Router-level: JWT, cap(ACCESS_ADMIN), cfg, `syncAccess.attachBaseSkillSyncConfig`. `syncAccess` comes from `createAdminSkillsSyncAccess` and the handlers from `createAdminSkillsSyncHandlers`, both in `P/admin/skills.ts`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /sync/status | requireReadSkills, attachCredentialReadAccess | getSyncStatus |
| POST | /sync/run | requireSyncRunCapability | runSync |
| PUT | /sync/credentials/:credentialKey | requirePlatformManageSkills | setCredential |
| DELETE | /sync/credentials/:credentialKey | requirePlatformManageSkills | deleteCredential |

### `/api/admin/users`: `routes/admin/users.js`
Router-level: ADMIN. Handlers are in `P/admin/users.ts`. A `DELETE /:id` route exists in the file but is commented out.

| Method | Path | Cap | Handler |
|---|---|---|---|
| GET | / | READ_USERS | listUsers |
| GET | /search | READ_USERS | searchUsers |

### `/api/admin/audit-log`: `routes/admin/audit.js`
Router-level: JWT, cap(ACCESS_ADMIN), cap(READ_AUDIT_LOG). Handlers are in `P/admin/auditLog.ts`.

| Method | Path | Handler | Notes |
|---|---|---|---|
| GET | / | listAuditLog | |
| GET | /export.csv | exportAuditLogCsv | CSV file download (`text/csv`, attachment) |
| GET | /verify | verifyAuditLog | |
| GET | /:id | getAuditLogEntry | |

### `/api/actions`: `routes/actions.js` (agent/assistant action OAuth)

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| POST | /api/actions/:action_id/oauth/bind | JWT, setOAuthSession (P) | inline | sets a CSRF cookie at path `/api/actions` |
| GET | /api/actions/:action_id/oauth/callback | – | inline (`getAccessToken`, P; flowManager) | public; the JWT-signed `state` is verified; CSRF/session cookie check; redirects to `/oauth/success` or `/oauth/error` |

### `/api/keys`: `routes/keys.js`
All routes: JWT. Handlers are inline and use `updateUserKey`, `deleteUserKey` and `getUserKeyExpiry` from `~/models`.

| Method | Path | Notes |
|---|---|---|
| PUT | / | body `{name, value, expiresAt}`; returns 201 |
| DELETE | /:name | returns 204 |
| DELETE | / | requires `?all=true`; returns 204 |
| GET | / | `?name=`; returns the expiry |

### `/api/api-keys`: `routes/apiKeys.js`
All routes: JWT + gCA(REMOTE_AGENTS:USE). Handlers come from `createApiKeyHandlers` in `P/apiKeys/handlers.ts`.

| Method | Path | Handler |
|---|---|---|
| POST | / | createApiKey |
| GET | / | listApiKeys |
| GET | /:id | getApiKey |
| DELETE | /:id | deleteApiKey |

### `/api/user`: `routes/user.js`
Controllers are in `A/controllers/UserController.js`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /api/user | JWT | getUserController |
| PATCH | /api/user/preferences | JWT, cfg | `createUserPreferencesHandler` (`P/user/preferences.ts`) |
| GET | /api/user/terms | JWT | getTermsStatusController |
| POST | /api/user/terms/accept | JWT | acceptTermsController |
| POST | /api/user/plugins | JWT | updateUserPluginsController |
| DELETE | /api/user/delete | JWT, canDeleteAccount, cfg | deleteUserController |
| POST | /api/user/verify | verifyEmailSubmissionLimiter | verifyEmailController (public) |
| POST | /api/user/verify/resend | verifyEmailLimiter | resendVerificationController (public) |

**`/api/user/settings`** (`routes/settings.js`). Every route has JWT.

| Method | Path | Handler (file) |
|---|---|---|
| GET | /settings/favorites/tools | `createToolFavoritesHandlers.listToolFavorites` (`P/favorites/handlers.ts`) |
| PUT | /settings/favorites/tools/:itemType/:itemId | addToolFavorite |
| DELETE | /settings/favorites/tools/:itemType/:itemId | removeToolFavorite |
| GET | /settings/favorites | getFavoritesController (`A/controllers/FavoritesController.js`) |
| POST | /settings/favorites | updateFavoritesController |
| GET | /settings/pinned-order | `createPinnedOrderHandlers.getPinnedOrder` (`P/favorites/pinned.ts`) |
| POST | /settings/pinned-order | updatePinnedOrder |
| GET | /settings/skills/active | getSkillStatesController (`A/controllers/SkillStatesController.js`) |
| POST | /settings/skills/active | updateSkillStatesController |

### `/api/search`: `routes/search.js`

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| GET | /api/search/enable | JWT | inline | returns a bare boolean (Meilisearch health, `SEARCH` env) |

### `/api/messages`: `routes/messages.js`
Router-level: JWT. All handlers are inline in `routes/messages.js`.
- `mutation` = `[validateMessageReq, cfg]`
- `storedMutation` = `mutation` + `filterStoredMessageContent` (`createContentFilter`)

| Method | Path | Middleware | Notes |
|---|---|---|---|
| GET | / | – | search and paginate messages (cursor, search, conversationId, messageId) |
| POST | /branch | cfg | branch a message |
| POST | /artifact/:messageId | cfg | edit artifact content (`A/services/Artifacts/update`) |
| GET | /:conversationId | prepareMessageRequestValidation | messages for a conversation |
| POST | /:conversationId | storedMutation | save a message |
| GET | /:conversationId/:messageId | validateMessageReq | |
| PUT | /:conversationId/:messageId | mutation | edit text or content |
| PUT | /:conversationId/:messageId/feedback | validateMessageReq, cfg, requireFeedbackEnabled (P), filterFeedbackContent | |
| DELETE | /:conversationId/:messageId | validateMessageReq | |

### `/api/convos`: `routes/convos.js`
Router-level: JWT. Handlers are inline unless named.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| GET | / | – | inline | cursor pagination; params limit, cursor, isArchived, pinned, search, sortBy, sortDirection, projectId, tags |
| GET | /:parentConversationId/subagents/:threadId/tasks/:taskId/activity | – | `createSubagentActivityStreamHandler` (`P/agents/subagentActivity.ts`) | **SSE** |
| POST | /:parentConversationId/subagents/:threadId/control | cfg, message IP/user limiters (env-gated; agent-trigger exempt), validateSubagentControlRequest, filterSubagentControlMessage, moderateSubagentControlMessage | `createSubagentControlHandler` (`P/agents/control.ts`) | |
| GET | /:parentConversationId/subagents | – | `createParentSubagentIndexHandler` (`P/agents/view.ts`) | |
| GET | /:parentConversationId/subagents/:threadId | – | `createSubagentThreadViewHandler` (`P/agents/view.ts`) | |
| GET | /:conversationId | – | inline | 404 for subagent threads |
| GET | /gen_title/:conversationId | – | inline | long-polls the `GEN_TITLE` cache with backoff (~15s) |
| DELETE | / | cfg | inline | body `{arg:{conversationId, source, thread_id, endpoint}}`; also deletes assistant threads, shared links and checkpoints |
| DELETE | /all | cfg | inline | |
| POST | /archive | validateConvoAccess | inline | body `{arg:{conversationId, isArchived}}` |
| POST | /archive/all | – | `createArchiveAllHandler` (`P/conversations/archive.ts`) | |
| POST | /pin | validateConvoAccess | inline | |
| POST | /update | validateConvoAccess, cfg | inline | title update |
| POST | /import | importIpLimiter, importUserLimiter, cfg, handleUpload (multer `file`), restoreTenantContextFromReq | inline → `importConversations` | multipart |
| POST | /fork | forkIpLimiter, forkUserLimiter, cfg | `forkConversation` (`A/utils/import/fork`) | |
| POST | /duplicate | forkIpLimiter, forkUserLimiter, cfg, filterConversationTitle | `duplicateConversation` | |

### `/api/traces`: `routes/traces.js`
Router-level: JWT, cfg. Handlers come from `createTraceHandlers` in `P/traces/handlers.ts` (a Langfuse reader).

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /:conversationId/availability | – | availability |
| GET | /:conversationId/records | traceReadLimiter | records |
| GET | /:conversationId/records/:recordId | traceReadLimiter | record |

### `/api/presets`: `routes/presets.js`
Router-level: JWT. Handlers are inline.

| Method | Path | Middleware | Notes |
|---|---|---|---|
| GET | / | cfg | |
| POST | / | cfg, filterPresetContent | 201 |
| POST | /delete | – | body `{presetId?}` |

### `/api/projects`: `routes/projects.js`
Router-level: JWT. Handlers come from `createProjectHandlers` in `P/projects/handlers.ts`.

| Method | Path | Handler |
|---|---|---|
| GET | / | listProjects |
| POST | / | createProject |
| PUT | /conversations/:conversationId | assignConversationToProject |
| GET | /:projectId | getProject |
| PATCH | /:projectId | updateProject |
| DELETE | /:projectId | deleteProject |

### `/api/prompts`: `routes/prompts.js`
Router-level: JWT, gCA(PROMPTS:USE).
- `create` = gCA(PROMPTS:USE+CREATE)
- `grpAcc(P)` = `canAccessPromptGroupResource({requiredPermission: PermissionBits.P})`
- `pAcc(P)` = `canAccessPromptViaGroup({requiredPermission: P, resourceIdParam: 'promptId'})`
- All handlers are inline in the file, including the named ones: createNewPromptGroup, addPromptToGroup, patchPromptGroup, deletePromptController, deletePromptGroupController.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /groups/:groupId | grpAcc(VIEW), cfg | inline |
| GET | /all | cfg | inline (ACL-aware list) |
| GET | /groups | cfg | inline (paginated groups) |
| POST | / | create, cfg | createNewPromptGroup |
| POST | /groups/:groupId/prompts | checkPromptAccess, grpAcc(EDIT), cfg | addPromptToGroup |
| POST | /groups/:groupId/use | promptUsageLimiter, grpAcc(VIEW) | inline (incrementPromptGroupUsage) |
| PATCH | /groups/:groupId | create, grpAcc(EDIT), cfg | patchPromptGroup |
| PATCH | /:promptId/tags/production | create, pAcc(EDIT), cfg | inline (makePromptProduction) |
| GET | /:promptId | pAcc(VIEW), cfg | inline |
| GET | / | cfg | inline (`?groupId`) |
| DELETE | /:promptId | create, pAcc(DELETE) | deletePromptController |
| DELETE | /groups/:groupId | create, grpAcc(DELETE) | deletePromptGroupController |

### `/api/skills`: `routes/skills.js`
Router-level: JWT, cfg, gCA(SKILLS:USE).
- `create` = gCA(SKILLS:USE+CREATE)
- `sAcc(P)` = `canAccessSkillResource({requiredPermission: P})`
- Handlers: `getSkillsHandlers()` (`A/services/Skills/handlers.js`) wraps `createSkillsHandlers` (`P/skills/handlers.ts`).
- A router-level error handler turns Multer errors and errors starting with "Only " into 400.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| POST | /import | create, fileUploadIpLimiter, fileUploadUserLimiter, skillUpload (multer memory, `.md`/`.zip`/`.skill`), restoreTenantContextFromReq | `createImportHandler` (`P/skills/import.ts`) | multipart |
| GET | / | maybeStartRequestSkillSync | handlers.list | |
| POST | / | create | handlers.create | |
| GET | /:id | sAcc(VIEW) | handlers.get | |
| PATCH | /:id | create, sAcc(EDIT) | handlers.patch | |
| DELETE | /:id | create, sAcc(DELETE) | handlers.delete | |
| GET | /:id/files | sAcc(VIEW) | handlers.listFiles | |
| POST | /:id/files | sAcc(EDIT), upload limiters, multer `file` (10 MB), restoreTenantContextFromReq | uploadFileHandler (inline) | multipart |
| GET | /:id/files/*relativePath | sAcc(VIEW) | handlers.downloadFile | file download |
| DELETE | /:id/files/*relativePath | sAcc(EDIT) | handlers.deleteFile | |

### Small routers

| Method | Path | Middleware | Handler (file) | Notes |
|---|---|---|---|---|
| GET | /api/categories | JWT | inline (`getCategories`) | |
| GET | /api/endpoints | JWT, cfg | endpointController (`A/controllers/EndpointController.js`) | |
| GET | /api/endpoints/token-config | JWT, cfg | tokenConfigController (`A/controllers/TokenConfigController.js`) | |
| GET | /api/balance | JWT, setBalanceConfig | `A/controllers/Balance.js` | |
| GET | /api/models | JWT | modelController (`A/controllers/ModelController.js`) | |
| GET | /api/config | (mount) preTenant, optJWT | inline in `routes/config.js` | startup config; returns a reduced pre-login payload when no user |
| GET | /api/banner | optJWT | inline (`getBanner`) | |

### `/api/assistants`: `routes/assistants/*`
Router-level (`index.js`): JWT, checkBan, uaParser, cfg. The v2 router adds cfg again.

Controllers:
- `A/controllers/assistants/v1.js` (c1)
- `A/controllers/assistants/v2.js` (c2)
- `A/controllers/assistants/chatV1.js`, `chatV2.js`

`filterAssistantContent` = `createContentFilter`. The chat chain is: `createMessageFilterPii`, validateModel, buildEndpointOption, validateAssistant, validateConvoAccess, guardSubagentThreadTurn.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| POST | /api/assistants/v1/actions/:assistant_id | – | inline (`routes/assistants/actions.js`) | |
| DELETE | /api/assistants/v1/actions/:assistant_id/:action_id/:model | – | inline | |
| GET | /api/assistants/v1/tools | – | getAvailableTools (`A/controllers/PluginController.js`) | |
| GET | /api/assistants/v1/documents | – | c1.getAssistantDocuments | |
| POST | /api/assistants/v1 | filterAssistantContent | c1.createAssistant | |
| GET | /api/assistants/v1/:id | – | c1.retrieveAssistant | |
| PATCH | /api/assistants/v1/:id | filterAssistantContent | c1.patchAssistant | |
| DELETE | /api/assistants/v1/:id | – | c1.deleteAssistant | |
| GET | /api/assistants/v1 | – | c1.listAssistants | |
| POST | /api/assistants/v1/chat/abort | – | `handleAbort()` | |
| POST | /api/assistants/v1/chat | chat chain | chatV1 controller | **SSE** (`sendEvent`) |
| POST | /api/assistants/v2/actions/:assistant_id | – | inline | |
| DELETE | /api/assistants/v2/actions/:assistant_id/:action_id/:model | – | inline | |
| GET | /api/assistants/v2/tools | – | getAvailableTools | |
| GET | /api/assistants/v2/documents | – | c1.getAssistantDocuments | |
| POST | /api/assistants/v2 | filterAssistantContent | c2.createAssistant | |
| GET | /api/assistants/v2/:id | – | c1.retrieveAssistant | |
| PATCH | /api/assistants/v2/:id | filterAssistantContent | c2.patchAssistant | |
| DELETE | /api/assistants/v2/:id | – | c1.deleteAssistant | |
| GET | /api/assistants/v2 | – | c1.listAssistants | |
| POST | /api/assistants/v2/avatar/:assistant_id | – | c1.uploadAssistantAvatar | |
| POST | /api/assistants/v2/chat/abort | – | handleAbort() | |
| POST | /api/assistants/v2/chat | chat chain | chatV2 controller | **SSE** |

### `/api/files`: `routes/files/index.js` (async `initialize()`)
Router-level: JWT, cfg, checkBan, uaParser.
- After the `/speech` mount, every POST outside `/speech` goes through the upload limiters (`fileUploadIpLimiter` then `fileUploadUserLimiter`). `/usage` uses `fileUsageLimiter` instead.
- Upload routes first run `upload.single('file')` (multer from `routes/files/multer.js`) and then `restoreTenantContextFromReq`.
- File uploads optionally reply as **SSE** when `FILE_UPLOAD_SSE_ENABLED` is set and the client sends `Accept: text/event-stream` (`P/files/sse.ts`).

| Method | Path | Middleware | Handler (file) | Notes |
|---|---|---|---|---|
| POST | /api/files/speech/stt | multer `audio`, sttIpLimiter, sttUserLimiter | speechToText (`A/services/Files/Audio`) | multipart |
| POST | /api/files/speech/tts/manual | ttsIp/UserLimiter, `multer().none()`, PII check | textToSpeech | audio stream |
| POST | /api/files/speech/tts | tts limiters | inline → streamAudio | chunked audio stream |
| GET | /api/files/speech/tts/voices | tts limiters | getVoices | |
| GET | /api/files/speech/config/get | – | getCustomConfigSpeech | |
| GET | /api/files | – | inline (`routes/files/files.js`) | the user's files |
| GET | /api/files/agent/:agent_id | – | inline | the agent's files |
| GET | /api/files/config | – | inline | merged fileConfig |
| POST | /api/files/usage | fileUsageLimiter | inline | |
| DELETE | /api/files | – | inline | body `{files:[...], agent_id?, assistant_id?, tool_resource?}` |
| GET | /api/files/code/download/:session_id/:fileId | – | inline | **file download** (octet-stream piped from the code env) |
| GET | /api/files/:file_id/preview | fileAccess | inline | preview status / text |
| GET | /api/files/download-url/:userId/:file_id | fileAccess | inline | JSON `{url, filename, type, metadata}`; 501 if unsupported |
| GET | /api/files/download/:userId/:file_id | fileAccess | inline | **file download** (stream, or a 302 redirect to a signed URL) |
| POST | /api/files | multer file, upload limiters | handleFileUpload (`files.js`) | multipart; JSON or SSE |
| POST | /api/files/images | multer, limiters | inline (`routes/files/images.js`) | multipart; JSON or SSE |
| POST | /api/files/images/avatar | multer, limiters | inline (`routes/files/avatar.js`) | returns `{url}` |
| POST | /api/files/images/agents/:agent_id/avatar | multer, limiters, gCA(AGENTS:USE), canAccessAgentResource(EDIT, `agent_id`) | v1.uploadAgentAvatar (`A/controllers/agents/v1.js`) | |
| POST | /api/files/images/assistants/:assistant_id/avatar | multer, limiters | c1.uploadAssistantAvatar | |

`fileAccess` is defined in `A/middleware/accessResources/fileAccess.js`.

### `/images/`: `routes/static.js`
Mounted as `createValidateImageRequest({secureImageLinks})` (`A/middleware/validateImageRequest.js`), which uses `createImageAuthorizationMiddleware` from P. The router is `staticCache(paths.imageOutput)`. It serves static image files.

### `/api/share`: `routes/share.js`
Mounted with `preTenant`. The first 6 routes are registered only when `ALLOW_SHARED_LINKS` is on (the default).
- `sharedCfg` = `createSharedLinkConfigMiddleware` (`P/shared-links/config.ts`)
- `shareCreate` = gCA(SHARED_LINKS:CREATE)
- Handlers are inline.

| Method | Path | Middleware | Notes |
|---|---|---|---|
| GET | /:shareId/config | optJWT, canAccessSharedLink, sharedCfg | share startup config |
| GET | /:shareId | optJWT, shareIpLimiter, shareUserLimiter, canAccessSharedLink, sharedCfg | shared conversation |
| POST | /:shareId/fork | JWT, forkIpLimiter, forkUserLimiter, canAccessSharedLink, sharedCfg | `forkSharedConversation` |
| GET | /:shareId/files/:file_id/preview | optJWT, optionalShareFileAuth, canAccessSharedLink, sharedCfg, resolveShareFile, enforceSharedFileContentPolicy | |
| GET | /:shareId/files/:file_id/download | same chain as preview | **file download** (attachment) |
| GET | /:shareId/files/:file_id | same chain as preview | **file stream** (inline) |
| GET | / | JWT | the user's links (cursor, pageSize, search) |
| GET | /link/:conversationId | JWT | |
| POST | /:conversationId | JWT, cfg, shareCreate | create a link |
| PATCH | /:shareId | JWT, cfg, shareCreate | |
| DELETE | /:shareId | JWT | |

### `/api/roles`: `routes/roles.js`
Router-level: JWT. `manage` = cap(MANAGE_ROLES). Handlers are inline via `createPermissionUpdateHandler(key)`.

| Method | Path | Middleware | Notes |
|---|---|---|---|
| GET | /:roleName | – | inline; READ_ROLES is required only for other or non-default roles |
| PUT | /:roleName/prompts | manage | |
| PUT | /:roleName/agents | manage | |
| PUT | /:roleName/memories | manage | |
| PUT | /:roleName/people-picker | manage | |
| PUT | /:roleName/mcp-servers | manage | |
| PUT | /:roleName/marketplace | manage | |
| PUT | /:roleName/remote-agents | manage | |
| PUT | /:roleName/skills | manage | |

### `/api/agents`: `routes/agents/index.js` and its sub-routers
Middleware applied before this mount, for `/api/agents/chat` only: agentStartupIngressMiddleware, agentStartupTelemetryMiddleware, rejectChatStartsUntilReady.

The sub-routers are evaluated in this order. The machine-auth routers come before `router.use(requireJwtAuth)`.

**(a) `/api/agents/v1/responses`** (`routes/agents/responses.js`), the Open Responses API.
- Router-level: preTenant, `requireRemoteAgentAuth` (`createRemoteAgentAuth`, `P/middleware/remoteAgentAuth.ts`), cfg, `checkRemoteAgentsFeature` (gCA REMOTE_AGENTS:USE).
- `requireRemoteAgentAuth` accepts an OIDC Bearer token checked against JWKS, falling back to an API key via `createRequireApiKeyAuth` (`P/apiKeys/middleware.ts`).
- Controllers are in `A/controllers/agents/responses.js`.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| POST | /api/agents/v1/responses | checkAgentPermission (`createCheckRemoteAgentAccess`) | createResponse | JSON, or **SSE** when `stream: true` |
| GET | /api/agents/v1/responses/models | – | listModels | |
| GET | /api/agents/v1/responses/:id | – | getResponse | |

**(b) `/api/agents/v1/agents`** (`routes/agents/management.js`), agent management over M2M.
- Router-level: `requireAgentManagementAuth` (`createAgentManagementAuth`, `P/middleware/management.ts`: an OIDC client-credentials Bearer token plus a client binding), checkBan.
- There is a catch-all 404 JSON (`mapAgentManagementError`).
- Handlers are in `P/agents/{creates,reads,updates,deletion,files}.ts`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| POST | / | cfg | createAgentManagementCreateHandler |
| GET | / | – | readHandlers.list |
| POST | /:id/files | cfg, fileUploadIp/UserLimiter, fileHandlers.authorizeUpload, multer single, restoreTenantContextFromReq, then handleUploadError | fileHandlers.upload (multipart) |
| GET | /:id/files | – | fileHandlers.list |
| DELETE | /:id/files/:fileId | – | fileHandlers.remove |
| GET | /:id | – | readHandlers.get |
| PATCH | /:id | cfg | updateHandler |
| DELETE | /:id | – | deleteHandler |

**(c) `/api/agents/v1/skills`** (`routes/agents/skills.js`).
- Router-level: requireAgentManagementAuth, checkBan, cfg. There is a catch-all 404.
- Handlers come from `createSkillManagementHandlers` in `P/skills/management.ts`.

| Method | Path | Handler |
|---|---|---|
| GET | / | list |
| GET | /:id | get |
| PATCH | /:id | update |
| GET | /:id/files | listFiles |
| GET | /:id/files/*relativePath | getFile |
| PUT | /:id/files/*relativePath | updateFile |

**(d) `/api/agents/v1`** (`routes/agents/openai.js`), OpenAI-compatible routes plus event triggers.
- Router-level: preTenant, requireRemoteAgentAuth, cfg, checkRemoteAgentsFeature.
- These apply to every `/api/agents/v1/*` request that reaches this router.

| Method | Path | Middleware | Handler (file) | Notes |
|---|---|---|---|---|
| POST | /api/agents/v1/events/bindings | agentEventUserLimiter, checkAgentTriggerPermission | `createAgentEventBindingHandlers.register` (`P/agents/triggers/bindings.ts`) | |
| POST | /api/agents/v1/events | agentEventUserLimiter, createMessageFilterPii, eventBindingHandlers.resolve, checkAgentTriggerPermission | `createAgentTriggerIngressHandlers.enqueueEvent` (`P/agents/triggers/ingress.ts`) | |
| GET | /api/agents/v1/events/:id | – | eventHandlers.getEvent | |
| POST | /api/agents/v1/chat/completions | checkAgentPermission | OpenAIChatCompletionController (`A/controllers/agents/openai.js`) | JSON, or **SSE** when `stream: true` |
| GET | /api/agents/v1/models | – | ListModelsController | |
| GET | /api/agents/v1/models/:model | – | GetModelController | |

**(e) JWT routes.** These run after `router.use(requireJwtAuth)`, then an inline step that sets `req._isAgentTrigger` and `captureScheduleFireContext`, then `checkBan`, then `uaParser`.
- `steerLim` = messageIpLimiter / messageUserLimiter, each gated by `LIMIT_MESSAGE_IP` / `LIMIT_MESSAGE_USER`.
- `pii` = `createMessageFilterPii`.
- Unless noted, handlers are inline in `routes/agents/index.js`.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| GET | /api/agents/chat/stream/:streamId | – | inline | **SSE** (`GenerationJobManager` subscribe with replay; `?resume=true`, `generationCreatedAt`) |
| GET | /api/agents/chat/active | – | inline | `{activeJobIds}` |
| GET | /api/agents/chat/status/:conversationId | – | inline | job status and resume state |
| POST | /api/agents/chat/abort | cfg | inline | |
| POST | /api/agents/chat/steer | cfg, steerLim, pii, moderateText | SteerController (`A/controllers/agents/steer.js`) | |
| POST | /api/agents/chat/steer/deliver | cfg, steerLim, pii, moderateText | SteerController.SteerDeliveryController | |
| POST | /api/agents/chat/steer/cancel | cfg, steerLim | SteerCancelController | |
| POST | /api/agents/chat/steer/arm | cfg, steerLim | SteerArmController | |
| POST | /api/agents/chat/queued-turns | cfg, steerLim, pii, moderateText | AgentQueuedTurnEnqueueController (`A/controllers/agents/queuedTurns.js`) | |
| GET | /api/agents/chat/queued-turns | cfg | AgentQueuedTurnListController | |
| DELETE | /api/agents/chat/queued-turns/:queuedTurnId | cfg, steerLim | AgentQueuedTurnCancelController | |

**(f) `router.use('/', v1)`** (`routes/agents/v1.js`), still under JWT.
- Controllers are in `A/controllers/agents/v1.js`.
- `use` = gCA(AGENTS:USE), `create` = gCA(AGENTS:USE+CREATE).
- `acc(P)` = `canAccessAgentResource({requiredPermission: P, resourceIdParam: 'id'})`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /api/agents/actions | cfg | inline (`routes/agents/actions.js`) |
| POST | /api/agents/actions/:agent_id | cfg, canAccessAgentResource(EDIT, `agent_id`), create | inline |
| DELETE | /api/agents/actions/:agent_id/:action_id | cfg, canAccessAgentResource(EDIT), create | inline |
| GET | /api/agents/tools | cfg | getAvailableTools (PluginController) |
| GET | /api/agents/tools/calls | cfg | getToolCalls (`A/controllers/tools.js`) |
| GET | /api/agents/tools/:toolId/auth | cfg | verifyToolAuth |
| POST | /api/agents/tools/:toolId/call | cfg, toolCallLimiter, filterToolArguments | callTool |
| GET | /api/agents/categories | – | v1.getAgentCategories |
| POST | /api/agents | create, cfg | v1.createAgent |
| GET | /api/agents/:id | use, acc(VIEW) | v1.getAgent |
| GET | /api/agents/:id/expanded | use, acc(EDIT) | v1.getAgent(expanded=true) |
| GET | /api/agents/:id/versions | use, acc(EDIT) | v1.getAgentVersions |
| PATCH | /api/agents/:id | create, cfg, acc(EDIT), cfg | v1.updateAgent |
| POST | /api/agents/:id/duplicate | create, cfg, acc(EDIT), cfg | v1.duplicateAgent |
| DELETE | /api/agents/:id | create, acc(DELETE) | v1.deleteAgent |
| POST | /api/agents/:id/revert | create, cfg, acc(EDIT), cfg | v1.revertAgentVersion |
| GET | /api/agents | use | v1.getListAgents |

**(g) `/api/agents/chat`**: a `chatRouter` wrapping `routes/agents/chat.js`.
- `chatRouter`: cfg. When message limiting is enabled, it adds generationRetryProbeLimiter, detectGenerationRetry, generationRetryLimiter, messageIpLimiter and messageUserLimiter, with exemptions for agent triggers and schedules.
- `chat.js` router-level: restoreResumeContext, createMessageFilterPii, moderateText, checkAgentAccess (gCA AGENTS:USE with skipAgentCheck), `canAccessAgentFromBody(VIEW)`, validateConvoAccess, guardSubagentThreadTurn, buildEndpointOption.

| Method | Path | Handler | Notes |
|---|---|---|---|
| POST | /api/agents/chat/resume | ResumeController (`A/controllers/agents/resume.js`) | HITL resume; JSON ack, stream via /chat/stream |
| POST | /api/agents/chat | AgentController (`A/controllers/agents/request.js`) | JSON ack `{streamId, conversationId, generationProtocolVersion,...}`; the client then opens the GET /chat/stream/:streamId SSE |
| POST | /api/agents/chat/:endpoint | AgentController | ephemeral agent; same response shape |

### `/api/memories`: `routes/memories.js`
Router-level: JWT. Handlers are inline, except the `/id/:id` routes, which use `createMemoryManagementHandlers` from `P/memory/handlers.ts`.
- `lim` = `express.json({limit:'100kb'})`
- The partition validators come from `createAgentMemoryPartitionMiddleware`: body, query, and a delete variant with `allowMissingAgent`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | / | gCA(MEMORIES:USE+READ), cfg | inline |
| POST | / | lim, gCA(USE+CREATE), validateBodyAgentPartition, cfg | inline |
| PATCH | /preferences | gCA(USE+OPT_OUT) | inline |
| PATCH | /id/:id | lim, gCA(USE+UPDATE), validateQueryAgentPartition, cfg | opaqueMemoryHandlers.updateById |
| DELETE | /id/:id | gCA(USE+UPDATE), validateDeletedAgentPartition | opaqueMemoryHandlers.deleteById |
| PATCH | /:key | lim, gCA(USE+UPDATE), validateQueryAgentPartition, cfg | inline |
| DELETE | /:key | gCA(USE+UPDATE), validateDeletedAgentPartition | inline |

### `/api/schedules`: `routes/schedules.js`
Mounted behind `rejectScheduleWritesUntilReady`. Router-level: JWT, cfg. Handlers come from `createSchedulesHandlers` in `P/schedules/handlers.ts`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | / | gCA(SCHEDULES:USE) | listSchedules |
| GET | /:id | gCA(USE) | getSchedule |
| POST | / | gCA(USE+CREATE) | createSchedule |
| PATCH | /:id | gCA(USE+CREATE) | updateSchedule |
| DELETE | /:id | gCA(USE+CREATE) | deleteSchedule |
| POST | /:id/run | messageIpLimiter (if `LIMIT_MESSAGE_IP`), gCA(USE+CREATE) | runScheduleNow |

### `/api/permissions`: `routes/accessPermissions.js`
Router-level: JWT, checkBan, uaParser. Controllers are in `A/controllers/PermissionsController.js`.

`checkResourcePermissionAccess(SHARE)` uses `createAgentAdminPermissionAccess`. Its fallback dispatches `canAccessResource` by resourceType:
- agent, remoteAgent, promptGroup
- mcpServer (resolved with `findMCPServerByObjectId`)
- skill (resolved with `getSkillById`)
- codeEnvironment, sharedLink
- any other type returns 400

| Method | Path | Middleware | Handler |
|---|---|---|---|
| GET | /search-principals | checkPeoplePickerAccess | searchPrincipals |
| GET | /:resourceType/roles | – | getResourceRoles |
| GET | /:resourceType/:resourceId | checkResourcePermissionAccess(SHARE) | getResourcePermissions |
| PUT | /:resourceType/:resourceId | checkResourcePermissionAccess(SHARE), checkShareAccessUnlessAgentAdmin, checkSharePublicAccess, rejectSharedLinkOwnerPermissionChanges | updateResourcePermissions |
| GET | /:resourceType/effective/all | – | getAllEffectivePermissions |
| GET | /:resourceType/:resourceId/effective | – | getUserEffectivePermissions |

### `/api/tags`: `routes/tags.js`
Router-level: JWT, gCA(BOOKMARKS:USE). Handlers are inline.

| Method | Path | Notes |
|---|---|---|
| GET | / | |
| POST | / | |
| PUT | /:tag | the tag is URI-decoded |
| DELETE | /:tag | |
| PUT | /convo/:conversationId | body `{tags}` |

### `/api/mcp`: `routes/mcp.js`
No router-level middleware. Controllers are in `A/controllers/mcp.js`; other handlers are inline.
- `mcpUse` = gCA(MCP_SERVERS:USE), `mcpCreate` = gCA(MCP_SERVERS:USE+CREATE).
- `srvAcc(P)` = `canAccessMCPServerResource({requiredPermission: P, resourceIdParam: 'serverName'})`.

| Method | Path | Middleware | Handler | Notes |
|---|---|---|---|---|
| GET | /tools | JWT, cfg, mcpUse | getMCPTools | |
| GET | /:serverName/oauth/initiate | JWT, setOAuthSession | inline | redirect to the IdP |
| GET | /:serverName/oauth/callback | – | inline | public (state + CSRF cookie); redirects to `/oauth/success` or `/oauth/error` |
| GET | /oauth/tokens/:flowId | JWT | inline | |
| POST | /:serverName/oauth/bind | JWT, setOAuthSession | inline | sets a CSRF cookie at path `/api/mcp` |
| GET | /oauth/status/:flowId | JWT | inline | |
| POST | /oauth/cancel/:serverName | JWT | inline | |
| POST | /:serverName/reinitialize | JWT, cfg, mcpUse, setOAuthSession | inline | |
| GET | /connection/status | JWT, cfg | inline | |
| GET | /connection/status/:serverName | JWT, cfg | inline | |
| GET | /:serverName/auth-values | JWT, mcpUse | inline | |
| GET | /servers | JWT, mcpUse | getMCPServersList | |
| POST | /servers | JWT, mcpCreate | createMCPServerController | |
| GET | /servers/:serverName | JWT, mcpUse, srvAcc(VIEW) | getMCPServerById | |
| PATCH | /servers/:serverName | JWT, mcpCreate, srvAcc(EDIT) | updateMCPServerController | |
| DELETE | /servers/:serverName | JWT, mcpCreate, srvAcc(DELETE) | deleteMCPServerController (with maybeUninstallOAuthMCP) | |

### `/api/rum`: `routes/rum.js`
Both routes are an OTLP passthrough to the upstream collector (`proxyRumRequest`, `P/rum/proxy.ts`). `requireRumProxyAuth` is JWT that answers 204 instead of an auth error; it lives in `A/middleware/requireJwtAuth.js`.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| POST | /v1/traces | requireRumProxyEnabled (404 if off), requireRumProxyAuth, `express.raw` (protobuf/octet-stream) | proxyTelemetry |
| POST | /v1/logs | same as above | proxyTelemetry |

### `/metrics` and `/api` openapi

| Method | Path | Auth | Handler | Notes |
|---|---|---|---|---|
| GET | /metrics | `Authorization: Bearer $METRICS_SECRET` | metricsHandler (`P/app/metrics.ts` ~L1035) | Prometheus text; 401 if the secret is unset |
| GET | /api/openapi.json | public | `P/openapi/router.ts` | 404 unless `config.openapi.enabled` |
| GET | /api/docs | public | same file | Swagger HTML |
| GET | /api/docs/assets/* | public | `express.static(swagger-ui-dist)` | |

---

## 4. Things a port needs to preserve
- **Route order matters.** `rejectChatStartsUntilReady` is mounted on `/api/agents/chat` before the `/api/agents` router. Inside the agents router, the `/v1/*` machine-auth routers are registered before JWT, and the `/chat/*` routes registered before the `chatRouter` skip the message rate limiters and `buildEndpointOption`.
- **Streaming (SSE):**
  - `GET /api/agents/chat/stream/:streamId`
  - `GET /api/convos/:p/subagents/:t/tasks/:k/activity`
  - `POST /api/assistants/v{1,2}/chat`
  - `POST /api/agents/v1/chat/completions` and `/api/agents/v1/responses`, when `stream: true`
  - file and image uploads, when opted in (see `/api/files`)
- **Downloads and streamed bodies:**
  - `/api/files/download/:userId/:file_id`
  - `/api/files/code/download/:session_id/:fileId`
  - `/api/share/:shareId/files/:file_id[/download]`
  - `/api/admin/audit-log/export.csv`
  - `/api/skills/:id/files/*relativePath`
  - TTS audio responses
  - `/images/*` static files
- **Multipart uploads:**
  - `/api/files`, `/api/files/images`, `/api/files/images/avatar`, `/api/files/images/agents/:agent_id/avatar`, `/api/files/images/assistants/:assistant_id/avatar`
  - `/api/files/speech/stt` (field `audio`)
  - `/api/convos/import`
  - `/api/skills/import` and `/api/skills/:id/files`
  - `/api/agents/v1/agents/:id/files`
- **Auth types:**
  - Cookie- or Bearer-based JWT, with OpenID token reuse.
  - Remote-agent OIDC-or-API-key on `/api/agents/v1/{responses, chat/completions, models, events}`.
  - M2M OIDC client-credentials on `/api/agents/v1/{agents, skills}`.
  - Bearer `METRICS_SECRET` on `/metrics`.
  - Public: `/health`, `/livez`, `/readyz`, `/oauth/*`, the login/register/reset/refresh routes, the `/api/admin/oauth/*` start and callback routes plus `/exchange` and `/refresh`, the action and MCP OAuth callbacks, `/api/user/verify*`, `/api/config` (optional auth), `/api/banner` (optional auth), share views (optional auth), and openapi.
