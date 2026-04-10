# Feature Deep Dive: Features 13-15

**Batch:** Git Integration, Cloud Infrastructure, Container Images
**Date:** 2026-03-26
**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`

---

## Feature 13: Slack Integration

**Priority:** Secondary
**Location:** `packages/slack/src/index.ts`

### Core Implementation

The Slack integration is a Slack App that bridges Slack messages with OpenCode sessions. It uses the `@slack/bolt` framework for Slack event handling and the `@opencode-ai/sdk` for communicating with the OpenCode server.

**Key File:** `packages/slack/src/index.ts`

```typescript
import { App } from "@slack/bolt"
import { createOpencode, type ToolPart } from "@opencode-ai/sdk"

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  socketMode: true,
  appToken: process.env.SLACK_APP_TOKEN,
})
```

### Architecture Pattern: Per-Thread Session

The integration creates a **separate OpenCode session per Slack thread**:

```typescript
const sessions = new Map<string, { client: any; server: any; sessionId: string; channel: string; thread: string }>()

app.message(async ({ message, say }) => {
  const channel = message.channel
  const thread = (message as any).thread_ts || message.ts
  const sessionKey = `${channel}-${thread}`

  let session = sessions.get(sessionKey)
  if (!session) {
    // Create new opencode session for this thread
    const createResult = await client.session.create({
      body: { title: `Slack thread ${thread}` },
    })
    session = { client, server, sessionId: createResult.data.id, channel, thread }
    sessions.set(sessionKey, session)
  }

  // Send message to opencode
  const result = await session.client.session.prompt({
    path: { id: session.sessionId },
    body: { parts: [{ type: "text", text: message.text }] },
  })
})
```

### Event Streaming for Tool Updates

The integration subscribes to OpenCode server events to post real-time tool execution updates back to Slack:

```typescript
const events = await opencode.client.event.subscribe()
for await (const event of events.stream) {
  if (event.type === "message.part.updated") {
    const part = event.properties.part
    if (part.type === "tool" && part.state.status === "completed") {
      const toolMessage = `*${part.tool}* - ${part.state.title}`
      await app.client.chat.postMessage({
        channel,
        thread_ts: thread,
        text: toolMessage,
      })
    }
  }
}
```

### Session Sharing

When a new session is created, it generates a shareable URL so users can view the session in OpenCode's web UI:

```typescript
const shareResult = await client.session.share({ path: { id: createResult.data.id } })
if (!shareResult.error && shareResult.data) {
  const sessionUrl = shareResult.data.share?.url
  await app.client.chat.postMessage({ channel, thread_ts: thread, text: sessionUrl })
}
```

### Clever Solutions

1. **Socket Mode**: Uses Slack's Socket Mode for webhook-free event handling
2. **Thread Affinity**: Each thread maps to a dedicated OpenCode session, preserving conversation context
3. **Tool Streaming**: Subscribes to server events and forwards tool completion notifications to Slack

### Technical Debt / Notes

- Environment variables required: `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`, `SLACK_APP_TOKEN`
- No explicit error recovery for session creation failures
- No cleanup of old sessions from the Map

---

## Feature 14: Plugin System

**Priority:** Secondary
**Locations:**
- `packages/plugin/src/index.ts` - Plugin SDK package
- `packages/plugin/src/tool.ts` - Tool definition helper
- `packages/plugin/src/shell.ts` - BunShell interface
- `packages/opencode/src/plugin/index.ts` - Plugin service integration
- `packages/opencode/src/mcp/index.ts` - MCP client implementation

### Plugin SDK (`packages/plugin/`)

The `@opencode-ai/plugin` package provides the foundation for building OpenCode plugins.

#### Tool Definition (`packages/plugin/src/tool.ts`)

Plugins can define custom tools using the `tool()` helper:

```typescript
export function tool<Args extends z.ZodRawShape>(input: {
  description: string
  args: Args
  execute(args: z.infer<z.ZodObject<Args>>, context: ToolContext): Promise<string>
}) {
  return input
}

