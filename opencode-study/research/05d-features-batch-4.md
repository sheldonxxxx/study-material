# Feature Deep Dive: Web Dashboard, Enterprise Features, Plugin System

## Feature 10: Web Dashboard (SolidStart App)

### Overview
The Web Dashboard (`packages/app/`) is a SolidStart-based web application that provides a browser-based interface for OpenCode. It connects to a running OpenCode server and provides a full UI for interacting with sessions, files, and the agent.

### Architecture

**Tech Stack:**
- **Framework:** SolidStart (SolidJS meta-framework with file-based routing)
- **Styling:** Tailwind CSS
- **State Management:** SolidJS stores + TanStack Query
- **UI Components:** @opencode-ai/ui (shared component library)
- **Terminal:** ghostty-web (browser-based terminal emulator)

### Key Entry Points

**`packages/app/src/entry.tsx`** - Browser entry point that bootstraps the application:
```typescript
const getCurrentUrl = () => {
  if (location.hostname.includes("opencode.ai")) return "http://localhost:4096"
  if (import.meta.env.DEV)
    return `http://${import.meta.env.VITE_OPENCODE_SERVER_HOST ?? "localhost"}:${import.meta.env.VITE_OPENCODE_SERVER_PORT ?? "4096"}`
  return location.origin
}
```

The app auto-detects whether it's running locally (connects to localhost:4096) or in production (uses current origin as the server).

**`packages/app/src/app.tsx`** - Main application shell with:
- Provider hierarchy (ThemeProvider, I18nProvider, QueryProvider, DialogProvider, etc.)
- Router configuration with routes for `/` (home), `/:dir` (directory layout), `/session/:id` (individual sessions)
- ConnectionGate - handles server health checking before rendering UI
- ServerProvider - manages server connections (http, sidecar, ssh)

### Core Contexts

**ServerConnection (`packages/app/src/context/server.tsx`):**
- Manages multiple server connections (http, sidecar, SSH)
- Health polling every 10 seconds
- Project tracking per server
- Supports localhost auto-detection

**File Context (`packages/app/src/context/file.tsx`):**
- File tree management
- File content viewing/editing with LSP integration
- Selection tracking for code highlighting

**Terminal Context (`packages/app/src/context/terminal.tsx`):**
- PTY session management
- Terminal output streaming via WebSocket

### Session Page (`packages/app/src/pages/session.tsx` - 61,487 bytes)

The main session interface is a complex component handling:
- Message timeline with lazy loading (initial 10 turns, batch of 8)
- Session history window with prefetching
- File diff viewing
- Terminal panel integration
- Comment threads
- Session commands

### Notable Patterns

1. **Chunked History Loading:** Session history loads in batches to handle large sessions:
```typescript
const turnInit = 10
const turnBatch = 8
const turnScrollThreshold = 200
const turnPrefetchBuffer = 16
```

2. **Server Health Checks:** Before rendering, the app pings the server:
```typescript
const res = yield* Effect.promise(() => checkServerHealth(http))
if (res.healthy) return true
if (checkMode() === "background" || type === "http") return false
```

3. **Platform Abstraction:** The `Platform` interface abstracts browser-specific functionality (notifications, links, navigation) for cross-platform compatibility.

### Clever Solutions

- **Ghostty Web Terminal:** Uses `ghostty-web` for a real terminal experience in the browser
- **History Window Optimization:** Bounded initial paint with smart prefetching as user scrolls
- **Multiple Server Support:** Can connect to multiple OpenCode servers and switch between them

### Technical Debt / Shortcuts

- Session page is 61KB+ - monolithic component that could benefit from splitting
- Complex session history window logic duplicated in some places
- No server-side rendering for initial load (SolidStart but client-heavy)

---

## Feature 12: Enterprise Features

### Overview
Enterprise features enable organizations to deploy OpenCode with authentication, access control, and organizational management. The enterprise tier uses Cloudflare infrastructure (Workers, R2, KV) for data residency and authentication.

### Key Packages

**`packages/enterprise/`** - SolidStart app for enterprise management
**`packages/identity/`** - Authentication/identity assets (logos only)
**`packages/opencode/src/auth/`** - Auth subsystem in core CLI

### Infrastructure (`infra/enterprise.ts`)

```typescript
const storage = new sst.cloudflare.Bucket("EnterpriseStorage")

