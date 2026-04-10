# Code Quality: gstack

## Overview

gstack demonstrates strong testing practices and error handling, with moderate type safety and weak linting enforcement. The project uses Bun's built-in TypeScript support without visible strict configuration.

---

## Testing

### Framework & Infrastructure

| Aspect | Implementation |
|--------|----------------|
| Test Runner | `bun test` (fast, built-in) |
| E2E Framework | Playwright (Chromium) |
| LLM Evaluation | Claude -p + LLM judge (paid tier) |
| Test Selection | Diff-based (auto-selects tests based on git changes) |
| Test Tiers | `gate` (blocks merge) and `periodic` (weekly cron) |

### Test Coverage

**Unit/Integration Tests** (`browse/test/`, `test/`):

- **config.test.ts** (317 lines): Path resolution, gitignore handling, version hash, server health checks
- **commands.test.ts** (1,837 lines): Comprehensive command testing across navigation, content extraction, interaction, dialogs, security, workflows
- **skill-validation.test.ts**: Validates all `$B` commands and snapshot flags

**Coverage Areas**:
- Happy path: Multiple positive test cases per command
- Error paths: Missing args, invalid format, bad URLs
- Security: Value redaction, path traversal, blocked metadata endpoints
- Edge cases: Buffer bounds (50k cap), empty pages, SPA rendering, dialog handling
- Workflows: Multi-step sequences (navigate -> snapshot -> click -> verify)

**Quality Patterns**:
- Real fixtures (HTML files via test server)
- Temporary file cleanup in `finally` blocks
- Async timeout handling (15-20s for browser operations)
- HTTP health checks over PID-based detection
- Permission error simulation

**E2E Eval Tests** (`test/skill-e2e-*.test.ts`):
- Multi-category E2E (BWS, CSO, design, plan, QA, review, workflow)
- Routing E2E test
- Codex and Gemini integration tests
- Tier classification via `touchfiles.ts`

### Gaps

- No dedicated unit tests for utility functions in isolation
- Some tests use `require()` instead of ES module imports
- Skill validation skips non-existent skills rather than failing

---

## Type Systems

### TypeScript Usage

- **Runtime**: Bun (native TypeScript support, no build step)
- **Type Coverage**: Comprehensive in core modules
- **Annotations**: Interfaces for all major types (`BrowseConfig`, `ServerState`, `RefEntry`, `BrowserState`)

### Configuration Gap

**No `tsconfig.json` found** in the repository. This suggests:
- TypeScript runs via Bun's built-in transpiler
- Strict mode may not be enforced
- No explicit compiler options visible

### Example Types

```typescript
export interface BrowseConfig {
  projectDir: string;
  stateDir: string;
  stateFile: string;
  consoleLog: string;
  networkLog: string;
  dialogLog: string;
}

export interface RefEntry {
  locator: Locator;
  role: string;
  name: string;
}
```

### Gaps

- No visible TypeScript strictness configuration
- Unknown whether `strictNullChecks`, `noImplicitAny`, or `strictFunctionTypes` are enabled

---

## Linting & Formatting

### Tooling Inventory

| Tool | Status |
|------|--------|
| ESLint | NOT FOUND |
| Prettier | NOT FOUND |
| actionlint | Present (`.github/actionlint.yaml`) |

### Code Style Observations

- 2-space indentation
- Opening brace on same line
- `import * as fs from 'fs'` namespace style
- No enforced formatting standard

### actionlint

Only CI workflow files are linted. No application code linting.

```yaml
# .github/actionlint.yaml
self-hosted-runner:
  labels:
    - ubicloud-standard-2
```

### Gaps

- No automated formatting enforcement
- No ESLint rules for TypeScript best practices
- Style consistency is convention-based, not tool-enforced

---

## Error Handling

### Patterns

**Specific error types with context**:
```typescript
throw new Error(`Cannot create state directory ${config.stateDir}: permission denied`);
throw new Error(`Cannot find server.ts. Set BROWSE_SERVER_SCRIPT env or run from the browse source tree.`);
```

**Typed catch blocks**:
```typescript
} catch (err: any) {
  if (err.code === 'EACCES') { ... }
  if (err.code === 'ENOTDIR') { ... }
  throw err;
}
```

