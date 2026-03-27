# Design Patterns Catalog: memory-lancedb-pro

## Overview

This document enumerates all design patterns identified in the memory-lancedb-pro codebase, with specific file references and code evidence.

---

## Pattern Index

| # | Pattern | File(s) | Purpose |
|---|---------|---------|---------|
| 1 | Strategy Pattern | `retriever.ts` | Configurable retrieval modes |
| 2 | Adapter Pattern | `retriever.ts` | Multi-provider reranking |
| 3 | Factory Pattern | `retriever.ts`, `embedder.ts`, `scopes.ts`, `migrate.ts` | Object creation with defaults |
| 4 | Template Method Pattern | `retriever.ts` | Retrieval pipeline stages |
| 5 | Proxy/Cache Pattern | `embedder.ts` | LRU cache with TTL |
| 6 | Observer Pattern | `access-tracker.ts`, `retrieval-trace.ts` | Decoupled statistics collection |
| 7 | Retry Pattern | `embedder.ts` | Key rotation on rate-limits |
| 8 | Update Queue Pattern | `store.ts` | Serialized operations |
| 9 | File Lock Pattern | `store.ts` | Cross-process write safety |
| 10 | Dependency Injection | `retriever.ts`, `store.ts` | Constructor-based dependencies |
| 11 | Builder Pattern | `retriever.ts`, `embedder.ts` | Fluent configuration |
| 12 | Null Object Pattern | `decay-engine.ts`, `tier-manager.ts` | Optional dependencies |

---

## 1. Strategy Pattern

**Purpose:** Multiple retrieval strategies selectable at runtime

**File:** `src/retriever.ts`

**Evidence:**

```typescript
interface RetrievalConfig {
  mode: "hybrid" | "vector";
  vectorWeight: number;
  bm25Weight: number;
  // ...
}

// Strategy selection based on config
async retrieve(query, config: RetrievalConfig) {
  if (config.mode === "hybrid") {
    return this.hybridRetrieval(query, config);
  } else {
    return this.vectorOnlyRetrieval(query, config);
  }
}

// Tag prefix detection switches to BM25-only for exact-match queries
if (query.startsWith("tag:")) {
  return this.bm25OnlyRetrieval(query.slice(4), config);
}
```

**Benefit:** Runtime flexibility to choose between hybrid (vector + BM25) or vector-only retrieval based on query type and performance requirements.

---

## 2. Adapter Pattern

**Purpose:** Unified interface across multiple reranker provider APIs

**File:** `src/retriever.ts`

**Evidence:**

```typescript
// Provider-specific request/response handling
function buildRerankRequest(provider, apiKey, model, query, candidates, topN)
function parseRerankResponse(provider, data): RerankItem[]

// Supported providers: jina, siliconflow, voyage, pinecone, dashscope, tei

// Usage in retrieval pipeline
const rerankResults = await this.rerankWithProvider(
  query,
  candidates,
  this.config.rerankProvider,
  this.config.rerankApiKey,
  this.config.rerankModel,
  this.config.rerankTopN
);
```

**Benefit:** Swappable reranking providers without changing the retrieval logic. New providers can be added by implementing the adapter interface.

---

## 3. Factory Pattern

**Purpose:** Controlled object instantiation with sensible defaults

**Files:** `src/retriever.ts`, `src/embedder.ts`, `src/scopes.ts`, `src/migrate.ts`

**Evidence:**

```typescript
// retriever.ts
export function createRetriever(
  store: MemoryStore,
  embedder: Embedder,
  config?: Partial<RetrievalConfig>,
  options?: { decayEngine?: DecayEngine | null },
): MemoryRetriever {
  return new MemoryRetriever(
    store,
    embedder,
    {
      mode: "hybrid",
      vectorWeight: 0.7,
      bm25Weight: 0.3,
      minScore: 0.0,
      rerank: "none",
      // ... defaults
    },
    options?.decayEngine ?? null
  );
}

// embedder.ts
export function createEmbedder(config: EmbeddingConfig): Embedder {
  return new Embedder(
    config.apiKeys,
    config.model ?? "text-embedding-3-small",
    config.embeddingDimensions ?? 1536,
    // ... defaults
  );
}

// scopes.ts
export function createScopeManager(config?: Partial<ScopeConfig>): MemoryScopeManager {
  return new MemoryScopeManager(config ?? {});
}
```

**Benefit:** Centralizes default values, ensures proper initialization order, and provides a single point for configuration changes.

---

## 4. Template Method Pattern

**Purpose:** Consistent retrieval pipeline with swappable steps

**File:** `src/retriever.ts`

