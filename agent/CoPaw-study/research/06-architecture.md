# CoPaw Architecture & Design Patterns Analysis

## Architectural Pattern: Multi-Layered Event-Driven Architecture with Plugin Patterns

CoPaw implements a **layered, modular architecture** with clear separation of concerns and plugin-based extensibility. The system combines:

1. **FastAPI Layer** (Web/REST API)
2. **Agent Runtime Layer** (AgentScope-based execution)
3. **Channel Abstraction Layer** (messaging integrations)
4. **Provider Layer** (LLM integrations)
5. **CLI Layer** (command-driven interface)

The architecture follows principles of:
- **Dependency Injection** via factory functions and managers
- **Observer Pattern** for event streaming and hooks
- **Registry Pattern** for channel and provider discovery
- **Strategy Pattern** for multi-channel routing

---

## Module Map with Responsibilities

### Layer 1: Entry Points & CLI

| Module | Path | Responsibility |
|--------|------|----------------|
| `cli/main.py` | `src/copaw/cli/main.py` | Click-based CLI entry point, delegates to subcommands |
| `__main__.py` | `src/copaw/__main__.py` | `python -m copaw` entry point |

### Layer 2: Web Application (FastAPI)

| Module | Path | Responsibility |
|--------|------|----------------|
| `app/__init__` | `src/copaw/app/` | FastAPI application factory |
| `routers/*` | `src/copaw/app/routers/` | Route handlers (agents, providers, skills, channels, etc.) |
| `runner/runner.py` | `src/copaw/app/runner/runner.py` | Agent execution runtime (AgentRunner) |

### Layer 3: Agent Core

| Module | Path | Responsibility |
|--------|------|----------------|
| `agents/react_agent.py` | `src/copaw/agents/react_agent.py` | CoPawAgent - ReAct-based agent with tools, skills, memory |
| `agents/model_factory.py` | `src/copaw/agents/model_factory.py` | Factory for creating chat models and formatters |
| `agents/skills_manager.py` | `src/copaw/agents/skills_manager.py` | Dynamic skill loading and management |
| `agents/command_handler.py` | `src/copaw/agents/command_handler.py` | Agent command processing (/compact, /new, etc.) |
| `agents/tool_guard_mixin.py` | `src/copaw/agents/tool_guard_mixin.py` | Security mixin for tool execution guards |

### Layer 4: Multi-Agent Management

| Module | Path | Responsibility |
|--------|------|----------------|
| `multi_agent_manager.py` | `src/copaw/multi_agent_manager.py` | Manager for multiple agent instances |
| `agent_context.py` | `src/copaw/agent_context.py` | Thread-local agent context storage |
| `agent_config_watcher.py` | `src/copaw/agent_config_watcher.py` | Hot-reload agent configuration on file changes |

### Layer 5: Messaging Channels

| Module | Path | Responsibility |
|--------|------|----------------|
| `channels/base.py` | `src/copaw/app/channels/base.py` | Abstract BaseChannel class |
| `channels/manager.py` | `src/copaw/app/channels/manager.py` | Channel lifecycle and message routing |
| `channels/registry.py` | `src/copaw/app/channels/registry.py` | Built-in + custom channel discovery |

### Layer 6: LLM Providers

| Module | Path | Responsibility |
|--------|------|----------------|
| `providers/provider.py` | `src/copaw/providers/provider.py` | Abstract Provider base class |
| `providers/provider_manager.py` | `src/copaw/providers/provider_manager.py` | Provider registry and lifecycle |
| `providers/openai_provider.py` | `src/copaw/providers/openai_provider.py` | OpenAI API implementation |
| `providers/anthropic_provider.py` | `src/copaw/providers/anthropic_provider.py` | Anthropic API implementation |
| `providers/gemini_provider.py` | `src/copaw/providers/gemini_provider.py` | Google Gemini API implementation |
| `providers/ollama_provider.py` | `src/copaw/providers/ollama_provider.py` | Ollama local model implementation |

---

