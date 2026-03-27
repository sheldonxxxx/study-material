# Feature Deep Dive: Batch 1

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Analysis Date:** 2026-03-27
**Features Analyzed:**
1. Hybrid Search Pipeline (BM25 + vector + RRF + LLM reranking)
2. Collection Management (Directory indexing with glob patterns)
3. Context System (Hierarchical metadata for path-based search improvement)

---

## 1. Hybrid Search Pipeline

**Core File:** `src/store.ts` (lines 3712-3990)
**Supporting Files:** `src/llm.ts`, `src/db.ts`

### Overview

The hybrid search pipeline combines multiple retrieval signals into a unified ranking. It uses BM25 keyword search, vector semantic search, Reciprocal Rank Fusion (RRF), and LLM-based reranking.

### Pipeline Architecture

```
Query Input
    │
    ▼
┌─────────────────────────────┐
│ Step 1: BM25 Probe          │  Fast FTS5 query to detect strong signal
│ (Strong Signal Bypass)       │  If score >= 0.85 AND gap >= 0.15 → skip LLM expansion
└─────────────────────────────┘
    │
    ▼ (if no strong signal)
┌─────────────────────────────┐
│ Step 2: Query Expansion     │  LLM generates lex/vec/hyde variants
│ (Qwen3 model)               │  Grammar-constrained output: "lex: ...", "vec: ...", "hyde: ..."
└─────────────────────────────┘
    │
    ├──► FTS5 (BM25) for 'lex' queries
    │
    └──► Batch Embed + sqlite-vec for 'vec'/'hyde' queries
              │
              ▼
┌─────────────────────────────┐
│ Step 3: RRF Fusion          │  Combine ranked lists with weights
│ k=60, first 2 lists get 2x  │  Top 40 candidates advance
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ Step 4: Chunk Selection     │  Smart chunking: 900 tokens, 15% overlap
│ (Keyword-based best chunk)   │  Picks chunk with most query term overlap
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ Step 5: LLM Reranking       │  Reranks chunks (NOT full bodies)
│ (Qwen3-reranker model)      │  Cached results by chunk text
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ Step 6: Position-Aware      │  Top 3: 75% RRF, 25% rerank
│ Score Blending              │  Ranks 4-10: 60% RRF, 40% rerank
│                             │  Ranks 11+: 40% RRF, 60% rerank
└─────────────────────────────┘
    │
    ▼
Dedup + Filter + Limit → Results
```

### Key Implementation Details

#### Strong Signal Bypass (lines 3733-3746)

```typescript
const topScore = initialFts[0]?.score ?? 0;
const secondScore = initialFts[1]?.score ?? 0;
const hasStrongSignal = !intent && initialFts.length > 0
  && topScore >= STRONG_SIGNAL_MIN_SCORE  // 0.85
  && (topScore - secondScore) >= STRONG_SIGNAL_MIN_GAP;  // 0.15
```

**Insight:** The intent parameter disables this bypass—important for domain-specific queries where obvious matches may not be relevant.

#### Query Expansion (llm.ts lines 1013-1093)

```typescript
const grammar = await llama.createGrammar({
  grammar: `
    root ::= line+
    line ::= type ": " content "\\n"
    type ::= "lex" | "vec" | "hyde"
    content ::= [^\\n]+
  `
});
const prompt = intent
  ? `/no_think Expand this search query: ${query}\nQuery intent: ${intent}`
  : `/no_think Expand this search query: ${query}`;
```

**Query Types:**
- `lex`: Keyword-focused variations for BM25
- `vec`: Semantic variations for vector search
- `hyde`: Hypothetical Document Embedding (generates ideal answer description)

**Fallback:** If expansion fails or produces no valid results:
```typescript
const fallback: Queryable[] = [
  { type: 'hyde', text: `Information about ${query}` },
  { type: 'lex', text: query },
  { type: 'vec', text: query },
];
```

#### RRF Fusion (store.ts lines 3056-3085)

