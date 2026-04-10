# Feature Deep Dive - Batch 1: Agent Core, Multi-Channel, Multi-Provider

**Project:** nanobot v0.1.4.post5
**Date:** 2026-03-26
**Files Analyzed:**
- `nanobot/agent/loop.py` (645 lines)
- `nanobot/agent/context.py` (280 lines)
- `nanobot/agent/memory.py` (450 lines)
- `nanobot/agent/skills.py` (229 lines)
- `nanobot/agent/subagent.py` (237 lines)
- `nanobot/channels/base.py` (178 lines)
- `nanobot/channels/manager.py` (201 lines)
- `nanobot/channels/registry.py` (72 lines)
- `nanobot/channels/telegram.py` (37KB)
- `nanobot/channels/discord.py` (15KB)
- `nanobot/providers/base.py` (369 lines)
- `nanobot/providers/registry.py` (355 lines)
- `nanobot/providers/openai_compat_provider.py` (573 lines)
- `nanobot/providers/anthropic_provider.py` (442 lines)

---

## Feature 1: Agent Core Loop

### Overview

The Agent Core Loop is the central processing engine that orchestrates LLM-driven tool execution, memory management, and response generation. It consumes inbound messages from a message bus, processes them through an iterative LLM loop, executes tools, and publishes responses.

### Key Files and Classes

| File | Class/Functions | Purpose |
|------|-----------------|---------|
| `agent/loop.py` | `AgentLoop` | Main processing engine |
| `agent/context.py` | `ContextBuilder` | Prompt construction with history/memory/skills |
| `agent/memory.py` | `MemoryConsolidator` | Token-based memory management |
| `agent/skills.py` | `SkillsLoader` | Skill discovery and loading |
| `agent/subagent.py` | `SubagentManager` | Background task execution |
| `agent/tools/registry.py` | `ToolRegistry` | Tool registration and execution |

### Architecture

```
InboundMessage --> MessageBus --> AgentLoop
                                      |
                                      v
                              ContextBuilder
                              (builds prompt)
                                      |
                                      v
                              LLMProvider.chat()
                                      |
                    +-----------------+-----------------+
                    |                                   |
              has_tool_calls?                      no tool_calls
                    |                                   |
                    v                                   v
            ToolRegistry.execute()              final_content
            (concurrent via gather)                    |
                    |                                   |
                    v                                   v
            add_tool_result                       OutboundMessage
            to messages                                   |
                    |                                   v
                    +-----------> MessageBus --> ChannelManager
```

### Core Processing Flow (agent/loop.py)

The main `_run_agent_loop` method implements the iterative tool execution:

```python
while iteration < self.max_iterations:
    iteration += 1

    tool_defs = self.tools.get_definitions()

    if on_stream:
        response = await self.provider.chat_stream_with_retry(...)
    else:
        response = await self.provider.chat_with_retry(...)

    if response.has_tool_calls:
        # Execute ALL tool calls concurrently
        results = await asyncio.gather(*(
            self.tools.execute(tc.name, tc.arguments)
            for tc in response.tool_calls
        ), return_exceptions=True)

        for tool_call, result in zip(response.tool_calls, results):
            if isinstance(result, BaseException):
                result = f"Error: {type(result).__name__}: {result}"
            messages = self.context.add_tool_result(
                messages, tool_call.id, tool_call.name, result
            )
    else:
        final_content = self._strip_think(response.content)
        break
```

**Key observations:**
1. **Concurrent tool execution**: Uses `asyncio.gather` with `return_exceptions=True` to execute all tool calls in parallel while collecting all results
2. **Tool context isolation**: `_set_tool_context()` is called right before tool execution to ensure concurrent sessions don't clobber each other's routing
3. **Max iterations protection**: Default 25 iterations prevents infinite loops
4. **Error handling**: BaseExceptions are caught and converted to error strings

### Session and Concurrency Management

```python
async def _dispatch(self, msg: InboundMessage) -> None:
    """Process a message: per-session serial, cross-session concurrent."""
    lock = self._session_locks.setdefault(msg.session_key, asyncio.Lock())
    gate = self._concurrency_gate or nullcontext()
    async with lock, gate:
        # Per-session processing
```

