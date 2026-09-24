# @librechat/agents：消息格式化与上下文管理

> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) `v3.9.3`（`2d5653d`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。

本报告涵盖 SDK 中消息格式化与上下文管理这一部分，作为 Python 重新实现的参考。内容基于阅读代码，而非实际运行。

**主要组成：**
- `src/messages/format.ts`：`formatAgentMessages` 把存储的 LibreChat 消息转换为 LangChain 消息。
- `src/messages/prune.ts`：`createPruneMessages` 负责预算计算、校准、淡化（fading）、裁剪和孤立块修复。
- `src/messages/fading.ts`：锁存式（latched）截断档位。
- `src/summarization/node.ts`：压缩（compaction）节点。
- `src/utils/tokens.ts`：token 计数。

**粘合代码：**
- `src/graphs/Graph.ts` 约第 3290–3500 行装配（wiring）裁剪器和摘要触发逻辑。
- `src/agents/AgentContext.ts` 处理摘要注入、指令 token 和失败上限。

**过时文档：** `docs/summarization-behavior.md` 部分内容已过时。它描述的是“对整个对话做完全压缩，不保留任何消息”。现在的代码默认保留一段最近的尾部（2 个由用户发起的轮次），并写入一个 `coverage` 锚点。两者不一致时，以代码为准。

---

## 1. 存储的内容片段 → LangChain 消息

### 1.1 输入模型
`TPayload = Array<Partial<TMessage>>`（`src/types/stream.ts`），其中 `TMessage = { role?, content?: string | MessageContentComplex[], messageId?/id?, ... }`。

内容片段的 `type` 值来自 `ContentTypes`（`src/common/enum.ts`）：

| type | 形态 | 含义 |
|---|---|---|
| `text` | `{type, text, tool_call_ids?: string[], phase?}` | 可见文本。`tool_call_ids` 标记位于工具调用之前的文本 |
| `think` | `{type, think}` | 经 LibreChat 规范化的推理 |
| `thinking` / `reasoning` / `reasoning_content` / `redacted_thinking` | 提供商的推理形态 | 按推理处理（`src/messages/reasoningTypes.ts`） |
| `tool_call` | `{type, tool_call: {id, name, args: string\|object, output?, outcome?, auth?, expires_at?}}` | 调用和结果一起持久化在同一个片段中 |
| `image_url`、`image_file`、文档/媒体 | 提供商的块 | 原样透传 |
| `summary` | `SummaryContentBlock {content:[{type:'text',text}], tokenCount, coverage?:{retainedFromMessageId}, boundary?:{messageId,contentIndex}, summaryVersion, model, provider, createdAt}` | 压缩检查点 |
| `steer` | `{type, steer: string, media?: ContentPart[]}` | 运行中途的用户发言，存储在助手消息内部 |
| `agent_update`、`error` | | 仅供 UI；被丢弃 |
| `activity_label` | `{activity_label, activity_label_type, activity_start_index, ...}` | 仅供 UI；输入语义索引 |
| `tool_result`/`web_search_tool_result` | `{tool_use_id, content}` | Anthropic 服务端工具的结果（`srvtoolu_` id） |

### 1.2 `formatAgentMessages(payload, indexTokenCountMap?, tools?, skills?, options?)`（format.ts:2801）

**返回值：** `{ messages, indexTokenCountMap?, summary?: {text, tokenCount}, boundaryTokenAdjustment?, compactionSemanticIndex?, compactionSemanticIndexSnapshot? }`。

**选项：**
- `provider`
- `legacyContent`：把全是文本的数组拍平为字符串
- `preserveReasoningContent`：对 DeepSeek 默认为 true
- `skipSkillBodyNames`
- `compactionSemanticIndex: {baseSnapshot?, intentToolNames?}`

