---
sidebar_position: 12
---

# Knowledge Base Architecture

This document describes the end-to-end knowledge base flow — creation, ingestion, indexing, and query-time invocation — and which service owns each step. For indexing concurrency protection see [Knowledge Indexing Protection](./knowledge-indexing-protection.md); for external integrations see [External Knowledge Sources](./external-knowledge-sources.md) and [External Knowledge MCP](./external-knowledge-mcp.md).

## Service Responsibilities

| Module | Responsibility |
| --- | --- |
| `backend/` | Knowledge HTTP APIs, document orchestration, permissions, RAG / direct-injection routing, knowledge MCP server |
| `knowledge_runtime/` | Internal RAG service (port 8200) that executes indexing and retrieval |
| `knowledge_engine/` | Execution core called by `knowledge_runtime`: readers, splitters, embeddings, vector stores |
| `knowledge_doc_converter/` | Standalone Celery worker on the `knowledge_conversion` queue; converts PDF/PPTX to Markdown with MinerU |
| `chat_shell/` | Exposes knowledge tools to the LLM and formats retrieval results |
| `backend-rs/` | Provides `knowledge`, `knowledge-access`, `knowledge-mcp`, and `knowledge-jobs` crates, but does not yet serve knowledge traffic (see [Ownership](#python-and-rust-ownership)) |

## Data Model

- **KnowledgeBase**: a CRD stored in the `kinds` table. `spec.retrievalConfig` references a Retriever (vector store configuration) and an Embedding Model; documents are not indexed if either is missing. Knowledge base types are notebook, classic, and code_wiki.
- **Documents**: the `knowledge_documents` table (`backend/app/models/knowledge.py`). Key fields:
  - `source_type`: `file` / `text` / `web` / `attachment` / `code` / `external`
  - `attachment_id`, `source_config` (stores `converted_attachment_id` after conversion)
  - `index_status`, `index_generation`: index state machine and generation
  - `splitter_config`, `summary`, `chunks`
- **External sources**: the `knowledge_document_external_sources` table tracks the origin and sync state of DingTalk and other external documents.

## Build Pipeline

```
Upload / text / web page / external import
        │  backend: orchestrator._create_and_index_document
        ▼
Pre-check (skip types unsuitable for RAG, such as Excel)
        │
        ▼
Enqueue with dedup (index_state_machine: bump generation, skip in-flight documents)
        │
        ├── Needs conversion (PDF/PPTX) ──► knowledge_conversion queue ──► knowledge_doc_converter
        │                                                                  │ MinerU → Markdown → S3
        │                          internal callback stores converted_attachment_id ◄┘
        ├── Image / video ──► multimodal pipeline (produces Markdown)
        ▼
Celery index_document_task (per-document Redis lock + generation check)
        │  RemoteRagGateway
        ▼
knowledge_runtime IndexExecutor ──► knowledge_engine: split → embed → write to vector store
        │
        ▼
Write back success / failed → asynchronously generate document and knowledge base summaries
```

### 1. Entry Points

Every entry point converges on `_create_and_index_document` in `backend/app/services/knowledge/orchestrator.py`:

| Source | Entry point |
| --- | --- |
| Attachment upload | `create_document_from_attachment` |
| Text | `create_document_with_content` |
| Web scraping | `create_web_document` |
| External documents (DingTalk, etc.) | `attach_external_document_content`, plus the Celery task `import_external_document` |

`indexing.get_rag_indexing_skip_reason` decides by extension and size whether to skip RAG indexing.

### 2. Enqueue and Dedup

`prepare_document_index_enqueue` (`index_state_machine.py`) increments `index_generation` and skips documents that are already in progress or succeeded. A new request may take over when a state has lasted longer than its threshold:

| Setting | Default |
| --- | --- |
| `KNOWLEDGE_INDEX_STALE_QUEUED_SECONDS` | 600 (10 min) |
| `KNOWLEDGE_INDEX_STALE_PENDING_CONVERSION_SECONDS` | 7200 (120 min) |
| `KNOWLEDGE_INDEX_STALE_INDEXING_SECONDS` | 2700 (45 min) |

### 3. Document Conversion

Documents that need conversion move to `pending_conversion` and are dispatched to the `knowledge_conversion` queue. `knowledge_doc_converter` uploads the result to S3 and notifies the backend through an internal callback, which stores `converted_attachment_id` and re-dispatches indexing. Images and videos use the multimodal pipeline (`multimodal_pipeline.py`), which is disabled by default and controlled by `MULTIMODAL_ENABLED`.

### 4. Index Execution

- `index_document_task` (`backend/app/tasks/knowledge_tasks.py`) acquires a per-document Redis lock renewed by a watchdog (`KNOWLEDGE_INDEX_LOCK_TIMEOUT_SECONDS=120`, `KNOWLEDGE_INDEX_LOCK_EXTEND_INTERVAL_SECONDS=30`), checks the generation, and calls `indexing.run_document_indexing`.
- `rag/gateway_factory.get_rag_gateway` returns only `RemoteRagGateway`; the backend never indexes locally. Requests carry only references (knowledge base, document, index owner).
- `IndexExecutor` in `knowledge_runtime` resolves the Retriever, Embedding, and splitter configuration itself, loads the content, and calls `knowledge_engine`.

### 5. Splitting, Embedding, and Storage

| Stage | Implementation |
| --- | --- |
| Splitting | `semantic` (default when unconfigured), `flat` (SmartSplitter with Markdown enhancement when `file_aware`), `hierarchical` (`ingestion/pipeline.py` builds parent and child nodes and indexes the child nodes) |
| Embedding | `knowledge_engine/embedding/`, OpenAI-compatible or custom models |
| Vector stores | Elasticsearch, Qdrant, Milvus (`storage/capabilities.py`) |
| Index naming | `per_dataset`, `per_user`, `rolling`, `fixed` (`storage/index_strategy.py`) |

### 6. Completion and Scheduled Jobs

After a successful index, `generate_document_summary` and `update_kb_summary` are dispatched asynchronously. Celery Beat runs `scan_stale_index_tasks` every 5 minutes to recover stuck indexing and periodically runs `cleanup_knowledge_attachment_orphans`.

## Query-Time Flow

### Knowledge Base Binding Sources

The knowledge bases used in a conversation come from three sources, merged and deduplicated by `backend/app/services/chat/selected_knowledge.py`; explicit selections in the current turn take precedence:

1. Agent default knowledge bases: `spec.defaultKnowledgeBaseRefs` on member Bot Ghosts (`task_default_knowledge_bases.py`, ACL-filtered).
2. Task-level knowledge bases: stored in `Task.spec` and managed by `TaskKnowledgeBaseService`; group chat bindings use this layer.
3. Knowledge bases the user explicitly selects in the current message.

The merged result produces knowledge prompts delivered through the `ExecutionRequest` dynamic context (see [Dynamic Context](./dynamic-context.md)).

### Chat Shell Request Flow

```
User message → backend builds ExecutionRequest (knowledge_base_ids, kb_tool_access_mode, kb_meta_prompt)
        → chat_shell prepare_knowledge_base_tools: register tools + choose prompt
        → LLM calls knowledge_base_search
        → POST backend /api/internal/rag/retrieve (route_mode, runtime_context, restricted_mode)
        → KnowledgeSearchRunner: direct injection (read source text) or RAG (knowledge_runtime POST /query)
        → chat_shell formats results with citation indices → returned to the LLM as the tool result
```

`prepare_knowledge_base_tools` in `chat_shell/tools/knowledge_factory.py` selects the prompt (defined in `shared/prompts/knowledge_base.py`) and tool set per scenario:

| Scenario | Prompt | Tools |
| --- | --- | --- |
| Restricted role | `KB_PROMPT_RESTRICTED_ANALYST` | `knowledge_base_search` only |
| No knowledge base has a retriever | `KB_PROMPT_NO_RAG` | Browsing tools |
| Explicitly selected this turn | `KB_PROMPT_STRICT` | Full tool set |
| Inherited from the task | `KB_PROMPT_RELAXED` (general knowledge allowed) | Full tool set |

The full tool set — `knowledge_base_search`, `kb_ls`, `kb_head`, and `knowledge_list_documents` — shares a limit of 10 calls per conversation (`DEFAULT_MAX_CALLS_PER_CONVERSATION`).

### Choosing RAG or Direct Injection

The backend decides in `decide_route_mode_for_chat_shell` (`rag/direct_injection.py`):

1. No knowledge base, or a `metadata_condition` is present: use RAG.
2. The caller sets `route_mode` explicitly: follow it.
3. `auto` mode with `RAG_AUTO_DISABLE_DIRECT_INJECTION` enabled: use RAG.
4. Otherwise compute available budget = context window − used tokens − reserved output (4096) − 10% buffer. If the estimated tokens of the selected content are at most available budget × `CHAT_SHELL_DIRECT_INJECTION_RATIO` (0.3), inject the source text directly; otherwise use RAG.
5. If direct injection fails, `KnowledgeSearchRunner` falls back to RAG.

### Executors (ClaudeCode / Codex)

`SUPPORTED_PROVIDER_NATIVE_SHELLS` in `selected_knowledge.py` covers `Chat`, `Codex`, and `ClaudeCode`. These executors access knowledge bases through the backend knowledge MCP server (`backend/app/mcp_server/tools/knowledge.py`), with tools such as `wegent_kb_search_knowledge_base`, `wegent_kb_list_knowledge_bases`, `wegent_kb_list_documents`, `wegent_kb_read_document_content`, and write tools for creating documents and importing DingTalk documents. Trusted external systems use `/mcp/knowledge-external/sse`.

### Permissions

- `knowledge_access_policy.py` enforces ACLs at query time.
- The Restricted Analyst role uses `kb_tool_access_mode=restricted_search_only`. Retrieved chunks pass through `protected_mediation.py` and `chat_shell/tools/restricted_kb_summary.py` to produce non-extractive safe summaries, so the main model never sees the source text.

### External Interfaces

- Knowledge Open API: `/api/knowledge/list`, `/api/knowledge/search`, and more; see [Knowledge Open API](../reference/knowledge-open-api.md).
- `/api/v1/responses` can use knowledge bases through the `knowledge_base` tool.
- `KnowledgeBaseQAService` only queries knowledge Q&A history and does not perform retrieval.

## Python and Rust Ownership

- **HTTP APIs**: `backend-rs/config/routes.toml` states that knowledge APIs stay Python-owned. Rust handlers for `/api/knowledge-bases*`, `/api/knowledge/*`, `/api/mcp/knowledge/sse`, `/api/retrievers`, and others are registered without active route rules, so all requests are forwarded to `backend/`.
- **Background jobs**: `backend-rs/knowledge-jobs` registers knowledge jobs with the same names as the Python Celery tasks, but they are gated by `WEGENT_RS_JOB_ALLOWLIST` and disabled by default. In the default deployment, Python Celery consumes indexing, summary, and sync jobs. See `backend-rs/docs/en/background-jobs.md` for the cutover procedure; never let Python and Rust schedule the same job family concurrently.

## Current Implementation Notes

The following reflects the code as of 2026-11 and should be addressed when changing the related modules:

- `decide_injection_mode` and `execute_injection_strategy` in `chat_shell/tools/knowledge_injection_strategy.py` are only called by tests, since routing moved to the backend; the `0.3` ratio is defined in both chat_shell and the backend.
- The frontend "advanced knowledge mode" (`AdvancedKnowledgeToggle`) is a display preference only; the backend has no corresponding retrieval mode.
- Files such as `orchestrator.py`, `chat_shell/tools/builtin/knowledge_base.py`, and `chat_shell/tools/builtin/knowledge_listing.py` exceed 1000 lines.
