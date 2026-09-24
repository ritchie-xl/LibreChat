# LLM 端点、提供商、模型、模型规格、Assistants、标题与语音

> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 `v0.8.8-rc4`（`361553f`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。

`@librechat/agents` SDK 是外部包，没有内置（vendor）在本仓库中，因此它的内部细节（例如默认的标题提示）只能通过本仓库代码对它的调用方式来描述。

---

## 1. 端点类型及其与提供商的映射

**端点枚举**（`packages/data-provider/src/schemas.ts:23`），`EModelEndpoint`：
- `openAI`、`azureOpenAI`、`google`、`anthropic`、`assistants`、`azureAssistants`、`agents`、`custom`、`bedrock`

**提供商枚举**（`schemas.ts:36`），`Providers`，与 `@librechat/agents` 保持一致：
- `openAI`、`anthropic`、`azureOpenAI`、`google`、`vertexai`、`bedrock`、`mistralai`、`mistral`、`deepseek`、`moonshot`、`openrouter`、`xai`

**已知的自定义端点**（`packages/data-provider/src/config.ts:3144`），`KnownEndpoints`：
- anyscale、apipie、cohere、fireworks、deepseek、moonshot、groq、helicone、huggingface、lemonade、mistral、mlx、ollama、openrouter、perplexity、shuttleai、together.ai、unify、vercel、xai
- `FetchTokenConfig` = {openrouter, helicone}。对这些端点，`/models` 响应带有价格和上下文长度，会被缓存为 token 配置。

**启用的端点**
- `getEnabledEndpoints()`（`packages/data-provider/src/parsers.ts:78`）读取 `ENDPOINTS` 环境变量（逗号分隔列表）。
- 默认值为 openAI、agents、assistants、azureAssistants、azureOpenAI、google、anthropic、bedrock。
- 该列表的顺序就是显示顺序（`orderEndpointsConfig`）。

**提供商 → 初始化器映射**（`packages/api/src/endpoints/config/providers.ts`，`providerConfigMap`）：

| 提供商键 | 初始化器 |
|---|---|
| `openAI`、`azureOpenAI` | `initializeOpenAI`（`endpoints/openai/initialize.ts`） |
| `anthropic` | `initializeAnthropic`（`endpoints/anthropic/initialize.ts`） |
| `google`、`vertexai` | `initializeGoogle`（`endpoints/google/initialize.ts`） |
| `bedrock` | `initializeBedrock`（`endpoints/bedrock/initialize.ts`） |
| `xai`、`deepseek`、`moonshot`、`openrouter` | `initializeCustom`（这些是“已知自定义提供商”） |
| 其他任意值 | 按名称在 `endpoints.custom[]` 中查找，然后调用 `initializeCustom`，并设置 `overrideProvider = openAI` |

`getProviderConfig({provider, appConfig})` 按以下顺序解析提供商：
1. 精确键。
2. 小写键。
3. 自定义端点的名称。
   - 已知自定义提供商还会尝试不区分大小写的名称匹配。
   - 如果有多个条目不区分大小写地匹配，则抛出异常。
4. 如果自定义端点设置了 `provider: anthropic`，`overrideProvider` 变为 `anthropic`，从而使用原生 `/v1/messages` 客户端。

它返回 `{getOptions, overrideProvider, customEndpointConfig}`。

**`getOptions` 返回什么：** 每个初始化器返回一个 `InitializeResultBase`：
- `llmConfig`：LangChain 风格的客户端 kwargs。
- `configOptions`：OpenAI SDK 的 `ClientOptions`（baseURL、defaultHeaders、defaultQuery、fetchOptions、organization）。
- 可选的 `tools`（提供商原生工具，如 web_search、googleSearch、urlContext）。
- 可选的 `provider` 覆盖（例如 `openrouter`、`vertexai`）。
- 可选的 `endpointTokenConfig`、`useLegacyContent`、`azureOptions`。

**智能体在哪里使用它：** `packages/api/src/agents/initialize.ts:1299`。
1. `agent.provider` 经过 `getProviderConfig`，然后以 `model_parameters` 调用 `getOptions`。
2. `maxContextTokens` 按以下顺序解析：显式参数 → `getModelMaxTokens(model, providerEndpointMap[overrideProvider], endpointTokenConfig)` → `DEFAULT_MAX_CONTEXT_TOKENS`。
3. 对于没有 `azureOpenAIApiInstanceName` 的 Azure，提供商会切换为 `openAI`。

同一个解析器也用于标题（`api/server/controllers/agents/client.js:5954`）、摘要（`agents/run.ts:1160`）和活动标签（`agents/activityLabels/host.ts`）。

**仍在使用的旧版（非智能体）端点：** `assistants` 和 `azureAssistants`（OpenAI Assistants API）。其他端点的所有聊天都走 `agents` 管线，普通的模型聊天使用“临时智能体”。

---

## 2. 逐步构建提供商配置

### 共享概念
- **环境变量语法。** `extractEnvVariable("${VAR}")` 解析环境变量；`envVarRegex` 检测未能解析的值。`resolveConfigSecret`（`~/admin/secrets`）解析管理员存储的密钥。
- **用户提供的密钥。** 值等于字面量 `"user_provided"` 表示由用户提供（`isUserProvided`，`packages/api/src/utils/common.ts:50`）。
- **用户密钥存放在哪里。** 它们加密存储在 `Key` 集合中（`packages/data-schemas/src/methods/key.ts`，使用 encrypt/decrypt），以 `{userId, name: endpoint}` 为键。`getUserKey` 返回解密后的字符串。`getUserKeyValues` 返回解密后的 JSON，如 `{apiKey, baseURL}`。
- **密钥过期。** 请求体的 `key` 字段是过期时间戳。`checkUserKeyExpiry` 抛出 JSON 错误 `{type: EXPIRED_USER_KEY}`。缺少密钥时抛出 `{type: NO_USER_KEY}`，缺少 URL 时抛出 `NO_BASE_URL`。
- **密钥路由**（`api/server/routes/keys.js`）：`PUT /api/keys`（upsert name/value/expiresAt）、`GET /api/keys?name=`（过期时间）、`DELETE /api/keys/:name`、`DELETE /api/keys?all=true`。
- **代理。** `PROXY` 环境变量变成一个 undici `ProxyAgent` dispatcher（`getProxyDispatcher`，`utils/proxy.ts`）。对 AWS 则是一个 HTTPS 代理 agent。
- **SSRF 防护。** 当 base URL 由用户提供时：
  - 先执行 `validateEndpointURL(url, endpoint, endpoints.allowedAddresses)`。
  - 请求经过一个 SSRF 安全的 undici `Agent`，设置 `redirect: 'error'`。
  - 配置的请求头会被**扣留**，因此用户令牌或 OIDC 令牌绝不会到达用户控制的目标地址。
