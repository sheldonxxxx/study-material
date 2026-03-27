# Code Quality Assessment: GitClaw

## Type System

**TypeScript 5.7 with strict mode enabled.**

| Compiler Option | Value | Impact |
|-----------------|-------|--------|
| `strict` | `true` | Full strict type checking |
| `skipLibCheck` | `true` | Speeds build, skips declaration errors |
| `forceConsistentCasingInFileNames` | `true` | Cross-platform compatibility |
| `declaration` | `true` | Generates `.d.ts` type files |

**Issues identified:**
- `src/sandbox.ts:18` — `gitMachine: any` and `machine: any` due to optional peer dependency
- `src/sdk.ts:259` — `Record<string, any>` for modelOptions, losing type safety
- `src/sdk.ts:519` — `handler: (args: any, signal?: AbortSignal)` uses `any` input
- `src/tool-loader.ts:19` — `buildTypeboxSchema` returns `any`

**Positive patterns:**
- Typebox (`@sinclair/typebox`) used for runtime schema validation with TypeScript types
- All exported interfaces defined in `sdk-types.ts`
- `AgentManifest` fully typed in `loader.ts`

---

## Testing

**Single test file:** `test/sdk.test.ts` (274 lines)

Uses Node.js native test runner with `--experimental-strip-types` flag.

### Test Coverage

| Test Suite | What It Covers |
|------------|----------------|
| `exports` | Verifies `query`, `tool`, `loadAgent` are functions |
| `tool()` | Tool definition creation, handler return types (string, object with text/details) |
| `buildTypeboxSchema()` | Schema conversion for string, number, boolean, array, empty, optional fields |
| `wrapToolWithProgrammaticHooks()` | Hook blocking, allowing, modifying args, context passing |
| `query()` | Error message emission, Query object methods, onError hook, message collection |

### Test Gaps

- No tests for the CLI entry (`src/index.ts`)
- No tests for built-in tools (read, write, cli, memory, etc.)
- No tests for plugin loading (`src/plugins.ts`)
- No tests for compliance checking (`src/compliance.ts`)
- No tests for session management (`src/session.ts`)
- No integration tests
- No tests for sandbox functionality
- No tests for agent loading (`src/loader.ts`) beyond being called

**Test execution:**
```bash
npm test  # node --test test/*.test.ts --experimental-strip-types
```

---

## Linting and Formatting

**No ESLint or Prettier configuration found.**

The project relies entirely on TypeScript compiler checks (`tsc`) for code quality.

**Observations:**
- No `.eslintrc`, `.eslintrc.json`, or `eslint.config.*`
- No `.prettierrc` or `.prettierrc.json`
- No combined config like `eslint-config-prettier`

This is a gap compared to industry standards for TypeScript projects.

---

## Code Review Patterns

### Commit Messages

Conventional commit format observed in git history:

```
fix: default to cwd instead of interactive directory prompt (#18)
fix: resolve SkillsFlow skills display issue (fixes #12) (#23)
fix: allow creating markdown (.md) files (fixes #11) (#22)
ci: add npm publish workflow on tagged releases
fix: install.sh — strip Windows \r from .env
```

**Pattern:** `<type>: <description> (fixes #<issue>) (<#PR>)`

Types used: `fix:`, `ci:`, `feat:` (from CONTRIBUTING.md examples)

### Pull Request Descriptions

PRs reference issues with `fixes #<number>` and `(#<PR>)` syntax. The merge commit carries the full reference.

### CONTRIBUTING.md Guidelines

States:
- "One concern per PR — keep pull requests focused and reviewable"
- "Test your changes — add or update tests in `test/` when applicable"
- Commit message examples: "Add voice mode...", "Fix memory tool...", "Update SDK query..."

---

## Error Handling Patterns

### Silent Error Suppression (Anti-pattern)

**`src/audit.ts:40-41`:**
```typescript
try {
  await mkdir(dirname(this.logPath), { recursive: true });
  await appendFile(this.logPath, JSON.stringify(entry) + "\n", "utf-8");
} catch {
  // Audit logging failures are non-fatal
}
```
Audit logging failures are silently swallowed with empty catch block.

### Try/Catch with Best-Effort Cleanup

**`src/sdk.ts:427-434`:**
```typescript
if (localSession) {
  try { localSession.finalize(); } catch { /* best-effort */ }
}
if (sandboxCtx) {
  await sandboxCtx.gitMachine.stop().catch(() => {});
}
```
Best-effort cleanup pattern — errors from cleanup are ignored.

### Timeout and Abort Handling

**`src/hooks.ts:90-93`:**
```typescript
const timeout = setTimeout(() => {
  child.kill("SIGTERM");
  reject(new Error(`Hook "${hook.script}" timed out after 10s`));
}, 10_000);
```
10-second timeout for hook scripts.