const teams = new sst.cloudflare.x.SolidStart("Teams", {
  domain: shortDomain,
  path: "packages/enterprise",
  buildCommand: "bun run build:cloudflare",
  environment: {
    OPENCODE_STORAGE_ADAPTER: "r2",
    OPENCODE_STORAGE_ACCOUNT_ID: sst.cloudflare.DEFAULT_ACCOUNT_ID,
    OPENCODE_STORAGE_ACCESS_KEY_ID: SECRET.R2AccessKey.value,
    OPENCODE_STORAGE_SECRET_ACCESS_KEY: SECRET.R2SecretKey.value,
    OPENCODE_STORAGE_BUCKET: storage.name,
  },
})
```

### Auth Subsystem (`packages/opencode/src/auth/index.ts`)

The auth system uses Effect framework for service management with three auth types:

```typescript
export namespace Auth {
  export class Oauth extends Schema.Class<Oauth>("OAuth")({
    type: Schema.Literal("oauth"),
    refresh: Schema.String,
    access: Schema.String,
    expires: Schema.Number,
    accountId: Schema.optional(Schema.String),
    enterpriseUrl: Schema.optional(Schema.String),
  }) {}

  export class Api extends Schema.Class<Api>("ApiAuth")({
    type: Schema.Literal("api"),
    key: Schema.String,
  }) {}

  export class WellKnown extends Schema.Class<WellKnown>("WellKnownAuth")({
    type: Schema.Literal("wellknown"),
    key: Schema.String,
    token: Schema.String,
  }) {}
}
```

Auth data stored in `~/.opencode/auth.json` with file-based storage using proper permissions (0o600).

### Enterprise Storage (`packages/enterprise/src/core/storage.ts`)

Storage abstraction supporting both AWS S3 and Cloudflare R2:

```typescript
function s3(): Adapter {
  const bucket = process.env.OPENCODE_STORAGE_BUCKET!
  const client = new AwsClient({
    region: process.env.OPENCODE_STORAGE_REGION || "us-east-1",
    accessKeyId: process.env.OPENCODE_STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.OPENCODE_STORAGE_SECRET_ACCESS_KEY!,
  })
  return createAdapter(client, `https://s3.${region}.amazonaws.com`, bucket)
}

