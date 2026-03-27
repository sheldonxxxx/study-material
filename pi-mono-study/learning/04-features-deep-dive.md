# pi-mono Features Deep Dive

Synthesized from research batches 1-3 and the feature index.

---

## Core Features

### 1. Unified LLM API (`packages/ai`)

**Priority:** P0 - Foundational

The unified LLM API provides a single interface for 18+ LLM providers with automatic model discovery, token/cost tracking, and cross-provider handoffs. This is the foundational layer that powers all other features.

#### Architecture

The architecture centers on a registry pattern (`src/api-registry.ts`) where providers register via `registerApiProvider()`. Key design decisions:

**Lazy Loading.** Provider modules use dynamic imports in `register-builtins.ts` to avoid loading unused providers. This keeps bundle sizes small and allows Node.js-only providers (like AWS Bedrock) to be conditionally loaded.

**Streaming via EventStream.** The `EventStream<T, R>` class (`src/utils/event-stream.ts`) implements a push-based async iteration pattern with queueing for async consumers:

```typescript
export class EventStream<T, R = T> implements AsyncIterable<T> {
    private queue: T[] = [];
    private waiting: ((value: IteratorResult<T>) => void)[] = [];
    private done = false;
}
```

The streaming protocol defines fine-grained events (`src/types.ts`):

```typescript
export type AssistantMessageEvent =
    | { type: "start"; partial: AssistantMessage }
    | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
    | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

This granularity enables responsive UI rendering of partial results.

**Partial JSON Parsing.** Streaming tool call arguments can arrive mid-chunk. `src/utils/json-parse.ts` handles this with a fallback chain:

```typescript
export function parseStreamingJson<T = any>(partialJson: string | undefined): T {
    if (!partialJson || partialJson.trim() === "") {
        return {} as T;
    }
    try {
        return JSON.parse(partialJson) as T;  // Fast path for complete JSON
    } catch {
        try {
            const result = partialParse(partialJson);  // partial-json library
            return (result ?? {}) as T;
        } catch {
            return {} as T;
        }
    }
}
```

**Cross-Provider Handoffs.** The `Model<TApi>` interface includes `api` (API type like "anthropic-messages") and `provider` (provider identifier). The `streamSimple` function routes to the appropriate provider based on `model.api`.

#### Clever Solutions

1. **Compatibility Overrides.** The `OpenAICompletionsCompat` interface allows per-provider customization for OpenAI-compatible APIs without breaking the unified interface.

2. **Contract-Based Errors.** Providers must not throw for request/model/runtime failures. Errors are encoded in the stream protocol with `stopReason: "error" | "aborted"`.

3. **Auto-Generated Models.** `models.generated.ts` (351KB) contains all model definitions. While large, it keeps the codebase DRY.

#### Technical Debt

1. **Large generated file** affects IDE load times
2. **Provider-specific hacks** exist for `requiresThinkingAsText`, `requiresAssistantAfterToolResult`, etc.
3. **Weak validation** - `Provider = KnownProvider | string` allows arbitrary provider strings

---

### 2. Agent Runtime (`packages/agent`)

**Priority:** P0 - Foundational

Stateful agent with tool execution and event streaming. Built on `pi-ai`, it powers both the CLI and Slack bot.

#### Architecture

Two-tier design: the `Agent` class (`src/agent.ts`) provides a high-level convenience API, while `runAgentLoop` functions (`src/agent-loop.ts`) expose the low-level state machine:

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
}
```

**Event-Driven.** The agent emits typed events (`src/types.ts`) covering the full lifecycle:

```typescript
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

**Tool Execution.** Tools execute via `executeToolCallsParallel` which:
1. Filters tool calls from assistant message
2. Calls `prepareToolCall` for each (validates args, calls `beforeToolCall` hook)
3. Executes allowed tools concurrently
4. Calls `finalizeExecutedToolCall` (calls `afterToolCall` hook)

**Hooks System.** The `AgentLoopConfig` provides permission gates:

```typescript
beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>;
afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>;
```

`BeforeToolCallResult` can block execution; `AfterToolCallResult` can override tool results with field-by-field merge (no deep merge).

**Steering and Follow-up Queues.** Two message queues interrupt or follow agent work:

```typescript
private steeringQueue: AgentMessage[] = [];
private followUpQueue: AgentMessage[] = [];
private steeringMode: "all" | "one-at-a-time";
private followUpMode: "all" | "one-at-a-time";
```

- **Steering:** Injected before the next LLM call (agent is working)
- **Follow-up:** Processed after agent would stop (agent finished current task)

**Extensible Message Types.** Declaration merging allows custom messages without runtime overhead:

```typescript
export interface CustomAgentMessages {
    // Empty by default - apps extend via declaration merging
}
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