```typescript
export function reciprocalRankFusion(
  resultLists: RankedResult[][],
  weights: number[] = [],
  k: number = 60
): RankedResult[] {
  const scores = new Map<string, { result: RankedResult; rrfScore: number; topRank: number }>();

  for (let listIdx = 0; listIdx < resultLists.length; listIdx++) {
    const list = resultLists[listIdx];
    const weight = weights[listIdx] ?? 1.0;

    for (let rank = 0; rank < list.length; rank++) {
      const result = list[rank];
      const rrfContribution = weight / (k + rank + 1);

      if (existing) {
        existing.rrfScore += rrfContribution;
        existing.topRank = Math.min(existing.topRank, rank);
      } else {
        scores.set(result.file, {
          result, rrfScore: rrfContribution, topRank: rank,
        });
      }
    }
  }
  // Sort by rrfScore descending, secondary sort by topRank
}
```

**Weighted RRF:** First 2 result lists get 2x weight (original query + first expansion).

#### Chunk Selection (lines 3838-3862)

**Critical Optimization:** Reranking full bodies is O(tokens)—this was the motivating refactor.

```typescript
const queryTerms = query.toLowerCase().split(/\s+/).filter(t => t.length > 2);
const intentTerms = intent ? extractIntentTerms(intent) : [];

for (const cand of candidates) {
  const chunks = chunkDocument(cand.body);  // Smart chunking
  let bestIdx = 0;
  let bestScore = -1;

  for (let i = 0; i < chunks.length; i++) {
    const chunkLower = chunks[i]!.text.toLowerCase();
    let score = queryTerms.reduce((acc, term) =>
      acc + (chunkLower.includes(term) ? 1 : 0), 0);
    for (const term of intentTerms) {
      if (chunkLower.includes(term)) score += INTENT_WEIGHT_CHUNK;  // 0.5
    }
    if (score > bestScore) { bestScore = score; bestIdx = i; }
  }
  docChunkMap.set(cand.file, { chunks, bestIdx });
}
```

#### LLM Reranking (llm.ts lines 1106-1186)

```typescript
async rerank(query, documents, options = {}): Promise<RerankResult> {
  // Truncate docs to fit rerank context size
  const maxDocTokens = LlamaCpp.RERANK_CONTEXT_SIZE
    - LlamaCpp.RERANK_TEMPLATE_OVERHEAD
    - queryTokens;

  // Deduplicate identical texts before scoring
  const textToDocs = new Map<string, { file: string; index: number }[]>();

  // Parallel evaluation across multiple contexts
  const activeContextCount = Math.max(1, Math.min(
    contexts.length,
    Math.ceil(texts.length / LlamaCpp.RERANK_TARGET_DOCS_PER_CONTEXT)
  ));

  const allScores = await Promise.all(
    chunks.map((chunk, i) => activeContexts[i]!.rankAll(query, chunk))
  );
}
```

**Cache Key Innovation:** Cache key includes chunk text, not file path—different queries can select different chunks from the same file.

#### Position-Aware Blending (lines 3928-3942)

```typescript
const blended = reranked.map(r => {
  const rrfRank = rrfRankMap.get(r.file) || candidateLimit;
  let rrfWeight: number;
  if (rrfRank <= 3) rrfWeight = 0.75;
  else if (rrfRank <= 10) rrfWeight = 0.60;
  else rrfWeight = 0.40;
  const rrfScore = 1 / rrfRank;
  const blendedScore = rrfWeight * rrfScore + (1 - rrfWeight) * r.score;
  // ...
});
```

### Technical Debt / Shortcuts

1. **Sequential Vector Embedding:** Comment at line 4040: "concurrent embed() hangs"—node-llama-cpp limitation
2. **Fallback expansion is generic:** Uses `Information about ${query}` which may not be helpful
3. **No query normalization before BM25:** Could improve recall with query preprocessing

### Search Hooks (lines 3652-3667)

```typescript
export interface SearchHooks {
  onStrongSignal?: (topScore: number) => void;
  onExpandStart?: () => void;
  onExpand?: (original: string, expanded: ExpandedQuery[], elapsedMs: number) => void;
  onEmbedStart?: (count: number) => void;
  onEmbedDone?: (elapsedMs: number) => void;
  onRerankStart?: (chunkCount: number) => void;
  onRerankDone?: (elapsedMs: number) => void;
}
```

