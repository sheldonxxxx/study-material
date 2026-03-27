# Feature Batch Report: Features 4-6

**Project:** OpenClaw - Personal AI Assistant
**Source:** `src/`, `extensions/`, `apps/`, `docs/`
**Date:** 2026-03-26
**Batch:** Voice Wake + Talk Mode, Live Canvas Visual Workspace, Browser Control (CDP)

---

## Feature 4: Voice Wake + Talk Mode

### Overview

Voice interaction system supporting hands-free wake word activation on macOS/iOS and continuous voice input on Android. Integrates ElevenLabs for voice synthesis with system TTS fallback.

### Key Components

#### 1. TTS Core (`src/tts/`)

**Files:**
- `tts.ts` (29KB) - Main TTS orchestration
- `tts-core.ts` (21KB) - TTS provider implementations
- `provider-registry.ts` - Speech provider registration

**Architecture:**
```
TTS Providers:
├── ElevenLabs (primary - premium voices)
├── OpenAI TTS (gpt-4o-mini-tts, tts-1, tts-1-hd)
├── Edge TTS (free Microsoft voices - fallback)
└── Custom OpenAI-compatible endpoints (Kokoro, LocalAI)
```

**Directive System:**
TTS supports inline directives in text that override settings:
```
[[tts:text]]Custom spoken text[[/tts:text]]
[[tts:voice=ash]]
[[tts:voiceid=pMsXgVXv3BLzUgSXRplE]]
[[tts:stability=0.5]]
[[tts:model=eleven_multilingual_v2]]
[[tts:language=en]]
```

**Output Formats:**
- Voice notes: Opus 48kHz/64kbps (`.opus`)
- General TTS: MP3 44.1kHz/128kbps (`.mp3`)
- Telephony: PCM 22.05kHz

**Text Handling:**
- Max text length: 1500 chars (configurable)
- Auto-summarization via AI when text exceeds limit
- Markdown stripping before synthesis
- 5-minute temp file cleanup

#### 2. ElevenLabs Extension (`extensions/elevenlabs/`)

**Speech Provider Plugin:**
- Registers via `api.registerSpeechProvider()`
- Models: `eleven_multilingual_v2`, `eleven_turbo_v2_5`, `eleven_monolingual_v1`
- `listVoices()` - fetches voice catalog from API
- `synthesize()` - generates audio buffer
- `synthesizeTelephony()` - PCM output for phone calls

**Voice Settings:**
```typescript
{
  stability: 0.5,
  similarityBoost: 0.75,
  style: 0.0,
  useSpeakerBoost: true,
  speed: 1.0
}
```

#### 3. Voice Wake Infrastructure (`src/infra/voicewake.ts`)

**Global Wake Words:**
- Stored at `~/.openclaw/settings/voicewake.json`
- Default triggers: `["openclaw", "claude", "computer"]`
- Gateway-owned global list (not per-node)
- Syncs across all connected nodes via WebSocket broadcast

**Protocol:**
- `voicewake.get` - retrieve current triggers
- `voicewake.set` - update triggers (broadcasts to all clients)
- `voicewake.changed` - event broadcast on update

#### 4. macOS Voice Wake (`apps/macos/Sources/OpenClaw/`)

**Swift Components (13 files):**
- `VoiceWakeRuntime.swift` - Core wake word detection
- `VoiceWakeOverlay.swift` - Overlay UI component
- `VoiceWakeOverlayController+Window.swift` - Window management
- `VoiceWakeOverlayView.swift` - Visual overlay
- `VoiceWakeSettings.swift` - Settings UI
- `VoiceWakeForwarder.swift` - Forwards wake events to gateway
- `VoiceWakeChime.swift` - Audio feedback
- `VoicePushToTalk.swift` - Push-to-talk hotkey support
- `VoiceSessionCoordinator.swift` - Coordinates voice sessions
- `TalkModeRuntime.swift` - Android-style continuous voice
- `MicLevelMonitor.swift` - Audio level visualization
- `PermissionManager.swift` - macOS permissions handling

**Key Features:**
- Global hotkey for push-to-talk
- Visual overlay showing transcription
- Wake word detection gates activation
- Settings sync with gateway via `voicewake.get/set`
- Chime on successful wake detection

