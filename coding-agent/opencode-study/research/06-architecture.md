# OpenCode Architecture and Design Patterns

## Overview

OpenCode is a **TypeScript monorepo** (Turborepo + Bun) implementing an AI-powered development tool. The primary deliverable is a CLI application that orchestrates LLM interactions with a rich tool system for file operations, code editing, and external integrations.

**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Main Package:** `packages/opencode/src/`

---

## Architectural Pattern: Layered Service-Oriented with Event Bus

OpenCode follows a **layered service-oriented architecture** with heavy use of the **Effect** functional programming framework for dependency injection and service management. Components communicate through:

1. **Event Bus** (pub/sub for in-process events)
2. **HTTP REST API** (client-server communication)
3. **Direct function calls** (within the same process via Effect services)

```
┌─────────────────────────────────────────────────────────────┐
│                      CLI Entry Point                         │
│                   (packages/opencode/src/index.ts)           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Server (Hono + Bun)                      │
│     Routes: /session, /project, /mcp, /config, /provider    │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │   Session    │  │    Tools     │  │     MCP      │
    │   (State)    │  │  (Executor)  │  │   (External) │
    └──────────────┘  └──────────────┘  └──────────────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              ▼
                    ┌──────────────────┐
                    │   SQLite DB     │
                    │  (via Drizzle)  │
                    └──────────────────┘
```

---

## Core Abstractions

### 1. Tool System (`src/tool/`)

**Interface Pattern: Strategy + Factory**

```typescript
// Tool.Info defines the contract
export namespace Tool {
  export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
    id: string
    init: (ctx?: InitContext) => Promise<{
      description: string
      parameters: Parameters
      execute(args: z.infer<Parameters>, ctx: Context): Promise<Result>
      formatValidationError?(error: z.ZodError): string
    }>
  }

  // Factory function for defining tools
  export function define<Parameters extends z.ZodType, Result extends Metadata>(
    id: string,
    init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
  ): Info<Parameters, Result>
}
```

**Key Tool Implementations:**
- `bash.ts` - Shell command execution with tree-sitter parsing
- `edit.ts` / `write.ts` - File editing
- `read.ts` / `glob.ts` / `grep.ts` - File reading and search
- `lsp.ts` - Language Server Protocol integration
- `webfetch.ts` / `websearch.ts` - Web access
- `task.ts` / `todowrite.ts` - Task management
- `apply_patch.ts` - Patch application

**Registry Pattern:**
```typescript
// ToolRegistry is a singleton service managing all tools
export namespace ToolRegistry {
  export interface Interface {
    readonly register: (tool: Tool.Info) => Effect.Effect<void>
    readonly ids: () => Effect.Effect<string[]>
    readonly tools: (model, agent?) => Effect.Effect<Tool.Info[]>
  }
}
```

### 2. Agent System (`src/agent/`)

**Strategy Pattern with Multiple Agent Types**

```typescript
export namespace Agent {
  export interface Info {
    name: string
    description: string
    mode: "subagent" | "primary" | "all"
    native: boolean
    permission: Permission.Ruleset
    model?: { providerID: ProviderID; modelID: ModelID }
    prompt?: string
    options: Record<string, any>
  }
}
```

**Built-in Agents:**
- `build` - Primary agent for task execution
- `plan` - Planning mode (no edit tools)
- `general` - General-purpose research agent
- `explore` - Fast codebase exploration
- `compaction` / `title` / `summary` - Hidden utility agents

### 3. Session System (`src/session/`)

**State Machine Pattern for Conversation State**

```typescript
export namespace Session {
  // Messages with parts (text, tool calls, reasoning, etc.)
  export const Message = {
    User: z.object({ role: "user", ... }),
    Assistant: z.object({ role: "assistant", ... }),
  }

  export const Part = z.discriminatedUnion("type", [
    TextPart,           // Regular text output
    ReasoningPart,       // Chain-of-thought reasoning
    ToolPart,           // Tool calls and results
    StepStartPart,      // Step boundaries
    StepFinishPart,     // Step completion with usage
    SnapshotPart,       // Filesystem snapshots
    PatchPart,          // Git-like diffs
    ...
  ])
}
```

**Message Flow:**
1. User submits prompt via `/session/:id/message` POST
2. `SessionPrompt.prompt()` creates assistant message
3. `SessionProcessor.process()` streams LLM response
4. Tool calls are intercepted, executed, and results returned
5. Final response stored in SQLite with full audit trail

### 4. Provider System (`src/provider/`)

**Adapter Pattern for Multiple LLM Providers**

