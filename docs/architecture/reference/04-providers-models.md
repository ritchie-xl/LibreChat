# LLM Endpoints, Providers, Models, Model Specs, Assistants, Titles & Speech

> Reference chapter for the [LibreChat backend architecture](../README.md). It was produced by
> static reading of the code at `v0.8.8-rc4` (`361553f`). Paths are relative to the repository root.
> Line numbers are approximate and move as the code changes; check the code when a detail matters.

The `@librechat/agents` SDK is an external package that is not vendored here, so its internals (such as the default title prompt) are described only through how this code calls them.

---

## 1. Endpoint types and how they map to providers

**Endpoint enum** (`packages/data-provider/src/schemas.ts:23`), `EModelEndpoint`:
- `openAI`, `azureOpenAI`, `google`, `anthropic`, `assistants`, `azureAssistants`, `agents`, `custom`, `bedrock`

**Provider enum** (`schemas.ts:36`), `Providers`, mirrors `@librechat/agents`:
- `openAI`, `anthropic`, `azureOpenAI`, `google`, `vertexai`, `bedrock`, `mistralai`, `mistral`, `deepseek`, `moonshot`, `openrouter`, `xai`

**Known custom endpoints** (`packages/data-provider/src/config.ts:3144`), `KnownEndpoints`:
- anyscale, apipie, cohere, fireworks, deepseek, moonshot, groq, helicone, huggingface, lemonade, mistral, mlx, ollama, openrouter, perplexity, shuttleai, together.ai, unify, vercel, xai
- `FetchTokenConfig` = {openrouter, helicone}. For these, `/models` responses carry pricing and context length, which are cached as a token config.

**Enabled endpoints**
- `getEnabledEndpoints()` (`packages/data-provider/src/parsers.ts:78`) reads the `ENDPOINTS` env var (a comma list).
- The default is openAI, agents, assistants, azureAssistants, azureOpenAI, google, anthropic, bedrock.
- The order of this list is the display order (`orderEndpointsConfig`).

**Provider → initializer map** (`packages/api/src/endpoints/config/providers.ts`, `providerConfigMap`):

| Provider key | Initializer |
|---|---|
| `openAI`, `azureOpenAI` | `initializeOpenAI` (`endpoints/openai/initialize.ts`) |
| `anthropic` | `initializeAnthropic` (`endpoints/anthropic/initialize.ts`) |
| `google`, `vertexai` | `initializeGoogle` (`endpoints/google/initialize.ts`) |
| `bedrock` | `initializeBedrock` (`endpoints/bedrock/initialize.ts`) |
| `xai`, `deepseek`, `moonshot`, `openrouter` | `initializeCustom` (these are "known custom providers") |
| anything else | looked up in `endpoints.custom[]` by name, then `initializeCustom`, with `overrideProvider = openAI` |

`getProviderConfig({provider, appConfig})` resolves the provider in this order:
1. Exact key.
2. Lowercase key.
3. The name of a custom endpoint.
   - Known custom providers also try a case-insensitive name match.
   - If more than one entry matches case-insensitively, it throws.
4. If the custom endpoint sets `provider: anthropic`, `overrideProvider` becomes `anthropic`, so the native `/v1/messages` client is used.

It returns `{getOptions, overrideProvider, customEndpointConfig}`.

**What `getOptions` returns:** each initializer returns an `InitializeResultBase`:
- `llmConfig`: LangChain-style client kwargs.
- `configOptions`: OpenAI SDK `ClientOptions` (baseURL, defaultHeaders, defaultQuery, fetchOptions, organization).
- Optional `tools` (provider-native tools such as web_search, googleSearch, urlContext).
- Optional `provider` override (for example `openrouter`, `vertexai`).
- Optional `endpointTokenConfig`, `useLegacyContent`, `azureOptions`.

**Where agents use it:** `packages/api/src/agents/initialize.ts:1299`.
1. `agent.provider` goes through `getProviderConfig`, and then `getOptions` is called with `model_parameters`.
2. `maxContextTokens` resolves in this order: explicit param → `getModelMaxTokens(model, providerEndpointMap[overrideProvider], endpointTokenConfig)` → `DEFAULT_MAX_CONTEXT_TOKENS`.
3. For Azure without `azureOpenAIApiInstanceName`, the provider is switched to `openAI`.

The same resolver is used for titles (`api/server/controllers/agents/client.js:5954`), summarization (`agents/run.ts:1160`) and activity labels (`agents/activityLabels/host.ts`).

**Legacy (non-agent) endpoints still in use:** `assistants` and `azureAssistants` (the OpenAI Assistants API). All chat for the other endpoints goes through the `agents` pipeline, using an "ephemeral agent" for plain model chats.

---

## 2. Building the provider config, step by step

### Shared concepts
- **Env var syntax.** `extractEnvVariable("${VAR}")` resolves env vars; `envVarRegex` detects values that did not resolve. `resolveConfigSecret` (`~/admin/secrets`) resolves admin-stored secrets.
- **User-provided keys.** A value equal to the literal `"user_provided"` means the user supplies it (`isUserProvided`, `packages/api/src/utils/common.ts:50`).
- **Where user keys live.** They are stored encrypted in the `Key` collection (`packages/data-schemas/src/methods/key.ts`, which uses encrypt/decrypt), keyed by `{userId, name: endpoint}`. `getUserKey` returns the decrypted string. `getUserKeyValues` returns decrypted JSON such as `{apiKey, baseURL}`.
- **Key expiry.** The request body's `key` field is the expiry timestamp. `checkUserKeyExpiry` throws a JSON error `{type: EXPIRED_USER_KEY}`. A missing key throws `{type: NO_USER_KEY}`, and a missing URL throws `NO_BASE_URL`.
- **Key routes** (`api/server/routes/keys.js`): `PUT /api/keys` (upsert name/value/expiresAt), `GET /api/keys?name=` (expiry), `DELETE /api/keys/:name`, `DELETE /api/keys?all=true`.
- **Proxy.** The `PROXY` env var becomes an undici `ProxyAgent` dispatcher (`getProxyDispatcher`, `utils/proxy.ts`). For AWS it is an HTTPS proxy agent.
- **SSRF protection.** When the base URL is user-provided:
  - `validateEndpointURL(url, endpoint, endpoints.allowedAddresses)` runs first.
  - Requests go through an SSRF-safe undici `Agent` with `redirect: 'error'`.
  - Configured headers are **withheld**, so user or OIDC tokens never reach a destination the user controls.
