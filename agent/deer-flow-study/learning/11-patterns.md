# deer-flow Design Patterns

## Pattern Catalog

| # | Pattern | Location | Type |
|---|---------|----------|------|
| 1 | Pub/Sub | `app/channels/message_bus.py` | Messaging |
| 2 | Abstract Base Class | `app/channels/base.py` | Creational |
| 3 | Manager/Dispatcher | `app/channels/manager.py` | Behavioral |
| 4 | Middleware Chain | `packages/harness/deerflow/agents/lead_agent/agent.py` | Behavioral |
| 5 | Factory | `packages/harness/deerflow/models/factory.py` | Creational |
| 6 | Provider | `packages/harness/deerflow/sandbox/sandbox_provider.py` | Creational |
| 7 | Singleton | `frontend/src/core/api/api-client.ts` | Creational |
| 8 | Hook Pattern | `frontend/src/core/threads/hooks.ts` | Reactive |
| 9 | Async Queue | `app/channels/message_bus.py` | Messaging |
| 10 | Strategy | `app/channels/manager.py` | Behavioral |
| 11 | State Reducer | `packages/harness/deerflow/agents/thread_state.py` | State |
| 12 | Lazy Initialization | Multiple files | Performance |
| 13 | Boundary Enforcement | `tests/test_harness_boundary.py` | Architecture |

---

## 1. Pub/Sub (MessageBus)

**File**: `app/channels/message_bus.py`

Decouples inbound message sources from the dispatcher. Channels publish without knowing who consumes; dispatcher subscribes without knowing message sources.

### Evidence

```python
class MessageBus:
    """Async pub/sub hub connecting channels and the agent dispatcher."""

    def __init__(self) -> None:
        self._inbound_queue: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self._outbound_listeners: list[OutboundCallback] = []

    async def publish_inbound(self, msg: InboundMessage) -> None:
        """Enqueue an inbound message from a channel."""
        await self._inbound_queue.put(msg)

    async def get_inbound(self) -> InboundMessage:
        """Block until the next inbound message is available."""
        return await self._inbound_queue.get()

    def subscribe_outbound(self, callback: OutboundCallback) -> None:
        """Register an async callback for outbound messages."""
        self._outbound_listeners.append(callback)

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        """Dispatch an outbound message to all registered listeners."""
        for callback in self._outbound_listeners:
            await callback(msg)
```

### Usage

- Channels call `bus.publish_inbound()` when receiving external messages
- ChannelManager calls `bus.subscribe_outbound()` to receive outbound messages
- Channel implementations register `self._on_outbound` callback

---

## 2. Abstract Base Class (Channel)

**File**: `app/channels/base.py`

Defines the contract for platform integrations. All IM platforms implement `start()`, `stop()`, `send()`.

### Evidence

```python
class Channel(ABC):
    """Base class for all IM channel implementations."""

    def __init__(self, name: str, bus: MessageBus, config: dict[str, Any]) -> None:
        self.name = name
        self.bus = bus
        self.config = config
        self._running = False

    @property
    def is_running(self) -> bool:
        return self._running

    @abstractmethod
    async def start(self) -> None:
        """Start listening for messages from the external platform."""

    @abstractmethod
    async def stop(self) -> None:
        """Gracefully stop the channel."""

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """Send a message back to the external platform."""

    async def send_file(self, msg: OutboundMessage, attachment: ResolvedAttachment) -> bool:
        """Upload a single file attachment to the platform. Default: no-op."""
        return False
```

### Implementations

- `app/channels/feishu.py` - Feishu (Lark) integration
- `app/channels/slack.py` - Slack integration
- `app/channels/telegram.py` - Telegram integration

---

## 3. Manager/Dispatcher (ChannelManager)

**File**: `app/channels/manager.py`

Central coordinator that reads from MessageBus, creates/reuses threads, routes commands, handles streaming vs blocking responses.

### Evidence

```python
class ChannelManager:
    """Core dispatcher that bridges IM channels to the DeerFlow agent."""

    def __init__(self, bus: MessageBus, store: ChannelStore, ...) -> None:
        self.bus = bus
        self.store = store

    async def start(self) -> None:
        self._running = True
        self._task = asyncio.create_task(self._dispatch_loop())

    async def _dispatch_loop(self) -> None:
        while self._running:
            msg = await asyncio.wait_for(self.bus.get_inbound(), timeout=1.0)
            task = asyncio.create_task(self._handle_message(msg))

    async def _handle_message(self, msg: InboundMessage) -> None:
        if msg.msg_type == InboundMessageType.COMMAND:
            await self._handle_command(msg)
        else:
            await self._handle_chat(msg)
```

### Key Methods

