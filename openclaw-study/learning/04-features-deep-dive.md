# OpenClaw Features Deep Dive

**Project:** OpenClaw - Personal AI Assistant
**Study Root:** `/Users/sheldon/Documents/claw/openclaw-study`
**Source:** Batch feature reports (05a-05d) and feature index (04-features-index.md)
**Date:** 2026-03-26

---

## Core Features (Priority Tier 1)

---

## 1. Multi-Channel Messaging Hub

### Feature Overview

The Multi-Channel Messaging Hub connects 20+ messaging platforms through a unified plugin architecture. It serves as the "front door" for user interactions, accepting inbound messages from diverse channels and delivering outbound responses.

### How It Works

**Plugin Registration System**

Channels register via `defineChannelPluginEntry()` in `src/plugin-sdk/core.ts`. The registry maintains two arrays:
- `channelSetups`: All registered plugins (for setup wizard rendering)
- `channels`: Fully activated plugins (for runtime operations)

```typescript
export function defineChannelPluginEntry<TPlugin>({
  id, name, description, plugin, configSchema, setRuntime, registerFull
}) {
  return definePluginEntry({
    id, name, description, configSchema,
    register(api: OpenClawPluginApi) {
      setRuntime?.(api.runtime);
      api.registerChannel({ plugin: plugin as ChannelPlugin });
      if (api.registrationMode !== "full") return;
      registerFull?.(api);
    },
  });
}
```

**Channel Abstraction**

All chat channels use `createChatChannelPlugin()` which composes:
1. Base metadata: `id`, `meta`, `setup`, `setupWizard`, `capabilities`
2. Security: DM policy (`resolveDmPolicy`, `resolveAllowFrom`), group warnings
3. Pairing: Text-based pairing with `idLabel`, `message`, `notify()`
4. Threading: `topLevelReplyToMode` per channel
5. Outbound: `deliveryMode`, `chunker`, `sendPayload`, `sendText`, `sendMedia`, `sendPoll`

**Session Routing**

Outbound session routing builds a `ChannelOutboundSessionRoute`:
```typescript
{
  sessionKey: "agent:main:acp:uuid",
  baseSessionKey: "...",
  peer: { kind: "direct" | "group" | "channel", id: "..." },
  chatType: "direct" | "group" | "channel",
  from: "telegram:12345",
  to: "telegram:67890",
  threadId: "..."
}
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/channels/registry.ts` | Channel metadata, normalization |
| `src/plugin-sdk/core.ts` | `createChatChannelPlugin()`, `defineChannelPluginEntry()` |
| `src/plugins/registry.ts` | Plugin registry, `registerChannel()` |
| `extensions/discord/src/channel.ts` | Discord plugin (700+ lines) |
| `extensions/telegram/src/channel.ts` | Telegram plugin (800+ lines) |
| `extensions/whatsapp/src/channel.ts` | WhatsApp plugin (400+ lines) |

**Target Normalization Patterns:**
```typescript
// Discord
looksLikeDiscordTargetId(target: string): boolean // /\d+/

// Telegram
parseTelegramTarget(raw: string): { chatId, chatType, messageThreadId }

// WhatsApp
normalizeWhatsAppTarget(raw: string): string | null // E.164 or JID
```

### Notable Patterns

1. **Lazy runtime loading**: Discord provider uses `loadDiscordProviderRuntime()` with promise caching to avoid eager loading heavy Discord.js dependency
2. **Channel-specific text chunking**: Telegram uses `chunkMarkdownText()` with 4000 char limit; Discord uses plain 2000 char chunks
3. **ACP conversation binding**: Each channel implements `compileConfiguredBinding` and `matchInboundConversation` for thread binding
4. **Dual registration mode**: `setup-only` (wizard rendering) vs `full` (runtime activation)

### Technical Debt / Concerns

1. **Massive channel.ts files**: Discord channel.ts is 700+ lines, Telegram is 800+ lines - monolithic files
2. **Inconsistent patterns**: WhatsApp uses `runtime-api.js` for shared exports; Discord has its own `runtime-api.ts`
3. **Plugin boundary violations**: Extensions import from `openclaw/plugin-sdk/*` but some internal modules are imported directly
4. **WhatsApp Web-based implementation**: Uses QR code login requiring persistent browser session - fragile

### Learning Value

This feature demonstrates:
- **Plugin architecture patterns**: How to build a unified interface for disparate external systems
- **Lazy loading strategies**: Handling heavy dependencies without impacting cold startup
- **Channel-specific customization**: Balancing unified abstractions with platform quirks
- **Session key design**: How routing keys can encode rich context (channel, user, thread)

---

## 2. Local-First Gateway Control Plane

### Feature Overview

The Gateway is a WebSocket-based control plane running on `ws://127.0.0.1:18789` by default. It serves as the central hub for sessions, channels, tools, cron jobs, and device nodes - the nervous system of OpenClaw.

### How It Works

**Server Stack**
```
server-http.ts       - HTTP server (webhooks, health checks)
server-ws-runtime.ts - WebSocket upgrade handling
server.impl.ts       - Main gateway implementation (2000+ lines)
server-methods.ts    - Core gateway methods
server-plugins.ts    - Plugin integration
```

**Channel Manager**
```typescript
createChannelManager({
  loadConfig,           // Config getter
  channelLogs,          // Per-channel loggers
  channelRuntimeEnvs,   // Runtime environments
  channelRuntime?,       // Optional Plugin SDK channel runtime
  resolveChannelRuntime? // Lazy resolver
}) => ChannelManager

// Interface
interface ChannelManager {
  getRuntimeSnapshot(): ChannelRuntimeSnapshot;
  startChannels(): Promise<void>;
  startChannel(channel: ChannelId, accountId?: string): Promise<void>;
  stopChannel(channel: ChannelId, accountId?: string): Promise<void>;
  markChannelLoggedOut(channelId, cleared, accountId?): void;
  isManuallyStopped(channelId, accountId): boolean;
}
```

**Boot Process**

