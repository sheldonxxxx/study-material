# Security and Performance Patterns: gstack

## Overview

**gstack** is a TypeScript-based CLI tool and skill framework for Claude Code. The codebase demonstrates mature security engineering practices with particular emphasis on cloud metadata endpoint protection, shell injection prevention, and efficient resource management. This analysis covers secrets management, authentication/authorization, input validation, and performance optimization patterns.

---

## Security Patterns

### Secrets Management

**Environment Variables (No Vault)**

| Aspect | Pattern | Implementation |
|--------|---------|----------------|
| API Keys | `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` | Via `.env` file (local) or GitHub Actions secrets (CI) |
| Key Access | Direct `process.env.VAR` | Read at runtime, not hardcoded |
| CI Injection | `${{ secrets.VAR }}` | Passed as environment variables to test jobs |
| Local Dev | `.env.example` template | Documents required keys, not committed |

**Key Files:**
- `.env.example` - Minimal template (only `ANTHROPIC_API_KEY` documented)
- `.github/workflows/evals.yml` - Secrets injected via `secrets.` context
- `test/helpers/llm-judge.ts` - Requires `ANTHROPIC_API_KEY` env var

**Pattern: Secrets never logged or displayed**
```typescript
// setup-deploy/SKILL.md:339
echo $RENDER_API_KEY | head -c 4  // Partial reveal only
```

### Authentication/Authorization

**No Traditional Auth System**

gstack is a CLI tool, not a web application with user authentication. Auth patterns observed:

| Context | Auth Method | Notes |
|---------|-------------|-------|
| Anthropic API | `ANTHROPIC_API_KEY` | Bearer token in HTTP headers |
| OpenAI Codex | `~/.codex/` config | Uses Codex's own auth, no `OPENAI_API_KEY` |
| Google Gemini | `GEMINI_API_KEY` or `~/.gemini/` | Config file or env var |
| GitHub Actions | `secrets.GITHUB_TOKEN` | Automatically available |
| Browser Cookies | Chromium's cookie database | Read-only access to decrypt |

**Supabase (Telemetry):**
- Row-Level Security (RLS) policies on all telemetry tables
- Anon key denied direct access
- All reads/writes through validated edge functions with schema checks
- Event type allowlists and field length limits

**Key File:** `supabase/migrations/002_tighten_rls.sql`

### Input Validation and Sanitization

**1. URL Validation (Cloud Metadata Protection)**

The browse CLI implements thorough URL validation to prevent SSRF attacks against cloud metadata endpoints:

```typescript
// browse/src/url-validation.ts
const BLOCKED_METADATA_HOSTS = new Set([
  '169.254.169.254',  // AWS/GCP/Azure instance metadata
  'fd00::',           // IPv6 unique local
  'metadata.google.internal', // GCP metadata
  'metadata.azure.internal',  // Azure IMDS
]);
```

**Key Protections:**
- Protocol allowlist: Only `http:` and `https:` permitted
- DNS rebinding protection: Async DNS resolution to check resolved IPs
- Loopback/private IP bypass: Skips DNS check for localhost/127.0.0.1/::1 and private networks (10.x, 172.16-31.x, 192.168.x) to avoid latency in E2E tests
- Hex/decimal IP normalization: Catches obfuscated metadata IP representations

**2. Shell Injection Prevention**

**gstack-slug hardened output:**
```bash
# bin/gstack-slug:6-13
# Strip any characters that aren't alphanumeric, dot, hyphen, or underscore
SLUG=$(printf '%s' "$RAW_SLUG" | tr -cd 'a-zA-Z0-9._-')
BRANCH=$(printf '%s' "$RAW_BRANCH" | tr -cd 'a-zA-Z0-9._-')
```

**Safe subprocess execution:**
```typescript
// bin/gstack-repo-mode:24
# Validate: only allow known values (prevent shell injection via source <(...))
```

**Prompt-to-temp-file pattern:**
```typescript
// scripts/gen-skill-docs.ts:2184
// scripts/resolvers/review.ts:279
2. **Write the assembled prompt to a temp file** (prevents shell injection from user-derived content):
```

**3. SQL Injection Prevention**

**Parameterized queries in cookie decryption:**
```typescript
// browse/src/cookie-import-browser.ts:254-256
// Parameterized query — no SQL injection
const rows = db.query(
  `SELECT name, value, host_key, encrypted_value FROM cookies WHERE ${whereClause}`,
  params  // Parameters passed separately
);
```

**4. CSO Skill (OWASP Top 10)**