## Key Interfaces and Abstractions

### 1. Provider Interface (`Provider`)

```python
# src/copaw/providers/provider.py:100-202
class Provider(ProviderInfo, ABC):
    """Represents a provider instance with its configuration."""

    @abstractmethod
    async def check_connection(self, timeout: float = 5) -> tuple[bool, str]: ...

    @abstractmethod
    async def fetch_models(self, timeout: float = 5) -> List[ModelInfo]: ...

    @abstractmethod
    async def check_model_connection(self, model_id: str, timeout: float = 5) -> tuple[bool, str]: ...

    @abstractmethod
    def get_chat_model_instance(self, model_id: str) -> ChatModelBase: ...
```

**Pattern**: Abstract Base Class (ABC) with Template Method pattern for connection checking.

### 2. BaseChannel Interface (`BaseChannel`)

```python
# src/copaw/app/channels/base.py:69-868
class BaseChannel(ABC):
    channel: ChannelType

    async def consume_one(self, payload: Any) -> None: ...
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
    async def send(self, to_handle: str, text: str, meta: Optional[Dict[str, Any]] = None) -> None: ...
```

**Pattern**: Template Method - `consume_one()` delegates to `_consume_one_request()` with hooks for debouncing.

### 3. Agent Runner Interface (`AgentRunner`)

```python
# src/copaw/app/runner/runner.py:61-539
class AgentRunner(Runner):
    async def query_handler(self, msgs, request: AgentRequest = None, **kwargs): ...
    async def init_handler(self, *args, **kwargs): ...
    async def shutdown_handler(self, *args, **kwargs): ...
```

**Pattern**: Inherits from `agentscope_runtime.engine.runner.Runner` - strategy pattern for different runtime engines.

### 4. CoPawAgent Class Hierarchy

```
ReActAgent (AgentScope)
    └── ToolGuardMixin (Security interception)
            └── CoPawAgent (Main implementation)
```

```python
# src/copaw/agents/react_agent.py:67
class CoPawAgent(ToolGuardMixin, ReActAgent):
    """CoPaw Agent with integrated tools, skills, and memory management."""
```

**Pattern**: Mixin inheritance for cross-cutting concerns (security).

---

## Communication Patterns Between Components

### Pattern 1: FastAPI Router -> Service -> Provider

```
HTTP Request
    |
    v
routers/*.py (Route Handlers)
    |
    v
ProviderManager (provider_manager.py)
    |
    v
Provider (openai_provider.py, etc.)
    |
    v
LLM API (OpenAI, Anthropic, etc.)
```

**Evidence** (`src/copaw/app/routers/providers.py`):
```python
@router.get("/providers")
async def list_providers() -> List[dict]:
    providers = provider_manager.list_providers()
    return [p.get_info() for p in providers]
```

### Pattern 2: Channel -> Manager -> AgentRunner -> Agent

```
External Message (Telegram, Discord, etc.)
    |
    v
BaseChannel (channel-specific implementation)
    |
    v
ChannelManager (message queue, debouncing)
    |
    v
AgentRunner.query_handler() (streaming)
    |
    v
CoPawAgent (tool execution, reasoning)
```

**Evidence** (`src/copaw/app/channels/manager.py:322-364`):
```python
async def _consume_channel_loop(self, channel_id: str, worker_index: int) -> None:
    q = self._queues.get(channel_id)
    while True:
        payload = await q.get()
        batch = _drain_same_key(q, ch, key, payload)
        await _process_batch(ch, batch)
```

### Pattern 3: Factory Pattern for Model Creation

```
CoPawAgent.__init__()
    |
    v
model_factory.create_model_and_formatter()
    |
    v
ProviderManager -> Provider.get_chat_model_instance()
    |
    v
AgentScope ChatModel (OpenAIChatModel, AnthropicChatModel, etc.)
```

**Evidence** (`src/copaw/agents/model_factory.py`):
```python
def create_model_and_formatter(agent_id: str = "default") -> Tuple[ChatModelBase, FormatterBase]:
    """Create model and formatter based on agent configuration."""
    # ... delegates to ProviderManager
```

