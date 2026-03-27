# OpenCode Code Quality

## Overview

OpenCode maintains high code quality through comprehensive testing, typed schemas, functional error handling, and consistent formatting. The project uses the Effect framework for structured concurrency, Zod for runtime validation, and Bun's native test runner.

---

## Testing Infrastructure

### Test Frameworks

| Framework | Usage |
|-----------|-------|
| `bun:test` | Unit tests across all packages |
| `@playwright/test` | E2E tests for the app package |
| `effect/testing` | Effect-based service tests |

### Unit Test Distribution

**packages/opencode (core):**
- 60+ test files covering: bus events, config, file operations, git, formatting, effects, MCP, tools
- Uses Effect's `testEffect` helper for dependency injection
- Custom fixture system with `tmpdir()`, `provideInstance()`, `provideTmpdirInstance()`

**packages/app (SolidJS frontend):**
- 50+ test files covering: context, components, utils, pages
- Uses Happy DOM for DOM testing
- 60+ E2E specs with Playwright

### Effect Testing Pattern

```typescript
import { describe, expect } from "bun:test"
import { Effect, Layer } from "effect"
import { testEffect } from "../lib/effect"

const it = testEffect(Layer.mergeAll(Service.layer, NodeFileSystem.layer))

describe("Feature", () => {
  it.effect("description", () =>
    Effect.gen(function* () {
      const result = yield* service.method()
      expect(result).toEqual(expected)
    }),
  )
})
```

### E2E Configuration

```typescript
// packages/app/playwright.config.ts
export default defineConfig({
  testDir: "./e2e",
  timeout: 60_000,
  expect: { timeout: 10_000 },
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 5 : undefined,
})
```

---

## TypeScript Practices

### Configuration

Root `tsconfig.json` extends `@tsconfig/bun/tsconfig.json`:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json"
}
```

### Package-Level Variations

| Package | strict | noUncheckedIndexedAccess |
|---------|--------|--------------------------|
| `packages/opencode` | implicit | false |
| `packages/app` | implicit | false |
| `packages/github` | implicit | **true** |
| `packages/web` | implicit | false |

### Effect TypeScript Plugin

The `@effect/language-service` plugin is configured in `packages/app/tsconfig.json` to provide better IDE support for Effect's type system.

**Note:** Most packages do not have `strict: true` explicitly set, relying on Bun's defaults.

---

## Linting and Formatting

### EditorConfig

```ini
# .editorconfig
root = true
charset = utf-8
insert_final_newline = true
end_of_line = lf
indent_style = space
indent_size = 2
max_line_length = 80
```

### Prettier Configuration

```json
// package.json
{
  "prettier": {
    "semi": false,
    "printWidth": 120
  }
}
```

**Inconsistency:** `.editorconfig` specifies 80 character max line length, but Prettier uses 120. Prettier takes precedence for formatted files.

### Git Hooks

Pre-push hook in `.husky/pre-push`:

```bash
#!/bin/sh
set -e
bun -e '/* bun version check */'
bun typecheck
```

Enforces:
1. Bun version matches `packageManager` field
2. TypeScript type checking passes

---

## Error Handling Patterns

### Effect Framework

The project uses `Effect` extensively for typed error handling and structured concurrency:

```typescript
// Pattern 1: Effect.catch for error recovery
yield* effect.pipe(Effect.catch(() => Effect.void))

// Pattern 2: Effect.catchIf for conditional recovery
yield* write.pipe(
  Effect.catchIf(
    (e) => e.reason._tag === "NotFound",
    () => Effect.gen(function* () {
      yield* fs.makeDirectory(dirname(path), { recursive: true })
      yield* write
    }),
  ),
)

// Pattern 3: Effect.tryPromise for sync-to-async wrapping
const glob = Effect.fn("FileSystem.glob")(function* (pattern, options) {
  return yield* Effect.tryPromise({
    try: () => Glob.scan(pattern, options),
    catch: (cause) => new FileSystemError({ method: "glob", cause }),
  })
})
```

### NamedError Pattern

Custom error class with Zod schema support in `packages/util/src/error.ts`:

```typescript
export abstract class NamedError extends Error {
  abstract schema(): z.core.$ZodType
  abstract toObject(): { name: string; data: any }

