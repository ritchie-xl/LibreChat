# HTTP API 清单

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

本章覆盖 `api/server/routes/**` 下每个路由器文件中的每条路由，以及 `api/server/index.js` 自身定义的路由。

**路径。** `A` = `api/server`。`P` = `packages/api/src`。在代码中，`~/` 解析为 `api/`。

**表格中使用的缩写：**
- **JWT** = `requireJwtAuth`（`A/middleware/requireJwtAuth.js`）。它检查 passport `jwt` Bearer 令牌；当设置了 `token_provider=openid` cookie 且开启了 `OPENID_REUSE_TOKENS` 时，改为检查 `openidJwt`。之后运行 `tenantContextMiddleware`。
- **optJWT** = `optionalJwtAuth`。
- **cfg** = `configMiddleware`（`A/middleware/config/app`）。它把 `req.config` 设为按用户/角色/租户解析的应用配置。
- **gCA(X:[perms])** = 来自 `P/middleware/access.ts` 的 `generateCheckAccess({permissionType: PermissionTypes.X, permissions: [...]})`。它是一个角色权限检查。
- **cap(X)** = `requireCapability(SystemCapabilities.X)`（`A/middleware/roles/capabilities`）。
- **ADMIN** = `JWT + cap(ACCESS_ADMIN)`。
- **preTenant** = `preAuthTenantMiddleware`（读取 `X-Tenant-Id`）。
- 除非某行另有说明，响应均为 JSON。

---

## 1. 汇总：前缀 → 路由器文件 → 认证 → 路由数量

| 挂载前缀 | 路由器文件 | 认证 | 路由数 |
|---|---|---|---|
| `/health`、`/livez`、`/readyz` | 内联于 `A/index.js` | 公开 | 3 |
| `/oauth` | `routes/oauth.js` | 公开（passport 社交登录） | 15 |
| `/api/auth` | `routes/auth.js` | 混合：公开 / JWT | 14 |
| `/api/insights` | `routes/insights.js` | JWT | 2 |
| `/api/admin` | `routes/admin/auth.js` | 混合：公开的登录和 OAuth / ADMIN | 19 |
| `/api/admin/config` | `routes/admin/config.js` | ADMIN | 9 |
| `/api/admin/code-environments` | `routes/admin/code.js` | ADMIN + cap(MANAGE_CODE_ENVIRONMENTS, platformOnly) | 2 |
| `/api/code-environments` | `routes/code-environments.js` | JWT | 7 |
| `/api/admin/langfuse` | `routes/admin/langfuse.js` | ADMIN + langfuse 配置能力 | 4 |
| `/api/admin/grants` | `routes/admin/grants.js` | ADMIN | 5 |
| `/api/admin/groups` | `routes/admin/groups.js` | ADMIN + READ/MANAGE_GROUPS | 8 |
| `/api/admin/roles` | `routes/admin/roles.js` | ADMIN + READ/MANAGE_ROLES | 9 |
| `/api/admin/skills` | `routes/admin/skills.js` | ADMIN + 技能同步相关能力 | 4 |
| `/api/admin/users` | `routes/admin/users.js` | ADMIN + READ_USERS | 2 |
| `/api/admin/audit-log` | `routes/admin/audit.js` | ADMIN + READ_AUDIT_LOG | 4 |
| `/api/actions` | `routes/actions.js` | 混合：JWT / 公开的 OAuth 回调 | 2 |
| `/api/keys` | `routes/keys.js` | JWT | 4 |
| `/api/api-keys` | `routes/apiKeys.js` | JWT + gCA(REMOTE_AGENTS:USE) | 4 |
| `/api/user`（+ `/api/user/settings`） | `routes/user.js`、`routes/settings.js` | JWT；邮箱验证路由为公开 | 8 + 9 |
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
| `/api/files` | `routes/files/*`（由异步 `initialize()` 构建） | JWT | 19 |
| `/images/` | `routes/static.js` | `createValidateImageRequest`（secureImageLinks cookie/JWT） | 静态 |
| `/api/share` | `routes/share.js` | preTenant；混合 optJWT / JWT | 11（其中 6 条仅在开启 `ALLOW_SHARED_LINKS` 时存在） |
| `/api/roles` | `routes/roles.js` | JWT | 9 |
| `/api/agents` | `routes/agents/*` | 混合：远程智能体 API 密钥/OIDC、M2M OIDC、JWT | 54 |
| `/api/banner` | `routes/banner.js` | optJWT | 1 |
| `/api/memories` | `routes/memories.js` | JWT | 7 |
| `/api/schedules` | `routes/schedules.js` | JWT（+ 启动期写入闸门） | 6 |
| `/api/permissions` | `routes/accessPermissions.js` | JWT + checkBan + uaParser | 6 |
| `/api/tags` | `routes/tags.js` | JWT + gCA(BOOKMARKS:USE) | 5 |
| `/api/mcp` | `routes/mcp.js` | JWT；OAuth 回调为公开 | 16 |
| `/api/rum` | `routes/rum.js` | `requireRumProxyAuth` | 2 |
| `/metrics` | 来自 `createMetrics()`（`P/app/metrics.ts`）的 `metricsRouter` | Bearer `METRICS_SECRET` | 1 |
| `/api`（openapi） | `routes/openapi.js` → `createOpenApiRouter`（`P/openapi/router.ts`） | 公开（除非 `config.openapi.enabled`，否则返回 404） | 3 |
| `/api/*` 兜底 | `apiNotFound`（`P/middleware/notFound.ts`） | — | 404 JSON `{message:'Endpoint not found'}` |
| `*` | `createSpaFallback(sendIndexHtml)`（`A/utils/fallback`） | — | 返回 SPA 的 `index.html` |