**`src/tool-loader.ts:91-94`:**
```typescript
const timeout = setTimeout(() => {
  child.kill("SIGTERM");
  reject(new Error(`Tool "${def.name}" timed out after 120s`));
}, 120_000);
```
120-second timeout for declarative tools.

### Path Traversal Guard

**`src/hooks.ts:63-69`:**
```typescript
const resolvedScript = resolve(scriptPath);
const allowedBase = resolve(baseDir);
if (!resolvedScript.startsWith(allowedBase + "/") && resolvedScript !== allowedBase) {
  reject(new Error(`Hook "${hook.script}" escapes its base directory`));
  return;
}
```
Hooks are sandboxed to their base directory.

### Error Propagation via Messages

**`src/sdk.ts:326-338`:**
```typescript
if (msg.stopReason === "error") {
  pushMsg({
    type: "system",
    subtype: "error",
    content: msg.errorMessage || "LLM request failed (unknown error)",
  });
}
```
LLM errors are emitted as system messages to the stream, not thrown.

---

## Logging Practices

### console.error for SDK Diagnostics

**`src/sdk.ts:179` — Plugin tool collision warning:**
```typescript
console.warn(`[plugin:${plugin.manifest.id}] Tool "${t.name}" collides with existing tool — skipping`);
```

**`src/sdk.ts:191` — External tool injection:**
```typescript
console.error(`[sdk] Injected ${converted.length} external tools: ${converted.map(t => t.name).join(", ")}`);
```

**`src/sdk.ts:193` — Tool count:**
```typescript
console.error(`[sdk] Total tools before filtering: ${tools.length} → ${tools.map(t => t.name).join(", ")}`);
```

**Pattern:** Uses `console.error` for operational diagnostics (not errors), `console.warn` for warnings.

### Hook Error Logging

**`src/hooks.ts:136`:**
```typescript
console.error(`Hook error: ${err.message}`);
```
Hook errors are logged but do not block execution.

---

## Configuration Management

### YAML-Based Config with Environment Inheritance

**`src/config.ts`:**
- Loads `config/default.yaml`
- Merges `config/<env>.yaml` on top via `deepMerge`
- Environment selected via `--env` flag or `GITCLAW_ENV` env var

**`src/loader.ts`:**
- Agent manifest (`agent.yaml`) loaded and parsed with `js-yaml`
- Deep merge for inheritance (`manifest.extends`)
- Model override: `envConfig.model_override` > CLI flag > manifest `model.preferred`

### Environment Variables for Secrets

Secrets are read from environment variables, never hardcoded:

| Variable | Used In |
|----------|---------|
| `GITHUB_TOKEN` / `GIT_TOKEN` | `sandbox.ts`, `sdk.ts` |
| `ANTHROPIC_API_KEY` | Expected at runtime (documented) |
| `OPENAI_API_KEY` | Voice mode (documented) |
| `GITCLAW_ENV` | Config environment selection |

---

## Security Observations

### Positive

1. **Path traversal guard** in hook execution prevents directory escape
2. **Timeout limits** on external script execution (10s hooks, 120s tools)
3. **Abort signal support** throughout async operations
4. **Process environment isolation** in child_process spawns
5. **Audit logging** for tool use and session events

### Concerns

1. **Silent error swallowing** in audit logger could mask security-relevant failures
2. **No input sanitization** visible for YAML-loaded configurations
3. **`any` types** in critical paths reduce TypeScript's ability to catch type errors
4. **`skipLibCheck: true`** bypasses standard declaration file validation

---

## CI/CD

**`.github/workflows/publish.yml`:**
- Trigger: Tag matching `v*`
- Steps: `npm ci` → `npm run build` → `npm publish`
- No test step in CI
- No linting step

**Gap:** CI does not run tests before publishing. Only builds.

---

## Summary

| Aspect | Assessment |
|--------|------------|
| Type Safety | Good — strict TypeScript, but `any` leaks in places |
| Test Coverage | Minimal — single test file, core SDK tested, tools/CLI not tested |
| Linting | None — no ESLint/Prettier |
| Code Review | Good commit conventions, PR-focused guidelines |
| Error Handling | Inconsistent — silent catches, best-effort cleanup |
| Logging | `console.error` for diagnostics, no structured logging |
| Config | YAML-based with inheritance, env vars for secrets |
| Security | Path guards, timeouts present; silent failures concerning |

### Key Recommendations

1. Add ESLint with `@typescript-eslint` and Prettier for consistent formatting
2. Expand test coverage — tools, CLI, plugins, compliance all untested
3. Replace silent catch blocks with structured error logging
4. Add test step to CI pipeline
5. Reduce `any` type usage, especially in core SDK paths
6. Add integration tests for the full query lifecycle
