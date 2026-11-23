---
sidebar_position: 12
---

# 知识库架构

本文说明知识库从建库、入库、索引到查询调用的完整链路，以及各步骤由哪个服务负责。索引并发保护的细节见[知识库索引保护](./knowledge-indexing-protection.md)，外部系统接入见[外部知识源](./external-knowledge-sources.md)和[外部知识库 MCP](./external-knowledge-mcp.md)。

## 服务划分

| 模块 | 职责 |
| --- | --- |
| `backend/` | 知识库 HTTP API、文档编排、权限、RAG / 直接注入路由决策、知识库 MCP 服务 |
| `knowledge_runtime/` | 内部 RAG 服务（端口 8200），执行索引与检索 |
| `knowledge_engine/` | 被 `knowledge_runtime` 调用的执行内核：读取、切分、向量化、向量库存储 |
| `knowledge_doc_converter/` | 独立 Celery worker，消费 `knowledge_conversion` 队列，用 MinerU 将 PDF/PPTX 转为 Markdown |
| `chat_shell/` | 向 LLM 暴露知识库工具，并格式化检索结果 |
| `backend-rs/` | 已有 `knowledge`、`knowledge-access`、`knowledge-mcp`、`knowledge-jobs` 等 crate，但尚未接管知识库流量（见[所有权](#python-与-rust-的所有权)） |

## 数据模型

- **KnowledgeBase**：存储在 `kinds` 表中的 CRD。`spec.retrievalConfig` 引用一个 Retriever（向量库配置）和一个 Embedding Model；两者缺一时文档不会建立索引。知识库形态包括 notebook、classic 和 code_wiki。
- **文档**：`knowledge_documents` 表（`backend/app/models/knowledge.py`）。关键字段：
  - `source_type`：`file` / `text` / `web` / `attachment` / `code` / `external`
  - `attachment_id`、`source_config`（转换完成后写入 `converted_attachment_id`）
  - `index_status`、`index_generation`：索引状态机与代次
  - `splitter_config`、`summary`、`chunks`
- **外部来源**：`knowledge_document_external_sources` 表记录钉钉等外部文档的来源与同步状态。

## 构建链路

```
上传 / 文本 / 网页 / 外部导入
        │  backend: orchestrator._create_and_index_document
        ▼
预检（跳过不适合 RAG 的类型，如 Excel）
        │
        ▼
入队去重（index_state_machine：递增 generation，跳过进行中的文档）
        │
        ├── 需要转换（PDF/PPTX）──► knowledge_conversion 队列 ──► knowledge_doc_converter
        │                                                        │ MinerU → Markdown → S3
        │                                  内部回调写入 converted_attachment_id ◄┘
        ├── 图片 / 视频 ──► 多模态管线（生成 Markdown）
        ▼
Celery index_document_task（文档级 Redis 锁 + generation 校验）
        │  RemoteRagGateway
        ▼
knowledge_runtime IndexExecutor ──► knowledge_engine：切分 → 向量化 → 写入向量库
        │
        ▼
状态回写 success / failed → 异步生成文档摘要与知识库摘要
```

### 1. 入口

所有入口最终都进入 `backend/app/services/knowledge/orchestrator.py` 中的 `_create_and_index_document`：

| 来源 | 入口 |
| --- | --- |
| 附件上传 | `create_document_from_attachment` |
| 文本 | `create_document_with_content` |
| 网页抓取 | `create_web_document` |
| 外部文档（钉钉等） | `attach_external_document_content`，以及 Celery 任务 `import_external_document` |

`indexing.get_rag_indexing_skip_reason` 按扩展名和大小判断是否跳过 RAG 索引。

### 2. 入队与去重

`prepare_document_index_enqueue`（`index_state_machine.py`）负责递增 `index_generation`，并跳过已在进行中或已成功的文档。状态停留超过阈值时，新的请求可以接管：

| 配置 | 默认值 |
| --- | --- |
| `KNOWLEDGE_INDEX_STALE_QUEUED_SECONDS` | 600（10 分钟） |
| `KNOWLEDGE_INDEX_STALE_PENDING_CONVERSION_SECONDS` | 7200（120 分钟） |
| `KNOWLEDGE_INDEX_STALE_INDEXING_SECONDS` | 2700（45 分钟） |

### 3. 文档转换

需要转换的文档先置为 `pending_conversion`，再派发到 `knowledge_conversion` 队列。`knowledge_doc_converter` 将转换结果上传到 S3，通过内部回调通知 backend 写入 `converted_attachment_id`，随后重新派发索引任务。图片和视频走多模态管线（`multimodal_pipeline.py`），默认关闭，由 `MULTIMODAL_ENABLED` 控制。

### 4. 索引执行

- `index_document_task`（`backend/app/tasks/knowledge_tasks.py`）获取文档级 Redis 锁，锁由 watchdog 续期（`KNOWLEDGE_INDEX_LOCK_TIMEOUT_SECONDS=120`，`KNOWLEDGE_INDEX_LOCK_EXTEND_INTERVAL_SECONDS=30`），校验 generation 后调用 `indexing.run_document_indexing`。
- `rag/gateway_factory.get_rag_gateway` 只返回 `RemoteRagGateway`，backend 本地不执行索引。请求只携带引用信息（知识库、文档、索引所有者）。
- `knowledge_runtime` 的 `IndexExecutor` 自行解析 Retriever、Embedding 与切分配置，读取内容后调用 `knowledge_engine`。

### 5. 切分、向量化与存储

| 环节 | 实现 |
| --- | --- |
| 切分策略 | `semantic`（未配置时的默认值）、`flat`（`file_aware` 时使用 SmartSplitter，支持 Markdown 增强）、`hierarchical`（`ingestion/pipeline.py` 生成父块与子块，以子块建立索引） |
| 向量化 | `knowledge_engine/embedding/`，支持 OpenAI 协议与自定义模型 |
| 向量库 | Elasticsearch、Qdrant、Milvus（`storage/capabilities.py`） |
| 索引命名 | `per_dataset`、`per_user`、`rolling`、`fixed`（`storage/index_strategy.py`） |

### 6. 收尾与定时任务

索引成功后异步派发 `generate_document_summary` 和 `update_kb_summary`。Celery Beat 每 5 分钟运行 `scan_stale_index_tasks` 处理卡住的索引，并定期运行 `cleanup_knowledge_attachment_orphans` 清理孤儿附件。

## 调用链路

### 知识库绑定来源

一次对话使用的知识库来自三处，由 `backend/app/services/chat/selected_knowledge.py` 合并去重，本轮显式选择优先：

1. 智能体默认知识库：成员 Bot 的 Ghost `spec.defaultKnowledgeBaseRefs`（`task_default_knowledge_bases.py`，按 ACL 过滤）。
2. 任务级知识库：保存在 `Task.spec` 中，由 `TaskKnowledgeBaseService` 管理，群聊绑定也使用这一层。
3. 用户在本条消息中显式选择的知识库。

合并结果生成知识库提示词，并通过 `ExecutionRequest` 的 dynamic context 下发（参见[动态上下文](./dynamic-context.md)）。

### Chat Shell 请求流程

```
用户消息 → backend 组装 ExecutionRequest（knowledge_base_ids、kb_tool_access_mode、kb_meta_prompt）
        → chat_shell prepare_knowledge_base_tools：注册工具 + 选择提示词
        → LLM 调用 knowledge_base_search
        → POST backend /api/internal/rag/retrieve（route_mode、runtime_context、restricted_mode）
        → KnowledgeSearchRunner：直接注入（读取原文）或 RAG（knowledge_runtime POST /query）
        → chat_shell 格式化为带引用编号的结果 → 作为工具结果进入 LLM 上下文
```

`chat_shell/tools/knowledge_factory.py` 中的 `prepare_knowledge_base_tools` 按场景选择提示词（定义于 `shared/prompts/knowledge_base.py`）和工具集：

| 场景 | 提示词 | 工具 |
| --- | --- | --- |
| 受限角色 | `KB_PROMPT_RESTRICTED_ANALYST` | 仅 `knowledge_base_search` |
| 所有知识库均未配置检索器 | `KB_PROMPT_NO_RAG` | 浏览类工具 |
| 用户本轮显式选择 | `KB_PROMPT_STRICT` | 完整工具集 |
| 从任务继承 | `KB_PROMPT_RELAXED`（允许结合通用知识） | 完整工具集 |

完整工具集包括 `knowledge_base_search`、`kb_ls`、`kb_head` 和 `knowledge_list_documents`，共享每个对话 10 次的调用上限（`DEFAULT_MAX_CALLS_PER_CONVERSATION`）。

### RAG 与直接注入的选择

决策由 backend 的 `rag/direct_injection.py` 中的 `decide_route_mode_for_chat_shell` 完成：

1. 没有知识库，或带有 `metadata_condition`：走 RAG。
2. 调用方显式指定 `route_mode`：按指定方式执行。
3. `auto` 模式且开启 `RAG_AUTO_DISABLE_DIRECT_INJECTION`：走 RAG。
4. 否则计算可用预算 = 上下文窗口 − 已用 token − 预留输出（4096）− 10% 缓冲。所选内容的估算 token 数不超过可用预算 × `CHAT_SHELL_DIRECT_INJECTION_RATIO`（0.3）时，直接注入原文；否则走 RAG。
5. 直接注入失败时，`KnowledgeSearchRunner` 回退到 RAG。

### 执行器（ClaudeCode / Codex）

`selected_knowledge.py` 中的 `SUPPORTED_PROVIDER_NATIVE_SHELLS` 包括 `Chat`、`Codex` 和 `ClaudeCode`。这些执行器通过 backend 的知识库 MCP 服务（`backend/app/mcp_server/tools/knowledge.py`）访问知识库，工具包括 `wegent_kb_search_knowledge_base`、`wegent_kb_list_knowledge_bases`、`wegent_kb_list_documents`、`wegent_kb_read_document_content`，以及创建文档、导入钉钉文档等写入工具。受信任外部系统使用 `/mcp/knowledge-external/sse`。

### 权限

- 查询时由 `knowledge_access_policy.py` 校验 ACL。
- 受限分析师角色使用 `kb_tool_access_mode=restricted_search_only`。检索片段先经 `protected_mediation.py` 和 `chat_shell/tools/restricted_kb_summary.py` 生成非摘录式安全摘要，主模型不接触原文。

### 对外接口

- Knowledge Open API：`/api/knowledge/list`、`/api/knowledge/search` 等，详见[知识库 Open API](../reference/knowledge-open-api.md)。
- `/api/v1/responses` 可以通过 `knowledge_base` tool 使用知识库。
- `KnowledgeBaseQAService` 只查询知识库问答历史，不参与检索。

## Python 与 Rust 的所有权

- **HTTP API**：`backend-rs/config/routes.toml` 明确知识库 API 由 Python 负责。`/api/knowledge-bases*`、`/api/knowledge/*`、`/api/mcp/knowledge/sse`、`/api/retrievers` 等 Rust handler 已注册，但没有启用的路由规则，请求全部转发到 `backend/`。
- **后台任务**：`backend-rs/knowledge-jobs` 注册了与 Python Celery 同名的知识处理任务，但受 `WEGENT_RS_JOB_ALLOWLIST` 控制，默认不执行。默认部署中由 Python Celery 消费索引、摘要和同步任务。切换步骤见 `backend-rs/docs/zh/background-jobs.md`，部署时不应让 Python 和 Rust 同时调度同一执行族。

## 当前实现备注

以下现状基于 2026-11 的代码，修改相关模块时应一并处理：

- `chat_shell/tools/knowledge_injection_strategy.py` 中的 `decide_injection_mode` 和 `execute_injection_strategy` 只被测试调用，路由决策已移到 backend；`0.3` 比例在 chat_shell 和 backend 各定义了一次。
- 前端的"高级知识模式"（`AdvancedKnowledgeToggle`）只是显示偏好，后端没有对应的检索模式。
- `orchestrator.py`、`chat_shell/tools/builtin/knowledge_base.py`、`chat_shell/tools/builtin/knowledge_listing.py` 等文件超过 1000 行。
