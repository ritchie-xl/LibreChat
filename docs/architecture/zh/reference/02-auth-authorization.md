# 认证、会话、授权、管理、API 密钥与余额

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

服务端使用 Express、Passport 和 Mongoose。大部分逻辑位于 `packages/api/src`（TS，`@librechat/api`），schema 和数据库方法位于 `packages/data-schemas/src`，共享枚举位于 `packages/data-provider/src`。`api/server` 层基本只是轻量的装配（wiring）。

---

## 1. 令牌与会话模型

### 1.1 凭据

| 凭据 | 内容 | 密钥 / 存储 | 有效期 |
|---|---|---|---|
| **访问令牌**（JSON 响应中的 "token"） | HS256 JWT `{id, username, provider, email}` | `JWT_SECRET`；客户端保存在内存中，通过 `Authorization: Bearer` 发送 | `SESSION_EXPIRY` 毫秒，默认 15 分钟（`DEFAULT_SESSION_EXPIRY`，`packages/data-schemas/src/methods/user.ts` 的 `generateToken`） |
| **刷新令牌**（本地、LDAP、社交登录、SAML，以及未开启复用的 OpenID） | HS256 JWT `{id, sessionId}` | `JWT_REFRESH_SECRET`；`refreshToken` cookie | `REFRESH_TOKEN_EXPIRY` 毫秒，默认 7 天（`DEFAULT_REFRESH_TOKEN_EXPIRY`，`packages/data-schemas/src/methods/session.ts`） |
| **会话文档**（`sessions` 集合） | `{refreshTokenHash (SHA-256 hex), expiration (TTL index, expires:0), user, tenantId}`，在 `(user, refreshTokenHash)` 上有唯一索引 | Mongo，`packages/data-schemas/src/schema/session.ts` | = 刷新令牌有效期 |
| **2FA 临时令牌** | JWT `{userId, twoFAPending:true}` | `JWT_SECRET` | 5 分钟（`api/server/services/twoFactorService.js` 的 `generate2FATempToken`） |
| **短期令牌** | JWT `{id}` | `JWT_SECRET` | 5 分钟（`generateShortLivedToken`，`packages/api/src/crypto/jwt.ts`） |
| **智能体触发器令牌** | JWT `{id, scope:'agent_trigger'}` | `JWT_SECRET` | 60 s。仅在 `POST /api/agents/chat/agents` 和 `/api/agents/chat/steer/deliver` 上有效，且必须带请求头 `x-lc-agent-trigger: 1`（`api/strategies/jwtStrategy.js`） |
| **OpenID express-session** | `req.session.openidTokens`，存放在 Keyv/Redis 存储中（`CacheKeys.OPENID_SESSION`），由 `OPENID_SESSION_SECRET` 签名，cookie 为 `connect.sid` | 服务端 | 开启复用时 maxAge = max(SESSION_EXPIRY, OPENID_REUSE_MAX_SESSION_AGE_MS)（`api/server/socialLogins.js`） |
| **SAML express-session** | 存储 `CacheKeys.SAML_SESSION`，密钥 `SAML_SESSION_SECRET` | | SESSION_EXPIRY |

### 1.2 Cookie

所有 cookie 都是 `httpOnly`。认证 cookie 使用 `sameSite:'strict'`。`secure` 由 `packages/api/src/oauth/csrf.ts` 中的 `shouldUseSecureCookie()` 决定：`SESSION_COOKIE_SECURE` 优先；否则取决于是否 `NODE_ENV==='production'` 且 DOMAIN_SERVER 不是 localhost 主机。

- `refreshToken`：LibreChat 的刷新 JWT；对 OpenID 而言是 IdP 的原始刷新令牌。
- `token_provider`：`'librechat'` 或 `'openid'`。它决定使用哪种刷新和认证策略。
- `openid_user_id`：用 `JWT_REFRESH_SECRET` 签名的 JWT，载荷为 `{id, refreshTokenHash: sha256-base64url(refreshToken)}`。仅在开启 `OPENID_REUSE_TOKENS` 时设置。它让无需认证的 `/refresh` 端点得知用户 id，并把该 id 与刷新令牌绑定（`setOpenIDMarkerCookies`）。
- `openid_access_token` / `openid_id_token`：旧的回退方案，仅在不存在 express-session 时使用。
- OAuth 登录 state：`[__Host-]oauth_state_<provider>.<id>`。这是保存在 SameSite=Lax cookie 中的一个随机绑定值。`state` 参数为 `issuedAtBase36.nonce.HMAC_SHA256(JWT_SECRET, "librechat:oauth-login-state:<provider>:<binding>:<issuedAt>:<nonce>")`，其 TTL 为 `registration.oauthStateTtlMs`（`packages/api/src/oauth/state.ts`）。Apple 使用 SameSite=None。
- MCP/Action OAuth：`oauth_csrf`（flowId 的 HMAC，10 分钟，Lax）和 `oauth_session`（userId 的 HMAC，24 小时，路径 `/api`）。
- 图片的 CloudFront 签名 cookie 是可选的（`setCloudFrontAuthCookies`，`api/server/services/AuthService.js`）。

### 1.3 邮件和密码重置令牌（`tokens` 集合）

schema 为 `packages/data-schemas/src/schema/token.ts`：`{userId, email(lowercase,trim), type, identifier, token, createdAt, expiresAt (TTL), metadata, tenantId}`。

- **邮箱验证与密码重置：** 生成 32 个随机字节并转为 hex，存储 `bcrypt(token,10)`，`expiresIn: 900` 秒。`type` 为 `email_verification` 或 `password_reset`。旧文档的 `type:null` 仍被接受。
- **邀请：** 存储令牌的 SHA-256 哈希以及邮箱。
- 同一集合还存储 MCP 和 Actions 的 OAuth 令牌（`type:'oauth'`，加密存储）。

---

## 2. Passport 策略与中间件

注册发生在 `api/server/index.js`（约第 376 行）和 `api/server/socialLogins.js` 中：

- `jwt`（始终注册）
- `local`（始终注册）
- `ldapauth`（设置了 `LDAP_URL` 和 `LDAP_USER_SEARCH_BASE` 时）
- 社交登录，仅在 `ALLOW_SOCIAL_LOGIN` 为 true 时：
  - `google`、`facebook`、`github`、`discord`、`apple`，每个都有 `*Admin` 变体
  - `openid` 和 `openidAdmin`（需要 `OPENID_CLIENT_ID`、`OPENID_ISSUER`、`OPENID_SCOPE` 和 `OPENID_SESSION_SECRET`，以及 `OPENID_CLIENT_SECRET` 或 `OPENID_USE_PKCE` 之一）
  - `openidJwt`，仅在开启 `OPENID_REUSE_TOKENS` 时注册
  - `saml` 和 `samlAdmin`

**`requireJwtAuth`**（`api/server/middleware/requireJwtAuth.js`）：
1. 解析 cookie。如果 `token_provider==='openid'`、开启了 `OPENID_REUSE_TOKENS`、`openidJwt` 策略存在且 `openid_user_id` cookie 校验通过，策略链为 `['openidJwt','jwt']`；否则为 `['jwt']`。
2. 依次尝试每个策略。如果 `openidJwt` 解析出的用户 id 与 `openid_user_id` cookie 不一致，就继续尝试下一个策略。
3. 失败时返回 401 `{message}`，适用时附带 `code: ACCOUNT_DELETION_IN_PROGRESS`。
4. 成功时设置 `req.user` 和 `req.authStrategy`，然后串联 `tenantContextMiddleware`（AsyncLocalStorage 中的租户/用户上下文），并可选地刷新 CloudFront cookie。

