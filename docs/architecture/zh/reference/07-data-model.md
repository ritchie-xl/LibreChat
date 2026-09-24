# 数据层（MongoDB / packages/data-schemas）

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

除非是绝对路径，下文所有路径均相对于 `packages/data-schemas/src/`。

---

## 1. 数据层的构建方式

### 1.1 包结构
- `schema/*.ts`：纯 Mongoose `Schema` 对象，不绑定模型。`schema/index.ts` 统一重新导出。
- `models/*.ts`：每个模型一个 `createXModel(mongoose)` 工厂。每个工厂：
  1. 调用 `applyTenantIsolation(schema)`，下文列出的五个模型除外；
  2. 可选地挂载 `mongoMeili` 插件；
  3. 返回 `mongoose.models.X || mongoose.model('X', schema)`，因此重复调用是安全的。
- `models/index.ts` → `createModels(mongoose)` 构建 **47 个模型**。它还会给每个模型挂一个 `'index'` 事件监听器。该监听器记录后台索引构建失败的日志，否则这类失败是静默的（例如在 DocumentDB 上 `partialFilterExpression` 被拒绝时）。对于遗留的租户索引冲突（错误码 85/86），它会附加提示，建议运行 `npm run migrate:tenant-indexes`（`migrations/tenantIndexes.ts` 中的 `getTenantIndexMigrationHint`）。
- `methods/*.ts`：每个领域一个 `createXMethods(mongoose, deps?)` 工厂。每个工厂返回普通的 async 函数，这些函数通过 `mongoose.models.X` 惰性获取模型。
- `methods/index.ts` → `createMethods(mongoose, deps)` 将所有领域展开合并为一个扁平对象（`AllMethods`）。它按层级装配（wiring）各领域之间的依赖：
  - `txMethods`（费率表）→ `transactionMethods`（`getMultiplier`、`getCacheMultiplier`）→ `spendTokensMethods`。
  - `messageMethods` 和 `agentQueuedTurnMethods` → `agentTriggerDeliveryMethods`，后者接收 `purgeQueuedTurnsForUser`。
  - `conversationMethods` 接收 `getMessages`、`deleteMessages`、`searchMessages`、触发器投递的擦除钩子，以及一个较大的 `deleteAgentQueuedTurns` 闭包。该闭包会先让投递回执退役，再删除排队轮次的行。
  - `aclEntryMethods` 向提示词、技能和智能体的方法提供 `removeAllPermissions`、`getSoleOwnedResourceIds` 和 `findAccessibleResources`。`userGroupMethods` 提供 `getUserPrincipals`。
  - `agentMethods` 接收 `removeAllPermissions`、`getActions`、`getSoleOwnedResourceIds`、`getUserPrincipals`、`findAccessibleResources` 和 `isExternalSkillId`。
  - `CreateMethodsDeps` 由 API 层注入：`matchModelName`、`findMatchingPattern`、`removeAllPermissions`、`getCache`（`getLogStores` 缓存工厂）和 `isExternalSkillId`。
- 使用方：
  - `api/db/models.js` 运行 `createModels(mongoose)`。
  - `api/models/index.js` 运行 `createMethods(...)` 并定义 `seedDatabase()`，后者调用 `initializeRoles`、`seedDefaultRoles`（AccessRoles）、`ensureDefaultCategories`（AgentCategory）和 `seedSystemGrants`。
  - `api/db/indexSync.js` 为 `Message` 和 `Conversation` 运行 Meili 的 `syncWithMeili()`。
- `index.ts`（包根目录）导出：
  - schema、`createModels`、`createMethods`；
  - 日志器（`config/winston.ts`）和 `meiliLogger`；
  - 租户上下文辅助函数：`tenantStorage`、`getTenantId`、`runAsSystem`、`scopedCacheKey`、`SYSTEM_TENANT_ID`；
  - 迁移、加密，以及 `utils`：保留期辅助函数、`tenantSafeBulkWrite`、`retryWithBackoff`、`buildIndexWithRetry`、`initializeOrgCollections`。

### 1.2 多租户
**上下文传播**位于 `config/tenantContext.ts`。
- `tenantStorage = new AsyncLocalStorage<{tenantId, userId, requestId, requestMethod, requestPath}>()`。
- `SYSTEM_TENANT_ID = '__SYSTEM__'`。`runAsSystem(fn)` 在一个租户为系统哨兵值的上下文中重新运行 `fn`。
- `scopedCacheKey(base)` 在缓存键后追加 `:${tenantId}`。
- 上下文由 `packages/api/src/middleware/tenant.ts` 通过 `tenantStorage.run(buildTenantContext(req), ...)` 设置。租户来自 `req.tenantId ?? req.user.tenantId`，即来自 JWT/用户记录。`preAuthTenant.ts` 在认证之前设置上下文，后台服务（定时任务、触发器、技能同步）则自行设置。

**策略**位于 `tenant/policy.ts`，以纯函数编写，不依赖任何数据库引擎。
- `currentTenantScope()` 返回三种类型之一：`scoped{tenantId}`、`system` 或 `unscoped`。
- 当作用域为 unscoped 且 `TENANT_ISOLATION_STRICT=true` 时，`resolveTenantScope(op)` 抛出 `TenantIsolationError`。严格模式下失败即拒绝（fail closed）。
- `tenantFilter(scope)` 仅对 scoped 类型返回 `{tenantId}`。
- `sanitizeTenantMutation(scope, update, 'guard'|'strip')` 从顶层以及 `$set`、`$setOnInsert`、`$unset` 和 `$rename` 中移除 `tenantId`。
  - `guard` 模式遇到跨租户的值时抛出异常。
  - 系统作用域可以设置 `tenantId`。
- `scopeReplacement` 在替换文档上打上租户标记，并拒绝外部租户的值。
- `tenantWritePredicate` 用于保存已持久化的文档。它添加 `$where.tenantId: {$in:[tenant, null, '']}`，使多租户启用之前创建的行可以被原子地认领。
- `stampTenantOnDocument` 在插入时设置 `tenantId`。严格模式下它会拒绝不匹配的租户。

**Mongoose 绑定**位于 `models/plugins/tenantIsolation.ts`（`applyTenantIsolation`）。
- 在 `find`、`findOne`、`distinct`、`findOneAndUpdate/Delete/Replace`、`updateOne/Many`、`deleteOne/Many`、`countDocuments` 和 `replaceOne` 上的前置钩子会添加 `this.where({tenantId})`。
- 在更新和替换查询上，它还会运行更新守卫或替换守卫。如果更新内容被清空，查询会被改写为 `_id $in []` 并设置 `upsert:false`。
- `aggregate` 会在管道最前面插入 `$match:{tenantId}`。
- `save` 为新文档打上 `tenantId`，并为已有文档的更新添加 `$where` 租户谓词。保存出错时会回滚该标记。
- `insertMany` 为每个文档打上标记。
- 一个 `Symbol` 守卫防止插件被重复应用。

**`bulkWrite`** 不会运行 Mongoose 中间件，因此 `utils/tenantBulkWrite.ts` 提供了 `tenantSafeBulkWrite(model, ops)`。它从更新中剥离 `tenantId`，并向每个过滤条件和插入的文档中注入 `tenantId`。

**有意不使用该插件的模型：**
- `AuditLog`：改为按 `chainKey` 划分作用域。
- `SystemGrant`：跨租户的控制面，使用自己的租户级与平台级条件组成的 `$or`（`methods/systemGrant.ts` 中的 `tenantCondition`）。
- `RefreshTokenBridge`：从未认证的刷新请求中查询，因此由方法显式检查租户。
- `OpenIDRefreshFlight`。
- `SkillSyncCredential` 和 `SkillSyncStatus`：应用级（全局）。

**辅助工具：**
- `tenant/conformance.ts`：可移植的契约测试套件（`TenantEngineHarness`），任何存储引擎绑定都可以运行。
- `tenant/probe.ts`：基于 `monitorCommands` 的驱动层探针。它检查发往数据库的每条命令，是否在过滤条件顶层带有 `{tenantId}`，或以管道 `$match` 的形式带有它。

几乎所有租户作用域的 schema 都声明了 `tenantId: {type:String, index:true}`，唯一索引都与 `tenantId` 组成复合索引。

### 1.3 插件
- **`mongoMeili`**（`models/plugins/mongoMeili.ts`，约 1300 行）。
  - 仅应用于 `Conversation`（索引 `convos`，主键 `conversationId`，`excludeFromIndexPath:'subagentThread'`）和 `Message`（索引 `messages`，主键 `messageId`，`excludeFromIndexPath:'subagentTask'`）。
  - 仅在设置了 `MEILI_HOST` 和 `MEILI_MASTER_KEY` 时生效，搜索还需要 `SEARCH=true`。
  - 标记为 `meiliIndex: true` 的字段会被索引：
    - convo：`conversationId`、`title`、`user`、`tags`；
    - message：`messageId`、`conversationId`、`user`、`sender`、`text`、`content`。
    - `user` 总是会被加入，并被设为可过滤属性。
  - 插件添加隐藏的簿记字段：`_meiliIndex`、`_meiliIndexAttempted`、`_meiliIndexVersion`（ObjectId 字符串，每次保存都会变化）、`_meiliIndexSchemaVersion`（当前为 1）和 `_meiliCleanupVersion`。
  - `pre('save')` 钩子分配新版本。
  - 保存后的同步（`syncVersionedDocument`）写入 Meili，等待任务完成，然后用 `collection.updateOne({_id, _meiliIndexVersion: version}, {$set:{_meiliIndex:true,...}})` 确认。这种基于版本的比较并交换（CAS）处理并发写入，最多进行 3 次对账尝试。
  - 临时的、已过期的、被排除的或 `unfinished` 的文档会改为从 Meili 中删除。
  - 可索引规则：显式存储了 `isTemporary === false` 且未过期，**或者**没有 `isTemporary` 标志且 `expiredAt == null` 的遗留文档。
  - `preprocessObjectForIndex` 通过 `parseTextParts` 将 `content[]` 展平为 `text`，并把 `conversationId` 中的 `|` 替换为 `--`。
  - 静态方法：
    - `syncWithMeili()`：按 `MEILI_SYNC_BATCH_SIZE`（默认 100）和 `MEILI_SYNC_DELAY_MS`（默认 100）分批。它为 `_meiliIndex != true` 的文档以及 schema 版本过旧的文档建立索引，然后运行 `cleanupMeiliIndex`，删除 Mongo 中已不存在的 Meili 文档。
    - `getSyncProgress()`、`setMeiliIndexSettings()`。
    - `meiliSearch(q, params, populate)`：在 Meili 中搜索，然后从 Mongo 重新读取命中结果。
  - 钩子还覆盖 `findOneAndUpdate`、`updateOne`、`deleteOne` 和 `deleteMany`；`deleteMany` 会分批从 Meili 中删除匹配的文档。
  - 它添加部分索引 `meili_excluded_*_cleanup_v3/v4`。两个 schema 还都声明了 `{_meiliIndex:1, isTemporary:1, expiredAt:1}`。
