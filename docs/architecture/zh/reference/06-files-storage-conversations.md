# 文件、存储、RAG、对话、搜索与分享

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。


## 1. 存储策略抽象

**分派器。** `api/server/services/Files/strategies.js` 提供 `getStrategyFunctions(source)`。它返回一个普通对象，其中包含九个可选函数。`null` 表示该后端不支持此操作。

| 函数 | 签名（概念性） |
|---|---|
| `handleFileUpload` | `({req, file(multer: path, originalname, mimetype, size), file_id, basePath?, entity_id?, openai?}) → {filepath, bytes, storageKey?, storageRegion?, embedded?, width?, height?, filename?, id?}` |
| `handleImageUpload` | `({req, file, file_id, endpoint, resolution='high'}) → {filepath, bytes, width, height, storageKey?}`（缩放和格式转换在内部完成） |
| `saveBuffer` | `({userId, buffer, fileName, basePath='images', tenantId?}) → filepath/URL` |
| `saveURL` | `({userId, URL, fileName, basePath, tenantId}) → string or {filepath, storageKey, bytes, type, dimensions}` |
| `getFileURL` | `({userId, fileName, basePath, tenantId}) → URL` |
| `deleteFile` | `(req, fileRecord, openai?) → void` |
| `getDownloadStream` | `(req, pathOrKey) → Readable` |
| `getDownloadURL` | `({req, file, customFilename, contentType}) → signed URL`（仅 S3 和 CloudFront） |
| `prepareImagePayload`、`processAvatar` | 发给 LLM 的图片载荷 / 头像持久化 |

**`FileSources` 枚举**（`packages/data-provider/src/types/files.ts`）：

- 字节存储：`local`、`firebase`、`azure_blob`、`s3`、`cloudfront`。
- 远程提供商存储：`openai`、`azure`（Azure OpenAI Assistants；同样路由到 OpenAI 策略）。
- 伪来源：
  - `vectordb`（RAG）
  - `execute_code`（代码沙箱 API）
  - `mistral_ocr`、`azure_mistral_ocr`、`vertexai_mistral_ocr`
  - `document_parser`（内置的 pdfjs/mammoth/xlsx 解析器，`packages/api/src/files/documents/crud.ts`）
  - `text`（仅提取的文本；使用本地策略；没有需要删除的字节）

**`FileContext` 枚举：** `avatar`、`unknown`、`agents`、`assistants`、`execute_code`、`image_generation`、`assistants_output`、`message_attachment`、`run_artifact`、`skill_file`。此外还有一些排序/过滤用的键。

**策略选择。** `api/server/utils/getFileStrategy.js` 负责选择后端：

- 只设置了 `appConfig.fileStrategy` 时，使用该值（默认 `local`）。
- 使用细粒度的 `fileStrategies: {default, avatar, image, document, skills}` 时，选择取决于标志 `isAvatar`、`isImage` 以及 `context`。
- 一些较旧的路径直接使用 `appConfig.fileStrategy`（例如 `processFileUpload`）。

**实现：**

- **Local**（`api/server/services/Files/Local/{crud,images}.js`）
  - 图片写入 `paths.imageOutput/<userId>/<file_id>__<name>.<imageOutputType>`，并以 `/images/<userId>/...` 的形式作为静态资源提供。
  - 文档写入 `paths.uploads/<userId>/<file_id>__<name>`；`filepath` 为 `/uploads/<userId>/...`，只能通过 API 访问。
  - 删除时检查路径穿越（`isValidPath`），并确认路径属于发起请求的用户。
- **S3**（`packages/api/src/storage/s3/crud.ts`，客户端在 `packages/api/src/cdn/s3.ts`）
  - 环境变量：`AWS_REGION`、`AWS_BUCKET_NAME`、`AWS_ENDPOINT_URL`、`AWS_ACCESS_KEY_ID/SECRET`、`AWS_FORCE_PATH_STYLE`。
  - `getS3Key` 生成的键格式：`[i|a/][r/<region>/][t/<tenantId>/]<basePath>/<userId>/<fileName>`，其中 `fileName = <file_id>__<originalname>`。`basePath` 默认为 `images`；头像使用 `avatars`。
  - `filepath` 保存的是一个 **预签名 GET URL**（有效期 `S3_URL_EXPIRY_SECONDS`，默认 120 秒）。`storageKey` 和 `storageRegion` 一并保存。
  - `GET /api/files` 会运行 `refreshS3FileUrls`，借助缓存限制为每个用户每 30 分钟一次。`needsRefresh` 解析 `X-Amz-Date` 和 `X-Amz-Expires` 来判断是否需要刷新。
  - 删除依次执行 HeadObject、DeleteObject、`deleteRagFile`。在此之前先确认键中的 `userId` 和 `tenantId` 段与记录的所有者一致。
  - 大缓冲区使用分段上传（multipart upload）。
- **CloudFront**（`packages/api/src/storage/cloudfront/crud.ts`、`packages/api/src/cdn/cloudfront*.ts`）
  - 字节存入 S3；URL 为 `domain/<percent-encoded key>`。
  - URL 可选地用 `CLOUDFRONT_KEY_PAIR_ID` / `CLOUDFRONT_PRIVATE_KEY` 签名，使用预设策略（canned policy），URL 带查询字符串时使用自定义策略（custom policy）。
  - 另一种方式是按用户/租户作用域设置签名 cookie（`setCloudFrontCookies`、`maybeRefreshCloudFrontAuthCookiesMiddleware`）。cookie 模式使用内联路径前缀 `i/`（图片）和 `a/`（头像）。
