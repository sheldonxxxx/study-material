# My Action Items: Building a gstack-Inspired Project

**Based on:** 12-lessons-learned.md
**Context:** Applying gstack patterns to a similar AI-augmented development toolkit

---

## Phase 1: Foundation (Do First)

### 1.1 Implement Daemon + State File Pattern

**Why:** Latency is non-negotiable for AI tool calling. Cold-start browser launches make sub-second tool calls impossible.

**Action:** For any tool involving browser/process automation:
1. Create state file (`~/.mytool/state.json`) with `{pid, port, token}`
2. Implement health check endpoint (`GET /health`)
3. CLI checks health on every call; auto-restarts if dead
4. Server auto-detects binary version mismatch and restarts

**Verification:** `time mytool command` shows <200ms for subsequent calls after first-startup.

---

### 1.2 Add tsconfig.json with Strict Mode

**Why:** gstack has no visible tsconfig.json -- TypeScript strictness is unknown. Adding strict mode from the start prevents technical debt.

**Action:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "esModuleInterop": true
  }
}
```

**Verification:** `bun build` with no errors; `bun run tsc --noEmit` clean.

---

### 1.3 Set Up ESLint + Prettier

**Why:** gstack has no linting -- code style is convention-based. Enforcing style prevents PR debates.

**Action:**
```bash
bun add -d eslint prettier eslint-config-prettier @typescript-eslint/eslint-plugin
```

**Verification:** `bun lint` and `bun fmt` pass in CI; pre-commit hook runs format check.

---

## Phase 2: Skill Architecture

### 2.1 Design Skill Boundaries as Vertical Slices

**Why:** gstack's 28 skills are self-contained units organized by user job-to-be-done, not by technical concern.

**Action:** For each skill, define:
1. The single command that invokes it (`/plan`, `/review`, `/ship`)
2. The artifacts it produces (where, what format)
3. The artifacts it consumes from other skills
4. The single most important output

**Verification:** Each skill directory is self-contained with no cross-skill imports.

---

### 2.2 Implement Template-Driven Generation for Skill Docs

**Why:** gstack's SKILL.md files are generated from .tmpl sources. This allows shared infrastructure (preambles, command references) without copy-paste.

**Action:**
1. Create `scripts/gen-skill-docs.ts`
2. Define placeholder system: `{{PREAMBLE}}`, `{{COMMAND_REFERENCE}}`, etc.
3. Create resolver modules for each placeholder
4. Each skill has `SKILL.md.tmpl` as source, `SKILL.md` generated

**Verification:** `bun run gen:skill-docs --dry-run && git diff --exit-code` passes in CI.

---

### 2.3 Build Diff-Aware Mode into Every Analysis Skill

**Why:** gstack skills auto-detect feature branches and scope to changed files. This reduces user friction to zero.

**Action:** For review/analysis skills:
1. On invocation, check `git diff --name-only` against base branch
2. Map changed files to affected resources (pages, endpoints, components)
3. Scope analysis to affected resources; don't scan entire project
4. Fall back to full analysis if no diff detectable

**Verification:** Invoke skill on feature branch with known changed file; confirm only that file's area is analyzed.

---

## Phase 3: Quality Assurance

### 3.1 Implement Two-Tier Testing (Gate + Periodic)

**Why:** LLM-judged evals are expensive. Gate tests run on every PR; periodic tests run weekly.

**Action:**
1. Create `test/gate/` -- fast, deterministic, no LLM judgment
2. Create `test/periodic/` -- LLM-judged quality tests
3. Gate tests block merge; periodic tests run on cron
4. Implement diff-based test selection: only run tests affected by PR

**Verification:** PR with unrelated file change only runs affected tests; gate tests block merge on failure.

---

### 3.2 Add Comprehensive Error Messages for AI Agents

**Why:** gstack errors tell the AI what to do next. Generic errors ("Element not found") leave the AI stuck.

**Action:** For every error type, implement:
```
[what failed] -- [specific action to take]
```

Example:
```typescript
throw new Error(`Ref ${ref} is stale — run 'snapshot' to get fresh refs`);
```

**Verification:** Invoke skill with intentionally stale state; observe error message guides correct recovery action.

---

### 3.3 Add Self-Regulation Heuristics

**Why:** Without explicit limits, AI agents will over-work and cause damage.

**Action:** Implement quantitative self-regulation for fix loops:
```typescript
let wtfLikelihood = 0;
let fixCount = 0;

