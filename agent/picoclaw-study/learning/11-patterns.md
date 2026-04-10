# PicoClaw Design Patterns

## Enumerated Patterns with Evidence

### 1. Pub/Sub (Message Bus Pattern)

**Intent:** Decouple message producers from consumers via a central event hub.

**Implementation:** `pkg/bus/bus.go`

```go
type MessageBus struct {
    inbound       chan InboundMessage
    outbound      chan OutboundMessage
    outboundMedia chan OutboundMediaMessage
    closeOnce      sync.Once
    done           chan struct{}
    closed         atomic.Bool
    wg             sync.WaitGroup
    streamDelegate atomic.Value // stores StreamDelegate
}

func NewMessageBus() *MessageBus {
    return &MessageBus{
        inbound:       make(chan InboundMessage, defaultBusBufferSize),
        outbound:      make(chan OutboundMessage, defaultBusBufferSize),
        outboundMedia: make(chan OutboundMediaMessage, defaultBusBufferSize),
        done:          make(chan struct{}),
    }
}
```

**Usage:** Channels publish inbound messages; the AgentLoop consumes them. Outbound messages flow through the bus to channel workers.

**Variations:**
- Three separate channels (inbound, outbound, outboundMedia) prevent head-of-line blocking
- Buffered channels (default 64) provide backpressure handling
- `StreamDelegate` interface allows channels to provide streaming without circular imports

---

### 2. Work Queue Pattern

**Intent:** Distribute work across concurrent workers with rate limiting per channel.

**Implementation:** `pkg/channels/manager.go`

```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage
    mediaQueue chan bus.OutboundMediaMessage
    done       chan struct{}
    mediaDone  chan struct{}
    limiter    *rate.Limiter  // per-channel rate limiting
}

var channelRateConfig = map[string]float64{
    "telegram": 20,
    "discord":  1,
    "slack":    1,
    "matrix":   2,
    "line":     10,
    "qq":       5,
    "irc":      2,
}
```

**Usage:** Each channel has a dedicated goroutine processing its outbound queue with rate limiting. Retry with exponential backoff handles transient failures.

---

### 3. Strategy Pattern

**Intent:** Encapsulate algorithms (AI providers) behind a common interface.

**Implementation:** `pkg/providers/types.go`

```go
type LLMProvider interface {
    Chat(ctx context.Context, messages []Message, tools []ToolDefinition,
         model string, options map[string]any) (*LLMResponse, error)
    GetDefaultModel() string
}

type StreamingProvider interface {
    LLMProvider
    ChatStream(ctx context.Context, ..., onChunk func(accumulated string), ...) (*LLMResponse, error)
}

type ThinkingCapable interface { SupportsThinking() bool }
type NativeSearchCapable interface { SupportsNativeSearch() bool }
```

**Usage:** 42 provider implementations (Anthropic, OpenAI, AWS Bedrock, Groq, etc.) satisfy this interface. The agent loop uses the interface without knowing the concrete provider.

---

### 4. Circuit Breaker Pattern

**Intent:** Prevent cascading failures by tracking provider health and skipping unavailable providers.

**Implementation:** `pkg/providers/fallback.go`

```go
type FallbackChain struct {
    cooldown *CooldownTracker  // per-provider/model cooldown
}

func (fc *FallbackChain) Execute(ctx, candidates, run) (*FallbackResult, error) {
    // Check cooldown before each attempt
    cooldownKey := ModelKey(candidate.Provider, candidate.Model)
    if !fc.cooldown.IsAvailable(cooldownKey) {
        remaining := fc.cooldown.CooldownRemaining(cooldownKey)
        result.Attempts = append(result.Attempts, FallbackAttempt{
            Provider: candidate.Provider,
            Model:    candidate.Model,
            Skipped:  true,
            Reason:   FailoverRateLimit,
        })
        continue
    }

    // Execute and mark failure/success
    resp, err := run(ctx, candidate.Provider, candidate.Model)
    if err == nil {
        fc.cooldown.MarkSuccess(cooldownKey)
        // ...
    } else {
        fc.cooldown.MarkFailure(cooldownKey, failErr.Reason)
    }
}
```

**Error Classification:**
```go
type FailoverReason string
const (
    FailoverAuth            FailoverReason = "auth"
    FailoverRateLimit       FailoverReason = "rate_limit"
    FailoverBilling         FailoverReason = "billing"
    FailoverTimeout         FailoverReason = "timeout"
    FailoverFormat          FailoverReason = "format"        // non-retriable
    FailoverContextOverflow FailoverReason = "context_overflow"  // non-retriable
    FailoverOverloaded      FailoverReason = "overloaded"
    FailoverUnknown         FailoverReason = "unknown"
)
```

