# Data Layer (MongoDB / packages/data-schemas)

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

All paths below are relative to `packages/data-schemas/src/` unless they are absolute.

---

## 1. How the data layer is built

### 1.1 Package layout
- `schema/*.ts`: plain Mongoose `Schema` objects with no model binding. `schema/index.ts` re-exports them.
- `models/*.ts`: one `createXModel(mongoose)` factory per model. Each one:
  1. calls `applyTenantIsolation(schema)`, except for five models listed below;
  2. optionally attaches the `mongoMeili` plugin;
  3. returns `mongoose.models.X || mongoose.model('X', schema)`, so repeated calls are safe.
- `models/index.ts` → `createModels(mongoose)` builds **47 models**. It also attaches an `'index'` event listener to every model. The listener logs failed background index builds, which otherwise fail silently (for example on DocumentDB when a `partialFilterExpression` is rejected). For legacy tenant-index conflicts (codes 85/86) it adds a hint to run `npm run migrate:tenant-indexes` (`getTenantIndexMigrationHint` in `migrations/tenantIndexes.ts`).
- `methods/*.ts`: one `createXMethods(mongoose, deps?)` factory per domain. Each returns plain async functions that fetch models lazily through `mongoose.models.X`.
- `methods/index.ts` → `createMethods(mongoose, deps)` builds one flat object (`AllMethods`) by spreading every domain. It wires dependencies between domains in tiers:
  - `txMethods` (rate tables) → `transactionMethods` (`getMultiplier`, `getCacheMultiplier`) → `spendTokensMethods`.
  - `messageMethods` and `agentQueuedTurnMethods` → `agentTriggerDeliveryMethods`, which receives `purgeQueuedTurnsForUser`.
  - `conversationMethods` receives `getMessages`, `deleteMessages`, `searchMessages`, the trigger-delivery erasure hooks, and a large `deleteAgentQueuedTurns` closure. That closure retires delivery receipts before it deletes queued-turn rows.
  - `aclEntryMethods` supplies `removeAllPermissions`, `getSoleOwnedResourceIds` and `findAccessibleResources` to the prompt, skill and agent methods. `userGroupMethods` supplies `getUserPrincipals`.
  - `agentMethods` receives `removeAllPermissions`, `getActions`, `getSoleOwnedResourceIds`, `getUserPrincipals`, `findAccessibleResources` and `isExternalSkillId`.
  - `CreateMethodsDeps` is injected from the API layer: `matchModelName`, `findMatchingPattern`, `removeAllPermissions`, `getCache` (the `getLogStores` cache factory) and `isExternalSkillId`.
- Consumers:
  - `api/db/models.js` runs `createModels(mongoose)`.
  - `api/models/index.js` runs `createMethods(...)` and defines `seedDatabase()`, which calls `initializeRoles`, `seedDefaultRoles` (AccessRoles), `ensureDefaultCategories` (AgentCategory) and `seedSystemGrants`.
  - `api/db/indexSync.js` runs Meili `syncWithMeili()` for `Message` and `Conversation`.
- `index.ts` (package root) exports:
  - schemas, `createModels`, `createMethods`;
  - the logger (`config/winston.ts`) and `meiliLogger`;
  - tenant context helpers: `tenantStorage`, `getTenantId`, `runAsSystem`, `scopedCacheKey`, `SYSTEM_TENANT_ID`;
  - migrations, crypto, and `utils`: retention helpers, `tenantSafeBulkWrite`, `retryWithBackoff`, `buildIndexWithRetry`, `initializeOrgCollections`.

### 1.2 Multi-tenancy
**Context propagation** is in `config/tenantContext.ts`.
- `tenantStorage = new AsyncLocalStorage<{tenantId, userId, requestId, requestMethod, requestPath}>()`.
- `SYSTEM_TENANT_ID = '__SYSTEM__'`. `runAsSystem(fn)` re-runs `fn` in a context whose tenant is the system sentinel.
- `scopedCacheKey(base)` appends `:${tenantId}` to cache keys.
- The context is set by `packages/api/src/middleware/tenant.ts` using `tenantStorage.run(buildTenantContext(req), ...)`. The tenant comes from `req.tenantId ?? req.user.tenantId`, i.e. from the JWT/user record. `preAuthTenant.ts` sets context before authentication, and background services (schedules, triggers, skill sync) set it themselves.

**Policy** is in `tenant/policy.ts`, written as pure functions that do not depend on any database engine.
- `currentTenantScope()` returns one of three kinds: `scoped{tenantId}`, `system`, or `unscoped`.
- `resolveTenantScope(op)` throws `TenantIsolationError` when the scope is unscoped and `TENANT_ISOLATION_STRICT=true`. Strict mode fails closed.
- `tenantFilter(scope)` returns `{tenantId}` only for the scoped kind.
- `sanitizeTenantMutation(scope, update, 'guard'|'strip')` removes `tenantId` from the top level and from `$set`, `$setOnInsert`, `$unset` and `$rename`.
  - `guard` mode throws on a cross-tenant value.
  - The system scope may set `tenantId`.
- `scopeReplacement` stamps the tenant onto replacement documents and refuses a foreign one.
- `tenantWritePredicate` is used when saving an already-persisted document. It adds `$where.tenantId: {$in:[tenant, null, '']}`, so rows created before tenancy can be claimed atomically.
- `stampTenantOnDocument` sets `tenantId` on inserts. In strict mode it refuses a mismatched tenant.

**Mongoose binding** is in `models/plugins/tenantIsolation.ts` (`applyTenantIsolation`).
- Pre-hooks on `find`, `findOne`, `distinct`, `findOneAndUpdate/Delete/Replace`, `updateOne/Many`, `deleteOne/Many`, `countDocuments` and `replaceOne` add `this.where({tenantId})`.
- On update and replace queries it also runs the update guard or replace guard. If the update is emptied, the query is rewritten to `_id $in []` with `upsert:false`.
- `aggregate` gets `$match:{tenantId}` unshifted onto the front of the pipeline.
- `save` stamps `tenantId` on new documents and adds a `$where` tenant predicate for updates to existing documents. On a save error the stamp is rolled back.
- `insertMany` stamps every document.
- A `Symbol` guard stops the plugin from being applied twice.

**`bulkWrite`** does not run Mongoose middleware, so `utils/tenantBulkWrite.ts` provides `tenantSafeBulkWrite(model, ops)`. It strips `tenantId` from updates and injects `tenantId` into every filter and inserted document.

**Models that deliberately skip the plugin:**
- `AuditLog`: scoped by `chainKey` instead.
- `SystemGrant`: a cross-tenant control plane that uses its own `$or` of tenant and platform-level conditions (`tenantCondition` in `methods/systemGrant.ts`).
- `RefreshTokenBridge`: looked up from unauthenticated refresh requests, so methods check the tenant explicitly.
- `OpenIDRefreshFlight`.
- `SkillSyncCredential` and `SkillSyncStatus`: application-wide.

**Supporting tools:**
- `tenant/conformance.ts`: a portable contract test suite (`TenantEngineHarness`) that any storage engine binding can run.
- `tenant/probe.ts`: a driver-level probe built on `monitorCommands`. It checks that every command sent to the database carries `{tenantId}` at the top level of its filter, or as a pipeline `$match`.

Almost every tenant-scoped schema declares `tenantId: {type:String, index:true}`, and unique indexes are compound with `tenantId`.

### 1.3 Plugins
- **`mongoMeili`** (`models/plugins/mongoMeili.ts`, about 1300 lines).
  - Applied only to `Conversation` (index `convos`, primary key `conversationId`, `excludeFromIndexPath:'subagentThread'`) and `Message` (index `messages`, primary key `messageId`, `excludeFromIndexPath:'subagentTask'`).
  - Only active when `MEILI_HOST` and `MEILI_MASTER_KEY` are set, and search further requires `SEARCH=true`.
  - Fields marked `meiliIndex: true` are indexed:
    - convo: `conversationId`, `title`, `user`, `tags`;
    - message: `messageId`, `conversationId`, `user`, `sender`, `text`, `content`.
    - `user` is always added and is made a filterable attribute.
  - The plugin adds hidden bookkeeping fields: `_meiliIndex`, `_meiliIndexAttempted`, `_meiliIndexVersion` (an ObjectId string that changes on every save), `_meiliIndexSchemaVersion` (currently 1) and `_meiliCleanupVersion`.
  - The `pre('save')` hook assigns a new version.
  - The post-save sync (`syncVersionedDocument`) writes to Meili, waits for the task, then acknowledges with `collection.updateOne({_id, _meiliIndexVersion: version}, {$set:{_meiliIndex:true,...}})`. This compare-and-swap on the version handles concurrent writes, with up to 3 reconcile attempts.
  - Documents that are temporary, expired, excluded or `unfinished` are deleted from Meili instead.
  - Indexability rule: `isTemporary === false` stored explicitly and not expired, **or** a legacy document with no `isTemporary` flag and `expiredAt == null`.
  - `preprocessObjectForIndex` flattens `content[]` to `text` via `parseTextParts` and replaces `|` with `--` in `conversationId`.
  - Static methods:
    - `syncWithMeili()`: batched with `MEILI_SYNC_BATCH_SIZE` (default 100) and `MEILI_SYNC_DELAY_MS` (default 100). It indexes `_meiliIndex != true` documents and documents on an old schema version, then runs `cleanupMeiliIndex` to delete Meili documents that no longer exist in Mongo.
    - `getSyncProgress()`, `setMeiliIndexSettings()`.
    - `meiliSearch(q, params, populate)`: searches Meili, then re-reads the hits from Mongo.
  - Hooks also cover `findOneAndUpdate`, `updateOne`, `deleteOne` and `deleteMany`; `deleteMany` removes the matching documents from Meili in batches.
  - It adds partial indexes `meili_excluded_*_cleanup_v3/v4`. Both schemas also declare `{_meiliIndex:1, isTemporary:1, expiredAt:1}`.
- **`tenantIsolation`**: described in 1.2.
- **Schema-level hooks:**
  - `AuditLog` is append-only. Its pre-hooks reject all update, delete, `bulkWrite` and `insertMany` operations, plus `save` on a document that is not new.
  - `MCPServer` computes `normalizedServerName` in `pre('validate')`.

### 1.4 Other infrastructure
- **`utils/transactions.ts`**: `supportsTransactions` probes transaction support using the `__transaction_test__` collection. On DocumentDB it creates the collection and retries. `getTransactionSupport` caches the result and shares one in-flight probe between callers. Transactions are used in `mcpAuthority` (snapshot read concern with majority write concern) and in `userGroup` (`runAfterTransaction`).
- **`utils/retry.ts`**:
  - `retryWithBackoff`;
  - `buildIndexWithRetry` and `createIndexesWithRetry`, which retry when an index build is already in progress;
  - `initializeOrgCollections(models)`, which creates collections and indexes sequentially.
  - Several method files build their own indexes lazily, once, on first use: Session, RefreshTokenBridge, OpenIDRefreshFlight, AuditLog, Assistant, QueuedTurn, Schedule, TriggerDelivery.
