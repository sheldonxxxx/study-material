# Nanobot Architecture & Design Patterns

## Architectural Pattern: Event-Driven with Message Bus

Nanobot employs an **event-driven architecture** with an **asynchronous message bus** as the central communication hub. This design decouples chat platform integrations (channels) from the core agent processing engine, enabling:

- Multiple chat platforms to operate concurrently
- Hot-swapping of LLM providers without affecting other components
- Per-session serialization with cross-session parallelism
- Clean separation of concerns

```
┌─────────────────────────────────────────────────────────────────────┐
│                         nanobot Process                              │
│                                                                      │
│  ┌──────────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │   Channels   │         │  MessageBus │         │  AgentLoop   │  │
│  │  (adapters) │◄───────►│  (queues)   │◄───────►│  (engine)    │  │
│  └──────────────┘         └─────────────┘         └──────────────┘  │
│         │                        ▲                        │         │
│         │                        │                        │         │
│         ▼                        │                        ▼         │
│  ┌──────────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │  InboundMsg  │         │  OutboundMsg│         │  SessionMgr  │  │
│  │  (events)   │         │  (events)   │         │  (JSONL)     │  │
│  └──────────────┘         └─────────────┘         └──────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Supporting Services                      │  │
│  │  CronService  │  HeartbeatService  │  ConfigService         │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ WebSocket
                              ▼
                    ┌──────────────────┐
                    │   WhatsApp Bridge │
                    │   (TypeScript)    │
                    └──────────────────┘
```

---

## Major Modules and Responsibilities

### 1. Bus Layer (`nanobot/bus/`)

**Purpose:** Decouple channels from the agent via async message queues.

| File | Class/Module | Responsibility |
|------|--------------|----------------|
| `events.py` | `InboundMessage` | Dataclass for incoming messages from chat platforms |
| `events.py` | `OutboundMessage` | Dataclass for outgoing messages to chat platforms |
| `queue.py` | `MessageBus` | Async queue manager with `inbound` and `outbound` queues |

**Key interfaces:**

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]

    async def publish_inbound(msg: InboundMessage)
    async def consume_inbound() -> InboundMessage
    async def publish_outbound(msg: OutboundMessage)
    async def consume_outbound() -> OutboundMessage
```

**Design:** Pure queue-based decoupling. No topic routing, no subscription management. Simple but effective for the use case.

---

### 2. Channel Layer (`nanobot/channels/`)

**Purpose:** Abstract adapter layer for chat platform integrations.

| File | Class | Responsibility |
|------|-------|----------------|
| `base.py` | `BaseChannel` | Abstract base defining the channel interface |
| `manager.py` | `ChannelManager` | Routes messages to/from channel instances |
| `registry.py` | `ChannelRegistry` | Plugin registry for channels |
| `telegram.py` | `TelegramChannel` | Telegram Bot API integration |
| `discord.py` | `DiscordChannel` | Discord bot integration |
| `slack.py` | `SlackChannel` | Slack app integration |
| `feishu.py` | `FeishuChannel` | Feishu/Lark integration |
| `whatsapp.py` | `WhatsAppChannel` | WhatsApp (via bridge) integration |

**BaseChannel interface:**

```python
class BaseChannel(ABC):
    name: str
    display_name: str

    def __init__(self, config: Any, bus: MessageBus)

    @abstractmethod
    async def start(self) -> None

    @abstractmethod
    async def stop(self) -> None

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None

    async def _handle_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        media: list[str] | None = None,
        metadata: dict | None = None,
        session_key: str | None = None,
    ) -> None
