# Security: gstack

## Overview

gstack is a CLI tool with mature security engineering practices. Key strengths include cloud metadata endpoint protection, shell injection prevention, parameterized queries, and comprehensive secrets redaction. No traditional auth system exists (appropriate for a CLI tool).

---

## Secrets Management

### Environment Variables (No Vault)

| Aspect | Pattern | Implementation |
|--------|---------|----------------|
| API Keys | `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` | Via `.env` file or GitHub Actions secrets |
| Key Access | Direct `process.env.VAR` | Read at runtime, not hardcoded |
| CI Injection | `${{ secrets.VAR }}` | Passed as environment variables to test jobs |
| Local Dev | `.env.example` template | Documents required keys, not committed |

### Key Files

- `.env.example` — Minimal template (only `ANTHROPIC_API_KEY` documented)
- `.github/workflows/evals.yml` — Secrets injected via `secrets.` context
- `test/helpers/llm-judge.ts` — Requires `ANTHROPIC_API_KEY` env var

### Pattern: Partial Reveal for Confirmation

```bash
# setup-deploy/SKILL.md:339
echo $RENDER_API_KEY | head -c 4  // Partial reveal only
```

---

## Authentication/Authorization

### No Traditional Auth System

gstack is a CLI tool, not a web application. Auth patterns:

| Context | Auth Method | Notes |
|---------|-------------|-------|
| Anthropic API | `ANTHROPIC_API_KEY` | Bearer token in HTTP headers |
| OpenAI Codex | `~/.codex/` config | Uses Codex's own auth |
| Google Gemini | `GEMINI_API_KEY` or `~/.gemini/` | Config file or env var |
| GitHub Actions | `secrets.GITHUB_TOKEN` | Automatically available |
| Browser Cookies | Chromium's cookie database | Read-only access |

### Supabase (Telemetry)

- Row-Level Security (RLS) policies on all telemetry tables
- Anon key denied direct access
- All reads/writes through validated edge functions
- Event type allowlists and field length limits

**Key File**: `supabase/migrations/002_tighten_rls.sql`

---

## Input Validation

### URL Validation (Cloud Metadata Protection)

Prevents SSRF attacks against cloud metadata endpoints:

```typescript
// browse/src/url-validation.ts
const BLOCKED_METADATA_HOSTS = new Set([
  '169.254.169.254',  // AWS/GCP/Azure instance metadata
  'fd00::',           // IPv6 unique local
  'metadata.google.internal', // GCP metadata
  'metadata.azure.internal',  // Azure IMDS
]);
```

**Key Protections**:
- Protocol allowlist: Only `http:` and `https:` permitted
- DNS rebinding protection: Async DNS resolution to check resolved IPs
- Loopback/private IP bypass: Skips DNS check for localhost/127.0.0.1/::1 and private networks (10.x, 172.16-31.x, 192.168.x)
- Hex/decimal IP normalization: Catches obfuscated metadata IP representations

### Shell Injection Prevention

**Hardened output** (`bin/gstack-slug:6-13`):
```bash
# Strip any characters that aren't alphanumeric, dot, hyphen, or underscore
SLUG=$(printf '%s' "$RAW_SLUG" | tr -cd 'a-zA-Z0-9._-')
BRANCH=$(printf '%s' "$RAW_BRANCH" | tr -cd 'a-zA-Z0-9._-')
```

**Safe subprocess execution** (`bin/gstack-repo-mode:24`):
```bash
# Validate: only allow known values (prevent shell injection via source <(...))
```

**Prompt-to-temp-file pattern** (prevents shell injection from user-derived content):
```typescript
// scripts/gen-skill-docs.ts:2184
// scripts/resolvers/review.ts:279
2. **Write the assembled prompt to a temp file** (prevents shell injection from user-derived content):
```

### SQL Injection Prevention

**Parameterized queries** (`browse/src/cookie-import-browser.ts:254-256`):
```typescript
// Parameterized query — no SQL injection
const rows = db.query(
  `SELECT name, value, host_key, encrypted_value FROM cookies WHERE ${whereClause}`,
  params  // Parameters passed separately
);
```

