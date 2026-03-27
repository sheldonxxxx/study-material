# Code Quality: QMD

**Repository:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Analyzed:** 2026-03-27

---

## Executive Summary

QMD demonstrates strong TypeScript practices with comprehensive test coverage and strict compiler settings. However, notable gaps exist: no ESLint/Prettier configuration, inconsistent runtime validation, and direct console usage for logging. The codebase would benefit from standard linting, formatting rules, and Zod schema validation for configuration.

**Overall Score: 5.9/10**

| Dimension | Score | Max |
|-----------|-------|-----|
| Test Coverage | 8 | 10 |
| TypeScript Strictness | 8 | 10 |
| Linting/Formatting | 3 | 10 |
| Error Handling | 6 | 10 |
| Config Validation | 5 | 10 |
| Logging | 4 | 10 |
| Type Safety | 7 | 10 |

---

## 1. Test Coverage and Quality

### Test Suite Structure

The project uses **Vitest** as its test runner with 16 test files totaling approximately 110KB of test code.

| Test File | Purpose | Lines |
|-----------|---------|-------|
| `store.test.ts` | Core database operations | ~3,500 |
| `cli.test.ts` | End-to-end CLI integration tests | ~1,600 |
| `mcp.test.ts` | MCP server functionality | ~1,200 |
| `sdk.test.ts` | SDK/API surface | ~1,300 |
| `llm.test.ts` | LLM abstraction layer | ~800 |
| `intent.test.ts` | Intent detection | ~600 |
| `eval.test.ts` | Evaluation harness | ~500 |
| `store.helpers.unit.test.ts` | Pure function unit tests | ~300 |
| `formatter.test.ts` | Output formatting | ~300 |
| `eval-bm25.test.ts` | BM25 algorithm | ~200 |
| `multi-collection-filter.test.ts` | Collection filtering | ~150 |
| `collections-config.test.ts` | Config parsing | ~80 |
| `structured-search.test.ts` | Structured query search | ~500 |

### Coverage Assessment

**Strengths:**
- Comprehensive integration tests covering CLI commands via process spawning
- Unit tests for pure functions (path utilities, chunking logic, breakpoints)
- LLM operations tested with real models (no mocking)
- Temporary database isolation per test via `INDEX_PATH` env var
- Proper test lifecycle: `beforeAll`, `afterAll`, `beforeEach`, `afterEach`

**Weaknesses:**
- No explicit coverage reporting (no `--coverage` flag in vitest config)
- `store.test.ts` is very large (114KB) - could benefit from splitting

**Test Quality Example:**
```typescript
// Proper test isolation pattern (store.helpers.unit.test.ts)
test("getDefaultDbPath throws in test mode without INDEX_PATH", () => {
  const originalIndexPath = process.env.INDEX_PATH;
  delete process.env.INDEX_PATH;

  expect(() => getDefaultDbPath()).toThrow("Database path not set");

  if (originalIndexPath) {
    process.env.INDEX_PATH = originalIndexPath;
  }
});
```

---

## 2. TypeScript Configuration

### tsconfig.json Analysis

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false
  }
}
```

### Strictness Assessment

| Flag | Enabled | Impact |
|------|--------|--------|
| `strict` | Yes | Enables strictNullChecks, strictFunctionTypes, etc. |
| `noUncheckedIndexedAccess` | Yes | Forces undefined checks on array/object access |
| `noImplicitOverride` | Yes | Prevents accidental method overriding |
| `noFallthroughCasesInSwitch` | Yes | Catches missing break statements |
| `verbatimModuleSyntax` | Yes | Enforces explicit type exports |
| `noUnusedLocals` | **No** | Disabled - allows unused variables |
| `noUnusedParameters` | **No** | Disabled - allows unused params |
| `noPropertyAccessFromIndexSignature` | **No** | Disabled - allows `obj.key` on index types |

### Type System Usage

**Patterns observed:**

1. **Interface-first design:**
```typescript
export interface Collection {
  path: string;
  pattern: string;
  ignore?: string[];
  context?: ContextMap;
}
```

2. **Discriminated unions for result types:**
```typescript
export type DocumentNotFound = {
  error: "not_found";
  query: string;
  similarFiles: string[];
};
```

3. **Generic type parameters:**
```typescript
export async function searchVec<T>(
  query: string,
  model: string,
  limit: number = 20,
  collectionName?: string,
  session?: ILLMSession
): Promise<SearchResult[]>
```

4. **Explicit return types on exported functions:**
```typescript
export function scanBreakPoints(text: string): BreakPoint[]
export function findCodeFences(text: string): CodeFenceRegion[]
```

---

## 3. Linting and Formatting

### Configuration Status

| Tool | Status |
|------|--------|
| ESLint | **Not configured** - no `.eslintrc*` files |
| Prettier | **Not configured** - no `.prettierrc*` files |
| TypeScript | Configured (tsconfig.json) |

### Impact

Without linting/formatting:
- No consistent code style enforcement
- No complexity limits
- No banned patterns
- No import sorting rules
- No unused variable detection at lint level (only TSC when flags enabled)

### Observations

- The codebase appears to follow a consistent style manually (4-space indentation, single quotes)
- No `import { a, b }` style sorting enforcement
- No max line length enforcement

---

## 4. Error Handling Patterns

### Observed Patterns

#### 1. Throwing Errors with Context
```typescript
// store.ts
throw new Error("resolve: at least one path segment is required");
throw new Error("sqlite-vec is not available. Vector operations require...");
throw new Error(`${name} must be a positive integer`);
```

#### 2. Result Objects with Error Fields
```typescript
// store.ts - discriminated unions
return { error: "not_found", query: filename, similarFiles: [] };