The `/cso` skill provides systematic security auditing:
- **A01: Broken Access Control** - Authorization bypass patterns
- **A02: Cryptographic Failures** - Hardcoded secrets, weak crypto
- **A03: Injection** - SQL, command, template, LLM prompt injection
- **A04: Insecure Design** - Trust boundary violations
- **A05: Security Misconfiguration** - Default credentials, error handling
- **A06: Vulnerable Components** - Outdated dependencies
- **A07: Auth Failures** - Session management, credential handling
- **A08: Data Integrity** - Supply chain, CI/CD integrity
- **A09: Logging Failures** - Missing audit trails
- **A10: SSRF** - Server-side request forgery patterns

**Special CSO focus areas:**
- Prompt injection: User input flowing into system prompts or tool schemas
- GitHub Actions security: `pull_request_target`, script injection via `${{ github.event.* }}`
- Credential scanning: Database connection strings in config files

### Security Review Workflow

**The `/review` skill implements security-first code review:**
```bash
# review/SKILL.md:517
1. **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Enum & Value Completeness
```

---

## Performance Patterns

### Caching

**1. CircularBuffer for Console/Network/Dialog Logs**

Fixed-capacity ring buffer with O(1) insertions:

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
      // Buffer full — advance head (overwrites oldest)
      this.head = (this.head + 1) % this.capacity;
    }
    this._totalAdded++;
  }
}
```

**Benefits:**
- Memory-bounded (50,000 entries max per buffer type)
- O(1) insertion (no array shifting)
- Eviction of oldest entries automatically
- Three separate buffers: console, network, dialog

**2. Pre-installed Dependencies in CI**

```yaml
# .github/workflows/evals.yml:113-120
# Restore pre-installed node_modules from Docker image via symlink (~0s vs ~15s install)
- name: Restore deps
  run: |
    if [ -d /opt/node_modules_cache ] && diff -q /opt/node_modules_cache/.package.json package.json >/dev/null 2>&1; then
      ln -s /opt/node_modules_cache node_modules
    else
      bun install
    fi
```

**3. Playwright Browser Caching**

```yaml
# .github/workflows/evals.yml:138
PLAYWRIGHT_BROWSERS_PATH: /opt/playwright-browsers
```

Browsers pre-installed in Docker image, not downloaded at runtime.

**4. Docker Layer Caching**

```yaml
# .github/workflows/evals.yml:25-56
- id: meta
  run: echo "tag=${{ env.IMAGE }}:${{ hashFiles('.github/docker/Dockerfile.ci', 'package.json') }}" >> "$GITHUB_OUTPUT"
# Image only rebuilt when Dockerfile or package.json changes
```

### Lazy Loading and Pagination

**1. Git History Digest (On-Demand Loading)**

```bash
# Project specifies to use history-digest for context selection
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" history-digest
```

Generates lightweight index first, then loads full summaries only for relevant phases.

**2. Diff-Based Test Selection**

```typescript
// test/helpers/e2e-helpers.ts:28-36
export const evalsEnabled = !!process.env.EVALS;
if (evalsEnabled && !process.env.EVALS_ALL) {
  const baseBranch = process.env.EVALS_BASE
  // Only runs tests affected by the current diff
}
```

**Test tier classification:**
- `gate` tier: Runs on every PR (deterministic, safety guardrails)
- `periodic` tier: Runs weekly or manually (quality benchmarks, non-deterministic)

**3. Lazy Evaluation of LLM Judges**

```typescript
// test/helpers/llm-judge.ts:32-56
// Only calls LLM when EVALS=1 is set
export async function callJudge<T>(prompt: string): Promise<T> {
  const client = new Anthropic();
  // ... only instantiated when needed
}
```

### Query Optimization

**1. Git Operations with `--show-toplevel` Fallback**

```typescript
// browse/src/config.ts:28-40
export function getGitRoot(): string | null {
  try {
    const proc = Bun.spawnSync(['git', 'rev-parse', '--show-toplevel'], {
      stdout: 'pipe',
      stderr: 'pipe',
      timeout: 2_000, // Don't hang if .git is broken
    });
    if (proc.exitCode !== 0) return null;
    return proc.stdout.toString().trim() || null;
  } catch {
    return null;
  }
}
```

**2. Read-Only Browser Cookie Access**

```typescript
// ARCHITECTURE.md:102
3. **Database is read-only.** gstack copies the Chromium cookie DB to a temp file (to avoid SQLite lock conflicts with the running browser) and opens it read-only.
```

**3. Browser State Save/Restore with Skipped Failures**

```typescript
// browse/src/browser-manager.ts:339-351
// Individual page failures are swallowed — partial restore is better than none
try {
  storage = await page.evaluate(() => ({
    localStorage: { ...localStorage },
    sessionStorage: { ...sessionStorage },
  }));
} catch {}
```

### Concurrency

**1. E2E Test Parallelization**

```yaml
# .github/workflows/evals.yml:139
EVALS_CONCURRENCY: "40"
run: EVALS=1 bun test --retry 2 --concurrent --max-concurrency 40 ${{ matrix.suite.file }}
```

**2. Matrix Strategy for Test Suites**

```yaml
# .github/workflows/evals.yml:68-96
strategy:
  fail-fast: false
  matrix:
    suite:
      - name: llm-judge
      - name: e2e-browse
        runner: ubicloud-standard-8  # Heavy tests get bigger runner
      - name: e2e-plan
      # ... 12 parallel jobs
