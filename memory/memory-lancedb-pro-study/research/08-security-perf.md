# Security & Performance Patterns Analysis

## Project: memory-lancedb-pro

---

## 1. Secrets Management

### Approach: Environment Variable + Config File

The project uses a layered approach to secrets management:

**Environment Variable Substitution:**
- Config values support `${VAR}` syntax for environment variable interpolation
- `resolveEnvVars()` in `embedder.ts` and `index.ts` validates that referenced env vars exist
- Example from `index.ts`:
  ```typescript
  function resolveEnvVars(value: string): string {
    return value.replace(/\$\{([^}]+)\}/g, (_, envVar) => {
      const envValue = process.env[envVar];
      if (!envValue) {
        throw new Error(`Environment variable ${envVar} is not set`);
      }
      return envValue;
    });
  }
  ```

**API Key Handling:**
- Embedding API keys passed via `embedding.apiKey` config (single key or array for round-robin)
- LLM API keys via `llm.apiKey` or `embedding.apiKey` fallback
- Azure OpenAI uses `api-key` header instead of `Authorization: Bearer`
- Keys are resolved at construction time via `resolveEnvVars()`

**OAuth Secrets:**
- OAuth client ID can be overridden via `MEMORY_PRO_OAUTH_CLIENT_ID`
- OAuth authorize/token URLs overrideable via `MEMORY_PRO_OAUTH_AUTHORIZE_URL`, `MEMORY_PRO_OAUTH_TOKEN_URL`
- Redirect URI overrideable via `MEMORY_PRO_OAUTH_REDIRECT_URI`
- OAuth session file saved with **mode 0o600** (user read/write only):
  ```typescript
  await writeFile(authPath, JSON.stringify(payload, null, 2) + "\n", {
    encoding: "utf8",
    mode: 0o600,
  });
  ```

**`.gitignore` Completeness:**
- Includes `.memory-lancedb-pro/` (local database directory)
- Includes `memory-plugin-*` dev directories
- Does NOT explicitly exclude `.env` files (but no `.env` files exist in repo)
- Missing: `*.local.*` pattern for local override configs

---

## 2. Authentication & Authorization Patterns

### OAuth Implementation (llm-oauth.ts)

**Protocol:** OAuth 2.0 with PKCE (Proof Key for Code Exchange)

**Security Measures:**
| Feature | Implementation |
|---------|---------------|
| PKCE Challenge | SHA256 hash of verifier, base64url encoded |
| State Parameter | 16-byte random hex for CSRF protection |
| Token Storage | File with 0o600 permissions |
| Token Refresh | Automatic with 60s expiry skew buffer |
| Timeout Handling | AbortController with configurable timeout |

**PKCE Flow:**
```typescript
function createPkceVerifier(): string {
  return toBase64Url(randomBytes(32));
}

function createPkceChallenge(verifier: string): string {
  return createHash("sha256").update(verifier).digest("base64url");
}
```

**Token Refresh Logic:**
```typescript
const EXPIRY_SKEW_MS = 60_000;

export function needsRefresh(session: OAuthSession): boolean {
  return !!session.refreshToken && !!session.expiresAt &&
    session.expiresAt - EXPIRY_SKEW_MS <= Date.now();
}
```

**OAuth Providers:**
- Hardcoded provider: OpenAI Codex (`app_EMoamEEZ73f0CkXaXp7hrann`)
- Supports environment variable overrides for all OAuth endpoints
- Local callback server on `localhost:1455/auth/callback`

### Scope-Based Access Control (scopes.ts)

**Multi-Scope Isolation:**
- `MemoryScopeManager` implements scope-based access control
- Scope patterns:
  - `global` - shared across all agents
  - `agent:<agentId>` - per-agent isolated memory
  - `custom:<name>` - user-defined scopes
  - `project:<projectId>` - project-scoped memory
  - `user:<userId>` - user-scoped memory
  - `reflection:agent:<agentId>` - agent's reflection memories

