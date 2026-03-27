# QMD Architecture

**Repo path:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Output path:** `/Users/sheldon/Documents/claw/qmd-study/learning/02-architecture.md`

---

## Architectural Pattern: Layered Monolith with Plugin Extensions

QMD implements a **layered monolith** architecture with an embedded plugin system (MCP server). The codebase is organized around clear separation of concerns:

1. **Core Layer**: SQLite database with FTS5 and sqlite-vec for storage/search
2. **Service Layer**: LLM abstraction (LlamaCpp), search orchestration (hybridQuery, structuredSearch)
3. **Application Layer**: SDK facade (createStore), CLI commands
4. **Interface Layer**: MCP server exposing tools and resources

---

## Major Modules and Responsibilities

### `src/store.ts` - The Core Monolith (~4,379 lines)

Despite its name, `store.ts` contains nearly the entire application logic. It is organized into these sections:

| Section | Lines | Responsibility |
|---------|-------|----------------|
| Configuration constants | 38-67 | Defaults, chunk sizes, token limits |
| Smart Chunking | 71-224 | Break point detection, code fence avoidance |
| Virtual Path Utilities | 476-607 | `qmd://` URI parsing and resolution |
| Database Initialization | 610-781 | Schema creation, triggers, migrations |
| Store Collections CRUD | 783-916 | DB-backed collection management |
| Store Factory | 980-1533 | `createStore()` factory function |
| Document Operations | 1536-2360 | Hashing, title extraction, chunking |
| FTS Search | 2650-2814 | `searchFTS()` - BM25 keyword search |
| Vector Search | 2816-2900 | `searchVec()` - sqlite-vec similarity |
| Hybrid Query | 3712+ | `hybridQuery()` - multi-signal fusion |
| Reindex/Embed | 1078-1447 | Filesystem scanning, embedding pipeline |
| Context Management | 2260-2410 | Hierarchical context inheritance |
| Cleanup/Maintenance | 1788-1881 | Orphan cleanup, vacuum, cache management |

**Key architectural insight**: `store.ts` is intentionally monolithic despite size because:
- All database operations are co-located for transaction consistency
- Search functions access internals directly (avoiding interface overhead)
- The Store object is the primary dependency injection mechanism

### `src/llm.ts` - LLM Abstraction Layer (~1,547 lines)

Provides a clean interface to local GGUF models via node-llama-cpp:

```
LLM (interface)
    └── LlamaCpp (implementation)
            ├── embed() / embedBatch() - embedding generation
            ├── expandQuery() - query expansion via fine-tuned model
            ├── rerank() - cross-encoder reranking
            └── generate() - text generation

LLMSessionManager (lifecycle)
    └── LLMSession (scoped wrapper with abort/maxDuration)

withLLMSession<T>() - scoped operation helper
```

**Session management pattern** (lines 1276-1502):
- `LLMSessionManager` tracks active sessions with reference counting
- `LLMSession` wraps operations with abort signals and max duration
- `withLLMSession()` provides try/finally guarantee for session release
- Inactivity timer unloads idle resources after 5 minutes

**Model lifecycle** (lines 575-794):
- Lazy loading: models loaded on first use
- Parallel contexts: GPU VRAM-aware parallelism calculation
- Flash attention: enabled for reranker contexts (20x VRAM reduction)

### `src/collections.ts` - Configuration Management (~501 lines)

Manages the YAML configuration file (`~/.config/qmd/index.yml`) independently from the database:

```
CollectionConfig
    ├── global_context?: string
    └── collections: Record<string, Collection>
            ├── path: string
            ├── pattern: string
            ├── ignore?: string[]
            ├── context?: ContextMap
            └── includeByDefault?: boolean
```

**Dual-write pattern**: Changes write to both SQLite (for runtime queries) and YAML (for persistence). See `syncConfigToDb()` in store.ts line 921.

### `src/index.ts` - SDK Facade (~529 lines)

Exposes the public TypeScript API for programmatic access. Creates a `QMDStore` wrapper around the internal `Store`:

