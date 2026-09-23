# Files, Storage, RAG, Conversations, Search & Sharing

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.


## 1. Storage strategy abstraction

**Dispatcher.** `api/server/services/Files/strategies.js` has `getStrategyFunctions(source)`. It returns a plain object with nine optional functions. `null` means the backend does not support that operation.

| fn | signature (conceptual) |
|---|---|
| `handleFileUpload` | `({req, file(multer: path, originalname, mimetype, size), file_id, basePath?, entity_id?, openai?}) → {filepath, bytes, storageKey?, storageRegion?, embedded?, width?, height?, filename?, id?}` |
| `handleImageUpload` | `({req, file, file_id, endpoint, resolution='high'}) → {filepath, bytes, width, height, storageKey?}` (resize and convert happen inside) |
| `saveBuffer` | `({userId, buffer, fileName, basePath='images', tenantId?}) → filepath/URL` |
| `saveURL` | `({userId, URL, fileName, basePath, tenantId}) → string or {filepath, storageKey, bytes, type, dimensions}` |
| `getFileURL` | `({userId, fileName, basePath, tenantId}) → URL` |
| `deleteFile` | `(req, fileRecord, openai?) → void` |
| `getDownloadStream` | `(req, pathOrKey) → Readable` |
| `getDownloadURL` | `({req, file, customFilename, contentType}) → signed URL` (S3 and CloudFront only) |
| `prepareImagePayload`, `processAvatar` | image payload for the LLM / avatar persistence |

**`FileSources` enum** (`packages/data-provider/src/types/files.ts`):

- Byte stores: `local`, `firebase`, `azure_blob`, `s3`, `cloudfront`.
- Remote provider stores: `openai`, `azure` (Azure OpenAI Assistants; also routed to the OpenAI strategy).
- Pseudo-sources:
  - `vectordb` (RAG)
  - `execute_code` (code sandbox API)
  - `mistral_ocr`, `azure_mistral_ocr`, `vertexai_mistral_ocr`
  - `document_parser` (built-in pdfjs/mammoth/xlsx parser, `packages/api/src/files/documents/crud.ts`)
  - `text` (extracted text only; uses the local strategy; no bytes to delete)

**`FileContext` enum:** `avatar`, `unknown`, `agents`, `assistants`, `execute_code`, `image_generation`, `assistants_output`, `message_attachment`, `run_artifact`, `skill_file`. There are also some sort/filter keys.

**Strategy selection.** `api/server/utils/getFileStrategy.js` picks the backend:

- With only `appConfig.fileStrategy` set, that value is used (default `local`).
- With granular `fileStrategies: {default, avatar, image, document, skills}`, the choice depends on the flags `isAvatar`, `isImage` and on `context`.
- Some older paths use `appConfig.fileStrategy` directly (for example `processFileUpload`).

**Implementations:**

- **Local** (`api/server/services/Files/Local/{crud,images}.js`)
  - Images go to `paths.imageOutput/<userId>/<file_id>__<name>.<imageOutputType>` and are served statically as `/images/<userId>/...`.
  - Documents go to `paths.uploads/<userId>/<file_id>__<name>`; `filepath` is `/uploads/<userId>/...` and they are served only through the API.
  - Delete checks for path traversal (`isValidPath`) and that the path belongs to the requesting user.
- **S3** (`packages/api/src/storage/s3/crud.ts`, client in `packages/api/src/cdn/s3.ts`)
  - Env: `AWS_REGION`, `AWS_BUCKET_NAME`, `AWS_ENDPOINT_URL`, `AWS_ACCESS_KEY_ID/SECRET`, `AWS_FORCE_PATH_STYLE`.
  - Key format from `getS3Key`: `[i|a/][r/<region>/][t/<tenantId>/]<basePath>/<userId>/<fileName>`, where `fileName = <file_id>__<originalname>`. `basePath` defaults to `images`; avatars use `avatars`.
  - `filepath` stores a **presigned GET URL** (expiry `S3_URL_EXPIRY_SECONDS`, default 120s). `storageKey` and `storageRegion` are stored alongside it.
  - `GET /api/files` runs `refreshS3FileUrls`, throttled to once per 30 min per user via cache. `needsRefresh` parses `X-Amz-Date` and `X-Amz-Expires` to decide.
  - Delete does HeadObject, then DeleteObject, then `deleteRagFile`. It first verifies that the key's `userId` and `tenantId` segments match the record's owner.
  - Large buffers use multipart upload.
- **CloudFront** (`packages/api/src/storage/cloudfront/crud.ts`, `packages/api/src/cdn/cloudfront*.ts`)
  - Bytes go to S3; URLs are `domain/<percent-encoded key>`.
  - URLs are optionally signed with `CLOUDFRONT_KEY_PAIR_ID` / `CLOUDFRONT_PRIVATE_KEY`, either as a canned policy or a custom policy when the URL has a query string.
  - Alternatively, signed cookies are set per user/tenant scope (`setCloudFrontCookies`, `maybeRefreshCloudFrontAuthCookiesMiddleware`). Cookie mode uses the inline path prefixes `i/` (images) and `a/` (avatars).