- `_handle_chat()` - Non-streaming chat (Slack, Telegram)
- `_handle_streaming_chat()` - Streaming chat (Feishu)
- `_handle_command()` - Local commands (`/new`, `/status`, `/models`, `/memory`, `/help`)
- `_resolve_run_params()` - Merges default, channel, and user session config

---

## 4. Middleware Chain (Lead Agent)

**File**: `packages/harness/deerflow/agents/lead_agent/agent.py`

Sequential processing where each middleware wraps the next, modifying state or behavior at each stage.

### Evidence

```python
def _build_middlewares(config: RunnableConfig, model_name: str | None, agent_name: str | None = None):
    """Build middleware chain based on runtime configuration."""
    middlewares = build_lead_runtime_middlewares(lazy_init=True)

    # Add summarization middleware if enabled
    summarization_middleware = _create_summarization_middleware()
    if summarization_middleware is not None:
        middlewares.append(summarization_middleware)

    # Add TodoList middleware if plan mode is enabled
    is_plan_mode = config.get("configurable", {}).get("is_plan_mode", False)
    todo_list_middleware = _create_todo_list_middleware(is_plan_mode)
    if todo_list_middleware is not None:
        middlewares.append(todo_list_middleware)

    # ... more middlewares ...

    # ClarificationMiddleware should always be last
    middlewares.append(ClarificationMiddleware())
    return middlewares
```

### Middleware Order

1. ThreadDataMiddleware - Creates per-thread directories
2. UploadsMiddleware - Tracks and injects uploaded files
3. SandboxMiddleware - Acquires sandbox, stores `sandbox_id`
4. DanglingToolCallMiddleware - Injects placeholder tool responses
5. GuardrailMiddleware - Pre-tool-call authorization (optional)
6. SummarizationMiddleware - Context reduction (optional)
7. TodoListMiddleware - Task tracking (optional)
8. TitleMiddleware - Auto-generates thread title
9. MemoryMiddleware - Queues conversations for memory update
10. ViewImageMiddleware - Injects base64 image data (vision models)
11. SubagentLimitMiddleware - Enforces max concurrent subagents
12. ClarificationMiddleware - Intercepts `ask_clarification` tool calls

---

## 5. Factory (Model Factory)

**File**: `packages/harness/deerflow/models/factory.py`

Config-driven object creation with reflection (`resolve_class()`).

### Evidence

```python
def create_chat_model(name: str | None = None, thinking_enabled: bool = False, **kwargs) -> BaseChatModel:
    """Create a chat model instance from the config."""
    config = get_app_config()
    if name is None:
        name = config.models[0].name
    model_config = config.get_model_config(name)
    if model_config is None:
        raise ValueError(f"Model {name} not found in config")

    model_class = resolve_class(model_config.use, BaseChatModel)
    model_settings_from_config = model_config.model_dump(
        exclude={"use", "name", "display_name", "description",
                 "supports_thinking", "supports_reasoning_effort",
                 "when_thinking_enabled", "thinking", "supports_vision"}
    )

    # Compute effective when_thinking_enabled by merging thinking shortcut
    effective_wte = dict(model_config.when_thinking_enabled) if model_config.when_thinking_enabled else {}
    if model_config.thinking is not None:
        merged_thinking = {**(effective_wte.get("thinking") or {}), **model_config.thinking}
        effective_wte = {**effective_wte, "thinking": merged_thinking}

    if thinking_enabled and has_thinking_settings:
        model_settings_from_config.update(effective_wte)

    model_instance = model_class(**kwargs, **model_settings_from_config)
    return model_instance
```

### Usage in Config

```yaml
# config.yaml
models:
  - name: claude-sonnet-4-20250514
    use: deerflow.models.claude_provider:ClaudeChatModel
    display_name: Claude Sonnet 4
    supports_thinking: true
    supports_vision: true
```

---

## 6. Provider (Sandbox)

**File**: `packages/harness/deerflow/sandbox/sandbox_provider.py`

Abstract resource acquisition lifecycle (`acquire`/`get`/`release`) enabling pluggable backends.

### Evidence

```python
class SandboxProvider(ABC):
    """Abstract base class for sandbox providers."""

    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str:
        """Acquire a sandbox environment and return its ID."""

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None:
        """Get a sandbox environment by ID."""

    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """Release a sandbox environment."""


_default_sandbox_provider: SandboxProvider | None = None


def get_sandbox_provider(**kwargs) -> SandboxProvider:
    """Get the sandbox provider singleton."""
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider
```

### Implementations

- `packages/harness/deerflow/sandbox/local/local_sandbox_provider.py` - Local filesystem execution
- `packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py` - Docker-based isolation

---

## 7. Singleton (APIClient)

**File**: `frontend/src/core/api/api-client.ts`

Module-level singleton ensures single HTTP client connection pool.

### Evidence