#### 5. Voice Call Extension (`extensions/voice-call/`)

**Providers:**
- Twilio (Programmable Voice + Media Streams)
- Telnyx (Call Control v2)
- Plivo (Voice API + XML transfer)
- Mock (local dev)

**Architecture:**
- Runs inside Gateway process
- Webhook-based inbound call handling
- Streaming audio via WebSocket

**Features:**
- Outbound notification calls
- Multi-turn conversation mode
- Inbound allowlist policy
- TTS integration for spoken responses
- Auto-retry failed speech playback
- Stale call reaper (prevents stuck calls)

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Voice Wake Flow                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  macOS App                    Gateway                    Agent    │
│  ┌──────────┐               ┌──────────┐              ┌────────┐ │
│  │ Wake Word│───hotkey──────│voicewake│              │  Pi    │ │
│  │Detector  │               │ .get/set│              │ Agent  │ │
│  └────┬─────┘               └────┬─────┘              └────┬───┘ │
│       │                          │                        │     │
│       │ transcribe                │ broadcast              │     │
│       ▼                          ▼                        ▼     │
│  ┌──────────┐               ┌──────────┐              ┌────────┐ │
│  │ Overlay  │───forward─────▶│ Gateway  │────message──▶│ Tool  │ │
│  │ (PTT)    │               │ WS       │              │ Exec  │ │
│  └──────────┘               └──────────┘              └────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     TTS Flow                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Agent Response                                                   │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────┐    ┌─────────────┐    ┌──────────────────┐          │
│  │ [[tts:]]│───▶│ Directive   │───▶│ Provider Select  │          │
│  │directive│   │  Parser     │    │ (11Labs/OpenAI/  │          │
│  └─────────┘    └─────────────┘    │  Edge)           │          │
│                                    └────────┬─────────┘          │
│                                             │                    │
│                                             ▼                    │
│                                    ┌──────────────────┐          │
│                                    │ Audio Buffer     │          │
│                                    │ (.mp3/.opus)     │          │
│                                    └────────┬─────────┘          │
│                                             │                    │
│                              ┌──────────────┼──────────────┐     │
│                              ▼              ▼              ▼     │
│                        ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│                        │ Channel  │   │ Voice    │   │ Voice    │ │
│                        │ Delivery │   │ Call     │   │ Message  │ │
│                        └──────────┘   └──────────┘   └──────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Clever Solutions

1. **Directive-based overrides** - Agents embed `[[tts:voice=ash]]` in responses for precise control
2. **Auto-summarization** - Long text is AI-summarized before synthesis (prevents truncation)
3. **Voice-compatible format detection** - Automatically selects Opus for voice messages
4. **Provider fallback chain** - Tries ElevenLabs, then OpenAI, then Edge TTS
5. **Custom endpoint support** - Allows Kokoro/LocalAI at OpenAI-compatible endpoints
6. **Global wake words** - Single source of truth synced across all nodes

### Technical Debt / Shortcuts

1. **No wake word engine** - No Porcupine/SpeechRecognition in core; wake detection is macOS-native only
2. **Android uses manual mic** - No wake word on Android; uses push-to-talk in Voice tab
3. **Twilio streaming reconnection** - 2000ms grace period for stream reconnects (some calls may get stuck)
4. **No per-node wake word customization** - Global list only; cannot customize per device

### Error Handling

- API key validation before requests
- Voice ID format validation (`/^[a-zA-Z0-9]{10,40}$/`)
- Timeout handling (30s default, configurable)
- Empty audio detection (throws error if 0 bytes)
- Abort controller for cancellable requests
- Fallback to next provider on failure

---

## Feature 5: Live Canvas Visual Workspace

### Overview

Agent-driven visual workspace rendering a controllable canvas interface via A2UI (Agent-to-UI) protocol. Rendered in macOS/iOS/Android apps using native WebViews.

### Key Components

#### 1. Canvas Host (`src/canvas-host/`)

**Server Architecture:**
```typescript
// Paths
const A2UI_PATH = "/__openclaw__/a2ui";      // A2UI protocol handler
const CANVAS_HOST_PATH = "/__openclaw__/canvas"; // Local file serving
const CANVAS_WS_PATH = "/__openclaw__/ws";   // Live reload WebSocket
```

