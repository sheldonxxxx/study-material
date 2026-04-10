# Feature Batch 1 Deep Dive

**Date:** 2026-03-27
**Repository:** `/Users/sheldon/Documents/claw/reference/memory-lancedb-pro`
**Analyzed Features:**
1. Hybrid Retrieval (Vector + BM25)
2. Cross-Encoder Reranking
3. Multi-Scope Isolation

---

## Feature 1: Hybrid Retrieval (Vector + BM25)

### Description
Combines semantic vector similarity search with BM25 full-text keyword matching for recall that is both contextually aware and keyword-exact.

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/retriever.ts` | 1,295 | Hybrid retrieval engine with RRF fusion |
| `src/store.ts` | 1,157 | LanceDB storage with FTS indexing |

### Core Logic Flow

#### 1. Entry Point: `MemoryRetriever.retrieve()`
Location: `src/retriever.ts:393-432`

```typescript
async retrieve(context: RetrievalContext): Promise<RetrievalResult[]>
```

**Decision Tree:**
```
Check query for tag prefixes (e.g. "proj:AIF")
    |
    v
Tag tokens found? --> BM25-only + mustContain filtering
    |
    v (no)
config.mode == "vector" or no FTS support? --> Vector-only retrieval
    |
    v (no)
Standard Hybrid Retrieval
```

#### 2. Hybrid Retrieval Pipeline (`src/retriever.ts:617-704`)

```
1. Parallel Search Stage (Promise.all)
   - Vector search: embedQuery() -> store.vectorSearch()
   - BM25 search: store.bm25Search()