有两个文件没有被直接挂载：
- `routes/settings.js` 只能通过 `/api/user/settings` 访问。
- `routes/types/assistants.js` 只包含 JSDoc 类型，没有路由。

`A/experimental.js` 是一个独立的集群入口（`npm run backend:experimental`）。它挂载同一批路由器中的一部分，不包含 adminConfig、langfuse、grants、groups、roles、users、audit、traces、rum、metrics 以及智能体聊天的启动闸门。

---

## 2. 全局 Express 中间件（按顺序，`A/index.js` 约 L204–464）

1. `app.disable('x-powered-by')`，然后 `app.set('trust proxy', Number(TRUST_PROXY) || 1)`
2. `createSecurityHeaders()`，仅在已配置时
3. **在任何其他中间件之前注册的路由：**`GET /health` → `"OK"`，`GET /livez` → `"OK"`，`GET /readyz` → `"OK"`，在 `serverReady` 之前返回 503 `"NOT_READY"`。三者均为纯文本。
4. `requestContextMiddleware`
5. `app.use('/api/agents/chat', agentStartupIngressMiddleware)`
6. `metricsMiddleware`
7. `noIndex`
8. `express.json({limit:'3mb'})`
9. `express.urlencoded({extended:true, limit:'3mb'})`
10. `handleJsonParseError`
11. 一个让 `req.query` 可写的垫片（Express 5）
12. `mongoSanitize()`
13. `cors()`
14. `cookieParser()`
15. `compression()`，除非设置了 `DISABLE_COMPRESSION`
16. `GET /index.html` → `sendIndexHtml`（注入 lang、CSP nonce、页脚引导数据）
17. `staticCache(dist)`、`staticCache(fonts)`、`staticCache(assets)`
18. `telemetry.telemetryMiddleware`，如果启用了遥测
19. `app.use('/api/agents/chat', agentStartupTelemetryMiddleware)`
20. `passport.initialize()`，策略为 `jwt`、`local`，以及 `ldap`（当设置了 `LDAP_URL` 和 `LDAP_USER_SEARCH_BASE` 时）
21. 如果 `ALLOW_SOCIAL_LOGIN`：`configureSocialLogins(app)`（`A/socialLogins.js`）。它为 OpenID（和 SAML）添加 `express-session` + `passport.session()`，并注册社交登录和管理后台的策略。
22. `capabilityContextMiddleware`
23. 路由挂载，顺序与第 1 节表格一致。挂载时应用两个闸门：
    - `app.use('/api/agents/chat', rejectChatStartsUntilReady)` 紧挨着放在 `/api/agents` 之前。在服务器就绪之前，除 `/abort` 之外的任何 POST 都返回 503 `{code:'SERVER_NOT_READY'}` 并带 `Retry-After: 1`。
    - `/api/schedules` 挂载在 `rejectScheduleWritesUntilReady`（`createScheduleWriteGate`）之后。
