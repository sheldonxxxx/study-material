# Feature Batch 2 Analysis

**Batch:** Context Engineering, Long-Term Memory, Multi-Channel IM (Telegram, Slack, Feishu/Lark)
**Repo:** /Users/sheldon/Documents/claw/reference/deer-flow
**Analyzed:** 2026-03-26

---

## Feature 1: Context Engineering

### Core Implementation

**Location:** `backend/packages/harness/deerflow/agents/memory/` + `backend/packages/harness/deerflow/agents/middlewares/`

Context Engineering is implemented through a middleware chain and memory system that aggressively manages context to prevent window blowout.

### Key Components

#### 1. UploadsMiddleware (`uploads_middleware.py`)

Injects file upload context before agent execution:

```python
# Creates <uploaded_files> block prepended to human message
files_message = _create_files_message(new_files, historical_files)
updated_message = HumanMessage(
    content=f"{files_message}\n\n{original_content}",
    id=last_message.id,
    additional_kwargs=last_message.additional_kwargs,
)
```

**Clever pattern:** Scans the thread's uploads directory to build `historical_files` list, excluding new uploads. Preserves `additional_kwargs` so frontend can read structured file info from streamed messages.

**Edge case handled:** Validates file existence before listing. Skips entries where `filename` is not just a name (guards against path injection).

#### 2. MemoryMiddleware (`memory_middleware.py`)

Queues conversations for async memory update after agent execution:

```python
def _filter_messages_for_memory(messages: list[Any]) -> list[Any]:
    # Strips <uploaded_files> blocks from human messages
    # Filters to: human messages (with uploads stripped) + AI messages without tool_calls
    # Drop entire human+AI pair if the human was upload-only
```

**Key insight:** Uses `skip_next_ai` flag to handle upload-only messages. When a human message contains only `<uploaded_files>` and nothing else after stripping, the paired AI response is also dropped.

#### 3. Memory Queue (`queue.py`)

Debounced update queue with per-thread deduplication:

```python
# Each thread dedupes itself - newer context replaces older pending context
self._queue = [c for c in self._queue if c.thread_id != thread_id]
self._queue.append(context)
```

**Design pattern:** Uses `threading.Timer` for debouncing. Processes all queued contexts in batch after debounce period. Small delay (0.5s) between updates to avoid rate limiting.

#### 4. Memory Updater (`updater.py`)

LLM-based memory extraction with atomic file I/O:

```python
# Atomic write via temp file + rename
temp_path = file_path.with_suffix(".tmp")
with open(temp_path, "w", encoding="utf-8") as f:
    json.dump(memory_data, f, indent=2, ensure_ascii=False)
temp_path.replace(file_path)  # Atomic on most systems
```

**Important:** File-upload mentions are stripped from summaries before saving:
```python
_UPLOAD_SENTENCE_RE = re.compile(
    r"[^.!?]*\b(?:upload(?:ed|ing)?(?:\s+\w+){0,3}\s+(?:file|files?|document...)"
    # ... deliberately narrow to avoid removing legitimate facts
)
```
This prevents the agent from searching for non-existent files in future sessions.

**Fact deduplication:** Whitespace-normalized comparison before appending:
```python
def _fact_content_key(content: Any) -> str | None:
    stripped = content.strip()
    return stripped if stripped else None
```

#### 5. Memory Prompt Template (`prompt.py`)

Rich prompt engineering for memory extraction:

```
Memory Section Guidelines:
- workContext: Professional role, company, key projects (2-3 sentences)
- personalContext: Languages, preferences, interests (1-2 sentences)
- topOfMind: Multiple ongoing focus areas (3-5 sentences) - updated most frequently
- recentMonths: Detailed 1-3 month timeline (4-6 sentences)
- earlierContext: 3-12 month patterns (3-5 sentences)
- longTermBackground: Persistent foundational facts (2-4 sentences)
```

**Facts have confidence scores:** 0.9-1.0 for explicit statements, 0.7-0.8 for strongly implied, 0.5-0.6 for inferred patterns.

#### 6. Token-Aware Injection (`prompt.py`)

```python
def format_memory_for_injection(memory_data: dict[str, Any], max_tokens: int = 2000) -> str:
    # Sorts facts by confidence (descending)
    # Accumulates tokens incrementally to avoid full-string re-tokenization
    # Truncates at token budget with "..." suffix
```

**Optimization:** Computes base token count once, then adds fact lines incrementally. Each line preceded by newline (except first) to mimic joined string.

### Data Flow

```
User Message → UploadsMiddleware (injects file context)
             → Agent executes
             → MemoryMiddleware (after_agent hook)
             → MemoryQueue.add() [debounced 30s]
             → MemoryUpdater.update_memory() [LLM extraction]
             → Atomic file save to memory.json
             → Next interaction: format_memory_for_injection() → system prompt
```

