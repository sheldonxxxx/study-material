# QMD Feature Deep-Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Synthesized:** 2026-03-27
**Source Documents:** Batch research 05a-05d, topology 01-topology

---

## Introduction

QMD (Query My Documents) is a local-first knowledge retrieval system combining BM25 full-text search, vector semantic search, and LLM-powered reranking. The project exposes a dual-mode interface: CLI commands for direct use, and an MCP server for AI agent integration. This document provides a comprehensive technical deep-dive into all features, ordered by priority.

The architecture centers on a single massive `store.ts` (~4,200 lines) handling all database operations, search, and indexing. This cohesion was likely driven by performance considerations and organic growth rather than strict separation of concerns.

---

## Part I: Core Features

### 1. Hybrid Search Pipeline

**Files:** `src/store.ts`, `src/llm.ts`, `src/index.ts`

QMD's flagship feature combines three retrieval signals through a sophisticated multi-stage pipeline:

```
Query Input
    │
    ▼
┌─────────────────────────────┐
│ Step 1: BM25 Probe          │
│ (Strong Signal Bypass)       │  If top BM25 score >= 0.85 AND gap >= 0.15
└─────────────────────────────┘
    │ (skip expansion if strong signal)
    ▼
┌─────────────────────────────┐
│ Step 2: Query Expansion     │  LLM generates lex/vec/hyde variants
│ (Qwen3 model, grammar-      │
│  constrained output)        │
└─────────────────────────────┘
    │
    ├──► FTS5 (BM25) for 'lex' queries
    └──► Batch Embed + sqlite-vec for 'vec'/'hyde' queries
              │
              ▼
┌─────────────────────────────┐
│ Step 3: RRF Fusion          │  k=60, first 2 lists get 2x weight
│ Reciprocal Rank Fusion      │  Top 40 candidates advance
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
│ (Qwen3-reranker model)     │  Cached results by chunk text
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

#### Strong Signal Bypass

The pipeline first runs a lightweight BM25 probe. If the top result has score >= 0.85 AND the gap to the second result >= 0.15, LLM expansion is skipped entirely. This optimization avoids expensive inference for obvious queries.

```typescript
// src/store.ts lines 3733-3746
const topScore = initialFts[0]?.score ?? 0;
const secondScore = initialFts[1]?.score ?? 0;
const hasStrongSignal = !intent && initialFts.length > 0
  && topScore >= STRONG_SIGNAL_MIN_SCORE  // 0.85
  && (topScore - secondScore) >= STRONG_SIGNAL_MIN_GAP;  // 0.15
```

#### Query Expansion

When expansion runs, a fine-tuned Qwen3-1.7B model generates three query types:

- **lex:** Keyword-focused variations for BM25
- **vec:** Semantic variations for vector search
- **hyde:** Hypothetical Document Embedding (generates ideal answer description)

Grammar-constrained output ensures valid structured format without post-hoc validation:

```typescript
// src/llm.ts lines 1013-1100
const grammar = await llama.createGrammar({
  grammar: `
    root ::= line+
    line ::= type ": " content "\\n"
    type ::= "lex" | "vec" | "hyde"
    content ::= [^\\n]+
  `
});
```

#### RRF Fusion

Reciprocal Rank Fusion combines ranked lists with configurable weights. The first two result lists (original query + first expansion) receive 2x weight:

```typescript
// src/store.ts lines 3056-3085
export function reciprocalRankFusion(
  resultLists: RankedResult[][],
  weights: number[] = [],
  k: number = 60
): RankedResult[] {
  for (let listIdx = 0; listIdx < resultLists.length; listIdx++) {
    const weight = weights[listIdx] ?? 1.0;
    for (let rank = 0; rank < list.length; rank++) {
      const rrfContribution = weight / (k + rank + 1);
      // Accumulate scores, track top rank for tiebreaking
    }
  }
}
```

#### Chunk-Based Reranking

Reranking full document bodies is O(tokens) - prohibitively expensive at scale. Instead, the pipeline:

1. Chunks each candidate document (900 tokens, 15% overlap)
2. Selects the chunk with highest keyword overlap to the query
3. Reranks only these selected chunks

```typescript
// src/store.ts lines 3838-3862
const queryTerms = query.toLowerCase().split(/\s+/).filter(t => t.length > 2);
for (const cand of candidates) {
  const chunks = chunkDocument(cand.body);
  let bestIdx = 0;
  let bestScore = -1;
  for (let i = 0; i < chunks.length; i++) {
    const chunkLower = chunks[i]!.text.toLowerCase();
    let score = queryTerms.reduce((acc, term) =>
      acc + (chunkLower.includes(term) ? 1 : 0), 0);
    if (score > bestScore) { bestScore = score; bestIdx = i; }
  }
  docChunkMap.set(cand.file, { chunks, bestIdx });
}
```

#### Position-Aware Blending

The final ranking blends RRF scores with reranker scores based on position:

| Rank | RRF Weight | Reranker Weight |
|------|-----------|-----------------|
| 1-3 | 75% | 25% |
| 4-10 | 60% | 40% |
| 11+ | 40% | 60% |

This protects top results from reranker disagreements while allowing the reranker to reorder lower results where confident.

#### Search Hooks

The pipeline exposes progress callbacks for CLI/MCP integration:

```typescript
// src/store.ts lines 3652-3667
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

#### Technical Debt

1. **Sequential vector embedding** - Comment at line 4040 notes "concurrent embed() hangs" due to node-llama-cpp limitations
2. **Generic fallback** - Uses `Information about ${query}` when expansion fails
3. **No query normalization** - BM25 could benefit from preprocessing (stemming, stopword removal)

