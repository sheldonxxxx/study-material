# Feature Deep Dive: Batch 2

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Date:** 2026-03-27

## Features Analyzed
1. MCP Server Integration
2. SDK/Library API
3. Smart Document Chunking

---

## 1. MCP Server Integration

**File:** `src/mcp/server.ts`

### Overview

The MCP server exposes QMD as a Model Context Protocol server, enabling AI agents (Claude, Cursor, etc.) to search and retrieve documents through MCP tools and resources. The implementation supports two transport mechanisms: stdio (for local CLI integration) and Streamable HTTP (for networked access).

### Transport Architecture

**Stdio Transport (lines 522-527):**
```typescript
export async function startMcpServer(): Promise<void> {
  const store = await createStore({ dbPath: getDefaultDbPath() });
  const server = await createMcpServer(store);
  const transport = new StdioServerTransport();
  await server.connect(transport);
}
```

**HTTP Transport (lines 543-802):**
- Uses `@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js`
- Session-based: each client gets own McpServer + Transport pair
- Session ID generated via `randomUUID()`
- Shared store (SQLite is safe for concurrent access)
- Endpoints:
  - `GET /health` - health check with uptime
  - `POST /query` or `POST /search` - REST-style search (no MCP protocol overhead)
  - `POST /mcp` - MCP JSON-RPC endpoint
  - `GET /mcp` - Session state retrieval
  - `DELETE /mcp` - Session cleanup

**Session Management (lines 549-572):**
```typescript
const sessions = new Map<string, WebStandardStreamableHTTPServerTransport>();

async function createSession(): Promise<WebStandardStreamableHTTPServerTransport> {
  const transport = new WebStandardStreamableHTTPServerTransport({
    sessionIdGenerator: () => randomUUID(),
    enableJsonResponse: true,
    onsessioninitialized: (sessionId: string) => {
      sessions.set(sessionId, transport);
    },
  });
  // ...
}
```

### Tool Definitions

#### Tool: `query` (Primary search tool)

Signature (lines 224-341):
```typescript
server.registerTool(
  "query",
  {
    title: "Query",
    inputSchema: {
      searches: z.array(subSearchSchema).min(1).max(10),
      limit: z.number().optional().default(10),
      minScore: z.number().optional().default(0),
      candidateLimit: z.number().optional(),
      collections: z.array(z.string()).optional(),
      intent: z.string().optional(),
    },
  },
  async ({ searches, limit, minScore, candidateLimit, collections, intent }) => {
    // ... executes search via store.search()
  }
);
```

Sub-search schema supports three query types:
- `lex` - BM25 keyword search
- `vec` - Semantic vector search
- `hyde` - Hypothetical document search

#### Tool: `get` (Retrieve single document)

Signature (lines 347-406):
- Supports `:line` suffix parsing (e.g., `"foo.md:120"`)
- Falls back to similar file suggestions on 404
- Returns document as MCP resource with `qmd://` URI

#### Tool: `multi_get` (Batch retrieval)

Signature (lines 412-479):
- Supports glob patterns and comma-separated lists
- Skips large files with reason explanation
- Returns truncated notice if maxLines exceeded

#### Tool: `status` (Index health)

Returns structured data:
```typescript
type StatusResult = {
  totalDocuments: number;
  needsEmbedding: number;
  hasVectorIndex: boolean;
  collections: {
    name: string;
    path: string | null;
    pattern: string | null;
    documents: number;
    lastUpdated: string;
  }[];
};
```

### Resource URI Scheme

Documents accessible via `qmd://` URIs (lines 64-67, 172-207):
```typescript
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

### Dynamic Instructions

The server builds instructions at startup (lines 92-152) that are injected into the LLM's system prompt via the MCP initialize response. This gives the AI immediate context about:
- Total documents and collections
- What search types are available
- Search syntax examples
- Tips for best results

### Clever Design Decisions

1. **REST endpoint bypass** (lines 626-675): `POST /query` allows structured search without MCP protocol overhead, useful for direct HTTP integration.

2. **First lex/vec query extraction** (lines 320-322):
```typescript
const primaryQuery = searches.find(s => s.type === 'lex')?.query
  || searches.find(s => s.type === 'vec')?.query
  || searches[0]?.query || "";