**Bypass Protection:**
```typescript
const SYSTEM_BYPASS_IDS = new Set(["system", "undefined"]);

export function isSystemBypassId(agentId?: string): boolean {
  return typeof agentId === "string" && SYSTEM_BYPASS_IDS.has(agentId);
}
```

**Scope Validation:**
- `validateScope()` checks scope format validity
- `isAccessible()` enforces agent-specific scope access
- `getScopeFilter()` provides store-layer filtering

---

## 3. Input Validation & Sanitization

### SQL Injection Prevention (store.ts)

**Escaped Values:**
```typescript
function escapeSqlLiteral(value: string): string {
  return value.replace(/'/g, "''");
}
```

Used for:
- Memory IDs in queries
- Scope filters in WHERE clauses
- Category filters

**Query Building Pattern:**
```typescript
.map((scope) => `scope = '${escapeSqlLiteral(scope)}'`)
```

### Reflection Content Sanitization (reflection-slices.ts)

**Dangerous Pattern Detection:**
```typescript
const INJECTABLE_REFLECTION_BLOCK_PATTERNS: RegExp[] = [
  /^\s*(?:(?:next|this)\s+run\s+)?(?:ignore|disregard|forget|override|bypass)\b[\s\S]{0,80}\b(?:instructions?|guardrails?|policy|developer|system)\b/i,
  /\b(?:reveal|print|dump|show|output)\b[\s\S]{0,80}\b(?:system prompt|developer prompt|hidden prompt|hidden instructions?|full prompt|prompt verbatim|secrets?|keys?|tokens?)\b/i,
  /<\s*\/?\s*(?:system|assistant|user|tool|developer|inherited-rules|derived-focus)\b[^>]*>/i,
  /^(?:system|assistant|user|developer|tool)\s*:/i,
];
```

**Sanitization Functions:**
- `sanitizeReflectionSliceLines()` - removes placeholders, normalizes formatting
- `sanitizeInjectableReflectionLines()` - filters out unsafe injection patterns
- `isUnsafeInjectableReflectionLine()` - checks against dangerous patterns

### Regex Safety (tools.ts, auto-capture-cleanup.ts)

```typescript
function escapeRegExp(input: string): string {
  return input.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

### Query Normalization (store.ts)

```typescript
function normalizeSearchText(value: string): string {
  return value.toLowerCase().trim();
}
```

---

## 4. Performance Patterns

### Embedding Cache (embedder.ts)

**LRU Cache with TTL:**
```typescript
class EmbeddingCache {
  private cache = new Map<string, CacheEntry>();
  private readonly maxSize: number;  // 256 entries
  private readonly ttlMs: number;    // 30 minutes

  private key(text: string, task?: string): string {
    const hash = createHash("sha256")
      .update(`${task || ""}:${text}`)
      .digest("hex").slice(0, 24);
    return hash;
  }
}
```

**Cache Statistics:**
- Tracks hits/misses for monitoring
- LRU eviction (deletes oldest entry when full)
- Per-task cache keys for query vs passage embeddings

### Multi-Key API Rotation (embedder.ts)

**Round-Robin with Failover:**
```typescript
private async embedWithRetry(payload: any, signal?: AbortSignal): Promise<any> {
  const maxAttempts = this.clients.length;
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const client = this.nextClient();
    try {
      return await client.embeddings.create(payload, signal ? { signal } : undefined);
    } catch (error) {
      if (this.isRateLimitError(error) && attempt < maxAttempts - 1) {
        continue; // Rotate to next key
      }
      throw error;
    }
  }
}
```

### Pagination & Limits (retriever.ts, store.ts)

**Safe Limit Clamping:**
```typescript
function clampInt(value: number, min: number, max: number): number {
  if (!Number.isFinite(value)) return min;
  return Math.min(max, Math.max(min, Math.floor(value)));
}

