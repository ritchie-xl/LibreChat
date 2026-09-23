# Authentication, Sessions, Authorization, Admin, API Keys & Balance

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

The server uses Express, Passport and Mongoose. Most of the logic lives in `packages/api/src` (TS, `@librechat/api`), with schemas and DB methods in `packages/data-schemas/src` and shared enums in `packages/data-provider/src`. The `api/server` layer is mostly thin wiring.

---

## 1. Token and session model

### 1.1 Credentials

| Artifact | What it is | Secret / storage | Lifetime |
|---|---|---|---|
| **Access token** ("token" in JSON responses) | HS256 JWT `{id, username, provider, email}` | `JWT_SECRET`; client holds it in memory and sends `Authorization: Bearer` | `SESSION_EXPIRY` ms, default 15 min (`DEFAULT_SESSION_EXPIRY`, `packages/data-schemas/src/methods/user.ts` `generateToken`) |
| **Refresh token** (local, LDAP, social, SAML, and OpenID without reuse) | HS256 JWT `{id, sessionId}` | `JWT_REFRESH_SECRET`; `refreshToken` cookie | `REFRESH_TOKEN_EXPIRY` ms, default 7 days (`DEFAULT_REFRESH_TOKEN_EXPIRY`, `packages/data-schemas/src/methods/session.ts`) |
| **Session doc** (`sessions` collection) | `{refreshTokenHash (SHA-256 hex), expiration (TTL index, expires:0), user, tenantId}`, unique index on `(user, refreshTokenHash)` | Mongo, `packages/data-schemas/src/schema/session.ts` | = refresh expiry |
| **2FA temp token** | JWT `{userId, twoFAPending:true}` | `JWT_SECRET` | 5 min (`api/server/services/twoFactorService.js` `generate2FATempToken`) |
| **Short-lived token** | JWT `{id}` | `JWT_SECRET` | 5 min (`generateShortLivedToken`, `packages/api/src/crypto/jwt.ts`) |
| **Agent trigger token** | JWT `{id, scope:'agent_trigger'}` | `JWT_SECRET` | 60 s. Only valid on `POST /api/agents/chat/agents` and `/api/agents/chat/steer/deliver`, and only with header `x-lc-agent-trigger: 1` (`api/strategies/jwtStrategy.js`) |
| **OpenID express-session** | `req.session.openidTokens` in a Keyv/Redis store (`CacheKeys.OPENID_SESSION`), signed by `OPENID_SESSION_SECRET`, cookie `connect.sid` | server-side | maxAge = max(SESSION_EXPIRY, OPENID_REUSE_MAX_SESSION_AGE_MS) when reuse is on (`api/server/socialLogins.js`) |
| **SAML express-session** | store `CacheKeys.SAML_SESSION`, secret `SAML_SESSION_SECRET` | | SESSION_EXPIRY |

### 1.2 Cookies

All cookies are `httpOnly`. Auth cookies use `sameSite:'strict'`. `secure` comes from `shouldUseSecureCookie()` in `packages/api/src/oauth/csrf.ts`: `SESSION_COOKIE_SECURE` overrides; otherwise it is `NODE_ENV==='production'` and DOMAIN_SERVER is not a localhost host.

- `refreshToken`: the LibreChat refresh JWT, or the raw IdP refresh token for OpenID.
- `token_provider`: `'librechat'` or `'openid'`. This selects the refresh and auth strategy.
- `openid_user_id`: a JWT signed with `JWT_REFRESH_SECRET`, payload `{id, refreshTokenHash: sha256-base64url(refreshToken)}`. It is only set when `OPENID_REUSE_TOKENS` is on. It lets the unauthenticated `/refresh` endpoint know the user id, and it binds that id to the refresh token (`setOpenIDMarkerCookies`).
- `openid_access_token` / `openid_id_token`: legacy fallbacks, used only when no express-session exists.
- OAuth login state: `[__Host-]oauth_state_<provider>.<id>`. This is a random binding held in a SameSite=Lax cookie. The `state` parameter is `issuedAtBase36.nonce.HMAC_SHA256(JWT_SECRET, "librechat:oauth-login-state:<provider>:<binding>:<issuedAt>:<nonce>")`, and its TTL is `registration.oauthStateTtlMs` (`packages/api/src/oauth/state.ts`). Apple uses SameSite=None.
- MCP/Action OAuth: `oauth_csrf` (HMAC of flowId, 10 min, Lax) and `oauth_session` (HMAC of userId, 24 h, path `/api`).
- CloudFront signed cookies for images are optional (`setCloudFrontAuthCookies`, `api/server/services/AuthService.js`).

### 1.3 Email and password-reset tokens (`tokens` collection)

The schema is `packages/data-schemas/src/schema/token.ts`: `{userId, email(lowercase,trim), type, identifier, token, createdAt, expiresAt (TTL), metadata, tenantId}`.

- **Email verification and password reset:** generate 32 random bytes as hex, store `bcrypt(token,10)`, `expiresIn: 900` s. `type` is `email_verification` or `password_reset`. Legacy docs have `type:null` and are still accepted.
- **Invites:** stored as a SHA-256 hash of the token plus the email.
- The same collection also stores OAuth tokens for MCP and Actions (`type:'oauth'`, encrypted).

---

## 2. Passport strategies and middleware

Registration happens in `api/server/index.js` (around line 376) and `api/server/socialLogins.js`:

- `jwt` (always)
- `local` (always)
- `ldapauth` (when `LDAP_URL` and `LDAP_USER_SEARCH_BASE` are set)
- social logins, only when `ALLOW_SOCIAL_LOGIN` is true:
  - `google`, `facebook`, `github`, `discord`, `apple`, each with an `*Admin` variant
  - `openid` and `openidAdmin` (need `OPENID_CLIENT_ID`, `OPENID_ISSUER`, `OPENID_SCOPE` and `OPENID_SESSION_SECRET`, plus either `OPENID_CLIENT_SECRET` or `OPENID_USE_PKCE`)
  - `openidJwt`, registered only when `OPENID_REUSE_TOKENS` is on
  - `saml` and `samlAdmin`

**`requireJwtAuth`** (`api/server/middleware/requireJwtAuth.js`):
1. Parse cookies. If `token_provider==='openid'`, `OPENID_REUSE_TOKENS` is on, the `openidJwt` strategy exists and the `openid_user_id` cookie verifies, the strategy chain is `['openidJwt','jwt']`. Otherwise it is `['jwt']`.
2. Try each strategy in turn. If `openidJwt` resolves a user whose id differs from the `openid_user_id` cookie, fall through to the next strategy.
3. On failure, return 401 `{message}` with `code: ACCOUNT_DELETION_IN_PROGRESS` when that applies.
4. On success, set `req.user` and `req.authStrategy`, then chain `tenantContextMiddleware` (AsyncLocalStorage tenant/user context) and optionally refresh CloudFront cookies.

