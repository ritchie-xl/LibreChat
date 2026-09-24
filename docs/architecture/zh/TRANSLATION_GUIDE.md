# 翻译说明（内部使用）

This file is the shared brief for translating `docs/architecture/**` into Simplified Chinese. It is removed
once the translation is finished.

## Output location

Mirror the English tree under `docs/architecture/zh/`:

| English source | Chinese output |
|---|---|
| `docs/architecture/README.md` | `docs/architecture/zh/README.md` |
| `docs/architecture/reference/NN-*.md` | `docs/architecture/zh/reference/NN-*.md` (same file name) |
| `docs/architecture/agents-sdk/README.md` | `docs/architecture/zh/agents-sdk/README.md` |
| `docs/architecture/agents-sdk/reference/NN-*.md` | `docs/architecture/zh/agents-sdk/reference/NN-*.md` (same file name) |

## Rules

1. **Translate everything a reader reads**: headings, prose, list items, table cells, blockquotes, and the
   human-readable labels inside Mermaid diagrams (node text, edge labels, subgraph titles, sequence-diagram
   messages and notes, state-transition labels).
2. **Never translate** code, identifiers or literals:
   - anything inside backticks, and fenced code blocks (translate only `#` comments inside `text`/`python`
     layout trees if they are plain prose);
   - file paths, env var names, HTTP methods and paths, JSON field names, enum values, event names
     (`on_run_step`), class and function names, config keys, CLI commands, URLs;
   - Mermaid syntax, node IDs, participant IDs, state names used as identifiers (`running`, `requires_action`,
     `staging`, `in_progress`, …). Translate only the display text.
3. **Keep structure identical**: same heading levels and numbering, same tables (same columns and rows), same
   list nesting, same Mermaid diagrams, same order. Do not add or drop content. Do not summarize.
4. **Links**
   - Links between documents keep the same relative path, which now resolves inside `zh/`
     (e.g. `reference/03-chat-agent-pipeline.md` still works from `zh/README.md`).
   - External URLs are unchanged.
   - **Anchor links** (`#...`) must match the *translated* heading, using GitHub's slug rules: lowercase
     ASCII letters, drop punctuation except `-` and `_` (this drops `.`, `,`, `:`, `(`, `)`, `/`, `&`, `+`,
     `·`, Chinese punctuation such as `、，：（）`), turn each space into `-`, keep Chinese characters as they are.
     Example: heading `## 8.2 SSE 契约` → anchor `#82-sse-契约`; heading `## 12. Python 移植计划` →
     `#12-python-移植计划`. When you link to a heading in another zh document, use that document's translated
     heading. The canonical headings for the two overview documents are listed below; use them exactly.
5. **Mermaid safety**: keep every label that contains non-ASCII text, spaces or punctuation inside double
   quotes (`A["中文标签"]`, `-->|"中文"|`). In `sequenceDiagram` messages avoid `;` and `#`. In `stateDiagram`
   transition labels avoid `:` inside the label text. Keep `<br/>` line breaks.
6. **Style**: clear, plain technical Chinese, as a senior engineer would write it for colleagues. Short
   sentences. Use full-width Chinese punctuation in Chinese prose（，。：；（））, but keep half-width
   punctuation inside code and identifiers. Put a space between Chinese and adjacent Latin words or numbers
   (`使用 Redis 作为缓存`, `默认 15 分钟`). Keep numbers, units and limits exactly as in the source.
7. **Reference-chapter header note**: translate the blockquote at the top of each reference chapter, keeping
   its link. For backend chapters use:
   `> 本章是 [LibreChat 后端架构](../README.md) 的参考章节，基于对 \`v0.8.8-rc4\`（\`361553f\`）代码的静态阅读整理。路径均相对于仓库根目录。行号为近似值，会随代码变动；细节以代码为准。`
   For SDK chapters use:
   `> 本章是 [@librechat/agents SDK 架构](../README.md) 的参考章节，基于对 [LibreChat-AI/agents](https://github.com/LibreChat-AI/agents) \`v3.9.3\`（\`2d5653d\`）代码的静态阅读整理。路径均相对于该仓库根目录。行号为近似值；细节以代码为准。`
8. Do not edit any English file. Do not commit. Only write your assigned output files.

## Glossary (use consistently)