function r2() {
  const accountId = process.env.OPENCODE_STORAGE_ACCOUNT_ID!
  const client = new AwsClient({
    accessKeyId: process.env.OPENCODE_STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.OPENCODE_STORAGE_SECRET_ACCESS_KEY!,
  })
  return createAdapter(client, `https://${accountId}.r2.cloudflarestorage.com`, process.env.OPENCODE_STORAGE_BUCKET!)
}
```

### Enterprise Share System (`packages/enterprise/src/core/share.ts`)

Session sharing with secret-based access:
```typescript
export const create = fn(z.object({ sessionID: z.string() }), async (body) => {
  const isTest = process.env.NODE_ENV === "test" || body.sessionID.startsWith("test_")
  const info: Info = {
    id: (isTest ? "test_" : "") + body.sessionID.slice(-8),
    sessionID: body.sessionID,
    secret: crypto.randomUUID(),
  }
  // ...
})
```

Share data types include: session, message, part, session_diff, model

### Auth Plugin Integration

Enterprise features integrate via the plugin system with built-in auth plugins:

**CodexAuthPlugin (`packages/opencode/src/plugin/codex.ts`):**
- OAuth with PKCE for ChatGPT Pro/Plus authentication
- Built-in OAuth server on port 1455 for callback handling
- Two auth methods: browser-based and headless device flow
- Token refresh handling with automatic retry

**CopilotAuthPlugin (`packages/opencode/src/plugin/copilot.ts`):**
- GitHub Copilot authentication
- Device code OAuth flow with polling
- Enterprise GitHub (GHE) support via domain normalization

### Clever Solutions

1. **Multi-Storage Adapter:** Single interface supporting both S3 and R2 based on env var
2. **Effect-based Auth:** Clean service isolation with Effect framework for auth operations
3. **Share Snapshot Compaction:** Automatic compaction of share data to prevent unbounded growth

### Technical Debt

- Enterprise app routes are minimal placeholder (`src/routes/index.tsx` returns "Hello World")
- Stage/domain configuration spread across multiple files
- Storage path encoding uses `.json` extension on all keys, limiting flexibility

---

## Feature 14: Plugin System

### Overview
OpenCode has a comprehensive plugin system that allows extending functionality through hooks, custom tools, auth providers, and MCP server integration. The system supports both npm packages and local file-based plugins.

### Package Structure

**`packages/plugin/`** - Plugin SDK/package:
```
packages/plugin/
├── src/
│   ├── index.ts    # Main exports (Plugin type, Hooks interface)
│   ├── codex.ts    # Codex auth plugin implementation
│   └── copilot.ts  # Copilot auth plugin implementation
├── addons/         # Addon hooks
└── package.json    # @opencode-ai/plugin
```

### Core Plugin Interface (`packages/plugin/src/index.ts`)

```typescript
export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  serverUrl: URL
  $: BunShell
}

export type Plugin = (input: PluginInput) => Promise<Hooks>

