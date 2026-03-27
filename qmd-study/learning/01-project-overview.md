# Project Overview: QMD

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Version:** v2.0.1 (`@tobilu/qmd`)
**Synthesized:** 2026-03-27

---

## What is QMD?

QMD (Query Markup Documents) is an **on-device search engine** for personal knowledge bases. It indexes markdown notes, meeting transcripts, documentation, and any text-heavy files, enabling keyword and natural language search with hybrid retrieval and LLM-powered re-ranking.

**Core characteristics:**
- Local-only: All data stays on the user's machine
- No cloud dependencies: Models run locally via node-llama-cpp with GGUF files
- Agentic-ready: Designed for AI agent workflows via CLI and MCP (Model Context Protocol)
- Dual-mode package: Works as both a CLI tool and a TypeScript SDK library

---

## Core Purpose and Target Users

### Purpose
QMD solves the "where did I put that?" problem for personal knowledge bases. It provides semantic search over markdown documents without requiring cloud APIs or subscription services.

### Target Users
1. **Individual knowledge workers** with large markdown-based note systems (Obsidian, Logseq, etc.)
2. **Developers** maintaining code documentation and README files
3. **AI practitioners** building agentic workflows that need document retrieval
4. **Privacy-conscious users** who want search without sending data to third parties

---

## High-Level Architecture

### Technology Stack

| Layer | Technology |
|-------|------------|
| Runtime | Node.js 22+ or Bun |
| Language | TypeScript (strict mode) |
| Package Manager | Bun (primary), npm compatible |
| Test Runner | Vitest |
| Database | SQLite + sqlite-vec (vector extension) + better-sqlite3 |
| Full-Text Search | SQLite FTS5 (BM25) |
| Vector Search | sqlite-vec |
| LLM/Embeddings | node-llama-cpp (GGUF models) |
| MCP | @modelcontextprotocol/sdk |
| Config | YAML (collections.ts) |
| Glob | fast-glob, picomatch |

### Search Pipeline Architecture

```
User Query
     │
     ├── Query Expansion (fine-tuned Qwen3) ──► Original + 2 variants
     │                                                    │
     │                      ┌────────────────────────────┼────────────────────────────┐
     │                      ▼                            ▼                            ▼
     │              ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
     │              │ BM25 (FTS5) │            │ BM25 (FTS5) │            │ BM25 (FTS5) │
     │              └──────┬───────┘            └──────┬───────┘            └──────┬───────┘
     │              ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
     │              │   Vector     │            │   Vector     │            │   Vector     │
     │              │   (sqlite-   │            │   (sqlite-   │            │   (sqlite-   │
     │              │    vec)      │            │    vec)      │            │    vec)      │
     │              └──────┬───────┘            └──────┬───────┘            └──────┬───────┘
     │                      │                            │                            │
     │                      └────────────────────────────┼────────────────────────────┘
     │                                                     │
     │                                         RRF Fusion (k=60)
     │                                         Original query ×2 weight
     │                                         Top-rank bonus: +0.05/#1, +0.02/#2-3
     │                                                     │
     │                                          Top 30 candidates
     │                                                     │
     │                                         LLM Re-ranking (Qwen3-Reranker)
     │                                                     │
     │                                         Position-Aware Blending
     │                                         Rank 1-3:  75% RRF / 25% reranker
     │                                         Rank 4-10: 60% RRF / 40% reranker
     │                                         Rank 11+:  40% RRF / 60% reranker
     │                                                     │
     └─────────────────────────────────────────► Final Results
```

### Local Models (Auto-Downloaded)

| Model | Purpose | Size |
|-------|---------|------|
| `embeddinggemma-300M-Q8_0` | Vector embeddings (default) | ~300MB |
| `qwen3-reranker-0.6b-q8_0` | Re-ranking | ~640MB |
| `qmd-query-expansion-1.7B-q4_k_m` | Query expansion (fine-tuned) | ~1.1GB |

Models cached at `~/.cache/qmd/models/` with ETag-based refresh.

### Data Storage

- **Index:** `~/.cache/qmd/index.sqlite` (SQLite with FTS5 and sqlite-vec)
- **Config:** `~/.config/qmd/index.yml` (collections, contexts)
- **Cache:** `~/.cache/qmd/` (XDG Base Directory compliant)

### Schema