export type ToolContext = {
  sessionID: string
  messageID: string
  agent: string
  directory: string
  worktree: string
  abort: AbortSignal
  metadata(input: { title?: string; metadata?: { [key: string]: any } }): void
  ask(input: AskInput): Promise<void>
}
```

#### Hooks Interface (`packages/plugin/src/index.ts`)

The Plugin interface returns a `Hooks` object with many extension points:

```typescript
export interface Hooks {
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>
  tool?: { [key: string]: ToolDefinition }
  auth?: AuthHook

  // Chat hooks
  "chat.message"?: (input, output) => Promise<void>
  "chat.params"?: (input, output) => Promise<void>
  "chat.headers"?: (input, output) => Promise<void>

  // Permission hooks
  "permission.ask"?: (input, output) => Promise<void>

  // Command hooks
  "command.execute.before"?: (input, output) => Promise<void>

  // Tool hooks
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  "tool.definition"?: (input, output) => Promise<void>

  // Shell hooks
  "shell.env"?: (input, output) => Promise<void>

  // Session hooks
  "experimental.session.compacting"?: (input, output) => Promise<void>

  // Text completion
  "experimental.text.complete"?: (input, output) => Promise<void>
}
```

### Plugin Service Integration (`packages/opencode/src/plugin/index.ts`)

The plugin system is integrated into OpenCode's core using Effect framework:

```typescript
export namespace Plugin {
  // Built-in internal plugins
  const INTERNAL_PLUGINS: PluginInstance[] = [
    CodexAuthPlugin,
    CopilotAuthPlugin,
    GitlabAuthPlugin,
    PoeAuthPlugin,
  ]

  // Deprecated npm packages to skip
  const DEPRECATED_PLUGIN_PACKAGES = [
    "opencode-openai-codex-auth",
    "opencode-copilot-auth"
  ]

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      // Loads internal plugins first
      // Then loads user plugins from config
      // Plugins are installed via Bun and imported dynamically
    })
  )
}
```

#### Plugin Loading Flow

1. Internal plugins loaded first (Codex, Copilot, GitLab, Poe)
2. User plugins from config are installed via `BunProc.install()`
3. Plugins imported via dynamic `import()` and deduplicated
4. Each plugin's hooks are registered with the bus event system

### MCP Implementation (`packages/opencode/src/mcp/index.ts`)

The MCP (Model Context Protocol) client is a sophisticated plugin system supporting multiple transport types:

#### Supported Transports

```typescript
// Remote MCP servers
type TransportWithAuth = StreamableHTTPClientTransport | SSEClientTransport

// Local MCP servers
new StdioClientTransport({
  command: cmd,
  args,
  cwd,
  env: { ...process.env, ...mcp.environment },
})
```

#### OAuth for Remote MCP Servers

```typescript
const authProvider = new McpOAuthProvider(
  key,
  mcp.url,
  {
    clientId: oauthConfig?.clientId,
    clientSecret: oauthConfig?.clientSecret,
    scope: oauthConfig?.scope,
  },
  {
    onRedirect: async (url) => { /* capture URL */ },
  },
)