- **Azure Blob**（`api/server/services/Files/Azure/crud.js`、`packages/api/src/cdn/azure.ts`）
  - 认证方式为 `AZURE_STORAGE_CONNECTION_STRING`，或 `AZURE_STORAGE_ACCOUNT_NAME` 加 `DefaultAzureCredential`。
  - 容器为 `AZURE_CONTAINER_NAME`（默认 `files`）；blob 路径为 `<basePath>/<userId>/<fileName>`。
  - `AZURE_STORAGE_PUBLIC_ACCESS=true` 设置 blob 的访问级别。`filepath` 为 blob URL。
  - 删除时检查 blob 路径中包含 `req.user.id`。
- **Firebase**（`api/server/services/Files/Firebase/*`）：使用 `FIREBASE_*` 环境变量，路径形如 `images/<userId>/<name>`。
- **OpenAI**（`Files/OpenAI/crud.js`）：`openai.files.create/del/content`。`filepath = <baseURL>/files/<id>`。删除操作经过一个漏桶队列（`LB_QueueAsyncCall`）串行执行。
- **Code**（`Files/Code/crud.js`）
  - 调用外部代码 API（来自 `@librechat/agents` 的 `getCodeBaseURL()`）：`POST /upload`、`POST /upload/batch`、`GET /download/<session>/<fileId>?kind=&id=`、`DELETE /files/<session_id>/<fileId>`。
  - 引用保存在 `metadata.codeEnvRef(s) = {kind: user|agent, id, storage_session_id, file_id, executionProfile}` 中。
- **VectorDB**：见第 3 节。
- **OCR**（`packages/api/src/files/mistral/crud.ts`）
  1. `POST {baseURL}/files`（multipart，purpose 为 ocr）。
  2. `GET /files/{id}/url?expiry=`。
  3. `POST /ocr`，请求体 `{model, image_limit:0, include_image_base64:false, document:{type:'document_url'|'image_url', ...}}`。
  4. 把各页的 markdown 拼接为文本。
  5. `DELETE /files/{id}`。
  - 凭据来自 `appConfig.ocr.{apiKey, baseURL, model, strategy}`，支持环境变量插值。
  - Azure 和 Vertex 变体采用相同结构。

**图片**（`api/server/services/Files/images/resize.js`、`convert.js`，基于 sharp）：

- `resizeImageBuffer(buf, 'low'|'high'|{percentage|px}, endpoint)`：
  - `low` 缩放到 512×512 以内。
  - `high` 将短边上限设为 768、长边上限设为 2000（Anthropic 为 1568），并用 `rotate()` 处理 EXIF 方向。
- `resizeAndConvert` 生成 150px 宽的缩略图。
- 输出格式为 `appConfig.imageOutputType`（默认 `png`）。
- 头像经过 `images/avatar.js` 的 `resizeAvatar`，然后是 `strategy.processAvatar`，后者同时更新 `user.avatar`。
- `process.js` 中的 `saveBase64Image` 处理图片生成产出的 data-URL 图片：解码、缩放、根据重新编码后的字节记录真实 MIME 类型、`saveBuffer`、`createFile`。

## 2. 文件上传与处理流程

**路由器**（`api/server/routes/files/index.js`），挂载在 `/api/files`：

- 中间件：`requireJwtAuth`、`configMiddleware`、`checkBan`、`uaParser`。
- Multer 磁盘存储（`multer.js`）写入 `uploads/temp/<userId>/<sanitized name>`，并设置 `req.file_id = randomUUID()`。文件名先做 URI 解码；编码错误返回 400。
- Multer 的 `fileFilter` 规范化 MIME 类型（`inferMimeType`），并与端点的 `supportedMimeTypes` 比对。对 `agents`，比对的是所有端点的并集。拒绝时返回 415。
- Multer 大小上限为 `serverFileSizeLimit`（默认 512MB）。
- 除 `/usage` 和 `/speech` 外，所有 POST 都经过按 IP 和按用户的上传限流器。

**路由：**

- `GET /` — 用户的文件；默认排除 `text`。
- `GET /agent/:agent_id` — 需要是智能体作者或拥有 EDIT 权限。
- `GET /config`。
- `POST /usage` — 为排队中的上传延长上传 TTL。
- `DELETE /` — 请求体 `{files:[{file_id, filepath}], agent_id?, assistant_id?, tool_resource?}`。
- `GET /code/download/:session_id/:fileId`。
- `GET /:file_id/preview` — 轮询延迟预览状态 `pending|ready|failed`。停留在 `pending` 超过 2 分钟的记录会被惰性标记为 `failed`。
- `GET /download-url/:userId/:file_id` — 返回签名 URL 的 JSON。
- `GET /download/:userId/:file_id` — 流式返回文件。响应头：`Content-Disposition`、`application/octet-stream`，以及 `X-File-Metadata`（允许列表字段集合的 URI 编码 JSON）。`?direct=true` 时返回指向签名 URL 的 302。`text` 来源的文件以 `.txt` 形式返回其存储的文本。
- `POST /` — 通用上传。
- `POST /images` — 图片上传。
- `POST /images/avatar`、`POST /images/agents/:id/avatar`、`POST /images/assistants/:id/avatar`。
- 语音相关路由。

