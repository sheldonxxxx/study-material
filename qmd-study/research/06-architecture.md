# QMD Architecture Analysis

## Architectural Pattern: Layered Monolith with Plugin Extensions

QMD implements a **layered monolith** architecture with an embedded plugin system (MCP server). The codebase is organized around clear separation of concerns:

1. **Core Layer**: SQLite database with FTS5 and sqlite-vec for storage/search
2. **Service Layer**: LLM abstraction (LlamaCpp), search orchestration (hybridQuery, structuredSearch)
3. **Application Layer**: SDK facade (createStore), CLI commands
4. **Interface Layer**: MCP server exposing tools and resources

**Key architectural files and their roles:**

| File | Size | Role |
|------|------|------|
| `src/store.ts` | ~4379 lines | **Monolith core** - DB operations, search, chunking, document management |
| `src/llm.ts` | ~1547 lines | LLM abstraction with node-llama-cpp, session management |
| `src/collections.ts` | ~501 lines | YAML configuration management |
| `src/index.ts` | ~529 lines | SDK facade exposing public API |
| `src/mcp/server.ts` | ~808 lines | MCP protocol server with stdio/HTTP transports |

---

## Module Responsibilities and Boundaries

### 1. `src/store.ts` - The Core Monolith

Despite its name, `store.ts` contains nearly the entire application logic (4379 lines). It is organized into these sections:

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

### 2. `src/llm.ts` - LLM Abstraction Layer

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

### 3. `src/collections.ts` - Configuration Management

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

### 4. `src/index.ts` - SDK Facade

Exposes the public TypeScript API for programmatic access. Creates a `QMDStore` wrapper around the internal `Store`:

```typescript
createStore(options: StoreOptions) -> Promise<QMDStore>
  ├── Creates internal store via createStoreInternal()
  ├── Sets up config source (YAML or inline)
  ├── Creates per-store LlamaCpp instance
  └── Returns QMDStore facade with typed methods
```

**Facade pattern**: The `QMDStore` interface (lines 212-306) provides ergonomic async methods while the internal `Store` (lines 983-1054) is more imperative and synchronous.

### 5. `src/mcp/server.ts` - MCP Protocol Server

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

## How Components Communicate

### Direct Import (Primary Pattern)

Most communication is via direct module imports:

```typescript
// store.ts imports llm.ts types
import { LlamaCpp, ... } from "./llm.js";

// mcp/server.ts imports store
import { createStore, ... } from "../index.js";

// index.ts imports store internals
import { createStore as createStoreInternal, ... } from "./store.js";
```

### Dependency Injection via Store Object

The `Store` type (store.ts lines 983-1054) is the primary DI container:

```typescript
export type Store = {
  db: Database;
  llm?: LlamaCpp;  // Optional override for global singleton
  close: () => void;
  // ... 30+ methods
}
```

### Session Abstraction for LLM Operations

LLM operations use scoped sessions to prevent resource conflicts:

```typescript
// LlamaCpp methods called within session context
await withLLMSessionForLlm(llm, async (session) => {
  const embeddings = await session.embedBatch(texts);
  const reranked = await session.rerank(query, docs);
  return reranked;
}, { maxDuration: 30 * 60 * 1000 });
```

---

## Key Abstractions and Interfaces

### 1. LLM Interface (llm.ts lines 314-346)

```typescript
export interface LLM {
  embed(text: string, options?: EmbedOptions): Promise<EmbeddingResult | null>;
  generate(prompt: string, options?: GenerateOptions): Promise<GenerateResult | null>;
  modelExists(model: string): Promise<ModelInfo>;
  expandQuery(query: string, options?: {...}): Promise<Queryable[]>;
  rerank(query: string, documents: RerankDocument[], options?: RerankOptions): Promise<RerankResult>;
  dispose(): Promise<void>;
}
```

**Swappable backend**: Could implement `LLM` for OpenAI, Anthropic, etc.

### 2. ILLMSession Interface (llm.ts lines 156-165)