**`optionalJwtAuth`** 做同样的事，但从不失败。

**`jwtStrategy`**（`api/strategies/jwtStrategy.js`）用 `JWT_SECRET` 读取 Bearer 令牌：
- 在准入路由之外拒绝触发器作用域的令牌。
- 用 `runAsSystem` 加载用户，排除 password、totpSecret 和 backupCodes。
- 拒绝带有 `agentTriggerDeletionStartedAt` 的用户。
- 设置 `user.id`；角色缺失时默认 `role=USER`（并持久化）。

**`openIdJwtStrategy`**（`api/strategies/openIdJwtStrategy.js`）：
- 通过 JWKS 校验 Bearer 令牌（`jwks-rsa`；缓存由 `OPENID_JWKS_URL_CACHE_ENABLED` 和 `OPENID_JWKS_URL_CACHE_TIME` 控制）。
- audience 为 `OPENID_CLIENT_ID` 加上 `OPENID_AUDIENCE`（CSV）。
- issuer 必须与发现文档一致。支持 `{tenantid}` 模板，用于多租户 Entra。
- 通过 `findOpenIDUser` 解析用户，可选使用认证用户文档缓存（`CacheKeys.AUTH_USER_DOC`）。
- 设置 `user.federatedTokens = {access_token, id_token, refresh_token, expires_at}`，依次取自 `req.session.openidTokens`、cookie，以及（如果能识别为访问令牌）原始 bearer。

其他认证中间件：
- `requireLocalAuth`：没有用户时返回 404；任何 `info.message`（例如 "Email not verified."）返回 422。
- `requireLdapAuth`：模式相同。
- `setTwoFactorTempUser`：从临时令牌设置 `req.user={id}`，使限流器和封禁检查可以按用户做键。
- `requireSameOrigin`：用 `DOMAIN_CLIENT`、`DOMAIN_SERVER` 和 `ADMIN_PANEL_URL` 检查 Origin/Referer。
- `preAuthTenantMiddleware`：读取 `X-Tenant-Id` 请求头，并拒绝 `__SYSTEM__` 哨兵值。

---

## 3. 认证路由与流程

`/api/auth` 路由器为 `api/server/routes/auth.js`。`/oauth` 路由器为 `api/server/routes/oauth.js`。

| 路由 | 中间件链 |
|---|---|
| POST `/api/auth/login` | logHeaders、requireSameOrigin、loginLimiter（`LOGIN_WINDOW`=5 分钟，`LOGIN_MAX`=7）、checkBan、validateEmailLogin（`ALLOW_EMAIL_LOGIN`、`ALLOW_EMAIL_LOGIN_OVERRIDE`）、requireLdapAuth 或 requireLocalAuth、setBalanceConfig、loginController |
| POST `/api/auth/logout` | requireJwtAuth、logoutController |
| POST `/api/auth/refresh` | refreshController（无需认证，由 cookie 驱动） |
| POST `/api/auth/register` | registerLimiter、checkBan、checkInviteUser、validateRegistration（`ALLOW_REGISTRATION` 或有效邀请）、registrationController |
| POST `/api/auth/requestPasswordReset`、`/resetPassword` | 限流器、checkBan、validatePasswordReset |
| POST `/api/auth/2fa/{enable,verify,confirm,disable,backup/regenerate}` | requireJwtAuth |
| POST `/api/auth/2fa/verify-temp` | requireSameOrigin、setTwoFactorTempUser、twoFactorTempLimiter、checkBan |
| GET `/api/auth/graph-token?scopes=` | requireJwtAuth；Microsoft OBO |
| POST `/api/auth/cloudfront/refresh` | requireJwtAuth |

### 3.1 本地登录
1. `passport-local` 读取 `usernameField:'email'`，并校验 `loginSchema`（email；密码长度从 `MIN_PASSWORD_LENGTH`（默认 8）到 128）（`api/strategies/localStrategy.js`、`api/strategies/validators.js`）。
2. `findUser({email}, '+password')`。如果用户不存在或没有密码，以 "Email does not exist." 失败。用 `bcrypt.compare` 比较密码。
3. 旧数据自动验证：如果未配置邮件服务，且用户创建于 2024-06-07 之前，则标记 `emailVerified`。
4. 如果用户未验证且 `ALLOW_UNVERIFIED_EMAIL_LOGIN` 关闭，以 "Email not verified."（422）失败。
5. `loginController`（`api/server/controllers/auth/LoginController.js`）：如果 `user.twoFactorEnabled`，返回 `{twoFAPending:true, tempToken}` 并结束。
6. 否则调用 `setAuthTokens(userId,res,null,req)`：
   1. `createSession(userId,{expiresIn:REFRESH_TOKEN_EXPIRY})`。
   2. `generateRefreshToken` 签发 `{id, sessionId}`，并把它的 SHA-256 哈希存到会话上。
   3. `generateToken` 生成访问 JWT。
   4. 设置 cookie `refreshToken` 和 `token_provider=librechat`（过期时间 = session.expiration）。
   5. 返回 `{token, user}`，不含 password、totpSecret 或 `__v`。

### 3.2 注册
1. `registerSchema` 校验 name（3 到 80 个字符）、可选的 username、email、password 和 confirm_password。
2. 检查租户应用配置中的 `registration.allowedDomains`。
3. 如果邮箱已存在，休眠 1 s 后返回通用的 200（防止账户枚举）。
4. 在未分租户的部署中，第一个用户获得 `ADMIN` 角色；其他人获得 `USER`。用 bcrypt 哈希密码（salt 10）。
5. `createUser(data, appConfig.balance, disableTTL=ALLOW_UNVERIFIED_EMAIL_LOGIN, true)`。如果未禁用 TTL，用户文档带有 `expiresAt` = 7 天（TTL 索引），这样未验证的账户会自动删除。如果开启了余额功能，会以 `startBalance` 创建一条 Balance 记录。
6. 如果配置了邮件服务，发送验证邮件（链接 `${DOMAIN_CLIENT}/verify?token=&email=`）。否则设置 `emailVerified=true`。
7. 如果使用了邀请且用户确实已创建，删除邀请令牌。

### 3.3 邮箱验证
`POST /api/user/verify {email, token}`：
1. 查找用户和最新的验证令牌。
2. 检查令牌的 userId 是否匹配，然后 `bcrypt.compare`。
3. 设置 `emailVerified=true` 并删除令牌。

`POST /api/user/verify/resend` 重新生成令牌。两者都返回通用消息。

### 3.4 密码重置
1. `requestPasswordReset` 先用基础配置做域名检查（被阻止的域名返回 400），然后查找用户并用租户配置再检查一次（此时被阻止的域名返回通用的 200）。
2. 删除旧的重置令牌，创建一个新的 bcrypt 哈希令牌（15 分钟）。
3. 链接：`${DOMAIN_CLIENT}/reset-password?token=&userId=`。如果未配置邮件服务，链接会直接在 JSON 中返回。
4. `resetPassword(userId, token, password)` 用 bcrypt 比较，设置 `password=bcrypt(pw,10)`，发送确认邮件并删除令牌。
5. 控制器随后调用 `deleteAllUserSessions({userId})`，吊销所有刷新令牌。