24. `/metrics`，然后是 `/api` openapi，然后是 `/api` `apiNotFound`，最后是 SPA 兜底
25. `telemetry.telemetryErrorMiddleware`（如果启用），然后是 `ErrorController`

---

## 3. 路由表

### `/oauth`：`routes/oauth.js`
路由器级中间件：`logHeaders`、`markOAuthNavigation`、`loginLimiter`。挂载时带 `preTenant`。
- 回调依次运行 `passport.authenticate(<strategy>)`、`setBalanceConfig`（`createSetBalanceConfig`）、`checkDomainAllowed`、`oauthHandler`。
- `oauthHandler` 是来自 `A/controllers/auth/oauth.js` 的 `createOAuthHandler()`。它设置认证 cookie 并重定向到客户端。
- 这里的每条路由都返回重定向。

| 方法 | 路径 | 中间件 | 处理器（文件） | 说明 |
|---|---|---|---|---|
| GET | /oauth/error | – | 内联；`redirectToAuthFailure`（P） | 重定向 |
| GET | /oauth/google | passport `google` | passport | 重定向到 IdP |
| GET | /oauth/google/callback | passport google, setBalanceConfig, checkDomainAllowed | oauthHandler | |
| GET | /oauth/facebook | passport `facebook` | passport | |
| GET | /oauth/facebook/callback | 与 google 相同的链 | oauthHandler | |
| GET | /oauth/openid | passport `openid`（randomState） | 内联 | |
| GET | /oauth/openid/callback | `createOpenIDCallbackAuthenticator`（P）, setBalanceConfig, checkDomainAllowed | oauthHandler | |
| GET | /oauth/github | passport `github` | | |
| GET | /oauth/github/callback | 与 google 相同的链 | oauthHandler | |
| GET | /oauth/discord | passport `discord` | | |
| GET | /oauth/discord/callback | 与 google 相同的链 | oauthHandler | |
| GET | /oauth/apple | passport `apple` | | |
| POST | /oauth/apple/callback | 与 google 相同的链 | oauthHandler | 表单 POST |
| GET | /oauth/saml | passport `saml` | | |
| POST | /oauth/saml/callback | passport saml（failureMessage） | oauthHandler | 不做余额和域名检查 |

### `/api/auth`：`routes/auth.js`
挂载时带 `preTenant`。控制器位于：
- `A/controllers/AuthController.js`
- `A/controllers/TwoFactorController.js`
- `A/controllers/auth/{LoginController, LogoutController, TwoFactorAuthController}.js`

| 方法 | 路径 | 中间件 | 处理器（文件） | 说明 |
|---|---|---|---|---|
| POST | /api/auth/logout | JWT | logoutController | |
| POST | /api/auth/login | logHeaders, requireSameOrigin, loginLimiter, checkBan, validateEmailLogin, `requireLdapAuth` 或 `requireLocalAuth`, setBalanceConfig | loginController | 公开；设置刷新 cookie 并返回 `{token,user}` |
| POST | /api/auth/refresh | – | refreshController | 使用刷新 cookie |
| POST | /api/auth/cloudfront/refresh | JWT | 内联（`forceRefreshCloudFrontAuthCookies`，P） | CloudFront 禁用时返回 404 |
| POST | /api/auth/register | registerLimiter, checkBan, checkInviteUser, validateRegistration | registrationController | 公开 |
| POST | /api/auth/requestPasswordReset | resetPasswordLimiter, checkBan, validatePasswordReset | resetPasswordRequestController | 公开 |
| POST | /api/auth/resetPassword | resetPasswordSubmissionLimiter, checkBan, validatePasswordReset | resetPasswordController | 公开 |
| POST | /api/auth/2fa/enable | JWT | enable2FA | |
| POST | /api/auth/2fa/verify | JWT | verify2FA | |
| POST | /api/auth/2fa/verify-temp | requireSameOrigin, setTwoFactorTempUser, twoFactorTempLimiter, checkBan | verify2FAWithTempToken | 临时令牌认证 |
| POST | /api/auth/2fa/confirm | JWT | confirm2FA | |
| POST | /api/auth/2fa/disable | JWT | disable2FA | |
| POST | /api/auth/2fa/backup/regenerate | JWT | regenerateBackupCodes | |
| GET | /api/auth/graph-token | JWT | graphTokenController | |