- **Header templating** (`packages/api/src/utils/env.ts`, `resolveHeaders`, `processSingleValue`). Values are resolved in this order:
  1. `${ENV}`
  2. `{{customUserVar}}`
  3. `{{LIBRECHAT_USER_<FIELD>}}`. FIELD is one of: id, name, username, email, provider, role, googleId, facebookId, openidId, samlId, ldapId, githubId, discordId, appleId, emailVerified, twoFactorEnabled, termsAccepted, termsAcceptedAt. Plus `LIBRECHAT_USER_TENANTID` / `TENANT_ID`.
  4. `{{LIBRECHAT_BODY_CONVERSATIONID|PARENTMESSAGEID|MESSAGEID}}`
  5. `{{LIBRECHAT_OPENID_ACCESS_TOKEN|ID_TOKEN|TOKEN|...}}`
  6. `{{LIBRECHAT_GRAPH_ACCESS_TOKEN}}`

  Env vars are resolved before user values, which prevents second-order injection. With `stripUnresolved`, unresolved placeholders are removed. A header whose credential placeholder cannot be resolved (for example an expired OIDC token) is omitted.
- **When headers are resolved.** Most providers resolve them at request time: `resolveConfigHeaders({llmConfig, user, tenantId, body})` (`utils/headers.ts:163`) rewrites `configuration.defaultHeaders` (OpenAI), `clientOptions.defaultHeaders` (Anthropic) and `customHeaders` (Google). Google resolves at init. Azure group headers resolve at init.
- **Header merging.** `mergeHeaders(base, override)` is case-insensitive and comma-unions `anthropic-beta` values. Precedence: `endpoints.all.headers` < endpoint headers < provider-managed headers (auth, version, beta).
- **Stream rate.** `streamRate` becomes `llmConfig._lc_stream_delay`. `endpoints.all.streamRate` wins over the endpoint's own value.

### OpenAI / Azure OpenAI (`endpoints/openai/initialize.ts`)
1. **Credentials.** `OPENAI_API_KEY` or `AZURE_API_KEY`. The base URL is `OPENAI_REVERSE_PROXY` or `AZURE_OPENAI_BASEURL` (either may be `user_provided`).
2. **Azure with YAML groups.** `mapModelToAzureConfig({modelName, modelGroupMap, groupMap})` (`packages/data-provider/src/azure.ts:138`) gives `{azureOptions: {azureOpenAIApiKey, azureOpenAIApiInstanceName, azureOpenAIApiDeploymentName, azureOpenAIApiVersion}, baseURL?, headers (additionalHeaders), serverless?}`.
   - **Serverless:** `defaultQuery: {'api-version'}`, an `api-key` header, and `useLegacyContent = true`.
3. **Azure from env only.** `getAzureCredentials()` reads the `AZURE_OPENAI_API_*` env vars. A user key is a JSON-encoded azureOptions object.
4. **addParams / dropParams.** For Azure they come from the group (`getOpenAIEndpointParameters`). Native OpenAI has none.
5. **Build the config.** `getOpenAIConfig(apiKey, {proxy, reverseProxyUrl, headers, azure, addParams, dropParams, modelOptions: {...model_parameters, model, user: userId}, customParams, directEndpoint, streaming: true})` (`openai/config.ts`) branches as follows:
   - If `customParams.defaultParamsEndpoint` is `anthropic` or `google`, it builds the native config and then `transformToOpenAIConfig` maps it to OpenAI-client form. This is for gateways that take Anthropic or Gemini params over the OpenAI protocol.
   - Otherwise it calls `getOpenAILLMConfig` (`openai/llm.ts:593`), which does the following:
     - Strips nullish and empty values. Maps `frequency_penalty`/`presence_penalty` to camelCase.
     - Applies `defaultParams` (from `customParams.paramDefinitions[].default`, only when a field is unset), then `addParams`, which overrides.
     - `web_search`:
       - OpenRouter: `modelKwargs.plugins=[{id:'web'}]`.
       - Otherwise: `useResponsesApi=true` and tool `{type:'web_search'}`.
     - Reasoning (`reasoning_effort`, `reasoning_summary`, `reasoning_mode`, `reasoning_context`):
       - Responses API: a `reasoning` object.
       - Chat Completions: a flat `reasoning_effort`.
       - OpenRouter: `reasoning: {effort}`, `include_reasoning`, and `verbosity` for adaptive Claude models.
       - `reasoningFormat` (`customParams.reasoningFormat`, or `reasoningObject` for Vercel) selects the shape.
     - Model-specific routing:
       - gpt-5.6 with reasoning → Responses API.
       - gpt-6 → Responses API, first-party only.
       - For o1/o3/gpt-5 (not gpt-5.x or -chat), sampling params are dropped.
       - For gpt-4o-search, almost everything is dropped.
       - For gpt-5+, `maxTokens` becomes `max_completion_tokens` (Chat Completions) or `max_output_tokens` (Responses).
       - DeepSeek → `includeReasoningContent` (replay reasoning on tool turns).
     - `dropParams` deletes keys from both llmConfig and modelKwargs, with special handling for reasoning subfields, `web_search` and `promptCache`.
     - Azure only:
       - Deployment name: `AZURE_USE_MODEL_AS_DEPLOYMENT_NAME` → sanitized model; otherwise it is derived from the base URL.
       - `azureOpenAIBasePath` is constructed.
       - `AZURE_OPENAI_DEFAULT_MODEL` is applied.
       - For the Responses API on Azure, the Azure fields are removed and a Responses URL is built (`constructAzureResponsesURL`) with an `api-key` header and `api-version` (default `preview`).
   - **OpenRouter or Vercel** (detected from the base URL or endpoint name): default headers `HTTP-Referer: https://librechat.ai`, `X-Title: LibreChat`, etc.; `promptCache: true` by default; the result's `provider='openrouter'`.
   - `OPENAI_ORGANIZATION` becomes `organization`.
   - `directEndpoint: true`: a custom fetch always POSTs to the exact base URL (it does not append `/chat/completions`).
6. **Stream rate.** `streamRate` from the azure, openAI or all config.

