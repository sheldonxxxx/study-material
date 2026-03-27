# Code Quality Assessment: nanoclaw

## Overview

| Dimension | Assessment |
|-----------|------------|
| Type Safety | Strong (TypeScript strict mode) |
| Testing | Good coverage, Vitest framework |
| Linting | ESLint 9 flat config with TypeScript |
| Formatting | Prettier (minimal config) |
| Error Handling | Structured logging, global handlers |
| Commit Style | Conventional commits |

---

## Type System

### TypeScript Configuration

The project uses **TypeScript strict mode** with a modern configuration:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

**Strengths:**
- Strict mode enabled throughout
- ES2022 target with NodeNext module resolution (modern idiom)
- Declaration maps enabled for debugger stepping
- ForceConsistentCasingInFileNames prevents cross-platform issues
- skipLibCheck: true to avoid type-checking third-party libs

**Note:** The `container/agent-runner/tsconfig.json` is a separate project with its own strict TypeScript config (no sourceMap for lighter container images).

### Observed Issues

1. **Implicit `any` in test files**: The `npm run typecheck` output shows multiple `Parameter 'call' implicitly has an 'any' type` errors in `group-queue.test.ts`. These errors appear in test files which typically run with `--skipLibCheck` but the typecheck is strict.

2. **Missing type declarations at typecheck time**: The `npm run typecheck` run shows errors for modules like `vitest`, `pino`, `cron-parser`, `@onecli-sh/sdk` — these are dev/runtime dependencies that would be present after `npm ci` but aren't available in the bare checkout. This is expected behavior and handled by CI which runs `npm ci` first.

### Type Practices Observed

- **Type exports**: `src/types.ts` centralizes shared interfaces (`Channel`, `NewMessage`, `RegisteredGroup`)
- **Guards**: `sender-allowlist.ts` uses `isValidEntry(entry: unknown): entry is ChatAllowlistEntry` type guard pattern
- **Explicit typing in IPC**: The `processTaskIpc` function uses explicit interface for the IPC data object
- **Unused variable detection**: ESLint rule `@typescript-eslint/no-unused-vars` set to error with ignore patterns (`^_` prefix)

---

## Testing

### Test Framework

| Tool | Version | Config |
|------|---------|--------|
| Vitest | 4.0.x | `vitest.config.ts`, `vitest.skills.config.ts` |
| Coverage | @vitest/coverage-v8 | Not enabled in base config |

### Test Coverage

**19 test files** found:

**In `src/`:**
- `channels/registry.test.ts`
- `claw-skill.test.ts`
- `container-runner.test.ts`
- `container-runtime.test.ts`
- `db-migration.test.ts`
- `db.test.ts`
- `formatting.test.ts`
- `group-folder.test.ts`
- `group-queue.test.ts`
- `ipc-auth.test.ts`
- `remote-control.test.ts`
- `routing.test.ts`
- `sender-allowlist.test.ts`
- `task-scheduler.test.ts`
- `timezone.test.ts`

**In `setup/`:**
- `environment.test.ts`
- `platform.test.ts`
- `register.test.ts`
- `service.test.ts`

### Test Quality Observations

**Strengths:**

1. **Descriptive test names**: Tests use clear BDD names like `getChannelFactory returns undefined for unknown channel`

2. **Test isolation**: The `db.test.ts` uses `_initTestDatabase()` in `beforeEach` to ensure clean state

3. **Helper functions**: `db.test.ts` defines a `store()` helper that creates messages with sensible defaults

4. **Edge case coverage**: `db.test.ts` tests:
   - Empty content filtering
   - `is_from_me` flag persistence
   - Upsert behavior on duplicate ID+chat_jid
   - Bot message filtering via `is_bot_message` flag
   - Legacy bot message filtering via content prefix backstop
   - LIMIT behavior (caps to limit, returns most recent, preserves chronological order)

5. **Module-level state awareness**: `registry.test.ts` explicitly acknowledges the shared registry state and orders tests to account for cumulative registrations