### Clever Solutions

1. **Per-thread queue deduplication:** Same thread's pending update is replaced by newer context rather than accumulating
2. **Whitespace-normalized fact dedup:** Trims leading/trailing whitespace before comparing fact content
3. **Atomic file I/O:** Temp file + rename pattern prevents corrupted memory.json on crash
4. **Upload-mention scrubbing:** Regex carefully excludes legitimate facts like "user works with CSV files"

### Technical Debt / Shortcuts

1. **Global singleton queue:** `get_memory_queue()` returns a module-level singleton. Tests must call `reset_memory_queue()` to isolate.
2. **No fact expiration:** Facts accumulate until `max_facts` (100) is hit, then lowest-confidence are evicted. No time-based expiration.
3. **Planned TF-IDF retrieval not implemented:** `MEMORY_IMPROVEMENTS.md` documents planned similarity-based retrieval but it's not shipped yet.
4. **`print()` for logging in queue:** Uses `print()` instead of proper logging in `queue.py` lines 64, 82, 103, 110, 117, 121.

---

## Feature 2: Long-Term Memory

### Core Implementation

**Location:** `backend/packages/harness/deerflow/agents/memory/` + `backend/docs/MEMORY_IMPROVEMENTS.md`

Long-term memory persists across sessions through a JSON file with structured data.

### Data Structure (`updater.py`)

```python
def _create_empty_memory() -> dict[str, Any]:
    return {
        "version": "1.0",
        "lastUpdated": "...",
        "user": {
            "workContext": {"summary": "", "updatedAt": ""},
            "personalContext": {"summary": "", "updatedAt": ""},
            "topOfMind": {"summary": "", "updatedAt": ""},
        },
        "history": {
            "recentMonths": {"summary": "", "updatedAt": ""},
            "earlierContext": {"summary": "", "updatedAt": ""},
            "longTermBackground": {"summary": "", "updatedAt": ""},
        },
        "facts": [],  # List of fact entries
    }

# Fact entry shape:
{
    "id": "fact_abc12345",
    "content": "User works primarily with Python and TypeScript",
    "category": "knowledge",  # preference|knowledge|context|behavior|goal
    "confidence": 0.85,
    "createdAt": "2026-03-26T...",
    "source": "thread_xxx",
}
```

### Memory Cache with File Mtime Invalidation

```python
_memory_cache: dict[str | None, tuple[dict[str, Any], float | None]] = {}

def get_memory_data(agent_name: str | None = None) -> dict[str, Any]:
    file_path = _get_memory_file_path(agent_name)
    current_mtime = file_path.stat().st_mtime if file_path.exists() else None
    cached = _memory_cache.get(agent_name)
    if cached is None or cached[1] != current_mtime:
        memory_data = _load_memory_from_file(agent_name)
        _memory_cache[agent_name] = (memory_data, current_mtime)
    return cached[0]
```

**Pattern:** Cache keyed by `(memory_data, file_mtime)`. Any external modification to the file invalidates cache.

### Fact Confidence Threshold

```python
# Facts below threshold are not stored
if confidence >= config.fact_confidence_threshold:  # default: 0.7
    # ... add fact
```

### Max Facts Limit

```python
if len(current_memory["facts"]) > config.max_facts:  # default: 100
    current_memory["facts"] = sorted(
        current_memory["facts"],
        key=lambda f: f.get("confidence", 0),
        reverse=True,
    )[:config.max_facts]
```

### Per-Agent Memory Support

```python
def _get_memory_file_path(agent_name: str | None = None) -> Path:
    if agent_name is not None:
        return get_paths().agent_memory_file(agent_name)
    # Falls back to global memory file
```

### Missing Planned Features

From `MEMORY_IMPROVEMENTS.md`:
- **TF-IDF similarity-based fact retrieval** - Not implemented
- **Context-aware scoring** with `current_context` input - Not implemented
- **Configurable similarity/confidence weights** - Not implemented
- **Middleware/runtime wiring for context-aware retrieval** - Not implemented

Current behavior: Facts ranked purely by confidence, injected until token budget exhausted.

---

## Feature 3: Multi-Channel IM (Telegram, Slack, Feishu/Lark)

### Architecture Overview

```
External Platform → Channel (telegram.py/slack.py/feishu.py)
                  → MessageBus.publish_inbound()
                  → ChannelManager._dispatch_loop()
                  → LangGraph Server (thread create/invoke)
                  → MessageBus.publish_outbound()
                  → Channel.send()
                  → External Platform reply
```

### Core Components

#### 1. MessageBus (`message_bus.py`)

Async pub/sub hub decoupling channels from dispatcher:

```python
class MessageBus:
    def __init__(self) -> None:
        self._inbound_queue: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self._outbound_listeners: list[OutboundCallback] = []

    async def publish_inbound(self, msg: InboundMessage) -> None:
        await self._inbound_queue.put(msg)

    async def get_inbound(self) -> InboundMessage:
        return await self._inbound_queue.get()

    def subscribe_outbound(self, callback: OutboundCallback) -> None:
        self._outbound_listeners.append(callback)
```

**Design pattern:** Channels are publishers (produce `InboundMessage`), ChannelManager is consumer. ChannelManager publishes `OutboundMessage`, channels are listeners.

#### 2. ChannelStore (`store.py`)

JSON-file persistence mapping `channel:chat_id[:topic_id]` to `thread_id`:

```python
# Atomic save via temp file
fd = tempfile.NamedTemporaryFile(mode="w", dir=self._path.parent, suffix=".tmp", delete=False)
json.dump(self._data, fd, fd.close())
Path(fd.name).replace(self._path)
```

**Key format:**
- `"channel:chat_id"` for root conversations
- `"channel:chat_id:topic_id"` for threaded conversations

**Note:** JSON file rewritten atomically on every mutation. Comment warns this is "intentionally simple" and can be swapped for a database.

#### 3. ChannelManager (`manager.py`)

Core dispatcher bridging IM to LangGraph:

```python
class ChannelManager:
    async def _handle_chat(self, msg: InboundMessage) -> None:
        # Look up or create thread
        thread_id = self.store.get_thread_id(msg.channel_name, msg.chat_id, topic_id=msg.topic_id)
        if thread_id is None:
            thread_id = await self._create_thread(client, msg)

        # Streaming vs blocking based on channel capability
        if self._channel_supports_streaming(msg.channel_name):
            await self._handle_streaming_chat(...)
        else:
            result = await client.runs.wait(...)  # Blocking
```

**Streaming capability map:**
```python
CHANNEL_CAPABILITIES = {
    "feishu": {"supports_streaming": True},
    "slack": {"supports_streaming": False},
    "telegram": {"supports_streaming": False},
}
```

**Concurrency control:** `asyncio.Semaphore(max_concurrency=5)` limits concurrent message processing.

#### 4. Base Channel (`base.py`)

Abstract base class with common patterns:

```python
class Channel(ABC):
    async def _on_outbound(self, msg: OutboundMessage) -> None:
        if msg.channel_name == self.name:
            await self.send(msg)  # Text first
            for attachment in msg.attachments:
                await self.send_file(msg, attachment)  # Files only if text succeeded
```

**Pattern:** Sends text message first. If text fails, skips file uploads entirely to avoid partial deliveries.

### Platform-Specific Implementations

#### Telegram (`telegram.py`)

- **Transport:** Long-polling via `python-telegram-bot`
- **Threading:** Runs polling in dedicated thread with own event loop
- **Threading model:** `topic_id = None` for private chats (single thread), `topic_id = reply_to.message_id` for group chats
- **Retry:** 3 retries with exponential backoff (1s, 2s)
- **Running reply:** Replies "Working on it..." to user message before publishing inbound
- **File limits:** 10MB photos, 50MB documents

```python
def _run_polling(self) -> None:
    # Cannot use run_polling() because it calls add_signal_handler()
    # which only works in the main thread
    self._tg_loop.run_until_complete(self._application.initialize())
    self._tg_loop.run_until_complete(self._application.start())
    self._tg_loop.run_until_complete(self._application.updater.start_polling())
```

#### Slack (`slack.py`)

- **Transport:** Socket Mode (WebSocket, no public IP needed)
- **SDK:** `slack-sdk` SocketModeClient
- **Threading model:** Uses `thread_ts` as `topic_id`. Threaded messages share topic, non-threaded each get new topic
- **Reactions:** Adds `eyes` reaction on receive, `white_check_mark` on success, `x` on failure
- **Markdown conversion:** Uses `markdown_to_mrkdwn` for output formatting

```python
# Running reply sent fire-and-forget from SDK thread
self._send_running_reply(channel_id, thread_ts)
asyncio.run_coroutine_threadsafe(self.bus.publish_inbound(inbound), self._loop)
```

#### Feishu (`feishu.py`)

- **Transport:** WebSocket long-connection via `lark-oapi`
- **Threading workaround:** Runs in dedicated thread with plain asyncio loop (not uvloop) because lark-oapi captures event loop at import time

```python
def _run_ws(self, app_id: str, app_secret: str) -> None:
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    import lark_oapi.ws.client as _ws_client_mod
    _ws_client_mod.loop = loop  # Patch SDK's module-level loop reference
    ws_client = lark.ws.Client(...)
    ws_client.start()
```

- **Running card pattern:** Creates interactive card, patches same card for each update (uses `update_multi: true`)