### Anthropic (`endpoints/anthropic/initialize.ts`, `llm.ts`, `helpers.ts`, `vertex.ts`)
1. **Vertex mode.** Enabled if `endpoints.anthropic.vertex(Config)` exists with `enabled !== false`, or if `ANTHROPIC_USE_VERTEX` is set.
   - It loads GCP service-account credentials (`GOOGLE_SERVICE_KEY_FILE` or the YAML `serviceKeyFile`) and uses `region` / `projectId`.
   - `getVertexDeploymentName` maps the visible model to its deployment (per model `deploymentName`, else the global one, else the model).
   - `createClient` builds an `AnthropicVertex` client.
2. **Direct mode.** `ANTHROPIC_API_KEY` (or the user key). The base URL is `ANTHROPIC_REVERSE_PROXY`, set as both `anthropicApiUrl` and `clientOptions.baseURL`.
3. **`getLLMConfig`:**
   - Pulls the system options `thinking`, `promptCache`, `promptCacheTtl` (`5m`/`1h`), `thinkingBudget`, `effort` and `thinkingDisplay` out of `modelOptions`.
   - `maxTokens` = `maxOutputTokens`, or the model default from `anthropicSettings.maxOutputTokens.reset(model)`.
   - `invocationKwargs.metadata.user_id = userId`.
4. **`configureReasoning`:**
   - Adaptive-thinking models (Opus/Sonnet 4.6+): `thinking: {type:'adaptive', display?}` and `output_config: {effort}`.
   - Older thinking models (3.7, 4.x): `thinking: {type:'enabled', budget_tokens}`, with the budget clamped to at most 90% of `max_tokens`.
   - Sonnet 5 and Opus 5: turning thinking off requires sending an explicit `{type:'disabled'}`.
   - `topP`/`topK` are sent only when thinking is inactive; some models omit sampling params entirely.
5. **Prompt caching.** If `promptCache` is on and the model supports it, `promptCache=true` (plus TTL) is passed down, and the SDK applies `cache_control`. Beta headers are added by model (for example `token-efficient-tools-2025-02-19,output-128k-2025-02-19` for 3.7), and `fine-grained-tool-streaming-2025-05-14` is always added (unless `dropParams` includes `clientOptions`).
6. **Web search.** Tool `{type:'web_search_20250305', name:'web_search'}`. Vertex also gets the beta header `web-search-2025-03-05`.
7. **Parameters and headers.** `defaultParams`, `addParams` and `dropParams` apply only to the known Anthropic params. Custom headers are merged underneath the provider headers.

### Google / Vertex AI (`endpoints/google/initialize.ts`, `llm.ts`)
1. **Credentials.**
   - `GOOGLE_KEY` is an API key (or `user_provided`).
   - Otherwise a service key is read from `GOOGLE_SERVICE_KEY_FILE` or `api/data/auth.json`.
   - The `vertexai` provider forces Vertex, with the project from `VERTEX_PROJECT_ID`, `GOOGLE_CLOUD_PROJECT`, etc.
2. **Provider choice.** A project id or `forceVertex` means Vertex; otherwise Google AI Studio (`apiKey`). Vertex `location` comes from `GOOGLE_LOC` (default `us-central1`), including multi-region endpoints.
3. **Safety settings.** `getSafetySettings` builds `GOOGLE_SAFETY_*` env thresholds for five harm categories. `GOOGLE_EXCLUDE_SAFETY_SETTINGS` disables them.
4. **Thinking.**
   - Gemini 3+ and Gemma 4+: `thinkingConfig: {thinkingLevel, includeThoughts: true}`.
   - Older models: `thinkingBudget`.
5. **Tools.** `web_search` becomes `{googleSearch:{}}`. `url_context` becomes `{urlContext:{}}` (Gemini 2.5+ only).
6. **Proxy and auth.** `GOOGLE_REVERSE_PROXY` becomes `baseUrl`. `GOOGLE_AUTH_HEADER` puts the key in an `Authorization` header via `customHeaders`.

### Bedrock (`endpoints/bedrock/initialize.ts`)
1. **Credentials (env).** `BEDROCK_AWS_ACCESS_KEY_ID`, `_SECRET_ACCESS_KEY`, `_SESSION_TOKEN`, `_BEARER_TOKEN` (a Bedrock API key) and `_PROFILE`. Any of these can be `user_provided`; the user key is JSON `{accessKeyId, secretAccessKey, sessionToken?, bearerToken?}`.
2. **Other env.** `BEDROCK_AWS_DEFAULT_REGION` and `BEDROCK_REVERSE_PROXY`.
3. **Parsing params.** `bedrockInputParser` then `bedrockOutputParser` (`packages/data-provider/src/bedrock.ts`). The output parser maps thinking, effort and promptCache into `additionalModelRequestFields` and `anthropic_beta`.
4. **YAML options.**
   - `guardrailConfig` (identifier, version, trace, streamProcessingMode).
   - `inferenceProfiles` (model → ARN, set as `applicationInferenceProfile`).
5. **Client construction.** A custom `BedrockRuntimeClient` is built when there is a proxy or a bearer token (`authSchemePreference: ['httpBearerAuth']`). Otherwise credentials, profile or `endpointHost` are passed through to `ChatBedrockConverse`.

### Custom endpoints (`endpoints/custom/initialize.ts`)
1. Resolve `apiKey` and `baseURL` with `${ENV}`. If either still matches `envVarRegex`, throw "Missing API Key" / "Missing Base URL".
2. Handle user-provided key and/or URL. A user-provided URL triggers `validateEndpointURL`.
3. **Token config.**
   - A static YAML `tokenConfig` is used directly (`cacheRead`/`cacheWrite` map to `read`/`write`).
   - For endpoints in `FetchTokenConfig`, the cached config is read, or `fetchModels` is called, which populates the cache from `/models` pricing.
   - Cache key: `getTokenConfigKey` scopes it to endpoint, endpoint:user, or a tenant hash.
4. **`provider: anthropic`.** Uses Anthropic `getLLMConfig` with the base URL as `reverseProxyUrl`.
5. **Otherwise.** `getOpenAIConfig(apiKey, {reverseProxyUrl: baseURL, headers, addParams, dropParams, customParams, directEndpoint, titleConvo, titleModel, titleMethod, titleMessageRole, streamRate, modelDisplayLabel, ...})`, with `useLegacyContent = true`.

---