**Request Handling:**
1. A2UI requests routed to `handleA2uiHttpRequest()`
2. Canvas file requests routed to `handler.handleHttpRequest()`
3. WebSocket upgrade for live reload

**File Resolution:**
Multiple candidate paths for development flexibility:
- Running from source: `src/canvas-host/a2ui/`
- Running from dist: `dist/canvas-host/a2ui/`
- Running from bundled assets

**Live Reload:**
- Chokidar file watcher on canvas root
- WebSocket broadcast on file changes
- 75ms debounce to batch rapid changes
- Injects reload snippet into HTML responses

#### 2. A2UI Protocol (`src/canvas-host/a2ui.ts`)

**Protocol Version:** v0.8 (v0.9 `createSurface` not supported)

**Server-to-Client Messages:**
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

**Client-to-Server Actions:**
- `userAction` events via `window.openclawSendUserAction()`
- iOS: `webkit.messageHandlers.openclawCanvasA2UIAction.postMessage()`
- Android: `window.openclawCanvasA2UIAction.postMessage()`

**Action Bridge (injected into all HTML):**
```javascript
globalThis.OpenClaw.postMessage = postToNode;
globalThis.OpenClaw.sendUserAction = sendUserAction;
globalThis.openclawPostMessage = postToNode;
globalThis.openclawSendUserAction = sendUserAction;
```

#### 3. macOS Canvas (`apps/macos/Sources/OpenClaw/`)

**Swift Components:**
- `CanvasManager.swift` - Canvas lifecycle management
- `CanvasWindowController.swift` - Window management
- `CanvasWindowController+Window.swift` - Window positioning
- `CanvasWindowController+Navigation.swift` - URL navigation
- `CanvasChromeContainerView.swift` - Chrome-less browser container
- `CanvasSchemeHandler.swift` - Custom `openclaw-canvas://` scheme
- `CanvasA2UIActionMessageHandler.swift` - A2UI action bridge
- `CanvasFileWatcher.swift` - Local file change detection

**Canvas URL Scheme:**
```
openclaw-canvas://<session>/<path>
```
- `openclaw-canvas://main/` → `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css`
- Directory traversal blocked for security

**Panel Behavior:**
- Borderless, resizable panel near menu bar
- Remembers size/position per session
- Only one Canvas panel visible at a time
- Can be disabled via Settings

#### 4. Canvas Tool (`src/agents/tools/canvas-tool.ts`)

**Actions:**
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

**Tool Schema:**
```typescript
{
  action: "present" | "hide" | "navigate" | "eval" | "snapshot" | "a2ui_push" | "a2ui_reset",
  gatewayUrl?: string,
  gatewayToken?: string,
  timeoutMs?: number,
  node?: string,
  // present
  target?: string,
  x?: number,
  y?: number,
  width?: number,
  height?: number,
  // navigate
  url?: string,
  // eval
  javaScript?: string,
  // snapshot
  outputFormat?: "png" | "jpg" | "jpeg",
  maxWidth?: number,
  quality?: number,
  delayMs?: number,
  // a2ui_push
  jsonl?: string,
  jsonlPath?: string
}
```

**CLI Integration:**
```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> --url "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
openclaw nodes canvas a2ui push --jsonl /tmp/commands.jsonl --node <id>
```

#### 5. Canvas Server Routes (`src/browser/routes/`)

Canvas uses the same route infrastructure as browser for agent commands.

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Canvas Flow                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Agent                        Gateway                     macOS   │
│  ────                         ───────                     ─────   │
│    │                             │                           │     │
│    │ canvas.present()            │                           │     │
│    │────────────────────────────▶│                           │     │
│    │                             │ node.invoke(canvas.       │     │
│    │                             │ present)                   │     │
│    │                             │───────────────────────────▶│     │
│    │                             │                           │     │──┐
│    │                             │                           │     │  │
│    │ canvas.snapshot()           │                           │     │  │ Create
│    │────────────────────────────▶│                           │     │  │ WKWebView
│    │                             │                           │◀────│──┘
│    │                             │                           │     │
│    │                             │  http://gateway:18789      │     │
│    │                             │  /__openclaw__/a2ui/       │     │
│    │                             │◀───────────────────────────│     │
│    │                             │                           │     │
│    │                             │                           │◀────│──
│    │                             │                           │     │  Load
│    │                             │                           │     │  A2UI
│    │                             │                           │     │  HTML
│    │                             │                           │◀────│──
│    │                             │                           │     │
│    │ a2ui_push (JSONL)          │                           │     │
│    │────────────────────────────▶│                           │     │
│    │                             │ WebSocket / HTTP push      │     │
│    │                             │────────────────────────────▶│     │
│    │                             │                           │     │
│    │                             │                           │◀────│──
│    │                             │                           │     │  Render
│    │                             │                           │     │  Surface
│    │                             │                           │◀────│──
│    │                             │                           │     │
│    │ userAction from JS          │                           │     │
│    │◀────────────────────────────│                           │     │
│    │                             │                           │     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Clever Solutions