- **`tenantIsolation`**：见 1.2。
- **Schema 级钩子：**
  - `AuditLog` 只允许追加。它的前置钩子拒绝所有更新、删除、`bulkWrite` 和 `insertMany` 操作，以及对非新文档的 `save`。
  - `MCPServer` 在 `pre('validate')` 中计算 `normalizedServerName`。

### 1.4 其他基础设施
- **`utils/transactions.ts`**：`supportsTransactions` 使用 `__transaction_test__` 集合探测事务支持。在 DocumentDB 上它会创建该集合并重试。`getTransactionSupport` 缓存结果，并让多个调用方共享同一个进行中的探测。事务用于 `mcpAuthority`（snapshot 读关注配合 majority 写关注）和 `userGroup`（`runAfterTransaction`）。
- **`utils/retry.ts`**：
  - `retryWithBackoff`；
  - `buildIndexWithRetry` 和 `createIndexesWithRetry`，在索引构建已在进行中时重试；
  - `initializeOrgCollections(models)`，依次创建集合和索引。
  - 有几个方法文件会在首次使用时惰性地、只构建一次自己的索引：Session、RefreshTokenBridge、OpenIDRefreshFlight、AuditLog、Assistant、QueuedTurn、Schedule、TriggerDelivery。
- **`crypto/index.ts`**：
  - `signPayload`（JWT）；
  - `hashToken`（SHA-256 十六进制）；
  - `encrypt`/`decrypt`：使用静态 `CREDS_KEY` 和 `CREDS_IV` 的 AES-CBC；
  - `encryptV2`/`decryptV2`：使用随机 IV 的 AES-CBC，存储为 `iv:cipher`；
  - `encryptV3`/`decryptV3`：`aes-256-ctr`，存储为 `v3:iv:cipher`；
  - `getRandomValues`。
  - 这些函数保护 `Key.value`、`PluginAuth.value`、Action 的 OAuth 密钥、`SkillSyncCredential.encryptedToken`、`RefreshTokenBridge.encryptedNewRefreshToken` 和 `OpenIDRefreshFlight.encryptedResult`。
- **`config/winston.ts`**：Winston 日志器。
  - 按天轮转的 `error-%DATE%.log` 文件，设置 `DEBUG_LOGGING` 时还有调试日志。
  - 控制台输出，可选 JSON 格式（`CONSOLE_JSON`）。
  - `redactFormat`（位于 `parsers.ts`）和请求上下文增强（位于 `requestLogContext.ts`，数据来自 ALS 上下文）。
- **保留期**（`utils/retention.ts`、`utils/tempChatRetention.ts`）：
  - 默认保留期为 720 小时（30 天），限制在 1–8760 小时之间。
  - 来源顺序：先取 `TEMP_CHAT_RETENTION_HOURS` 环境变量，再由 `interfaceConfig.temporaryChatRetention` 覆盖。
  - 当 `retentionMode === 'all'` 时，普通聊天也会过期，使用 `generalChatRetention`。
  - `buildRetentionVisibilityFilter()` 隐藏已过期和临时的行。

---

## 2. 按领域划分的集合

下文使用的约定：
- **T** 表示存在 `tenantId: String`（带索引）且应用了租户插件。
- `ts` 表示 schema 使用 `timestamps: true`（`createdAt` 和 `updatedAt`）。
- **TTL** 表示 TTL 索引。

### 2.1 身份与访问

**User**（`schema/user.ts`，集合 `users`），T，ts。应用的用户账户。
- 资料字段：`name`、`username`（小写）、`email`（必填，小写，正则校验）、`emailVerified`。
- `password`：`select:false`，8–128 个字符。
- `avatar`；`provider`（默认 `local`）；`role`（默认 `USER`，对应 `Role.name`）。
- OAuth ID：`googleId`、`facebookId`、`openidId` + `openidIssuer`、`samlId`、`ldapId`、`githubId`、`discordId`、`appleId`。
- `plugins[]`。
- 双因素认证：`twoFactorEnabled`，以及 `totpSecret`、`backupCodes[{codeHash, used, usedAt}]`、`pendingTotpSecret`、`pendingBackupCodes`（均为 `select:false`）。
- `refreshToken[{refreshToken}]`（遗留）。
- `expiresAt`：**TTL `expires: 604800`**（7 天）。在未验证的新用户上设置；使用 `disableTTL` 时 `createUser` 会移除它。
- `termsAccepted`、`termsAcceptedAt`。
- `agentTriggerDeletionStartedAt` 和 `subagentAdmissionFences[{token, expiresAt}]`（均为隐藏字段）。
- `personalization{memories: Bool, statefulCodeEnvironment: enum}`。
- `favorites[{agentId, model, endpoint, spec}]`。
- `pinnedOrder[String]`（隐藏）。
- `skillStates: Map<String,Boolean>`。
- `idOnTheSource`（sparse）：外部 ID，例如来自 Entra。
- 索引：
  - 唯一 `{email, tenantId}`；
  - `{role, tenantId}`；
  - `{idOnTheSource, openidIssuer, tenantId}`；
  - 每个提供方各一个唯一部分索引 `{<oauthId>, tenantId}`，其中 `openidId` 与 `openidIssuer` 组成复合索引。

**Session**（`sessions`），T。刷新令牌会话。
- `refreshTokenHash`（刷新 JWT 的 SHA-256）。
- `expiration`：**TTL `expires:0`**。
- `user` → User。
- 索引：唯一 `{user, refreshTokenHash}`。
- 默认刷新有效期为 7 天。访问会话有效期 `DEFAULT_SESSION_EXPIRY` 为 15 分钟（定义在 `methods/user.ts` 中）。

**Token**（`tokens`），T。用于邮箱验证、密码重置、邀请和 OAuth 的一次性令牌。
- `userId` → User；`email`（小写，去除首尾空白）；`type`；`identifier`；`token`。
- `createdAt`；`expiresAt`，**TTL 建在 `expiresAt` 上**。
- `metadata: Map<Mixed>`。
- 索引：`{userId, type, identifier, tenantId}`。

**Role**（`roles`），T。带布尔权限矩阵的命名角色。
- `name`；`description`。
- `permissions`：按权限类型嵌套的布尔值。类型包括 BOOKMARKS、PROMPTS、MEMORIES、AGENTS、MULTI_CONVO、TEMPORARY_CHAT、RUN_CODE、WEB_SEARCH、PEOPLE_PICKER、MARKETPLACE、FILE_SEARCH、FILE_CITATIONS、MCP_SERVERS（包括 CONFIGURE_OBO）、REMOTE_AGENTS、SKILLS、SHARED_LINKS 和 SCHEDULES。键包括 USE、CREATE、SHARE、SHARE_PUBLIC、UPDATE、READ、OPT_OUT 和 VIEW_*。
- 索引：唯一 `{name, tenantId}`。

**Group**（`groups`），T，ts。用户组，可以是本地组，也可以从 Entra ID 同步。
- `name`、`description`、`email`、`avatar`。
- `memberIds[String]`：用户 `_id` 字符串，外部用户则为 `idOnTheSource`。
- `source`：枚举 `local|entra`。
- `idOnTheSource`：除非 `source` 为 `local`，否则必填。
- 索引：唯一部分索引 `{idOnTheSource, source, tenantId}`；`{memberIds, tenantId}`。

**AccessRole**（`accessroles`），T，ts。按资源类型划分的命名权限位组合，例如 `agent_viewer`、`agent_editor` 和 `agent_owner`。
- `accessRoleId`（字符串键）、`name`、`description`。
- `resourceType`：枚举 `agent|codeEnvironment|project|file|promptGroup|mcpServer|remoteAgent|skill|sharedLink`。
- `permBits: Number`。
- 索引：唯一 `{accessRoleId, tenantId}`。

**AclEntry**（`aclentries`），T，ts。按资源的访问控制列表（ACL）授权。
- `principalType`：`user|group|public|role`。
- `principalId`：Mixed。user 和 group 为 ObjectId，role 为角色名字符串，public 则不存在。
- `principalModel`：`User|Group|Role`。
- `resourceType`：`agent|codeEnvironment|promptGroup|mcpServer|remoteAgent|skill|sharedLink`。
- `resourceId`：ObjectId，指向资源文档的 `_id`。
- `permBits`：0 到 `MAX_PERM_BITS` = 31 的整数。各位为 VIEW=1、EDIT=2、DELETE=4、SHARE=8、VIEW_INSIGHTS=16。
- `roleId` → AccessRole；`inheritedFrom`（ObjectId，sparse）；`grantedBy` → User；`grantedAt`。
- `expiredAt`：**TTL**。
- 索引：
  - `{principalId, principalType, resourceType, resourceId, tenantId}`；
  - `{resourceId, principalType, principalId, tenantId}`；
  - `{principalId, permBits, resourceType, tenantId}`；
  - `{principalType, resourceType, permBits, resourceId}`，用于公开资源查询。

**SystemGrant**（`systemgrants`），不使用插件，ts。在平台级或租户级生效的管理员能力（capability）授权。
- `principalType`；`principalId`（Mixed）。
- `capability`：由 `admin/capabilities.ts` 的 `isValidCapability` 校验，例如 `read:configs`、`manage:configs:<section>`。
- `tenantId`：平台级授权必须**不存在**该字段，不能为 null 或空字符串。
- `grantedBy`、`grantedAt`、`expiresAt`（预留，尚未强制执行）。
- 索引：唯一 `{principalType, principalId, capability, tenantId}`；`{capability, tenantId}`；`{principalType, capability, tenantId}`。

**AuditLog**（`auditlogs`），不使用插件。只追加、哈希链式的合规日志。
- 所有字段都是 `immutable`。
- `schemaVersion`。
- `category`：`grant|agent_run|tool_call|mcp|config|permission|auth|approval`。
- `action`：目前为 `grant.assigned|grant.removed|permission.insights_assigned|permission.insights_removed`。
- `outcome`：`success|failure|denied|pending`。
- `severity`：`info|warning|critical`。
- `actor{type: user|system|agent|service|schedule|webhook|api, id, name}`。
- `target{type, id, name}`；`metadata`（Mixed）；`context{requestId, ip, userAgent, sessionId}`。
- `tenantId`（平台条目省略）；`chainKey`（即 `tenantId`，或 `__platform__`）。
- `seq`、`prevHash`、`hash`、`createdAt`。
- 索引：**唯一 `{chainKey, seq}`**；`{chainKey, createdAt:-1, seq:-1}`；`{chainKey, category, createdAt:-1}`；`{chainKey, target.type, target.id, createdAt:-1}`。

