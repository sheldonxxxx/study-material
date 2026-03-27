# pi-mono Design Patterns

## Pattern Inventory

| # | Pattern | Layer | Evidence |
|---|---------|-------|----------|
| 1 | Adapter Pattern | Provider | `packages/ai/src/providers/anthropic.ts` |
| 2 | Observer Pattern | Runtime | `packages/agent/src/agent.ts` |
| 3 | Strategy Pattern | Runtime | `packages/agent/src/agent-loop.ts` |
| 4 | Hook Pattern | Runtime | `packages/agent/src/types.ts` |
| 5 | Decorator/Wrapper Pattern | Application | `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` |
| 6 | Dependency Injection | Application | `packages/coding-agent/src/core/extensions/types.ts` |
| 7 | Factory Pattern | Application | `packages/coding-agent/src/core/tools/read.ts` |
| 8 | Registry Pattern | Provider | `packages/ai/src/models.ts` |
| 9 | Proxy/Transport Pattern | Runtime | `packages/agent/src/proxy.ts` |
| 10 | Session/State Management | Application | `packages/coding-agent/src/core/session-manager.ts` |
| 11 | Event Stream Pattern | Provider | `packages/ai/src/utils/event-stream.ts` |
| 12 | Pub/Sub Pattern | Application | `packages/coding-agent/src/core/event-bus.ts` |
| 13 | Builder Pattern | Provider | `packages/ai/src/providers/anthropic.ts` (request building) |

---

## 1. Adapter Pattern

**Purpose:** Convert provider-specific APIs into unified `StreamFunction` interface

**Evidence:** `packages/ai/src/providers/anthropic.ts`

```typescript
// Lines 199-441
export const streamAnthropic: StreamFunction<"anthropic-messages", AnthropicOptions> = (
  model: Model<"anthropic-messages">,
  context: Context,
  options?: AnthropicOptions,
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream();
  // Converts pi-ai types -> Anthropic SDK types
  // Wraps Anthropic streaming events -> AssistantMessageEventStream
  return stream;
};
```

**How it works:**
- Each provider (`anthropic.ts`, `openai-responses.ts`, `google.ts`, etc.) implements the same `StreamFunction` signature
- Provider adapters handle API-specific request/response transformations
- Unified interface allows runtime to switch providers without code changes

**Files:**
- `packages/ai/src/providers/anthropic.ts`
- `packages/ai/src/providers/openai-responses.ts`
- `packages/ai/src/providers/google.ts`
- `packages/ai/src/providers/azure.ts`
- `packages/ai/src/providers/mistral.ts`
- `packages/ai/src/providers/bedrock.ts`

---

## 2. Observer Pattern

**Purpose:** Enable UI components to subscribe to agent lifecycle events

**Evidence:** `packages/agent/src/agent.ts`

```typescript
// Lines 260-263
export class Agent {
  private listeners = new Set<(e: AgentEvent) => void>();

  subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }

  private notify(event: AgentEvent): void {
    for (const listener of this.listeners) {
      listener(event);
    }
  }
}
```

**Usage in pi-coding-agent:**
```typescript
agent.subscribe((event) => {
  switch (event.type) {
    case "message_end": updateUI(event.message); break;
    case "tool_execution_start": showTool(event.toolName); break;
    case "tool_execution_end": hideTool(event.toolName); break;
  }
});
```

**Event Types:** `agent_start`, `agent_end`, `turn_start`, `turn_end`, `message_start`, `message_update`, `message_end`, `tool_execution_start`, `tool_execution_update`, `tool_execution_end`

**Files:**
- `packages/agent/src/agent.ts`
- `packages/agent/src/types.ts` (AgentEvent definition)

---

## 3. Strategy Pattern

**Purpose:** Allow configurable tool execution modes (sequential vs parallel)

**Evidence:** `packages/agent/src/agent-loop.ts`

```typescript
// Lines 34-35, 336-348
export type ToolExecutionMode = "sequential" | "parallel";

async function executeToolCalls(
  currentContext: AgentContext,
  assistantMessage: AssistantMessage,
  config: AgentLoopConfig,
  ...
): Promise<ToolResultMessage[]> {
  if (config.toolExecution === "sequential") {
    return executeToolCallsSequential(...);
  }
  return executeToolCallsParallel(...);
}
```

**Strategy implementations:**
- `executeToolCallsSequential` - Tools run one after another
- `executeToolCallsParallel` - Tools run concurrently with Promise.all

**Files:**
- `packages/agent/src/agent-loop.ts`

---

## 4. Hook Pattern

**Purpose:** Enable interception and modification of tool execution lifecycle

**Evidence:** `packages/agent/src/types.ts`

```typescript
// Lines 45-66
export interface BeforeToolCallResult {
  block?: boolean;
  reason?: string;
}

export interface AfterToolCallResult {
  content?: (TextContent | ImageContent)[];
  details?: unknown;
  isError?: boolean;
}

// Usage in agent-loop.ts
if (config.beforeToolCall) {
  const beforeResult = await config.beforeToolCall(context, signal);
  if (beforeResult?.block) {
    // Return error result instead of executing
    return { content: [{ type: "text", text: `Blocked: ${beforeResult.reason}` }], isError: true };
  }
}

// ... tool execution ...

if (config.afterToolCall) {
  await config.afterToolCall(context, toolCallId, result, signal);
}
```

