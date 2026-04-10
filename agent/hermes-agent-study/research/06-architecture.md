# Architecture & Design Analysis: hermes-agent

## Architectural Pattern

**hermes-agent** is a **modular monolith with plugin-based extensions**, combining:

- **Layered architecture** for the core (CLI -> Agent -> Tools)
- **Plugin/adapter pattern** for platform integrations (gateway)
- **Registry pattern** for tool discovery and composition
- **Event-driven messaging** for gateway platform adapters

The architecture prioritizes flexibility over purity: the CLI and agent core are tightly integrated monoliths, while the gateway uses a plugin architecture to support 12+ messaging platforms.

---

## Major Modules and Responsibilities

### 1. Core Agent (`agent/`)

**Files:**
- `anthropic_adapter.py` - Anthropic API integration (translates between OpenAI-style format and Anthropic Messages API)
- `auxiliary_client.py` - Auxiliary model support (vision, web extraction)
- `context_compressor.py` - Context compression for long conversations
- `prompt_builder.py` - Prompt construction and management
- `display.py` - Terminal display rendering with spinners and formatting
- `model_metadata.py` - Model capability detection and token estimation
- `usage_pricing.py` - Cost tracking
- `insights.py` - Usage analytics
- `prompt_caching.py` - Anthropic prompt caching support
- `trajectory.py` - Training trajectory management
- `context_references.py` - Context reference handling
- `skill_commands.py` - Skill command definitions
- `title_generator.py` - Session title generation
- `smart_model_routing.py` - Adaptive model selection

**Responsibility:** Core AI logic - message handling, tool orchestration, context management, API adapters.

---

### 2. CLI (`hermes_cli/`)

**Files:**
- `main.py` (~165KB) - Main CLI entry point using `fire.Fire()`
- `commands.py` - Slash command registry (`COMMAND_REGISTRY`)
- `config.py` (~76KB) - Configuration management
- `setup.py` (~139KB) - Interactive setup wizard
- `gateway.py` - Gateway management
- `models.py` - Model provider handling
- `skills_hub.py` - Skills hub integration
- `browser_tool.py` - Playwright browser automation
- `terminal_tool.py` - Terminal execution
- `file_operations.py` - File operations
- `voice_mode.py` - Voice/TTS tools
- `delegate_tool.py` - Subagent delegation
- `skills_guard.py` - Skills permission guard
- `mcp_tool.py` - MCP server integration
- `rl_training_tool.py` - RL training tool
- `env_loader.py` - Environment variable loading

**Responsibility:** User-facing command-line interface, configuration, command dispatch.

---

### 3. Gateway (`gateway/`)

**Files:**
- `run.py` (~262KB) - Gateway runner with platform lifecycle management
- `session.py` - Session management (context tracking, persistence, reset policies)
- `config.py` - Gateway and platform configuration
- `delivery.py` - Message routing for cron job outputs
- `channel_directory.py` - Channel/session directory
- `platforms/` - Platform adapters

**Platform Adapters (`gateway/platforms/`):**
- `base.py` - `BasePlatformAdapter` abstract class
- `discord.py` (~94KB) - Discord bot
- `telegram.py` (~78KB) - Telegram bot
- `slack.py` - Slack integration
- `whatsapp.py` - WhatsApp integration
- `email.py` - Email integration
- `signal.py` - Signal messaging
- `sms.py` - SMS integration
- `matrix.py` - Matrix protocol
- `mattermost.py` - Mattermost integration
- `dingtalk.py` - DingTalk
- `homeassistant.py` - Home Assistant integration
- `webhook.py` - Generic webhook
- `api_server.py` - HTTP API server

**Responsibility:** Multi-platform messaging integration, session persistence, message normalization.

---

### 4. Tools (`tools/`)

**Tool Modules:**
- `web_tools.py` - Web search, extraction, crawling
- `terminal_tool.py` - Command execution (local, Docker, Modal, SSH, Singularity, Daytona)
- `file_tools.py` - File read/write/patch/search
- `vision_tools.py` - Image analysis
- `browser_tool.py` - Browser automation (Playwright)
- `mixture_of_agents_tool.py` - Multi-model collaborative reasoning
- `image_generation_tool.py` - Text-to-image generation
- `skills_tool.py` - Skills listing/viewing
- `skill_manager_tool.py` - Skill management
- `tts_tool.py` - Text-to-speech
- `todo_tool.py` - Task management
- `clarify_tool.py` - Interactive Q&A
- `code_execution_tool.py` - Sandboxed code execution
- `delegate_tool.py` - Subagent spawning
- `cronjob_tools.py` - Cron job management
- `rl_training_tool.py` - RL training
- `homeassistant_tool.py` - Smart home control
- `mcp_tool.py` - MCP server tools
- `session_search_tool.py` - Session history search
- `send_message_tool.py` - Cross-platform messaging

**Core Infrastructure:**
- `registry.py` - `ToolRegistry` singleton (central tool registration)
- `interrupt.py` - Interrupt handling for long-running operations
- `environments/` - Execution backend abstractions (docker, modal, singularity, ssh, daytona)