**Key**（`keys`），T。用户按端点提供的 API 密钥，加密存储。
- `userId` → User；`name`（端点）；`value`（加密）。
- `expiresAt`：**TTL**。

**PluginAuth**（`pluginauths`），T，ts。插件和工具的按用户加密凭据，包括 MCP customUserVars。
- `authField`、`value`（加密）、`userId`（字符串）、`pluginKey`。
- 索引：`{userId, pluginKey, authField, tenantId}`。

**AgentApiKey**（`agentapikeys`），T，ts。远程智能体（兼容 OpenAI）API 的 API 密钥。
- `userId` → User；`name`（最长 100）。
- `keyHash`：`select:false`，带索引。
- `keyPrefix`（带索引）；`lastUsedAt`。
- `expiresAt`：**TTL**。
- 索引：`{userId, name, tenantId}`。

**Balance**（`balances`），T。token 额度余额、自动充值设置和进行中的预留。
- `user` → User。
- `tokenCredits`：1000 额度 = $0.001。
- `autoRefillEnabled`、`refillIntervalValue`、`refillIntervalUnit`（枚举）、`lastRefill`、`refillAmount`。
- 隐藏字段：`reservations[{id, amount, expiresAt}]`、`reservedCredits`、`pendingRefill{transactionId, rawAmount}`。

**Transaction**（`transactions`），T，ts。token 消费与额度流水账。
- `user` → User；`conversationId`（字符串）→ Conversation。
- `tokenType`：`prompt|completion|credits`。
- `model`、`context`、`valueKey`、`rate`、`rawAmount`、`tokenValue`。
- `inputTokens`、`writeTokens`、`readTokens`（提示缓存计费）；`messageId`。

**RefreshTokenBridge**（`refreshtokenbridges`），不使用插件。将已轮换的旧刷新令牌映射到新令牌，用于处理并发刷新竞争。
- `oldRefreshTokenHash`、`encryptedNewRefreshToken`、`userId`（字符串）、`tenantId`、`openidIssuer`、`version`、`createdAt`。
- `expiresAt`：**TTL**。
- 索引：唯一 `{oldRefreshTokenHash, userId, tenantId}`。

**OpenIDRefreshFlight**（`openidrefreshflights`），不使用插件。跨工作进程的内联 OIDC 令牌刷新所用的分布式锁和结果信箱。
- `key`（唯一，哈希后的上下文）、`ownerId`。
- `status`：`pending|completed|failed|revoked`。
- `encryptedResult`、`errorMessage`、`deliveryId`、`deliveryExpiresAt`、`revocationRequestedAt`。
- `lockExpiresAt`（带索引）。
- `expiresAt`：**TTL**。

### 2.2 聊天

**Conversation**（`conversations`），T，ts，Meili。聊天线程头部以及智能体运行时状态。
- `conversationId`：字符串 UUID，业务键。
- `title`（默认 "New Chat"）；`user`（**字符串**形式的用户 ID）。
- `messages[ObjectId → Message]`。
- `isTemporary`。
- 完整的 **`conversationPreset`** 被展开并入（`schema/defaults.ts`）：`endpoint`（必填）、`endpointType`、`model`、`region`、`chatGptLabel`、`examples`、`modelLabel`、`promptPrefix`、`temperature`、`top_p`/`topP`/`topK`、`maxOutputTokens`/`maxTokens`/`max_tokens`、各类惩罚参数、`file_ids`、`resendImages`/`resendFiles`、`promptCache`/`promptCacheTtl`、`thinking`/`thinkingBudget`/`thinkingLevel`、`effort`、`system`、`imageDetail`、`agent_id`、`codeApprovalMode`、`codeEnvironmentMode`、`codeWorkspaces[{environmentId, workspaceId}]`、`assistant_id`、`instructions`、`stop`、**`isArchived`**、`iconURL`、`greeting`、`spec`、`tools`、`maxContextTokens`、`useResponsesApi`、`web_search`、`url_context`、`disableStreaming`、`fileTokenLimit`、`reasoning_*`、`verbosity`。
- `agent_id`；`initial_agent_id`（隐藏、不可变的主智能体，用于 Insights）。
- `subagentThread{rootConversationId, parentConversationId, parentMessageId, parentToolCallId, parentAgentId, subagentType, subagentKind: agent|graph, depth}`：标记子线程，并将其排除在搜索之外。
- `subagentThreadLease{token, taskId, expiresAt}`（隐藏）。
- 隐藏的事件执行者（event-actor）状态：
  - `agentEventBinding{bindingId, sourceKeyId, actorId}`；
  - `agentEventActor{generation, checkpoint{threadId, checkpointId, checkpointNs}, contextFingerprint, skillManifest[], discoveredToolNames[], summary, contextMeta (with fading tiers), compactionSemanticIndex{version, entries[]}, previousCheckpoint, requiresColdStart}`；
  - `agentEventActorCleanup[]`、`agentEventActorReconciliations[]`（status 枚举）、`agentEventActorEpoch`、`agentEventActorLegacyTurn`、`agentEventActorSuspension{..., status enum}`。
- `tags[String]`（Meili），关联到 `ConversationTag.tag`。
- `chatProjectId`（字符串 → `ChatProject._id`）；`files[String]`。
- `expiredAt`：**TTL**，保留期截止时间。
- `pinned`；`archivedAt`。
- 索引：
  - 唯一 `{conversationId, user, tenantId}`；
  - TTL `{expiredAt}`；
  - 多个侧边栏游标索引：`{user, isArchived, updatedAt:-1, _id:-1}`、`{user, isArchived, createdAt:-1, updatedAt:-1, _id:-1}`、`{user, isArchived, title, updatedAt, _id}`、`{user, isArchived, archivedAt:-1, createdAt:-1, _id:-1}`、`{user, pinned, updatedAt:-1, _id:-1}`、`{user, chatProjectId, updatedAt|createdAt:-1, _id:-1}`；
  - Insights：`{tenantId, isTemporary, createdAt:-1, _id:-1}`，以及带 `agent_id` / `initial_agent_id` 的变体；
  - `{user, isTemporary, expiredAt}`、`{user, subagentThread.parentConversationId}`、`{user, subagentThreadLease.expiresAt}`；
  - 唯一 sparse 索引 `agentEventBinding.bindingId`；
  - 对账指标索引；Meili 同步索引。

**Message**（`messages`），T，ts，Meili。单条聊天消息，通过 `parentMessageId` 组成树结构。
- `messageId`（字符串）、`conversationId`（字符串）、`user`（字符串）。
- `model`、`endpoint`、`conversationSignature`、`clientId`、`invocationId`。
- `parentMessageId`：指向另一个 `messageId`；根消息使用 `00000000-...`。
- `tokenCount`、`summaryTokenCount`、`sender`、`text`、`summary`。
- `isCreatedByUser`、`isUserSubmitted`、`userSubmittedPaths[]`、`userSubmittedMessageFieldPaths[{path, field}]`。
- `isTemporary`、`unfinished`、`error`、`finish_reason`。
- `feedback{rating: thumbsUp|thumbsDown, tag, text}`。
- `langfuseSampled`、`langfuseDestinationIds`、`langfuseRunId`。
- `files[Mixed]`；`content[Mixed]`（结构化内容片段）；`thread_id`；`iconURL`；`metadata`。
- 隐藏字段：`subagentTranscript{taskId, mode, messagesJson}`、`subagentActivityProjection`、`subagentTriggerProjection`。
- 隐藏字段 `subagentTask{attemptKey, parentRunId, requestFingerprint, status: running|completed|error|cancelled, resultClaim, controlReceipts[]}`。
- `contextMeta{calibrationRatio, encoding, fading, fadingTiers}`。
- `attachments[Mixed]`、`manualSkills[]`、`alwaysAppliedSkills[]`、`quotes[]`。
- `expiredAt`：**TTL**。
- `addedConvo`。
- 索引：
  - 唯一 `{messageId, user, tenantId}`；
  - `{conversationId, user, createdAt, _id}`，主要的读取路径；
  - `{user, conversationId, createdAt:-1, _id:-1}`，名为 `subagent_thread_latest_message`；
  - 部分索引 `subagent_parent_run_status_updated`；
  - Insights 索引；Meili 同步索引；TTL。

**ConversationTag**（`conversationtags`），T，ts。用户书签/标签。
- `tag`、`user`（字符串）、`description`、`count`、`position`。
- 索引：唯一 `{tag, user, tenantId}`。

**SharedLink**（模型名 `SharedLink`，`schema/share.ts`，集合 `sharedlinks`），T，ts。对话的公开或共享只读快照。
- `conversationId`、`title`、`user`。
- `messages[ObjectId → Message]`。
- `shareId`：随机 nanoid 公开 ID。
- `targetMessageId`。
- `expiredAt`：**TTL**。
- `snapshotFiles`；`fileSnapshots[{file_id, source, storageKey, filepath, type, filename, bytes, width, height, model, llmDeliveryPath, previewRevision, ...}]`。
- 索引：`{conversationId, user, targetMessageId, tenantId}`；`{updatedAt:-1}`。
- 访问还通过 ACL `resourceType: sharedLink` 控制。

**Preset**（`presets`），T，ts。保存的对话设置。
- `presetId`、`title`、`user`、`defaultPreset`、`order`，以及完整的 `conversationPreset`。
- 索引：唯一 `{presetId, tenantId}`。

**ToolCall**（`toolcalls`），T，ts。持久化的工具调用结果，例如代码执行输出。
- `conversationId`、`messageId`、`toolId`、`user` → User。
- `result`（Mixed）、`attachments`（Mixed）、`blockIndex`、`partIndex`。
- `expiredAt`：**TTL**。
- 索引：`{messageId, user, tenantId}`；`{conversationId, user, tenantId}`。

**ChatProject**（集合 `chatprojects`），T，ts。用于归组对话的用户“项目”（文件夹）。
- `name`（最长 100）、`description`、`user`（字符串）。
- 反范式化的统计字段：`conversationCount`、`lastConversationAt`、`lastConversationId`。
- 索引：`{user, name, _id}`、`{user, createdAt:-1, _id:-1}`、`{user, lastConversationAt:-1, _id:-1}`。