**Hook points:**
- `beforeToolCall` - Called before tool execution, can block or modify
- `afterToolCall` - Called after tool execution, can modify results

**Files:**
- `packages/agent/src/types.ts` (BeforeToolCallResult, AfterToolCallResult)
- `packages/agent/src/agent-loop.ts` (hook invocation)

---

## 5. Decorator/Wrapper Pattern

**Purpose:** Add extension context and error handling to tool definitions

**Evidence:** `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`

```typescript
// Lines 368-378
export function wrapToolDefinition<T extends TSchema, TDetails, TState>(
  definition: ToolDefinition<T, TDetails, TState>
): AgentTool<T> {
  return {
    ...definition,
    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
      // Adds extension context, error handling, logging
      try {
        return await definition.execute(toolCallId, params, signal, onUpdate, ctx);
      } catch (error) {
        // Standardized error handling
        return { content: [{ type: "text", text: `Error: ${error.message}` }], isError: true };
      }
    },
  };
}
```

**Wrapping concerns:**
- Extension context injection
- Error handling and standardization
- Logging and metrics
- Signal propagation

**Files:**
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`

---

## 6. Dependency Injection

**Purpose:** Provide extensions with all capabilities via context object

**Evidence:** `packages/coding-agent/src/core/extensions/types.ts`

```typescript
// Lines 262-289
export interface ExtensionContext {
  ui: ExtensionUIContext;           // UI methods
  hasUI: boolean;
  cwd: string;
  sessionManager: ReadonlySessionManager;
  modelRegistry: ModelRegistry;
  model: Model<any> | undefined;
  isIdle(): boolean;
  abort(): void;
  compact(options?: CompactOptions): void;
  // ... many more capabilities
}

export type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

**Extension API surface:**
```typescript
export interface ExtensionAPI {
  on(event: string, handler: ExtensionHandler): void;
  registerTool(tool: ToolDefinition): void;
  registerCommand(name: string, options: CommandOptions): void;
  registerShortcut(shortcut: KeyId, options: ShortcutOptions): void;
  registerProvider(name: string, config: ProviderConfig): void;
  // ... more methods
}
```

**Files:**
- `packages/coding-agent/src/core/extensions/types.ts`

---

## 7. Factory Pattern

**Purpose:** Create configurable tool instances for testability and flexibility

**Evidence:** `packages/coding-agent/src/core/tools/read.ts`

```typescript
// Lines 114-264
export function createReadToolDefinition(
  cwd: string,
  options?: ReadToolOptions,
): ToolDefinition<typeof readSchema, ReadToolDetails | undefined> {
  return {
    name: "read",
    label: "read",
    description: "...",
    parameters: readSchema,
    async execute(toolCallId, params, signal, onUpdate, ctx) { /* ... */ },
    renderCall(toolCallId, params, ctx) { /* ... */ },
    renderResult(toolCallId, result, ctx) { /* ... */ },
  };
}
```

**Factory benefits:**
- CWD injected at creation time
- Options for different modes (verbose,限额, etc.)
- Consistent interface across tool instances
- Easy to test with mocked dependencies

**Files:**
- `packages/coding-agent/src/core/tools/read.ts`
- `packages/coding-agent/src/core/tools/write.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/edit.ts`

---

## 8. Registry Pattern

**Purpose:** Centralized model lookup by provider and model ID

**Evidence:** `packages/ai/src/models.ts`

```typescript
// Lines 4-26
const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();

export function getModel<TProvider extends KnownProvider, TModelId extends ...>(
  provider: TProvider,
  modelId: TModelId,
): Model<...> {
  const providerModels = modelRegistry.get(provider);
  return providerModels?.get(modelId as string);
}

// Registration happens at build time via models.generated.ts
```

**Registry structure:**
- Outer Map: provider name -> Inner Map
- Inner Map: model ID -> Model instance
- Supports `listModels()`, `getModel()`, `hasModel()`

**Files:**
- `packages/ai/src/models.ts`
- `packages/ai/src/models.generated.ts` (generated registrations)

---

## 9. Proxy/Transport Pattern

**Purpose:** Route agent requests through proxy server for auth, routing, logging

**Evidence:** `packages/agent/src/proxy.ts`

```typescript
// Lines 85-206
export function streamProxy(
  model: Model<any>,
  context: Context,
  options: ProxyStreamOptions
): ProxyMessageEventStream {
  // Routes through proxy server instead of calling providers directly
  const response = await fetch(`${options.proxyUrl}/api/stream`, {
    method: "POST",
    headers: { Authorization: `Bearer ${options.authToken}` },
    body: JSON.stringify({ model, context, options }),
  });
  // Events reconstructed from proxy protocol
}
```

**Transport options:**
- Direct: `streamSimple()` calls provider directly
- Proxy: `streamProxy()` routes through proxy server

**Files:**
- `packages/agent/src/proxy.ts`