## 3. librechat.yaml `endpoints` schema (`packages/data-provider/src/config.ts`)

The root is `endpoints` (strict, line ~3067):
- `allowedAddresses`
- `all`: `baseEndpointSchema` minus `baseURL`; global overrides that win at read sites
- `openAI`, `google`: `baseEndpointSchema`
- `anthropic`: `anthropicEndpointSchema`
- `azureOpenAI`: `azureEndpointSchema`
- `assistants`, `azureAssistants`: `assistantEndpointSchema`
- `agents`: `agentsEndpointSchema`
- `custom`: array of `endpointSchema.partial()`
- `bedrock`: `bedrockEndpointSchema`

**`baseEndpointSchema` (line 672)**
- Basic: `streamRate`, `baseURL`, `headers` (record).
- Titles: `titlePrompt`, `titleModel` (`current_model` means use the chat model), `titleConvo`, `titleMethod` (`completion`|`functions`|`structured`), `titleEndpoint`, `titlePromptTemplate`, `titleTiming` (`immediate`|`final`).
- Agent UI labels: `activityLabel`, `activityModel`, `activityEndpoint`, `activityPrompt`, `activityMaxPerRun`, `activityCharLimit`, `activityPhase*`, `reasoningLabel*`.
- `maxToolResultChars`.

**`endpointSchema` (custom, line 1606)** extends the base with:
- `name`: must not be an `EModelEndpoint` value.
- `apiKey`, `apiKeyPreview`, `baseURL`.
- `models: {default: (string|{name, description})[] (min 1), fetch?, userIdQuery?}`.
- `iconURL`, `modelDisplayLabel`.
- `provider?: 'anthropic'`.
- `headers`.
- `addParams`: record; `web_search` must be boolean.
- `dropParams`: string[].
- `customParams` (strict):
  - `defaultParamsEndpoint` (default `custom`), `reasoningFormat`, `reasoningKey`.
  - `includeReasoningContent`, `includeReasoningHistory`.
  - `paramDefinitions[]`: UI setting definitions with key, type, default, range, options, component, etc.
- `directEndpoint`, `titleMessageRole` (`system`|`user`|`assistant`).
- `tokenConfig: Record<model, {prompt, completion, context, cacheRead?, cacheWrite?}>`, with rates per 1M tokens.

**`azureEndpointSchema` (line 1668)**
- `groups[]` (min 1). Each group has: `group`, `apiKey`, `serverless?`, `instanceName?`, `deploymentName?`, `version?`, `baseURL?`, `additionalHeaders?`, `addParams?`, `dropParams?`, `assistants?`.
  - `models: Record<modelName, true | {deploymentName?, version?, assistants?}>`.
- Top level: `assistants?`, plus a `.pick()` of the title, activity and label fields (not all base fields).
- Validated into `{modelNames, groupMap, modelGroupMap, assistantModels, assistantGroups}` (`validateAzureGroups`, `azure.ts:13`).

**`anthropicEndpointSchema` (line 1766)**
- Base fields, plus `models?: string[]`.
- `vertex?: {enabled?, projectId?, region (default us-east5), serviceKeyFile?, deploymentName?, models?: string[] | Record<model, true | {deploymentName?}>}`. At runtime it is exposed as `vertexConfig` with `modelNames` and `modelDeploymentMap`.

**`bedrockEndpointSchema` (line 766)**
- Base fields, plus `availableRegions?`, `models?`, `guardrailConfig?`, `inferenceProfiles?`.

**`assistantEndpointSchema` (line 783)**
- Base fields, plus `disableBuilder`, `pollIntervalMs`, `timeoutMs`, `version` (default 2).
- `supportedIds` / `excludedIds` / `privateAssistants`.
- `retrievalModels`, `capabilities` (code_interpreter, image_vision, retrieval, actions, tools).
- `apiKey`, `models{default, fetch, userIdQuery}`, `headers`.

**`agentsEndpointSchema` (line 1267)** is large: capabilities, allowedProviders, statefulCodeSessions, toolApproval, and more. It is outside this slice.

**Env-only endpoint enablement** (`api/server/services/Config/EndpointService.js`)
- `generateConfig(key, baseURL)` gives `{userProvide, userProvideURL}`, or `false` when there is no key.
- Assistants endpoints also get `retrievalModels`, `capabilities` and `version`.
- Google is enabled if `GOOGLE_KEY` or a service key file exists (`loadAsyncEndpoints.js`).

---

## 4. Model list resolution and caching

**Routes**
- `GET /api/models` (`routes/models.js` → `controllers/ModelController.js`) returns `{...loadDefaultModels(req), ...loadConfigModels(req)}`, a map from endpoint name to `string[]`.
- `GET /api/endpoints` (`routes/endpoints.js` → `EndpointController` → `createEndpointsConfigService`, `packages/api/src/endpoints/config/endpoints.ts`) returns the default env-based endpoints config merged with `loadCustomEndpointsConfig(appConfig.endpoints.custom)`. That config carries `userProvide`, `userProvideURL`, `order`, `iconURL`, `modelDisplayLabel`, and similar.
  - Azure: `{userProvide:false}` when YAML exists.
  - Adds `responsesApiRouting` per model for openAI and azure (`config/responses.ts`).
  - Adds `azureAssistants` when `azureOpenAI.assistants` is set.
  - Assistants: version, retrievalModels, disableBuilder, capabilities.
  - Agents: capabilities, allowedProviders, statefulCodeSessions (filtered), maxSubagents.
  - Bedrock: availableRegions and the `userProvide*` flags.
  - The result is ordered.
- `GET /api/endpoints/token-config` (`TokenConfigController`) returns context windows, plus pricing if `interface.contextCost` is on.

**Default models** (`api/server/services/Config/loadDefaultModels.js`; functions in `packages/api/src/endpoints/models.ts`)
- **OpenAI.**
  1. If `OPENAI_MODELS` is set, use that list.
  2. Else if the key is user-provided, use `defaultModels`.
  3. Else fetch `GET {OPENAI_REVERSE_PROXY or https://api.openai.com/v1}/models`. On api.openai.com the list is filtered by `/(text-davinci-003|gpt-|o\d+|chat-latest)/` and excludes `audio`/`realtime`.