```
Used for snippet extraction priority.

3. **Write-through config** (line 421-423): When YAML config exists, mutations write to both SQLite and YAML.

---

## 2. SDK/Library API

**File:** `src/index.ts`

### Overview

The SDK provides programmatic Node.js/Bun access to QMD's search, retrieval, and indexing capabilities. The public API is clean and minimal, wrapping an internal store implementation.

### Core Exports

**Types exported** (lines 85-106):
- `DocumentResult`, `DocumentNotFound` - document retrieval results
- `SearchResult`, `HybridQueryResult` - search results
- `HybridQueryOptions`, `HybridQueryExplain` - search configuration
- `ExpandedQuery` - typed query expansion result
- `StructuredSearchOptions`, `MultiGetResult` - advanced search
- `IndexStatus`, `IndexHealthInfo` - health checks
- `SearchHooks` - progress callbacks
- `ReindexProgress`, `ReindexResult` - indexing status
- `EmbedProgress`, `EmbedResult` - embedding status
- `Collection`, `CollectionConfig`, `NamedCollection`, `ContextMap` - config types

**Functions exported** (lines 112-118):
```typescript
export { extractSnippet, addLineNumbers, DEFAULT_MULTI_GET_MAX_BYTES };
export { getDefaultDbPath } from "./store.js";
export { Maintenance } from "./maintenance.js";
```

### `createStore` Implementation

The main entry point (lines 333-528):

```typescript
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

  // Return QMDStore interface
  return store;
}
```

### QMDStore Interface

The interface (lines 212-306) provides:

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
- `addCollection(name, opts)` - Add/update collection
- `removeCollection(name)` - Remove collection
- `renameCollection(oldName, newName)` - Rename
- `listCollections()` - List with stats
- `getDefaultCollectionNames()` - Collections included by default

**Context Management:**
- `addContext(collectionName, pathPrefix, contextText)` - Add path context
- `removeContext(collectionName, pathPrefix)` - Remove context
- `setGlobalContext(context)` - Set global context
- `getGlobalContext()` - Get global context
- `listContexts()` - List all contexts

**Indexing:**
- `update(options?)` - Re-index from filesystem
- `embed(options?)` - Generate vector embeddings

**Health:**
- `getStatus()` - Index status
- `getIndexHealth()` - Health info

**Lifecycle:**
- `close()` - Release all resources

### Write-Through Pattern

For collection and context mutations, when YAML config exists, the SDK writes to both SQLite AND the YAML file (lines 419-437):

```typescript
addCollection: async (name, opts) => {
  upsertStoreCollection(db, name, { path: opts.path, pattern: opts.pattern, ignore: opts.ignore });
  if (hasYamlConfig || options.config) {
    collectionsAddCollection(name, opts.path, opts.pattern);
  }
},
```

### Progress Callbacks

Both `update()` and `embed()` support progress callbacks:

```typescript
update(options?: {
  collections?: string[];
  onProgress?: (info: UpdateProgress) => void;
}): Promise<UpdateResult>;

embed(options?: {
  force?: boolean;
  model?: string;
  maxDocsPerBatch?: number;
  maxBatchBytes?: number;
  onProgress?: (info: EmbedProgress) => void;
}): Promise<EmbedResult>;
```

### Type System Design

The SDK re-exports types from internal store but also exposes `InternalStore` for advanced consumers (line 109):
```typescript
export type { InternalStore };
```

This allows advanced users to access internal methods while maintaining type safety.

---

## 3. Smart Document Chunking

**File:** `src/store.ts`

### Overview

Smart chunking splits documents into ~900 token chunks with 15% overlap, preferring markdown-aware boundaries (headings, code blocks, paragraph breaks). This preserves semantic coherence better than fixed-size splits.

### Constants (lines 50-59)

```typescript
// Chunking: 900 tokens per chunk with 15% overlap
export const CHUNK_SIZE_TOKENS = 900;
export const CHUNK_OVERLAP_TOKENS = Math.floor(CHUNK_SIZE_TOKENS * 0.15);  // 135 tokens