---

## 10. Session/State Management Pattern

**Purpose:** Persist conversation state across invocations with parent-child relationships

**Evidence:** `packages/coding-agent/src/core/session-manager.ts`

```typescript
// Session entry structure
export interface SessionEntry {
  id: string;
  parentId: string | null;    // Supports branching/forking
  messages: AgentMessage[];
  timestamp: number;
  model?: Model<any>;
}

// Session operations
- createSession(): string
- forkSession(parentId): string
- getSession(id): SessionEntry
- listSessions(): SessionEntry[]
- deleteSession(id): void
```

**Features:**
- Parent-child relationships for session branching
- Message history persistence
- Model selection per session
- Timestamp tracking

**Files:**
- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/agent-session.ts`

---

## 11. Event Stream Pattern

**Purpose:** Async iteration over streaming events with completion detection

**Evidence:** `packages/ai/src/utils/event-stream.ts`

```typescript
// Lines 4-66
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  constructor(
    private isComplete: (event: T) => boolean,
    private extractResult: (event: T) => R,
  ) {}

  push(event: T): void { /* ... */ }
  end(result?: R): void { /* ... */ }

  async *[Symbol.asyncIterator](): AsyncIterator<T> { /* ... */ }

  result(): Promise<R> { return this.finalResultPromise; }
}
```

**Usage:**
```typescript
const stream = new AssistantMessageEventStream();
// Provider pushes events
provider.push({ type: "content_block_delta", ... });
// Consumer iterates
for await (const event of stream) { /* ... */ }
// Final result when complete
const result = await stream.result();
```

**Files:**
- `packages/ai/src/utils/event-stream.ts`

---

## 12. Pub/Sub Pattern

**Purpose:** Decouple extension communication via event bus

**Evidence:** `packages/coding-agent/src/core/event-bus.ts`

```typescript
// EventBus implementation
export class EventBus {
  private handlers = new Map<string, Set<HandlerFn>>();

  emit(event: ExtensionEvent): void {
    const handlers = this.handlers.get(event.type);
    if (handlers) {
      for (const handler of handlers) {
        handler(event);
      }
    }
  }

  on(event: string, handler: HandlerFn): void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler);
  }

  off(event: string, handler: HandlerFn): void {
    this.handlers.get(event)?.delete(handler);
  }
}
```

**Extension events:**
- `context` - Before LLM call
- `before_provider_request` - Inspect/replace payloads
- `agent_start`, `agent_end` - Lifecycle
- `tool_call`, `tool_result` - Tool hooks
- `session_start`, `session_switch`, `session_fork` - Session management

**Files:**
- `packages/coding-agent/src/core/event-bus.ts`

---

## 13. Builder Pattern

**Purpose:** Construct complex provider requests step by step

**Evidence:** `packages/ai/src/providers/anthropic.ts`

```typescript
// Request building (illustrative)
const request = AnthropicMessagesBuilder.create()
  .withModel(model.id)
  .withSystemPrompt(systemPrompt)
  .withMessages(messages.map(toAnthropicFormat))
  .withMaxTokens(options?.maxTokens ?? 4096)
  .withTemperature(options?.temperature ?? 1.0)
  .withStreaming(true)
  .build();
```

**Benefits:**
- Readable construction of complex objects
- Validation at each step
- Optional parameters handled cleanly
- Immutable builders support reuse

**Files:**
- `packages/ai/src/providers/anthropic.ts`
- `packages/ai/src/providers/openai-responses.ts`

---

## Pattern Relationship Diagram

```
Extension System (DI + Hooks)
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                   pi-coding-agent                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │   Factory   │  │   Wrapper   │  │  EventBus (Pub) │  │
│  │  (tools)    │  │  (tools)    │  │                  │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────────────────────────┐
│                    pi-agent-core                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  Strategy   │  │   Hooks     │  │    Observer      │  │
│  │  (tool exec) │  │ (before/    │  │  (Agent events)  │  │
│  │             │  │  after)      │  │                  │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Proxy/Transport                     │   │
│  │         (direct vs proxy routing)                │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                      pi-ai                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  Registry   │  │  Adapter    │  │  EventStream    │  │
│  │  (models)   │  │  (providers)│  │  (async iter)   │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Builder (request construction)          │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Pattern Summary Table

| Pattern | Purpose | Key Benefit |
|---------|---------|-------------|
| Adapter | Provider unification | Swap providers without code changes |
| Observer | Event subscription | Decouple agent from UI |
| Strategy | Tool execution modes | Configurable parallelism |
| Hook | Tool lifecycle interception | Extensibility without forking |
| Decorator | Tool wrapper | Add concerns non-invasively |
| DI | Extension context | Extensions get all capabilities |
| Factory | Tool creation | Configurable, testable tools |
| Registry | Model lookup | Centralized, type-safe access |
| Proxy | Transport abstraction | Auth/routing flexibility |
| Session | State persistence | Conversation continuity |
| EventStream | Async streaming | Backpressure-friendly iteration |
| Pub/Sub | Extension events | Loose coupling |
| Builder | Request construction | Readable, validated construction |