---

## Multi-Agent Runtime Architecture

### Agent Instance Lifecycle

```
copaw app (CLI)
    |
    v
AgentApp (FastAPI app factory)
    |
    v
MultiAgentManager
    |
    v
AgentRunner (per session)
    |
    v
CoPawAgent (per query, with session state)
```

### Session Management

**Evidence** (`src/copaw/app/runner/session.py`):
- `SafeJSONSession`: Persists agent memory and state to JSON files
- Session state includes: memory content, system prompt, agent configuration

### Hot-Reload Mechanism

**Evidence** (`src/copaw/agent_config_watcher.py`):
- `AgentConfigWatcher`: Filesystem watcher for agent configuration changes
- Triggers `schedule_agent_reload()` to update running agents without restart

---

## Design Patterns in Use

### 1. Registry Pattern (Channel Discovery)

**File**: `src/copaw/app/channels/registry.py`

```python
_BUILTIN_SPECS: dict[str, tuple[str, str]] = {
    "imessage": (".imessage", "IMessageChannel"),
    "discord": (".discord_", "DiscordChannel"),
    "dingtalk": (".dingtalk", "DingTalkChannel"),
    # ...
}

def get_channel_registry() -> dict[str, type[BaseChannel]]:
    """Built-in channel classes + custom channels from custom_channels/."""
    out = _get_cached_builtin_channels()
    out.update(_discover_custom_channels())
    return out
```

**Benefit**: Plugins discovered at runtime without hard-coded imports.

### 2. Strategy Pattern (Multi-Channel Routing)

**File**: `src/copaw/app/channels/base.py`

```python
class BaseChannel(ABC):
    channel: ChannelType

    def resolve_session_id(self, sender_id: str, channel_meta: Optional[Dict] = None) -> str:
        """Map sender and optional channel meta to session_id.
        Override in subclasses for channel-specific session keys."""
        return f"{self.channel}:{sender_id}"
```

**Benefit**: Each channel implements its own session resolution strategy.

### 3. Factory Pattern (Model Creation)

**File**: `src/copaw/agents/model_factory.py`

```python
_CHAT_MODEL_FORMATTER_MAP: dict[Type[ChatModelBase], Type[FormatterBase]] = {
    OpenAIChatModel: OpenAIChatFormatter,
    AnthropicChatModel: AnthropicChatFormatter,
    GeminiChatModel: GeminiChatFormatter,
}

def create_model_and_formatter(agent_id: str = "default") -> Tuple[ChatModelBase, FormatterBase]:
    # ... dynamic lookup and instantiation
```

**Benefit**: Decouples model type from instantiation logic.

### 4. Mixin Pattern (Cross-Cutting Concerns)

**File**: `src/copaw/agents/tool_guard_mixin.py`

```python
class ToolGuardMixin:
    """Security mixin that intercepts tool calls for approval."""

    async def _acting(self, *args, **kwargs) -> Msg:
        # Intercepts before tool execution
        # Checks tool_guard approval
```

**Usage** (`src/copaw/agents/react_agent.py:67`):
```python
class CoPawAgent(ToolGuardMixin, ReActAgent):
    """MRO: CoPawAgent -> ToolGuardMixin -> ReActAgent"""
```

**Benefit**: Reusable security logic across different agent implementations.

### 5. Observer Pattern (Event Streaming)

**File**: `src/copaw/app/runner/runner.py`

```python
async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),
):
    yield msg, last
```

**Benefit**: Async iteration over agent response stream.

### 6. Template Method Pattern (Channel Processing)

**File**: `src/copaw/app/channels/base.py`

```python
async def consume_one(self, payload: Any) -> None:
    """Process one payload - template method with hooks."""
    if self._debounce_seconds > 0 and self._is_native_payload(payload):
        # ... debounce logic
        return
    await self._consume_one_request(payload)

async def _consume_one_request(self, payload: Any) -> None:
    """Subclass Override Point - actual processing."""
    request = self._payload_to_request(payload)
    await self._run_process_loop(request, to_handle, send_meta)
```