- **请求头模板**（`packages/api/src/utils/env.ts`，`resolveHeaders`、`processSingleValue`）。值按以下顺序解析：
  1. `${ENV}`
  2. `{{customUserVar}}`
  3. `{{LIBRECHAT_USER_<FIELD>}}`。FIELD 取以下之一：id、name、username、email、provider、role、googleId、facebookId、openidId、samlId、ldapId、githubId、discordId、appleId、emailVerified、twoFactorEnabled、termsAccepted、termsAcceptedAt。另有 `LIBRECHAT_USER_TENANTID` / `TENANT_ID`。
  4. `{{LIBRECHAT_BODY_CONVERSATIONID|PARENTMESSAGEID|MESSAGEID}}`
  5. `{{LIBRECHAT_OPENID_ACCESS_TOKEN|ID_TOKEN|TOKEN|...}}`
  6. `{{LIBRECHAT_GRAPH_ACCESS_TOKEN}}`

  环境变量先于用户值解析，这防止了二阶注入。启用 `stripUnresolved` 时，未解析的占位符会被移除。凭据占位符无法解析的请求头（例如 OIDC 令牌已过期）会被省略。
- **何时解析请求头。** 大多数提供商在请求时解析：`resolveConfigHeaders({llmConfig, user, tenantId, body})`（`utils/headers.ts:163`）重写 `configuration.defaultHeaders`（OpenAI）、`clientOptions.defaultHeaders`（Anthropic）和 `customHeaders`（Google）。Google 在初始化时解析。Azure 分组的请求头在初始化时解析。
- **请求头合并。** `mergeHeaders(base, override)` 不区分大小写，并对 `anthropic-beta` 的值做逗号并集。优先级：`endpoints.all.headers` < 端点请求头 < 提供商管理的请求头（认证、版本、beta）。
- **流速率。** `streamRate` 变成 `llmConfig._lc_stream_delay`。`endpoints.all.streamRate` 优先于端点自身的值。

### OpenAI / Azure OpenAI（`endpoints/openai/initialize.ts`）
1. **凭据。** `OPENAI_API_KEY` 或 `AZURE_API_KEY`。base URL 为 `OPENAI_REVERSE_PROXY` 或 `AZURE_OPENAI_BASEURL`（都可以是 `user_provided`）。
2. **使用 YAML 分组的 Azure。** `mapModelToAzureConfig({modelName, modelGroupMap, groupMap})`（`packages/data-provider/src/azure.ts:138`）给出 `{azureOptions: {azureOpenAIApiKey, azureOpenAIApiInstanceName, azureOpenAIApiDeploymentName, azureOpenAIApiVersion}, baseURL?, headers (additionalHeaders), serverless?}`。
   - **Serverless：** `defaultQuery: {'api-version'}`、一个 `api-key` 请求头，以及 `useLegacyContent = true`。
3. **仅用环境变量配置的 Azure。** `getAzureCredentials()` 读取 `AZURE_OPENAI_API_*` 环境变量。用户密钥是一个 JSON 编码的 azureOptions 对象。
4. **addParams / dropParams。** Azure 的来自分组（`getOpenAIEndpointParameters`）。原生 OpenAI 没有。
5. **构建配置。** `getOpenAIConfig(apiKey, {proxy, reverseProxyUrl, headers, azure, addParams, dropParams, modelOptions: {...model_parameters, model, user: userId}, customParams, directEndpoint, streaming: true})`（`openai/config.ts`）按如下方式分支：
   - 如果 `customParams.defaultParamsEndpoint` 为 `anthropic` 或 `google`，它先构建原生配置，再由 `transformToOpenAIConfig` 映射为 OpenAI 客户端形式。这用于通过 OpenAI 协议接收 Anthropic 或 Gemini 参数的网关。
   - 否则调用 `getOpenAILLMConfig`（`openai/llm.ts:593`），它执行以下操作：
     - 去除 nullish 和空值。把 `frequency_penalty`/`presence_penalty` 映射为 camelCase。
     - 应用 `defaultParams`（来自 `customParams.paramDefinitions[].default`，仅在字段未设置时），然后应用 `addParams`，后者会覆盖。
     - `web_search`：
       - OpenRouter：`modelKwargs.plugins=[{id:'web'}]`。
       - 其他：`useResponsesApi=true`，并加上工具 `{type:'web_search'}`。
     - 推理（`reasoning_effort`、`reasoning_summary`、`reasoning_mode`、`reasoning_context`）：
       - Responses API：一个 `reasoning` 对象。
       - Chat Completions：扁平的 `reasoning_effort`。
       - OpenRouter：`reasoning: {effort}`、`include_reasoning`，对自适应 Claude 模型还有 `verbosity`。
       - `reasoningFormat`（`customParams.reasoningFormat`，Vercel 为 `reasoningObject`）决定结构。
     - 按模型路由：
       - 带推理的 gpt-5.6 → Responses API。
       - gpt-6 → Responses API，仅限第一方。
       - 对 o1/o3/gpt-5（不含 gpt-5.x 或 -chat），丢弃采样参数。
       - 对 gpt-4o-search，几乎丢弃所有参数。
       - 对 gpt-5+，`maxTokens` 变为 `max_completion_tokens`（Chat Completions）或 `max_output_tokens`（Responses）。
       - DeepSeek → `includeReasoningContent`（在工具轮次中回放推理）。
     - `dropParams` 从 llmConfig 和 modelKwargs 中同时删除键，对推理子字段、`web_search` 和 `promptCache` 有特殊处理。
     - 仅 Azure：
       - 部署名：`AZURE_USE_MODEL_AS_DEPLOYMENT_NAME` → 清洗后的模型名；否则从 base URL 推导。
       - 构造 `azureOpenAIBasePath`。
       - 应用 `AZURE_OPENAI_DEFAULT_MODEL`。
       - 对 Azure 上的 Responses API，移除 Azure 字段，构造 Responses URL（`constructAzureResponsesURL`），带 `api-key` 请求头和 `api-version`（默认 `preview`）。
   - **OpenRouter 或 Vercel**（通过 base URL 或端点名检测）：默认请求头 `HTTP-Referer: https://librechat.ai`、`X-Title: LibreChat` 等；默认 `promptCache: true`；结果的 `provider='openrouter'`。
   - `OPENAI_ORGANIZATION` 变为 `organization`。
   - `directEndpoint: true`：自定义 fetch 始终 POST 到精确的 base URL（不追加 `/chat/completions`）。
6. **流速率。** `streamRate` 来自 azure、openAI 或 all 配置。