- **Azure.** `AZURE_OPENAI_MODELS` env (the YAML `modelNames` override it, via config models).
- **Assistants.** `ASSISTANTS_MODELS`, or a fetch from `ASSISTANTS_BASE_URL`.
- **Anthropic.** Vertex `modelNames` first, then `ANTHROPIC_MODELS`, then a fetch of `GET {base}/models` with `x-api-key` and `anthropic-version: ANTHROPIC_VERSION || 2023-06-01`.
- **Google.** `GOOGLE_MODELS` or the defaults.
- **Bedrock.** `BEDROCK_AWS_MODELS` or the defaults. YAML `bedrock.models` overrides.
- The built-in defaults are in `packages/data-provider/src/config.ts` (`defaultModels`, `sharedOpenAIModels`, `sharedAnthropicModels`, `bedrockModels`).

**Config/custom models** (`packages/api/src/endpoints/config/models.ts`, `createLoadConfigModels`)
- Azure `modelNames`, `azureAssistants` `assistantModels`, and bedrock `models` come first.
- For each custom endpoint with baseURL, apiKey, name and `models`:
  - **`fetch: true` with an admin key:** fetches are deduped per request by `baseURL__apiKey__sha256(headers)`, the token config is copied to sibling endpoints, and the result falls back to `models.default` when empty.
  - **`fetch: true` with a user-provided key or URL:** loads the user's key values and fetches with `skipCache`. Headers are not forwarded when the URL is user-supplied.
  - **Otherwise:** `models.default` (names only).
- Endpoint names are normalized: `ollama` is lowercased.

**`fetchModels`** (`models.ts`)
- **Headers.** `Authorization: Bearer <apiKey>` unless the configured headers already provide one. Headers are template-resolved against the user. `OpenAI-Organization` is added for openai URLs.
- **Ollama.** Names starting with `ollama` try `GET {base}/api/tags` first.
- **Request.** 5s timeout. `?user=<id>` when `userIdQuery`. An SSRF-safe agent when the URL is user-provided.
- **Token config.** If the response matches the OpenRouter-style `inputSchema` (`data[].{id, pricing.{prompt, completion}, context_length}`), `processModelData` converts pricing to per-1M rates and stores it in `tokenConfigCache` under `tokenKey`, plus a model-cache-scoped key for backfill.
- **Cache.** `standardCache(CacheKeys.MODEL_QUERIES)` keyed by `sha256(baseURL:apiKey)[:32]`, TTL **2 minutes**. It is skipped when user-scoped headers are forwarded, or when `userIdQuery` is set with a user.

**Model validation** (`api/server/middleware/validateModel.js`)
- Checks the model against the regex `^[a-zA-Z0-9][a-zA-Z0-9_.:/@+-]*$`, maximum 256 characters.
- Skipped when the endpoint is `userProvide`.
- Otherwise the model must be in `modelsConfig[resolveModelCatalogKey(endpoint)]`. If it is not, an `ILLEGAL_MODEL_REQUEST` violation is logged and the request is denied.

---

## 5. Tokens, pricing and context tables

**Context windows** (`packages/api/src/utils/tokens.ts`)
- **Tables.** Static per-family maps: openAIModels, mistral, cohere, google, anthropic, deepseek, moonshot, meta, qwen, amazon, bedrock, xAI, and `aggregateModels` (the union).
- **`maxTokensMap` by endpoint:** azureOpenAI → openAI; openAI, agents and custom → aggregate; google; anthropic; bedrock.
- **Output limits.** `maxOutputTokensMap` and `modelMaxOutputs` hold output limits.
- **Lookup** (`getModelTokenValue`):
  1. Exact key.
  2. `findMatchingPattern`: the longest key that is a substring of the lowercased model name. For `a/b` names it tries the full name, then the part after the last `/`. On same-length ties, the last-defined key wins.
  3. `system_default`.
- **`getModelMaxTokens(model, endpoint, endpointTokenConfig)`.** The endpoint token config override comes first, then Anthropic 1M-context special cases, then the table.
- **`getModelMaxOutputTokens`.** Similar, with Opus 5.5 and Sonnet 4.6+ special cases.
- **Conversation override.** `maxContextTokens` on the conversation or preset overrides everything.

**Pricing** (`packages/data-schemas/src/methods/tx.ts`)
- **Tables** (USD per 1M tokens):
  - `tokenValues`: model key → `{prompt, completion}`.
  - `cacheTokenValues`: key → `{write, read}`.
  - `premiumTokenValues` and `premiumCacheTokenValues`: `{threshold, prompt, completion}`. Long-context tiers apply when the input token count exceeds the threshold (for example gpt-5.4 above 272k, gemini-3.1 above 200k).
  - `bedrockValues`.
  - `defaultRate = 6`.
- **`getValueKey(model, endpoint)`.** A pattern match on `tokenValues`, with legacy gpt-3.5/gpt-4 buckets (`4k`, `16k`, `8k`, `32k`, `gpt-4-1106`).
- **`getMultiplier({valueKey, model, endpoint, tokenType, inputTokenCount, endpointTokenConfig})`.** Order: the endpointTokenConfig per-model rate → the premium rate → `tokenValues` → `defaultRate`.
- **`getCacheMultiplier`.** Same order, but returns `null` when there is no cache price.
- **Where these are used.**
  - `buildTokenConfigMap` / `resolveTokenConfigMap` (`packages/api/src/endpoints/pricing.ts`, `tokenConfig.ts`) produce `{endpoint: {model: {context, prompt?, completion?, cacheWrite?, cacheRead?}}}` for the client.
  - Balance and transaction billing use the same multipliers (tokens × rate).
- **Cache keys** (`packages/api/src/endpoints/keys.ts`). Scoped token-config keys are `\0token-config:v2\0{tenant|tenant-user}\0sha256(parts)`.

---

## 6. modelSpecs semantics

**Schema** (`packages/data-provider/src/models.ts`), `specsConfigSchema`:
- `enforce` (default false), `prioritize` (default true), `list[]`, `addedEndpoints[]`.

**Each `TModelSpec`**
- `name` (the id), `label`, and `preset`. The preset is `tModelSpecPresetSchema`: preset fields minus ids, user and DB fields.
- Display: `order`, `default`, `softDefault`, `description`, `group` (an endpoint name nests the spec under that endpoint; any other string makes a custom group), `groupIcon`, `showIconInMenu`, `showIconInHeader`, `showOnLanding`, `conversation_starters`, `showInMenu` (false hides it from the menu and startup config but still allows use by name), `iconURL`, `authType`, `hideBadgeRow`.
- Agent tool toggles: `webSearch`, `fileSearch`, `executeCode`, `memory`, `askUserQuestion`, `runInBackground`, `describeIntent`, `artifacts`, `mcpServers[]`, `skills`, `subagents{enabled, allowSelf, shareFiles, agent_ids}`.

