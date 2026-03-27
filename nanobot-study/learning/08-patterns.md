# Design Patterns

## Architectural Patterns

### 1. Event-Driven Architecture

**Core pattern:** Async message bus decoupling producers from consumers.

```
Channel → MessageBus → AgentLoop → MessageBus → Channel
```

**Implementation:** `nanobot/bus/queue.py` with `asyncio.Queue`

**Benefit:** Channels and agent operate independently. Channels produce, AgentLoop consumes.

---

### 2. Adapter Pattern

**Usage:** Channel implementations adapt platform-specific APIs to unified `BaseChannel` interface.

```python
class BaseChannel(ABC):
    @abstractmethod async def start(self) -> None
    @abstractmethod async def stop(self) -> None
    @abstractmethod async def send(self, msg: OutboundMessage) -> None
```

Each platform (Telegram, Discord, Slack) implements these methods with platform-specific logic.

---

### 3. Strategy Pattern

**Usage:** `LLMProvider` abstract base with multiple implementations.

```python
class LLMProvider(ABC):
    @abstractmethod
    async def chat(messages, tools, model, max_tokens, temperature) -> LLMResponse
```

Implementations: `AnthropicProvider`, `OpenAICompatProvider`, `AzureOpenAIProvider`, etc.

---

### 4. Registry/Plugin Pattern

**Usage:** Centralized registries enable runtime discovery without code changes.

```python
# Channel discovery
for _, name, ispkg in pkgutil.iter_modules(pkg.__path__):
    if name not in _INTERNAL and not ispkg:
        channel_names.append(name)

# Provider lookup
spec = find_by_name("anthropic")
if not spec:
    spec = PROVIDERS[-1]  # custom fallback
```

**Registries:** Channels, Providers, Tools, Skills

---

### 5. Template Method Pattern

**Usage:** `BaseChannel` defines skeleton with `_handle_message()`, subclasses implement `start()`, `stop()`, `send()`.

```python
class BaseChannel(ABC):
    async def _handle_message(self, sender_id, chat_id, content, ...):
        # Common validation and message creation
        if not self.is_allowed(sender_id):
            return
        msg = InboundMessage(...)
        await self.bus.publish_inbound(msg)
```

---

### 6. Command Pattern

**Usage:** `CommandRouter` dispatches slash commands to handlers.

**Four-tier dispatch:**
1. **Priority** (exact match, outside session lock) - `/stop`, `/restart`
2. **Exact** (exact match, inside session lock)
3. **Prefix** (longest-prefix-first)
4. **Interceptors** (fallback predicates)

---

### 7. Message Queue Pattern

**Usage:** `MessageBus` uses `asyncio.Queue` for async producer-consumer decoupling.

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]
```

---

## Implementation Patterns

### 8. Zero-Import Discovery

**Usage:** `pkgutil.iter_modules` for discovering channels without importing.

```python
def discover_channel_names() -> list[str]:
    import nanobot.channels as pkg
    return [
        name
        for _, name, ispkg in pkgutil.iter_modules(pkg.__path__)
        if name not in _INTERNAL and not ispkg
    ]
```

**Benefit:** Fast startup, no circular imports.

---

### 9. Concurrent Tool Execution

**Usage:** `asyncio.gather` with `return_exceptions=True` for parallel tool calls.

```python
results = await asyncio.gather(*(
    self.tools.execute(tc.name, tc.arguments)
    for tc in response.tool_calls
), return_exceptions=True)

for tool_call, result in zip(response.tool_calls, results):
    if isinstance(result, BaseException):
        result = f"Error: {type(result).__name__}: {result}"
```

---

### 10. Session Isolation with Cross-Session Parallelism

**Usage:** Per-session locks for serial processing within session, parallel across sessions.

```python
async def _dispatch(self, msg: InboundMessage) -> None:
    lock = self._session_locks.setdefault(msg.session_key, asyncio.Lock())
    async with lock:
        # Per-session processing (serial)
```

---

### 11. Streaming Think-Block Filtering

**Usage:** Incrementally strips `<think>` blocks during streaming.

```python
async def _filtered_stream(delta: str) -> None:
    prev_clean = strip_think(_stream_buf)
    _stream_buf += delta
    new_clean = strip_think(_stream_buf)
    incremental = new_clean[len(prev_clean):]
    if incremental and _raw_stream:
        await _raw_stream(incremental)
```

---

### 12. Tool Call ID Normalization

**Usage:** 9-char alphanumeric IDs compatible with all providers.

```python
@staticmethod
def _normalize_tool_call_id(tool_call_id: Any) -> Any:
    if len(tool_call_id) == 9 and tool_call_id.isalnum():
        return tool_call_id
    return hashlib.sha1(tool_call_id.encode()).hexdigest()[:9]
```

---

### 13. Graceful Degradation

**Usage:** Fall back to degraded mode on repeated failures.

```python
# Memory consolidation fallback
if _consecutive_failures < _MAX_FAILURES_BEFORE_RAW_ARCHIVE:
    return False
_raw_archive(messages)  # Dump without summarization
return True  # "Success" even though degraded
```

---

### 14. Error Hints Appended

**Usage:** Tool errors get hint encouraging self-correction.

```python
_HINT = "\n\n[Analyze the error above and try a different approach.]"
if result.startswith("Error"):
    return result + _HINT
```

---

### 15. ContextVar Guard

**Usage:** Prevents cron jobs from scheduling new cron jobs.

```python
_in_cron_context: ContextVar[bool] = ContextVar("cron_in_context", default=False)

def set_cron_context(self, active: bool):
    return self._in_cron_context.set(active)
```

---

### 16. Notification Gate

**Usage:** LLM decides if result warrants notification.

```python
should_notify = await evaluate_response(response, tasks, provider, channel)
if should_notify and self.on_notify:
    await self.on_notify(response)
```

Prevents notification spam for routine/empty results.

---

### 17. Tool Parameter Casting

**Usage:** Auto-converts string values to expected types.

```python
if target_type == "integer" and isinstance(val, str):
    try:
        return int(val)
    except ValueError:
        return val
```

---

## Anti-Patterns to Avoid

1. **Global mutable state for config path** - `_current_config_path` not thread-safe
2. **In-memory session locks** - Don't work across multiple nanobot instances
3. **Shell subprocess with regex guards** - Not a true sandbox
4. **No delete in memory system** - Facts persist indefinitely