These allow CLI/MCP to show progress feedback during the search pipeline.

---

## 2. Collection Management

**Core File:** `src/collections.ts`
**Supporting File:** `src/store.ts` (lines 1078-1178, 885-907)

### Overview

Collections define directories to index with glob patterns. The system maintains configuration in both YAML files and SQLite, with write-through synchronization.

### Configuration Architecture

**Dual Storage:**
1. **YAML file** (`~/.config/qmd/index.yml`) — Human-editable, version-controllable
2. **SQLite** (`store_collections` table) — Runtime queryable, hash-tracked for sync

```typescript
// collections.ts line 59
let configSource: { type: 'file'; path?: string } | { type: 'inline'; config: CollectionConfig }
  = { type: 'file' };
```

### Collection Schema

```typescript
export interface Collection {
  path: string;              // Absolute path to index
  pattern: string;           // Glob pattern (e.g., "**/*.md")
  ignore?: string[];         // Glob patterns to exclude
  context?: ContextMap;      // Path-based context definitions
  update?: string;           // Optional bash command during qmd update
  includeByDefault?: boolean; // Include in queries (default: true)
}
```

### Indexing Pipeline (store.ts lines 1078-1178)

```typescript
export async function reindexCollection(
  store: Store,
  collectionPath: string,
  globPattern: string,
  collectionName: string,
  options?: { ignorePatterns?: string[]; onProgress?: (info: ReindexProgress) => void }
): Promise<ReindexResult> {
  const excludeDirs = ["node_modules", ".git", ".cache", "vendor", "dist", "build"];
  const allIgnore = [
    ...excludeDirs.map(d => `**/${d}/**`),
    ...(options?.ignorePatterns || []),
  ];

  // fastGlob for efficient recursive file discovery
  const allFiles: string[] = await fastGlob(globPattern, {
    cwd: collectionPath,
    onlyFiles: true,
    followSymbolicLinks: false,
    dot: false,
    ignore: allIgnore,
  });

  // Filter hidden files/folders
  const files = allFiles.filter(file => {
    const parts = file.split("/");
    return !parts.some(part => part.startsWith("."));
  });

  // Hash-based change detection
  for (const relativeFile of files) {
    const hash = await hashContent(content);
    const existing = findActiveDocument(db, collectionName, path);

    if (existing) {
      if (existing.hash === hash) {
        // Unchanged
      } else {
        // Content changed - update
        insertContent(db, hash, content, now);
        updateDocument(db, existing.id, title, hash, mtime);
      }
    } else {
      // New document
      insertContent(db, hash, content, now);
      insertDocument(db, collectionName, path, title, hash, birthtime, mtime);
    }
  }

  // Deactivate documents that no longer exist
  const allActive = getActiveDocumentPaths(db, collectionName);
  for (const path of allActive) {
    if (!seenPaths.has(path)) {
      deactivateDocument(db, collectionName, path);
    }
  }

  const orphanedCleaned = cleanupOrphanedContent(db);
}
```

### Key Optimizations

1. **Hash-based change detection:** Only updates content if hash changes
2. **Batched glob with ignore patterns:** Excludes common non-relevant directories
3. **Soft delete via `active` flag:** Documents marked inactive, not deleted immediately
4. **Orphan cleanup:** Removes content hashes with no active document references

### Configuration Sync (store.ts lines 921-970)

```typescript
export function syncConfigToDb(db: Database, config: CollectionConfig): void {
  // Check config hash — skip sync if unchanged
  const configJson = JSON.stringify(config);
  const hash = createHash('sha256').update(configJson).digest('hex');

  const existingHash = db.prepare(
    `SELECT value FROM store_config WHERE key = 'config_hash'`
  ).get();

  if (existingHash?.value === hash) {
    return; // Config unchanged, skip sync
  }
  // ... sync collections to DB
  // Update hash after sync
}
```

### Collection Operations

```typescript
// Add collection
export function addCollection(name: string, path: string, pattern: string = "**/*.md"): void {
  config.collections[name] = { path, pattern };
  saveConfig(config);
}