---

### 2. Smart Document Chunking

**Files:** `src/store.ts`

Documents are split into ~900 token chunks with 15% overlap, preferring markdown-aware boundaries. This preserves semantic coherence better than fixed-size splits.

#### Break Point Scoring

Markdown elements receive different scores based on their semantic importance:

```typescript
// src/store.ts lines 97-110
export const BREAK_PATTERNS: [RegExp, number, string][] = [
  [/\n#{1}(?!#)/g, 100, 'h1'],     // H1 = most important
  [/\n#{2}(?!#)/g, 90, 'h2'],
  [/\n#{3}(?!#)/g, 80, 'h3'],
  [/\n#{4}(?!#)/g, 70, 'h4'],
  [/\n#{5}(?!#)/g, 60, 'h5'],
  [/\n#{6}(?!#)/g, 50, 'h6'],
  [/\n```/g, 80, 'codeblock'],       // Code blocks protected
  [/\n(?:---|\*\*\*|___)\s*\n/g, 60, 'hr'],
  [/\n\n+/g, 20, 'blank'],          // Paragraph breaks
  [/\n[-*]\s/g, 5, 'list'],
  [/\n\d+\.\s/g, 5, 'numlist'],
  [/\n/g, 1, 'newline'],            // Last resort
];
```

#### Code Fence Protection

Code blocks are never split. A pre-scan identifies fenced regions:

```typescript
// src/store.ts lines 144-166
export function findCodeFences(text: string): CodeFenceRegion[] {
  const regions: CodeFenceRegion[] = [];
  const fencePattern = /\n```/g;
  let inFence = false;
  let fenceStart = 0;

  for (const match of text.matchAll(fencePattern)) {
    if (!inFence) {
      fenceStart = match.index!;
      inFence = true;
    } else {
      regions.push({ start: fenceStart, end: match.index! + match[0].length });
      inFence = false;
    }
  }
  // Handle unclosed fence at end
  if (inFence) {
    regions.push({ start: fenceStart, end: text.length });
  }
  return regions;
}
```

#### Best Cut Selection

When approaching the chunk boundary, the algorithm searches a 200-token window for the highest-scoring break point. A squared distance decay favors breaks closer to the target:

```typescript
// src/store.ts lines 188-224
export function findBestCutoff(
  breakPoints: BreakPoint[],
  targetCharPos: number,
  windowChars: number = CHUNK_WINDOW_CHARS,
  decayFactor: number = 0.7,
  codeFences: CodeFenceRegion[] = []
): number {
  const windowStart = targetCharPos - windowChars;
  let bestScore = -1;
  let bestPos = targetCharPos;

  for (const bp of breakPoints) {
    if (bp.pos < windowStart) continue;
    if (bp.pos > targetCharPos) break;

    if (isInsideCodeFence(bp.pos, codeFences)) continue;

    const distance = targetCharPos - bp.pos;
    const normalizedDist = distance / windowChars;
    const multiplier = 1.0 - (normalizedDist * normalizedDist) * decayFactor;
    const finalScore = bp.score * multiplier;

    if (finalScore > bestScore) {
      bestScore = finalScore;
      bestPos = bp.pos;
    }
  }
  return bestPos;
}
```

**Distance decay behavior:**
- At target position: multiplier = 1.0 (100%)
- At 25% back in window: multiplier = 0.94
- At 50% back: multiplier = 0.83
- At 75% back: multiplier = 0.61

This allows important headings far back in the window to win over low-quality breaks near the target.

#### Constants

```typescript
// src/store.ts lines 50-59
export const CHUNK_SIZE_TOKENS = 900;
export const CHUNK_OVERLAP_TOKENS = Math.floor(CHUNK_SIZE_TOKENS * 0.15);  // 135

// Fallback char-based approximation (~4 chars per token)
export const CHUNK_SIZE_CHARS = CHUNK_SIZE_TOKENS * 4;  // 3600
export const CHUNK_OVERLAP_CHARS = CHUNK_OVERLAP_TOKENS * 4;  // 540

export const CHUNK_WINDOW_TOKENS = 200;
export const CHUNK_WINDOW_CHARS = CHUNK_WINDOW_TOKENS * 4;  // 800
```

#### Snippet Extraction

Given a query, `extractSnippet` finds the most relevant passage:

```typescript
// src/store.ts lines 3558-3624
export function extractSnippet(body: string, query: string, maxLen = 500,
  chunkPos?: number, chunkLen?: number, intent?: string): SnippetResult {
  // Search within chunk region if provided, with padding
  if (chunkPos && chunkPos > 0) {
    const contextStart = Math.max(0, chunkPos - 100);
    const contextEnd = Math.min(body.length, chunkPos + (chunkLen || CHUNK_SIZE_CHARS) + 100);
    searchBody = body.slice(contextStart, contextEnd);
  }

  // Score each line for query terms + intent terms
  const queryTerms = query.toLowerCase().split(/\s+/);
  const intentTerms = intent ? extractIntentTerms(intent) : [];
  let bestLine = 0, bestScore = -1;

  for (const line of lines) {
    let score = 0;
    for (const term of queryTerms) {
      if (lineLower.includes(term)) score += 1.0;
    }
    for (const term of intentTerms) {
      if (lineLower.includes(term)) score += INTENT_WEIGHT_SNIPPET;
    }
    if (score > bestScore) { bestScore = score; bestLine = i; }
  }

  // Return 1 line before + 3 lines after best match
  const start = Math.max(0, bestLine - 1);
  const end = Math.min(lines.length, bestLine + 3);
  const header = `@@ -${absoluteStart},${snippetLineCount} @@ ...`;
}
```

---

### 3. Collection Management

**Files:** `src/collections.ts`, `src/store.ts`

Collections define directories to index with glob patterns. The system maintains configuration in both YAML files and SQLite with write-through synchronization.

#### Dual Storage Architecture

| Storage | Purpose | Format |
|---------|---------|--------|
| YAML file | Human-editable, version-controllable | `~/.config/qmd/index.yml` |
| SQLite | Fast runtime queries, hash-tracked | `store_collections` table |

#### Collection Schema

```typescript
// src/collections.ts
export interface Collection {
  path: string;              // Absolute path to index
  pattern: string;           // Glob pattern (e.g., "**/*.md")
  ignore?: string[];         // Glob patterns to exclude
  context?: ContextMap;      // Path-based context definitions
  update?: string;           // Optional bash command during qmd update
  includeByDefault?: boolean; // Include in queries (default: true)
}
```

#### Indexing Pipeline

```typescript
// src/store.ts lines 1078-1178
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

  // Hash-based change detection
  for (const relativeFile of files) {
    const hash = await hashContent(content);
    const existing = findActiveDocument(db, collectionName, path);

    if (existing) {
      if (existing.hash === hash) {
        // Unchanged - skip
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

#### Key Optimizations

1. **Hash-based change detection** - Only updates content if hash changes
2. **Batched glob with ignore patterns** - Excludes common non-relevant directories
3. **Soft delete via `active` flag** - Documents marked inactive, not deleted immediately
4. **Orphan cleanup** - Removes content hashes with no active document references

#### Configuration Sync

```typescript
// src/store.ts lines 921-970
export function syncConfigToDb(db: Database, config: CollectionConfig): void {
  const configJson = JSON.stringify(config);
  const hash = createHash('sha256').update(configJson).digest('hex');

  const existingHash = db.prepare(
    `SELECT value FROM store_config WHERE key = 'config_hash'`
  ).get();

  if (existingHash?.value === hash) {
    return; // Config unchanged, skip sync
  }
  // ... sync collections to DB
}
```

---

### 4. Context System

**Files:** `src/collections.ts`, `src/store.ts`, `src/db.ts`

Context provides hierarchical, path-based metadata attached to collection paths. This metadata improves search relevance by providing domain-specific context to the reranker.

#### Context Types

1. **Global Context** - Applies to all collections (`global_context` in config)
2. **Collection Context** - Path-prefixed contexts within a specific collection
3. **Virtual Path Context** - `qmd://collection/path` format for explicit targeting

#### YAML Storage

```yaml
# ~/.config/qmd/index.yml
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

#### Path Prefix Matching Algorithm

```typescript
// src/collections.ts lines 438-470
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

#### Context Retrieval

```typescript
// src/store.ts lines 2314-2394
export function getContextForFile(db: Database, filepath: string): string | null {
  // Parse virtual path format: qmd://collection/path
  const parsedVirtual = filepath.startsWith('qmd://') ? parseVirtualPath(filepath) : null;

  // Collect ALL matching contexts (most general first)
  const contexts: string[] = [];

  // Global context first (most general)
  const globalCtx = getStoreGlobalContext(db);
  if (globalCtx) contexts.push(globalCtx);

  // Collection contexts (most general to most specific)
  if (coll.context) {
    // ... collect all matching, sort by prefix length
  }

  return contexts.length > 0 ? contexts.join('\n\n') : null;
}
```

**Important observation:** Contexts are aggregated additively (not replaced), and contexts are NOT currently used in the ranking pipeline - they are metadata only, exposed for the caller to display or use.

---

### 5. MCP Server Integration

**File:** `src/mcp/server.ts`

The MCP server exposes QMD as a Model Context Protocol server, enabling AI agents to search and retrieve documents through MCP tools and resources.

#### Transport Architecture

The server supports two transport mechanisms:

1. **Stdio Transport** - Default, for local CLI integration
2. **Streamable HTTP** - For networked access with session-per-client model

```typescript
// src/mcp/server.ts lines 522-527 (Stdio)
export async function startMcpServer(): Promise<void> {
  const store = await createStore({ dbPath: getDefaultDbPath() });
  const server = await createMcpServer(store);
  const transport = new StdioServerTransport();
  await server.connect(transport);
}
```

#### Tool Definitions

**`query` tool** - Primary search tool supporting multiple query types:

```typescript
// src/mcp/server.ts lines 224-341
server.registerTool("query", {
  title: "Query",
  inputSchema: {
    searches: z.array(subSearchSchema).min(1).max(10),
    limit: z.number().optional().default(10),
    minScore: z.number().optional().default(0),
    candidateLimit: z.number().optional(),
    collections: z.array(z.string()).optional(),
    intent: z.string().optional(),
  },
  async ({ searches, limit, minScore, candidateLimit, collections, intent }) {
    // ... executes search via store.search()
  }
});
```

**`get` tool** - Retrieve single document:
- Supports `:line` suffix parsing (e.g., `"foo.md:120"`)
- Falls back to similar file suggestions on 404
- Returns document as MCP resource with `qmd://` URI

**`multi_get` tool** - Batch retrieval:
- Supports glob patterns and comma-separated lists
- Skips large files with reason explanation

**`status` tool** - Index health check returning structured data

#### Resource URI Scheme

Documents accessible via `qmd://` URIs:

```typescript
// src/mcp/server.ts lines 127-143
function encodeQmdPath(path: string): string {
  return path.split('/').map(segment => encodeURIComponent(segment)).join('/');
}

server.registerResource(
  "document",
  new ResourceTemplate("qmd://{+path}", { list: undefined }),
  {
    title: "QMD Document",
    mimeType: "text/markdown",
  },
  async (uri, { path }) => {
    const decodedPath = decodeURIComponent(path);
    const result = await store.get(decodedPath, { includeBody: true });
    // ...
  }
);
```

#### Clever Design Decisions

1. **REST endpoint bypass** - `POST /query` allows structured search without MCP protocol overhead
2. **First lex/vec query extraction** - Used for snippet extraction priority
3. **Write-through config** - When YAML config exists, mutations write to both SQLite and YAML

---

### 6. SDK/Library API

**File:** `src/index.ts`

The SDK provides programmatic Node.js/Bun access to QMD's search, retrieval, and indexing capabilities.

#### Core Exports

```typescript
// src/index.ts lines 112-118
export { extractSnippet, addLineNumbers, DEFAULT_MULTI_GET_MAX_BYTES };
export { getDefaultDbPath } from "./store.js";
export { Maintenance } from "./maintenance.js";
```

#### `createStore` Implementation

```typescript
// src/index.ts lines 333-528
export async function createStore(options: StoreOptions): Promise<QMDStore> {
  // Options validation
  if (!options.dbPath) throw new Error("dbPath is required");
  if (options.configPath && options.config) {
    throw new Error("Provide either configPath or config, not both");
  }

  // Create internal store (opens DB, creates tables)
  const internal = createStoreInternal(options.dbPath);

  // Config mode detection
  const hasYamlConfig = !!options.configPath;

  // Sync config to SQLite
  if (options.configPath) {
    setConfigSource({ configPath: options.configPath });
    const config = loadConfig();
    syncConfigToDb(db, config);
  } else if (options.config) {
    setConfigSource({ config: options.config });
    syncConfigToDb(db, options.config);
  }
  // else: DB-only mode

  // Per-store LlamaCpp instance - lazy loads, auto-unloads after 5 min
  const llm = new LlamaCpp({
    inactivityTimeoutMs: 5 * 60 * 1000,
    disposeModelsOnInactivity: true,
  });
  internal.llm = llm;

  return store;
}
```

#### QMDStore Interface

The interface provides:

**Search:**
- `search(options: SearchOptions)` - Full hybrid search
- `searchLex(query, options?)` - BM25 only
- `searchVector(query, options?)` - Vector only
- `expandQuery(query, options?)` - Manual expansion

**Document Retrieval:**
- `get(pathOrDocid, options?)` - Single document
- `getDocumentBody(pathOrDocid, opts?)` - Body with line slicing
- `multiGet(pattern, options?)` - Batch by glob/comma-separated

**Collection Management:**
- `addCollection(name, opts)`, `removeCollection(name)`, `renameCollection(oldName, newName)`
- `listCollections()`, `getDefaultCollectionNames()`

**Context Management:**
- `addContext(collectionName, pathPrefix, contextText)`
- `removeContext(collectionName, pathPrefix)`
- `setGlobalContext(context)`, `getGlobalContext()`, `listContexts()`

**Indexing:**
- `update(options?)` - Re-index from filesystem
- `embed(options?)` - Generate vector embeddings

**Health:**
- `getStatus()`, `getIndexHealth()`

**Lifecycle:**
- `close()` - Release all resources

---

### 7. Query Expansion Model

**Files:** `src/llm.ts`, `finetune/train.py`, `finetune/reward.py`

A fine-tuned Qwen3-1.7B model transforms raw queries into structured multi-format search variations.

#### Model Configuration

```typescript
// src/llm.ts:199
const DEFAULT_GENERATE_MODEL = "hf:tobil/qmd-query-expansion-1.7B-gguf/qmd-query-expansion-1.7B-q4_k_m.gguf";
```

The model is trained on ~2,290 examples using SFT-only (GRPO moved to experiments).

#### Training Pipeline

**Data format:**
```json
{"query": "auth config", "output": [["hyde", "..."], ["lex", "..."], ["vec", "..."]]}
```

**Scoring rubric** (`finetune/SCORING.md`):
- Structure: +10 for each of lex/vec present, -10 if missing
- Diversity bonus: +5 for multiple diverse lex/vec lines
- Format penalty: -5 per invalid line

#### Inference Pipeline

```typescript
// src/llm.ts lines 1013-1100
async expandQuery(query: string, options: { context?: string, includeLexical?: boolean, intent?: string } = {}): Promise<Queryable[]> {
  const llama = await this.ensureLlama();
  await this.ensureGenerateModel();

  // Create grammar to enforce structured output format
  const grammar = await llama.createGrammar({
    grammar: `root ::= line+\nline ::= type ": " content "\\n"`
  });

  const prompt = intent
    ? `/no_think Expand this search query: ${query}\nQuery intent: ${intent}`
    : `/no_think Expand this search query: ${query}`;

  // Create bounded context (2048 tokens) to prevent large VRAM allocations
  const genContext = await this.generateModel!.createContext({
    contextSize: this.expandContextSize,
  });

  const result = await session.prompt(prompt, {
    grammar,
    maxTokens: 600,
    temperature: 0.7,
    topK: 20,
    topP: 0.8,
    repeatPenalty: { lastTokens: 64, presencePenalty: 0.5 },
  });
}
```

#### Response Parsing

```typescript
// src/llm.ts lines 1061-1079
const lines = result.trim().split("\n");
const queryTerms = query.toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter(Boolean);

const queryables: Queryable[] = lines.map(line => {
  const colonIdx = line.indexOf(":");
  if (colonIdx === -1) return null;
  const type = line.slice(0, colonIdx).trim();
  if (type !== 'lex' && type !== 'vec' && type !== 'hyde') return null;
  const text = line.slice(colonIdx + 1).trim();
  // Filter out expansions that don't contain any query terms
  if (!queryTerms.some(term => text.toLowerCase().includes(term))) return null;
  return { type: type as QueryType, text };
}).filter((q): q is Queryable => q !== null);
```

#### Fallback Handling

On parse failure or empty results:

```typescript
const fallback: Queryable[] = [
  { type: 'hyde', text: `Information about ${query}` },
  { type: 'lex', text: query },
  { type: 'vec', text: query },
];
```

#### Clever Solutions

1. **Grammar-guided decoding** - Output format enforced at inference time
2. **Query term filtering** - Expansions must contain at least one term from original query
3. **IncludeLexical flag** - Allows callers to disable `lex:` expansions
4. **Intent injection** - Optional query intent guides expansion style

---

## Part II: Secondary Features

### 8. Document Retrieval

**File:** `src/store.ts`

Document retrieval supports fetching by virtual path, absolute path, short docid, or glob patterns with fuzzy matching.

#### Docid System

Documents use a two-tier identifier system:

```typescript
// src/store.ts lines 1559-1561
export function getDocid(hash: string): string {
  return hash.slice(0, 6);  // First 6 chars of SHA-256
}

// src/store.ts lines 2203-2217
export function findDocumentByDocid(db: Database, docid: string): { filepath: string; hash: string } | null {
  const shortHash = normalizeDocid(docid);
  const doc = db.prepare(`
    SELECT 'qmd://' || d.collection || '/' || d.path as filepath, d.hash
    FROM documents d
    WHERE d.hash LIKE ? AND d.active = 1
    LIMIT 1
  `).get(`${shortHash}%`) as { filepath: string; hash: string } | null;
  return doc;
}
```

#### Lookup Chain

```typescript
// src/store.ts lines 3193-3296
export function findDocument(db: Database, filename: string, options: { includeBody?: boolean } = {}): DocumentResult | DocumentNotFound {
  let filepath = filename;

  // 1. Check for :line suffix
  const colonMatch = filepath.match(/:(\d+)$/);
  if (colonMatch) {
    filepath = filepath.slice(0, -colonMatch[0].length);
  }

  // 2. Try docid lookup if looks like hash
  if (isDocid(filepath)) {
    const docidMatch = findDocumentByDocid(db, filepath);
    if (docidMatch) {
      filepath = docidMatch.filepath;
    } else {
      return { error: "not_found", query: filename, similarFiles: [] };
    }
  }

  // 3. Try exact virtual path match
  let doc = db.prepare(`... WHERE 'qmd://' || d.collection || '/' || d.path = ? ...`).get(filepath);

  // 4. Try fuzzy suffix match (LIKE %suffix)
  if (!doc) {
    doc = db.prepare(`... WHERE 'qmd://' || d.collection || '/' || d.path LIKE ? ...`).get(`%${filepath}`);
  }

  // 5. Try absolute path via collection resolution
  if (!doc && !filepath.startsWith('qmd://')) {
    // ... iterate collections, find owning collection, match by relative path
  }

  // 6. Return similar files if not found
  if (!doc) {
    const similar = findSimilarFiles(db, filepath, 5, 5);
    return { error: "not_found", query: filename, similarFiles: similar };
  }
}
```

#### Multi-Document Retrieval

```typescript
// src/store.ts lines 3352-3456
export function findDocuments(
  db: Database,
  pattern: string,
  options: { includeBody?: boolean; maxBytes?: number } = {}
): { docs: MultiGetResult[]; errors: string[] } {
  const isCommaSeparated = pattern.includes(',') && !pattern.includes('*') && !pattern.includes('?');

  if (isCommaSeparated) {
    // Handle comma-separated list
    const names = pattern.split(',').map(s => s.trim()).filter(Boolean);
    for (const name of names) {
      let doc = db.prepare(`...`).get(name);
      if (!doc) doc = db.prepare(`...`).get(`%${name}`);  // fuzzy
      if (!doc) errors.push(`File not found: ${name} (did you mean: ${similar.join(', ')}?)`);
    }
  } else {
    // Handle glob pattern
    const matched = matchFilesByGlob(db, pattern);
    if (matched.length === 0) {
      errors.push(`No files matched pattern: ${pattern}`);
    }
  }
}
```

#### Fuzzy Matching

```typescript
// src/store.ts lines 2219-2232
export function findSimilarFiles(db: Database, query: string, maxDistance: number = 3, limit: number = 5): string[] {
  const allFiles = db.prepare(`SELECT d.path FROM documents d WHERE d.active = 1`).all();
  const queryLower = query.toLowerCase();

  const scored = allFiles
    .map(f => ({ path: f.path, dist: levenshtein(f.path.toLowerCase(), queryLower) }))
    .filter(f => f.dist <= maxDistance)
    .sort((a, b) => a.dist - b.dist)
    .slice(0, limit);

  return scored.map(f => f.path);
}
```

#### Technical Debt

1. **Sequential comma-separated lookups** - Each file looked up with individual DB calls
2. **All files loaded for glob** - `matchFilesByGlob` loads ALL active paths into memory before filtering
3. **No docid collision reporting** - Multiple documents sharing short hash return first silently
4. **Levenshtein on full paths** - Computes edit distance on full paths, not just filename components

---

### 9. HTTP Transport for MCP

**File:** `src/mcp/server.ts`

HTTP transport enables long-lived daemon operation with persistent model loading.

#### Daemon Mode Implementation

```typescript
// src/cli/qmd.ts lines 3023-3078
const cacheDir = process.env.XDG_CACHE_HOME
  ? resolve(process.env.XDG_CACHE_HOME, "qmd")
  : resolve(homedir(), ".cache", "qmd");
const pidPath = resolve(cacheDir, "mcp.pid");

// Stale PID detection
if (existsSync(pidPath)) {
  const existingPid = parseInt(readFileSync(pidPath, "utf-8").trim());
  try {
    process.kill(existingPid, 0); // alive?
    console.error(`Already running (PID ${existingPid}).`);
    process.exit(1);
  } catch {
    // Stale PID file — continue
  }
}

// Background spawn
const child = nodeSpawn(process.execPath, spawnArgs, {
  stdio: ["ignore", logFd, logFd],
  detached: true,
});
child.unref();
writeFileSync(pidPath, String(child.pid));
```

#### Session Management

```typescript
// src/mcp/server.ts lines 549-572
const sessions = new Map<string, WebStandardStreamableHTTPServerTransport>();

async function createSession(): Promise<WebStandardStreamableHTTPServerTransport> {
  const transport = new WebStandardStreamableHTTPServerTransport({
    sessionIdGenerator: () => randomUUID(),
    enableJsonResponse: true,
    onsessioninitialized: (sessionId: string) => {
      sessions.set(sessionId, transport);
    },
  });
}

transport.onclose = () => {
  if (transport.sessionId) {
    sessions.delete(transport.sessionId);
  }
};
```

MCP spec requires one server instance per client session. The store is shared - SQLite is safe for concurrent access.

#### HTTP Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Health check with uptime |
| `/query` or `/search` | POST | Direct REST search (no MCP protocol) |
| `/mcp` | GET/POST/DELETE | MCP protocol handler |

#### REST Search Endpoint

```typescript
// src/mcp/server.ts lines 626-675
if ((pathname === "/query" || pathname === "/search") && nodeReq.method === "POST") {
  const rawBody = await collectBody(nodeReq);
  const params = JSON.parse(rawBody);

  if (!params.searches || !Array.isArray(params.searches)) {
    nodeRes.writeHead(400, { "Content-Type": "application/json" });
    nodeRes.end(JSON.stringify({ error: "Missing required field: searches (array)" }));
    return;
  }

  const results = await store.search({ queries, ... });
  nodeRes.writeHead(200, { "Content-Type": "application/json" });
  nodeRes.end(JSON.stringify({ results: formatted }));
}
```

#### Graceful Shutdown

```typescript
// src/mcp/server.ts lines 132-153
let stopping = false;
const stop = async () => {
  if (stopping) return;
  stopping = true;
  for (const transport of sessions.values()) {
    await transport.close();
  }
  sessions.clear();
  httpServer.close();
  await store.close();
};

process.on("SIGTERM", async () => { await stop(); process.exit(0); });
process.on("SIGINT", async () => { await stop(); process.exit(0); });
```

---

### 10. Multi-Format Output

**File:** `src/cli/formatter.ts`

The formatter system provides 6 output formats for search results and document retrieval.

#### Format Types

```typescript
// src/cli/formatter.ts:36
export type OutputFormat = "cli" | "csv" | "md" | "xml" | "files" | "json";
```

#### JSON Format

```typescript
// src/cli/formatter.ts lines 97-123
export function searchResultsToJson(results: SearchResult[], opts: FormatOptions = {}): string {
  const output = results.map(row => {
    const bodyStr = row.body || "";
    let body = opts.full ? bodyStr : undefined;
    let snippet = !opts.full ? extractSnippet(bodyStr, query, 300, row.chunkPos, undefined, opts.intent).snippet : undefined;

    return {
      docid: `#${row.docid}`,
      score: Math.round(row.score * 100) / 100,
      file: row.displayPath,
      title: row.title,
      ...(row.context && { context: row.context }),
      ...(body && { body }),
      ...(snippet && { snippet }),
    };
  });
  return JSON.stringify(output, null, 2);
}
```

#### Universal Format Functions

```typescript
// src/cli/formatter.ts lines 381-404
export function formatSearchResults(results: SearchResult[], format: OutputFormat, opts: FormatOptions = {}): string {
  switch (format) {
    case "json": return searchResultsToJson(results, opts);
    case "csv": return searchResultsToCsv(results, opts);
    case "files": return searchResultsToFiles(results);
    case "md": return searchResultsToMarkdown(results, opts);
    case "xml": return searchResultsToXml(results, opts);
    case "cli": return searchResultsToMarkdown(results, opts);  // Fallback to md
    default: return searchResultsToJson(results, opts);
  }
}