**`optionalJwtAuth`** does the same thing but never fails.

**`jwtStrategy`** (`api/strategies/jwtStrategy.js`) reads the Bearer token with `JWT_SECRET`:
- Rejects trigger-scoped tokens outside the admission routes.
- Loads the user with `runAsSystem`, excluding password, totpSecret and backupCodes.
- Rejects users with `agentTriggerDeletionStartedAt`.
- Sets `user.id`, and defaults `role=USER` (persisting it) when the role is missing.

**`openIdJwtStrategy`** (`api/strategies/openIdJwtStrategy.js`):
- Verifies the Bearer token through JWKS (`jwks-rsa`; cache controlled by `OPENID_JWKS_URL_CACHE_ENABLED` and `OPENID_JWKS_URL_CACHE_TIME`).
- Audience is `OPENID_CLIENT_ID` plus `OPENID_AUDIENCE` (CSV).
- The issuer must match discovery. A `{tenantid}` template is supported for multi-tenant Entra.
- Resolves the user through `findOpenIDUser`, with an optional auth-user doc cache (`CacheKeys.AUTH_USER_DOC`).
- Sets `user.federatedTokens = {access_token, id_token, refresh_token, expires_at}`, taken from `req.session.openidTokens`, then from cookies, then from the raw bearer if it is identifiable as an access token.

Other auth middleware:
- `requireLocalAuth`: no user returns 404; any `info.message` (for example "Email not verified.") returns 422.
- `requireLdapAuth`: same pattern.
- `setTwoFactorTempUser`: sets `req.user={id}` from the temp token so the limiter and ban check can key on the user.
- `requireSameOrigin`: checks Origin/Referer against `DOMAIN_CLIENT`, `DOMAIN_SERVER` and `ADMIN_PANEL_URL`.
- `preAuthTenantMiddleware`: reads the `X-Tenant-Id` header and rejects the `__SYSTEM__` sentinel.

---

## 3. Auth routes and flows

The `/api/auth` router is `api/server/routes/auth.js`. The `/oauth` router is `api/server/routes/oauth.js`.

| Route | Chain |
|---|---|
| POST `/api/auth/login` | logHeaders, requireSameOrigin, loginLimiter (`LOGIN_WINDOW`=5 min, `LOGIN_MAX`=7), checkBan, validateEmailLogin (`ALLOW_EMAIL_LOGIN`, `ALLOW_EMAIL_LOGIN_OVERRIDE`), requireLdapAuth or requireLocalAuth, setBalanceConfig, loginController |
| POST `/api/auth/logout` | requireJwtAuth, logoutController |
| POST `/api/auth/refresh` | refreshController (unauthenticated, cookie-driven) |
| POST `/api/auth/register` | registerLimiter, checkBan, checkInviteUser, validateRegistration (`ALLOW_REGISTRATION` or a valid invite), registrationController |
| POST `/api/auth/requestPasswordReset`, `/resetPassword` | limiters, checkBan, validatePasswordReset |
| POST `/api/auth/2fa/{enable,verify,confirm,disable,backup/regenerate}` | requireJwtAuth |
| POST `/api/auth/2fa/verify-temp` | requireSameOrigin, setTwoFactorTempUser, twoFactorTempLimiter, checkBan |
| GET `/api/auth/graph-token?scopes=` | requireJwtAuth; Microsoft OBO |
| POST `/api/auth/cloudfront/refresh` | requireJwtAuth |

### 3.1 Local login
1. `passport-local` reads `usernameField:'email'` and validates `loginSchema` (email, password length `MIN_PASSWORD_LENGTH` (default 8) to 128) (`api/strategies/localStrategy.js`, `api/strategies/validators.js`).
2. `findUser({email}, '+password')`. If the user is missing or has no password, fail with "Email does not exist." Compare with `bcrypt.compare`.
3. Legacy auto-verify: if email is not configured and the user was created before 2024-06-07, mark `emailVerified`.
4. If the user is unverified and `ALLOW_UNVERIFIED_EMAIL_LOGIN` is off, fail with "Email not verified." (422).
5. `loginController` (`api/server/controllers/auth/LoginController.js`): if `user.twoFactorEnabled`, return `{twoFAPending:true, tempToken}` and stop.
6. Otherwise call `setAuthTokens(userId,res,null,req)`:
   1. `createSession(userId,{expiresIn:REFRESH_TOKEN_EXPIRY})`.
   2. `generateRefreshToken` signs `{id, sessionId}` and stores its SHA-256 hash on the session.
   3. `generateToken` builds the access JWT.
   4. Set the cookies `refreshToken` and `token_provider=librechat` (expiry = session.expiration).
   5. Return `{token, user}` without password, totpSecret or `__v`.

### 3.2 Register
1. `registerSchema` checks name 3 to 80 characters, an optional username, email, password and confirm_password.
2. Check `registration.allowedDomains` from the tenant app config.
3. If the email already exists, sleep 1 s and return a generic 200 (anti-enumeration).
4. The first user in an untenanted deployment gets role `ADMIN`; everyone else gets `USER`. Hash the password with bcrypt (salt 10).
5. `createUser(data, appConfig.balance, disableTTL=ALLOW_UNVERIFIED_EMAIL_LOGIN, true)`. If the TTL is not disabled, the user doc has `expiresAt` = 7 days (TTL index), so unverified accounts self-delete. A Balance record is created with `startBalance` if balance is enabled.
6. If email is configured, send a verification email (link `${DOMAIN_CLIENT}/verify?token=&email=`). Otherwise set `emailVerified=true`.
7. If an invite was used and the user was actually created, delete the invite token.

### 3.3 Email verification
`POST /api/user/verify {email, token}`:
1. Find the user and the newest verification token.
2. Check the token's userId matches, then `bcrypt.compare`.
3. Set `emailVerified=true` and delete the token.

`POST /api/user/verify/resend` regenerates the token. Both return generic messages.

### 3.4 Password reset
1. `requestPasswordReset` runs a domain check against the base config (a blocked domain returns 400), then finds the user and re-checks the tenant config (a blocked domain now returns a generic 200).
2. Delete old reset tokens and create a new bcrypt-hashed token (15 min).
3. Link: `${DOMAIN_CLIENT}/reset-password?token=&userId=`. If email is not configured, the link is returned in the JSON.
4. `resetPassword(userId, token, password)` compares with bcrypt, sets `password=bcrypt(pw,10)`, sends a confirmation email and deletes the token.
5. The controller then calls `deleteAllUserSessions({userId})`, which revokes all refresh tokens.