```

**Key behaviors:**
- `_handle_message()` validates sender against `allow_from` config, then publishes `InboundMessage` to the bus
- `send()` raises on delivery failure for retry handling by ChannelManager
- `supports_streaming` property detects if subclass overrides `send_delta()`
- `is_allowed()` implements allowlist security

---

### 3. Agent Layer (`nanobot/agent/`)

**Purpose:** Core processing engine that consumes messages and produces responses.

| File | Class | Responsibility |
|------|-------|----------------|
| `loop.py` | `AgentLoop` | Main async processing loop, tool execution |
| `context.py` | `ContextBuilder` | Constructs prompts from templates, history, memory, skills |
| `memory.py` | `MemoryConsolidator` | Token-based history consolidation |
| `subagent.py` | `SubagentManager` | Spawns sub-agents for background tasks |

**AgentLoop processing flow:**

```
consume_inbound() → _dispatch()
  ├── Priority commands (/stop, /restart) → dispatch_priority()
  └── Per-session lock → _process_message()
        ├── Slash command dispatch
        ├── memory_consolidator.maybe_consolidate_by_tokens()
        ├── context.build_messages()  ← builds prompt
        └── _run_agent_loop()
              ├── provider.chat_stream_with_retry() or chat_with_retry()
              ├── If tool_calls → execute all concurrently via asyncio.gather()
              │     └── Loop back with tool results
              └── If no tool_calls → return final content
```

**Concurrency model:**
- Per-session serial: `asyncio.Lock` per `session_key`
- Cross-session parallel: different locks, different tasks
- Priority commands bypass the session lock
- Optional `_concurrency_gate` for global throttling

---

### 4. Session Layer (`nanobot/session/`)

**Purpose:** Manage conversation history with JSONL persistence.

| File | Class | Responsibility |
|------|-------|----------------|
| `manager.py` | `SessionManager` | Factory and registry for sessions |
| `manager.py` | `Session` | Single conversation session (messages + metadata) |

**Storage:** JSONL files at `{workspace}/sessions/{session_key}.jsonl`

**Key operations:**
- `get_or_create(key)` - Get existing or create new session
- `save(session)` - Persist to JSONL
- `get_history(max_messages=0)` - Retrieve history with optional truncation

---

### 5. Provider Layer (`nanobot/providers/`)

**Purpose:** Pluggable LLM provider abstraction.

| File | Class | Responsibility |
|------|-------|----------------|
| `base.py` | `LLMProvider` | Abstract base with common interface |
| `base.py` | `ToolCallRequest` | Tool call request dataclass |
| `base.py` | `LLMResponse` | Response dataclass with content + tool_calls |
| `base.py` | `GenerationSettings` | Default generation parameters |
| `anthropic_provider.py` | `AnthropicProvider` | Native Anthropic SDK for Claude |
| `openai_compat_provider.py` | `OpenAICompatProvider` | OpenAI-compatible API (OpenAI, DeepSeek, etc.) |
| `azure_openai_provider.py` | `AzureOpenAIProvider` | Azure OpenAI Service |
| `registry.py` | `PROVIDERS` tuple | All provider specs |
| `registry.py` | `find_by_name()` | Lookup provider by name |

**LLMProvider interface:**

```python
class LLMProvider(ABC):
    generation: GenerationSettings  # defaults (temperature, max_tokens)

    @abstractmethod
    async def chat(
        messages: list[dict],
        tools: list[dict] | None = None,
        model: str | None = None,
        max_tokens: int = 4096,
        temperature: float = 0.7,
    ) -> LLMResponse

    async def chat_stream(
        self,
        messages: list[dict],
        on_content_delta: Callable[[str], Awaitable[None]] | None = None,
    ) -> LLMResponse

    async def chat_with_retry(...) -> LLMResponse
    async def chat_stream_with_retry(...) -> LLMResponse

    @abstractmethod
    def get_default_model(self) -> str
