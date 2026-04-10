# Features Batch 2 Research

Research date: 2026-03-26
Features: 4. GPU Pod Management, 5. Terminal UI Framework, 6. Web UI Components, 7. Slack Bot Integration

---

## 4. GPU Pod Management (pi-pods)

### Core Implementation

**Package:** `packages/pods`

**Key Files:**
- `src/model-configs.ts` - Model configuration loader
- `src/commands/pods.ts` - Pod setup and management
- `src/commands/models.ts` - Model deployment commands
- `src/models.json` - Predefined model configurations
- `src/commands/prompt.ts` - Agent integration (INCOMPLETE)

### Tool Calling Configuration

The `models.json` file contains predefined configurations for agentic workloads. Each model specifies:

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

**Supported tool calling parsers:**
| Parser | Models | Notes |
|--------|--------|-------|
| `hermes` | Qwen2.5-Coder | |
| `qwen3_coder` | Qwen3-Coder, Qwen3-Coder-FP8 | |
| `glm45` | GLM-4.5, GLM-4.5-Air | Also has `--reasoning-parser glm45` |
| `kimi_k2` | Kimi-K2 | Requires vLLM v0.10.0rc1+ |
| `--async-scheduling` | GPT-OSS | Tools only via `/v1/responses` endpoint |

**Notable configurations:**
- `Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8` uses `--enable-expert-parallel --data-parallel-size 8` instead of tensor parallelism to avoid weight quantization errors
- `GLM-4.5` models default to thinking mode with `--reasoning-parser glm45`

### Deployment Flow

1. **Pod setup** (`setupPod`):
   - Tests SSH connection
   - Copies `scripts/pod_setup.sh` to remote
   - Runs setup with vLLM installation
   - Detects GPU configuration via `nvidia-smi`
   - Saves pod config to `~/.pi/pods.json`

2. **Model start** (`startModel`):
   - Validates GPU count against available hardware
   - Selects GPUs using round-robin based on current load (`selectGPUs`)
   - Customizes `model_run.sh` template with vLLM arguments
   - Uploads script via heredoc over SSH
   - Uses `script` command for pseudo-TTY and log preservation
   - Uses `setsid` for background process survival over SSH disconnection
   - Streams logs with startup detection and failure handling

### Clever Solutions

**GPU allocation strategy:** Round-robin selection based on current GPU usage across running models, ensuring balanced load.

**Background process survival:** Uses `setsid` to create new session, detached from SSH connection.

**Log streaming:** Uses `tail -f` over SSH with `script` command to preserve colors and capture logs.

**Memory/context overrides:** Filters and replaces vLLM args for `--gpu-memory-utilization` and `--max-model-len`.

### Technical Debt / Issues

**Incomplete agent integration:** The `promptModel` function in `src/commands/prompt.ts` throws "Not implemented" error:

```typescript
try {
    throw new Error("Not implemented");
} catch (err: any) {
    console.error(chalk.red(`Agent error: ${err.message}`));
    process.exit(1);
}
```

The CLI command `pi agent <name>` is defined in the help text but the actual agent prompting is not implemented.

### Error Handling

- OOM detection via `torch.OutOfMemoryError` and `CUDA out of memory` in log output
- vLLM engine initialization failure detection
- Process liveness verification on `pi list`
- Helpful suggestions printed on OOM failures

---

## 5. Terminal UI Framework (pi-tui)

### Core Implementation

**Package:** `packages/tui`

**Key Files:**
- `src/tui.ts` - Core TUI class with differential rendering
- `src/terminal.ts` - Terminal interface (ProcessTerminal)
- `src/stdin-buffer.ts` - Input buffering and sequence splitting
- `src/keys.ts` - Key parsing and Kitty protocol support
- `src/components/input.ts` - Input component with Emacs-style editing

### Three-Strategy Differential Rendering

The TUI implements three rendering strategies based on what changed:

1. **First render:** Full output without clearing (assumes clean screen)
2. **Width/height changes:** Full clear and re-render (wrapping can change)
3. **Content changes:** Incremental render from first changed line

**Synchronized output:** Uses CSI 2026 (`\x1b[?2026h` ... `\x1b[?2026l`) for atomic screen updates, preventing partial renders from showing.

### Differential Rendering Algorithm

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

Only changed lines are re-rendered, minimizing flicker.

### Viewport Tracking

```typescript
private maxLinesRendered = 0;  // Terminal's working area
private previousViewportTop = 0;

// Differential rendering can only touch what's visible
if (firstChanged < prevViewportTop) {
    fullRender(true);  // Need full redraw
    return;
}
```