2. RRF Fusion (fuseResults, lines 750-827)
   - Collect all unique document IDs from both result sets
   - Score formula:
     * vectorResult? weightedFusion = (vectorScore * 0.7) + (bm25Score * 0.3)
     * bm25Score >= 0.75? boost with bm25Score * 0.92 (preserve keyword hits)
   - Ghost entry detection (#15 fix): verify BM25-only results still exist

3. Min Score Filter
   - threshold: config.minScore (default 0.3)

4. Optional Rerank (if config.rerank !== "none")

5. Recency Boost (applyRecencyBoost, lines 981-1000)
   - Formula: boost = exp(-ageDays / recencyHalfLifeDays) * recencyWeight
   - Default half-life: 14 days, weight: 0.1

6. Importance Weight (applyImportanceWeight, lines 1009-1020)
   - Formula: score *= (0.7 + 0.3 * importance)

7. Length Normalization (applyLengthNormalization, lines 1047-1070)
   - Penalize long entries via: score *= 1 / (1 + 0.5 * log2(charLen / anchor))
   - Anchor: 500 chars, prevents long entries dominating

8. Hard Cutoff
   - filter: score >= config.hardMinScore (default 0.35)

9. Time Decay (applyTimeDecay, lines 1082-1113)
   - Formula: score *= 0.5 + 0.5 * exp(-ageDays / effectiveHalfLife)
   - Access reinforcement extends half-life for frequently accessed memories

10. Noise Filter
    - Calls filterNoise() from noise-filter.ts

11. MMR Diversity (applyMMRDiversity, lines 1204-1236)
    - Cosine similarity threshold: 0.85
    - Similar entries deferred to end (not removed)
```

#### 3. LanceDB Storage (`src/store.ts`)

**FTS Index Creation** (`src/store.ts:353-373`):
```typescript
await table.createIndex("text", {
  config: (lancedb as any).Index.fts(),
});
```

**Vector Search** (`src/store.ts:479-543`):
- Uses cosine distance: `score = 1 / (1 + distance)`
- Over-fetch multiplier: 10x (or 20x when filtering inactive)
- Max fetch limit: 200

**BM25 Search** (`src/store.ts:545-628`):
- Uses LanceDB FTS: `table.search(query, "fts")`
- Score normalization: sigmoid `1 / (1 + exp(-rawScore / 5))`
- Fallback: `lexicalFallbackSearch()` if FTS unavailable

**Lexical Fallback** (`src/store.ts:630-695`):
- Simple substring matching across text, l0_abstract, l1_overview, l2_content
- Weights: text(1.0), l0_abstract(0.98), l1_overview(0.92), l2_content(0.96)
- Score formula: `Math.max(score, Math.min(0.95, 0.72 + queryLen * 0.02))`

### Notable Patterns

#### Tag Prefix Detection (`src/retriever.ts:477-484`)
```typescript
private extractTagTokens(query: string): string[] {
  const pattern = this.config.tagPrefixes.join("|"); // "proj|env|team|scope"
  const regex = new RegExp(`(?:${pattern}):[\\w-]+`, "gi");
  return matches || [];
}
```
Queries like "proj:AIF" bypass hybrid search entirely, using BM25-only with mustContain filtering. This prevents semantic false positives.

#### Ghost Entry Handling (Fix #15, `src/retriever.ts:776-786`)
```typescript
// BM25-only results may be "ghost" entries whose vector data was
// deleted but whose FTS index entry lingers until the next index rebuild.
if (!vectorResult && bm25Result) {
  const exists = await this.store.hasId(id);
  if (!exists) continue; // Skip ghost entry
}
```

#### Cross-Process Locking (`src/store.ts:205-217`)
```typescript
private async runWithFileLock<T>(fn: () => Promise<T>): Promise<T> {
  const release = await lockfile.lock(lockPath, {
    retries: { retries: 5, factor: 2, minTimeout: 100, maxTimeout: 2000 },
    stale: 10000,
  });
}
```
Uses `proper-lockfile` for atomic write operations across multiple processes.

#### Delete+Readd Pattern (`src/store.ts:956-989`)
LanceDB doesn't support in-place updates. The update method:
1. Fetches current record (rollback candidate)
2. Deletes the record
3. Adds the updated record
4. If add fails, attempts rollback to rollback candidate

### Technical Debt / Concerns

1. **Update race condition**: The delete-readd pattern means concurrent readers might see missing data briefly
2. **FTS index fragility**: FTS index status is cached; if index gets corrupted, detection may fail
3. **Hardcoded weights**: vectorWeight(0.7) and bm25Weight(0.3) are not configurable at runtime
4. **BM25 score normalization**: Sigmoid parameters (5) appear heuristic-based rather than data-driven
5. **Ghost entry detection overhead**: hasId() call for every BM25-only result adds latency

---

## Feature 2: Cross-Encoder Reranking

### Description
Post-retrieval reranking using cross-encoder models from Jina, SiliconFlow, Voyage AI, Pinecone, DashScope, or TEI. Hybrid scoring: 60% cross-encoder + 40% original fused score.

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/retriever.ts` | 1,295 | Reranking logic and provider adapters |
| `src/embedder.ts` | 942 | Embedding abstraction layer |

### Core Logic Flow

#### Entry Point: `rerankResults()` (`src/retriever.ts:833-959`)

```
1. Check if rerank enabled and API key provided
    |
    v
2. Build provider-specific request (buildRerankRequest, lines 175-256)
    |
    v
3. 5-second timeout via AbortController
    |
    v
4. POST to rerankEndpoint
    |
    v (response.ok)
5. Parse response via provider-specific parser (parseRerankResponse, lines 258-335)
    |
    v (parse success)
6. Blend scores: 60% cross-encoder + 40% original fused
    |
    v (parse failure or non-ok response)
7. Fallback: Cosine similarity between query and result vectors
```

#### Provider Request Building (`src/retriever.ts:175-256`)

Each provider has distinct request formats:

| Provider | Request Shape | Auth Header |
|----------|--------------|-------------|
| Jina/SiliconFlow | `{model, query, documents, top_n}` | `Bearer` |
| Voyage | `{model, query, documents, top_k}` | `Bearer` |
| Pinecone | `{model, query, documents: [{text}], top_n, rank_fields}` | `Api-Key` |
| DashScope | `{model, input: {query, documents}}` | `Bearer` |
| TEI | `{query, texts}` | `Bearer` |

#### Response Parsing (`src/retriever.ts:258-335`)

Unified format: `Array<{index: number, score: number}>`

Provider-specific score fields:
- Jina/SiliconFlow: `relevance_score`
- Voyage: `relevance_score`
- Pinecone: `score`
- TEI: `score`

#### Score Blending (`src/retriever.ts:888-921`)

```typescript
const blendedScore = clamp01WithFloor(
  item.score * 0.6 + original.score * 0.4,
  floor,  // Preservation floor based on BM25 score
);
```

**Preservation Floors** (`src/retriever.ts:961-973`):
- BM25 >= 0.75: `score * (unreturned ? 1.0 : 0.95)` - Exact keyword hits preserved
- BM25 >= 0.60: `score * (unreturned ? 0.95 : 0.9)`
- Otherwise: `score * (unreturned ? 0.8 : 0.5)` - More aggressive demotion

**Unreturned candidates** (not in API response):
- Score penalty: `original.score * 0.8`
- Floor: based on BM25 score (see above)

#### Cosine Similarity Fallback (`src/retriever.ts:337-355`)

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  // Returns 0 if norm is 0
  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

When fallback is used, blend is 70% cosine + 30% original (line 942):
```typescript
const combinedScore = result.score * 0.7 + cosineScore * 0.3;
```

### Notable Patterns

#### Abort Controller Timeout (`src/retriever.ts:861-872`)
```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);
const response = await fetch(endpoint, {
  method: "POST",
  headers,
  body: JSON.stringify(body),
  signal: controller.signal,
});
clearTimeout(timeout);
```
5-second timeout prevents stalling the retrieval pipeline.

#### Multi-Provider Adapter Pattern
Both request building and response parsing use switch statements on provider type. This is functional but could benefit from a strategy pattern for extensibility.

#### Error Handling Strategy
1. Invalid response shape: warn + fallback to cosine
2. HTTP error: warn + fallback to cosine
3. Timeout: warn + fallback to cosine
4. Network error: warn + fallback to cosine
5. Reranking completely fails: return original results

### Technical Debt / Concerns

1. **Hardcoded blend weights**: 60/40 cross-encoder/fused and 70/30 cosine/fused are not configurable
2. **No retry on transient failures**: Rate limits, 5xx errors not retried
3. **Provider-specific parsing is fragile**: Multiple fallback paths in each switch case suggest inconsistent API behavior
4. **No circuit breaker**: If rerank service is degraded, every query will try and fail
5. **API key in memory**: Rerank API key stored in config, not a secrets manager

---

## Feature 3: Multi-Scope Isolation

### Description
Memory isolation across `global`, `agent:<id>`, `project:<id>`, `user:<id>`, and `custom:<name>` scopes with configurable agent access control.

### Key Implementation Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/scopes.ts` | 539 | Core scope management |
| `src/clawteam-scope.ts` | 64 | Team-level scope extension |

### Core Logic Flow

#### 1. Scope Patterns (`src/scopes.ts:62-69`)

```typescript
const SCOPE_PATTERNS = {
  GLOBAL: "global",
  AGENT: (agentId: string) => `agent:${agentId}`,
  CUSTOM: (name: string) => `custom:${name}`,
  REFLECTION: (agentId: string) => `reflection:agent:${agentId}`,
  PROJECT: (projectId: string) => `project:${projectId}`,
  USER: (userId: string) => `user:${userId}`,
};
```

#### 2. Access Check Flow (`src/scopes.ts:188-207`)

```typescript
getAccessibleScopes(agentId?: string): string[] {
  if (isSystemBypassId(agentId) || !agentId) {
    return this.getAllScopes();  // Bypass: enumerate all scopes
  }

  // Check explicit ACL first
  const explicitAccess = this.config.agentAccess[normalizedAgentId];
  if (explicitAccess) {
    return withOwnReflectionScope(explicitAccess, normalizedAgentId);
  }

  // Default: global + agent's own scope + reflection scope
  return withOwnReflectionScope([
    "global",
    SCOPE_PATTERNS.AGENT(normalizedAgentId),
  ], normalizedAgentId);
}
```

#### 3. Store Layer Filter Semantics (`src/scopes.ts:222-230`)

```typescript
getScopeFilter(agentId?: string): string[] | undefined {
  if (!agentId || isSystemBypassId(agentId)) {
    return undefined;  // Full bypass, no scope filtering
  }
  return this.getAccessibleScopes(agentId);
}
```

| Return Value | Store Behavior |
|-------------|----------------|
| `undefined` | No scope filtering (full bypass) |
| `[]` | Deny all reads |
| `["global", ...]` | Restrict to listed scopes |

#### 4. Write Scope Resolution (`src/scopes.ts:232-251`)

```typescript
getDefaultScope(agentId?: string): string {
  if (!agentId) return this.config.default;

  if (isSystemBypassId(agentId)) {
    throw new Error("Reserved bypass agent must provide explicit write scope");
  }

  // For agents with access, default to their private agent scope
  const agentScope = SCOPE_PATTERNS.AGENT(agentId);
  const accessibleScopes = this.getAccessibleScopes(agentId);

  if (accessibleScopes.includes(agentScope)) {
    return agentScope;  // Agents write to their own scope by default
  }
  return this.config.default;
}
```

#### 5. Scope Validation (`src/scopes.ts:263-275`)

```typescript
validateScope(scope: string): boolean {
  return (
    this.config.definitions[trimmedScope] !== undefined ||
    this.isBuiltInScope(trimmedScope)  // global, agent:*, custom:*, project:*, user:*, reflection:*
  );
}
```

#### 6. ClawTeam Scope Extension (`src/clawteam-scope.ts:37-63`)

```typescript
export function applyClawteamScopes(
  scopeManager: MemoryScopeManager,
  scopes: string[],
): void {
  // 1. Register unknown scope definitions
  // 2. Wrap getAccessibleScopes to inject extra scopes for all agents
  scopeManager.getAccessibleScopes = (agentId?: string): string[] => {
    const base = originalGetAccessibleScopes(agentId);
    const result = [...base];
    for (const s of scopes) {
      if (!result.includes(s)) result.push(s);
    }
    return result;
  };
}
```

Environment variable driven: `CLAWTEAM_MEMORY_SCOPE` env var parsed via `parseClawteamScopes()`.

### Notable Patterns

#### System Bypass ID Set (`src/scopes.ts:71`)
```typescript
const SYSTEM_BYPASS_IDS = new Set(["system", "undefined"]);
```
Internal system tasks use these IDs to bypass scope filtering entirely.

#### Session Key Parsing (`src/scopes.ts:91-103`)
```typescript
export function parseAgentIdFromSessionKey(sessionKey: string | undefined): string | undefined {
  // "agent:main:discord:channel:123" -> "main"
  // "agent:main" -> "main"
  if (!sk.startsWith("agent:")) return undefined;
  const rest = sk.slice("agent:".length);
  const colonIdx = rest.indexOf(":");
  const candidate = (colonIdx === -1 ? rest : rest.slice(0, colonIdx)).trim();
}
```

#### Reflection Scope Auto-Grant (`src/scopes.ts:105-108`)
```typescript
function withOwnReflectionScope(scopes: string[], agentId: string): string[] {
  const reflectionScope = SCOPE_PATTERNS.REFLECTION(agentId);
  return scopes.includes(reflectionScope) ? [...scopes] : [...scopes, reflectionScope];
}
```
Agents always have access to their own reflection scope, even if not explicitly configured.

#### Configuration Import with Validation (`src/scopes.ts:375-405`)
```typescript
importConfig(config: Partial<ScopeConfig>): void {
  // Build new config, suppress warnings during validation
  // Rollback on failure, emit warnings only after success
}
```

### Data Structures

#### ScopeConfig (`src/scopes.ts:15-19`)
```typescript
interface ScopeConfig {
  default: string;
  definitions: Record<string, ScopeDefinition>;
  agentAccess: Record<string, string[]>;  // agentId -> accessible scopes
}
```

#### ScopeDefinition (`src/scopes.ts:10-13`)
```typescript
interface ScopeDefinition {
  description: string;
  metadata?: Record<string, unknown>;
}
```

### Technical Debt / Concerns

1. **Monomorphic scope patterns**: Scope types are hardcoded; adding new patterns requires code changes
2. **No hierarchical scopes**: Scopes like `project:X` don't inherit from anything; no wildcards
3. **Reflection scope counted as "user"** (`src/scopes.ts:433-436`):
   ```typescript
   // TODO: add a dedicated `reflection` bucket once downstream dashboards accept it.
   // For now, reflection scopes are counted under `user` for schema compatibility.
   ```
4. **Bypass ID "undefined"**: Using "undefined" as a bypass ID is a magic string that could cause issues
5. **No scope delegation**: A scope cannot grant access to another scope (no transitive access)
6. **Global scope is special-cased**: Cannot be removed, always exists, but the logic is scattered

---

## Cross-Feature Observations

### 1. Configuration-Driven Design
All three features rely heavily on configuration objects that are merged with defaults. This allows runtime customization but creates complex configuration validation logic.

### 2. Error Handling Philosophy
- **Hybrid Retrieval**: Fail-open for ghost entries, fallback to lexical search
- **Cross-Encoder Reranking**: Fail-open with cosine fallback, timeout protection
- **Multi-Scope**: Fail-closed for invalid scopes, bypass for system tasks

### 3. Performance Optimizations
- Over-fetching in vector/BM25 search (10-20x) to account for filtering
- LRU cache with TTL for embeddings (256 entries, 30 min)
- Round-robin API key rotation for embeddings
- Cross-process file locking for writes

### 4. Consistency Issues
- **Weight configuration**: Vector(0.7)/BM25(0.3) hardcoded, but rerank blend(0.6/0.4) also hardcoded
- **Normalization**: BM25 uses sigmoid, vector uses inverse distance, both different approaches
- **Scope filter behavior**: Empty array `[]` is explicit deny-all, `undefined` is bypass

---

## Summary Table

| Aspect | Hybrid Retrieval | Cross-Encoder Reranking | Multi-Scope |
|--------|-----------------|----------------------|-------------|
| **Primary File** | retriever.ts | retriever.ts | scopes.ts |
| **Storage Dependency** | LanceDB | External API | In-memory config |
| **Fallback Strategy** | Lexical search | Cosine similarity | System bypass |
| **Key Algorithm** | RRF fusion | Score blending | ACL enumeration |
| **External I/O** | None | Rerank API (5s timeout) | None |
| **Configurability** | Medium | Low | High |
