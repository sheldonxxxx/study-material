# CoPaw Architecture

## Architectural Pattern

CoPaw implements a **Multi-Layered Event-Driven Architecture with Plugin Extensibility**, combining:

1. **FastAPI Layer** - REST API and WebSocket endpoints
2. **Agent Runtime Layer** - AgentScope-based execution with ReAct pattern
3. **Channel Abstraction Layer** - Messaging platform integrations
4. **Provider Layer** - LLM provider abstractions
5. **CLI Layer** - Command-driven interface

### Design Principles

- **Dependency Injection** via factory functions and managers
- **Observer Pattern** for event streaming and hooks
- **Registry Pattern** for channel and provider discovery
- **Strategy Pattern** for multi-channel routing
- **Mixin Pattern** for cross-cutting concerns (security)

---

## Module Map

### Layer 1: Entry Points

| Module | Path | Responsibility |
|--------|------|----------------|
| CLI Entry | `src/copaw/cli/main.py` | Click-based command dispatcher |
| Module Entry | `src/copaw/__main__.py` | `python -m copaw` entry |

### Layer 2: Web Application (FastAPI)

| Module | Path | Responsibility |
|--------|------|----------------|
| Application Factory | `src/copaw/app/__init__.py` | FastAPI app creation |
| Route Handlers | `src/copaw/app/routers/*.py` | REST endpoints (agents, providers, skills, channels) |
| Agent Runner | `src/copaw/app/runner/runner.py` | Agent execution runtime (AgentRunner) |
| Session | `src/copaw/app/runner/session.py` | JSON-based session persistence |

### Layer 3: Agent Core

| Module | Path | Responsibility |
|--------|------|----------------|
| CoPawAgent | `src/copaw/agents/react_agent.py` | ReAct-based agent with tools, skills, memory |
| Model Factory | `src/copaw/agents/model_factory.py` | Chat model and formatter creation |
| Skills Manager | `src/copaw/agents/skills_manager.py` | Dynamic skill loading |
| Command Handler | `src/copaw/agents/command_handler.py` | Agent commands (/compact, /new, etc.) |
| Tool Guard Mixin | `src/copaw/agents/tool_guard_mixin.py` | Security interception for tool execution |

### Layer 4: Multi-Agent Management

| Module | Path | Responsibility |
|--------|------|----------------|
| MultiAgentManager | `src/copaw/app/multi_agent_manager.py` | Multiple agent instance lifecycle |
| Agent Context | `src/copaw/app/agent_context.py` | Thread-local agent context storage |
| Config Watcher | `src/copaw/agent_config_watcher.py` | Hot-reload agent configuration |

### Layer 5: Messaging Channels

| Module | Path | Responsibility |
|--------|------|----------------|
| BaseChannel | `src/copaw/app/channels/base.py` | Abstract channel base class |
| Channel Manager | `src/copaw/app/channels/manager.py` | Channel lifecycle, message routing, debouncing |
| Channel Registry | `src/copaw/app/channels/registry.py` | Built-in + custom channel discovery |
| Channel Implementations | `src/copaw/app/channels/{discord_,telegram,dingtalk,...}/` | Platform-specific integrations |

### Layer 6: LLM Providers

| Module | Path | Responsibility |
|--------|------|----------------|
| Provider Base | `src/copaw/providers/provider.py` | Abstract provider interface |
| Provider Manager | `src/copaw/providers/provider_manager.py` | Provider registry and lifecycle |
| OpenAI | `src/copaw/providers/openai_provider.py` | OpenAI API implementation |
| Anthropic | `src/copaw/providers/anthropic_provider.py` | Anthropic API implementation |
| Gemini | `src/copaw/providers/gemini_provider.py` | Google Gemini API implementation |
| Ollama | `src/copaw/providers/ollama_provider.py` | Local model implementation |

---

## Communication Patterns

### Pattern 1: Router -> Manager -> Provider

```
HTTP Request
    |
    v
routers/*.py (Route Handlers)
    |
    v
ProviderManager
    |
    v
Provider (OpenAI, Anthropic, etc.)
    |
    v
LLM API
```

**Evidence** (`src/copaw/app/routers/providers.py:168-171`):
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
BaseChannel (channel-specific)
    |
    v
ChannelManager (queue, debouncing)
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

### Pattern 3: Factory for Model Creation

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

**Evidence** (`src/copaw/agents/model_factory.py:219`):
```python
def create_model_and_formatter(agent_id: str = "default") -> Tuple[ChatModelBase, FormatterBase]:
    """Create model and formatter based on agent configuration."""
```