1. **Custom URL scheme** - `openclaw-canvas://` for local files without loopback server
2. **Platform bridge abstraction** - Single code path works on iOS (webkit) and Android
3. **A2UI declarative UI** - Agent sends component tree, client renders
4. **Live reload** - File watcher + WebSocket for instant preview during development
5. **Dual serve paths** - Separate paths for A2UI protocol vs local canvas files
6. **Canvas snapshot via gateway** - Image captured server-side, returned as base64

### Technical Debt / Shortcuts

1. **v0.8 only** - `createSurface` (v0.9) not implemented; limits dynamic surface creation
2. **No per-component interactivity** - Actions only at surface level
3. **JSONL streaming** - No streaming support; entire JSONL must fit in memory
4. **Canvas disabled when not configured** - No graceful fallback
5. **Single panel limitation** - Cannot show multiple canvases simultaneously

### Error Handling

- Path traversal blocked in scheme handler
- JSONL path validation against media roots
- File existence checks before navigation
- WebSocket connection error recovery
- Snapshot timeout handling

---

## Feature 6: Browser Control (CDP)

### Overview

Dedicated OpenClaw-managed Chrome/Chromium browser instance with Chrome DevTools Protocol (CDP) control. Enables web navigation, screenshots, form filling, and web interactions.

### Key Components

#### 1. Browser Module (`src/browser/`)

**139 files** organized into:

| Directory | Purpose |
|-----------|---------|
| `cdp*.ts` | CDP protocol helpers, WebSocket management |
| `chrome*.ts` | Chrome process spawning, lifecycle |
| `client*.ts` | Browser API client |
| `config*.ts` | Configuration resolution |
| `navigation-guard*.ts` | URL allowlist/blocklist |
| `paths*.ts` | File path resolution |
| `profiles*.ts` | Browser profile management |
| `pw-*.ts` | Playwright integration |
| `routes/` | HTTP route handlers |
| `server-context*.ts` | Server-side context |
| `screenshot*.ts` | Screenshot capture |

#### 2. Chrome Launch (`src/browser/chrome.ts`)

**Launch Args:**
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

**Platform-Specific:**
- Linux: `--disable-dev-shm-usage`
- Headless: `--headless=new --disable-gpu`
- No sandbox: `--no-sandbox --disable-setuid-sandbox`

**Process Management:**
- Spawn via Node.js `child_process.spawn`
- Bootstrap timeout: waits for CDP readiness
- Preference timeout: waits for profile decoration
- PID tracking for cleanup

#### 3. CDP Protocol (`src/browser/cdp.ts` + `cdp.helpers.ts`)

**WebSocket Communication:**
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

**URL Normalization:**
Handles containerized browsers (browserless):
- Rewrites `ws://0.0.0.0:<port>` to external host
- Protocol upgrade (ws → wss for https)
- Auth credential passing

**Screenshot:**
```typescript
captureScreenshot({ wsUrl, fullPage?: boolean, format?: "png" | "jpeg" })
captureScreenshotPng({ wsUrl, fullPage?: boolean })
```

#### 4. Profile System (`src/browser/profiles*.ts`)

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

**Profile Storage:**
- `~/.openclaw/browser/<profile>/user-data/`
- Per-profile CDP port allocation
- Color-coded DevTools for identification

#### 5. Chrome MCP (`src/browser/chrome-mcp.ts`)

