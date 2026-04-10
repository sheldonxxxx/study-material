# Security and Performance Analysis: QMD

**Project:** Query Markup Documents (qmd)
**Analysis Date:** 2026-03-27
**Files Analyzed:** `src/db.ts`, `src/llm.ts`, `src/store.ts`, `src/mcp/server.ts`, `src/collections.ts`, `src/index.ts`, `src/cli/qmd.ts`, `src/maintenance.ts`

---

## Executive Summary

QMD is a local-first knowledge base search system with no built-in authentication or access control. Security relies on filesystem permissions and network binding choices. The system demonstrates strong input sanitization for SQL and FTS queries, comprehensive output encoding, and thoughtful resource management for LLM operations.

---

## 1. Secrets Management

### Approach: Environment Variables + Local Model Cache

| Aspect | Implementation | Assessment |
|--------|---------------|------------|
| **API Keys/Secrets** | None required - all models run locally via HuggingFace GGUF files | Strong - no external API dependencies |
| **Model Caching** | `~/.cache/qmd/models/` with etag-based refresh | Good - follows XDG base directory spec |
| **Config Storage** | `~/.config/qmd/index.yml` (collections, contexts) | Good - user-controlled, not in code |
| **Database Path** | `~/.cache/qmd/index.sqlite` (default) | Good - follows XDG cache spec |

### Environment Variables Used

```
QMD_EMBED_MODEL        # Override embedding model
QMD_EXPAND_CONTEXT_SIZE # Query expansion context size
QMD_CONFIG_DIR         # Custom config directory (testing)
XDG_CONFIG_HOME        # Config base (XDG compliance)
XDG_CACHE_HOME         # Cache base (XDG compliance)
INDEX_PATH            # Custom index database path
HOME                  # Fallback for homedir
PWD                   # Current working directory
CI                    # Disable LLM operations in tests
WSL_DISTRO_NAME       # WSL detection
```

### Strengths
- No hardcoded credentials or API keys
- Models downloaded from HuggingFace (public models)
- ETag-based cache validation prevents unnecessary re-downloads

### Gaps
- No mechanism to encrypt the SQLite database at rest
- No secrets management for potential future remote API integrations
- Model cache directory permissions depend on OS-level protection

---

## 2. Input Validation and Sanitization

### FTS5 Query Sanitization (`store.ts`)

```typescript
function sanitizeFTS5Term(term: string): string {
  return term.replace(/[^\p{L}\p{N}']/gu, '').toLowerCase();
}
```

**Lex Query Validation:**
- Newlines rejected: `validateLexQuery()` returns error if query contains `\r\n`
- Unmatched quotes detected: odd quote count triggers error
- Special FTS5 operators handled: negation (`-term`), phrase matching (`"exact"`)

**Query Building:**
```typescript
// Positive terms AND'd together
let result = positive.join(' AND ');
// Negative terms added as NOT clauses
for (const neg of negative) {
  result = `${result} NOT ${neg}`;
}
```

### Semantic Query Validation (`validateSemanticQuery`)

```typescript
export function validateSemanticQuery(query: string): string | null {
  // Check for negation syntax - not supported in vec/hyde
  if (/-\w/.test(query) || /-"/.test(query)) {
    return 'Negation (-term) is not supported in vec/hyde queries...';
  }
  return null;
}
```

### SQL Injection Prevention

**All database queries use parameterized statements:**

```typescript
// store.ts line 2780
const params: (string | number)[] = [ftsQuery];
if (collectionName) {
  sql += ` AND d.collection = ?`;
  params.push(String(collectionName));
}
const rows = db.prepare(sql).all(...params);
```

**Batch queries use placeholder arrays:**
```typescript
// store.ts line 1284
const placeholders = batch.map(() => "?").join(",");
const rows = db.prepare(`
  SELECT hash, doc as body FROM content
  WHERE hash IN (${placeholders})
`).all(...batch.map(doc => doc.hash));
```

### Output Encoding (`cli/formatter.ts`)

```typescript
export function escapeCSV(value: string | null | number): string {
  // CSV escaping with proper quote handling
}

export function escapeXml(str: string): string {
  // XML entity encoding: & < > " '
}
```

**XML escaping applied to:**
- File paths in results
- Document titles
- Context fields
- Snippet content

### Path Validation

- Collection names validated: `/^[a-zA-Z0-9_-]+$/` (alphanumeric, hyphens, underscores only)
- File paths processed through `handelize()` for token-friendly normalization
- Virtual path parsing with `qmd://` URI scheme

---

## 3. Authentication and Authorization

### Local-Only Design

QMD has **no authentication layer**:
- No user accounts
- No access control lists
- No permission system