`boot.ts` implements a startup check system:
```typescript
runBootOnce({
  cfg, deps, workspaceDir, agentId?
}): Promise<BootRunResult>

// Reads ~/.openclaw/workspace/BOOT.md
// Prompts Pi agent with BOOT.md content
// Uses SILENT_REPLY_TOKEN to suppress output if no action needed
```

The boot session uses a generated ID (`boot-{timestamp}-{uuid}`) and snapshots/restores main session mappings to avoid side effects.

**Session Lifecycle**
```typescript
type SessionLifecycleState =
  | "pending"
  | "input"
  | "thinking"
  | "streaming"
  | "tool_use"
  | "completed"
  | "error";
```

**Session key structure**:
```
agent:{agentId}:{mode}:{sessionId}
agent:main:acp:{uuid}           // Main ACP session
agent:{agentId}:cli:{uuid}      // CLI session
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/gateway/server-http.ts` | HTTP server with webhook handlers |
| `src/gateway/server.impl.ts` | Main gateway implementation |
| `src/gateway/server-channels.ts` | Channel lifecycle manager |
| `src/gateway/boot.ts` | Startup boot check system |
| `src/gateway/server-ws-runtime.ts` | WebSocket handling |
| `config/sessions/` | Session storage, main session resolution |

**Gateway Methods** (exposed via WebSocket/HTTP):
```typescript
const coreGatewayHandlers = {
  "sessions.patch": handleSessionsPatch,
  "sessions.send": handleSessionsSend,
  "sessions.history": handleSessionsHistory,
  "agent": handleAgentMessage,
  "tools.invoke": handleToolsInvoke,
  "channels.start": handleChannelsStart,
  "channels.stop": handleChannelsStop,
  ...
}
```

### Notable Patterns

1. **Health endpoints**: `/health` (liveness) and `/ready` (readiness) probes
2. **Channel health monitor**: Background monitoring with configurable intervals and callbacks
3. **Config hot-reload**: Sophisticated handling of mutable config with clean reload
4. **Session snapshot/restore**: Boot process isolates its work from main session state

### Technical Debt / Concerns

1. **Server monolithic**: `server.impl.ts` is 2000+ lines handling initialization
2. **Config hot-reload complexity**: `config-reload.ts` and `server-reload-handlers.ts` show tension between mutable config and clean reload
3. **Session store race conditions**: Multiple write paths to session store with potential race conditions
4. **Memory pressure**: Sessions store full transcripts; no automatic archival

### Learning Value

This feature demonstrates:
- **WebSocket server design**: How to structure a real-time communication hub
- **Session lifecycle management**: State machines for complex session flows
- **Health monitoring**: Background tasks with event callbacks
- **Boot-time orchestration**: How to sequence startup checks without blocking

---

## 3. AI Agent Runtime (Pi Agent)

### Feature Overview

The Pi Agent is an RPC-based runtime that processes messages, executes tools, and coordinates responses. It uses `@mariozechner/pi-agent-core` and `@mariozechner/pi-ai` as underlying engines. This is the "brain" that understands user intent and orchestrates responses.

### How It Works

**Agent Spawning - ACP**

```typescript
spawnAcpDirect(
  params: {
    task: string;           // Task description
    label?: string;         // Human-readable label
    agentId?: string;      // Target agent (defaults to config)
    resumeSessionId?: string;
    cwd?: string;
    mode?: "run" | "session";  // Oneshot vs persistent
    thread?: boolean;       // Thread-bound spawn
    sandbox?: "inherit" | "require";
    streamTo?: "parent";   // Stream output to parent
  },
  ctx: SpawnAcpContext
): Promise<SpawnAcpResult>
```

**Block Reply Pipeline**

```typescript
createBlockReplyPipeline({
  onBlockReply(payload, { abortSignal?, timeoutMs? }): Promise<void>;
  timeoutMs: number;
  coalescing?: BlockStreamingCoalescing;
  buffer?: BlockReplyBuffer;
}) => BlockReplyPipeline

// Pipeline stages:
// 1. buffer (optional) - buffers until condition met
// 2. coalesce - batches text chunks to reduce network calls
// 3. send - delivers to channel
```

**Coalescing configuration**:
```typescript
{
  maxChars: 200,           // Flush when buffered text exceeds
  minChars: 50,            // Minimum before flush
  idleMs: 1000,            // Flush after idle period
  joiner: "\n\n",          // Text joiner
  flushOnParagraph: true   // Flush at paragraph boundary
}
```

**Context Engine Registry**

```typescript
registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory,
  owner: string,
  opts?: { allowSameOwnerRefresh?: boolean }
): ContextEngineRegistrationResult
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/agents/acp-spawn.ts` | ACP session spawning (900+ lines) |
| `src/agents/agent-command.ts` | Agent message processing (43KB, 900+ lines) |
| `src/context-engine/registry.ts` | Context engine factory registry |
| `src/auto-reply/reply/block-reply-pipeline.ts` | Streaming pipeline with coalescing |
| `src/auto-reply/reply/block-streaming.ts` | Chunking configuration |
| `src/agents/lanes.js` | Lane system for session isolation |

**Agent Command Processing Flow** (`agent-command.ts`):
1. **Ingress**: `agentCommand()` receives message + session key
2. **Session resolution**: `resolveSession()` loads/creates session entry
3. **Model selection**: `resolveConfiguredModelRef()`, `resolveDefaultModelForAgent()`
4. **Auth profiles**: `ensureAuthProfileStore()` for API key rotation
5. **Embedded runner**: `runEmbeddedPiAgent()` executes with tools
6. **Response delivery**: `deliverAgentCommandResult()` sends to channel

### Notable Patterns

1. **Idempotency keys**: ACP spawn uses `crypto.randomUUID()` as idempotency key to prevent duplicate runs
2. **Session snapshot/restore**: Boot process snapshots main session mapping, runs agent, restores - avoids side effects
3. **ACP stream relay**: Fast failures caught before relay registration; gateway returns `runId` that updates relay
4. **Legacy compat proxy**: Context engine registry wraps engines with sessionKey compat proxy that auto-detects API shape
5. **ACP visible text accumulator**: Handles silent reply tokens, cumulative snapshots