### Anthropic（`endpoints/anthropic/initialize.ts`、`llm.ts`、`helpers.ts`、`vertex.ts`）
1. **Vertex 模式。** 当 `endpoints.anthropic.vertex(Config)` 存在且 `enabled !== false`，或设置了 `ANTHROPIC_USE_VERTEX` 时启用。
   - 加载 GCP 服务账号凭据（`GOOGLE_SERVICE_KEY_FILE` 或 YAML 中的 `serviceKeyFile`），并使用 `region` / `projectId`。
   - `getVertexDeploymentName` 把可见模型映射到其部署（按模型的 `deploymentName`，否则用全局的，否则用模型名）。
   - `createClient` 构建一个 `AnthropicVertex` 客户端。
2. **直连模式。** `ANTHROPIC_API_KEY`（或用户密钥）。base URL 为 `ANTHROPIC_REVERSE_PROXY`，同时设置为 `anthropicApiUrl` 和 `clientOptions.baseURL`。
3. **`getLLMConfig`：**
   - 从 `modelOptions` 中取出系统选项 `thinking`、`promptCache`、`promptCacheTtl`（`5m`/`1h`）、`thinkingBudget`、`effort` 和 `thinkingDisplay`。
   - `maxTokens` = `maxOutputTokens`，或来自 `anthropicSettings.maxOutputTokens.reset(model)` 的模型默认值。
   - `invocationKwargs.metadata.user_id = userId`。
4. **`configureReasoning`：**
   - 自适应思考模型（Opus/Sonnet 4.6+）：`thinking: {type:'adaptive', display?}` 和 `output_config: {effort}`。
   - 较早的思考模型（3.7、4.x）：`thinking: {type:'enabled', budget_tokens}`，预算被限制为不超过 `max_tokens` 的 90%。
   - Sonnet 5 和 Opus 5：关闭思考需要显式发送 `{type:'disabled'}`。
   - `topP`/`topK` 只在思考未启用时发送；有些模型完全省略采样参数。
5. **提示缓存。** 如果 `promptCache` 开启且模型支持，就向下传递 `promptCache=true`（加上 TTL），由 SDK 应用 `cache_control`。beta 请求头按模型添加（例如 3.7 为 `token-efficient-tools-2025-02-19,output-128k-2025-02-19`），并且总是添加 `fine-grained-tool-streaming-2025-05-14`（除非 `dropParams` 包含 `clientOptions`）。
6. **网页搜索。** 工具 `{type:'web_search_20250305', name:'web_search'}`。Vertex 还会加上 beta 请求头 `web-search-2025-03-05`。
7. **参数与请求头。** `defaultParams`、`addParams` 和 `dropParams` 只作用于已知的 Anthropic 参数。自定义请求头合并在提供商请求头之下。

### Google / Vertex AI（`endpoints/google/initialize.ts`、`llm.ts`）
1. **凭据。**
   - `GOOGLE_KEY` 是 API 密钥（或 `user_provided`）。
   - 否则从 `GOOGLE_SERVICE_KEY_FILE` 或 `api/data/auth.json` 读取服务密钥。
   - `vertexai` 提供商强制使用 Vertex，项目来自 `VERTEX_PROJECT_ID`、`GOOGLE_CLOUD_PROJECT` 等。
2. **提供商选择。** 有项目 id 或 `forceVertex` 即为 Vertex；否则为 Google AI Studio（`apiKey`）。Vertex 的 `location` 来自 `GOOGLE_LOC`（默认 `us-central1`），包括多区域端点。
3. **安全设置。** `getSafetySettings` 根据 `GOOGLE_SAFETY_*` 环境变量为五个危害类别构建阈值。`GOOGLE_EXCLUDE_SAFETY_SETTINGS` 禁用它们。
4. **思考。**
   - Gemini 3+ 和 Gemma 4+：`thinkingConfig: {thinkingLevel, includeThoughts: true}`。
   - 较早的模型：`thinkingBudget`。
5. **工具。** `web_search` 变为 `{googleSearch:{}}`。`url_context` 变为 `{urlContext:{}}`（仅 Gemini 2.5+）。
6. **代理与认证。** `GOOGLE_REVERSE_PROXY` 变为 `baseUrl`。`GOOGLE_AUTH_HEADER` 通过 `customHeaders` 把密钥放进 `Authorization` 请求头。

### Bedrock（`endpoints/bedrock/initialize.ts`）
1. **凭据（环境变量）。** `BEDROCK_AWS_ACCESS_KEY_ID`、`_SECRET_ACCESS_KEY`、`_SESSION_TOKEN`、`_BEARER_TOKEN`（Bedrock API 密钥）和 `_PROFILE`。其中任何一个都可以是 `user_provided`；用户密钥是 JSON `{accessKeyId, secretAccessKey, sessionToken?, bearerToken?}`。
2. **其他环境变量。** `BEDROCK_AWS_DEFAULT_REGION` 和 `BEDROCK_REVERSE_PROXY`。
3. **解析参数。** 先 `bedrockInputParser`，再 `bedrockOutputParser`（`packages/data-provider/src/bedrock.ts`）。输出解析器把 thinking、effort 和 promptCache 映射到 `additionalModelRequestFields` 和 `anthropic_beta`。
4. **YAML 选项。**
   - `guardrailConfig`（identifier、version、trace、streamProcessingMode）。
   - `inferenceProfiles`（模型 → ARN，设置为 `applicationInferenceProfile`）。
5. **客户端构建。** 存在代理或 bearer 令牌时（`authSchemePreference: ['httpBearerAuth']`），会构建一个自定义的 `BedrockRuntimeClient`。否则把凭据、profile 或 `endpointHost` 直接传给 `ChatBedrockConverse`。

### 自定义端点（`endpoints/custom/initialize.ts`）
1. 用 `${ENV}` 解析 `apiKey` 和 `baseURL`。如果任一仍匹配 `envVarRegex`，抛出 “Missing API Key” / “Missing Base URL”。
2. 处理用户提供的密钥和/或 URL。用户提供的 URL 会触发 `validateEndpointURL`。
3. **token 配置。**
   - 静态 YAML `tokenConfig` 直接使用（`cacheRead`/`cacheWrite` 映射为 `read`/`write`）。
   - 对 `FetchTokenConfig` 中的端点，读取缓存的配置，或调用 `fetchModels`，由它从 `/models` 的价格信息填充缓存。
   - 缓存键：`getTokenConfigKey` 把它限定到 endpoint、endpoint:user 或租户哈希。
4. **`provider: anthropic`。** 使用 Anthropic 的 `getLLMConfig`，把 base URL 作为 `reverseProxyUrl`。
5. **否则。** `getOpenAIConfig(apiKey, {reverseProxyUrl: baseURL, headers, addParams, dropParams, customParams, directEndpoint, titleConvo, titleModel, titleMethod, titleMessageRole, streamRate, modelDisplayLabel, ...})`，并设置 `useLegacyContent = true`。

---

## 3. librechat.yaml 的 `endpoints` schema（`packages/data-provider/src/config.ts`）

