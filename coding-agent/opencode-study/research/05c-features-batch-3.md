# Features 7-9 Deep Dive: Client/Server Architecture, VS Code Extension, SDK

## Feature 7: Client/Server Architecture

### Overview
OpenCode's client/server architecture enables distributed operation where the server runs on one machine while controlled remotely. This supports mobile app clients and remote development scenarios.

### Core Implementation

**Main Server Entry Point**
`packages/opencode/src/server/server.ts`

The server is built with **Hono** (a lightweight web framework similar to Express) and runs on Bun. Key architectural decisions:

```typescript
// Line 38: WebSocket support via Hono/Bun
import { websocket } from "hono/bun"

// Line 63-66: App creation with lazy initialization
export const createApp = (opts: { cors?: string[] }): Hono => {
  const app = new Hono()
  return app
}

// Lines 574-618: Server listen with graceful port fallback
const tryServe = (port: number) => {
  try {
    return Bun.serve({ ...args, port })
  } catch {
    return undefined
  }
}
const server = opts.port === 0 ? (tryServe(4096) ?? tryServe(0)) : tryServe(opts.port)
```

**Server Route Structure** (lines 250-261)
```
/project  - ProjectRoutes()
/pty      - PtyRoutes()
/config   - ConfigRoutes()
/experimental - ExperimentalRoutes()
/session  - SessionRoutes()
/permission - PermissionRoutes()
/question - QuestionRoutes()
/provider - ProviderRoutes()
/         - FileRoutes(), EventRoutes()
/mcp      - McpRoutes()
/tui      - TuiRoutes()
```

### Key Architectural Patterns

**1. mDNS Discovery** (`packages/opencode/src/server/mdns.ts`)
Enables automatic service discovery on local networks:
```typescript
const service = bonjour.publish({
  name: `opencode-${port}`,
  type: "http",
  host: domain ?? "opencode.local",
  port,
  txt: { path: "/" },
})
```

**2. Event Bus Architecture** (`packages/opencode/src/bus/index.ts`)
Uses the **Effect** framework for pub/sub messaging:
```typescript
export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(
    def: D,
    properties: z.output<D["properties"]>,
  ) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
  readonly subscribeAll: () => Stream.Stream<Payload>
}
```
The bus integrates with `GlobalBus.emit()` for cross-instance events.

**3. Session-Based Request Routing**
```typescript
// Lines 200-226: Middleware extracts workspace/directory context
const rawWorkspaceID = c.req.query("workspace") || c.req.header("x-opencode-workspace")
const raw = c.req.query("directory") || c.req.header("x-opencode-directory") || process.cwd()

return WorkspaceContext.provide({
  workspaceID: rawWorkspaceID ? WorkspaceID.make(rawWorkspaceID) : undefined,
  async fn() {
    return Instance.provide({
      directory,
      init: InstanceBootstrap,
      async fn() { return next() },
    })
  },
})
```

**4. Projectors for Event Transformation** (`packages/opencode/src/server/projectors.ts`)
Initializes event projectors that transform session events into sync-friendly formats:
```typescript
SyncEvent.init({
  projectors: sessionProjectors,
  convertEvent: (type, data) => {
    if (type === "session.updated") {
      const row = Database.use((db) => db.select().from(SessionTable)...).get()
      return { sessionID: id, info: Session.fromRow(row) }
    }
    return data
  },
})
```

### TUI Remote Control Routes (`packages/opencode/src/server/routes/tui.ts`)
The server exposes rich TUI control endpoints:
- `POST /tui/append-prompt` - Append text to prompt
- `POST /tui/submit-prompt` - Submit prompt for processing
- `POST /tui/execute-command` - Execute TUI commands (agent_cycle, session_new, etc.)
- `POST /tui/publish` - Publish TUI events
- `POST /tui/select-session` - Navigate to specific session

**Clever Pattern - AsyncQueue for Request/Response** (lines 18-28)
```typescript
const request = new AsyncQueue<TuiRequest>()
const response = new AsyncQueue<any>()

export async function callTui(ctx: Context) {
  const body = await ctx.req.json()
  request.push({ path: ctx.req.path, body })
  return response.next()
}
```
Uses async queues to bridge HTTP requests with internal TUI processing.

