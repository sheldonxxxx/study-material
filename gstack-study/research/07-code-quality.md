# Code Quality Assessment: gstack

## Overview

**gstack** is a TypeScript-based CLI tool and skill framework. Code quality practices are generally strong, with comprehensive testing, good error handling patterns, and security-conscious design. Notable gaps exist in tooling (no ESLint/Prettier) and TypeScript configuration visibility.

---

## 1. Testing

### Framework & Infrastructure

| Aspect | Finding |
|--------|---------|
| Test runner | Bun test (`bun test`) — fast, built-in |
| E2E framework | Playwright (Chromium) |
| LLM evaluation | Claude -p + LLM judge (paid tier) |
| Test selection | Diff-based (auto-selects tests based on git changes) |
| Test tiers | `gate` (blocks merge) and `periodic` (weekly cron) |

### Test Coverage Analysis

**Unit/Integration Tests** (`browse/test/`, `test/`):
- **config.test.ts**: 317 lines covering path resolution, gitignore handling, version hash, server health checks
- **commands.test.ts**: 1,837 lines of comprehensive command testing across navigation, content extraction, interaction, dialogs, security (redaction, path traversal), workflows
- **skill-validation.test.ts**: Validates all `$B` commands and snapshot flags across skill files

**What tests actually cover**:
- Happy path: Each command has multiple positive test cases
- Error paths: "throws on missing args", "throws on invalid format", "throws on bad URLs"
- Security: Sensitive value redaction (storage, cookies, headers, form passwords), path traversal prevention, blocked metadata endpoints
- Edge cases: Buffer bounds (50k cap), empty pages, SPA rendering, dialog auto-accept/dismiss
- Workflows: Multi-step sequences (navigation -> snapshot -> click -> verify)

**Notable test quality patterns**:
- Tests use real fixtures (HTML files served via test server)
- Temporary file cleanup in `finally` blocks
- Async timeout handling (15-20s for browser operations)
- HTTP health checks rather than PID-based process detection
- Permission error simulation (e.g., read-only .gitignore)

**E2E Eval Tests** (`test/skill-e2e-*.test.ts`):
- Multi-category E2E tests (BWS, CSO, design, plan, QA, review, workflow)
- Routing E2E test
- Codex and Gemini integration tests
- Tier classification in `touchfiles.ts` for gate vs periodic

### Gaps

- No dedicated unit tests for utility functions (e.g., `CircularBuffer`, URL validation logic) in isolation
- Some tests use `require()` instead of ES module imports (e.g., `require('../src/cli')`) — pragmatic but inconsistent
- Skill validation tests skip non-existent skills rather than failing

---

## 2. Type Systems

### TypeScript Usage

- **Runtime**: Bun (has native TypeScript support — no build step needed for .ts files)
- **Type coverage**: Appears comprehensive in core modules
- **Type annotations**: Interfaces defined for all major types (`BrowseConfig`, `ServerState`, `RefEntry`, `BrowserState`, etc.)

### Configuration Visibility Issue

**No `tsconfig.json` found** in the repository. This suggests:
- TypeScript is run directly via Bun's built-in transpiler
- Strict mode may not be enforced
- No explicit compiler options (e.g., `strict`, `noImplicitAny`, `esModuleInterop`)

### Types Observed

```typescript
// From config.ts
export interface BrowseConfig {
  projectDir: string;
  stateDir: string;
  stateFile: string;
  consoleLog: string;
  networkLog: string;
  dialogLog: string;
}

// From browser-manager.ts
export interface RefEntry {
  locator: Locator;
  role: string;
  name: string;
}
```

### Gap

- No visible TypeScript strictness configuration
- Unknown whether `strictNullChecks`, `noImplicitAny`, or `strictFunctionTypes` are enabled

---

## 3. Linting & Formatting

### Tooling Inventory

| Tool | Status |
|------|--------|
| ESLint | **NOT FOUND** |
| Prettier | **NOT FOUND** |
| rustfmt/gofmt | N/A (TypeScript project) |
| actionlint | **Present** (`.github/actionlint.yaml`) |

