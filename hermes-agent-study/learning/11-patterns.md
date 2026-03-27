# Design Patterns: hermes-agent

**Project:** Self-improving AI agent by Nous Research
**Source:** `/Users/sheldon/Documents/claw/hermes-agent-study/research/06-architecture.md`
**Date:** 2026-03-27

---

## Pattern Index

| # | Pattern | Location | Type |
|---|---------|----------|------|
| 1 | Registry/Plugin | `tools/registry.py`, `hermes_cli/commands.py` | Creational |
| 2 | Adapter | `agent/anthropic_adapter.py` | Structural |
| 3 | Factory/Composition | `toolsets.py` | Creational |
| 4 | Strategy | `tools/terminal_tool.py`, `tools/environments/` | Behavioral |
| 5 | Template Method | `gateway/platforms/base.py` | Behavioral |
| 6 | Observer/Handler | `gateway/platforms/base.py` | Behavioral |
| 7 | Command | `hermes_cli/commands.py` | Behavioral |
| 8 | Context Object | `gateway/session.py` | Structural |
| 9 | Interrupt/Polling | `tools/interrupt.py` | Behavioral |
| 10 | Singleton | `tools/registry.py` | Creational |

---

## Pattern 1: Registry/Plugin

**Purpose:** Self-registering tool discovery without central configuration

**Evidence:**

```python
# tools/registry.py
class ToolRegistry:
    def register(self, name, toolset, schema, handler, check_fn,
                 requires_env, is_async, description, emoji):
        """Register a tool at module import time."""

    def get_definitions(self, tool_names, quiet) -> List[dict]:
        """Return OpenAI-format tool schemas."""

    def dispatch(self, name, args, **kwargs) -> str:
        """Execute a tool handler by name."""
```

**Registration Call (in each tool module):**

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

**Discovery Flow:**

```python
# model_tools.py
# Imports trigger registration at module load time
from tools import file_tools, web_tools, terminal_tool
# ... other tool imports

# Later: query registry for all registered tools
def get_tool_definitions():
    return registry.get_definitions(...)
```

**Benefit:** Add a new tool by creating a module and calling `register()` - no changes elsewhere.

---

## Pattern 2: Adapter

**Purpose:** Translate between internal OpenAI-style format and Anthropic Messages API

**Evidence:**

```python
# agent/anthropic_adapter.py

class AnthropicAdapter:
    def __init__(self, api_key: str, model: str = "claude-3-5-sonnet-20241022"):
        self.api_key = api_key
        self.model = model

    def convert_to_anthropic_format(self, messages: List[Dict]) -> Dict:
        """Convert OpenAI-style messages to Anthropic Messages API format."""
        # Handles role mapping, content block construction, system prompt extraction
```

**Message Format Translation:**

```python
# Internal OpenAI-style format:
{"role": "user", "content": "Hello"}
{"role": "assistant", "content": "Hi there"}

# Anthropic Messages API format:
{
    "role": "user",
    "content": [{"type": "text", "text": "Hello"}]
}
```

**Benefit:** Provider-specific logic isolated; swapping LLMs only requires a new adapter.

---

## Pattern 3: Factory/Composition

**Purpose:** Compose toolsets from individual tools or nested toolsets

**Evidence:**

```python
# toolsets.py
TOOLSETS = {
    "research": {
        "tools": ["web_search", "web_extract"],
        "includes": ["file", "terminal"]
    },
    "coding": {
        "tools": ["read_file", "write_file", "edit_file", "terminal"],
        "includes": []
    },
    "default": {
        "tools": ["web_search", "file_operations"],
        "includes": ["research"]
    }
}
```

**Resolution Logic:**

```python
def resolve_toolset(toolset_name: str) -> List[str]:
    """Recursively resolve tools and included toolsets."""
    resolved = []
    for item in TOOLSETS[toolset_name]["tools"]:
        resolved.append(item)
    for included in TOOLSETS[toolset_name].get("includes", []):
        resolved.extend(resolve_toolset(included))
    return resolved
```

**Benefit:** Reusable tool combinations for different scenarios (research, coding, default).

---

## Pattern 4: Strategy

**Purpose:** Select execution backend at runtime without code changes

**Evidence:**

