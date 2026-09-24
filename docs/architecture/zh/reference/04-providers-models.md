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
