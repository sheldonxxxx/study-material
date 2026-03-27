# PicoClaw Feature Deep Dive - Batch 1

**Date:** 2026-03-26
**Features Covered:**
1. Ultra-Lightweight Agent Core (<10MB RAM)
2. Multi-Provider LLM Support (30+ providers)
3. Multi-Channel Chat Integration (17+ platforms)

---

## Feature 1: Ultra-Lightweight Agent Core (<10MB RAM)

### Overview

PicoClaw's agent core is designed for minimal resource usage, targeting <10MB RAM footprint. Built entirely in Go (pure Go, no C dependencies), the agent implements a clean loop-based architecture with session management, tool execution, and memory persistence.

### Core Architecture

#### Agent Loop (`pkg/agent/loop.go`)

The `AgentLoop` struct is the central coordinator:

```go
type AgentLoop struct {
    // Core dependencies
    bus      *bus.MessageBus
    cfg      *config.Config
    registry *AgentRegistry
    state    *state.Manager

    // Event system
    eventBus *EventBus
    hooks    *HookManager

    // Runtime state
    running        atomic.Bool
    summarizing    sync.Map
    fallback       *providers.FallbackChain
    channelManager *channels.Manager
    // ... additional fields
}
```

Key design patterns:
- **Atomic booleans** for lock-free running state
- **sync.Map** for concurrent turn state management
- **Event-driven** architecture with hooks and observers
- **Dependency injection** of all core services

#### Agent Instance (`pkg/agent/instance.go`)

The `AgentInstance` represents a configured agent with its own workspace:

```go
type AgentInstance struct {
    ID                        string
    Name                      string
    Model                     string
    Fallbacks                 []string
    Workspace                 string
    MaxIterations             int
    MaxTokens                 int
    Temperature               float64
    ThinkingLevel             ThinkingLevel
    ContextWindow             int
    Provider                  providers.LLMProvider
    Sessions                  session.SessionStore
    ContextBuilder            *ContextBuilder
    Tools                     *tools.ToolRegistry
    Router                    *routing.Router
    LightCandidates           []providers.FallbackCandidate
}
```

#### Memory Store (`pkg/agent/memory.go`)

Lightweight file-based memory with two tiers:

```go
type MemoryStore struct {
    workspace  string
    memoryDir  string
    memoryFile string  // MEMORY.md for long-term
}
```

**Memory hierarchy:**
1. **Long-term memory:** `memory/MEMORY.md` - persistent facts
2. **Daily notes:** `memory/YYYYMM/YYYYMMDD.md` - time-based notes
3. **Session history:** JSONL-based session storage (auto-migrates from JSON)

**Session persistence (instance.go:286-311):**
```go
func initSessionStore(dir string) session.SessionStore {
    store, err := memory.NewJSONLStore(dir)
    if err != nil {
        return session.NewSessionManager(dir)  // fallback
    }
    // Auto-migrate legacy JSON sessions
    if n, merr := memory.MigrateFromJSON(ctx, dir, store); merr != nil {
        store.Close()
        return session.NewSessionManager(dir)  // fallback on failure
    }
    return session.NewJSONLBackend(store)
}
```

### Memory Optimization Techniques

1. **JSONL over JSON:** Append-only format reduces memory churn during session updates
2. **Atomic file writes:** `fileutil.WriteFileAtomic` with fsync for flash storage reliability
3. **Lazy initialization:** Channels, tools, and providers only created when needed
4. **Context-based cancellation:** All operations respect context cancellation
5. **sync.Map usage:** Lock-free concurrent access for active turn states

### Turn State Management

Each conversation turn is managed by `turnState`:

```go
type turnState struct {
    mu sync.RWMutex
    agent *AgentInstance
    opts  processOptions
    scope turnEventScope

    turnID     string
    agentID    string
    sessionKey string
    channel    string
    chatID     string

    phase        TurnPhase
    iteration    int
    startedAt    time.Time
    finalContent string

    gracefulInterrupt     bool
    gracefulInterruptHint string
    hardAbort             bool
    providerCancel        context.CancelFunc
    turnCancel            context.CancelFunc
    // ...
}
```

**Turn phases:** `setup -> running -> tools -> finalizing -> completed/aborted`