export function formatDocuments(results: MultiGetFile[], format: OutputFormat): string {
  switch (format) {
    case "json": return documentsToJson(results);
    case "csv": return documentsToCsv(results);
    case "files": return documentsToFiles(results);
    case "md": return documentsToMarkdown(results);
    case "xml": return documentsToXml(results);
    case "cli": return documentsToMarkdown(results);  // Fallback to md
    default: return documentsToJson(results);
  }
}
```

#### Format Options

```typescript
// src/cli/formatter.ts lines 38-44
export type FormatOptions = {
  full?: boolean;       // Show full document content instead of snippet
  query?: string;       // Query for snippet extraction and highlighting
  useColor?: boolean;   // Enable terminal colors
  lineNumbers?: boolean;// Add line numbers to output
  intent?: string;      // Domain intent for snippet extraction disambiguation
};
```

#### Escape Helpers

```typescript
// src/cli/formatter.ts lines 72-88
export function escapeCSV(value: string | null | number): string {
  if (str.includes(",") || str.includes('"') || str.includes("\n")) {
    return `"${str.replace(/"/g, '""')}"`;
  }
  return str;
}

export function escapeXml(str: string): string {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&apos;");
}
```

#### Technical Debt

1. **CLI format is markdown fallback** - Not truly CLI-optimized colored output
2. **Inconsistent score precision** - JSON rounds to 2 decimals, CSV uses 4
3. **No streaming support** - All formats build complete string in memory

---

### 11. Custom Embedding Models

**File:** `src/llm.ts`

QMD supports custom embedding models via environment variables.

#### Configuration

```typescript
// src/llm.ts
const DEFAULT_EMBED_MODEL = process.env.QMD_EMBED_MODEL
  ?? "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf";