### Code Style Observations

- 2-space indentation (observed in all .ts files)
- Opening brace on same line (`function foo() {`)
- No enforced formatting standard (relies on editor conventions)
- `import * as fs from 'fs'` style (namespace imports)

### actionlint Configuration

Only CI workflow files are linted via actionlint. No application code linting.

```yaml
# .github/actionlint.yaml
self-hosted-runner:
  labels:
    - ubicloud-standard-2
```

### Gap

- No automated formatting enforcement
- No ESLint rules for TypeScript best practices
- Style consistency is convention-based, not tool-enforced

---

## 4. Error Handling

### Patterns Observed

**Specific error types with context**:
```typescript
// From config.ts
throw new Error(`Cannot create state directory ${config.stateDir}: permission denied`);
throw new Error(`Cannot find server.ts. Set BROWSE_SERVER_SCRIPT env or run from the browse source tree.`);
```

**Typed catch blocks**:
```typescript
// From cli.ts
} catch (err: any) {
  if (err.code === 'EACCES') { ... }
  if (err.code === 'ENOTDIR') { ... }
  throw err;
}
```

**Async error wrapping**:
```typescript
// From cli.ts - clear error propagation
if (err.name === 'AbortError') {
  console.error('[browse] Command timed out after 30s');
  process.exit(1);
}
```

**Try/catch with fallbacks**:
```typescript
// From url-validation.ts - graceful degradation
async function resolvesToBlockedIp(hostname: string): Promise<boolean> {
  try {
    const dns = await import('node:dns');
    const addresses = await resolve4(hostname);
    return addresses.some(addr => BLOCKED_METADATA_HOSTS.has(addr));
  } catch {
    return false;  // DNS resolution failed — not a rebinding risk
  }
}
```

### Quality Assessment

- Error messages are descriptive and actionable
- Error codes checked for specific handling (`EACCES`, `ENOENT`)
- Best-effort operations wrapped in try/catch with silent fallback
- No bare `throw` without context

### Gap

- No centralized error type hierarchy
- No structured error codes enum
- Error handling varies by module (some return null, some throw)

---

## 5. Logging

### Logging Patterns

**File-based logging** with bounded buffers:
- Console, network, and dialog logs stored in project `.gstack/` directory
- `CircularBuffer<T>` caps at 50,000 entries (memory-bounded)

**Structured log format**:
```typescript
// From config.ts - ISO timestamp prefix
fs.appendFileSync(logPath, `[${new Date().toISOString()}] Warning: ...`);
```

**Error visibility**:
- Errors written to `browse-startup-error.log` (Windows-compatible)
- Server errors streamed to stderr
- Startup messages use `[browse]` prefix for filtering

### Redaction in Logs

Security-conscious redaction for sensitive values:
- `storage` command redacts keys matching: `auth_token`, `api_key`, `secret`, `password`, `token`, `jwt`, `ghp_`, `sk-`
- JWT pattern detection: `eyJhbGciOiJIUzI1NiJ9.payload.sig`
- GitHub PAT pattern detection: `ghp_` prefix
- Length preservation: `[REDACTED — 6 chars]`

### Gap

- No log level configuration (DEBUG, INFO, WARN, ERROR)
- No centralized logger interface
- No log rotation strategy

---

## 6. Configuration Management

### Config Resolution Hierarchy

```typescript
// From config.ts - clear priority order
// 1. BROWSE_STATE_FILE env → derive stateDir from parent
// 2. git rev-parse --show-toplevel → projectDir/.gstack/
// 3. process.cwd() fallback (non-git environments)
```

### Environment-Based Config

- `BROWSE_STATE_FILE` for state isolation (used in tests and server spawning)
- `BROWSE_SERVER_SCRIPT` for custom server path
- `CI` / `CONTAINER` detection for sandbox flags
- `BROWSE_EXTENSIONS_DIR` for Chrome extension support