---

## Key Design Decisions

### 1. AgentScope Foundation

**Decision**: Inherit from `agentscope.agent.ReActAgent` instead of building from scratch.

**Rationale**: AgentScope provides battle-tested agent primitives (reasoning loop, tool execution, memory management) allowing CoPaw to focus on application-specific logic.

**Evidence** (`src/copaw/agents/react_agent.py:67`):
```python
class CoPawAgent(ToolGuardMixin, ReActAgent):
    """CoPaw Agent with integrated tools, skills, and memory management."""
```

### 2. Channel Plugin Architecture

**Decision**: Registry-based channel discovery with lazy loading.

**Rationale**: Allows adding new messaging platforms without modifying core code. Custom channels discovered from `custom_channels/` directory at runtime.

**Evidence** (`src/copaw/app/channels/registry.py:267-278`):
```python
_BUILTIN_SPECS: dict[str, tuple[str, str]] = {
    "imessage": (".imessage", "IMessageChannel"),
    "discord": (".discord_", "DiscordChannel"),
    "dingtalk": (".dingtalk", "DingTalkChannel"),
}

def get_channel_registry() -> dict[str, type[BaseChannel]]:
    out = _get_cached_builtin_channels()
    out.update(_discover_custom_channels())
    return out
```

### 3. Mixin for Security

**Decision**: Use mixin inheritance for tool guard security rather than composition.

**Rationale**: MRO (Method Resolution Order) interception cleanly wraps tool execution without modifying AgentScope core. Security checks apply to all agent variants automatically.

**Evidence** (`src/copaw/agents/tool_guard_mixin.py:324-327`):
```python
class ToolGuardMixin:
    async def _acting(self, *args, **kwargs) -> Msg:
        # Intercepts before tool execution
        # Checks tool_guard approval
```

### 4. Session Persistence

**Decision**: JSON file-based session storage via `SafeJSONSession`.

**Rationale**: Simple, inspectable, version-control-friendly state management without database dependency.

**Evidence** (`src/copaw/app/runner/session.py`):
```python
class SafeJSONSession:
    """Persists agent memory and state to JSON files."""
```

### 5. Hot-Reload Configuration

**Decision**: Filesystem watcher for agent config changes.

**Rationale**: Enables configuration updates without service restart in development and production.

**Evidence** (`src/copaw/agent_config_watcher.py`):
```python
class AgentConfigWatcher:
    """Filesystem watcher for agent configuration changes."""
    # Triggers schedule_agent_reload() on file changes
```

---

## Data Flow

### Incoming Message Flow

```
Telegram/Discord/DingTalk Webhook
    -> Channel Route Handler (FastAPI)
    -> ChannelManager.enqueue()
    -> ChannelManager._consume_channel_loop()
    -> BaseChannel.consume_one()
    -> BaseChannel._consume_one_request()
    -> AgentRunner.query_handler()
    -> CoPawAgent.reply()
    -> ReAct loop: reasoning() -> acting() -> tools
    -> Stream response back
    -> ChannelManager.send_text/send_event()
    -> Platform API (Telegram, Discord, etc.)
```

### Configuration Flow

```
CLI: copaw agents create
    -> config/*.py (pydantic models)
    -> ProviderManager
    -> AgentConfigWatcher
    -> MultiAgentManager
```

---

## Technology Integration

### AgentScope Integration Points

| Component | AgentScope Class |
|-----------|------------------|
| Base Agent | `agentscope.agent.ReActAgent` |
| Model Abstraction | `agentscope.model.ChatModelBase` |
| Message Formatting | `agentscope.formatter.FormatterBase` |
| Session Memory | `agentscope.memory.InMemoryMemory` |
| Runtime Interface | `agentscope_runtime.engine.runner.Runner` |

### MCP Integration

| Component | Usage |
|-----------|-------|
| `agentscope.mcp.HttpStatefulClient` | HTTP-based MCP servers |
| `agentscope.mcp.StdIOStatefulClient` | STDIO-based MCP servers |

---

## Extension Points

| Purpose | Primary File | Template/Reference |
|---------|--------------|-------------------|
| Add LLM provider | `providers/provider.py` | `providers/openai_provider.py` |
| Add messaging channel | `channels/base.py` | `channels/registry.py` |
| Add built-in tool | `agents/tools/*.py` | `agents/react_agent.py` |
| Add skill type | `agents/skills_manager.py` | `agents/skills_hub.py` |
| Modify reasoning | `agents/react_agent.py` | `agents/tool_guard_mixin.py` |