const DEFAULT_RERANK_MODEL = "hf:ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF/qwen3-reranker-0.6b-q8_0.gguf";
```

Alternative models supported:

```typescript
// LiquidAI LFM2 models
export const LFM2_GENERATE_MODEL = "hf:LiquidAI/LFM2-1.2B-GGUF/LFM2-1.2B-Q4_K_M.gguf";
export const LFM2_INSTRUCT_MODEL = "hf:LiquidAI/LFM2.5-1.2B-Instruct-GGUF/LFM2.5-1.2B-Instruct-Q4_K_M.gguf";
```

#### Model Format Detection

Different models use different prompting formats:

```typescript
// src/llm.ts
export function isQwen3EmbeddingModel(modelUri: string): boolean {
  return /qwen.*embed/i.test(modelUri) || /embed.*qwen/i.test(modelUri);
}

export function formatQueryForEmbedding(query: string, modelUri?: string): string {
  const uri = modelUri ?? process.env.QMD_EMBED_MODEL ?? DEFAULT_EMBED_MODEL;
  if (isQwen3EmbeddingModel(uri)) {
    return `Instruct: Retrieve relevant documents for the given query\nQuery: ${query}`;
  }
  return `task: search result | query: ${query}`;
}
```

#### Model Caching

Models cached in `~/.cache/qmd/models/` with ETag-based refresh:

```typescript
// src/llm.ts
async function getRemoteEtag(ref: HfRef): Promise<string | null> {
  const url = `https://huggingface.co/${ref.repo}/resolve/main/${ref.file}`;
  const resp = await fetch(url, { method: "HEAD" });
  return resp.headers.get("etag");
}
```

#### Parallelism Computation

```typescript
// src/llm.ts lines 315-329
private async computeParallelism(perContextMB: number): Promise<number> {
  const llama = await this.ensureLlama();

  if (llama.gpu) {
    const vram = await llama.getVramState();
    const freeMB = vram.free / (1024 * 1024);
    const maxByVram = Math.floor((freeMB * 0.25) / perContextMB);
    return Math.max(1, Math.min(8, maxByVram));
  }

  // CPU: split cores across contexts. At least 4 threads per context.
  const cores = llama.cpuMathCores || 4;
  const maxContexts = Math.floor(cores / 4);
  return Math.max(1, Math.min(4, maxContexts));
}
```

For embeddings (~150MB per context), allows up to 8 parallel contexts on GPU or up to 4 on CPU.

---

### 12. Index Maintenance

**Files:** `src/maintenance.ts`, `src/store.ts`

QMD provides comprehensive maintenance operations for database cleanup.

#### Maintenance Class

```typescript
// src/maintenance.ts
export class Maintenance {
  private store: Store;

