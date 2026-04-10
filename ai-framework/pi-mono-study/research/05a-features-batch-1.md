# Features Batch 1 - Deep Dive Research

## Overview

Research into three core features of pi-mono:
1. Unified LLM API (`packages/ai`) - 18+ provider support with cross-provider handoffs
2. Agent Runtime (`packages/agent`) - Stateful tool execution with event streaming
3. Interactive Coding Agent CLI (`packages/coding-agent`) - Terminal coding harness with extensions

---

## Feature 1: Unified LLM API (pi-ai)

### Core Architecture

The unified LLM API (`packages/ai`) provides a single interface for 18+ LLM providers. The architecture is built around:

**Key Files:**
- `src/index.ts` - Main exports (types, models, providers, stream utilities)
- `src/types.ts` - Core type definitions (Model, Context, Message, StreamFunction)
- `src/stream.ts` - High-level streaming API (`streamSimple`, `completeSimple`)
- `src/api-registry.ts` - Provider registration system
- `src/providers/register-builtins.ts` - Lazy-loaded provider initialization
- `src/providers/` - Individual provider implementations
- `src/utils/event-stream.ts` - Generic async event stream class

### Provider Registration System

The API uses a registry pattern (`api-registry.ts`):

```typescript
export interface ApiProvider<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> {
    api: TApi;
    stream: StreamFunction<TApi, TOptions>;
    streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

Providers register via `registerApiProvider()`. The registry maps API types to provider implementations.

**Notable:** Providers are lazily loaded via dynamic imports in `register-builtins.ts`. This keeps the initial bundle small and allows optional providers (like AWS Bedrock which requires Node.js-only dependencies).

### Streaming Architecture

The `EventStream<T, R>` class (`utils/event-stream.ts`) provides a generic async iteration pattern:

```typescript
export class EventStream<T, R = T> implements AsyncIterable<T> {
    private queue: T[] = [];
    private waiting: ((value: IteratorResult<T>) => void)[] = [];
    private done = false;
    // ...
}
```

Key features:
- Push-based event delivery with queueing for async consumers
- Final result resolution via `Promise<R>` returned by `result()`
- `AssistantMessageEventStream` extends this for the LLM protocol

The streaming protocol (`types.ts`) defines fine-grained events:
```typescript
export type AssistantMessageEvent =
    | { type: "start"; partial: AssistantMessage }
    | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
    | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

This allows UI components to render partial results in real-time.

### Partial JSON Parsing

`utils/json-parse.ts` uses `partial-json` library to handle streaming JSON:

```typescript
export function parseStreamingJson<T = any>(partialJson: string | undefined): T {
    if (!partialJson || partialJson.trim() === "") {
        return {} as T;
    }
    // Try standard parsing first (fastest for complete JSON)
    try {
        return JSON.parse(partialJson) as T;
    } catch {
        // Try partial-json for incomplete JSON
        try {
            const result = partialParse(partialJson);
            return (result ?? {}) as T;
        } catch {
            return {} as T;
        }
    }
}
```

This handles the case where streaming chunks arrive mid-JSON-object.

### Model System

`models.ts` provides a typed model registry:

```typescript
export function getModel<TProvider extends KnownProvider, TModelId extends keyof (typeof MODELS)[TProvider]>(
    provider: TProvider,
    modelId: TModelId,
): Model<ModelApi<TProvider, TModelId>>
```

Models are auto-generated in `models.generated.ts` (351KB - a large file of model definitions).

### Cross-Provider Handoffs

The architecture supports switching models mid-conversation. The `Model<TApi>` interface includes:
- `api: TApi` - The API type (e.g., "anthropic-messages", "openai-responses")
- `provider: Provider` - The provider identifier

The `streamSimple` function accepts any registered model and routes to the appropriate provider based on `model.api`.

### Error Handling

**Contract:** Providers must not throw for request/model/runtime failures. Errors are encoded in the stream via the protocol:
```typescript
| { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage }
```

The `AssistantMessage` includes `errorMessage` and `stopReason: "error" | "aborted"`.

### Clever Solutions

1. **Lazy Loading:** Provider modules use dynamic imports to avoid loading unused providers
2. **Partial JSON parsing:** Handles streaming tool call arguments that arrive mid-chunk
3. **Event protocol granularity:** Separate events for text/thinking/toolcall deltas enable fine-grained UI updates
4. **Compatibility overrides:** `OpenAICompletionsCompat` interface allows per-provider customization for OpenAI-compatible APIs