**上传响应：**

- 默认为 JSON `{message, ...fileRecord}`。
- 如果 `Accept` 请求头包含 `text/event-stream`，响应为 SSE（`packages/api/src/files/sse.ts`）：`event:data`、`event:error {message, code, temp_file_id, tool_resource}`，然后是 `event:close`，中间穿插心跳。
- 客户端在表单中发送 `file_id`。它会成为 `temp_file_id`（用于客户端关联），服务端使用自己的 `req.file_id`。
- 临时文件总是在 `finally` 中删除。

**公共前置步骤**，位于 `files.js` 的 `handleFileUpload` 和 `images.js`：

1. `verifyAgentUploadPermission` — 智能体 EDIT 权限，除非是消息附件，或用户拥有 `MANAGE_AGENTS` 能力。
2. `resolveUploadEndpoint` — 解析智能体的提供商。
3. `filterFile`（`process.js`）：`file_id` 必须是 UUID，文件不能为空，按端点的 `fileSizeLimit` 检查大小，检查 `disabled` 标志和 MIME 类型。图片还要求请求体中带有 `width` 和 `height`。
4. `resolveEffectiveToolResource`。
5. `checkToolResourceUploadPermission` — 角色权限。
6. `assertUploadContentAllowed` — 内容过滤器与 PII 策略。
7. 分派：assistants 走 `processFileUpload`；其他全部走 `processAgentFileUpload` 或 `processImageFile`。

**(a) 图片发送到聊天**（`POST /api/files/images`，消息附件，无 tool_resource）：

1. `processImageFile`：`source = getFileStrategy({isImage:true})`。
2. `handleImageUpload` 缩放（`high`），转换为 `imageOutputType`，并存储字节。
3. `resolveUploadLLMDeliveryPath` 把 `llmDeliveryPath` 设为 `provider`、`text` 或 `none`。
4. `createFile({user, file_id, temp_file_id, bytes, filepath, storageKey, filename, context:'message_attachment', source, type:'image/png', width, height, tenantId, llmDeliveryPath, metadata:{destinationChosen, routingMimeType?}, expiredAt?}, disableTTL=true)`。
5. 发送时，智能体管线从存储中读取文件，并为提供商做 base64 编码（`packages/api/src/files/encode/*`）。`updateFilesUsage` 递增 `usage` 并清除 `expiresAt`。

**(b) 用于 file_search / RAG 的文档**（`POST /api/files`，带 `tool_resource=file_search`、`agent_id`）：

1. `processAgentFileUpload` → `resolveUploadDestination` 检查能力：`checkCapability(file_search)` 和角色授权 `fileSearch`。
2. “双重存储”第 1 步：`handleFileUpload` 把字节存入配置的存储。
3. 双重存储第 2 步：`uploadVectors` → `POST {RAG_API_URL}/embed`，multipart 字段为 `file_id`、`file`、`entity_id`（智能体知识文件为智能体 id；用户附件省略）。
4. `metadata.embeddedEntities = [agent_id]`；`embedded = known_type`。
5. `db.addAgentResourceFile({agent_id, tool_resource, file_id})`。
6. `createFile(...)`，`context: agents`（或 `message_attachment`）。
7. 未嵌入的消息附件会在轮次进行时通过 `provisionToVectorDB`（`packages/api/src/files/provision/service.ts`）惰性嵌入，它从存储流式读取并调用 `uploadVectors`。

**(c) 智能体的 tool_resources：**

- **`execute_code`**
  - 需要 `checkCapability(execute_code)` 和 `roleGrants.runCode`。
  - 先存储字节。然后，仅当用户显式选择了这个目标时，`withCodeApiUploadRecovery` 从存储流式读取并发送到 `codeapi POST /upload`，`kind` = `agent`（附件则为 `user`）。这一步有限流和重试。
  - 结果：设置 `metadata.codeEnvRef`。如果上传失败，已存储的字节会被删除。
  - 由系统推断而非用户选择的目标，会在每个轮次惰性配置（`provisionToCodeEnv`）。
- **`context`** — 文本提取；见 (d)。记录上会保存一个 `text` 字段，并设 `llmDeliveryPath:'text'`。
- **`image_edit` / `ocr`** — 同样是有效的资源键（`AGENT_TOOL_RESOURCE_KEYS`）。
- 智能体的 `tool_resources.<key>.file_ids` 数组把智能体与它的文件关联起来。

**(d) 文本提取 / OCR**（context 资源），顺序由 `getUploadExtractedTextPlan` 决定：

1. **configuredOCR**（设置了 `appConfig.ocr` 且类型支持 OCR 时）：`getStrategyFunctions(ocr.strategy ?? document_parser).handleFileUpload`。出错时回退到 `document_parser`。需要智能体的 `ocr` 能力。
2. **documentParser**：pdfjs、处理 docx 的 mammoth、xlsx/ods、odt。
3. **STT**，用于 `fileConfig.stt.supportedMimeTypes` 中的音频类型：`STTService`，然后得到转写文本。
4. **configuredRAG**：`parseText`（`packages/api/src/files/text.ts`）调用 `GET {RAG}/health`（10 秒超时），然后以 multipart `file_id`、`file` 调用 `POST {RAG}/text`（5 分钟超时），返回 `{text}`。失败时回退到文档解析器。Markdown 总是跳过 RAG，直接原生读取。
5. **原生 UTF-8 读取**：仅用于可原生读取的文本类型。

