# PicoClaw Architecture and Design Patterns

## Architectural Overview

PicoClaw is an **event-driven, channel-based multi-agent system** with clear separation between AI processing, chat platform integrations, and provider abstractions. The architecture follows a **layered + plugin** pattern with heavy use of interfaces for loose coupling.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Entry Points                             │
│   CLI Agent (picoclaw)  │  Web Server  │  TUI Launcher           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      pkg/agent                                   │
│   AgentLoop (107KB) ──► AgentRegistry ──► AgentInstance(s)       │
│         │                      │                                 │
│         ▼                      ▼                                 │
│   EventBus, HookManager    Tools, Skills, Routing                │
└─────────────────────────────────────────────────────────────────┘
         │                                        │
         ▼                                        ▼
┌─────────────────────┐              ┌─────────────────────────────────┐
│     pkg/bus         │              │         pkg/providers          │
│  MessageBus (pub/sub)│              │  LLMProvider interface          │
│  - inbound          │              │  FallbackChain                  │
│  - outbound         │              │  42 provider implementations    │
│  - outboundMedia    │              └─────────────────────────────────┘
└─────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    pkg/channels                                  │
│   Manager ──► ChannelWorker(s) ──► Channel implementations       │
│         │                                                        │
│         ▼                                                        │
│   BaseChannel (common) + platform-specific channels              │
│   Telegram, Discord, Slack, Matrix, WhatsApp, 15+ total           │
└─────────────────────────────────────────────────────────────────┘
```

## Architectural Pattern: Event-Driven with Actor-like Components

### 1. Message Bus (Central Event Hub)

**File:** `pkg/bus/bus.go`

The `MessageBus` is the central nervous system, implementing a **pub/sub channel-based message passing** pattern:

```go
type MessageBus struct {
    inbound       chan InboundMessage
    outbound      chan OutboundMessage
    outboundMedia chan OutboundMediaMessage
    streamDelegate atomic.Value // stores StreamDelegate
}
```

Key design decisions:
- **Three separate channels** for inbound, outbound text, and outbound media - prevents head-of-line blocking
- **Buffered channels** (default 64) for backpressure handling
- **StreamDelegate** pattern allows channels to provide streaming capabilities without circular imports
- **Graceful shutdown** with drain等待 for in-flight messages

### 2. Channel Manager (Work Queue Pattern)

**File:** `pkg/channels/manager.go`

The Manager implements a **work queue pattern** with per-channel workers:

```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage    // rate-limited outbound queue
    mediaQueue chan bus.OutboundMediaMessage
    limiter    *rate.Limiter              // per-channel rate limiting
}
```

Features:
- **Rate limiting** per channel (Telegram: 20 msg/s, Discord: 1 msg/s)
- **Retry with exponential backoff** (base 500ms, max 8s)
- **Placeholder editing** for streaming responses
- **TTL janitor** for cleaning up stale typing indicators

## Key Abstractions and Interfaces

### Provider Abstraction (Strategy Pattern)

**File:** `pkg/providers/types.go`

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

The **FallbackChain** implements the **Circuit Breaker + Retry** pattern:

```go
type FallbackChain struct {
    cooldown *CooldownTracker  // per-provider/model cooldown
}

func (fc *FallbackChain) Execute(ctx, candidates, run) (*FallbackResult, error)
    // - Skips providers in cooldown
    // - Retriable errors (rate limit, timeout) trigger fallback
    // - Non-retriable errors (format) abort immediately
```

### Channel Interface (Template Method + Adapter Pattern)

**File:** `pkg/channels/interfaces.go`

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

**Optional interfaces** for platform-specific capabilities:
- `TypingCapable` - Show typing indicator
- `MessageEditor` - Edit existing messages
- `MessageDeleter` - Delete messages
- `ReactionCapable` - Add reactions
- `PlaceholderCapable` - Send "Thinking..." placeholder
- `StreamingCapable` - Real-time token streaming
- `CommandRegistrarCapable` - Platform command menus

This **interface composition** pattern allows channels to implement only what the platform supports.

### Agent Registry (Registry Pattern)

**File:** `pkg/agent/registry.go`

```go
type AgentRegistry struct {
    agents   map[string]*AgentInstance
    resolver *routing.RouteResolver
    mu       sync.RWMutex
}
```

Supports:
- **Multiple agents** with workspace isolation
- **Routing** to select agent based on message characteristics
- **Subagent spawning** with permission control

### Tool System (Plugin Pattern)

**File:** `pkg/tools/registry.go`

The registry implements a **plugin architecture** where tools are registered by name:

```go
type ToolRegistry struct {
    tools map[string]Tool
}