根节点是 `endpoints`（strict，约第 3067 行）：
- `allowedAddresses`
- `all`：去掉 `baseURL` 的 `baseEndpointSchema`；在读取处优先生效的全局覆盖
- `openAI`、`google`：`baseEndpointSchema`
- `anthropic`：`anthropicEndpointSchema`
- `azureOpenAI`：`azureEndpointSchema`
- `assistants`、`azureAssistants`：`assistantEndpointSchema`
- `agents`：`agentsEndpointSchema`
- `custom`：`endpointSchema.partial()` 的数组
- `bedrock`：`bedrockEndpointSchema`

**`baseEndpointSchema`（第 672 行）**
- 基础：`streamRate`、`baseURL`、`headers`（record）。
- 标题：`titlePrompt`、`titleModel`（`current_model` 表示使用聊天模型）、`titleConvo`、`titleMethod`（`completion`|`functions`|`structured`）、`titleEndpoint`、`titlePromptTemplate`、`titleTiming`（`immediate`|`final`）。
- 智能体 UI 标签：`activityLabel`、`activityModel`、`activityEndpoint`、`activityPrompt`、`activityMaxPerRun`、`activityCharLimit`、`activityPhase*`、`reasoningLabel*`。
- `maxToolResultChars`。

**`endpointSchema`（自定义，第 1606 行）** 在基础之上扩展：
- `name`：不能是 `EModelEndpoint` 的值。
- `apiKey`、`apiKeyPreview`、`baseURL`。
- `models: {default: (string|{name, description})[] (min 1), fetch?, userIdQuery?}`。
- `iconURL`、`modelDisplayLabel`。
- `provider?: 'anthropic'`。
- `headers`。
- `addParams`：record；`web_search` 必须是布尔值。
- `dropParams`：string[]。
- `customParams`（strict）：
  - `defaultParamsEndpoint`（默认 `custom`）、`reasoningFormat`、`reasoningKey`。
  - `includeReasoningContent`、`includeReasoningHistory`。
  - `paramDefinitions[]`：UI 设置定义，包含 key、type、default、range、options、component 等。
- `directEndpoint`、`titleMessageRole`（`system`|`user`|`assistant`）。
- `tokenConfig: Record<model, {prompt, completion, context, cacheRead?, cacheWrite?}>`，费率按每 1M token 计。

**`azureEndpointSchema`（第 1668 行）**
- `groups[]`（至少 1 个）。每个分组包含：`group`、`apiKey`、`serverless?`、`instanceName?`、`deploymentName?`、`version?`、`baseURL?`、`additionalHeaders?`、`addParams?`、`dropParams?`、`assistants?`。
  - `models: Record<modelName, true | {deploymentName?, version?, assistants?}>`。
- 顶层：`assistants?`，外加标题、活动和标签字段的 `.pick()`（并非全部基础字段）。
- 校验后得到 `{modelNames, groupMap, modelGroupMap, assistantModels, assistantGroups}`（`validateAzureGroups`，`azure.ts:13`）。

**`anthropicEndpointSchema`（第 1766 行）**
- 基础字段，外加 `models?: string[]`。
- `vertex?: {enabled?, projectId?, region (default us-east5), serviceKeyFile?, deploymentName?, models?: string[] | Record<model, true | {deploymentName?}>}`。运行时以 `vertexConfig` 暴露，带 `modelNames` 和 `modelDeploymentMap`。

**`bedrockEndpointSchema`（第 766 行）**
- 基础字段，外加 `availableRegions?`、`models?`、`guardrailConfig?`、`inferenceProfiles?`。

**`assistantEndpointSchema`（第 783 行）**
- 基础字段，外加 `disableBuilder`、`pollIntervalMs`、`timeoutMs`、`version`（默认 2）。
- `supportedIds` / `excludedIds` / `privateAssistants`。
- `retrievalModels`、`capabilities`（code_interpreter、image_vision、retrieval、actions、tools）。
- `apiKey`、`models{default, fetch, userIdQuery}`、`headers`。

**`agentsEndpointSchema`（第 1267 行）** 很大：capabilities、allowedProviders、statefulCodeSessions、toolApproval 等。它不在本章范围内。

**仅通过环境变量启用端点**（`api/server/services/Config/EndpointService.js`）
- `generateConfig(key, baseURL)` 给出 `{userProvide, userProvideURL}`，没有密钥时为 `false`。
- Assistants 端点还会得到 `retrievalModels`、`capabilities` 和 `version`。
- 存在 `GOOGLE_KEY` 或服务密钥文件时启用 Google（`loadAsyncEndpoints.js`）。

---

## 4. 模型列表的解析与缓存

**路由**
- `GET /api/models`（`routes/models.js` → `controllers/ModelController.js`）返回 `{...loadDefaultModels(req), ...loadConfigModels(req)}`，即端点名到 `string[]` 的映射。
- `GET /api/endpoints`（`routes/endpoints.js` → `EndpointController` → `createEndpointsConfigService`，`packages/api/src/endpoints/config/endpoints.ts`）返回基于环境变量的默认端点配置，合并 `loadCustomEndpointsConfig(appConfig.endpoints.custom)`。该配置带有 `userProvide`、`userProvideURL`、`order`、`iconURL`、`modelDisplayLabel` 等。
  - Azure：存在 YAML 时为 `{userProvide:false}`。
  - 为 openAI 和 azure 按模型添加 `responsesApiRouting`（`config/responses.ts`）。
  - 设置了 `azureOpenAI.assistants` 时添加 `azureAssistants`。
  - Assistants：version、retrievalModels、disableBuilder、capabilities。
  - Agents：capabilities、allowedProviders、statefulCodeSessions（过滤后）、maxSubagents。
  - Bedrock：availableRegions 和 `userProvide*` 标志。
  - 结果是排好序的。
- `GET /api/endpoints/token-config`（`TokenConfigController`）返回上下文窗口；如果 `interface.contextCost` 开启，还返回价格。

**默认模型**（`api/server/services/Config/loadDefaultModels.js`；函数在 `packages/api/src/endpoints/models.ts`）
- **OpenAI。**
  1. 如果设置了 `OPENAI_MODELS`，使用该列表。
  2. 否则如果密钥由用户提供，使用 `defaultModels`。
  3. 否则拉取 `GET {OPENAI_REVERSE_PROXY or https://api.openai.com/v1}/models`。在 api.openai.com 上，列表按 `/(text-davinci-003|gpt-|o\d+|chat-latest)/` 过滤，并排除 `audio`/`realtime`。