export interface Hooks {
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>
  tool?: {
    [key: string]: ToolDefinition
  }
  auth?: AuthHook
  "chat.message"?: (input: {...}, output: { message: UserMessage; parts: Part[] }) => Promise<void>
  "chat.params"?: (input: {...}, output: {...}) => Promise<void>
  "chat.headers"?: (input: {...}, output: { headers: Record<string, string> }) => Promise<void>
  "permission.ask"?: (input: Permission, output: { status: "ask" | "deny" | "allow" }) => Promise<void>
  "command.execute.before"?: (input: {...}, output: { parts: Part[] }) => Promise<void>
  "tool.execute.before"?: (input: {...}, output: { args: any }) => Promise<void>
  "shell.env"?: (input: {...}, output: { env: Record<string, string> }) => Promise<void>
  "tool.execute.after"?: (input: {...}, output: {...}) => Promise<void>
  // ... more experimental hooks
}
```

### Plugin Loading (`packages/opencode/src/plugin/index.ts`)

```typescript
const INTERNAL_PLUGINS: PluginInstance[] = [CodexAuthPlugin, CopilotAuthPlugin, GitlabAuthPlugin, PoeAuthPlugin]

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    // ...
    for (const plugin of INTERNAL_PLUGINS) {
      log.info("loading internal plugin", { name: plugin.name })
      const init = await plugin(input).catch((err) => { /* error handling */ })
      if (init) hooks.push(init)
    }

    let plugins = cfg.plugin ?? []
    for (let plugin of plugins) {
      if (DEPRECATED_PLUGIN_PACKAGES.some((pkg) => plugin.includes(pkg))) continue
      // npm install or file:// import
      await import(plugin).then(async (mod) => {
        const seen = new Set<PluginInstance>()
        for (const [name, fn] of Object.entries<PluginInstance>(mod)) {
          if (seen.has(fn)) continue
          seen.add(fn)
          hooks.push(await fn(input))
        }
      })
    }
  })
)
```

Key features:
- **Duplicate prevention:** Uses Set to prevent double-initialization when plugins export both named and default exports
- **Deprecated package filtering:** Skips old npm package names that are now built-in
- **Async npm installation:** Uses `BunProc.install()` to install plugins from npm

### MCP Integration (`packages/opencode/src/mcp/index.ts`)

MCP (Model Context Protocol) support enables connecting to external MCP servers:

**Supported Transport Types:**
1. **Remote MCP Servers:**
   - StreamableHTTP (primary)
   - SSE (Server-Sent Events)
   - OAuth authentication support

2. **Local MCP Servers:**
   - StdioClientTransport for spawning local processes

**MCP Status Types:**
```typescript
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])
```

**Tool Conversion:**
```typescript
function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Tool {
  const inputSchema = mcpTool.inputSchema
  const schema: JSONSchema7 = {
    ...(inputSchema as JSONSchema7),
    type: "object",
    properties: (inputSchema.properties ?? {}) as JSONSchema7["properties"],
    additionalProperties: false,
  }

  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        { name: mcpTool.name, arguments: (args || {}) as Record<string, unknown> },
        CallToolResultSchema,
        { resetTimeoutOnProgress: true, timeout }
      )
    },
  })
}
```

### MCP OAuth (`packages/opencode/src/mcp/oauth-provider.ts`)

Full OAuth 2.0 implementation for MCP server authentication:
- Dynamic client registration support
- Token storage and refresh
- Code verifier management for PKCE
- State management for CSRF protection

### Auth Hook System

The plugin auth hook allows providers to define custom authentication flows:

```typescript
export type AuthHook = {
  provider: string
  loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
  methods: (
    | {
        type: "oauth"
        label: string
        prompts?: Array<{ type: "text" | "select"; key: string; message: string; ... }>
        authorize(inputs?: Record<string, string>): Promise<AuthOuathResult>
      }
    | {
        type: "api"
        label: string
        prompts?: Array<...>
        authorize?(inputs?: Record<string, string>): Promise<{ type: "success" | "failed" }>
      }
  )[]
}
```

### Clever Solutions

1. **Hook Trigger Pattern:** Generic `trigger<Name>()` method with type-safe hook names
2. **Transport Fallback:** Multiple transports tried in sequence (StreamableHTTP -> SSE) with graceful degradation
3. **OAuth State Machine:** Complex OAuth flows handled with clear state transitions
4. **Tool Name Sanitization:** MCP tools get client-prefixed names to avoid collisions:
```typescript
result[sanitizedClientName + "_" + sanitizedToolName] = convertMcpTool(...)
```

5. **Process Cleanup:** Proper subprocess tree termination on disconnect:
```typescript
const descendants = function* (pid: number) {
  // Recursively find all child PIDs
  // Then kill them on cleanup
}
```

### Technical Debt / Shortcuts

1. **OAuth Callback Port Fixed:** Uses hardcoded port 19876 - could conflict with other services
2. **No Plugin Unload:** Plugins are loaded but never unloaded (memory leak for dynamically updated plugins)
3. **MCP Timeout Handling:** Timeout logic relies on `resetTimeoutOnProgress` which may not work for all operations
4. **Credential Invalidation:** `invalidateCredentials()` modifies in-memory entry but uses `await` on potentially async operation without proper error handling

---

## Summary

### Web Dashboard
- **Architecture:** SolidStart app with complex session management
- **Connection:** WebSocket + HTTP to OpenCode server
- **Strengths:** Real terminal in browser (ghostty-web), smart history loading, multi-server support
- **Weaknesses:** Monolithic session component (61KB), client-heavy without SSR

### Enterprise Features
- **Architecture:** Cloudflare Workers/R2, SolidStart for management UI
- **Auth:** Effect-based service with file + cloud storage
- **Strengths:** Multi-cloud storage abstraction, clean auth type system
- **Weaknesses:** Minimal enterprise UI, storage path encoding limitations

### Plugin System
- **Architecture:** Hook-based extension with MCP integration
- **Plugins:** npm or file:// based, with internal auth plugins
- **Strengths:** Comprehensive hooks, OAuth support, transport fallback, type-safe triggers
- **Weaknesses:** No plugin unloading, fixed OAuth callback port, hardcoded timeouts