**Async error wrapping with graceful degradation**:
```typescript
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

### Assessment

- Error messages are descriptive and actionable
- Error codes checked for specific handling (`EACCES`, `ENOENT`)
- Best-effort operations wrapped in try/catch with silent fallback
- No bare `throw` without context

### Gaps

- No centralized error type hierarchy
- No structured error codes enum
- Error handling varies by module (return null vs throw)

---

## Logging

### Patterns

**Bounded buffers**:
- Console, network, and dialog logs stored in `.gstack/` directory
- `CircularBuffer<T>` caps at 50,000 entries

**Structured format**:
```typescript
fs.appendFileSync(logPath, `[${new Date().toISOString()}] Warning: ...`);
```

**Error visibility**:
- Errors written to `browse-startup-error.log`
- Server errors streamed to stderr
- Startup messages use `[browse]` prefix

### Security-Conscious Redaction

- Storage reads redact: `auth_token`, `api_key`, `secret`, `password`, `token`, `jwt`, `ghp_`, `sk-`
- JWT pattern: `eyJhbGciOiJIUzI1NiJ9.payload.sig`
- GitHub PAT prefix: `ghp_`
- Length preservation: `[REDACTED — 6 chars]`

### Gaps

- No log level configuration (DEBUG, INFO, WARN, ERROR)
- No centralized logger interface
- No log rotation strategy

---

## Configuration Management

### Resolution Hierarchy

```typescript
// 1. BROWSE_STATE_FILE env → derive stateDir from parent
// 2. git rev-parse --show-toplevel → projectDir/.gstack/
// 3. process.cwd() fallback (non-git environments)
```

### Environment Variables

- `BROWSE_STATE_FILE` — state isolation
- `BROWSE_SERVER_SCRIPT` — custom server path
- `CI` / `CONTAINER` — sandbox flags
- `BROWSE_EXTENSIONS_DIR` — Chrome extension support

### Assessment

- Centralized config module (`config.ts`)
- Clear resolution logic with documented priority
- State directory auto-created with `.gitignore` update
- No secrets in config (env vars only)

### Gaps

- No schema validation for config values
- No config file format (all env-based)

---

## Code Review Patterns

### Commit Hygiene

From `CLAUDE.md`:
- **Bisect commits**: Every commit should be a single logical change
- Examples: Rename separate from behavior, test infrastructure separate from tests

### CHANGELOG Discipline

- CHANGELOG.md is **for users**, not contributors
- Entry describes what the user can now do
- Written at ship time, not during development

### E2E Eval Failure Protocol

1. Never claim "not related" without proof
2. Run same eval on main/base branch
3. If it passes on main but fails on branch — it IS your change
4. Unverified claims flagged as risk in PR body

### Gaps

- No PR template observed
- No required reviewers or branch protection rules documented
- No automated code review tools (CodeClimate, SonarQube)

---

## Observability

### Telemetry

- `test/telemetry.test.ts` exists
- `test/helpers/eval-store.ts` stores eval runs to `~/.gstack-dev/evals/`

### Session Runners

- `session-runner.ts` (primary)
- `codex-session-runner.ts`
- `gemini-session-runner.ts`

### Gaps

- No structured logging library
- No metrics collection (Prometheus, StatsD)
- No distributed tracing

---

## Summary

| Category | Rating | Notes |
|----------|--------|-------|
| Testing | **Strong** | Comprehensive integration tests, security tests, E2E with tier system |
| Type Safety | **Moderate** | TypeScript used, but no tsconfig.json or strict mode visibility |
| Linting | **Weak** | No ESLint/Prettier — code style is convention-based |
| Error Handling | **Strong** | Descriptive errors, typed catch blocks, graceful fallbacks |
| Logging | **Moderate** | Bounded buffers, structured format, no log levels |
| Config | **Strong** | Clear hierarchy, env-based, centralized |
| Code Review | **Strong** | Bisect commits, CHANGELOG discipline, eval failure protocol |

---

## Recommendations

1. **Add tsconfig.json** with `strict: true` to enforce TypeScript safety
2. **Add ESLint** with TypeScript rules for consistent code quality
3. **Add Prettier** for automated formatting
4. **Add npm audit** or dependency scanning to CI
5. **Consider structured logging** (e.g., pino) for production observability