  constructor(store: Store) { this.store = store; }

  vacuum(): void { db.exec(`VACUUM`); }
  cleanupOrphanedContent(): number { ... }
  cleanupOrphanedVectors(): number { ... }
  clearLLMCache(): number { ... }
  deleteInactiveDocs(): number { ... }
  clearEmbeddings(): void { ... }
}
```

#### Operations Detail

**Vacuum Database** - Reclaims unused space by rebuilding SQLite:

```typescript
vacuumDatabase(db: Database): void {
  db.exec(`VACUUM`);
}
```

**Cleanup Orphaned Content** - Removes content hashes no longer referenced:

```typescript
export function cleanupOrphanedContent(db: Database): number {
  const result = db.prepare(`
    DELETE FROM content
    WHERE hash NOT IN (SELECT DISTINCT hash FROM documents WHERE active = 1)
  `).run();
  return result.changes;
}
```

**Cleanup Orphaned Vectors** - Removes vector embeddings for deleted content:

```typescript
export function cleanupOrphanedVectors(db: Database): number {
  if (!isSqliteVecAvailable()) {
    return 0;  // Safety check for Bun's bun:sqlite
  }
  // Delete orphaned vectors...
}
```

**Clear LLM Cache** - Deletes cached query expansion and reranking results:

```typescript
export function deleteLLMCache(db: Database): number {
  const result = db.prepare(`DELETE FROM llm_cache`).run();
  return result.changes;
}
```

**Delete Inactive Documents** - Removes documents marked as inactive:

```typescript
export function deleteInactiveDocuments(db: Database): number {
  const result = db.prepare(`DELETE FROM documents WHERE active = 0`).run();
  return result.changes;
}
```

**Clear All Embeddings** - Removes all vector embeddings:

```typescript
export function clearAllEmbeddings(db: Database): void {
  db.exec(`DELETE FROM content_vectors`);
  db.exec(`DROP TABLE IF EXISTS vectors_vec`);
}
```

#### Index Health Monitoring

```typescript
// src/store.ts lines 453-472
export type IndexHealthInfo = {
  needsEmbedding: number;
  totalDocs: number;
  daysStale: number | null;
};