**Responsibility:** Capability providers for the agent. Each tool is a self-contained module that registers itself via `registry.register()`.

---

### 5. Tool Orchestration (`model_tools.py`)

**Files:**
- `model_tools.py` - Thin orchestration layer over the tool registry
- `toolsets.py` - Tool grouping and composition

**Responsibility:** Tool discovery (by importing all tool modules), tool definition export, tool dispatch, toolset resolution.

---

### 6. Skills System (`skills/`, `acp_registry/`)

**Structure:**
- `skills/` - 35 skill categories (apple/, autonomous-ai-agents/, creative/, data-science/, diagramming/, dogfood/, domain/, email/, feeds/, gaming/, gifs/, github/, mlops/, music-creation/, note-taking/, productivity/, red-teaming/, research/, smart-home/, social-media/, software-development/)
- `acp_registry/` - Additional skill registry (blockchain/, devops/, health/, mcp/, migration/, productivity/, research/, security/)

**Responsibility:** Procedural memory, skill definitions, domain-specific capabilities.

---

### 7. Other Components

| Module | Purpose |
|--------|---------|
| `acp_adapter/` | Adapter protocol for editor integration |
| `cron/` | Cron scheduling for background jobs |
| `environments/` | Agent training/testing environments |
| `honcho_integration/` | Honcho AI memory integration |
| `tests/` | Comprehensive test suite |

---

## Component Communication Patterns

### 1. Direct Import (Tight Coupling)

**Pattern:** Python `import` statements

**Used for:**
- `cli.py` imports from `agent.*`, `hermes_cli.*`, `model_tools`, `toolsets`
- `run_agent.py` imports `agent.*` modules, `model_tools`, `hermes_cli.env_loader`
- `gateway/run.py` imports `gateway.*`, `hermes_cli.config`, `hermes_constants`

**Example:**
```python
# cli.py
from agent.usage_pricing import CanonicalUsage, estimate_usage_cost
from hermes_cli.banner import _format_context_length
from model_tools import get_tool_definitions, handle_function_call
```

### 2. Registry Pattern (Loose Coupling)

**Pattern:** Module-level `registry.register()` calls at import time

**Used for:**
- Tool registration (`tools/registry.py`)
- Command registration (`hermes_cli/commands.py`)

**Example:**
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
3. `model_tools.py` queries the registry for tool definitions and dispatches calls

### 3. Async/Await with Event Loop Bridge

**Pattern:** `asyncio` for I/O operations with sync bridging

**Used for:**
- Tool execution (especially network tools)
- Gateway platform adapters
- Concurrent tool execution

**Example:**
```python
# model_tools.py
def _run_async(coro):
    """Run an async coroutine from a sync context."""
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        # Inside async context — run in fresh thread
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            return pool.submit(asyncio.run, coro).result(timeout=300)
    # CLI path — use persistent event loop
    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

### 4. Message Passing (Event-Driven)

**Pattern:** `MessageEvent` dataclass flows through the gateway

**Used for:**
- Gateway platform adapters -> agent
- Session context propagation

**Example:**
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

### 5. Session Context

**Pattern:** `SessionContext` / `SessionSource` propagates through components

**Used for:**
- Tracking message origin across platforms
- Routing responses back to the correct platform
- Context injection into system prompts

**Example:**
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

### 6. Configuration-Driven

**Pattern:** `config.yaml` + `.env` files

**Used for:**
- All component configuration
- Environment variable bridging

---

## Key Abstractions and Interfaces

### Tool Registry (`tools/registry.py`)

```python
class ToolRegistry:
    def register(self, name, toolset, schema, handler, check_fn,
                 requires_env, is_async, description, emoji):
        """Register a tool at module import time."""

    def get_definitions(self, tool_names, quiet) -> List[dict]:
        """Return OpenAI-format tool schemas."""

    def dispatch(self, name, args, **kwargs) -> str:
        """Execute a tool handler by name."""

    def get_all_tool_names(self) -> List[str]
    def get_toolset_for_tool(self, name) -> Optional[str]
    def is_toolset_available(self, toolset) -> bool
```

### Base Platform Adapter (`gateway/platforms/base.py`)

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

### Session Management (`gateway/session.py`)

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

### Delivery Routing (`gateway/delivery.py`)

```python
@dataclass
class DeliveryTarget:
    platform: Platform
    chat_id: Optional[str]
    thread_id: Optional[str]
    is_origin: bool
    is_explicit: bool

class DeliveryRouter:
    """Routes cron job outputs to appropriate channels."""
