# Security and Performance Patterns - OpenCode

## Overview

OpenCode is an AI-powered coding assistant that runs locally on the user's machine. The project takes a pragmatic approach to security, explicitly acknowledging that the permission system is a UX feature rather than a security boundary.

---

## Security Patterns

### 1. Secrets Management

**Environment Variables via Flag System**

All secrets are managed through environment variables parsed via the `Flag` namespace in `packages/opencode/src/flag/flag.ts`:

```typescript
export const OPENCODE_SERVER_PASSWORD = process.env["OPENCODE_SERVER_PASSWORD"]
export const OPENCODE_SERVER_USERNAME = process.env["OPENCODE_SERVER_USERNAME"]
export const OPENCODE_DB = process.env["OPENCODE_DB"]
```

**File Permissions**

Sensitive files are written with restricted permissions (0o600):

```typescript
// packages/opencode/src/auth/index.ts:78
yield* Effect.tryPromise({
  try: () => Filesystem.writeJson(file, { ...data, [norm]: info }, 0o600),
})

// packages/opencode/src/mcp/auth.ts:82
yield* fs.writeJson(filepath, { ...data, [mcpName]: entry }, 0o600)
```

**Auth Storage**

Provider credentials stored in JSON files:
- `auth.json` - Provider API keys and OAuth tokens (`Global.Path.data`)
- `mcp-auth.json` - MCP server authentication (`Global.Path.data`)

### 2. Authentication & Authorization

**Server Mode Basic Auth**

HTTP Basic Auth is available for server mode when `OPENCODE_SERVER_PASSWORD` is set:

```typescript
// packages/opencode/src/server/server.ts:89-93
const password = Flag.OPENCODE_SERVER_PASSWORD
if (!password) return next()
const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
return basicAuth({ username, password })(c, next)
```

**OAuth + API Key Support**

The `Auth` namespace supports multiple authentication types:

```typescript
// packages/opencode/src/auth/index.ts
export class Oauth extends Schema.Class<Oauth>("OAuth")({
  type: Schema.Literal("oauth"),
  refresh: Schema.String,
  access: Schema.String,
  expires: Schema.Number,
})

export class Api extends Schema.Class<Api>("ApiAuth")({
  type: Schema.Literal("api"),
  key: Schema.String,
})
```

**MCP OAuth Authentication**

MCP servers support OAuth 2.0 with PKCE via `McpAuth` namespace (`packages/opencode/src/mcp/auth.ts`):
- Token storage with expiry tracking
- Code verifier for PKCE
- OAuth state for CSRF protection

**Provider Auth Hooks**

The `ProviderAuth` system (`packages/opencode/src/provider/auth.ts`) allows plugins to define custom auth methods:
- OAuth flows with prompts
- API key validation

### 3. Permission System

**Permission Model**

The permission system is explicitly a UX feature, not a security boundary:

```typescript
// packages/opencode/src/permission/index.ts
export const Action = z.enum(["allow", "deny", "ask"])
export const Rule = z.object({
  permission: z.string(),
  pattern: z.string(),
  action: Action,
})
```

**Permission Evaluation**

Rules are evaluated against patterns with wildcard matching:

```typescript
// packages/opencode/src/permission/evaluate.ts
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule
```

**File Path Boundaries**

The `assertExternalDirectory` function checks if file operations stay within project boundaries:

```typescript
// packages/opencode/src/tool/external-directory.ts
export async function assertExternalDirectory(ctx: Tool.Context, target?: string, options?: Options) {
  if (Instance.containsPath(target)) return
  // Prompts for external directory access
  await ctx.ask({
    permission: "external_directory",
    patterns: [glob],
  })
}
```

### 4. Input Validation & Sanitization

**Zod Schema Validation**

All tool parameters use Zod for validation:

```typescript
// packages/opencode/src/tool/write.ts
parameters: z.object({
  content: z.string().describe("The content to write to the file"),
  filePath: z.string().describe("The absolute path to the file"),
})
```

**Tool Execution Validation**

Tools are validated before execution:

```typescript
// packages/opencode/src/tool/tool.ts
toolInfo.execute = async (args, ctx) => {
  try {
    toolInfo.parameters.parse(args)
  } catch (error) {
    if (error instanceof z.ZodError && toolInfo.formatValidationError) {
      throw new Error(toolInfo.formatValidationError(error), { cause: error })
    }
  }
}
```

**Batch Tool Validation**

```typescript
// packages/opencode/src/tool/batch.ts:62
const validatedParams = tool.parameters.parse(call.parameters)
```

### 5. Content Security Policy

CSP headers are dynamically generated for the proxy endpoint:

```typescript
// packages/opencode/src/server/server.ts:53-54
const csp = (hash = "") =>
  `default-src 'self'; script-src 'self' 'wasm-unsafe-eval'${hash ? ` 'sha256-${hash}'` : ""}; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; media-src 'self' data:; connect-src 'self' data:`
```

### 6. CORS Configuration

CORS is restrictive, allowing only specific origins:

```typescript
// packages/opencode/src/server/server.ts:113-134
origin(input) {
  if (input.startsWith("http://localhost:")) return input
  if (input.startsWith("http://127.0.0.1:")) return input
  if (input === "tauri://localhost" || input === "http://tauri.localhost" || input === "https://tauri.localhost")
    return input
  if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) return input
  if (opts?.cors?.includes(input)) return input
  return
}
```

### 7. Session Sharing

Share URLs use a secret token:

```typescript
// packages/opencode/src/share/share.sql.ts
export const SessionShareTable = sqliteTable("session_share", {
  session_id: text().primaryKey().references(() => SessionTable.id, { onDelete: "cascade" }),
  id: text().notNull(),
  secret: text().notNull(),  // Secret for share URL
  url: text().notNull(),
})
```

---

## Performance Patterns

### 1. Caching

**Scoped Cache Utility**

Generic TTL + max entries cache with disposal callbacks:

```typescript
// packages/app/src/utils/scoped-cache.ts
export function createScopedCache<T>(createValue: (key: string) => T, options: ScopedCacheOptions<T> = {}) {
  const store = new Map<string, Entry<T>>()
  // TTL-based expiration
  // Max entries with LRU-style eviction
  // Dispose callbacks for cleanup
}
```

**Session Cache**

Client-side session data caching:

```typescript
// packages/app/src/context/global-sync/session-cache.ts
export const SESSION_CACHE_LIMIT = 40
type SessionCache = {
  session_status: Record<string, SessionStatus | undefined>
  session_diff: Record<string, FileDiff[] | undefined>
  todo: Record<string, Todo[] | undefined>
  message: Record<string, Message[] | undefined>
}
```

**Lazy Initialization**

Lazy loading pattern for expensive resources:

```typescript
// packages/opencode/src/util/lazy.ts
export function lazy<T>(fn: () => T) {
  let value: T | undefined
  let loaded = false
  const result = (): T => {
    if (loaded) return value as T
    value = fn()
    loaded = true
    return value as T
  }
  result.reset = () => { loaded = false; value = undefined }
  return result
}
```

### 2. Pagination & Limits

**Read Tool Limits**

File reads have explicit limits to prevent memory issues:

```typescript
// packages/opencode/src/tool/read.ts
const DEFAULT_READ_LIMIT = 2000
const MAX_LINE_LENGTH = 2000
const MAX_BYTES = 50 * 1024  // 50KB limit

const limit = params.limit ?? DEFAULT_READ_LIMIT
const start = offset - 1
const sliced = entries.slice(start, start + limit)
```

**Directory Listing Limits**

Directory reads also paginate results:

```typescript
const limit = params.limit ?? DEFAULT_READ_LIMIT
const sliced = entries.slice(start, start + limit)
const truncated = start + sliced.length < entries.length
```

### 3. Database Optimization

**SQLite Configuration**

```typescript
// packages/opencode/src/storage/db.ts
db.run("PRAGMA journal_mode = WAL")
db.run("PRAGMA synchronous = NORMAL")
db.run("PRAGMA busy_timeout = 5000")
db.run("PRAGMA cache_size = -64000")  // 64MB cache
db.run("PRAGMA foreign_keys = ON")
```

**Indexed Queries**