function onRevert() { wtfLikelihood += 15; }
function onMultiFileFix() { wtfLikelihood += 5; }
function onEveryNthFix(n: number) { wtfLikelihood += 1 * Math.floor(fixCount / n); }

const MAX_FIXES = 50;
const MAX_WTF = 80;

if (fixCount >= MAX_FIXES || wtfLikelihood >= MAX_WTF) {
  throw new Error(`Self-regulation triggered: ${fixCount} fixes, ${wtfLikelihood}% WTF-likelihood. Stopping.`);
}
```

**Verification:** Simulate a session that triggers the threshold; confirm it stops appropriately.

---

## Phase 4: Security

### 4.1 Implement URL Validation with SSRF Protection

**Why:** Browser automation tools are SSRF targets. Cloud metadata endpoints (169.254.169.254) give access to credentials.

**Action:**
```typescript
const BLOCKED_HOSTS = new Set([
  '169.254.169.254',
  'metadata.google.internal',
  'metadata.azure.internal',
]);

async function validateUrl(url: string): Promise<void> {
  // Protocol allowlist
  const parsed = new URL(url);
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new Error(`Only http/https allowed, got ${parsed.protocol}`);
  }

  // Cloud metadata blocklist with DNS rebinding check
  const dns = await import('node:dns');
  try {
    const addresses = await dns.promises.resolve4(parsed.hostname);
    if (addresses.some(addr => BLOCKED_HOSTS.has(addr))) {
      throw new Error(`Blocked metadata endpoint: ${parsed.hostname}`);
    }
  } catch {
    // DNS resolution failed -- not a rebinding risk
  }
}
```

**Verification:** Attempt navigation to 169.254.169.254; confirm blocked with clear error.

---

### 4.2 Implement Secrets Redaction

**Why:** AI agents see tool output. Credentials in logs are exposed.

**Action:** Redact sensitive values in all output:
```typescript
const PATTERNS = [
  { regex: /ghp_[a-zA-Z0-9]{36}/, replacement: 'ghp_[REDACTED]' },
  { regex: /sk-[a-zA-Z0-9]{48}/, replacement: 'sk-[REDACTED]' },
  { regex: /eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+/, replacement: '[JWT_REDACTED]' },
  { regex: /"api_key"\s*:\s*"[^"]+"/gi, replacement: '"api_key": "[REDACTED]"' },
];
```

**Verification:** Run command that produces credential-like values; confirm redaction in output.

---

### 4.3 Parameterize All Database Queries

**Why:** Cookie import reads from Chromium's SQLite database. Unparameterized queries are SQL injection vectors.

**Action:**
```typescript
// WRONG
const rows = db.query(`SELECT * FROM cookies WHERE host_key = '${hostKey}'`);

// RIGHT
const stmt = db.prepare('SELECT * FROM cookies WHERE host_key = ?');
const rows = stmt.all(hostKey);
```

**Verification:** Run skill with malicious cookie value containing SQL; confirm no injection.

---

## Phase 5: Operational Excellence

### 5.1 Implement Bounded Logging with CircularBuffer

**Why:** Unbounded logs can fill disk. gstack's CircularBuffer caps at 50,000 entries with O(1) push.

**Action:**
```typescript
export class CircularBuffer<T> {
  private buffer: (T | undefined)[];
  private head = 0;
  private size = 0;
  private totalAdded = 0;

  constructor(private capacity: number) {
    this.buffer = new Array(capacity);
  }

  push(item: T): void {
    const index = (this.head + this.size) % this.capacity;
    this.buffer[index] = item;
    if (this.size < this.capacity) {
      this.size++;
    } else {
      this.head = (this.head + 1) % this.capacity;
    }
    this.totalAdded++;
  }