  static create<Name extends string, Data extends z.core.$ZodType>(
    name: Name,
    data: Data
  ) {
    const schema = z.object({
      name: z.literal(name),
      data,
    }).meta({ ref: name })

    return class extends NamedError {
      public static readonly Schema = schema
      constructor(public readonly data: z.input<Data>, options?: ErrorOptions) {
        super(name, options)
      }
    }
  }
}
```

### Error Cause Chaining

```typescript
throw new Error("Primary error", { cause: originalError })
```

The `Log.formatError()` method walks error causes up to 10 levels deep.

---

## Logging

### Custom Log Utility

```typescript
// packages/opencode/src/util/log.ts
export namespace Log {
  export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])

  export interface Logger {
    debug(message?: any, extra?: Record<string, any>): void
    info(message?: any, extra?: Record<string, any>): void
    error(message?: any, extra?: Record<string, any>): void
    warn(message?: any, extra?: Record<string, any>): void
    tag(key: string, value: string): Logger
    clone(): Logger
    time(message: string, extra?: Record<string, any>): { stop(): void }
  }
}
```

### Features

- **Structured tags:** Service name, directory, session ID
- **Timing:** `logger.time()` for operation duration tracking
- **Level filtering:** DEBUG < INFO < WARN < ERROR
- **File output:** Rotating log files in `Global.Path.log`
- **Error formatting:** Recursive cause chain formatting

### Usage Pattern

```typescript
const log = Log.create({ service: "config" })
log.debug("message", { key: "value" })

// Timed operation
using timer = log.time("file write")
// ... operation ...
timer.stop()
```

---

## Zod Schema Usage

### Schema Validation in Tools

Tools define parameters with Zod schemas that are validated before execution:

```typescript
// packages/opencode/src/tool/tool.ts
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info["init"]
): Info<Parameters, Result> {
  return {
    id,
    init: async (initCtx) => {
      const toolInfo = init instanceof Function ? await init(initCtx) : init
      toolInfo.execute = async (args, ctx) => {
        try {
          toolInfo.parameters.parse(args)  // Zod validation
        } catch (error) {
          if (error instanceof z.ZodError && toolInfo.formatValidationError) {
            throw new Error(toolInfo.formatValidationError(error), { cause: error })
          }
        }
      }
      return toolInfo
    },
  }
}
```

### Bus Event Schema

```typescript
// packages/opencode/src/bus/bus-event.ts
export namespace BusEvent {
  export function define<Type extends string, Properties extends ZodType>(
    type: Type,
    properties: Properties
  ) {
    return { type, properties }
  }
}
```

### Effect-Zod Bridge

```typescript
// packages/opencode/src/util/effect-zod.ts
export function zod<S extends Schema.Top>(schema: S): z.ZodType<Schema.Schema.Type<S>> {
  return walk(schema.ast) as z.ZodType<Schema.Schema.Type<S>>
}
```

Enables seamless conversion between Effect's `Schema` and Zod for interoperability.

---

## Configuration Management

### Config Precedence (low to high)

1. Remote `.well-known/opencode` (org defaults)
2. Global config `~/.config/opencode/opencode.json`
3. Custom config (`OPENCODE_CONFIG`)
4. Project config (`opencode.json`)
5. `.opencode` directories
6. Inline config (`OPENCODE_CONFIG_CONTENT`)
7. Managed config directory (enterprise)

### Managed Config Directories

```typescript
function systemManagedConfigDir(): string {
  switch (process.platform) {
    case "darwin": return "/Library/Application Support/opencode"
    case "win32": return path.join(process.env.ProgramData || "C:\\ProgramData", "opencode")
    default: return "/etc/opencode"
  }
}
```

---

## Commit Conventions

### Pattern: `type(scope): message (#issue)`

```
fix(opencode): image paste on Windows Terminal 1.25+
chore: update nix node_modules hashes
feat(core): initial implementation of syncing
test: restore 5 workers on Windows e2e
fix+refactor(mcp): lifecycle tests, cancelPending fix, Effect migration
```

### Type Prefixes

| Prefix | Usage |
|--------|-------|
| `fix` | Bug fixes |
| `feat` | New features |
| `chore` | Maintenance, deps, generate |
| `test` | Test changes |
| `wip` | Work in progress |
| `fix+refactor` | Combined fix and refactor |

---

## Strengths

1. **Comprehensive test coverage** - Unit tests with Effect infrastructure + E2E with Playwright
2. **Typed error handling** - Effect framework provides compile-time error tracking
3. **Schema-first design** - Zod schemas define tool parameters, events, and file types
4. **Consistent formatting** - Prettier with Husky pre-push enforcement
5. **Structured logging** - Custom Log utility with service tagging and timing
6. **Config layering** - Clear precedence order for configuration sources
7. **Effect-Zod bridge** - Seamless interoperability between Effect and Zod

---

## Areas for Improvement

1. **Strict TypeScript** - Most packages lack `strict: true` in tsconfig; relies on Bun defaults
2. **Index access safety** - `noUncheckedIndexedAccess` is `false` in most packages (only `github` has `true`)
3. **ESLint** - No ESLint configuration found; relies solely on Prettier and TypeScript
4. **Line length inconsistency** - `.editorconfig` says 80 chars but Prettier uses 120
5. **Coverage tooling** - No visible code coverage configuration