### 3.5 Refresh (non-OpenID), `api/server/controllers/AuthController.js` `refreshController`
1. Read the `refreshToken` cookie. If absent, return 200 "Refresh token not provided".
2. Verify it with `JWT_REFRESH_SECRET` and load the user.
3. `findSession({userId, refreshToken})`, which hashes the token and requires `expiration > now`.
4. If the session is valid, call `setAuthTokens` again with the existing session. The refresh JWT is re-signed with the same expiration (a sliding token, not a sliding expiry) and a new access token is minted. Return `{token, user}`.
5. Failure cases:
   - with `?retry`: 403 "No session found";
   - JWT expired: 403 redirect to `/login`;
   - otherwise: 401.

### 3.6 Logout (`api/server/controllers/auth/LogoutController.js`)
1. Collect refresh tokens from the cookie and, for OpenID users, from the session.
2. OpenID users additionally get:
   - `revokeOpenIDRefreshTokenChain` (tombstones refresh flights);
   - deletion of all RefreshTokenBridges;
   - removal of `session.openidTokens`.
3. `logoutUser` deletes the matching Session doc(s) and destroys the express-session.
4. Clear the cookies `refreshToken`, `openid_access_token`, `openid_id_token`, `openid_user_id`, `token_provider` and the CloudFront cookies.
5. If `OPENID_USE_END_SESSION_ENDPOINT` is set, return `{redirect: end_session_endpoint?post_logout_redirect_uri=...}`. It uses `id_token_hint`, or falls back to `logout_hint`+`client_id` when the URL would exceed `OPENID_MAX_LOGOUT_URL_LENGTH` (default 2000).

### 3.7 2FA (TOTP; RFC 6238, SHA-1, 30 s, 6 digits, ±1 step window, constant-time compare)
Files: `api/server/controllers/TwoFactorController.js`, `api/server/controllers/auth/TwoFactorAuthController.js`, `api/server/services/twoFactorService.js`.

1. **Enable:** generate a 10-byte base32 secret and 10 backup codes (8 hex chars each), stored as `{codeHash:sha256hex, used, usedAt}`. The secret is encrypted with `encryptV3` (AES-256-CTR) into `pendingTotpSecret`, and the codes go to `pendingBackupCodes`. Return `otpauthUrl` and the plain codes. Re-enrolling when 2FA is already on requires a TOTP or backup code.
2. **Verify** (optional): check against the pending or active secret.
3. **Confirm:** a valid TOTP promotes pending to `totpSecret` and `backupCodes`, and sets `twoFactorEnabled=true`.
4. **Login with 2FA:** after password success the client receives a `tempToken`. `POST /2fa/verify-temp {tempToken, token|backupCode}` verifies the JWT, loads `+totpSecret +backupCodes`, and decrypts the secret (a `v3:` prefix means decryptV3; a colon-separated value means decryptV2; otherwise it is plaintext). A backup code is marked used. Then `setAuthTokens`.
5. **Disable / regenerate backup codes:** both require a TOTP or backup code when 2FA is enabled.

### 3.8 Social OAuth (Google, Facebook, GitHub, Discord, Apple)
Files: `api/strategies/socialLogin.js`, `api/strategies/process.js`, `api/server/controllers/auth/oauth.js`.

1. `GET /oauth/<provider>` redirects to the IdP with a signed `state` (the state store above). Scopes: google `openid profile email`; github `user:email read:user`; discord `identify email`; facebook `public_profile`. Apple and SAML callbacks are POST.
2. The callback runs the passport verify function, which calls `getProfileDetails` to get email, id, avatar, username, name and emailVerified.
3. Check the email domain against the base config.
4. `findUser({<provider>Id: id})`, falling back to `findUser({email})`.
5. Re-check the domain against the tenant config.
6. Resolve the user:
   - Existing user with the same provider: update the avatar (unless it ends in `?manual=true`) and the email.
   - Existing user with a different provider: fail with `AUTH_FAILED`.
   - New user: requires `ALLOW_SOCIAL_REGISTRATION`, then `createSocialUser` (with balance).
   - Admin variants (`existingUsersOnly`) never create users.
7. Chain: `setBalanceConfig`, `checkDomainAllowed`, `createOAuthHandler`. The handler runs `checkBan`. For an admin-panel redirect it creates an **exchange code** (see §7). Otherwise it calls `setAuthTokens` and redirects to `DOMAIN_CLIENT`.
8. Failures redirect to `${DOMAIN_CLIENT}/login?error=...`.

### 3.9 OpenID Connect (`api/strategies/openidStrategy.js`, uses `openid-client` v6)
1. `GET /oauth/openid` builds an authorization request with a random state and optional PKCE (`OPENID_USE_PKCE`) and nonce (`OPENID_GENERATE_NONCE`). Callback URL is `DOMAIN_SERVER + OPENID_CALLBACK_URL`; clock tolerance is `OPENID_CLOCK_TOLERANCE` (default 300 s).
2. The callback exchanges the code, then `processOpenIDAuth(tokenset)` runs:
   1. Merge the claims with userinfo. Userinfo may be fetched with an OBO-exchanged token when `OPENID_ON_BEHALF_FLOW_FOR_USERINFO_REQUIRED` and `OPENID_ON_BEHALF_FLOW_USERINFO_SCOPE` are set.
   2. Get the email from `OPENID_EMAIL_CLAIM`, falling back to email, preferred_username or upn. Get the issuer.
   3. Domain check.
   4. `findOpenIDUser` (`packages/api/src/auth/openid.ts`):
      - Primary lookup by `openidId=sub`, or `idOnTheSource=oid`, bound to the issuer.
      - Fallback lookup by email. If the email user has a different provider, or a different stored `openidId`, fail with `AUTH_FAILED`. If the email user has no `openidId`, migrate it.
   5. **Required role:** `OPENID_REQUIRED_ROLE` (CSV), read from `OPENID_REQUIRED_ROLE_PARAMETER_PATH` in `OPENID_REQUIRED_ROLE_TOKEN_KIND` (`access|id|userinfo`). Azure group overage is resolved through Graph `/me/getMemberObjects` via OBO.
   6. Username comes from `OPENID_USERNAME_CLAIM` or preferred_username; the name from `OPENID_NAME_CLAIM` or given+family name.
   7. Create or update the user (`provider:'openid'`, `openidId`, `openidIssuer`, `idOnTheSource=oid`).
   8. **Admin role mapping:** `OPENID_ADMIN_ROLE` with `..._PARAMETER_PATH` and `..._TOKEN_KIND` promotes to ADMIN, or demotes ADMIN to USER when the role is absent.
   9. Otherwise run **role sync** (`packages/api/src/auth/openidRoleSync.ts`): `OPENID_ROLE_SYNC_ENABLED`, `_SOURCE`, `_CLAIM`, `_ROLE_PRIORITY` (CSV, cannot include ADMIN), `_FALLBACK_ROLE`. It picks the highest-priority matching custom role.
   10. Download the avatar and save the user.