**Behavior:**
- Retriable errors (rate limit, timeout): mark cooldown, try next provider
- Non-retriable errors (format): abort immediately
- Context cancellation: abort immediately (user abort, no fallback)

---

### 5. Registry Pattern

**Intent:** Manage named components with versioning for cache invalidation.

**Implementation:** `pkg/tools/registry.go`

```go
type ToolRegistry struct {
    tools      map[string]*ToolEntry
    mu         sync.RWMutex
    version    atomic.Uint64 // incremented on Register/RegisterHidden
    mediaStore media.MediaStore
}

type ToolEntry struct {
    Tool   Tool
    IsCore bool
    TTL    int  // 0 = core (never expires), >0 = decremented by TickTTL
}

func (r *ToolRegistry) Register(tool Tool) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.tools[name] = &ToolEntry{Tool: tool, IsCore: true, TTL: 0}
    r.version.Add(1)
}

func (r *ToolRegistry) Get(name string) (Tool, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    entry, ok := r.tools[name]
    if !ok { return nil, false }
    // Hidden tools with expired TTL are not callable
    if !entry.IsCore && entry.TTL <= 0 { return nil, false }
    return entry.Tool, true
}
```

**Additional Registries:**
- `pkg/agent/registry.go` - Agent instances
- `pkg/channels/registry.go` - Channel factories

---

### 6. Template Method Pattern

**Intent:** Define skeleton algorithm in base class; let subclasses override specific steps.

**Implementation:** `pkg/channels/base.go`

```go
type BaseChannel struct {
    config              any
    bus                 *bus.MessageBus
    running             atomic.Bool
    name                string
    allowList           []string
    maxMessageLength    int
    groupTrigger        config.GroupTriggerConfig
    mediaStore          media.MediaStore
    placeholderRecorder PlaceholderRecorder
    owner               Channel // the concrete channel that embeds this BaseChannel
    reasoningChannelID  string
}

func (c *BaseChannel) HandleMessage(ctx context.Context, peer bus.Peer,
    messageID, senderID, chatID, content string, media []string,
    metadata map[string]string, senderOpts ...bus.SenderInfo) {
    // 1. Allow check
    if !c.IsAllowed(senderID) { return }

    // 2. Build message
    scope := BuildMediaScope(c.name, chatID, messageID)
    msg := bus.InboundMessage{...}

    // 3. Auto-trigger typing, reaction, placeholder (if owner implements)
    if c.owner != nil && c.placeholderRecorder != nil {
        if tc, ok := c.owner.(TypingCapable); ok { /* ... */ }
        if rc, ok := c.owner.(ReactionCapable); ok { /* ... */ }
        if pc, ok := c.owner.(PlaceholderCapable); ok { /* ... */ }
    }

    // 4. Publish to bus
    c.bus.PublishInbound(ctx, msg)
}

func (c *BaseChannel) ShouldRespondInGroup(isMentioned bool, content string) (bool, string) {
    // Algorithm with hook points:
    // - Mentioned → always respond
    // - mention_only → require mention
    // - Prefix matching
    // - Default: respond to all
}
```

**Usage:** Concrete channels (Telegram, Discord, etc.) embed `BaseChannel` and override specific methods.

---

### 7. Adapter Pattern

**Intent:** Convert platform-specific interfaces to a common interface.

**Implementation:** Channel implementations in `pkg/channels/{platform}/`

Each channel adapts its platform's API to the `Channel` interface:

```go
type Channel interface {
    Name() string
    Start(ctx) error
    Stop(ctx) error
    Send(ctx, OutboundMessage) error
    IsRunning() bool
    IsAllowed(senderID string) bool
    IsAllowedSender(sender SenderInfo) bool
    ReasoningChannelID() string
}
```

**Examples:**
- `pkg/channels/telegram/` - Telegram Bot API adapter
- `pkg/channels/discord/` - Discordgo adapter
- `pkg/channels/slack/` - Slack API adapter

---

### 8. Interceptor/Chain of Responsibility Pattern

**Intent:** Allow preprocessing and postprocessing of requests without modifying core logic.

**Implementation:** `pkg/agent/hooks.go`