```python
async def _send_card_message(self, msg: OutboundMessage) -> None:
    running_card_id = self._running_card_ids.get(source_message_id)
    if running_card_id:
        await self._update_card(running_card_id, msg.text)  # Patch in place
    else:
        await self._ensure_running_card(source_message_id, msg.text)
```

- **File limits:** 10MB images, 30MB files

### Command Handling

All channels support slash commands routed via `InboundMessageType.COMMAND`:

```
/bootstrap — Initialize workspace (creates thread with is_bootstrap context)
/new — Start new conversation (creates new thread)
/status — Show current thread
/models — List available models (queries Gateway API)
/memory — Show memory status (queries Gateway API)
/help — Show available commands
```

### Artifact/File Delivery

```python
def _resolve_attachments(thread_id: str, artifacts: list[str]) -> list[ResolvedAttachment]:
    # SECURITY: Only allows paths under /mnt/user-data/outputs/
    if not virtual_path.startswith(_OUTPUTS_VIRTUAL_PREFIX):
        logger.warning("[Manager] rejected non-outputs artifact path: %s", virtual_path)
        continue
    # Verifies resolved path is actually under outputs directory (guards against traversal)
```

**Security measures:**
1. Only `/mnt/user-data/outputs/` artifacts can be sent
2. Path traversal check after resolution
3. File existence verified before upload

### Clever Patterns

1. **Thread reuse:** `topic_id` maps to same LangGraph thread. Messages in same Feishu thread/Slack thread share conversation.
2. **Streaming debouncing:** `STREAM_UPDATE_MIN_INTERVAL_SECONDS = 0.35` prevents message flooding
3. **Background task tracking:** Feishu keeps strong references to fire-and-forget tasks to surface errors
4. **Retry with emoji feedback:** Slack adds checkmark on success, X on failure after all retries exhausted
5. **Lazy LangGraph client:** `_get_client()` creates SDK client on first use

### Technical Debt / Issues

1. **uvloop conflict in Feishu:** Requires dedicated thread workaround due to lark-oapi capturing module-level event loop
2. **No graceful degradation:** If LangGraph Server is down, all channel messages fail with no queue/retry
3. **JSON store limitations:** Comment admits "for production workloads with high concurrency, this can be swapped for a proper database backend"
4. **print() in queue.py:** Uses `print()` instead of logging (shared technical debt with memory system)
5. **No message retry queue:** Failed dispatches are not retried, just logged
6. **Allowed users filter in Telegram:** Parses `allowed_users` as integers but stores as string in `InboundMessage.user_id`

---

## Cross-Feature Observations

### Shared Patterns

1. **Atomic file operations:** Both memory system and channel store use temp file + rename pattern
2. **Async pub/sub decoupling:** MessageBus pattern used for channel-agent communication
3. **Config-driven:** All features configured via `config.yaml`
4. **Lazy initialization:** Model clients, LangGraph clients all lazy-created

### Shared Technical Debt

1. **`print()` for logging** in `queue.py` lines 64, 82, 103, 110, 117, 121
2. **No structured logging** - uses basic `logger.info/warning/error` without contextual fields
3. **Singleton patterns** - `get_memory_queue()`, `get_channel_service()` - harder to test

### Architecture Strengths

1. **Clean separation:** Harness (`deerflow.*`) vs App (`app.*`) boundary enforced
2. **Plugin architecture:** New channels addable via registry + config
3. **Middleware chain:** Context management composable via middleware pattern

---

## Files Reference

### Context Engineering + Long-Term Memory
| File | Purpose |
|------|---------|
| `agents/memory/prompt.py` | Memory injection prompts and `format_memory_for_injection()` |
| `agents/memory/updater.py` | `MemoryUpdater` class, atomic file I/O, fact extraction |
| `agents/memory/queue.py` | `MemoryUpdateQueue` with debounce, per-thread dedup |
| `agents/memory/__init__.py` | Exports |
| `agents/middlewares/memory_middleware.py` | Queues messages after agent execution |
| `agents/middlewares/uploads_middleware.py` | Injects file context before agent execution |
| `docs/MEMORY_IMPROVEMENTS.md` | Roadmap documenting planned TF-IDF retrieval |
| `docs/summarization.md` | Summarization middleware configuration |

### Multi-Channel IM
| File | Purpose |
|------|---------|
| `channels/base.py` | Abstract `Channel` base class |
| `channels/message_bus.py` | `InboundMessage`, `OutboundMessage`, `MessageBus` pub/sub |
| `channels/manager.py` | `ChannelManager` - dispatcher, streaming, artifact resolution |
| `channels/store.py` | `ChannelStore` - JSON file persistence |
| `channels/service.py` | `ChannelService` - lifecycle, lazy channel loading |
| `channels/telegram.py` | `TelegramChannel` - long-polling |
| `channels/slack.py` | `SlackChannel` - Socket Mode |
| `channels/feishu.py` | `FeishuChannel` - WebSocket with card patching |