### Technical Debt / Concerns

1. **Massive files**: `agent-command.ts` is 43KB+ monolithic, hard to navigate
2. **ACP spawn complexity**: 900+ line function with deep nested logic for thread binding, stream relay, delivery planning
3. **Session store mutations**: `mergeSessionEntry()` with `OVERRIDE_FIELDS_CLEARED_BY_DELETE` is fragile
4. **Auth profile leakage**: Session-level auth profile overrides persist across sessions if not cleared

### Learning Value

This feature demonstrates:
- **Streaming coalescing**: Batching network writes to reduce chat spam
- **Idempotency design**: Preventing duplicate work with unique keys
- **Session isolation**: Using lanes to separate agent contexts
- **Context engine abstraction**: Factory pattern for model-agnostic AI runtime

---

## 4. Voice Wake + Talk Mode

### Feature Overview

Voice interaction system that allows hands-free wake word activation on macOS/iOS and continuous voice input on Android. Uses ElevenLabs for voice synthesis with system TTS fallback.

### How It Works

**TTS Architecture**

```
TTS Providers:
├── ElevenLabs (primary - premium voices)
├── OpenAI TTS (gpt-4o-mini-tts, tts-1, tts-1-hd)
├── Edge TTS (free Microsoft voices - fallback)
└── Custom OpenAI-compatible endpoints (Kokoro, LocalAI)
```

**Directive System**

TTS supports inline directives in text that override settings:
```
[[tts:text]]Custom spoken text[[/tts:text]]
[[tts:voice=ash]]
[[tts:voiceid=pMsXgVXv3BLzUgSXRplE]]
[[tts:stability=0.5]]
[[tts:model=eleven_multilingual_v2]]
[[tts:language=en]]
```

**Voice Wake Infrastructure**

Global wake words stored at `~/.openclaw/settings/voicewake.json`:
- Default triggers: `["openclaw", "claude", "computer"]`
- Gateway-owned global list (not per-node)
- Syncs across all connected nodes via WebSocket broadcast

Protocol:
- `voicewake.get` - retrieve current triggers
- `voicewake.set` - update triggers (broadcasts to all clients)
- `voicewake.changed` - event broadcast on update

**macOS Voice Wake Components:**
- `VoiceWakeRuntime.swift` - Core wake word detection
- `VoiceWakeOverlay.swift` - Overlay UI component
- `VoiceWakeSettings.swift` - Settings UI
- `VoicePushToTalk.swift` - Push-to-talk hotkey support
- `VoiceSessionCoordinator.swift` - Coordinates voice sessions
- `MicLevelMonitor.swift` - Audio level visualization

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/tts/tts.ts` (29KB) | Main TTS orchestration |
| `src/tts/tts-core.ts` (21KB) | TTS provider implementations |
| `src/tts/provider-registry.ts` | Speech provider registration |
| `extensions/elevenlabs/` | ElevenLabs speech provider plugin |
| `extensions/voice-call/` | Voice call extension (Twilio, Telnyx, Plivo) |
| `src/infra/voicewake.ts` | Global wake word management |
| `apps/macos/Sources/OpenClaw/VoiceWake/` | macOS voice wake implementation (13 files) |

**Output Formats:**
- Voice notes: Opus 48kHz/64kbps (`.opus`)
- General TTS: MP3 44.1kHz/128kbps (`.mp3`)
- Telephony: PCM 22.05kHz

### Notable Patterns

1. **Directive-based overrides**: Agents embed `[[tts:voice=ash]]` in responses for precise control
2. **Auto-summarization**: Long text is AI-summarized before synthesis (prevents truncation)
3. **Voice-compatible format detection**: Automatically selects Opus for voice messages
4. **Provider fallback chain**: Tries ElevenLabs, then OpenAI, then Edge TTS
5. **Custom endpoint support**: Allows Kokoro/LocalAI at OpenAI-compatible endpoints
6. **Global wake words**: Single source of truth synced across all nodes

### Technical Debt / Concerns

1. **No wake word engine**: No Porcupine/SpeechRecognition in core; wake detection is macOS-native only
2. **Android uses manual mic**: No wake word on Android; uses push-to-talk in Voice tab
3. **Twilio streaming reconnection**: 2000ms grace period for stream reconnects (some calls may get stuck)
4. **No per-node wake word customization**: Global list only; cannot customize per device

### Learning Value

This feature demonstrates:
- **Provider abstraction**: Building fallback chains across multiple service providers
- **Directive parsing**: Embedding control signals in natural language output
- **Cross-device synchronization**: WebSocket broadcast for shared state
- **Native platform integration**: Swift for macOS voice capabilities

---

## 5. Live Canvas Visual Workspace

### Feature Overview

Agent-driven visual workspace rendering a controllable canvas interface via A2UI (Agent-to-UI) protocol. Rendered in macOS/iOS/Android apps using native WebViews.

### How It Works

**Canvas Host Routes**
```typescript
const A2UI_PATH = "/__openclaw__/a2ui";      // A2UI protocol handler
const CANVAS_HOST_PATH = "/__openclaw__/canvas"; // Local file serving
const CANVAS_WS_PATH = "/__openclaw__/ws";   // Live reload WebSocket
```

**A2UI Protocol (v0.8)**

Server-to-Client Messages:
```typescript
{ "beginRendering": { "surfaceId": "main", "root": "rootId" } }
{ "surfaceUpdate": { "surfaceId": "...", "components": [...] } }
{ "dataModelUpdate": { "surfaceId": "...", "data": {...} } }
{ "deleteSurface": { "surfaceId": "..." } }
```

**Component System:**
- Declarative UI via component tree
- Components: `Column`, `Text`, `Row`, `Image`, etc.
- `usageHint` for semantic role hints
- Data binding via explicitList

Client-to-Server Actions:
```typescript
window.openclawSendUserAction()  // Injected into all HTML
```

**Canvas URL Scheme:**
```
openclaw-canvas://<session>/<path>
```
- `openclaw-canvas://main/` maps to `<canvasRoot>/main/index.html`
- Directory traversal blocked for security