// Rename collection
export function renameCollection(oldName: string, newName: string): boolean {
  if (config.collections[newName]) {
    throw new Error(`Collection '${newName}' already exists`);
  }
  config.collections[newName] = config.collections[oldName];
  delete config.collections[oldName];
  saveConfig(config);
}
```

### File Path Handling

**Handelize function** converts filesystem paths to storage format:
- `/Users/john/docs/notes.md` → `Users/john/docs/notes.md`
- Preserves forward slashes, removes leading slash

---

## 3. Context System

**Core Files:**
- `src/collections.ts` (lines 324-470) — YAML configuration
- `src/store.ts` (lines 885-915, 2314-2394) — SQLite storage and retrieval

### Overview

The context system provides hierarchical, path-based metadata that can be attached to any path prefix within a collection. This metadata improves search relevance by providing domain-specific context to the reranker.

### Context Types

1. **Global Context** — Applies to all collections (`global_context` in config)
2. **Collection Context** — Path-prefixed contexts within a specific collection
3. **Virtual Path Context** — `qmd://collection/path` format for explicit targeting

### Storage Architecture

**YAML Storage (collections.ts):**
```yaml
global_context: "Always treat these documents as project documentation"
collections:
  journals:
    path: /Users/john/journals
    pattern: "**/*.md"
    context:
      /2024: "Journal entries from 2024"
      /2024/Q1: "First quarter reflections"
      /Board: "Board meeting notes and resolutions"
```

**SQLite Storage (store_collections table):**
```sql
-- context stored as JSON
UPDATE store_collections SET context = ? WHERE name = ?
-- Stored: '{"path": "context text", ...}'
```

### Context Retrieval (store.ts lines 2314-2394)

```typescript
export function getContextForFile(db: Database, filepath: string): string | null {
  // Parse virtual path format: qmd://collection/path
  const parsedVirtual = filepath.startsWith('qmd://') ? parseVirtualPath(filepath) : null;
  if (parsedVirtual) {
    collectionName = parsedVirtual.collectionName;
    relativePath = parsedVirtual.path;
  } else {
    // Filesystem path: find which collection owns this file
    for (const coll of collections) {
      if (filepath.startsWith(coll.path + '/') || filepath === coll.path) {
        collectionName = coll.name;
        relativePath = filepath.slice(coll.path.length + 1);
        break;
      }
    }
  }

  // Verify document exists
  const doc = db.prepare(`
    SELECT d.path FROM documents d
    WHERE d.collection = ? AND d.path = ? AND d.active = 1
  `).get(collectionName, relativePath);
  if (!doc) return null;

  // Collect ALL matching contexts
  const contexts: string[] = [];

  // Global context first (most general)
  const globalCtx = getStoreGlobalContext(db);
  if (globalCtx) contexts.push(globalCtx);

  // Collection contexts (most general to most specific)
  if (coll.context) {
    const normalizedPath = relativePath.startsWith("/") ? relativePath : `/${relativePath}`;

    const matchingContexts: { prefix: string; context: string }[] = [];
    for (const [prefix, context] of Object.entries(coll.context)) {
      const normalizedPrefix = prefix.startsWith("/") ? prefix : `/${prefix}`;
      if (normalizedPath.startsWith(normalizedPrefix)) {
        matchingContexts.push({ prefix: normalizedPrefix, context });
      }
    }

    // Sort by prefix length (shortest/most general first)
    matchingContexts.sort((a, b) => a.prefix.length - b.prefix.length);

    for (const match of matchingContexts) {
      contexts.push(match.context);
    }
  }

  return contexts.length > 0 ? contexts.join('\n\n') : null;
}
```

### Path Prefix Matching Algorithm