### 3.5 刷新（非 OpenID），`api/server/controllers/AuthController.js` 的 `refreshController`
1. 读取 `refreshToken` cookie。如果不存在，返回 200 "Refresh token not provided"。
2. 用 `JWT_REFRESH_SECRET` 校验并加载用户。
3. `findSession({userId, refreshToken})`，对令牌做哈希并要求 `expiration > now`。
4. 如果会话有效，用现有会话再次调用 `setAuthTokens`。刷新 JWT 以相同的过期时间重新签发（令牌滚动，而过期时间不滚动），并签发新的访问令牌。返回 `{token, user}`。
5. 失败情况：
   - 带 `?retry`：403 "No session found"；
   - JWT 已过期：403，重定向到 `/login`；
   - 其他：401。

### 3.6 登出（`api/server/controllers/auth/LogoutController.js`）
1. 从 cookie 中收集刷新令牌；对 OpenID 用户，还从会话中收集。
2. OpenID 用户额外执行：
   - `revokeOpenIDRefreshTokenChain`（对刷新 flight 做墓碑标记）；
   - 删除所有 RefreshTokenBridge；
   - 移除 `session.openidTokens`。
3. `logoutUser` 删除匹配的 Session 文档，并销毁 express-session。
4. 清除 cookie `refreshToken`、`openid_access_token`、`openid_id_token`、`openid_user_id`、`token_provider` 以及 CloudFront cookie。
5. 如果设置了 `OPENID_USE_END_SESSION_ENDPOINT`，返回 `{redirect: end_session_endpoint?post_logout_redirect_uri=...}`。它使用 `id_token_hint`；当 URL 会超过 `OPENID_MAX_LOGOUT_URL_LENGTH`（默认 2000）时，回退为 `logout_hint`+`client_id`。

### 3.7 2FA（TOTP；RFC 6238，SHA-1，30 s，6 位数字，±1 步窗口，恒定时间比较）
文件：`api/server/controllers/TwoFactorController.js`、`api/server/controllers/auth/TwoFactorAuthController.js`、`api/server/services/twoFactorService.js`。

1. **启用：** 生成一个 10 字节的 base32 密钥和 10 个备用码（每个 8 个 hex 字符），备用码存为 `{codeHash:sha256hex, used, usedAt}`。密钥用 `encryptV3`（AES-256-CTR）加密后存入 `pendingTotpSecret`，备用码存入 `pendingBackupCodes`。返回 `otpauthUrl` 和明文备用码。已开启 2FA 时重新注册需要提供 TOTP 或备用码。
2. **验证**（可选）：用待定或当前生效的密钥进行校验。
3. **确认：** 一个有效的 TOTP 会把待定数据提升为 `totpSecret` 和 `backupCodes`，并设置 `twoFactorEnabled=true`。
4. **使用 2FA 登录：** 密码校验成功后，客户端收到 `tempToken`。`POST /2fa/verify-temp {tempToken, token|backupCode}` 校验 JWT，加载 `+totpSecret +backupCodes`，并解密密钥（`v3:` 前缀表示 decryptV3；以冒号分隔的值表示 decryptV2；否则为明文）。备用码会被标记为已使用。然后调用 `setAuthTokens`。
5. **禁用 / 重新生成备用码：** 开启 2FA 时，两者都需要 TOTP 或备用码。

### 3.8 社交 OAuth（Google、Facebook、GitHub、Discord、Apple）
文件：`api/strategies/socialLogin.js`、`api/strategies/process.js`、`api/server/controllers/auth/oauth.js`。

1. `GET /oauth/<provider>` 带着签名的 `state`（即上文的 state 存储）重定向到 IdP。scope：google `openid profile email`；github `user:email read:user`；discord `identify email`；facebook `public_profile`。Apple 和 SAML 的回调是 POST。
2. 回调执行 passport 的 verify 函数，它调用 `getProfileDetails` 获取 email、id、头像、username、name 和 emailVerified。
3. 用基础配置检查邮箱域名。
4. `findUser({<provider>Id: id})`，找不到时回退到 `findUser({email})`。
5. 用租户配置再次检查域名。
6. 确定用户：
   - 已存在且提供商相同的用户：更新头像（除非以 `?manual=true` 结尾）和邮箱。
   - 已存在但提供商不同的用户：以 `AUTH_FAILED` 失败。
   - 新用户：需要 `ALLOW_SOCIAL_REGISTRATION`，然后调用 `createSocialUser`（带余额）。
   - 管理员变体（`existingUsersOnly`）从不创建用户。
7. 中间件链：`setBalanceConfig`、`checkDomainAllowed`、`createOAuthHandler`。该处理器会运行 `checkBan`。如果是重定向到管理面板，它会创建一个**交换码**（见 §7）。否则调用 `setAuthTokens` 并重定向到 `DOMAIN_CLIENT`。
8. 失败时重定向到 `${DOMAIN_CLIENT}/login?error=...`。

### 3.9 OpenID Connect（`api/strategies/openidStrategy.js`，使用 `openid-client` v6）
1. `GET /oauth/openid` 构建授权请求，带随机 state，以及可选的 PKCE（`OPENID_USE_PKCE`）和 nonce（`OPENID_GENERATE_NONCE`）。回调 URL 为 `DOMAIN_SERVER + OPENID_CALLBACK_URL`；时钟容差为 `OPENID_CLOCK_TOLERANCE`（默认 300 s）。
2. 回调交换授权码，然后执行 `processOpenIDAuth(tokenset)`：
   1. 把 claims 与 userinfo 合并。当设置了 `OPENID_ON_BEHALF_FLOW_FOR_USERINFO_REQUIRED` 和 `OPENID_ON_BEHALF_FLOW_USERINFO_SCOPE` 时，userinfo 可能用 OBO 交换得到的令牌获取。
   2. 从 `OPENID_EMAIL_CLAIM` 取邮箱，回退顺序为 email、preferred_username、upn。获取 issuer。
   3. 域名检查。
   4. `findOpenIDUser`（`packages/api/src/auth/openid.ts`）：
      - 主查找按 `openidId=sub` 或 `idOnTheSource=oid`，并与 issuer 绑定。
      - 回退按邮箱查找。如果该邮箱对应的用户属于不同提供商，或存储的 `openidId` 不同，以 `AUTH_FAILED` 失败。如果该邮箱用户没有 `openidId`，则进行迁移。
   5. **必需角色：** `OPENID_REQUIRED_ROLE`（CSV），从 `OPENID_REQUIRED_ROLE_TOKEN_KIND`（`access|id|userinfo`）中的 `OPENID_REQUIRED_ROLE_PARAMETER_PATH` 读取。Azure 组数量超限（group overage）时，通过 OBO 调用 Graph `/me/getMemberObjects` 解析。
   6. username 来自 `OPENID_USERNAME_CLAIM` 或 preferred_username；name 来自 `OPENID_NAME_CLAIM` 或 given+family name。
   7. 创建或更新用户（`provider:'openid'`、`openidId`、`openidIssuer`、`idOnTheSource=oid`）。
   8. **管理员角色映射：** `OPENID_ADMIN_ROLE` 配合 `..._PARAMETER_PATH` 和 `..._TOKEN_KIND`，把用户提升为 ADMIN；角色缺失时把 ADMIN 降为 USER。
   9. 否则执行**角色同步**（`packages/api/src/auth/openidRoleSync.ts`）：`OPENID_ROLE_SYNC_ENABLED`、`_SOURCE`、`_CLAIM`、`_ROLE_PRIORITY`（CSV，不能包含 ADMIN）、`_FALLBACK_ROLE`。它选出优先级最高的匹配自定义角色。
   10. 下载头像并保存用户。
