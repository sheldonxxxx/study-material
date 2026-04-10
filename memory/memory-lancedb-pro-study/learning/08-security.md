# Security Assessment: memory-lancedb-pro

## Secrets Management

### Approach: Environment Variable + Config File

The project uses a layered approach to secrets management:

**Environment Variable Substitution:**
- Config values support `${VAR}` syntax for environment variable interpolation
- `resolveEnvVars()` validates that referenced env vars exist at runtime
- Missing env vars throw explicit errors rather than silent failures

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
- Embedding API keys via `embedding.apiKey` config (supports single key or array for round-robin)
- LLM API keys via `llm.apiKey` or `embedding.apiKey` fallback
- Azure OpenAI uses `api-key` header instead of `Authorization: Bearer`
- Keys resolved at construction time via `resolveEnvVars()`

**OAuth Secrets:**
- OAuth client ID overridable via `MEMORY_PRO_OAUTH_CLIENT_ID`
- OAuth authorize/token URLs overridable via env vars
- OAuth redirect URI overridable via `MEMORY_PRO_OAUTH_REDIRECT_URI`
- **Secure file permissions**: OAuth session file saved with mode `0o600` (user read/write only)

### `.gitignore` Assessment

- Includes `.memory-lancedb-pro/` (local database directory)
- Includes `memory-plugin-*` dev directories
- Does NOT explicitly exclude `.env` files (no `.env` files exist in repo)
- Missing: `*.local.*` pattern for local override configs

---

## Authentication Implementation

### OAuth 2.0 with PKCE

**Security Measures:**

| Feature | Implementation |
|---------|---------------|
| PKCE Challenge | SHA256 hash of verifier, base64url encoded |
| State Parameter | 16-byte random hex for CSRF protection |
| Token Storage | File with 0o600 permissions |
| Token Refresh | Automatic with 60s expiry skew buffer |
| Timeout Handling | AbortController with configurable timeout |

**PKCE Implementation:**
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

**OAuth Provider:**
- Hardcoded provider: OpenAI Codex (`app_EMoamEEZ73f0CkXaXp7hrann`)
- Supports environment variable overrides for all OAuth endpoints
- Local callback server on `localhost:1455/auth/callback`

### Scope-Based Access Control

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

## Input Validation and Sanitization

### SQL Injection Prevention

**Escaped Values:**
```typescript
function escapeSqlLiteral(value: string): string {
  return value.replace(/'/g, "''");
}
```

Used for memory IDs, scope filters, and category filters in WHERE clauses.

### Reflection Content Sanitization

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

### Regex Safety

```typescript
function escapeRegExp(input: string): string {
  return input.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

### Query Normalization

```typescript
function normalizeSearchText(value: string): string {
  return value.toLowerCase().trim();
}
```

---

## Privacy Controls

### Memory Isolation

- Multi-scope system prevents cross-scope data leakage
- Access validation at store layer
- System bypass IDs protected from unauthorized access

### Data Storage

- Local filesystem storage (`.memory-lancedb-pro/`)
- No cloud sync or external data transmission
- LanceDB tables with scope-based filtering

---

## Security Strengths and Concerns

### Strengths

| Feature | Implementation |
|---------|---------------|
| OAuth 2.0 with PKCE | Full PKCE flow with CSRF protection |
| Scope-based isolation | 7 scope types with access validation |
| SQL injection prevention | Escaped literals in queries |
| Prompt injection detection | Regex patterns for dangerous reflection content |
| Secure file permissions | OAuth tokens stored with 0o600 |
| Environment variable validation | Explicit error on missing vars |
| Input sanitization | Normalization and escaping throughout |

### Concerns

| Issue | Severity | Notes |
|-------|----------|-------|
| OAuth client ID hardcoded | Low | Can be overridden via env var |
| No .env in .gitignore | Low | No .env files in repo currently |
| Database path not encrypted | Medium | Local filesystem storage |
| OAuth tokens stored as JSON | Medium | File permissions 0o600 mitigate |

### Overall Security Posture

**Strong foundation** with:
- Industry-standard OAuth 2.0 + PKCE implementation
- Defense-in-depth with multiple sanitization layers
- Scope-based access control for multi-tenant scenarios
- Prompt injection detection for reflection content

**Areas for improvement:**
- Add `tsconfig.json` strict mode for compile-time safety
- Consider encryption at rest for sensitive deployments
- Add rate limiting on OAuth callback endpoint
- Add `.env` and `*.local.*` to `.gitignore`

---

## Performance Patterns (Security Adjacent)

### Cross-Process Safety

**File Locking:**
```typescript
import properLockfile from "proper-lockfile";

// Used for LanceDB write operations to prevent concurrent corruption
```

### Embedding Cache Security

- LRU cache with TTL prevents memory exhaustion
- SHA256 hash of input for cache keys (no plaintext storage)
- Per-task cache keys prevent cross-contamination

---

## Summary

**Security Posture**: Solid security practices with OAuth 2.0 + PKCE, scope-based isolation, SQL injection prevention, and prompt injection detection. Local storage model limits exposure. File permissions appropriately restricted.

**Trade-offs**: Architecture prioritizes functionality and local control over cloud-native security features. Suitable for single-user and trusted multi-user environments.
