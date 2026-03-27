# QMD Feature Index

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Extracted:** 2026-03-27

## Core Features (Priority Tier 1)

### 1. Hybrid Search Pipeline
Combines BM25 full-text search, vector semantic search, and LLM re-ranking via Reciprocal Rank Fusion (RRF).

- **Description:** QMD's flagship feature uses a three-stage pipeline: (1) Query expansion via fine-tuned Qwen3 model generates lex/vec/hyde variations, (2) Parallel BM25 and vector searches retrieve candidates, (3) LLM reranking with position-aware blending produces final results.
- **Key Files:** `src/store.ts`, `src/llm.ts`, `src/index.ts`
- **Priority:** core

### 2. Collection Management
Index and manage multiple directory collections with glob patterns and ignore rules.

- **Description:** Users create named collections from directories, each with custom glob patterns (e.g., `**/*.md`). Collections track document counts, modification times, and can be listed, renamed, or removed. The index automatically detects changes on update.
- **Key Files:** `src/collections.ts`, `src/cli/qmd.ts` (collection commands)
- **Priority:** core

### 3. Context System
Hierarchical descriptive metadata attached to collection paths to improve search relevance.

- **Description:** Context adds "system prompts" to paths—descriptions like "API documentation" or "Meeting transcripts" that are returned alongside matching sub-documents. This tree structure (collection -> folder -> file) helps LLMs make better contextual choices. Global context applies to all collections.
- **Key Files:** `src/store.ts` (context operations), `src/db.ts` (path_contexts table)
- **Priority:** core

### 4. MCP Server Integration
Model Context Protocol server exposing search and retrieval tools to AI agents.

- **Description:** QMD runs as an MCP server providing `query`, `get`, `multi_get`, and `status` tools. Supports stdio transport (subprocess per request) or HTTP transport (long-lived daemon with persistent LLM loading). Works with Claude Desktop, Claude Code, and any MCP-compatible client.
- **Key Files:** `src/mcp/server.ts`, `src/cli/qmd.ts` (mcp command)
- **Priority:** core

### 5. SDK/Library API
Programmatic access to QMD's search and storage capabilities as a Node.js/Bun library.

- **Description:** Applications import `createStore()` to open an index, then call `search()`, `get()`, `addCollection()`, and other methods. Supports inline config, YAML config file, or DB-only reopening. Exported types cover SearchOptions, HybridQueryResult, DocumentResult, and more.
- **Key Files:** `src/store.ts` (main exports), `src/index.ts` (SDK entry)
- **Priority:** core

### 6. Smart Document Chunking
Intelligent text splitting that preserves markdown structure boundaries.

- **Description:** Instead of hard token boundaries, QMD scores break points (headings get 50-100 points, code blocks 80, blank lines 20). When approaching the 900-token chunk limit, it searches a 200-token window and cuts at the highest-scoring boundary. Code blocks are protected from splitting. 15% overlap between chunks.
- **Key Files:** `src/index.ts` (chunking logic), `src/store.ts` (chunk storage)
- **Priority:** core

### 7. Query Expansion Model
Fine-tuned Qwen3-1.7B model that generates structured search query variations.

- **Description:** Given a raw query like "auth config", the model outputs structured `hyde:` (hypothetical document), `lex:` (BM25 keywords), and `vec:` (semantic variations) lines. Trained via SFT on ~2,290 examples. Production path is SFT-only; GRPO is experimental.
- **Key Files:** `finetune/train.py`, `finetune/eval.py`, `finetune/reward.py`
- **Priority:** core

---

## Secondary Features (Priority Tier 2)

### 8. Multi-Format Output
Structured output formats for search results: JSON, CSV, Markdown, XML.

- **Description:** Search and multi-get commands support `--json`, `--csv`, `--md`, `--xml`, and `--files` output formats. JSON includes snippets and scores; `--files` outputs docid, score, filepath, and context for agent workflows.
- **Key Files:** `src/cli/formatter.ts`, `src/cli/qmd.ts`
- **Priority:** secondary

### 9. Document Retrieval
Fetch documents by path, short docid, or glob patterns with optional line ranges.