### Secrets Management

`.env.example` contains only `ANTHROPIC_API_KEY` — minimal secrets surface.

### Quality Assessment

- Centralized config module (`config.ts`)
- Clear resolution logic with documented priority
- State directory auto-created with `.gitignore` update
- No secrets in config (env vars only)

### Gap

- No schema validation for config values
- No config file format (all env-based)

---

## 7. Security Practices

### Security Features Implemented

**Input Validation**:
- URL scheme blocking (only `http:` and `https:` allowed)
- Cloud metadata endpoint blocking (169.254.169.254, metadata.google.internal, etc.)
- DNS rebinding protection via async DNS resolution for non-loopback/non-private hosts
- Path traversal prevention for file operations (`/tmp` whitelist, `..` blocking)
- Path validation: "Path must be within" error for screenshot/pdf/eval

**Sensitive Value Redaction**:
- Storage reads redact sensitive keys and JWT/GitHub PAT patterns
- Form field password values redacted
- CLI output redacts typed passwords, cookie values, Authorization headers
- Error messages never leak sensitive data

**Platform-Specific Handling**:
- Windows: No-sandbox by default, Node.js subprocess for detached process
- CI/Container: `--no-sandbox` flag auto-added

### Gap

- No security-focused linting rules (e.g., no `eslint-plugin-security`)
- No dependency vulnerability scanning (npm audit, Snyk, etc.)

---

## 8. Code Review Patterns

### Commit Hygiene

From `CLAUDE.md`:
- **Bisect commits**: Every commit should be a single logical change
- Examples: Rename separate from behavior, test infrastructure separate from tests, template changes separate from generated file regeneration

### CHANGELOG Discipline

- CHANGELOG.md is **for users**, not contributors
- Entry describes what the user can now do
- Written at ship time, not during development
- Version bumps are branch-scoped

### E2E Eval Failure Protocol

Clear blame assignment:
1. Never claim "not related" without proof
2. Run same eval on main/base branch
3. If it passes on main but fails on branch — it IS your change
4. Unverified claims flagged as risk in PR body

### CI/CD Review

- actionlint runs on push/PR for workflow linting
- Gate-tier tests block merge
- Periodic tests run weekly via cron

### Gap

- No PR template observed
- No required reviewers or branch protection rules documented
- No automated code review tools (e.g., CodeClimate, SonarQube)

---

## 9. Observability

### Telemetry

`test/telemetry.test.ts` exists — project uses telemetry for eval tracking.

### Eval Store

`test/helpers/eval-store.ts` provides structured eval run storage to `~/.gstack-dev/evals/`.

### Session Runners

Multiple session runner implementations:
- `session-runner.ts` (primary)
- `codex-session-runner.ts`
- `gemini-session-runner.ts`

### Gap

- No structured logging library
- No metrics collection (Prometheus, StatsD)
- No distributed tracing

---

## Summary Table

| Category | Rating | Notes |
|----------|--------|-------|
| Testing | **Strong** | Comprehensive integration tests, security tests, E2E with tier system |
| Type Safety | **Moderate** | TypeScript used, but no tsconfig.json or strict mode visibility |
| Linting | **Weak** | No ESLint/Prettier — code style is convention-based |
| Error Handling | **Strong** | Descriptive errors, typed catch blocks, graceful fallbacks |
| Logging | **Moderate** | Bounded buffers, structured format, no log levels |
| Config | **Strong** | Clear hierarchy, env-based, centralized |
| Secrets | **Strong** | Minimal surface, redaction everywhere |
| Code Review | **Strong** | Bisect commits, CHANGELOG discipline, eval failure protocol |

---

## Recommendations

1. **Add tsconfig.json** with `strict: true` to enforce TypeScript safety
2. **Add ESLint** with TypeScript rules for consistent code quality
3. **Add Prettier** for automated formatting (2-space, single quotes, etc.)
4. **Add npm audit** or dependency scanning to CI
5. **Consider structured logging** (e.g., pino) for production observability