对所有路径：

- 提取的文本上限为 `MAX_STORED_EXTRACTED_TEXT_BYTES`，并经过内容过滤器检查。
- 字节仍会通过 `handleFileUpload` 存储。

**Assistants**（`processFileUpload`）：使用 OpenAI 或 Azure 策略。它还会调用 `assistants.files.create` 或 `addResourceFileId`。对图片，由一份本地转换后的副本提供 `filepath`。

## 3. RAG 集成契约（外部 `rag_api`）

**部署。** `rag.yml` 在 `RAG_PORT` 上运行 `pgvector/pgvector:0.8.0-pg15` 和镜像 `registry.librechat.ai/danny-avila/librechat-rag-api-dev:latest`。LibreChat 只需要 `RAG_API_URL`。

**认证。** 每次调用都发送 `Authorization: Bearer <JWT>`。该令牌是对 `{id: userId}` 的 HS256 签名，用 `JWT_SECRET` 签发，5 分钟后过期（`generateShortLivedToken`，`packages/api/src/crypto/jwt.ts`）。RAG API 必须共享同一个 `JWT_SECRET`。

**调用的端点：**

| 方法/路径 | 调用方 | 请求 | 使用的响应 |
|---|---|---|---|
| `GET /health` | 启动检查 `packages/api/src/app/checks.ts`；`parseText` | – | 200 |
| `POST /embed` | `Files/VectorDB/crud.js` `uploadVectors` | multipart：`file_id`、`file`、可选 `entity_id`、可选 `storage_metadata`（JSON） | `{status: bool, known_type: bool}` → `embedded = known_type`；`known_type === false` 或 `!status` 时报错 |
| `POST /text` | `packages/api/src/files/text.ts` | multipart `file_id`、`file` | `{text}` |
| `POST /query` | `api/app/clients/tools/util/fileSearch.js`（工具）；`api/app/clients/prompts/createContextHandlers.js`（旧版） | JSON `{file_id, query, k:5, entity_id?}`（工具）/ `k:4`（旧版） | `[[{page_content, metadata:{source, page}}, distance], ...]` |
| `GET /documents/{file_id}/context` | 设置 `RAG_USE_FULL_CONTEXT` 时的旧版上下文处理器 | – | 全文 |
| `DELETE /documents` | `deleteVectors`、`packages/api/src/files/rag.ts` `deleteRagFile` | JSON 请求体 `[file_id]` | 404 视为已删除 |

**`file_search` 工具**（`fileSearch.js`）：

1. `primeFiles` 收集 `tool_resources.file_search.file_ids` 以及本轮附加的文件，经 `filterFilesByAgentAccess` 过滤。它构建一段列出文件名的工具上下文说明。
2. 工具对每个文件并行发起一次 `/query` 调用。
   - 只有智能体知识文件（`fromAgent`）才发送 `entity_id`。用户附件嵌入在用户命名空间下。
3. 合并结果，按距离升序排序，保留前 10 条。
4. 来源为 `{type:'file', fileId, content, fileName, relevance: 1-distance, pages, pageRelevance}`。
5. 返回的字符串对每条结果列出 File / Anchor / Relevance / Content。锚点使用标记 `turn0file<i>`。
6. artifact 为 `{file_search: {sources, fileCitations}}`（工具 responseFormat 为 `content_and_artifact`）。

**引用。** `processFileCitations`（`Files/Citations/index.js`）：

1. 检查 `FILE_CITATIONS` 角色权限。
2. `selectFileCitationSources` 应用智能体端点配置中的 `maxCitations`、`maxCitationsPerFile` 和 `minRelevanceScore`。
3. 用 File 集合中的信息（filename、storageType、fileType、bytes）补充每个来源。
4. 发出一个消息附件 `{type:'file_search', file_search:{sources}, toolCallId, messageId, conversationId}`。

## 4. 文件生命周期

**File schema**（`packages/data-schemas/src/schema/file.ts`，带时间戳）：

- 身份与归属：`user`（ObjectId）、`conversationId`、`messageId`、`file_id`（UUID，有索引）、`temp_file_id`、`tenantId`。
- 大小与命名：`bytes`、`filename`、`filepath`、`storageKey`、`storageRegion`、`object='file'`、`type`（MIME）。
- 标志与内容：`embedded`、`text`、`textFormat`（`html|text`）、`context`、`usage`（默认 0）、`source`（默认 local）、`model`、`width`、`height`。
- 延迟预览：`status`（`pending|ready|failed`）、`previewError`、`previewRevision`。
- `metadata`：`runFile` 来源信息、`codeEnvRef(s)`、`sourceDispatchedAt`、`embeddedEntities[]`、`destinationChosen`、`routingMimeType`。
- `llmDeliveryPath`（`provider|text|none`）。
- 生命周期：`expiresAt`、`expiredAt`、`deletionAttempts`、`deletionRetryAt`。
- 索引：`expiredAt`；在 `context=execute_code` 时唯一的 `(filename, conversationId, context, tenantId)`；唯一的 `run_artifact_identity`。

**两种 TTL 机制：**