- **Azure Blob** (`api/server/services/Files/Azure/crud.js`, `packages/api/src/cdn/azure.ts`)
  - Auth is `AZURE_STORAGE_CONNECTION_STRING`, or `AZURE_STORAGE_ACCOUNT_NAME` plus `DefaultAzureCredential`.
  - Container is `AZURE_CONTAINER_NAME` (default `files`); blob path is `<basePath>/<userId>/<fileName>`.
  - `AZURE_STORAGE_PUBLIC_ACCESS=true` sets the blob access level. `filepath` is the blob URL.
  - Delete checks that the blob path contains `req.user.id`.
- **Firebase** (`api/server/services/Files/Firebase/*`): uses `FIREBASE_*` env vars and paths like `images/<userId>/<name>`.
- **OpenAI** (`Files/OpenAI/crud.js`): `openai.files.create/del/content`. `filepath = <baseURL>/files/<id>`. Deletes are serialized through a leaky-bucket queue (`LB_QueueAsyncCall`).
- **Code** (`Files/Code/crud.js`)
  - Calls the external code API (`getCodeBaseURL()` from `@librechat/agents`): `POST /upload`, `POST /upload/batch`, `GET /download/<session>/<fileId>?kind=&id=`, `DELETE /files/<session_id>/<fileId>`.
  - Refs are stored in `metadata.codeEnvRef(s) = {kind: user|agent, id, storage_session_id, file_id, executionProfile}`.
- **VectorDB**: see section 3.
- **OCR** (`packages/api/src/files/mistral/crud.ts`)
  1. `POST {baseURL}/files` (multipart, purpose ocr).
  2. `GET /files/{id}/url?expiry=`.
  3. `POST /ocr` with `{model, image_limit:0, include_image_base64:false, document:{type:'document_url'|'image_url', ...}}`.
  4. Join the page markdown into text.
  5. `DELETE /files/{id}`.
  - Credentials come from `appConfig.ocr.{apiKey, baseURL, model, strategy}` with env-variable interpolation.
  - The Azure and Vertex variants use the same shape.

**Images** (`api/server/services/Files/images/resize.js`, `convert.js`, sharp):

- `resizeImageBuffer(buf, 'low'|'high'|{percentage|px}, endpoint)`:
  - `low` fits within 512×512.
  - `high` caps the short side at 768 and the long side at 2000 (1568 for Anthropic), with `rotate()` for EXIF orientation.
- `resizeAndConvert` makes 150px-wide thumbnails.
- Output format is `appConfig.imageOutputType` (default `png`).
- Avatars go through `images/avatar.js` `resizeAvatar`, then `strategy.processAvatar`, which also updates `user.avatar`.
- `saveBase64Image` in `process.js` handles data-URL images from image generation: decode, resize, record the true MIME type from the re-encoded bytes, `saveBuffer`, `createFile`.

## 2. File upload and processing flows

**Router** (`api/server/routes/files/index.js`), mounted at `/api/files`:

- Middleware: `requireJwtAuth`, `configMiddleware`, `checkBan`, `uaParser`.
- Multer disk storage (`multer.js`) writes to `uploads/temp/<userId>/<sanitized name>` and sets `req.file_id = randomUUID()`. The filename is URI-decoded first; bad encoding gives 400.
- Multer `fileFilter` normalizes the MIME type (`inferMimeType`) and checks it against the endpoint's `supportedMimeTypes`. For `agents`, it checks the union of all endpoints. Rejection is 415.
- Multer size limit is `serverFileSizeLimit` (default 512MB).
- Every POST except `/usage` and `/speech` goes through IP and user upload rate limiters.

**Routes:**

- `GET /` — the user's files; `text` is excluded by default.
- `GET /agent/:agent_id` — requires agent author or EDIT permission.
- `GET /config`.
- `POST /usage` — extends the upload TTL for queued uploads.
- `DELETE /` — body `{files:[{file_id, filepath}], agent_id?, assistant_id?, tool_resource?}`.
- `GET /code/download/:session_id/:fileId`.
- `GET /:file_id/preview` — polls deferred preview status `pending|ready|failed`. Records stuck in `pending` for more than 2 min are lazily marked `failed`.
- `GET /download-url/:userId/:file_id` — signed URL JSON.
- `GET /download/:userId/:file_id` — streams the file. Headers: `Content-Disposition`, `application/octet-stream`, and `X-File-Metadata` (URI-encoded JSON of an allowlisted field set). `?direct=true` gives a 302 to the signed URL. `text`-source files return their stored text as `.txt`.
- `POST /` — generic upload.
- `POST /images` — image upload.
- `POST /images/avatar`, `POST /images/agents/:id/avatar`, `POST /images/assistants/:id/avatar`.
- Speech routes.