```python
# tools/terminal_tool.py
TERMINAL_ENV = os.environ.get("TERMINAL_ENV", "local")

# Execution backend selection
if TERMINAL_ENV == "docker":
    from tools.environments.docker import DockerBackend
    backend = DockerBackend()
elif TERMINAL_ENV == "modal":
    from tools.environments.modal import ModalBackend
    backend = ModalBackend()
elif TERMINAL_ENV == "ssh":
    from tools.environments.ssh import SSHBackend
    backend = SSHBackend()
# ... etc
```

**Backend Interface:**

```python
# tools/environments/base.py
class ExecutionBackend(ABC):
    @abstractmethod
    async def execute(self, command: str, cwd: str = None) -> ExecutionResult:
        pass

    @abstractmethod
    async def upload_file(self, local_path: str, remote_path: str):
        pass

    @abstractmethod
    async def download_file(self, remote_path: str, local_path: str):
        pass
```

**Available Backends:** local, docker, modal, ssh, singularity, daytona

**Benefit:** Runtime selection of execution strategy; same tool interface across all backends.

---

## Pattern 5: Template Method

**Purpose:** Common protocol in base class; platform-specific behavior via overrides

**Evidence:**

```python
# gateway/platforms/base.py
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool:
        """Platform-specific connection logic."""
        pass

    @abstractmethod
    async def disconnect(self) -> None:
        pass

    @abstractmethod
    async def send(self, chat_id, content, reply_to, metadata) -> SendResult:
        """Platform-specific message sending."""
        pass

    # Default implementations for common operations
    async def send_image(self, chat_id, image_url, caption=None):
        """Default: download and send as file attachment."""
        # Common implementation
        pass

    async def send_voice(self, chat_id, audio_path, caption=None):
        """Default: send audio file."""
        # Common implementation
        pass
```

**Platform Override Example:**

```python
# gateway/platforms/discord.py
class DiscordAdapter(BasePlatformAdapter):
    async def send(self, chat_id, content, reply_to, metadata):
        # Discord-specific: use Discord channel IDs, embeds, etc.
        await self.discord_client.send_message(
            channel=chat_id,
            content=content,
            reply_to=reply_to
        )
```

**Benefit:** Common logic (image sending, voice sending) lives in base; platforms only override what's different.

---

## Pattern 6: Observer/Handler

**Purpose:** Decoupled message processing; gateway doesn't know handler details

**Evidence:**

```python
# gateway/platforms/base.py
class BasePlatformAdapter(ABC):
    def __init__(self):
        self._message_handler: Optional[MessageHandler] = None

    def set_message_handler(self, handler: MessageHandler):
        """Accept a handler for incoming messages."""
        self._message_handler = handler

    async def handle_message(self, event: MessageEvent):
        """Process incoming message and invoke handler."""
        if self._message_handler:
            await self._message_handler(event)
```

**Handler Registration:**

```python
# gateway/run.py
async def setup_platforms():
    for platform in config.enabled_platforms:
        adapter = create_platform_adapter(platform)
        adapter.set_message_handler(handle_incoming_message)
        await adapter.connect()
```

**Benefit:** Platform adapters are oblivious to message processing logic; handler can be swapped.

---

## Pattern 7: Command

**Purpose:** Single source of truth for slash commands across CLI, gateway, autocomplete

**Evidence:**

```python
# hermes_cli/commands.py
@dataclass
class CommandDef:
    name: str
    description: str
    handler: Callable
    parameters: List[ParameterDef]
    requires_auth: bool = False
    modes: List[str] = field(default_factory=lambda: ["cli", "gateway"])
    autocomplete: Optional[List[str]] = None

COMMAND_REGISTRY: Dict[str, CommandDef] = {}

def register_command(cmd: CommandDef):
    COMMAND_REGISTRY[cmd.name] = cmd
```

**Registration:**

```python
# hermes_cli/commands.py
register_command(CommandDef(
    name="model",
    description="Switch the active model provider",
    handler=handle_model_command,
    parameters=[
        ParameterDef(name="provider", type="string", required=True),
        ParameterDef(name="model", type="string", required=False)
    ]
))
```

**Usage in CLI:**

```python
# hermes_cli/main.py
async def handle_command(input_str: str):
    parts = input_str.strip().split()
    cmd_name = parts[0][1:]  # Remove leading '/'
    cmd = COMMAND_REGISTRY.get(cmd_name)
    if cmd:
        await cmd.handler(parts[1:])
```