1. **`expiresAt`** 是 3600 秒的 Mongo TTL（上传窗口）。`createFile(data, disableTTL)` 会把它设为当前时间 +1 小时，除非传入 `disableTTL`；大多数上传路径都传 `true`。
   - `updateFileUsage` / `updateFilesUsage`（在发送消息时）执行 `$inc usage` 以及 `$unset expiresAt, temp_file_id`。更新操作也会清除它。
   - `POST /files/usage` → `extendFilesTTL` 为排队中的上传推后截止时间（带 `$lt` 保护）。
   - 注意：Mongo TTL 只删除元数据行。
2. **`expiredAt`** 是保留期截止时间（`api/server/services/Files/retention.js`、`packages/api/src/files/retention.ts`）。
   - 它来自对话的 `expiredAt`，或来自 `interfaceConfig.retentionMode`（`all` / `temporary`）结合 `isTemporary`。小时数取自 `temporaryChatRetention` / `generalChatRetention` / `TEMP_CHAT_RETENTION_HOURS`（默认 720 小时，限制在 1–8760 之间）。
   - 持久的智能体资源上传不受此限，除非 `retentionMode=all` 且未设置 `retainAgentFiles`。
   - 应用层清扫（`packages/api/src/files/sweep.ts`，通过 `startExpiredFileSweep` 启动）只在领导者上运行，每隔 `FILE_RETENTION_SWEEP_INTERVAL_MS`（默认 1 小时；0 表示禁用）执行一次。
   - 每一轮：`getExpiredFiles(100)` 取 `expiredAt<=now` 且（`deletionRetryAt` 为 null 或 `<=now`）的记录，然后调用 `processDeleteRequest`。
   - 失败时：`incrementFileDeletionAttempts`，并以指数退避调用 `deferExpiredFile`（基数为清扫间隔，最小 60 秒，上限 24 小时）。达到 `FILE_RETENTION_SWEEP_MAX_ATTEMPTS`（10）次后，该文件被搁置 30 天。

**删除**（`processDeleteRequest`，`process.js`）：

1. 对每个文件，调用主策略的 `deleteFile`。
2. 同时删除次要副本：若 `embedded` 则删除向量，若设置了 `codeEnvRef` 则删除代码环境中的副本。
3. 把“不存在”类错误（404、`ENOENT`、`NoSuchKey` 等）视为成功。
4. `db.deleteFiles(ids)` 只针对存储已删除的 ID 执行，然后调用 `removeAgentResourceFilesFromAllAgents`。
5. 返回 `{deletedFileIds, failedFileIds}`。

DELETE 路由要求所有权。对智能体资源，它检查智能体作者身份或 EDIT 权限；被其他资源共享的文件只会解除关联（`packages/api/src/files/deletion.ts` 中的 `deleteAgentResourceFiles`）。

**权限：**

- `api/server/middleware/accessResources/fileAccess.js` 在租户匹配且满足以下任一条件时允许访问：
  - 用户拥有该文件，或
  - 用户对某个智能体拥有 VIEW 权限（或是其作者），且该智能体的任一 `tool_resources.*.file_ids` 中包含此 `file_id`。
- `Files/permissions.js`：`hasAccessToFilesViaAgent` 和 `filterFilesByAgentAccess`（删除需要 EDIT；包含远程智能体 ACL 变体）。
- ACL 服务位于 `packages/api/src/acl`。

## 5. 对话与消息

**Conversation schema**（`packages/data-schemas/src/schema/convo.ts`，带时间戳）：

- 核心字段：`conversationId`（UUID）、`title`（默认 `New Chat`）、`user`（字符串 id）、`messages[ObjectId]`、`isTemporary`。
- `schema/defaults.ts` 中的全部预设字段：endpoint、endpointType、model、temperature、promptPrefix、agent_id、assistant_id、tools、file_ids、`isArchived`、iconURL、spec 等。
- `tags[]`、`chatProjectId`、`files[]`、`expiredAt`、`tenantId`、`pinned`、`archivedAt`。
- 子智能体与事件执行者（event-actor）的内部字段（`subagentThread`、租约等）属于智能体部分。
- 索引：`expiredAt` 上的 TTL（`expireAfterSeconds:0`）；唯一的 `(conversationId, user, tenantId)`。

**Message schema**（`schema/message.ts`）：

- 身份：`messageId`、`conversationId`、`user`、`parentMessageId`（根消息使用 `Constants.NO_PARENT` = `00000000-0000-0000-0000-000000000000`）。
- 模型信息：`model`、`endpoint`、`sender`。
- 内容：`text`、`content[]`（带类型的片段）、`isCreatedByUser`、`tokenCount`、`summary`、`unfinished`、`error`、`finish_reason`。
- `feedback{rating: thumbsUp|thumbsDown, tag, text}`。
- `files[]`、`attachments[]`、`metadata`、`contextMeta`（服务端私有）、`quotes`、`expiredAt`、`tenantId`。
- Langfuse 链路追踪引用。
- 索引：`expiredAt` 上的 TTL；唯一的 `(messageId, user, tenantId)`；`(conversationId, user, createdAt, _id)`。

**分支。** 消息通过 `parentMessageId` 构成一棵树。兄弟节点是重新生成或编辑的结果。客户端选择一条路径；`BaseClient.getMessagesForConversation` 从叶子走到根。

**路由，`/api/convos`**（`api/server/routes/convos.js`）：

