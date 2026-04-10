# Architecture & Design Patterns Analysis: memory-lancedb-pro

## Architectural Pattern: Layered Architecture with Plugin Integration

The project follows a **Layered Architecture** (also called Multitier Architecture) with **Plugin Integration** for OpenClaw. This is characteristic of systems that need clear separation of concerns while remaining extensible.

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Plugin Layer                     │
│                      (index.ts - 155KB)                     │
│  - Hooks: onUserMessage, onAssistantMessage, onToolCall     │
│  - Tool registration (MCP protocol)                         │
│  - Session management                                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                   Tool/Service Layer                         │
│                       (tools.ts)                             │
│  - MCP tool implementations                                 │
│  - Memory CRUD operations                                   │
│  - Administrative tools                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                 Retrieval Pipeline Layer                      │
│                    (retriever.ts)                           │
│  - Hybrid search (vector + BM25)                            │
│  - Reranking (cross-encoder)                                │
│  - Score post-processing (recency, decay, diversity)        │
└───────────┬─────────────────────────────┬───────────────────┘
            │                             │
┌───────────▼───────────┐     ┌───────────▼───────────┐
│   Embedding Layer     │     │    Storage Layer       │
│    (embedder.ts)      │     │     (store.ts)         │
│  - OpenAI-compatible  │     │  - LanceDB             │
│  - Multi-key rotation │     │  - Multi-scope         │
│  - LRU cache + TTL   │     │  - FTS index           │
│  - Auto-chunking     │     │                        │
└──────────────────────┘     └────────────────────────┘
```

## Major Modules and Responsibilities

### Core Domain Modules

| Module | Class/Function | Responsibility |
|--------|---------------|---------------|
| `store.ts` | `MemoryStore` | LanceDB persistence, CRUD operations, scope filtering |
| `retriever.ts` | `MemoryRetriever` | Hybrid search, RRF fusion, reranking, score post-processing |
| `embedder.ts` | `Embedder` | OpenAI-compatible embeddings, caching, chunking, key rotation |
| `scopes.ts` | `MemoryScopeManager` | Multi-scope isolation, ACL management |

### Supporting Service Modules

| Module | Purpose |
|--------|---------|
| `smart-extractor.ts` | Intelligent fact extraction from conversations |
| `decay-engine.ts` | Memory importance decay calculations |
| `tier-manager.ts` | Memory tier (core/working/peripheral) lifecycle |
| `memory-compactor.ts` | Old memory consolidation |
| `session-compressor.ts` | Session summarization |
| `access-tracker.ts` | Memory access frequency tracking |
| `noise-filter.ts` | Filter low-value memories |
| `chunker.ts` | Long-text splitting for embedding |
| `llm-client.ts` | LLM API abstraction (supports OAuth) |
| `migrate.ts` | Schema migrations |

### Plugin Integration

| Module | Role |
|--------|------|
| `index.ts` | OpenClaw plugin entry, hook handlers, tool registration |
| `tools.ts` | MCP tool definitions (~74KB, ~30 tools) |
| `cli.ts` | Standalone management CLI |

## Communication Patterns

### 1. Direct Dependency Injection (Primary Pattern)

Components receive dependencies via constructors, enabling testability and loose coupling.

```typescript
// retriever.ts - MemoryRetriever constructor
constructor(
  private store: MemoryStore,
  private embedder: Embedder,
  private config: RetrievalConfig,
  private decayEngine: DecayEngine | null,
) { }
```

### 2. Factory Functions

Creator functions provide controlled instantiation with defaults:

```typescript
// Factory pattern usage
export function createRetriever(
  store: MemoryStore,
  embedder: Embedder,
  config?: Partial<RetrievalConfig>,
  options?: { decayEngine?: DecayEngine | null },
): MemoryRetriever { ... }

export function createEmbedder(config: EmbeddingConfig): Embedder { ... }

export function createScopeManager(config?: Partial<ScopeConfig>): MemoryScopeManager { ... }
```

### 3. Event-like Patterns (Internal)

Access tracking and retrieval tracing use observer-like patterns:

```typescript
// access-tracker.ts - records access without direct coupling
this.accessTracker.recordAccess(results.map((r) => r.entry.id));

// retrieval-trace.ts - TraceCollector stages
trace?.startStage("vector_search", []);
trace?.endStage(mapped.map((r) => r.entry.id), mapped.map((r) => r.score));
```

### 4. Plugin Hooks (External)

OpenClaw invokes plugin via defined hooks:

```typescript
// index.ts exports plugin hooks
export const memoryPlugin: OpenClawPluginApi = {
  onUserMessage: async (event) => { /* capture + store */ },
  onAssistantMessage: async (event) => { /* extract facts */ },
  onToolCall: async (event) => { /* track tool usage */ },
};
```

## Data Flow Diagram

### Memory Storage Flow

```
User Message
    │
    ▼