**Upload responses:**

- Default is JSON `{message, ...fileRecord}`.
- If the `Accept` header includes `text/event-stream`, the response is SSE (`packages/api/src/files/sse.ts`): `event:data`, `event:error {message, code, temp_file_id, tool_resource}`, then `event:close`, with heartbeats in between.
- The client sends `file_id` in the form. It becomes `temp_file_id` (used for client correlation), and the server uses its own `req.file_id`.
- The temp file is always deleted in `finally`.

**Common preamble** in `files.js` `handleFileUpload` and `images.js`:

1. `verifyAgentUploadPermission` — agent EDIT permission, unless it is a message attachment or the user has the `MANAGE_AGENTS` capability.
2. `resolveUploadEndpoint` — resolve the agent's provider.
3. `filterFile` (`process.js`): `file_id` must be a UUID, file must be non-empty, check size against the endpoint's `fileSizeLimit`, the `disabled` flag, and the MIME type. Images also require `width` and `height` in the body.
4. `resolveEffectiveToolResource`.
5. `checkToolResourceUploadPermission` — role permission.
6. `assertUploadContentAllowed` — content filters and PII policy.
7. Dispatch: assistants go to `processFileUpload`; everything else goes to `processAgentFileUpload` or `processImageFile`.

**(a) Image to chat** (`POST /api/files/images`, message attachment, no tool_resource):

1. `processImageFile`: `source = getFileStrategy({isImage:true})`.
2. `handleImageUpload` resizes (`high`), converts to `imageOutputType`, and stores the bytes.
3. `resolveUploadLLMDeliveryPath` sets `llmDeliveryPath` to `provider`, `text` or `none`.
4. `createFile({user, file_id, temp_file_id, bytes, filepath, storageKey, filename, context:'message_attachment', source, type:'image/png', width, height, tenantId, llmDeliveryPath, metadata:{destinationChosen, routingMimeType?}, expiredAt?}, disableTTL=true)`.
5. At send time the agent pipeline reads the file from storage and base64-encodes it for the provider (`packages/api/src/files/encode/*`). `updateFilesUsage` increments `usage` and unsets `expiresAt`.

**(b) Document for file_search / RAG** (`POST /api/files` with `tool_resource=file_search`, `agent_id`):

1. `processAgentFileUpload` → `resolveUploadDestination` checks capabilities: `checkCapability(file_search)` and the role grant `fileSearch`.
2. "Dual storage" step 1: `handleFileUpload` stores the bytes in the configured store.
3. Dual storage step 2: `uploadVectors` → `POST {RAG_API_URL}/embed` with multipart `file_id`, `file`, `entity_id` (the agent id for agent knowledge; omitted for user attachments).
4. `metadata.embeddedEntities = [agent_id]`; `embedded = known_type`.
5. `db.addAgentResourceFile({agent_id, tool_resource, file_id})`.
6. `createFile(...)` with `context: agents` (or `message_attachment`).
7. Message attachments that were not embedded are embedded lazily at turn time via `provisionToVectorDB` (`packages/api/src/files/provision/service.ts`), which streams from storage and calls `uploadVectors`.

**(c) Agent tool_resources:**

- **`execute_code`**
  - Requires `checkCapability(execute_code)` and `roleGrants.runCode`.
  - Bytes are stored first. Then, only if the user chose this destination explicitly, `withCodeApiUploadRecovery` streams from storage to `codeapi POST /upload` with `kind` = `agent` (or `user` for attachments). This step is rate-limited and retried.
  - Result: `metadata.codeEnvRef` is set. If the upload fails, the stored bytes are deleted.
  - Destinations the system inferred rather than the user choosing them are provisioned lazily per turn (`provisionToCodeEnv`).
- **`context`** — text extraction; see (d). A `text` field is saved on the record with `llmDeliveryPath:'text'`.
- **`image_edit` / `ocr`** — also valid resource keys (`AGENT_TOOL_RESOURCE_KEYS`).
- The agent's `tool_resources.<key>.file_ids` array links the agent to its files.

**(d) Text extraction / OCR** (context resource), ordered by `getUploadExtractedTextPlan`:

1. **configuredOCR** (when `appConfig.ocr` is set and the type is OCR-able): `getStrategyFunctions(ocr.strategy ?? document_parser).handleFileUpload`. On error it falls back to `document_parser`. Requires the agents `ocr` capability.
2. **documentParser**: pdfjs, mammoth for docx, xlsx/ods, odt.
3. **STT** for audio types in `fileConfig.stt.supportedMimeTypes`: `STTService`, then transcript.
4. **configuredRAG**: `parseText` (`packages/api/src/files/text.ts`) calls `GET {RAG}/health` (10s timeout), then `POST {RAG}/text` with multipart `file_id`, `file` (5 min timeout), returning `{text}`. On failure it falls back to the document parser. Markdown always skips RAG and is read natively.
5. **Native UTF-8 read**: only for natively readable text types.