```typescript
// packages/opencode/src/session/session.sql.ts
export const SessionTable = sqliteTable("session", {
  // ...
}, (table) => [
  index("session_project_idx").on(table.project_id),
  index("session_workspace_idx").on(table.workspace_id),
  index("session_parent_idx").on(table.parent_id),
])

export const MessageTable = sqliteTable("message", {
  // ...
}, (table) => [
  index("message_session_time_created_id_idx").on(table.session_id, table.time_created, table.id),
])
```

### 4. Instance Management

**Per-Directory Instance Caching**

```typescript
// packages/opencode/src/project/instance.ts
const cache = new Map<string, Promise<Shape>>()

export const Instance = {
  async provide<R>(input: { directory: string; fn: () => R }): Promise<R> {
    const directory = Filesystem.resolve(input.directory)
    let existing = cache.get(directory)
    if (!existing) {
      existing = track(directory, boot({ directory, init: input.init }))
    }
    return context.provide(ctx, async () => input.fn())
  }
}
```

### 5. Effect System for Concurrency

Uses the `effect` library for structured concurrency:

```typescript
// packages/opencode/src/storage/db.ts
export namespace Database {
  export function transaction<T>(callback: (tx: TxOrDb) => T): T {
    return Client().transaction((tx) => {
      return ctx.provide({ tx, effects }, () => callback(tx))
    }, { behavior: options?.behavior })
  }

  export function effect(fn: () => any | Promise<any>) {
    try {
      ctx.use().effects.push(fn)
    } catch {
      fn()
    }
  }
}
```

### 6. Bus/PubSub for Decoupling

Event-based communication with typed events:

```typescript
// packages/opencode/src/bus/index.ts
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define("permission.replied", Reply),
}

export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(def: D, properties: z.output<D["properties"]>) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
}
```

---

## Security Considerations

### Threat Model (from SECURITY.md)

1. **No Sandbox** - OpenCode does not sandbox the agent; use Docker/VM for isolation
2. **Server Mode** - Opt-in HTTP Basic Auth when enabled via `OPENCODE_SERVER_PASSWORD`
3. **MCP Servers** - Outside trust boundary; users configure themselves
4. **LLM Provider Data** - Governed by provider policies

### Out of Scope

- Server access when server mode is enabled
- Sandbox escapes (no sandbox implemented)
- LLM provider data handling
- MCP server behavior
- Malicious config files (user-controlled)

---

## Database Schema (Drizzle ORM)

**Core Tables**

| Table | Purpose | Indexes |
|-------|---------|---------|
| `project` | Project metadata, worktree paths | - |
| `session` | Chat sessions | project_id, workspace_id, parent_id |
| `message` | Messages within sessions | session_id + time_created + id |
| `part` | Message parts (streaming) | message_id + id, session_id |
| `todo` | Task tracking | session_id |
| `permission` | Permission rulesets | project_id (primary) |
| `session_share` | Share URLs with secrets | session_id (primary) |

---

## Key Security Files

| File | Purpose |
|------|---------|
| `SECURITY.md` | Threat model and reporting |
| `packages/opencode/src/auth/index.ts` | Provider auth (OAuth, API keys) |
| `packages/opencode/src/mcp/auth.ts` | MCP OAuth tokens |
| `packages/opencode/src/permission/index.ts` | Permission evaluation |
| `packages/opencode/src/tool/external-directory.ts` | Path boundary checks |
| `packages/opencode/src/server/server.ts` | CSP, CORS, basic auth |

---

## Summary

OpenCode implements security through:
- **File permissions** (0o600 for secrets)
- **CORS restrictions** (localhost + opencode.ai domains)
- **CSP headers** (dynamic with hash)
- **Basic Auth** for server mode (opt-in)
- **Path boundary checks** (external directory prompts)
- **Input validation** (Zod schemas on all tools)
- **No encryption at rest** for local data (trusting local filesystem)

Performance is optimized through:
- **Scoped caches** with TTL and max entries
- **SQLite WAL mode** with optimized pragmas
- **Indexed database queries**
- **Lazy initialization** of resources
- **Effect-based concurrency** model
- **Pagination** on file/directory reads