**AgentQueuedTurn**（`schema/queuedTurn.ts`，集合 `agentqueuedturns`），T，ts。智能体仍在处理某个对话时，用户发送的轮次所进入的持久队列。
- `user` → User；`conversationId`；`agentId`；`parentMessageId`。
- `clientRequestId`：幂等键。
- `fingerprint`、`laneId`。
- `sequence`：来自 AgentQueuedTurnSequence 的排序号。
- `reservationWriterId`；`activeSlot`（0–99，容量 100）；`admissionSlot`。
- `status`：`reserving|queued|claimed|admitted|cancelled|dead`。
- `priority`、`text`（最大 32 KB）、`files[fileRef]`、`quotes`、`manualSkills`。
- `attempts`、`availableAt`。
- `deliveryKey` → `AgentTriggerDelivery.deliveryKey`。
- `deliveryState`：`pending|publishing|published|retiring|retired`。
- 认领字段：`claimId`、`claimBy`、`claimUntil`。
- 准入字段：`admissionStartedAt`、`admissionId`、`admission*`。
- `reconciliation*` 字段。
- `terminalReceipt{outcome: admitted|cancelled|dead, settledAt, ..., failure}`。
- 索引，均以 `{tenantId, user, conversationId, ...}` 为作用域：
  - `clientRequestId` 上的唯一索引；
  - `sequence`、`activeSlot`、已认领通道（`status:'claimed'`）、已开始准入的通道以及 `admissionSlot` 上的部分唯一索引；
  - 认领顺序索引 `{status, priority:-1, availableAt, sequence}`；
  - 对账索引。

**AgentQueuedTurnSequence**（`queuedTurnSequence.ts`），T，ts。按通道（lane）的序号计数器和单写者租约。
- `_id`：字符串，`laneKey = sha256([tenantId, user, conversationId])`，base64url 编码。
- `user`、`conversationId`、`laneId`、`value`、`reservationId`、`writerId`、`writerUntil`、`retiredAt`。
- `expiresAt`：**TTL**。已退役的通道保留 24 小时。

### 2.3 智能体与工具

**Agent**（`agents`），T，ts。带版本历史的智能体定义。
- `id`：业务 ID 字符串，例如 `agent_xxx`。
- `name`、`description`、`instructions`、`avatar`（Mixed）。
- `provider`、`model`（均必填）；`model_parameters`。
- `artifacts`、`access_level`、`recursion_limit`。
- `tools[]`；`skills[]`（Skill 的 `_id` 字符串）；`skills_enabled`、`skill_authoring_enabled`、`skills_scope`（枚举）。
- `tool_kwargs`。
- `actions[]`：形如 `domain<delimiter>action_id` 的字符串。
- `author` → User；`authorName`。
- `hide_sequential_outputs`、`end_after_tools`、`stateful_code_sessions`。
- `stateful_code_environment`：`user|agent-user|conversation`。
- `code_environment_id` → `CodeEnvironment.environmentId`；`code_workspace_id`。
- `repositoryInstructions`；`git_identity{name, email}`。
- `agent_ids`（已弃用）；`edges[Mixed]`（图的边；`edges.to` 保存智能体 ID）。
- `conversation_starters`。
- `tool_resources`（Mixed：`{file_search:{file_ids}, execute_code:{file_ids}, ...}` → `File.file_id`）。
- **`versions[Mixed]`**：完整快照。
- `category`（→ `AgentCategory.value`）；`support_contact`；`is_promoted`。
- `mcpServerNames[]`（→ `MCPServer.serverName`）；`tool_options`；`subagents`。
- `memory_scope`：`user|agent`。
- 索引：唯一 `{id, tenantId}`；`{mcpServerNames, tenantId}`；`{updatedAt:-1, _id}`；`{tenantId, updatedAt:-1, _id}`；`{edges.to}`。

**AgentCategory**（`agentcategories`），T，ts。市场分类。
- `value`（小写）、`label`、`description`、`order`、`isActive`、`custom`。
- 索引：唯一 `{value, tenantId}`；`{isActive, order}`；`{order, label}`。

**Action**（`actions`），T。挂载到智能体或 Assistant 上的 OpenAPI “actions”。
- `user` → User；`action_id`；`type`（默认 `action_prototype`）；`settings`。
- `agent_id` → `Agent.id`；`assistant_id`。
- `metadata{api_key, auth{authorization_type, custom_auth_header, type: service_http|oauth|none, authorization_url, client_url, scope, token_exchange_method}, domain (required), privacy_policy_url, raw_spec, oauth_client_id, oauth_client_secret}`。

**Assistant**（`assistants`），T，ts。OpenAI/Azure Assistants 的本地元数据。
- `user` → User；`assistant_id`；`endpoint`；`avatar`；`conversation_starters`；`access_level`；`file_ids`；`actions`；`append_current_datetime`。
- 索引：`{tenantId, avatar.filepath}`。

**MCPServer**（`mcpservers`），T，ts。用户或管理员在数据库中创建的 MCP 服务器定义。
- `serverName`；`normalizedServerName`（派生字段）。
- `config`（Mixed：title、url、oauth 等）。
- `author` → User。
- 索引：唯一 `{serverName, tenantId}`；唯一部分索引 `{normalizedServerName, tenantId}`；`{updatedAt:-1, _id}`。
- ACL `resourceType: mcpServer`。

**MemoryEntry**（`memoryentries`），T。用户记忆的键值存储。
- `userId` → User。
- `key`：必须匹配 `^[a-z_]+$`。
- `value`、`agentId`（分区；null 表示共享池）、`tokenCount`、`updated_at`。
- 索引：`{userId, agentId, key}`。

**ToolFavorite**（`schema/favorite.ts`，`toolfavorites`），T，ts。
- `user` → User；`itemType`：`builtin|tool|mcp|skill`；`itemId`。
- 索引：唯一 `{user, itemType, itemId}`。该索引有意不包含 `tenantId`。

**Skill**（`skills`），T，ts。Claude 风格的技能：SKILL.md 正文加 frontmatter。
- `name`：kebab-case，最长 64。保留前缀 `anthropic-`/`claude-` 和保留字会被拒绝。
- `displayTitle`、`description`（最长 1024）、`body`（最大 100 KB）、`frontmatter`（Mixed）。
- `disableModelInvocation`、`userInvocable`、`allowedTools[]`、`category`。
- `author` → User；`authorName`。
- `version`（≥1）。
- `source`：`inline|github|notion`。
- `sourceMetadata{sourceId, upstreamId, path, sha, ...}`。
- `fileCount`、`alwaysApply`。
- 索引：唯一 `{name, author, tenantId}`；`{author, tenantId}`；`{category, updatedAt:-1}`；`{updatedAt:-1, _id}`；`{source, sourceMetadata.upstreamId, tenantId}`；`{source, sourceMetadata.sourceId, tenantId}`。
- ACL `resourceType: skill`。

**SkillFile**（`skillfiles`），T，ts。技能包内的辅助文件。
- `skillId` → Skill。
- `relativePath`：校验规则为不含 `..`、不是绝对路径、且不是 `SKILL.md`。
- `file_id`、`filename`、`filepath`、`storageKey`、`storageRegion`、`source`、`sourceMetadata`、`mimeType`、`bytes`。
- `category`：`script|reference|asset|other`。
- `isExecutable`、`author`、`content`、`isBinary`。
- `codeEnvRef`：子 schema `{kind: skill|agent|user, id, storage_session_id, file_id, version, executionProfile, executionRouteKey, provisionedAt, sandboxFilename}`（`schema/codeEnvRef.ts`）。
- `codeEnvRefs`：以路由为键的 Mixed 映射。
- 索引：唯一 `{skillId, relativePath}`；`{skillId, category}`。

**SkillSyncCredential**（`skillsynccredentials`），不使用插件，ts。用于技能同步的 GitHub 令牌。
- `provider`：`github`；`credentialKey`。
- `encryptedToken` 和 `tokenHash`：均为隐藏字段。
- `createdBy`、`updatedBy`。
- 索引：唯一 `{provider, credentialKey}`。

**SkillSyncStatus**（`skillsyncstatuses`），不使用插件（但有 `tenantId` 字段），ts。每个来源的同步运行状态和锁。
- `provider`、`sourceId`。
- `status`：`idle|running|succeeded|partial|failed|skipped`。
- `credentialKey`、`owner`、`repo`、`ref`、`paths`。
- 时间戳：`startedAt`、`finishedAt`、`lastSuccessAt`、`lastFailureAt`。
- 计数：已同步、已删除和已跳过的技能与文件数；`skippedSkills[]`、`skippedFiles[]`。
- 锁：`lockOwner`、`lockExpiresAt`。
- 索引：唯一 `{provider, sourceId, tenantId}`。

**CodeEnvironment**（`codeenvironments`），T，ts。已注册的代码执行沙箱，分为托管型和挂接型。
- `environmentId`、`name`。
- `type`：`managed|attached`。
- `baseURL`、`controlPlaneId`。
- `createdBy` → User。
- `ownerSlot`：每个所有者的数量上限。`_id` 被设为所有者和槽位的确定性哈希，因此上限由唯一的 `_id` 保证。
- `pendingAgentReferences[{reservationId, expiresAt}]`。
- 用于删除、注册和吊销的租约字段。
- `workerId`、`revocationTokenEnv`。
- `workerPrincipal{type: deployment|tenant|user|role|group, id}`。
- `settings.permissions{fileWrite, commandExecution: allow|ask|deny}`。
- 索引：唯一 `{environmentId, tenantId}`；`{updatedAt:-1, _id}`。
- 还使用原始集合 `code_environment_tombstones`（`methods/codeEnvironment.ts`）。
- ACL `resourceType: codeEnvironment`。

**CodeEnvRef**：仅为嵌入式子 schema，不是集合。

### 2.4 文件

**File**（`files`），T，ts。上传文件和生成文件的元数据；字节内容存放在本地存储、S3、Azure、Firebase 或向量数据库中。
- `user` → User；`conversationId` → Conversation；`messageId`。
- `file_id`：UUID 业务键；`temp_file_id`。
- `bytes`、`filename`、`filepath`、`storageKey`、`storageRegion`、`object`、`embedded`、`type`（MIME）。
- `text`；`textFormat`：`html|text`。
- `status`：`pending|ready|failed`，用于延迟生成的预览；`previewError`；`previewRevision`。
- `context`：FileContext，例如 `execute_code`、`run_artifact`、`agents`、`avatar`。
- `usage`、`source`（FileSources）、`model`、`width`、`height`。
- `metadata{runFile{runId, executionId, agentId, parentExecutionId, ..., inputFileIds}, codeEnvRef, codeEnvRefs, sourceDispatchedAt, embeddedEntities[], destinationChosen, routingMimeType}`。
- `llmDeliveryPath`：`provider|text|none`。
- `expiresAt`：**TTL `expires:3600`**，用于短期上传。
- `expiredAt`：保留期截止时间，由应用层清理任务删除，**不是** TTL 索引。
- `deletionAttempts`、`deletionRetryAt`。
- 索引：
  - `{expiredAt}`；`{createdAt, updatedAt}`；
  - 唯一部分索引 `{filename, conversationId, context, tenantId}`，条件为 `context=execute_code`；
  - 唯一部分索引 `run_artifact_identity`，条件为 `context=run_artifact`。

