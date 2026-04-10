# Feature Deep Dive - Batch 4

**Date:** 2026-03-27
**Features Analyzed:**
1. HTTP Transport for MCP
2. Custom Embedding Models
3. Index Maintenance

---

## 1. HTTP Transport for MCP

### Overview

The MCP server supports two transports: stdio (default) and Streamable HTTP. HTTP mode enables long-lived daemon operation with persistent model loading.

### Architecture

**Core file:** `src/mcp/server.ts`

The MCP server uses the Model Context Protocol SDK with a custom HTTP transport implementation:

```typescript
import { WebStandardStreamableHTTPServerTransport }
  from "@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js";
```

### Daemon Mode Implementation

**CLI entry point:** `src/cli/qmd.ts`

When `qmd mcp --http --daemon` is invoked:

1. **PID File Management** (lines 3023-3078):
```typescript
const cacheDir = process.env.XDG_CACHE_HOME
  ? resolve(process.env.XDG_CACHE_HOME, "qmd")
  : resolve(homedir(), ".cache", "qmd");
const pidPath = resolve(cacheDir, "mcp.pid");
```

2. **Stale PID Detection:**
```typescript
if (existsSync(pidPath)) {
  const existingPid = parseInt(readFileSync(pidPath, "utf-8").trim());
  try {
    process.kill(existingPid, 0); // alive?
    console.error(`Already running (PID ${existingPid}). Run 'qmd mcp stop' first.`);
    process.exit(1);
  } catch {
    // Stale PID file — continue
  }
}
```

3. **Background Spawn:**
```typescript
const child = nodeSpawn(process.execPath, spawnArgs, {
  stdio: ["ignore", logFd, logFd],  // Redirect stdout/stderr to log file
  detached: true,                    // Detach from parent process
});
child.unref();                       // Allow parent to exit independently
writeFileSync(pidPath, String(child.pid));
```

### Session Management

HTTP mode requires **one MCP server instance per client session** (per MCP spec). The implementation handles this with a session map:

```typescript
// Session map: each client gets its own McpServer + Transport pair (MCP spec requirement).
// The store is shared — it's stateless SQLite, safe for concurrent access.
const sessions = new Map<string, WebStandardStreamableHTTPServerTransport>();
```

**Session lifecycle:**
```typescript
const transport = new WebStandardStreamableHTTPServerTransport({
  sessionIdGenerator: () => randomUUID(),
  enableJsonResponse: true,
  onsessioninitialized: (sessionId: string) => {
    sessions.set(sessionId, transport);
    log(`${ts()} New session ${sessionId} (${sessions.size} active)`);
  },
});

transport.onclose = () => {
  if (transport.sessionId) {
    sessions.delete(transport.sessionId);
  }
};
```

### HTTP Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Health check with uptime |
| `/query` or `/search` | POST | Direct REST search (no MCP protocol) |
| `/mcp` | GET/POST/DELETE | MCP protocol handler |

**REST Search endpoint (lines 626-675):**
```typescript
if ((pathname === "/query" || pathname === "/search") && nodeReq.method === "POST") {
  const rawBody = await collectBody(nodeReq);
  const params = JSON.parse(rawBody);

  // Validate required fields
  if (!params.searches || !Array.isArray(params.searches)) {
    nodeRes.writeHead(400, { "Content-Type": "application/json" });
    nodeRes.end(JSON.stringify({ error: "Missing required field: searches (array)" }));
    return;
  }

  // Map to internal format
  const queries: ExpandedQuery[] = params.searches.map((s: any) => ({
    type: s.type as 'lex' | 'vec' | 'hyde',
    query: String(s.query || ""),
  }));

  const results = await store.search({ queries, ... });
  nodeRes.writeHead(200, { "Content-Type": "application/json" });
  nodeRes.end(JSON.stringify({ results: formatted }));
}
```

### Graceful Shutdown

The server handles SIGTERM and SIGINT signals for clean shutdown:

```typescript
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

process.on("SIGTERM", async () => {
  console.error("Shutting down (SIGTERM)...");
  await stop();
  process.exit(0);
});
process.on("SIGINT", async () => {
  console.error("Shutting down (SIGINT)...");
  await stop();
  process.exit(0);
});
```

### Model Lifecycle in Daemon Mode

The `LlamaCpp` class manages model lifecycle with inactivity timeouts:

```typescript
// Default inactivity timeout: 5 minutes (keep models warm during typical search sessions)
const DEFAULT_INACTIVITY_TIMEOUT_MS = 5 * 60 * 1000;
```

**Auto-unload behavior:**
- Models stay loaded for 5 minutes after last activity
- Contexts are unloaded first (per node-llama-cpp lifecycle guidance)
- If `disposeModelsOnInactivity` is true, models are also disposed

### Stop Command

```typescript
if (sub === "stop") {
  if (!existsSync(pidPath)) {
    console.log("Not running (no PID file).");
    process.exit(0);
  }
  const pid = parseInt(readFileSync(pidPath, "utf-8").trim());
  try {
    process.kill(pid, 0); // alive?
    process.kill(pid, "SIGTERM");
    unlinkSync(pidPath);
    console.log(`Stopped QMD MCP server (PID ${pid}).`);
  } catch {
    unlinkSync(pidPath);
    console.log("Cleaned up stale PID file (server was not running).");
  }
  process.exit(0);
}
```