3. `oauthHandler` then branches on `OPENID_REUSE_TOKENS`:
   - **Off:** issue normal LibreChat tokens with `setAuthTokens`, and optionally keep `session.openidLogoutIdToken`.
   - **On:**
     1. Sync Entra group memberships when `USE_ENTRA_ID_FOR_PEOPLE_SEARCH` is on.
     2. `sendOpenIDAuthResponse` calls `setOpenIDAuthTokens`: set the `refreshToken` cookie to the IdP refresh token and store `session.openidTokens = {accessToken, idToken, refreshToken, browserRefreshToken, expiresAt, lastRefreshedAt, accessTokenExpiresAt, appUserId, openidSubject, openidIssuer, tenantId}`.
     3. Set the marker cookies `token_provider=openid` and `openid_user_id`.
     4. `storeOpenIdSession` upserts a `sessions` doc keyed by the hash of the IdP refresh token, so logout and bans can revoke it.
     5. The app bearer token is the IdP **id_token** if it is unexpired, else the session id_token, else the access_token. `requireJwtAuth` then validates it through JWKS.
4. **OpenID refresh** (`refreshController`, OpenID branch):
   1. Pick the refresh token (the session copy is preferred; the browser cookie is used for drift detection).
   2. **Reuse shortcut:** if the session was refreshed within `OPENID_REUSE_MAX_SESSION_AGE_MS` (default 15 min), the stored id/access token has more than the buffer seconds left, the `openid_user_id` cookie verifies and the session identity matches the user, return the cached token without calling the IdP.
   3. Otherwise call `refreshOpenIDUser`, which runs `openid-client.refreshTokenGrant` with `OPENID_SCOPE` and `OPENID_REFRESH_AUDIENCE`. It then re-resolves the user and requires the resolved user to equal the one being refreshed, migrates `openidId` if needed, and publishes the new tokens.
   4. If the grant returns invalid_grant, retry once with the distinct browser refresh token. If that fails too, try the **RefreshTokenBridge** (below).
   5. Final failures: 403 "Invalid OpenID refresh token", or 401 `{code:'OPENID_SESSION_MISSING'}`.
5. **Concurrency machinery** (can be simplified in Python but the semantics should be kept):
   - `packages/api/src/auth/openid/flight.ts`, `openidRefreshFlight` collection: a distributed single-flight lease so only one IdP refresh runs per identity, with delivery claims so logout can't race a response.
   - `packages/api/src/auth/openid/bridge.ts`, `refreshTokenBridge` collection: `{oldRefreshTokenHash, encryptedNewRefreshToken (encryptV2), userId, tenantId, openidIssuer, expiresAt}`. It is written when a mid-stream (SSE) OBO refresh rotated the IdP refresh token but could not set the cookie. Grace period is `OPENID_REFRESH_BRIDGE_GRACE_MS` (60 s).
   - `packages/api/src/auth/openid/session.ts` (`createOpenIDSessionTokenProvider`, `refreshOpenIDSession`): an inline refresh used by MCP/OBO consumers.
6. **Microsoft OBO / Graph** (`api/server/services/OboTokenService.js`, `GraphTokenService.js`):
   - `genericGrantRequest(config,'urn:ietf:params:oauth:grant-type:jwt-bearer',{assertion: accessToken, scope, requested_token_use:'on_behalf_of'})`.
   - Cached by (identity tuple, scopes, sha256(assertion)) with a skewed TTL. In-flight requests are coalesced, with one retry on 429/5xx or network errors.
   - `GET /api/auth/graph-token` requires `provider==='openid'` and `OPENID_REUSE_TOKENS`.
   - `OboPolicyService` allows OBO for DB-sourced MCP configs only while the author's role still has `MCP_SERVERS.CONFIGURE_OBO`.

### 3.10 LDAP (`api/strategies/ldapStrategy.js`, `passport-ldapauth`)
1. Env: `LDAP_URL`, `LDAP_BIND_DN`, `LDAP_BIND_CREDENTIALS`, `LDAP_USER_SEARCH_BASE`, `LDAP_SEARCH_FILTER` (default `mail={{username}}`), `LDAP_CA_CERT_PATH`, `LDAP_TLS_REJECT_UNAUTHORIZED`, `LDAP_STARTTLS`, and the attribute maps `LDAP_ID`, `LDAP_USERNAME`, `LDAP_EMAIL`, `LDAP_FULL_NAME` (CSV).
2. Service bind, user search, then user bind with the password, which yields `userinfo`.
3. `ldapId` = `LDAP_ID` attribute, else uid, sAMAccountName or mail. The email falls back to `<username>@ldap.local`.
4. Domain checks (base and tenant). `findUser({ldapId})`. A different provider fails with `AUTH_FAILED`.
5. Create the user (the first user becomes ADMIN, `emailVerified:true`) or overwrite its profile from LDAP. Then `loginController` runs as for local login, including 2FA.

### 3.11 SAML (`api/strategies/samlStrategy.js`, `@node-saml/passport-saml`)
- Env: `SAML_ENTRY_POINT`, `SAML_ISSUER`, `SAML_CERT` (a path or inline cert), `SAML_CALLBACK_URL`, `SAML_SESSION_SECRET`, `SAML_USE_AUTHN_RESPONSE_SIGNED` (true means the response must be signed, otherwise the assertion must be), `SAML_NAME_ID_FORMAT` (transient is rejected), `SAML_IDP_ISSUER`, and the claim maps `SAML_*_CLAIM`.
- Flow: resolve the subject NameID, then look up by `samlId`, falling back to email. A provider conflict or NameID mismatch fails with `AUTH_FAILED`. Otherwise create the user or atomically claim the SAML identity. `POST /oauth/saml/callback` then runs `oauthHandler`.

---

## 4. Authorization model

### 4.1 System roles and role capabilities (feature permissions)
- `SystemRoles = { ADMIN:'ADMIN', USER:'USER' }` (`packages/data-provider/src/roles.ts`). Custom roles are allowed.
- The `roles` collection (`packages/data-schemas/src/schema/role.ts`) is `{name, description, permissions: {<PermissionType>: {<Permission>: bool}}, tenantId}`, unique on `(name, tenantId)`.
- Defaults come from `roleDefaults`. At startup they are overlaid from the YAML `interface.*` config by `updateInterfacePermissions` (`packages/api/src/app/permissions.ts`).