const transport = new StreamableHTTPClientTransport(new URL(mcp.url), {
  authProvider,
  requestInit: mcp.headers ? { headers: mcp.headers } : undefined,
})
```

#### Tool Conversion

MCP tools are converted to AI SDK format:

```typescript
function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Tool {
  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        { name: mcpTool.name, arguments: args as Record<string, unknown> },
        CallToolResultSchema,
        { resetTimeoutOnProgress: true, timeout }
      )
    },
  })
}
```

### Built-in Auth Plugins

#### Codex Auth (`packages/opencode/src/plugin/codex.ts`)

Implements OAuth for OpenAI's Codex/ChatGPT integration:
- Browser-based OAuth flow with PKCE
- Device code flow for headless authentication
- Automatic token refresh
- Filters models to Codex-enabled ones only

#### Copilot Auth (`packages/opencode/src/plugin/copilot.ts`)

GitHub Copilot authentication:
- Device code OAuth flow
- Enterprise GitHub support via custom domain
- Sets appropriate headers (`x-initiator`, `Copilot-Vision-Request`)

### Clever Solutions

1. **Effect Framework Integration**: Plugin system uses Effect for clean dependency injection
2. **Deduplication**: `seen` Set prevents plugins from being initialized twice when exported both as named and default
3. **Transport Fallback**: Multiple transport types tried in sequence (StreamableHTTP, SSE)
4. **Deferred Client Installation**: Plugins with deprecated npm names are silently skipped
5. **Sanitized Names**: MCP client and tool names are sanitized to only contain `a-zA-Z0-9_-`

### Technical Debt / Notes

- Plugin loading errors are published to the session bus but don't fail the session
- No plugin hot-reload capability
- MCP client cleanup relies on process tree killing, platform-specific (`pgrep` on Unix)

---

## Feature 15: Database & Storage

**Priority:** Secondary
**Location:** `packages/opencode/src/storage/`

### Dual Storage Architecture

OpenCode maintains two parallel storage systems:
1. **SQLite** via Drizzle ORM - for persistent structured data
2. **JSON file storage** - legacy storage with migration path to SQLite

### SQLite Database (`packages/opencode/src/storage/db.ts`)

```typescript
export namespace Database {
  export function getChannelPath() {
    const channel = Installation.CHANNEL
    if (["latest", "beta"].includes(channel) || Flag.OPENCODE_DISABLE_CHANNEL_DB)
      return path.join(Global.Path.data, "opencode.db")
    const safe = channel.replace(/[^a-zA-Z0-9._-]/g, "-")
    return path.join(Global.Path.data, `opencode-${safe}.db`)
  }

  export const Client = lazy(() => {
    const db = init(Path)

    // Performance optimizations
    db.run("PRAGMA journal_mode = WAL")
    db.run("PRAGMA synchronous = NORMAL")
    db.run("PRAGMA busy_timeout = 5000")
    db.run("PRAGMA cache_size = -64000")
    db.run("PRAGMA foreign_keys = ON")
    db.run("PRAGMA wal_checkpoint(PASSIVE)")

    // Apply migrations
    migrate(db, entries)

    return db
  })
}
```

#### SQLite Pragmas for Performance

| Pragma | Value | Purpose |
|--------|-------|---------|
| `journal_mode` | WAL | Write-Ahead Logging for concurrent access |
| `synchronous` | NORMAL | Balanced durability/speed |
| `busy_timeout` | 5000 | 5 second wait for locks |
| `cache_size` | -64000 | 64MB cache |
| `foreign_keys` | ON | Enforce referential integrity |

### Database Schema (`packages/opencode/src/storage/schema.ts`)

The schema re-exports from various domain modules:

```typescript
export { AccountTable, AccountStateTable, ControlAccountTable } from "../account/account.sql"
export { ProjectTable } from "../project/project.sql"
export { SessionTable, MessageTable, PartTable, TodoTable, PermissionTable } from "../session/session.sql"
export { SessionShareTable } from "../share/share.sql"
export { WorkspaceTable } from "../control-plane/workspace.sql"
```

### JSON Storage (`packages/opencode/src/storage/storage.ts`)

The legacy JSON-based storage with file locking:

```typescript
export namespace Storage {
  export async function read<T>(key: string[]) {
    const dir = await state().then((x) => x.dir)
    const target = path.join(dir, ...key) + ".json"
    return withErrorHandling(async () => {
      using _ = await Lock.read(target)  // File locking
      const result = await Filesystem.readJson<T>(target)
      return result as T
    })
  }