**Chrome DevTools MCP Bridge:**
Uses `chrome-devtools-mcp@latest` npm package as MCP server.

**Transport:**
- StdioClientTransport for local processes
- Session-based tool invocation

**Commands:**
- `npx -y chrome-devtools-mcp@latest --autoConnect`
- `--experimentalStructuredContent` for structured output
- `--experimental-page-id-routing`

**Sessions:**
```typescript
const sessions = new Map<string, ChromeMcpSession>();
// Profile name → MCP session
```

#### 6. Navigation Guard (`src/browser/navigation-guard.ts`)

**Security Policy:**
- Allowlist/blocklist for URLs
- SSRF protection for CDP endpoints
- Redirect chain validation
- Result URL verification

**Policy Types:**
```typescript
{ allowPrivateNetwork: boolean }
{ allowedHostnames: string[] }
{ hostnameAllowlist: string[] }
```

#### 7. Playwright Integration (`src/browser/pw-session.ts` + `pw-tools-core*.ts`)

**Session Management:**
```typescript
cachedBrowser = Playwright.chromium.connectOverCDP(wsUrl)
page = browser.newPage({ CDPSession })
```

**Interactions (`pw-tools-core.interactions.ts`):**
```typescript
clickViaPlaywright({ ref?: string, selector?: string, ... })
fillViaPlaywright({ ref?, selector?, value })
hoverViaPlaywright({ ref?, selector? })
scrollIntoViewViaPlaywright({ ref?, selector? })
selectViaPlaywright({ ref?, selector?, values })
```

**Snapshot Modes:**
- `aria` - Accessibility tree (via `Accessibility.getFullAXTree`)
- `ai` - Custom DOM snapshot (via `Runtime.evaluate`)
- `dom` - Lightweight DOM representation

#### 8. Browser Tool (`src/agents/tools/browser-tool.ts`)

**Actions:**
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

**Tool Schema:**
```typescript
{
  action: string,        // Action kind
  request?: object,     // Action parameters
  targetId?: string,     // Tab target ID
  profile?: string,      // Browser profile
  timeoutMs?: number,
  // Legacy flat params
  ref?: string, url?: string, text?: string, ...
}
```

**Node Routing:**
- Prefers remote nodes with browser capability
- Falls back to local gateway browser
- CLI: `openclaw browser ...`

#### 9. Browser Client (`src/browser/client.ts`)

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

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Browser Control Flow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Agent                       Gateway                      Chrome  │
│  ────                        ───────                     ──────  │
│    │                           │                           │      │
│    │ browser.navigate(url)     │                           │      │
│    │──────────────────────────▶│                           │      │
│    │                           │                           │      │
│    │                           │ Target.createTarget       │      │
│    │                           │──────────────────────────▶│      │
│    │                           │                           │      │──┐
│    │                           │                           │      │  │ Create
│    │                           │                           │◀─────│──┘
│    │                           │ { targetId }              │      │
│    │                           │◀───────────────────────────│      │
│    │                           │                           │      │
│    │                           │ Page.navigate             │      │
│    │                           │───────────────────────────▶│      │
│    │                           │                           │◀─────│──
│    │                           │                           │      │  Navigate
│    │                           │                           │◀─────│──
│    │                           │                           │      │
│    │ browser.snapshot()        │                           │      │
│    │──────────────────────────▶│                           │      │
│    │                           │                           │      │
│    │                           │ Accessibility.getFullAXTree│      │
│    │                           │───────────────────────────▶│      │
│    │                           │                           │◀─────│──
│    │                           │                           │      │  Capture
│    │                           │                           │◀─────│──
│    │                           │ { nodes: [...] }          │      │
│    │                           │◀───────────────────────────│      │
│    │                           │                           │      │
│    │ browser.act({ click: {    │                           │      │
│    │   ref: "e1" } })          │                           │      │
│    │──────────────────────────▶│                           │      │
│    │                           │                           │      │
│    │                           │ DOM.dispatchEvent         │      │
│    │                           │ (via Playwright)           │      │
│    │                           │───────────────────────────▶│      │
│    │                           │                           │◀─────│──
│    │                           │                           │      │
│    │                           │ { ok: true }              │      │
│    │                           │◀───────────────────────────│      │
│    │                           │                           │      │
│    │ browser.screenshot()      │                           │      │
│    │──────────────────────────▶│                           │      │
│    │                           │                           │      │
│    │                           │ Page.captureScreenshot    │      │
│    │                           │───────────────────────────▶│      │
│    │                           │                           │◀─────│──
│    │                           │                           │      │  PNG/JPEG
│    │                           │                           │◀─────│──
│    │                           │                           │      │
│    │                           │ { imagePath }             │      │
│    │                           │◀───────────────────────────│      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Clever Solutions