- **Per-session serial**: Messages for the same session are processed sequentially via `asyncio.Lock`
- **Cross-session concurrent**: Different sessions can run in parallel
- **Active task tracking**: `_active_tasks` dictionary tracks running tasks per session for `/stop` command

### Streaming Implementation

The streaming implementation has a clever `<think>` block filtering mechanism:

```python
async def _filtered_stream(delta: str) -> None:
    nonlocal _stream_buf
    from nanobot.utils.helpers import strip_think
    prev_clean = strip_think(_stream_buf)
    _stream_buf += delta
    new_clean = strip_think(_stream_buf)
    incremental = new_clean[len(prev_clean):]
    if incremental and _raw_stream:
        await _raw_stream(incremental)
```

This ensures downstream consumers (CLI, channels) never see `<think>` blocks - they're stripped incrementally as they complete.

### Memory Consolidation

After each message, `MemoryConsolidator.maybe_consolidate_by_tokens()` is called:

```python
await self.memory_consolidator.maybe_consolidate_by_tokens(session)
# ... process message ...
self._schedule_background(self.memory_consolidator.maybe_consolidate_by_tokens(session))
```

The consolidation is scheduled as a background task to avoid blocking the response.

### Subagent System

`SubagentManager` spawns background agents with a restricted tool set:

```python
# Build subagent tools (no message tool, no spawn tool)
tools = ToolRegistry()
tools.register(ReadFileTool(...))
tools.register(WriteFileTool(...))
tools.register(EditFileTool(...))
tools.register(ListDirTool(...))
tools.register(ExecTool(...))
tools.register(WebSearchTool(...))
tools.register(WebFetchTool(...))
```