```typescript
export interface ILLMSession {
  embed(text: string, options?: EmbedOptions): Promise<EmbeddingResult | null>;
  embedBatch(texts: string[]): Promise<(EmbeddingResult | null)[]>;
  expandQuery(query: string, options?: {...}): Promise<Queryable[]>;
  rerank(query: string, documents: RerankDocument[], options?: RerankOptions): Promise<RerankResult>;
  readonly isValid: boolean;
  readonly signal: AbortSignal;
}
```

### 3. Document Types (store.ts lines 1543-1708)

```typescript
export type DocumentResult = {
  filepath: string;
  displayPath: string;
  title: string;
  context: string | null;
  hash: string;
  docid: string;       // Short hash for CLI display
  collectionName: string;
  modifiedAt: string;
  bodyLength: number;
  body?: string;
};

export type SearchResult = DocumentResult & {
  score: number;        // 0-1 relevance score
  source: "fts" | "vec";
  chunkPos?: number;
};

export type HybridQueryResult = {
  file: string;
  displayPath: string;
  title: string;
  body: string;
  bestChunk: string;   // Reranked chunk for display
  bestChunkPos: number;
  score: number;
  context: string | null;
  docid: string;
  explain?: HybridQueryExplain;  // Debug info if requested
};
```

### 4. Collection Configuration (collections.ts lines 27-49)

```typescript
export interface Collection {
  path: string;
  pattern: string;
  ignore?: string[];
  context?: ContextMap;
  update?: string;           // Pre-index command
  includeByDefault?: boolean;
}

export interface NamedCollection extends Collection {
  name: string;
}
```

### 5. Virtual Path System (store.ts lines 478-607)

Documents are identified by `qmd://collection/path` URIs:

```typescript
export type VirtualPath = {
  collectionName: string;
  path: string;
};

export function parseVirtualPath(virtualPath: string): VirtualPath | null;
export function buildVirtualPath(collectionName: string, path: string): string;
export function resolveVirtualPath(db: Database, virtualPath: string): string | null;
export function toVirtualPath(db: Database, absolutePath: string): string | null;
```

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
│     └─ searchFTS() → documents_fts MATCH                        │
│                                                                 │
│  2. Query expansion (if no strong signal)                       │
│     └─ LlamaCpp.expandQuery() → lex/vec/hyde sub-queries        │
│                                                                 │
│  3. Parallel search execution                                    │
│     ├─ FTS for lex queries (sync)                               │
│     └─ embedBatch() + searchVec() for vec/hyde (async)         │
│                                                                 │
│  4. Reciprocal Rank Fusion (RRF)                                │
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

## Design Patterns in Actual Use

### 1. Repository Pattern (Database Access)

`store.ts` provides a repository-style interface over SQLite:

```typescript
// Collection access
getStoreCollections(db): NamedCollection[]
getStoreCollection(db, name): NamedCollection | null

// Document access
findDocument(db, filename, options): DocumentResult | DocumentNotFound
findDocuments(db, pattern, options): { docs: MultiGetResult[]; errors: string[] }

// Search access
searchFTS(db, query, limit, collectionName?): SearchResult[]
searchVec(db, query, model, limit, collectionName?): Promise<SearchResult[]>
```

### 2. Factory Pattern (Store Creation)

```typescript
// store.ts:1456
export function createStore(dbPath?: string): Store {
  const resolvedPath = dbPath || getDefaultDbPath();
  const db = openDatabase(resolvedPath);
  initializeDatabase(db);
  // ... create and return Store object
}

// index.ts:333
export async function createStore(options: StoreOptions): Promise<QMDStore> {
  const internal = createStoreInternal(options.dbPath);
  // ... wrap with QMDStore facade
}
```

### 3. Strategy Pattern (Query Types)

The hybrid query supports multiple strategies:

```typescript
// store.ts:242-247
export type ExpandedQuery = {
  type: 'lex' | 'vec' | 'hyde';
  query: string;
};

// llm.ts:1013-1100 - expandQuery generates strategies
const queryables: Queryable[] = lines.map(line => {
  const type = line.slice(0, colonIdx).trim();
  if (type !== 'lex' && type !== 'vec' && type !== 'hyde') return null;
  return { type: type as QueryType, text };
});
```

### 4. Session/Context Manager Pattern (LLM Resources)