- `GET /` → `getConvosByCursor`（`methods/conversation.ts`）。
  - 查询参数：`limit`、`cursor`、`isArchived`、`pinned`、`tags[]`、`search`、`sortBy`（`title|createdAt|updatedAt|archivedAt`）、`sortDirection`、`projectId`（ObjectId 或 `unassigned`）。
  - 过滤条件：`user`、归档状态、保留期可见性，以及“不是子智能体线程”。
  - `search` 在 Meili 中查询按 `user = "<id>"` 过滤的对话和消息，然后按 `conversationId $in` 过滤。
  - 游标是 base64 编码的 JSON `{primary, secondary, id}`。它按（排序字段，`updatedAt` 或 `createdAt`，`_id`）精确分页，并处理 null 分组。查询取 `limit+1` 行。
  - 返回 `{conversations, nextCursor}`，每条都附带 `isShared`。
- `GET /:conversationId`。
- `GET /gen_title/:id` — 以 0.5–8 秒的退避轮询缓存。
- `POST /update {arg:{conversationId, title}}` — 截断到 1024 个字符，并经过内容过滤。
- `POST /archive {arg:{conversationId, isArchived}}` — 保留 `updatedAt`；设置 `archivedAt`。
- `POST /archive/all`。
- `POST /pin`。
- `DELETE / {arg:{conversationId}}` — 级联删除：
  1. 排空正在运行的生成任务，并取消子智能体。
  2. `deleteConvos` 按波次删除子智能体的子线程，递减标签计数，并刷新项目统计。
  3. 删除消息、工具调用和分享链接，并清理 ACL。
  4. 裁剪检查点。
- `DELETE /all`。
- `POST /import` — multer，仅支持 JSON，大小取自 `resolveImportMaxFileSize`。
- `POST /fork {conversationId, messageId, option, splitAtTarget, latestMessageId}`。
- `POST /duplicate {conversationId, title}`。
- 子智能体线程相关端点（属于其他部分）。

**路由，`/api/messages`**（`api/server/routes/messages.js`）：

- `GET /?conversationId&cursor&pageSize&sortBy&sortDirection` → `getMessagesByCursor`。游标是原始的 `createdAt` 值；取 `limit+1` 行。
- `GET /?search=` → Meili 消息搜索，从 Mongo 补全数据，并通过 `getConvosQueried` 关联对话标题。
- `GET /:conversationId` 返回按 `createdAt` 排序的全部消息，使用 `CLIENT_MESSAGE_SELECT` 投影，该投影会剔除 `contextMeta`。
- `POST /:conversationId` → `saveMessage`，然后以 `appendMessageIds` 调用 `saveConvo`。
- `PUT /:conversationId/:messageId {text, index?, model}` 编辑文本或某个内容片段，并重新计算 token。
- `PUT .../feedback` — 同时向 Langfuse 发送评分。
- `DELETE /:conversationId/:messageId`。
- `POST /branch {messageId, agentId}` — 把并行的智能体内容拆分为一个新的兄弟消息。
- `POST /artifact/:messageId`。

**`saveConvo` / `saveMessage`：**

- `saveConvo` 按 `(conversationId, user)` upsert。`messages` 由消息的 `_id` 重新计算（或追加）。
- `saveMessage` 拒绝非 UUID 的 `conversationId`，按 `(messageId, user)` upsert。
- 两者都根据 `retentionMode`、`isTemporary` 和 `interfaceConfig` 应用保留期 `expiredAt`（`createChatExpirationDate`）。
- `saveConvo` 校验 `chatProjectId` 归该用户所有。
- `deleteMessagesSince` 删除 `createdAt` 晚于锚点消息的消息。

**Fork**（`api/server/utils/import/fork.js`）。选项：

- `DIRECT_PATH` — 只包含祖先。
- `INCLUDE_BRANCHES` — 路径加上每一层的兄弟节点。
- `TARGET_LEVEL`（默认）— 在所有分支上做广度优先遍历，直到目标所在深度。
- `splitAtTarget` — 从目标所在层级开始新对话。

实现：

1. `cloneLineage`（`packages/api/src/conversations/lineage.ts`）把每个 `messageId` 重新映射为新的 UUIDv4，并改写 `parentMessageId`，保持时间戳有序。
2. `ImportBatchBuilder`（`importBatchBuilder.js`）创建新的 `conversationId`，并运行 `bulkSaveConvos` 和 `bulkSaveMessages`。
3. 写入经过一个 16MB 减 64KB 的 BSON 大小保护（`packages/api/src/conversations/import.ts`）。
4. `forkSharedConversation` 从分享链接派生（fork）。

**导入**（`importers.js`、`getImporter`）：

| 格式 | 识别依据 |
|---|---|
| Claude | 数组，且第一个元素含有 `chat_messages` |
| ChatGPT | 带 `mapping` 树的对象数组；谱系和引用由 `packages/api/src/conversations/chatgpt.ts` 处理（`createChatGptLineage`、思考内容、`model_slug`） |
| ChatbotUI v1 | `{version, history[]}` |
| LibreChat | `{conversationId, messagesTree \| messages}` |

每次导入都经过内容过滤和 PII 检查，并应用保留期字段。导出由客户端基于 GET 端点构建；没有专门的导出路由。

**搜索 / Meilisearch：**

