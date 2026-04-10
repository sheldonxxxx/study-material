# pi-mono Architecture

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/pi-mono`
**Type:** Monorepo (npm workspaces)
**Purpose:** Tools for building AI agents and managing LLM deployments
**Node.js:** >=20.0.0

## Layered Architecture

pi-mono follows a **layered provider architecture** with five distinct layers:

```
┌─────────────────────────────────────────────────────────────┐
│  Integration Layer  │  pi-mom (Slack bot), pi-pods (GPU)  │
├─────────────────────────────────────────────────────────────┤
│  Presentation Layer │  pi-tui (Terminal), pi-web-ui (Web) │
├─────────────────────────────────────────────────────────────┤
│  Application Layer   │  pi-coding-agent (CLI, session mgmt)│
├─────────────────────────────────────────────────────────────┤
│  Runtime Layer       │  pi-agent-core (execution, tools)   │
├─────────────────────────────────────────────────────────────┤
│  Provider Layer      │  pi-ai (multi-provider LLM client)   │
└─────────────────────────────────────────────────────────────┘
```

## Module Responsibilities

### Provider Layer: `pi-ai`

**Purpose:** Unified multi-provider LLM API client

**Location:** `packages/ai/src/`

**Responsibilities:**
- Define `Model<TApi>` interface with cost, context window, capabilities
- Define `StreamFunction<TApi>` contract for provider implementations
- Maintain `KnownApi` and `KnownProvider` type unions
- Implement adapter pattern for each LLM provider (Anthropic, OpenAI, Google, Azure, Mistral, Amazon Bedrock)
- Provide async `EventStream<T>` for streaming event handling

**Key Exports:**
```typescript
// packages/ai/src/index.ts
export type KnownApi = "openai-completions" | "mistral-conversations" |
  "openai-responses" | "azure-openai-responses" | "anthropic-messages" |
  "bedrock-converse-stream" | "google-generative-ai" | ...;

export type KnownProvider = "amazon-bedrock" | "anthropic" | "google" |
  "openai" | "azure-openai-responses" | "mistral" | ...;

export interface Model<TApi extends Api> {
  id: string; name: string; api: TApi; provider: Provider;
  baseUrl: string; reasoning: boolean; input: ("text" | "image")[];
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number; };
  contextWindow: number; maxTokens: number;
}

export type StreamFunction<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> = (
  model: Model<TApi>, context: Context, options?: TOptions,
) => AssistantMessageEventStream;
```

### Runtime Layer: `pi-agent-core`

**Purpose:** Transport-agnostic agent runtime with tool calling

**Location:** `packages/agent/src/`

**Responsibilities:**
- Implement `Agent` class with event subscription model
- Implement `agent-loop.ts` execution loop with tool orchestration
- Support `beforeToolCall` and `afterToolCall` hooks for interception
- Provide `streamProxy` for proxy-based transport
- Define `AgentTool<TParameters>` interface with execute contract

**Key Abstractions:**
```typescript
// packages/agent/src/types.ts
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;
  execute: (
    toolCallId: string, params: Static<TParameters>,
    signal?: AbortSignal, onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
}

export interface AgentContext {
  systemPrompt: string; messages: AgentMessage[]; tools?: AgentTool<any>[];
}

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

### Application Layer: `pi-coding-agent`

**Purpose:** Interactive CLI (the main `pi` command) with session management and tools

**Location:** `packages/coding-agent/src/`

**Responsibilities:**
- Provide CLI entry point (`cli.ts`) and main function (`main.ts`)
- Manage `AgentSession` for conversation state persistence
- Implement file system tools: bash, edit, find, grep, ls, read, write
- Provide `ExtensionContext` via dependency injection
- Maintain TUI mode with interactive components (ModelSelectorComponent, FooterComponent)
- Support `EventBus` for extension pub/sub

**Directory Structure:**
```
packages/coding-agent/src/
├── cli.ts              # CLI entry point (#!/usr/bin/env node)
├── main.ts             # Main function export
├── core/
│   ├── agent-session.ts
│   ├── session-manager.ts
│   ├── tools/          # Tool implementations (bash, edit, read, write, etc.)
│   ├── extensions/     # Extension system with ExtensionContext
│   ├── compaction/     # Context compaction strategies
│   └── event-bus.ts    # Pub/sub event bus
└── modes/
    └── interactive/
        ├── components/  # TUI components
        └── theme/      # Theme utilities
```

### Presentation Layer: `pi-tui`

**Purpose:** Terminal UI library with differential rendering

**Location:** `packages/tui/src/`

**Responsibilities:**
- Provide `TUI` class and `Container` for terminal layout
- Implement UI components: Box, Editor, Input, Loader, Markdown, SelectList, SettingsList, Text
- Handle keybindings and key parsing
- Support terminal image rendering
- Buffer stdin input for interactive mode