type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]any
    Execute(ctx, args map[string]any) (*ToolResult, error)
}
```

55 tool implementations including:
- `shell` - Execute shell commands
- `edit` - File editing
- `web` - Web search
- `spawn` - Spawn subagents
- `mcp` - Model Context Protocol tools

### Hook System (Interceptor Pattern)

**File:** `pkg/agent/hooks.go`

A sophisticated **interceptor chain** with ordering and timeouts:

```go
type HookManager struct {
    hooks []HookRegistration  // ordered by priority
}

type LLMInterceptor interface {
    BeforeLLM(ctx, *LLMHookRequest) (*LLMHookRequest, HookDecision, error)
    AfterLLM(ctx, *LLMHookResponse) (*LLMHookResponse, HookDecision, error)
}

type ToolInterceptor interface {
    BeforeTool(ctx, *ToolCallHookRequest) (*ToolCallHookRequest, HookDecision, error)
    AfterTool(ctx, *ToolResultHookResponse) (*ToolResultHookResponse, HookDecision, error)
}

type ToolApprover interface {
    ApproveTool(ctx, *ToolApprovalRequest) (ApprovalDecision, error)
}
```

Hook actions:
- `Continue` - Proceed normally
- `Modify` - Change the request/response
- `DenyTool` - Block specific tool calls
- `AbortTurn` - Stop the current turn
- `HardAbort` - Stop all processing

### Session Management

**File:** `pkg/session/session.go`

Session context is built from markdown definition files:
- `AGENT.md` / `AGENTS.md` - Agent prompt definitions
- `SOUL.md` - Personality/voice definition
- `USER.md` - User context

## Component Communication Patterns

### 1. Bus-Centric Communication

```
Channels ──► MessageBus ──► AgentLoop
    │             │
    │             ▼
    │        Channels (outbound queue)
    │
    └──────────► Channels (for streaming)
```

### 2. Dependency Injection

Components receive dependencies via constructors:

```go
func NewAgentLoop(
    cfg *config.Config,
    msgBus *bus.MessageBus,
    provider providers.LLMProvider,
) *AgentLoop
```

### 3. Functional Options Pattern

Used extensively for configuration:

```go
func NewBaseChannel(name string, config any, bus *bus.MessageBus,
    allowList []string, opts ...BaseChannelOption) *BaseChannel

func WithMaxMessageLength(n int) BaseChannelOption
func WithGroupTrigger(gt config.GroupTriggerConfig) BaseChannelOption
func WithReasoningChannelID(id string) BaseChannelOption
```

## Design Patterns in Use

| Pattern | Location | Usage |
|---------|----------|-------|
| **Pub/Sub** | `pkg/bus` | Message routing between components |
| **Work Queue** | `pkg/channels/manager.go` | Per-channel outbound processing |
| **Strategy** | `pkg/providers` | Pluggable AI provider implementations |
| **Circuit Breaker** | `pkg/providers/fallback.go` | Model fallback with cooldown tracking |
| **Registry** | `pkg/agent/registry.go`, `pkg/tools/registry.go` | Named component registration |
| **Template Method** | `pkg/channels/base.go` | BaseChannel with hook points |
| **Adapter** | Channel implementations | Platform-specific interfaces |
| **Interceptor** | `pkg/agent/hooks.go` | LLM and tool call hooks |
| **Plugin** | `pkg/tools` | Extensible tool system |
| **Factory** | Channel initialization | `getFactory(name)` registry |
| **Decorator** | `finalizeHookStreamer` | Streaming wrapper for cleanup hooks |

## Major Modules and Responsibilities

| Module | Files | Responsibility |
|--------|-------|----------------|
| `pkg/agent` | 37 | Core AI loop, context building, turn management, hooks |
| `pkg/channels` | 39 | Channel manager, base channel, platform implementations |
| `pkg/providers` | 42 | LLM provider abstractions, fallback, model resolution |
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
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/bus/bus.go` - Message bus
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/channels/manager.go` - Channel management
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/channels/interfaces.go` - Channel interface hierarchy
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/channels/base.go` - BaseChannel template
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/agent/loop.go` - Main agent loop
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/agent/registry.go` - Agent registry
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/agent/hooks.go` - Hook system
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/providers/types.go` - Provider interfaces
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/providers/fallback.go` - Fallback chain
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/tools/registry.go` - Tool registry
- `/Users/sheldon/Documents/claw/reference/picoclaw/pkg/routing/router.go` - Model routing