**Endpoint inference.** `resolveModelSpecEndpoint` uses `preset.endpoint`; if it is absent and `agent_id` is set, it infers `agents`. `materializeModelSpecEndpoints` writes this back once, at config load.

**Server application** (`api/server/middleware/buildEndpointOption.js`, helpers in `packages/api/src/modelSpecs/index.ts`)
- **`enforce: true`.**
  1. The request must carry `spec`. Otherwise the error is "No model spec selected".
  2. The spec must exist ("Invalid model spec").
  3. The spec's endpoint must equal the request endpoint ("Model spec mismatch").
  4. `applyModelSpecPreset(includePresetDefaults: true)` replaces the body with the preset, keeping only `chatProjectId` from the request and setting `spec`.
- **Not enforced, but `spec` is present.** The request is merged with the spec. Private preset fields fill in only when the request lacks them.
- **Re-parsing.** The merged result goes back through `parseCompactConvo`, and `iconURL` comes from the spec.
- **Private fields.** `promptPrefix`, `instructions`, `additional_instructions`, `system`, `context` and `examples` are removed from the client payload (`sanitizeModelSpecs`). For non-agent endpoints, `promptPrefix` special variables (`{{current_date}}`, `{{current_user}}`, `{{iso_datetime}}`, `{{current_datetime}}`) are resolved server-side, and the user's name is content-filtered.
- **Startup config.** `modelSpecs: sanitizeModelSpecs(excludeHiddenModelSpecs(appConfig.modelSpecs))` (`routes/config.js:303`); `skills` and `subagents.agent_ids` are also removed.

---

## 7. Assistants API (OpenAI and Azure), brief

**Routes** (`api/server/routes/assistants/index.js`, mounted at `/api/assistants`, with JWT, ban check and config middleware)
- `/v1/`, `/v2/`: CRUD on assistants, proxied to OpenAI: `POST /`, `GET /:id`, `PATCH /:id`, `DELETE /:id`, `GET /` (list), `POST /avatar/:assistant_id`.
- Sub-routers `/actions` (`POST /:assistant_id`, `DELETE /:assistant_id/:action_id/:model`), `/tools`, `/documents`.
- `/v1/chat`, `/v2/chat`: `POST /` with middleware `filterMessageContent → validateModel → buildEndpointOption → validateAssistant → validateConvoAccess → guardSubagentThreadTurn → chatController`, plus `POST /abort`.

**Clients**
- `services/Endpoints/assistants/initalize.js`: the `openai` SDK with `ASSISTANTS_API_KEY`, `ASSISTANTS_BASE_URL` (either may be user-provided), header `OpenAI-Beta: assistants=v{version}`, PROXY and org.
- `azureAssistants/initialize.js`: maps the model to the Azure group, builds the URL `https://{instance}.openai.azure.com/openai`, adds `api-version` as a query param, the `api-key` header and resolved group headers; model = deployment.

**Chat flow** (`controllers/assistants/chatV2.js`)
1. Balance check.
2. `initThread` (create or reuse the OpenAI thread, add the message with attachments).
3. `saveUserMessage`.
4. `createRun` with `instructions` / `additional_instructions` (promptPrefix, optional current datetime).
5. Either `StreamRunManager` (SSE streaming of run events, tool calls, and `submitToolOutputs` for function tools / actions) or polling (`RunManager`, `waitForRun`, `runAssistant` in `services/AssistantService.js`).
6. `syncMessages` / `processMessages` (`services/Threads/manage.js`) convert the thread messages, handling file citations and image files.
7. `saveAssistantMessage`.
8. `addTitle`.
9. `recordUsage` from `run.usage`.

`filterAssistants` applies `privateAssistants` (`metadata.author`), `supportedIds` and `excludedIds`. `packages/api/src/assistants/protection.ts` applies PII/content filtering to assistant thread messages and files.

**Title for assistants** (`services/Endpoints/assistants/title.js`)
- A hardcoded `gpt-3.5-turbo` chat completion: "Please generate a concise title (max 40 characters)...", temperature 0.7, max_tokens 20.
- On error it falls back to the text truncated to 40 characters.

---

## 8. Conversation and preset parameter schema

**`tConversationSchema`** (`packages/data-provider/src/schemas.ts:1115`)
- **Identity and state:** conversationId, endpoint, endpointType, title (default "New Chat"), user, messages[], tags[], chatProjectId, createdAt, updatedAt, isArchived, archivedAt, pinned, isShared, expiredAt, isTemporary, parentMessageId.
- **Code:** codeApprovalMode, codeEnvironmentMode, codeWorkspaces[].
- **Common model params:** model, modelLabel, userLabel, promptPrefix, temperature, topP, topK, top_p, frequency_penalty, presence_penalty, maxOutputTokens, maxContextTokens, max_tokens, maxTokens, stop[], stream, disableStreaming.
- **Anthropic:** promptCache, promptCacheTtl(`5m`|`1h`), system, thinking, thinkingBudget, effort, thinkingDisplay.
- **Google:** thinkingLevel, context, examples[{input, output}], url_context.
- **OpenAI:** reasoning_effort, reasoning_summary, reasoning_mode, reasoning_context, verbosity, useResponsesApi, imageDetail.
- **Shared:** web_search, artifacts, resendFiles, file_ids, fileTokenLimit.
- **Assistants and agents:** assistant_id, instructions, additional_instructions, append_current_datetime, agent_id, subagentThread.
- **Bedrock:** region, additionalModelRequestFields.
- **UI:** greeting, spec, iconURL, tools.
- **Other:** presetOverride.
- **Deprecated:** chatGptLabel, resendImages.

**`tPresetSchema`.** The conversation schema minus conversationId, chatProjectId, timestamps and title, plus `presetId`, `title`, `defaultPreset`, `order`, and `endpoint` (any string).