```go
type HookManager struct {
    hooks   map[string]HookRegistration
    ordered []HookRegistration  // sorted by priority
}

type HookRegistration struct {
    Name     string
    Priority int
    Source   HookSource
    Hook     any
}

type LLMInterceptor interface {
    BeforeLLM(ctx context.Context, req *LLMHookRequest) (*LLMHookRequest, HookDecision, error)
    AfterLLM(ctx context.Context, resp *LLMHookResponse) (*LLMHookResponse, HookDecision, error)
}

type ToolInterceptor interface {
    BeforeTool(ctx context.Context, call *ToolCallHookRequest) (*ToolCallHookRequest, HookDecision, error)
    AfterTool(ctx context.Context, result *ToolResultHookResponse) (*ToolResultHookResponse, HookDecision, error)
}

type ToolApprover interface {
    ApproveTool(ctx context.Context, req *ToolApprovalRequest) (ApprovalDecision, error)
}
```

**Hook Actions:**
```go
const (
    HookActionContinue  HookAction = "continue"
    HookActionModify    HookAction = "modify"
    HookActionDenyTool  HookAction = "deny_tool"
    HookActionAbortTurn HookAction = "abort_turn"
    HookActionHardAbort HookAction = "hard_abort"
)
```

**Processing:**
```go
func (hm *HookManager) BeforeLLM(ctx context.Context, req *LLMHookRequest) (*LLMHookRequest, HookDecision) {
    current := req.Clone()
    for _, reg := range hm.snapshotHooks() {
        interceptor, ok := reg.Hook.(LLMInterceptor)
        if !ok { continue }

        next, decision, ok := hm.callBeforeLLM(ctx, reg.Name, interceptor, current.Clone())
        if !ok { continue }

        switch decision.normalizedAction() {
        case HookActionContinue, HookActionModify:
            if next != nil { current = next }
        case HookActionAbortTurn, HookActionHardAbort:
            return current, decision
        }
    }
    return current, HookDecision{Action: HookActionContinue}
}
```

---

### 9. Plugin Pattern

**Intent:** Extend system capabilities through dynamically registered components.

**Implementation:** `pkg/tools/registry.go` + `pkg/tools/base.go`

```go
type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]any
    Execute(ctx context.Context, args map[string]any) *ToolResult
}

type AsyncExecutor interface {
    Tool
    ExecuteAsync(ctx context.Context, args map[string]any, cb AsyncCallback) *ToolResult
}
```

**Tool Implementations (55 files):**
- `shell.go` - Execute shell commands
- `edit.go` - File editing
- `web.go` - Web search
- `spawn.go` - Spawn subagents
- `mcp.go` - Model Context Protocol tools
- `cron.go` - Scheduled job execution
- And more...

**Registration:**
```go
func (r *ToolRegistry) Register(tool Tool) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.tools[name] = &ToolEntry{Tool: tool, IsCore: true, TTL: 0}
    r.version.Add(1)
}
```

---

### 10. Factory Pattern

**Intent:** Create channel instances without coupling to concrete types.

**Implementation:** `pkg/channels/registry.go`

```go
var factories = make(map[string]func(...) Channel)

func RegisterFactory(name string, f func(...)) {
    factories[name] = f
}

func getFactory(name string) (func(...), bool) {
    f, ok := factories[name]
    return f, ok
}
```

**Registration:**
```go
func init() {
    RegisterFactory("telegram", NewTelegramChannel)
    RegisterFactory("discord", NewDiscordChannel)
    RegisterFactory("slack", NewSlackChannel)
    // ...
}
```

---

### 11. Decorator Pattern

**Intent:** Wrap functionality around core behavior with cleanup guarantees.

**Implementation:** `finalizeHookStreamer` wrapper

The streaming capability is wrapped with hook finalization:

```go
// Pseudocode representation
type finalizeHookStreamer struct {
    inner   Streamer
    hooks   *HookManager
    channel string
    chatID  string
}

func (s *finalizeHookStreamer) Update(ctx context.Context, content string) error {
    return s.inner.Update(ctx, content)
}

func (s *finalizeHookStreamer) Finalize(ctx context.Context, content string) error {
    // Run after_stream hooks
    // Then finalize
    return s.inner.Finalize(ctx, content)
}

func (s *finalizeHookStreamer) Cancel(ctx context.Context) {
    // Run cancel hooks
    // Then cancel
    s.inner.Cancel(ctx)
}
```

---

### 12. Functional Options Pattern

**Intent:** Configure objects with optional parameters without constructor bloat.

**Implementation:** `pkg/channels/base.go`