```typescript
// collections.ts lines 438-470
export function findContextForPath(collectionName: string, filePath: string): string | undefined {
  const collection = config.collections[collectionName];
  if (!collection?.context) {
    return config.global_context;  // Fallback to global
  }

  const matches: Array<{ prefix: string; context: string }> = [];

  for (const [prefix, context] of Object.entries(collection.context)) {
    const normalizedPath = filePath.startsWith("/") ? filePath : `/${filePath}`;
    const normalizedPrefix = prefix.startsWith("/") ? prefix : `/${prefix}`;

    if (normalizedPath.startsWith(normalizedPrefix)) {
      matches.push({ prefix: normalizedPrefix, context });
    }
  }

  // Return most specific match (longest prefix)
  if (matches.length > 0) {
    matches.sort((a, b) => b.prefix.length - a.prefix.length);
    return matches[0]!.context;
  }

  return config.global_context;
}
```

### Context Usage in Search

Contexts are retrieved per-result and exposed in the search result:

```typescript
// store.ts line 3900
return {
  file: cand.file,
  context: store.getContextForFile(cand.file),  // Attached to each result
  // ...
};
```

**The context is not currently used in the ranking pipeline**—it is exposed for the caller (CLI/MCP) to display or use for additional processing.

### Context Management Operations

```typescript
// Add context
export function addContext(collectionName: string, pathPrefix: string, contextText: string): boolean {
  const collection = config.collections[collectionName];
  if (!collection) return false;

  if (!collection.context) collection.context = {};
  collection.context[pathPrefix] = contextText;
  saveConfig(config);
  return true;
}

// Remove context
export function removeContext(collectionName: string, pathPrefix: string): boolean {
  const collection = config.collections[collectionName];
  if (!collection?.context?.[pathPrefix]) return false;

  delete collection.context[pathPrefix];
  if (Object.keys(collection.context).length === 0) {
    delete collection.context;
  }
  saveConfig(config);
  return true;
}
```

### Virtual Path Format

The system supports explicit virtual paths for context targeting:

```typescript
// qmd://collection/path format
qmd://journals/2024          → collection "journals", path "/2024"
qmd://journals/2024/Q1       → collection "journals", path "/2024/Q1"
```

### Observations

1. **Context aggregation is additive:** Multiple matching prefixes are combined, not replaced
2. **Path normalization:** Both paths and prefixes are normalized to `/prefix` format before comparison
3. **No context in ranking:** Currently contexts are metadata only, not factored into retrieval scores
4. **Soft delete consideration:** Context retrieval checks `d.active = 1`, so contexts for deleted files return null

---

## Cross-Feature Integration

### Search Flow with Context

```
User Query
    │
    ▼
hybridQuery() ─────────────────────────────────────────┐
    │                                                 │
    ├─► BM25/FTS search                               │
    ├─► Vector search with embeddings                 │
    ├─► RRF fusion                                    │
    ├─► Chunk selection                               │
    ├─► LLM reranking                                 │
    │                                                 │
    ▼                                                 │
getContextForFile() ◄───────────────────────────────────┘
    │  For each result file
    ▼
Return results with context attached
```

### Collection Indexing Flow

```
qmd update
    │
    ▼
syncConfigToDb() ──── Loads from YAML
    │
    ▼
reindexCollection() ──── Per collection
    │
    ├─► fastGlob() ──── File discovery
    ├─► hashContent() ──── Change detection
    ├─► insertDocument() ──── New/updated
    └─► deactivateDocument() ──── Removed
```

### Config Write-Through

```
store.addCollection()
    │
    ├─► upsertStoreCollection() ──── SQLite
    │
    └─► collectionsAddCollection() ──── YAML (if config file exists)
```

---

## Summary of Key Design Decisions

| Feature | Decision | Rationale |
|---------|----------|-----------|
| Search | Strong signal bypass | Avoid expensive LLM expansion when BM25 clearly matches |
| Search | Chunk-based reranking | Full body reranking is O(tokens) — unaffordable at scale |
| Search | Position-aware blending | Top results need RRF protection from reranker disagreement |
| Collections | Dual storage (YAML + SQLite) | Human-editable config + fast runtime queries |
| Collections | Hash-based change detection | Avoid reindexing unchanged documents |
| Context | Hierarchical path matching | Allows fine-grained context at any directory level |
| Context | Global context fallback | System-wide context when no collection match |
