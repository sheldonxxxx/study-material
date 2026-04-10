# pi-mono Architecture

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/pi-mono`
**Type:** Monorepo (npm workspaces)
**Purpose:** Tools for building AI agents and managing LLM deployments
**Node.js:** >=20.0.0

## Architectural Pattern

pi-mono follows a **layered provider architecture** with clear separation between:

1. **Provider Layer** (`pi-ai`) - Unified multi-provider LLM API client
2. **Runtime Layer** (`pi-agent-core`) - Agent execution engine with tool calling
3. **Application Layer** (`pi-coding-agent`) - Interactive CLI with session management
4. **Presentation Layer** (`pi-tui`, `pi-web-ui`) - Terminal and web UI components
5. **Integration Layer** (`pi-mom`, `pi-pods`) - External system integrations

The architecture emphasizes **extensibility** through plugins/extensions and **transport abstraction** that allows agents to run locally or through proxy servers.

## Module Map

```
pi-mono (monorepo root)
├── packages/
│   ├── ai/                    # Provider Layer
│   │   ├── src/
│   │   │   ├── index.ts       # Main exports
│   │   │   ├── types.ts       # Core type definitions
│   │   │   ├── models.ts      # Model registry
│   │   │   ├── stream.ts      # Streaming functions
│   │   │   ├── api-registry.ts # Provider registration
│   │   │   ├── providers/     # Provider implementations
│   │   │   │   ├── anthropic.ts
│   │   │   │   ├── google.ts
│   │   │   │   ├── openai-responses.ts
│   │   │   │   └── ... (10+ providers)
│   │   │   └── utils/
│   │   │       ├── event-stream.ts
│   │   │       ├── json-parse.ts
│   │   │       └── oauth/
│   │   └── dist/              # Built output
│   │
│   ├── agent/                 # Runtime Layer
│   │   ├── src/
│   │   │   ├── index.ts       # Core exports
│   │   │   ├── agent.ts       # Agent class
│   │   │   ├── agent-loop.ts  # Execution loop
│   │   │   ├── proxy.ts       # Proxy streaming
│   │   │   └── types.ts       # Type definitions
│   │   └── dist/
│   │
│   ├── coding-agent/          # Application Layer
│   │   ├── src/
│   │   │   ├── cli.ts         # CLI entry point
│   │   │   ├── main.ts        # Main function
│   │   │   ├── core/
│   │   │   │   ├── agent-session.ts
│   │   │   │   ├── tools/     # Tool implementations
│   │   │   │   ├── extensions/ # Extension system
│   │   │   │   ├── compaction/ # Context compaction
│   │   │   │   ├── event-bus.ts
│   │   │   │   └── ...
│   │   │   ├── modes/
│   │   │   │   └── interactive/ # TUI mode
│   │   │   └── modes/interactive/components/
│   │   └── dist/
│   │
│   ├── tui/                   # Presentation Layer (Terminal)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── tui.ts         # Core TUI class
│   │   │   ├── components/    # UI components
│   │   │   └── ...
│   │   └── dist/
│   │
│   ├── web-ui/                # Presentation Layer (Web)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── ChatPanel.ts
│   │   │   ├── components/
│   │   │   └── ...
│   │   └── example/
│   │
│   ├── mom/                   # Integration Layer
│   │   ├── src/
│   │   │   ├── main.ts        # Slack bot entry
│   │   │   ├── agent.ts      # Agent runner
│   │   │   ├── slack.ts       # Slack integration
│   │   │   └── ...
│   │   └── dist/
│   │
│   └── pods/                  # Integration Layer
│       ├── src/
│       │   ├── cli.ts         # CLI entry
│       │   ├── commands/      # CLI commands
│       │   └── ...
│       └── dist/
```

## Key Abstractions

### 1. Provider Abstraction (pi-ai)

**Purpose:** Unified interface across multiple LLM providers

```
packages/ai/src/types.ts
```

**Core Types:**
```typescript
// Provider and API types
export type KnownApi =
  | "openai-completions"
  | "mistral-conversations"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex";

export type KnownProvider =
  | "amazon-bedrock"
  | "anthropic"
  | "google"
  | "openai"
  | "azure-openai-responses"
  | "mistral"
  | ...;