- **`crypto/index.ts`**:
  - `signPayload` (JWT);
  - `hashToken` (SHA-256 hex);
  - `encrypt`/`decrypt`: AES-CBC with the static `CREDS_KEY` and `CREDS_IV`;
  - `encryptV2`/`decryptV2`: AES-CBC with a random IV, stored as `iv:cipher`;
  - `encryptV3`/`decryptV3`: `aes-256-ctr`, stored as `v3:iv:cipher`;
  - `getRandomValues`.
  - These protect `Key.value`, `PluginAuth.value`, Action OAuth secrets, `SkillSyncCredential.encryptedToken`, `RefreshTokenBridge.encryptedNewRefreshToken` and `OpenIDRefreshFlight.encryptedResult`.
- **`config/winston.ts`**: a Winston logger.
  - Daily-rotating `error-%DATE%.log` files, plus debug logs when `DEBUG_LOGGING` is set.
  - Console output, optionally JSON (`CONSOLE_JSON`).
  - `redactFormat` (in `parsers.ts`) and request-context enrichment (in `requestLogContext.ts`, fed from the ALS context).
- **Retention** (`utils/retention.ts`, `utils/tempChatRetention.ts`):
  - Default retention is 720 hours (30 days), clamped to 1–8760 hours.
  - Source order: `TEMP_CHAT_RETENTION_HOURS` env var, then overridden by `interfaceConfig.temporaryChatRetention`.
  - With `retentionMode === 'all'`, regular chats also expire, using `generalChatRetention`.
  - `buildRetentionVisibilityFilter()` hides expired and temporary rows.

---

## 2. Collections by domain

Conventions used below:
- **T** means `tenantId: String` (indexed) is present and the tenant plugin applies.
- `ts` means the schema uses `timestamps: true` (`createdAt` and `updatedAt`).
- **TTL** means a TTL index.

### 2.1 Identity and access

**User** (`schema/user.ts`, collection `users`), T, ts. Application user account.
- Profile fields: `name`, `username` (lowercase), `email` (required, lowercase, regex-validated), `emailVerified`.
- `password`: `select:false`, 8–128 characters.
- `avatar`; `provider` (default `local`); `role` (default `USER`, matches `Role.name`).
- OAuth IDs: `googleId`, `facebookId`, `openidId` + `openidIssuer`, `samlId`, `ldapId`, `githubId`, `discordId`, `appleId`.
- `plugins[]`.
- 2FA: `twoFactorEnabled`, and `totpSecret`, `backupCodes[{codeHash, used, usedAt}]`, `pendingTotpSecret`, `pendingBackupCodes` (all `select:false`).
- `refreshToken[{refreshToken}]` (legacy).
- `expiresAt`: **TTL `expires: 604800`** (7 days). Set on unverified new users; `createUser` removes it when `disableTTL` is used.
- `termsAccepted`, `termsAcceptedAt`.
- `agentTriggerDeletionStartedAt` and `subagentAdmissionFences[{token, expiresAt}]` (both hidden).
- `personalization{memories: Bool, statefulCodeEnvironment: enum}`.
- `favorites[{agentId, model, endpoint, spec}]`.
- `pinnedOrder[String]` (hidden).
- `skillStates: Map<String,Boolean>`.
- `idOnTheSource` (sparse): external ID, for example from Entra.
- Indexes:
  - unique `{email, tenantId}`;
  - `{role, tenantId}`;
  - `{idOnTheSource, openidIssuer, tenantId}`;
  - unique partial `{<oauthId>, tenantId}` for each provider, with `openidId` compound with `openidIssuer`.

**Session** (`sessions`), T. Refresh-token sessions.
- `refreshTokenHash` (SHA-256 of the refresh JWT).
- `expiration`: **TTL `expires:0`**.
- `user` → User.
- Indexes: unique `{user, refreshTokenHash}`.
- Default refresh expiry is 7 days. The access session expiry `DEFAULT_SESSION_EXPIRY` is 15 minutes (defined in `methods/user.ts`).

**Token** (`tokens`), T. One-time tokens for email verification, password reset, invites and OAuth.
- `userId` → User; `email` (lowercase, trimmed); `type`; `identifier`; `token`.
- `createdAt`; `expiresAt` with **TTL on `expiresAt`**.
- `metadata: Map<Mixed>`.
- Indexes: `{userId, type, identifier, tenantId}`.

**Role** (`roles`), T. Named role with a boolean permission matrix.
- `name`; `description`.
- `permissions`: nested booleans per permission type. Types are BOOKMARKS, PROMPTS, MEMORIES, AGENTS, MULTI_CONVO, TEMPORARY_CHAT, RUN_CODE, WEB_SEARCH, PEOPLE_PICKER, MARKETPLACE, FILE_SEARCH, FILE_CITATIONS, MCP_SERVERS (including CONFIGURE_OBO), REMOTE_AGENTS, SKILLS, SHARED_LINKS and SCHEDULES. Keys include USE, CREATE, SHARE, SHARE_PUBLIC, UPDATE, READ, OPT_OUT and VIEW_*.
- Indexes: unique `{name, tenantId}`.

**Group** (`groups`), T, ts. User groups, either local or synced from Entra ID.
- `name`, `description`, `email`, `avatar`.
- `memberIds[String]`: user `_id` strings, or `idOnTheSource` for external users.
- `source`: enum `local|entra`.
- `idOnTheSource`: required unless `source` is `local`.
- Indexes: unique partial `{idOnTheSource, source, tenantId}`; `{memberIds, tenantId}`.

**AccessRole** (`accessroles`), T, ts. Named permission-bit bundles per resource type, such as `agent_viewer`, `agent_editor` and `agent_owner`.
- `accessRoleId` (string key), `name`, `description`.
- `resourceType`: enum `agent|codeEnvironment|project|file|promptGroup|mcpServer|remoteAgent|skill|sharedLink`.
- `permBits: Number`.
- Indexes: unique `{accessRoleId, tenantId}`.

**AclEntry** (`aclentries`), T, ts. Per-resource ACL grants.
- `principalType`: `user|group|public|role`.
- `principalId`: Mixed. An ObjectId for user and group, the role name string for role, absent for public.
- `principalModel`: `User|Group|Role`.
- `resourceType`: `agent|codeEnvironment|promptGroup|mcpServer|remoteAgent|skill|sharedLink`.
- `resourceId`: ObjectId, pointing at the resource document's `_id`.
- `permBits`: integer from 0 to `MAX_PERM_BITS` = 31. Bits are VIEW=1, EDIT=2, DELETE=4, SHARE=8, VIEW_INSIGHTS=16.
- `roleId` → AccessRole; `inheritedFrom` (ObjectId, sparse); `grantedBy` → User; `grantedAt`.
- `expiredAt`: **TTL**.
- Indexes:
  - `{principalId, principalType, resourceType, resourceId, tenantId}`;
  - `{resourceId, principalType, principalId, tenantId}`;
  - `{principalId, permBits, resourceType, tenantId}`;
  - `{principalType, resourceType, permBits, resourceId}`, used for public lookups.

**SystemGrant** (`systemgrants`), no plugin, ts. Admin capability grants that apply at platform or tenant level.
- `principalType`; `principalId` (Mixed).
- `capability`: validated by `admin/capabilities.ts` `isValidCapability`, for example `read:configs`, `manage:configs:<section>`.
- `tenantId`: must be **absent** for platform-level grants, never null or empty.
- `grantedBy`, `grantedAt`, `expiresAt` (reserved, not yet enforced).
- Indexes: unique `{principalType, principalId, capability, tenantId}`; `{capability, tenantId}`; `{principalType, capability, tenantId}`.

**AuditLog** (`auditlogs`), no plugin. Append-only, hash-chained compliance log.
- All fields are `immutable`.
- `schemaVersion`.
- `category`: `grant|agent_run|tool_call|mcp|config|permission|auth|approval`.
- `action`: currently `grant.assigned|grant.removed|permission.insights_assigned|permission.insights_removed`.
- `outcome`: `success|failure|denied|pending`.
- `severity`: `info|warning|critical`.
- `actor{type: user|system|agent|service|schedule|webhook|api, id, name}`.
- `target{type, id, name}`; `metadata` (Mixed); `context{requestId, ip, userAgent, sessionId}`.
- `tenantId` (omitted for platform entries); `chainKey` (the `tenantId`, or `__platform__`).
- `seq`, `prevHash`, `hash`, `createdAt`.
- Indexes: **unique `{chainKey, seq}`**; `{chainKey, createdAt:-1, seq:-1}`; `{chainKey, category, createdAt:-1}`; `{chainKey, target.type, target.id, createdAt:-1}`.

**Key** (`keys`), T. User-provided API keys per endpoint, stored encrypted.
- `userId` → User; `name` (endpoint); `value` (encrypted).
- `expiresAt`: **TTL**.

**PluginAuth** (`pluginauths`), T, ts. Encrypted per-user credentials for plugins and tools, including MCP customUserVars.
- `authField`, `value` (encrypted), `userId` (string), `pluginKey`.
- Indexes: `{userId, pluginKey, authField, tenantId}`.

**AgentApiKey** (`agentapikeys`), T, ts. API keys for the remote-agent (OpenAI-compatible) API.
- `userId` → User; `name` (max 100).
- `keyHash`: `select:false`, indexed.
- `keyPrefix` (indexed); `lastUsedAt`.
- `expiresAt`: **TTL**.
- Indexes: `{userId, name, tenantId}`.

**Balance** (`balances`), T. Token-credit balance, auto-refill settings and in-flight reservations.
- `user` → User.
- `tokenCredits`: 1000 credits = $0.001.
- `autoRefillEnabled`, `refillIntervalValue`, `refillIntervalUnit` (enum), `lastRefill`, `refillAmount`.
- Hidden fields: `reservations[{id, amount, expiresAt}]`, `reservedCredits`, `pendingRefill{transactionId, rawAmount}`.

**Transaction** (`transactions`), T, ts. Token spend and credit ledger.
- `user` → User; `conversationId` (string) → Conversation.
- `tokenType`: `prompt|completion|credits`.
- `model`, `context`, `valueKey`, `rate`, `rawAmount`, `tokenValue`.
- `inputTokens`, `writeTokens`, `readTokens` (prompt-cache accounting); `messageId`.

