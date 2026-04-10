# Architecture & Module Map: hermes-agent

**Project:** Self-improving AI agent by Nous Research
**Source:** `/Users/sheldon/Documents/claw/hermes-agent-study/research/06-architecture.md`
**Date:** 2026-03-27

---

## Architectural Pattern

**hermes-agent** is a **modular monolith with plugin-based extensions**:

| Layer | Pattern | Description |
|-------|---------|-------------|
| Core | Layered architecture | CLI -> Agent -> Tools |
| Gateway | Plugin/Adapter pattern | 12+ messaging platform integrations |
| Tools | Registry pattern | Self-registering tool modules |
| Gateway | Event-driven messaging | Platform adapters emit normalized events |

**Design Philosophy:** Flexibility over purity. The CLI and agent core are tightly integrated monoliths; the gateway uses a plugin architecture for multi-platform support.

---

## Module Map

### 1. Core Agent (`agent/`)

| File | Responsibility |
|------|-----------------|
| `anthropic_adapter.py` | Translates OpenAI-style format to Anthropic Messages API |
| `auxiliary_client.py` | Auxiliary model support (vision, web extraction) |
| `context_compressor.py` | Context compression for long conversations |
| `prompt_builder.py` | Prompt construction and management |
| `display.py` | Terminal display rendering (spinners, formatting) |
| `model_metadata.py` | Model capability detection and token estimation |
| `usage_pricing.py` | Cost tracking |
| `insights.py` | Usage analytics |
| `prompt_caching.py` | Anthropic prompt caching support |
| `trajectory.py` | Training trajectory management |
| `smart_model_routing.py` | Adaptive model selection |

**Entry Points:** `run_agent.py` (~387KB), `cli.py` (~329KB)

---

### 2. CLI (`hermes_cli/`)

| File | Responsibility |
|------|-----------------|
| `main.py` (~165KB) | Main CLI entry point using `fire.Fire()` |
| `commands.py` | Slash command registry (`COMMAND_REGISTRY`) |
| `config.py` (~76KB) | Configuration management |
| `setup.py` (~139KB) | Interactive setup wizard |
| `gateway.py` | Gateway management |
| `models.py` | Model provider handling |
| `browser_tool.py` | Playwright browser automation |
| `terminal_tool.py` | Terminal execution |
| `skills_hub.py` | Skills hub integration |

---

### 3. Gateway (`gateway/`)

| File | Responsibility |
|------|-----------------|
| `run.py` (~262KB) | Gateway runner with platform lifecycle management |
| `session.py` | Session management (context, persistence, reset policies) |
| `config.py` | Gateway and platform configuration |
| `delivery.py` | Message routing for cron job outputs |
| `channel_directory.py` | Channel/session directory |

**Platform Adapters (`gateway/platforms/`):**

| Platform | File Size | Notes |
|----------|-----------|-------|
| Discord | ~94KB | Primary platform |
| Telegram | ~78KB | Primary platform |
| Slack | - | Integration |
| WhatsApp | - | Integration |
| Email | - | Integration |
| Signal | - | Integration |
| SMS | - | Integration |
| Matrix | - | Integration |
| Mattermost | - | Integration |
| DingTalk | - | Integration |
| Home Assistant | - | Smart home |
| Webhook | - | Generic |
| API Server | - | HTTP API |

---

### 4. Tools (`tools/`)

| File | Purpose |
|------|---------|
| `web_tools.py` | Web search, extraction, crawling |
| `terminal_tool.py` | Command execution (local, Docker, Modal, SSH, Singularity, Daytona) |
| `file_tools.py` | File read/write/patch/search |
| `vision_tools.py` | Image analysis |
| `browser_tool.py` | Browser automation (Playwright) |
| `mixture_of_agents_tool.py` | Multi-model collaborative reasoning |
| `image_generation_tool.py` | Text-to-image generation |
| `skills_tool.py` | Skills listing/viewing |
| `tts_tool.py` | Text-to-speech |
| `code_execution_tool.py` | Sandboxed code execution |
| `delegate_tool.py` | Subagent spawning |
| `mcp_tool.py` | MCP server tools |
| `registry.py` | `ToolRegistry` singleton (central registration) |
| `interrupt.py` | Interrupt handling for long-running operations |

**Execution Backends (`tools/environments/`):** docker, modal, singularity, ssh, daytona

---

### 5. Skills System

| Location | Description |
|----------|-------------|
| `skills/` | 35 skill categories (apple/, autonomous-ai-agents/, creative/, data-science/, etc.) |
| `acp_registry/` | Additional registry (blockchain/, devops/, health/, mcp/, etc.) |

---

## Communication Patterns

### 1. Direct Import (Tight Coupling)

Python `import` statements used for closely related modules:

```python
# cli.py
from agent.usage_pricing import CanonicalUsage, estimate_usage_cost
from hermes_cli.banner import _format_context_length
from model_tools import get_tool_definitions, handle_function_call
```

**Used for:** CLI -> Agent, CLI -> Tools, Gateway -> Hermes modules

---

### 2. Registry Pattern (Loose Coupling)

Module-level `registry.register()` calls at import time:

```python
# tools/file_tools.py
registry.register(
    name="read_file",
    toolset="file",
    schema=READ_FILE_SCHEMA,
    handler=_handle_read_file,
    check_fn=_check_file_reqs,
    emoji="📖"
)
```

**Flow:**
1. `model_tools.py` imports all tool modules
2. Each module calls `registry.register()` at import time
3. `model_tools.py` queries registry for tool definitions and dispatches calls

---

### 3. Async/Await with Event Loop Bridge

`asyncio` for I/O operations with sync bridging:

```python
# model_tools.py
def _run_async(coro):
    """Run an async coroutine from a sync context."""
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            return pool.submit(asyncio.run, coro).result(timeout=300)
    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

**Used for:** Tool execution, Gateway platform adapters, Concurrent operations

---

### 4. Message Passing (Event-Driven)

`MessageEvent` dataclass flows through the gateway:

```python
# gateway/platforms/base.py
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
    raw_message: Any = None
    media_urls: List[str] = field(default_factory=list)
    timestamp: datetime = field(default_factory=datetime.now)
```

---

### 5. Session Context Propagation

`SessionSource` / `SessionContext` carries request context:

```python
# gateway/session.py
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    chat_name: Optional[str] = None
    chat_type: str = "dm"
    user_id: Optional[str] = None
    thread_id: Optional[str] = None
```

---

### 6. Configuration-Driven

`config.yaml` + `.env` files for all component configuration.

---

## Key Interfaces

### ToolRegistry (`tools/registry.py`)

```python
class ToolRegistry:
    def register(self, name, toolset, schema, handler, check_fn,
                 requires_env, is_async, description, emoji)

    def get_definitions(self, tool_names, quiet) -> List[dict]
    """Return OpenAI-format tool schemas."""

    def dispatch(self, name, args, **kwargs) -> str
    """Execute a tool handler by name."""

    def get_all_tool_names(self) -> List[str]
    def get_toolset_for_tool(self, name) -> Optional[str]
    def is_toolset_available(self, toolset) -> bool
```

---

### BasePlatformAdapter (`gateway/platforms/base.py`)

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool
    @abstractmethod
    async def disconnect(self) -> None
    @abstractmethod
    async def send(self, chat_id, content, reply_to, metadata) -> SendResult
    @abstractmethod
    async def get_chat_info(self, chat_id) -> Dict[str, Any]

    def set_message_handler(self, handler: MessageHandler)
    async def handle_message(self, event: MessageEvent)
    async def send_image(self, chat_id, image_url, caption)
    async def send_voice(self, chat_id, audio_path, caption)
```

---

### SessionStore (`gateway/session.py`)

```python
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    chat_name: Optional[str]
    chat_type: str
    user_id: Optional[str]
    thread_id: Optional[str]

class SessionStore:
    """Persistent conversation storage."""
```

---

## System Interaction Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                            CLI Layer                                 │
│  hermes_cli/main.py  ←→  hermes_cli/commands.py                     │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Agent Core                                   │
│  run_agent.py  ←→  agent/* (anthropic_adapter, context_compressor)   │
│         │                        │                                   │
│         ▼                        ▼                                   │
│  model_tools.py  ←→  tools/registry.py  ←→  tools/*.py              │
│         │                                                          │
│         ▼                                                          │
│  toolsets.py                                                      │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │  skills/│   │  acp_    │   │ environments/│
        │          │   │ registry/│   │              │
        └──────────┘   └──────────┘   └──────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        Gateway Layer                                 │
│  gateway/run.py  ←→  gateway/session.py  ←→  gateway/delivery.py    │
│         │                                                          │
│         ▼                                                          │
│  gateway/platforms/base.py  ←→  discord.py, telegram.py, etc.      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architectural Decisions Summary

| Decision | Rationale |
|----------|-----------|
| Modular monolith for core | Simplicity for CLI-first tool; complexity at edges (gateway, tools) |
| Registry pattern for tools | Zero-config tool discovery; add new tools by importing |
| Adapter pattern for LLMs | Provider-specific logic isolated; swap LLMs via new adapter |
| Template Method for platforms | Common protocol in base class; platform-specific overrides |
| Event loop bridge for sync/async | CLI is sync; tools/gateway are async; bridge handles cleanly |
| Session context propagation | Track message origin across 12+ platforms without global state |