```

**Key design decisions:**
- `chat_with_retry()` and `chat_stream_with_retry()` implement exponential backoff for transient errors
- `_safe_chat()` wraps exceptions into error `LLMResponse` (never raises)
- Message sanitization handles empty content blocks, image_url stripping
- `ToolCallRequest.to_openai_tool_call()` normalizes all providers to OpenAI format

---

### 6. Tool Layer (`nanobot/agent/tools/`)

**Purpose:** Agent capabilities (file operations, shell, web, messaging, etc.)

| File | Class | Responsibility |
|------|-------|----------------|
| `base.py` | `Tool` | Abstract base with parameter validation |
| `registry.py` | `ToolRegistry` | Tool registration and execution |
| `filesystem.py` | `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool` | File operations |
| `shell.py` | `ExecTool` | Shell command execution |
| `web.py` | `WebFetchTool`, `WebSearchTool` | Web access |
| `message.py` | `MessageTool` | Send messages to channels |
| `spawn.py` | `SpawnTool` | Spawn subprocesses |
| `cron.py` | `CronTool` | Cron scheduling |
| `mcp.py` | `MCPTool` | Model Context Protocol integration |

**Tool interface:**

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str

    @property
    @abstractmethod
    def description(self) -> str

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]  # JSON Schema

    @abstractmethod
    async def execute(self, **kwargs: Any) -> Any

    def validate_params(params: dict) -> list[str]
    def cast_params(params: dict) -> dict
    def to_schema(self) -> dict  # OpenAI function format
```

**ToolRegistry execution:**

```python
class ToolRegistry:
    async def execute(self, name: str, arguments: dict) -> Any:
        tool = self._tools[name]
        validated = tool.cast_params(arguments)
        return await tool.execute(**validated)
```

---

### 7. Command Layer (`nanobot/command/`)

**Purpose:** Slash command routing.

| File | Class | Responsibility |
|------|-------|----------------|
| `router.py` | `CommandRouter` | Dict-based command dispatch |
| `router.py` | `CommandContext` | Dataclass with msg, session, key, raw, args |
| `builtin.py` | (handlers) | Built-in command implementations |

**Command dispatch tiers:**
1. **Priority** (exact match, outside session lock) - `/stop`, `/restart`
2. **Exact** (exact match, inside session lock)
3. **Prefix** (longest-prefix-first, e.g., `/team `)
4. **Interceptors** (fallback predicates)

---

### 8. Skills Layer (`nanobot/skills/`)

**Purpose:** Agent skill templates loaded from workspace.

| File | Class | Responsibility |
|------|-------|----------------|
| `base.py` | `Skill` | Base class for skills |
| `registry.py` | `SkillRegistry` | Skill registration |
| `skills.py` | `SkillsLoader` | Loads skill markdown from workspace |

**Skill templates:** AGENTS.md, SOUL.md, USER.md, TOOLS.md, memory/

**ContextBuilder loads:**
1. Bootstrap files from workspace
2. Memory context from `{workspace}/memory/MEMORY.md`
3. Active (always-on) skills
4. Skills summary with availability status

---

### 9. Configuration Layer (`nanobot/config/`)

| File | Class | Responsibility |
|------|-------|----------------|
| `schema.py` | Pydantic models | `Config`, `ChannelsConfig`, `ProvidersConfig`, etc. |
| `service.py` | `ConfigService` | Config loading and validation |
| `loader.py` | Config file loading | YAML/JSON config file parsing |
| `paths.py` | Path utilities | Workspace, session, history path resolution |

---

### 10. WhatsApp Bridge (`bridge/`)

**Purpose:** TypeScript/Baileys-based WhatsApp Web integration.

```
bridge/src/
  index.ts      - Entry point
  server.ts     - WebSocket server for Python bridge communication
  whatsapp.ts   - Baileys WhatsApp client
  types.d.ts    - TypeScript type definitions
```

**Communication:** WebSocket between TypeScript bridge and Python backend (which presents as a `whatsapp` channel).

---

## Component Communication Flows

### Message Flow: Inbound

```
1. Chat platform (Telegram/Discord/etc.)
   │
2. Channel adapter (e.g., TelegramChannel)
   │
3. BaseChannel._handle_message()
   - Checks is_allowed(sender_id)
   - Creates InboundMessage
   │
4. MessageBus.publish_inbound()
   │
5. AgentLoop.consume_inbound()
```

### Message Flow: Outbound

```
1. AgentLoop._run_agent_loop() produces final_content
   │
2. OutboundMessage created
   │
3. MessageBus.publish_outbound()
   │
4. ChannelManager.consume_outbound()  [in separate task per channel]
   │
5. Channel adapter (e.g., TelegramChannel.send())
   │
6. Chat platform API
```