**RefreshTokenBridge** (`refreshtokenbridges`), no plugin. Maps a rotated old refresh token to its new one, for concurrent-refresh races.
- `oldRefreshTokenHash`, `encryptedNewRefreshToken`, `userId` (string), `tenantId`, `openidIssuer`, `version`, `createdAt`.
- `expiresAt`: **TTL**.
- Indexes: unique `{oldRefreshTokenHash, userId, tenantId}`.

**OpenIDRefreshFlight** (`openidrefreshflights`), no plugin. Distributed lock and result mailbox for inline OIDC token refreshes across workers.
- `key` (unique, hashed context), `ownerId`.
- `status`: `pending|completed|failed|revoked`.
- `encryptedResult`, `errorMessage`, `deliveryId`, `deliveryExpiresAt`, `revocationRequestedAt`.
- `lockExpiresAt` (indexed).
- `expiresAt`: **TTL**.

### 2.2 Chat

**Conversation** (`conversations`), T, ts, Meili. Chat thread header plus agent-runtime state.
- `conversationId`: string UUID, the business key.
- `title` (default "New Chat"); `user` (**string** user ID).
- `messages[ObjectId → Message]`.
- `isTemporary`.
- The whole **`conversationPreset`** is spread in (`schema/defaults.ts`): `endpoint` (required), `endpointType`, `model`, `region`, `chatGptLabel`, `examples`, `modelLabel`, `promptPrefix`, `temperature`, `top_p`/`topP`/`topK`, `maxOutputTokens`/`maxTokens`/`max_tokens`, penalties, `file_ids`, `resendImages`/`resendFiles`, `promptCache`/`promptCacheTtl`, `thinking`/`thinkingBudget`/`thinkingLevel`, `effort`, `system`, `imageDetail`, `agent_id`, `codeApprovalMode`, `codeEnvironmentMode`, `codeWorkspaces[{environmentId, workspaceId}]`, `assistant_id`, `instructions`, `stop`, **`isArchived`**, `iconURL`, `greeting`, `spec`, `tools`, `maxContextTokens`, `useResponsesApi`, `web_search`, `url_context`, `disableStreaming`, `fileTokenLimit`, `reasoning_*`, `verbosity`.
- `agent_id`; `initial_agent_id` (hidden, immutable primary agent used for Insights).
- `subagentThread{rootConversationId, parentConversationId, parentMessageId, parentToolCallId, parentAgentId, subagentType, subagentKind: agent|graph, depth}`: marks a child thread and excludes it from search.
- `subagentThreadLease{token, taskId, expiresAt}` (hidden).
- Hidden event-actor state:
  - `agentEventBinding{bindingId, sourceKeyId, actorId}`;
  - `agentEventActor{generation, checkpoint{threadId, checkpointId, checkpointNs}, contextFingerprint, skillManifest[], discoveredToolNames[], summary, contextMeta (with fading tiers), compactionSemanticIndex{version, entries[]}, previousCheckpoint, requiresColdStart}`;
  - `agentEventActorCleanup[]`, `agentEventActorReconciliations[]` (status enum), `agentEventActorEpoch`, `agentEventActorLegacyTurn`, `agentEventActorSuspension{..., status enum}`.
- `tags[String]` (Meili), linked to `ConversationTag.tag`.
- `chatProjectId` (string → `ChatProject._id`); `files[String]`.
- `expiredAt`: **TTL**, the retention deadline.
- `pinned`; `archivedAt`.
- Indexes:
  - unique `{conversationId, user, tenantId}`;
  - TTL `{expiredAt}`;
  - many sidebar cursor indexes: `{user, isArchived, updatedAt:-1, _id:-1}`, `{user, isArchived, createdAt:-1, updatedAt:-1, _id:-1}`, `{user, isArchived, title, updatedAt, _id}`, `{user, isArchived, archivedAt:-1, createdAt:-1, _id:-1}`, `{user, pinned, updatedAt:-1, _id:-1}`, `{user, chatProjectId, updatedAt|createdAt:-1, _id:-1}`;
  - Insights: `{tenantId, isTemporary, createdAt:-1, _id:-1}` and variants with `agent_id` / `initial_agent_id`;
  - `{user, isTemporary, expiredAt}`, `{user, subagentThread.parentConversationId}`, `{user, subagentThreadLease.expiresAt}`;
  - unique sparse `agentEventBinding.bindingId`;
  - reconciliation-metrics index; Meili sync index.

**Message** (`messages`), T, ts, Meili. Individual chat message, tree-linked through `parentMessageId`.
- `messageId` (string), `conversationId` (string), `user` (string).
- `model`, `endpoint`, `conversationSignature`, `clientId`, `invocationId`.
- `parentMessageId`: points to another `messageId`; the root uses `00000000-...`.
- `tokenCount`, `summaryTokenCount`, `sender`, `text`, `summary`.
- `isCreatedByUser`, `isUserSubmitted`, `userSubmittedPaths[]`, `userSubmittedMessageFieldPaths[{path, field}]`.
- `isTemporary`, `unfinished`, `error`, `finish_reason`.
- `feedback{rating: thumbsUp|thumbsDown, tag, text}`.
- `langfuseSampled`, `langfuseDestinationIds`, `langfuseRunId`.
- `files[Mixed]`; `content[Mixed]` (structured content parts); `thread_id`; `iconURL`; `metadata`.
- Hidden: `subagentTranscript{taskId, mode, messagesJson}`, `subagentActivityProjection`, `subagentTriggerProjection`.
- Hidden `subagentTask{attemptKey, parentRunId, requestFingerprint, status: running|completed|error|cancelled, resultClaim, controlReceipts[]}`.
- `contextMeta{calibrationRatio, encoding, fading, fadingTiers}`.
- `attachments[Mixed]`, `manualSkills[]`, `alwaysAppliedSkills[]`, `quotes[]`.
- `expiredAt`: **TTL**.
- `addedConvo`.
- Indexes:
  - unique `{messageId, user, tenantId}`;
  - `{conversationId, user, createdAt, _id}`, the main fetch path;
  - `{user, conversationId, createdAt:-1, _id:-1}` named `subagent_thread_latest_message`;
  - partial `subagent_parent_run_status_updated`;
  - Insights indexes; Meili sync index; TTL.

**ConversationTag** (`conversationtags`), T, ts. User bookmarks/tags.
- `tag`, `user` (string), `description`, `count`, `position`.
- Indexes: unique `{tag, user, tenantId}`.

**SharedLink** (model name `SharedLink`, `schema/share.ts`, collection `sharedlinks`), T, ts. Public or shared read-only snapshot of a conversation.
- `conversationId`, `title`, `user`.
- `messages[ObjectId → Message]`.
- `shareId`: random nanoid public ID.
- `targetMessageId`.
- `expiredAt`: **TTL**.
- `snapshotFiles`; `fileSnapshots[{file_id, source, storageKey, filepath, type, filename, bytes, width, height, model, llmDeliveryPath, previewRevision, ...}]`.
- Indexes: `{conversationId, user, targetMessageId, tenantId}`; `{updatedAt:-1}`.
- Access is also controlled through ACL `resourceType: sharedLink`.

**Preset** (`presets`), T, ts. Saved conversation settings.
- `presetId`, `title`, `user`, `defaultPreset`, `order`, plus the whole `conversationPreset`.
- Indexes: unique `{presetId, tenantId}`.

**ToolCall** (`toolcalls`), T, ts. Persisted tool-call results, such as code execution output.
- `conversationId`, `messageId`, `toolId`, `user` → User.
- `result` (Mixed), `attachments` (Mixed), `blockIndex`, `partIndex`.
- `expiredAt`: **TTL**.
- Indexes: `{messageId, user, tenantId}`; `{conversationId, user, tenantId}`.

**ChatProject** (collection `chatprojects`), T, ts. User "projects" (folders) that group conversations.
- `name` (max 100), `description`, `user` (string).
- Denormalized stats: `conversationCount`, `lastConversationAt`, `lastConversationId`.
- Indexes: `{user, name, _id}`, `{user, createdAt:-1, _id:-1}`, `{user, lastConversationAt:-1, _id:-1}`.

**AgentQueuedTurn** (`schema/queuedTurn.ts`, collection `agentqueuedturns`), T, ts. Durable queue of user turns sent while an agent is still busy on a conversation.
- `user` → User; `conversationId`; `agentId`; `parentMessageId`.
- `clientRequestId`: idempotency key.
- `fingerprint`, `laneId`.
- `sequence`: ordering number from AgentQueuedTurnSequence.
- `reservationWriterId`; `activeSlot` (0–99, capacity 100); `admissionSlot`.
- `status`: `reserving|queued|claimed|admitted|cancelled|dead`.
- `priority`, `text` (max 32 KB), `files[fileRef]`, `quotes`, `manualSkills`.
- `attempts`, `availableAt`.
- `deliveryKey` → `AgentTriggerDelivery.deliveryKey`.
- `deliveryState`: `pending|publishing|published|retiring|retired`.
- Claim fields: `claimId`, `claimBy`, `claimUntil`.
- Admission fields: `admissionStartedAt`, `admissionId`, `admission*`.
- `reconciliation*` fields.
- `terminalReceipt{outcome: admitted|cancelled|dead, settledAt, ..., failure}`.
- Indexes, all scoped by `{tenantId, user, conversationId, ...}`:
  - unique on `clientRequestId`;
  - partial-unique on `sequence`, `activeSlot`, claimed lane (`status:'claimed'`), admission-started lane, and `admissionSlot`;
  - claim-order index `{status, priority:-1, availableAt, sequence}`;
  - reconciliation indexes.

**AgentQueuedTurnSequence** (`queuedTurnSequence.ts`), T, ts. Per-lane sequence counter and single-writer lease.
- `_id`: string, `laneKey = sha256([tenantId, user, conversationId])` in base64url.
- `user`, `conversationId`, `laneId`, `value`, `reservationId`, `writerId`, `writerUntil`, `retiredAt`.
- `expiresAt`: **TTL**. Retired lanes are kept for 24 hours.

### 2.3 Agents and tooling