```typescript
createStore(options: StoreOptions) -> Promise<QMDStore>
  ├── Creates internal store via createStoreInternal()
  ├── Sets up config source (YAML or inline)
  ├── Creates per-store LlamaCpp instance
  └── Returns QMDStore facade with typed methods
```

**Facade pattern**: The `QMDStore` interface (lines 212-306) provides ergonomic async methods while the internal `Store` (lines 983-1054) is more imperative and synchronous.

### `src/mcp/server.ts` - MCP Protocol Server (~808 lines)

Exposes QMD as a Model Context Protocol server with two transport options:

| Transport | Pattern | Use Case |
|-----------|---------|----------|
| Stdio | `StdioServerTransport` | CLI integration, local AI tools |
| HTTP | `WebStandardStreamableHTTPServerTransport` | Remote clients, web apps |

**Session-per-client pattern** (lines 550-572):
- Each HTTP client gets its own `McpServer` + `Transport` pair
- Store is shared (SQLite is concurrent-read safe)
- Sessions tracked in a Map for cleanup on close

**REST endpoint** (lines 624-675):
- `POST /query` - structured search without MCP protocol overhead
- Bypasses MCP round-trip for high-frequency queries

---

## Data Flow Through the System

### Search Flow (hybridQuery)

```
User Query
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ hybridQuery() - store.ts:3712                                  │
│                                                                 │
│  1. BM25 probe (strong signal detection)                        │
│     └─ searchFTS() → documents_fts MATCH                       │
│                                                                 │
│  2. Query expansion (if no strong signal)                      │
│     └─ LlamaCpp.expandQuery() → lex/vec/hyde sub-queries        │
│                                                                 │
│  3. Parallel search execution                                   │
│     ├─ FTS for lex queries (sync)                               │
│     └─ embedBatch() + searchVec() for vec/hyde (async)          │
│                                                                 │
│  4. Reciprocal Rank Fusion (RRF)                                 │
│     └─ weighted fusion of ranked lists                          │
│                                                                 │
│  5. Chunk selection per candidate                               │
│     └─ keyword overlap scoring                                  │
│                                                                 │
│  6. LLM reranking (optional)                                    │
│     └─ LlamaCpp.rerank() → cross-encoder scoring                │
│                                                                 │
│  7. Final blending + snippet extraction                         │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Search Results
```

### Indexing Flow (reindexCollection + generateEmbeddings)

```
Collection Path
    │
    ▼
reindexCollection() - store.ts:1078
    │
    ├─ fastGlob() → file list
    ├─ For each file:
    │   ├─ readFileSync()
    │   ├─ hashContent() → SHA256
    │   ├─ extractTitle()
    │   └─ upsert into documents table
    ├─ FTS triggers auto-sync
    └─ cleanupOrphanedContent()

    ▼
generateEmbeddings() - store.ts:1303
    │
    ├─ getPendingEmbeddingDocs()
    ├─ buildEmbeddingBatches()
    ├─ For each batch:
    │   ├─ chunkDocumentByTokens() → 900-token chunks
    │   ├─ session.embedBatch()
    │   ├─ store.ensureVecTable() → sqlite-vec schema
    │   └─ insert into vectors_vec
    │
    ▼
Vector Index Ready
```

### MCP Query Flow (mcp/server.ts)

```
MCP Client
    │
    ├─ POST /mcp (JSON-RPC)
    │   └─ createMcpServer()
    │       ├─ query tool → store.search()
    │       ├─ get tool → store.get() + store.getDocumentBody()
    │       └─ multi_get tool → store.multiGet()
    │
    └─ POST /query (REST shortcut)
        └─ store.search() directly

    ▼
MCP Response (structuredContent + text)
```

---

## Key Architectural Decisions

### 1. SQLite as Primary Store

**Rationale**: Single-file, ACID-compliant, no separate service needed.

**Implications**:
- All data access goes through `store.ts` repository functions
- Concurrent reads allowed, writes serialized
- WAL mode enabled for better concurrency (`store.ts:651`)

### 2. sqlite-vec for Vector Search

**Rationale**: Native SQLite extension, cosine similarity in SQL.

**Constraint** (documented at `store.ts:2827-2830`):
> "IMPORTANT: We use a two-step query approach here because sqlite-vec virtual tables hang indefinitely when combined with JOINs in the same query."