| English | 中文 |
|---|---|
| backend / frontend | 后端 / 前端 |
| agent (the product concept) | 智能体（代码名如 `Agent`、`agent_id` 保持英文） |
| ephemeral agent | 临时智能体 |
| subagent | 子智能体 |
| handoff | 移交 |
| fan-out / fan-in | 扇出 / 扇入 |
| endpoint (LLM endpoint type) | 端点 |
| provider (LLM) | 模型提供商（简称提供商） |
| model spec | 模型规格（modelSpec） |
| tool / tool call / tool result | 工具 / 工具调用 / 工具结果 |
| run / run step | 运行 / 运行步骤（run step） |
| generation / generation job | 生成 / 生成任务 |
| stream, streaming, SSE stream | 流、流式、SSE 流 |
| event / handler / hook | 事件 / 处理器 / 钩子 |
| content part / content aggregation | 内容片段 / 内容聚合 |
| checkpoint / checkpointer | 检查点 / 检查点存储（checkpointer） |
| interrupt / resume | 中断 / 恢复 |
| human-in-the-loop (HITL) | 人机协同（HITL） |
| approval / decision | 审批 / 决策 |
| steer / steering | 插话（steer）/ 插话引导 |
| preemption / seal / restart | 抢占 / 封存（seal）/ 重启（restart） |
| queued turn / turn | 排队轮次 / 轮次 |
| context window / budget | 上下文窗口 / 预算 |
| pruning / fading / calibration | 裁剪 / 淡化（fading）/ 校准 |
| summarization / compaction | 摘要 / 压缩（compaction） |
| prompt caching / cache breakpoint | 提示缓存 / 缓存断点 |
| token (LLM) | token |
| access token / refresh token / session | 访问令牌 / 刷新令牌 / 会话 |
| tenant / multi-tenancy | 租户 / 多租户 |
| principal / capability / permission / role | 主体 / 能力（capability）/ 权限 / 角色 |
| ACL / permission bits | 访问控制列表（ACL）/ 权限位 |
| share / share link | 共享 / 分享链接 |
| rate limiter / ban / violation | 限流器 / 封禁 / 违规 |
| lease / fence / epoch | 租约 / 栅栏（fence）/ 纪元（epoch） |
| compare-and-swap (CAS) | 比较并交换（CAS） |
| leader election | 领导者选举 |
| background loop / worker | 后台循环 / 工作进程 |
| schedule / scheduled run | 定时任务 / 定时运行 |
| trigger / delivery / dead letter | 触发器 / 投递 / 死信 |
| lane (ordering lane) | 通道（lane） |
| retention / TTL | 保留期 / TTL |
| file storage strategy | 文件存储策略 |
| upload / download | 上传 / 下载 |
| observability / tracing / metrics / span | 可观测性 / 链路追踪 / 指标 / span |
| deployment / replica / container | 部署 / 副本 / 容器 |
| reference chapter | 参考章节 |
| porting / replication / blueprint | 移植 / 复刻 / 蓝图 |
| byte-compatible | 字节级兼容 |
| golden test / fixture | 黄金测试 / 测试夹具 |
| sidecar | sidecar（边车进程） |
| wiring | 装配（wiring） |
| middleware | 中间件 |
| router / route | 路由器 / 路由 |
| collection / index (Mongo) | 集合 / 索引 |

## Canonical translated headings (overview documents)

### `zh/README.md` (title: `# LibreChat 后端架构`)

```
## 目录
## 1. 系统上下文
## 2. 部署拓扑
## 3. 代码组织与分层
## 4. 启动流程与请求管线
### 4.1 启动顺序（`api/server/index.js`）
### 4.2 全局中间件（按顺序）
### 4.3 聊天请求的中间件链
## 5. 配置系统
## 6. 认证与会话
### 6.1 凭据模型
### 6.2 登录与刷新
## 7. 授权与多租户
## 8. 聊天生成管线
### 8.1 端到端时序
### 8.2 SSE 契约
### 8.3 生成任务的生命周期
### 8.4 持久化时机
### 8.5 其他接入协议
## 9. 智能体运行时
## 10. LLM 提供商与模型
## 11. 工具、MCP、Actions、技能与代码执行
### 11.1 工具解析
### 11.2 MCP
### 11.3 Actions、技能、代码与网页搜索
## 12. 文件、存储与 RAG
## 13. 对话、消息、搜索与分享
## 14. 余额与 token 计费
## 15. 后台处理
## 16. 缓存、Redis 与集群
## 17. 可观测性
## 18. 数据模型
### 18.1 身份与访问
### 18.2 聊天
### 18.3 智能体、工具与内容
### 18.4 定时任务与触发器
### 18.5 集合清单
## 19. HTTP API 面
## 20. Python 复刻蓝图
### 20.1 技术映射
### 20.2 建议的包结构
### 20.3 需要保持字节级兼容的契约
### 20.4 分阶段路线图
### 20.5 可以简化的部分
### 20.6 主要风险
```

Resulting anchors used by other documents: `#1-系统上下文`, `#82-sse-契约`, `#15-后台处理`, `#12-文件存储与-rag`,
`#20-python-复刻蓝图`.

### `zh/agents-sdk/README.md` (title: `# @librechat/agents SDK 架构`)

```
## 目录
## 1. SDK 所处的位置
## 2. 模块结构
## 3. 运行生命周期
## 4. 图拓扑
### 4.1 单智能体
### 4.2 多智能体
## 5. 智能体节点
## 6. 流式事件契约
## 7. 工具执行
## 8. 人机协同、钩子与插话
## 9. LLM 提供商层
## 10. 消息与上下文管理
## 11. 可观测性
## 12. Python 移植计划
### 12.1 LangGraph Python 已提供的与需要自行实现的
### 12.2 包结构（无循环依赖）
### 12.3 移植顺序
### 12.4 降低风险的做法
```

Resulting anchors used by other documents: `#12-python-移植计划`, `#124-降低风险的做法`.