**Agent** (`agents`), T, ts. Agent definition with a version history.
- `id`: business ID string, for example `agent_xxx`.
- `name`, `description`, `instructions`, `avatar` (Mixed).
- `provider`, `model` (both required); `model_parameters`.
- `artifacts`, `access_level`, `recursion_limit`.
- `tools[]`; `skills[]` (Skill `_id` strings); `skills_enabled`, `skill_authoring_enabled`, `skills_scope` (enum).
- `tool_kwargs`.
- `actions[]`: strings of the form `domain<delimiter>action_id`.
- `author` → User; `authorName`.
- `hide_sequential_outputs`, `end_after_tools`, `stateful_code_sessions`.
- `stateful_code_environment`: `user|agent-user|conversation`.
- `code_environment_id` → `CodeEnvironment.environmentId`; `code_workspace_id`.
- `repositoryInstructions`; `git_identity{name, email}`.
- `agent_ids` (deprecated); `edges[Mixed]` (graph edges; `edges.to` holds agent IDs).
- `conversation_starters`.
- `tool_resources` (Mixed: `{file_search:{file_ids}, execute_code:{file_ids}, ...}` → `File.file_id`).
- **`versions[Mixed]`**: full snapshots.
- `category` (→ `AgentCategory.value`); `support_contact`; `is_promoted`.
- `mcpServerNames[]` (→ `MCPServer.serverName`); `tool_options`; `subagents`.
- `memory_scope`: `user|agent`.
- Indexes: unique `{id, tenantId}`; `{mcpServerNames, tenantId}`; `{updatedAt:-1, _id}`; `{tenantId, updatedAt:-1, _id}`; `{edges.to}`.

**AgentCategory** (`agentcategories`), T, ts. Marketplace categories.
- `value` (lowercase), `label`, `description`, `order`, `isActive`, `custom`.
- Indexes: unique `{value, tenantId}`; `{isActive, order}`; `{order, label}`.

**Action** (`actions`), T. OpenAPI "actions" attached to agents or assistants.
- `user` → User; `action_id`; `type` (default `action_prototype`); `settings`.
- `agent_id` → `Agent.id`; `assistant_id`.
- `metadata{api_key, auth{authorization_type, custom_auth_header, type: service_http|oauth|none, authorization_url, client_url, scope, token_exchange_method}, domain (required), privacy_policy_url, raw_spec, oauth_client_id, oauth_client_secret}`.

**Assistant** (`assistants`), T, ts. Local metadata for OpenAI/Azure Assistants.
- `user` → User; `assistant_id`; `endpoint`; `avatar`; `conversation_starters`; `access_level`; `file_ids`; `actions`; `append_current_datetime`.
- Indexes: `{tenantId, avatar.filepath}`.

**MCPServer** (`mcpservers`), T, ts. MCP server definitions created by users or admins in the database.
- `serverName`; `normalizedServerName` (derived).
- `config` (Mixed: title, url, oauth, and so on).
- `author` → User.
- Indexes: unique `{serverName, tenantId}`; unique partial `{normalizedServerName, tenantId}`; `{updatedAt:-1, _id}`.
- ACL `resourceType: mcpServer`.

**MemoryEntry** (`memoryentries`), T. User memory key/value store.
- `userId` → User.
- `key`: must match `^[a-z_]+$`.
- `value`, `agentId` (partition; null means the shared pool), `tokenCount`, `updated_at`.
- Indexes: `{userId, agentId, key}`.

**ToolFavorite** (`schema/favorite.ts`, `toolfavorites`), T, ts.
- `user` → User; `itemType`: `builtin|tool|mcp|skill`; `itemId`.
- Indexes: unique `{user, itemType, itemId}`. This index deliberately has no `tenantId`.

**Skill** (`skills`), T, ts. Claude-style skills: a SKILL.md body plus frontmatter.
- `name`: kebab-case, max 64. Reserved prefixes `anthropic-`/`claude-` and reserved words are rejected.
- `displayTitle`, `description` (max 1024), `body` (max 100 KB), `frontmatter` (Mixed).
- `disableModelInvocation`, `userInvocable`, `allowedTools[]`, `category`.
- `author` → User; `authorName`.
- `version` (≥1).
- `source`: `inline|github|notion`.
- `sourceMetadata{sourceId, upstreamId, path, sha, ...}`.
- `fileCount`, `alwaysApply`.
- Indexes: unique `{name, author, tenantId}`; `{author, tenantId}`; `{category, updatedAt:-1}`; `{updatedAt:-1, _id}`; `{source, sourceMetadata.upstreamId, tenantId}`; `{source, sourceMetadata.sourceId, tenantId}`.
- ACL `resourceType: skill`.

**SkillFile** (`skillfiles`), T, ts. Supporting files inside a skill bundle.
- `skillId` → Skill.
- `relativePath`: validated with no `..`, no absolute paths, and not `SKILL.md`.
- `file_id`, `filename`, `filepath`, `storageKey`, `storageRegion`, `source`, `sourceMetadata`, `mimeType`, `bytes`.
- `category`: `script|reference|asset|other`.
- `isExecutable`, `author`, `content`, `isBinary`.
- `codeEnvRef`: sub-schema `{kind: skill|agent|user, id, storage_session_id, file_id, version, executionProfile, executionRouteKey, provisionedAt, sandboxFilename}` (`schema/codeEnvRef.ts`).
- `codeEnvRefs`: Mixed map keyed by route.
- Indexes: unique `{skillId, relativePath}`; `{skillId, category}`.

**SkillSyncCredential** (`skillsynccredentials`), no plugin, ts. GitHub tokens for skill sync.
- `provider`: `github`; `credentialKey`.
- `encryptedToken` and `tokenHash`: both hidden.
- `createdBy`, `updatedBy`.
- Indexes: unique `{provider, credentialKey}`.

**SkillSyncStatus** (`skillsyncstatuses`), no plugin (but has a `tenantId` field), ts. Sync run state and lock per source.
- `provider`, `sourceId`.
- `status`: `idle|running|succeeded|partial|failed|skipped`.
- `credentialKey`, `owner`, `repo`, `ref`, `paths`.
- Timestamps: `startedAt`, `finishedAt`, `lastSuccessAt`, `lastFailureAt`.
- Counts: synced, deleted and skipped skills and files; `skippedSkills[]`, `skippedFiles[]`.
- Lock: `lockOwner`, `lockExpiresAt`.
- Indexes: unique `{provider, sourceId, tenantId}`.

**CodeEnvironment** (`codeenvironments`), T, ts. Registered code-execution sandboxes, managed or attached.
- `environmentId`, `name`.
- `type`: `managed|attached`.
- `baseURL`, `controlPlaneId`.
- `createdBy` → User.
- `ownerSlot`: per-owner cap. `_id` is set to a deterministic hash of owner and slot, so the cap is enforced by the unique `_id`.
- `pendingAgentReferences[{reservationId, expiresAt}]`.
- Lease fields for deletion, registration and revocation.
- `workerId`, `revocationTokenEnv`.
- `workerPrincipal{type: deployment|tenant|user|role|group, id}`.
- `settings.permissions{fileWrite, commandExecution: allow|ask|deny}`.
- Indexes: unique `{environmentId, tenantId}`; `{updatedAt:-1, _id}`.
- Also uses the raw collection `code_environment_tombstones` (`methods/codeEnvironment.ts`).
- ACL `resourceType: codeEnvironment`.

**CodeEnvRef**: an embedded sub-schema only, not a collection.

### 2.4 Files

**File** (`files`), T, ts. Metadata for uploaded and generated files; the bytes live in local storage, S3, Azure, Firebase or the vector DB.
- `user` → User; `conversationId` → Conversation; `messageId`.
- `file_id`: UUID business key; `temp_file_id`.
- `bytes`, `filename`, `filepath`, `storageKey`, `storageRegion`, `object`, `embedded`, `type` (MIME).
- `text`; `textFormat`: `html|text`.
- `status`: `pending|ready|failed`, for deferred previews; `previewError`; `previewRevision`.
- `context`: FileContext, for example `execute_code`, `run_artifact`, `agents`, `avatar`.
- `usage`, `source` (FileSources), `model`, `width`, `height`.
- `metadata{runFile{runId, executionId, agentId, parentExecutionId, ..., inputFileIds}, codeEnvRef, codeEnvRefs, sourceDispatchedAt, embeddedEntities[], destinationChosen, routingMimeType}`.
- `llmDeliveryPath`: `provider|text|none`.
- `expiresAt`: **TTL `expires:3600`**, for short-lived uploads.
- `expiredAt`: retention deadline, deleted by an application sweep, **not** a TTL index.
- `deletionAttempts`, `deletionRetryAt`.
- Indexes:
  - `{expiredAt}`; `{createdAt, updatedAt}`;
  - unique partial `{filename, conversationId, context, tenantId}` where `context=execute_code`;
  - unique partial `run_artifact_identity` where `context=run_artifact`.

### 2.5 Prompts

**PromptGroup** (`promptgroups`), T, ts. A named prompt with a "production" version and an optional slash command.
- `name`, `numberOfGenerations`, `oneliner`, `category`.
- `productionId` → Prompt.
- `author` → User; `authorName`.
- `command`: `^[a-z0-9-]+$`.
- Indexes: `{numberOfGenerations:-1, updatedAt:-1, _id}`.
- ACL `resourceType: promptGroup`.

**Prompt** (`prompts`), T, ts. One version of a prompt.
- `groupId` → PromptGroup; `author` → User; `prompt` (text).
- `type`: `text|chat`.
- Indexes: `{createdAt, updatedAt}`.

### 2.6 Scheduling and triggers

**Schedule** (`schedules`), T, ts. Recurring agent prompts, similar to cron jobs.
- `id` (string), `user` → User, `name`, `prompt` (max 32 KB).
- `agent_id` → `Agent.id`.
- `cadence{frequency: hourly|daily|weekdays|weekly|cron, hour, minute, daysOfWeek[], expression}`.
- `timezone`; `target`: `new`.
- `chatProjectId`, `file_ids`, `tools`, `cron`.
- `enabled`; `disabledReason` (enum: `mcp_reauth_required`, `too_many_failures`, `agent_deleted`, `insufficient_balance`, `project_deleted`, ...).
- `nextRunAt`.
- Lease: `leaseUntil`, `leaseBy`, `claimToken` (fencing token).
- `configRevision`, `deleting`, `deletionSuspension{token, enabled, nextRunAt}`.
- `erased`, `erasedAt`.
- `slot`: per-user cap.
- `clientRequestId`, `clientRequestDigest`: create idempotency.
- `eraseAttemptedAt`.
- `lastRun{conversationId, status, error, mcp[], firedAt, scheduledFor}`.
- Counters: `runCount`, `countedFor[Date]`, `countersAsOf`, `failureCount`, `balanceSkipCount`.
- Indexes:
  - **TTL `{erasedAt}` 24 hours, partial on `erased:true`**;
  - unique `{id, tenantId}`;
  - `{enabled, nextRunAt}`;
  - unique partial `{user, slot}` (non-deleting);
  - unique partial `{user, clientRequestId}`;
  - `{deleting, eraseAttemptedAt}`.