### Clever Solutions

1. **SubTurn support:** Parallel task execution via child turn states with concurrency semaphore
2. **Restore points:** Turn state captures session snapshot for recovery after failures
3. **Token budget tracking:** `atomic.Int64` for shared token counters across turns
4. **Graceful degradation:** Hard abort cascades to child turns

### Technical Debt / Concerns

1. **Large loop.go file:** 28,901+ tokens, suggests potential for splitting
2. **Feature velocity noted:** Feature index notes memory footprint is 10-20MB in recent builds due to rapid feature additions
3. **Resource optimization planned:** At v1.0 after feature stabilization

---

## Feature 2: Multi-Provider LLM Support (30+ providers)

### Overview

PicoClaw implements a unified provider abstraction supporting 30+ LLM providers through a factory pattern, with sophisticated fallback routing and error classification for production reliability.

### Provider Architecture

#### Core Interface (`pkg/providers/interfaces.go`)

The `LLMProvider` interface defines the contract:
```go
type LLMProvider interface {
    Chat(ctx context.Context, req *ChatRequest) (*LLMResponse, error)
    GetModel() string
}
```

#### Factory Pattern (`pkg/providers/factory_provider.go`)

The `CreateProviderFromConfig` function routes to appropriate providers:

```go
func CreateProviderFromConfig(cfg *config.ModelConfig) (LLMProvider, string, error) {
    protocol, modelID := ExtractProtocol(cfg.Model)

    switch protocol {
    case "openai":
        // OAuth or API key authentication
    case "azure", "azure-openai":
        // Azure-specific with deployment URLs
    case "bedrock":
        // AWS SDK with region/endpoint resolution
    case "anthropic":
        // Anthropic with OAuth or API key
    case "anthropic-messages":
        // Native Anthropic Messages API
    case "claude-cli", "codex-cli":
        // CLI-based providers
    case "github-copilot":
        // gRPC-based Copilot
    // + 30+ OpenAI-compatible providers
    }
}
```

### Supported Providers

**Direct providers (native implementations):**
- OpenAI (oauth/token/API key)
- Anthropic (oauth/API key)
- Anthropic Messages API
- AWS Bedrock (with AWS SDK)
- Azure OpenAI
- GitHub Copilot (gRPC)
- Claude CLI / Codex CLI (local execution)

**OpenAI-compatible providers (via HTTPProvider):**
| Provider | Default Base URL |
|----------|-----------------|
| openrouter | https://openrouter.ai/api/v1 |
| groq | https://api.groq.com/openai/v1 |
| gemini | https://generativelanguage.googleapis.com/v1beta |
| deepseek | https://api.deepseek.com/v1 |
| ollama | http://localhost:11434/v1 |
| vllm | http://localhost:8000/v1 |
| mistral | https://api.mistral.ai/v1 |
| moonshot | https://api.moonshot.cn/v1 |
| cerebras | https://api.cerebras.ai/v1 |
| novita | https://api.novita.ai/openai |
| zhipu | https://open.bigmodel.cn/api/paas/v4 |
| minimax | https://api.minimaxi.com/v1 |
| qwen | https://dashscope.aliyuncs.com/compatible-mode/v1 |
| modelscope | https://api-inference.modelscope.cn/v1 |
| + many more |

**Total: 30+ providers via protocol-based routing**

### Fallback Chain (`pkg/providers/fallback.go`)

The `FallbackChain` orchestrates multi-candidate failover:

```go
type FallbackChain struct {
    cooldown *CooldownTracker
}

type FallbackCandidate struct {
    Provider string
    Model    string
}

func (fc *FallbackChain) Execute(
    ctx context.Context,
    candidates []FallbackCandidate,
    run func(ctx context.Context, provider, model string) (*LLMResponse, error),
) (*FallbackResult, error)
```

**Key behaviors:**
1. **Cooldown tracking:** Per-provider/model cooldown prevents hammering failing providers
2. **Error classification:** Determines if error is retriable
3. **Context respect:** Cancels immediately on user abort
4. **Aggregate errors:** Returns all attempts on complete failure

### Error Classification (`pkg/providers/error_classifier.go`)

Sophisticated pattern matching for error categorization:

```go
// Failover reasons
const (
    FailoverRateLimit        // 429, rate_limit patterns
    FailoverOverloaded       // Overloaded_error
    FailoverTimeout          // 408, deadline exceeded, timeout patterns
    FailoverBilling          // 402, payment required, insufficient credits
    FailoverAuth             // 401/403, invalid api key, unauthorized
    FailoverFormat           // 400, invalid request format
    FailoverContextOverflow  // Context length exceeded
    FailoverTransient        // 5xx server errors -> timeout
)
```

**Classification approach:**
```go
func ClassifyError(err error, provider, model string) *FailoverError {
    // 1. Context cancellation -> nil (no failover)
    // 2. Deadline exceeded -> timeout (fallback)
    // 3. HTTP status code extraction -> reason mapping
    // 4. Pattern matching -> reason mapping
    // 5. Unknown -> nil (no fallback)
}
```

**Pattern categories:**
- Rate limit: `rate_limit`, `too many requests`, `429`, `exceeded quota`
- Auth: `invalid api key`, `unauthorized`, `token has expired`, `401`, `403`
- Context: `context length exceeded`, `token limit`, `prompt is too long`
- Format: `string should match pattern`, `tool_use.id`

### Cooldown Tracker (`pkg/providers/cooldown.go`)

Per-provider cooldown to prevent cascade failures:
- Success: Clears cooldown
- Failure: Applies cooldown based on failure reason
- Rate limit: Longer cooldown (e.g., 60s)
- Auth error: Extended cooldown (provider may be blocked)

### Model Reference Parsing

```go
func ExtractProtocol(model string) (protocol, modelID string) {
    // "openai/gpt-4o" -> ("openai", "gpt-4o")
    // "gpt-4o" -> ("openai", "gpt-4o")  // default
}
```

### Clever Solutions

1. **Protocol prefix:** Model string includes provider (`provider/model`)
2. **Region as endpoint:** Bedrock accepts region names, SDK resolves endpoint
3. **Extra body injection:** Minimax requires `reasoning_split: true` auto-injected
4. **Multi-key support:** Cooldown key includes both provider and model for multi-key failover
5. **Fallback exhaustion error:** Aggregate error with all attempt details

### Technical Debt / Edge Cases

1. **Hardcoded defaults:** Many provider defaults in switch statement
2. **OAuth token refresh:** Requires auth store integration
3. **AWS SDK init timeout:** 30s timeout for credential resolution
4. **No circuit breaker:** Cooldown is time-based, not failure-count based

---

## Feature 3: Multi-Channel Chat Integration (17+ platforms)

### Overview

PicoClaw connects to 17+ messaging platforms through a unified channel abstraction. The architecture uses a factory registry pattern, with per-channel rate limiting, streaming support, and placeholder editing for natural UX.

### Supported Channels

| Channel | Directory | Protocol |
|---------|-----------|----------|
| Telegram | `pkg/channels/telegram/` | Bot API (Long Polling) |
| Discord | `pkg/channels/discord/` | Bot API (WebSocket) |
| Slack | `pkg/channels/slack/` | Bot API |
| Matrix | `pkg/channels/matrix/` | Client-Server API |
| WhatsApp | `pkg/channels/whatsapp/`, `whatsapp_native/` | Bridge/Native |
| WeChat | `pkg/channels/weixin/` | Official API |
| QQ | `pkg/channels/qq/` | Official API |
| LINE | `pkg/channels/line/` | Messaging API |
| Feishu | `pkg/channels/feishu/` | Open Platform |
| DingTalk | `pkg/channels/dingtalk/` | Open API |
| WeCom | `pkg/channels/wecom/` | Enterprise WeChat |
| IRC | `pkg/channels/irc/` | IRC Protocol |
| OneBot | `pkg/channels/onebot/` | WebSocket |
| MaixCam | `pkg/channels/maixcam/` | Custom Protocol |
| Pico | `pkg/channels/pico/` | PicoClaw Native |
| Pico Client | `pkg/channels/pico/` | WebSocket Client |

**Total: 17 channels**

### Channel Manager (`pkg/channels/manager.go`)

The `Manager` orchestrates all channels:

```go
type Manager struct {
    channels      map[string]Channel
    workers       map[string]*channelWorker
    bus           *bus.MessageBus
    config        *config.Config
    mediaStore    media.MediaStore

    placeholders  sync.Map   // "channel:chatID" -> placeholderID
    typingStops   sync.Map   // "channel:chatID" -> stop func
    reactionUndos sync.Map   // "channel:chatID" -> undo func
    streamActive  sync.Map   // "channel:chatID" -> true
}
```

**Worker architecture:**
```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage
    mediaQueue chan bus.OutboundMediaMessage
    done       chan struct{}
    limiter    *rate.Limiter
}
```

### Factory Registry (`pkg/channels/registry.go`)

Self-registering channel factories:

```go
type ChannelFactory func(cfg *config.Config, bus *bus.MessageBus) (Channel, error)

var factories = map[string]ChannelFactory{}

func RegisterFactory(name string, f ChannelFactory) {
    factories[name] = f
}
```

Each channel package has an `init()` that registers itself:
```go
// In each channel package
func init() {
    channels.RegisterFactory("telegram", NewTelegramChannel)
    channels.RegisterFactory("discord", NewDiscordChannel)
}
```

### Channel Interface

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

### Rate Limiting

Per-channel rate limits prevent API throttling:

```go
var channelRateConfig = map[string]float64{
    "telegram": 20,   // 20 msg/s
    "discord":  1,    // 1 msg/s (Discord is strict)
    "slack":    1,
    "matrix":   2,
    "line":     10,
    "qq":       5,
    "irc":      2,
}
```

Uses `golang.org/x/time/rate` with burst = rate/2.

### Message Flow

1. **Inbound:** Channel receives message -> publishes to `bus.InboundChan()`
2. **Agent processing:** `AgentLoop.Run()` consumes from bus
3. **Outbound:** Agent publishes to `bus.OutboundChan()`
4. **Dispatch:** `Manager.dispatchOutbound()` routes to channel worker
5. **Worker:** Rate-limited send with retry logic

### Placeholder/Streaming System

For natural UX, channels can send a "thinking..." placeholder and edit it later:

```go
// Manager records placeholder
func (m *Manager) RecordPlaceholder(channel, chatID, placeholderID string) {
    key := channel + ":" + chatID
    m.placeholders.Store(key, placeholderEntry{...})
}

// On outbound, preSend checks if placeholder exists and edits it
func (m *Manager) preSend(ctx context.Context, name string, msg bus.OutboundMessage, ch Channel) bool {
    // 1. Stop typing
    // 2. Undo reaction
    // 3. If stream finalized, delete placeholder
    // 4. Try editing placeholder
    // 5. If edit succeeds, skip Send
}
```

### Retry Logic

```go
func (m *Manager) sendWithRetry(ctx context.Context, name string, w *channelWorker, msg bus.OutboundMessage) {
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
        time.Sleep(backoff)
    }
}
```

### Telegram Implementation Highlights

```go
type TelegramChannel struct {
    *channels.BaseChannel
    bot     *telego.Bot
    bh      *th.BotHandler
    config  *config.Config
    chatIDs map[string]int64
}
```

**Features:**
- Long polling via `UpdatesViaLongPolling`
- MarkdownV2 or HTML parsing with fallback
- Message splitting at Telegram's 4096-char limit
- Reply-to-message support
- Command registration system