// Fallback char-based approximation (~4 chars per token)
export const CHUNK_SIZE_CHARS = CHUNK_SIZE_TOKENS * 4;  // 3600 chars
export const CHUNK_OVERLAP_CHARS = CHUNK_OVERLAP_TOKENS * 4;  // 540 chars

// Search window for finding optimal break points (~200 tokens)
export const CHUNK_WINDOW_TOKENS = 200;
export const CHUNK_WINDOW_CHARS = CHUNK_WINDOW_TOKENS * 4;  // 800 chars
```

### Break Point Types and Scores

**BREAK_PATTERNS** (lines 97-110):
```typescript
export const BREAK_PATTERNS: [RegExp, number, string][] = [
  [/\n#{1}(?!#)/g, 100, 'h1'],     // # but not ##
  [/\n#{2}(?!#)/g, 90, 'h2'],      // ## but not ###
  [/\n#{3}(?!#)/g, 80, 'h3'],      // ### but not ####
  [/\n#{4}(?!#)/g, 70, 'h4'],      // #### but not #####
  [/\n#{5}(?!#)/g, 60, 'h5'],      // ##### but not ######
  [/\n#{6}(?!#)/g, 50, 'h6'],      // ######
  [/\n```/g, 80, 'codeblock'],     // code block boundary
  [/\n(?:---|\*\*\*|___)\s*\n/g, 60, 'hr'],  // horizontal rule
  [/\n\n+/g, 20, 'blank'],         // paragraph boundary
  [/\n[-*]\s/g, 5, 'list'],         // unordered list item
  [/\n\d+\.\s/g, 5, 'numlist'],     // ordered list item
  [/\n/g, 1, 'newline'],            // minimal break
];
```

Key insight: **H1 headings score 100, H2 score 90, down to newlines at 1.** This ensures headings decisively beat lower-quality breaks.

### Code Fence Protection

**findCodeFences** (lines 144-166) - Never split inside code blocks:
```typescript
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
  // Handle unclosed fence
  if (inFence) {
    regions.push({ start: fenceStart, end: text.length });
  }
  return regions;
}
```

### Break Point Scanning

**scanBreakPoints** (lines 117-138):
```typescript
export function scanBreakPoints(text: string): BreakPoint[] {
  const points: BreakPoint[] = [];
  const seen = new Map<number, BreakPoint>();  // pos -> best at that pos

  for (const [pattern, score, type] of BREAK_PATTERNS) {
    for (const match of text.matchAll(pattern)) {
      const pos = match.index!;
      const existing = seen.get(pos);
      // Keep higher score if position already seen
      if (!existing || score > existing.score) {
        seen.set(pos, { pos, score, type });
      }
    }
  }
  return Array.from(seen.values()).sort((a, b) => a.pos - b.pos);
}
```

Important: When multiple patterns match the same position, **higher score wins**. So a line matching both `# Header` (h1, 100) and `---` (hr, 60) gets the h1 score.

### Finding Best Cut Position

**findBestCutoff** (lines 188-224) uses squared distance decay:
```typescript
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
    if (bp.pos > targetCharPos) break;  // sorted, can stop early

    if (isInsideCodeFence(bp.pos, codeFences)) continue;

    const distance = targetCharPos - bp.pos;
    // Squared distance decay
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
- At window edge: multiplier = 0.3

This allows headings far back to still win over low-quality breaks near the target.

### Main Chunking Function

**chunkDocument** (lines 2024-2085):
```typescript
export function chunkDocument(
  content: string,
  maxChars: number = CHUNK_SIZE_CHARS,
  overlapChars: number = CHUNK_OVERLAP_CHARS,
  windowChars: number = CHUNK_WINDOW_CHARS
): { text: string; pos: number }[] {
  if (content.length <= maxChars) {
    return [{ text: content, pos: 0 }];
  }

  // Pre-scan all break points and code fences once
  const breakPoints = scanBreakPoints(content);
  const codeFences = findCodeFences(content);

  const chunks: { text: string; pos: number }[] = [];
  let charPos = 0;

  while (charPos < content.length) {
    const targetEndPos = Math.min(charPos + maxChars, content.length);
    let endPos = targetEndPos;

    if (endPos < content.length) {
      const bestCutoff = findBestCutoff(
        breakPoints, targetEndPos, windowChars, 0.7, codeFences
      );
      if (bestCutoff > charPos && bestCutoff <= targetEndPos) {
        endPos = bestCutoff;
      }
    }

    // Ensure progress
    if (endPos <= charPos) {
      endPos = Math.min(charPos + maxChars, content.length);
    }

    chunks.push({ text: content.slice(charPos, endPos), pos: charPos });

    // Overlap: move back by overlap amount
    if (endPos >= content.length) break;
    charPos = endPos - overlapChars;

    // Prevent infinite loop
    const lastChunkPos = chunks.at(-1)!.pos;
    if (charPos <= lastChunkPos) {
      charPos = endPos;
    }
  }
  return chunks;
}
```

### Token-Based Refinement

**chunkDocumentByTokens** (lines 2091-2137) is the async version that:
1. First chunks in character space with conservative estimate (3 chars/token)
2. Then tokenizes each chunk
3. If any chunk exceeds limit, re-splits with actual token ratio

```typescript
export async function chunkDocumentByTokens(
  content: string,
  maxTokens: number = CHUNK_SIZE_TOKENS,
  overlapTokens: number = CHUNK_OVERLAP_TOKENS,
  windowTokens: number = CHUNK_WINDOW_TOKENS
): Promise<{ text: string; pos: number; tokens: number }[]> {
  const llm = getDefaultLlamaCpp();

  // Conservative estimate: 3 chars per token
  const maxChars = maxTokens * 3;
  const overlapChars = overlapTokens * 3;
  const windowChars = windowTokens * 3;

  let charChunks = chunkDocument(content, maxChars, overlapChars, windowChars);
  const results: { text: string; pos: number; tokens: number }[] = [];

  for (const chunk of charChunks) {
    const tokens = await llm.tokenize(chunk.text);

    if (tokens.length <= maxTokens) {
      results.push({ text: chunk.text, pos: chunk.pos, tokens: tokens.length });
    } else {
      // Re-split with actual ratio
      const actualCharsPerToken = chunk.text.length / tokens.length;
      const safeMaxChars = Math.floor(maxTokens * actualCharsPerToken * 0.95);

      const subChunks = chunkDocument(chunk.text, safeMaxChars,
        Math.floor(overlapChars * actualCharsPerToken / 2),
        Math.floor(windowChars * actualCharsPerToken / 2));

      for (const subChunk of subChunks) {
        const subTokens = await llm.tokenize(subChunk.text);
        results.push({
          text: subChunk.text,
          pos: chunk.pos + subChunk.pos,
          tokens: subTokens.length,
        });
      }
    }
  }
  return results;
}
```

### Snippet Extraction

**extractSnippet** (lines 3558-3624) finds the best snippet given a query:
```typescript
export function extractSnippet(body: string, query: string, maxLen = 500,
  chunkPos?: number, chunkLen?: number, intent?: string): SnippetResult {
  // If chunkPos provided, search within that region with padding
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

  // Format: @@ -start,count @@ (linesBefore before, linesAfter after)
  const header = `@@ -${absoluteStart},${snippetLineCount} @@ ...`;
}
```

### Technical Debt / Observations

1. **BREAK_PATTERNS uses global `/g` flag but then uses `matchAll`** (line 122) - This is correct but potentially confusing; the `g` is needed for `matchAll` to work properly.

2. **Comment says 800 tokens but constant is 900** (line 51): "Increased from 800 to accommodate smart chunking" - suggests 800 was previous value.

3. **Hardcoded 0.7 decay factor** (line 54): Passed explicitly but not configurable via options.

4. **Levenshtein distance function** (lines 2143-2161) exists but is only used for similar file suggestions - not used in chunking itself.

5. **No minimum chunk size enforcement** - Very small chunks could result from aggressive break point preferences near document boundaries.