### Technical Debt / Concerns

1. **Large generated models file** (351KB): Auto-generated, but affects IDE load times
2. **Provider-specific hacks:** Some providers have special handling (e.g., `requiresThinkingAsText`, `requiresAssistantAfterToolResult`)
3. **Limited validation:** The `Provider = KnownProvider | string` type allows arbitrary provider strings

---

## Feature 2: Agent Runtime (pi-agent-core)

### Core Architecture

The agent runtime (`packages/agent`) provides stateful tool execution and event streaming. It builds on `pi-ai` and is consumed by the coding agent CLI and Slack bot.

**Key Files:**
- `src/index.ts` - Main exports
- `src/agent.ts` - `Agent` class (high-level interface)
- `src/agent-loop.ts` - Low-level `runAgentLoop` functions
- `src/types.ts` - Type definitions (AgentEvent, AgentLoopConfig, etc.)
- `src/proxy.ts` - Proxy utilities

### Agent Class vs Agent Loop

The `Agent` class (`agent.ts`) is a high-level wrapper around `runAgentLoop`:

```typescript
export class Agent {
    private _state: AgentState = {
        systemPrompt: "",
        model: getModel("google", "gemini-2.5-flash-lite-preview-06-17"),
        thinkingLevel: "off",
        tools: [],
        messages: [],
        isStreaming: false,
        streamMessage: null,
        pendingToolCalls: new Set<string>(),
        error: undefined,
    };
    // ...
}
```

The `runAgentLoop` functions (`agent-loop.ts`) do the actual work:
- `agentLoop()` - Start with new prompt messages
- `agentLoopContinue()` - Continue from existing context (for retries)
- `runAgentLoop()` / `runAgentLoopContinue()` - Internal async implementations

### Event-Driven Architecture

The agent emits typed events (`types.ts`):