  last(n: number): T[] { /* return most recent n */ }
  toArray(): T[] { /* in insertion order */ }
}
```

**Verification:** Push 60,000 items to 50,000-capacity buffer; confirm oldest 10,000 are evicted.

---

### 5.2 Implement Health Score Computation

**Why:** gstack's QA skill computes a 0-100 health score with weighted categories. This enables tracking over time.

**Action:** Define rubric:
```typescript
const CATEGORY_WEIGHTS = {
  console: 0.15,
  functional: 0.20,
  ux: 0.15,
  accessibility: 0.15,
  visual: 0.10,
  links: 0.10,
  performance: 0.10,
  content: 0.05,
};

const SEVERITY_DEDUCTIONS = {
  critical: 25,
  high: 15,
  medium: 8,
  low: 3,
};

function computeHealthScore(issues: Issue[]): number {
  // Per-category scoring with deductions
  // Weighted average across categories
  // Return 0-100
}
```

**Verification:** Run QA on known-buggy app; confirm score reflects bug severity accurately.

---

### 5.3 Add Pre-Baked CI Image with Cached Dependencies

**Why:** gstack's Docker image pre-installs node_modules and Playwright browsers. Dependency install takes ~0s via symlink instead of ~15s via `bun install`.

**Action:** In Dockerfile:
```dockerfile
FROM ubuntu:24.04
RUN apt-get install -y bun nodejs
COPY package.json ./
RUN bun install --frozen-lockfile && \
    mv node_modules /opt/node_modules_cache
```

In CI workflow:
```yaml
- name: Restore deps
  run: |
    if [ -d /opt/node_modules_cache ] && \
       diff -q /opt/node_modules_cache/.package-lock package-lock.json >/dev/null 2>&1; then
      ln -s /opt/node_modules_cache node_modules
    else bun install; fi
```

**Verification:** CI run with cache hit shows ~0s for dependency restore step.

---

## Phase 6: Documentation and Governance (Do Last)

### 6.1 Add CONTRIBUTING.md

**Why:** gstack has no CONTRIBUTING guide despite 49K stars. External contributors have no guidance.

**Action:** Create CONTRIBUTING.md with:
- Development setup (bun, Node 22)
- How to run tests locally
- Skill development workflow (edit .tmpl, run `bun run gen:skill-docs`)
- PR standards (bisect commits, CHANGELOG discipline)
- Code review standards

**Verification:** CONTRIBUTING.md exists and accurately reflects the development workflow.

---

### 6.2 Add CODEOWNERS File

**Why:** Single-maintainer projects have bus factor = 1. CODEOWNERS enables review assignment.

**Action:**
```
# Default owners for everything
* @maintainer

# Skill-specific owners
skills/* @maintainer
browse/* @maintainer
```

**Verification:** New PR auto-assigns maintainer based on CODEOWNERS.

---

### 6.3 Set Up GitHub Releases with Changelog

**Why:** gstack uses tags only; no formal release notes. Users miss important changes.

**Action:**
1. Enable GitHub Releases in repository settings
2. On version bump, generate release notes from CHANGELOG entries since last tag
3. Use `gh release create` in release workflow

**Verification:** Creating a tag triggers GitHub release with auto-generated notes.

---

## Priority Order Summary

| Priority | Action Item | Phase | Impact |
|----------|-------------|-------|--------|
| 1 | Daemon + state file pattern | Foundation | Enables viable AI tool calling |
| 2 | URL validation + SSRF protection | Security | Prevents credential theft |
| 3 | Two-tier testing | QA | Balances cost and coverage |
| 4 | Error messages for AI agents | QA | Enables self-recovery |
| 5 | Template-driven SKILL.md | Skill Architecture | Enables code sharing |
| 6 | Diff-aware mode | Skill Architecture | Reduces user friction |
| 7 | Self-regulation heuristics | QA | Prevents runaway AI |
| 8 | Secrets redaction | Security | Prevents credential exposure |
| 9 | tsconfig.json strict mode | Foundation | Prevents TypeScript debt |
| 10 | CircularBuffer logging | Operations | Bounded memory |
| 11 | ESLint + Prettier | Foundation | Enforced code style |
| 12 | Health score computation | Operations | Trackable quality |
| 13 | Pre-baked CI image | Operations | Fast CI feedback |
| 14 | CONTRIBUTING.md | Governance | Enables external contributions |
| 15 | GitHub Releases | Governance | User-facing release notes |