3. 随后 `oauthHandler` 根据 `OPENID_REUSE_TOKENS` 分支：
   - **关闭：** 用 `setAuthTokens` 签发普通的 LibreChat 令牌，并可选地保留 `session.openidLogoutIdToken`。
   - **开启：**
     1. 开启 `USE_ENTRA_ID_FOR_PEOPLE_SEARCH` 时同步 Entra 组成员关系。
     2. `sendOpenIDAuthResponse` 调用 `setOpenIDAuthTokens`：把 `refreshToken` cookie 设为 IdP 刷新令牌，并存储 `session.openidTokens = {accessToken, idToken, refreshToken, browserRefreshToken, expiresAt, lastRefreshedAt, accessTokenExpiresAt, appUserId, openidSubject, openidIssuer, tenantId}`。
     3. 设置标记 cookie `token_provider=openid` 和 `openid_user_id`。
     4. `storeOpenIdSession` 以 IdP 刷新令牌的哈希为键 upsert 一个 `sessions` 文档，以便登出和封禁时可以吊销它。
     5. 应用使用的 bearer 令牌依次为：未过期的 IdP **id_token**、会话中的 id_token、access_token。随后由 `requireJwtAuth` 通过 JWKS 校验。
4. **OpenID 刷新**（`refreshController` 的 OpenID 分支）：
   1. 选择刷新令牌（优先使用会话中的副本；浏览器 cookie 用于检测偏移）。
   2. **复用捷径：** 如果会话在 `OPENID_REUSE_MAX_SESSION_AGE_MS`（默认 15 分钟）内刷新过、存储的 id/access 令牌剩余时间超过缓冲秒数、`openid_user_id` cookie 校验通过且会话身份与用户一致，则直接返回缓存的令牌，不调用 IdP。
   3. 否则调用 `refreshOpenIDUser`，用 `OPENID_SCOPE` 和 `OPENID_REFRESH_AUDIENCE` 执行 `openid-client.refreshTokenGrant`。然后重新解析用户，要求解析出的用户与正在刷新的用户相同，必要时迁移 `openidId`，并发布新令牌。
   4. 如果授权返回 invalid_grant，用另一个（浏览器端的）刷新令牌重试一次。如果仍然失败，尝试 **RefreshTokenBridge**（见下文）。
   5. 最终失败：403 "Invalid OpenID refresh token"，或 401 `{code:'OPENID_SESSION_MISSING'}`。
5. **并发控制机制**（在 Python 中可以简化，但语义需要保留）：
   - `packages/api/src/auth/openid/flight.ts`，`openidRefreshFlight` 集合：分布式的单飞（single-flight）租约，保证每个身份同时只有一个 IdP 刷新在执行，并通过投递认领（delivery claim）防止登出与响应发生竞争。
   - `packages/api/src/auth/openid/bridge.ts`，`refreshTokenBridge` 集合：`{oldRefreshTokenHash, encryptedNewRefreshToken (encryptV2), userId, tenantId, openidIssuer, expiresAt}`。当流式（SSE）过程中的 OBO 刷新轮换了 IdP 刷新令牌、却无法设置 cookie 时写入。宽限期为 `OPENID_REFRESH_BRIDGE_GRACE_MS`（60 s）。
   - `packages/api/src/auth/openid/session.ts`（`createOpenIDSessionTokenProvider`、`refreshOpenIDSession`）：供 MCP/OBO 使用方调用的内联刷新。
6. **Microsoft OBO / Graph**（`api/server/services/OboTokenService.js`、`GraphTokenService.js`）：
   - `genericGrantRequest(config,'urn:ietf:params:oauth:grant-type:jwt-bearer',{assertion: accessToken, scope, requested_token_use:'on_behalf_of'})`。
   - 按（身份元组、scopes、sha256(assertion)）缓存，TTL 带偏移量。进行中的请求会合并，遇到 429/5xx 或网络错误时重试一次。
   - `GET /api/auth/graph-token` 要求 `provider==='openid'` 且开启 `OPENID_REUSE_TOKENS`。
   - `OboPolicyService` 只在作者的角色仍拥有 `MCP_SERVERS.CONFIGURE_OBO` 时，才允许来自数据库的 MCP 配置使用 OBO。

### 3.10 LDAP（`api/strategies/ldapStrategy.js`，`passport-ldapauth`）
1. 环境变量：`LDAP_URL`、`LDAP_BIND_DN`、`LDAP_BIND_CREDENTIALS`、`LDAP_USER_SEARCH_BASE`、`LDAP_SEARCH_FILTER`（默认 `mail={{username}}`）、`LDAP_CA_CERT_PATH`、`LDAP_TLS_REJECT_UNAUTHORIZED`、`LDAP_STARTTLS`，以及属性映射 `LDAP_ID`、`LDAP_USERNAME`、`LDAP_EMAIL`、`LDAP_FULL_NAME`（CSV）。
2. 服务账号绑定、搜索用户，然后用密码做用户绑定，得到 `userinfo`。
3. `ldapId` = `LDAP_ID` 属性，否则依次取 uid、sAMAccountName 或 mail。邮箱缺失时回退为 `<username>@ldap.local`。
4. 域名检查（基础配置和租户配置）。`findUser({ldapId})`。提供商不同则以 `AUTH_FAILED` 失败。
5. 创建用户（第一个用户成为 ADMIN，`emailVerified:true`），或用 LDAP 数据覆盖其资料。然后与本地登录一样运行 `loginController`，包括 2FA。

### 3.11 SAML（`api/strategies/samlStrategy.js`，`@node-saml/passport-saml`）
- 环境变量：`SAML_ENTRY_POINT`、`SAML_ISSUER`、`SAML_CERT`（路径或内联证书）、`SAML_CALLBACK_URL`、`SAML_SESSION_SECRET`、`SAML_USE_AUTHN_RESPONSE_SIGNED`（true 表示响应必须签名，否则断言必须签名）、`SAML_NAME_ID_FORMAT`（拒绝 transient）、`SAML_IDP_ISSUER`，以及 claim 映射 `SAML_*_CLAIM`。
- 流程：解析主体的 NameID，然后按 `samlId` 查找，回退到邮箱。提供商冲突或 NameID 不匹配时以 `AUTH_FAILED` 失败。否则创建用户，或原子地认领该 SAML 身份。随后 `POST /oauth/saml/callback` 运行 `oauthHandler`。

---

## 4. 授权模型