### Termux Special Case

Height changes in Termux (when software keyboard shows/hides) do NOT trigger full redraws because it would cause entire history to replay. Check:

```typescript
if (heightChanged && !isTermuxSession()) {
    fullRender(true);
    return;
}
```

### Overlay System

Overlays use anchor-based positioning with percentage sizing:

```typescript
interface OverlayOptions {
    anchor?: "center" | "top-left" | "top-right" | "bottom-left" | "bottom-right" | ...;
    offsetX?: number;
    offsetY?: number;
    row?: SizeValue;      // number or "50%"
    col?: SizeValue;      // number or "50%"
    width?: SizeValue;    // number or "50%"
    maxHeight?: SizeValue;
    visible?: (termWidth: number, termHeight: number) => boolean;
    nonCapturing?: boolean;  // Don't capture keyboard focus
}
```

Compositing happens by rendering overlays, calculating positions, then splicing into base lines at appropriate columns.

### IME Support

**CURSOR_MARKER:** Zero-width APC sequence (`\x1b_pi:c\x07`) emitted by focused components at cursor position. TUI extracts this marker, positions hardware cursor there for proper IME candidate window placement.

```typescript
const cursorPos = this.extractCursorPosition(newLines, height);
// Then position hardware cursor at cursorPos.row, cursorPos.col
```

### StdinBuffer

Splits batched terminal input into individual sequences for correct key matching:

```typescript
const stdinBuffer = new StdinBuffer({ timeout: 10 });
stdinBuffer.on("data", (sequence) => {
    // Forward individual sequences
});
stdinBuffer.on("paste", (content) => {
    // Bracketed paste content \x1b[200~ ... \x1b[201~
});
```

### Bracketed Paste Mode

Enabled on terminal start: `\x1b[?2004h`. Pasted content is wrapped with `\x1b[200~...content...\x1b[201~` markers.

### Input Component

Full Emacs-style editing in a single-line input:
- Kill ring for deleted text
- Undo stack with coalescing
- Word-wise cursor movement using Intl.Segmenter
- Kitty CSI-u printable character decoding
- Grapheme-aware cursor positioning

### Crash Prevention

Before rendering, width overflow is detected and prevented:

```typescript
if (visibleWidth(line) > width) {
    // Log crash data and throw error
    throw new Error(`Rendered line ${i} exceeds terminal width...`);
}
```

### Environment Variables

- `PI_HARDWARE_CURSOR=1` - Show hardware cursor
- `PI_CLEAR_ON_SHRINK=1` - Clear empty rows when content shrinks
- `PI_TUI_WRITE_LOG=/path` - Log all terminal output
- `PI_DEBUG_REDRAW=1` - Log redraw reasons

---

## 6. Web UI Components (pi-web-ui)

### Core Implementation

**Package:** `packages/web-ui`

**Key Files:**
- `src/ChatPanel.ts` - Main chat panel with artifacts
- `src/components/AgentInterface.ts` - Agent interface component
- `src/components/MessageList.ts` - Message list rendering
- `src/components/StreamingMessageContainer.ts` - Streaming message display
- `src/components/MessageEditor.ts` - Input editor
- `src/tools/artifacts/` - Artifact renderers (HTML, SVG, Markdown, etc.)
- `src/storage/` - IndexedDB storage implementation

### Architecture

Built with **Lit** (web components library) using decorators. Main components:

```
ChatPanel (pi-chat-panel)
  ├── AgentInterface (agent-interface)
  │     ├── MessageList (message-list)
  │     ├── StreamingMessageContainer (streaming-message-container)
  │     └── MessageEditor (message-editor)
  └── ArtifactsPanel (artifacts-panel)
```

### Streaming Implementation

Uses pi-ai's `streamSimple` for streaming:

```typescript
import { streamSimple } from "@mariozechner/pi-ai";

// In AgentInterface.setupSessionSubscription:
this.session.streamFn = createStreamFn(async () => {
    const enabled = await getAppStorage().settings.get<boolean>("proxy.enabled");
    return enabled ? (await getAppStorage().settings.get<string>("proxy.url")) || undefined : undefined;
});
```