- `mongoMeili` 插件（`packages/data-schemas/src/models/plugins/mongoMeili.ts`）注册到 Conversation 模型（索引 `convos`，主键 `conversationId`）和 Message 模型（索引 `messages`，主键 `messageId`）。
- 被索引的字段是标记了 `meiliIndex: true` 的 schema 路径：
  - 对话：`conversationId`、`title`、`user`、`tags`
  - 消息：`messageId`、`conversationId`、`user`、`sender`、`text`、`content`
- 内容片段通过 `parseTextParts` 展平为 `text`，ID 中的 `|` 替换为 `--`。`filterableAttributes: ['user']`。
- Mongoose 的保存、更新和删除钩子异步写入 Meili，带重试、版本号（`_meiliIndexVersion`）和 `_meiliIndex` 标志。临时文档和已过期文档被排除。
- `api/db/indexSync.js` 在启动时于流程锁（`MEILI_SYNC`）下运行 `syncWithMeili()`，按 `MEILI_SYNC_BATCH_SIZE` / `MEILI_SYNC_DELAY_MS` / `MEILI_SYNC_THRESHOLD` 分批。它还会清理缺少 `user` 的文档。`MEILI_NO_SYNC` 可禁用它。
- 环境变量：`SEARCH=true`、`MEILI_HOST`、`MEILI_MASTER_KEY`。
- `GET /api/search/enable` 以布尔值返回 Meili 的健康状态。

## 6. 分享链接、标签、提示词、项目、收藏、横幅、分类、洞察与链路追踪

**分享链接**（`api/server/routes/share.js`、`packages/data-schemas/src/methods/share.ts`、`packages/api/src/shared-links/*`）：

- Schema `SharedLink`：`{conversationId, title, user, messages[ObjectId], shareId (nanoid), targetMessageId, tenantId, expiredAt (TTL), snapshotFiles, fileSnapshots[{file_id, source, storageKey, filepath, type, filename, bytes, width, height, llmDeliveryPath, ...}]}`。
- 只有当 `ALLOW_SHARED_LINKS` 未设置或为 true 时，公开路由才存在。
- **查看者路由**（`optionalJwtAuth`，然后 `canAccessSharedLink`）：
  - `GET /:shareId`、`GET /:shareId/config`、`POST /:shareId/fork`（需要认证）。
  - `GET /:shareId/files/:file_id[/preview|/download]` — 从快照流式返回，而不是依据所有者当前的 ACL。
- **`canAccessSharedLink`**（`shared-links/access.ts`）：
  1. 在未过期的记录中按 `shareId` 查找分享，先在查看者的租户中找，再在全系统范围内找。
  2. 自动迁移旧版的 `isPublic` 链接。
  3. 如果 PUBLIC 主体对 `SHARED_LINK` 拥有 VIEW：仅当设置了 `ALLOW_SHARED_LINKS_PUBLIC` 时允许匿名访问；否则要求用户已登录。
  4. 否则要求用户具有 ACL VIEW 权限。
- **输出匿名化。** 响应把 ID 改写为 `convo_<nanoid>`、`msg_<nanoid>`、`a_<nanoid>`，在单个响应内做记忆化。敏感的文件字段（`user`、`_id`、`storageKey` 等）被剔除。设置了 `targetMessageId` 时，结果被裁剪到通往目标消息的路径。
- **所有者路由：**
  - `GET /` — 游标分页与搜索。
  - `GET /link/:conversationId`。
  - `POST /:conversationId {targetMessageId?, snapshotFiles?}` — 需要 `SHARED_LINKS` 的 CREATE 权限。
    - `expiredAt` 来自对话的保留期；通过竞态检查强制执行 `SHARE_EXISTS`。
    - 授权：向用户授予 `SHARED_LINK_OWNER`；如果角色拥有 `SHARE_PUBLIC`，还向 PUBLIC 授予 `SHARED_LINK_VIEWER`。
  - `PATCH /:shareId`。
  - `DELETE /:shareId` — 同时清理 ACL 条目。

**标签 / 书签**（`routes/tags.js`、`methods/conversationTag.ts`）：

- Schema `{tag, user, description, count, position, tenantId}`，唯一键 `(tag, user, tenantId)`。
- 路由：`GET /`、`POST / {tag, description, addToConversation, conversationId}`、`PUT /:tag`（重命名、带位移的重新排序、描述）、`DELETE /:tag`（同时从对话中移除该标签）、`PUT /convo/:conversationId {tags}`（计算差异并递增/递减计数）。
- 受 `BOOKMARKS` 权限控制（`checkBookmarkAccess`）。

**提示词**（`routes/prompts.js`、`methods/prompt.ts`、`packages/api/src/prompts`）：

- `PromptGroup {name, numberOfGenerations, oneliner, category, productionId→Prompt, author, authorName, command /^[a-z0-9-]+$/}`。
- `Prompt {groupId, author, prompt, type: text|chat}`。
- 路由：
  - `GET /groups` — 游标分页；按名称/分类过滤；包含通过 ACL 找到的共享分组。
  - `GET /all`、`GET /groups/:groupId`（VIEW）、`POST /`（创建分组及其第一条提示词）。
  - `POST /groups/:groupId/prompts`（EDIT）、`POST /groups/:groupId/use`（递增 `numberOfGenerations`）。
  - `PATCH /groups/:groupId`、`PATCH /:promptId/tags/production`（设置 `productionId`）。
  - `GET /:promptId`、`GET /?groupId`、`DELETE /:promptId`、`DELETE /groups/:groupId`。