### 2.5 提示词

**PromptGroup**（`promptgroups`），T，ts。一个命名提示词，带一个“生产”版本和可选的斜杠命令。
- `name`、`numberOfGenerations`、`oneliner`、`category`。
- `productionId` → Prompt。
- `author` → User；`authorName`。
- `command`：`^[a-z0-9-]+$`。
- 索引：`{numberOfGenerations:-1, updatedAt:-1, _id}`。
- ACL `resourceType: promptGroup`。

**Prompt**（`prompts`），T，ts。提示词的一个版本。
- `groupId` → PromptGroup；`author` → User；`prompt`（文本）。
- `type`：`text|chat`。
- 索引：`{createdAt, updatedAt}`。

### 2.6 定时任务与触发器

**Schedule**（`schedules`），T，ts。周期性的智能体提示，类似 cron 作业。
- `id`（字符串）、`user` → User、`name`、`prompt`（最大 32 KB）。
- `agent_id` → `Agent.id`。
- `cadence{frequency: hourly|daily|weekdays|weekly|cron, hour, minute, daysOfWeek[], expression}`。
- `timezone`；`target`：`new`。
- `chatProjectId`、`file_ids`、`tools`、`cron`。
- `enabled`；`disabledReason`（枚举：`mcp_reauth_required`、`too_many_failures`、`agent_deleted`、`insufficient_balance`、`project_deleted`，……）。
- `nextRunAt`。
- 租约：`leaseUntil`、`leaseBy`、`claimToken`（栅栏（fence）令牌）。
- `configRevision`、`deleting`、`deletionSuspension{token, enabled, nextRunAt}`。
- `erased`、`erasedAt`。
- `slot`：每个用户的数量上限。
- `clientRequestId`、`clientRequestDigest`：创建操作的幂等性。
- `eraseAttemptedAt`。
- `lastRun{conversationId, status, error, mcp[], firedAt, scheduledFor}`。
- 计数器：`runCount`、`countedFor[Date]`、`countersAsOf`、`failureCount`、`balanceSkipCount`。
- 索引：
  - **TTL `{erasedAt}` 24 小时，部分索引条件为 `erased:true`**；
  - 唯一 `{id, tenantId}`；
  - `{enabled, nextRunAt}`；
  - 唯一部分索引 `{user, slot}`（非删除中）；
  - 唯一部分索引 `{user, clientRequestId}`；
  - `{deleting, eraseAttemptedAt}`。

**ScheduleRun**（`scheduleruns`），T，ts。定时任务每触发一次对应一行。
- `scheduleId` → `Schedule.id`；`user`；`scheduledFor`；`firedAt`。
- `conversationId`；`checkpointNamespace`（隐藏）。
- `deliveryKey` → AgentTriggerDelivery；`chatProjectId`；`resumeClaimedAt`。
- `status`：`started|requires_action|success|error|interrupted|skipped_overlap|skipped_balance`。
- `error`、`mcp[]`、`droppedFileIds`、`durationMs`、`bookkept`。
- `settledAt`：**TTL 90 天**，仅在运行进入终态时设置。
- `capacitySlot`、`admissionOnly`。
- 中止字段：`abortRequestedAt`、`abortSource`（`stop|deletion`）、`abortPersistedAt`。
- `reconciledAt`、`configRevision`。
- 索引：
  - 唯一 `{scheduleId, scheduledFor}`；
  - 唯一部分索引 `{scheduleId}`，条件为 `status:'started'`：每个定时任务最多一个活动运行；
  - 唯一部分索引 `{capacitySlot}`，条件为 `status:'started'`：全局并发上限；
  - `{scheduleId, firedAt:-1}`、`{status, firedAt}`、`{status, reconciledAt, firedAt}`。

**AgentTriggerDelivery**（`triggerDelivery.ts`，`agenttriggerdeliveries`），T，ts。唤醒智能体的事件（webhook、定时任务、后台工具完成、排队轮次）所用的持久 outbox/队列。
- `deliveryKey`（唯一）、`fingerprint`。
- `orderingKey`：通道 → `AgentTriggerLaneSequence._id`。
- `laneSequence`；`envelope`（Mixed）；`user`。
- `status`：`staging|capability_staging|batched|pending|capability_pending|leased|capability_leased|succeeded|capability_dead|dead`。
- `requiredWorkerCapability`、`capabilityStatus`、`claimAvailableAt`、`capabilityLease*`、`producerLeaseUntil`。
- `backgroundToolResult{status, output, settledAt, resultClaim}`（隐藏）。
- `attempts`、`availableAt`、`envelopeBytes`。
- 合并与批处理：`coalesceKey`、`coalesceFrom`、`coalesceUntil`、`batchSize`、`batchBytes`、`batchMemberIds[→self]`、`batchRootId`。
- `awaitTerminalHandling`。
- `handling{status: started|applied|completed_no_action|failed|cancelled, conversationId, streamId, generationCreatedAt, action{toolName, toolCallId}}`。
- `actorReceipt`、`actorDetachedAction`（带历史记录）、`actorActionAdmitted*`。
- `leaseBy`、`leaseUntil`、`claimToken`、`lastError`、`result`、`history[]`（最多 64 条）、`settledAt`。
- `expiresAt`：**TTL**，仅在成功时设置；成功的行保留 90 天。
- `requeueCount`、`stagingRecoveryAt`、`laneCleanupPendingAt`。
- 索引：唯一 `deliveryKey`；约 16 个认领、通道和 sparse 恢复索引。

**AgentTriggerLaneSequence**（`triggerLaneSequence.ts`），T，ts。按 `orderingKey` 的序号计数器加发布者锁。
- `_id` = `orderingKey`；`value`；`user`。
- `tailDeliveryId` 和 `publisherDeliveryId` → AgentTriggerDelivery。
- `publisherRequeueCount`、`publisherStartedAt`（sparse 索引）、`cleanupRequestedAt`。

**AgentTriggerUserPurge**（`triggerUserPurge.ts`），T，ts。删除某个用户的触发器数据期间使用的栅栏。
- `_id` = User `_id`；`fenceStartedAt`。

### 2.7 配置、UI 与其他

**Config**（`configs`），T，ts。管理员按主体对 `librechat.yaml` 的覆盖配置。
- `principalType`、`principalId`、`principalModel`。基础配置使用主体 `role:__base__`（`admin/capabilities.ts`）。
- `priority`：合并顺序。
- `overrides`（Mixed）、`tombstones[String]`（已删除的路径）、`isActive`、`configVersion`。
- 索引：唯一 `{principalType, principalId, tenantId}`；`{principalType, principalId, isActive, tenantId}`；`{priority, isActive, tenantId}`。

**Banner**（`banners`），T，ts。
- `bannerId`、`message`、`displayFrom`、`displayTo`。
- `type`：`banner|popup`。
- `isPublic`、`persistable`。

**Categories**（`schema/categories.ts`）：schema 存在（label 和 value，均与 `tenantId` 组成唯一索引），但**没有注册模型**。`methods/categories.ts` 返回 9 个硬编码的提示词分类。

**Fading**（`schema/fading.ts`）：不是集合。它是 `contextMeta.fading` / `fadingTiers[{agentId, v, budgetTokens, masked}]` 的共享定义，嵌入在 Message 和 Conversation 中。

**Insights**（`methods/insights.ts`）：不是集合。它是基于 Conversation、Message 和 User 的聚合管道，由 Insights 索引支撑。

**不作为模型管理的原始集合：**
- `mcp_authorization_fence_retries`（`methods/mcpAuthorizationFenceRetry.ts`）：`_id` 是 `[tenant, user, server, version]` 的 JSON。
- `code_environment_tombstones`。
- `__transaction_test__`。

---

## 3. 关系（ER 列表，可直接用于 Mermaid）

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

注意：`user` 在 Conversation、Message、ConversationTag、SharedLink、Preset、ChatProject 和 PluginAuth 中是**字符串**，但在 File、ToolCall、Balance、Transaction、Agent、Action 以及大多数较新的集合中是 **ObjectId**。

---

## 4. 按领域划分的关键数据访问方法

### 身份与访问
- **user**（`methods/user.ts`）：
  - `findUser`、`findUsers`、`countUsers`、`getUserById`、`updateUser`、`deleteUserById`、`acceptTerms`、`toggleUserMemories`、`updateUserStatefulCodeEnvironment`、`updateUserPlugins`。
  - `createUser(data, balanceConfig, disableTTL)` 插入用户，除非指定 `disableTTL`，否则带 7 天的 `expiresAt`。启用余额时，它会以 `$inc: startBalance` 加上自动充值设置 upsert Balance。
  - `generateToken` 签发 JWT（`DEFAULT_SESSION_EXPIRY` 为 15 分钟）。`invalidateAuthUserDocCache` 清除缓存的认证用户文档。
  - `claimSamlIdentity`。
  - 子智能体准入栅栏：`fenceSubagentAdmission`、`renew…`、`release…`、`isSubagentOwnerAdmissible`。
  - 账户删除栅栏：`beginAgentTriggerUserDeletion`、`recover…`、`cancel…`、`isAgentTriggerPrincipalActive`。
- **session**：`createSession` 保存 Session，用 `JWT_REFRESH_SECRET` 签发刷新 JWT `{id, sessionId}`，并存储 `sha256(token)`。另有 `upsertSession`、`findSession`（按哈希令牌或 ID）、`updateExpiration`、`deleteSession`、`deleteAllUserSessions`、`countActiveSessions`。
- **token**：`createToken`、`findToken`（邮箱去除首尾空白并转为小写）、`updateToken`、`deleteTokens`。
- **refreshTokenBridge** 和 **openidRefreshFlight**：
  - `upsert/find/deleteRefreshTokenBridge(s)`。
  - `acquireOpenIDRefreshFlight`（基于 `key` 和 `lockExpiresAt` 的比较并交换（CAS）锁）、`complete`、`renew`、`fail`、`claim/releaseDelivery`、`revoke`。