**Canvas Tool Actions:**
```typescript
const CANVAS_ACTIONS = [
  "present",   // Show canvas panel
  "hide",      // Hide canvas panel
  "navigate",  // Navigate to URL/path
  "eval",      // Execute JavaScript
  "snapshot",  // Capture screenshot
  "a2ui_push", // Push A2UI JSONL
  "a2ui_reset" // Reset A2UI state
];
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/canvas-host/a2ui.ts` | A2UI protocol implementation |
| `src/canvas-host/` | Canvas server (file serving, live reload) |
| `apps/macos/Sources/OpenClaw/Canvas/` | macOS Canvas (12+ files) |
| `src/agents/tools/canvas-tool.ts` | Agent tool interface |
| `apps/shared/OpenClawKit/` | Shared Swift code |

**macOS Canvas Components:**
- `CanvasManager.swift` - Canvas lifecycle management
- `CanvasWindowController.swift` - Window management
- `CanvasWindowController+Window.swift` - Window positioning
- `CanvasChromeContainerView.swift` - Chrome-less browser container
- `CanvasSchemeHandler.swift` - Custom `openclaw-canvas://` scheme
- `CanvasA2UIActionMessageHandler.swift` - A2UI action bridge
- `CanvasFileWatcher.swift` - Local file change detection

### Notable Patterns

1. **Custom URL scheme**: `openclaw-canvas://` for local files without loopback server
2. **Platform bridge abstraction**: Single code path works on iOS (webkit) and Android
3. **A2UI declarative UI**: Agent sends component tree, client renders
4. **Live reload**: File watcher + WebSocket for instant preview during development
5. **Dual serve paths**: Separate paths for A2UI protocol vs local canvas files

### Technical Debt / Concerns

1. **v0.8 only**: `createSurface` (v0.9) not implemented; limits dynamic surface creation
2. **No per-component interactivity**: Actions only at surface level
3. **JSONL streaming**: No streaming support; entire JSONL must fit in memory
4. **Canvas disabled when not configured**: No graceful fallback
5. **Single panel limitation**: Cannot show multiple canvases simultaneously

### Learning Value

This feature demonstrates:
- **Declarative UI protocols**: Agent sending component trees to clients
- **Custom URL schemes**: Security-conscious file access patterns
- **Live reload architecture**: File watching + WebSocket for development experience
- **Platform abstraction**: Single protocol targeting multiple native WebViews

---

## 6. Browser Control (CDP)

### Feature Overview

Dedicated OpenClaw-managed Chrome/Chromium browser instance with Chrome DevTools Protocol (CDP) control. Enables web navigation, screenshots, form filling, and web interactions.

### How It Works

**Chrome Launch Args**
```typescript
[
  `--remote-debugging-port=${profile.cdpPort}`,
  `--user-data-dir=${userDataDir}`,
  "--no-first-run",
  "--no-default-browser-check",
  "--disable-sync",
  "--disable-background-networking",
  "--disable-component-update",
  "--disable-features=Translate,MediaRouter",
  "--disable-session-crashed-bubble",
  "--hide-crash-restore-bubble",
  "--password-store=basic",
]
```

**CDP Protocol**

WebSocket communication:
```typescript
// Message protocol
{ id: number, method: string, params?: object }
{ id: number, result?: unknown }
{ id: number, error?: { message: string } }

// CDP commands
await send("Page.enable")
await send("Runtime.evaluate", { expression: "...", returnByValue: true })
await send("Target.createTarget", { url: "..." })
await send("Accessibility.getFullAXTree")
```

**Browser Tool Actions:**
```typescript
const BROWSER_ACT_REQUEST_KINDS = [
  "click", "dblclick", "tripleclick",
  "hover", "scroll", "snap",
  "fill", "check", "uncheck", "select",
  "submit", "press", "type",
  "goto", "back", "forward", "reload",
  "screenshot", "pdf", "innerText", "evaluate",
  "waitForSelector", "waitForTimeout",
  "uploadFile", "setInputFiles",
  "openTab", "closeTab", "switchTab"
];
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/browser/cdp.ts` + `cdp.helpers.ts` | CDP protocol helpers |
| `src/browser/chrome.ts` | Chrome process spawning |
| `src/browser/client.ts` | Browser API client |
| `src/browser/profiles.ts` | Profile management |
| `src/browser/navigation-guard.ts` | URL security policies |
| `src/browser/chrome-mcp.ts` | Chrome DevTools MCP bridge |
| `src/browser/pw-session.ts` + `pw-tools-core*.ts` | Playwright integration |
| `src/agents/tools/browser-tool.ts` | Agent tool interface |

**Profile Configuration:**
```typescript
{
  name: string,
  cdpPort: number,
  cdpUrl: string,
  userDataDir?: string,
  color: string,  // DevTools color indicator
  driver: "openclaw" | "existing-session",
  attachOnly: boolean
}
```

**REST API:**
```typescript
browserStatus(baseUrl?, { profile? })      // GET /:profile
browserProfiles(baseUrl?)                   // GET /profiles
browserStart(baseUrl?, { profile? })       // POST /start/:profile
browserStop(baseUrl?, { profile? })        // POST /stop/:profile
browserResetProfile(baseUrl?, { profile? }) // POST /reset/:profile
browserTabs(baseUrl?, { profile? })        // GET /tabs/:profile
browserOpenTab(baseUrl?, { profile?, url? }) // POST /tabs/:profile
browserCloseTab(baseUrl?, { profile?, targetId }) // DELETE /tabs/:profile/:targetId
```

### Notable Patterns

1. **CDP over WebSocket**: Direct browser protocol without Playwright overhead for simple operations
2. **Profile isolation**: Separate user data dirs enable multiple concurrent browser sessions
3. **Chrome MCP bridge**: Uses official chrome-devtools-mcp for structured accessibility snapshots
4. **Role refs cache**: Stable element references even across page reloads
5. **WS URL normalization**: Handles containerized browser URL rewriting
6. **Navigation guard**: SSRF protection for browser automation