// Usage: const safeLimit = clampInt(limit, 1, 20);
```

**Query Patterns:**
- Vector search: `limit` parameter for top-k results
- BM25 search: configurable limit with scope filtering
- Candidate pool: `candidatePoolSize` for reranking (default: 20)
- Final results: `deduplicated.slice(0, limit)`

### Hybrid Retrieval (retriever.ts)

**RRF Fusion (Reciprocal Rank Fusion):**
```typescript
// Config weights
vectorWeight: 0.7,   // 70% vector search
bm25Weight: 0.3,     // 30% BM25 full-text
```

**Reranking Pipeline:**
1. Fetch `candidatePoolSize` candidates from vector + BM25
2. Cross-encoder reranking via Jina AI (or other providers)
3. Apply recency boost, importance weighting, length normalization
4. Hard cutoff at `hardMinScore` (default: 0.35)

### Document Chunking (embedder.ts)

**Smart Chunking for Long Documents:**
```typescript
const MAX_EMBED_DEPTH = 3;
const STRICT_REDUCTION_FACTOR = 0.5; // 50% reduction per retry

// On context length error:
const chunkResult = smartChunk(text, this._model);
const chunkEmbeddings = await Promise.all(
  chunkResult.chunks.map(async (chunk) => this.embedSingle(chunk, task, depth + 1, signal))
);
// Average embeddings across chunks
```

**Recursion Depth Protection:**
- Max 3 levels of chunking retries
- Force truncation to 50% of previous size if chunking fails to reduce

### Cross-Process Safety (store.ts)

**File Locking:**
```typescript
import properLockfile from "proper-lockfile";

// Used for LanceDB write operations to prevent concurrent corruption
```

### Async Streaming (llm-oauth.ts)

**SSE Response Handling:**
```typescript
export function extractOutputTextFromSse(bodyText: string): string | null {
  const chunks = bodyText.split(/\r?\n\r?\n/);
  // Parses SSE events: data:..., event:...
  // Handles delta updates and final output_text.done
}
```

---

## 5. Potential Security Concerns

| Issue | Severity | Location | Notes |
|-------|----------|----------|-------|
| OAuth client ID hardcoded | Low | llm-oauth.ts:48 | Can be overridden via env var |
| No .env in .gitignore | Low | .gitignore | No .env files in repo currently |
| Database path not encrypted | Medium | store.ts | Local filesystem storage |
| OAuth tokens stored as JSON | Medium | llm-oauth.ts:462 | File permissions 0o600 mitigate |
| No rate limiting on local OAuth callback | Low | llm-oauth.ts | localhost-only, short timeout |

---

## 6. Performance Optimization Techniques

| Technique | File | Details |
|-----------|------|---------|
| LRU Embedding Cache | embedder.ts | 256 entries, 30min TTL |
| Multi-key Rotation | embedder.ts | Round-robin with rate-limit failover |
| Smart Document Chunking | embedder.ts | Auto-split long docs, average embeddings |
| Hybrid Retrieval | retriever.ts | Vector + BM25 + RRF fusion |
| Cross-encoder Reranking | retriever.ts | Jina/voyage/pinecone/tei providers |
| Recency Boost | retriever.ts | Half-life decay for newer memories |
| Time Decay | retriever.ts | Multiplicative penalty for old entries |
| Noise Filtering | retriever.ts | Filter low-value memories |
| Session Compression | session-compressor.ts | Compress conversation history |
| Memory Compaction | memory-compactor.ts | Cluster and compact old memories |

---

## Summary

**Security Posture:** The project demonstrates solid security practices:
- OAuth 2.0 with PKCE for authentication
- Scope-based memory isolation
- SQL injection prevention via escaping
- Prompt injection detection in reflection content
- Secure file permissions for sensitive data

**Performance Posture:** The project uses multiple optimization layers:
- Caching at embedding layer
- Hybrid retrieval combining vector and keyword search
- Intelligent pagination and limits
- Document chunking for large inputs
- Cross-process file locking for safety

**Trade-offs:** The architecture prioritizes functionality and safety over absolute performance, with sensible defaults that can be tuned via configuration.