- **role**：
  - `initializeRoles` 根据 data-provider 的默认值初始化 ADMIN 和 USER。
  - `getRoleByName`（带缓存）、`updateAccessPermissions`（合并权限布尔值，使缓存失效）、`create/deleteRoleByName`。
  - `updateUsersByRole`、`listUsersByRole`、`migrateRoleSchema`。
  - 抛出 `RoleConflictError`。
- **accessRole**：`seedDefaultRoles`；`findRoleByIdentifier`、`findRolesByResourceType`；`getRoleForPermissions` 把权限位映射到角色；`visibleTenantRoles`。
- **aclEntry**（`methods/aclEntry.ts`），授权的核心：
  - `permissionBitSupersets(bit)` 枚举 [0..31] 中所有包含该位的值，用于 `permBits: {$in: [...]}`。这替代了 Cosmos 和 DocumentDB 无法处理的 `$bitsAllSet`。
  - `hasPermission(principals, type, id, bit)` 执行 `findOne({$or: principals, resourceType, resourceId, permBits: $in})`。
  - `getEffectivePermissions` / `…ForResources` 计算匹配条目的按位或。
  - `findAccessibleResources(principals, type, bit, ids?, readPrimary)` 返回 `distinct('resourceId')`。列表端点使用它：其结果成为智能体、技能、提示词和 MCP 服务器的 `accessibleIds`。
  - `findPublicResourceIds`。
  - `grantPermission`（upsert）、`revokePermission`、`modifyPermissionBits`、`replaceRoleBits`、`mutatePermissionEntry`（有限次重试，`PERM_BITS_WRITE_ATTEMPTS`）。
  - `bulkWriteAclEntries`（通过 `tenantSafeBulkWrite`）、`deleteAclEntries`。
  - `getSoleOwnedResourceIds`（用于删除用户时的级联操作）；`aggregateAclEntries`。
- **userGroup**：
  - `getUserPrincipals({userId, role, idOnTheSource})` 返回 `[USER ObjectId, ROLE name, ...GROUP ids (memberIds contains idOnTheSource or userId), PUBLIC]`。组 ID 通过 `getCache` 缓存，并采用双重失效。
  - 组的增删改查：`createGroup`、`upsertGroupByExternalId`、`addUserToGroup`/`removeUserFromGroup`、`syncUserEntraGroups`、`searchPrincipals`（在用户和组之间按相关度评分）、`listGroups`、`bulkUpdateGroups`。
  - `runAfterTransaction` 将副作用推迟到提交之后。
- **systemGrant**：
  - `tenantCondition(tenantId)` = `{$or: [{tenantId}, {tenantId: {$exists: false}}]}`，因此平台级授权对每个租户都生效。
  - `hasCapabilityForPrincipals`；`hasAnyConfigReadAccess`（正则 `^(read|manage):configs(:section)?$`）；`getHeldCapabilities`。
  - `grantCapability`：唯一允许的写入路径；会规范化 `principalId`。
  - `revokeCapability`、`listGrants`、`seedSystemGrants`、`deleteGrantsForPrincipal`。
- **auditLog**：
  - `recordAuditEntry` 读取链上最后的 `{seq, hash}`，计算 `hash = sha256(stableStringify(canonical entry incl. prevHash, seq, createdAt))`，然后插入；若 `{chainKey, seq}` 上发生 E11000 冲突则重试。
  - `listAuditLogPage`（基于 `seq`/`createdAt` 的键集分页）、`streamAuditLogEntries`（导出）、`verifyAuditChain`。
  - `purgeAuditLogEntries`：通过原始 `Model.collection` 执行保留期清理，从而绕过只追加钩子。
- **agentApiKey**：`generateApiKey` 返回随机密钥及其前缀，只存储哈希；`validateAgentApiKey` 按哈希查找并更新 `lastUsedAt`；`list`、`delete`、`deleteAll`。
- **key**、**pluginAuth**：针对加密值的增删改查（`getUserKey`、`getUserKeyValues`、`getUserKeyExpiry`、`updateUserKey`；`findOnePluginAuth`、`findPluginAuthsByKeys`、`updatePluginAuth`）。
- **计费**（`tx.ts`、`transaction.ts`、`spendTokens.ts`）：
  - `tx` 保存静态费率表 `tokenValues`、`cacheTokenValues` 和 `premiumTokenValues`，以及 `getValueKey`、`getMultiplier`、`getCacheMultiplier` 和按上下文长度计算的高价费率。
  - `createTransaction` 计算 `tokenValue = rawAmount × rate`，保存，然后调用 `updateBalance`。
  - `updateBalance` 是乐观并发的比较并交换（CAS）：`findOneAndUpdate({_id, tokenCredits: current}, {$set: max(0, current + inc)})`，带 10 次重试和退避。
  - `createStructuredTransaction` 将输入、写入和读取 token 分开记账。
  - `reserveBalance`、`renewBalanceReservation`、`releaseBalanceReservation` 通过条件更新在 `reservations[]` 和 `reservedCredits` 中预留额度。过期的预留会被清理。
  - `applyAutoRefill` / `settleAutoRefill` 是两阶段充值：先写 `pendingRefill`，再插入交易记录。
  - `spendTokens` / `spendStructuredTokens` 创建 prompt 和 completion 交易记录。
  - 另有 `bulkInsertTransactions`、`getTransactions`、`findBalanceByUser`、`upsertBalanceFields`。

### 聊天
- **conversation**（`methods/conversation.ts`，约 3500 行）：
  - `saveConvo(ctx, convo, meta)`：
    - 按 `{conversationId, user}` upsert；
    - 应用保留期规则（通过 `createChatExpirationDate` 设置 `isTemporary`/`expiredAt`）；
    - 仅在插入时设置 `initial_agent_id`；
    - 默认从 `getMessages` 刷新 `messages[]`，或者追加 `appendMessageIds`；
    - 剥离隐藏的执行者字段；支持 `unsetFields`、`noUpsert` 和 `preserveUpdatedAt`。
  - `bulkSaveConvos`（导入）。
  - `getConvo`、`getConvoTitle`、`getConvoFiles`、`setConvoPinned`。
  - **`getConvosByCursor`**：键集分页。
    - 过滤条件：用户、已归档/未归档、置顶、`tags $in`、项目（`unassigned`）、保留期可见性，以及“人类”对话（没有 `subagentThread`）。
    - 可选地通过 Meili 搜索对话和消息，命中的 ID 作为 `conversationId $in` 应用。
    - 按标题、`createdAt`、`updatedAt` 或 `archivedAt` 排序，带次级键和 `_id` 作为最终排序依据。
    - 游标为 `base64(JSON{primary, secondary, id})`。排序值为 null 的情况有单独的查询子句。
    - 读取 `limit+1` 行，然后执行 `attachSharedFlags`。
  - `getConvosQueried`、`searchConversation`。
  - `deleteConvos(user, filter)` 级联删除消息、子智能体子线程（`subagentThread.rootConversationId`）、排队轮次和触发器投递结果，并更新项目统计。
  - `archiveAllConvos`、`deleteNullOrEmptyConversations`。
  - 子智能体租约：`reserveSubagentThread`、`acquire/renew/releaseSubagentThreadLease`。
  - 事件执行者状态机（对隐藏字段做比较并交换更新）：`commitAgentEventActorState`、`store/claim/settle/cancelAgentEventActorSuspension`、`record/resolveAgentEventActorReconciliation`、`begin/completeAgentEventActorLegacyTurn`。
- **message**：
  - `saveMessage(ctx, params, meta)` 按 `{messageId, user}` upsert，应用相同的保留期规则，清洗 `tokenCount`，并合并用户提交内容的来源路径。
  - `bulkSaveMessages`（通过 `tenantSafeBulkWrite`）、`recordMessage`、`updateMessage`、`updateMessageText`、`updateToolCallResult`。
  - `deleteMessagesSince`（分支/重新生成）。
  - `getMessages(filter, select)`；`CLIENT_MESSAGE_SELECT` 是发送给客户端的投影。
  - `getMessagesByCursor`、`searchMessages`（Meili）。
  - 子智能体任务回执与认领：`claimSubagentTaskResult`、`recordSubagentTaskControlReceipt`、`claim/releaseBackgroundToolResults`。
  - `getMessagesForSubagentThreadView`、`listSubagentTasksForThreads`、`getConversationTraceRefs`（Langfuse）。
- **conversationTag**：`createConversationTag`（通过 `adjustPositions` 调整 `position`）、`updateConversationTag`（同时在对话中重命名该标签）、`deleteConversationTag`（从对话中 `$pull`）、`updateTagsForConversation`、`bulkIncrementTagCounts`。
- **share**：
  - `createSharedLink` 生成 nanoid 形式的 `shareId`，对消息的 `_id` 做快照，可选地对文件做快照，并创建 ACL。
  - `getSharedMessages` 会**匿名化** ID（`anonymizeSharedContent`，为 ID 加前缀并替换为 nanoid，同时把文件 URL 改写为分享路由）。
  - `getSharedLinks`（分页）、`updateSharedLink`（刷新快照）、`deleteSharedLink`、`getSharedLinkFile`、`backfillSharedLinkFiles`。
- **preset**：`getPreset(s)`；`savePreset`（upsert；设置 `defaultPreset` 时，其他预设会取消默认）；`deletePresets`。
- **toolCall**：按消息或对话的增删改查。
- **chatProject**：`create`、`get`、`list`（基于游标）、`update`、`deleteChatProject`（对其下对话 `$unset` `chatProjectId`）、`assignConversationToProject`、`refreshChatProjectStats`（重新计数，带比较并交换重试）。
- **import**：`deleteImportedMessages` / `deleteImportedConversations`。
- **queuedTurn**：
  - `enqueueAgentQueuedTurn`：按 `clientRequestId` 保证幂等；在 Sequence 文档上获取通道写者租约；以 100 的容量上限分配 `sequence` 和 `activeSlot`。
  - `claimNextAgentQueuedTurn`、`beginAgentQueuedTurnAdmission`、`markAgentQueuedTurnAdmitted`、`deadLetterAgentQueuedTurn`、`cancelAgentQueuedTurn`。
  - `reserveAgentQueuedTurnDelivery` 关联到一条触发器投递。
  - `retireConversationLane`、`deleteAllAgentQueuedTurnsForUser`。
  - 抛出 `AgentQueuedTurnConflictError`、`AgentQueuedTurnCapacityError` 和 `AgentQueuedTurnLaneRetiredError`。