### Tool Call Flow

```
1. LLM returns LLMResponse with tool_calls
   │
2. For each tool_call:
   - ToolRegistry.execute(name, arguments)
   - Tool.execute(**validated_kwargs)
   │
3. Results added to messages as tool role messages
   │
4. Loop back to LLM with updated messages
```

### Streaming Flow

```
1. LLM response arrives in chunks via on_content_delta callback
   │
2. _filtered_stream() strips <think> blocks
   │
3. OutboundMessage published with:
   - content = delta text
   - metadata._stream_delta = True
   - metadata._stream_id = unique per segment
   │
4. Channel adapter receives via send_delta() override
   │
5. When complete, stream_end published with _stream_end=True
```

---

## Key Design Patterns

### 1. Adapter Pattern
Channel implementations adapt different chat platform APIs to the unified `BaseChannel` interface. Each channel handles platform-specific quirks (Telegram's 4000 char limit, markdown rendering, etc.).

### 2. Strategy Pattern
`LLMProvider` is an abstract base with multiple implementations. The agent code is provider-agnostic; any provider can be swapped in via configuration.

### 3. Registry/Plugin Pattern
Centralized registries for channels, providers, tools, and skills enable:
- Runtime registration
- Discovery via configuration
- Extensibility without modifying core code

```python
# Example: Provider discovery
spec = find_by_name("anthropic")
if not spec:
    spec = PROVIDERS[-1]  # custom fallback
```

### 4. Template Method Pattern
`BaseChannel` defines the skeleton with `_handle_message()`, while subclasses implement `start()`, `stop()`, `send()`. Similarly, `Tool` defines the execution skeleton with `validate_params()` and `cast_params()` hooks.

### 5. Command Pattern
`CommandRouter` dispatches slash commands to handlers. Four-tier dispatch (priority, exact, prefix, interceptors) provides flexibility.

### 6. Message Queue Pattern
`MessageBus` uses `asyncio.Queue` for async producer-consumer decoupling. Channels produce, AgentLoop consumes. ChannelManager consumes outbound and routes to channels.

### 7. Observer Pattern (via callbacks)
Streaming uses callback functions (`on_content_delta`, `on_stream_end`, `on_progress`) for real-time response delivery to UI.

### 8. Semaphore/Gate Pattern
Optional `_concurrency_gate` in AgentLoop for global throttling when needed.

---

## Data Flow Diagram

```
                           ┌─────────────────────────────────────────────────────────┐
                           │                      nanobot                             │
                           │                                                         │
Chat Platform              │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
(e.g. Telegram) ─────────►│   │   Channel   │───►│  MessageBus │◄───│ AgentLoop    │  │
                           │   │   Adapter   │    │   (queue)   │    │             │  │
                           │   └─────────────┘    └─────────────┘    └──────┬──────┘  │
                           │          ▲                                        │         │
                           │          │                                        ▼         │
                           │          │                              ┌─────────────────┐│
                           │          │                              │ ContextBuilder ││
                           │          │                              │  - Bootstrap   ││
                           │          │                              │  - Memory      ││
                           │          │                              │  - Skills      ││
                           │          │                              └────────┬────────┘│
                           │          │                                       │         │
                           │          │                                       ▼         │
                           │          │                              ┌─────────────────┐│
                           │          │                              │  LLMProvider   ││
                           │          │                              │  (Strategy)    ││
                           │          │                              └────────┬────────┘│
                           │          │                                       │         │
                           │          │                                       ▼         │
                           │          │                              ┌─────────────────┐│
                           │          │                              │  ToolRegistry   ││
                           │          │                              │  + Tools        ││
                           │          │                              └────────┬────────┘│
                           │          │                                       │         │
                           │          │                              ┌────────▼────────┐│
                           │          │                              │ SessionManager  ││
                           │          │                              │ (JSONL storage) ││
                           │          │                              └─────────────────┘│
                           │          │                                        │         │
                           │          │                                        ▼         │
                           │          │                              ┌─────────────────┐│
                           │          │                              │ OutboundMessage ││
                           │          │                              └────────┬────────┘│
                           │          │                                       │         │
                           └──────────┼───────────────────────────────────────┼─────────┘
                                      │                                       │
                                      ▼                                       ▼
                           ┌───────────────────┐                   ┌───────────────────┐
                           │  Chat Platform    │◄──────────────────│  Another Channel  │
                           │  (Response)       │                   │  (Response)        │
                           └───────────────────┘                   └───────────────────┘
```

---

## Session Concurrency Model

```
Session A (user:alice) ────► [Lock A] ───► Processing...
Session B (user:bob) ──────► [Lock B] ───► Processing... (parallel)

AgentLoop.run():
    while running:
        msg = await consume_inbound()
        if is_priority(msg):
            dispatch_priority()      # no lock
        else:
            task = create_task(_dispatch(msg))
            _active_tasks[session_key].append(task)
```

---

## Configuration-Driven Architecture

All components are instantiated and wired via `Config` object:

```python
Config:
  providers: ProvidersConfig  # which LLM providers
  channels: ChannelsConfig    # which channels to enable
  agent_defaults: AgentDefaults
  workspace: Path
  model: str
  timezone: str
```

The `AgentLoop` receives fully-configured dependencies:

```python
AgentLoop(
    bus=message_bus,
    provider=selected_provider,  # from config
    workspace=workspace,
    model=model,
    session_manager=session_manager,
    cron_service=cron_service,
    channels_config=channels_config,
    # ...
)
```

---

## Dependency Injection Points

| Component | Injected Into | Via |
|-----------|---------------|-----|
| `MessageBus` | `AgentLoop`, `BaseChannel` | Constructor |
| `LLMProvider` | `AgentLoop` | Constructor |
| `SessionManager` | `AgentLoop` | Constructor |
| `ToolRegistry` | `AgentLoop` | Constructor |
| `CronService` | `AgentLoop` | Constructor |
| `ContextBuilder` | `AgentLoop` | Constructor (internal) |

---

## Key Abstractions Summary

| Abstraction | Location | Purpose |
|-------------|----------|---------|
| `InboundMessage` | `bus/events.py` | Unified inbound event |
| `OutboundMessage` | `bus/events.py` | Unified outbound event |
| `MessageBus` | `bus/queue.py` | Async queue hub |
| `BaseChannel` | `channels/base.py` | Channel plugin interface |
| `LLMProvider` | `providers/base.py` | LLM strategy interface |
| `Tool` | `agent/tools/base.py` | Tool capability interface |
| `Session` | `session/manager.py` | Conversation context |
| `CommandContext` | `command/router.py` | Command execution context |
| `ContextBuilder` | `agent/context.py` | Prompt construction |
| `GenerationSettings` | `providers/base.py` | Default generation params |

---

## File Organization Patterns

| Pattern | Example | Purpose |
|---------|---------|---------|
| `base.py` | `channels/base.py`, `providers/base.py`, `agent/tools/base.py` | Abstract base class |
| `registry.py` | `channels/registry.py`, `providers/registry.py`, `agent/tools/registry.py` | Plugin registry |
| `manager.py` | `session/manager.py`, `channels/manager.py` | Central management |
| `*/tools/*` | `agent/tools/` | Plugin implementations |
| `*/skills/*` | `agent/skills/`, `nanobot/skills/` | Skill templates |

---

## Conclusion

Nanobot's architecture follows a **modular, event-driven design** that prioritizes:

1. **Decoupling** via async message bus
2. **Extensibility** via plugin registries
3. **Pluggable strategies** via provider abstraction
4. **Security** via allowlist-based access control
5. **Resilience** via retry logic and graceful error handling
6. **Observability** via structured logging (Loguru)

The architecture is well-suited for a personal AI assistant that must connect to multiple chat platforms while remaining provider-agnostic and tool-extensible.