### 4.1 系统角色与角色能力（功能权限）
- `SystemRoles = { ADMIN:'ADMIN', USER:'USER' }`（`packages/data-provider/src/roles.ts`）。允许自定义角色。
- `roles` 集合（`packages/data-schemas/src/schema/role.ts`）为 `{name, description, permissions: {<PermissionType>: {<Permission>: bool}}, tenantId}`，在 `(name, tenantId)` 上唯一。
- 默认值来自 `roleDefaults`。启动时由 `updateInterfacePermissions`（`packages/api/src/app/permissions.ts`）用 YAML 的 `interface.*` 配置覆盖。

`Permissions` 枚举：`USE, CREATE, UPDATE, READ, READ_AUTHOR, SHARE, OPT_OUT, VIEW_USERS, VIEW_GROUPS, VIEW_ROLES, SHARE_PUBLIC, CONFIGURE_OBO`。

`PermissionTypes` 及其权限键和默认值（`packages/data-provider/src/permissions.ts`）：

| PermissionType | 键 | USER 默认值 | ADMIN 默认值 |
|---|---|---|---|
| PROMPTS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | 全部 T |
| BOOKMARKS | USE | T（schema 默认值） | T |
| MEMORIES | USE, CREATE, UPDATE, READ, OPT_OUT | schema 默认值 T | T |
| AGENTS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | T |
| MULTI_CONVO, TEMPORARY_CHAT, RUN_CODE, WEB_SEARCH, FILE_SEARCH, FILE_CITATIONS | USE | T | T |
| PEOPLE_PICKER | VIEW_USERS, VIEW_GROUPS, VIEW_ROLES | F,F,F | T |
| MARKETPLACE | USE | F | T |
| MCP_SERVERS | USE, CREATE, SHARE, SHARE_PUBLIC, CONFIGURE_OBO | T,F,F,F,F | T |
| REMOTE_AGENTS | USE, CREATE, SHARE, SHARE_PUBLIC | 全部 F | T |
| SKILLS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | T |
| SHARED_LINKS | CREATE, SHARE, SHARE_PUBLIC | T,T,T | T |
| SCHEDULES | USE, CREATE | T,T | T |

### 4.2 系统能力（管理 RBAC，`systemgrants` 集合）
- 能力字符串定义在 `packages/data-schemas/src/admin/capabilities.ts`：`access:admin`、`read|manage:users`、`read|manage:groups`、`read|manage:roles`、`read|manage:configs`、`assign:configs`、`read:usage`、`read:insights`、`read|manage:agents`、`manage:mcpservers`、`manage:code_environments`、`read|manage:prompts`、`read|manage:skills`、`read|manage:sharedlinks`、`read|manage:assistants`、`read:audit_log`。
- 动态形式：`manage:configs:<section>`、`read:configs:<section>`、`assign:configs:<user|group|role>`。
- 蕴含关系：`manage:X` 蕴含 `read:X`。分区能力也可以由其父能力满足。
- 授权文档为 `{principalType, principalId (ObjectId for user/group, role-name string for role), capability, tenantId (omitted for platform-level), grantedBy, grantedAt, expiresAt}`，在 `(principalType, principalId, capability, tenantId)` 上唯一（`packages/data-schemas/src/schema/systemGrant.ts`）。
- `seedSystemGrants` 在启动时授予 `ADMIN` 角色全部能力。
- `ResourceCapabilityMap` 把资源类型映射到可绕过其 ACL 的能力：agent 和 remoteAgent 映射到 manage:agents，promptGroup 映射到 manage:prompts，mcpServer 映射到 manage:mcpservers，skill 映射到 manage:skills，sharedLink 映射到 manage:sharedlinks，codeEnvironment 映射到 manage:code_environments。
- 少数旧检查仍使用 `checkAdmin`（`role==='ADMIN'`，`api/server/middleware/roles/admin.js`）。

### 4.3 ACL（按资源共享）
定义在 `packages/data-provider/src/accessPermissions.ts`：
- `PrincipalType`：`user | group | role | public`。`PrincipalModel`：`User | Group | Role`。
- `ResourceType`：`agent, codeEnvironment, promptGroup, mcpServer, remoteAgent, skill, sharedLink`。
- `PermissionBits`：`VIEW=1, EDIT=2, DELETE=4, SHARE=8, VIEW_INSIGHTS=16`。`MAX_PERM_BITS=31`。
- `RoleBits`：`VIEWER=1, EDITOR=3, MANAGER=7, OWNER=15`。
- `AccessRoleIds`（由 `seedDefaultRoles` 写入 `accessroles`，`packages/data-schemas/src/methods/accessRole.ts`）：为 agent、codeEnvironment、promptGroup、mcpServer、remoteAgent 和 skill 提供 `<type>_viewer`（1）、`<type>_editor`（3）、`<type>_owner`（15）。sharedLink 只有 viewer 和 owner。
- `aclentries` 文档（`packages/data-schemas/src/schema/aclEntry.ts`）：`{principalType, principalId (Mixed), principalModel, resourceType, resourceId (ObjectId), permBits (int 0..31), roleId, inheritedFrom, grantedBy, grantedAt, expiredAt (TTL), tenantId}`。
- 资源创建者被授予 `*_owner` 角色。**有效权限是**所有匹配用户主体的条目的 `permBits` **按位或**。
- `hasPermission` 查询 `permBits: {$in: permissionBitSupersets(required)}`，以找出任何满足 `(v & required) === required` 的值 v。这是为兼容 Cosmos DB 而采用的枚举变通方法；在 Python 中可以使用 `$bitsAllSet`，或在 SQL 中计算这个检查。
- **主体解析**（`getUserPrincipals`，`packages/data-schemas/src/methods/userGroup.ts` 第 855 行）：`[{user, ObjectId}, {role, roleName}, ...{group, id} for groups whose memberIds contain (idOnTheSource || userId), {public}]`。
- **组**（`packages/data-schemas/src/schema/group.ts`）：`{name, description, email, avatar, memberIds[string], source:'local'|'entra', idOnTheSource, tenantId}`。Entra 组在登录时从 Graph 同步（`syncUserEntraGroupMemberships`，`api/server/services/PermissionService.js`），受 `USE_ENTRA_ID_FOR_PEOPLE_SEARCH` 和 `OPENID_REUSE_TOKENS` 控制，另有 `ENTRA_ID_INCLUDE_OWNERS_AS_MEMBERS`。
- **租户：** 大多数集合带有 `tenantId`。`tenantIsolation` 是一个 Mongoose 插件（`packages/data-schemas/src/models/plugins/tenantIsolation.ts`），根据 AsyncLocalStorage（`tenantStorage`）自动限定查询范围。`TENANT_ISOLATION_STRICT` 使其变为严格模式，`runAsSystem` 可绕过它。

### 4.4 端到端：(a) 角色能力检查
使用 `packages/api/src/middleware/access.ts` 中的 `generateCheckAccess`，例如 `generateCheckAccess({permissionType: REMOTE_AGENTS, permissions:[USE], getRoleByName})`。
1. `requireJwtAuth` 设置 `req.user`（含 `role`）。
2. 中间件调用 `checkAccess`。如果 `skipCheck(req)` 为 true，直接放行。
3. 没有用户或没有 `user.role`：false。
4. `getRoleByName(user.role)`，按请求记忆化（并缓存在角色缓存中）。
5. `pv = role.permissions[permissionType]`。请求的**所有**权限在 `pv` 中都必须为真值。某个为 false 的权限，如果 `bodyProps[perm]` 列出的请求体字段在 `req.body` 中全部存在，仍然可以通过。
6. 允许则 `next()`。否则返回 403 `{message:'Forbidden: Insufficient permissions'}`。