- **Azure。** `AZURE_OPENAI_MODELS` 环境变量（YAML 的 `modelNames` 通过配置模型覆盖它）。
- **Assistants。** `ASSISTANTS_MODELS`，或从 `ASSISTANTS_BASE_URL` 拉取。
- **Anthropic。** 先用 Vertex 的 `modelNames`，再用 `ANTHROPIC_MODELS`，再以 `x-api-key` 和 `anthropic-version: ANTHROPIC_VERSION || 2023-06-01` 拉取 `GET {base}/models`。
- **Google。** `GOOGLE_MODELS` 或默认值。
- **Bedrock。** `BEDROCK_AWS_MODELS` 或默认值。YAML 的 `bedrock.models` 会覆盖。
- 内置默认值在 `packages/data-provider/src/config.ts`（`defaultModels`、`sharedOpenAIModels`、`sharedAnthropicModels`、`bedrockModels`）。

**配置/自定义模型**（`packages/api/src/endpoints/config/models.ts`，`createLoadConfigModels`）
- Azure 的 `modelNames`、`azureAssistants` 的 `assistantModels` 和 bedrock 的 `models` 排在最前。
- 对每个具有 baseURL、apiKey、name 和 `models` 的自定义端点：
  - **`fetch: true` 且为管理员密钥：** 每个请求内按 `baseURL__apiKey__sha256(headers)` 对拉取去重，token 配置复制到同源的兄弟端点，结果为空时回退到 `models.default`。
  - **`fetch: true` 且密钥或 URL 由用户提供：** 加载用户的密钥值，并以 `skipCache` 拉取。URL 由用户提供时不转发请求头。
  - **否则：** `models.default`（仅名称）。
- 端点名会被规范化：`ollama` 转为小写。

**`fetchModels`**（`models.ts`）
- **请求头。** `Authorization: Bearer <apiKey>`，除非配置的请求头已经提供了一个。请求头会针对用户做模板解析。对 openai URL 添加 `OpenAI-Organization`。
- **Ollama。** 以 `ollama` 开头的名称先尝试 `GET {base}/api/tags`。
- **请求。** 5 秒超时。设置 `userIdQuery` 时带 `?user=<id>`。URL 由用户提供时使用 SSRF 安全的 agent。
- **token 配置。** 如果响应匹配 OpenRouter 风格的 `inputSchema`（`data[].{id, pricing.{prompt, completion}, context_length}`），`processModelData` 把价格换算为每 1M 的费率，并以 `tokenKey` 存入 `tokenConfigCache`，另存一个按模型缓存作用域的键用于回填。
- **缓存。** `standardCache(CacheKeys.MODEL_QUERIES)`，以 `sha256(baseURL:apiKey)[:32]` 为键，TTL **2 分钟**。转发了用户范围的请求头，或设置了 `userIdQuery` 且有用户时，跳过缓存。

**模型校验**（`api/server/middleware/validateModel.js`）
- 用正则 `^[a-zA-Z0-9][a-zA-Z0-9_.:/@+-]*$` 检查模型，最长 256 个字符。
- 端点为 `userProvide` 时跳过。
- 否则模型必须在 `modelsConfig[resolveModelCatalogKey(endpoint)]` 中。如果不在，记录一次 `ILLEGAL_MODEL_REQUEST` 违规并拒绝请求。

---

## 5. token、价格与上下文表

**上下文窗口**（`packages/api/src/utils/tokens.ts`）
- **表。** 按模型家族的静态映射：openAIModels、mistral、cohere、google、anthropic、deepseek、moonshot、meta、qwen、amazon、bedrock、xAI，以及 `aggregateModels`（并集）。
- **按端点的 `maxTokensMap`：** azureOpenAI → openAI；openAI、agents 和 custom → aggregate；google；anthropic；bedrock。
- **输出上限。** `maxOutputTokensMap` 和 `modelMaxOutputs` 保存输出上限。
- **查找**（`getModelTokenValue`）：
  1. 精确键。
  2. `findMatchingPattern`：取作为小写模型名子串的最长键。对 `a/b` 形式的名称，先试完整名称，再试最后一个 `/` 之后的部分。长度相同时，最后定义的键胜出。
  3. `system_default`。
- **`getModelMaxTokens(model, endpoint, endpointTokenConfig)`。** 先用端点 token 配置的覆盖，再用 Anthropic 1M 上下文的特例，最后查表。
- **`getModelMaxOutputTokens`。** 类似，带 Opus 5.5 和 Sonnet 4.6+ 的特例。
- **对话覆盖。** 对话或预设上的 `maxContextTokens` 覆盖一切。

**价格**（`packages/data-schemas/src/methods/tx.ts`）
- **表**（每 1M token 的美元价格）：
  - `tokenValues`：模型键 → `{prompt, completion}`。
  - `cacheTokenValues`：键 → `{write, read}`。
  - `premiumTokenValues` 和 `premiumCacheTokenValues`：`{threshold, prompt, completion}`。输入 token 数超过阈值时适用长上下文档位（例如 gpt-5.4 超过 272k、gemini-3.1 超过 200k）。
  - `bedrockValues`。
  - `defaultRate = 6`。
- **`getValueKey(model, endpoint)`。** 在 `tokenValues` 上做模式匹配，带旧版 gpt-3.5/gpt-4 分桶（`4k`、`16k`、`8k`、`32k`、`gpt-4-1106`）。
- **`getMultiplier({valueKey, model, endpoint, tokenType, inputTokenCount, endpointTokenConfig})`。** 顺序：endpointTokenConfig 的按模型费率 → premium 费率 → `tokenValues` → `defaultRate`。
- **`getCacheMultiplier`。** 顺序相同，但没有缓存价格时返回 `null`。
- **这些在哪里使用。**
  - `buildTokenConfigMap` / `resolveTokenConfigMap`（`packages/api/src/endpoints/pricing.ts`、`tokenConfig.ts`）为客户端生成 `{endpoint: {model: {context, prompt?, completion?, cacheWrite?, cacheRead?}}}`。
  - 余额和交易计费使用相同的乘数（token × 费率）。
- **缓存键**（`packages/api/src/endpoints/keys.ts`）。带作用域的 token 配置键为 `\0token-config:v2\0{tenant|tenant-user}\0sha256(parts)`。

---

## 6. modelSpecs 语义

**Schema**（`packages/data-provider/src/models.ts`），`specsConfigSchema`：
- `enforce`（默认 false）、`prioritize`（默认 true）、`list[]`、`addedEndpoints[]`。

**每个 `TModelSpec`**
- `name`（id）、`label` 和 `preset`。preset 是 `tModelSpecPresetSchema`：预设字段去掉 id、用户和数据库字段。
- 显示：`order`、`default`、`softDefault`、`description`、`group`（端点名会把该规格嵌套在该端点下；其他任意字符串则形成自定义分组）、`groupIcon`、`showIconInMenu`、`showIconInHeader`、`showOnLanding`、`conversation_starters`、`showInMenu`（false 会把它从菜单和启动配置中隐藏，但仍允许按名称使用）、`iconURL`、`authType`、`hideBadgeRow`。
- 智能体工具开关：`webSearch`、`fileSearch`、`executeCode`、`memory`、`askUserQuestion`、`runInBackground`、`describeIntent`、`artifacts`、`mcpServers[]`、`skills`、`subagents{enabled, allowSelf, shareFiles, agent_ids}`。