```typescript
export type AgentEvent =
    // Agent lifecycle
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }
    // Turn lifecycle
    | { type: "turn_start" }
    | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
    // Message lifecycle
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }
    // Tool execution lifecycle
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

### Tool Execution

Tool execution is handled in `executeToolCallsSequential` and `executeToolCallsParallel`:

```typescript
async function executeToolCallsParallel(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCalls: AgentToolCall[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
): Promise<ToolResultMessage[]>
```

Key flow:
1. Filter tool calls from assistant message
2. Call `prepareToolCall` for each (validates args, calls `beforeToolCall` hook)
3. Execute tools in parallel via `executePreparedToolCall`
4. Call `finalizeExecutedToolCall` (calls `afterToolCall` hook)
5. Emit results

### Hooks System

The `AgentLoopConfig` provides hooks for permission gates and auditing:

```typescript
beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>;
afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>;
```

`BeforeToolCallResult` can block execution:
```typescript
export interface BeforeToolCallResult {
    block?: boolean;
    reason?: string;
}
```

`AfterToolCallResult` can override tool results (merge semantics are field-by-field, no deep merge).

### Steering and Follow-up Messages

The agent supports interrupting/resuming via queues:

```typescript
private steeringQueue: AgentMessage[] = [];
private followUpQueue: AgentMessage[] = [];

// Mode: "all" or "one-at-a-time"
private steeringMode: "all" | "one-at-a-time";
private followUpMode: "all" | "one-at-a-time";
```

- **Steering:** Injected before the next LLM call (agent is working)
- **Follow-up:** Processed after agent would stop (agent finished current task)

### Custom Message Types

The agent uses declaration merging for extensibility (`types.ts`):

```typescript
export interface CustomAgentMessages {
    // Empty by default - apps extend via declaration merging
}

export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

Apps can extend by declaring module augmentation:
```typescript
declare module "@mariozechner/agent" {
    interface CustomAgentMessages {
        artifact: ArtifactMessage;
        notification: NotificationMessage;
    }
}
```

### Clever Solutions

1. **Two-tier architecture:** `Agent` class for convenience, `runAgentLoop` for custom implementations
2. **Parallel tool execution:** Preflight checks sequentially, then execute allowed tools concurrently
3. **Streaming event protocol:** Matches LLM streaming granularity for responsive UI
4. **Declarative message extension:** Type-safe custom messages without runtime overhead

### Error Handling

1. **Tool validation:** `validateToolArguments` (from pi-ai) validates tool call args against schema
2. **Error encoding:** Tool errors are returned as `ToolResultMessage` with `isError: true`
3. **Abort signal handling:** All async operations respect `AbortSignal`
4. **Contract enforcement:** Config functions (`convertToLlm`, `transformContext`) must not throw - return safe fallbacks instead

### Technical Debt / Concerns

1. **No tool timeout:** Tools can run indefinitely (no timeout per tool)
2. **Complex parallel execution:** Results are collected by index, but execution is concurrent - potential ordering issues
3. **Partial message in-place mutation:** `context.messages[context.messages.length - 1] = partialMessage` mutates the context array

---

## Feature 3: Interactive Coding Agent CLI (pi-coding-agent)

### Core Architecture

The coding agent (`packages/coding-agent`) is the main CLI application. It provides an interactive terminal UI with session management, tools, and extensibility.

**Key Files:**
- `src/main.ts` - CLI entry point with argument parsing (~900 lines)
- `src/index.ts` - SDK exports (~350 lines of re-exports)
- `src/core/sdk.ts` - `createAgentSession` factory function
- `src/core/agent-session.ts` - `AgentSession` class (~1000+ lines)
- `src/core/tools/` - Tool implementations (read, bash, edit, write, grep, find, ls)
- `src/modes/interactive/` - Interactive TUI mode

### CLI Entry Point (`main.ts`)

The main function orchestrates:
1. Argument parsing (two-pass for extension flags)
2. Resource loading (extensions, skills, themes)
3. Session management (create, fork, resume, continue)
4. Mode selection (interactive, print, RPC)

Key flow:
```typescript
export async function main(args: string[]) {
    // Early load extensions to discover their CLI flags
    const firstPass = parseArgs(args);
    // ... setup ...

    // Second pass: parse args with extension flags
    const parsed = parseArgs(args, extensionFlags);

    // Create session
    const { session, modelFallbackMessage } = await createAgentSession(sessionOptions);

    // Run mode
    if (mode === "rpc") {
        await runRpcMode(session);
    } else if (isInteractive) {
        const interactiveMode = new InteractiveMode(session, {...});
        await interactiveMode.run();
    } else {
        await runPrintMode(session, {...});
    }
}
```

### Session Management (`core/session-manager.ts`)

The `SessionManager` handles:
- Session creation in `.pi/sessions/` directory
- JSONL-based message storage
- Tree-based branching with session forking
- Session continuation and resume

Key operations:
```typescript
SessionManager.create(cwd, sessionDir)      // New session
SessionManager.open(path, sessionDir)        // Open existing
SessionManager.forkFrom(sourcePath, cwd)    // Fork session
SessionManager.continueRecent(cwd)           // Continue most recent
```

### AgentSession (`core/agent-session.ts`)

`AgentSession` wraps the `Agent` with:
- Event subscription and session persistence
- Model and thinking level management
- Compaction (automatic context pruning)
- Bash execution with abort support
- Session switching and branching
- Extension integration

Key design: The session subscribes to agent events and persists messages automatically:
```typescript
private _handleAgentEvent = (event: AgentEvent): void => {
    // ...
    // Handle session persistence
    if (event.type === "message_end") {
        if (event.message.role === "user" || ...) {
            this.sessionManager.appendMessage(event.message);
        }
    }
    // ...
}
```

### Tools (`core/tools/`)

The coding agent provides 6 built-in tools:

| Tool | File | Purpose |
|------|------|---------|
| read | `read.ts` | Read file contents with truncation |
| bash | `bash.ts` | Execute shell commands |
| edit | `edit.ts` | Edit files with diff-based changes |
| write | `write.ts` | Write/replace file contents |
| grep | `grep.ts` | Search file contents |
| find | `find.ts` | Find files by name |
| ls | `ls.ts` | List directory contents |

Each tool has:
- A **definition** (schema for LLM tool calling)
- A **factory function** (creates tool with cwd context)
- An **operations object** (actual implementation)

Example from `bash.ts`:
```typescript
export function createBashTool(cwd: string, options?: BashToolOptions): AgentTool<...> {
    return {
        name: "bash",
        description: "Execute shell commands in a terminal",
        parameters: BashToolSchema,
        label: "Terminal",
        execute: async (toolCallId, params, signal?, onUpdate?) => {
            // ...
        }
    };
}
```

### Compaction System (`core/compaction/`)

The compaction system prevents context overflow:
- `compaction.ts` - Core compaction logic
- `branch-summarization.ts` - Branch summarization for tree navigation
- `utils.ts` - Token estimation and cut point finding

Compaction is triggered:
- Automatically when context exceeds threshold
- Manually via `/compact` command

Key concept: "Cut points" are identified where context can be safely summarized without losing conversation coherence.

### Extension System (`core/extensions/`)

Extensions provide:
- Custom tools
- Custom commands
- Custom keyboard shortcuts
- Event handlers
- Custom UI components

Extension API (`ExtensionRuntime`):
```typescript
export interface ExtensionRuntime {
    // Emit events
    emit(event: ExtensionEvent): Promise<void>;
    // Register handlers
    on<K extends ExtensionEventType>(type: K, handler: ExtensionHandler<K>): void;
    // Get/set tools
    getTools(): ToolDefinition[];
    setTools(tools: ToolDefinition[]): void;
    // Register commands
    registerCommand(command: RegisteredCommand): void;
    // Register keybindings
    registerKeybinding(shortcut: string, handler: () => void | Promise<void>): void;
}
```

### Clever Solutions

1. **Two-pass argument parsing:** First pass loads extensions to discover their flags, second pass parses with those flags known
2. **File mutation queue:** `withFileMutationQueue` batches file modifications to handle race conditions
3. **Model fallback:** If saved model isn't available, gracefully falls back to first available model
4. **Dynamic convertToLlm:** Filters images if `blockImages` setting changes mid-session
5. **Extension runner swap:** Tool hooks read `extensionRunner` at execution time, allowing hot reload without reinstalling hooks

### Error Handling

1. **Settings errors:** Collected and reported at startup, don't block startup
2. **Extension errors:** Logged but don't fail extension loading
3. **Model resolution:** Creates fallback message if model can't be restored
4. **Session errors:** Graceful handling of corrupted session files

### Technical Debt / Concerns

1. **Large files:** `agent-session.ts` and `main.ts` are very large (~1000+ lines each)
2. **Complex session state:** Many internal state variables (`_steeringMessages`, `_followUpMessages`, `_pendingNextTurnMessages`, etc.)
3. **Synchronous event queue:** `this._agentEventQueue = this._agentEventQueue.then(...)` chains promises for ordering
4. **Image blocking:** `convertToLlmWithBlockImages` has deduplication logic that could miss edge cases

### Run Modes

The CLI supports multiple modes:

| Mode | Description |
|------|-------------|
| interactive | Full TUI with real-time streaming |
| print | Non-interactive, prints final response |
| rpc | JSON-RPC for programmatic usage |

---

## Cross-Cutting Concerns

### Authentication

All three features share auth handling:
- `pi-ai` defines OAuth types and utilities
- `pi-coding-agent` has `AuthStorage` for API key management
- `pi-agent-core` has `getApiKey` config option for dynamic key resolution

### Event Streaming

The event streaming pattern is consistent across:
- `pi-ai`: `AssistantMessageEvent` protocol
- `pi-agent-core`: `AgentEvent` protocol
- `pi-coding-agent`: Extension events build on agent events

### Tool Calling

Tool calling is standardized:
- `pi-ai` defines the `Tool` interface and `validateToolArguments`
- `pi-agent-core` executes tools with before/after hooks
- `pi-coding-agent` provides concrete implementations

### Session Management

Session persistence flows:
1. `pi-coding-agent` manages session files (JSONL)
2. `pi-agent-core` emits events for message lifecycle
3. `pi-coding-agent`'s `AgentSession` subscribes and persists

---

## Dependencies Between Features

```
pi-ai (providers)
    |
    v
pi-agent-core (agent runtime)
    |
    v
pi-coding-agent (CLI application)
```

- `pi-agent-core` depends on `pi-ai` for streaming and types
- `pi-coding-agent` depends on both `pi-ai` and `pi-agent-core`
- `pi-mom` (Slack bot) also depends on `pi-agent-core`
- `pi-pods` has its own agent implementation (not using `pi-agent-core`)

---

## Key Insights

1. **Lazy loading is a first-class pattern:** Used for providers, extensions, and optional dependencies
2. **Event-driven enables reactive UI:** Fine-grained events allow responsive terminal UI
3. **Registry pattern for extensibility:** Providers, tools, extensions all use registration
4. **Contract-based error handling:** Failures encoded in return types, not thrown
5. **Session abstraction enables features:** Branching, compaction, and resume all build on session management