系统能力检查（`requireCapability(cap)`，`packages/api/src/middleware/capabilities.ts`）：
1. 解析主体（用户、角色、组），由 `capabilityContextMiddleware` 按请求缓存。
2. `SystemGrant.exists({$or: principals (excluding public), capability: {$in:[cap, ...impliers, ...parents]}, tenantId matches OR platform-level})`。
3. 如果设置了 `platformOnly`，只计算拥有平台级授权的 USER 主体。
4. 结果：403 `{message:'Forbidden'}` 或 `next()`。

### 4.5 端到端：(b) 资源 ACL 检查
使用 `api/server/middleware/accessResources/canAccessResource.js` 中的 `canAccessResource({resourceType, requiredPermission, resourceIdParam, idResolver})`。`canAccessAgentResource` 这类包装器会把自定义的 `agent_xxx` id 解析为 ObjectId。
1. 读取 `req.params[resourceIdParam]`；缺失返回 400。没有 `req.user.id` 返回 401。
2. **能力绕过：** 如果 `hasCapability(user, ResourceCapabilityMap[resourceType])`，调用 `next()`。
3. 可选的 `idResolver` 把自定义 id 映射为文档或 `_id`。找不到返回 404。
4. `PermissionService.checkPermission` 调用 `getUserPrincipals({userId, role})`，再调用 `aclEntry.hasPermission(principals, type, id, bits)`。
5. 允许时设置 `req.resourceAccess = {resourceType, resourceId, customResourceId, permission, userId, resourceInfo}` 并调用 `next()`。否则返回 403。

列表类端点使用 `findAccessibleResources`（有效权限位为所需权限超集的资源 ID）或 `getResourcePermissionsMap`（批量）。

### 4.6 共享 API（`/api/permissions`）
路由器：`api/server/routes/accessPermissions.js`。控制器：`api/server/controllers/PermissionsController.js`。

- `GET /search-principals`：需要 PEOPLE_PICKER（VIEW_USERS/GROUPS/ROLES）；搜索本地用户和组，并通过 Graph 搜索 Entra。
- `GET /:resourceType/roles`：列出访问角色。
- `GET /:resourceType/:resourceId`：需要 ACL SHARE（8）。对智能体而言，ADMIN 可绕过此检查。
- `PUT /:resourceType/:resourceId`，请求体为 `{updated:[{type,id,accessRoleId,...}], removed:[...], public, publicAccessRoleId}`。按顺序检查：
  1. ACL SHARE；
  2. 角色级别的 `<PermissionType>.SHARE`（`checkShareAccess`）；
  3. 如果要把资源设为公开，检查 `SHARE_PUBLIC`（`createSharePolicyMiddleware`，`packages/api/src/middleware/share.ts`）；
  4. sharedLink 所有者保护。

  然后确保主体存在（Entra 组按需创建），并调用 `bulkUpdateResourcePermissions`。VIEW_INSIGHTS 的变更会记入审计。
- `GET /:resourceType/effective/all` 和 `GET /:resourceType/:resourceId/effective`：返回调用者的有效权限位。
- `/api/roles`（`api/server/routes/roles.js`）：`GET /:roleName` 对自己的角色或非 ADMIN 的默认角色开放；其他情况需要 `read:roles`。`PUT /:roleName/{prompts,agents,memories,people-picker,mcp-servers,marketplace,remote-agents,skills}` 需要 `manage:roles`。

---

## 5. 管理 API（`/api/admin/*`）
路由位于 `api/server/routes/admin/`；处理器工厂位于 `packages/api/src/admin/`。每个路由器都以 `requireJwtAuth` 和 `requireCapability('access:admin')` 开头。

- **auth.js**（管理面板是位于 `ADMIN_PANEL_URL` 的独立 SPA）：
  - `POST /api/admin/login/local`：本地登录加 `access:admin`。
  - `GET /verify`。
  - `GET /oauth/{openid,google,github,discord,facebook,apple,saml}` 及其回调使用 `*Admin` 策略（仅限已存在的用户）。PKCE challenge 保存 5 分钟，且必须是 64 个 hex 字符。
  - 回调把 `{userId, user, token, refreshToken, origin, codeChallenge, expiresAt}` 以一个 32 字节 hex 码为键放入缓存 `ADMIN_OAUTH_EXCHANGE`，然后带 `?code=` 重定向。
  - `POST /oauth/exchange {code, code_verifier}` 只能使用一次。它检查来源，并用 hex SHA-256 校验 verifier（不是 base64url S256），返回 `{token, refreshToken, user, expiresAt}`（`packages/api/src/auth/exchange.ts`）。
  - `POST /oauth/refresh {refresh_token, user_id?, provider:'openid'|'google'}` 在 IdP 处刷新，然后重新检查 `access:admin`、域名允许列表、issuer 和租户，并签发新的 LibreChat JWT。
- **users.js：** 列表和搜索（`read:users`）。删除功能被注释掉了。
- **roles.js：** CRUD、`PATCH /:name/permissions` 以及成员管理（`read|manage:roles`）。成员即 `role` 等于该角色名的用户。
- **groups.js：** CRUD 和成员管理（`read|manage:groups`）。
- **grants.js：** `GET /`、`GET /effective`、`GET /:type/:id`、`POST /`、`DELETE /:type/:id/:capability`。
  - 授予需要目标主体类型的 manage 能力，且调用者自己必须持有被授予的能力（禁止提权）。
  - 撤销只需要 manage 能力。
  - 两者都以失败即关闭的方式写入审计条目；审计写入失败会回滚授权。
- **config.js：** 按主体的配置覆盖，存放在 `configs` 集合中，文档为 `{principalType, principalId, principalModel, priority, overrides, tombstones, isActive, configVersion, tenantId}`。端点涵盖列表、基础配置、读取、写入、修改字段、对字段做墓碑标记、删除字段、删除和切换启用状态。
- **audit.js**（`read:audit_log`）：`GET /`、`/export.csv`、`/verify`（哈希链校验）、`/:id`。
  - 审计日志（`packages/data-schemas/src/schema/auditLog.ts`）只追加；更新和删除会被钩子阻止。
  - 字段：`{schemaVersion, category, action, outcome, severity, actor{type,id,name}, target{type,id,name}, metadata, context{requestId,ip,userAgent,sessionId}, tenantId, chainKey('__platform__' or tenant), seq, prevHash (genesis '0'*64), hash = sha256(stableStringify(canonical entry)), createdAt}`。
  - 当前的动作：`grant.assigned`、`grant.removed`、`permission.insights_assigned`、`permission.insights_removed`。
- **code.js**（manage:code_environments）、**langfuse.js**、**skills.js**：各自对应特定功能。

---

## 6. API 密钥与用户密钥