For every path:

- Extracted text is capped at `MAX_STORED_EXTRACTED_TEXT_BYTES` and checked by content filters.
- The bytes are still stored via `handleFileUpload`.

**Assistants** (`processFileUpload`): uses the OpenAI or Azure strategy. It also calls `assistants.files.create` or `addResourceFileId`. For images, a local converted copy provides the `filepath`.

## 3. RAG integration contract (external `rag_api`)

**Deployment.** `rag.yml` runs `pgvector/pgvector:0.8.0-pg15` plus the image `registry.librechat.ai/danny-avila/librechat-rag-api-dev:latest` on `RAG_PORT`. LibreChat only needs `RAG_API_URL`.

**Auth.** Every call sends `Authorization: Bearer <JWT>`. The token is HS256 over `{id: userId}`, signed with `JWT_SECRET`, expiring in 5m (`generateShortLivedToken`, `packages/api/src/crypto/jwt.ts`). The RAG API must share `JWT_SECRET`.

**Endpoints called:**

| Method/path | Caller | Request | Response used |
|---|---|---|---|
| `GET /health` | startup check `packages/api/src/app/checks.ts`; `parseText` | – | 200 |
| `POST /embed` | `Files/VectorDB/crud.js` `uploadVectors` | multipart: `file_id`, `file`, optional `entity_id`, optional `storage_metadata` (JSON) | `{status: bool, known_type: bool}` → `embedded = known_type`; errors if `known_type === false` or `!status` |
| `POST /text` | `packages/api/src/files/text.ts` | multipart `file_id`, `file` | `{text}` |
| `POST /query` | `api/app/clients/tools/util/fileSearch.js` (tool); `api/app/clients/prompts/createContextHandlers.js` (legacy) | JSON `{file_id, query, k:5, entity_id?}` (tool) / `k:4` (legacy) | `[[{page_content, metadata:{source, page}}, distance], ...]` |
| `GET /documents/{file_id}/context` | legacy context handler when `RAG_USE_FULL_CONTEXT` | – | full text |
| `DELETE /documents` | `deleteVectors`, `packages/api/src/files/rag.ts` `deleteRagFile` | JSON body `[file_id]` | 404 treated as already deleted |

**`file_search` tool** (`fileSearch.js`):

1. `primeFiles` collects `tool_resources.file_search.file_ids` plus files attached this turn, filtered by `filterFilesByAgentAccess`. It builds a tool-context note that lists the filenames.
2. The tool makes one `/query` call per file, in parallel.
   - `entity_id` is sent only for agent knowledge files (`fromAgent`). User attachments are embedded under the user namespace.
3. Results are merged, sorted by distance ascending, and the top 10 kept.
4. Sources are `{type:'file', fileId, content, fileName, relevance: 1-distance, pages, pageRelevance}`.
5. The returned string lists File / Anchor / Relevance / Content per result. Anchors use the markers `turn0file<i>`.
6. The artifact is `{file_search: {sources, fileCitations}}` (tool responseFormat `content_and_artifact`).

**Citations.** `processFileCitations` (`Files/Citations/index.js`):

1. Check the `FILE_CITATIONS` role permission.
2. `selectFileCitationSources` applies `maxCitations`, `maxCitationsPerFile` and `minRelevanceScore` from the agents endpoint config.
3. Enrich each source from the File collection (filename, storageType, fileType, bytes).
4. Emit a message attachment `{type:'file_search', file_search:{sources}, toolCallId, messageId, conversationId}`.

## 4. File lifecycle

**File schema** (`packages/data-schemas/src/schema/file.ts`, timestamps):

- Identity and ownership: `user` (ObjectId), `conversationId`, `messageId`, `file_id` (UUID, indexed), `temp_file_id`, `tenantId`.
- Size and naming: `bytes`, `filename`, `filepath`, `storageKey`, `storageRegion`, `object='file'`, `type` (MIME).
- Flags and content: `embedded`, `text`, `textFormat` (`html|text`), `context`, `usage` (default 0), `source` (default local), `model`, `width`, `height`.
- Deferred preview: `status` (`pending|ready|failed`), `previewError`, `previewRevision`.
- `metadata`: `runFile` provenance, `codeEnvRef(s)`, `sourceDispatchedAt`, `embeddedEntities[]`, `destinationChosen`, `routingMimeType`.
- `llmDeliveryPath` (`provider|text|none`).
- Lifecycle: `expiresAt`, `expiredAt`, `deletionAttempts`, `deletionRetryAt`.
- Indexes: `expiredAt`; unique `(filename, conversationId, context, tenantId)` where `context=execute_code`; unique `run_artifact_identity`.

**Two TTL mechanisms:**