6. **Mock usage**: `group-queue.test.ts` uses `vi.mock()` for `config.js` and `vi.useFakeTimers()` for deterministic timing tests

**Issues:**

1. **No coverage reporting configured**: `vitest.config.ts` does not include a coverage reporter. The `02-tech-stack.md` mentions `@vitest/coverage-v8` is installed, but the config does not enable it.

2. **Vitest config missing coverage block**:
   ```typescript
   // vitest.config.ts - current
   export default defineConfig({
     test: {
       include: ['src/**/*.test.ts', 'setup/**/*.test.ts'],
     },
   });
   ```
   To enable coverage, it would need a `coverage:` block with v8 provider.

3. **TypeScript errors in test files**: The `group-queue.test.ts` has untyped `call` parameters at multiple line numbers (315, 358, 401, 468, 477). This suggests these are callback parameters that should be typed.

---

## Linting

### ESLint Configuration

**Version**: ESLint 9.35.x with flat config

```javascript
export default [
  { ignores: ['node_modules/', 'dist/', 'container/', 'groups/'] },
  { files: ['src/**/*.{js,ts}'] },
  { languageOptions: { globals: globals.node } },
  pluginJs.configs.recommended,
  ...tseslint.configs.recommended,
  {
    rules: {
      'preserve-caught-error': ['error', { requireCatchParameter: true }],
      '@typescript-eslint/no-unused-vars': ['error', { /* strict config */ }],
      'no-catch-all/no-catch-all': 'warn',
      '@typescript-eslint/no-explicit-any': 'warn',
    },
  },
];
```

### Rules Applied

| Rule | Level | Purpose |
|------|-------|---------|
| `preserve-caught-error` | error | Requires catch parameter (no `catch {}`) |
| `@typescript-eslint/no-unused-vars` | error | Strict unused variable detection |
| `@typescript-eslint/no-explicit-any` | warn | Warns against `any` type |
| `no-catch-all/no-catch-all` | warn | Warns against catching `ProcessExit` |

### Strengths

1. **Strict unused vars**: Configured to catch all unused variables, arguments, and caught errors
2. **Catch parameter required**: Prevents silent error swallowing
3. **Scope-aware ignores**: Uses `^_` prefix pattern and `argsIgnorePattern` for intentional unused params

---

## Formatting

### Prettier Configuration

```json
{
  "singleQuote": true
}
```

**Minimal config** — only `singleQuote: true` is set.

**Scripts:**
- `npm run format` / `npm run format:fix` — format source
- `npm run format:check` — verify formatting

### Observations

1. The Prettier config is minimal. Additional sane defaults typically seen:
   - `printWidth: 100` or similar
   - `trailingComma: "es5"` for consistent output
   - `semi: true` for statement termination

2. The ESLint config does not include Prettier integration (no `eslint-plugin-prettier`), meaning format and lint are separate concerns.

---

## Error Handling & Logging

### Structured Logging with Pino

The project uses **pino** for structured logging:

```typescript
// src/logger.ts
export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: { target: 'pino-pretty', options: { colorize: true } },
});
```

**Global error handlers:**
```typescript
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception');
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  logger.error({ err: reason }, 'Unhandled rejection');
});
```

### Error Handling Patterns

**Observed in `ipc.ts`:**

1. **Try-catch with logging and recovery**:
   ```typescript
   try {
     // operation
   } catch (err) {
     logger.error({ err }, 'Error reading IPC base directory');
     setTimeout(processIpcFiles, IPC_POLL_INTERVAL);  // recover
     return;
   }
   ```

2. **Error file isolation**: Failed IPC files are moved to an `errors/` subdirectory rather than deleted, enabling post-mortem analysis

3. **Authorization errors are logged as warnings** (not errors), distinguishing security-relevant events from system failures

4. **Defensive validation with early returns**:
   ```typescript
   if (!targetGroupEntry) {
     logger.warn({ targetJid }, 'Cannot schedule task: target group not registered');
     break;
   }
   ```