### 智能体与工具
- **agent**：
  - `createAgent`：剔除无效的技能 ID；以 `versions: [snapshot]` 初始化；从 `tools` 推导 `mcpServerNames`；预留一个 CodeEnvironment 引用（`withCodeEnvironmentReference`）。
  - **`updateAgent(search, update, {updatingUserId, forceVersion, skipVersioning})`**：
    - 加载当前智能体；
    - 基于 action 元数据计算 `actionsHash`；
    - 推入一个版本快照 `{...current, ...directUpdates, updatedAt, updatedBy, actionsHash}`，除非 `isDuplicateVersion` 发现它与最新版本相同；
    - 修改代码环境时，`findOneAndUpdate` 基于 `updatedAt` 做乐观并发控制；
    - 响应中带有 `version = versions.length`。
  - `revertAgentVersion(search, index)`、`getAgentVersions`、`getAgentWithVersionCount`（使用 `$size` 的聚合）；`getAgent` 默认排除 `versions`。
  - `getListAgentsByAccess({accessibleIds, otherParams, limit, after})`：基于 `updatedAt`/`_id` 的键集游标，最多 1000。
  - `getAgentManagementListByAccess`、`resolveAgentGraphAccess` / `getAgentGraphNodes`（遍历 `edges` 并检查每个节点的 ACL）。
  - `getAgentIdsByMCPServerName`。
  - `add/removeAgentResourceFile(s)`（对 `tool_resources.<tool>.file_ids` 执行 `$addToSet`/`$pull`）、`removeAgentResourceFilesFromAllAgents`。
  - `deleteAgent` 还会移除 ACL、用户收藏和图的边。`deleteUserAgents` 只删除唯一所有者为该用户的智能体。`countPromotedAgents`。
- **agentCategory**：`ensureDefaultCategories`/`seedCategories`、`getActiveCategories`、`getCategoriesWithCounts`（对 Agent 做聚合）、增删改查。
- **action**：`updateAction`（upsert）、`getActions`（可剥离敏感字段）、`deleteAction(s)`。
- **assistant**：`updateAssistantDoc`（upsert）、`getAssistant(s)`、删除。
- **mcpServer**：`createMCPServer`（冲突时 `findNextAvailableServerName` 会添加后缀）、按名称/ObjectId/作者查找、`getListMCPServersByIds`/`ByNames`、更新、删除。
- **mcpAuthority**：构建“权威证明”（authority proof），即当前与 MCP 相关的授权状态（User、Group、Role、Config、MCPServer、Agent、PluginAuth、Token、AclEntry）的摘要。它在一个**使用 snapshot 读关注的事务**中读取这些数据，`assertMCPAuthorityProofsCurrent` 比较证明，以栅栏方式拦截过时的授权。启用此功能前必须先运行 `createMCPAuthorityLookupIndexes`。
- **memory**：`setMemory`（按 `{userId, key, agentId partition}` upsert）、`createMemory`、`deleteMemory`、`setMemoryById`、`getAllUserMemories`、`getFormattedMemories`（带编号的文本块加 token 总数）、`deleteAllUserMemories`。
- **favorite**：`getToolFavorites`、`add`（受 `MAX_TOOL_FAVORITES` 上限约束；通过唯一索引保证幂等）、`remove`。
- **skill**：
  - 校验器：`validateSkillName`、`validateSkillFrontmatter`、`validateSkillDescription`、`validateRelativePath`、`deriveStructuredFrontmatterFields`。
  - `createSkill`、`updateSkill`（递增 `version`）、`getSkillById/ByName`。
  - `listSkillsByAccess`（基于 `updatedAt` 降序 / `_id` 升序的游标，base64 编码；按 `accessibleIds` 或 `manageTenantId` 过滤，另有分类和正则搜索）。
  - `listAlwaysApplySkills`。
  - `deleteSkill`（一系列清理步骤：文件、ACL、从智能体允许列表中移除）、`deleteUserSkills`。
  - `findSkillBySourceIdentity`、`listSkillsBySource`（GitHub 同步）。
  - 文件操作：`upsertSkillFile`、`deleteSkillFile`、`getSkillFileByPath`、`updateSkillFileContent`、`updateSkillFileCodeEnvIds`、`bumpSkillVersionAndAdjustFileCount`。
- **skillSync**：加密凭据的增删改查（`upsertSkillSyncCredential`、`getSkillSyncCredentialToken`）；状态 upsert；`tryAcquireSkillSyncLock` / `refresh` / `release`（基于 `lockOwner` 和 `lockExpiresAt` 的比较并交换）。
- **codeEnvironment**：`createCodeEnvironmentWithinOwnerLimit`（确定性的槽位 `_id`）、注册与移除租约（`begin`/`cancel`/`commitCodeEnvironmentRemoval`、墓碑记录）、`updateCodeEnvironmentSettings`、`deleteUserCodeEnvironments`。

### 文件
- **file**：
  - `createFile`（按 `file_id` upsert）、`updateFile`（可选的 `previewRevision` 守卫）、`findFileById`、`getFiles`。
  - `getToolFilesByIds`、`getCodeGeneratedFiles`、`getUserCodeFiles`。
  - `claimCodeFile` / `commitCodeFile`：在 `execute_code` 上下文中，以 `filename`/`conversationId` 为键、按 `sourceDispatchedAt` 排序的并发安全 upsert。
  - 运行产物：`claimRunArtifactFile`、`publishRunArtifactFile`、`listRunArtifacts`。
  - `updateFileUsage`/`updateFilesUsage`（`$inc`）、`addFileEmbeddedEntity`、`updateFileCodeEnvRef`。
  - 删除：`deleteFile`、`deleteFiles`、`deleteFileByFilter`、`batchUpdateFiles`。
  - 保留期清理：`getExpiredFiles`、`incrementFileDeletionAttempts`、`deferExpiredFile`（退避）。
  - `extendFilesTTL`、`sweepOrphanedPreviews`。

### 提示词
- **prompt**：
  - `createPromptGroup` 创建组及其第一个提示词，然后设置 `productionId`。
  - `savePrompt`、`makePromptProduction`、`updatePromptGroup`、`deletePrompt`（如果被删的是生产版本，则改为指向最新的提示词，或删除整个组）、`deletePromptGroup`（连同 ACL）。
  - `getListPromptGroupsByAccess`（基于游标，按可访问 ID 过滤）、`getPromptGroupsWithPrompts`、`getRandomPromptGroups`、`incrementPromptGroupUsage`。
  - `getPromptGroupAccessContext`（带缓存）、`deleteUserPrompts`。
  - 它还会删除遗留索引 `prompts.groupId_1_version_1`。

### 定时任务与触发器
- **schedule**：
  - `createScheduleWithSlot`：逐个尝试槽位，直到唯一部分索引接受其中一个；按 `clientRequestId` 加摘要保证幂等。
  - `armSchedule`、`updateScheduleById`（轮换 `claimToken`，递增 `configRevision`）。
  - `claimDueSchedule`：原子的 `findOneAndUpdate({enabled, !deleting, nextRunAt <= now, lease free or stale}, {$set: lease + claimToken}, sort by nextRunAt)`。
  - `revalidateClaim`、`advanceSchedule`、`disableSchedule`。
  - `reserveStartedRun` / `insertScheduleRun`：通过唯一部分索引分配容量槽位。
  - `recordRunOutcome`：通过 `countedFor` 和 `countersAsOf` 保证每次触发只计数一次（幂等）。
  - 对账：`getRunsForReconciliation`、`markRunsReconciled`。
  - 中止：`requestRunAbort`、`markRunAbortPersisted`。
  - 删除：`markScheduleDeleting` / `eraseScheduleIfDrained`（留下一个带 TTL 的墓碑记录）、`suspendUserSchedulesForDeletion` / `restoreUserSchedulesFromDeletion`。
  - `ensureScheduleIndexes`。
- **triggerDelivery**（约 4000 行）：持久 outbox，支持有序通道、租约、批处理、能力门控、死信和重新入队。
  - `enqueueAgentTriggerDelivery`：先插入一条 `laneSequence` 为 0 的 staging 行，分配通道序号，然后发布。按 `deliveryKey` 保证幂等。
  - `claimNextAgentTriggerDelivery`：在 8 个候选上做比较并交换，最多 16 次尝试。
  - `beginAgentTriggerDeliveryAttempt`、`completeAgentTriggerDelivery`（将 `expiresAt` 设为当前时间 + 90 天）、`retryAgentTriggerDelivery`、`deadLetterAgentTriggerDelivery`。
  - 死信：`getAgentTriggerDeadLetters`、`requeueAgentTriggerDelivery`。
  - 后台工具结果：`persist`/`get`/`claimAgentBackgroundToolResults`。
  - 执行者动作准入与分离动作（detached action）的生命周期。
  - 用户清理：`prepareAgentTriggerUserPurge` 及相关函数。

### 配置、UI 与其他
- **config**：
  - `getApplicableConfigs(principals)` 查找 `{$or: [base role __base__, ...principals], isActive: true}`，按 `priority` 升序排序；API 层按此顺序深度合并覆盖配置。
  - `upsertConfig`、`patchConfigFields`（对点号路径执行 `$set`，递增 `configVersion`）、`tombstoneConfigField`、`unsetConfigField`、`toggleConfigActive`、`deleteConfig`、`listAllConfigs`、`findConfigByPrincipal`。
- **banner**：`getBanner(user)` 返回类型为 `banner`、满足 `displayFrom <= now <= displayTo|null` 的活动横幅。横幅为公开时返回，或在用户已登录时返回。
- **categories**：静态列表。
- **insights**：`getInsights({tenantId, from, to, agentIds, page, pageSize, search, timeZone})` 运行聚合管道，覆盖时间线、活跃用户与流失用户（30 天窗口）以及按智能体统计的消息数。

---

## 5. 迁移（`migrations/`）

所有迁移都是离线运行或由管理员运行，并接收一个 `mongoose.Connection`。

1. **`tenantIndexes.ts`**
   - `dropSupersededTenantIndexes(conn, {dryRun})` 在 13 个集合上删除已被 `tenantId` 复合版本取代的旧**全局唯一**索引。例如：
     - users：`email_1`、`googleId_1`、`openidId_1`、…
     - roles：`name_1`；agents：`id_1`
     - conversations：`conversationId_1_user_1`；messages：`messageId_1_user_1`
     - presets、agentcategories、accessroles、conversationtags、mcpservers
     - files：`filename_1_conversationId_1_context_1`
     - groups、skillsyncstatuses
   - `migrateTenantIndexes` 分三步运行：先构建新的租户作用域唯一索引，再删除被取代的索引，最后创建所有当前的 schema 索引。运行期间必须停止所有写入方，并禁用 `autoIndex`。
   - `getTenantIndexMigrationHint` 提供 `createModels` 所用的日志提示。