**Evidence:**

```typescript
// Main retrieval method defines the skeleton
async retrieve(query, config: RetrievalConfig) {
  const results = config.mode === "hybrid"
    ? await this.hybridRetrieval(query, config)
    : await this.vectorOnlyRetrieval(query, config);

  return this.applyPostProcessing(results, config);
}

// Post-processing pipeline (called identically regardless of retrieval mode)
private applyPostProcessing(results: ScoredEntry[], config: RetrievalConfig): Promise<ScoredEntry[]> {
  let processed = await this.applyRecencyBoost(results, config);
  processed = this.applyImportanceWeight(processed, config);
  processed = this.applyLengthNormalization(processed, config);
  processed = await this.applyTimeDecay(processed, config);
  processed = this.filterNoise(processed, config);
  processed = this.applyMMRDiversity(processed, config);
  return processed;
}
```

**Benefit:** Ensures all retrieval modes go through the same post-processing steps while allowing individual steps to be customized or skipped.

---

## 5. Proxy/Cache Pattern

**Purpose:** Reduce API calls and improve latency with LRU cache and TTL

**File:** `src/embedder.ts`

**Evidence:**

```typescript
class EmbeddingCache {
  private cache = new Map<string, CacheEntry>(); // SHA-256 hash key
  private readonly ttlMs: number; // 30 minute default

  get(text, task?): number[] | undefined {
    const key = this.computeKey(text, task);
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    if (Date.now() - entry.timestamp > this.ttlMs) {
      this.cache.delete(key);
      return undefined;
    }
    // LRU: move to end
    this.cache.delete(key);
    this.cache.set(key, entry);
    return entry.vector;
  }

  set(text, task?, vector): void {
    if (this.cache.size >= this.maxSize) {
      // Evict oldest (first entry)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    const key = this.computeKey(text, task);
    this.cache.set(key, { vector, timestamp: Date.now() });
  }
}
```

**Benefit:** Dramatically reduces API calls for repeated queries, improves latency from ~100ms to ~0.1ms for cache hits.

---

## 6. Observer Pattern (Lightweight)

**Purpose:** Decouple statistics collection from retrieval logic

**Files:** `src/access-tracker.ts`, `src/retrieval-trace.ts`, `src/retriever.ts`

**Evidence:**

```typescript
// retriever.ts - optional observer registration
private accessTracker: AccessTracker | null = null;

setAccessTracker(tracker: AccessTracker) {
  this.accessTracker = tracker;
}

// Called during retrieval without direct coupling
if (this.accessTracker) {
  this.accessTracker.recordAccess(results.map((r) => r.entry.id));
}

// retrieval-trace.ts - TraceCollector with stages
trace?.startStage("vector_search", []);
trace?.endStage(mapped.map((r) => r.entry.id), mapped.map((r) => r.score));
```

**Benefit:** Access tracking and tracing can be enabled/disabled without modifying retrieval logic. Multiple observers can be attached.

---

## 7. Retry Pattern

**Purpose:** Handle transient API failures with key rotation

**File:** `src/embedder.ts`

**Evidence:**

```typescript
private async embedWithRetry(payload, signal?): Promise<any> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const client = this.nextClient(); // Round-robin key selection
    try {
      return await client.embeddings.create(payload);
    } catch (error) {
      if (this.isRateLimitError(error) && attempt < maxAttempts - 1) {
        await this.delay(1000 * Math.pow(2, attempt)); // Exponential backoff
        continue; // Rotate to next key
      }
      throw error;
    }
  }
}

// Key rotation
private keyIndex = 0;
private nextClient() {
  const client = this.clients[this.keyIndex % this.clients.length];
  this.keyIndex++;
  return client;
}
```

**Benefit:** Resilient to rate limiting by rotating through multiple API keys. Exponential backoff prevents hammering failing endpoints.

---

## 8. Update Queue Pattern (Serialized Operations)

**Purpose:** Prevent race conditions in concurrent update scenarios

**File:** `src/store.ts`

**Evidence:**

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

// Usage
async update(id, updates) {
  return this.runSerializedUpdate(async () => {
    // Actual update logic
  });
}
```

**Benefit:** Ensures atomic updates without explicit locks. Simple implementation using promise chaining.

---

## 9. File Lock Pattern

**Purpose:** Safe cross-process writes to shared resources

**File:** `src/store.ts`

**Evidence:**

```typescript
import { loadLockfile } from 'proper-lockfile';