```sql
collections      -- Indexed directories with name and glob patterns
path_contexts    -- Context descriptions by virtual path (qmd://...)
documents        -- Markdown content with metadata and docid (6-char hash)
documents_fts    -- FTS5 full-text index
content_vectors  -- Embedding chunks (hash, seq, pos, 900 tokens each)
vectors_vec      -- sqlite-vec vector index (hash_seq key)
llm_cache        -- Cached LLM responses (query expansion, rerank scores)
```

---

## Key Differentiators

### 1. Context System
QMD's unique **context tree** feature allows users to attach descriptive metadata to collection paths. When a document matches, its context is returned alongside it, providing LLMs with better understanding of document purpose.

```bash
qmd context add qmd://notes "Personal notes and ideas"
qmd context add qmd://docs "Work documentation"
```

### 2. Smart Chunking
Documents are chunked at ~900 tokens with 15% overlap, but the algorithm prefers natural markdown boundaries (headings, code blocks, paragraphs) over hard token cuts.

**Break point scoring:**
- `# Heading`: 100 points (major section)
- `## Heading`: 90 points (subsection)
- Code fences: 80 points
- Horizontal rules: 60 points
- Paragraph breaks: 20 points

### 3. Hybrid Search with Position-Aware Blending
Unlike simple RRF fusion, QMD uses position-aware blending that preserves exact matches at top ranks while trusting the reranker more for lower ranks.

### 4. Dual Transport MCP Server
- **Stdio transport:** For Claude Desktop and subprocess-based integration
- **HTTP transport:** For shared long-lived server with VRAM persistence across requests

### 5. No Cloud Dependencies
All inference runs locally via node-llama-cpp. No API keys, no data leaving the machine.

---

## Entry Points

### CLI Entry Point
```
bin/qmd (shell wrapper) -> dist/cli/qmd.js -> src/cli/qmd.ts
```

### SDK Entry Point
```typescript
import { createStore } from '@tobilu/qmd'
const store = await createStore({ dbPath: './index.sqlite', config: {...} })
```

### MCP Server Entry Point
```
qmd mcp           # stdio transport
qmd mcp --http    # HTTP transport (localhost:8181)
```

---

## Directory Structure

```
qmd/
├── bin/qmd                    # Shell wrapper (runtime detection)
├── src/
│   ├── index.ts               # SDK exports (~530 lines)
│   ├── store.ts               # Core DB/search operations (~4,200 lines)
│   ├── llm.ts                 # LlamaCpp wrapper (~1,400 lines)
│   ├── cli/qmd.ts             # CLI commands (~3,200 lines)
│   ├── cli/formatter.ts       # Output formatters
│   ├── mcp/server.ts          # MCP server (~850 lines)
│   └── ...
├── test/                      # Vitest test suite (16 test files)
├── skills/                    # Claude Code skills
├── docs/SYNTAX.md             # QMD file syntax
└── finetune/                  # Python model fine-tuning (separate)
```

---

## Notable Patterns

### Pattern 1: Dual-Mode Package
QMD operates as both CLI tool and SDK library via package.json exports.

### Pattern 2: Shell Wrapper for Runtime Detection
`bin/qmd` detects node vs bun via lockfile presence, preventing ABI mismatches with native modules.

### Pattern 3: Single Massive Core File
`src/store.ts` (~154KB) contains all core operations. Organic growth pattern prioritizing cohesion.

### Pattern 4: Test Colocation
Tests in `test/` directory (not `src/*.test.ts`), running against compiled `dist/` or directly via `tsx`.

---

## Commands Summary

| Command Group | Description |
|---------------|-------------|
| `collection` | Add, list, remove, rename collections |
| `context` | Add, list, check, remove context for paths |
| `get` | Retrieve single document by path or docid |
| `multi-get` | Batch retrieve by glob or docids |
| `embed` | Generate vector embeddings |
| `search` | BM25 keyword search |
| `vsearch` | Vector similarity search |
| `query` | Hybrid search with expansion + reranking (recommended) |
| `rerank` | Rerank pre-fetched results |
| `mcp` | Start MCP server (stdio or HTTP) |
| `status` | Show index status |

---

## Requirements

- **Node.js** >= 22 or **Bun** >= 1.0.0
- **macOS:** Homebrew SQLite for extension support
- **GGUF Models:** Auto-downloaded from HuggingFace on first use