`Permissions` enum: `USE, CREATE, UPDATE, READ, READ_AUTHOR, SHARE, OPT_OUT, VIEW_USERS, VIEW_GROUPS, VIEW_ROLES, SHARE_PUBLIC, CONFIGURE_OBO`.

`PermissionTypes` with their permission keys and defaults (`packages/data-provider/src/permissions.ts`):

| PermissionType | Keys | USER default | ADMIN default |
|---|---|---|---|
| PROMPTS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | all T |
| BOOKMARKS | USE | T (schema default) | T |
| MEMORIES | USE, CREATE, UPDATE, READ, OPT_OUT | schema defaults T | T |
| AGENTS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | T |
| MULTI_CONVO, TEMPORARY_CHAT, RUN_CODE, WEB_SEARCH, FILE_SEARCH, FILE_CITATIONS | USE | T | T |
| PEOPLE_PICKER | VIEW_USERS, VIEW_GROUPS, VIEW_ROLES | F,F,F | T |
| MARKETPLACE | USE | F | T |
| MCP_SERVERS | USE, CREATE, SHARE, SHARE_PUBLIC, CONFIGURE_OBO | T,F,F,F,F | T |
| REMOTE_AGENTS | USE, CREATE, SHARE, SHARE_PUBLIC | all F | T |
| SKILLS | USE, CREATE, SHARE, SHARE_PUBLIC | T,T,F,F | T |
| SHARED_LINKS | CREATE, SHARE, SHARE_PUBLIC | T,T,T | T |
| SCHEDULES | USE, CREATE | T,T | T |

### 4.2 System capabilities (admin RBAC, `systemgrants` collection)
- Capability strings live in `packages/data-schemas/src/admin/capabilities.ts`: `access:admin`, `read|manage:users`, `read|manage:groups`, `read|manage:roles`, `read|manage:configs`, `assign:configs`, `read:usage`, `read:insights`, `read|manage:agents`, `manage:mcpservers`, `manage:code_environments`, `read|manage:prompts`, `read|manage:skills`, `read|manage:sharedlinks`, `read|manage:assistants`, `read:audit_log`.
- Dynamic forms: `manage:configs:<section>`, `read:configs:<section>`, `assign:configs:<user|group|role>`.
- Implication: `manage:X` implies `read:X`. A section capability is also satisfied by its parent capability.
- A grant doc is `{principalType, principalId (ObjectId for user/group, role-name string for role), capability, tenantId (omitted for platform-level), grantedBy, grantedAt, expiresAt}`, unique on `(principalType, principalId, capability, tenantId)` (`packages/data-schemas/src/schema/systemGrant.ts`).
- `seedSystemGrants` gives the `ADMIN` role every capability at startup.
- `ResourceCapabilityMap` maps a resource type to the capability that bypasses its ACL: agent and remoteAgent map to manage:agents, promptGroup to manage:prompts, mcpServer to manage:mcpservers, skill to manage:skills, sharedLink to manage:sharedlinks, codeEnvironment to manage:code_environments.
- A few legacy checks still use `checkAdmin` (`role==='ADMIN'`, `api/server/middleware/roles/admin.js`).

### 4.3 ACL (per-resource sharing)
Defined in `packages/data-provider/src/accessPermissions.ts`:
- `PrincipalType`: `user | group | role | public`. `PrincipalModel`: `User | Group | Role`.
- `ResourceType`: `agent, codeEnvironment, promptGroup, mcpServer, remoteAgent, skill, sharedLink`.
- `PermissionBits`: `VIEW=1, EDIT=2, DELETE=4, SHARE=8, VIEW_INSIGHTS=16`. `MAX_PERM_BITS=31`.
- `RoleBits`: `VIEWER=1, EDITOR=3, MANAGER=7, OWNER=15`.
- `AccessRoleIds` (seeded into `accessroles` by `seedDefaultRoles`, `packages/data-schemas/src/methods/accessRole.ts`): `<type>_viewer` (1), `<type>_editor` (3), `<type>_owner` (15) for agent, codeEnvironment, promptGroup, mcpServer, remoteAgent and skill. sharedLink has only viewer and owner.
- `aclentries` doc (`packages/data-schemas/src/schema/aclEntry.ts`): `{principalType, principalId (Mixed), principalModel, resourceType, resourceId (ObjectId), permBits (int 0..31), roleId, inheritedFrom, grantedBy, grantedAt, expiredAt (TTL), tenantId}`.
- Resource creators are granted the `*_owner` role. **Effective permission is the bitwise OR** of `permBits` across all entries matching the user's principals.
- `hasPermission` queries `permBits: {$in: permissionBitSupersets(required)}` to find any value v with `(v & required) === required`. This is an enumeration workaround for Cosmos DB; in Python you can use `$bitsAllSet` or compute the check in SQL.
- **Principal resolution** (`getUserPrincipals`, `packages/data-schemas/src/methods/userGroup.ts` line 855): `[{user, ObjectId}, {role, roleName}, ...{group, id} for groups whose memberIds contain (idOnTheSource || userId), {public}]`.
- **Groups** (`packages/data-schemas/src/schema/group.ts`): `{name, description, email, avatar, memberIds[string], source:'local'|'entra', idOnTheSource, tenantId}`. Entra groups are synced from Graph on login (`syncUserEntraGroupMemberships`, `api/server/services/PermissionService.js`), gated by `USE_ENTRA_ID_FOR_PEOPLE_SEARCH` and `OPENID_REUSE_TOKENS`, plus `ENTRA_ID_INCLUDE_OWNERS_AS_MEMBERS`.
- **Tenancy:** most collections have `tenantId`. `tenantIsolation` is a Mongoose plugin (`packages/data-schemas/src/models/plugins/tenantIsolation.ts`) that auto-scopes queries from AsyncLocalStorage (`tenantStorage`). `TENANT_ISOLATION_STRICT` makes it strict, and `runAsSystem` bypasses it.

### 4.4 End-to-end: (a) role capability check
Uses `generateCheckAccess` in `packages/api/src/middleware/access.ts`, for example `generateCheckAccess({permissionType: REMOTE_AGENTS, permissions:[USE], getRoleByName})`.
1. `requireJwtAuth` sets `req.user` (with `role`).
2. The middleware calls `checkAccess`. If `skipCheck(req)` is true, allow.
3. No user or no `user.role`: false.
4. `getRoleByName(user.role)`, memoised per request (and cached in the role cache).
5. `pv = role.permissions[permissionType]`. **All** requested permissions must be truthy in `pv`. A permission that is false may still pass when `bodyProps[perm]` lists body fields that are all present in `req.body`.
6. If allowed, `next()`. Otherwise 403 `{message:'Forbidden: Insufficient permissions'}`.