---

## 2. Custom Embedding Models

### Overview

QMD supports custom embedding models via environment variables and provides multiple model options with different prompting strategies.

### Configuration

**Environment variable:** `QMD_EMBED_MODEL`

**Default models** (`src/llm.ts`):

```typescript
const DEFAULT_EMBED_MODEL = process.env.QMD_EMBED_MODEL
  ?? "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf";

const DEFAULT_RERANK_MODEL = "hf:ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF/qwen3-reranker-0.6b-q8_0.gguf";

const DEFAULT_GENERATE_MODEL = "hf:tobil/qmd-query-expansion-1.7B-gguf/qmd-query-expansion-1.7B-q4_k_m.gguf";
```

### Alternative Models

The codebase includes support for alternative models:

```typescript
// LiquidAI LFM2 - hybrid architecture optimized for edge/on-device inference
export const LFM2_GENERATE_MODEL = "hf:LiquidAI/LFM2-1.2B-GGUF/LFM2-1.2B-Q4_K_M.gguf";
export const LFM2_INSTRUCT_MODEL = "hf:LiquidAI/LFM2.5-1.2B-Instruct-GGUF/LFM2.5-1.2B-Instruct-Q4_K_M.gguf";
```

### Model Format Detection

The system detects model type by URI to apply appropriate prompting:

```typescript
export function isQwen3EmbeddingModel(modelUri: string): boolean {
  return /qwen.*embed/i.test(modelUri) || /embed.*qwen/i.test(modelUri);
}
```

### Query Formatting

Different models use different prompting formats:

**For Qwen3-Embedding models:**
```typescript
export function formatQueryForEmbedding(query: string, modelUri?: string): string {
  const uri = modelUri ?? process.env.QMD_EMBED_MODEL ?? DEFAULT_EMBED_MODEL;
  if (isQwen3EmbeddingModel(uri)) {
    return `Instruct: Retrieve relevant documents for the given query\nQuery: ${query}`;
  }
  return `task: search result | query: ${query}`;
}
```

**For Nomic/embeddinggemma models (default):**
```typescript
  return `task: search result | query: ${query}`;
```

### Document Formatting

```typescript
export function formatDocForEmbedding(text: string, title?: string, modelUri?: string): string {
  const uri = modelUri ?? process.env.QMD_EMBED_MODEL ?? DEFAULT_EMBED_MODEL;
  if (isQwen3EmbeddingModel(uri)) {
    // Qwen3-Embedding: documents are raw text, no task prefix
    return title ? `${title}\n${text}` : text;
  }
  return `title: ${title || "none"} | text: ${text}`;
}
```

### Model Caching

Models are cached in `~/.cache/qmd/models/` with ETag-based refresh:

```typescript
async function getRemoteEtag(ref: HfRef): Promise<string | null> {
  const url = `https://huggingface.co/${ref.repo}/resolve/main/${ref.file}`;
  const resp = await fetch(url, { method: "HEAD" });
  return resp.headers.get("etag");
}

export async function pullModels(models: string[], options: { refresh?: boolean }): Promise<PullResult[]> {
  // Downloads models from HuggingFace if not cached or if etag changed
  const shouldRefresh = options.refresh || !remoteEtag || remoteEtag !== localEtag || cached.length === 0;
}
```

### Using Custom Models in Code

**Via environment variable:**
```bash
export QMD_EMBED_MODEL=hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf
```

**Via SDK** (`src/index.ts`):
```typescript
embed(options?: {
  force?: boolean;
  model?: string;  // Custom model URI
  maxDocsPerBatch?: number;
  maxBatchBytes?: number;
  onProgress?: (info: EmbedProgress) => void;
}): Promise<EmbedResult>;
```

**Via store** (`src/store.ts`):
```typescript
const model = options?.model ?? DEFAULT_EMBED_MODEL;
// ...
const result = await session.embedBatch(formatted, { model });
```

### Parallel Embedding Contexts

The system automatically computes parallelism based on hardware:

```typescript
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

For embeddings (~150MB per context), this allows up to 8 parallel contexts on GPU or up to 4 on CPU.

---

## 3. Index Maintenance

### Overview

QMD provides comprehensive maintenance operations for database cleanup, including orphaned content removal, cache clearing, and database vacuuming.

### Maintenance Class

**File:** `src/maintenance.ts`

```typescript
export class Maintenance {
  private store: Store;

  constructor(store: Store) {
    this.store = store;
  }

  vacuum(): void { ... }
  cleanupOrphanedContent(): number { ... }
  cleanupOrphanedVectors(): number { ... }
  clearLLMCache(): number { ... }
  deleteInactiveDocs(): number { ... }
  clearEmbeddings(): void { ... }
}
```

### Operations Detail

#### 1. Vacuum Database