### `/api/insights`：`routes/insights.js`
路由器级中间件：JWT。处理器位于 `P/insights/handlers.ts`。两条路由都受 `ENABLE_INSIGHTS` 控制。

| 方法 | 路径 | 中间件 | 处理器 | 说明 |
|---|---|---|---|---|
| GET | /api/insights/access | JWT | `createInsightsAccessHandler` | |
| GET | /api/insights | JWT | `createInsightsHandler` | |

### `/api/admin`：`routes/admin/auth.js`（管理后台登录与 OAuth）
管理后台的 OAuth 回调依次运行：`passport.authenticate(<x>Admin)`、`tenantContextMiddleware`、`retrievePkceChallenge`、cap(ACCESS_ADMIN)、setBalanceConfig、checkDomainAllowed、`createOAuthHandler(<adminPanelUrl>/auth/<p>/callback)`。回调会携带一次性交换码重定向到管理后台。

每个起始路由（`GET /oauth/<p>`）由 `requireAdminStrategy('<p>Admin')` 保护，OpenID 则由 `requireOpenIdConfig` 保护；策略未配置时两者都返回 404 JSON。起始路由把 PKCE challenge 存入 `ADMIN_OAUTH_EXCHANGE` 缓存，并重定向到 IdP。

| 方法 | 路径 | 中间件 | 处理器 | 说明 |
|---|---|---|---|---|
| POST | /api/admin/login/local | logHeaders, requireSameOrigin, loginLimiter, checkBan, validateEmailLogin, requireLocalAuth, tenantContextMiddleware, cap(ACCESS_ADMIN), setBalanceConfig | loginController | |
| GET | /api/admin/verify | JWT, cap(ACCESS_ADMIN) | 内联 | 返回 `{user}` |
| GET | /api/admin/oauth/openid/check | – | 内联 | 未配置 OpenID 时返回 404 |
| GET | /api/admin/oauth/openid | requireOpenIdConfig | 内联 → passport `openidAdmin` | 重定向 |
| GET | /api/admin/oauth/openid/callback | 管理后台回调链 | createOAuthHandler | 重定向 |
| GET | /api/admin/oauth/saml | requireAdminStrategy(samlAdmin) | passport | |
| POST | /api/admin/oauth/saml/callback | 回调链（state 取自 `RelayState`） | createOAuthHandler | |
| GET / GET | /api/admin/oauth/google, /google/callback | googleAdmin | 同一模式 | |
| GET / GET | /api/admin/oauth/github, /github/callback | githubAdmin | 同一模式 | |
| GET / GET | /api/admin/oauth/discord, /discord/callback | discordAdmin | 同一模式 | |
| GET / GET | /api/admin/oauth/facebook, /facebook/callback | facebookAdmin | 同一模式 | |
| GET / POST | /api/admin/oauth/apple, /apple/callback (POST) | appleAdmin | 同一模式 | |
| POST | /api/admin/oauth/exchange | loginLimiter | 内联；`exchangeAdminCode`（P） | 请求体 `{code (64 hex), code_verifier?}`；返回 `{token, refreshToken, user}` |
| POST | /api/admin/oauth/refresh | loginLimiter, checkBan, preTenant | 内联；`applyAdminRefresh` / `applyGoogleAdminRefresh`（P） | 请求体 `{refresh_token, user_id?, provider: openid\|google}` |

### `/api/admin/config`：`routes/admin/config.js`
路由器级：ADMIN。处理器来自 `P/admin/config.ts` 中的 `createAdminConfigHandlers`。

| 方法 | 路径 | 处理器 |
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

### `/api/admin/code-environments`：`routes/admin/code.js`
路由器级：ADMIN + cap(MANAGE_CODE_ENVIRONMENTS, platformOnly)。处理器来自 `P/admin/code.ts` 中的 `createAdminCodeEnvironmentHandlers`。