System capability checks (`requireCapability(cap)`, `packages/api/src/middleware/capabilities.ts`):
1. Resolve principals (user, role, groups), cached per request by `capabilityContextMiddleware`.
2. `SystemGrant.exists({$or: principals (excluding public), capability: {$in:[cap, ...impliers, ...parents]}, tenantId matches OR platform-level})`.
3. If `platformOnly` is set, only USER principals with platform-level grants count.
4. Result: 403 `{message:'Forbidden'}` or `next()`.

### 4.5 End-to-end: (b) resource ACL check
Uses `canAccessResource({resourceType, requiredPermission, resourceIdParam, idResolver})` in `api/server/middleware/accessResources/canAccessResource.js`. Wrappers such as `canAccessAgentResource` resolve a custom `agent_xxx` id to an ObjectId.
1. Read `req.params[resourceIdParam]`; missing returns 400. No `req.user.id` returns 401.
2. **Capability bypass:** if `hasCapability(user, ResourceCapabilityMap[resourceType])`, call `next()`.
3. Optional `idResolver` maps a custom id to a doc or `_id`. Not found returns 404.
4. `PermissionService.checkPermission` calls `getUserPrincipals({userId, role})` then `aclEntry.hasPermission(principals, type, id, bits)`.
5. If allowed, set `req.resourceAccess = {resourceType, resourceId, customResourceId, permission, userId, resourceInfo}` and call `next()`. Otherwise 403.

Listing endpoints use `findAccessibleResources` (resource IDs where the effective bits are a superset of the requirement) or `getResourcePermissionsMap` (batch).

### 4.6 Sharing API (`/api/permissions`)
Router: `api/server/routes/accessPermissions.js`. Controller: `api/server/controllers/PermissionsController.js`.

- `GET /search-principals`: requires PEOPLE_PICKER (VIEW_USERS/GROUPS/ROLES); searches local users and groups plus Entra via Graph.
- `GET /:resourceType/roles`: lists access roles.
- `GET /:resourceType/:resourceId`: needs ACL SHARE (8). ADMIN on an agent bypasses this.
- `PUT /:resourceType/:resourceId` with body `{updated:[{type,id,accessRoleId,...}], removed:[...], public, publicAccessRoleId}`. Checks, in order:
  1. ACL SHARE;
  2. role-level `<PermissionType>.SHARE` (`checkShareAccess`);
  3. `SHARE_PUBLIC` if making the resource public (`createSharePolicyMiddleware`, `packages/api/src/middleware/share.ts`);
  4. sharedLink owner protection.

  It then ensures the principals exist (Entra groups are created on demand) and calls `bulkUpdateResourcePermissions`. VIEW_INSIGHTS changes are audited.
- `GET /:resourceType/effective/all` and `GET /:resourceType/:resourceId/effective`: return the caller's effective bits.
- `/api/roles` (`api/server/routes/roles.js`): `GET /:roleName` is open for your own role or non-ADMIN default roles; anything else needs `read:roles`. `PUT /:roleName/{prompts,agents,memories,people-picker,mcp-servers,marketplace,remote-agents,skills}` needs `manage:roles`.

---

## 5. Admin API (`/api/admin/*`)
Routes are in `api/server/routes/admin/`; handler factories are in `packages/api/src/admin/`. Every router starts with `requireJwtAuth` and `requireCapability('access:admin')`.

- **auth.js** (the admin panel is a separate SPA at `ADMIN_PANEL_URL`):
  - `POST /api/admin/login/local`: local login plus `access:admin`.
  - `GET /verify`.
  - `GET /oauth/{openid,google,github,discord,facebook,apple,saml}` and their callbacks use the `*Admin` strategies (existing users only). PKCE challenges are stored for 5 min and must be 64 hex characters.
  - The callback puts `{userId, user, token, refreshToken, origin, codeChallenge, expiresAt}` in cache `ADMIN_OAUTH_EXCHANGE` under a 32-byte hex code, then redirects with `?code=`.
  - `POST /oauth/exchange {code, code_verifier}` is one-time use. It checks the origin and the verifier with a hex SHA-256 check (not base64url S256), and returns `{token, refreshToken, user, expiresAt}` (`packages/api/src/auth/exchange.ts`).
  - `POST /oauth/refresh {refresh_token, user_id?, provider:'openid'|'google'}` refreshes at the IdP, then re-checks `access:admin`, the domain allowlist, issuer and tenant, and mints a new LibreChat JWT.
- **users.js:** list and search (`read:users`). Delete is commented out.
- **roles.js:** CRUD, `PATCH /:name/permissions`, and members (`read|manage:roles`). Members are users whose `role` equals the role name.
- **groups.js:** CRUD and members (`read|manage:groups`).
- **grants.js:** `GET /`, `GET /effective`, `GET /:type/:id`, `POST /`, `DELETE /:type/:id/:capability`.
  - Assigning needs the manage capability for the target principal type, and the caller must hold the capability being granted (no escalation).
  - Revoking needs manage only.
  - Both write an audit entry fail-closed; a failed audit rolls the grant back.
- **config.js:** per-principal config overrides in the `configs` collection `{principalType, principalId, principalModel, priority, overrides, tombstones, isActive, configVersion, tenantId}`. Endpoints cover list, base config, get, put, patch field, tombstone field, delete field, delete and toggle active.
- **audit.js** (`read:audit_log`): `GET /`, `/export.csv`, `/verify` (hash-chain verification), `/:id`.
  - Audit log (`packages/data-schemas/src/schema/auditLog.ts`) is append-only; updates and deletes are blocked by hooks.
  - Fields: `{schemaVersion, category, action, outcome, severity, actor{type,id,name}, target{type,id,name}, metadata, context{requestId,ip,userAgent,sessionId}, tenantId, chainKey('__platform__' or tenant), seq, prevHash (genesis '0'*64), hash = sha256(stableStringify(canonical entry)), createdAt}`.
  - Current actions: `grant.assigned`, `grant.removed`, `permission.insights_assigned`, `permission.insights_removed`.
- **code.js** (manage:code_environments), **langfuse.js**, **skills.js**: feature-specific.

---

## 6. API keys and user keys