```typescript
export namespace Provider {
  // Bundled providers via @ai-sdk packages
  const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
    "@ai-sdk/amazon-bedrock": createAmazonBedrock,
    "@ai-sdk/anthropic": createAnthropic,
    "@ai-sdk/openai": createOpenAI,
    "@openrouter/ai-sdk-provider": createOpenRouter,
    // ... 15+ providers
  }
}
```

---

## Key Design Patterns in Use

### 1. Service Map Pattern (Effect Framework)

OpenCode uses the **Effect** library's Service pattern for dependency injection:

```typescript
// Define a service interface
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

// Create layer with implementation
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    // ... implementation
    return Service.of({ get, list, defaultAgent })
  })
)

// Use in other services
const agent = yield* Agent.Service
```

**Scoped Cache Pattern for Instance State:**
```typescript
export namespace InstanceState {
  export const make = <A>(
    init: (ctx: Shape) => Effect.Effect<A>
  ): Effect.Effect<InstanceState<A>> =>
    Effect.gen(function* () {
      const cache = yield* ScopedCache.make({ capacity: Infinity, lookup: () => init(Instance.current) })
      // ... cleanup on dispose
      return { [TypeId]: TypeId, cache }
    })

  export const get = <A>(self: InstanceState<A>) =>
    Effect.suspend(() => ScopedCache.get(self.cache, Instance.directory))
}
```

This enables **per-instance singletons** - each working directory gets its own isolated state.

### 2. Observer Pattern via Event Bus

```typescript
export namespace Bus {
  // Define event types
  export const InstanceDisposed = BusEvent.define("server.instance.disposed", z.object({ directory: z.string() }))

  // Pub/Sub interface
  export interface Interface {
    readonly publish: <D extends BusEvent.Definition>(def: D, properties: ...) => Effect.Effect<void>
    readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
    readonly subscribeAll: () => Stream.Stream<Payload>
  }
}
```

**Usage:**
- Session events (message updated, part delta, errors)
- MCP tools changed notifications
- TUI events (toast show, refresh)
- Plugin lifecycle events

### 3. Factory Pattern for Tools

Tools are defined via the `Tool.define()` factory:

```typescript
export const BashTool = Tool.define("bash", async () => ({
  description: DESCRIPTION,
  parameters: z.object({
    command: z.string().describe("The command to execute"),
    timeout: z.number().optional(),
    workdir: z.string().optional(),
  }),
  async execute(params, ctx) {
    // Implementation
    return { title: "...", output: "...", metadata: {} }
  }
}))
```

### 4. Strategy Pattern for Permissions

```typescript
export namespace Permission {
  export interface Ruleset {
    some(action: string, patterns: string[]): Promise<"allow" | "deny" | "ask">
  }

  // Merged from defaults + user config + agent config
  export function merge(...rulesets: Ruleset[]): Ruleset
  export function fromConfig(config: Record<string, "allow" | "deny" | "ask">): Ruleset
}
```

### 5. Repository Pattern for Data Access

```typescript
export namespace Session {
  export const page = fn(
    z.object({ sessionID, limit, before }),
    async (input) => {
      const rows = Database.use((db) =>
        db.select().from(MessageTable).where(...).orderBy(...).limit(...).all()
      )
      return hydrate(rows) // Map DB rows to domain objects
    }
  )
}
```

### 6. Middleware Pattern for HTTP Routes

```typescript
// Workspace context middleware
app.use(async (c, next) => {
  const directory = c.req.query("directory") || c.req.header("x-opencode-directory")
  return WorkspaceContext.provide({
    workspaceID: rawWorkspaceID ? WorkspaceID.make(rawWorkspaceID) : undefined,
    fn() {
      return Instance.provide({ directory, init: InstanceBootstrap, fn: next })
    }
  })
})
```

---

## Component Communication

### 1. In-Process: Effect Services + Event Bus

```
Tool execution:
  SessionProcessor → ToolRegistry.tools() → Tool.define().execute()
                              ↓
                    Permission.ask() if needed
                              ↓
                    Return result to LLM stream

Event publishing:
  Tool error → Bus.publish(Session.Event.Error, { sessionID, error })
                    ↓
  Plugin subscribers notified via Bus.subscribeAll()
```

### 2. Client-Server: HTTP REST API

The CLI can run in **server mode** (`opencode serve`) exposing HTTP endpoints:

```
Client (CLI, Desktop, Web)
    │
    ▼ HTTP/WebSocket
Server (packages/opencode/src/server/server.ts)
    │
    ├── Hono routes for session management
    ├── Tool execution (via tool registry)
    ├── MCP server connections
    └── SQLite database
```