**端点推断。** `resolveModelSpecEndpoint` 使用 `preset.endpoint`；如果没有该字段且设置了 `agent_id`，则推断为 `agents`。`materializeModelSpecEndpoints` 在加载配置时一次性写回。

**服务端应用**（`api/server/middleware/buildEndpointOption.js`，辅助函数在 `packages/api/src/modelSpecs/index.ts`）
- **`enforce: true`。**
  1. 请求必须带 `spec`。否则报错 “No model spec selected”。
  2. 该规格必须存在（“Invalid model spec”）。
  3. 规格的端点必须等于请求的端点（“Model spec mismatch”）。
  4. `applyModelSpecPreset(includePresetDefaults: true)` 用预设替换请求体，只保留请求中的 `chatProjectId`，并设置 `spec`。
- **未强制但带有 `spec`。** 请求与规格合并。私有预设字段只在请求缺少时填入。
- **重新解析。** 合并结果再次经过 `parseCompactConvo`，`iconURL` 来自规格。
- **私有字段。** `promptPrefix`、`instructions`、`additional_instructions`、`system`、`context` 和 `examples` 从客户端负载中移除（`sanitizeModelSpecs`）。对非智能体端点，`promptPrefix` 的特殊变量（`{{current_date}}`、`{{current_user}}`、`{{iso_datetime}}`、`{{current_datetime}}`）在服务端解析，用户名会经过内容过滤。
- **启动配置。** `modelSpecs: sanitizeModelSpecs(excludeHiddenModelSpecs(appConfig.modelSpecs))`（`routes/config.js:303`）；`skills` 和 `subagents.agent_ids` 也会被移除。

---

## 7. Assistants API（OpenAI 与 Azure）简述

**路由**（`api/server/routes/assistants/index.js`，挂载在 `/api/assistants`，带 JWT、封禁检查和配置中间件）
- `/v1/`、`/v2/`：助手的 CRUD，代理到 OpenAI：`POST /`、`GET /:id`、`PATCH /:id`、`DELETE /:id`、`GET /`（列表）、`POST /avatar/:assistant_id`。
- 子路由器 `/actions`（`POST /:assistant_id`、`DELETE /:assistant_id/:action_id/:model`）、`/tools`、`/documents`。
- `/v1/chat`、`/v2/chat`：`POST /`，中间件为 `filterMessageContent → validateModel → buildEndpointOption → validateAssistant → validateConvoAccess → guardSubagentThreadTurn → chatController`，另有 `POST /abort`。

**客户端**
- `services/Endpoints/assistants/initalize.js`：`openai` SDK，使用 `ASSISTANTS_API_KEY`、`ASSISTANTS_BASE_URL`（都可以由用户提供）、请求头 `OpenAI-Beta: assistants=v{version}`、PROXY 和 org。
- `azureAssistants/initialize.js`：把模型映射到 Azure 分组，构建 URL `https://{instance}.openai.azure.com/openai`，把 `api-version` 作为查询参数，加上 `api-key` 请求头和解析后的分组请求头；model = 部署名。

**聊天流程**（`controllers/assistants/chatV2.js`）
1. 余额检查。
2. `initThread`（创建或复用 OpenAI 线程，添加带附件的消息）。
3. `saveUserMessage`。
4. 带 `instructions` / `additional_instructions`（promptPrefix、可选的当前日期时间）调用 `createRun`。
5. 使用 `StreamRunManager`（以 SSE 流式输出运行事件、工具调用，以及为函数工具 / actions 调用 `submitToolOutputs`），或使用轮询（`services/AssistantService.js` 中的 `RunManager`、`waitForRun`、`runAssistant`）。
6. `syncMessages` / `processMessages`（`services/Threads/manage.js`）转换线程消息，处理文件引用和图片文件。
7. `saveAssistantMessage`。
8. `addTitle`。
9. 根据 `run.usage` 执行 `recordUsage`。

`filterAssistants` 应用 `privateAssistants`（`metadata.author`）、`supportedIds` 和 `excludedIds`。`packages/api/src/assistants/protection.ts` 对助手线程的消息和文件做 PII/内容过滤。

**Assistants 的标题**（`services/Endpoints/assistants/title.js`）
- 一次硬编码的 `gpt-3.5-turbo` chat completion：“Please generate a concise title (max 40 characters)...”，temperature 0.7，max_tokens 20。
- 出错时回退为截断到 40 个字符的文本。

---

## 8. 对话与预设参数 schema

**`tConversationSchema`**（`packages/data-provider/src/schemas.ts:1115`）
- **身份与状态：** conversationId、endpoint、endpointType、title（默认 “New Chat”）、user、messages[]、tags[]、chatProjectId、createdAt、updatedAt、isArchived、archivedAt、pinned、isShared、expiredAt、isTemporary、parentMessageId。
- **代码：** codeApprovalMode、codeEnvironmentMode、codeWorkspaces[]。
- **通用模型参数：** model、modelLabel、userLabel、promptPrefix、temperature、topP、topK、top_p、frequency_penalty、presence_penalty、maxOutputTokens、maxContextTokens、max_tokens、maxTokens、stop[]、stream、disableStreaming。
- **Anthropic：** promptCache、promptCacheTtl（`5m`|`1h`）、system、thinking、thinkingBudget、effort、thinkingDisplay。
- **Google：** thinkingLevel、context、examples[{input, output}]、url_context。
- **OpenAI：** reasoning_effort、reasoning_summary、reasoning_mode、reasoning_context、verbosity、useResponsesApi、imageDetail。
- **共享：** web_search、artifacts、resendFiles、file_ids、fileTokenLimit。
- **Assistants 与智能体：** assistant_id、instructions、additional_instructions、append_current_datetime、agent_id、subagentThread。
- **Bedrock：** region、additionalModelRequestFields。
- **UI：** greeting、spec、iconURL、tools。
- **其他：** presetOverride。
- **已弃用：** chatGptLabel、resendImages。

**`tPresetSchema`。** 对话 schema 去掉 conversationId、chatProjectId、时间戳和 title，再加上 `presetId`、`title`、`defaultPreset`、`order` 和 `endpoint`（任意字符串）。