2. **`promptGroupIndexes.ts`**：`dropSupersededPromptGroupIndexes` 删除 `promptgroups.createdAt_1_updatedAt_1`，它已被 `{numberOfGenerations:-1, updatedAt:-1, _id:1}` 取代。
3. **`mcpAuthorityIndexes.ts`**：`createMCPAuthorityLookupIndexes` 创建 `groups{memberIds, tenantId}`、`agents{mcpServerNames, tenantId}`、`pluginauths{userId, pluginKey, authField, tenantId}` 和 `tokens{userId, type, identifier, tenantId}`，带重试。
4. **`mcpServerNames.ts`**：`backfillMCPServerNormalizedNames` 使用主节点读取和 majority 读关注，对 `mcpservers` 做两遍扫描。
   - 它计算 `normalizeServerName(serverName)`，如果同一租户内有两个服务器冲突，则抛出 `MCPServerNameMigrationError`。
   - 然后以每批 500 条批量 `$set` `normalizedServerName`，并创建唯一部分索引。

其他 schema 升级在运行时于方法内部完成：
- `migrateRoleSchema`（role.ts）。
- `quarantineDuplicateAdmissionLanes`（queuedTurn），在构建其唯一索引之前执行。
- 提示词遗留索引的删除。
- Meili `_meiliIndexSchemaVersion` 的版本递增，会强制文档重新建立索引。

---

## 6. Python 重新实现建议

**ODM 选型**
- 使用 **Beanie**（基于 Motor 的 Pydantic v2，或在 PyMongo ≥ 4.9 时基于 PyMongo 的异步 API）。它支持带 `IndexModel` 的 `Settings.indexes`（部分过滤、通过 `expireAfterSeconds` 实现 TTL、唯一、命名、sparse）、事件钩子（`@before_event(Insert, Replace, Save)`）以及投影。
- ODMantic 对索引和钩子的支持较弱。对于热点路径，以及任何依赖比较并交换更新的地方，直接编写原始的 Motor `find_one_and_update` 调用。

**模型映射**
- 将每个 schema 映射为一个 `Document`，并设置 `class Settings: name = "<plural lowercase collection>"`。集合名遵循 Mongoose 的复数化规则：`conversations`、`messages`、`aclentries`、`agentqueuedturns`、`agenttriggerdeliveries`、`agenttriggerlanesequences`、`agentqueuedturnsequences`、`agenttriggeruserpurges`、`openidrefreshflights`、`refreshtokenbridges`、`memoryentries`、`toolfavorites`、`sharedlinks`、`chatprojects`、`skillsynccredentials`、`skillsyncstatuses`、`scheduleruns`、`systemgrants`、`auditlogs`、`configs` 等。
  - 保留 `_id: ObjectId`。
  - 对于字符串 `_id` 的文档（AgentTriggerLaneSequence、AgentQueuedTurnSequence），使用 `id: str = Field(alias="_id")`。
  - AgentTriggerUserPurge 使用用户的 ObjectId 作为 `_id`。
- 用一个基础 mixin 重现 Mongoose 的时间戳（`createdAt`/`updatedAt`），在插入时以及每个更新辅助函数中设置它们（`$set: {updatedAt: now}`、`$setOnInsert: {createdAt: now}`）。
  - AuditLog 自行设置 `createdAt`，且不得有 `updatedAt`。
  - OpenIDRefreshFlight 自行管理这两个字段。
- 将 `select:false` 字段作为默认投影处理：为每个模型构建一个“公开投影”，以及显式的 “+field” 选择加入。涉及的字段包括 `password`、`totpSecret`、`backupCodes`、`keyHash`、隐藏的执行者字段和子智能体字段、`Balance.reservations`、`encryptedToken` 以及 `_meili*` 字段。
- 用 Pydantic 校验器替代 Mongoose 校验器：
  - 邮箱正则；记忆键 `^[a-z_]+$`；技能名称和保留字规则；技能文件 `relativePath` 规则；提示词 `command`。
  - `permBits` 必须是 ≤ 31 的整数。
  - `SystemGrant`/`AuditLog` 的 `tenantId` 绝不能为 null 或空字符串；用 `exclude_none` 省略它。
  - 定时任务 cadence 的条件必填字段。
- 将 `Mixed` 字段保留为 `dict[str, Any]` 或 `Any`：`envelope`、`config`、`overrides`、`tool_resources`、`versions`、`content`、`metadata`。
- 保持每个集合中 `user` 的字符串与 ObjectId 区分，因为现有数据依赖于此。

**租户隔离**
- 用 `contextvars.ContextVar[TenantContext]` 代替 AsyncLocalStorage。在 FastAPI 中间件或依赖项中根据 JWT 用户的 `tenantId` 设置它，并提供作为上下文管理器的 `run_as_system()`，用于设置 `SYSTEM_TENANT_ID`。
- 纯策略函数可以从 `tenant/policy.ts` 一对一翻译：`current_scope`、`resolve_scope(strict)`、`tenant_filter`、`sanitize_mutation(guard|strip)`、`scope_replacement`、`stamp_document`、`write_predicate`。
- Beanie 和 Motor 都没有能覆盖所有路径的 Mongoose 中间件等价物，因此要**包装集合**。一个 `TenantScopedCollection` 代理应拦截 `find*`、`count_documents`、`distinct`、`update*`、`delete*`、`replace_one`、`find_one_and_*`、`aggregate`（在前面插入 `$match`）、`insert_one/many`（打标记）和 `bulk_write`（`tenantSafeBulkWrite` 的规则）。
  - 只在仓储层使用该代理。
  - 为 AuditLog、SystemGrant、RefreshTokenBridge、OpenIDRefreshFlight 和 SkillSync* 保留一个 `raw` 逃生通道。
- 将一致性测试套件（`tenant/conformance.ts`）移植为 pytest 测试套件。
- 探针可以用 PyMongo 的 `CommandListener`（`monitoring.register`）复刻，用于检查每条命令都带有 `tenantId` 谓词。
- 遵循 `TENANT_ISOLATION_STRICT`。保存已有文档时，应按 `_id` 加上 `tenantId in [t, None, ""]` 过滤。

**索引与 TTL**
- 用 `IndexModel(..., expireAfterSeconds=N)` 声明第 2 节列出的每个索引。
  - `User.expiresAt` 上的 `expires: 604800` 对应 604800 秒的 TTL。
  - `File.expiresAt` 对应 3600 秒。
  - Session 的 `expiration`、Token、Key、AgentApiKey、AclEntry 的 `expiredAt`、Conversation、Message、ToolCall 和 SharedLink 的 `expiredAt`、RefreshTokenBridge、OpenIDRefreshFlight、AgentQueuedTurnSequence 和 AgentTriggerDelivery 的 `expiresAt`，都使用 0 秒的 TTL。
  - `ScheduleRun.settledAt` 为 90 天。
  - `Schedule.erasedAt` 为 24 小时，部分索引条件为 `erased: true`。
- 注意 `File.expiredAt` 和 ScheduleRun 的活动行依赖“缺失字段永不过期”的行为。
- 在启动时带重试地构建索引，类似 `createIndexesWithRetry` 和 `initializeOrgCollections`，并记录失败日志。
- 提供一个执行租户索引迁移的 CLI：构建新的唯一索引，删除被取代的索引，再构建其余索引。

**基于唯一性的并发控制**
- 许多不变量依赖唯一部分索引和 E11000 处理：定时任务槽位、运行容量槽位、每个定时任务只有一个已启动的运行、排队轮次的槽位与通道、代码环境所有者槽位、`deliveryKey`，以及 AuditLog 的 `{chainKey, seq}`。
- 用相同的部分索引加上 `DuplicateKeyError` 重试循环来复刻这些机制。
- 对每一个租约和认领的比较并交换使用 `find_one_and_update(..., return_document=AFTER)`：定时任务、触发器投递、排队轮次、技能同步锁、OIDC flight，以及余额 `tokenCredits` 的比较并交换。

**Meilisearch 同步**
- 使用官方的 `meilisearch-python-sdk`（异步）。
- 在保存或 upsert Conversation 或 Message 时，写入 `_meiliIndexVersion = str(ObjectId())` 和 `_meiliIndex = False`。然后运行一个后台任务，在 Meili 中执行 upsert 或删除，等待任务完成，再用 `update_one({_id, _meiliIndexVersion: v}, {$set: {_meiliIndex: True, _meiliIndexSchemaVersion: 1}})` 确认。如果比较并交换未命中，则重新读取并重试。
- 应用相同的可索引规则（显式 `isTemporary: false` 且未过期，或没有过期设置的遗留文档；未被 `subagentThread`/`subagentTask` 排除；不是 `unfinished`）和相同的预处理（将内容片段展平为文本，把 `|` 替换为 `--`）。
- 定期执行 `sync_with_meili()`：对需要建立索引的文档分批处理，然后分页遍历 Meili，删除 Mongo 中已不存在的 ID。
- 将 `user` 设为可过滤属性，每次搜索都用 `user = "<id>"` 过滤。
- 删除操作（对对话或消息的 `delete_many`）也必须从 Meili 中删除。

**其他需要移植的内容**
- 加密：保持与 AES-CBC（静态 IV，v1）、`iv:cipher` 十六进制（v2）和 `v3:iv:cipher` AES-256-CTR（v3）的字节级兼容，使用十六进制的 `CREDS_KEY`/`CREDS_IV`。使用 `cryptography` 库。
- 令牌哈希：SHA-256 十六进制。
- AuditLog 哈希：精确移植 `stableStringify`（键排序），使现有的哈希链仍能校验通过。
- ACL：移植 `permissionBitSupersets`，使查询使用 `permBits: {$in: [...]}`，或在支持的地方使用 `$bitsAllSet`。移植 `getUserPrincipals`，用于构建主体列表 `[user, role, groups..., public]`。
- 事务：启动时探测是否支持（需要副本集）。对 mcpAuthority 和组操作使用 `async with await client.start_session() as s: async with s.start_transaction(read_concern=ReadConcern('snapshot'), write_concern=WriteConcern('majority'))`。
- 智能体版本管理：保留 `versions[]` 快照语义，包括去重规则和 `actionsHash`。在常规读取中投影排除 `versions`，并返回 `version = len(versions)`。
- 游标分页：保留复合游标格式（`base64 JSON {primary, secondary, id}`）和 `limit+1` 的“是否还有更多”技巧，使前端游标保持兼容。