Reclaims unused space by rebuilding the SQLite database:

```typescript
vacuumDatabase(db: Database): void {
  db.exec(`VACUUM`);
}
```

**When to use:** After deleting many documents or clearing embeddings to reclaim disk space.

#### 2. Cleanup Orphaned Content

Removes content hashes no longer referenced by any active document:

```typescript
export function cleanupOrphanedContent(db: Database): number {
  const result = db.prepare(`
    DELETE FROM content
    WHERE hash NOT IN (SELECT DISTINCT hash FROM documents WHERE active = 1)
  `).run();
  return result.changes;
}
```

**Returns:** Number of orphaned content rows deleted.

#### 3. Cleanup Orphaned Vectors

Removes vector embeddings for content that no longer exists:

```typescript
export function cleanupOrphanedVectors(db: Database): number {
  // sqlite-vec may not be loaded (e.g. Bun's bun:sqlite lacks loadExtension).
  if (!isSqliteVecAvailable()) {
    return 0;
  }
  // Delete orphaned vectors...
}
```

**Safety check:** Returns 0 if sqlite-vec is not available (prevents crash on Bun).

#### 4. Clear LLM Cache

Deletes cached LLM responses used for query expansion and reranking:

```typescript
export function deleteLLMCache(db: Database): number {
  const result = db.prepare(`DELETE FROM llm_cache`).run();
  return result.changes;
}
```

**Returns:** Number of cached entries deleted.

#### 5. Delete Inactive Documents

Removes documents marked as inactive (removed from filesystem):

```typescript
export function deleteInactiveDocuments(db: Database): number {
  const result = db.prepare(`DELETE FROM documents WHERE active = 0`).run();
  return result.changes;
}
```

**Returns:** Number of inactive documents deleted.

#### 6. Clear All Embeddings

Removes all vector embeddings, forcing re-embedding on next query:

```typescript
export function clearAllEmbeddings(db: Database): void {
  db.exec(`DELETE FROM content_vectors`);
  db.exec(`DROP TABLE IF EXISTS vectors_vec`);
}
```

**Note:** This drops and recreates the vectors table on next embedding operation.

### Index Health Monitoring

**File:** `src/store.ts`

```typescript
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

**Health indicators:**
- `needsEmbedding`: Documents that have content but no vectors
- `totalDocs`: Active documents in the index
- `daysStale`: Days since last document modification (null if no docs)

### Re-indexing

**File:** `src/store.ts` - `reindexCollection` function

```typescript
export async function reindexCollection(
  store: Store,
  collectionPath: string,
  globPattern: string,
  collectionName: string,
  options?: ReindexOptions
): Promise<ReindexResult> {
  // 1. Scan filesystem for files matching pattern
  // 2. Compare with existing documents
  // 3. Mark removed files as inactive (active = 0)
  // 4. Add/update changed files
  // 5. Return statistics

  const orphanedCleaned = cleanupOrphanedContent(db);
  return { indexed, updated, unchanged, removed, orphanedCleaned };
}
```

### Embedding Generation with Options

```typescript
export type EmbedOptions = {
  force?: boolean;           // Clear all embeddings before generating
  model?: string;            // Custom embedding model URI
  maxDocsPerBatch?: number;  // Max docs per batch (default: 10)
  maxBatchBytes?: number;    // Max bytes per batch (default: 10MB)
  onProgress?: (info: EmbedProgress) => void;
};

export type EmbedProgress = {
  chunksEmbedded: number;
  totalChunks: number;
  bytesProcessed: number;
  totalBytes: number;
  errors: number;
};
```

### Using Maintenance via CLI

```bash
# Re-index all collections (syncs filesystem to database)
qmd update

# Generate embeddings
qmd embed

# Force re-generate all embeddings
qmd embed --force

# Show index status
qmd status
```

### Using Maintenance via SDK

```typescript
import { createStore, Maintenance } from 'qmd';

const store = await createStore({ dbPath: './my-index.sqlite' });
const maintenance = new Maintenance(store.internal);

// Run maintenance operations
maintenance.vacuum();
maintenance.clearEmbeddings();
const orphanedContent = maintenance.cleanupOrphanedContent();
const orphanedVectors = maintenance.cleanupOrphanedVectors();
const cacheCleared = maintenance.clearLLMCache();
const inactiveDeleted = maintenance.deleteInactiveDocs();

// Check health
const health = await store.getIndexHealth();
console.log(`Needs embedding: ${health.needsEmbedding}`);
console.log(`Days stale: ${health.daysStale}`);

await store.close();
```

---

## Summary

| Feature | Key Implementation | Notable Details |
|---------|-------------------|-----------------|
| **HTTP Transport** | `src/mcp/server.ts` | Session-per-client model, SQLite shared across sessions, PID-based daemon management |
| **Custom Embeddings** | `src/llm.ts` | `QMD_EMBED_MODEL` env var, Qwen3 vs nomic prompting, ETag-based model refresh |
| **Index Maintenance** | `src/maintenance.ts` + `src/store.ts` | 6 cleanup operations, health monitoring, sqlite-vec availability check |