1. **`expiresAt`** is a Mongo TTL of 3600s (upload window). `createFile(data, disableTTL)` sets it to now+1h unless `disableTTL`; most upload paths pass `true`.
   - `updateFileUsage` / `updateFilesUsage` (on message send) do `$inc usage` and `$unset expiresAt, temp_file_id`. Updates also unset it.
   - `POST /files/usage` → `extendFilesTTL` moves the deadline forward (`$lt` guard) for queued uploads.
   - Note: the Mongo TTL deletes only the metadata row.
2. **`expiredAt`** is the retention deadline (`api/server/services/Files/retention.js`, `packages/api/src/files/retention.ts`).
   - It comes from the conversation's `expiredAt`, or from `interfaceConfig.retentionMode` (`all` / `temporary`) together with `isTemporary`. Hours come from `temporaryChatRetention` / `generalChatRetention` / `TEMP_CHAT_RETENTION_HOURS` (default 720h, clamp 1–8760).
   - Persistent agent-resource uploads are exempt unless `retentionMode=all` and `retainAgentFiles` is not set.
   - The application sweep (`packages/api/src/files/sweep.ts`, started via `startExpiredFileSweep`) is leader-only and runs every `FILE_RETENTION_SWEEP_INTERVAL_MS` (default 1h; 0 disables it).
   - Each pass: `getExpiredFiles(100)` where `expiredAt<=now` and (`deletionRetryAt` is null or `<=now`), then `processDeleteRequest`.
   - On failure: `incrementFileDeletionAttempts`, and `deferExpiredFile` with exponential backoff (base is the interval, minimum 60s, cap 24h). After `FILE_RETENTION_SWEEP_MAX_ATTEMPTS` (10) the file is parked for 30 days.

**Deletion** (`processDeleteRequest`, `process.js`):

1. Per file, call the primary strategy's `deleteFile`.
2. Also delete secondary copies: vectors if `embedded`, and the code-env copy if `codeEnvRef` is set.
3. Treat "missing" errors (404, `ENOENT`, `NoSuchKey`, and similar) as success.
4. `db.deleteFiles(ids)` runs only for storage-deleted IDs, then `removeAgentResourceFilesFromAllAgents`.
5. Returns `{deletedFileIds, failedFileIds}`.

The DELETE route requires ownership. For agent resources it checks agent author or EDIT permission, and a file shared by other resources is only unlinked (`deleteAgentResourceFiles` in `packages/api/src/files/deletion.ts`).

**Permissions:**

- `api/server/middleware/accessResources/fileAccess.js` allows access when the tenant matches and either:
  - the user owns the file, or
  - the user can VIEW (or is author of) an agent that holds this `file_id` in any `tool_resources.*.file_ids`.
- `Files/permissions.js`: `hasAccessToFilesViaAgent` and `filterFilesByAgentAccess` (EDIT is needed for delete; remote-agent ACL variant included).
- The ACL service is in `packages/api/src/acl`.

## 5. Conversations and messages

**Conversation schema** (`packages/data-schemas/src/schema/convo.ts`, timestamps):

- Core: `conversationId` (UUID), `title` (default `New Chat`), `user` (string id), `messages[ObjectId]`, `isTemporary`.
- All preset fields from `schema/defaults.ts`: endpoint, endpointType, model, temperature, promptPrefix, agent_id, assistant_id, tools, file_ids, `isArchived`, iconURL, spec, and more.
- `tags[]`, `chatProjectId`, `files[]`, `expiredAt`, `tenantId`, `pinned`, `archivedAt`.
- Subagent and event-actor internals (`subagentThread`, leases, and so on) belong to the agent slice.
- Indexes: TTL on `expiredAt` (`expireAfterSeconds:0`); unique `(conversationId, user, tenantId)`.

**Message schema** (`schema/message.ts`):

- Identity: `messageId`, `conversationId`, `user`, `parentMessageId` (the root uses `Constants.NO_PARENT` = `00000000-0000-0000-0000-000000000000`).
- Model info: `model`, `endpoint`, `sender`.
- Content: `text`, `content[]` (typed parts), `isCreatedByUser`, `tokenCount`, `summary`, `unfinished`, `error`, `finish_reason`.
- `feedback{rating: thumbsUp|thumbsDown, tag, text}`.
- `files[]`, `attachments[]`, `metadata`, `contextMeta` (server-private), `quotes`, `expiredAt`, `tenantId`.
- Langfuse trace refs.
- Indexes: TTL on `expiredAt`; unique `(messageId, user, tenantId)`; `(conversationId, user, createdAt, _id)`.

**Branching.** Messages form a tree through `parentMessageId`. Siblings are regenerations or edits. The client picks a path; `BaseClient.getMessagesForConversation` walks from a leaf to the root.

**Routes, `/api/convos`** (`api/server/routes/convos.js`):