┌─────────────────┐
│  SmartExtractor │ ──fact extraction──► MemoryStore
│   (on hook)     │                     (LanceDB)
└─────────────────┘
    │
    ▼
┌─────────────────┐
│   Embedder      │ ──vector──► MemoryStore.vector
│  (async HTTP)   │
└─────────────────┘
```

### Memory Retrieval Flow

```
User Query
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                   MemoryRetriever                         │
│  ┌────────────┐    ┌────────────┐                       │
│  │  Embedder  │───►│  (query    │                       │
│  │ (query vec)│    │   vector)  │                       │
│  └────────────┘    └─────┬──────┘                       │
│                          │                               │
│  ┌───────────────────────┼───────────────────────────┐  │
│  │                                               ▼     │  │
│  │  ┌─────────────────┐    ┌─────────────────────┐   │  │
│  │  │  Vector Search  │    │   BM25 Search       │   │  │
│  │  │  (LanceDB)      │    │   (FTS index)       │   │  │
│  │  └────────┬────────┘    └──────────┬──────────┘   │  │
│  │           │                        │              │  │
│  │           └────────┬───────────────┘              │  │
│  │                    ▼                               │  │
│  │            ┌──────────────┐                        │  │
│  │            │ RRF Fusion   │                        │  │
│  │            │ (score comb) │                        │  │
│  │            └──────┬───────┘                        │  │
│  │                   │                               │  │
│  │            ┌──────▼───────┐                       │  │
│  │            │ Cross-Encoder│ (optional)            │  │
│  │            │  Reranking   │                       │  │
│  │            └──────┬───────┘                       │  │
│  │                   │                               │  │
│  │  ┌────────────────┼────────────────────────────────┐ │
│  │  │                ▼                                │ │
│  │  │  Post-processing pipeline:                      │ │
│  │  │  - Recency boost                              │ │
│  │  │  - Importance weight                          │ │
│  │  │  - Length normalization                       │ │
│  │  │  - Time decay                                │ │
│  │  │  - Noise filter                              │ │
│  │  │  - MMR diversity                             │ │
│  │  └───────────────────────────────────────────────┘ │
│  └────────────────────────────────────────────────────┘
    │
    ▼
RetrievalResult[]
```

### Scope Isolation Flow

```
Agent Request
    │
    ▼
┌──────────────────────────────────┐
│   MemoryScopeManager             │
│  getScopeFilter(agentId)          │
│    │                             │
│    ├── System bypass? ──► undefined (full access)
│    │                             │
│    └── Normal agent ──► string[] (allowed scopes)
└──────────────────────────────────┘
    │
    ▼
MemoryStore query with scopeFilter
    │
    ▼
SQL: WHERE scope IN ('global', 'agent:main', ...)
```

## Key Abstractions/Interfaces

### MemoryEntry

Core domain object stored in LanceDB:

```typescript
interface MemoryEntry {
  id: string;
  text: string;
  vector: number[];
  category: "preference" | "fact" | "decision" | "entity" | "other" | "reflection";
  scope: string;
  importance: number;
  timestamp: number;
  metadata?: string; // JSON string for extensible metadata
}
```

### ScopeManager Interface

Abstracts scope enforcement:

```typescript
interface ScopeManager {
  getAccessibleScopes(agentId?: string): string[];
  getScopeFilter?(agentId?: string): string[] | undefined; // Store-layer filter
  getDefaultScope(agentId?: string): string;
  isAccessible(scope: string, agentId?: string): boolean;
  validateScope(scope: string): boolean;
  getAllScopes(): string[];
  getScopeDefinition(scope: string): ScopeDefinition | undefined;
}
```

### RetrievalConfig

Strategy pattern - configurable retrieval behavior:

```typescript
interface RetrievalConfig {
  mode: "hybrid" | "vector";
  vectorWeight: number;
  bm25Weight: number;
  minScore: number;
  rerank: "cross-encoder" | "lightweight" | "none";
  candidatePoolSize: number;
  recencyHalfLifeDays: number;
  recencyWeight: number;
  // ... 20+ configurable parameters
}
```

## Design Patterns in Use

### 1. Strategy Pattern
Multiple retrieval strategies selectable at runtime:
- `mode: "hybrid"` - Combined vector + BM25
- `mode: "vector"` - Vector-only
- Tag prefix detection switches to BM25-only for exact-match queries

### 2. Adapter Pattern
Reranker provider abstraction handles multiple API formats:

```typescript
// Provider-specific request/response handling
function buildRerankRequest(provider, apiKey, model, query, candidates, topN)
function parseRerankResponse(provider, data): RerankItem[]