**ScheduleRun** (`scheduleruns`), T, ts. One row per firing of a schedule.
- `scheduleId` → `Schedule.id`; `user`; `scheduledFor`; `firedAt`.
- `conversationId`; `checkpointNamespace` (hidden).
- `deliveryKey` → AgentTriggerDelivery; `chatProjectId`; `resumeClaimedAt`.
- `status`: `started|requires_action|success|error|interrupted|skipped_overlap|skipped_balance`.
- `error`, `mcp[]`, `droppedFileIds`, `durationMs`, `bookkept`.
- `settledAt`: **TTL 90 days**, set only when the run is terminal.
- `capacitySlot`, `admissionOnly`.
- Abort fields: `abortRequestedAt`, `abortSource` (`stop|deletion`), `abortPersistedAt`.
- `reconciledAt`, `configRevision`.
- Indexes:
  - unique `{scheduleId, scheduledFor}`;
  - unique partial `{scheduleId}` where `status:'started'`: at most one active run per schedule;
  - unique partial `{capacitySlot}` where `status:'started'`: global concurrency cap;
  - `{scheduleId, firedAt:-1}`, `{status, firedAt}`, `{status, reconciledAt, firedAt}`.

**AgentTriggerDelivery** (`triggerDelivery.ts`, `agenttriggerdeliveries`), T, ts. Durable outbox/queue of events that wake agents (webhooks, schedules, background tool completions, queued turns).
- `deliveryKey` (unique), `fingerprint`.
- `orderingKey`: lane → `AgentTriggerLaneSequence._id`.
- `laneSequence`; `envelope` (Mixed); `user`.
- `status`: `staging|capability_staging|batched|pending|capability_pending|leased|capability_leased|succeeded|capability_dead|dead`.
- `requiredWorkerCapability`, `capabilityStatus`, `claimAvailableAt`, `capabilityLease*`, `producerLeaseUntil`.
- `backgroundToolResult{status, output, settledAt, resultClaim}` (hidden).
- `attempts`, `availableAt`, `envelopeBytes`.
- Coalescing and batching: `coalesceKey`, `coalesceFrom`, `coalesceUntil`, `batchSize`, `batchBytes`, `batchMemberIds[→self]`, `batchRootId`.
- `awaitTerminalHandling`.
- `handling{status: started|applied|completed_no_action|failed|cancelled, conversationId, streamId, generationCreatedAt, action{toolName, toolCallId}}`.
- `actorReceipt`, `actorDetachedAction` (with history), `actorActionAdmitted*`.
- `leaseBy`, `leaseUntil`, `claimToken`, `lastError`, `result`, `history[]` (max 64), `settledAt`.
- `expiresAt`: **TTL**, set only on success; succeeded rows are kept 90 days.
- `requeueCount`, `stagingRecoveryAt`, `laneCleanupPendingAt`.
- Indexes: unique `deliveryKey`; roughly 16 claim, lane and sparse recovery indexes.

**AgentTriggerLaneSequence** (`triggerLaneSequence.ts`), T, ts. Per-`orderingKey` sequence counter plus publisher lock.
- `_id` = `orderingKey`; `value`; `user`.
- `tailDeliveryId` and `publisherDeliveryId` → AgentTriggerDelivery.
- `publisherRequeueCount`, `publisherStartedAt` (sparse index), `cleanupRequestedAt`.

**AgentTriggerUserPurge** (`triggerUserPurge.ts`), T, ts. Fence used while a user's trigger data is being deleted.
- `_id` = User `_id`; `fenceStartedAt`.

### 2.7 Configuration, UI and miscellaneous

**Config** (`configs`), T, ts. Admin overrides to `librechat.yaml`, per principal.
- `principalType`, `principalId`, `principalModel`. The base config uses principal `role:__base__` (`admin/capabilities.ts`).
- `priority`: merge order.
- `overrides` (Mixed), `tombstones[String]` (deleted paths), `isActive`, `configVersion`.
- Indexes: unique `{principalType, principalId, tenantId}`; `{principalType, principalId, isActive, tenantId}`; `{priority, isActive, tenantId}`.

**Banner** (`banners`), T, ts.
- `bannerId`, `message`, `displayFrom`, `displayTo`.
- `type`: `banner|popup`.
- `isPublic`, `persistable`.

**Categories** (`schema/categories.ts`): the schema exists (label and value, both unique with `tenantId`) but **no model is registered**. `methods/categories.ts` returns 9 hard-coded prompt categories.

**Fading** (`schema/fading.ts`): not a collection. It is a shared definition of `contextMeta.fading` / `fadingTiers[{agentId, v, budgetTokens, masked}]`, embedded in Message and Conversation.

**Insights** (`methods/insights.ts`): not a collection. Aggregation pipelines over Conversation, Message and User, backed by the Insights indexes.

**Raw collections not managed as models:**
- `mcp_authorization_fence_retries` (`methods/mcpAuthorizationFenceRetry.ts`): `_id` is a JSON of `[tenant, user, server, version]`.
- `code_environment_tombstones`.
- `__transaction_test__`.

---

## 3. Relationships (ER list, ready for Mermaid)

```
User ||--o{ Session : "Session.user -> User._id"
User ||--o{ Token : "Token.userId -> User._id"
User ||--o| Balance : "Balance.user -> User._id"
User ||--o{ Transaction : "Transaction.user -> User._id"
Conversation ||--o{ Transaction : "Transaction.conversationId -> Conversation.conversationId"
User ||--o{ Key : "Key.userId -> User._id"
User ||--o{ PluginAuth : "PluginAuth.userId(str) -> User._id"
User ||--o{ AgentApiKey : "AgentApiKey.userId -> User._id"
Role ||--o{ User : "User.role -> Role.name"
Group }o--o{ User : "Group.memberIds[] -> User._id(str) | User.idOnTheSource"
AccessRole ||--o{ AclEntry : "AclEntry.roleId -> AccessRole._id"
User ||--o{ AclEntry : "AclEntry.principalId (principalType=user)"
Group ||--o{ AclEntry : "AclEntry.principalId (principalType=group)"
Role ||--o{ AclEntry : "AclEntry.principalId = Role.name (principalType=role)"
AclEntry }o--|| Agent : "resourceId -> Agent._id (resourceType=agent|remoteAgent)"
AclEntry }o--|| PromptGroup : "resourceId -> PromptGroup._id"
AclEntry }o--|| MCPServer : "resourceId -> MCPServer._id"
AclEntry }o--|| Skill : "resourceId -> Skill._id"
AclEntry }o--|| CodeEnvironment : "resourceId -> CodeEnvironment._id"
AclEntry }o--|| SharedLink : "resourceId -> SharedLink._id"
User ||--o{ AclEntry : "AclEntry.grantedBy"
User|Group|Role ||--o{ SystemGrant : "SystemGrant.principalId"
User|Group|Role ||--o| Config : "Config.principalId(str)"
User ||--o{ RefreshTokenBridge : "RefreshTokenBridge.userId(str)"
User ||--o{ Conversation : "Conversation.user(str) -> User._id"
Conversation ||--o{ Message : "Message.conversationId -> Conversation.conversationId"
Conversation }o--o{ Message : "Conversation.messages[] -> Message._id"
Message ||--o{ Message : "Message.parentMessageId -> Message.messageId"
Conversation ||--o{ Conversation : "subagentThread.parentConversationId/rootConversationId -> Conversation.conversationId"
Message ||--o{ Conversation : "subagentThread.parentMessageId -> Message.messageId"
ChatProject ||--o{ Conversation : "Conversation.chatProjectId -> ChatProject._id(str)"
ConversationTag }o--o{ Conversation : "Conversation.tags[] <-> ConversationTag.tag (per user)"
Agent ||--o{ Conversation : "Conversation.agent_id / initial_agent_id -> Agent.id"
Conversation ||--o{ SharedLink : "SharedLink.conversationId -> Conversation.conversationId"
SharedLink }o--o{ Message : "SharedLink.messages[] -> Message._id; targetMessageId -> Message.messageId"
Message ||--o{ ToolCall : "ToolCall.messageId -> Message.messageId; ToolCall.conversationId"
User ||--o{ Preset : "Preset.user(str)"
Conversation ||--o{ File : "File.conversationId -> Conversation.conversationId"
Message ||--o{ File : "File.messageId -> Message.messageId"
User ||--o{ File : "File.user -> User._id"
Agent }o--o{ File : "Agent.tool_resources.*.file_ids[] -> File.file_id"
User ||--o{ Agent : "Agent.author -> User._id"
Agent ||--o{ Agent : "Agent.edges[].to / agent_ids[] -> Agent.id"
AgentCategory ||--o{ Agent : "Agent.category -> AgentCategory.value"
Agent }o--o{ Action : "Agent.actions[] ~ Action.action_id; Action.agent_id -> Agent.id"
Agent }o--o{ MCPServer : "Agent.mcpServerNames[] -> MCPServer.serverName"
Agent }o--o{ Skill : "Agent.skills[] -> Skill._id(str)"
Agent }o--|| CodeEnvironment : "Agent.code_environment_id -> CodeEnvironment.environmentId"
User ||--o{ Action : "Action.user"
User ||--o{ Assistant : "Assistant.user"
User ||--o{ MCPServer : "MCPServer.author"
User ||--o{ MemoryEntry : "MemoryEntry.userId"
Agent ||--o{ MemoryEntry : "MemoryEntry.agentId -> Agent.id (partition)"
User ||--o{ ToolFavorite : "ToolFavorite.user"
User ||--o{ Skill : "Skill.author"
Skill ||--o{ SkillFile : "SkillFile.skillId -> Skill._id"
SkillSyncCredential ||--o{ SkillSyncStatus : "SkillSyncStatus.credentialKey -> SkillSyncCredential.credentialKey"
SkillSyncStatus ||--o{ Skill : "Skill.sourceMetadata.sourceId -> SkillSyncStatus.sourceId (source=github)"
User ||--o{ CodeEnvironment : "CodeEnvironment.createdBy"
PromptGroup ||--o{ Prompt : "Prompt.groupId -> PromptGroup._id"
PromptGroup ||--|| Prompt : "PromptGroup.productionId -> Prompt._id"
User ||--o{ PromptGroup : "PromptGroup.author"
User ||--o{ Schedule : "Schedule.user"
Agent ||--o{ Schedule : "Schedule.agent_id -> Agent.id"
ChatProject ||--o{ Schedule : "Schedule.chatProjectId"
Schedule ||--o{ ScheduleRun : "ScheduleRun.scheduleId -> Schedule.id"
ScheduleRun }o--|| Conversation : "ScheduleRun.conversationId"
ScheduleRun }o--|| AgentTriggerDelivery : "ScheduleRun.deliveryKey -> AgentTriggerDelivery.deliveryKey"
AgentTriggerLaneSequence ||--o{ AgentTriggerDelivery : "AgentTriggerDelivery.orderingKey -> AgentTriggerLaneSequence._id"
AgentTriggerDelivery ||--o{ AgentTriggerDelivery : "batchRootId / batchMemberIds[]"
AgentTriggerDelivery }o--|| Conversation : "handling.conversationId"
User ||--o| AgentTriggerUserPurge : "AgentTriggerUserPurge._id = User._id"
Conversation ||--o{ AgentQueuedTurn : "AgentQueuedTurn.conversationId (+user)"
AgentQueuedTurnSequence ||--o{ AgentQueuedTurn : "lane = sha256(tenant,user,conversationId)"
AgentQueuedTurn }o--|| AgentTriggerDelivery : "AgentQueuedTurn.deliveryKey"
Conversation ||--o| (agentEventBinding) : "Conversation.agentEventBinding.bindingId <- AgentTriggerDelivery.actorReceipt.bindingId"
Tenant(string) ||--o{ * : "tenantId on almost all; AuditLog.chainKey = tenantId|__platform__"
```

