# Design Patterns in OpenCode

## Overview

This document enumerates the design patterns used in OpenCode with specific file references and code evidence.

---

## 1. Service Map Pattern (Effect Framework)

**Purpose:** Dependency injection and service management

**Location:** `packages/opencode/src/effect/`, `packages/opencode/src/agent/index.ts`

**Implementation:**

The Effect framework's Service pattern provides a type-safe way to define and access services:

```typescript
// From packages/opencode/src/agent/index.ts
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

// From packages/opencode/src/effect/
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    return Service.of({ get, list, defaultAgent })
  })
)
```

**Usage pattern:**

```typescript
// Access service via Effect context
const agent = yield* Agent.Service
```

**Scoped Cache for Instance State:**

```typescript
// From packages/opencode/src/effect/
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

---

## 2. Observer Pattern (Event Bus Pub/Sub)

**Purpose:** Decoupled event communication between components

**Location:** `packages/opencode/src/bus/index.ts`

**Implementation:**

```typescript
// Event definition
export namespace Bus {
  // Define event types
  export const InstanceDisposed = BusEvent.define("server.instance.disposed", z.object({
    directory: z.string()
  }))

  // Pub/Sub interface
  export interface Interface {
    readonly publish: <D extends BusEvent.Definition>(
      def: D,
      properties: ...
    ) => Effect.Effect<void>
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

```typescript
// Publishing events
Bus.publish(Session.Event.Error, { sessionID, error })

// Subscribing to events
Bus.subscribe(Session.Event.Error)
```

---

## 3. Factory Pattern (Tool.define)

**Purpose:** Standardized tool creation with consistent interface

**Location:** `packages/opencode/src/tool/index.ts`, `packages/opencode/src/tool/bash.ts`

**Implementation:**

```typescript
// From packages/opencode/src/tool/index.ts
export namespace Tool {
  export function define<Parameters extends z.ZodType, Result extends Metadata>(
    id: string,
    init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
  ): Info<Parameters, Result>
}
```

**Usage (from bash.ts):**

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

---

## 4. Strategy Pattern (Agent Types)

**Purpose:** Different agent behaviors based on task requirements

**Location:** `packages/opencode/src/agent/index.ts`

**Implementation:**

```typescript
// Agent definition with strategy configuration
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

---

## 5. Adapter Pattern (Provider System)

**Purpose:** Unified interface over multiple LLM providers

**Location:** `packages/opencode/src/provider/index.ts`

**Implementation:**

```typescript
// From packages/opencode/src/provider/index.ts
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

**Key Files:**
- `packages/opencode/src/provider/index.ts` - Main provider logic
- `packages/opencode/src/provider/service.ts` - Service layer

---

## 6. Repository Pattern (Data Access)

**Purpose:** Abstract database operations behind domain interfaces

**Location:** `packages/opencode/src/session/index.ts`, `packages/opencode/src/storage/index.ts`

**Implementation:**

```typescript
// From packages/opencode/src/session/index.ts
export const page = fn(
  z.object({ sessionID, limit, before }),
  async (input) => {
    const rows = Database.use((db) =>
      db.select().from(MessageTable).where(...).orderBy(...).limit(...).all()
    )
    return hydrate(rows) // Map DB rows to domain objects
  }
)
```

**Key Files:**
- `packages/opencode/src/session/index.ts` - Session repository
- `packages/opencode/src/storage/index.ts` - Storage abstraction

---

## 7. Middleware Pattern (HTTP Routes)

**Purpose:** Cross-cutting concerns (auth, CORS, context) via composable middleware

**Location:** `packages/opencode/src/server/server.ts`

**Implementation:**

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

## 8. Builder Pattern (Tool Context)

**Purpose:** Fluent tool execution context with metadata, permission requests, and abort capability

**Location:** `packages/opencode/src/tool/index.ts` (Context type)

**Implementation:**

Tool execution receives a context object with fluent methods:

```typescript
async execute(params, ctx: Context) {
  // ctx provides:
  // - metadata: Record<string, any> for attaching data
  // - ask(): Request user permission
  // - abort(): Cancel execution
  // - Various utility methods
}
```

---

## 9. Registry Pattern (Tool Registry)

**Purpose:** Central registry for all available tools

**Location:** `packages/opencode/src/tool/index.ts`

**Implementation:**

```typescript
// From packages/opencode/src/tool/index.ts
export namespace ToolRegistry {
  export interface Interface {
    readonly register: (tool: Tool.Info) => Effect.Effect<void>
    readonly ids: () => Effect.Effect<string[]>
    readonly tools: (model, agent?) => Effect.Effect<Tool.Info[]>
  }
}
```

---

## 10. Scoped Singleton Pattern (Instance State)

**Purpose:** Per-working-directory isolated state

**Location:** `packages/opencode/src/effect/instance.ts`

**Implementation:**

```typescript
// From packages/opencode/src/effect/instance.ts
export namespace InstanceState {
  // Creates a cache scoped to the current working directory
  export const make = <A>(
    init: (ctx: Shape) => Effect.Effect<A>
  ): Effect.Effect<InstanceState<A>> => ...
}
```

---

## Pattern Summary Table

| Pattern | Location | Implementation |
|---------|----------|----------------|
| **Service Map** | `src/effect/`, `src/agent/index.ts` | Effect Service pattern with Layer |
| **Observer** | `src/bus/index.ts` | Bus.publish() / Bus.subscribe() |
| **Factory** | `src/tool/index.ts`, `src/tool/*.ts` | Tool.define() factory function |
| **Strategy** | `src/agent/index.ts` | Multiple Agent.Info types |
| **Adapter** | `src/provider/index.ts` | Unified Provider interface |
| **Repository** | `src/session/index.ts`, `src/storage/` | Domain object mapping |
| **Middleware** | `src/server/server.ts` | Hono .use() chains |
| **Builder** | `src/tool/index.ts` | Fluent Context object |
| **Registry** | `src/tool/index.ts` | ToolRegistry singleton |
| **Scoped Singleton** | `src/effect/instance.ts` | InstanceState.make() |