- `GET /` → `getConvosByCursor` (`methods/conversation.ts`).
  - Query: `limit`, `cursor`, `isArchived`, `pinned`, `tags[]`, `search`, `sortBy` (`title|createdAt|updatedAt|archivedAt`), `sortDirection`, `projectId` (ObjectId or `unassigned`).
  - Filters: `user`, archived state, retention visibility, and "not a subagent thread".
  - `search` queries Meili for convos and messages filtered by `user = "<id>"`, then filters by `conversationId $in`.
  - The cursor is base64 JSON `{primary, secondary, id}`. It paginates exactly on (sort field, `updatedAt` or `createdAt`, `_id`) with null-group handling. The query fetches `limit+1` rows.
  - Returns `{conversations, nextCursor}`, each with `isShared` attached.
- `GET /:conversationId`.
- `GET /gen_title/:id` — polls a cache with backoff 0.5–8s.
- `POST /update {arg:{conversationId, title}}` — trimmed to 1024 characters and content-filtered.
- `POST /archive {arg:{conversationId, isArchived}}` — preserves `updatedAt`; sets `archivedAt`.
- `POST /archive/all`.
- `POST /pin`.
- `DELETE / {arg:{conversationId}}` — cascade:
  1. Drain running generations and cancel subagents.
  2. `deleteConvos` deletes in waves over subagent child threads, decrements tag counts, and refreshes project stats.
  3. Delete messages, tool calls, and shared links with ACL cleanup.
  4. Prune checkpoints.
- `DELETE /all`.
- `POST /import` — multer, JSON only, size from `resolveImportMaxFileSize`.
- `POST /fork {conversationId, messageId, option, splitAtTarget, latestMessageId}`.
- `POST /duplicate {conversationId, title}`.
- Subagent thread endpoints (other slice).

**Routes, `/api/messages`** (`api/server/routes/messages.js`):

- `GET /?conversationId&cursor&pageSize&sortBy&sortDirection` → `getMessagesByCursor`. The cursor is the raw `createdAt` value; `limit+1` rows are fetched.
- `GET /?search=` → Meili message search, hydrated from Mongo, joined with convo titles via `getConvosQueried`.
- `GET /:conversationId` returns all messages sorted by `createdAt`, using the `CLIENT_MESSAGE_SELECT` projection, which strips `contextMeta`.
- `POST /:conversationId` → `saveMessage`, then `saveConvo` with `appendMessageIds`.
- `PUT /:conversationId/:messageId {text, index?, model}` edits text or a content part and recounts tokens.
- `PUT .../feedback` — also sends a Langfuse score.
- `DELETE /:conversationId/:messageId`.
- `POST /branch {messageId, agentId}` — splits parallel agent content into a new sibling.
- `POST /artifact/:messageId`.

**`saveConvo` / `saveMessage`:**

- `saveConvo` upserts by `(conversationId, user)`. `messages` is recomputed from the message `_id`s (or appended).
- `saveMessage` rejects non-UUID `conversationId` values and upserts by `(messageId, user)`.
- Both apply retention `expiredAt` from `retentionMode`, `isTemporary` and `interfaceConfig` (`createChatExpirationDate`).
- `saveConvo` validates that `chatProjectId` is owned by the user.
- `deleteMessagesSince` deletes messages with `createdAt` greater than the anchor message's.

**Fork** (`api/server/utils/import/fork.js`). Options:

- `DIRECT_PATH` — ancestors only.
- `INCLUDE_BRANCHES` — the path plus siblings at each level.
- `TARGET_LEVEL` (default) — breadth-first over all branches up to the target's depth.
- `splitAtTarget` — starts the new conversation at the target's level.

Implementation:

1. `cloneLineage` (`packages/api/src/conversations/lineage.ts`) remaps every `messageId` to a new UUIDv4 and rewrites `parentMessageId`, keeping timestamps ordered.
2. `ImportBatchBuilder` (`importBatchBuilder.js`) creates a new `conversationId` and runs `bulkSaveConvos` and `bulkSaveMessages`.
3. Writes go through a BSON size guard of 16MB minus 64KB (`packages/api/src/conversations/import.ts`).
4. `forkSharedConversation` forks from a shared link.

**Import** (`importers.js`, `getImporter`):

| Format | Detected by |
|---|---|
| Claude | array whose first element has `chat_messages` |
| ChatGPT | array of objects with a `mapping` tree; lineage and citations handled by `packages/api/src/conversations/chatgpt.ts` (`createChatGptLineage`, thinking content, `model_slug`) |
| ChatbotUI v1 | `{version, history[]}` |
| LibreChat | `{conversationId, messagesTree \| messages}` |

Every import is content-filtered and PII-checked. Retention fields are applied. Export is built on the client side from the GET endpoints; there is no dedicated export route.

**Search / Meilisearch:**