export function getIndexHealth(db: Database): IndexHealthInfo {
  const needsEmbedding = getHashesNeedingEmbedding(db);
  const totalDocs = db.prepare(`SELECT COUNT(*) as count FROM documents WHERE active = 1`).get().count;

  const mostRecent = db.prepare(`SELECT MAX(modified_at) as latest FROM documents WHERE active = 1`).get();
  let daysStale: number | null = null;

  if (mostRecent.latest) {
    const ageMs = Date.now() - new Date(mostRecent.latest).getTime();
    daysStale = Math.floor(ageMs / (1000 * 60 * 60 * 24));
  }

  return { needsEmbedding, totalDocs, daysStale };
}
```

---

## Part III: Cross-Cutting Concerns

### File Organization

The project exhibits a distinctive pattern: a single massive `store.ts` (~4,200 lines) handling all database operations, search, and indexing. This contrasts with typical enterprise patterns of per-class file separation.

**File sizes (from 01-topology.md):**

| File | Lines | Purpose |
|------|-------|---------|
| `src/store.ts` | ~4,200 | Core: DB ops, search, indexing, chunking |
| `src/cli/qmd.ts` | ~3,200 | CLI commands and argument parsing |
| `src/llm.ts` | ~1,400 | LlamaCpp embeddings, reranking, query expansion |
| `src/index.ts` | ~530 | SDK public API and QMDStore interface |
| `src/mcp/server.ts` | ~850 | MCP server tools and resources |

### Technology Stack

| Layer | Technology |
|-------|------------|
| Runtime | Node.js 22+ or Bun |
| Language | TypeScript |
| Package Manager | Bun (primary), npm compatible |
| Test Runner | Vitest |
| Database | SQLite + sqlite-vec (vector extension) + better-sqlite3 |
| Full-Text Search | SQLite FTS5 (BM25) |
| Vector Search | sqlite-vec |
| LLM/Embeddings | node-llama-cpp (GGUF models) |
| MCP | @modelcontextprotocol/sdk |
| Config | YAML (collections.ts) |
| Glob | fast-glob, picomatch |

### Dual-Mode Package

QMD operates as both CLI tool and SDK library:

```json
{
  "bin": { "qmd": "bin/qmd" },
  "exports": { ".": { "import": "./dist/index.js" } }
}
```

---

## Notable Design Patterns

### 1. Strong Signal Bypass

The search pipeline first probes BM25 and skips LLM expansion for obvious matches. This is a pragmatic optimization that avoids expensive inference when simple keyword matching suffices.

### 2. Chunk-Based Reranking

Instead of reranking full documents (O(tokens)), the pipeline selects the most relevant chunk per document, then reranks only chunks (O(chunks)). This was a motivated refactor based on performance analysis.

### 3. Position-Aware Score Blending

Final rankings blend retrieval scores with reranker scores based on position: top results favor retrieval signal (protection), lower results trust reranker more (refinement).

### 4. Virtual Path Format

The `qmd://collection/path` URI scheme provides explicit, unambiguous document addressing independent of filesystem paths.

