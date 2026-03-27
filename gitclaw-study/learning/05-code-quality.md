# Code Quality: GitClaw

## Language and Build System

- **TypeScript** with strict mode enabled
- **ESM** modules (`"type": "module"`)
- **Build:** `npm run build` compiles via `tsc` and copies static assets
- **Tests:** `npm test` runs Node.js native tests with `--experimental-strip-types`

## Type Safety

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "declaration": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### Runtime Validation

Uses `@sinclair/typebox` for runtime type validation in tool schemas:

```typescript
import { Type } from "@sinclair/typebox";

export const cliSchema = Type.Object({
  command: Type.String({ description: "Shell command to execute" }),
  timeout: Type.Optional(Type.Number({ description: "Timeout in seconds" })),
});
```

### Key Interfaces Exported

Public API surface in `exports.ts` exports types for:
- Query/Message types (`GCMessage`, `GCAssistantMessage`, etc.)
- Tool definitions (`GCToolDefinition`)
- Hooks (`GCHooks`, `GCHookResult`)
- Sandbox types
- Plugin types

## Testing

### Test Setup

```json
{
  "scripts": {
    "test": "node --test test/*.test.ts --experimental-strip-types"
  }
}
```

Uses Node.js native test runner with experimental strip-types flag.

### Test Files

Located in `test/` directory:
- `sdk.test.ts` -- SDK functionality tests
- `cli.ts`, `memory.ts`, `read.ts`, `write.ts` -- Tool implementation tests
- `sandbox-*.ts` -- Sandbox isolation tests
- `capture-photo.ts`, `task-tracker.ts`, `skill-learner.ts` -- Feature tests

### Test Coverage Approach

Tests verify core functionality:
- Query/response cycles
- Tool invocation and results
- Error handling
- Sandbox isolation

## Code Organization

### Single Responsibility

Files have narrow, focused purposes:

| File | Responsibility |
|------|----------------|
| `sdk.ts` | Core query execution, event streaming |
| `loader.ts` | Agent manifest loading and configuration |
| `hooks.ts` | Script-based hook execution |
| `sdk-hooks.ts` | Programmatic hook wrapping |
| `plugins.ts` | Plugin discovery and loading |
| `tool-loader.ts` | Declarative YAML tool loading |
| `compliance.ts` | Compliance validation |
| `audit.ts` | Audit logging |

### Module Boundaries

- `exports.ts` -- Clean public API surface
- Internal modules use `.js` extension for ESM compatibility
- Types explicitly exported for external consumption

## Constants and Limits

Defined in `tools/shared.ts`:

```typescript
export const MAX_OUTPUT = 100_000;  // ~100KB max output to LLM
export const MAX_LINES = 2000;
export const MAX_BYTES = 100_000;
export const DEFAULT_TIMEOUT = 120;
export const DEFAULT_MEMORY_PATH = "memory/MEMORY.md";
```

These prevent resource exhaustion in tool implementations.

## Error Handling

### Query Errors

Errors captured in catch block and emitted as system messages:

```typescript
} catch (err) {
  pushMsg({
    type: "system",
    subtype: "error",
    content: err.message,
  });
  channel.finish();
}
```

### Tool Result Errors

Errors surfaced via `isError` flag in tool results:

```typescript
pushMsg({
  type: "tool_result",
  toolCallId: event.toolCallId,
  toolName: event.toolName,
  content: text,
  isError: event.isError,
});
```

### Validation Warnings

Compliance module generates warnings for configuration issues:

```typescript
export function validateCompliance(manifest: AgentManifest): ComplianceWarning[] {
  // Returns array of rule violations with severity levels
}
```

## Async Patterns

### Channel-Based Streaming

Custom channel implementation for async message streaming:

```typescript
interface Channel<T> {
  push(v: T): void;
  finish(): void;
  pull(): Promise<IteratorResult<T>>;
}
```

### Async Generator Query

Returns `AsyncGenerator<GCMessage>` for streaming responses:

```typescript
export function query(options: QueryOptions): Query {
  // Returns object implementing AsyncGenerator protocol
}
```

## Development Workflow

From `CONTRIBUTING.md`:

1. Fork and clone the repository
2. Create feature branch from `main`
3. Make changes in `src/`
4. Run `npm run build` to verify compilation
5. Run `npm test` to ensure tests pass
6. Commit with clear message and open PR

## Code Style

- Strict TypeScript with no `any` types (except in dynamic plugin contexts)
- Explicit return types on exported functions
- JSDoc comments for complex logic
- Constants over magic numbers
- Descriptive variable names

## Dependency Management

Minimal dependencies:
- `pi-agent-core`, `pi-ai` -- Core agent functionality
- `typebox` -- Runtime types
- `js-yaml`, `yaml` -- Config parsing
- `ws` -- WebSocket support
- `node-cron` -- Scheduled tasks

Peer dependencies:
- `gitmachine` -- Optional sandbox support

Dev dependencies:
- `@types/*` packages for TypeScript support
- `typescript` itself