### Security Features
- Basic auth support (lines 89-92):
```typescript
if (!password) return next()
const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
return basicAuth({ username, password })(c, next)
```
- CORS with strict origin validation (lines 111-136)
- Directory path encoding for non-ASCII paths

### Serve CLI Command (`packages/opencode/src/cli/cmd/serve.ts`)
```typescript
export const ServeCommand = cmd({
  command: "serve",
  describe: "starts a headless opencode server",
  handler: async (args) => {
    const opts = await resolveNetworkOptions(args)
    const server = Server.listen(opts)
    console.log(`opencode server listening on http://${server.hostname}:${server.port}`)
    await new Promise(() => {})  // Keep alive
  },
})
```

---

## Feature 8: VS Code Extension

### Overview
A lightweight VS Code extension that integrates OpenCode as a terminal-based interface within the IDE.

### Implementation Location
`sdks/vscode/src/extension.ts`

### Architecture

The extension is remarkably simple (~137 lines) - it leverages VS Code's Terminal API to spawn the OpenCode CLI:

```typescript
// Line 45-91: Core terminal creation
async function openTerminal() {
  const port = Math.floor(Math.random() * (65535 - 16384 + 1)) + 16384
  const terminal = vscode.window.createTerminal({
    name: TERMINAL_NAME,
    env: {
      _EXTENSION_OPENCODE_PORT: port.toString(),
      OPENCODE_CALLER: "vscode",
    },
  })
  terminal.show()
  terminal.sendText(`opencode --port ${port}`)
  // Polls for server readiness
  let tries = 10
  do {
    await new Promise((resolve) => setTimeout(resolve, 200))
    try {
      await fetch(`http://localhost:${port}/app`)
      connected = true
      break
    } catch (e) {}
    tries--
  } while (tries > 0)
}
```

### Key Features

**1. File Path Insertion with Line Numbers** (`getActiveFile()`, lines 103-136)
```typescript
function getActiveFile() {
  const relativePath = vscode.workspace.asRelativePath(document.uri)
  let filepathWithAt = `@${relativePath}`

  // Check if there's a selection and add line numbers
  const selection = activeEditor.selection
  if (!selection.isEmpty) {
    const startLine = selection.start.line + 1
    const endLine = selection.end.line + 1
    if (startLine === endLine) {
      filepathWithAt += `#L${startLine}`
    } else {
      filepathWithAt += `#L${startLine}-${endLine}`
    }
  }
  return filepathWithAt
}
```

**2. Remote Prompt Appending** (lines 93-101)
```typescript
async function appendPrompt(port: number, text: string) {
  await fetch(`http://localhost:${port}/tui/append-prompt`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
  })
}
```

### VS Code Integration Points

**Commands** (`package.json`, lines 26-46):
- `opencode.openTerminal` - Open opencode in existing/new terminal
- `opencode.openNewTerminal` - Open in new tab
- `opencode.addFilepathToTerminal` - Insert @mentioned filepath

**Keybindings** (lines 56-80):
- `cmd+escape` - Open opencode
- `cmd+shift+escape` - Open in new tab
- `cmd+alt+k` - Insert @-mentioned filepath

**Menus** (lines 48-54):
- Editor title bar button for opening new terminal

### Technical Debt / Shortcuts

1. **Hardcoded port range**: Uses random port 16384-65535, assumes localhost
2. **No real error handling**: Fetch failures silently ignored in polling loop
3. **Simple terminal reuse check**: Only checks terminal name, not actual connection state
4. **No WebSocket support**: Relies on HTTP for communication

---

## Feature 9: SDK (`@opencode-ai/sdk`)

### Overview
Public SDK for building external tools and integrations. Auto-generated from OpenAPI spec.

### Package Structure
```
packages/sdk/js/
├── src/
│   ├── index.ts          # Main exports (v1)
│   ├── client.ts         # v1 client factory
│   ├── server.ts         # v1 server factory
│   ├── gen/              # Auto-generated v1 types/client
│   └── v2/              # Newer, cleaner API
│       ├── index.ts
│       ├── client.ts
│       ├── server.ts
│       └── gen/         # Auto-generated v2 types/client (100KB+ sdk.gen.ts)
├── package.json
└── script/build.ts      # SDK generation script
```

### SDK Generation Pipeline (`packages/sdk/js/script/build.ts`)

```typescript
// 1. Generate OpenAPI spec from running server
await $`bun dev generate > ${dir}/openapi.json`.cwd(path.resolve(dir, "../../opencode"))

