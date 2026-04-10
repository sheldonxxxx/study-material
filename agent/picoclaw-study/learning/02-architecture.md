# PicoClaw Architecture

## Overview

PicoClaw is an **event-driven, channel-based multi-agent system** with clear separation between AI processing, chat platform integrations, and provider abstractions. The architecture follows a **layered + plugin** pattern with heavy use of interfaces for loose coupling.

```
                    Entry Points
    CLI Agent (picoclaw)  |  Web Server  |  TUI Launcher
                               |
                               v
                      +-------------+
                      |  pkg/agent  |
                      |  AgentLoop  |
                      |  Registry   |
                      +-------------+
                               |
          +--------------------+--------------------+
          |                                         |
          v                                         v
   +-------------+                         +----------------+
   |   pkg/bus   |                         | pkg/providers |
   | MessageBus  |                         |  LLMProvider  |
   +-------------+                         +----------------+
          |                                         |
          v                                         v
   +----------------+                        (42 providers)
   | pkg/channels   |
   | Manager        |
   | BaseChannel    |
   | +platform impl |
   +----------------+
```

## Architectural Decisions

### 1. Central Message Bus for Component Communication

**Decision:** Use a central pub/sub message bus instead of direct component calls.

**Rationale:** Decouples channels from agents. Channels publish inbound messages and consume outbound messages without knowing which agent handles them. Multiple agents can consume the same bus.

**Evidence:** `pkg/bus/bus.go`

```go
type MessageBus struct {
    inbound       chan InboundMessage
    outbound      chan OutboundMessage
    outboundMedia chan OutboundMediaMessage
    streamDelegate atomic.Value // stores StreamDelegate
}
```

Three separate channels prevent head-of-line blocking between text and media.

### 2. Interface Composition for Optional Capabilities

**Decision:** Channels implement optional interfaces rather than a monolithic interface.

**Rationale:** Different platforms support different features (Telegram supports editing messages, IRC does not). Interface composition allows channels to opt into capabilities at compile time.

**Evidence:** `pkg/channels/interfaces.go`

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

// Optional interfaces
type TypingCapable interface {
    StartTyping(ctx context.Context, chatID string) (stop func(), err error)
}
type MessageEditor interface {
    EditMessage(ctx context.Context, chatID string, messageID string, content string) error
}
type PlaceholderCapable interface {
    SendPlaceholder(ctx context.Context, chatID string) (messageID string, err error)
}
type StreamingCapable interface {
    BeginStream(ctx context.Context, chatID string) (Streamer, error)
}
```

### 3. Strategy Pattern for AI Providers

**Decision:** Abstract AI providers behind an interface with pluggable implementations.

**Rationale:** Users may want to use different AI providers based on cost, capability, or availability. The interface allows runtime provider selection and fallback chains.

**Evidence:** `pkg/providers/types.go`

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

### 4. Functional Options for Configuration

**Decision:** Use the functional options pattern for channel configuration instead of constructor parameters.

**Rationale:** Channels have many optional configuration parameters. Functional options allow selective configuration without breaking API compatibility when adding new options.

**Evidence:** `pkg/channels/base.go`

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

func NewBaseChannel(name string, config any, bus *bus.MessageBus,
    allowList []string, opts ...BaseChannelOption) *BaseChannel
```

### 5. Registry Pattern for Tools and Agents

**Decision:** Named registries track all tools and agents with versioning for cache invalidation.

**Rationale:** Tools and agents are discovered by name. Version counters enable cache invalidation when tools are registered or unregistered. Hidden tools with TTL provide ephemeral capabilities.

**Evidence:** `pkg/tools/registry.go`

```go
type ToolRegistry struct {
    tools      map[string]*ToolEntry
    mu         sync.RWMutex
    version    atomic.Uint64 // incremented on Register/RegisterHidden for cache invalidation
    mediaStore media.MediaStore
}

type ToolEntry struct {
    Tool   Tool
    IsCore bool
    TTL    int  // 0 = core (never expires), >0 = decremented by TickTTL
}
```

### 6. Interceptor Chain for Hooks

**Decision:** A chain of interceptors with priority ordering processes LLM requests/responses and tool calls.

**Rationale:** Allows external modification of agent behavior (logging, analytics, content filtering) without modifying core agent code. Interceptors can Continue, Modify, DenyTool, or AbortTurn.

**Evidence:** `pkg/agent/hooks.go`