**按端点的 pick schema**（`parsers.ts:42`，`endpointSchemas`）。它们以白名单方式限定每个端点接受的参数，并去除 nullish 值：
- **openAI、azureOpenAI、custom（`openAISchema`）：** model、modelLabel、promptPrefix、temperature、top_p、presence/frequency_penalty、resendFiles、artifacts、imageDetail、stop、max_tokens、reasoning_*、verbosity、useResponsesApi、web_search、disableStreaming、fileTokenLimit、maxContextTokens、spec、iconURL、greeting。
- **openrouter：** 上述字段，外加 promptCache 和 promptCacheTtl。
- **google：** model、modelLabel、promptPrefix、examples、temperature、maxOutputTokens、topP、topK、thinking、thinkingBudget、thinkingLevel、web_search、url_context 等。
- **anthropic：** model、modelLabel、promptPrefix、temperature、maxOutputTokens、topP、topK、promptCache、promptCacheTtl、thinking、thinkingBudget、effort、thinkingDisplay、web_search、stop、stream 等。
- **bedrock：** `bedrockInputSchema`。
- **agents：** `compactAgentsSchema`。
- **assistants：** `assistantSchema`。

`parseCompactConvo({endpoint, endpointType, conversation, defaultParamsEndpoint})` 根据自定义端点的 `customParams.defaultParamsEndpoint` 选择 schema。

**默认值**（`schemas.ts:439+` 中的 `openAISettings`、`anthropicSettings`、`googleSettings`、`agentsSettings`）：例如 OpenAI 的 temperature 为 1、top_p 为 1、惩罚项为 0；Anthropic 的默认模型、promptCache 为 true、`maxOutputTokens.reset(model)`。

**预设路由**（`api/server/routes/presets.js`；存储在 `packages/data-schemas/src/methods/preset.ts`）
- `GET /api/presets` 返回用户的预设，按 `order` 再按 `updatedAt` 降序排序，并经过内容过滤器投影。
- `POST /api/presets`（带内容过滤器）执行 `savePreset`：
  - 以 `{presetId, user}` 为键 upsert；`presetId` 默认为 UUID；`newPresetId` 用于重命名。
  - `tools` 规范化为 pluginKey 字符串。
  - `defaultPreset: true` 设置 `order = 0` 并取消之前的默认预设。`defaultPreset: false` 同时取消两者。
- `POST /api/presets/delete {presetId?}` 删除一个预设，或删除该用户的所有预设。

---

## 9. 标题生成、内容审核与提示前缀

**智能体标题**（`api/server/services/Endpoints/agents/title.js`、`client.js:5954 titleConvo`）
- **启用条件。** 环境变量 `TITLE_CONVO`（默认 true）。端点的 `titleConvo !== false`。不是临时聊天。
- **时机。** `resolveTitleTiming`（`providers.ts`）：先看 `endpoints.all.titleTiming`，再看端点候选，再看自定义配置；默认 `immediate`，即从第一条用户消息开始并行运行。`final` 在响应之后运行。
- **端点配置。** `endpoints.all`，否则 `endpoints[endpoint]`，否则自定义配置。
- **提供商与模型。** `titleEndpoint` 可以切换提供商/凭据。`titleModel` 覆盖模型（`current_model` 表示保持不变）。选项通过 `getProviderConfig().getOptions` 重新构建，移除最大 token 参数，并解析请求头。
- **生成。** `@librechat/agents` 中的 `run.generateTitle({provider, clientOptions, inputText, contentParts, titleMethod, titlePrompt, titlePromptTemplate})`。
  - `completion` = 普通提示；`functions` / `structured` = 工具或 JSON 输出（Google 使用 `json: true`）。
  - 用量以 `context: 'title'` 记录（计费）。
  - 结果经过 `sanitizeTitle`。
- **超时与存储。** 45 秒超时。结果依次经过 `resolveConversationTitle`，然后写入缓存 `GEN_TITLE`，键为 `${userId}-${convoId}`（TTL 120 秒），然后 `onTitleGenerated`（SSE 事件），然后 `saveConvo({title}, noUpsert)`。如果该流已被取代，标题会被丢弃。

**titlePolicy**（`api/server/services/Endpoints/titlePolicy.js` → `packages/api/src/protection/title.ts`）
- 如果配置了 `filters.conversationTitles.pii` 且标题违反它，使用回退值 `"New Chat"`（前提是回退值被允许）；否则标题为 null（不保存）。

**内容审核**（`api/server/middleware/moderateText.js`）
- 在启用 `OPENAI_MODERATION` 时运行。
- 收集 `text`/`answer`、ask-user 的回答、引用文本和工具审批决策文本。
- 以 `Bearer OPENAI_MODERATION_API_KEY` POST 到 `OPENAI_MODERATION_REVERSE_PROXY || https://api.openai.com/v1/moderations`。
- 任何 `flagged` 结果 → 以 `ErrorTypes.MODERATION` 执行 `denyRequest`。API 出错也会拒绝。
- 用于智能体聊天和 convos 路由。

**提示前缀。** `promptPrefix`（系统指令；对智能体而言，`instructions` 会与 `promptPrefix` 合并）。特殊变量 `{{current_date}}`、`{{current_user}}`、`{{iso_datetime}}`、`{{current_datetime}}` 经过 `replaceSpecialVars`（`specialVariables`，`config.ts:4455`）。设置了 `artifacts` 时会追加 artifacts 提示。

---

## 10. 语音转文字与文字转语音

**路由**（`api/server/routes/files/speech/*`，挂载在 `/api/files/speech`，带按 IP 和按用户的限流器，限流前缀为 `STT` 和 `TTS`）
- `POST /stt`：multer 单字段 `audio`，然后 `speechToText`。
- `POST /tts/manual` `{input, voice}` → `textToSpeech`，以流式返回音频。对输入有 PII 过滤。
- `POST /tts` `{messageId, runId, voice}` → `streamAudio`。它每 1.25 秒轮询一次消息缓存 / 数据库以获取新的文本分块，并增量合成。`AUDIO_RUNS` 缓存防止重复运行。
- `GET /tts/voices`。
- `GET /config/get` → `getCustomConfigSpeech`。返回 `sttExternal`、`ttsExternal` 和 `speechTab` 设置。旧版引擎名会被规范化为 `external`。

**配置。** `speech.tts` 和 `speech.stt`（`config.ts:1781-1851`）。必须恰好配置一个提供商（`isSpeechProviderConfigured`）。`allowedAddresses` 提供 SSRF 控制。

**TTS 提供商**（`services/Files/Audio/TTSService.js`）
- **openai：** `url`（默认 `https://api.openai.com/v1/audio/speech`）、apiKey、model、voices。请求体 `{input, model, voice}`，Bearer 认证。
- **azureOpenAI：** instanceName、deploymentName、apiVersion、model、voices。URL `https://{instance}.openai.azure.com/openai/deployments/{dep}/audio/speech?api-version=`，`api-key` 请求头。
- **elevenlabs：** url（默认 `https://api.elevenlabs.io/v1/text-to-speech/{voice}[/stream]`）、websocketUrl、model、voices、voice_settings、pronunciation_dictionary_locators。`xi-api-key` 请求头。
- **localai：** url、apiKey?、voices、backend。请求体 `{input, model: voice, backend}`。
- 对所有提供商，voice 必须在 `voices` 中（或 `voices` 包含 `ALL`）。

