# Architecture

## Architectural Pattern: Event-Driven with Message Bus

Nanobot uses an **event-driven architecture** with an **asynchronous message bus** as the central hub. This decouples chat platform integrations (channels) from the core agent processing engine.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         nanobot Process                              │
│                                                                      │
│  ┌──────────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │   Channels   │         │  MessageBus │         │  AgentLoop   │  │
│  │  (adapters) │◄───────►│  (queues)   │◄───────►│  (engine)    │  │
│  └──────────────┘         └─────────────┘         └──────────────┘  │
│         │                        ▲                        │         │
│         ▼                        │                        ▼         │
│  ┌──────────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │  InboundMsg  │         │  OutboundMsg │         │  SessionMgr  │  │
│  │  (events)   │         │  (events)   │         │  (JSONL)     │  │
│  └──────────────┘         └─────────────┘         └──────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │ WebSocket
                              ▼
                    ┌──────────────────┐
                    │   WhatsApp Bridge │
                    │   (TypeScript)    │
                    └──────────────────┘
```

## Major Modules

### Bus Layer (`nanobot/bus/`)

**Purpose:** Decouple channels from agent via async queues.

| Class | Responsibility |
|-------|----------------|
| `InboundMessage` | Dataclass for incoming messages |
| `OutboundMessage` | Dataclass for outgoing messages |
| `MessageBus` | Async queue manager with `inbound` and `outbound` queues |

```python
class MessageBus:
    async def publish_inbound(msg: InboundMessage)
    async def consume_inbound() -> InboundMessage
    async def publish_outbound(msg: OutboundMessage)
    async def consume_outbound() -> OutboundMessage
```

### Channel Layer (`nanobot/channels/`)

**Purpose:** Abstract adapter layer for chat platforms.

| Class | Responsibility |
|-------|----------------|
| `BaseChannel` | Abstract base defining channel interface |
| `ChannelManager` | Routes messages to/from channel instances |
| `TelegramChannel`, `DiscordChannel`, etc. | Platform-specific implementations |

**BaseChannel interface:**
```python
class BaseChannel(ABC):
    name: str
    display_name: str

    @abstractmethod async def start(self) -> None
    @abstractmethod async def stop(self) -> None
    @abstractmethod async def send(self, msg: OutboundMessage) -> None

    async def _handle_message(sender_id, chat_id, content, media, metadata, session_key)
    def is_allowed(sender_id) -> bool  # Allowlist security
```

### Agent Layer (`nanobot/agent/`)

**Purpose:** Core processing engine.

| Class | File | Responsibility |
|-------|------|----------------|
| `AgentLoop` | `loop.py` | Main async loop, tool execution |
| `ContextBuilder` | `context.py` | Constructs prompts from templates, history, memory |
| `MemoryConsolidator` | `memory.py` | Token-based history consolidation |
| `SubagentManager` | `subagent.py` | Spawns subagents for background tasks |

**Concurrency model:**
- Per-session serial: `asyncio.Lock` per `session_key`
- Cross-session parallel: different sessions run concurrently
- Priority commands (`/stop`, `/restart`) bypass session lock

### Provider Layer (`nanobot/providers/`)

**Purpose:** Pluggable LLM abstraction.

| Class | Responsibility |
|-------|----------------|
| `LLMProvider` | Abstract base with common interface |
| `OpenAICompatProvider` | OpenAI-compatible APIs (DeepSeek, Gemini, etc.) |
| `AnthropicProvider` | Native Anthropic SDK for Claude |
| `PROVIDERS` tuple | Registry of 20+ provider specs |

**Retry pattern:**
```python
_CHAT_RETRY_DELAYS = (1, 2, 4)  # Exponential backoff
_TRANSIENT_ERROR_MARKERS = ("429", "rate limit", "500", "502", ...)
```

### Tool Layer (`nanobot/agent/tools/`)

**Purpose:** Agent capabilities.

| Tool | Purpose |
|------|---------|
| `ExecTool` | Shell command execution |
| `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool` | File operations |
| `WebSearchTool`, `WebFetchTool` | Web access |
| `CronTool` | Cron scheduling |
| `SpawnTool` | Subagent spawning |
| `MCPToolWrapper` | MCP protocol integration |

**Tool base class:**
```python
class Tool(ABC):
    @property @abstractmethod def name(self) -> str
    @property @abstractmethod def description(self) -> str
    @property @abstractmethod def parameters(self) -> dict  # JSON Schema
    @abstractmethod async def execute(self, **kwargs) -> Any
```

### Session Layer (`nanobot/session/`)

**Storage:** JSONL files at `{workspace}/sessions/{session_key}.jsonl`

| Class | Responsibility |
|-------|----------------|
| `SessionManager` | Factory and registry for sessions |
| `Session` | Single conversation session |

### Configuration Layer (`nanobot/config/`)

**Approach:** Pydantic Settings with type-safe schema.

```python
class NanobotConfig(BaseSettings):
    model_config = ConfigDict(
        alias_generator=to_camel,
        env_prefix="NANOBOT_",
    )
```

## Key Design Patterns

| Pattern | Usage |
|---------|-------|
| **Adapter** | Channel implementations adapt platform APIs to BaseChannel |
| **Strategy** | LLMProvider is abstract with multiple implementations |
| **Registry/Plugin** | Channels, providers, tools, skills use centralized registries |
| **Template Method** | BaseChannel defines skeleton, subclasses implement specifics |
| **Command** | CommandRouter dispatches slash commands |
| **Message Queue** | MessageBus uses asyncio.Queue for async decoupling |

## Configuration-Driven Architecture

All components are instantiated via `Config` object:

```python
AgentLoop(
    bus=message_bus,
    provider=selected_provider,
    workspace=workspace,
    model=model,
    session_manager=session_manager,
    cron_service=cron_service,
    channels_config=channels_config,
)
```