**Per-endpoint pick schemas** (`parsers.ts:42`, `endpointSchemas`). These whitelist the parameters accepted per endpoint and strip nullish values:
- **openAI, azureOpenAI, custom (`openAISchema`):** model, modelLabel, promptPrefix, temperature, top_p, presence/frequency_penalty, resendFiles, artifacts, imageDetail, stop, max_tokens, reasoning_*, verbosity, useResponsesApi, web_search, disableStreaming, fileTokenLimit, maxContextTokens, spec, iconURL, greeting.
- **openrouter:** the above plus promptCache and promptCacheTtl.
- **google:** model, modelLabel, promptPrefix, examples, temperature, maxOutputTokens, topP, topK, thinking, thinkingBudget, thinkingLevel, web_search, url_context, and more.
- **anthropic:** model, modelLabel, promptPrefix, temperature, maxOutputTokens, topP, topK, promptCache, promptCacheTtl, thinking, thinkingBudget, effort, thinkingDisplay, web_search, stop, stream, and more.
- **bedrock:** `bedrockInputSchema`.
- **agents:** `compactAgentsSchema`.
- **assistants:** `assistantSchema`.

`parseCompactConvo({endpoint, endpointType, conversation, defaultParamsEndpoint})` selects the schema using the custom endpoint's `customParams.defaultParamsEndpoint`.

**Defaults** (`openAISettings`, `anthropicSettings`, `googleSettings`, `agentsSettings` in `schemas.ts:439+`): for example OpenAI temperature 1, top_p 1, penalties 0; Anthropic default model and promptCache true, `maxOutputTokens.reset(model)`.

**Preset routes** (`api/server/routes/presets.js`; storage in `packages/data-schemas/src/methods/preset.ts`)
- `GET /api/presets` returns the user's presets, sorted by `order` then `updatedAt` descending, and projected through content filters.
- `POST /api/presets` (with a content filter) runs `savePreset`:
  - Upsert by `{presetId, user}`; `presetId` defaults to a UUID; `newPresetId` renames it.
  - `tools` are normalized to pluginKey strings.
  - `defaultPreset: true` sets `order = 0` and unsets the previous default. `defaultPreset: false` unsets both.
- `POST /api/presets/delete {presetId?}` deletes one preset, or all of the user's presets.

---

## 9. Title generation, moderation and prompt prefix

**Agents title** (`api/server/services/Endpoints/agents/title.js`, `client.js:5954 titleConvo`)
- **Gating.** Env `TITLE_CONVO` (default true). The endpoint's `titleConvo !== false`. Not a temporary chat.
- **Timing.** `resolveTitleTiming` (`providers.ts`): `endpoints.all.titleTiming`, then endpoint candidates, then the custom config; default `immediate`, which runs in parallel from the first user message. `final` runs after the response.
- **Endpoint config.** `endpoints.all`, else `endpoints[endpoint]`, else the custom config.
- **Provider and model.** `titleEndpoint` can switch the provider/credentials. `titleModel` overrides the model (`current_model` keeps it). The options are rebuilt through `getProviderConfig().getOptions`, max-token params are removed, and headers are resolved.
- **Generation.** `run.generateTitle({provider, clientOptions, inputText, contentParts, titleMethod, titlePrompt, titlePromptTemplate})` in `@librechat/agents`.
  - `completion` = a plain prompt; `functions` / `structured` = tool or JSON output (Google gets `json: true`).
  - Usage is recorded as `context: 'title'` (billed).
  - The result goes through `sanitizeTitle`.
- **Timeout and storage.** 45s timeout. The result goes through `resolveConversationTitle`, then the cache `GEN_TITLE` key `${userId}-${convoId}` (TTL 120s), then `onTitleGenerated` (SSE event), then `saveConvo({title}, noUpsert)`. The title is discarded if the stream was superseded.

**titlePolicy** (`api/server/services/Endpoints/titlePolicy.js` → `packages/api/src/protection/title.ts`)
- If `filters.conversationTitles.pii` is configured and the title violates it, the fallback `"New Chat"` is used (if the fallback is allowed); otherwise the title is null (not saved).

**Moderation** (`api/server/middleware/moderateText.js`)
- Runs when `OPENAI_MODERATION` is enabled.
- Gathers `text`/`answer`, ask-user answers, quoted text and tool-approval decision texts.
- POSTs to `OPENAI_MODERATION_REVERSE_PROXY || https://api.openai.com/v1/moderations` with `Bearer OPENAI_MODERATION_API_KEY`.
- Any `flagged` result → `denyRequest` with `ErrorTypes.MODERATION`. An API error also denies.
- Used on agents chat and convos routes.

**Prompt prefix.** `promptPrefix` (the system instructions; for agents, `instructions` is combined with `promptPrefix`). Special variables `{{current_date}}`, `{{current_user}}`, `{{iso_datetime}}`, `{{current_datetime}}` go through `replaceSpecialVars` (`specialVariables`, `config.ts:4455`). Artifacts prompts are appended when `artifacts` is set.

---

## 10. Speech-to-text and text-to-speech

**Routes** (`api/server/routes/files/speech/*`, mounted at `/api/files/speech`, with per-IP and per-user limiters from the rate-limit prefixes `STT` and `TTS`)
- `POST /stt`: a multer single field `audio`, then `speechToText`.
- `POST /tts/manual` `{input, voice}` → `textToSpeech`, which streams audio. It has a PII filter on the input.
- `POST /tts` `{messageId, runId, voice}` → `streamAudio`. It polls the message cache / DB every 1.25s for new text chunks and synthesizes them incrementally. `AUDIO_RUNS` cache guards against duplicate runs.
- `GET /tts/voices`.
- `GET /config/get` → `getCustomConfigSpeech`. Returns `sttExternal`, `ttsExternal` and the `speechTab` settings. Legacy engine names are normalized to `external`.

**Configuration.** `speech.tts` and `speech.stt` (`config.ts:1781-1851`). Exactly one provider must be configured (`isSpeechProviderConfigured`). `allowedAddresses` gives SSRF control.

**TTS providers** (`services/Files/Audio/TTSService.js`)
- **openai:** `url` (default `https://api.openai.com/v1/audio/speech`), apiKey, model, voices. Body `{input, model, voice}`, Bearer auth.
- **azureOpenAI:** instanceName, deploymentName, apiVersion, model, voices. URL `https://{instance}.openai.azure.com/openai/deployments/{dep}/audio/speech?api-version=`, `api-key` header.
- **elevenlabs:** url (default `https://api.elevenlabs.io/v1/text-to-speech/{voice}[/stream]`), websocketUrl, model, voices, voice_settings, pronunciation_dictionary_locators. `xi-api-key` header.
- **localai:** url, apiKey?, voices, backend. Body `{input, model: voice, backend}`.
- For all providers, the voice must be in `voices` (or `voices` contains `ALL`).