#### Clever Solutions

1. **Parallel tool execution with sequential preflight.** Preflight checks run sequentially, then allowed tools execute concurrently.

2. **Streaming event protocol.** Matches LLM streaming granularity for responsive UI updates.

3. **Declarative extension.** Custom messages via declaration merging avoids wrapper classes.

#### Technical Debt

1. **No tool timeout.** Tools can run indefinitely.
2. **Complex parallel execution.** Results collected by index but execution is concurrent - potential ordering issues.
3. **In-place mutation.** `context.messages[context.messages.length - 1] = partialMessage` mutates the context array.

---

### 3. Interactive Coding Agent CLI (`packages/coding-agent`)

**Priority:** P0 - Primary Product

Terminal coding harness with interactive mode, session management, and deep customization.

#### Architecture

**CLI Entry Point.** `src/main.ts` (~900 lines) orchestrates argument parsing, resource loading, and mode selection:

```typescript
export async function main(args: string[]) {
    // Two-pass parsing: first pass loads extensions to discover their flags
    const firstPass = parseArgs(args);
    // ... setup ...
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

**Session Management.** `SessionManager` (`src/core/session-manager.ts`) handles:
- Session creation in `.pi/sessions/` directory
- JSONL-based message storage
- Tree-based branching with session forking
- Session continuation and resume

**AgentSession.** `AgentSession` (~1000+ lines) wraps `Agent` with event subscription and session persistence:

```typescript
private _handleAgentEvent = (event: AgentEvent): void => {
    if (event.type === "message_end") {
        if (event.message.role === "user" || ...) {
            this.sessionManager.appendMessage(event.message);
        }
    }
}
```

#### Built-in Tools

| Tool | File | Purpose |
|------|------|---------|
| read | `core/tools/read.ts` | Read file contents with truncation |
| bash | `core/tools/bash.ts` | Execute shell commands |
| edit | `core/tools/edit.ts` | Edit files with diff-based changes |
| write | `core/tools/write.ts` | Write/replace file contents |
| grep | `core/tools/grep.ts` | Search file contents |
| find | `core/tools/find.ts` | Find files by name |
| ls | `core/tools/ls.ts` | List directory contents |

Each tool follows a consistent pattern: definition (schema), factory function (creates tool with cwd context), and operations object.

#### Compaction System

Prevents context overflow via automatic context pruning. Key concept: "cut points" identify where context can be safely summarized without losing conversation coherence.

#### Run Modes

| Mode | Description |
|------|-------------|
| interactive | Full TUI with real-time streaming |
| print | Non-interactive, prints final response |
| rpc | JSON-RPC for programmatic usage |

#### Clever Solutions

1. **Two-pass argument parsing.** First pass loads extensions to discover their flags, second pass parses with those flags known.

2. **File mutation queue.** `withFileMutationQueue` batches file modifications to handle race conditions.

3. **Model fallback.** If saved model isn't available, gracefully falls back to first available model.

4. **Dynamic convertToLlm.** Filters images if `blockImages` setting changes mid-session.

#### Technical Debt

1. **Large files.** `agent-session.ts` and `main.ts` exceed 1000 lines each.
2. **Complex session state.** Many internal state variables (`_steeringMessages`, `_followUpMessages`, `_pendingNextTurnMessages`, etc.).
3. **Synchronous event queue.** `this._agentEventQueue = this._agentEventQueue.then(...)` chains promises for ordering.

---

### 4. GPU Pod Management (`packages/pods`)

**Priority:** P1 - Infrastructure

CLI for deploying and managing vLLM on GPU pods with automatic configuration for agentic workloads.

#### Architecture

**Model Configuration.** `models.json` contains predefined configurations for agentic workloads:

```json
{
  "Qwen/Qwen2.5-Coder-32B-Instruct": {
    "configs": [
      {
        "gpuCount": 1,
        "gpuTypes": ["H100", "H200"],
        "args": ["--tool-call-parser", "hermes", "--enable-auto-tool-choice"]
      }
    ]
  }
}
```

**Supported Tool Calling Parsers:**
| Parser | Models |
|--------|--------|
| `hermes` | Qwen2.5-Coder |
| `qwen3_coder` | Qwen3-Coder, Qwen3-Coder-FP8 |
| `glm45` | GLM-4.5, GLM-4.5-Air |
| `kimi_k2` | Kimi-K2 (requires vLLM v0.10.0rc1+) |

#### Deployment Flow

1. **Pod setup** (`setupPod`):
   - Tests SSH connection
   - Copies `scripts/pod_setup.sh` to remote
   - Runs setup with vLLM installation
   - Detects GPU configuration via `nvidia-smi`
   - Saves pod config to `~/.pi/pods.json`

2. **Model start** (`startModel`):
   - Validates GPU count against available hardware
   - Selects GPUs using round-robin based on current load
   - Customizes `model_run.sh` template with vLLM arguments
   - Uses `script` command for pseudo-TTY and log preservation
   - Uses `setsid` for background process survival over SSH disconnection

#### Clever Solutions

1. **Round-robin GPU allocation.** Balances load across running models.

2. **Background process survival.** Uses `setsid` to create new session, detached from SSH.

3. **Log streaming.** Uses `tail -f` over SSH with `script` command to preserve colors.

4. **Memory/context overrides.** Filters and replaces vLLM args for `--gpu-memory-utilization` and `--max-model-len`.

#### Technical Debt

**Incomplete agent integration.** The `promptModel` function in `src/commands/prompt.ts` throws "Not implemented":

```typescript
try {
    throw new Error("Not implemented");
} catch (err: any) {
    console.error(chalk.red(`Agent error: ${err.message}`));
    process.exit(1);
}
```

The CLI command `pi agent <name>` is defined but not implemented.

---

### 5. Terminal UI Framework (`packages/tui`)

**Priority:** P1 - Infrastructure

Minimal TUI framework with differential rendering and synchronized output for flicker-free updates.

#### Three-Strategy Differential Rendering

1. **First render:** Full output without clearing (assumes clean screen)
2. **Width/height changes:** Full clear and re-render (wrapping can change)
3. **Content changes:** Incremental render from first changed line

```typescript
// Find first and last changed lines
let firstChanged = -1;
let lastChanged = -1;
for (let i = 0; i < maxLines; i++) {
    const oldLine = i < previousLines.length ? previousLines[i] : "";
    const newLine = i < newLines.length ? newLines[i] : "";
    if (oldLine !== newLine) {
        if (firstChanged === -1) firstChanged = i;
        lastChanged = i;
    }
}
```

**CSI 2026 Synchronized Output.** Uses `\x1b[?2026h` ... `\x1b[?2026l` for atomic screen updates, preventing partial renders from showing.

#### Termux Special Case

Height changes in Termux (when software keyboard shows/hides) do NOT trigger full redraws because it would cause entire history to replay:

```typescript
if (heightChanged && !isTermuxSession()) {
    fullRender(true);
    return;
}
```

#### IME Support

**CURSOR_MARKER:** Zero-width APC sequence (`\x1b_pi:c\x07`) emitted by focused components at cursor position. TUI extracts this marker and positions hardware cursor there for proper IME candidate window placement.

#### Input Component

Full Emacs-style editing in a single-line input:
- Kill ring for deleted text
- Undo stack with coalescing
- Word-wise cursor movement using `Intl.Segmenter`
- Kitty CSI-u printable character decoding
- Grapheme-aware cursor positioning

#### Environment Variables

- `PI_HARDWARE_CURSOR=1` - Show hardware cursor
- `PI_CLEAR_ON_SHRINK=1` - Clear empty rows when content shrinks
- `PI_TUI_WRITE_LOG=/path` - Log all terminal output
- `PI_DEBUG_REDRAW=1` - Log redraw reasons

---

### 6. Web UI Components (`packages/web-ui`)

**Priority:** P1 - User-Facing

Reusable web components for building AI chat interfaces powered by `pi-ai` and `pi-agent-core`.

#### Architecture

Built with **Lit** (web components library). Component hierarchy:

```
ChatPanel (pi-chat-panel)
  ├── AgentInterface (agent-interface)
  │     ├── MessageList (message-list)
  │     ├── StreamingMessageContainer (streaming-message-container)
  │     └── MessageEditor (message-editor)
  └── ArtifactsPanel (artifacts-panel)
```

**Dual rendering for streaming:**
1. `message-list` for completed messages (stable, doesn't re-render during streaming)
2. `streaming-message-container` for the currently streaming message

#### Artifacts System

Sandboxed rendering of multiple types:

| Type | Renderer | Notes |
|------|----------|-------|
| HTML | `HtmlArtifact` | Sandboxed iframe |
| SVG | `SvgArtifact` | Inline SVG |
| Markdown | `MarkdownArtifact` | Rendered markdown |
| PDF | `PdfArtifact` | Preview component |
| DOCX | `DocxArtifact` | Text extraction |
| XLSX | `ExcelArtifact` | Sheet rendering |

#### Storage Backend

IndexedDB with multiple stores:

```typescript
const stores = {
    sessions: SessionsStore,           // Conversation history
    providerKeys: ProviderKeysStore,   // API keys
    settings: SettingsStore,           // User preferences
    customProviders: CustomProvidersStore  // Custom LLM providers
};
```

#### Auto-scroll Behavior

```typescript
private _handleScroll = (_ev: any) => {
    const distanceFromBottom = scrollHeight - currentScrollTop - clientHeight;
    if (currentScrollTop !== 0 && currentScrollTop < lastScrollTop && distanceFromBottom > 50) {
        this._autoScroll = false;  // Disable if user scrolled up
    } else if (distanceFromBottom < 10) {
        this._autoScroll = true;   // Re-enable if close to bottom
    }
};
```

---

## Secondary Features

### 7. Extensions System (`packages/coding-agent`)

**Priority:** P2

TypeScript module system for extending the coding agent with custom tools, commands, keyboard shortcuts, and UI.

#### Key Architecture Decisions

**jiti-Based TypeScript Execution.** Extensions load using the `jiti` library:

```typescript
const jiti = createJiti(import.meta.url, {
  moduleCache: false,
  ...(isBunBinary
    ? { virtualModules: VIRTUAL_MODULES, tryNative: false }
    : { alias: getAliases() }),
});
const module = await jiti.import(extensionPath, { default: true });
```

This allows TypeScript extensions to run without compilation. `VIRTUAL_MODULES` bundles allowed packages (`@sinclair/typebox`, `@mariozechner/pi-*`) directly into the binary.

**Separation of Extension API and Runtime.** The `ExtensionAPI` only contains registration methods; action methods delegate to `ExtensionRuntime`. This allows runtime stub methods to throw helpful errors if called during extension load before binding.

**Lazy Action Binding with Provider Queuing.** Provider registrations during extension load are queued and flushed when the runner binds core actions:

```typescript
registerProvider: (name, config, extensionPath = "<unknown>") => {
  runtime.pendingProviderRegizations.push({ name, config, extensionPath });
},
// In runner.bindCore():
for (const { name, config, extensionPath } of this.runtime.pendingProviderRegistrations) {
  this.modelRegistry.registerProvider(name, config);
}
```

#### Session Event Model

Extensions can hook into comprehensive lifecycle events:
- `session_directory`, `session_start`, `session_switch`, `session_shutdown`
- `session_before_fork`, `session_fork` - branching hooks
- `session_before_compact`, `session_compact` - context compaction
- `session_before_tree`, `session_tree` - tree navigation

#### Technical Debt

1. **No isolation.** Extensions share the same process. Security is trust-based.
2. **Hot reload complexity.** `ctx.reload()` behavior is subtle - documentation warns about in-memory state assumptions.
3. **Circular dependency risk.** `index.ts` doesn't re-export from loader to avoid circular deps.

---

### 8. Skills and Prompt Templates (`packages/coding-agent`)

**Priority:** P2

Skills are self-contained capability packages invoked via `/skill:name`, following the Agent Skills standard. Prompt templates expand via `/name`.

#### Skills Implementation

**Discovery.** Skills load from:
- `~/.pi/agent/skills/` (user global)
- `~/.agents/skills/` (Agent Skills compat)
- `.pi/skills/` (project)
- `skills/` in npm packages

**Validation.** Follows Agent Skills standard strictly:

```typescript
function validateName(name: string, parentDirName: string): string[] {
  const errors = [];
  if (name !== parentDirName) errors.push("name does not match parent directory");
  if (name.length > 64) errors.push("name exceeds 64 characters");
  if (!/^[a-z0-9-]+$/.test(name)) errors.push("invalid characters");
  if (name.startsWith("-") || name.endsWith("-")) errors.push("no leading/trailing hyphens");
  if (name.includes("--")) errors.push("no consecutive hyphens");
  return errors;
}
```

**Progressive Disclosure.** System prompt includes skill names/descriptions. The agent must use `read` to load actual SKILL.md content on-demand.

#### Prompt Templates Implementation

**Argument Substitution (Bash-style):**

```typescript
export function substituteArgs(content: string, args: string[]): string {
  let result = content;
  result = result.replace(/\$(\d+)/g, (_, num) => args[parseInt(num) - 1] ?? "");
  result = result.replace(/\$\{@:(\d+)(?::(\d+))?\}/g, (_, startStr, lengthStr) => {
    let start = parseInt(startStr) - 1;
    if (start < 0) start = 0;
    if (lengthStr) return args.slice(start, start + parseInt(lengthStr)).join(" ");
    return args.slice(start).join(" ");
  });
  result = result.replace(/\$ARGUMENTS/g, args.join(" "));
  result = result.replace(/\$@/g, args.join(" "));
  return result;
}
```

Order matters: positional must be replaced before wildcards to avoid `$1` being caught by `$@`.

---

### 9. Multi-Model Authentication (`packages/ai`, `packages/coding-agent`)

**Priority:** P2

Supports API keys via environment variables, OAuth flows, and token refresh with file locking.

#### OAuth Provider Interface

```typescript
export interface OAuthProviderInterface {
  readonly id: OAuthProviderId;
  readonly name: string;
  usesCallbackServer?: boolean;
  login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
  refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;
  getApiKey(credentials: OAuthCredentials): string;
  modifyModels?(models, credentials): Model[];
}
```

#### Token Refresh Race Condition Prevention

When multiple pi instances start with expired OAuth tokens, `proper-lockfile` prevents simultaneous refresh:

```typescript
async refreshOAuthTokenWithLock(providerId: string) {
  const result = await this.storage.withLockAsync(async (current) => {
    const currentData = JSON.parse(current);
    if (Date.now() < currentData[providerId].expires) {
      return { result: currentData[providerId].apiKey };  // Another instance refreshed
    }
    const refreshed = await getOAuthApiKey(providerId, currentData);
    return { result: refreshed, next: JSON.stringify(merged) };
  });
  return result;
}
```

#### Auth Storage Location

`~/.pi/agent/auth.json` with 0o600 permissions:

```typescript
constructor(authPath: string = join(getAgentDir(), "auth.json")) {
  // Creates parent dir with 0o700, file with 0o600
}
```

---

### 10. Session Persistence and Branching (`packages/coding-agent`, `packages/pods`)

**Priority:** P2

Tree-based session storage with JSONL files. Supports branching, compaction, labels, and session forking.

#### Session File Format

Sessions are JSONL (JSON Lines) with a tree structure:

```
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"...","message":{...}}
{"type":"compaction","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"...","summary":"..."}
```

#### Tree Data Structure

```typescript
class SessionManager {
  private leafId: string | null = null;  // Current position
  private fileEntries: FileEntry[] = [];  // All entries
  private byId: Map<string, SessionEntry> = new Map();

  appendMessage(message): string {
    const entry: SessionMessageEntry = {
      id: generateId(this.byId),
      parentId: this.leafId,
      // ...
    };
    this.leafId = entry.id;  // Advance leaf
    return entry.id;
  }

  branch(branchFromId: string): void {
    this.leafId = branchFromId;  // Move leaf back
    // Next append creates child of this earlier entry
  }
}
```

#### Entry Types

| Type | Purpose | In Context |
|------|---------|------------|
| `message` | User, assistant, tool result | Yes |
| `thinking_level_change` | Model thinking level | No |
| `model_change` | Provider/model switch | No |
| `compaction` | Summarized old context | Yes |
| `branch_summary` | Abandoned branch summary | Yes |
| `custom` | Extension state | No |
| `label` | Entry bookmark | No |

#### Session Persistence Strategy

Session files are not created until the first assistant response arrives. This avoids orphaned session files for sessions that never get a response:

```typescript
const hasAssistant = this.fileEntries.some(
  e => e.type === "message" && e.message.role === "assistant"
);
if (!hasAssistant) {
  this.flushed = false;  // Defer write
  return;
}
```

#### Clever Solutions

1. **ID Generation with Collision Check.** 8-char hex IDs with retry on collision:
   ```typescript
   function generateId(byId: { has(id: string): boolean }): string {
     for (let i = 0; i < 100; i++) {
       const id = randomUUID().slice(0, 8);
       if (!byId.has(id)) return id;
     }
     return randomUUID();  // Fallback
   }
   ```

2. **Symlink-Aware Collision Detection.** Uses `realpathSync()` to detect if same file accessed via different paths.

---

### 11. Slack Bot Integration (`packages/mom`)

**Priority:** P2

Slack bot (Master Of Mischief) that delegates to the coding agent with workspace memory and self-managing skills.

#### Architecture

Uses Socket Mode for real-time events:

```typescript
const socketClient = new SocketModeClient({ appToken: config.appToken });
const webClient = new WebClient(config.botToken);
```

**Per-Channel Queue.** Messages queue per-channel with sequential processing. Maximum 5 events queued (discards with warning if full):

```typescript
class ChannelQueue {
    enqueue(work: QueuedWork): void {
        this.queue.push(work);
        this.processNext();
    }
}
```

#### Workspace Structure

```
<working-dir>/
├── MEMORY.md                    # Global memory (all channels)
├── SYSTEM.md                    # Environment modifications log
├── skills/                      # Global CLI tools
├── events/                      # Scheduled events
├── C0A34FL8PMH/                 # Channel directory
│   ├── MEMORY.md                # Channel-specific memory
│   ├── log.jsonl                # Message history
│   ├── context.jsonl            # Agent session context
│   ├── attachments/             # Downloaded files
│   ├── scratch/                 # Working directory
│   └── skills/                  # Channel-specific tools
```

#### Memory System

```typescript
function getMemory(channelDir: string): string {
    const workspaceMemoryPath = join(channelDir, "..", "MEMORY.md");
    const channelMemoryPath = join(channelDir, "MEMORY.md");
    return parts.join("\n\n");
}
```

#### Events System

JSON files in `events/` directory support three types:
- `immediate` - Run immediately
- `one-shot` - Run once at specified time
- `periodic` - Run on cron schedule

#### Technical Debt

1. **Hardcoded model.** Uses `claude-sonnet-4-5` (issue #63 - make configurable).
2. **Incomplete implementation.** pi-pods agent integration is `throw new Error("Not implemented")`.

---

## Cross-Cutting Patterns

### Shared Dependencies

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

### Design Patterns

1. **Event-driven.** All features use event subscriptions for async updates.
2. **Registry pattern.** Providers, tools, extensions all use runtime registration.
3. **Contract-based errors.** Failures encoded in return types, not thrown.
4. **Tree over linear.** Session model explicitly represents branching conversation history.
5. **Locking for race conditions.** `proper-lockfile` prevents multiple processes from corrupting auth state during refresh.

### Recurring Technical Debt

1. **Hardcoded models.** pi-mom hardcodes `claude-sonnet-4-5`.
2. **Incomplete implementations.** pi-pods agent integration is not implemented.
3. **Large files.** `agent-session.ts`, `main.ts`, and `models.generated.ts` are oversized.

---

## Key Insights

1. **Lazy loading is first-class.** Used for providers, extensions, and optional dependencies to minimize bundle size and enable conditional loading.

2. **Event-driven enables reactive UI.** Fine-grained events (text_delta, toolcall_delta, etc.) allow responsive terminal and web interfaces.

3. **Extensibility via registration.** Tools, providers, commands, shortcuts all registered at runtime rather than configured statically.

4. **Session abstraction enables powerful features.** Branching, compaction, and cross-project forking all build on the tree-based session model.

5. **Standard compliance.** Agent Skills spec followed (leniently) for skills; Agent Skills standard compatible.

6. **OAuth with race condition prevention.** File locking ensures only one instance refreshes tokens at a time.

7. **Standard compliance where it matters.** Agent Skills standard followed strictly for skills naming/validation, but custom extensions to everything else.