```go
type BaseChannelOption func(*BaseChannel)

func WithMaxMessageLength(n int) BaseChannelOption {
    return func(c *BaseChannel) { c.maxMessageLength = n }
}

func WithGroupTrigger(gt config.GroupTriggerConfig) BaseChannelOption {
    return func(c *BaseChannel) { c.groupTrigger = gt }
}

func WithReasoningChannelID(id string) BaseChannelOption {
    return func(c *BaseChannel) { c.reasoningChannelID = id }
}

func NewBaseChannel(
    name string,
    config any,
    bus *bus.MessageBus,
    allowList []string,
    opts ...BaseChannelOption,  // variadic options
) *BaseChannel {
    bc := &BaseChannel{...}
    for _, opt := range opts {
        opt(bc)  // apply each option
    }
    return bc
}
```

**Usage:**
```go
channel := NewBaseChannel(
    "telegram",
    config,
    bus,
    allowList,
    WithMaxMessageLength(4096),
    WithGroupTrigger(gt),
    WithReasoningChannelID("reasoning_chat"),
)
```

---

### 13. TTL/Ephemeral Component Pattern

**Intent:** Provide temporary capabilities that auto-expire.

**Implementation:** Tool registry with TTL

```go
func (r *ToolRegistry) PromoteTools(names []string, ttl int) {
    r.mu.Lock()
    defer r.mu.Unlock()
    for _, name := range names {
        if entry, exists := r.tools[name]; exists {
            if !entry.IsCore {
                entry.TTL = ttl
            }
        }
    }
}

func (r *ToolRegistry) TickTTL() {
    r.mu.Lock()
    defer r.mu.Unlock()
    for _, entry := range r.tools {
        if !entry.IsCore && entry.TTL > 0 {
            entry.TTL--
        }
    }
}
```

**Usage:** Skills can register ephemeral tools that expire after a certain number of turns.

---

### 14. Context Carrying Pattern

**Intent:** Pass request-scoped values without global state.

**Implementation:** `pkg/tools/base.go`

```go
type toolCtxKey struct{ name string }
var ctxKeyChannel = &toolCtxKey{"channel"}
var ctxKeyChatID = &toolCtxKey{"chatID"}

func WithToolContext(ctx context.Context, channel, chatID string) context.Context {
    ctx = context.WithValue(ctx, ctxKeyChannel, channel)
    ctx = context.WithValue(ctx, ctxKeyChatID, chatID)
    return ctx
}

func ToolChannel(ctx context.Context) string {
    v, _ := ctx.Value(ctxKeyChannel).(string)
    return v
}
```

**Usage:** Each tool execution carries its channel and chatID via context, enabling tools to know where they were invoked.

---

### 15. Snapshot Pattern

**Intent:** Provide consistent views of mutable state for iteration.

**Implementation:** `ToolRegistry.SnapshotHiddenTools()`

```go
type HiddenToolSnapshot struct {
    Docs    []HiddenToolDoc
    Version uint64
}

func (r *ToolRegistry) SnapshotHiddenTools() HiddenToolSnapshot {
    r.mu.RLock()
    defer r.mu.RUnlock()
    docs := make([]HiddenToolDoc, 0, len(r.tools))
    // ... collect docs
    return HiddenToolSnapshot{
        Docs:    docs,
        Version: r.version.Load(),  // consistent version
    }
}
```

**Usage:** BM25 search tool uses snapshot for consistent indexing across concurrent requests.

---

## Pattern Summary Table

| Pattern | Location | Key Types |
|---------|----------|-----------|
| Pub/Sub | `pkg/bus/bus.go` | `MessageBus`, `InboundMessage`, `OutboundMessage` |
| Work Queue | `pkg/channels/manager.go` | `channelWorker`, `Manager` |
| Strategy | `pkg/providers/types.go` | `LLMProvider`, `StreamingProvider` |
| Circuit Breaker | `pkg/providers/fallback.go` | `FallbackChain`, `CooldownTracker` |
| Registry | `pkg/tools/registry.go`, `pkg/agent/registry.go` | `ToolRegistry`, `AgentRegistry` |
| Template Method | `pkg/channels/base.go` | `BaseChannel`, `BaseChannelOption` |
| Adapter | `pkg/channels/{platform}/` | Channel implementations |
| Interceptor | `pkg/agent/hooks.go` | `HookManager`, `LLMInterceptor`, `ToolInterceptor` |
| Plugin | `pkg/tools/` | `Tool`, `AsyncExecutor` |
| Factory | `pkg/channels/registry.go` | `factories` map |
| Decorator | streaming wrappers | `finalizeHookStreamer` |
| Functional Options | `pkg/channels/base.go` | `BaseChannelOption` |
| TTL/Ephemeral | `pkg/tools/registry.go` | `ToolEntry.TTL` |
| Context Carrying | `pkg/tools/base.go` | `WithToolContext`, `ToolChannel` |
| Snapshot | `pkg/tools/registry.go` | `HiddenToolSnapshot` |
