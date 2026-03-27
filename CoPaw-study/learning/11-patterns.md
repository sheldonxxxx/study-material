# CoPaw Design Patterns

## Pattern Catalog

| # | Pattern | Layer | Purpose |
|---|---------|-------|---------|
| 1 | Registry | Channels | Plugin discovery without hard imports |
| 2 | Strategy | Channels | Per-channel session resolution |
| 3 | Factory | Providers/Agents | Decoupled model instantiation |
| 4 | Mixin | Agents | Cross-cutting security concerns |
| 5 | Observer | Runner | Async response streaming |
| 6 | Template Method | Channels | Hookable message processing |
| 7 | Hook | Agents | Pluggable pre/post processing |
| 8 | Debounce Queue | Channels | Message coalescing |

---

## 1. Registry Pattern

**Purpose**: Discover and load channel plugins at runtime without hard-coded imports.

**File**: `src/copaw/app/channels/registry.py`

**Evidence**:
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

**Textbook Difference**: Standard registry pattern uses `dict` lookup. CoPaw extends with runtime discovery via importlib for custom channels in `custom_channels/` directory.

---

## 2. Strategy Pattern

**Purpose**: Each channel implements its own session resolution strategy.

**File**: `src/copaw/app/channels/base.py`

**Evidence**:
```python
class BaseChannel(ABC):
    channel: ChannelType

    def resolve_session_id(self, sender_id: str, channel_meta: Optional[Dict] = None) -> str:
        """Map sender and optional channel meta to session_id.
        Override in subclasses for channel-specific session keys."""
        return f"{self.channel}:{sender_id}"
```

**Textbook Difference**: Standard Strategy uses interface with multiple implementations. CoPaw uses inheritance with overridable method - simpler but less explicit polymorphism.

---

## 3. Factory Pattern

**Purpose**: Decouple model type from instantiation logic, support multiple LLM providers.

**File**: `src/copaw/agents/model_factory.py`

**Evidence**:
```python
_CHAT_MODEL_FORMATTER_MAP: dict[Type[ChatModelBase], Type[FormatterBase]] = {
    OpenAIChatModel: OpenAIChatFormatter,
    AnthropicChatModel: AnthropicChatFormatter,
    GeminiChatModel: GeminiChatFormatter,
}

def create_model_and_formatter(agent_id: str = "default") -> Tuple[ChatModelBase, FormatterBase]:
    """Create model and formatter based on agent configuration."""
    # ... dynamic lookup and instantiation
```

**Textbook Difference**: Classic Factory Method pattern. CoPaw uses dictionary mapping + factory function, avoiding subclass-per-product overhead.

---

## 4. Mixin Pattern

**Purpose**: Reusable security logic (ToolGuard) applied to any agent via MRO.

**File**: `src/copaw/agents/tool_guard_mixin.py`

**Evidence**:
```python
class ToolGuardMixin:
    """Security mixin that intercepts tool calls for approval."""

    async def _acting(self, *args, **kwargs) -> Msg:
        # Intercepts before tool execution
        # Checks tool_guard approval

class CoPawAgent(ToolGuardMixin, ReActAgent):
    """MRO: CoPawAgent -> ToolGuardMixin -> ReActAgent"""
```

**Usage** (`src/copaw/agents/react_agent.py:67`):
```python
class CoPawAgent(ToolGuardMixin, ReActAgent):
    """CoPaw Agent with integrated tools, skills, and memory management."""
```

**Textbook Difference**: Python mixins differ from traditional mixin (interface + delegation). CoPaw uses MRO method resolution to intercept `_acting()` call chain without modifying base class.

---

## 5. Observer Pattern

**Purpose**: Async iteration over streaming agent responses.

**File**: `src/copaw/app/runner/runner.py`

**Evidence**:
```python
async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),
):
    yield msg, last
```

**Textbook Difference**: Classic Observer has Subject + Observers with explicit `attach/detach`. CoPaw uses Python async generator - more Pythonic but less explicit coupling.

---

## 6. Template Method Pattern

**Purpose**: Fixed message processing algorithm with customizable steps.

**File**: `src/copaw/app/channels/base.py`

**Evidence**:
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

**Textbook Difference**: Matches GoF Template Method closely. `consume_one()` is the template; `_consume_one_request()` is the primitive operation overridden by subclasses.

---

## 7. Hook Pattern

**Purpose**: Pluggable pre/post processing steps for agent lifecycle.

**Files**: `src/copaw/agents/hooks/*.py`

**Evidence**:
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

**Textbook Difference**: Resembles Strategy but with lifecycle integration. Hooks are registered by type (pre_reasoning, post_reasoning) and called at specific points - more declarative than explicit pipeline composition.

---

## 8. Debounce Queue Pattern

**Purpose**: Coalesce rapid messages from same session before processing.

**File**: `src/copaw/app/channels/manager.py`

**Evidence**:
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

**Textbook Difference**: Not a GoF pattern. Combines Queue with debouncing logic. Key-based batching ensures messages from same conversation are processed together.

---

## Pattern Interactions

### Agent Initialization Flow

```
CoPawAgent.__init__()
    |
    +-- ToolGuardMixin (MRO intercepts _acting)
    |
    +-- ReActAgent base
    |       |
    |       +-- model_factory.create_model_and_formatter() [Factory]
    |       |
    |       +-- register_instance_hook() [Hook Pattern]
    |
    +-- SkillsManager.load_skill()
    |       |
    |       +-- Skill Scanner (security)
    |
    +-- Memory initialization
```

### Channel Message Processing

```
Channel.consume_one() [Template Method]
    |
    +-- debounce check
    |
    +-- _consume_one_request() [override point]
            |
            +-- _payload_to_request()
            |
            +-- _run_process_loop()
                    |
                    +-- resolve_session_id() [Strategy]
                    |
                    +-- AgentRunner.query_handler()
                            |
                            +-- stream_printing_messages() [Observer]
```

---

## Pattern Summary by Layer

| Layer | Patterns |
|-------|----------|
| CLI | (Command dispatch via Click) |
| Web/Routers | Factory (model creation) |
| Agent | Mixin, Hook, Factory |
| Channels | Registry, Strategy, Template Method, Debounce Queue |
| Runner | Observer |
| Providers | Factory |