  export async function write<T>(key: string[], content: T) {
    const dir = await state().then((x) => x.dir)
    const target = path.join(dir, ...key) + ".json"
    return withErrorHandling(async () => {
      using _ = await Lock.write(target)  // File locking
      await Filesystem.writeJson(target, content)
    })
  }
}
```

### JSON to SQLite Migration (`packages/opencode/src/storage/json-migration.ts`)

Comprehensive migration from JSON files to SQLite:

```typescript
export namespace JsonMigration {
  export async function run(sqlite: Database, options?: Options) {
    // Pre-scan all files upfront to avoid repeated glob operations
    const [projectFiles, sessionFiles, messageFiles, partFiles,
           todoFiles, permFiles, shareFiles] = await Promise.all([
      list("project/*.json"),
      list("session/*/*.json"),
      list("message/*/*.json"),
      list("part/*/*.json"),
      list("todo/*.json"),
      list("permission/*.json"),
      list("session_share/*.json"),
    ])

    // Batch processing with 1000 item batches
    const batchSize = 1000
    sqlite.exec("BEGIN TRANSACTION")

    // FK-aware ordering: projects first, then sessions, then messages, etc.
    // Orphan handling: skip items whose parents don't exist
    sqlite.exec("COMMIT")
  }
}
```

#### Migration Statistics Tracked

- `projects`, `sessions`, `messages`, `parts`, `todos`, `permissions`, `shares`
- Orphan counts for FK-violating items
- Error log with first 20 errors

### Transaction System

```typescript
export function transaction<T>(
  callback: (tx: TxOrDb) => NotPromise<T>,
  options?: { behavior?: "deferred" | "immediate" | "exclusive" },
): NotPromise<T> {
  return Client().transaction(
    (tx: TxOrDb) => {
      return ctx.provide({ tx, effects }, () => callback(tx))
    },
    { behavior: options?.behavior },
  )
}
```

SQLite transaction behaviors:
- **deferred** (default): Lock acquired on first statement
- **immediate**: Lock acquired immediately
- **exclusive**: Full database lock

### Context-based Transaction Propagation

Uses Effect's Context system for dependency injection:

```typescript
const ctx = Context.create<{
  tx: TxOrDb
  effects: (() => void | Promise<void>)[]
}>("database")

export function use<T>(callback: (trx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)
  } catch (err) {
    if (err instanceof Context.NotFound) {
      const effects: (() => void | Promise<void>)[] = []
      const result = ctx.provide({ effects, tx: Client() }, () => callback(Client()))
      for (const effect of effects) effect()
      return result
    }
    throw err
  }
}
```

### Clever Solutions

1. **Channel-based Database Isolation**: Different channels (beta, etc.) get separate DB files
2. **WAL Checkpointing**: Passive checkpoints avoid blocking writes
3. **Lazy Initialization**: Database client uses `lazy()` for deferred opening
4. **Migrations via VFS**: Migrations can be bundled in the binary via `OPENCODE_MIGRATIONS`
5. **File Locking**: JSON storage uses `Lock.read()` and `Lock.write()` for safe concurrent access
6. **Batch Processing**: JSON migration processes in 1000-item batches to manage memory

### Technical Debt / Notes

- No schema migrations for SQLite (schema defined via `.sql` files, not managed migrations)
- JSON storage still exists even after migration (no cleanup)
- No compaction or vacuum scheduling for SQLite
- `OPENCODE_SKIP_MIGRATIONS` flag exists but doesn't skip migration table writes

---

## Summary Table

| Feature | Primary Location | Key Pattern |
|---------|-----------------|-------------|
| Slack Integration | `packages/slack/src/index.ts` | Per-thread session mapping, event streaming |
| Plugin System | `packages/plugin/` + `packages/opencode/src/plugin/` | Hooks interface, Effect-based service |
| MCP Client | `packages/opencode/src/mcp/index.ts` | Transport abstraction, OAuth support |
| Database | `packages/opencode/src/storage/db.ts` | Drizzle ORM, WAL mode, lazy init |
| JSON Storage | `packages/opencode/src/storage/storage.ts` | File locking, key-path addressing |
| JSON Migration | `packages/opencode/src/storage/json-migration.ts` | Batch processing, FK-aware ordering |
