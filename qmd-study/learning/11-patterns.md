# QMD Design Patterns

**Repo path:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Output path:** `/Users/sheldon/Documents/claw/qmd-study/learning/11-patterns.md`

---

## Pattern Catalog

### 1. Repository Pattern (Database Access)

**Purpose**: Encapsulate all database access behind clean interfaces.

**File**: `src/store.ts`

**Evidence**:
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

**Benefit**: All SQL queries are centralized; swapping the database backend requires only changing these functions.

---

### 2. Factory Pattern (Store Creation)

**Purpose**: Centralize object creation logic behind a unified entry point.

**Files**: `src/store.ts`, `src/index.ts`

**Evidence**:
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

**Benefit**: Consumers never instantiate Store directly; the factory handles initialization, configuration, and resource setup.

---

### 3. Strategy Pattern (Query Types)

**Purpose**: Support multiple query strategies that can be selected at runtime.

**Files**: `src/store.ts`, `src/llm.ts`

**Evidence**:
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

**Benefit**: Hybrid query can combine BM25 (lex), vector (vec), and HyDE (hyde) strategies without hardcoding the combination.

---

### 4. Session/Context Manager Pattern (LLM Resources)

**Purpose**: Guarantee release of scarce LLM resources (GPU memory, model contexts) even on errors.

**File**: `src/llm.ts` (lines 1276-1502)

**Evidence**:
```typescript
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

// Usage - guaranteed release
await withLLMSession(async (session) => {
  // session guaranteed to release even on error
}, { maxDuration: 10 * 60 * 1000 });
```

**Benefit**: Prevents GPU memory leaks when queries timeout or fail; resources cleaned up after 5 minutes of inactivity.

---

### 5. Lazy Initialization (LLM Models)

**Purpose**: Defer expensive model loading until first use; prevent concurrent loading of the same model.

**File**: `src/llm.ts` (lines 577-601)

**Evidence**:
```typescript
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

**Benefit**: Applications that never query embeddings avoid loading the embedding model entirely.

---

### 6. Content-Addressable Storage

**Purpose**: Deduplicate document content by hashing; store content once, reference by hash.

**File**: `src/store.ts`

**Evidence**:
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

**Benefit**: Identical files across collections or after rename only stored once; FTS index references content via hash.

---

### 7. Hierarchical Context Inheritance

**Purpose**: Allow context metadata to be specified at multiple levels (global, collection, subdirectory) and inherit downward.

**File**: `src/store.ts` (lines 2270-2308)

**Evidence**:
```typescript
// getContextForPath - collects ALL matching contexts from most general to most specific
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

**Benefit**: Subdirectories inherit parent context but can override or extend it; global context applies everywhere.

---

### 8. Dual-Write Pattern (Config Persistence)

**Purpose**: Maintain configuration in both human-editable (YAML) and runtime-efficient (SQLite) formats.

**File**: `src/index.ts` (lines 419-437), `src/collections.ts`

**Evidence**:
```typescript
// index.ts:419-437 - addCollection
addCollection: async (name, opts) => {
  upsertStoreCollection(db, name, {...});  // Write to SQLite
  if (hasYamlConfig || options.config) {
    collectionsAddCollection(name, opts.path, opts.pattern);  // Write to YAML
  }
}
```

**Benefit**: Users can edit YAML directly; SDK can use inline config without YAML; runtime queries use SQLite.

---

### 9. Trigger-Based Denormalization (FTS Sync)

**Purpose**: Automatically keep the FTS index in sync with the documents table without manual updates.

**File**: `src/store.ts` (lines 745-780)

**Evidence**:
```typescript
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

**Benefit**: Insert/update/delete on documents table automatically updates FTS; no application-level sync logic needed.

---

### 10. Facade Pattern (SDK vs Internal Store)

**Purpose**: Provide an ergonomic async API externally while keeping an imperative internal API.

**Files**: `src/index.ts`, `src/store.ts`

**Evidence**:
```typescript
// QMDStore interface (index.ts:212-306) - async, ergonomic
export interface QMDStore {
  search(options: SearchOptions): Promise<HybridQueryResult[]>;
  get(docid: string): Promise<DocumentResult | null>;
  // ...
}

// Internal Store type (store.ts:983-1054) - sync, imperative
export type Store = {
  db: Database;
  llm?: LlamaCpp;
  close: () => void;
  // ...
}
```

**Benefit**: Public API is simpler (async/await, promises); internal API is more flexible and avoids promise overhead in hot paths.

---

### 11. Visitor/Registration Pattern (MCP Tools)

**Purpose**: Register tools and resources with the MCP server using a declarative schema.

**File**: `src/mcp/server.ts` (lines 224-341)

**Evidence**:
```typescript
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

**Benefit**: Adding a new MCP tool requires only registering it; the server handles protocol details.

---

### 12. Hybrid Search Fusion (RRF)

**Purpose**: Combine multiple retrieval strategies (BM25, vector, HyDE) into a single ranked result set.

**File**: `src/store.ts` (hybridQuery function, ~3712+)

**Evidence**:
```typescript
// Reciprocal Rank Fusion
// RRF score = sum(1 / (k + rank_i)) for each strategy's ranking
const fused = new Map<string, { doc: SearchResult; scores: Record<string, number> }>();
for (const [strategy, results] of Object.entries(strategyResults)) {
  for (let i = 0; i < results.length; i++) {
    const rrfScore = 1 / (60 + i);  // k=60 constant
    // combine scores...
  }
}
```

**Benefit**: No single strategy dominates; weak signals from multiple strategies can combine to surface relevant results.

---

## Pattern Summary Table

| Pattern | File(s) | Key Lines | Purpose |
|---------|---------|-----------|---------|
| Repository | store.ts | 783-916, 2650-2900 | Database access abstraction |
| Factory | store.ts, index.ts | 1456, 333 | Centralized object creation |
| Strategy | store.ts, llm.ts | 242-247, 1013-1100 | Runtime query strategy selection |
| Session/Context Manager | llm.ts | 1276-1502 | Guaranteed LLM resource release |
| Lazy Initialization | llm.ts | 577-601 | Deferred expensive operations |
| Content-Addressable Storage | store.ts | 659-665, 1132 | Deduplication via hashing |
| Hierarchical Context | store.ts | 2270-2308 | Context inheritance chain |
| Dual-Write | index.ts | 419-437 | YAML + SQLite config sync |
| Trigger-Based Denormalization | store.ts | 745-780 | Auto-sync FTS index |
| Facade | index.ts, store.ts | 212-306, 983-1054 | Public async vs internal sync API |
| Visitor/Registration | mcp/server.ts | 224-341 | Tool registration with server |
| Hybrid Search Fusion (RRF) | store.ts | 3712+ | Multi-strategy result fusion |