**第 0 步：扫描摘要边界**（`scanSummaryBlocks`）
- 遍历所有消息和片段。每个文本非空的 `summary` 片段都会覆盖边界，所以最后一个生效。
- 摘要文本是 `content[].text` 拼接后去除首尾空白的结果。若为空，则回退到旧版的顶层 `text`。
- `tokenCount` 缺省时回退为 0。
- **覆盖（coverage）模式：** 如果 `coverage.retainedFromMessageId` 与摘要自身索引处或之前某条消息的 `messageId`/`id` 匹配，边界为 `{mode:'coverage', messageIndex: retainedIndex}`。索引 < retainedIndex 的所有载荷条目都被丢弃。锚点消息及其后的所有内容完整保留。
- **位置（positional）模式（旧版）：** 丢弃摘要所在消息之前的载荷条目。摘要自身所在的消息被切为 `content.slice(contentIndex+1)`。
- 摘要**不会**作为消息发出。它以 `summary` 返回，由宿主作为 `initialSummary` 传入。

**第 1 步：逐条处理**
- `sourceMessageId` = 去除首尾空白后的 `messageId ?? id`。
- 字符串内容变为 `[{type:'text', text}]`。
- 空数组不产出任何消息。

**第 2 步：非助手角色**经过 `formatMessage(..., langChain:true)`：
- 角色映射：
  - `role` 优先。
  - 若存在 `lc_id[2]`（且不是 langChain 模式），把 SystemMessage/HumanMessage/AIMessage 映射为 system/user/assistant。
  - 若缺少 `role`，使用 `sender`：`'user'`（不区分大小写）→ user，其他 → assistant。
  - 输出类：`user` → `HumanMessage`，`assistant` → `AIMessage`，其他 → `SystemMessage`。
- `name` 被清理为符合 `^[a-zA-Z0-9_-]{1,64}$`（非法字符变为 `_`，再截断到 64）。
- 媒体：**user** 消息上的顶层 `documents`、`videos`、`audios` 和 `image_urls` 变为数组内容。对 Anthropic，顺序为先媒体后文本；其他提供商为先文本后媒体。
- 消息被打上来源信息（provenance）标记（见 1.5）。

**第 3 步：助手条目**

*工具过滤*只在提供了 `tools: Set<string>` 时进行：
- 如果 `tool_call` 片段的名称在集合中、是被发现的工具，或由 SDK 管理（`subagent`、`conditional_transfer`、`lc_transfer_to_*`），则保留。
- 被发现的工具来自 `tool_search` 调用，其 `output` 能解析（或用正则提取）为 JSON `{tools:[{name}]}`。这些名称会加入集合，供后续片段使用。
- 没有名称的片段被丢弃。
- 对每个被丢弃的调用，其 id 也会从兄弟文本片段的 `tool_call_ids` 中移除（列表变空时删除该键）。
- 名为 `skill` 且带 `args.skillName` 的片段进入 `pendingSkillNames`（仅在提供了 `skills` map 时）。

*然后 `formatAssistantMessage` 按顺序遍历片段。* 它维护一个 `currentContent` 缓冲区、`lastAIMessage` 和 `pendingReasoningContent`。

- `activity_label`、`error`、`agent_update`、`summary` → 跳过。活动标签只会被收集进语义索引。
- 空白 `text`（只有空白字符）→ 跳过。
- **推理**（`think`/`thinking`/...）→ 文本追加到 `pendingReasoningContent`，并设 `hasReasoning=true`。它**永远不会作为内容发出**。如果设置了 `preserveReasoningContent`，它会成为下一条新建 AIMessage 的 `additional_kwargs.reasoning_content`；或者当工具调用挂到 `lastAIMessage` 上时，成为 `lastAIMessage` 的该字段。
- **带 `tool_call_ids` 的 `text`** 开启一条新的 AIMessage，用来承载工具调用：
  - 如果 `currentContent` 中有任何非文本片段：把这段文本推入其中，并发出 `AIMessage(array content)`。
  - 否则：把缓冲的文本按 `acc + text + "\n"` 拼接，再接上 `"\n" + thisText`，去除首尾空白后发出 `AIMessage(string)`。（Anthropic 只有在内容为字符串时才认 `tool_calls`。）
  - 如果缓冲区为空：`AIMessage(text)`。
  - 设置 `lastAIMessage`。