### 3. Local GGUF Models via node-llama-cpp

**Rationale**: Privacy, no API costs, offline capability.

**Trade-offs**:
- VRAM management critical (parallelism calculation at lines 611-629)
- Model loading is lazy with 5-minute inactivity timeout
- Session management prevents context conflicts

### 4. Dual Configuration (YAML + SQLite)

**Rationale**: YAML is human-editable, SQLite is runtime-efficient.

**Sync mechanism**:
- `syncConfigToDb()` uses config hash to skip unnecessary syncs
- Changes via CLI write to both stores
- SDK can use inline config without YAML

### 5. Smart Chunking with Break Point Detection

**Rationale**: Preserve semantic boundaries (headings, code blocks) rather than fixed-size splits.

**Algorithm** (lines 73-224):
- Score potential break points by type (h1=100, h2=90, codeblock=80, etc.)
- Find best cutoff within window using squared distance decay
- Never split inside code fences

### 6. Reranking with Chunk Selection

**Rationale**: Full document reranking is O(tokens); selecting best chunk reduces cost.

**Approach** (lines 3838-3862):
1. Chunk each candidate document
2. Score chunks by keyword overlap with query
3. Rerank only the best chunk per document
4. Re-expand results with full document after reranking

---

## Database Schema

```sql
-- Content-addressable storage (source of truth)
CREATE TABLE content (
  hash TEXT PRIMARY KEY,
  doc TEXT NOT NULL,
  created_at TEXT NOT NULL
);

-- Document metadata (filesystem layer)
CREATE TABLE documents (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  collection TEXT NOT NULL,
  path TEXT NOT NULL,
  title TEXT NOT NULL,
  hash TEXT NOT NULL REFERENCES content(hash),
  created_at TEXT NOT NULL,
  modified_at TEXT NOT NULL,
  active INTEGER NOT NULL DEFAULT 1,
  UNIQUE(collection, path)
);

-- Full-text search index (auto-synced via triggers)
CREATE VIRTUAL TABLE documents_fts USING fts5(
  filepath, title, body,
  tokenize='porter unicode61'
);

-- Vector embeddings (sqlite-vec)
CREATE VIRTUAL TABLE vectors_vec USING vec0(
  hash_seq TEXT PRIMARY KEY,
  embedding float[N] distance_metric=cosine
);

-- Collection configuration (DB-backed, YAML-synced)
CREATE TABLE store_collections (
  name TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  pattern TEXT NOT NULL DEFAULT '**/*.md',
  ignore_patterns TEXT,
  include_by_default INTEGER DEFAULT 1,
  update_command TEXT,
  context TEXT
);

-- LLM response cache
CREATE TABLE llm_cache (
  hash TEXT PRIMARY KEY,
  result TEXT NOT NULL,
  created_at TEXT NOT NULL
);
```

---

## Dependency Graph (Entry Points to Core)

```
bin/qmd (shell)
  └─> dist/cli/qmd.js
        └─> src/cli/qmd.ts
              ├─> src/store.ts (core database operations)
              ├─> src/llm.ts (LlamaCpp wrapper)
              ├─> src/collections.ts (YAML config)
              ├─> src/cli/formatter.ts (output)
              └─> src/db.ts (SQLite setup)

src/index.ts (SDK entry)
  └─> src/store.ts
        ├─> src/db.ts
        └─> src/llm.ts

src/mcp/server.ts (MCP entry)
  └─> src/index.ts (re-exports from store.ts)
```

---

## Summary

QMD is a well-structured local-first search engine with clear architectural layers. The monolith in `store.ts` is a deliberate choice for data locality and transaction consistency, not an accidental growth. The LLM layer is cleanly abstracted for swappable backends, and the MCP server provides a standard protocol interface.

The most distinctive patterns are:
- **Content-addressable storage** for deduplication
- **Hierarchical context inheritance** for metadata
- **Hybrid search fusion** combining BM25, vector similarity, and LLM reranking
- **Session-based LLM resource management** with inactivity cleanup
- **Dual-write configuration** for human-editability and runtime efficiency