---

## Security Review Workflows

### CSO Skill (OWASP Top 10)

The `/cso` skill provides systematic security auditing:

| Category | Focus |
|----------|-------|
| A01: Broken Access Control | Authorization bypass patterns |
| A02: Cryptographic Failures | Hardcoded secrets, weak crypto |
| A03: Injection | SQL, command, template, LLM prompt injection |
| A04: Insecure Design | Trust boundary violations |
| A05: Security Misconfiguration | Default credentials, error handling |
| A06: Vulnerable Components | Outdated dependencies |
| A07: Auth Failures | Session management, credential handling |
| A08: Data Integrity | Supply chain, CI/CD integrity |
| A09: Logging Failures | Missing audit trails |
| A10: SSRF | Server-side request forgery patterns |

**Special focus areas**:
- Prompt injection: User input flowing into system prompts
- GitHub Actions security: `pull_request_target`, script injection
- Credential scanning: Database connection strings in config files

### Review Skill

The `/review` skill implements security-first code review:

```bash
# review/SKILL.md:517
1. **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Enum & Value Completeness
```

---

## Security Posture

### Strengths

- Comprehensive input validation (URLs, shell args, SQL)
- Defense in depth (multiple security layers for cloud metadata)
- Security review is a first-class workflow (`/review`, `/cso`)
- Secrets never logged or exposed
- Read-only database access pattern
- Well-tested security-critical paths

### Considerations

- No dependency scanning for known vulnerabilities (mentioned in CSO skill but not automated)
- API keys stored as env vars (acceptable for CLI tool)
- LLM prompt injection detection relies on skill implementation, not runtime protection

---

## Key Security Files

| File | Pattern |
|------|---------|
| `browse/src/url-validation.ts` | SSRF protection, DNS rebinding |
| `browse/src/cookie-import-browser.ts` | Parameterized SQL, read-only DB |
| `bin/gstack-slug` | Shell injection hardening |
| `supabase/migrations/002_tighten_rls.sql` | RLS policies |
| `cso/SKILL.md` | OWASP Top 10 security auditing |
| `review/SKILL.md` | Security-first code review |

---

## Performance Patterns (Security-Related)

### CircularBuffer for Memory Bounds

```typescript
// browse/src/buffers.ts
const HIGH_WATER_MARK = 50_000;

export class CircularBuffer<T> {
  push(entry: T): void {
    const index = (this.head + this._size) % this.capacity;
    this.buffer[index] = entry;
    if (this._size < this.capacity) {
      this._size++;
    } else {
      this.head = (this.head + 1) % this.capacity;
    }
    this._totalAdded++;
  }
}
```

**Benefits**:
- Memory-bounded (50,000 entries max)
- O(1) insertion
- Eviction of oldest entries automatically

### Read-Only Browser Cookie Access

```typescript
// ARCHITECTURE.md:102
3. **Database is read-only.** gstack copies the Chromium cookie DB to a temp file (to avoid SQLite lock conflicts with the running browser) and opens it read-only.
```

### Graceful Browser Close

```typescript
// browse/src/browser-manager.ts:126-130
await Promise.race([
  this.browser.close(),
  new Promise(resolve => setTimeout(resolve, 5000)),
]).catch(() => {});
```

---

## Summary

| Category | Pattern | Impact |
|----------|---------|--------|
| **Secrets** | Env vars via `.env`/secrets | No hardcoded credentials |
| **SSRF Protection** | Cloud metadata blocklist + DNS rebinding check | Prevents credential theft |
| **Shell Injection** | Output sanitization (alphanumeric only) | Safe `eval`/`source` consumption |
| **SQL Injection** | Parameterized queries | Safe database access |
| **Auth** | API key + Supabase RLS | Defense in depth |
| **Memory** | CircularBuffer (50K cap) | Bounded memory usage |