| 方法 | 路径 | 处理器 |
|---|---|---|
| POST | /:environmentId/pairings | createPairing |
| POST | /:environmentId/revoke | revokeWorker |

### `/api/code-environments`：`routes/code-environments.js`
路由器级：JWT。处理器来自 `P/code/http.ts` 中的 `createCodeEnvironmentHttpHandlers`，惰性创建。

| 方法 | 路径 | 中间件 | 处理器 |
|---|---|---|---|
| GET | / | – | list |
| POST | /pairings | codeEnvironmentPairingLimiter | pair |
| POST | / | cap(MANAGE_CODE_ENVIRONMENTS) | register |
| GET | /:environmentId/status | codeEnvironmentStatusIpLimiter, codeEnvironmentStatusLimiter | status |
| PATCH | /:environmentId/settings | – | updateSettings |
| DELETE | /:environmentId | – | remove |
| PATCH | /conversations/:conversationId/decision | codeEnvironmentStatusIpLimiter, codeEnvironmentStatusLimiter | moveConversationDecision |

### `/api/admin/langfuse`：`routes/admin/langfuse.js`
路由器级：JWT、cap(ACCESS_ADMIN)、`requireLangfuseManage`（运行 `hasConfigCapability(user,'langfuse')`）、cfg。处理器位于 `P/admin/langfuse.ts`。

| 方法 | 路径 | 处理器 |
|---|---|---|
| GET | /connection | getConnection |
| GET | /connection/session/:conversationId | getSessionLink |
| PUT | /connection | updateConnection |
| POST | /connection/test | testConnection |

### `/api/admin/grants`：`routes/admin/grants.js`
路由器级：ADMIN。处理器位于 `P/admin/grants.ts`。

| 方法 | 路径 | 处理器 |
|---|---|---|
| GET | / | listGrants |
| GET | /effective | getEffectiveCapabilities |
| GET | /:principalType/:principalId | getPrincipalGrants |
| POST | / | assignGrant |
| DELETE | /:principalType/:principalId/:capability | revokeGrant（capability 经过 URI 编码） |

### `/api/admin/groups`：`routes/admin/groups.js`
路由器级：ADMIN。处理器位于 `P/admin/groups.ts`。按路由区分，R = cap(READ_GROUPS)，M = cap(MANAGE_GROUPS)。

| 方法 | 路径 | 能力 | 处理器 |
|---|---|---|---|
| GET | / | R | listGroups |
| POST | / | M | createGroup |
| GET | /:id | R | getGroup |
| PATCH | /:id | M | updateGroup |
| DELETE | /:id | M | deleteGroup |
| GET | /:id/members | R | getGroupMembers |
| POST | /:id/members | M | addGroupMember |
| DELETE | /:id/members/:userId | M | removeGroupMember |

### `/api/admin/roles`：`routes/admin/roles.js`
路由器级：ADMIN。处理器位于 `P/admin/roles.ts`。R = READ_ROLES，M = MANAGE_ROLES。

| 方法 | 路径 | 能力 | 处理器 |
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

### `/api/admin/skills`：`routes/admin/skills.js`
路由器级：JWT、cap(ACCESS_ADMIN)、cfg、`syncAccess.attachBaseSkillSyncConfig`。`syncAccess` 来自 `createAdminSkillsSyncAccess`，处理器来自 `createAdminSkillsSyncHandlers`，两者都在 `P/admin/skills.ts` 中。

| 方法 | 路径 | 中间件 | 处理器 |
|---|---|---|---|
| GET | /sync/status | requireReadSkills, attachCredentialReadAccess | getSyncStatus |
| POST | /sync/run | requireSyncRunCapability | runSync |
| PUT | /sync/credentials/:credentialKey | requirePlatformManageSkills | setCredential |
| DELETE | /sync/credentials/:credentialKey | requirePlatformManageSkills | deleteCredential |

### `/api/admin/users`：`routes/admin/users.js`
路由器级：ADMIN。处理器位于 `P/admin/users.ts`。文件中有一条 `DELETE /:id` 路由，但已被注释掉。

| 方法 | 路径 | 能力 | 处理器 |
|---|---|---|---|
| GET | / | READ_USERS | listUsers |
| GET | /search | READ_USERS | searchUsers |