// Returns { docs: [], errors: string[] } pattern
return { docs: [], errors: [`No files matched pattern: ${pattern}`] };
```

#### 3. Try-Catch with Console Error
```typescript
// llm.ts
this.unloadIdleResources().catch(err => {
  console.error("Error unloading idle resources:", err);
});
```

#### 4. Silent Failure with Fallback (Problematic)
```typescript
// db.ts
try {
  BunDatabase.setCustomSQLite(p);
  break;
} catch {}

// store.ts
try { mkdirSync(qmdCacheDir, { recursive: true }); } catch { }
```

### Error Handling Assessment

| Pattern | Usage | Quality |
|---------|-------|---------|
| `throw new Error` | Prevalent | Good - descriptive messages |
| Result objects | Common | Good - avoids exceptions for expected cases |
| Silent catch | Occasional | **Problematic** - hides errors |
| console.error | ~15 occurrences | **Problematic** - not structured logging |

### Critical Issues

1. **Silent catch blocks hide errors:**
```typescript
try { mkdirSync(qmdCacheDir, { recursive: true }); } catch { }
```
This swallows all errors including permission issues.

2. **No structured logging:**
```typescript
console.error("Embedding error:", error);
console.error("Batch embedding error:", error);
```
No log levels, no structured metadata, no log aggregation support.

---

## 5. Configuration Validation

### Zod Usage

**Location:** Only in `src/mcp/server.ts`

Zod is in `dependencies` but only imported in one file. No evidence of Zod schemas for:
- Collection configuration
- CLI arguments
- Environment variables
- API request/response validation

### Type-Only Validation

Most configuration relies on TypeScript types without runtime validation:

```typescript
// collections.ts - no Zod, just casting
const config = YAML.parse(content) as CollectionConfig;

// store.ts - JSON.parse with any
const parsed = JSON.parse(cached) as any[];
```

### Validation Gaps

| Configuration | Validation Method | Risk |
|---------------|-------------------|------|
| YAML config | `as CollectionConfig` | No runtime checks |
| JSON cache | `as any[]` | No schema validation |
| Env vars | Manual parsing | Inconsistent |
| CLI args | Not validated | High risk |

### Environment Variable Handling (Good Pattern)

```typescript
// llm.ts - Example of good validation
function resolveExpandContextSize(configValue?: number): number {
  if (configValue !== undefined) {
    const parsed = Number(configValue);
    if (!Number.isInteger(parsed) || parsed <= 0) {
      console.warn(`QMD Warning: invalid QMD_EXPAND_CONTEXT_SIZE="${configValue}"...`);
      return DEFAULT_EXPAND_CONTEXT_SIZE;
    }
    return parsed;
  }
  // ...
}
```

This pattern validates and warns - good approach, but not consistently applied.

---

## 6. Logging Practices

### Current State

```typescript
// Direct console usage throughout codebase
console.log(`${c.bold}QMD Status${c.reset}\n`);
console.error("Error unloading idle resources:", err);
console.warn(`⚠ Text truncated to fit embedding context...`);
```

### Issues

1. **No log levels** - `console.log` used for progress, `console.error` for errors
2. **No structured logging** - Plain text, not JSON
3. **No log destinations** - Always stdout/stderr
4. **No filtering** - No way to control verbosity
5. **No correlation IDs** - Hard to trace requests

### Positive Patterns

- Uses `console.warn` for recoverable issues (truncation warnings)
- Uses `console.error` for actual errors
- Color codes in CLI output (`c.bold`, `c.green`, etc.)

---

## 7. Type Safety Practices

### Explicit vs Inferred Types

**Good patterns:**
```typescript
// Explicit return type
export function scanBreakPoints(text: string): BreakPoint[]

// Explicit parameter types
export async function embedBatch(texts: string[]): Promise<(EmbeddingResult | null)[]>

// Interface definitions
export interface Collection {
  path: string;
  pattern: string;
  context?: ContextMap;
}
```

**Problematic patterns:**
```typescript
// JSON.parse with any
const parsed = JSON.parse(cached) as any[];

// Non-null assertion
const pos = match.index!;  // Uses ! operator to assert
```

### Null/Undefined Handling

```typescript
// Good - optional chaining
this.embedContexts.length > 0

// Problematic - non-null assertion
const pos = match.index!;
```

---

## 8. Summary: Areas of Strength and Weakness

### Strengths

1. **Comprehensive test coverage** - 16 test files with good isolation
2. **Strict TypeScript configuration** - Core strict flags enabled
3. **Interface-first design** - Clear type boundaries
4. **Discriminated unions** - Good error result typing
5. **Documentation** - JSDoc comments on exported functions
6. **Test isolation** - Proper use of temp directories, env var override

### Weaknesses

1. **No ESLint/Prettier** - Major tooling gap
2. **Silent catch blocks** - Errors hidden silently
3. **No structured logging** - Console-only output
4. **Minimal runtime validation** - No Zod for config
5. **Disabled strict flags** - `noUnusedLocals`, `noUnusedParameters`, `noPropertyAccessFromIndexSignature`
6. **`any[]` casts** - Type safety bypasses
7. **Non-null assertions** - `match.index!` pattern

### Recommendations

| Priority | Recommendation |
|----------|----------------|
| High | Add ESLint + Prettier configuration |
| High | Enable `noUnusedLocals` and `noUnusedParameters` |
| High | Replace silent catch blocks with proper error handling |
| Medium | Add Zod schemas for CollectionConfig |
| Medium | Replace console.* with structured logger |
| Medium | Enable `noPropertyAccessFromIndexSignature` |
| Low | Add test coverage reporting |
| Low | Parameterize large test files |