```

**3. GitHub Actions Concurrency Control**

```yaml
# .github/workflows/evals.yml:7-9
concurrency:
  group: evals-${{ github.head_ref }}
  cancel-in-progress: true
```

### Timeout Management

**1. Browser Launch Timeouts**

```typescript
// browse/src/cli.ts:18
const MAX_START_WAIT = IS_WINDOWS ? 15000 : (process.env.CI ? 30000 : 8000);

// browse/src/browser-manager.ts:143
page.evaluate('1'),
new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 2000)),
```

**2. Graceful Browser Close with Timeout**

```typescript
// browse/src/browser-manager.ts:126-130
await Promise.race([
  this.browser.close(),
  new Promise(resolve => setTimeout(resolve, 5000)),
]).catch(() => {});
```

**3. DNS Resolution Timeout**

```typescript
// browse/src/url-validation.ts:51-61
async function resolvesToBlockedIp(hostname: string): Promise<boolean> {
  try {
    const dns = await import('node:dns');
    const { resolve4 } = dns.promises;
    const addresses = await resolve4(hostname);  // OS-level timeout
    return addresses.some(addr => BLOCKED_METADATA_HOSTS.has(addr));
  } catch {
    return false;  // DNS resolution failed — not a rebinding risk
  }
}
```

---

## Summary Table

| Category | Pattern | Impact |
|----------|---------|--------|
| **Secrets** | Env vars via `.env`/secrets | No hardcoded credentials |
| **SSRF Protection** | Cloud metadata blocklist + DNS rebinding check | Prevents credential theft |
| **Shell Injection** | Output sanitization (alphanumeric only) | Safe `eval`/`source` consumption |
| **SQL Injection** | Parameterized queries | Safe database access |
| **Auth** | API key + Supabase RLS | Defense in depth |
| **Memory** | CircularBuffer (50K cap) | Bounded memory usage |
| **CI Cache** | Pre-installed node_modules + Playwright | ~15s saved per run |
| **Test Select** | Diff-based + tiers | Only runs affected tests |
| **Concurrency** | 40 parallel E2E jobs + matrix | Fastest CI feedback |
| **Timeouts** | Platform-aware defaults | Prevents hangs |

---

## Security Posture Assessment

**Strengths:**
- Comprehensive input validation (URLs, shell args, SQL)
- Defense in depth (multiple security layers for cloud metadata)
- Security review is a first-class workflow (`/review`, `/cso`)
- Secrets never logged or exposed
- Read-only database access pattern
- Well-tested security-critical paths (SQL injection E2E test)

**Considerations:**
- No dependency scanning for known vulnerabilities (mentioned in CSO skill but not automated)
- API keys stored as env vars (acceptable for CLI tool, vault would be overkill)
- LLM prompt injection detection relies on skill implementation, not runtime protection

---

## Key Files Referenced

| File | Security/Performance Pattern |
|------|----------------------------|
| `browse/src/url-validation.ts` | SSRF protection, DNS rebinding |
| `browse/src/buffers.ts` | CircularBuffer for bounded memory |
| `browse/src/cookie-import-browser.ts` | Parameterized SQL, read-only DB |
| `browse/src/browser-manager.ts` | Graceful shutdown, state save/restore |
| `browse/src/config.ts` | Git root fallback, state directory |
| `bin/gstack-slug` | Shell injection hardening |
| `.github/workflows/evals.yml` | CI caching, concurrency |
| `supabase/migrations/002_tighten_rls.sql` | RLS policies |
| `test/helpers/llm-judge.ts` | Lazy LLM evaluation |
| `cso/SKILL.md` | OWASP Top 10 security auditing |
| `review/SKILL.md` | Security-first code review |
