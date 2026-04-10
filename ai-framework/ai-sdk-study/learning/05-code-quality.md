# Vercel AI SDK - Code Quality Practices

## TypeScript Configuration

### Shared Configuration Strategy

The project uses a centralized TypeScript configuration approach:

- **Base config**: `@vercel/ai-tsconfig` (workspace protocol) provides shared settings
- **Root config**: `tsconfig.json` contains project references with `strictNullChecks: true`
- **Package-level configs**: Each package extends the base with package-specific options

```json
{
  "extends": "./node_modules/@vercel/ai-tsconfig/base.json",
  "compilerOptions": {
    "target": "ES2018",
    "stripInternal": true,
    "lib": ["dom", "dom.iterable", "esnext"],
    "types": ["@types/node"],
    "composite": true
  }
}
```

### Key Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `strictNullChecks` | `true` | Strict null/undefined checking |
| `composite` | `true` | Enable project references for faster builds |
| `target` | `ES2018` | Broad Node.js compatibility |
| `stripInternal` | `true` | Hide internal API from consumers |

## Linting and Formatting

### Tooling: Ultracite

The project uses **Ultracite** (v7.3.2), a unified linting/formatting solution combining oxlint and oxfmt.

**Commands:**
- `pnpm check` - Run linting + formatting checks
- `pnpm fix` - Auto-fix issues

### Formatter Configuration

```jsonc
{
  "arrowParens": "avoid",
  "bracketSpacing": true,
  "endOfLine": "lf",
  "printWidth": 80,
  "singleQuote": true,
  "semi": true,
  "tabWidth": 2,
  "trailingComma": "all",
  "useTabs": false
}
```

### Linter Configuration

The linter extends three preset configurations (core, react, next) but disables many rules in a pragmatic trade-off prioritizing developer flexibility over strict style enforcement:

| Category | Disabled Rules |
|----------|-----------------|
| Code style | `complexity`, `prefer-const`, `prefer-template`, `no-plusplus` |
| Best practices | `no-empty-function`, `no-empty`, `no-alert`, `no-eval` |
| Imports | All `import/*` rules (no cycle checking) |
| React | `react/jsx-key`, `react/self-closing-comp` |
| TypeScript | `typescript/no-explicit-any`, `typescript/ban-ts-comment` |

**Rationale:** The team prioritizes business logic correctness over style dogma, trusting reviewers and automated tooling for egregious issues.

## Pre-Commit Hooks

### Husky Integration

The pre-commit hook (`ARTISANAL_MODE` bypass):

```bash
if [ -n "$ARTISANAL_MODE" ]; then
  exit 0  # Skip hooks when set
fi

# Auto-run pnpm install when package.json changes
if git diff --cached --name-only | grep -q 'package\.json$'; then
  pnpm install
  git add pnpm-lock.yaml
fi

pnpm lint-staged
```

**Notable features:**
- `ARTISANAL_MODE` bypass for certain workflows
- Automatic lockfile updates on dependency changes
- `lint-staged` integration for staged file fixing

## Testing Practices

### Framework: Vitest (v4.1.0)

### Test Organization Pattern

Tests are co-located with source files:

```
src/
  generate-text/
    generate-text.ts          # Implementation
    generate-text.test.ts     # Unit tests
    generate-text.test-d.ts   # Type tests
    generate-text-result.ts   # Related types
    index.ts                  # Barrel export
```

### Type Testing Pattern

Dedicated `.test-d.ts` files ensure type safety:

```typescript
import { describe, expectTypeOf, it } from 'vitest';

describe('generateText types', () => {
  it('should infer object output type', async () => {
    const result = await generateText({
      model: new MockLanguageModelV4(),
      prompt: 'Hello, world!',
      output: Output.object({ schema: z.object({ value: z.string() }) }),
    });

    expectTypeOf<typeof result.output>().toEqualTypeOf<{ value: string }>();
  });
});
```

### Dual Runtime Testing

Tests run on both Node.js and Edge runtimes:
- `test:node` - Node.js only
- `test:edge` - Edge runtime via `@edge-runtime/vm`

### Mock Utilities

Custom mock utilities for testing:
- `MockLanguageModelV4` - Language model interactions
- `MockTracer` - Telemetry span testing
- `mockId` from provider-utils

## Error Handling Patterns

### Error Hierarchy

All custom errors extend `AISDKError` with a marker symbol pattern:

```typescript
const name = 'AI_InvalidArgumentError';
const marker = `vercel.ai.error.${name}`;
const symbol = Symbol.for(marker);

export class InvalidArgumentError extends AISDKError {
  private readonly [symbol] = true;

  static isInstance(error: unknown): boolean {
    return AISDKError.hasMarker(error, marker);
  }
}
```

### Custom Error Types (21 total)

- `InvalidArgumentError` - Invalid function arguments
- `InvalidStreamPartError` - Malformed stream data
- `InvalidToolApprovalError` - Tool approval issues
- `NoObjectGeneratedError` - No object in response
- `RetryError` - Retry exhaustion
- `UnsupportedModelVersionError` - Model version mismatch

### Pattern Elements

1. **Marker symbol**: `Symbol.for()` for unique identification
2. **Private symbol property**: Prevents external tampering
3. **Static factory method**: `isInstance()` for safe type narrowing
4. **Structured data**: Errors carry contextual data

## Configuration Management

### Typed Configuration Objects

Settings are validated via dedicated validation functions:

```typescript
export type CallSettings = {
  maxOutputTokens?: number;
  temperature?: number;
  topP?: number;
  topK?: number;
  presencePenalty?: number;
  frequencyPenalty?: number;
  stopSequences?: string[];
  seed?: number;
  abortSignal?: AbortSignal;
  headers?: Record<string, string | undefined>;
};

export function prepareCallSettings({
  maxOutputTokens,
  temperature,
  // ...
}: Omit<CallSettings, 'abortSignal' | 'headers' | 'maxRetries'>): Omit<...> {
  if (maxOutputTokens != null) {
    if (!Number.isInteger(maxOutputTokens)) {
      throw new InvalidArgumentError({
        parameter: 'maxOutputTokens',
        value: maxOutputTokens,
        message: 'maxOutputTokens must be an integer',
      });
    }
  }
}
```

### Environment Variables

**No direct `process.env` usage** in source code. Instead:
- Typed configuration objects passed to functions
- Provider settings via constructor/invocation options
- Build-time environment variables handled by Turborepo

## Code Organization

### Directory Structure

Feature-based grouping with barrel exports:

```
packages/ai/src/
  agent/          # Agent functionality
  embed/          # Embedding functionality
  error/          # Error types
  generate-image/ # Image generation
  generate-object/# Object generation
  generate-text/  # Text generation
  logger/         # Logging
  middleware/     # Middleware functions
  prompt/         # Prompt utilities
  registry/       # Provider registry
  telemetry/      # OpenTelemetry integration
  types/          # Shared types
  ui/             # UI utilities
```

### Barrel Export Pattern

Each directory has an `index.ts` re-exporting public members:

```typescript
// error/index.ts
export { InvalidArgumentError } from './invalid-argument-error';
export { InvalidStreamPartError } from './invalid-stream-part-error';

// src/index.ts aggregates all domain exports
export * from './agent';
export * from './embed';
export * from './error';
```

## Quality Summary

| Aspect | Practice |
|--------|----------|
| **TypeScript** | Strict mode via shared config, project references, ES2018 target |
| **Linting** | Ultracite (oxlint), permissive rules prioritizing productivity |
| **Formatting** | oxfmt, 80-char width, single quotes, trailing commas |
| **Pre-commit** | Husky + lint-staged, auto pnpm install on package.json changes |
| **Testing** | Vitest, co-located tests, type tests via test-d.ts, dual runtime |
| **Error Handling** | Structured errors with marker pattern, 21+ custom types |
| **Configuration** | Typed objects with explicit validation, no direct env vars |
| **Organization** | Feature directories, barrel exports, co-located tests |

## Notable Strengths

1. **Comprehensive type testing**: Dedicated `.test-d.ts` files ensure type safety
2. **Dual runtime testing**: Node + Edge ensures broad compatibility
3. **Structured error system**: Consistent error patterns across codebase
4. **Well-documented configuration**: Typed configs with JSDoc comments
5. **Pragmatic linting**: Permissive rules avoid dogmatism while maintaining quality
6. **Clean organization**: Feature-based structure with clear barrel exports

## Potential Concerns

1. **Extensive rule disabling**: Many lint rules disabled may lead to inconsistent code style
2. **No pre-commit type checking**: Type errors only caught in CI/build
3. **ARTISANAL_MODE bypass**: Could accidentally skip quality gates
4. **No direct env var access pattern**: May complicate some deployment scenarios