```

---

## Design Patterns in Use

### 1. Registry/Plugin Pattern

**Location:** `tools/registry.py`, `hermes_cli/commands.py`

**Implementation:** Module-level registration at import time, centralized registry queried at runtime.

**Benefit:** New tools can be added by creating a module and calling `register()` - no code changes needed elsewhere.

### 2. Adapter Pattern

**Location:** `agent/anthropic_adapter.py`

**Implementation:** Translates between Hermes's internal OpenAI-style message format and Anthropic's Messages API.

**Benefit:** Provider-specific logic is isolated; swapping LLMs only requires a new adapter.

### 3. Factory/Composition Pattern

**Location:** `toolsets.py`

**Implementation:** Toolsets compose from individual tools or other toolsets:

```python
TOOLSETS = {
    "research": {
        "tools": ["web_search", "web_extract"],
        "includes": ["file", "terminal"]
    }
}
```

**Benefit:** Reusable tool combinations for different scenarios.

### 4. Strategy Pattern

**Location:** `tools/terminal_tool.py`, `tools/environments/`

**Implementation:** Multiple execution backends (local, docker, modal, ssh, singularity, daytona) selectable via `TERMINAL_ENV`.

**Benefit:** Runtime selection of execution strategy without code changes.

### 5. Template Method Pattern

**Location:** `gateway/platforms/base.py`

**Implementation:** `BasePlatformAdapter` provides default implementations for `send_image`, `send_voice`, etc., with platform-specific overrides.

**Benefit:** Common logic lives in base class; platforms only override what's different.

### 6. Observer/Handler Pattern

**Location:** `gateway/platforms/base.py`

**Implementation:** `set_message_handler()` accepts a `MessageHandler` callable.

**Benefit:** Decoupled message processing; gateway doesn't know handler details.

### 7. Command Pattern

**Location:** `hermes_cli/commands.py`

**Implementation:** `CommandDef` dataclass registry for all slash commands.

**Benefit:** Single source of truth for commands across CLI, gateway, autocomplete.

### 8. Context Object Pattern

**Location:** `gateway/session.py`

**Implementation:** `SessionSource` and `SessionContext` carry request context through the system.

**Benefit:** Clean propagation of source information without global state.

### 9. Interrupt/Polling Pattern

**Location:** `tools/interrupt.py`

**Implementation:** Global interrupt event checked by long-running operations.

**Benefit:** Graceful interruption of terminal commands, batch operations.

---

## CLI, Agent Core, Gateway, and Skills Interaction

```
┌─────────────────────────────────────────────────────────────────────┐
│                            CLI Layer                                │
│  hermes_cli/main.py  ←→  hermes_cli/commands.py                    │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Agent Core                                  │
│  run_agent.py  ←→  agent/* (anthropic_adapter, context_compressor)  │
│         │                        │                                  │
│         ▼                        ▼                                  │
│  model_tools.py  ←→  tools/registry.py  ←→  tools/*.py            │
│         │                                                          │
│         ▼                                                          │
│  toolsets.py                                                       │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │  skills/ │   │  acp_    │   │ environments/│
        │          │   │ registry/│   │              │
        └──────────┘   └──────────┘   └──────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        Gateway Layer                                 │
│  gateway/run.py  ←→  gateway/session.py  ←→  gateway/delivery.py   │
│         │                                                          │
│         ▼                                                          │
│  gateway/platforms/base.py  ←→  discord.py, telegram.py, etc.      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  Messaging Platforms │
                    │  (Discord, Telegram) │
                    └─────────────────────┘
```

### Interaction Flow

1. **CLI Start:**
   - `cli.py` (or `hermes`) is the entry point
   - Loads configuration, initializes tools
   - Creates `AIAgent` instance from `run_agent.py`

2. **Agent Execution:**
   - `AIAgent.run_conversation()` manages the chat loop
   - Calls `model_tools.get_tool_definitions()` to get available tools
   - Dispatches tool calls via `model_tools.handle_function_call()`
   - Agent core handles API communication with LLM providers

3. **Tool Execution:**
   - Tools register themselves via `registry.register()`
   - `model_tools` imports all tool modules to trigger registration
   - Dispatch routes to appropriate tool handler

4. **Gateway Operation:**
   - Runs as separate service (or in-process)
   - Each platform adapter (`discord.py`, `telegram.py`) connects to its service
   - Incoming messages normalized to `MessageEvent`
   - Message handler invokes agent (via ACP or direct)
   - Responses routed back via `DeliveryRouter`

5. **Skills Integration:**
   - Skills are external packages that extend capabilities
   - Loaded via `skills_hub.py` or `skill_manager_tool.py`
   - Provide domain-specific prompts and tools

---

## File Reference Summary

| Pattern | File(s) | Purpose |
|---------|---------|---------|
| Registry | `tools/registry.py` | Central tool registration |
| Adapter | `agent/anthropic_adapter.py` | LLM provider translation |
| Factory | `toolsets.py` | Tool composition |
| Strategy | `tools/terminal_tool.py` | Multiple execution backends |
| Template Method | `gateway/platforms/base.py` | Platform adapter base class |
| Command | `hermes_cli/commands.py` | Slash command registry |
| Context | `gateway/session.py` | Session tracking |
| Event | `gateway/platforms/base.py` | Message normalization |
| Singleton | `tools/registry.py` | `registry = ToolRegistry()` |
| Async Bridge | `model_tools.py` | `_run_async()` for sync→async |