```typescript
// llm.ts:1276-1502
class LLMSessionManager {
  acquire(): void { _activeSessionCount++; }
  release(): void { _activeSessionCount = Math.max(0, _activeSessionCount - 1); }
  canUnload(): boolean { return _activeSessionCount === 0 && _inFlightOperations === 0; }
}

class LLMSession implements ILLMSession {
  constructor(manager: LLMSessionManager, options: LLMSessionOptions = {}) {
    this.abortController = new AbortController();
    manager.acquire();  // Reference counting
  }

  release(): void {
    this.abortController.abort();
    this.manager.release();
  }
}

// Usage
await withLLMSession(async (session) => {
  // session guaranteed to release even on error
}, { maxDuration: 10 * 60 * 1000 });
```

### 5. Lazy Initialization (LLM Models)

```typescript
// llm.ts:577-601
private async ensureEmbedModel(): Promise<LlamaModel> {
  if (this.embedModel) return this.embedModel;
  if (this.embedModelLoadPromise) {
    return await this.embedModelLoadPromise;  // Prevent concurrent loading
  }
  this.embedModelLoadPromise = (async () => {
    const llama = await this.ensureLlama();
    const modelPath = await this.resolveModel(this.embedModelUri);
    this.embedModel = await llama.loadModel({ modelPath });
    return this.embedModel;
  })();
  return await this.embedModelLoadPromise;
}
```

### 6. Content-Addressable Storage

```typescript
// store.ts:659-665
db.exec(`
  CREATE TABLE IF NOT EXISTS content (
    hash TEXT PRIMARY KEY,  -- SHA256 of content
    doc TEXT NOT NULL,
    created_at TEXT NOT NULL
  )
`);

// store.ts:1132
const hash = await hashContent(content);  // SHA256
insertContent(db, hash, content, now);     // Deduplicated by hash
```

### 7. Hierarchical Context Inheritance

```typescript
// store.ts:2270-2308 - getContextForPath
// Collects ALL matching contexts from most general to most specific
const contexts: string[] = [];
// Global context first
if (globalCtx) contexts.push(globalCtx);
// Collection path contexts (sorted by prefix length)
matchingContexts.sort((a, b) => a.prefix.length - b.prefix.length);
for (const match of matchingContexts) {
  contexts.push(match.context);
}
return contexts.join('\n\n');  // Concatenate with double newline
```

### 8. Dual-Write Pattern (Config Persistence)

```typescript
// index.ts:419-437 - addCollection
addCollection: async (name, opts) => {
  upsertStoreCollection(db, name, {...});  // Write to SQLite
  if (hasYamlConfig || options.config) {
    collectionsAddCollection(name, opts.path, opts.pattern);  // Write to YAML
  }
}
```

### 9. Trigger-Based Denormalization (FTS Sync)

```typescript
// store.ts:745-780 - FTS triggers
db.exec(`
  CREATE TRIGGER IF NOT EXISTS documents_ai AFTER INSERT ON documents
  WHEN new.active = 1
  BEGIN
    INSERT INTO documents_fts(rowid, filepath, title, body)
    SELECT new.id, new.collection || '/' || new.path, new.title,
           (SELECT doc FROM content WHERE hash = new.hash)
    WHERE new.active = 1;
  END
`);
```

### 10. Visitor/Template Pattern (MCP Server Registration)

```typescript
// mcp/server.ts:224-341 - Tool registration
server.registerTool("query", {
  title: "Query",
  description: "...",
  inputSchema: {...},
}, async ({ searches, limit, ... }) => {
  const results = await store.search({ queries, limit, ... });
  return {
    content: [{ type: "text", text: formatSearchSummary(...) }],
    structuredContent: { results: filtered },
  };
});
```

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

## Summary

QMD is a well-structured local-first search engine with clear architectural layers. The monolith in `store.ts` is a deliberate choice for data locality and transaction consistency, not an accidental growth. The LLM layer is cleanly abstracted for swappable backends, and the MCP server provides a standard protocol interface.

The most distinctive patterns are:
- **Content-addressable storage** for deduplication
- **Hierarchical context inheritance** for metadata
- **Hybrid search fusion** combining BM25, vector similarity, and LLM reranking
- **Session-based LLM resource management** with inactivity cleanup
- **Dual-write configuration** for human-editability and runtime efficiency
