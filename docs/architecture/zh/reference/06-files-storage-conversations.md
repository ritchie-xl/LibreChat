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