- `mongoMeili` plugin (`packages/data-schemas/src/models/plugins/mongoMeili.ts`) registers on the Conversation model (index `convos`, primary key `conversationId`) and the Message model (index `messages`, primary key `messageId`).
- Indexed fields are the schema paths marked `meiliIndex: true`:
  - convo: `conversationId`, `title`, `user`, `tags`
  - message: `messageId`, `conversationId`, `user`, `sender`, `text`, `content`
- Content parts are flattened into `text` with `parseTextParts`, and `|` in IDs becomes `--`. `filterableAttributes: ['user']`.
- Mongoose save, update and delete hooks write asynchronously to Meili with retries, versioning (`_meiliIndexVersion`) and a `_meiliIndex` flag. Temporary and expired documents are excluded.
- `api/db/indexSync.js` runs `syncWithMeili()` at startup under a flow lock (`MEILI_SYNC`), batching with `MEILI_SYNC_BATCH_SIZE` / `MEILI_SYNC_DELAY_MS` / `MEILI_SYNC_THRESHOLD`. It also cleans documents that lack `user`. `MEILI_NO_SYNC` disables it.
- Env: `SEARCH=true`, `MEILI_HOST`, `MEILI_MASTER_KEY`.
- `GET /api/search/enable` returns the Meili health status as a boolean.

## 6. Share links, tags, prompts, projects, favorites, banner, categories, insights, traces

**Share links** (`api/server/routes/share.js`, `packages/data-schemas/src/methods/share.ts`, `packages/api/src/shared-links/*`):

- Schema `SharedLink`: `{conversationId, title, user, messages[ObjectId], shareId (nanoid), targetMessageId, tenantId, expiredAt (TTL), snapshotFiles, fileSnapshots[{file_id, source, storageKey, filepath, type, filename, bytes, width, height, llmDeliveryPath, ...}]}`.
- Public routes exist only if `ALLOW_SHARED_LINKS` is unset or true.
- **Viewer routes** (`optionalJwtAuth`, then `canAccessSharedLink`):
  - `GET /:shareId`, `GET /:shareId/config`, `POST /:shareId/fork` (auth required).
  - `GET /:shareId/files/:file_id[/preview|/download]` — streams from the snapshot, not the owner's live ACL.
- **`canAccessSharedLink`** (`shared-links/access.ts`):
  1. Look up the share by `shareId` among non-expired rows, trying the viewer's tenant first and then system-wide.
  2. Auto-migrate legacy `isPublic` links.
  3. If the PUBLIC principal has VIEW on `SHARED_LINK`: allow anonymous access only when `ALLOW_SHARED_LINKS_PUBLIC` is set; otherwise require a logged-in user.
  4. Otherwise require user ACL VIEW.
- **Output anonymization.** The response rewrites IDs to `convo_<nanoid>`, `msg_<nanoid>`, `a_<nanoid>`, memoized per response. Sensitive file fields (`user`, `_id`, `storageKey`, and similar) are stripped. With `targetMessageId`, the result is trimmed to the path up to the target.
- **Owner routes:**
  - `GET /` — cursor pagination and search.
  - `GET /link/:conversationId`.
  - `POST /:conversationId {targetMessageId?, snapshotFiles?}` — requires the `SHARED_LINKS` CREATE permission.
    - `expiredAt` comes from conversation retention; `SHARE_EXISTS` is enforced with a race check.
    - Grants: `SHARED_LINK_OWNER` to the user, plus `SHARED_LINK_VIEWER` to PUBLIC if the role has `SHARE_PUBLIC`.
  - `PATCH /:shareId`.
  - `DELETE /:shareId` — also cleans up ACL entries.

**Tags / bookmarks** (`routes/tags.js`, `methods/conversationTag.ts`):

- Schema `{tag, user, description, count, position, tenantId}`, unique `(tag, user, tenantId)`.
- Routes: `GET /`, `POST / {tag, description, addToConversation, conversationId}`, `PUT /:tag` (rename, reposition with shifting, description), `DELETE /:tag` (also pulls it from conversations), `PUT /convo/:conversationId {tags}` (diff with count inc/dec).
- Gated by the `BOOKMARKS` permission (`checkBookmarkAccess`).

**Prompts** (`routes/prompts.js`, `methods/prompt.ts`, `packages/api/src/prompts`):

- `PromptGroup {name, numberOfGenerations, oneliner, category, productionId→Prompt, author, authorName, command /^[a-z0-9-]+$/}`.
- `Prompt {groupId, author, prompt, type: text|chat}`.
- Routes:
  - `GET /groups` — cursor pagination; name/category filters; includes shared groups found through the ACL.
  - `GET /all`, `GET /groups/:groupId` (VIEW), `POST /` (create group with first prompt).
  - `POST /groups/:groupId/prompts` (EDIT), `POST /groups/:groupId/use` (increments `numberOfGenerations`).
  - `PATCH /groups/:groupId`, `PATCH /:promptId/tags/production` (set `productionId`).
  - `GET /:promptId`, `GET /?groupId`, `DELETE /:promptId`, `DELETE /groups/:groupId`.