- **`tool_call`：**
  - 仅 Anthropic：对重复的 id，只保留首选片段（带 `output` 的那个）。
  - Anthropic 服务端工具（`srvtoolu_*` id）变为一个待发出的 `{type:'server_tool_use', id, name, input}` 块。它在匹配的结果片段出现时发出。未配对的会被丢弃，除非这是最后一个载荷条目。
  - 如果没有 `lastAIMessage`，通过发出 `AIMessage('')` 来“修复”序列。
  - `args`：字符串会被 `JSON.parse`；解析失败则变为 `{input: str}`。
  - Anthropic：当 `lastAIMessage.content` 是数组时，把 `{type:'tool_use', id: normalizeAnthropicToolCallId(id), name, input}` 推入其中。否则把 `{id, name, args}` 推入 `tool_calls`。不包含 `output`。
  - 发出 `ToolMessage({tool_call_id: id ?? '', name, content: compactToolContent(output, 400_000).content})`。如果缺少 `output`，内容为 `''`。
- **`steer`：**
  - 刷出 `currentContent`：拼接的文本变为 `AIMessage(string)`；若有任何非文本片段则为数组。
  - 如果缓冲区为空但有待处理且需保留的推理，发出一条空的锚点 `AIMessage('')`，以免推理丢失。
  - 发出 `HumanMessage({content: media ?? steer, additional_kwargs:{role:'user', source:'steer'}})`。
  - 重置 `lastAIMessage = null` 和推理状态，使插话之后的工具调用获得一个新的 AI 锚点，并落在该用户轮次之后。
- **其他任何片段**（图片、不带 `tool_call_ids` 的文本、……）追加到 `currentContent`。
- **最终刷出：** 追加待发出的服务端工具使用块。然后：
  - 若 `hasReasoning` 且缓冲的片段全是文本：发出拼接后的字符串（为空则跳过）。
  - 若有任何缓冲片段不是文本：发出数组内容。
  - 没有推理时：发出数组内容。

*插话锚点：*
- 如果某个助手条目的输出以插话 HumanMessage 结尾，则设置 `pendingSteerAnchor`。
- 只有当后面确实有消息发出时才会刷出它。如果下一条消息是助手消息，就只清除该标志。
- 否则先发出一条 `AIMessage('_')`（内容必须非空）。
- 这避免在严格交替的提供商上出现两个相邻的用户轮次，并且永远不会让锚点成为最后一个轮次。

*技能正文：*
- 在助手的消息之后，对每个不在 `skipSkillBodyNames` 中、且其正文存在于 `skills` 的待处理技能，发出 `HumanMessage(body, additional_kwargs:{role:'user', isMeta:true, source:'skill', skillName})`。
- 这些消息不参与该条目的 token 分摊。

*旧版内容：* 设置 `options.legacyContent` 时，任何内容为纯文本块数组的 human/ai/system 消息，在发出时被拍平为 `blocks.map(text).join("\n").trim()`。结果与 `src/messages/content.ts` 中的 `formatContentStrings` 相同。

**示例。** 输入：
```json
[{"role":"user","messageId":"u1","content":"find X"},
 {"role":"assistant","messageId":"a1","content":[
   {"type":"think","think":"plan"},
   {"type":"text","text":"Searching","tool_call_ids":["t1"]},
   {"type":"tool_call","tool_call":{"id":"t1","name":"search","args":"{\"q\":\"X\"}","output":"found"}},
   {"type":"steer","steer":"also Y"},
   {"type":"text","text":"Done"}]}]
```
输出：
1. `Human("find X")`，id=`u1`
2. `AI("Searching", tool_calls=[{id:t1,name:search,args:{q:"X"}}])`，id=`a1`
3. `Tool("found", tool_call_id=t1, name=search)`，id 未设置，`sourceMessageId=a1`
4. `Human("also Y", source:'steer')`
5. `AI("Done")`。因为 `hasReasoning` 为 true 且缓冲区全是文本，这里是字符串。