### 6.1 Agent API keys, for external callers of the agents API
- Management routes are `/api/api-keys` (`api/server/routes/apiKeys.js`), all behind requireJwtAuth and `REMOTE_AGENTS.USE`: POST create `{name, expiresAt?}`, GET list, GET `:id`, DELETE `:id`.
- Key generation (`packages/data-schemas/src/methods/agentApiKey.ts`): `key = 'sk-' + hex(32 random bytes)`, `keyHash = sha256hex(key)`, `keyPrefix = key[0:8]`.
- The key is returned once. The DB stores `{userId, name, keyHash (select:false), keyPrefix, lastUsedAt, expiresAt (TTL index), tenantId}` (`packages/data-schemas/src/schema/agentApiKey.ts`).
- **Auth path.** Endpoints are `/api/agents/v1/chat/completions` (OpenAI-compatible), `/api/agents/v1/responses` and `/api/agents/v1/models`. The chain is `preAuthTenantMiddleware`, `requireRemoteAgentAuth`, `configMiddleware`, `checkRemoteAgentsFeature` (role `REMOTE_AGENTS.USE`), and `checkAgentPermission` on execution routes.
  1. `requireRemoteAgentAuth` (`packages/api/src/middleware/remoteAgentAuth.ts`) reads the config `endpoints.agents.remoteApi.auth`.
  2. If `oidc.enabled`, it verifies the Bearer through JWKS against the configured issuer and audience, checks the scope, and resolves or provisions the user (optionally syncing roles).
  3. On OIDC failure, or when OIDC is disabled, it falls back to the API key if `apiKey.enabled !== false`.
  4. API-key middleware (`createRequireApiKeyAuth`, `packages/api/src/apiKeys/middleware.ts`): extract the Bearer, `sha256`, `findOne({keyHash})`, reject if expired, update `lastUsedAt`, load the user and check it is active (not being deleted, else 409). Set `req.user` and `req.apiKeyId`.
  5. Errors use the OpenAI error shape `{error:{message,type,code}}`.
  6. `checkAgentPermission` reads the agent id from `body.model`. `getRemoteAgentPermissions` returns all bits if the user has SHARE on the `agent` resource; otherwise it uses the ACL bits on the `remoteAgent` resource. VIEW is required.
- `/api/agents/v1/agents` (management) uses `requireAgentManagementAuth`, which is OIDC-only (`endpoints.agents.managementApi.auth`).

### 6.2 User-provided provider keys (`/api/keys`, `api/server/routes/keys.js`)
- `PUT {name, value, expiresAt}` upserts `keys` docs `{userId, name (endpoint), value = encrypt(value), expiresAt (TTL index), tenantId}`.
- `GET ?name=` returns `{expiresAt: Date|'never'|null}`. `DELETE /:name` or `DELETE ?all=true`.
- `getUserKey` decrypts; `getUserKeyValues` also JSON-parses. Missing keys throw `ErrorTypes.NO_USER_KEY`; bad values throw `INVALID_USER_KEY` (`packages/data-schemas/src/methods/key.ts`).
- Crypto (`packages/data-schemas/src/crypto/index.ts`, re-exported by `packages/api/src/crypto`), using `CREDS_KEY` (hex, 32 bytes) and `CREDS_IV` (hex, 16 bytes):
  - **v1** `encrypt`/`decrypt`: AES-CBC with fixed key and IV; hex output. Used for user keys.
  - **v2** `encryptV2`: AES-CBC with a random IV; `ivhex:cthex`.
  - **v3** `encryptV3`: AES-256-CTR with a random IV; `v3:ivhex:cthex`. Used for the TOTP secret.
  - `hashToken`: SHA-256 hex. `hashBackupCode`: SHA-256 hex.
- Plugin credentials are stored separately in `pluginAuth`.

---

## 7. Balance and transactions

**Config** (`balance` in librechat.yaml, `packages/data-provider/src/config.ts` line 2654): `{enabled=false, startBalance=20000, autoRefillEnabled=false, refillIntervalValue=30, refillIntervalUnit='days', refillAmount=10000, reservationTtlMs}`. There is also `transactions.enabled=true`. Config can vary per tenant, role or user.

**Balance doc** (`packages/data-schemas/src/schema/balance.ts`): `{user, tokenCredits (1000 credits = $0.001, i.e. 1 credit = $1e-6), autoRefillEnabled, refillIntervalValue, refillIntervalUnit, lastRefill, refillAmount, reservations[{id,amount,expiresAt}], reservedCredits, pendingRefill{transactionId,rawAmount}, tenantId}`.

**Sync on auth:** `createSetBalanceConfig` (`packages/api/src/middleware/balance.ts`) runs on login, OAuth callbacks and `GET /api/balance`. It creates the record with `startBalance` if missing and copies the auto-refill settings from config into the doc (using a per-user promise lock). `GET /api/balance` returns 204 if balance is disabled; otherwise it returns the doc, with refill fields removed when auto-refill is off.

**Pre-flight admission:** `checkBalance` (`packages/api/src/middleware/checkBalance.ts`).
1. `tokenCost = promptTokens * getMultiplier({valueKey|model|endpoint, tokenType, endpointTokenConfig})`.
2. `reserveBalance` does an atomic conditional `$push` to reservations and `$inc` to reservedCredits, matching only while `tokenCredits - reservedCredits >= amount`.
3. A due auto-refill is applied first with a fenced write, and its ledger `Transaction{tokenType:'credits', context:'autoRefill', rawAmount}` is recorded idempotently through `pendingRefill`.
4. If the reservation fails, `logViolation(TOKEN_BALANCE)` and throw a JSON error `{type:'token_balance', balance, tokenCost, promptTokens}`.
5. Reservations are released when the request settles, or pruned after the TTL.

**Spending** (`packages/data-schemas/src/methods/spendTokens.ts`, `packages/data-schemas/src/methods/transaction.ts`, `packages/data-schemas/src/methods/tx.ts`):
- `spendTokens(txData, {promptTokens, completionTokens})` creates two `Transaction` docs with `rawAmount = -tokens`.
- `calculateTokenValue`: `rate = |getMultiplier(...)|`, `tokenValue = rawAmount * rate`. If `tokenType==='completion'` and `context==='incomplete'` (aborted), `tokenValue = ceil(tokenValue * 1.15)` (cancelRate).
- **Structured (prompt caching):** `tokenValue = -(input*inMult + write*writeMult + read*readMult)`. Cache multipliers come from `getCacheMultiplier` and fall back to the input multiplier. `rate` is the weighted average.
- **Multiplier resolution:**
  1. `endpointTokenConfig[model][tokenType]`;
  2. premium long-context rate if `inputTokenCount > threshold` (e.g. `gpt-5.4`: threshold 272000);
  3. `tokenValues[valueKey][tokenType]`, where `valueKey = getValueKey(model, endpoint)` (a pattern match on the model name);
  4. default rate 6.

  Rates are USD per 1M tokens, which equals credits per token.