### Technical Debt / Concerns

1. **No browserless integration**: Only local Chrome supported (no cloud browser)
2. **Single Chrome process**: Cannot run multiple isolated browsers
3. **No headless toggle per session**: Global headless setting
4. **Playwright for interactions**: CDP simpler but less battle-tested
5. **No browser extension support**: Cannot inject content scripts
6. **Profile cleanup on reset**: Removes entire user data dir

### Learning Value

This feature demonstrates:
- **Protocol-level browser automation**: Direct CDP without higher-level abstractions
- **Profile isolation**: Separate browser contexts for different purposes
- **Accessibility tree traversal**: Capturing DOM state for AI understanding
- **SSRF protection patterns**: Validating URLs before navigation

---

## 7. Command-Line Interface

### Feature Overview

Full-featured CLI (`openclaw`) for gateway management, agent interaction, message sending, channel configuration, device pairing, plugin management, and more. Built on Node.js 22+ with sophisticated bootstrap logic.

### How It Works

**Entry Point Architecture**

`openclaw.mjs` (181 lines):
- Shebang: `#!/usr/bin/env node`
- Node.js version guard: Requires 22.12.0+
- Module compile cache enabled via `module.enableCompileCache()`
- Fast-path for `--help` with precomputed root help text
- Graceful fallback chain: `dist/entry.js` -> `dist/entry.mjs` -> `src/entry.ts` (source)

**Bootstrap Flow**
```
openclaw.mjs
  -> installProcessWarningFilter()
  -> tryImport(dist/entry.js)
  -> tryImport(dist/entry.mjs)
  -> throw Error (missing dist guidance)
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `openclaw.mjs` | CLI entry point |
| `src/cli/run-main.ts` | Main CLI runner |
| `src/cli/program/` | Commander.js root program |
| `src/cli/gateway-cli/` | Gateway subcommands |
| `src/cli/plugins-cli/` | Plugin management |
| `src/cli/config-cli.ts` | Config management |
| `src/cli/secrets-cli.ts` | Secrets management |
| `src/cli/nodes-cli/` | Node management |
| `src/cli/pairing-cli.ts` | Device pairing |

**CLI Commands:**
| Command | Purpose |
|---------|---------|
| `openclaw onboard` | Step-by-step setup wizard |
| `openclaw gateway` | Gateway management |
| `openclaw agent` | Agent interaction |
| `openclaw message send` | Direct messaging |
| `openclaw channels` | Channel management |
| `openclaw devices` | Device/node pairing |
| `openclaw plugins` | Plugin installation/management |
| `openclaw config` | Configuration management |
| `openclaw doctor` | Troubleshooting diagnostics |

### Notable Patterns

1. **Precomputed Help Text**: `--help` fast-path loads from JSON instead of running full CLI bootstrap
2. **Warning Filter**: Bootstrap warnings are filtered to match TypeScript runtime behavior
3. **Module Resolution**: Uses ESM-only imports with `await import()` for lazy loading
4. **Source Fallback**: If dist is missing but src exists, provides actionable error message

### Technical Observations

- The CLI relies heavily on Commander.js patterns
- Complex version detection and fallback logic in entry point
- No visible CLI test files in initial directory listing

### Learning Value

This feature demonstrates:
- **CLI bootstrap engineering**: Version checks, compile cache, graceful degradation
- **Help text optimization**: Fast-paths for common operations
- **Commander.js patterns**: Building composable CLI applications

---

## Secondary Features (Priority Tier 2)

---

## 8. Device Nodes (macOS/iOS/Android Companion Apps)

### Feature Overview

Companion apps that pair with the Gateway to expose device-local capabilities. Each device runs as a "node" advertising capabilities over WebSocket with TLS pinning.

### How It Works

**macOS App Architecture**

Menu Bar Extra App (SwiftUI):
- Uses `MenuBarExtra` for menu bar presence
- `NSApplicationDelegate` with `@MainActor` SwiftUI app
- Extensive modules: VoiceWake, Canvas, Gateway, Sessions, Settings

Key components:
| Component | Files | Purpose |
|-----------|-------|---------|
| VoiceWake | 15+ | Wake word detection, PTT overlay |
| Canvas | 12+ | Canvas window, scheme handler |
| Gateway | 30+ | Launch agent, connection, discovery |
| Sessions | 15+ | Session management |
| Settings | 40+ | Config UI, schema support |

**iOS App Architecture**

SwiftUI with UIKit bridges:
- Uses `Observation` framework (`@Observable`) per CLAUDE.md guidelines
- `OpenClawAppDelegate` for push notifications

Key components:
- `RootCanvas.swift` - Main canvas surface
- `Gateway/` - Gateway connection
- `Voice/` - Voice Wake + Talk Mode
- `Camera/` - Camera capture
- `Chat/` - Chat sessions
- `Onboarding/` - Setup wizard

**Android App Architecture**

Kotlin with Jetpack Compose:

Key components:
| Component | Purpose |
|-----------|---------|
| `NodeApp.kt` | Application class with runtime singleton |
| `MainActivity.kt` | Main entry with Compose UI |
| `MainViewModel.kt` | View model (41KB) |
| `NodeRuntime.kt` | Core runtime (41KB) |
| `node/` (25 files) | Node handlers |

**Android-Specific Capabilities:**
```
InvokeCommandRegistry.advertisedCapabilities:
- Canvas (always)
- Device (always)
- Notifications (always)
- System (always)
- Camera (CameraEnabled)
- SMS (SmsAvailable)
- VoiceWake (VoiceWakeEnabled)
- Location (LocationEnabled)
- Photos (always)
- Contacts (always)
- Calendar (always)
- Motion (MotionAvailable)
- CallLog (CallLogAvailable)
```

### Key Implementation Details

| Platform | Location | Key Technology |
|----------|----------|-----------------|
| macOS | `apps/macos/Sources/OpenClaw/` | Swift, SwiftUI, MenuBarExtra |
| iOS | `apps/ios/Sources/` | Swift, SwiftUI, Observation |
| Android | `apps/android/app/src/main/java/` | Kotlin, Jetpack Compose |
| Shared | `apps/shared/OpenClawKit/` | Swift Package (76 files) |

**Device Pairing Flow:**
1. App discovers gateway via Bonjour (`GatewayDiscovery`)
2. QR code / setup code generated via `openclaw qr`
3. TLS handshake with certificate pinning
4. Device auth store saves credentials
5. `InvokeCommandRegistry` advertises runtime capabilities
6. Gateway approves device via `openclaw devices approve`

### Notable Patterns

1. **TLS Pinning**: Supports both automatic TOFU and manual fingerprint verification
2. **Capability Advertisement**: Runtime flags determine which commands are available
3. **Invoke Dispatcher**: Central routing for all node commands
4. **Foreground Service**: Android uses `NodeForegroundService` for background operation
5. **SecurePrefs**: Android encrypted preferences for sensitive data

### Learning Value

This feature demonstrates:
- **Native platform development**: Swift for Apple platforms, Kotlin for Android
- **WebSocket security**: TLS pinning and certificate handling
- **Capability-based routing**: Advertising and routing based on device capabilities
- **Cross-platform code sharing**: OpenClawKit Swift package

---

## 9. Skills Platform (Extensible Capability Packages)

### Feature Overview

Skills are modular, self-contained packages that extend the AI agent's capabilities through specialized knowledge, workflows, and tool integrations. They follow a markdown-first design philosophy.

### How It Works

**Skill Anatomy**
```
skills/<name>/
├── SKILL.md (required) - YAML frontmatter + markdown instructions
├── scripts/ (optional) - Executable code (Python/Bash/etc.)
├── references/ (optional) - Documentation for context loading
├── assets/ (optional) - Files for output (templates, icons)
└── bin/ (optional) - Binary executables
```

**SKILL.md Frontmatter Schema:**
```yaml
---
name: skill-name
description: "When to use this skill + specific triggers"
metadata:
  openclaw:
    emoji: "..."
    requires:
      bins: ["gh"]        # CLI tools needed
      env: ["API_KEY"]    # Environment variables
      config: ["path"]     # Config files
    install:
      - id: brew
        kind: brew
        formula: gh