**Benefit**: Fixed algorithm with customizable steps.

### 7. Hook Pattern (Agent Extensibility)

**Files**: `src/copaw/agents/hooks/*.py`

```python
class BootstrapHook:
    """First-time setup guidance - called pre-reasoning."""
    def __call__(self, *args, **kwargs) -> None: ...

class MemoryCompactionHook:
    """Automatic context window management - called pre-reasoning."""
    def __call__(self, *args, **kwargs) -> None: ...

# Registration (react_agent.py:352-380)
self.register_instance_hook(
    hook_type="pre_reasoning",
    hook_name="bootstrap_hook",
    hook=bootstrap_hook.__call__,
)
```

**Benefit**: Pluggable pre/post processing steps.

### 8. Debounce/Queue Pattern (Message Handling)

**File**: `src/copaw/app/channels/manager.py`

```python
def _drain_same_key(q: asyncio.Queue, ch: BaseChannel, key: str, first_payload: Any) -> List[Any]:
    """Drain queue of payloads with same debounce key; return batch."""
    batch = [first_payload]
    while True:
        try:
            p = q.get_nowait()
        except asyncio.QueueEmpty:
            break
        if ch.get_debounce_key(p) == key:
            batch.append(p)
    return batch
```

**Benefit**: Coalesces rapid messages from same session before processing.

---

## Data Flow Summary

### Incoming Message Flow
```
Telegram/Discord/DingTalk/etc. Webhook
    -> Channel-specific Route Handler (FastAPI)
    -> ChannelManager.enqueue()
    -> ChannelManager._consume_channel_loop() [async workers]
    -> BaseChannel.consume_one()
    -> BaseChannel._consume_one_request()
    -> AgentRunner.query_handler() [streaming]
    -> CoPawAgent.reply()
    -> ReAct loop: _reasoning() -> _acting() -> tool execution
    -> Stream response back through Channel
    -> ChannelManager.send_text/send_event()
    -> Platform-specific send (Telegram API, Discord API, etc.)
```

### Configuration Flow
```
CLI: copaw agents create
    -> config/*.py (pydantic models)
    -> ProviderManager (provider instances)
    -> AgentConfigWatcher (filesystem monitoring)
    -> MultiAgentManager (runtime instances)
```

---

## Technology Integration Points

### AgentScope Integration
- `agentscope.agent.ReActAgent` - Base agent class
- `agentscope.model.ChatModelBase` - Model abstraction
- `agentscope.formatter.FormatterBase` - Message formatting
- `agentscope.memory.InMemoryMemory` - Session memory
- `agentscope_runtime.engine.runner.Runner` - Runtime interface

### MCP (Model Context Protocol) Integration
- `agentscope.mcp.HttpStatefulClient` - HTTP-based MCP
- `agentscope.mcp.StdIOStatefulClient` - STDIO-based MCP
- Hot-reload support via `_copaw_rebuild_info`

---

## Security Architecture

### Tool Guard (`ToolGuardMixin`)
- Intercepts `_acting()` method via MRO
- Checks tool against `ToolGuard`
- Approval workflow for sensitive tools
- Timeout handling for pending approvals

### Skill Scanner
- Scans skill code for security concerns before loading
- Applied during `SkillsManager.load_skill()`

---

## Key Files for Extension

| Purpose | Primary File | Secondary Files |
|---------|--------------|-----------------|
| Add new LLM provider | `providers/provider.py` | `providers/openai_provider.py` (template) |
| Add new messaging channel | `channels/base.py` | `channels/registry.py` |
| Add new built-in tool | `agents/tools/*.py` | `agents/react_agent.py` (_create_toolkit) |
| Add new skill type | `agents/skills_manager.py` | `agents/skills_hub.py` |
| Modify agent reasoning | `agents/react_agent.py` | `agents/tool_guard_mixin.py` |