### 6.1 智能体 API 密钥，供智能体 API 的外部调用方使用
- 管理路由为 `/api/api-keys`（`api/server/routes/apiKeys.js`），全部位于 requireJwtAuth 和 `REMOTE_AGENTS.USE` 之后：POST 创建 `{name, expiresAt?}`、GET 列表、GET `:id`、DELETE `:id`。
- 密钥生成（`packages/data-schemas/src/methods/agentApiKey.ts`）：`key = 'sk-' + hex(32 random bytes)`，`keyHash = sha256hex(key)`，`keyPrefix = key[0:8]`。
- 密钥只返回一次。数据库存储 `{userId, name, keyHash (select:false), keyPrefix, lastUsedAt, expiresAt (TTL index), tenantId}`（`packages/data-schemas/src/schema/agentApiKey.ts`）。
- **认证路径。** 端点为 `/api/agents/v1/chat/completions`（兼容 OpenAI）、`/api/agents/v1/responses` 和 `/api/agents/v1/models`。中间件链为 `preAuthTenantMiddleware`、`requireRemoteAgentAuth`、`configMiddleware`、`checkRemoteAgentsFeature`（角色 `REMOTE_AGENTS.USE`），执行类路由上还有 `checkAgentPermission`。
  1. `requireRemoteAgentAuth`（`packages/api/src/middleware/remoteAgentAuth.ts`）读取配置 `endpoints.agents.remoteApi.auth`。
  2. 如果 `oidc.enabled`，它通过 JWKS 按配置的 issuer 和 audience 校验 Bearer，检查 scope，并解析或开通用户（可选同步角色）。
  3. OIDC 失败或未启用 OIDC 时，如果 `apiKey.enabled !== false`，回退到 API 密钥。
  4. API 密钥中间件（`createRequireApiKeyAuth`，`packages/api/src/apiKeys/middleware.ts`）：提取 Bearer，`sha256`，`findOne({keyHash})`，过期则拒绝，更新 `lastUsedAt`，加载用户并检查其处于活动状态（不在删除过程中，否则返回 409）。设置 `req.user` 和 `req.apiKeyId`。
  5. 错误使用 OpenAI 的错误格式 `{error:{message,type,code}}`。
  6. `checkAgentPermission` 从 `body.model` 读取智能体 id。如果用户对 `agent` 资源有 SHARE 权限，`getRemoteAgentPermissions` 返回全部权限位；否则使用 `remoteAgent` 资源上的 ACL 权限位。要求具有 VIEW。
- `/api/agents/v1/agents`（管理）使用 `requireAgentManagementAuth`，只支持 OIDC（`endpoints.agents.managementApi.auth`）。

### 6.2 用户自带的提供商密钥（`/api/keys`，`api/server/routes/keys.js`）
- `PUT {name, value, expiresAt}` upsert `keys` 文档 `{userId, name (endpoint), value = encrypt(value), expiresAt (TTL index), tenantId}`。
- `GET ?name=` 返回 `{expiresAt: Date|'never'|null}`。`DELETE /:name` 或 `DELETE ?all=true`。
- `getUserKey` 负责解密；`getUserKeyValues` 还会做 JSON 解析。密钥缺失时抛出 `ErrorTypes.NO_USER_KEY`；值无效时抛出 `INVALID_USER_KEY`（`packages/data-schemas/src/methods/key.ts`）。
- 加密（`packages/data-schemas/src/crypto/index.ts`，由 `packages/api/src/crypto` 再导出），使用 `CREDS_KEY`（hex，32 字节）和 `CREDS_IV`（hex，16 字节）：
  - **v1** `encrypt`/`decrypt`：固定密钥和 IV 的 AES-CBC；hex 输出。用于用户密钥。
  - **v2** `encryptV2`：随机 IV 的 AES-CBC；格式 `ivhex:cthex`。
  - **v3** `encryptV3`：随机 IV 的 AES-256-CTR；格式 `v3:ivhex:cthex`。用于 TOTP 密钥。
  - `hashToken`：SHA-256 hex。`hashBackupCode`：SHA-256 hex。
- 插件凭据单独存放在 `pluginAuth` 中。

---

## 7. 余额与交易记录

**配置**（librechat.yaml 中的 `balance`，`packages/data-provider/src/config.ts` 第 2654 行）：`{enabled=false, startBalance=20000, autoRefillEnabled=false, refillIntervalValue=30, refillIntervalUnit='days', refillAmount=10000, reservationTtlMs}`。另有 `transactions.enabled=true`。配置可以按租户、角色或用户而不同。

**余额文档**（`packages/data-schemas/src/schema/balance.ts`）：`{user, tokenCredits (1000 credits = $0.001, i.e. 1 credit = $1e-6), autoRefillEnabled, refillIntervalValue, refillIntervalUnit, lastRefill, refillAmount, reservations[{id,amount,expiresAt}], reservedCredits, pendingRefill{transactionId,rawAmount}, tenantId}`。

**认证时同步：** `createSetBalanceConfig`（`packages/api/src/middleware/balance.ts`）在登录、OAuth 回调和 `GET /api/balance` 时运行。记录不存在时以 `startBalance` 创建，并把配置中的自动充值设置复制到文档中（使用按用户的 promise 锁）。余额功能关闭时，`GET /api/balance` 返回 204；否则返回该文档，自动充值关闭时会去掉充值相关字段。

**预检准入：** `checkBalance`（`packages/api/src/middleware/checkBalance.ts`）。
1. `tokenCost = promptTokens * getMultiplier({valueKey|model|endpoint, tokenType, endpointTokenConfig})`。
2. `reserveBalance` 执行原子的条件更新：对 reservations 做 `$push`，对 reservedCredits 做 `$inc`，仅在 `tokenCredits - reservedCredits >= amount` 时匹配。
3. 如果有到期的自动充值，先通过带栅栏的写入应用它，其账本记录 `Transaction{tokenType:'credits', context:'autoRefill', rawAmount}` 通过 `pendingRefill` 幂等地写入。
4. 预留失败时，`logViolation(TOKEN_BALANCE)` 并抛出 JSON 错误 `{type:'token_balance', balance, tokenCost, promptTokens}`。
5. 预留在请求结束时释放，或在 TTL 到期后被清理。

**扣费**（`packages/data-schemas/src/methods/spendTokens.ts`、`packages/data-schemas/src/methods/transaction.ts`、`packages/data-schemas/src/methods/tx.ts`）：
- `spendTokens(txData, {promptTokens, completionTokens})` 创建两个 `Transaction` 文档，`rawAmount = -tokens`。
- `calculateTokenValue`：`rate = |getMultiplier(...)|`，`tokenValue = rawAmount * rate`。如果 `tokenType==='completion'` 且 `context==='incomplete'`（已中止），则 `tokenValue = ceil(tokenValue * 1.15)`（cancelRate）。
- **结构化计费（提示缓存）：** `tokenValue = -(input*inMult + write*writeMult + read*readMult)`。缓存倍率来自 `getCacheMultiplier`，缺失时回退到输入倍率。`rate` 为加权平均值。
- **倍率解析：**
  1. `endpointTokenConfig[model][tokenType]`；
  2. 如果 `inputTokenCount > threshold`，使用高价的长上下文费率（例如 `gpt-5.4`：阈值 272000）；
  3. `tokenValues[valueKey][tokenType]`，其中 `valueKey = getValueKey(model, endpoint)`（对模型名做模式匹配）；
  4. 默认费率 6。

  费率单位为每 1M token 的美元价格，数值上等于每 token 的 credit 数。