Note: `user` is a **string** in Conversation, Message, ConversationTag, SharedLink, Preset, ChatProject and PluginAuth, but an **ObjectId** in File, ToolCall, Balance, Transaction, Agent, Action and most newer collections.

---

## 4. Key data-access methods by domain

### Identity and access
- **user** (`methods/user.ts`):
  - `findUser`, `findUsers`, `countUsers`, `getUserById`, `updateUser`, `deleteUserById`, `acceptTerms`, `toggleUserMemories`, `updateUserStatefulCodeEnvironment`, `updateUserPlugins`.
  - `createUser(data, balanceConfig, disableTTL)` inserts the user with a 7-day `expiresAt` unless `disableTTL`. When balance is enabled it upserts Balance with `$inc: startBalance` plus the auto-refill settings.
  - `generateToken` signs a JWT (`DEFAULT_SESSION_EXPIRY` is 15 minutes). `invalidateAuthUserDocCache` clears the cached auth user document.
  - `claimSamlIdentity`.
  - Subagent admission fences: `fenceSubagentAdmission`, `renew…`, `release…`, `isSubagentOwnerAdmissible`.
  - Account-deletion fencing: `beginAgentTriggerUserDeletion`, `recover…`, `cancel…`, `isAgentTriggerPrincipalActive`.
- **session**: `createSession` saves a Session, signs a refresh JWT `{id, sessionId}` with `JWT_REFRESH_SECRET`, and stores `sha256(token)`. Also `upsertSession`, `findSession` (by hashed token or ID), `updateExpiration`, `deleteSession`, `deleteAllUserSessions`, `countActiveSessions`.
- **token**: `createToken`, `findToken` (email trimmed and lowercased), `updateToken`, `deleteTokens`.
- **refreshTokenBridge** and **openidRefreshFlight**:
  - `upsert/find/deleteRefreshTokenBridge(s)`.
  - `acquireOpenIDRefreshFlight` (compare-and-swap lock on `key` and `lockExpiresAt`), `complete`, `renew`, `fail`, `claim/releaseDelivery`, `revoke`.
- **role**:
  - `initializeRoles` seeds ADMIN and USER from the data-provider defaults.
  - `getRoleByName` (cached), `updateAccessPermissions` (merges permission booleans, invalidates cache), `create/deleteRoleByName`.
  - `updateUsersByRole`, `listUsersByRole`, `migrateRoleSchema`.
  - Throws `RoleConflictError`.
- **accessRole**: `seedDefaultRoles`; `findRoleByIdentifier`, `findRolesByResourceType`; `getRoleForPermissions` maps bits to a role; `visibleTenantRoles`.
- **aclEntry** (`methods/aclEntry.ts`), central to authorization:
  - `permissionBitSupersets(bit)` enumerates every value in [0..31] that contains the bit, for use in `permBits: {$in: [...]}`. This replaces `$bitsAllSet`, which Cosmos and DocumentDB cannot handle.
  - `hasPermission(principals, type, id, bit)` runs `findOne({$or: principals, resourceType, resourceId, permBits: $in})`.
  - `getEffectivePermissions` / `…ForResources` compute the bitwise OR of matching entries.
  - `findAccessibleResources(principals, type, bit, ids?, readPrimary)` returns `distinct('resourceId')`. List endpoints use it: the result becomes `accessibleIds` for agents, skills, prompts and MCP servers.
  - `findPublicResourceIds`.
  - `grantPermission` (upsert), `revokePermission`, `modifyPermissionBits`, `replaceRoleBits`, `mutatePermissionEntry` (bounded retries, `PERM_BITS_WRITE_ATTEMPTS`).
  - `bulkWriteAclEntries` (through `tenantSafeBulkWrite`), `deleteAclEntries`.
  - `getSoleOwnedResourceIds` (used for user-deletion cascades); `aggregateAclEntries`.
- **userGroup**:
  - `getUserPrincipals({userId, role, idOnTheSource})` returns `[USER ObjectId, ROLE name, ...GROUP ids (memberIds contains idOnTheSource or userId), PUBLIC]`. Group IDs are cached with `getCache` and double invalidation.
  - Group CRUD: `createGroup`, `upsertGroupByExternalId`, `addUserToGroup`/`removeUserFromGroup`, `syncUserEntraGroups`, `searchPrincipals` (relevance-scored across users and groups), `listGroups`, `bulkUpdateGroups`.
  - `runAfterTransaction` defers side effects until a commit.
- **systemGrant**:
  - `tenantCondition(tenantId)` = `{$or: [{tenantId}, {tenantId: {$exists: false}}]}`, so platform grants apply to every tenant.
  - `hasCapabilityForPrincipals`; `hasAnyConfigReadAccess` (regex `^(read|manage):configs(:section)?$`); `getHeldCapabilities`.
  - `grantCapability`: the only permitted write path; normalizes `principalId`.
  - `revokeCapability`, `listGrants`, `seedSystemGrants`, `deleteGrantsForPrincipal`.
- **auditLog**:
  - `recordAuditEntry` reads the chain's last `{seq, hash}`, computes `hash = sha256(stableStringify(canonical entry incl. prevHash, seq, createdAt))`, inserts, and retries on an E11000 collision on `{chainKey, seq}`.
  - `listAuditLogPage` (keyset pagination on `seq`/`createdAt`), `streamAuditLogEntries` (export), `verifyAuditChain`.
  - `purgeAuditLogEntries`: retention purge via the raw `Model.collection`, which bypasses the append-only hooks.
- **agentApiKey**: `generateApiKey` returns a random key plus prefix and stores only the hash; `validateAgentApiKey` looks up by hash and touches `lastUsedAt`; `list`, `delete`, `deleteAll`.
- **key**, **pluginAuth**: CRUD over encrypted values (`getUserKey`, `getUserKeyValues`, `getUserKeyExpiry`, `updateUserKey`; `findOnePluginAuth`, `findPluginAuthsByKeys`, `updatePluginAuth`).
- **Billing** (`tx.ts`, `transaction.ts`, `spendTokens.ts`):
  - `tx` holds the static rate tables `tokenValues`, `cacheTokenValues` and `premiumTokenValues`, plus `getValueKey`, `getMultiplier`, `getCacheMultiplier` and premium rates by context length.
  - `createTransaction` computes `tokenValue = rawAmount × rate`, saves, then calls `updateBalance`.
  - `updateBalance` is optimistic-concurrency compare-and-swap: `findOneAndUpdate({_id, tokenCredits: current}, {$set: max(0, current + inc)})` with 10 retries and backoff.
  - `createStructuredTransaction` splits input, write and read tokens.
  - `reserveBalance`, `renewBalanceReservation`, `releaseBalanceReservation` hold credits in `reservations[]` and `reservedCredits` using conditional updates. Expired reservations are pruned.
  - `applyAutoRefill` / `settleAutoRefill` is a two-phase refill: `pendingRefill` first, then a transaction insert.
  - `spendTokens` / `spendStructuredTokens` create prompt and completion transactions.
  - Also `bulkInsertTransactions`, `getTransactions`, `findBalanceByUser`, `upsertBalanceFields`.

### Chat
- **conversation** (`methods/conversation.ts`, about 3500 lines):
  - `saveConvo(ctx, convo, meta)`:
    - upserts on `{conversationId, user}`;
    - applies retention (`isTemporary`/`expiredAt` via `createChatExpirationDate`);
    - sets `initial_agent_id` only on insert;
    - by default refreshes `messages[]` from `getMessages`, or appends `appendMessageIds`;
    - strips hidden actor fields; supports `unsetFields`, `noUpsert` and `preserveUpdatedAt`.
  - `bulkSaveConvos` (import).
  - `getConvo`, `getConvoTitle`, `getConvoFiles`, `setConvoPinned`.
  - **`getConvosByCursor`**: keyset pagination.
    - Filters: user, archived / not archived, pinned, `tags $in`, project (`unassigned`), retention visibility, and "human" conversations (no `subagentThread`).
    - Optional Meili search over conversations and messages, whose hit IDs are applied as `conversationId $in`.
    - Sort by title, `createdAt`, `updatedAt` or `archivedAt`, with a secondary key and an `_id` tie-breaker.
    - The cursor is `base64(JSON{primary, secondary, id})`. Null sort values get their own clauses.
    - Fetches `limit+1` rows, then `attachSharedFlags`.
  - `getConvosQueried`, `searchConversation`.
  - `deleteConvos(user, filter)` cascades to messages, subagent child threads (`subagentThread.rootConversationId`), queued turns and trigger-delivery results, and updates project stats.
  - `archiveAllConvos`, `deleteNullOrEmptyConversations`.
  - Subagent leases: `reserveSubagentThread`, `acquire/renew/releaseSubagentThreadLease`.
  - Event-actor state machine (compare-and-swap updates on the hidden fields): `commitAgentEventActorState`, `store/claim/settle/cancelAgentEventActorSuspension`, `record/resolveAgentEventActorReconciliation`, `begin/completeAgentEventActorLegacyTurn`.
- **message**:
  - `saveMessage(ctx, params, meta)` upserts on `{messageId, user}`, applies the same retention rules, sanitizes `tokenCount`, and merges user-submitted provenance paths.
  - `bulkSaveMessages` (via `tenantSafeBulkWrite`), `recordMessage`, `updateMessage`, `updateMessageText`, `updateToolCallResult`.
  - `deleteMessagesSince` (branch/regenerate).
  - `getMessages(filter, select)`; `CLIENT_MESSAGE_SELECT` is the projection sent to clients.
  - `getMessagesByCursor`, `searchMessages` (Meili).
  - Subagent task receipts and claims: `claimSubagentTaskResult`, `recordSubagentTaskControlReceipt`, `claim/releaseBackgroundToolResults`.
  - `getMessagesForSubagentThreadView`, `listSubagentTasksForThreads`, `getConversationTraceRefs` (Langfuse).