**Benefit:** Commands defined once, used in CLI parsing, gateway routing, and autocomplete.

---

## Pattern 8: Context Object

**Purpose:** Clean propagation of source information without global state

**Evidence:**

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

@dataclass
class SessionContext:
    source: SessionSource
    conversation_id: str
    message_count: int = 0
    last_activity: datetime = field(default_factory=datetime.now)
```

**Propagation:**

```python
# gateway/run.py
async def handle_incoming_message(event: MessageEvent):
    context = SessionContext(
        source=event.source,
        conversation_id=event.conversation_id
    )
    # Pass context through to agent
    response = await agent.process_message(event.text, context=context)
    # Route response back to source platform
    await delivery_router.send(response, target=context.source)
```

**Benefit:** Request context travels with the message; no global mutable state.

---

## Pattern 9: Interrupt/Polling

**Purpose:** Graceful interruption of long-running operations

**Evidence:**

```python
# tools/interrupt.py
import threading

_interrupt_event = threading.Event()

def set_interrupt():
    """Request interruption of running operations."""
    _interrupt_event.set()

def is_interrupted() -> bool:
    """Check if interruption was requested."""
    return _interrupt_event.is_set()

def wait_for_interrupt(timeout: float = None) -> bool:
    """Block until interrupt is set or timeout expires."""
    return _interrupt_event.wait(timeout)
```

**Usage in Terminal Tool:**

```python
# tools/terminal_tool.py
async def execute_command(command: str):
    process = await asyncio.create_subprocess_shell(command)

    while process.returncode is None:
        if is_interrupted():
            process.terminate()
            return {"status": "interrupted", "output": "..."}
        await asyncio.sleep(0.1)
        process.poll()

    return {"status": "completed", "returncode": process.returncode}
```

**Benefit:** User can cancel long-running terminal commands, batch operations.

---

## Pattern 10: Singleton

**Purpose:** Global registry instance accessed throughout the application

**Evidence:**

```python
# tools/registry.py
class ToolRegistry:
    _instance: Optional['ToolRegistry'] = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self):
        if self._initialized:
            return
        self._tools: Dict[str, ToolDef] = {}
        self._initialized = True

# Global singleton instance
registry = ToolRegistry()
```

**Access Throughout Codebase:**

```python
# Any tool module
from tools.registry import registry
registry.register(name="...", ...)

# model_tools.py
from tools.registry import registry
tools = registry.get_definitions(...)
```

**Benefit:** Single source of truth for tool registration; no need to pass registry instance.

---

## Pattern Relationships

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Application Start                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Singleton Pattern: tools/registry.py creates global registry        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Registry/Plugin Pattern: Tool modules import → register() calls     │
│  Command Pattern: hermes_cli/commands.py populates COMMAND_REGISTRY│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Factory Pattern: toolsets.py resolves tool combinations            │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  Adapter Pattern │ │  Strategy        │ │  Template Method │
│  (LLM providers) │ │  (Execution      │ │  (Platform      │
│                  │ │   backends)      │ │   adapters)      │
└──────────────────┘ └──────────────────┘ └──────────────────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Observer/Handler Pattern: Platform adapters → message handler      │
│  Context Object Pattern: SessionSource propagates through system    │
│  Interrupt/Polling Pattern: Long-running ops check interrupt flag  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

| Pattern | Key Files | Intent |
|---------|-----------|--------|
| Registry/Plugin | `tools/registry.py` | Self-registering tool discovery |
| Adapter | `agent/anthropic_adapter.py` | Format translation between LLM providers |
| Factory/Composition | `toolsets.py` | Tool grouping and reuse |
| Strategy | `tools/terminal_tool.py` | Runtime backend selection |
| Template Method | `gateway/platforms/base.py` | Common protocol, specific implementations |
| Observer/Handler | `gateway/platforms/base.py` | Decoupled message processing |
| Command | `hermes_cli/commands.py` | Unified slash command definition |
| Context Object | `gateway/session.py` | Request context propagation |
| Interrupt/Polling | `tools/interrupt.py` | Graceful operation cancellation |
| Singleton | `tools/registry.py` | Global single instance |