- Save the transaction. If balance is enabled, `updateBalance` uses optimistic concurrency: `findOneAndUpdate({_id, tokenCredits: current}, {$set:{tokenCredits: max(0, current + tokenValue)}})`, retried up to 10 times with exponential backoff and jitter.
- **Transaction doc:** `{user, conversationId, tokenType:'prompt'|'completion'|'credits', model, context, valueKey, rate, rawAmount, tokenValue, inputTokens, writeTokens, readTokens, messageId, tenantId, timestamps}`.
- **Formula:** `new_credits = max(0, credits + Σ(rawAmount_i × rate_i × (1.15 if incomplete completion)))`. `rawAmount` is negative for spend and positive for refill.

---

## 8. Users, bans, account deletion

- **User doc** (`packages/data-schemas/src/schema/user.ts`): `name, username, email (unique per tenant), emailVerified, password (select:false), avatar, provider, role, googleId/facebookId/openidId/openidIssuer/samlId/ldapId/githubId/discordId/appleId, plugins, twoFactorEnabled, totpSecret, backupCodes[{codeHash,used,usedAt}], pendingTotpSecret, pendingBackupCodes, refreshToken (legacy), expiresAt (TTL 7 days), termsAccepted/At, agentTriggerDeletionStartedAt, subagentAdmissionFences, personalization, favorites, pinnedOrder, skillStates, idOnTheSource, tenantId`.
- **`/api/user`** (`api/server/routes/user.js`):
  - `GET /` returns the sanitized user and refreshes an S3 avatar URL if needed.
  - `PATCH /preferences`; `GET /terms`, `POST /terms/accept`.
  - `POST /plugins {pluginKey, action, auth}`: install or uninstall tools and store encrypted plugin auth. MCP connections are torn down on uninstall.
  - `DELETE /delete`: requires `canDeleteAccount` (`ALLOW_ACCOUNT_DELETION`, default true; otherwise needs `access:admin`), plus TOTP or a backup code if 2FA is on.
  - `POST /verify`, `POST /verify/resend`.
- **Delete cascade** (`api/server/controllers/UserController.js`, around line 400): fence agent triggers, drain queued jobs and stop active runs, then delete, in order:
  1. conversations and checkpoints, messages;
  2. sessions, transactions, keys, balances, presets;
  3. plugin auth, shared links, files (storage and DB), tool calls;
  4. agents, agent API keys, assistants, conversation tags;
  5. memories, prompts, skills, MCP servers, actions, tokens;
  6. group memberships, ACL entries where `principalId=user`, schedules;
  7. finally the user, and revoke code-environment workers.
- **Bans** (`api/server/middleware/checkBan.js`, `api/cache/logViolation.js`, `api/cache/banViolation.js`):
  - Enabled by `BAN_VIOLATIONS`.
  - Limiters and other checks call `logViolation(type, score)`, which increments a per-user violation counter.
  - Each time the count crosses a multiple of `BAN_INTERVAL` (default 20), a ban is written for the user id and the IP with duration `BAN_DURATION` (default 2 h). All sessions are deleted and auth cookies cleared.
  - `checkBan` checks the IP and the userId (or the user found by `body.email`), using an in-memory Keyv cache in front of the ban store. It returns 403 "Your account has been temporarily banned…", or redirects for OAuth navigations.

---

## 9. Python re-implementation suggestions

- **Framework and dependencies:** FastAPI with `Depends`:
  - `get_current_user` tries a JWT (PyJWT or python-jose, HS256 `JWT_SECRET`) and, for the OpenID reuse path, a JWKS-verified RS256 token (`PyJWKClient` with a cached JWKS). It then loads the user and sets tenant context in a `contextvars.ContextVar`.
  - `optional_user`, `require_capability(cap)`, `require_role_permission(ptype, [perms])` and `require_resource(resource_type, bits, id_param, resolver)` mirror sections 4.4 and 4.5.
- **Passwords:** `bcrypt` or `passlib[bcrypt]`, cost 10, to stay compatible with existing hashes.
- **Refresh sessions:** keep the same design, a refresh JWT `{id, sessionId}` plus a `sessions` table or collection holding `sha256hex(refresh)` and `expiration`. Cookies: `response.set_cookie('refreshToken', httponly=True, samesite='strict', secure=...)`.
- **OAuth/OIDC:** `authlib` (Starlette integration) for Google, GitHub, Discord, Facebook, Apple and generic OIDC (PKCE, nonce, `refresh_token` grant, `jwt-bearer` OBO grant). Keep the HMAC-signed `state` bound to a `__Host-` cookie. Use `starlette-session` or a Redis-backed session for `openidTokens`. A Redis lock (`SET NX PX`) can replace the flight and bridge collections.
- **LDAP:** `ldap3`. Service bind, search with `LDAP_SEARCH_FILTER` (`{{username}}` replaced with an escaped value via `ldap3.utils.conv.escape_filter_chars`), then bind as the user DN. Supports StartTLS and a CA file.
- **SAML:** `python3-saml` (OneLogin) or `pysaml2`. Enforce signed assertions or responses per `SAML_USE_AUTHN_RESPONSE_SIGNED`, and reject transient NameID.
- **2FA:** `pyotp.TOTP(secret, digits=6, interval=30).verify(code, valid_window=1)`. Backup codes are `secrets.token_hex(4)` stored as SHA-256. Encrypt the secret with AES-256-CTR from `cryptography` using the `v3:iv:ct` format, to read existing data.
- **Crypto compatibility:** implement all three AES formats (CBC fixed IV, CBC random IV `iv:ct`, CTR `v3:`) with `cryptography.hazmat` using `CREDS_KEY` and `CREDS_IV`.
- **ACL:** store `perm_bits` as an int. The check is `(perm_bits & required) == required` over the OR of the user's principal entries. In SQL: `bit_or(perm_bits)` grouped by resource. In Mongo: `$bitsAllSet`.
- **Rate limiting and bans:** `slowapi` or `limits` backed by Redis, with a violation counter and ban keys in Redis carrying TTLs.
- **Balance:** do the atomic conditional update in one statement, e.g. `UPDATE balances SET token_credits = GREATEST(0, token_credits + :delta) WHERE id=:id AND token_credits=:old` with a retry loop, or `SELECT … FOR UPDATE` in Postgres. Implement reservations as rows with `expires_at`. Port the `tokenValues`, premium and cache tables from `packages/data-schemas/src/methods/tx.ts` as data.
- **Audit log:** an append-only table with `(chain_key, seq)` unique and `hash = sha256(canonical_json(entry_without_hash) incl. prevHash)`.