// Supported: jina, siliconflow, voyage, pinecone, dashscope, tei
```

### 3. Factory Pattern
All major components created via factory functions with defaults:

```typescript
createRetriever(store, embedder, config?, options?)
createEmbedder(config)
createScopeManager(config?)
createMigrator(store, embedder)
```

### 4. Template Method Pattern
Retrieval pipeline stages follow consistent flow:

```
vectorOnlyRetrieval / bm25OnlyRetrieval / hybridRetrieval
    │
    ├── applyRecencyBoost()
    ├── applyImportanceWeight()
    ├── applyLengthNormalization()
    ├── applyTimeDecay() or applyDecayBoost()
    ├── filterNoise()
    └── applyMMRDiversity()
```

### 5. Proxy/Cache Pattern
Embedder uses LRU cache with TTL:

```typescript
class EmbeddingCache {
  private cache = new Map<string, CacheEntry>(); // SHA-256 hash key
  private readonly ttlMs: number; // 30 minute default
  get(text, task?): number[] | undefined
  set(text, task?, vector): void
}
```

### 6. Observer Pattern (Lightweight)
Access tracking and retrieval tracing decouple statistics collection:

```typescript
// Access tracking (optional dependency)
retriever.setAccessTracker(tracker);

// Stats collection (optional dependency)
retriever.setStatsCollector(collector);

// Trace stages (no coupling to actual collector)
trace?.startStage("vector_search", []);
trace?.endStage(ids, scores);
```

### 7. Retry Pattern
Embedder implements key rotation on rate-limit errors:

```typescript
private async embedWithRetry(payload, signal?): Promise<any> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const client = this.nextClient(); // Round-robin
    try {
      return await client.embeddings.create(payload);
    } catch (error) {
      if (this.isRateLimitError(error) && attempt < maxAttempts - 1) {
        continue; // Rotate to next key
      }
      throw error;
    }
  }
}
```

### 8. Update Queue Pattern (Serialized Operations)
MemoryStore serializes updates to prevent race conditions:

```typescript
private updateQueue: Promise<void> = Promise.resolve();

private async runSerializedUpdate<T>(action: () => Promise<T>): Promise<T> {
  const previous = this.updateQueue;
  let release: (() => void) | undefined;
  const lock = new Promise<void>((resolve) => { release = resolve; });
  this.updateQueue = previous.then(() => lock);
  await previous;
  try {
    return await action();
  } finally {
    release?.();
  }
}
```

### 9. File Lock Pattern
Cross-process writes use proper-lockfile:

```typescript
private async runWithFileLock<T>(fn: () => Promise<T>): Promise<T> {
  const lockfile = await loadLockfile();
  const release = await lockfile.lock(lockPath, { retries: {...}, stale: 10000 });
  try {
    return await fn();
  } finally {
    await release();
  }
}
```

## Cross-Cutting Concerns

### Error Handling
- Structured error messages with actionable hints
- Fallback strategies (BM25 fallback, cosine rerank fallback)
- Graceful degradation (FTS unavailable = vector-only)

### Configuration
- Environment variable resolution (`${VAR}` syntax)
- Deep merging of config objects
- Validation with descriptive errors

### Observability
- Retrieval tracing with stage timing
- Stats collection for query patterns
- Diagnostic build tagging (`DIAG_BUILD_TAG`)

### Security
- Scope-based access control
- SQL injection prevention (escape literals)
- Bypass IDs for internal operations (`system`, `undefined`)

## Architectural Trade-offs

| Decision | Trade-off |
|----------|-----------|
| Single LanceDB table for all scopes | Simple, but scope filtering adds SQL overhead |
| JSON metadata field | Flexible, but no queryable without parsing |
| Delete + re-add for updates | LanceDB limitation, but enables vector re-embedding |
| Optional dependency injection (AccessTracker, TierManager) | Flexible, but null-checks scattered in code |
| 45 flat modules in `src/` | Easy to navigate, but some coupling exists |

## Dependencies Between Layers

```
index.ts (Plugin)
    │
    ├── tools.ts (Tool implementations)
    │     │
    │     └── smart-extractor.ts (Fact extraction)
    │
    ├── MemoryRetriever (retriever.ts)
    │     │
    │     ├── MemoryStore (store.ts) - LanceDB
    │     ├── Embedder (embedder.ts) - HTTP
    │     ├── AccessTracker (access-tracker.ts)
    │     ├── DecayEngine (decay-engine.ts)
    │     └── TierManager (tier-manager.ts)
    │
    └── MemoryScopeManager (scopes.ts)
```

## Summary

The architecture is a well-structured **layered system** with:
- Clear separation between storage, retrieval, and plugin integration
- Strategy pattern for configurable retrieval modes
- Adapter pattern for multi-provider reranking
- Factory pattern for object creation
- Extensive use of optional dependencies for flexibility
- Production-grade cross-cutting concerns (caching, retries, file locking, scope isolation)