- **conversationTag**: `createConversationTag` (bumps `position` via `adjustPositions`), `updateConversationTag` (renames the tag inside conversations too), `deleteConversationTag` (`$pull` from conversations), `updateTagsForConversation`, `bulkIncrementTagCounts`.
- **share**:
  - `createSharedLink` gets a nanoid `shareId`, snapshots message `_id`s, optionally snapshots files, and creates an ACL.
  - `getSharedMessages` **anonymizes** IDs (`anonymizeSharedContent`, which prefixes and nanoids IDs and rewrites file URLs to the share route).
  - `getSharedLinks` (paginated), `updateSharedLink` (refreshes the snapshot), `deleteSharedLink`, `getSharedLinkFile`, `backfillSharedLinkFiles`.
- **preset**: `getPreset(s)`; `savePreset` (upsert; when `defaultPreset` is set, other presets are un-defaulted); `deletePresets`.
- **toolCall**: CRUD by message or conversation.
- **chatProject**: `create`, `get`, `list` (cursor-based), `update`, `deleteChatProject` (`$unset` `chatProjectId` on its conversations), `assignConversationToProject`, `refreshChatProjectStats` (recount with compare-and-swap retries).
- **import**: `deleteImportedMessages` / `deleteImportedConversations`.
- **queuedTurn**:
  - `enqueueAgentQueuedTurn`: idempotent on `clientRequestId`; takes a lane-writer lease on the Sequence document; allocates `sequence` and `activeSlot` with the capacity cap of 100.
  - `claimNextAgentQueuedTurn`, `beginAgentQueuedTurnAdmission`, `markAgentQueuedTurnAdmitted`, `deadLetterAgentQueuedTurn`, `cancelAgentQueuedTurn`.
  - `reserveAgentQueuedTurnDelivery` links to a trigger delivery.
  - `retireConversationLane`, `deleteAllAgentQueuedTurnsForUser`.
  - Throws `AgentQueuedTurnConflictError`, `AgentQueuedTurnCapacityError` and `AgentQueuedTurnLaneRetiredError`.

### Agents and tools
- **agent**:
  - `createAgent`: prunes invalid skill IDs; seeds `versions: [snapshot]`; derives `mcpServerNames` from `tools`; reserves a CodeEnvironment reference (`withCodeEnvironmentReference`).
  - **`updateAgent(search, update, {updatingUserId, forceVersion, skipVersioning})`**:
    - loads the current agent;
    - computes an `actionsHash` over action metadata;
    - pushes a version snapshot `{...current, ...directUpdates, updatedAt, updatedBy, actionsHash}` unless `isDuplicateVersion` finds it matches the newest version;
    - `findOneAndUpdate` is optimistic on `updatedAt` when changing the code environment;
    - the response carries `version = versions.length`.
  - `revertAgentVersion(search, index)`, `getAgentVersions`, `getAgentWithVersionCount` (aggregation with `$size`); `getAgent` excludes `versions` by default.
  - `getListAgentsByAccess({accessibleIds, otherParams, limit, after})`: keyset cursor on `updatedAt`/`_id`, max 1000.
  - `getAgentManagementListByAccess`, `resolveAgentGraphAccess` / `getAgentGraphNodes` (walks `edges` and checks ACL on each node).
  - `getAgentIdsByMCPServerName`.
  - `add/removeAgentResourceFile(s)` (`$addToSet`/`$pull` on `tool_resources.<tool>.file_ids`), `removeAgentResourceFilesFromAllAgents`.
  - `deleteAgent` also removes ACLs, User favorites and edges. `deleteUserAgents` deletes only sole-owned agents. `countPromotedAgents`.
- **agentCategory**: `ensureDefaultCategories`/`seedCategories`, `getActiveCategories`, `getCategoriesWithCounts` (aggregate over Agent), CRUD.
- **action**: `updateAction` (upsert), `getActions` (can strip sensitive fields), `deleteAction(s)`.
- **assistant**: `updateAssistantDoc` (upsert), `getAssistant(s)`, delete.
- **mcpServer**: `createMCPServer` (`findNextAvailableServerName` adds a suffix on collision), find by name, ObjectId or author, `getListMCPServersByIds`/`ByNames`, update, delete.
- **mcpAuthority**: builds "authority proofs", i.e. digests of the current MCP-relevant authorization state (User, Group, Role, Config, MCPServer, Agent, PluginAuth, Token, AclEntry). It reads them inside a **snapshot-read-concern transaction**, and `assertMCPAuthorityProofsCurrent` compares proofs to fence stale authorization. `createMCPAuthorityLookupIndexes` must run before this is enabled.
- **memory**: `setMemory` (upsert on `{userId, key, agentId partition}`), `createMemory`, `deleteMemory`, `setMemoryById`, `getAllUserMemories`, `getFormattedMemories` (a numbered text block plus total tokens), `deleteAllUserMemories`.
- **favorite**: `getToolFavorites`, `add` (capped by `MAX_TOOL_FAVORITES`; idempotent through the unique index), `remove`.
- **skill**:
  - validators: `validateSkillName`, `validateSkillFrontmatter`, `validateSkillDescription`, `validateRelativePath`, `deriveStructuredFrontmatterFields`.
  - `createSkill`, `updateSkill` (bumps `version`), `getSkillById/ByName`.
  - `listSkillsByAccess` (cursor on `updatedAt` desc / `_id` asc, base64; filter by `accessibleIds` or `manageTenantId`, plus category and regex search).
  - `listAlwaysApplySkills`.
  - `deleteSkill` (a list of cleanup steps: files, ACLs, removal from agent allowlists), `deleteUserSkills`.
  - `findSkillBySourceIdentity`, `listSkillsBySource` (GitHub sync).
  - File operations: `upsertSkillFile`, `deleteSkillFile`, `getSkillFileByPath`, `updateSkillFileContent`, `updateSkillFileCodeEnvIds`, `bumpSkillVersionAndAdjustFileCount`.
- **skillSync**: encrypted credential CRUD (`upsertSkillSyncCredential`, `getSkillSyncCredentialToken`); status upsert; `tryAcquireSkillSyncLock` / `refresh` / `release` (compare-and-swap on `lockOwner` and `lockExpiresAt`).
- **codeEnvironment**: `createCodeEnvironmentWithinOwnerLimit` (deterministic slot `_id`s), registration and removal leases (`begin`/`cancel`/`commitCodeEnvironmentRemoval`, tombstones), `updateCodeEnvironmentSettings`, `deleteUserCodeEnvironments`.

### Files
- **file**:
  - `createFile` (upsert on `file_id`), `updateFile` (optional `previewRevision` guard), `findFileById`, `getFiles`.
  - `getToolFilesByIds`, `getCodeGeneratedFiles`, `getUserCodeFiles`.
  - `claimCodeFile` / `commitCodeFile`: race-safe upsert keyed on `filename`/`conversationId` with the `execute_code` context, ordered by `sourceDispatchedAt`.
  - Run artifacts: `claimRunArtifactFile`, `publishRunArtifactFile`, `listRunArtifacts`.
  - `updateFileUsage`/`updateFilesUsage` (`$inc`), `addFileEmbeddedEntity`, `updateFileCodeEnvRef`.
  - Deletion: `deleteFile`, `deleteFiles`, `deleteFileByFilter`, `batchUpdateFiles`.
  - Retention sweep: `getExpiredFiles`, `incrementFileDeletionAttempts`, `deferExpiredFile` (backoff).
  - `extendFilesTTL`, `sweepOrphanedPreviews`.

### Prompts
- **prompt**:
  - `createPromptGroup` creates the group and its first prompt, then sets `productionId`.
  - `savePrompt`, `makePromptProduction`, `updatePromptGroup`, `deletePrompt` (if it was production, re-points to the latest prompt or deletes the group), `deletePromptGroup` (with ACLs).
  - `getListPromptGroupsByAccess` (cursor-based, filtered by accessible IDs), `getPromptGroupsWithPrompts`, `getRandomPromptGroups`, `incrementPromptGroupUsage`.
  - `getPromptGroupAccessContext` (cached), `deleteUserPrompts`.
  - It also drops the legacy index `prompts.groupId_1_version_1`.

### Scheduling and triggers
- **schedule**:
  - `createScheduleWithSlot`: tries slots until the unique partial index accepts one; idempotent on `clientRequestId` plus digest.
  - `armSchedule`, `updateScheduleById` (rotates `claimToken`, bumps `configRevision`).
  - `claimDueSchedule`: atomic `findOneAndUpdate({enabled, !deleting, nextRunAt <= now, lease free or stale}, {$set: lease + claimToken}, sort by nextRunAt)`.
  - `revalidateClaim`, `advanceSchedule`, `disableSchedule`.
  - `reserveStartedRun` / `insertScheduleRun`: capacity slot via the unique partial index.
  - `recordRunOutcome`: idempotent per occurrence through `countedFor` and `countersAsOf`.
  - Reconciliation: `getRunsForReconciliation`, `markRunsReconciled`.
  - Abort: `requestRunAbort`, `markRunAbortPersisted`.
  - Deletion: `markScheduleDeleting` / `eraseScheduleIfDrained` (leaves a tombstone with a TTL), `suspendUserSchedulesForDeletion` / `restoreUserSchedulesFromDeletion`.
  - `ensureScheduleIndexes`.
- **triggerDelivery** (about 4000 lines): a durable outbox with ordered lanes, leasing, batching, capability gating, dead-lettering and requeue.
  - `enqueueAgentTriggerDelivery`: insert a staging row with `laneSequence` 0, allocate a lane sequence, then publish. Idempotent on `deliveryKey`.
  - `claimNextAgentTriggerDelivery`: compare-and-swap over 8 candidates, 16 attempts.
  - `beginAgentTriggerDeliveryAttempt`, `completeAgentTriggerDelivery` (sets `expiresAt` to now + 90 days), `retryAgentTriggerDelivery`, `deadLetterAgentTriggerDelivery`.
  - Dead letters: `getAgentTriggerDeadLetters`, `requeueAgentTriggerDelivery`.
  - Background tool results: `persist`/`get`/`claimAgentBackgroundToolResults`.
  - Actor action admission and detached-action lifecycle.
  - User purge: `prepareAgentTriggerUserPurge` and related functions.

### Config, UI and miscellaneous
- **config**:
  - `getApplicableConfigs(principals)` finds `{$or: [base role __base__, ...principals], isActive: true}` sorted by `priority` ascending; the API layer deep-merges the overrides in that order.
  - `upsertConfig`, `patchConfigFields` (`$set` of dotted paths, bumps `configVersion`), `tombstoneConfigField`, `unsetConfigField`, `toggleConfigActive`, `deleteConfig`, `listAllConfigs`, `findConfigByPrincipal`.