### Config Management

**Pattern in `config.ts`:**
- Reads from `.env` file via `readEnvFile()`
- Falls back to `process.env`
- Timezone resolution validates IANA identifiers before accepting
- Numeric env vars use `parseInt` with defaults
- Mount paths stored outside project root (`~/.config/nanoclaw/`) for security

---

## Git Commit Style

### Observed Convention

The project uses **conventional commits** with the following prefixes:

| Prefix | Use Case |
|--------|----------|
| `fix()` | Bug fixes |
| `feat()` | New features |
| `chore()` | Maintenance, version bumps, contributor updates |
| `docs()` | Documentation changes |
| `refactor()` | Code refactoring |

### Examples from Recent History

```
fix(init-onecli): only offer to migrate container-facing credentials
feat(init-onecli): offer to migrate non-Anthropic .env credentials to vault
feat: add /init-onecli skill for OneCLI Agent Vault setup and credential migration
docs: update README and security docs to reflect OneCLI Agent Vault adoption
chore: bump version to 1.2.34
feat(skill): add Emacs channel skill
```

**Merge commits** follow GitHub's convention: `Merge pull request #N from author/branch`

### Observations

1. **Scoped commits**: Uses `fix(scope):` and `feat(scope):` format for clarity
2. **No `fixup!` or `squash!` prefixes visible** in recent history
3. **Consistent capitalization**: Subject lines start with lowercase after scope
4. **Issue references**: PR numbers referenced in merge commits, not in subject lines

---

## CI/CD Quality Gates

### CI Workflow (`.github/workflows/ci.yml`)

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: npm
  - run: npm ci
  - name: Format check
    run: npm run format:check
  - name: Typecheck
    run: npx tsc --noEmit
  - name: Tests
    run: npx vitest run
```

**Gates in order:**
1. `npm ci` — install dependencies
2. Format check — Prettier validation
3. Typecheck — TypeScript type validation (no emit)
4. Tests — Vitest test runner

### Observations

1. **No coverage gate**: The CI runs tests but does not enforce coverage thresholds
2. **No security scanning**: No linting for security patterns (e.g., `eslint-plugin-security`)
3. **No dependency auditing**: No `npm audit` step
4. **Separate skills test**: `vitest.skills.config.ts` is not run in CI — it's likely for development verification of skill implementations

---

## Summary of Strengths

1. **TypeScript strict mode** throughout with modern module resolution
2. **Comprehensive test suite** with 19 test files covering critical paths
3. **Good test isolation** via `beforeEach` reset patterns
4. **Structured logging** with pino and global error handlers
5. **Proper error recovery** patterns (moving failed files to errors/, retry timeouts)
6. **Authorization checks** in IPC handlers with security event logging
7. **Conventional commit discipline** in git history
8. **CI gates** for format, types, and tests

## Summary of Issues

1. **No coverage reporting configured** in vitest.config.ts
2. **Minimal Prettier config** (only singleQuote set)
3. **No ESLint integration with Prettier** (separate tools)
4. **Implicit `any` in test callbacks** (group-queue.test.ts lines 315, 358, 401, 468, 477)
5. **No coverage gate in CI**
6. **No security scanning step in CI**
7. **Skills tests** (`vitest.skills.config.ts`) **not run in CI**

---

## Recommendations

1. **Enable coverage reporting**: Add coverage block to `vitest.config.ts`:
   ```typescript
   coverage: {
     provider: 'v8',
     reporter: ['text', 'html'],
     include: ['src/**/*.ts'],
     exclude: ['src/**/*.test.ts'],
   }
   ```

2. **Add coverage gate in CI**: Consider requiring >70% coverage on new PRs

3. **Fix implicit any in test callbacks**: Type the callback parameters in `group-queue.test.ts`

4. **Expand Prettier config**: Add `printWidth`, `trailingComma`, and `semi` for consistency

5. **Run skill tests in CI**: Add a step that runs `vitest run --config vitest.skills.config.ts`