`think` 片段会消失，除非设置了 `preserveReasoningContent`，此时第 2 条消息获得 `reasoning_content:"plan"`。

一个已知的顺序问题：`[text(no tool_call_ids), tool_call]` 会产生 AI('')、Tool、AI(text)。文本在最后才被刷出。宿主应当在工具调用之前的文本上打上 `tool_call_ids`。

### 1.3 `indexTokenCountMap` 重映射（format.ts:3214）
输入 map 以载荷索引为键。它被重映射到输出索引：
- **1 → 1：** 直接复制计数。
- **1 → N：** 按字符长度拆分。每条消息的长度是其文本/内容字符数，加上每个工具调用的名称和参数长度（上限 400k）。
  - 除最后一条外，每条消息得到 `floor(len_k/total * count)`；最后一条得到余数。
  - 如果所有长度都是 0，则平均分配，余数归最后一条。
- **位置边界条目：** 计数按 `retainedChars/totalChars` 缩放，最小为 1。只有当所有片段都是 `text` 或 `thinking` 时才这样做；出现任何其他片段都会取消缩放（宁可多算也不少算）。这一调整以 `boundaryTokenAdjustment` 报告。
- **覆盖模式：** 计数保持不变。

`shiftIndexTokenCountMap(map, systemTokens)` 把系统提示放在键 0，并把其他所有键加 1。

### 1.4 多智能体标注：`labelContentByAgent(parts, agentIdMap, agentNames?, {labelNonTransferContent?})`（format.ts:2330）
宿主在格式化之前对一次运行的内容片段调用它。`agentIdMap` 把片段索引映射到 agentId。

**默认（移交）模式：**
- 名称以 `lc_transfer_to_` 开头的 `tool_call` 被保留。
- 随后那个智能体的片段被缓冲，并折叠进该移交调用的 **`output`**：
  ```
  --- Transfer to {name} ---
  {name}: {"type":"think","think":...}
  {name}: {"type":"text","text":...}
  {name}: {"type":"tool_call","tool_call":{...}}
  --- End of {name} response ---
  ```
  各行之间用空行（`\n\n`）连接。
- 不是通过移交到来的智能体的内容原样通过。

**`labelNonTransferContent: true`（并行扇出）：**
- 同一智能体的每段连续片段变为一个文本片段：
  ```
  --- {name} ---
  {name}: text
  {name}: {"type":"think",...}
  {name}: {"type":"tool_call",...}
  --- End of {name} ---
  ```

**两种模式共同的规则：** `activity_label` 片段被跳过，且不改变状态。`steer` 片段刷出缓冲区，原样通过，并关闭任何未结束的移交捕获。

### 1.5 来源信息与 id（`src/messages/provenance.ts`、`ids.ts`、`injected.ts`）
- 每条格式化后的消息都获得 `additional_kwargs.sourceMessageId`。
- 每条格式化后的消息还获得 `additional_kwargs.provenance = {version:1, parts:[{attribution:'user'|'model'|'tool'|'synthetic', sourceMessageId?, sourceContentPartIndices?}]}`。索引指向*切片或过滤之前*的持久化内容数组。上限：256 个片段、每个片段 256 个索引，等等。
- 只有从一个条目派生出的第一条消息获得 `id = sourceMessageId`。派生消息和合成消息不设置 `id`，由 reducer 分配 UUID。这防止伪造的 id 泄漏进提供商载荷。
- `projectionInvariant.ts` 中有一个可选的检查，由 `AGENT_MESSAGE_PROJECTION_INVARIANT=observe|assert` 控制。它标记来源信息缺失或无效的消息，以及既不是合成的、也没有关联到来源 id 的片段。
- `ids.ts:getMessageId(stepKey, graph)` 为每个 step key 生成流消息 id `msg_<nanoid>`，如果已有预分配的 id 则复用。
- `convertInjectedMessages`（`injected.ts`）：运行中途注入的消息（角色为 user 或 system）**全部**变为 `HumanMessage`，带 `additional_kwargs {role, injected:true, isMeta?, source?, skillName?}`。空的或只有空白的文本条目被跳过。插话的来源归属为 `user`，其他为 `synthetic`。