```typescript
let _singleton: LangGraphClient | null = null;
export function getAPIClient(isMock?: boolean): LangGraphClient {
  _singleton ??= createCompatibleClient(isMock);
  return _singleton;
}

function createCompatibleClient(isMock?: boolean): LangGraphClient {
  const client = new LangGraphClient({
    apiUrl: getLangGraphBaseURL(isMock),
  });

  // Patch stream methods for compatibility
  const originalRunStream = client.runs.stream.bind(client.runs);
  client.runs.stream = ((threadId, assistantId, payload) =>
    originalRunStream(
      threadId,
      assistantId,
      sanitizeRunStreamOptions(payload),
    )) as typeof client.runs.stream;

  return client;
}
```

---

## 8. Hook Pattern (Frontend)

**Files**:
- `frontend/src/core/threads/hooks.ts`
- `frontend/src/core/agents/hooks.ts`
- `frontend/src/core/memory/hooks.ts`

Custom React hooks encapsulate stateful logic, providing reactive data interface to components.

### Evidence

```typescript
export function useThreadStream({
  threadId,
  context,
  isMock,
  onStart,
  onFinish,
  onToolEnd,
}: ThreadStreamOptions) {
  const { t } = useI18n();
  const [onStreamThreadId, setOnStreamThreadId] = useState(() => threadId);
  const threadIdRef = useRef<string | null>(threadId ?? null);
  const startedRef = useRef(false);

  const thread = useStream<AgentThreadState>({
    client: getAPIClient(isMock),
    assistantId: "lead_agent",
    threadId: onStreamThreadId,
    reconnectOnMount: true,
    // ... callbacks
  });

  const sendMessage = useCallback(
    async (threadId: string, message: PromptInputMessage, extraContext?: Record<string, unknown>) => {
      // ... implementation
    },
    [thread, _handleOnStart, t.uploads.uploadingFiles, context, queryClient],
  );

  return [mergedThread, sendMessage, isUploading] as const;
}
```

### Key Hooks

- `useThreadStream` - Streaming thread communication with optimistic updates
- `useThreads` - Thread list with pagination
- `useDeleteThread` - Thread deletion with cache invalidation
- `useRenameThread` - Thread renaming with optimistic updates

---

## 9. Async Queue (MessageBus)

**File**: `app/channels/message_bus.py`

Uses `asyncio.Queue` for producer/consumer decoupling with backpressure.

### Evidence

```python
def __init__(self) -> None:
    self._inbound_queue: asyncio.Queue[InboundMessage] = asyncio.Queue()
    # Default maxsize=0 means unlimited queue
    # Backpressure handled by consumer speed

async def publish_inbound(self, msg: InboundMessage) -> None:
    """Enqueue an inbound message from a channel."""
    await self._inbound_queue.put(msg)
    logger.info(
        "[Bus] inbound enqueued: channel=%s, queue_size=%d",
        msg.channel_name,
        self._inbound_queue.qsize(),
    )

async def get_inbound(self) -> InboundMessage:
    """Block until the next inbound message is available."""
    return await self._inbound_queue.get()
```

### Consumer Side (ChannelManager)

```python
async def _dispatch_loop(self) -> None:
    while self._running:
        try:
            msg = await asyncio.wait_for(self.bus.get_inbound(), timeout=1.0)
        except TimeoutError:
            continue
        task = asyncio.create_task(self._handle_message(msg))
```

---

## 10. Strategy (Streaming vs Blocking)

**File**: `app/channels/manager.py`

Different execution strategies based on channel capability.

### Evidence

```python
CHANNEL_CAPABILITIES = {
    "feishu": {"supports_streaming": True},
    "slack": {"supports_streaming": False},
    "telegram": {"supports_streaming": False},
}

async def _handle_chat(self, msg: InboundMessage, extra_context: dict[str, Any] | None = None) -> None:
    # ...
    if self._channel_supports_streaming(msg.channel_name):
        await self._handle_streaming_chat(
            client, msg, thread_id, assistant_id, run_config, run_context
        )
        return

    # Non-streaming path (Slack, Telegram)
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"messages": [{"role": "human", "content": msg.text}]},
        config=run_config,
        context=run_context,
    )
    response_text = _extract_response_text(result)
    # ... publish outbound
```

### Streaming Strategy (Feishu)

```python
async def _handle_streaming_chat(self, client, msg, thread_id, assistant_id, run_config, run_context):
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"messages": [{"role": "human", "content": msg.text}]},
        config=run_config,
        context=run_context,
        stream_mode=["messages-tuple", "values"],
    ):
        event = getattr(chunk, "event", "")
        data = getattr(chunk, "data", None)
        # Accumulate text, publish incremental outbound with is_final=False
    # Final publish with is_final=True
```

---

## 11. State Reducer (ThreadState)

**File**: `packages/harness/deerflow/agents/thread_state.py`

Annotated state fields with custom reducer functions for deterministic merge behavior.