**Notable restrictions:**
- No `message` tool (can't send messages)
- No `spawn` tool (can't spawn more subagents - prevents recursion)

Subagent results are announced back to the main agent via an `InboundMessage` on the `system` channel with `sender_id="subagent"`.

### Context Building

`ContextBuilder` assembles prompts from multiple sources:

1. **System prompt**: Runtime context (time, date, workspace)
2. **Skills**: Available skills loaded via `SkillsLoader`
3. **Session history**: Conversation history from `SessionManager`
4. **Current message**: The user's current input

```python
def build_messages(
    self,
    history: list[dict],
    current_message: str,
    media: list[str] | None = None,
    channel: str = "cli",
    chat_id: str = "direct",
) -> list[dict[str, Any]]:
```

### Tool Result Truncation

To prevent context overflow, tool results are truncated before saving to session history:

```python
_TOOL_RESULT_MAX_CHARS = 4000  # In AgentLoop

if isinstance(content, str) and len(content) > self._TOOL_RESULT_MAX_CHARS:
    entry["content"] = content[:self._TOOL_RESULT_MAX_CHARS] + "\n... (truncated)"
```

### Context Sanitization

Before persisting messages to session history:

```python
def _sanitize_persisted_blocks(self, content: list[dict[str, Any]], ...) -> list[dict[str Any]]:
    # 1. Drop runtime context tags
    # 2. Convert data:image URLs to [image: path] placeholders
    # 3. Truncate long text blocks
```

### Error Handling Patterns

1. **LLM Errors**: Return friendly message, log error
2. **Tool Errors**: Append hint to try different approach
3. **Max Iterations**: Return instructive message about breaking tasks into smaller steps

---

## Feature 2: Multi-Channel Integration

### Overview

Nanobot supports 12 chat platforms via a plugin-based channel architecture. Each channel is a separate Python module implementing the `BaseChannel` abstract interface.

### Supported Channels

| Channel | File | Size | Display Name |
|---------|------|------|--------------|
| Telegram | `telegram.py` | 37KB | Telegram |
| Discord | `discord.py` | 15KB | Discord |
| Slack | `slack.py` | 12KB | Slack |
| DingTalk | `dingtalk.py` | 23KB | DingTalk |
| Feishu | `feishu.py` | 49KB | Feishu/Lark |
| WeCom | `wecom.py` | 13KB | WeChat Work |
| WeChat | `weixin.py` | 39KB | WeChat |
| QQ | `qq.py` | 22KB | QQ |
| Matrix | `matrix.py` | 30KB | Matrix |
| Email | `email.py` | 17KB | Email |
| MoChat | `mochat.py` | 37KB | MoChat |
| WhatsApp | `whatsapp.py` | 10KB + TS bridge | WhatsApp |

### Plugin Discovery (channels/registry.py)

```python
def discover_channel_names() -> list[str]:
    """Return all built-in channel module names by scanning the package (zero imports)."""
    import nanobot.channels as pkg
    return [
        name
        for _, name, ispkg in pkgutil.iter_modules(pkg.__path__)
        if name not in _INTERNAL and not ispkg
    ]

def discover_plugins() -> dict[str, type[BaseChannel]]:
    """Discover external channel plugins registered via entry_points."""
    for ep in entry_points(group="nanobot.channels"):
        cls = ep.load()
        plugins[ep.name] = cls
```

**Key insight**: Discovery uses `pkgutil.iter_modules` for zero-import scanning. External plugins are discovered via `entry_points`.

### Base Channel Interface (channels/base.py)

```python
class BaseChannel(ABC):
    name: str = "base"
    display_name: str = "Base"
    transcription_api_key: str = ""

    async def start(self) -> None: pass      # Long-running async task
    async def stop(self) -> None: pass       # Cleanup
    async def send(self, msg: OutboundMessage) -> None: pass  # Send message
    async def send_delta(self, chat_id, delta, metadata) -> None: pass  # Streaming
```

### Channel Manager (channels/manager.py)

The `ChannelManager` orchestrates all channels:

```python
class ChannelManager:
    def _init_channels(self) -> None:
        """Initialize channels discovered via pkgutil scan + entry_points plugins."""
        from nanobot.channels.registry import discover_all

        for name, cls in discover_all().items():
            section = getattr(self.config.channels, name, None)
            if section is None:
                continue
            # ... check enabled ...
            channel = cls(section, self.bus)
            channel.transcription_api_key = groq_key
            self.channels[name] = channel
```

**Key responsibilities:**
1. Initialize discovered channels with their config
2. Start/stop all channels
3. Dispatch outbound messages with retry policy
4. Route messages to appropriate channel

### Outbound Message Retry

```python
_SEND_RETRY_DELAYS = (1, 2, 4)  # Exponential backoff

async def _send_with_retry(self, channel: BaseChannel, msg: OutboundMessage) -> None:
    for attempt in range(max_attempts):
        try:
            await self._send_once(channel, msg)
            return
        except asyncio.CancelledError:
            raise  # Propagate for graceful shutdown
        except Exception as e:
            if attempt == max_attempts - 1:
                logger.error("Failed to send after {} attempts", max_attempts)
                return
            delay = _SEND_RETRY_DELAYS[min(attempt, len(_SEND_RETRY_DELAYS) - 1)]
            await asyncio.sleep(delay)
```

### Message Handling Pattern

Each channel's `_handle_message()` method:

```python
async def _handle_message(
    self,
    sender_id: str,
    chat_id: str,
    content: str,
    media: list[str] | None = None,
    metadata: dict[str, Any] | None = None,
    session_key: str | None = None,
) -> None:
    if not self.is_allowed(sender_id):
        logger.warning("Access denied for sender {} on channel {}", sender_id, self.name)
        return

    msg = InboundMessage(
        channel=self.name,
        sender_id=str(sender_id),
        chat_id=str(chat_id),
        content=content,
        media=media or [],
        metadata=meta,
        session_key_override=session_key,
    )
    await self.bus.publish_inbound(msg)
```

### Permission Model

```python
def is_allowed(self, sender_id: str) -> bool:
    """Empty list = deny all; "*" = allow all."""
    allow_list = getattr(self.config, "allow_from", [])
    if not allow_list:
        logger.warning("{}: allow_from is empty — all access denied", self.name)
        return False
    if "*" in allow_list:
        return True
    return str(sender_id) in allow_list
```

### Streaming Support

```python
@property
def supports_streaming(self) -> bool:
    """True when config enables streaming AND this subclass implements send_delta."""
    cfg = self.config
    streaming = cfg.get("streaming", False) if isinstance(cfg, dict) else getattr(cfg, "streaming", False)
    return bool(streaming) and type(self).send_delta is not BaseChannel.send_delta
```

### WhatsApp Bridge Architecture

WhatsApp uses a separate TypeScript bridge (`bridge/`) because WhatsApp Web requires the Baileys library:

```
WhatsApp (Baileys/TypeScript)
        |
        v (WebSocket)
Python Backend (nanobot)
        |
        v
ChannelManager --> WhatsApp channel
```

The bridge is bundled into the Python wheel via `pyproject.toml`:
```toml
[tool.hatch.build.targets.wheel.force-include]
"bridge" = "nanobot/bridge"
```

### Channel-Specific Implementations

**Telegram** (`telegram.py`):
- Uses `python-telegram-bot` library
- Implements markdown-to-HTML conversion for messages
- Supports reply parameters, reactions
- Table rendering for markdown tables

**Discord** (`discord.py`):
- Uses native Discord Gateway WebSocket protocol
- Custom heartbeat implementation
- HTTP API for sending messages
- Attachment handling (20MB limit)

**Feishu** (`feishu.py` - largest at 49KB):
- Uses `lark-oapi` SDK
- Extensive event handling
- Card message support

---

## Feature 3: Multi-Provider LLM Support

### Overview

Nanobot supports 20+ LLM providers through a unified provider abstraction. The architecture uses a `ProviderSpec` registry that describes each provider's metadata, with backend implementations handling the actual API calls.

### Provider Registry (providers/registry.py)

The registry is a tuple of `ProviderSpec` dataclasses:

```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str                           # Config field name
    keywords: tuple[str, ...]          # Model-name keywords for matching
    env_key: str                        # Env var for API key
    display_name: str                   # Shown in status
    backend: str                        # "openai_compat" | "anthropic" | "azure_openai" | "openai_codex"
    is_gateway: bool                    # Routes any model (OpenRouter, AiHubMix)
    is_local: bool                      # Local deployment
    detect_by_key_prefix: str           # Match api_key prefix
    detect_by_base_keyword: str         # Match api_base URL substring
    default_api_base: str               # Default base URL
    strip_model_prefix: bool            # Strip "provider/" before sending
    model_overrides: tuple[...]         # Per-model param overrides
    is_oauth: bool                      # OAuth-based (no API key)
    is_direct: bool                     # Skip API-key validation
    supports_prompt_caching: bool       # Anthropic prompt caching
```

### Provider List

| Provider | Backend | Default API Base | Notes |
|----------|---------|------------------|-------|
| Custom | openai_compat | - | Direct endpoint |
| Azure OpenAI | azure_openai | - | Direct |
| OpenRouter | openai_compat | openrouter.ai/api/v1 | Gateway, key prefix sk-or- |
| AiHubMix | openai_compat | aihubmix.com/v1 | Gateway, strip prefix |
| SiliconFlow | openai_compat | api.siliconflow.cn/v1 | Gateway |
| VolcEngine | openai_compat | ark.cn-beijing.volces.com/api/v3 | Gateway |
| BytePlus | openai_compat | ark.ap-southeast.bytepluses.com | Gateway |
| Anthropic | anthropic | - | Native SDK |
| OpenAI | openai_compat | - | SDK default |
| OpenAI Codex | openai_codex | chatgpt.com/backend-api | OAuth |
| GitHub Copilot | openai_compat | api.githubcopilot.com | OAuth |
| DeepSeek | openai_compat | api.deepseek.com | - |
| Gemini | openai_compat | generativelanguage.googleapis.com/v1beta/openai/ | - |
| Zhipu | openai_compat | open.bigmodel.cn/api/paas/v4 | - |
| DashScope | openai_compat | dashscope.aliyuncs.com/compatible-mode/v1 | - |
| Moonshot | openai_compat | api.moonshot.ai/v1 | - |
| MiniMax | openai_compat | api.minimax.io/v1 | - |
| Mistral | openai_compat | api.mistral.ai/v1 | - |
| StepFun | openai_compat | api.stepfun.com/v1 | - |
| vLLM | openai_compat | - | Local |
| Ollama | openai_compat | localhost:11434/v1 | Local |
| OpenVINO Model Server | openai_compat | localhost:8000/v3 | Local |
| Groq | openai_compat | api.groq.com/openai/v1 | Auxiliary |

### Provider Detection Flow

1. **By config field**: `find_by_name("dashscope")` looks up `ProviderSpec` by name
2. **By api_key prefix**: OpenRouter detected by `sk-or-` prefix
3. **By api_base keyword**: AiHubMix detected by "aihubmix" in URL

### Backend Implementations

#### OpenAICompatProvider (providers/openai_compat_provider.py)

Handles all OpenAI-compatible APIs. Key features:

**Tool call ID normalization**:
```python
@staticmethod
def _normalize_tool_call_id(tool_call_id: Any) -> Any:
    """Normalize to a provider-safe 9-char alphanumeric form."""
    if not isinstance(tool_call_id, str):
        return tool_call_id
    if len(tool_call_id) == 9 and tool_call_id.isalnum():
        return tool_call_id
    return hashlib.sha1(tool_call_id.encode()).hexdigest()[:9]
```

**Prompt caching** (for OpenRouter, AiHubMix):
```python
@staticmethod
def _apply_cache_control(
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None,
) -> tuple[list[dict[str, Any]], list[dict[str, Any]] | None]:
    cache_marker = {"type": "ephemeral"}
    # Mark system message, second-to-last message, and last tool
```

**Model prefix stripping** (for gateways that don't understand org/model format):
```python
if spec and spec.strip_model_prefix:
    model_name = model_name.split("/")[-1]
```

**Model overrides** (e.g., Moonshot K2.5 requires temperature >= 1.0):
```python
for pattern, overrides in spec.model_overrides:
    if pattern in model_lower:
        kwargs.update(overrides)
```

#### AnthropicProvider (providers/anthropic_provider.py)

Native SDK integration with format conversion:

**Message format conversion** (OpenAI -> Anthropic Messages API):
```python
def _convert_messages(self, messages) -> tuple[str | list[dict], list[dict]]:
    # System messages extracted
    # User/assistant/tool roles mapped
    # Tool results become tool_result blocks
```

**Image conversion**:
```python
@staticmethod
def _convert_image_block(block: dict[str, Any]) -> dict[str, Any] | None:
    # data:image URLs -> base64 Anthropic format
    # Remote URLs -> url Anthropic format
```

**Extended thinking support**:
```python
if thinking_enabled:
    budget_map = {"low": 1024, "medium": 4096, "high": max(8192, max_tokens)}
    kwargs["thinking"] = {"type": "enabled", "budget_tokens": budget}
    kwargs["temperature"] = 1.0
```

### LLMProvider Base Class (providers/base.py)

Abstract base defining the provider interface:

```python
class LLMProvider(ABC):
    _CHAT_RETRY_DELAYS = (1, 2, 4)
    _TRANSIENT_ERROR_MARKERS = (
        "429", "rate limit", "500", "502", "503", "504",
        "overloaded", "timeout", "timed out", "connection",
        "server error", "temporarily unavailable",
    )

    @abstractmethod
    async def chat(...) -> LLMResponse: pass

    async def chat_with_retry(...) -> LLMResponse:
        """Call chat() with retry on transient failures."""
        for attempt, delay in enumerate(self._CHAT_RETRY_DELAYS):
            response = await self._safe_chat(**kw)
            if response.finish_reason != "error":
                return response
            if not self._is_transient_error(response.content):
                return response
            await asyncio.sleep(delay)

    async def chat_stream_with_retry(...) -> LLMResponse:
        """Streaming version with same retry logic"""
```

### LLMResponse Dataclass

```python
@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list[ToolCallRequest] = field(default_factory=list)
    finish_reason: str = "stop"
    usage: dict[str, int] = field(default_factory=dict)
    reasoning_content: str | None = None  # Kimi, DeepSeek-R1
    thinking_blocks: list[dict] | None = None  # Anthropic extended thinking

    @property
    def has_tool_calls(self) -> bool:
        return len(self.tool_calls) > 0
```

### ToolCallRequest Dataclass

```python
@dataclass
class ToolCallRequest:
    id: str
    name: str
    arguments: dict[str, Any]
    extra_content: dict[str, Any] | None = None
    provider_specific_fields: dict[str, Any] | None = None
    function_provider_specific_fields: dict[str, Any] | None = None

    def to_openai_tool_call(self) -> dict[str, Any]:
        """Serialize to OpenAI-style tool_call payload"""
```

### Message Sanitization

Both providers sanitize messages before sending:

```python
@staticmethod
def _sanitize_request_messages(
    messages: list[dict[str, Any]],
    allowed_keys: frozenset[str],
) -> list[dict[str, Any]]:
    """Keep only provider-safe message keys"""
    sanitized = []
    for msg in messages:
        clean = {k: v for k, v in msg.items() if k in allowed_keys}
        if clean.get("role") == "assistant" and "content" not in clean:
            clean["content"] = None
        sanitized.append(clean)
    return sanitized
```

### Error Handling

```python
@staticmethod
def _handle_error(e: Exception) -> LLMResponse:
    body = getattr(e, "doc", None) or getattr(getattr(e, "response", None), "text", None)
    msg = f"Error: {body.strip()[:500]}" if body and body.strip() else f"Error calling LLM: {e}"
    return LLMResponse(content=msg, finish_reason="error")
```

---

## Notable Patterns and Technical Debt

### Clever Solutions

1. **Zero-import channel discovery**: `pkgutil.iter_modules` allows discovering channels without importing them
2. **Tool context isolation**: `_set_tool_context()` called right before tool execution prevents cross-session contamination
3. **Concurrent tool execution**: `asyncio.gather` with `return_exceptions=True` for parallel tool calls
4. **Streaming think-block filtering**: Incremental stripping of `<think>` blocks during streaming
5. **Tool call ID normalization**: 9-char alphanumeric IDs compatible with all providers
6. **Message sanitization pipeline**: Strips non-standard keys before sending to providers
7. **Prompt caching abstraction**: `_apply_cache_control()` abstracts caching across providers

### Technical Debt / Concerns

1. **No provider failover**: If configured provider fails, no automatic fallback to another provider
2. **Global env var side effects**: `_setup_env()` modifies `os.environ` which could affect concurrent sessions
3. **Session locks in-memory only**: `self._session_locks` is in-memory, doesn't work across multiple nanobot instances
4. **Hardcoded retry delays**: `_SEND_RETRY_DELAYS = (1, 2, 4)` and `_CHAT_RETRY_DELAYS = (1, 2, 4)` not configurable
5. **Groq primarily for transcription**: Groq is listed as LLM provider but main use appears to be Whisper transcription
6. **OAuth providers incomplete**: OpenAI Codex and GitHub Copilot have `is_oauth=True` but OAuth flow implementation not visible in core provider files
7. **Image handling**: Image conversion between providers is one-way (OpenAI format -> Anthropic), no reverse conversion

---

## Code vs Documentation Discrepancies

No significant discrepancies found. The implementation matches the feature index:

1. **Agent Core Loop**: All components present (`loop.py`, `context.py`, `memory.py`, `skills.py`, `subagent.py`)
2. **Multi-Channel**: All 12 channels listed are implemented in `nanobot/channels/`
3. **Multi-Provider**: All 20+ providers defined in `providers/registry.py`

---

## Summary

| Feature | Implementation Quality | Key Strength | Key Weakness |
|---------|----------------------|--------------|--------------|
| Agent Core Loop | High | Sophisticated streaming, concurrent tool exec, session isolation | No provider failover |
| Multi-Channel | High | Plugin architecture, 12 platforms, retry policy | WhatsApp requires separate TS bridge |
| Multi-Provider | High | Unified interface, 20+ providers, prompt caching | OAuth flows incomplete, env var side effects |