- **Description:** `qmd get` retrieves a document by collection-relative path or 6-character docid (hash of content). Supports fuzzy matching suggestions for typos. `qmd multi-get` fetches multiple documents via glob pattern, comma-separated list, or docids with configurable max file size.
- **Key Files:** `src/store.ts` (get, multiGet), `src/cli/qmd.ts` (get, multi-get commands)
- **Priority:** secondary

### 10. HTTP Transport for MCP
Long-lived MCP daemon with persistent model loading across requests.

- **Description:** `qmd mcp --http --daemon` starts a background daemon (PID stored in `~/.cache/qmd/mcp.pid`). LLM models stay loaded in VRAM between requests. Embedding/reranking contexts are disposed after 5 min idle and recreated on next request (~1s penalty). Exposes `/mcp` (POST) and `/health` (GET) endpoints.
- **Key Files:** `src/mcp/server.ts` (HTTP transport), `src/cli/qmd.ts` (mcp command)
- **Priority:** secondary

### 11. Custom Embedding Models
Support for alternative embedding models via environment variable configuration.

- **Description:** `QMD_EMBED_MODEL` environment variable overrides the default `embeddinggemma-300M`. Supported families: `embeddinggemma` (English-optimized) and `Qwen3-Embedding` (multilingual, 119 languages including CJK). Switching models requires re-embedding with `qmd embed -f`.
- **Key Files:** `src/llm.ts` (model configuration), `src/store.ts` (embedding operations)
- **Priority:** secondary

### 12. Index Maintenance
Re-indexing, cleanup, and health monitoring for the SQLite-based index.

- **Description:** `qmd update` rescans filesystem and updates the SQLite index. `--pull` option runs `git pull` first for remote repos. `qmd cleanup` removes orphaned data. `qmd status` shows collection stats, context counts, and MCP daemon status. Index stored at `~/.cache/qmd/index.sqlite`.
- **Key Files:** `src/store.ts` (update, cleanup), `src/maintenance.ts`, `src/cli/qmd.ts`
- **Priority:** secondary

---

## Feature Summary Table

| Feature | Tier | Key Files |
|---------|------|-----------|
| Hybrid Search Pipeline | core | `src/store.ts`, `src/llm.ts`, `src/index.ts` |
| Collection Management | core | `src/collections.ts`, `src/cli/qmd.ts` |
| Context System | core | `src/store.ts`, `src/db.ts` |
| MCP Server Integration | core | `src/mcp/server.ts`, `src/cli/qmd.ts` |
| SDK/Library API | core | `src/store.ts`, `src/index.ts` |
| Smart Document Chunking | core | `src/index.ts`, `src/store.ts` |
| Query Expansion Model | core | `finetune/train.py`, `finetune/eval.py` |
| Multi-Format Output | secondary | `src/cli/formatter.ts`, `src/cli/qmd.ts` |
| Document Retrieval | secondary | `src/store.ts`, `src/cli/qmd.ts` |
| HTTP Transport for MCP | secondary | `src/mcp/server.ts`, `src/cli/qmd.ts` |
| Custom Embedding Models | secondary | `src/llm.ts`, `src/store.ts` |
| Index Maintenance | secondary | `src/store.ts`, `src/maintenance.ts` |

---

## Architecture Overview

```
QMD Hybrid Search Pipeline
├── Query Expansion (fine-tuned Qwen3-1.7B)
│   ├── hyde: hypothetical document passage
│   ├── lex: BM25 keyword variations
│   └── vec: semantic query variations
├── Retrieval (parallel)
│   ├── BM25 (SQLite FTS5)
│   └── Vector (sqlite-vec)
├── Fusion (RRF with k=60)
│   ├── Original query ×2 weight
│   └── Top-rank bonus (+0.05 for #1, +0.02 for #2-3)
├── Re-ranking (Qwen3-reranker)
└── Position-Aware Blending
    ├── Rank 1-3: 75% retrieval / 25% reranker
    ├── Rank 4-10: 60% retrieval / 40% reranker
    └── Rank 11+: 40% retrieval / 60% reranker
```

---

## Dependencies

- **Runtime:** Node.js >= 22, Bun >= 1.0.0
- **Search:** SQLite FTS5 (BM25), sqlite-vec (vector), better-sqlite3
- **LLM:** node-llama-cpp (GGUF models), custom query expansion model
- **MCP:** @modelcontextprotocol/sdk
- **Config:** YAML, Zod validation