**STT 提供商**（`STTService.js`）
- **openai：** url（默认 `/v1/audio/transcriptions`）、apiKey、model。multipart 字段 `file`、`model`，可选 `language`。
- **azureOpenAI：** instanceName、deploymentName、apiVersion。25MB 上限；接受的格式为 flac、mp3、mp4、mpeg、mpga、m4a、ogg、wav、webm。

---

## 11. Python 对应方案

| 关注点 | 建议的 Python 方案 |
|---|---|
| 统一的提供商调用 | **litellm**（`litellm.acompletion(model="openai/…", api_base=, api_key=, extra_headers=, drop_params=True)`）覆盖 OpenAI、Azure（`azure/<deployment>`、`api_version`）、Anthropic、Gemini/Vertex（`vertex_ai/`）、Bedrock（`bedrock/converse/…`、`aws_region_name`、`aws_session_token`）和 OpenRouter。它的 `drop_params` 和 `additional_drop_params` 大致对应 `dropParams`；`addParams` 就是 kwargs 合并。也可以用 LangChain（`langchain-openai` 的 ChatOpenAI/AzureChatOpenAI 配合 `use_responses_api`、`langchain-anthropic` 的 ChatAnthropic、`langchain-google-genai`/`langchain-google-vertexai`、`langchain-aws` 的 ChatBedrockConverse），它最接近当前基于 LangChain 的 `llmConfig` 结构。 |
| 原生 SDK | `openai`（`AsyncOpenAI(base_url, default_headers, default_query, organization, http_client=httpx.AsyncClient(proxy=…))`、`AsyncAzureOpenAI`）、`anthropic`（`AsyncAnthropic`、`AsyncAnthropicVertex(region, project_id)`、`extra_headers={'anthropic-beta': …}`、`thinking=`、`metadata={'user_id'}`）、`google-genai`（`genai.Client(api_key=)` 或 `Client(vertexai=True, project, location)`、`types.ThinkingConfig(thinking_level/budget, include_thoughts)`、`types.Tool(google_search=…, url_context=…)`、`SafetySetting`）、`boto3` / `aioboto3` 的 `bedrock-runtime` `converse_stream`（`guardrailConfig`、`additionalModelRequestFields`；bearer 令牌通过 `AWS_BEARER_TOKEN_BEDROCK` 环境变量或自定义签名器）。 |
| 代理与 SSRF | 自定义 `httpx.AsyncClient(proxy=PROXY, follow_redirects=False, transport=…)`，配合一个自定义的解析器/transport，按 `allowedAddresses` 校验解析出的 IP。 |
| 配置 schema | 用 Pydantic v2 模型镜像 zod schema。严格的部分使用 `model_config = ConfigDict(extra='forbid')`。用 validator 解析 `${ENV}`。 |
| 请求头模板 | 一个小的解析函数：先环境变量，再用户字段、请求体字段和 OIDC 令牌，使用正则占位符；丢弃未解析的值。 |
| 模型列表缓存 | `aiocache` / Redis，TTL 120 秒，以 `sha256(baseURL:apiKey)` 为键；用 `asyncio.gather` 并行拉取，并通过 future 字典做请求内去重。 |
| token/价格表 | Python 字典加上最长子串匹配器（精确移植 `findMatchingPattern`，包括 `/` 后缀回退和“最后定义者胜出”的平局规则）。litellm 的 `model_cost` 映射可以作为表的初始数据，但为保持一致，应移植 LibreChat 自己的表和 premium 阈值。 |
| Assistants API | `openai.beta.assistants/threads/runs`（用 `runs.stream` 和 `AssistantEventHandler` 做流式）。注意 OpenAI 正在弃用 Assistants API，因此要考虑是否需要移植它。 |
| 标题 | 一个小的异步任务：`asyncio.wait_for(…, 45)`，然后写缓存，再更新数据库。对 `structured`，使用提供商的 JSON schema 或工具调用（instructor 或 pydantic 输出）。 |
| 内容审核 | `openai.moderations.create(input=[...])`。 |
| STT/TTS | `openai.audio.transcriptions.create` / `audio.speech.with_streaming_response.create`；Azure 通过 `AsyncAzureOpenAI`；ElevenLabs 通过 `elevenlabs` SDK 或 httpx；LocalAI 通过 httpx POST。以 FastAPI `StreamingResponse(media_type='audio/mpeg')` 流式输出。 |
| 用户密钥 | 用 `cryptography` 的 AES 加密存储（如果需要迁移数据，须与现有的 encrypt/decrypt 格式一致），并做 `expiresAt` 检查，抛出结构化错误。 |

### 主要文件
- **提供商初始化与配置：** `packages/api/src/endpoints/{config/providers.ts, config/endpoints.ts, config/models.ts, config/responses.ts, openai/initialize.ts, openai/config.ts, openai/llm.ts, openai/transform.ts, anthropic/*.ts, google/*.ts, bedrock/initialize.ts, custom/initialize.ts, models.ts, pricing.ts, tokenConfig.ts, keys.ts}`
- **Schema 与模型数据：** `packages/data-provider/src/{schemas.ts, config.ts, parsers.ts, models.ts, azure.ts, bedrock.ts}`
- **模型规格、请求头、token、价格：** `packages/api/src/modelSpecs/index.ts`、`packages/api/src/utils/{env.ts, headers.ts, tokens.ts, key.ts}`、`packages/data-schemas/src/methods/{tx.ts, preset.ts, key.ts}`
- **服务端配置、控制器与中间件：** `api/server/services/Config/{EndpointService.js, loadDefaultEConfig.js, loadDefaultModels.js, loadAsyncEndpoints.js, getEndpointsConfig.js}`、`api/server/controllers/{ModelController.js, EndpointController.js, TokenConfigController.js}`、`api/server/middleware/{buildEndpointOption.js, validateModel.js, moderateText.js}`
- **路由：** `api/server/routes/{models.js, endpoints.js, config.js, presets.js, keys.js, assistants/*, files/speech/*}`
- **Assistants、标题与音频服务：** `api/server/services/{AssistantService.js, Runs/*, Threads/manage.js, Endpoints/assistants/*, Endpoints/azureAssistants/*, Endpoints/agents/title.js, Endpoints/titlePolicy.js, Files/Audio/*}`、`api/server/controllers/assistants/*`、`api/server/controllers/agents/client.js`（`titleConvo` 在第 5954 行）、`packages/api/src/agents/initialize.ts`（约第 1280 行，提供商装配（wiring））