```go
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

## Module Responsibilities

| Module | Files | Responsibility |
|--------|-------|----------------|
| `pkg/agent` | 37 | Core AI loop, context building, turn management, hooks |
| `pkg/channels` | 39 | Channel manager, base channel, 15+ platform implementations |
| `pkg/providers` | 42 | LLM provider abstractions, fallback chain, model resolution |
| `pkg/tools` | 55 | Tool registry, 20+ tool implementations |
| `pkg/skills` | 10 | Skill loading, installation, registry |
| `pkg/bus` | 5 | Message bus, event types |
| `pkg/routing` | 9 | Model selection routing, session key resolution |
| `pkg/session` | 7 | Session management, context building |
| `pkg/mcp` | 4 | Model Context Protocol client |
| `pkg/voice` | 10 | Voice processing, transcription |
| `pkg/media` | 5 | Media storage and handling |
| `pkg/commands` | 26 | Command parsing and execution |
| `pkg/config` | 18 | Configuration management |

## Communication Patterns

### 1. Bus-Centric Communication

```
Channels ──► MessageBus (inbound) ──► AgentLoop
    │                                        │
    │                                        v
    │                                 AgentLoop
    │                                        │
    └──────────────────────────────────► MessageBus (outbound)
                                             │
                                             v
                                       Channels (via workers)
```

Inbound messages flow: Channel -> MessageBus -> AgentLoop
Outbound messages flow: AgentLoop -> MessageBus -> Channel workers

### 2. Dependency Injection via Constructors

Components receive dependencies via constructors, not global state:

```go
func NewAgentLoop(
    cfg *config.Config,
    msgBus *bus.MessageBus,
    provider providers.LLMProvider,
) *AgentLoop
```

### 3. Context Propagation

Request-scoped values pass through context:

```go
// Tool context (channel/chatID)
func WithToolContext(ctx context.Context, channel, chatID string) context.Context
func ToolChannel(ctx context.Context) string
func ToolChatID(ctx context.Context) string
```

### 4. Work Queue with Rate Limiting

Each channel has its own outbound queue with rate limiting:

```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage    // rate-limited outbound queue
    mediaQueue chan bus.OutboundMediaMessage
    limiter    *rate.Limiter              // per-channel rate limiting
}
```

## Key Architectural Insights

### Strengths

1. **Clean separation** - Channels, providers, and agents are truly independent
2. **Interface-based design** - Easy to add new channels/providers
3. **Graceful degradation** - Streaming fallbacks to placeholders, channel features optional
4. **Robust error handling** - Classified errors with proper fallback logic
5. **Resource management** - Proper cleanup with TTL janitors and context cancellation

### Complexity Areas

1. **AgentLoop (107KB)** - The central loop handles many concerns; consider extracting:
   - Turn management to separate package
   - MCP runtime to separate package
   - Steering queue to separate package

2. **Channel initialization** - `initChannel()` uses factory registry with runtime type assertions

3. **Hook system** - Sophisticated but complex; the interceptor chain with modification support adds significant flexibility

4. **Session/context building** - Multi-file resolution with legacy format support adds indirection

### Trade-offs

- **Monorepo vs microservices**: Single Go module allows easy code sharing but requires careful dependency management
- **Interface vs concrete types**: Heavy interface use enables testing and flexibility but adds indirection
- **Global state via bus**: Central message bus is simple but can hide data flow
- **Feature flags**: Extensive use of config flags enables customization but increases configuration surface area

## File Reference Summary

Core architecture files:

| File | Purpose |
|------|---------|
| `pkg/bus/bus.go` | Message bus with three channels |
| `pkg/bus/types.go` | Bus message types (InboundMessage, OutboundMessage) |
| `pkg/channels/manager.go` | Channel worker management, rate limiting |
| `pkg/channels/interfaces.go` | Channel interface hierarchy |
| `pkg/channels/base.go` | BaseChannel with functional options |
| `pkg/agent/loop.go` | Main agent loop |
| `pkg/agent/registry.go` | Agent registry |
| `pkg/agent/hooks.go` | Hook manager and interceptor chain |
| `pkg/providers/types.go` | Provider interfaces |
| `pkg/providers/fallback.go` | Fallback chain implementation |
| `pkg/tools/registry.go` | Tool registry |
| `pkg/tools/base.go` | Tool interface |
| `pkg/routing/router.go` | Model routing |
