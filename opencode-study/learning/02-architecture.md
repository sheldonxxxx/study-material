# OpenCode Architecture

## Overview

OpenCode is a TypeScript monorepo (Turborepo + Bun) implementing an AI-powered development tool. The primary deliverable is a CLI application that orchestrates LLM interactions with a rich tool system for file operations, code editing, and external integrations.

**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Main Package:** `packages/opencode/src/`

---

## Architectural Pattern: Layered Service-Oriented with Event Bus

OpenCode follows a **layered service-oriented architecture** with heavy use of the **Effect** functional programming framework for dependency injection and service management. Components communicate through:

1. **Event Bus** (pub/sub for in-process events)
2. **HTTP REST API** (client-server communication)
3. **Direct function calls** (within the same process via Effect services)

```
CLI Entry Point
(packages/opencode/src/index.ts)
         │
         ▼
┌─────────────────────────────────┐
│      Server (Hono + Bun)        │
│ Routes: /session, /project,    │
│ /mcp, /config, /provider       │
└─────────────────────────────────┘
         │
   ┌─────┼─────┐
   ▼     ▼     ▼
┌──────┐┌──────┐┌──────┐
│Session││Tools ││  MCP │
│(State)││(Exec)││(Ext) │
└──────┘└──────┘└──────┘
         │
         ▼
┌──────────────────┐
│    SQLite DB    │
│  (via Drizzle)  │
└──────────────────┘
```

---

## Architectural Decisions

### 1. Effect Framework for Dependency Injection

OpenCode uses the **Effect** library as its primary dependency injection mechanism. This enables:

- **Referentially transparent** service access via `yield* ServiceName.Service`
- **Composable layers** that can be mocked, overridden, or extended
- **Scoped state** per working directory via `InstanceState.make()`
- **Error propagation** through the type system

```typescript
// Define a service interface
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

// Create layer with implementation
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    return Service.of({ get, list, defaultAgent })
  })
)
```

### 2. Service Map Pattern for Modular Services

Each major subsystem (Agent, Tool, Provider, Session, etc.) defines a Service interface and exports a Layer. Services are accessed via the Effect context, enabling easy testing and composition.

### 3. Per-Instance State Isolation

The `InstanceState` pattern ensures each working directory operates in isolation:

```typescript
export namespace InstanceState {
  export const make = <A>(
    init: (ctx: Shape) => Effect.Effect<A>
  ): Effect.Effect<InstanceState<A>> =>
    Effect.gen(function* () {
      const cache = yield* ScopedCache.make({
        capacity: Infinity,
        lookup: () => init(Instance.current)
      })
      return { [TypeId]: TypeId, cache }
    })

  export const get = <A>(self: InstanceState<A>) =>
    Effect.suspend(() => ScopedCache.get(self.cache, Instance.directory))
}
```

This enables **per-instance singletons** - each working directory gets its own isolated state.

### 4. SQLite for Local Persistence

All session data, messages, and project state persist to SQLite via Drizzle ORM:

```typescript
// Message persistence
const rows = Database.use((db) =>
  db.select().from(MessageTable).where(...).orderBy(...).limit(...).all()
)
return hydrate(rows) // Map DB rows to domain objects
```

### 5. Hono for HTTP Layer

The server uses Hono for its lightweight, fast HTTP framework with built-in OpenAPI validation support.

---

## Module Responsibilities

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

## Communication Patterns

### 1. In-Process: Effect Services + Event Bus

```
Tool execution:
  SessionProcessor → ToolRegistry.tools() → Tool.define().execute()
                              │
                    Permission.ask() if needed
                              │
                    Return result to LLM stream

Event publishing:
  Tool error → Bus.publish(Session.Event.Error, { sessionID, error })
                    │
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

## Client-Server Topology

### CLI Mode (Embedded)

When running `opencode run`, the server is embedded in the same process:

```
┌─────────────────────────────────────────┐
│  opencode run (single process)          │
│  ├── CLI argument parsing               │
│  ├── Instance.provide()                 │
│  └── Server.listen() on port 4096      │
└─────────────────────────────────────────┘
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

## Tool System Architecture

### Interface Pattern: Strategy + Factory

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

### Registry Pattern

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

### Key Tool Implementations

- `bash.ts` - Shell command execution with tree-sitter parsing
- `edit.ts` / `write.ts` - File editing
- `read.ts` / `glob.ts` / `grep.ts` - File reading and search
- `lsp.ts` - Language Server Protocol integration
- `webfetch.ts` / `websearch.ts` - Web access
- `task.ts` / `todowrite.ts` - Task management
- `apply_patch.ts` - Patch application

---

## Session System Architecture

### State Machine Pattern for Conversation State

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

### Message Flow

1. User submits prompt via `/session/:id/message` POST
2. `SessionPrompt.prompt()` creates assistant message
3. `SessionProcessor.process()` streams LLM response
4. Tool calls are intercepted, executed, and results returned
5. Final response stored in SQLite with full audit trail

---

## Provider System Architecture

### Adapter Pattern for Multiple LLM Providers

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
