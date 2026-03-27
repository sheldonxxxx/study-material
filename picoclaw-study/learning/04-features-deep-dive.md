# PicoClaw Features Deep Dive

**Date:** 2026-03-26
**Source:** `/Users/sheldon/Documents/claw/reference/picoclaw`
**Research Files:** 7 batch files (05a-05g)
**Synthesized from:** Feature Index, Batch 1-7 research documents

---

## Table of Contents

1. [Core Features](#core-features)
   - [Ultra-Lightweight Agent Core](#1-ultra-lightweight-agent-core)
   - [Multi-Provider LLM Support](#2-multi-provider-llm-support)
   - [Multi-Channel Chat Integration](#3-multi-channel-chat-integration)
   - [MCP Integration](#4-mcp-integration)
   - [Vision Pipeline](#5-vision-pipeline)
   - [Gateway Service](#6-gateway-service)
   - [Smart Model Routing](#7-smart-model-routing)
2. [Secondary Features](#secondary-features)
   - [Web UI Launcher](#8-web-ui-launcher)
   - [TUI Launcher](#9-tui-launcher)
   - [Skills System](#10-skills-system)
   - [Cron Scheduling](#11-cron-scheduling)
   - [Web Search Tools](#12-web-search-tools)
   - [Docker Deployment](#13-docker-deployment)
   - [Cross-Platform Binary](#14-cross-platform-binary)
3. [Extended Features](#extended-features)
   - [Hardware Device Support](#15-hardware-device-support)
   - [Sub-Agent Orchestration](#16-sub-agent-orchestration)
   - [Steering & Hooks System](#17-steering--hooks-system)
   - [Credential Management](#18-credential-management)
   - [Shell/Session Tools](#19-shellsession-tools)
   - [Voice Pipeline](#20-voice-pipeline)

---

## Core Features

Core features are essential capabilities that define PicoClaw's fundamental architecture. These represent the minimum viable implementation for an AI agent with multi-provider, multi-channel support.

---

### 1. Ultra-Lightweight Agent Core

**Target:** <10MB RAM footprint, 99% smaller than OpenClaw

**Core Files:**
- `pkg/agent/loop.go` - Main agent loop (1000+ lines)
- `pkg/agent/instance.go` - Agent instance management (326 lines)
- `pkg/agent/turn.go` - Turn processing (482 lines)
- `pkg/agent/memory.go` - Memory management (159 lines)

**Architecture:**

The `AgentLoop` struct is the central coordinator with atomic state management:

```go
type AgentLoop struct {
    bus      *bus.MessageBus
    cfg      *config.Config
    registry *AgentRegistry
    state    *state.Manager
    eventBus *EventBus
    hooks    *HookManager
    running        atomic.Bool
    summarizing    sync.Map
    fallback       *providers.FallbackChain
    channelManager *channels.Manager
}
```

**Key Design Patterns:**
- Atomic booleans for lock-free running state
- `sync.Map` for concurrent turn state management
- Event-driven architecture with hooks and observers
- Dependency injection of all core services

**Turn State Machine:**

Turns progress through phases: `setup -> running -> tools -> finalizing -> completed/aborted`

```go
type turnState struct {
    mu sync.RWMutex
    agent *AgentInstance
    turnID     string
    agentID    string
    sessionKey string
    phase        TurnPhase
    iteration    int
    gracefulInterrupt     bool
    hardAbort             bool
    providerCancel        context.CancelFunc
}
```

**Memory Hierarchy:**
1. Long-term: `memory/MEMORY.md` - persistent facts
2. Daily notes: `memory/YYYYMM/YYYYMMDD.md` - time-based notes
3. Session: JSONL-based storage (auto-migrates from JSON)

**Memory Optimization Techniques:**
- JSONL append-only format reduces memory churn
- Atomic file writes with `fileutil.WriteFileAtomic` and fsync
- Lazy initialization of channels, tools, providers
- Context-based cancellation throughout
- `sync.Map` for lock-free concurrent access

**Session Store Initialization (instance.go:286-311):**
```go
func initSessionStore(dir string) session.SessionStore {
    store, err := memory.NewJSONLStore(dir)
    if err != nil {
        return session.NewSessionManager(dir)
    }
    // Auto-migrate legacy JSON sessions
    if n, merr := memory.MigrateFromJSON(ctx, dir, store); merr != nil {
        store.Close()
        return session.NewSessionManager(dir)
    }
    return session.NewJSONLBackend(store)
}
```

**Clever Solutions:**
- SubTurn support via child turn states with concurrency semaphore
- Restore points: turn state captures session snapshot for recovery
- Token budget tracking with `atomic.Int64` shared across turns
- Graceful degradation: hard abort cascades to child turns

**Technical Debt:**
- `loop.go` is 28,901+ tokens - suggests potential for splitting
- Memory footprint is 10-20MB in recent builds due to rapid feature additions
- Resource optimization planned at v1.0 after feature stabilization

---

### 2. Multi-Provider LLM Support

**Target:** 30+ LLM providers with unified interface

**Core Files:**
- `pkg/providers/factory_provider.go` - Provider factory (358 lines)
- `pkg/providers/fallback.go` - Fallback chain (305 lines)
- `pkg/providers/error_classifier.go` - Error classification (266 lines)
- `pkg/providers/cooldown.go` - Cooldown tracking (~100 lines)
- `pkg/providers/openai_compat/` - OpenAI-compatible base

**Provider Interface:**
```go
type LLMProvider interface {
    Chat(ctx context.Context, req *ChatRequest) (*LLMResponse, error)
    GetModel() string
}
```

**Factory Pattern:**

The `CreateProviderFromConfig` function routes to appropriate providers via protocol prefix:

```go
func CreateProviderFromConfig(cfg *config.ModelConfig) (LLMProvider, string, error) {
    protocol, modelID := ExtractProtocol(cfg.Model)
    switch protocol {
    case "openai":      // OAuth or API key
    case "azure", "azure-openai":  // Azure-specific
    case "bedrock":     // AWS SDK with region resolution
    case "anthropic":   // Anthropic with OAuth or API key
    case "claude-cli", "codex-cli":  // CLI-based
    case "github-copilot":  // gRPC-based
    // + 30+ OpenAI-compatible providers
    }
}
```

**Supported Providers:**

| Category | Providers |
|----------|-----------|
| Direct | OpenAI, Anthropic, AWS Bedrock, Azure OpenAI, GitHub Copilot |
| OpenAI-Compatible | openrouter, groq, gemini, deepseek, ollama, vllm, mistral, moonshot, cerebras, novita, zhipu, minimax, qwen, modelscope + more |

**Fallback Chain:**
```go
type FallbackChain struct {
    cooldown *CooldownTracker
}

func (fc *FallbackChain) Execute(
    ctx context.Context,
    candidates []FallbackCandidate,
    run func(ctx context.Context, provider, model string) (*LLMResponse, error),
) (*FallbackResult, error)
```

**Key behaviors:**
- Cooldown tracking per-provider/model prevents hammering failing providers
- Error classification determines if error is retriable
- Context respect: cancels immediately on user abort
- Aggregate errors: returns all attempts on complete failure

**Error Classification:**
```go
const (
    FailoverRateLimit        // 429, rate_limit patterns
    FailoverOverloaded       // Overloaded_error
    FailoverTimeout          // 408, deadline exceeded
    FailoverBilling          // 402, payment required
    FailoverAuth             // 401/403, invalid api key
    FailoverFormat           // 400, invalid request
    FailoverContextOverflow  // Context length exceeded
    FailoverTransient        // 5xx server errors
)
```

**Pattern categories:**
- Rate limit: `rate_limit`, `too many requests`, `429`, `exceeded quota`
- Auth: `invalid api key`, `unauthorized`, `token has expired`, `401`, `403`
- Context: `context length exceeded`, `token limit`, `prompt is too long`

**Cooldown Tracker:**
- Success: clears cooldown
- Failure: applies cooldown based on failure reason
- Rate limit: longer cooldown (e.g., 60s)
- Auth error: extended cooldown (provider may be blocked)

**Clever Solutions:**
- Protocol prefix: `provider/model` format
- Region as endpoint: Bedrock accepts region names, SDK resolves
- Extra body injection: Minimax requires `reasoning_split: true` auto-injected
- Multi-key support: cooldown key includes provider and model

**Technical Debt:**
- Hardcoded defaults in switch statement
- OAuth token refresh requires auth store integration
- No circuit breaker: cooldown is time-based, not failure-count based

---

### 3. Multi-Channel Chat Integration

**Target:** 17+ messaging platforms via unified abstraction

**Core Files:**
- `pkg/channels/manager.go` - Channel orchestration (1145 lines)
- `pkg/channels/registry.go` - Factory registry (33 lines)
- `pkg/channels/split.go` - Message splitting (~300 lines)
- `pkg/channels/telegram/telegram.go` - Telegram (~700 lines)
- `pkg/channels/discord/discord.go` - Discord (~500 lines)

**Supported Channels:**

| Channel | Protocol |
|---------|----------|
| Telegram | Bot API (Long Polling) |
| Discord | Bot API (WebSocket) |
| Slack, Matrix, WhatsApp, WeChat, QQ, LINE, Feishu, DingTalk, WeCom, IRC, OneBot, MaixCam, Pico | Various |

**Channel Manager Architecture:**
```go
type Manager struct {
    channels      map[string]Channel
    workers       map[string]*channelWorker
    bus           *bus.MessageBus
    config        *config.Config
    mediaStore    media.MediaStore
    placeholders  sync.Map   // "channel:chatID" -> placeholderID
    typingStops   sync.Map   // "channel:chatID" -> stop func
    streamActive  sync.Map   // "channel:chatID" -> true
}
```

**Worker Structure:**
```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage
    mediaQueue chan bus.OutboundMediaMessage
    done       chan struct{}
    limiter    *rate.Limiter
}
```

**Factory Registry Pattern:**
```go
type ChannelFactory func(cfg *config.Config, bus *bus.MessageBus) (Channel, error)

var factories = map[string]ChannelFactory{}

func RegisterFactory(name string, f ChannelFactory) {
    factories[name] = f
}

// Each channel package has init() that registers itself
func init() {
    channels.RegisterFactory("telegram", NewTelegramChannel)
    channels.RegisterFactory("discord", NewDiscordChannel)
}
```

**Channel Interface:**
```go
type Channel interface {
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Send(ctx context.Context, msg OutboundMessage) error
    IsRunning() bool
}
```

**Optional interfaces:**
- `MediaSender`: Send files/images
- `MessageEditor`: Edit sent messages
- `MessageDeleter`: Delete messages
- `PlaceholderCapable`: Send/edit placeholder messages
- `StreamingCapable`: BeginStream for real-time responses
- `WebhookHandler`: Handle incoming webhooks

**Rate Limits by Channel:**
```go
var channelRateConfig = map[string]float64{
    "telegram": 20,   // msg/s
    "discord":  1,   // Discord is strict
    "slack":    1,
    "matrix":   2,
    "line":     10,
    "qq":       5,
    "irc":      2,
}
```

**Message Flow:**
1. Inbound: Channel receives message -> publishes to `bus.InboundChan()`
2. Agent processing: `AgentLoop.Run()` consumes from bus
3. Outbound: Agent publishes to `bus.OutboundChan()`
4. Dispatch: `Manager.dispatchOutbound()` routes to channel worker
5. Worker: Rate-limited send with retry logic

**Placeholder/Streaming System:**
```go
func (m *Manager) preSend(ctx context.Context, name string, msg bus.OutboundMessage, ch Channel) bool {
    // 1. Stop typing indicator
    // 2. Undo reaction
    // 3. If stream finalized, delete placeholder
    // 4. Try editing placeholder
    // 5. If edit succeeds, skip Send
}
```

**Retry Logic:**
```go
for attempt := 0; attempt <= maxRetries; attempt++ {
    lastErr = w.ch.Send(ctx, msg)
    if lastErr == nil { return }

    // Permanent failures - don't retry
    if errors.Is(lastErr, ErrNotRunning) || errors.Is(lastErr, ErrSendFailed) {
        break
    }
    // Rate limit - fixed delay
    if errors.Is(lastErr, ErrRateLimit) {
        time.Sleep(rateLimitDelay)
        continue
    }
    // Transient - exponential backoff
    backoff := min(baseBackoff * 2^attempt, maxBackoff)
}
```

**Telegram Implementation Highlights:**
```go
type TelegramChannel struct {
    *channels.BaseChannel
    bot     *telego.Bot
    bh      *th.BotHandler
    chatIDs map[string]int64
}
```
- Long polling via `UpdatesViaLongPolling`
- MarkdownV2 or HTML parsing with fallback
- Message splitting at 4096-char limit
- Reply-to-message support

**Discord Implementation Highlights:**
```go
type DiscordChannel struct {
    *channels.BaseChannel
    session    *discordgo.Session
    typingStop map[string]chan struct{}
    botUserID  string
}
```
- WebSocket via `discordgo.Session`
- Typing indicator with TTL
- Message referencing (reply support)
- 2000-char message limit with splitting

**Clever Solutions:**
- TTL Janitor: background goroutine cleans stale typing/placeholder entries
- Shared HTTP server: all channels requiring webhooks share one server
- Split-on-marker: LLM semantic markers for multi-message splits

**Technical Debt:**
- Health check not called in `runTTLJanitor`
- No channel restart if channel fails (requires full restart)

---

### 4. MCP Integration

**Model Context Protocol support for extending agent capabilities**

**Core Files:**
- `pkg/mcp/manager.go` - MCP server connection management
- `pkg/tools/mcp_tool.go` - MCP tool wrapper
- `pkg/agent/loop_mcp.go` - Agent loop integration

**Manager Architecture:**
```go
type ServerConnection struct {
    Name    string
    Client  *mcp.Client
    Session *mcp.ClientSession
    Tools   []*mcp.Tool
}

type Manager struct {
    servers map[string]*ServerConnection
    mu      sync.RWMutex
    closed  atomic.Bool
    wg      sync.WaitGroup // tracks in-flight CallTool calls
}
```

**Transport Support:**
1. **stdio** - For command-based servers (auto-detected when `Command` provided)
2. **SSE/HTTP** - For web-based servers (auto-detected when `URL` provided)

**Environment Variable Handling:**
```go
// Build environment with proper override semantics
envMap := make(map[string]string)
for _, e := range cmd.Environ() {  // Start with parent
    envMap[e[:idx]] = e[idx+1:]
}
for k, v := range envVars {  // File overrides
    envMap[k] = v
}
for k, v := range cfg.Env {  // Config overrides file
    envMap[k] = v
}
```

**Graceful Shutdown (Double-Checked Locking):**
```go
if m.closed.Swap(true) {  // Atomic swap - returns old value
    return nil // already closed
}
m.wg.Wait()  // Wait for in-flight calls
```

**Tool Name Sanitization:**
```go
// Format: mcp_{serverName}_{toolName}
// If sanitization is lossy or name too long, append 8-char FNV hash
full := fmt.Sprintf("mcp_%s_%s", sanitizedServer, sanitizedTool)
```

**Content Type Handling:**
- `TextContent` - Passed through with sanitization
- `ImageContent` - Stored to temp file, registered with MediaStore
- `AudioContent` - Same as ImageContent
- `ResourceLink` - Summarized as text
- `EmbeddedResource` - Stored (if blob) or summarized (if text)

**Media Delivery from MCP Tools:**
```go
scope := fmt.Sprintf(
    "tool:mcp:%s:%s:%s:%d",
    sanitizeIdentifierComponent(t.serverName),
    channel, chatID,
    time.Now().UnixNano(),
)
ref, err := t.mediaStore.Store(tmpPath, media.MediaMeta{...}, scope)
```

**Lazy One-Time Initialization:**
```go
type mcpRuntime struct {
    initOnce sync.Once
    mu       sync.Mutex
    manager  *mcp.Manager
    initErr  error
}
```

**Deferred Server Mode:**
Servers can be hidden from LLM until discovered via search:
```go
func serverIsDeferred(discoveryEnabled bool, serverCfg config.MCPServerConfig) bool {
    if !discoveryEnabled { return false }
    if serverCfg.Deferred != nil { return *serverCfg.Deferred }
    return true
}
```

**Technical Debt:**
- MCP SDK dependency: `github.com/modelcontextprotocol/go-sdk/mcp`
- No connection pooling or reconnection logic visible
- Tool names capped at 64 chars which may cause collisions with FNV hash

---

### 5. Vision Pipeline

**Send images and files directly to the agent with automatic base64 encoding**

**Core Files:**
- `pkg/agent/loop_media.go` - Media ref resolution
- `pkg/media/store.go` - Media storage and lifecycle
- `pkg/tools/send_file.go` - File sending tool

**Media Ref Resolution:**
```go
func resolveMediaRefs(messages []providers.Message, store media.MediaStore, maxSize int) []providers.Message
```

**Dual-Path Strategy:**

1. **Images** - Base64-encoded as data URLs for multimodal LLMs:
```go
if strings.HasPrefix(mime, "image/") {
    dataURL := encodeImageToDataURL(localPath, mime, info, maxSize)
    resolved = append(resolved, dataURL)  // "data:image/png;base64,iVBORw0KGgo..."
}
```

2. **Non-Images** - Local path injected as structured tags:
```go
// Creates [audio:/path], [video:/path], or [file:/path] tags
pathTags = append(pathTags, buildPathTag(mime, localPath))
result[i].Content = injectPathTags(result[i].Content, pathTags)
```

**MIME Detection:**
Uses `h2non/filetype` for magic-byte detection, falls back to extension lookup:
```go
func detectMIME(localPath string, meta media.MediaMeta) string {
    if meta.ContentType != "" { return meta.ContentType }
    kind, err := filetype.MatchFile(localPath)
    if err != nil || kind == filetype.Unknown { return "" }
    return kind.MIME.Value
}
```

**MediaStore Interface:**
```go
type MediaStore interface {
    Store(localPath string, meta MediaMeta, scope string) (ref string, err error)
    Resolve(ref string) (localPath string, err error)
    ResolveWithMeta(ref string) (localPath string, meta MediaMeta, err error)
    ReleaseAll(scope string) error
}
```

**Scope-Based Organization:**
```go
type FileMediaStore struct {
    refs        map[string]mediaEntry
    scopeToRefs map[string]map[string]struct{}  // scope -> refs
    refToScope  map[string]string
    refToPath   map[string]string
    pathStates  map[string]pathRefState
}
```

**Cleanup Policies:**
```go
const (
    CleanupPolicyDeleteOnCleanup = "delete_on_cleanup"  // Store-managed
    CleanupPolicyForgetOnly      = "forget_only"        // External, never delete
)
```

**Two-Phase Cleanup (Lock-Free File Deletion):**
```go
// Phase 1: under lock - remove entries from maps
s.mu.Lock()
for ref := range refs {
    if removablePath, shouldDelete := s.releaseRefLocked(ref, fallbackPath); shouldDelete {
        paths = append(paths, removablePath)
    }
}
delete(s.scopeToRefs, scope)
s.mu.Unlock()

// Phase 2: no lock - delete files from disk
for _, p := range paths {
    os.Remove(p)
}
```

**Technical Debt:**
- No image resizing/compression - large images sent as-is
- Temp files in `os.TempDir()` - could conflict with other processes
- No content-addressable storage - same file stored multiple times
- Scope TTL is fixed at store level, not per-entry

---

### 6. Gateway Service

**HTTP/WebSocket gateway bridging chat channels to AI agent**

**Core Files:**
- `pkg/gateway/gateway.go` - Main gateway orchestration
- `pkg/channels/manager.go` - Channel orchestration

**Service Startup Order:**
```go
func setupAndStartServices(cfg *config.Config, agentLoop *agent.AgentLoop, msgBus *bus.MessageBus) (*services, error) {
    // 1. Cron service
    // 2. Heartbeat service
    // 3. Media store
    // 4. Channel manager
    // 5. Device service
    // 6. Health server
}
```

**Hot Reload Support:**
```go
func setupConfigWatcherPolling(configPath string, debug bool) (chan *config.Config, func()) {
    ticker := time.NewTicker(2 * time.Second)
    for {
        select {
        case <-ticker.C:
            currentModTime := getFileModTime(configPath)
            if currentModTime.After(lastModTime) {
                newCfg, err := config.LoadConfig(configPath)
                if err := newCfg.ValidateModelList(); err != nil {
                    continue  // Use previous valid config
                }
                configChan <- newCfg
            }
        }
    }
}
```

**Per-Channel Capabilities via Interface Checks:**
```go
// Typing indicators
if pc, ok := ch.(PlaceholderCapable); ok {
    phID, _ := pc.SendPlaceholder(ctx, chatID)
    m.RecordPlaceholder(channel, chatID, phID)
}
// Message editing
if editor, ok := ch.(MessageEditor); ok {
    editor.EditMessage(ctx, msg.ChatID, entry.id, msg.Content)
}
// Streaming support
if sc, ok := ch.(StreamingCapable); ok {
    streamer, _ := sc.BeginStream(ctx, chatID)
}
```

**Shared HTTP Server:**
```go
func (m *Manager) SetupHTTPServer(addr string, healthServer *health.Server) {
    m.mux = http.NewServeMux()
    healthServer.RegisterOnMux(m.mux)
    for name, ch := range m.channels {
        if wh, ok := ch.(WebhookHandler); ok {
            m.mux.Handle(wh.WebhookPath(), wh)
        }
        if hc, ok := ch.(HealthChecker); ok {
            m.mux.HandleFunc(hc.HealthPath(), hc.HealthHandler)
        }
    }
}
```

**Graceful Shutdown Order:**
1. Shutdown HTTP server (5s timeout)
2. Cancel dispatcher
3. Drain worker queues
4. Stop all channels
5. Close agent loop

**Technical Debt:**
- No dead letter queue for failed messages
- Worker queue depth fixed at 16
- Config file polling every 2s is coarse-grained
- No back-pressure when channel send is slow

---

### 7. Smart Model Routing

**Rule-based cost optimization: simple queries to light model, complex to premium**

**Core Files:**
- `pkg/routing/router.go` - Router struct
- `pkg/routing/classifier.go` - RuleClassifier
- `pkg/routing/features.go` - Feature extraction

**Configuration:**
```go
type RoutingConfig struct {
    Enabled    bool    `json:"enabled"`
    LightModel string  `json:"light_model"`
    Threshold  float64 `json:"threshold"` // [0,1]; score >= threshold -> primary
}
```

**Feature Extraction (Language-Agnostic):**
```go
type Features struct {
    TokenEstimate     int     // CJK=1 token each, non-CJK=0.25 tokens each
    CodeBlockCount    int     // fenced ``` pairs / 2
    RecentToolCalls   int     // tool_call messages in last 6 history
    ConversationDepth int     // total history length
    HasAttachments    bool    // data URIs or media extensions
}
```

**Token Estimation (CJK-Aware):**
```go
func estimateTokens(msg string) int {
    total := utf8.RuneCountInString(msg)
    cjk := 0
    for _, r := range msg {
        if r >= 0x2E80 && r <= 0x9FFF || // CJK Unified Ideographs
           r >= 0xF900 && r <= 0xFAFF || // CJK Compatibility Ideographs
           r >= 0xAC00 && r <= 0xD7AF {  // Hangul Syllables
            cjk++
        }
    }
    return cjk + (total-cjk)/4
}
```

**Scoring Weights:**
```go
// token > 200:              0.35  — very long prompts
// token 50-200:             0.15  — medium length
// code block present:       0.40  — coding/technical task
// tool calls > 3 (recent):  0.25  — active agentic workflow
// tool calls 1-3 (recent): 0.10  — some tool activity
// conversation depth > 10:  0.10  — accumulated complexity
// attachments present:      1.00  — hard gate; always heavy
```

**Decision Examples:**
- Pure greetings: 0.00 -> light
- Medium prose (50-200 tokens): 0.15 -> light
- Message with code block: 0.40 -> heavy
- Long message (>200 tokens): 0.35 -> heavy
- Any message with image/audio: 1.00 -> heavy

**Integration Point (loop.go:2756):**
```go
_, usedLight, score := agent.Router.SelectModel(userMsg, history, agent.Model)
if !usedLight {
    return agent.Candidates, resolvedCandidateModel(agent.Candidates, agent.Model)
}
return agent.LightCandidates, resolvedCandidateModel(agent.LightCandidates, agent.Router.LightModel())
```

**Technical Debt:**
- No interface for alternative classifiers
- Token estimation assumes uniform distribution
- Hard-coded lookback window of 6

---

## Secondary Features

Secondary features are important capabilities that enhance the core experience but are not strictly required for basic operation.

---

### 8. Web UI Launcher

**Browser-based configuration on localhost:18800**

**Core Files:**
- `web/backend/main.go` - HTTP server entry point
- `web/backend/embed.go` - Frontend embedding
- `web/backend/api/models.go` - Model CRUD API
- `web/backend/api/gateway.go` - Gateway subprocess management

**Frontend Embedding:**
```go
//go:embed all:dist
var frontendFS embed.FS

func registerEmbedRoutes(mux *http.ServeMux) {
    subFS, err := fs.Sub(frontendFS, "dist")
    mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if strings.HasPrefix(r.URL.Path, "/api/") {
            http.NotFound(w, r)
            return
        }
        // SPA fallback to index.html
    }))
}
```

**Gateway Subprocess Management:**
```go
var gateway = struct {
    mu               sync.Mutex
    cmd              *exec.Cmd
    owned            bool    // true if we started the process
    bootDefaultModel string
    runtimeStatus    string  // "stopped", "starting", "running", "restarting", "error"
    logs             *LogBuffer
}{}
```

Key behaviors:
- Attaches to existing gateway via health endpoint PID check
- Forks `picoclaw gateway -E` subprocess
- Health probing: polls `/health` for 15 seconds after start
- Graceful shutdown via SIGTERM

**API Routes:**

| File | Routes | Purpose |
|------|--------|---------|
| `models.go` | GET/POST /api/models, PUT/DELETE /api/models/{index} | Model list CRUD |
| `channels.go` | GET /api/channels/catalog | List supported channels |
| `gateway.go` | GET /api/gateway/status, POST /api/gateway/{start,stop,restart} | Gateway lifecycle |

**Security:**
IP allowlist middleware validates `AllowedCIDRs` from launcher config.

---

### 9. TUI Launcher

**Terminal UI for headless environments and SSH access**

**Core Files:**
- `cmd/picoclaw-launcher-tui/main.go` - Entry point
- `cmd/picoclaw-launcher-tui/config/config.go` - TUI config (tui.toml)
- `cmd/picoclaw-launcher-tui/ui/` - All UI pages

**Configuration (TOML):**
```go
type TUIConfig struct {
    Version  string   `toml:"version"`
    Model    Model    `toml:"model"`
    Provider Provider `toml:"provider"`
}

type Provider struct {
    Schemes []Scheme `toml:"schemes"`
    Users   []User   `toml:"users"`
    Current ProviderCurrent `toml:"current"`
}
```

**UI Framework:**
Uses `github.com/rivo/tview` and `github.com/gdamore/tcell/v2`:

```go
// Cyberpunk Theme
tview.Styles.PrimitiveBackgroundColor = tcell.NewHexColor(0x050510)  // Deep Void
tview.Styles.BorderColor = tcell.NewHexColor(0x00f0ff)                // Neon Cyan
tview.Styles.GraphicsColor = tcell.NewHexColor(0xff00ff)              // Neon Magenta
tview.Styles.PrimaryTextColor = tcell.NewHexColor(0xe0e0e0)           // Off-white
```

**Navigation Model:**
```go
type App struct {
    tapp           *tview.Application
    pages          *tview.Pages
    pageStack      []string
    pageRefreshFns map[string]func()
    modalOpen      map[string]bool
}
```

ESC key pops the page stack.

**Gateway Management (Simpler than Web UI):**
- Uses PID files and `ps`/`tasklist` rather than health endpoint probing
- Cleans up stale PID file on status check

**Technical Debt:**
- Two independent gateway management implementations (Web UI + TUI)
- TUI config uses different format (TOML) than main config (JSON)
- Concurrent model cache refresh could cause thundering herd

---

### 10. Skills System

**Modular capability extensions from SKILL.md files**

**Core Files:**
- `pkg/skills/service.go` - SkillsLoader
- `pkg/skills/installer.go` - SkillInstaller
- `pkg/skills/registry.go` - RegistryManager
- `pkg/skills/clawhub_registry.go` - ClawHub registry
- `pkg/skills/search_cache.go` - Trigram-based caching

**Core Interface:**
```go
type SkillRegistry interface {
    Name() string
    Search(ctx context.Context, query string, limit int) ([]SearchResult, error)
    GetSkillMeta(ctx context.Context, slug string) (*SkillMeta, error)
    DownloadAndInstall(ctx context.Context, slug, version, targetDir string) (*InstallResult, error)
}
```

**Skill Loading Priority:**
1. `workspace/skills/{name}/SKILL.md` - Project-level
2. `~/.picoclaw/skills/{name}/SKILL.md` - Global
3. Builtin skills from embedded assets

**Metadata Extraction:**
- Supports YAML frontmatter: `{"name": "...", "description": "..."}`
- Falls back to JSON frontmatter
- Falls back to parsing H1 title and first paragraph

**Trigram-Based Search Cache:**
```go
const similarityThreshold = 0.7  // Jaccard similarity threshold

func (sc *SearchCache) Get(query string) ([]SearchResult, bool) {
    normalized := normalizeQuery(query)
    // Exact match first, then similarity match using trigram Jaccard
}
```

**Registry Search with Concurrency Control:**
```go
func (rm *RegistryManager) SearchAll(ctx context.Context, query string, limit int) ([]SearchResult, error) {
    sem := make(chan struct{}, rm.maxConcurrent)  // Default: 2
    resultsCh := make(chan regResult, len(regs))
    searchCtx, cancel := context.WithTimeout(ctx, 1*time.Minute)
    // Results merged and sorted by score descending
}
```

**Malware Detection Flow:**
1. Fetch metadata (moderation flags) before download
2. Block installation if `IsMalwareBlocked = true`
3. Warn if `IsSuspicious = true` but allow installation
4. Clean up partial installs on failure

**Atomic Writes:**
All skill metadata uses `fileutil.WriteFileAtomic` with explicit fsync.

**Technical Debt:**
- Workspace-level install lock: mutex locks entire workspace, not per-skill
- No version pinning in origin metadata
- GitHub token in plain text parameter may end in logs

---

### 11. Cron Scheduling

**Built-in scheduling: natural language, intervals, or cron expressions**

**Core Files:**
- `pkg/cron/service.go` - CronService
- `pkg/tools/cron.go` - CronTool

**Core Data Structures:**
```go
type CronSchedule struct {
    Kind    string `json:"kind"`  // "at", "every", "cron"
    AtMS    *int64 `json:"atMs,omitempty"`
    EveryMS *int64 `json:"everyMs,omitempty"`
    Expr    string `json:"expr,omitempty"`
    TZ      string `json:"tz,omitempty"`
}

type CronPayload struct {
    Kind    string `json:"kind"`
    Message string `json:"message"`
    Command string `json:"command,omitempty"`
    Deliver bool   `json:"deliver"`
    Channel string `json:"channel,omitempty"`
    To      string `json:"to,omitempty"`
}
```

**CronService Event Loop:**
```go
func (cs *CronService) runLoop(stopChan chan struct{}) {
    timer := time.NewTimer(time.Hour)
    defer timer.Stop()
    for {
        cs.mu.RLock()
        nextWake := cs.getNextWakeMS()
        cs.mu.RUnlock()

        var delay time.Duration
        if nextWake == nil {
            delay = time.Hour
        } else {
            diff := *nextWake - time.Now().UnixMilli()
            delay = time.Duration(diff) * time.Millisecond
        }
        timer.Reset(delay)
        select {
        case <-stopChan: return
        case <-cs.wakeChan:  // Wake on new job
            if !timer.Stop() { <-timer.C }
        case <-timer.C:
            cs.checkJobs()
        }
    }
}
```

**Wake Channel Pattern (Non-Blocking):**
```go
func (cs *CronService) notify() {
    select {
    case cs.wakeChan <- struct{}{}:
    default:  // Non-blocking
    }
}
```

**Pre-Reset Pattern (Prevents Double Execution):**
```go
// Reset NextRunAtMS before unlocking to prevent duplicate execution
for i := range cs.store.Jobs {
    if dueMap[cs.store.Jobs[i].ID] {
        cs.store.Jobs[i].State.NextRunAtMS = nil
    }
}
cs.saveStoreUnsafe()
cs.mu.Unlock()
```

**Security: Command Execution Restrictions:**
```go
// GHSA-pv8c-p6jf-3fpp mitigation
if command != "" {
    if !constants.IsInternalChannel(channel) {
        return ErrorResult("command execution restricted to internal channels")
    }
    if !t.allowCommand && !commandConfirm {
        return ErrorResult("command_confirm=true required")
    }
}
```

**Technical Debt:**
- Uses `adhocore/gronx` for cron parsing
- No distributed lock for multi-instance deployments
- In-memory job list is source of truth during runtime
- No visible limit on command output size

---

### 12. Web Search Tools

**Integrated web search: DuckDuckGo (built-in), Tavily, Brave, Perplexity, SearXNG, Baidu, GLM**

**Core Files:**
- `pkg/tools/web.go` - All search providers and WebFetchTool
- `pkg/tools/search_tool.go` - RegexSearchTool, BM25SearchTool

**Search Provider Interface:**
```go
type SearchProvider interface {
    Search(ctx context.Context, query string, count int) (string, error)
}
```

**Provider Priority Chain:**
```go
// Priority: Perplexity > Brave > SearXNG > Tavily > DuckDuckGo > Baidu > GLM
if opts.PerplexityEnabled && len(opts.PerplexityAPIKeys) > 0 {
    provider = &PerplexitySearchProvider{...}
} else if opts.BraveEnabled && len(opts.BraveAPIKeys) > 0 {
    // ...
} else if opts.DuckDuckGoEnabled {
    provider = &DuckDuckGoSearchProvider{...}
}
```

**API Key Pool (Round-Robin with Failover):**
```go
type APIKeyPool struct {
    keys    []string
    current uint32  // Atomic counter
}

func (it *APIKeyIterator) Next() (string, bool) {
    length := uint32(len(it.pool.keys))
    if length == 0 || it.attempt >= length { return "", false }
    key := it.pool.keys[(it.startIdx+it.attempt)%length]
    it.attempt++
    return key, true
}
```

**DuckDuckGo Provider (HTML Scraping):**
```go
func (p *DuckDuckGoSearchProvider) Search(ctx context.Context, query string, count int) (string, error) {
    searchURL := fmt.Sprintf("https://html.duckduckgo.com/html/?q=%s", url.QueryEscape(query))
    resp, err := p.client.Do(req)
    return p.extractResults(string(body), count, query)
}
```

**SSRF Protection in WebFetchTool:**

1. **Lightweight pre-flight:**
```go
hostname := parsedURL.Hostname()
if isObviousPrivateHost(hostname, t.whitelist) {
    return ErrorResult("fetching private hosts is not allowed")
}
```

2. **DNS rebinding mitigation:**
```go
return func(ctx context.Context, network, address string) (net.Conn, error) {
    ipAddrs, err := net.DefaultResolver.LookupIPAddr(ctx, host)
    for _, ipAddr := range ipAddrs {
        if shouldBlockPrivateIP(ipAddr.IP, whitelist) { continue }
        conn, err := dialer.DialContext(ctx, network, ...)
    }
}
```

3. **Private IP blocking:**
```go
func isPrivateOrRestrictedIP(ip net.IP) bool {
    if ip.IsLoopback() || ip.IsLinkLocalUnicast() || ip.IsMulticast() || ip.IsUnspecified() {
        return true
    }
    // RFC 1918, link-local, carrier-grade NAT, IPv6 unique-local, 6to4, Teredo
}
```

4. **Redirect following with SSRF check:**
```go
client.CheckRedirect = func(req *http.Request, via []*http.Request) error {
    if len(via) >= maxRedirects { return fmt.Errorf("stopped after %d redirects") }
    if isObviousPrivateHost(req.URL.Hostname(), whitelist) { return fmt.Errorf("redirect target is private") }
    return nil
}
```

**Cloudflare Bypass:**
```go
if resp.StatusCode == http.StatusForbidden && resp.Header.Get("Cf-Mitigated") == "challenge" {
    honestUA := fmt.Sprintf(userAgentHonest, config.Version)
    resp2, body2, err2 := doFetch(honestUA)
}
```

**Technical Debt:**
- HTML scraping fragility: regex-based extraction could break
- No response caching
- Limited error context when all API keys fail
- No retry with backoff for failed requests

---

### 13. Docker Deployment

**Containerized deployment with profiles for agent, gateway, and launcher**

**Core Files:**
- `docker/Dockerfile` - Minimal runtime (~40MB)
- `docker/Dockerfile.full` - Full MCP with Node.js + uv
- `docker/Dockerfile.heavy` - Multi-platform via goreleaser
- `docker/docker-compose.yml` - Standard compose with 3 profiles
- `docker/entrypoint.sh` - First-run initialization

**Multi-Stage Build:**
```dockerfile
# Stage 1: Build
FROM golang:1.25-alpine AS builder
RUN apk add --no-cache git make
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN make build

# Stage 2: Minimal runtime
FROM alpine:3.23
RUN apk add --no-cache ca-certificates tzdata curl
COPY --from=builder /src/build/picoclaw /usr/local/bin/picoclaw
RUN addgroup -g 1000 picoclaw && adduser -D -u 1000 -G picoclaw picoclaw
USER picoclaw
ENTRYPOINT ["picoclaw"]
CMD ["gateway"]
```

**Compose Profiles:**
```yaml
services:
  picoclaw-agent:
    profiles: [agent]  # One-shot query
  picoclaw-gateway:
    profiles: [gateway]  # Long-running bot
    restart: on-failure
  picoclaw-launcher:
    profiles: [launcher]  # Web console + gateway
    ports:
      - "127.0.0.1:18800:18800"
```

**First-Run Initialization:**
```sh
if [ ! -d "${HOME}/.picoclaw/workspace" ] && [ ! -f "${HOME}/.picoclaw/config.json" ]; then
    picoclaw onboard  # Interactive setup, non-blocking via TTY detection
    exit 0
fi
exec picoclaw gateway "$@"
```

**Docker Image Variants:**

| Variant | Base | Size | Use Case |
|---------|------|------|----------|
| `sipeed/picoclaw:latest` | alpine:3.23 | ~40MB | Minimal gateway |
| `sipeed/picoclaw:launcher` | node:24-alpine | ~200MB+ | Web console + MCP |

**Security:**
- Non-root user (picoclaw, UID 1000)
- Health checks on port 18790
- Config mounted read-only in production
- No API keys in image

---

### 14. Cross-Platform Binary

**Single binary for x86_64, ARM64, ARMv6/7, RISC-V64, MIPS, LoongArch**

**Build Configuration (.goreleaser.yaml):**

Three binaries:
1. `picoclaw` - Main CLI agent
2. `picoclaw-launcher` - Web console
3. `picoclaw-launcher-tui` - Terminal UI

**Build Environment:**
```yaml
env:
  - CGO_ENABLED=0  # Static binaries
ldflags:
  - -s -w
  - -X ...Version={{ .Version }}
  - -X ...GitCommit={{ .ShortCommit }}
```

**Supported Platforms:**

| OS | Architectures |
|----|---------------|
| linux | amd64, arm64, riscv64, loong64, arm, s390x, mipsle |
| darwin | amd64, arm64 |
| windows | amd64 |
| freebsd, netbsd | various |

**macOS Special Handling:**
```makefile
# Web launcher requires CGO on macOS
WEB_GO=CGO_ENABLED=1 go
```

**MIPS ELF Flags Patch:**
```makefile
define PATCH_MIPS_FLAGS
    @printf '\004\024\000\160' | dd of=$(1) bs=1 seek=36 count=4 conv=notrunc
endef
```

**Loong64 PTY Patch:**
```makefile
define PTY_PATCH_LOONG64
    @pty_dir=$$(go env GOMODCACHE)/github.com/creack/pty@v1.1.9; \
    if [ ! -f "$$pty_dir/ztypes_loong64.go" ]; then \
        printf '//go:build linux && loong64\npackage pty\ntype (_C_int int32; _C_uint uint32)\n' > "$$pty_dir/ztypes_loong64.go"; \
    fi
endef
```

**Verified Hardware:**
- **x86:** Intel/AMD any x86 CPU
- **ARM:** ARMv6 (Raspberry Pi 1/Zero), ARMv7 (LicheePi Zero), ARM64 (Raspberry Pi 4/5, Orange Pi Zero 3)
- **RISC-V:** SOPHGO SG2002, SpacemiT K1, Canaan K230
- **MIPS:** MediaTek MT7620
- **LoongArch:** Loongson 3A5000, 3A6000

**Minimum Requirements:**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 10MB free | 32MB+ free |
| Storage | 20MB | 50MB+ |
| CPU | 0.6GHz single core | Any |

---

## Extended Features

Extended features are specialized capabilities for specific use cases like IoT, security, or advanced orchestration.

---

### 15. Hardware Device Support

**I2C and SPI peripheral access for embedded/IoT deployments**

**Core Files:**
- `pkg/tools/i2c.go`, `pkg/tools/i2c_linux.go` - I2C implementation
- `pkg/tools/spi.go`, `pkg/tools/spi_linux.go` - SPI implementation
- `pkg/tools/i2c_other.go`, `pkg/tools/spi_other.go` - Stubs

**I2C Interface:**
```go
type I2CTool struct{}

func (t *I2CTool) Parameters() map[string]any {
    return map[string]any{
        "properties": map[string]any{
            "action":    map[string]any{"type": "string", "enum": []string{"detect", "scan", "read", "write"}},
            "bus":       map[string]any{"type": "string"},  // e.g., "1"
            "address":   map[string]any{"type": "integer"},  // 7-bit address
            "register":  map[string]any{"type": "integer"},
            "data":      map[string]any{"type": "array"},
            "length":    map[string]any{"type": "integer"},
            "confirm":   map[string]any{"type": "boolean", "required": true for write},
        },
    }
}
```

**Linux I2C Implementation (syscall.IOCTL):**
```go
const (
    i2cSlave       = 0x0703  // Set slave address
    i2cFuncs       = 0x0705  // Query adapter capabilities
    i2cSmbus       = 0x0720  // Perform SMBus transaction
)
```

**Scan Strategy (EEPROM-Safe):**
```go
func smbusProbe(fd int, addr int, hasQuick bool) bool {
    // EEPROM ranges: use read byte to avoid corruption
    useReadByte := (addr >= 0x30 && addr <= 0x37) || (addr >= 0x50 && addr <= 0x5F)
    if !useReadByte && hasQuick {
        // SMBus Quick Write
        args := i2cSmbusArgs{readWrite: i2cSmbusWrite, size: i2cSmbusQuick, data: nil}
        syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), i2cSmbus, ...)
        return errno == 0
    }
    // SMBus Read Byte for EEPROM ranges
}
```

**SPI Interface:**
```go
type SPITool struct{}

func (t *SPITool) Parameters() map[string]any {
    return map[string]any{
        "properties": map[string]any{
            "action":  map[string]any{"type": "string", "enum": []string{"list", "transfer", "read"}},
            "device":  map[string]any{"type": "string"},  // e.g., "2.0"
            "speed":   map[string]any{"type": "integer"},  // Hz, default 1MHz
            "mode":    map[string]any{"type": "integer"},  // SPI mode 0-3
            "bits":    map[string]any{"type": "integer"},  // bits per word, default 8
        },
    }
}
```

**SPI Transfer (ioctl):**
```go
const (
    spiIocWrMode        = 0x40016B01
    spiIocWrBitsPerWord = 0x40016B03
    spiIocWrMaxSpeedHz  = 0x40046B04
    spiIocMessage1      = 0x40206B00
)

xfer := spiTransfer{
    txBuf:       uint64(uintptr(unsafe.Pointer(&txBuf[0]))),
    rxBuf:       uint64(uintptr(unsafe.Pointer(&rxBuf[0]))),
    length:      uint32(len(txBuf)),
    speedHz:     speed,
    bitsPerWord: bits,
}
syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocMessage1, uintptr(unsafe.Pointer(&xfer)))
runtime.KeepAlive(txBuf)  // Critical: prevent GC before ioctl
runtime.KeepAlive(rxBuf)
```

**Key Technical Notes:**
- No external dependencies: only `syscall` package
- `runtime.KeepAlive()` is critical to prevent GC from collecting buffers
- Device detection via `filepath.Glob("/dev/i2c-*")` and `filepath.Glob("/dev/spidev*")`
- Platform stubs return errors on non-Linux

---

### 16. Sub-Agent Orchestration

**Spawn parallel async tasks with lifecycle management**

**Core Files:**
- `pkg/tools/subagent.go` - SubagentManager, SubagentTool
- `pkg/tools/spawn.go` - SpawnTool (async)
- `pkg/tools/spawn_status.go` - SpawnStatusTool
- `pkg/agent/subturn.go` - Core SubTurn engine (672 lines)

**SubTurnConfig:**
```go
type SubTurnConfig struct {
    Model        string
    Tools        []tools.Tool
    SystemPrompt string
    MaxTokens    int
    Async        bool    // true = fire-and-forget
    Critical     bool    // continues after parent finishes
    Timeout      time.Duration  // default: 5 minutes
    MaxContextRunes int         // 0=auto, -1=no limit
    InitialTokenBudget *atomic.Int64  // Shared budget
}
```

**Concurrency Control (Semaphore):**
```go
select {
case parentTS.concurrencySem <- struct{}{}:
    semAcquired = true
    defer func() {
        if semAcquired {
            <-parentTS.concurrencySem  // Release immediately after runTurn
        }
    }()
case <-timeoutCtx.Done():
    return nil, fmt.Errorf("%w: all %d slots occupied for %v",
        ErrConcurrencyTimeout, rtCfg.maxConcurrent, rtCfg.concurrencyTimeout)
}
```

**Critical: Semaphore released immediately after `runTurn`, not deferred.**

**Depth Limit:**
```go
if parentTS.depth >= rtCfg.maxDepth {
    return nil, ErrDepthLimitExceeded  // Default max: 3 levels
}
```

**Child Context Isolation:**
```go
// Create INDEPENDENT child context (not derived from parent ctx)
// This allows child to continue after parent finishes gracefully
childCtx, cancel := context.WithTimeout(context.Background(), timeout)
defer cancel()
```

**Ephemeral Session Store:**
```go
ephemeralStore := newEphemeralSession(nil)
agent := *baseAgent  // shallow copy
agent.Sessions = ephemeralStore
agent.Tools = baseAgent.Tools.Clone()  // Don't pollute parent's registry
```

**Async Result Delivery:**
```go
func deliverSubTurnResult(al *AgentLoop, parentTS *turnState, childID string, result *tools.ToolResult) {
    parentTS.mu.Lock()
    isFinished := parentTS.isFinished.Load()
    resultChan := parentTS.pendingResults
    parentTS.mu.Unlock()

    if isFinished || resultChan == nil {
        al.emitEvent(EventKindSubTurnOrphan, ...)  // Parent finished
        return
    }
    select {
    case resultChan <- result:
        al.emitEvent(EventKindSubTurnResultDelivered, ...)
    case <-parentTS.Finished():
        al.emitEvent(EventKindSubTurnOrphan, ...)
    }
}
```

**SpawnStatusTool with Conversation Scoping:**
```go
func (t *SpawnStatusTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    callerChannel := ToolChannel(ctx)
    callerChatID := ToolChatID(ctx)
    // Filter tasks to current conversation only
    for _, task := range tasks {
        if cpy.OriginChannel != "" && cpy.OriginChannel != callerChannel { continue }
        if cpy.OriginChatID != "" && cpy.OriginChatID != callerChatID { continue }
    }
}
```

**Default Configuration:**
```go
const (
    defaultMaxSubTurnDepth        = 3
    defaultMaxConcurrentSubTurns = 5
    defaultConcurrencyTimeout     = 30 * time.Second
    defaultSubTurnTimeout         = 5 * time.Minute
    maxEphemeralHistorySize       = 50
)
```

**Technical Observations:**
- `SubTurnSpawner` interface decouples tools from agent package
- Panic recovery with error event emission
- Parent-child relationship established thread-safely before goroutine starts

---

### 17. Steering & Hooks System

**Message injection and event-driven interceptors for running agents**

**Core Files:**
- `pkg/agent/steering.go` - Steering implementation
- `pkg/agent/hooks.go` - HookManager
- `pkg/agent/hook_process.go` - ProcessHook (JSON-RPC over stdio)

**Steering Architecture:**
```go
type steeringQueue struct {
    mu     sync.Mutex
    queues map[string][]providers.Message  // scoped by session key
    mode   SteeringMode
}
```

**Two Modes:**
- `SteeringOneAtATime` (default) - dequeues only first message per poll
- `SteeringAll` - drains entire queue in single poll

**MaxQueueSize = 10** (hard limit)

**Polling Points in Agent Loop:**
1. At loop start (before first LLM call)
2. After every tool completes
3. After a direct LLM response
4. Right before turn finalization

**Key Methods:**
```go
func (al *AgentLoop) Steer(msg providers.Message) error
func (al *AgentLoop) Continue(ctx context.Context, sessionKey, channel, chatID string) (string, error)
func (al *AgentLoop) InterruptGraceful(hint string) error
func (al *AgentLoop) InterruptHard() error
func (al *AgentLoop) InjectFollowUp(msg providers.Message) error
```

**Steering Flow:**
```
Steer("change direction")
    -> steeringQueue.push(message)
    -> Agent loop polls after current tool finishes
    -> Remaining tools SKIPPED
    -> Steering message injected into context
    -> LLM called with updated context
```

**Hooks System:**

| Type | Interface | Stage | Can Modify |
|------|-----------|-------|------------|
| Observer | `EventObserver` | EventBus broadcast | No |
| LLM Interceptor | `LLMInterceptor` | before_llm/after_llm | Yes |
| Tool Interceptor | `ToolInterceptor` | before_tool/after_tool | Yes |
| Tool Approver | `ToolApprover` | approve_tool | No |

**Hook Actions:**
```go
const (
    HookActionContinue   HookAction = "continue"
    HookActionModify     HookAction = "modify"
    HookActionDenyTool   HookAction = "deny_tool"
    HookActionAbortTurn  HookAction = "abort_turn"
    HookActionHardAbort  HookAction = "hard_abort"
)
```

**ProcessHook (JSON-RPC over stdio):**
```go
type ProcessHook struct {
    name string
    opts ProcessHookOptions
    cmd  *exec.Cmd
    stdin io.WriteCloser
    pending map[uint64]chan processHookRPCMessage
    nextID atomic.Uint64
}
```

**Hook Timeouts (Defaults):**
- Observer: 500ms
- Interceptor: 5 seconds
- Approval: 60 seconds

**Technical Debt:**
- No per-process-hook timeouts (only global defaults)
- External hooks cannot initiate RPCs back to PicoClaw
- Steering requires sequential tool execution (was parallel)
- Not suitable for human-in-loop approval workflows

---

### 18. Credential Management

**Secure credential resolution with AES-256-GCM encryption**

**Core Files:**
- `pkg/credential/credential.go` - Encryption/decryption
- `pkg/credential/keygen.go` - SSH key management
- `pkg/credential/store.go` - SecureStore

**Credential Formats:**
| Format | Example | Resolution |
|--------|---------|------------|
| Plaintext | `sk-abc123` | Returned as-is |
| File ref | `file://filename.key` | Content read from `configDir/filename.key` |
| Encrypted | `enc://<base64>` | AES-256-GCM decrypt via passphrase + SSH key |

**Encryption Architecture:**

Key Derivation:
```
HKDF-SHA256(
    ikm=HMAC-SHA256(SHA256(sshKeyBytes), passphrase),
    salt,
    info="picoclaw-credential-v1"
) -> 32 bytes
```

**Components:**
- **Salt:** 16 random bytes (stored with ciphertext)
- **Nonce:** 12 bytes for GCM (stored with ciphertext)
- **Passphrase:** From `PICOCLAW_KEY_PASSPHRASE` env var
- **SSH Key:** Ed25519 private key

**Passphrase Provider Pattern:**
```go
var PassphraseProvider func() string = func() string {
    return os.Getenv(PassphraseEnvVar)
}
// Can be replaced for custom sources (e.g., SecureStore)
credential.PassphraseProvider = apiHandler.passphraseStore.Get
```

**SSH Key Resolution Priority:**
1. Explicit argument to `Encrypt()` / `resolveEncrypted()`
2. `PICOCLAW_SSH_KEY_PATH` env var
3. `~/.ssh/picoclaw_ed25519.key`

**SecureStore (Lock-Free):**
```go
type SecureStore struct {
    val atomic.Pointer[string]  // lock-free reads/writes
}

func (s *SecureStore) SetString(passphrase string)
func (s *SecureStore) Get() string
func (s *SecureStore) Clear()
```

**Security Considerations:**
- No key stretching: HKDF-SHA256 is fast
- SSH key in key derivation makes offline attacks harder
- Atomic writes with 0o600 permissions
- No memory zeroing (standard Go behavior)

---

### 19. Shell/Session Tools

**Secure command execution with sandboxing**

**Core Files:**
- `pkg/tools/shell.go` - ExecTool (1137 lines)
- `pkg/tools/session.go` - SessionManager
- `pkg/tools/filesystem.go` - Filesystem tool

**ExecTool Actions:**
| Action | Description |
|--------|-------------|
| `run` | Execute command synchronously |
| `list` | List active sessions |
| `poll` | Check session status |
| `read` | Read session output |
| `write` | Write to session stdin |
| `kill` | Terminate session |
| `send-keys` | Send key sequences to PTY |

**Deny Patterns (40+):**
```go
defaultDenyPatterns = []*regexp.Regexp{
    regexp.MustCompile(`\brm\s+-[rf]{1,2}\b`),           // rm -rf
    regexp.MustCompile(`\b(del|format|mkfs|diskpart)\b`), // Disk wipe
    regexp.MustCompile(`\bdd\s+if=`),                     // dd with input
    regexp.MustCompile(`\b(shutdown|reboot|poweroff)\b`), // System control
    regexp.MustCompile(`\bsudo\b`),                        // Privilege escalation
    regexp.MustCompile(`\bcurl\b.*\|\s*(sh|bash)`),       // Pipe to shell
    regexp.MustCompile(`\bgit\s+push\b`),                  // Git push
    // ... 40+ patterns total
}
```

**PTY Key Mode Detection:**
```go
type PtyKeyMode uint8
const (
    PtyKeyModeCSI PtyKeyMode = iota  // Standard: \x1b[A for up
    PtyKeyModeSS3                     // Alternate: \x1bOA for up
)
```

**Process Group Termination:**
```go
func prepareCommandForTermination() error {
    cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
}
```

**Filesystem Abstraction:**
```go
type fileSystem interface {
    ReadFile(path string) ([]byte, error)
    WriteFile(path string, data []byte) error
    ReadDir(path string) ([]os.DirEntry, error)
    Open(path string) (fs.File, error)
}
```

Three implementations:
1. **hostFs** - Unrestricted
2. **sandboxFs** - Restricted via `os.Root`
3. **whitelistFs** - Sandbox + path exceptions

**Sandbox via os.Root:**
```go
func (r *sandboxFs) execute(path string, fn func(*os.Root, string) error) error {
    root, err := os.OpenRoot(r.workspace)
    relPath, err := getSafeRelPath(r.workspace, path)
    return fn(root, relPath)
}
```

**Atomic Writes:**
```go
func (h *hostFs) WriteFile(path string, data []byte) error {
    return fileutil.WriteFileAtomic(path, data, 0o600)
}
```

**Remote Channel Restriction:**
```go
// GHSA-pv8c-p6jf-3fpp mitigation
if !t.allowRemote {
    channel := ToolChannel(ctx)
    if channel == "" || !constants.IsInternalChannel(channel) {
        return ErrorResult("exec is restricted to internal channels")
    }
}
```

**Technical Debt:**
- Windows PTY not supported
- No per-command timeout
- Session ID uses 8 hex chars = 32 bits entropy

---

### 20. Voice Pipeline

**Audio-to-text transcription for voice messages**

**Core Files:**
- `pkg/voice/transcriber.go` - Interface + provider detection
- `pkg/voice/audio_model_transcriber.go` - Multimodal LLM
- `pkg/voice/elevenlabs_transcriber.go` - ElevenLabs Scribe API
- `pkg/voice/groq_transcriber.go` - Groq Whisper API

**Core Interface:**
```go
type Transcriber interface {
    Name() string
    Transcribe(ctx context.Context, audioFilePath string) (*TranscriptionResponse, error)
}

type TranscriptionResponse struct {
    Text     string  `json:"text"`
    Language string  `json:"language,omitempty"`
    Duration float64 `json:"duration,omitempty"`
}
```

**Provider Detection Priority:**
```go
func DetectTranscriber(cfg *config.Config) Transcriber {
    // 1. Voice model name (highest priority)
    if modelName := strings.TrimSpace(cfg.Voice.ModelName); modelName != "" {
        if supportsAudioTranscription(modelCfg.Model) {
            return NewAudioModelTranscriber(modelCfg)
        }
    }
    // 2. ElevenLabs API key
    if key := strings.TrimSpace(cfg.Voice.ElevenLabsAPIKey); key != "" {
        return NewElevenLabsTranscriber(key)
    }
    // 3. Groq model fallback
    for _, mc := range cfg.ModelList {
        if strings.HasPrefix(mc.Model, "groq/") && mc.APIKey() != "" {
            return NewGroqTranscriber(mc.APIKey())
        }
    }
    return nil
}
```

**Supported Protocols for Audio Model Transcription:**
- openai, azure, litellm, openrouter, groq, gemini, nvidia
- ollama, moonshot, deepseek, cerebras
- qwen, mistral, minimax, modelscope, novita
- qwen-intl, qwen-us, dashscope-us, avian, longcat, vivgrid, volcengine, vllm
- coding-plan, alibaba-coding, qwen-coding

**AudioModelTranscriber:**
```go
resp, err := t.provider.Chat(ctx, []providers.Message{
    {
        Role:    "user",
        Content: t.prompt,
        Media: []string{
            fmt.Sprintf("data:audio/%s;base64,%s", format,
                base64.StdEncoding.EncodeToString(audioBytes)),
        },
    },
}, nil, t.modelID, map[string]any{"temperature": 0})
```

**ElevenLabsTranscriber:**
- Multipart form upload to `https://api.elevenlabs.io/v1/speech-to-text`
- Hardcoded to `scribe_v1` model
- 120 second timeout

**GroqTranscriber:**
```go
writer.WriteField("model", "whisper-large-v3")
writer.WriteField("response_format", "json")
```
- 60 second timeout

**Agent Integration (loop.go:1056-1080):**
```go
func (al *AgentLoop) transcribeAudioInMessage(ctx context.Context, msg bus.InboundMessage) (bus.InboundMessage, bool) {
    if al.transcriber == nil || al.mediaStore == nil || len(msg.Media) == 0 {
        return msg, false
    }
    var transcriptions []string
    for _, ref := range msg.Media {
        path, meta, err := al.mediaStore.ResolveWithMeta(ref)
        if err != nil { continue }
        if !utils.IsAudioFile(meta.Filename, meta.ContentType) { continue }
        result, err := al.transcriber.Transcribe(ctx, path)
        // ...
    }
    // Replaces audio refs in msg.Content with transcribed text
}
```

**Called at line 580:**
```go
msg, _ = al.transcribeAudioInMessage(ctx, msg)
```

**Technical Debt:**
- `supportsAudioTranscription()` is overly broad (whitelists entire protocols)
- Hardcoded model IDs in ElevenLabs and Groq transcripters

---

## Cross-Cutting Concerns

### Consistent Patterns

1. **Atomic file writes:** `fileutil.WriteFileAtomic` used by skills, cron, filesystem for flash storage reliability
2. **Context propagation:** All HTTP requests and goroutines properly propagate context with timeouts
3. **Structured logging:** `logger` package with `DebugCF`, `InfoCF`, `WarnCF` methods
4. **Two-phase operations:** Media cleanup, state cleanup use map modification under lock, then file deletion without lock

### Security Patterns

1. **SSRF mitigation:** DNS rebinding protection, private IP blocking, redirect validation
2. **Command restrictions:** Channel context checks, deny patterns, workspace sandboxing
3. **Credential encryption:** AES-256-GCM with SSH key key derivation
4. **Non-root Docker:** Images run as `picoclaw` user, not root

### Error Handling

1. **Sentinel errors:** `errors.Is()` for recoverable vs fatal
2. **Error classification:** Providers classify errors for retry/fallback decisions
3. **Graceful degradation:** Systems continue operating when optional components fail

---

## Feature Interactions

**Common Integration Patterns:**

| Feature A | Feature B | Integration |
|-----------|-----------|-------------|
| MCP | MediaStore | MCP tools return binary content stored temporarily |
| Vision | Gateway | Media refs resolved before LLM call |
| Voice | Agent Loop | Audio transcribed before steering check |
| Steering | Bus | Background goroutine drains inbound to steering queue |
| Hooks | Tool Execution | Interceptors can modify/deny tool calls |
| SubAgent | Session | Ephemeral sessions don't persist to parent |
| Skills | Tools | Skills installed to workspace, loaded into context |
| Cron | Shell | Command execution via exec tool with restrictions |

---

## Summary

PicoClaw is a well-architected, feature-rich AI agent framework with:

**Strengths:**
- Clean separation of concerns via interfaces and registries
- Comprehensive error handling with fallback chains
- Multi-provider and multi-channel flexibility
- Production-ready patterns: context propagation, atomic writes, structured logging
- Strong security: SSRF protection, credential encryption, command sandboxing
- Broad hardware support: 8+ architectures, IoT peripherals

**Technical Debt Indicators:**
- `loop.go` is large (28,901+ tokens)
- Some hardcoded values that could be configurable
- No circuit breakers in provider fallback
- Two independent gateway management implementations (Web UI + TUI)
- Voice transcription whitelist is overly broad

**Resource Status:**
- Memory: 10-20MB (target <10MB) due to feature additions
- Optimization planned post-v1.0 after feature stabilization
- All 20 documented features have corresponding implementation