### `/api/admin/audit-log`：`routes/admin/audit.js`
路由器级：JWT、cap(ACCESS_ADMIN)、cap(READ_AUDIT_LOG)。处理器位于 `P/admin/auditLog.ts`。

| 方法 | 路径 | 处理器 | 说明 |
|---|---|---|---|
| GET | / | listAuditLog | |
| GET | /export.csv | exportAuditLogCsv | CSV 文件下载（`text/csv`，attachment） |
| GET | /verify | verifyAuditLog | |
| GET | /:id | getAuditLogEntry | |

### `/api/actions`：`routes/actions.js`（智能体/助手 action 的 OAuth）

| 方法 | 路径 | 中间件 | 处理器 | 说明 |
|---|---|---|---|---|
| POST | /api/actions/:action_id/oauth/bind | JWT, setOAuthSession（P） | 内联 | 在路径 `/api/actions` 上设置 CSRF cookie |
| GET | /api/actions/:action_id/oauth/callback | – | 内联（`getAccessToken`，P；flowManager） | 公开；验证经 JWT 签名的 `state`；检查 CSRF/会话 cookie；重定向到 `/oauth/success` 或 `/oauth/error` |

### `/api/keys`：`routes/keys.js`
所有路由：JWT。处理器为内联，使用 `~/models` 中的 `updateUserKey`、`deleteUserKey` 和 `getUserKeyExpiry`。

| 方法 | 路径 | 说明 |
|---|---|---|
| PUT | / | 请求体 `{name, value, expiresAt}`；返回 201 |
| DELETE | /:name | 返回 204 |
| DELETE | / | 需要 `?all=true`；返回 204 |
| GET | / | `?name=`；返回过期时间 |

### `/api/api-keys`：`routes/apiKeys.js`
所有路由：JWT + gCA(REMOTE_AGENTS:USE)。处理器来自 `P/apiKeys/handlers.ts` 中的 `createApiKeyHandlers`。

| 方法 | 路径 | 处理器 |
|---|---|---|
| POST | / | createApiKey |
| GET | / | listApiKeys |
| GET | /:id | getApiKey |
| DELETE | /:id | deleteApiKey |

### `/api/user`：`routes/user.js`
控制器位于 `A/controllers/UserController.js`。

| 方法 | 路径 | 中间件 | 处理器 |
|---|---|---|---|
| GET | /api/user | JWT | getUserController |
| PATCH | /api/user/preferences | JWT, cfg | `createUserPreferencesHandler`（`P/user/preferences.ts`） |
| GET | /api/user/terms | JWT | getTermsStatusController |
| POST | /api/user/terms/accept | JWT | acceptTermsController |
| POST | /api/user/plugins | JWT | updateUserPluginsController |
| DELETE | /api/user/delete | JWT, canDeleteAccount, cfg | deleteUserController |
| POST | /api/user/verify | verifyEmailSubmissionLimiter | verifyEmailController（公开） |
| POST | /api/user/verify/resend | verifyEmailLimiter | resendVerificationController（公开） |

**`/api/user/settings`**（`routes/settings.js`）。每条路由都带 JWT。

| 方法 | 路径 | 处理器（文件） |
|---|---|---|
| GET | /settings/favorites/tools | `createToolFavoritesHandlers.listToolFavorites`（`P/favorites/handlers.ts`） |
| PUT | /settings/favorites/tools/:itemType/:itemId | addToolFavorite |
| DELETE | /settings/favorites/tools/:itemType/:itemId | removeToolFavorite |
| GET | /settings/favorites | getFavoritesController（`A/controllers/FavoritesController.js`） |
| POST | /settings/favorites | updateFavoritesController |
| GET | /settings/pinned-order | `createPinnedOrderHandlers.getPinnedOrder`（`P/favorites/pinned.ts`） |
| POST | /settings/pinned-order | updatePinnedOrder |
| GET | /settings/skills/active | getSkillStatesController（`A/controllers/SkillStatesController.js`） |
| POST | /settings/skills/active | updateSkillStatesController |

### `/api/search`：`routes/search.js`

| 方法 | 路径 | 中间件 | 处理器 | 说明 |
|---|---|---|---|---|
| GET | /api/search/enable | JWT | 内联 | 返回一个裸布尔值（Meilisearch 健康状况，`SEARCH` 环境变量） |