- 保存交易记录。如果开启了余额功能，`updateBalance` 使用乐观并发：`findOneAndUpdate({_id, tokenCredits: current}, {$set:{tokenCredits: max(0, current + tokenValue)}})`，最多重试 10 次，采用带抖动的指数退避。
- **交易记录文档：** `{user, conversationId, tokenType:'prompt'|'completion'|'credits', model, context, valueKey, rate, rawAmount, tokenValue, inputTokens, writeTokens, readTokens, messageId, tenantId, timestamps}`。
- **公式：** `new_credits = max(0, credits + Σ(rawAmount_i × rate_i × (1.15 if incomplete completion)))`。扣费时 `rawAmount` 为负，充值时为正。

---

## 8. 用户、封禁与账户删除

- **用户文档**（`packages/data-schemas/src/schema/user.ts`）：`name, username, email (unique per tenant), emailVerified, password (select:false), avatar, provider, role, googleId/facebookId/openidId/openidIssuer/samlId/ldapId/githubId/discordId/appleId, plugins, twoFactorEnabled, totpSecret, backupCodes[{codeHash,used,usedAt}], pendingTotpSecret, pendingBackupCodes, refreshToken (legacy), expiresAt (TTL 7 days), termsAccepted/At, agentTriggerDeletionStartedAt, subagentAdmissionFences, personalization, favorites, pinnedOrder, skillStates, idOnTheSource, tenantId`。
- **`/api/user`**（`api/server/routes/user.js`）：
  - `GET /` 返回脱敏后的用户，必要时刷新 S3 头像 URL。
  - `PATCH /preferences`；`GET /terms`、`POST /terms/accept`。
  - `POST /plugins {pluginKey, action, auth}`：安装或卸载工具，并存储加密的插件认证信息。卸载时会断开 MCP 连接。
  - `DELETE /delete`：需要 `canDeleteAccount`（`ALLOW_ACCOUNT_DELETION`，默认 true；否则需要 `access:admin`），开启 2FA 时还需要 TOTP 或备用码。
  - `POST /verify`、`POST /verify/resend`。
- **级联删除**（`api/server/controllers/UserController.js`，约第 400 行）：先给智能体触发器加栅栏，排空排队中的任务并停止正在进行的运行，然后按顺序删除：
  1. 对话和检查点、消息；
  2. 会话、交易记录、密钥、余额、预设；
  3. 插件认证、分享链接、文件（存储和数据库）、工具调用；
  4. 智能体、智能体 API 密钥、assistants、对话标签；
  5. 记忆、提示词、技能、MCP 服务器、actions、令牌；
  6. 组成员关系、`principalId=user` 的 ACL 条目、定时任务；
  7. 最后删除用户本身，并吊销代码环境工作进程。
- **封禁**（`api/server/middleware/checkBan.js`、`api/cache/logViolation.js`、`api/cache/banViolation.js`）：
  - 由 `BAN_VIOLATIONS` 启用。
  - 限流器和其他检查调用 `logViolation(type, score)`，累加按用户的违规计数。
  - 每当计数越过 `BAN_INTERVAL`（默认 20）的某个倍数，就为用户 id 和 IP 写入封禁，时长为 `BAN_DURATION`（默认 2 小时）。所有会话被删除，认证 cookie 被清除。
  - `checkBan` 检查 IP 和 userId（或由 `body.email` 找到的用户），在封禁存储前面使用一个内存 Keyv 缓存。它返回 403 "Your account has been temporarily banned…"，对 OAuth 导航则执行重定向。

---

## 9. Python 重新实现建议

- **框架与依赖：** 使用 FastAPI 的 `Depends`：
  - `get_current_user` 尝试 JWT（PyJWT 或 python-jose，HS256 `JWT_SECRET`），对 OpenID 复用路径则尝试经 JWKS 校验的 RS256 令牌（`PyJWKClient`，缓存 JWKS）。然后加载用户，并在 `contextvars.ContextVar` 中设置租户上下文。
  - `optional_user`、`require_capability(cap)`、`require_role_permission(ptype, [perms])` 和 `require_resource(resource_type, bits, id_param, resolver)` 对应 4.4 和 4.5 节。
- **密码：** `bcrypt` 或 `passlib[bcrypt]`，cost 10，以兼容现有哈希。
- **刷新会话：** 保持相同的设计，即刷新 JWT `{id, sessionId}` 加一个保存 `sha256hex(refresh)` 和 `expiration` 的 `sessions` 表或集合。Cookie：`response.set_cookie('refreshToken', httponly=True, samesite='strict', secure=...)`。
- **OAuth/OIDC：** 用 `authlib`（Starlette 集成）支持 Google、GitHub、Discord、Facebook、Apple 和通用 OIDC（PKCE、nonce、`refresh_token` 授权、`jwt-bearer` OBO 授权）。保留与 `__Host-` cookie 绑定的 HMAC 签名 `state`。`openidTokens` 使用 `starlette-session` 或基于 Redis 的会话。Redis 锁（`SET NX PX`）可以替代 flight 和 bridge 集合。
- **LDAP：** `ldap3`。服务账号绑定，用 `LDAP_SEARCH_FILTER` 搜索（`{{username}}` 用经 `ldap3.utils.conv.escape_filter_chars` 转义后的值替换），然后以用户 DN 绑定。支持 StartTLS 和 CA 文件。
- **SAML：** `python3-saml`（OneLogin）或 `pysaml2`。按 `SAML_USE_AUTHN_RESPONSE_SIGNED` 强制要求断言或响应签名，并拒绝 transient NameID。
- **2FA：** `pyotp.TOTP(secret, digits=6, interval=30).verify(code, valid_window=1)`。备用码为 `secrets.token_hex(4)`，以 SHA-256 存储。用 `cryptography` 的 AES-256-CTR 加密密钥，采用 `v3:iv:ct` 格式，以便读取现有数据。
- **加密兼容：** 用 `cryptography.hazmat` 实现全部三种 AES 格式（固定 IV 的 CBC、随机 IV 的 CBC `iv:ct`、CTR `v3:`），使用 `CREDS_KEY` 和 `CREDS_IV`。
- **ACL：** 把 `perm_bits` 存为整数。检查条件是对用户各主体条目取 OR 后满足 `(perm_bits & required) == required`。SQL 中：按资源分组做 `bit_or(perm_bits)`。Mongo 中：`$bitsAllSet`。
- **限流与封禁：** 使用基于 Redis 的 `slowapi` 或 `limits`，违规计数和封禁键存放在 Redis 中并带 TTL。
- **余额：** 用一条语句完成原子的条件更新，例如 `UPDATE balances SET token_credits = GREATEST(0, token_credits + :delta) WHERE id=:id AND token_credits=:old` 配合重试循环，或在 Postgres 中使用 `SELECT … FOR UPDATE`。把预留实现为带 `expires_at` 的行。把 `packages/data-schemas/src/methods/tx.ts` 中的 `tokenValues`、高价费率表和缓存费率表作为数据移植。
- **审计日志：** 一张只追加的表，`(chain_key, seq)` 唯一，`hash = sha256(canonical_json(entry_without_hash) incl. prevHash)`。