### 1.6 发送给提供商时的改写（作用于提供商投影，不作用于存储的历史）
- **`ensureThinkingBlockInMessages`**（format.ts:4271）。只考虑最后一个人类轮次之后的消息。一条有工具使用但没有推理块、早于 `runStartIndex`、且同一链中更早位置也没有推理的 AI 消息，会连同其后的 ToolMessage 一起折叠进一条 `HumanMessage`：
  ```
  [Previous agent context]
  AI: ...
  AI: [tool_call] name(args)
  Tool: ...
  ```
  （译：`[Previous agent context]` 意为“[此前的智能体上下文]”。）
  - 图片保留为块。
  - 预算：总计 400k 字符，每个折叠块 8k，100k 个工作单位。超出部分标记为 `… [additional folded context omitted]`（译：……[其余折叠上下文已省略]）。
  - 目的：切换到开启思考的 Claude 模型。
- **`foldToolBlocksForToollessAgent`**：同样的折叠，标题为 `[Previous tool interaction]`（译：[此前的工具交互]）。用于接收方智能体没有绑定任何工具的情况（Bedrock 在没有 toolConfig 时拒绝 toolUse 块）。
- **`coalesceAdjacentUserTurns`**（`alternation.ts`）：只用于 Bedrock 和 Mistral。相邻的人类轮次被合并，只含工具结果的人类轮次除外。合并后的消息保留后一条消息的 kwargs 和第一条消息的 id。
- **移交提示语**（`handoffCue.ts`）：
  - 当载荷以本次运行产生的 AI 消息结尾时，追加一个合成的用户轮次：`[The assistant message above is the completed output of a previous agent. Respond now according to your own role and instructions.]`（`source:'handoff'`）。
    （译：[上面的助手消息是前一个智能体已完成的输出。现在请按照你自己的角色和指令进行回复。]）
  - 对无指令的移交边，提示语为 `Continue as the receiving agent using the preceding user request and context.`（`source:'routing'`）。
    （译：以接收方智能体的身份，基于前面的用户请求和上下文继续。）
  - 这些只存在于发送的载荷中；对回退提供商，`removePredecessorHandoffCue` 会再次剥掉该提示语。
- **`assistantPhase.ts`**：按 `phase`（`commentary` | `final_answer`）把流式文本拆分到不同的消息创建步骤中。这只影响流式输出，不影响历史格式化。

---

## 2. 状态 reducer（`src/messages/reducer.ts`）

`messagesStateReducer(left, right)`。这是 LangGraph `add_messages` 的一个分支版本。

1. 把两侧都强制转换为消息，**跳过 null/undefined** 条目（不完整的提供商片段）。
2. 为两侧所有缺少 id 的消息分配 `uuid4`。该 id 也写入 `lc_kwargs.id`。
3. **全部移除哨兵：** 如果 `right` 包含 `RemoveMessage(id='__remove_all__')`（来自 `createRemoveAllMessage()`），返回 `right[idx+1:]` 并丢弃整个 `left`。摘要节点用它返回 `[RemoveAll, ...retainedTail]`。
4. 否则按 id 合并：
   - 右侧消息的 id 已存在时，就地替换左侧消息。
   - `remove` 消息把其 id 标记为待删除。移除未知 id 会抛错。
   - 新 id 追加到末尾。
   - 最后过滤掉被标记的 id。

Python：在 `langgraph` 中实现为 `Annotated[list[BaseMessage], reducer]`。较新版本的 Python `add_messages` 已支持 `RemoveMessage(id=REMOVE_ALL_MESSAGES)`；需要补上跳过 null 的逻辑。

---

## 3. Token 计数（`src/utils/tokens.ts`）

**编码。** `EncodingName = 'o200k_base' | 'claude'`，从 `ai-tokenizer` 包中加载。`encodingForModel(model)` 在小写模型名包含 "claude" 时返回 `claude`，否则返回 `o200k_base`。