### `/api/messages`：`routes/messages.js`
路由器级：JWT。所有处理器都内联在 `routes/messages.js` 中。
- `mutation` = `[validateMessageReq, cfg]`
- `storedMutation` = `mutation` + `filterStoredMessageContent`（`createContentFilter`）

| 方法 | 路径 | 中间件 | 说明 |
|---|---|---|---|
| GET | / | – | 搜索并分页消息（cursor、search、conversationId、messageId） |
| POST | /branch | cfg | 为消息创建分支 |
| POST | /artifact/:messageId | cfg | 编辑 artifact 内容（`A/services/Artifacts/update`） |
| GET | /:conversationId | prepareMessageRequestValidation | 某个对话的消息 |
| POST | /:conversationId | storedMutation | 保存消息 |
| GET | /:conversationId/:messageId | validateMessageReq | |
| PUT | /:conversationId/:messageId | mutation | 编辑文本或内容 |
| PUT | /:conversationId/:messageId/feedback | validateMessageReq, cfg, requireFeedbackEnabled（P）, filterFeedbackContent | |
| DELETE | /:conversationId/:messageId | validateMessageReq | |

### `/api/convos`：`routes/convos.js`
路由器级：JWT。除非给出了名称，否则处理器为内联。

| 方法 | 路径 | 中间件 | 处理器 | 说明 |
|---|---|---|---|---|
| GET | / | – | 内联 | 游标分页；参数 limit、cursor、isArchived、pinned、search、sortBy、sortDirection、projectId、tags |
| GET | /:parentConversationId/subagents/:threadId/tasks/:taskId/activity | – | `createSubagentActivityStreamHandler`（`P/agents/subagentActivity.ts`） | **SSE** |
| POST | /:parentConversationId/subagents/:threadId/control | cfg, 消息 IP/用户限流器（受环境变量控制；智能体触发器豁免）, validateSubagentControlRequest, filterSubagentControlMessage, moderateSubagentControlMessage | `createSubagentControlHandler`（`P/agents/control.ts`） | |
| GET | /:parentConversationId/subagents | – | `createParentSubagentIndexHandler`（`P/agents/view.ts`） | |
| GET | /:parentConversationId/subagents/:threadId | – | `createSubagentThreadViewHandler`（`P/agents/view.ts`） | |
| GET | /:conversationId | – | 内联 | 对子智能体线程返回 404 |
| GET | /gen_title/:conversationId | – | 内联 | 带退避地长轮询 `GEN_TITLE` 缓存（约 15s） |
| DELETE | / | cfg | 内联 | 请求体 `{arg:{conversationId, source, thread_id, endpoint}}`；同时删除助手线程、分享链接和检查点 |
| DELETE | /all | cfg | 内联 | |
| POST | /archive | validateConvoAccess | 内联 | 请求体 `{arg:{conversationId, isArchived}}` |
| POST | /archive/all | – | `createArchiveAllHandler`（`P/conversations/archive.ts`） | |
| POST | /pin | validateConvoAccess | 内联 | |
| POST | /update | validateConvoAccess, cfg | 内联 | 更新标题 |
| POST | /import | importIpLimiter, importUserLimiter, cfg, handleUpload（multer `file`）, restoreTenantContextFromReq | 内联 → `importConversations` | multipart |
| POST | /fork | forkIpLimiter, forkUserLimiter, cfg | `forkConversation`（`A/utils/import/fork`） | |
| POST | /duplicate | forkIpLimiter, forkUserLimiter, cfg, filterConversationTitle | `duplicateConversation` | |

### `/api/traces`：`routes/traces.js`
路由器级：JWT、cfg。处理器来自 `P/traces/handlers.ts` 中的 `createTraceHandlers`（一个 Langfuse 读取器）。

| 方法 | 路径 | 中间件 | 处理器 |
|---|---|---|---|
| GET | /:conversationId/availability | – | availability |
| GET | /:conversationId/records | traceReadLimiter | records |
| GET | /:conversationId/records/:recordId | traceReadLimiter | record |

### `/api/presets`：`routes/presets.js`
路由器级：JWT。处理器为内联。