### 5. Write-Through Config

Collection and context mutations write to both SQLite (runtime) and YAML (persistence) when config file exists. This ensures human-editability without losing runtime performance.

### 6. Session-Per-Client MCP

HTTP transport creates one MCP server instance per client session, following MCP spec, while sharing the SQLite store (safe for concurrent access).

### 7. Grammar-Constrained LLM Output

Query expansion uses grammar-guided decoding to produce structured output (lex:/vec:/hyde: lines) without post-hoc validation failures.

---

## Technical Debt Summary

| Area | Issue | Impact |
|------|-------|--------|
| **Search** | Sequential vector embedding (concurrent hangs) | Slower embedding phase |
| **Search** | Generic fallback expansion | Poor results on edge cases |
| **Search** | No query normalization for BM25 | Potentially lower recall |
| **Chunking** | Hardcoded 0.7 decay factor | Not configurable |
| **Retrieval** | All files loaded for glob matching | Memory inefficient |
| **Retrieval** | Levenshtein on full paths | Poor fuzzy suggestions for deep paths |
| **Retrieval** | No docid collision reporting | Silent data loss on collision |
| **Output** | CLI format is markdown fallback | No colored output |
| **Output** | Inconsistent score precision | Confusion in programmatic use |
| **Output** | No streaming support | Memory inefficient for large results |

---

## Appendix: Key Constants

```typescript
// Search
STRONG_SIGNAL_MIN_SCORE = 0.85;
STRONG_SIGNAL_MIN_GAP = 0.15;
RRF_K = 60;
POSITION_BLEND_RANK_1 = 0.75;  // Top 3: 75% RRF
POSITION_BLEND_RANK_2 = 0.60;  // Ranks 4-10: 60% RRF
POSITION_BLEND_RANK_3 = 0.40;  // Ranks 11+: 40% RRF

// Chunking
CHUNK_SIZE_TOKENS = 900;
CHUNK_OVERLAP_TOKENS = 135;  // 15% of 900
CHUNK_SIZE_CHARS = 3600;     // ~4 chars per token
CHUNK_WINDOW_TOKENS = 200;
CHUNK_WINDOW_CHARS = 800;

// MCP HTTP Transport
DEFAULT_INACTIVITY_TIMEOUT_MS = 5 * 60 * 1000;  // 5 minutes