**Key Files:**
```typescript
// packages/tui/src/tui.ts - Core TUI class
// packages/tui/src/components/ - UI component library
// packages/tui/src/keybindings.ts - Keybinding management
// packages/tui/src/terminal.ts - Terminal interface
```

### Presentation Layer: `pi-web-ui`

**Purpose:** Web components for AI chat interfaces

**Location:** `packages/web-ui/src/`

**Responsibilities:**
- Provide `ChatPanel.ts` main component
- Implement web components for chat interface
- Dialog components and tool rendering
- Storage utilities for persistence

### Integration Layer: `pi-mom`

**Purpose:** Slack bot that delegates messages to pi coding agent

**Location:** `packages/mom/src/`

**Responsibilities:**
- Slack bot integration (`slack.ts`)
- Agent runner integration (`agent.ts`)
- Channel store management (`store.ts`)
- Event handling and context management
- Sandbox configuration for tool execution

### Integration Layer: `pi-pods`

**Purpose:** CLI for managing vLLM deployments on GPU pods

**Location:** `packages/pods/src/`

**Responsibilities:**
- CLI entry and command handling
- GPU pod configuration and deployment
- Integration with pi-agent-core for agent execution on pods

## Communication Patterns

### Vertical Communication (Layered Flow)

```
User Input (CLI/UI)
    ↓
pi-coding-agent (session management, tools)
    ↓ (agent.prompt())
pi-agent-core (execution loop, tool orchestration)
    ↓ (streamSimple() or streamProxy())
pi-ai (provider adapters)
    ↓ (HTTP/WebSocket)
LLM Provider APIs (Anthropic, OpenAI, Google, etc.)
```

### Horizontal Communication (Within Layer)

**Agent Events** flow from `pi-agent-core` to UI via observer pattern:

```typescript
// packages/agent/src/agent.ts (lines 260-263)
export class Agent {
  private listeners = new Set<(e: AgentEvent) => void>();

  subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }
}

// Usage in pi-coding-agent
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

### Proxy Communication (Transport Abstraction)

When using `streamProxy` instead of direct provider calls:

```
pi-coding-agent
    ↓
pi-agent-core (with streamProxy enabled)
    ↓ (HTTP POST with SSE)
Proxy Server
    ↓ (direct to providers)
LLM Providers
```

```typescript
// packages/agent/src/proxy.ts (lines 85-206)
export function streamProxy(
  model: Model<any>, context: Context, options: ProxyStreamOptions
): ProxyMessageEventStream {
  const response = await fetch(`${options.proxyUrl}/api/stream`, {
    method: "POST",
    headers: { Authorization: `Bearer ${options.authToken}` },
    body: JSON.stringify({ model, context, options }),
  });
  // Events reconstructed from proxy protocol
}
```

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

pi-tui (standalone - no internal dependencies)
pi-web-ui (standalone - no internal dependencies)
pi-ai (standalone - no internal dependencies)
```

## Key Design Decisions

### 1. Stream-First Architecture
All provider communication is streaming, enabling:
- Real-time UI updates with partial tool results
- Progressive message rendering
- Efficient token handling for long conversations

### 2. Message Type Union
`AgentMessage` is a union type allowing extensibility:
```typescript
// AgentMessage = Message (LLM-native) | custom types
```
This maintains compatibility with LLM provider formats while enabling custom extensions.

### 3. Pluggable Provider Registry
New LLM providers added by implementing `StreamFunction` and registering:
```typescript
// packages/ai/src/api-registry.ts
registerApiProvider({
  api: "anthropic-messages",
  stream: streamAnthropic,
  streamSimple: streamSimpleAnthropic,
});
```

### 4. Transport Abstraction
Agent can use direct provider calls (`streamSimple`) or proxy server (`streamProxy`):
```typescript
// Enables server-side auth, routing, logging
```

### 5. Separation of Tool Concerns
Tool definitions include schema, execution, and rendering logic:
```typescript
// Same tool works in TUI, web UI, or headless modes
export interface ToolDefinition<T, TDetails, TState> {
  name: string; label: string; description: string;
  parameters: TSchema;
  async execute(...): Promise<AgentToolResult<TDetails>>;
  renderCall(...): Renderable;
  renderResult(...): Renderable;
}
```

### 6. Extension Hooks
Deep integration points at every stage:
- `context` - Before LLM call, can modify messages
- `before_provider_request` - Inspect/replace payloads
- `agent_start`, `agent_end` - Lifecycle events
- `tool_call`, `tool_result` - Tool execution hooks
- `session_start`, `session_switch`, `session_fork` - Session management

## Extension System

Extensions receive an `ExtensionContext` via dependency injection:

```typescript
// packages/coding-agent/src/core/extensions/types.ts
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
  // ... many more
}

export type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

Extension API provides:
- `on(event, handler)` - Subscribe to events
- `registerTool(tool)` - Add custom tools
- `registerCommand(name, options)` - Add CLI commands
- `registerShortcut(shortcut, options)` - Add keyboard shortcuts
- `registerProvider(name, config)` - Add model providers