### Evidence

```python
def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    """Reducer for artifacts list - merges and deduplicates artifacts."""
    if existing is None:
        return new or []
    if new is None:
        return existing
    return list(dict.fromkeys(existing + new))


def merge_viewed_images(existing: dict[str, ViewedImageData] | None, new: dict[str, ViewedImageData] | None) -> dict[str, ViewedImageData]:
    """Reducer for viewed_images dict - merges image dictionaries.

    Special case: If new is an empty dict {}, it clears the existing images.
    """
    if existing is None:
        return new or {}
    if new is None:
        return existing
    if len(new) == 0:
        return {}
    return {**existing, **new}


class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

---

## 12. Lazy Initialization

Multiple components use lazy initialization to defer expensive operations until first use.

### Evidence

#### ChannelManager (lazy LangGraph client)

```python
def _get_client(self):
    """Return the langgraph_sdk async client, creating it on first use."""
    if self._client is None:
        from langgraph_sdk import get_client
        self._client = get_client(url=self._langgraph_url)
    return self._client
```

#### MCP Client (lazy tool loading)

```python
# packages/harness/deerflow/mcp/tools.py
def get_cached_mcp_tools():
    """Return cached MCP tools if config hasn't changed, reload otherwise."""
    current_mtime = extensions_config_path.stat().st_mtime
    if current_mtime != _cached_mtime:
        # Reload tools
```

#### Sandbox Provider (lazy singleton)

```python
# packages/harness/deerflow/sandbox/sandbox_provider.py
_default_sandbox_provider: SandboxProvider | None = None

def get_sandbox_provider(**kwargs) -> SandboxProvider:
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider
```

---

## 13. Boundary Enforcement (Test)

**File**: `tests/test_harness_boundary.py`

Automated test enforces architectural boundary between harness and app layers.

### Evidence

```python
"""Boundary check: harness layer must not import from app layer."""
import ast
from pathlib import Path

HARNESS_ROOT = Path(__file__).parent.parent / "packages" / "harness" / "deerflow"
BANNED_PREFIXES = ("app.",)


def test_harness_does_not_import_app():
    violations: list[str] = []

    for py_file in sorted(HARNESS_ROOT.rglob("*.py")):
        for lineno, module in _collect_imports(py_file):
            if any(module == prefix.rstrip(".") or module.startswith(prefix) for prefix in BANNED_PREFIXES):
                rel = py_file.relative_to(HARNESS_ROOT.parent.parent.parent)
                violations.append(f"  {rel}:{lineno}  imports {module}")

    assert not violations, "Harness layer must not import from app layer:\n" + "\n".join(violations)
```

### Rule

```
Harness (deerflow.*) ──NO DEPENDENCY──► App (app.*)
                  ◄─── CAN IMPORT ─────
```

---

## Pattern Interactions

### Message Flow with Multiple Patterns

```
Channel (ABC Pattern)
    │
    ▼ publishes
MessageBus (Pub/Sub + Async Queue)
    │
    ▼ consumes
ChannelManager (Manager/Dispatcher + Strategy Pattern)
    │
    ├──► runs.wait() ──────► Non-streaming response
    │
    └──► runs.stream() ────► Streaming response (Strategy)
              │
              ▼
         Middleware Chain (Middleware Chain Pattern)
              │
              ▼
         Lead Agent (Factory Pattern)
              │
              ▼
         Sandbox (Provider Pattern)
```

### Thread State Updates

```
User Input
    │
    ▼
useThreadStream (Hook Pattern)
    │
    ├──► Optimistic update (Hook Pattern - local state)
    │
    ▼ submit()
LangGraph SDK (Singleton)
    │
    ▼
ThreadState (State Reducer Pattern)
    │
    ▼
useStream callback (Hook Pattern)
    │
    ▼
TanStack Query cache (React Query Pattern)
```

---

## References

### Primary Source Files

- `backend/app/channels/message_bus.py` - Pub/Sub, Async Queue
- `backend/app/channels/base.py` - Abstract Base Class
- `backend/app/channels/manager.py` - Manager/Dispatcher, Strategy
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py` - Middleware Chain
- `backend/packages/harness/deerflow/models/factory.py` - Factory
- `backend/packages/harness/deerflow/sandbox/sandbox_provider.py` - Provider
- `backend/packages/harness/deerflow/sandbox/sandbox.py` - Sandbox interface
- `backend/packages/harness/deerflow/agents/thread_state.py` - State Reducer
- `frontend/src/core/api/api-client.ts` - Singleton
- `frontend/src/core/threads/hooks.ts` - Hook Pattern
- `backend/tests/test_harness_boundary.py` - Boundary Enforcement

### Configuration Files

- `config.yaml` - Model and tool configurations
- `extensions_config.json` - MCP and skills configurations
