# Code Quality: oh-my-openagent

## Type System

### TypeScript Configuration

- **Strict mode** enabled in `tsconfig.json`
- `forceConsistentCasingInFileNames: true`
- `skipLibCheck: true`
- TypeScript compilation enforced in CI via `tsc --noEmit`
- Declaration files generated separately via `tsc --emitDeclarationOnly`

### Runtime Validation

- **Zod v4** extensively used for runtime schema validation
- 25+ separate schema modules in `src/config/schema/`
- All user-facing config validated via `OhMyOpenCodeConfigSchema.safeParse()`
- Test files co-located with source files (`.test.ts` alongside `.ts`)
- `tsconfig.json` explicitly excludes `**/*.test.ts` from compilation

### Type Coverage

- `bun-types: 1.3.10` provides Bun-specific type definitions
- Test files use `/// <reference types="bun-types" />` directive

## Testing

### Framework

- **Test runner**: `bun:test` (Bun's native test runner)
- **Assertions**: `expect()` from `bun:test`
- **Mocking**: `vi` from `bun:test` (Vitest-compatible API)

### Test Distribution

Tests are spread throughout the source tree, not centralized:

| Area | Approximate Test Files |
|------|----------------------|
| `src/cli/run/` | 25+ |
| `src/features/background-agent/` | 20+ |
| `src/config/` | 10+ |
| `src/tools/task/` | 6 |
| `src/cli/doctor/` | 8 |
| `src/agents/` | 15+ |

### Test Patterns

**Given-When-Then (GWT) Convention:**
```typescript
it("#given no arguments #when building anti-duplication section #then returns comprehensive rule section", () => {
  //#given: no special configuration needed
  //#when: building the anti-duplication section
  const result = buildAntiDuplicationSection()
  //#then: should contain the anti-duplication rule
  expect(result).toContain("Anti-Duplication Rule")
})
```

**Mock Isolation Strategy:**
- Tests using `mock.module()` run in separate `bun test` processes
- CI splits tests: mock-heavy tests run first in isolation

**Test Setup** (via `bunfig.toml` preload):
```typescript
beforeEach(() => {
  resetClaudeSessionState()
  resetModelFallbackState()
})
```

### Coverage Areas

- Schema validation (success/error/edge cases)
- Agent resolution priority chain (CLI > env > config > default)
- Background task concurrency and circuit breakers
- Configuration loading and plugin detection
- CLI commands (doctor checks, model resolution, OAuth flows)
- Error serialization and timeout handling

### Gaps

- No integration tests with real API calls (all mocked)
- No E2E tests
- No code coverage reporting

## Linting and Formatting

### Current State

| Tool | Status |
|------|--------|
| ESLint | Not found |
| Prettier | Not found |
| Biome | Not found |
| EditorConfig | Not found |

### Observed Code Style

- **Quotes**: Single quotes for strings
- **Semicolons**: Used consistently
- **Indentation**: 2 spaces
- **Braces**: K&R style (1TBS variant)

## Error Handling

### Centralized Error Serialization

```typescript
// src/cli/run/event-formatting.ts
export function serializeError(error: unknown): SerializedError {
  if (error instanceof Error) {
    return { name: error.name, message: error.message, stack: error.stack }
  }
  return { name: "UnknownError", message: String(error) }
}
```

### Pattern: JSON Error Returns

Tools return stringified error objects instead of throwing:
```typescript
if (!lock.acquired) {
  return JSON.stringify({ error: "task_lock_unavailable" })
}
```

### Signal Handling

```typescript
process.on("SIGINT", () => {
  console.log(pc.yellow("\nInterrupted. Shutting down..."))
  process.exit(130)
})
```

## Logging

### Terminal Styling

- `picocolors` for colored CLI output
- Structured formatting via `pc.dim()`, `pc.yellow()`, etc.

### Console Usage

- Low frequency (~35 files use `console.*`)
- Session metadata logged on startup (session ID, model)

## CI/CD Quality Gates

### Pipeline Jobs

| Job | Purpose |
|-----|---------|
| `block-master-pr` | Prevent direct PRs to master |
| `test` (mock-isolation) | Run mock-heavy tests separately |
| `test` (remaining) | Run remaining tests |
| `typecheck` | `tsc --noEmit` validation |
| `build` | `bun run build` + output verification |

### Build Verification

- Generates both `dist/index.js` and `dist/index.d.ts`
- Schema JSON generated via `script/build-schema.ts`

## Dependency Management

- **Trusted dependencies**: `@ast-grep/cli`, `@ast-grep/napi`, `@code-yeongyu/comment-checker`
- Simple Bun workspace (no Turborepo/Nx/Lerna)
- 11 platform-specific packages under `packages/`

## Commit Convention

```
<type>(<scope>): <description>

Types: fix, feat, docs, refactor, test, ci, merge
Scopes: (test), (ci), (agents), (skills), (commands), (shared), etc.
```

### PR Conventions

- PRs to `master` are blocked by CI
- All PRs must target `dev` branch
- CLA signature required

## Summary

| Area | Assessment |
|------|------------|
| Type Safety | Strong - strict TypeScript + Zod runtime validation |
| Test Coverage | Good - extensive unit tests with mock isolation |
| Config Management | Excellent - comprehensive Zod schemas |
| Error Handling | Good - centralized serialization, consistent patterns |
| CI/CD | Solid - typecheck, test isolation, PR branch protection |
| Linting/Formatting | Missing - no automated code style enforcement |
| Coverage Reporting | Missing - no visible coverage metrics |