- ACL resource type is `PROMPTGROUP`, with permission bits VIEW, EDIT, DELETE and SHARE.

**Projects** (`routes/projects.js`, `packages/api/src/projects/handlers.ts`, `methods/chatProject.ts`):

- `ChatProject {name ≤100, description ≤1000, user, conversationCount, lastConversationAt, lastConversationId}`.
- Routes: list (cursor/sort/search), create, `PUT /conversations/:id {projectId|null}` (assign or unassign; refreshes stats), get, patch, delete.
- Deleting a project unsets `chatProjectId` on its conversations; the conversations are not deleted.

**Favorites** (`routes/settings.js`):

- `GET/POST /api/user/settings/favorites` stores up to 50 entries `{agentId}` or `{model, endpoint}` on the User document (`FavoritesController.js`).
- Tool favorites: `ToolFavorite {user, itemType, itemId}`, unique; routes `PUT/DELETE /favorites/tools/:itemType/:itemId`.
- `pinned-order` is also stored on the user.

**Banner:** `GET /api/banner` (optional auth). Returns the active `{bannerId, message, displayFrom, displayTo, type:'banner', isPublic, persistable}`. Non-public banners are shown only to authenticated users.

**Categories:** `GET /api/categories` returns the prompt category list.

**Insights:**

- `GET /api/insights/access` and `GET /api/insights`. Requires `ENABLE_INSIGHTS` and at least one accessible agent.
- Params: `range`, `from/toTimestamp`, `timeZone`, `search`, `agentIds` (must be a subset of the accessible agents), `page`, `pageSize` 5–50.
- Backed by Mongo aggregations over Conversation and Message (`methods/insights.ts`): convos per day, messages per day, daily active users, churned users, per-user counts.

**Traces:**

- `GET /api/traces/:conversationId/{availability, records, records/:recordId}`.
- Checks conversation ownership, then reads Langfuse traces for the stored trace refs.

## 7. Python equivalents

- **Uploads:** FastAPI `UploadFile` with `SpooledTemporaryFile`. Stream to `uploads/temp/<user>/`, generate `file_id = uuid4()`, and enforce MIME and size in a dependency that mirrors `filterFile`. Return JSON or an SSE `StreamingResponse` (`event: data|error|close`) depending on `Accept`. Always unlink the temp file in `finally`.
- **Strategy interface:** a `typing.Protocol` or ABC `StorageStrategy` with optional methods (`upload_file`, `upload_image`, `save_buffer`, `save_url`, `get_url`, `delete`, `open_stream`, `get_download_url`, `process_avatar`), plus a registry dict keyed by a `FileSource` `StrEnum`.
- **Backends:**
  - Local: `aiofiles`, with a path-traversal check against the base directory.
  - S3 and CloudFront: `boto3` / `aioboto3` (`put_object`, `upload_fileobj` for multipart, `generate_presigned_url('get_object', Params={..., ResponseContentDisposition}, ExpiresIn)`, `head_object`, `delete_object`). For CloudFront, `botocore.signers.CloudFrontSigner` with an `rsa` / `cryptography` signer handles both URLs and cookie policies.
  - Azure: `azure-storage-blob` (`BlobServiceClient.from_connection_string` or `DefaultAzureCredential`).
  - Firebase: `firebase-admin` storage.
  - OpenAI: `openai.files`.
- **Images:** Pillow (`ImageOps.exif_transpose`, `thumbnail` with the 512 / 768×2000 rules, `save(format=...)`) or `pyvips` for speed.
- **Document parsing:** `pypdf` or `pdfplumber`, `python-docx` / `mammoth`, `openpyxl`, `odfpy`.
- **HTTP clients:** `httpx.AsyncClient` for RAG and Mistral. Mint the RAG JWT with `PyJWT` (`jwt.encode({'id': uid, 'exp': now+5m}, JWT_SECRET, 'HS256')`) and send multipart `/embed` and `/text` requests via `files=`.
- **Database:** Motor / Beanie (or PyMongo) with TTL indexes (`expireAfterSeconds=3600` on `expiresAt` for files; `0` on `expiredAt` for convos, messages and shares). Keep the compound unique indexes.
- **Pagination cursor:** `base64(json({primary, secondary, id}))` with the same `$or` boundary clauses.
- **Background retention sweep:** APScheduler or an asyncio task with a leader lock in Redis.
- **Search:** `meilisearch-python` (or `meilisearch-python-sdk` async): `index('convos'|'messages')`, `update_filterable_attributes(['user'])`, `search(q, {'filter': f'user = "{uid}"'})`. Use Beanie event hooks or an outbox and background worker instead of Mongoose post-hooks, plus a startup reconciliation job driven by a `_meiliIndex` flag.
- **Share IDs and ACL:** `nanoid` (`nanoid` package) or `secrets.token_urlsafe`. The ACL is a principal/resource/permission-bits table.