---
```

**Progressive Disclosure Design:**
1. **Metadata** (~100 words) - Always in context for triggering
2. **SKILL.md body** (<5k words) - Loaded after skill triggers
3. **Bundled resources** - Loaded as needed by agent

### Key Implementation Details

| Skill | Category | Technology |
|-------|----------|------------|
| `skills/github/` | Coding | `gh` CLI, JSON with `--json --jq` |
| `skills/notion/` | Notes | Notion API v2025-09-03 |
| `skills/slack/` | Communication | JSON tool actions |
| `skills/coding-agent/` | Coding | Delegates to Codex, Claude Code, Pi |
| `skills/spotify-player/` | Media | CLI integration |
| `skills/mcporter/` | Platform | MCP bridge CLI |

**ClawHub Integration:**
```bash
clawhub search "postgres backups"    # Search registry
clawhub install my-skill            # Install from registry
clawhub update my-skill             # Hash-based update
clawhub publish ./my-skill          # Publish to registry
```

### Notable Patterns

1. **Bash-First**: Many skills are bash command wrappers (e.g., `github`, `weather`, `summarize`)
2. **CLI Integration**: Skills often wrap existing CLI tools (`gh`, `mcporter`, `clawhub`)
3. **No Code in Skills**: Most skills are pure documentation with bash examples
4. **Token Efficiency**: SKILL.md kept lean with references to external docs
5. **Metadata-Driven**: Frontmatter drives skill selection and dependency checking

### Technical Observations

1. **Skills are Documentation-Heavy**: 53 skills but most contain only `SKILL.md`
2. **Bundled Resources Rare**: Only a few skills have `scripts/`, `references/`, or `bin/`
3. **Skill Creator as Meta-Skill**: `skill-creator` provides comprehensive authoring guidance
4. **Workspace Skills**: Users can add skills to `~/.openclaw/workspace/skills/`
5. **No Runtime Skill Code**: Skills are instructions for the AI, not executable code

### Learning Value

This feature demonstrates:
- **Documentation-as-code**: Using markdown for AI-instructable content
- **Progressive disclosure**: Layering information for token efficiency
- **Skill composition**: Building complex capabilities from simple parts
- **Package distribution**: ClawHub as a skill marketplace

---

## 10. Security & Sandboxing

### Feature Overview

Comprehensive security model with secure defaults, pairing-based access control, per-session Docker sandboxing, and fine-grained tool permissions. Spans 40 files in `src/security/` and 55 files in `src/secrets/`.

### How It Works

**Audit System**

`SecurityAuditReport` provides deep security scanning:
- **Filesystem checks**: Permission auditing, Windows ACL validation
- **Channel security**: Per-channel allowlist enforcement
- **Dangerous config detection**: Flags insecure flag combinations
- **Gateway probing**: Optional deep probe to verify gateway connectivity
- **Tool policy enforcement**: Sandbox tool allowlists/denylists

Key design: Lazy-loaded audit modules via dynamic imports to keep cold startup fast.

**DM Policy & Access Control**

```typescript
resolveDmGroupAccessDecision:
  - dmPolicy: "open" | "pairing" | "allowlist" | "disabled"
  - groupPolicy: "open" | "allowlist" | "disabled"
  - Effective allowFrom merged from config + pairing store
  - Returns: { decision: "allow" | "block" | "pairing", reasonCode, reason }