**STT providers** (`STTService.js`)
- **openai:** url (default `/v1/audio/transcriptions`), apiKey, model. Multipart `file`, `model`, optional `language`.
- **azureOpenAI:** instanceName, deploymentName, apiVersion. 25MB limit; accepted formats flac, mp3, mp4, mpeg, mpga, m4a, ogg, wav, webm.

---

## 11. Python equivalents

| Concern | Suggested Python approach |
|---|---|
| Unified provider calls | **litellm** (`litellm.acompletion(model="openai/…", api_base=, api_key=, extra_headers=, drop_params=True)`) covers OpenAI, Azure (`azure/<deployment>`, `api_version`), Anthropic, Gemini/Vertex (`vertex_ai/`), Bedrock (`bedrock/converse/…`, `aws_region_name`, `aws_session_token`) and OpenRouter. Its `drop_params` and `additional_drop_params` roughly match `dropParams`; `addParams` is kwargs merging. Alternatively use LangChain (`langchain-openai` ChatOpenAI/AzureChatOpenAI with `use_responses_api`, `langchain-anthropic` ChatAnthropic, `langchain-google-genai`/`langchain-google-vertexai`, `langchain-aws` ChatBedrockConverse), which is closest to the current LangChain-based `llmConfig` shapes. |
| Native SDKs | `openai` (`AsyncOpenAI(base_url, default_headers, default_query, organization, http_client=httpx.AsyncClient(proxy=…))`, `AsyncAzureOpenAI`), `anthropic` (`AsyncAnthropic`, `AsyncAnthropicVertex(region, project_id)`, `extra_headers={'anthropic-beta': …}`, `thinking=`, `metadata={'user_id'}`), `google-genai` (`genai.Client(api_key=)` or `Client(vertexai=True, project, location)`, `types.ThinkingConfig(thinking_level/budget, include_thoughts)`, `types.Tool(google_search=…, url_context=…)`, `SafetySetting`), `boto3` / `aioboto3` `bedrock-runtime` `converse_stream` (`guardrailConfig`, `additionalModelRequestFields`; bearer token via the `AWS_BEARER_TOKEN_BEDROCK` env var or a custom signer). |
| Proxy and SSRF | Custom `httpx.AsyncClient(proxy=PROXY, follow_redirects=False, transport=…)`, with a custom resolver/transport that validates resolved IPs against `allowedAddresses`. |
| Config schema | Pydantic v2 models mirroring the zod schemas. Use `model_config = ConfigDict(extra='forbid')` for strict sections. Resolve `${ENV}` with a validator. |
| Header templating | A small resolver function: env first, then user fields, body fields and OIDC tokens, using regex placeholders; drop unresolved values. |
| Model list cache | `aiocache` / Redis with a 120s TTL, keyed by `sha256(baseURL:apiKey)`; `asyncio.gather` for parallel fetches, with per-request dedupe via a dict of futures. |
| Token/pricing tables | Python dicts plus a longest-substring matcher (port `findMatchingPattern` exactly, including the `/` suffix fallback and last-wins ties). litellm's `model_cost` map can seed the tables, but LibreChat's own tables and premium thresholds should be ported for parity. |
| Assistants API | `openai.beta.assistants/threads/runs` (streaming with `runs.stream` and an `AssistantEventHandler`). Note that OpenAI is deprecating the Assistants API, so consider whether to port it at all. |
| Titles | A small async task: `asyncio.wait_for(…, 45)`, then a cache write, then a DB update. For `structured`, use provider JSON-schema or tool calling (instructor or pydantic output). |
| Moderation | `openai.moderations.create(input=[...])`. |
| STT/TTS | `openai.audio.transcriptions.create` / `audio.speech.with_streaming_response.create`; Azure via `AsyncAzureOpenAI`; ElevenLabs via the `elevenlabs` SDK or httpx; LocalAI via httpx POST. Stream to FastAPI `StreamingResponse(media_type='audio/mpeg')`. |
| User keys | Store encrypted with `cryptography` AES (matching the existing encrypt/decrypt format if data must be migrated), with an `expiresAt` check that raises structured errors. |

### Main files
- **Provider init and config:** `packages/api/src/endpoints/{config/providers.ts, config/endpoints.ts, config/models.ts, config/responses.ts, openai/initialize.ts, openai/config.ts, openai/llm.ts, openai/transform.ts, anthropic/*.ts, google/*.ts, bedrock/initialize.ts, custom/initialize.ts, models.ts, pricing.ts, tokenConfig.ts, keys.ts}`
- **Schemas and model data:** `packages/data-provider/src/{schemas.ts, config.ts, parsers.ts, models.ts, azure.ts, bedrock.ts}`
- **Model specs, headers, tokens, pricing:** `packages/api/src/modelSpecs/index.ts`, `packages/api/src/utils/{env.ts, headers.ts, tokens.ts, key.ts}`, `packages/data-schemas/src/methods/{tx.ts, preset.ts, key.ts}`
- **Server config, controllers and middleware:** `api/server/services/Config/{EndpointService.js, loadDefaultEConfig.js, loadDefaultModels.js, loadAsyncEndpoints.js, getEndpointsConfig.js}`, `api/server/controllers/{ModelController.js, EndpointController.js, TokenConfigController.js}`, `api/server/middleware/{buildEndpointOption.js, validateModel.js, moderateText.js}`
- **Routes:** `api/server/routes/{models.js, endpoints.js, config.js, presets.js, keys.js, assistants/*, files/speech/*}`
- **Assistants, titles and audio services:** `api/server/services/{AssistantService.js, Runs/*, Threads/manage.js, Endpoints/assistants/*, Endpoints/azureAssistants/*, Endpoints/agents/title.js, Endpoints/titlePolicy.js, Files/Audio/*}`, `api/server/controllers/assistants/*`, `api/server/controllers/agents/client.js` (`titleConvo` at line 5954), `packages/api/src/agents/initialize.ts` (line ~1280, provider wiring)