```

**Model Interface:**
```typescript
// packages/ai/src/types.ts (lines 313-337)
export interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  baseUrl: string;
  reasoning: boolean;
  input: ("text" | "image")[];
  cost: {
    input: number;    // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;
}
```

**Stream Function Contract:**
```typescript
// packages/ai/src/types.ts (lines 117-129)
export type StreamFunction<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions,
) => AssistantMessageEventStream;

// Contract: Must return AssistantMessageEventStream
// Failures encoded in stream, not thrown
```

### 2. Agent Abstraction (pi-agent-core)

**Purpose:** Transport-agnostic agent runtime with tool calling

```
packages/agent/src/types.ts
packages/agent/src/agent-loop.ts
packages/agent/src/agent.ts
```

**AgentTool Interface:**
```typescript
// packages/agent/src/types.ts (lines 272-282)
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
}
```

**AgentContext:**
```typescript
// packages/agent/src/types.ts (lines 284-289)
export interface AgentContext {
  systemPrompt: string;
  messages: AgentMessage[];
  tools?: AgentTool<any>[];
}
```

**AgentEvent Types (excerpt):**
```typescript
// packages/agent/src/types.ts (lines 295-310)
export type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

### 3. Event Stream Abstraction

**Purpose:** Async iteration over streaming events

```
packages/ai/src/utils/event-stream.ts
```

```typescript
// packages/ai/src/utils/event-stream.ts (lines 4-66)
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

### 4. API Provider Registry

**Purpose:** Pluggable provider registration

```
packages/ai/src/api-registry.ts
```

```typescript
// packages/ai/src/api-registry.ts (lines 23-27)
export interface ApiProvider<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

**Registration Pattern:**
```typescript
// Providers register themselves
registerApiProvider({
  api: "anthropic-messages",
  stream: streamAnthropic,
  streamSimple: streamSimpleAnthropic,
});
```

## Design Patterns in Use

### 1. Adapter Pattern (Provider Implementations)

Each provider in `packages/ai/src/providers/` adapts a specific LLM API to the unified `StreamFunction` interface.

**Example - Anthropic:**
```typescript
// packages/ai/src/providers/anthropic.ts (lines 199-441)
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

### 2. Observer/Event Subscription Pattern (Agent)

The Agent class uses an event subscription model for UI updates:

```typescript
// packages/agent/src/agent.ts (lines 260-263)
export class Agent {
  private listeners = new Set<(e: AgentEvent) => void>();

  subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }
}
```

### 3. Strategy Pattern (Tool Execution)

Tool execution supports both sequential and parallel modes:

```typescript
// packages/agent/src/agent-loop.ts (lines 34-35, 336-348)
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

### 4. Hook Pattern (Before/After Tool Call)

Allows interception and modification of tool execution:

```typescript
// packages/agent/src/types.ts (lines 45-66)
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
  }
}
```

### 5. Decorator/Wrapper Pattern (Tool Definitions)

Tools are wrapped with rendering and UI concerns:

```typescript
// packages/coding-agent/src/core/tools/tool-definition-wrapper.ts
export function wrapToolDefinition<T extends TSchema, TDetails, TState>(
  definition: ToolDefinition<T, TDetails, TState>
): AgentTool<T> {
  return {
    ...definition,
    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
      // Adds extension context, error handling
      return definition.execute(toolCallId, params, signal, onUpdate, ctx);
    },
  };
}
```

### 6. Dependency Injection (Extension System)

Extensions receive a context object with all capabilities:

```typescript
// packages/coding-agent/src/core/extensions/types.ts (lines 262-289)
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
  // ...
}
```

### 7. Factory Pattern (Tool Creation)

Tools are created via factory functions for testability and configurability:

```typescript
// packages/coding-agent/src/core/tools/read.ts (lines 114-264)
export function createReadToolDefinition(
  cwd: string,
  options?: ReadToolOptions,
): ToolDefinition<typeof readSchema, ReadToolDetails | undefined> {
  return {
    name: "read",
    label: "read",
    description: "...",
    parameters: readSchema,
    async execute(...) { /* ... */ },
    renderCall(...) { /* ... */ },
    renderResult(...) { /* ... */ },
  };
}
```