**Markdown parsing (telegram.go):**
```go
var (
    reHeading    = regexp.MustCompile(`(?m)^#{1,6}\s+([^\n]+)`)
    reBlockquote = regexp.MustCompile(`^>\s*(.*)$`)
    reBoldStar   = regexp.MustCompile(`\*\*(.+?)\*\*`)
    reItalic     = regexp.MustCompile(`_([^_]+)_`)
    // ... more patterns
)
```

### Discord Implementation Highlights

```go
type DiscordChannel struct {
    *channels.BaseChannel
    session    *discordgo.Session
    config     config.DiscordConfig
    typingStop map[string]chan struct{}
    botUserID  string
}
```

**Features:**
- WebSocket via `discordgo.Session`
- Typing indicator management with TTL
- Message referencing (reply support)
- Complex media sending via `ChannelMessageSendComplex`
- 2000-char message limit with splitting

### Clever Solutions

1. **TTL Janitor:** Background goroutine cleans up stale typing/placeholder entries (10s interval)
2. **Streaming delegate:** `GetStreamer()` checks channel capability and falls back gracefully
3. **Shared HTTP server:** All channels requiring webhooks share one server
4. **Health checking:** Channels register health endpoints via `HealthChecker` interface
5. **Split-on-marker:** LLM semantic markers for multi-message splits before channel length limits

### Technical Debt / Edge Cases

1. **Health check not called:** `Manager.runTTLJanitor` never invokes health handlers
2. **Proxy per-channel:** Telegram and Discord have separate proxy handling
3. **Error type matching:** Uses `errors.Is()` for sentinel errors, could be more flexible
4. **No channel restart:** If channel fails, requires full restart

---

## Cross-Cutting Concerns

### Error Handling

All three features use consistent error handling patterns:
- Context propagation for cancellation
- Sentinel errors for recoverable vs fatal
- Logging with structured fields (zerolog)

### Configuration

All features driven by `config.Config` struct:
```go
type Config struct {
    Agents    AgentsConfig
    Channels  ChannelsConfig
    Tools     ToolsConfig
    Providers ModelConfigs
}
```

### Testing

- `pkg/providers/factory_provider_test.go` - Provider factory tests
- `pkg/providers/error_classifier_test.go` - Error classification tests
- `pkg/providers/fallback_test.go` - Fallback chain tests
- `pkg/channels/manager_test.go` - Channel manager tests
- `pkg/channels/telegram/telegram_test.go` - Telegram tests

### Documentation

- `pkg/providers/README.md` - Provider documentation (54KB)
- `pkg/providers/README.zh.md` - Chinese translation
- `docs/channels/` - Channel-specific setup guides
- `docs/tools_configuration.md` - Tool configuration

---

## Files Reference

### Feature 1: Ultra-Lightweight Agent Core
| File | Purpose | Lines |
|------|---------|-------|
| `pkg/agent/loop.go` | Main agent loop coordinator | 1000+ |
| `pkg/agent/instance.go` | Agent instance management | 326 |
| `pkg/agent/turn.go` | Turn state management | 482 |
| `pkg/agent/memory.go` | Memory store | 159 |
| `pkg/session/` | Session persistence | - |
| `pkg/memory/` | JSONL store implementation | - |

### Feature 2: Multi-Provider LLM Support
| File | Purpose | Lines |
|------|---------|-------|
| `pkg/providers/factory_provider.go` | Provider factory | 358 |
| `pkg/providers/fallback.go` | Fallback chain | 305 |
| `pkg/providers/error_classifier.go` | Error classification | 266 |
| `pkg/providers/base.go` | Base provider types | ~200 |
| `pkg/providers/interfaces.go` | Provider interface | ~100 |
| `pkg/providers/cooldown.go` | Cooldown tracking | ~100 |
| `pkg/providers/openai_compat/` | OpenAI-compatible base | - |

### Feature 3: Multi-Channel Chat Integration
| File | Purpose | Lines |
|------|---------|-------|
| `pkg/channels/manager.go` | Channel orchestration | 1145 |
| `pkg/channels/registry.go` | Factory registry | 33 |
| `pkg/channels/base.go` | BaseChannel | ~200 |
| `pkg/channels/split.go` | Message splitting | ~300 |
| `pkg/channels/telegram/telegram.go` | Telegram implementation | ~700 |
| `pkg/channels/discord/discord.go` | Discord implementation | ~500 |
| `pkg/channels/weixin/` | WeChat implementation | - |
| `pkg/channels/feishu/` | Feishu implementation | - |

---

## Summary

PicoClaw demonstrates a well-architected system with:

1. **Ultra-Lightweight Agent Core:** Clean separation via AgentLoop/AgentInstance, efficient JSONL session storage, atomic operations, and event-driven design.

2. **Multi-Provider LLM Support:** Factory pattern supporting 30+ providers, sophisticated fallback with cooldown tracking, comprehensive error classification.

3. **Multi-Channel Integration:** Self-registering factory pattern, per-channel rate limiting, unified messaging interface with optional streaming/placeholder support.

All three features show production-ready patterns: context propagation, structured logging, retry logic, and graceful degradation. The codebase prioritizes reliability through error classification and fallback chains.