// 2. Generate TypeScript client with openapi-ts
await createClient({
  input: "./openapi.json",
  output: { path: "./src/v2/gen", tsConfigPath: ..., clean: true },
  plugins: [
    { name: "@hey-api/typescript", exportFromIndex: false },
    { name: "@hey-api/sdk", instance: "OpencodeClient", exportFromIndex: false },
    { name: "@hey-api/client-fetch", baseUrl: "http://localhost:4096" },
  ],
})
```

### SDK Usage Patterns

**Client Creation** (`packages/sdk/js/src/v2/client.ts`):
```typescript
export function createOpencodeClient(config?: Config & { directory?: string }) {
  if (!config?.fetch) {
    const customFetch: any = (req: any) => {
      req.timeout = false  // Disable fetch timeout
      return fetch(req)
    }
    config = { ...config, fetch: customFetch }
  }

  if (config?.directory) {
    config.headers = {
      ...config.headers,
      "x-opencode-directory": encodeURIComponent(config.directory),
    }
  }

  const client = createClient(config)
  return new OpencodeClient({ client })
}
```

**Server Creation** (`packages/sdk/js/src/v2/server.ts`):
```typescript
export async function createOpencodeServer(options?: ServerOptions) {
  const args = [`serve`, `--hostname=${options.hostname}`, `--port=${options.port}`]
  const proc = spawn(`opencode`, args, {
    signal: options.signal,
    env: { ...process.env, OPENCODE_CONFIG_CONTENT: JSON.stringify(options.config ?? {}) },
  })

  // Parse URL from stdout
  const url = await new Promise<string>((resolve, reject) => {
    proc.stdout?.on("data", (chunk) => {
      const line = chunk.toString()
      if (line.startsWith("opencode server listening")) {
        const match = line.match(/on\s+(https?:\/\/[^\s]+)/)
        resolve(match[1])
      }
    })
  })

  return { url, close() { proc.kill() } }
}
```

**Unified Creation** (`packages/sdk/js/src/v2/index.ts`):
```typescript
export async function createOpencode(options?: ServerOptions) {
  const server = await createOpencodeServer({ ...options })
  const client = createOpencodeClient({ baseUrl: server.url })
  return { client, server }
}
```

### Key Design Decisions

1. **CLI as Server Process**: The SDK spawns the `opencode` CLI binary rather than importing the server module directly. This provides process isolation and matches end-user deployment.

2. **Config via Environment**: Configuration passed through `OPENCODE_CONFIG_CONTENT` env var containing JSON.

3. **Timeout Disabled**: Custom fetch wrapper removes timeout for long-running operations.

4. **Directory Encoding**: Handles non-ASCII paths with `encodeURIComponent()`.

5. **Base URL Resolution**: Waits for "opencode server listening on..." stdout message to determine server URL.

### Exports (`package.json`)
```json
{
  "exports": {
    ".": "./src/index.ts",
    "./client": "./src/client.ts",
    "./server": "./src/server.ts",
    "./v2": "./src/v2/index.ts",
    "./v2/client": "./src/v2/client.ts",
    "./v2/gen/client": "./src/v2/gen/client/index.ts",
    "./v2/server": "./src/v2/server.ts"
  }
}
```

---

## Technical Debt and Shortcuts Observed

| Feature | Issue | Location |
|---------|-------|----------|
| Server | Hardcoded CSP hash extraction from HTML | `server.ts:546-550` |
| Server | `@deprecated do not use this dumb shit` comment | `server.ts:571` |
| SDK | Timeout disabled globally via ts-ignore | `client.ts:12` |
| VS Code | No error handling in connection polling | `extension.ts:73-84` |
| VS Code | Hardcoded localhost assumption | `extension.ts:78` |
| Bus | Complex Effect/PubSub layering for simple pub/sub | `bus/index.ts` |

## Clever Solutions

1. **Port Fallback Strategy**: Server tries 4096 first, then 0 (random) if port 0 specified
2. **Lazy Route Initialization**: `lazy()` wrapper for route modules to avoid circular deps
3. **AsyncQueue Bridge**: Bridges sync TUI processing with async HTTP requests
4. **Event Projectors**: Decouple event producers from consumers via transformation layer
5. **Workspace Context Middleware**: Clean separation of workspace isolation per request