private async runWithFileLock<T>(fn: () => Promise<T>): Promise<T> {
  const lockfile = await loadLockfile();
  const release = await lockfile.lock(this.dbPath, {
    retries: {
      retries: 5,
      minWait: 100,
      maxWait: 1000,
    },
    stale: 10000 // 10 second stale lock tolerance
  });
  try {
    return await fn();
  } finally {
    await release();
  }
}
```

**Benefit:** Prevents corruption from concurrent process access. Proper-lockfile handles crashed processes via stale detection.

---

## 10. Dependency Injection

**Purpose:** Loose coupling and testability through constructor injection

**Files:** `src/retriever.ts`, `src/store.ts`

**Evidence:**

```typescript
// retriever.ts
constructor(
  private store: MemoryStore,
  private embedder: Embedder,
  private config: RetrievalConfig,
  private decayEngine: DecayEngine | null,
  private tierManager: TierManager | null,
  private accessTracker: AccessTracker | null,
) { }

// store.ts
constructor(
  private dbPath: string,
  private scopeManager: MemoryScopeManager,
  private config: StoreConfig,
) { }
```

**Benefit:** Dependencies are explicit and replaceable. Enables mocking for unit tests without database or API access.

---

## 11. Builder Pattern

**Purpose:** Fluent configuration of complex objects

**Files:** `src/retriever.ts`, `src/embedder.ts`

**Evidence:**

```typescript
// Configuration builder usage
const retriever = createRetriever(store, embedder, {
  mode: "hybrid",
  vectorWeight: 0.6,
  bm25Weight: 0.4,
  rerank: "cross-encoder",
  candidatePoolSize: 100,
});

// embedder.ts - options bag pattern
interface EmbedderOptions {
  apiKeys: string[];
  model?: string;
  embeddingDimensions?: number;
  maxBatchSize?: number;
  cacheTtlMs?: number;
  // ... with defaults applied
}
```

**Benefit:** Clear, readable configuration with sensible defaults. Partial objects can be passed for specific overrides.

---

## 12. Null Object Pattern

**Purpose:** Avoid null checks with optional dependencies

**Files:** `src/decay-engine.ts`, `src/tier-manager.ts`, `src/retriever.ts`

**Evidence:**

```typescript
// retriever.ts - optional dependencies handled gracefully
constructor(
  private store: MemoryStore,
  private embedder: Embedder,
  private config: RetrievalConfig,
  private decayEngine: DecayEngine | null = null,  // Null object
  private tierManager: TierManager | null = null,  // Null object
) { }

// Usage with null checks
if (this.decayEngine) {
  results = await this.decayEngine.applyDecay(results);
}

// Or via null-coalescing
const tierManager = this.tierManager ?? defaultTierManager;
```

**Benefit:** Eliminates scattered null checks. Code works correctly whether optional components are present or not.

---

## Pattern Interactions

```
┌─────────────────────────────────────────────────────────────┐
│                      Factory Pattern                         │
│         creates MemoryRetriever with dependencies            │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│    Strategy   │  │   Observer    │  │     Retry     │
│    Pattern    │  │    Pattern    │  │    Pattern    │
│ (retrieval    │  │ (access track │  │  (embedder    │
│   modes)       │  │   + tracing)  │  │   key rot)    │
└───────────────┘  └───────────────┘  └───────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│    Adapter    │  │  Proxy/Cache  │  │  Update Queue │
│   Pattern     │  │   Pattern     │  │   Pattern     │
│ (reranker     │  │  (embedding   │  │   (store     │
│  providers)   │  │    cache)     │  │  serialized)  │
└───────────────┘  └───────────────┘  └───────────────┘
```

---

## Summary

| Pattern | Benefit | Key Files |
|---------|---------|-----------|
| Strategy | Runtime retrieval mode selection | `retriever.ts` |
| Adapter | Swappable reranking providers | `retriever.ts` |
| Factory | Controlled instantiation | `retriever.ts`, `embedder.ts`, `scopes.ts` |
| Template Method | Consistent pipeline | `retriever.ts` |
| Proxy/Cache | Reduced API calls, latency | `embedder.ts` |
| Observer | Decoupled observability | `access-tracker.ts`, `retrieval-trace.ts` |
| Retry | Resilient API calls | `embedder.ts` |
| Update Queue | Race condition prevention | `store.ts` |
| File Lock | Cross-process safety | `store.ts` |
| Dependency Injection | Testability, loose coupling | `retriever.ts`, `store.ts` |
| Builder | Fluent configuration | `retriever.ts`, `embedder.ts` |
| Null Object | Cleaner optional deps | `decay-engine.ts`, `tier-manager.ts` |

The codebase demonstrates mature software architecture with appropriate use of patterns for flexibility, resilience, and maintainability.