```

Key insight: The pairing store is DM-only; group auth explicitly does NOT inherit DM pairing approvals.

**Docker Sandbox**

Backend types:
- `docker-backend.ts`: Docker containers (default for non-main sessions)
- `ssh-backend.ts`: SSH-based remote sandbox
- `browser.ts`: Browser automation sandbox with VNC/noVNC

**Tool Policy**
```typescript
resolveSandboxToolPolicyForAgent:
  - allow: string[] (empty = allow all)
  - deny: string[] (denied tools)
  - alsoAllow: string[] (extra permissions)
  - Priority: agent config > global config > defaults
```

**Secrets Management**

Provider types:
- `env`: Environment variables
- `file`: JSON/env files (`~/.openclaw/secrets/`)
- `exec`: External secret executables

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/security/audit.ts` | Main audit engine |
| `src/security/dm-policy-shared.ts` | Access control decisions |
| `src/security/external-content.ts` | Prompt injection detection |
| `src/security/dangerous-tools.ts` | Tool risk registry |
| `src/security/fix.ts` | Auto-remediation |
| `src/agents/sandbox/docker.ts` | Docker container execution |
| `src/agents/sandbox/tool-policy.ts` | Per-agent tool permissions |
| `src/secrets/configure.ts` | Secrets setup wizard |
| `src/secrets/resolve.ts` | Secret ref resolution |

**Suspicious Pattern Detection:**
```typescript
SUSPICIOUS_PATTERNS:
  - ignore previous instructions
  - forget everything
  - new instructions:
  - system prompt override
  - exec/rm -rf commands
  - <system> tags
```

### Notable Patterns

1. **Lazy module loading**: Security audit modules loaded on-demand via dynamic imports
2. **Centralized danger constants**: One source of truth for tool risks
3. **Failover-safe errors**: Typed failover errors bubble up properly
4. **Windows-first spawn handling**: Docker spawn uses `windows-spawn.js` for proper Windows support
5. **Automated remediation**: `fix.ts` can chmod/icacls to correct permissions

### Technical Debt / Concerns

1. **Sandbox Docker dependency**: Requires Docker in PATH; friendly error exists but Docker must be pre-installed
2. **Complex allowlist merging**: Tool policy merging logic is intricate with multiple precedence rules
3. **Test file sizes**: `audit.test.ts` is 125KB+ - very large test file

### Learning Value

This feature demonstrates:
- **Defense in depth**: Multiple security layers (pairing, sandboxing, tool policy)
- **Prompt injection detection**: Recognizing malicious patterns in external content
- **Secret management**: Abstraction over multiple secret backends
- **Automated remediation**: Self-healing security configurations

---

## 11. Media Pipeline

### Feature Overview

Unified media processing pipeline for images, audio, and video with transcription hooks, size caps, and temp file lifecycle management. Spans `src/media/` (50 files), `src/media-understanding/` (58 files), and `src/image-generation/` (11 files).

### How It Works

**Media Store**
```typescript
MEDIA_MAX_BYTES = 5 * 1024 * 1024  // 5MB default
MEDIA_FILE_MODE = 0o644  // Intentional readable for Docker sandbox containers
DEFAULT_TTL_MS = 2 * 60 * 1000  // 2 minutes temp file TTL
```

**Image Processing**

Uses `sharp` library (with `sips` fallback on macOS/Bun):

Capabilities:
- EXIF orientation reading (handles JPEG rotation)
- HEIC->JPEG conversion
- Image resizing with quality reduction steps: [85, 75, 65, 55, 45, 35]
- Alpha channel detection
- PNG optimization

Backend selection:
```typescript
prefersSips():
  - env OPENCLAW_IMAGE_BACKEND=sips
  - or (Bun + macOS + no explicit backend)
```

**Media Understanding Pipeline**

Applies media understanding to message contexts:

**Capability order**: image -> audio -> video (processed in sequence)