**Dual rendering:**
1. Stable `message-list` for completed messages (doesn't re-render during streaming)
2. `streaming-message-container` for the currently streaming message

### Session Management

```typescript
interface AgentInterface {
    @property({ attribute: false }) session?: Agent;

    async setAgent(agent: Agent, config?: {
        onApiKeyRequired?: (provider: string) => Promise<boolean>;
        onBeforeSend?: () => void | Promise<void>;
        onCostClick?: () => void;
        onModelSelect?: () => void;
        sandboxUrlProvider?: () => string;
        toolsFactory?: (agent, agentInterface, artifactsPanel, runtimeProvidersFactory) => AgentTool[];
    });
}
```

### Tool Result Rendering

Tool results are inlined in assistant messages using a registry:

```typescript
// Build map of tool results by ID
const toolResultsById = new Map<string, ToolResultMessage<any>>();
for (const message of state.messages) {
    if (message.role === "toolResult") {
        toolResultsById.set(message.toolCallId, message);
    }
}
```

Custom tool renderers can be registered via `registerToolRenderer`.

### Artifacts System

Supports sandboxed rendering of:

| Type | Renderer | Notes |
|------|----------|-------|
| HTML | `HtmlArtifact` | Sandboxed iframe |
| SVG | `SvgArtifact` | Inline SVG |
| Markdown | `MarkdownArtifact` | Rendered markdown |
| PDF | `PdfArtifact` | Preview component |
| DOCX | `DocxArtifact` | Text extraction |
| XLSX | `ExcelArtifact` | Sheet rendering |
| Images | `ImageArtifact` | Display with fallback |
| Generic | `GenericArtifact` | Fallback renderer |

Artifacts have their own tool (`artifacts-panel.tool`) and runtime providers (`ArtifactsRuntimeProvider`).

### Storage Backend

IndexedDB with multiple stores:

```typescript
// app-storage.ts
const stores = {
    sessions: SessionsStore,      // Conversation history
    providerKeys: ProviderKeysStore,  // API keys
    settings: SettingsStore,       // User preferences
    customProviders: CustomProvidersStore  // Custom LLM providers
};
```

### CORS Proxy Handling

```typescript
// proxy-utils.ts
import { createStreamFn } from "./utils/proxy-utils.js";

// Creates a stream function that routes through proxy if enabled
this.session.streamFn = createStreamFn(async () => {
    const enabled = await getAppStorage().settings.get<boolean>("proxy.enabled");
    return enabled ? (await getAppStorage().settings.get<string>("proxy.url")) || undefined : undefined;
});
```

### Attachment Handling

Supports: PDF, DOCX, XLSX, PPTX, images

```typescript
interface Attachment {
    id: string;
    name: string;
    mimeType: string;
    url: string;
    size: number;
    extractedText?: string;  // For documents
}
```

Uses `extract-document.ts` for text extraction from documents.

### Auto-scroll Behavior

```typescript
private _handleScroll = (_ev: any) => {
    const distanceFromBottom = scrollHeight - currentScrollTop - clientHeight;

    // Disable auto-scroll if user scrolled UP
    if (currentScrollTop !== 0 && currentScrollTop < lastScrollTop && distanceFromBottom > 50) {
        this._autoScroll = false;
    } else if (distanceFromBottom < 10) {
        this._autoScroll = true;  // Re-enable if close to bottom
    }
};
```

---

## 7. Slack Bot Integration (pi-mom)

### Core Implementation

**Package:** `packages/mom`

**Key Files:**
- `src/main.ts` - CLI entry point and handler setup
- `src/agent.ts` - Agent runner with Slack integration
- `src/slack.ts` - Slack bot with Socket Mode
- `src/store.ts` - Channel store and attachment handling
- `src/events.ts` - Event scheduling system
- `src/sandbox.ts` - Docker/host execution sandbox

### Slack Integration

Uses Socket Mode for real-time events:

```typescript
const MOM_SLACK_APP_TOKEN = process.env.MOM_SLACK_APP_TOKEN;
const MOM_SLACK_BOT_TOKEN = process.env.MOM_SLACK_BOT_TOKEN;

const socketClient = new SocketModeClient({ appToken: config.appToken });
const webClient = new WebClient(config.botToken);
```

**Event handling:**
- `app_mention` - Channel @mentions
- `message` - All messages including DMs

### Per-Channel Queue

Messages are queued per-channel with sequential processing:

```typescript
class ChannelQueue {
    private queue: QueuedWork[] = [];
    private processing = false;

    enqueue(work: QueuedWork): void {
        this.queue.push(work);
        this.processNext();
    }
}
```

**Queue limits:** Maximum 5 events queued per channel (discards with warning if full).

### Message Flow

1. **Receive event:** App mention or DM
2. **Log to file:** `logUserMessage()` writes to `log.jsonl`
3. **Check busy:** If channel already running, respond "Already working"
4. **Queue:** Otherwise, enqueue for sequential processing
5. **Run agent:** `handler.handleEvent()` creates context and runs agent
6. **Respond:** Main message + tool results in thread

### Workspace Structure

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

### Memory System

```typescript
function getMemory(channelDir: string): string {
    // Global workspace memory
    const workspaceMemoryPath = join(channelDir, "..", "MEMORY.md");

    // Channel-specific memory
    const channelMemoryPath = join(channelDir, "MEMORY.md");

    return parts.join("\n\n");
}
```

### Skills System

Custom CLI tools via `SKILL.md` files:

```markdown
---
name: skill-name
description: Short description
---

# Skill Name

Usage instructions
Scripts are in: {baseDir}/
```

Skills can be workspace-level (shared across channels) or channel-specific.

### Events System

JSON files in `events/` directory:

```typescript
// Event types
interface ImmediateEvent { type: "immediate"; channelId: string; text: string; }
interface OneShotEvent { type: "one-shot"; channelId: string; text: string; at: string; }
interface PeriodicEvent { type: "periodic"; channelId: string; text: string; schedule: string; timezone: string; }
```

**Limits:** Maximum 5 events queued.

### Sandbox Execution

```typescript
interface SandboxConfig {
    type: "host" | "docker";
    container?: string;
}

const executor = createExecutor(sandboxConfig);
```

- **Host:** Direct execution on host machine
- **Docker:** Execution inside Alpine Linux container

### Agent Integration

The agent is built from pi-coding-agent components:

```typescript
const agent = new Agent({
    initialState: {
        systemPrompt,
        model,  // Hardcoded: anthropic/claude-sonnet-4-5
        thinkingLevel: "off",
        tools,
    },
    convertToLlm,
    getApiKey: async () => getAnthropicApiKey(authStorage),
});
```

**Note:** Model is hardcoded (TODO: make configurable per issue #63).

### Slack Formatting

Uses mrkdwn (not Markdown):

```typescript
// Format: Bold: *text*, Italic: _text_, Code: `code`, Block: ```code```
const channelMappings = channels.map(c => `${c.id}\t#${c.name}`).join("\n");
```

### Backfill on Startup

When bot starts, it backfills recent messages from channels that have existing `log.jsonl`:

```typescript
private async backfillChannel(channelId: string): Promise<number> {
    // Fetches messages newer than latest in log.jsonl
    // Limits to 3 pages (3000 messages)
    // Filters: mom's messages, user messages, skips duplicates
}
```

### Attachment Handling

Attachments are downloaded asynchronously:

```typescript
// In logUserMessage:
const attachments = event.files
    ? this.store.processAttachments(event.channel, event.files, event.ts)
    : [];
```

Files stored in `attachments/` directory with timestamps.

### Silent Responses

Agents can respond with `[SILENT]` to delete the message instead of posting:

```typescript
if (finalText.trim() === "[SILENT]") {
    await ctx.deleteMessage();
}
```

### Thinking Support

Thinking parts are extracted and posted to both main message and thread:

```typescript
for (const part of content) {
    if (part.type === "thinking") {
        thinkingParts.push(part.thinking);
    }
}
```

---

## Cross-Feature Observations

### Shared Dependencies

1. **pi-ai** - Used by all features requiring LLM access (pods agent, web-ui, mom)
2. **pi-agent-core** - Used by web-ui and mom for agent runtime
3. **pi-tui** - Used by coding-agent for terminal interface

### Design Patterns

1. **Event-driven:** All features use event subscriptions for async updates
2. **IndexedDB storage:** Both web-ui and mom use storage backends
3. **Streaming:** Both web-ui and mom support streaming responses
4. **Tool abstraction:** Tools are registered via registries in both web-ui and mom

### Technical Debt (Recurring)

1. **Hardcoded models:** pi-mom hardcodes `claude-sonnet-4-5` (issue #63)
2. **Incomplete implementation:** pi-pods agent integration is `throw new Error("Not implemented")`

### Interesting Patterns

1. **Round-robin GPU allocation** in pods for load balancing
2. **CSI 2026 synchronized output** for atomic terminal updates
3. **StdinBuffer** for input sequence parsing with timeout
4. **CURSOR_MARKER** for IME positioning
5. **Per-channel queue** in mom with sequential processing
6. **Backfill on startup** for message history reconstruction