### Security Model

Access control relies on:
1. **Filesystem permissions** - user must have read access to indexed directories
2. **Network binding** - HTTP MCP server binds to `localhost` only (not exposed externally)
3. **Process isolation** - each user runs their own instance

### MCP Server Security (HTTP Transport)

```typescript
// server.ts line 772
httpServer.listen(port, "localhost", () => resolve());
```

**Security features:**
- Binds to localhost only - not network-accessible
- Session-based authentication via `mcp-session-id` header
- JSON-RPC request validation
- Error handling with no stack trace exposure

**CORS:** Not configured - MCP HTTP is intended for local-only access

---

## 4. Performance Optimization Patterns

### Caching Strategy

**LLM Response Cache:**
- Query expansion results cached in SQLite
- Reranking results cached
- Reduces redundant LLM inference

**Document Cache:**
```typescript
// store.ts - document body caching
private cache = new Map<string, { body: string; mtime: number }>();
```

**Path Resolution Cache:**
```typescript
// store.ts - virtual path resolution
private pathCache = new Map<string, string>();
```

### Pagination and Batch Processing

**Embedding Batching:**
```typescript
function buildEmbeddingBatches(
  docs: PendingEmbeddingDoc[],
  maxDocsPerBatch: number,    // Default: 10 docs
  maxBatchBytes: number,      // Default: 10 MB
): PendingEmbeddingDoc[][]
```

**Search Result Limits:**
- Default limit: 10 results
- Configurable via `limit` parameter
- `minScore` threshold filtering

### SQLite Optimization

**FTS5 with BM25:**
```sql
bm25(documents_fts, 10.0, 1.0) as bm25_score
ORDER BY bm25_score ASC LIMIT ?
```

**Vector Search Two-Step Query:**
```typescript
// Step 1: Get vector matches (no JOINs - sqlite-vec limitation)
const vecResults = db.prepare(`
  SELECT hash_seq, distance FROM vectors_vec
  WHERE embedding MATCH ? AND k = ?
`).all(new Float32Array(embedding), limit * 3);

// Step 2: Resolve to full documents
```

**PRAGMA Settings:**
```typescript
// store.ts
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;  // 64MB cache
PRAGMA temp_store = MEMORY;
```

### Parallelism Management

**CPU-Bound Embedding Contexts:**
```typescript
private async computeParallelism(perContextMB: number): Promise<number> {
  if (llama.gpu) {
    // GPU: 25% of free VRAM, capped at 8 contexts
    const maxByVram = Math.floor((freeMB * 0.25) / perContextMB);
    return Math.max(1, Math.min(8, maxByVram));
  }
  // CPU: split cores, 4 threads minimum per context
  const maxContexts = Math.floor(cores / 4);
  return Math.max(1, Math.min(4, maxContexts));
}
```

### Memory Management (node-llama-cpp)

**Inactivity Timeout:**
```typescript
// 5 minute inactivity timeout - contexts unloaded after idle
private inactivityTimeoutMs: number = 5 * 60 * 1000;
```

**Context Disposal:**
```typescript
async unloadIdleResources(): Promise<void> {
  // Dispose contexts first
  for (const ctx of this.embedContexts) {
    await ctx.dispose();
  }
  this.embedContexts = [];
  // Optionally dispose models (opt-in)
  if (this.disposeModelsOnInactivity) {
    await this.embedModel.dispose();
  }
}
```

**Safe Dispose Pattern:**
```typescript
// llm.ts line 1246 - timeout to prevent hangs
const disposePromise = this.llama.dispose();
const timeoutPromise = new Promise<void>((resolve) => setTimeout(resolve, 1000));
await Promise.race([disposePromise, timeoutPromise]);
```

### Chunking Strategy

**Token-Based Chunking:**
```typescript
export const CHUNK_SIZE_TOKENS = 900;        // ~900 tokens per chunk
export const CHUNK_OVERLAP_TOKENS = 135;      // 15% overlap for context continuity
export const CHUNK_WINDOW_TOKENS = 200;      // Search window for break points
```

**Reranking Optimization:**
```typescript
// Rerank chunks NOT full bodies - O(chunks) not O(tokens)
// Critical perf lesson: reranking full bodies is O(tokens) trap
```

---

## 5. Resource Management

### Session Lifecycle (`LLMSessionManager`)

```typescript
class LLMSessionManager {
  private _activeSessionCount = 0;
  private _inFlightOperations = 0;

  canUnload(): boolean {
    return this._activeSessionCount === 0 && this._inFlightOperations === 0;
  }
}
```