**`createTokenCounter(encoding)`** 返回 `(msg) => n`：
- `n = getTokenCountForMessage(msg)`。
- 对 `claude`，结果为 `ceil(n * 1.1)`（`CLAUDE_TOKEN_CORRECTION`：API 报告的数字多约 10%）。
- 计数器的编码记录在一个 WeakMap 中，以便 `encodingOfTokenCounter` 区分 SDK 计数器和宿主提供的计数器。

**`getTokenCountForMessage`：**
- **基础：** `tokensPerMessage = 3`。角色、`name` 和 `additional_kwargs.reasoning_content` *不*计入。
- **字符串内容：** 按 8,192 字符分块分词，每个分块边界加 4 个 token。
- **数组内容，逐项：**
  - `error` → 0。
  - 图片（`image_url`/`image`/`computer_screenshot`/`input_image`）→ `ceil(estimateImageBlockTokens * 1.05)`。
  - `document`/`file`/`image_file` → `ceil(estimateDocumentBlockTokens * 1.05)`。
  - `media`/`video`/`audio`/`video_url`/`input_audio` → 按时长估算的媒体 token × 1.05。
  - `tool_call` → 名称 + 参数（字符串或结构化）+ 递归计算输出。
  - 其他带类型的项 → 递归进入 `item[item.type]`（例如 `text` → `item.text`，`think` → `item.think`）。如果该字段缺失，则把整个项序列化为 JSON。
  - 不带类型的对象 → 序列化为 JSON。
- **结构化值：** 最多序列化 200k 字符。超出部分：预览 token + 4 × 省略的字符数。
- **AI 消息：**
  - `tool_calls` 加上名称 + 参数，**但**已由内联 `tool_use`/`tool_call` 块表示的 id 除外（不重复计数）。
  - 如果没有已解析的 `tool_calls`，则把 `additional_kwargs.tool_calls` 按 JSON `{id,type,function:{name,arguments},custom}` 计数。
  - 旧版 `function_call` 按 JSON 计数。
- **Computer-call 输出：** `additional_kwargs.type==='computer_call_output'` 且内容为字符串的 `tool` 消息按截图计价。
- **安全：** proxy、访问器（accessor）和不安全的值会抛出 `UnsafeTokenMeasurementError`。Python 版本可以省掉大部分这类逻辑。

**媒体估算：**
- **图片：** 尺寸从 base64 头部字节读出（PNG/JPEG SOF0/SOF2、GIF、WebP VP8）。
  - Claude：`max(1024, ceil(w*h/750))`。
  - OpenAI：`85 + 170 * ceil(w/512) * ceil(h/512)`；`detail:'low'` 为 85。
  - 尺寸未知或只有 URL：1024。低细节的 `file_id`：85。
- **PDF（base64）：** `max(1, ceil(len/75000))` 页 × 2000（Claude）或 1500（OpenAI）。
- **其他文档：** 文本文档做分词；URL 文档为 2000。
- **视频：** `bytes/250_000` 秒 × 300 token/秒（只有 URL：9000）。**音频：** `bytes/16_000` 秒 × 32 token/秒（只有 URL：960）。

**校准**在裁剪器中进行，而不在计数器中（细节见 §4）。`indexTokenCountMap` 始终保持为本地分词器的**原始**单位。预算决策会乘以 `calibrationRatio`，它被限制在 [0.5, 5] 之间，由宿主以 `contextMeta.calibrationRatio` 持久化。

**摘要载体的开销。** `computeSummaryTokenCount` 测量 `buildSummaryCarrierText(text)`（在 o200k 上包装部分约 48 个 token）。它使用宿主计数器，除非该计数器记录的编码与*接收方智能体*的编码不一致。如果接收方智能体的模型名包含 "claude" 或其提供商家族是 anthropic，其编码为 `claude`。

`apportionTokenCounts` 做最大余数法缩放。它用于 `budget.ts` 中按工具的分解（`syncBudgetDerivedFields` / `createToolMessageUsageAccumulator`，为 UI 遥测报告工具占上下文的份额）。

---