**Supported MIME types**:
- Images: All standard image types
- Audio: audio/* plus text transcripts
- Video: video/* plus frame extraction
- Files: text/* types (CSV, TSV, Markdown, logs, config, JSON, YAML, XML)

**Transcript handling**:
- Audio: Extracted via transcription runner
- Video: Frame extraction + transcription
- Format: Human-readable transcript blocks

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/media/store.ts` | Media file lifecycle |
| `src/media/image-ops.ts` | Image processing with sharp |
| `src/media/web-media.ts` | Remote media fetching |
| `src/media-understanding/apply.ts` | Media understanding pipeline |
| `src/media-understanding/runner.ts` | Model orchestration |
| `src/image-generation/runtime.ts` | Image generation with fallback |

**Image Generation Runtime:**
```typescript
generateImage():
  - Candidates: primary + fallback models
  - Per-provider: generateImage() call
  - Error handling: FailoverError with reason/status/code
  - Result: { images, provider, model, attempts, metadata }
```

### Notable Patterns

1. **Temp file lifecycle**: Short 2-minute TTL for temp media files
2. **Docker-readable permissions**: Media files use 0o644 so containers can access
3. **HEIC as first-class citizen**: Native HEIC detection and conversion
4. **Capability-based routing**: Media understanding decides which models to use
5. **Sharp with Bun/sips fallback**: Graceful degradation

### Technical Debt / Concerns

1. **Provider env var coupling**: Image generation hard-codes env var patterns per provider
2. **Complex MIME sniffing**: Multiple detection strategies
3. **Limited video support**: Video understanding relies on frame extraction + transcription

### Learning Value

This feature demonstrates:
- **File lifecycle management**: Automatic temp file cleanup
- **Provider fallback chains**: Multiple backend options with graceful degradation
- **MIME type detection**: Content sniffing for ambiguous files
- **Image optimization pipelines**: Multi-step compression with quality selection

---

## 12. Automation (Cron, Webhooks, MCP)

### Feature Overview

Automation capabilities including scheduled cron jobs, webhook triggers, and MCP (Model Context Protocol) integration. Spans `src/cron/` (79 files) with service/or jobs/timer/store subsystems.

### How It Works

**Cron Service**
```typescript
class CronService:
  - start() / stop()
  - list() / listPage()
  - add() / update() / remove()
  - run() / enqueueRun()  // Manual trigger
  - wake()  // Session wakeup
  - getJob()
```

**Job State Machine**
```typescript
CronJob:
  - schedule: at | every | cron
  - sessionTarget: main | isolated | current | session:N
  - payload: systemEvent | agentTurn
  - delivery: none | announce | webhook
  - failureAlert: optional alert config
  - state: nextRunAtMs, runningAtMs, lastRunAtMs, lastStatus...
```

**Schedule Types:**
```typescript
{ kind: "at", at: string }              // One-time
{ kind: "every", everyMs: number }     // Recurring interval
{ kind: "cron", expr: string, tz?: string }  // Cron expression
```

**Webhook Handling**

```typescript
handleHookHttpRequest():
  - Token extraction
  - Body parsing (JSON)
  - Agent routing via hook config
  - Rate limiting (20 failures per 60s window)
  - External content source detection (gmail vs webhook)
```

**MCP Integration** (via mcporter CLI):
```bash
mcporter list [--schema]
mcporter call <server.tool> key=value
mcporter auth <server> [--reset]
mcporter daemon start|status|stop|restart
mcporter generate-cli --server <name>
```

### Key Implementation Details

| File | Purpose |
|------|---------|
| `src/cron/service.ts` | Cron service entry |
| `src/cron/service/ops.ts` | CRUD operations |
| `src/cron/service/timer.ts` | Timer management (largest file, 41KB) |
| `src/cron/service/jobs.ts` | Job state machine |
| `src/cron/schedule.ts` | Cron parsing with croner |
| `src/gateway/server-http.ts` | Webhook HTTP handlers |
| `skills/mcporter/SKILL.md` | MCP CLI documentation |

**Delivery System:**
```typescript
CronDelivery:
  mode: "none" | "announce" | "webhook"
  channel?: ChannelId | "last"
  to?: string
  accountId?: string
  bestEffort?: boolean
  failureDestination?: CronFailureDestination
```

### Notable Patterns

1. **croner library**: Leverages `croner` for cron expression parsing with timezone support and caching
2. **Heartbeat-based wake**: Cron jobs can "wake" sessions via heartbeat mechanism
3. **Delivery awareness**: Jobs track delivery status separately from execution status
4. **Stagger for cron**: Built-in staggerMs to spread cron job execution
5. **Gmail Pub/Sub**: Native Gmail webhook integration
6. **Run log retention**: Completed run logs pruned automatically

### Technical Debt / Concerns

1. **Timer persistence**: Timers are in-memory; restart requires job state reload
2. **Heartbeat coupling**: Job wake mode depends on session heartbeat system being active
3. **Delivery failure handling**: Best-effort delivery can silently fail
4. **mcporter external dependency**: MCP integration requires separate mcporter binary

### Learning Value

This feature demonstrates:
- **Cron expression parsing**: Library integration for complex schedules
- **Job state machines**: Persisting and resuming long-running tasks
- **Webhook security**: HMAC token verification and rate limiting
- **MCP bridging**: Connecting to external tool ecosystems

---

## Cross-Cutting Architecture Patterns

### Message Flow

```
Channel Plugin (e.g., Telegram)
  |
  v
gateway/server-http.ts (HTTP webhook) / server-ws-runtime.ts (WebSocket)
  |
  v
server-chat.ts (message handling)
  |
  v
agent-command.ts (Pi agent processing)
  |
  v
Tool execution (via tools-invoke-http.ts)
  |
  v
Block reply pipeline (coalescing, chunking)
  |
  v
Channel outbound adapter (sendMessageTelegram, etc.)
  |
  v
Channel Plugin (delivery to user)
```

### Shared Infrastructure

1. **WebSocket communication**: Used by Canvas (live reload), Browser (CDP), Gateway (RPC), Voice Wake (sync)
2. **Profile systems**: Browser uses profiles; Canvas uses sessions
3. **Node routing**: Both Canvas and Browser tools route through device nodes
4. **Gateway RPC**: All tools invoke via `node.invoke()` gateway command
5. **Image capture**: Both Canvas and Browser support snapshot/screenshot

### Security Integration Points

1. External content wrapping for webhooks/email (prompt injection detection)
2. Sandbox tool policy enforcement for cron job execution
3. Secret resolution for credential management
4. Gateway HTTP tool denylist for invoke endpoint

---

## Summary: Feature Complexity Matrix

| Feature | Complexity | Files | Key Technology | Standout Pattern |
|---------|------------|-------|----------------|------------------|
| Multi-Channel Messaging | High | ~20 plugins | Plugin SDK, Discord.js, Telegraf | Lazy runtime loading |
| Gateway Control Plane | High | 100+ | WebSocket, Hono | Boot-time orchestration |
| AI Agent Runtime | High | 642 | Pi Agent Core, streaming | Block coalescing |
| Voice Wake + Talk | Medium | 50+ | ElevenLabs API, Swift | Directive parsing |
| Live Canvas | Medium | 30+ | A2UI protocol, WKWebView | Declarative UI |
| Browser Control | Medium | 139 | CDP, Playwright | Profile isolation |
| CLI | Low | 196 | Commander.js, Node 22 | Bootstrap optimization |
| Device Nodes | High | 300+ | Swift, Kotlin | TLS pinning |
| Skills Platform | Low | 53 skills | Markdown, bash | Progressive disclosure |
| Security & Sandboxing | High | 95+ | Docker, secrets | Defense in depth |
| Media Pipeline | Medium | 119 | Sharp, Whisper | Provider fallback |
| Automation | Medium | 79 | Croner, MCP | State machine |

---

## Learning Priorities

For someone studying this codebase:

1. **Start with Gateway** (`src/gateway/`) - It's the central hub everything connects to
2. **Then Messaging** (`extensions/telegram/src/channel.ts`) - Large but self-contained plugin example
3. **Agent Runtime** (`src/agents/`) - The brain; start with `agent-command.ts` flow
4. **Security** (`src/security/`) - Understand the threat model before diving deep
5. **Device Nodes** - Only if you need native platform integration

---

*Document synthesized from batch reports 05a-05d and feature index 04-features-index.md*
