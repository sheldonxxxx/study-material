# Architecture: memory-lancedb-pro

## Overview

**memory-lancedb-pro** is an OpenClaw plugin providing long-term memory capabilities backed by LanceDB. It implements hybrid retrieval (vector + BM25), cross-encoder reranking, multi-scope isolation, and a sophisticated memory lifecycle management system.

**Type:** OpenClaw Plugin (Node.js/TypeScript ESM module)
**Version:** 1.1.0-beta.10
**Repository:** `memory-lancedb-pro`

---

## Architectural Pattern

### Layered Architecture with Plugin Integration

The project follows a **Layered Architecture** with **Plugin Integration** for OpenClaw. This pattern provides clear separation of concerns while remaining extensible through the OpenClaw plugin system.

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Plugin Layer                     │
│                      (index.ts - 155KB)                     │
│  - Hooks: onUserMessage, onAssistantMessage, onToolCall     │
│  - Tool registration (MCP protocol)                         │
│  - Session management                                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                   Tool/Service Layer                          │
│                       (tools.ts)                             │
│  - MCP tool implementations                                  │
│  - Memory CRUD operations                                    │
│  - Administrative tools                                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                 Retrieval Pipeline Layer                      │
│                    (retriever.ts)                            │
│  - Hybrid search (vector + BM25)                            │
│  - Reranking (cross-encoder)                                │
│  - Score post-processing (recency, decay, diversity)         │
└───────────┬─────────────────────────────┬───────────────────┘
            │                             │
┌───────────▼───────────┐     ┌───────────▼───────────┐
│   Embedding Layer     │     │    Storage Layer       │
│    (embedder.ts)      │     │     (store.ts)         │
│  - OpenAI-compatible  │     │  - LanceDB             │
│  - Multi-key rotation │     │  - Multi-scope         │
│  - LRU cache + TTL    │     │  - FTS index           │
│  - Auto-chunking      │     │                        │
└───────────────────────┘     └────────────────────────┘
```

---

## Module Responsibilities and Boundaries

### Core Domain Modules

| Module | Class/Function | Responsibility |
|--------|---------------|----------------|
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

---

## Communication Patterns

### 1. Direct Dependency Injection (Primary Pattern)

Components receive dependencies via constructors, enabling testability and loose coupling.

**Evidence from `retriever.ts`:**

```typescript
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

---

## Data Flow Through the System

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
│   MemoryScopeManager              │
│  getScopeFilter(agentId)         │
│    │                              │
│    ├── System bypass? ──► undefined (full access)
│    │                              │
│    └── Normal agent ──► string[] (allowed scopes)
└──────────────────────────────────┘
    │
    ▼
MemoryStore query with scopeFilter
    │
    ▼
SQL: WHERE scope IN ('global', 'agent:main', ...)
```

---

## Key Abstractions and Interfaces

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
  getScopeFilter?(agentId?: string): string[] | undefined;
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

---

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

---

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

---

## Architectural Trade-offs

| Decision | Trade-off |
|----------|-----------|
| Single LanceDB table for all scopes | Simple, but scope filtering adds SQL overhead |
| JSON metadata field | Flexible, but no queryable without parsing |
| Delete + re-add for updates | LanceDB limitation, but enables vector re-embedding |
| Optional dependency injection (AccessTracker, TierManager) | Flexible, but null-checks scattered in code |
| 45 flat modules in `src/` | Easy to navigate, but some coupling exists |

---

## Summary

The architecture is a well-structured **layered system** with:

- **Clear separation** between storage, retrieval, and plugin integration
- **Strategy pattern** for configurable retrieval modes
- **Adapter pattern** for multi-provider reranking
- **Factory pattern** for object creation
- **Extensive use of optional dependencies** for flexibility
- **Production-grade cross-cutting concerns** (caching, retries, file locking, scope isolation)

The plugin integrates into OpenClaw via hooks (`onUserMessage`, `onAssistantMessage`, `onToolCall`) and exposes tools via the MCP protocol. Memory storage uses LanceDB with a single-table design supporting multi-scope isolation through SQL filtering.