### 8. Registry Pattern (Model Registry)

```typescript
// packages/ai/src/models.ts (lines 4-26)
const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();

export function getModel<TProvider extends KnownProvider, TModelId extends ...>(
  provider: TProvider,
  modelId: TModelId,
): Model<...> {
  const providerModels = modelRegistry.get(provider);
  return providerModels?.get(modelId as string);
}
```

### 9. Proxy/Transport Pattern

```typescript
// packages/agent/src/proxy.ts (lines 85-206)
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

### 10. Session/State Management Pattern

```typescript
// packages/coding-agent/src/core/session-manager.ts
// Sessions persist conversation state across invocations
export interface SessionEntry {
  id: string;
  parentId: string | null;
  messages: AgentMessage[];
  timestamp: number;
  model?: Model<any>;
}
```

## Communication Patterns

### Vertical Communication (Layered)

```
CLI/UI
  ↓ (user input)
pi-coding-agent (session management, tools)
  ↓ (agent.prompt())
pi-agent-core (execution loop, tool orchestration)
  ↓ (streamSimple())
pi-ai (provider adapters)
  ↓ (HTTP/WebSocket)
LLM Provider APIs
```

### Horizontal Communication (Within Layer)

**Agent Events** flow from `pi-agent-core` to UI via observer pattern:
```typescript
// packages/coding-agent uses
agent.subscribe((event) => {
  switch (event.type) {
    case "message_end": /* update UI */ break;
    case "tool_execution_start": /* show tool */ break;
  }
});
```

**Extension Events** use a pub/sub event bus:
```typescript
// packages/coding-agent/src/core/event-bus.ts
export class EventBus {
  private handlers = new Map<string, Set<HandlerFn>>();
  emit(event: ExtensionEvent): void;
  on(event: string, handler: HandlerFn): void;
}
```

### Proxy Communication

When using `streamProxy` instead of direct provider calls:
```
pi-coding-agent
  ↓
pi-agent-core (with streamProxy)
  ↓ (HTTP POST with SSE)
Proxy Server
  ↓ (direct to providers)
LLM Providers
```

## Extension System

The coding-agent supports a TypeScript-based extension system:

```typescript
// packages/coding-agent/src/core/extensions/types.ts

// Extension factory
export type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;

// Extension API provides:
export interface ExtensionAPI {
  on(event: string, handler: ExtensionHandler): void;
  registerTool(tool: ToolDefinition): void;
  registerCommand(name: string, options: CommandOptions): void;
  registerShortcut(shortcut: KeyId, options: ShortcutOptions): void;
  registerProvider(name: string, config: ProviderConfig): void;
  // + many more
}
```

**Example Extension Events:**
- `context` - Before LLM call, can modify messages
- `before_provider_request` - Inspect/replace payloads
- `agent_start`, `agent_end` - Lifecycle events
- `tool_call`, `tool_result` - Tool execution hooks
- `session_start`, `session_switch`, `session_fork` - Session management

## Package Dependencies

```
pi-coding-agent
  ├── pi-agent-core
  ├── pi-ai
  └── pi-tui

pi-agent-core
  └── pi-ai

pi-mom
  └── pi-agent-core

pi-pods
  └── pi-agent-core

pi-tui (no internal dependencies)

pi-web-ui (no internal dependencies)

pi-ai (standalone, no internal dependencies)
```

## Key Design Decisions

1. **Stream-First Architecture:** All provider communication is streaming, allowing real-time UI updates and partial tool results.

2. **Message Type Union:** `AgentMessage` is a union of `Message` (LLM-native) and custom types, allowing extensibility while maintaining compatibility.

3. **Pluggable Providers:** New LLM providers can be added by implementing the `StreamFunction` interface and registering with the API registry.

4. **Transport Abstraction:** The agent can call providers directly (`streamSimple`) or through a proxy server (`streamProxy`), enabling server-side auth and routing.

5. **Separation of Concerns:** Tool definitions include schema, execution, and rendering logic. This allows the same tool to work in TUI, web UI, or headless modes.

6. **Extension Hooks:** Deep integration points at every stage of agent execution allow extensions to modify behavior without forking core code.