- ACL 资源类型为 `PROMPTGROUP`，权限位为 VIEW、EDIT、DELETE 和 SHARE。

**项目**（`routes/projects.js`、`packages/api/src/projects/handlers.ts`、`methods/chatProject.ts`）：

- `ChatProject {name ≤100, description ≤1000, user, conversationCount, lastConversationAt, lastConversationId}`。
- 路由：列表（游标/排序/搜索）、创建、`PUT /conversations/:id {projectId|null}`（分配或取消分配；刷新统计）、获取、修改、删除。
- 删除项目会清除其下对话的 `chatProjectId`；对话本身不会被删除。

**收藏**（`routes/settings.js`）：

- `GET/POST /api/user/settings/favorites` 在 User 文档上保存最多 50 个条目，形如 `{agentId}` 或 `{model, endpoint}`（`FavoritesController.js`）。
- 工具收藏：`ToolFavorite {user, itemType, itemId}`，唯一；路由为 `PUT/DELETE /favorites/tools/:itemType/:itemId`。
- `pinned-order` 同样保存在用户上。

**横幅：** `GET /api/banner`（可选认证）。返回当前生效的 `{bannerId, message, displayFrom, displayTo, type:'banner', isPublic, persistable}`。非公开横幅只向已认证用户展示。

**分类：** `GET /api/categories` 返回提示词分类列表。

**洞察：**

- `GET /api/insights/access` 和 `GET /api/insights`。需要 `ENABLE_INSIGHTS`，且至少有一个可访问的智能体。
- 参数：`range`、`from/toTimestamp`、`timeZone`、`search`、`agentIds`（必须是可访问智能体的子集）、`page`、`pageSize` 5–50。
- 基于对 Conversation 和 Message 的 Mongo 聚合（`methods/insights.ts`）：每日对话数、每日消息数、日活跃用户、流失用户、按用户计数。

**链路追踪：**

- `GET /api/traces/:conversationId/{availability, records, records/:recordId}`。
- 先检查对话所有权，再按存储的链路追踪引用读取 Langfuse 链路追踪。

## 7. Python 对应方案

- **上传：** FastAPI 的 `UploadFile` 配合 `SpooledTemporaryFile`。流式写入 `uploads/temp/<user>/`，生成 `file_id = uuid4()`，并在一个仿照 `filterFile` 的依赖中强制检查 MIME 和大小。根据 `Accept` 返回 JSON 或 SSE `StreamingResponse`（`event: data|error|close`）。总是在 `finally` 中删除临时文件。
- **策略接口：** 一个 `typing.Protocol` 或 ABC `StorageStrategy`，包含可选方法（`upload_file`、`upload_image`、`save_buffer`、`save_url`、`get_url`、`delete`、`open_stream`、`get_download_url`、`process_avatar`），外加一个以 `FileSource` `StrEnum` 为键的注册表 dict。
- **后端：**
  - Local：`aiofiles`，并针对基础目录做路径穿越检查。
  - S3 和 CloudFront：`boto3` / `aioboto3`（`put_object`、用于分段上传的 `upload_fileobj`、`generate_presigned_url('get_object', Params={..., ResponseContentDisposition}, ExpiresIn)`、`head_object`、`delete_object`）。对 CloudFront，`botocore.signers.CloudFrontSigner` 配合 `rsa` / `cryptography` 签名器，可同时处理 URL 和 cookie 策略。
  - Azure：`azure-storage-blob`（`BlobServiceClient.from_connection_string` 或 `DefaultAzureCredential`）。
  - Firebase：`firebase-admin` storage。
  - OpenAI：`openai.files`。
- **图片：** Pillow（`ImageOps.exif_transpose`、按 512 / 768×2000 规则的 `thumbnail`、`save(format=...)`），或为了速度使用 `pyvips`。
- **文档解析：** `pypdf` 或 `pdfplumber`、`python-docx` / `mammoth`、`openpyxl`、`odfpy`。
- **HTTP 客户端：** RAG 和 Mistral 使用 `httpx.AsyncClient`。用 `PyJWT` 签发 RAG JWT（`jwt.encode({'id': uid, 'exp': now+5m}, JWT_SECRET, 'HS256')`），并通过 `files=` 发送 multipart 的 `/embed` 和 `/text` 请求。
- **数据库：** Motor / Beanie（或 PyMongo），配合 TTL 索引（文件的 `expiresAt` 上 `expireAfterSeconds=3600`；对话、消息和分享的 `expiredAt` 上为 `0`）。保留复合唯一索引。
- **分页游标：** `base64(json({primary, secondary, id}))`，使用相同的 `$or` 边界条件。
- **后台保留期清扫：** APScheduler，或带 Redis 领导者锁的 asyncio 任务。
- **搜索：** `meilisearch-python`（或异步的 `meilisearch-python-sdk`）：`index('convos'|'messages')`、`update_filterable_attributes(['user'])`、`search(q, {'filter': f'user = "{uid}"'})`。用 Beanie 事件钩子或 outbox 加后台工作进程取代 Mongoose 后置钩子，再加一个由 `_meiliIndex` 标志驱动的启动时对账任务。
- **分享 ID 与 ACL：** `nanoid`（`nanoid` 包）或 `secrets.token_urlsafe`。ACL 是一张 主体/资源/权限位 表。