1. **CDP over WebSocket** - Direct browser protocol without Playwright overhead for simple operations
2. **Profile isolation** - Separate user data dirs enable multiple concurrent browser sessions
3. **Chrome MCP bridge** - Uses official chrome-devtools-mcp for structured accessibility snapshots
4. **Role refs cache** - Stable element references even across page reloads
5. **WS URL normalization** - Handles containerized browser URL rewriting
6. **Navigation guard** - SSRF protection for browser automation

### Technical Debt / Shortcuts

1. **No browserless integration** - Only local Chrome supported (no cloud browser)
2. **Single Chrome process** - Cannot run multiple isolated browsers
3. **No headless toggle per session** - Global headless setting
4. **Playwright for interactions** - CDP simpler but less battle-tested
5. **No browser extension support** - Cannot inject content scripts
6. **Profile cleanup on reset** - Removes entire user data dir

### Error Handling

- CDP endpoint SSRF validation
- Browser reachability checks (WebSocket handshake probe)
- Navigation blocked for disallowed URLs
- Target not found errors (4xx responses)
- Screenshot timeout (5s default)
- Playwright session cleanup on errors
- Graceful browser process termination

### Browser Routes (`src/browser/routes/`)

```
GET  /                          - Browser status
GET  /profiles                  - List profiles
POST /start/:profile            - Start browser
POST /stop/:profile             - Stop browser
POST /reset/:profile            - Reset profile
GET  /tabs/:profile            - List tabs
POST /tabs/:profile             - Open new tab
DELETE /tabs/:profile/:targetId - Close tab
GET  /tabs/:profile/:targetId/snapshot - Get tab snapshot
POST /tabs/:profile/:targetId/act - Act on tab
```

---

## Cross-Feature Insights

### Shared Infrastructure

1. **WebSocket communication** - Used by Canvas (live reload), Browser (CDP), Gateway (RPC)
2. **Profile systems** - Browser uses profiles; Canvas uses sessions
3. **Node routing** - Both Canvas and Browser tools route through nodes
4. **Gateway RPC** - All tools invoke via `node.invoke()` gateway command
5. **Image capture** - Both Canvas and Browser support snapshot/screenshot

### Security Patterns

1. **URL allowlisting** - Navigation guard + SSRF policy
2. **Path traversal prevention** - Canvas scheme handler validates paths
3. **Webhook signature verification** - Voice call Twilio/Telnyx/Plivo
4. **Broadcast auth** - All WS clients receive `voicewake.changed` but ownership is gateway

### Performance Considerations

1. **CDP connection pooling** - Single connection per browser session
2. **Role refs cache** - Avoids repeated accessibility queries
3. **A2UI debouncing** - 75ms write stability threshold
4. **TTS temp cleanup** - 5-minute delayed file deletion
5. **Browser process reuse** - Same Chrome instance for multiple tabs

---

## Summary Table

| Aspect | Voice Wake + Talk | Live Canvas | Browser Control |
|--------|-------------------|-------------|-----------------|
| **Primary Location** | `src/tts/`, `apps/macos/` | `src/canvas-host/` | `src/browser/` |
| **Native Component** | Swift (macOS) | Swift (macOS WKWebView) | Chromium |
| **Protocol** | Gateway WS RPC | A2UI JSONL over HTTP/WS | CDP WebSocket |
| **Key Extension** | `extensions/elevenlabs/` | - | `chrome-devtools-mcp` npm |
| **Agent Tool** | TTS directives in text | `canvas-tool.ts` | `browser-tool.ts` |
| **Primary Use** | Voice messages, calls | Visual workspace | Web automation |
| **Platforms** | macOS, iOS, Android | macOS, iOS, Android | Gateway (any) |