**Max Duration Timer:**
```typescript
// Sessions auto-expire after 10 minutes (configurable)
const maxDuration = options.maxDuration ?? 10 * 60 * 1000;
this.maxDurationTimer = setTimeout(() => {
  this.abortController.abort(new Error(`Session exceeded max duration`));
}, maxDuration);
```

### Abort Signal Propagation

```typescript
constructor(manager: LLMSessionManager, options: LLMSessionOptions = {}) {
  // Link external abort signal if provided
  if (options.signal) {
    options.signal.addEventListener("abort", () => {
      this.abortController.abort(options.signal!.reason);
    }, { once: true });
  }
}
```

### Graceful Shutdown

```typescript
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

---

## 6. MCP Transport Security Considerations

### Stdio Transport (Default)

**Security:** Process-to-process via stdin/stdout
- No network exposure
- Inherits parent process permissions
- Suitable for local MCP clients (Claude Desktop, etc.)

### HTTP Transport

**Security Model:**
- Binds to `localhost` only
- Session-based with UUID identifiers
- No TLS/SSL (local-only design)

**Session Management:**
```typescript
const sessions = new Map<string, WebStandardStreamableHTTPServerTransport>();
// Each client gets own McpServer + Transport pair
// Store is shared (stateless SQLite, safe for concurrent access)
```

**Request Validation:**
```typescript
// Content-Type validation missing - accepts any body
// Session ID validation on GET/DELETE methods
```

### Potential Concerns

| Concern | Severity | Mitigation |
|---------|----------|------------|
| No TLS on HTTP transport | Low | Localhost-only binding |
| No request rate limiting | Medium | Single-user local tool |
| No CORS configuration | Low | Not intended for browser access |
| Session ID entropy | Good | Uses `randomUUID()` (cryptographic) |
| No request size limits | Medium | Could be exploited via large payloads |

---

## 7. Notable Strengths

### Security Strengths

1. **Parameterized SQL queries** - Complete protection against SQL injection
2. **Comprehensive output encoding** - CSV and XML escaping for all user-facing output
3. **FTS5 query sanitization** - Unicode-aware term filtering
4. **Local-only design** - No remote attack surface for core functionality
5. **No hardcoded secrets** - All configuration via environment/filesystem
6. **Input validation** - Query syntax validation before execution
7. **Error handling** - No stack traces exposed to users

### Performance Strengths

1. **VRAM-aware parallelism** - Dynamically computes context count based on GPU memory
2. **Batched embedding** - Documents processed in batches to prevent OOM
3. **Chunk-level reranking** - Avoids O(tokens) trap by reranking chunks not full docs
4. **WAL journal mode** - Concurrent read/write performance
5. **In-memory temp store** - Faster sorting and temporary data
6. **Lazy model loading** - Models loaded on-demand, not at startup
7. **Activity-based resource cleanup** - Idle timer with session awareness

---

## 8. Weaknesses and Recommendations

### Security Weaknesses

| Issue | Severity | Recommendation |
|-------|----------|----------------|
| No database encryption | Medium | Consider SQLCipher for sensitive documents |
| No access control | Medium | Document that this is a single-user tool |
| HTTP server has no rate limiting | Low | Add request throttling for production |
| No audit logging | Low | Log access patterns for security review |
| No input size limits | Medium | Add maximum query length limits |

### Performance Weaknesses

| Issue | Impact | Recommendation |
|-------|--------|----------------|
| Single DB connection | Limits concurrency | Connection pooling for HTTP server |
| No query result caching | Repeated queries hit LLM | Add TTL-based result cache |
| Vector search is memory-bound | Large corpora slow | Consider approximate nearest neighbor |

---

## 9. Code Quality Observations

### Robust Patterns

1. **Zod schema validation** for MCP tool inputs
2. **TypeScript strict mode** with comprehensive type definitions
3. **Error recovery** with graceful fallbacks (e.g., fallback to CPU if GPU fails)
4. **Platform detection** for macOS Homebrew SQLite paths
5. **Extension availability testing** before use

### Anti-Patterns Avoided

1. No `eval()` or dynamic code execution
2. No string concatenation for SQL
3. No shell command construction from user input
4. No synchronous file operations on hot paths (except CLI display)

---

## 10. Conclusion

QMD demonstrates a security-conscious design for a local-only knowledge base tool. The primary attack surface is the filesystem and local network (if HTTP MCP is enabled), not the application itself. Key security measures (parameterized queries, output encoding, input validation) are well-implemented.

The performance architecture shows careful consideration of resource constraints, particularly for the LLM operations which are inherently memory-intensive. The chunking strategy, batching approach, and inactivity timeout demonstrate understanding of the node-llama-cpp lifecycle.

**Overall Assessment:** Production-ready for single-user local deployment with no sensitive data requiring encryption at rest.