**Key Routes:**
- `POST /session/:id/message` - Send message, stream response
- `GET /session/:id/message` - Get messages with pagination
- `POST /session/:id/init` - Initialize project with AGENTS.md
- `GET /project/*` - Project operations
- `POST /mcp/*` - MCP server management

### 3. MCP Integration (Model Context Protocol)

External tools exposed via MCP:

```typescript
export namespace MCP {
  // MCP tools converted to AI SDK tools
  function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient): Tool {
    return dynamicTool({
      description: mcpTool.description,
      inputSchema: jsonSchema(mcpTool.inputSchema),
      execute: async (args) => client.callTool({ name: mcpTool.name, arguments: args })
    })
  }
}
```

**Transport Types:**
- `local` - Stdio-based subprocess
- `remote` - StreamableHTTP or SSE with OAuth support

---

## Module Map and Responsibilities

| Module | Responsibility | Key Types |
|--------|----------------|-----------|
| `src/agent/` | Agent definitions and selection | `Agent.Info`, `Agent.Service` |
| `src/tool/` | Tool definitions and registry | `Tool.Info`, `ToolRegistry` |
| `src/session/` | Conversation state and persistence | `Session.*`, `MessageV2.*` |
| `src/provider/` | LLM provider abstraction | `Provider.Model`, `Provider.Service` |
| `src/bus/` | Event pub/sub system | `Bus.Event`, `Bus.publish/subscribe` |
| `src/server/` | HTTP API server | `Server.createApp()`, routes |
| `src/mcp/` | MCP client management | `MCP.Client`, `MCP.Tools` |
| `src/plugin/` | Plugin hooks and loading | `Plugin.Service`, `Hooks` |
| `src/config/` | Configuration management | `Config.Info`, `Config.get()` |
| `src/project/` | Project context and bootstrap | `Instance`, `Project` |
| `src/storage/` | SQLite database access | `Database`, `Table` |
| `src/permission/` | Permission checking | `Permission.Ruleset` |

---

## Client-Server Architecture

### CLI Mode (Embedded)

When running `opencode run`, the server is embedded in the same process:

```
┌─────────────────────────────────────────────┐
│  opencode run (single process)              │
│  ├── CLI argument parsing                   │
│  ├── Instance.provide()                    │
│  └── Server.listen() on port 4096          │
└─────────────────────────────────────────────┘
```

### Server Mode (Network)

When running `opencode serve`:

```
┌──────────────┐         ┌──────────────────┐
│  CLI Client  │ ──────▶ │  Server Process  │
│  (connects)  │ ◀────── │  (port 4096)     │
└──────────────┘         └──────────────────┘
```

### Desktop/Web Clients

Multiple UI entry points connect to the server:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Desktop   │  │    Web      │  │    App      │
│  (Tauri)    │  │  (Astro)    │  │ (SolidStart)│
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                 │                 │
       └────────────────┼─────────────────┘
                        ▼
              ┌──────────────────┐
              │   Server API     │
              │  (Hono + Bun)   │
              └──────────────────┘
```

---

## Design Pattern Summary

| Pattern | Location | Implementation |
|---------|----------|----------------|
| **Strategy** | Agent system | Multiple agent types with different permissions/prompts |
| **Factory** | Tool system | `Tool.define()` factory for creating tools |
| **Observer** | Event bus | `Bus.publish()` / `Bus.subscribe()` |
| **Repository** | Data access | `Session.page()`, `MessageV2.get()` |
| **Service Locator** | Effect | `yield* ServiceName.Service` |
| **Middleware** | HTTP | Hono `.use()` chains for auth, CORS, context |
| **Scoped Singleton** | Instance state | `InstanceState.make()` per working directory |
| **Adapter** | Providers | Unified interface over 15+ LLM providers |
| **Builder** | Tool execution | Fluent tool context with metadata, ask(), abort |

---

## Technology Stack

- **Runtime:** Bun
- **HTTP Framework:** Hono (with OpenAPI validation)
- **Database:** SQLite via Drizzle ORM
- **AI SDK:** Vercel AI SDK (`ai` package)
- **LLM Providers:** @ai-sdk/* (OpenAI, Anthropic, Google, etc.)
- **Functional Programming:** Effect framework
- **MCP:** @modelcontextprotocol/sdk
- **CLI:** Yargs
- **UI:** Solid.js (desktop/web app)
- **Desktop:** Tauri