- **banner**: `getBanner(user)` returns the active banner `displayFrom <= now <= displayTo|null` of type `banner`. It is returned when public, or when a user is logged in.
- **categories**: static list.
- **insights**: `getInsights({tenantId, from, to, agentIds, page, pageSize, search, timeZone})` runs aggregation pipelines covering the timeline, active and churned users (30-day window) and per-agent message counts.

---

## 5. Migrations (`migrations/`)

All migrations are offline or admin-run and take a `mongoose.Connection`.

1. **`tenantIndexes.ts`**
   - `dropSupersededTenantIndexes(conn, {dryRun})` drops old **global unique** indexes that have been replaced by `tenantId`-compound versions, across 13 collections. Examples:
     - users: `email_1`, `googleId_1`, `openidId_1`, …
     - roles: `name_1`; agents: `id_1`
     - conversations: `conversationId_1_user_1`; messages: `messageId_1_user_1`
     - presets, agentcategories, accessroles, conversationtags, mcpservers
     - files: `filename_1_conversationId_1_context_1`
     - groups, skillsyncstatuses
   - `migrateTenantIndexes` runs in three steps: build the new tenant-scoped unique indexes first, drop the superseded ones, then create every current schema index. Writers must be stopped and `autoIndex` disabled while it runs.
   - `getTenantIndexMigrationHint` supplies the log hint used by `createModels`.
2. **`promptGroupIndexes.ts`**: `dropSupersededPromptGroupIndexes` drops `promptgroups.createdAt_1_updatedAt_1`, which was replaced by `{numberOfGenerations:-1, updatedAt:-1, _id:1}`.
3. **`mcpAuthorityIndexes.ts`**: `createMCPAuthorityLookupIndexes` creates `groups{memberIds, tenantId}`, `agents{mcpServerNames, tenantId}`, `pluginauths{userId, pluginKey, authField, tenantId}` and `tokens{userId, type, identifier, tenantId}`, using retry.
4. **`mcpServerNames.ts`**: `backfillMCPServerNormalizedNames` makes two passes over `mcpservers` with primary reads and majority read concern.
   - It computes `normalizeServerName(serverName)` and throws `MCPServerNameMigrationError` if two servers in one tenant collide.
   - It then bulk-`$set`s `normalizedServerName` in batches of 500 and creates the unique partial index.

Other schema upgrades happen inside methods at runtime:
- `migrateRoleSchema` (role.ts).
- `quarantineDuplicateAdmissionLanes` (queuedTurn) before its unique index is built.
- the prompt legacy-index drop.
- the Meili `_meiliIndexSchemaVersion` bump, which forces documents to be re-indexed.

---

## 6. Advice for a Python re-implementation

**ODM choice**
- Use **Beanie** (Pydantic v2 on Motor, or on PyMongo's async API for PyMongo ≥ 4.9). It supports `Settings.indexes` with `IndexModel` (partial filters, TTL via `expireAfterSeconds`, unique, named, sparse), event hooks (`@before_event(Insert, Replace, Save)`), and projections.
- ODMantic has weaker index and hook support. For hot paths, and anything that relies on compare-and-swap updates, write raw Motor `find_one_and_update` calls.

**Model mapping**
- Map each schema to a `Document` with `class Settings: name = "<plural lowercase collection>"`. Collection names follow Mongoose pluralization: `conversations`, `messages`, `aclentries`, `agentqueuedturns`, `agenttriggerdeliveries`, `agenttriggerlanesequences`, `agentqueuedturnsequences`, `agenttriggeruserpurges`, `openidrefreshflights`, `refreshtokenbridges`, `memoryentries`, `toolfavorites`, `sharedlinks`, `chatprojects`, `skillsynccredentials`, `skillsyncstatuses`, `scheduleruns`, `systemgrants`, `auditlogs`, `configs`, and so on.
  - Keep `_id: ObjectId`.
  - For string-`_id` documents (AgentTriggerLaneSequence, AgentQueuedTurnSequence) use `id: str = Field(alias="_id")`.
  - AgentTriggerUserPurge uses the user's ObjectId as `_id`.
- Recreate Mongoose timestamps (`createdAt`/`updatedAt`) with a base mixin that sets them on insert and in every update helper (`$set: {updatedAt: now}`, `$setOnInsert: {createdAt: now}`).
  - AuditLog sets `createdAt` itself and must not have `updatedAt`.
  - OpenIDRefreshFlight manages both itself.
- Use `select:false` fields as the default projection: build a per-model "public projection" and an explicit "+field" opt-in. Affected fields include `password`, `totpSecret`, `backupCodes`, `keyHash`, the hidden actor and subagent fields, `Balance.reservations`, `encryptedToken` and the `_meili*` fields.
- Use Pydantic validators for the Mongoose validators:
  - email regex; memory key `^[a-z_]+$`; skill name and reserved-word rules; skill-file `relativePath` rules; prompt `command`.
  - `permBits` must be an integer ≤ 31.
  - `SystemGrant`/`AuditLog` `tenantId` must never be null or empty; omit it with `exclude_none`.
  - schedule cadence conditional required fields.
- Keep `Mixed` fields as `dict[str, Any]` or `Any`: `envelope`, `config`, `overrides`, `tool_resources`, `versions`, `content`, `metadata`.
- Keep the string vs ObjectId distinction for `user` per collection, because existing data depends on it.

**Tenant isolation**
- Use `contextvars.ContextVar[TenantContext]` in place of AsyncLocalStorage. Set it in FastAPI middleware or a dependency from the JWT user's `tenantId`, and provide `run_as_system()` as a context manager that sets `SYSTEM_TENANT_ID`.
- Pure policy functions translate one-to-one from `tenant/policy.ts`: `current_scope`, `resolve_scope(strict)`, `tenant_filter`, `sanitize_mutation(guard|strip)`, `scope_replacement`, `stamp_document`, `write_predicate`.
- Mongoose middleware has no Beanie or Motor equivalent that covers every path, so **wrap the collection**. A `TenantScopedCollection` proxy should intercept `find*`, `count_documents`, `distinct`, `update*`, `delete*`, `replace_one`, `find_one_and_*`, `aggregate` (prepend `$match`), `insert_one/many` (stamp) and `bulk_write` (the `tenantSafeBulkWrite` rules).
  - Only use this proxy from the repository layer.
  - Keep a `raw` escape hatch for AuditLog, SystemGrant, RefreshTokenBridge, OpenIDRefreshFlight and SkillSync*.
- Port the conformance suite (`tenant/conformance.ts`) as a pytest suite.
- The probe can be replicated with PyMongo `CommandListener` (`monitoring.register`) to check that every command carries a `tenantId` predicate.
- Honour `TENANT_ISOLATION_STRICT`. Save of an existing document should filter by `_id` plus `tenantId in [t, None, ""]`.

**Indexes and TTL**
- Declare every index listed in section 2 with `IndexModel(..., expireAfterSeconds=N)`.
  - `expires: 604800` on `User.expiresAt` becomes a TTL of 604800 seconds.
  - `File.expiresAt` becomes 3600 seconds.
  - Session `expiration`, Token, Key, AgentApiKey, AclEntry `expiredAt`, Conversation, Message, ToolCall and SharedLink `expiredAt`, RefreshTokenBridge, OpenIDRefreshFlight, AgentQueuedTurnSequence and AgentTriggerDelivery `expiresAt` all use a TTL of 0 seconds.
  - `ScheduleRun.settledAt` is 90 days.
  - `Schedule.erasedAt` is 24 hours, partial on `erased: true`.
- Remember that `File.expiredAt` and ScheduleRun live rows rely on the "missing field never expires" behaviour.
- Build indexes at startup with retry, like `createIndexesWithRetry` and `initializeOrgCollections`, and log failures.
- Provide a CLI that performs the tenant-index migration: build the new unique indexes, drop the superseded ones, then build the rest.

**Uniqueness-based concurrency**
- Many invariants depend on unique partial indexes and E11000 handling: schedule slots, run capacity slots, one started run per schedule, queued-turn slots and lanes, code-environment owner slots, `deliveryKey`, and AuditLog `{chainKey, seq}`.
- Replicate these with the same partial indexes plus `DuplicateKeyError` retry loops.
- Use `find_one_and_update(..., return_document=AFTER)` for every lease and claim compare-and-swap: schedules, trigger deliveries, queued turns, skill sync locks, OIDC flights and balance `tokenCredits` compare-and-swap.

**Meilisearch sync**
- Use the official `meilisearch-python-sdk` (async).
- On save or upsert of a Conversation or Message, write `_meiliIndexVersion = str(ObjectId())` and `_meiliIndex = False`. Then run a background task that upserts or deletes in Meili, waits for the task, and acknowledges with `update_one({_id, _meiliIndexVersion: v}, {$set: {_meiliIndex: True, _meiliIndexSchemaVersion: 1}})`. Re-read and retry if the compare-and-swap misses.
- Apply the same indexability rules (explicit `isTemporary: false` and not expired, or legacy with no expiry; not excluded by `subagentThread`/`subagentTask`; not `unfinished`) and the same preprocessing (flatten content parts to text, replace `|` with `--`).
- Periodic `sync_with_meili()`: batch documents that need indexing, then page through Meili and delete IDs no longer in Mongo.
- Make `user` filterable, and filter every search with `user = "<id>"`.
- Deletes (`delete_many` on conversations or messages) must also delete from Meili.

**Other items to port**
- Crypto: keep byte compatibility with AES-CBC (static IV, v1), `iv:cipher` hex (v2) and `v3:iv:cipher` AES-256-CTR (v3), using `CREDS_KEY`/`CREDS_IV` hex. Use the `cryptography` library.
- Token hashing: SHA-256 hex.
- AuditLog hash: port `stableStringify` exactly (sorted keys) so existing chains still verify.
- ACL: port `permissionBitSupersets` so queries use `permBits: {$in: [...]}`, or use `$bitsAllSet` where supported. Port `getUserPrincipals` to build the principal list `[user, role, groups..., public]`.
- Transactions: probe support at startup (replica set required). Use `async with await client.start_session() as s: async with s.start_transaction(read_concern=ReadConcern('snapshot'), write_concern=WriteConcern('majority'))` for mcpAuthority and group operations.
- Agent versioning: keep the `versions[]` snapshot semantics, including the duplicate-suppression rule and `actionsHash`. Project `versions` out of normal reads and return `version = len(versions)`.
- Cursor pagination: keep the composite-cursor format (`base64 JSON {primary, secondary, id}`) and the `limit+1` "has more" technique, so frontend cursors stay compatible.
