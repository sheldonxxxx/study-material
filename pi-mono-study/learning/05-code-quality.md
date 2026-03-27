# Code Quality Practices

## Overview

pi-mono demonstrates solid engineering practices with comprehensive testing, strict TypeScript configuration, and consistent tooling. Overall quality is rated **GOOD** with strong areas in type safety, formatting, and CI/CD.

## Testing

### Test Infrastructure

| Package | Test Runner | Test Count |
|---------|------------|------------|
| `@mariozechner/pi-ai` | vitest | ~40 test files |
| `@mariozechner/pi-agent-core` | vitest | ~4 test files |
| `@mariozechner/pi-coding-agent` | vitest | ~60 test files |
| `@mariozechner/pi-tui` | Node.js built-in test | ~22 test files |

### Testing Patterns

**Provider-level abort testing:** `abort.test.ts` tests cancellation across all LLM providers with retry configuration.

**Environment-gated tests:** Tests for paid providers are skipped when credentials are unavailable:
```typescript
describe.skipIf(!process.env.ANTHROPIC_OAUTH_TOKEN)("Anthropic Provider Abort", () => { ... });
```

**Vitest retry for flaky tests:**
```typescript
it("should abort mid-stream", { retry: 3 }, async () => { ... });
```

**Mock stream patterns:** `MockAssistantStream` class provides controlled testing without API calls.

### Coverage Gaps

- No visible unit tests for `pi-mom` (Slack bot) package
- No visible unit tests for `pi-pods` (GPU pod CLI) package
- Limited browser/E2E testing (only `check:browser-smoke` script)
- Web UI component testing not evident

## TypeScript Configuration

### Base Configuration

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "Node16",
        "strict": true,
        "declaration": true,
        "declarationMap": true,
        "sourceMap": true,
        "moduleResolution": "Node16"
    }
}
```

**Strict mode enabled** includes `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and more.

### Declaration Generation

Each package generates `.d.ts` files via `declaration: true`. Subpath exports explicitly define types for imports like `"./oauth": { "types": "./dist/oauth.d.ts" }`.

### Explicit `any` Suppressions

Biome configuration allows `any` in specific contexts:
- API key resolution (`getEnvApiKey` uses `any` for provider argument)
- Test file content extraction

## Biome Linting and Formatting

### Configuration

```json
{
    "linter": {
        "enabled": true,
        "rules": {
            "recommended": true,
            "style": {
                "noNonNullAssertion": "off",
                "useConst": "error"
            }
        }
    },
    "formatter": {
        "indentStyle": "tab",
        "indentWidth": 3,
        "lineWidth": 120
    }
}
```

**Notable settings:**
- 3-tab indent (deviation from common 2 or 4)
- 120 character line width (deviation from common 80)
- Biome runs in CI via `npm run check`

## Error Handling

### Patterns Used

**1. Synchronous Error Throwing:**
```typescript
if (!provider) {
    throw new Error(`No API provider registered for api: ${api}`);
}
```

**2. Try-Catch with Error Handling:**
```typescript
try {
    const response = await fetch(url);
    const data = await response.json();
} catch (error) {
    if (error instanceof Error) {
        return { error: error.message };
    }
    return { error: String(error) };
}
```

**3. Result Pattern (less common):** Errors silenced in validation, validation failure returned.

### Observations

- Standard `Error` class used throughout (no custom error types)
- Errors propagate via `throw` in stream wrapper
- Async errors caught with `.catch()` at CLI entry point

## Logging

### Current State

| Pattern | Count |
|---------|-------|
| `console.log` | ~700 |
| `console.error` | ~100 |
| `console.warn` | ~50 |
| `console.debug` | ~30 |

**The `mom` package** has the best logging practices with a dedicated `src/log.ts` module using structured logging with chalk. Other packages use ad-hoc `console.log`.

**Debug logging** only in mom package via `PI_TUI_WRITE_LOG` env var.

### Gap

No structured logging library (pino, winston) across packages. Secrets appear filtered from logs with `<authenticated>` placeholders.

## Commit Conventions

**Format:** `type(scope): message` (Conventional Commits)

**Examples:**
```
fix(tui): reset viewport state after shrink
feat(coding-agent): Add sessionDir support in settings.json (#2598)
test(tui): remove stale slash autocomplete chaining case
docs: complete changelog entries for unreleased changes
```

**Observations:**
- Very frequent small commits (1-3 per day)
- Issue references in commit messages (`closes #2429`)
- Commits go directly to main (no visible PR review process)

## Quality Metrics Summary

| Dimension | Score | Notes |
|-----------|-------|-------|
| Test Coverage | 7/10 | Good unit/integration tests, missing mom/pods tests |
| Type Safety | 8/10 | Strict mode, declaration maps, selective any usage |
| Code Formatting | 9/10 | Biome consistent, 120 char line width |
| Error Handling | 7/10 | Try-catch patterns, but no custom error types |
| Logging | 5/10 | Ad-hoc console.log, mom has best pattern |
| Secrets Management | 8/10 | Env vars, no logging of secrets |
| CI/CD | 8/10 | Good coverage, cross-platform builds |
| Commit Quality | 8/10 | Conventional commits, issue references |

## Recommendations

### Strengths to Maintain
- Comprehensive provider test coverage with abort.test.ts pattern
- Strict TypeScript with declaration maps
- Biome + TypeScript + vitest consistently across packages
- Conventional commits with issue references

### Areas for Improvement
- Adopt structured logging (pino/winston) across packages
- Add tests for mom and pods packages
- Consider custom error classes for better error handling
- Add web UI component tests
- Implement browser E2E tests (Playwright/Cypress)
