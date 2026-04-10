# Feature Deep-Dive: Batch 1

## Feature 1: Self-Improving Agent with Learning Loop

### Core Implementation Files
- `honcho_integration/session.py` - HonchoSessionManager (992 lines)
- `honcho_integration/client.py` - HonchoClientConfig (437 lines)
- `agent/skill_commands.py` - Shared slash command helpers (283 lines)
- `tools/skills_tool.py` - Skills listing and viewing (600+ lines)
- `tools/skill_manager_tool.py` - Skill management operations

### How the Feature Works

**1. Skills System (Procedural Memory)**

Skills are markdown files with YAML frontmatter stored in `~/.hermes/skills/`. The system implements progressive disclosure:

```
skills/
├── my-skill/
│   ├── SKILL.md           # Main instructions (required)
│   ├── references/        # Supporting documentation
│   ├── templates/         # Output templates
│   └── assets/            # Supplementary files
```

SKILL.md format follows agentskills.io standard with metadata:
```yaml
---
name: skill-name              # Required, max 64 chars
description: Brief description # Required, max 1024 chars
platforms: [macos]           # Optional OS restriction
prerequisites:
  env_vars: [API_KEY]       # Required environment variables
  commands: [curl, jq]       # Required commands
---
```

The `skill_view()` function loads skill content on-demand, and `scan_skill_commands()` builds a mapping of `/command-name` slash commands.

**2. Honcho Integration (Persistent Memory)**

The `HonchoSessionManager` provides cross-session memory via Honcho's AI-native memory system:

```python
class HonchoSessionManager:
    def __init__(self, honcho: Honcho, context_tokens: int | None, config: HonchoClientConfig):
        # Write frequency: "async" (background thread), "turn" (sync per turn),
        # "session" (flush on session end), or int (every N turns)
        self._write_frequency = write_frequency
        self._async_queue = queue.Queue()
        self._async_thread = threading.Thread(target=self._async_writer_loop, ...)
```

Key capabilities:
- **Dialectic queries**: LLM-powered queries about users (`dialectic_query()`)
- **Context prefetching**: Background fetching of user/AI representations
- **Session migration**: Upload local history to Honcho when activating mid-conversation
- **Memory file migration**: Transfer MEMORY.md, USER.md, SOUL.md to Honcho

**3. Session-Based Learning Loop**

```python
def create_conclusion(self, session_key: str, content: str) -> bool:
    """Write a conclusion about the user back to Honcho.
    Conclusions feed into the user's peer card and representation."""
```

The system observes user behavior across sessions and creates conclusions that inform future interactions.

### Notable Code Patterns

**1. Async Write Queue Pattern**
```python
def _async_writer_loop(self) -> None:
    """Background daemon thread: drains the async write queue."""
    while True:
        item = self._async_queue.get(timeout=5)
        if item is _ASYNC_SHUTDOWN:
            break
        try:
            success = self._flush_session(item)
        except Exception as e:
            logger.warning("Honcho async write failed, retrying once: %s", e)
            _time.sleep(2)
            retry_success = self._flush_session(item)
            if not retry_success:
                logger.error("Honcho async write retry failed, dropping batch")
```

**2. Dynamic Reasoning Level Selection**
```python
def _dynamic_reasoning_level(self, query: str) -> str:
    """Pick a reasoning level based on message complexity.
    < 120 chars  → default (typically "low")
    120–400 chars → one level above default
    > 400 chars  → two levels above default"""
    levels = self._REASONING_LEVELS  # ("minimal", "low", "medium", "high", "max")
    default_idx = levels.index(self._dialectic_reasoning_level)
    bump = 0 if len(query) < 120 else 1 if len(query) < 400 else 2
    idx = min(default_idx + bump, 3)  # Cap at "high" for auto-selection
    return levels[idx]
```

**3. Skill Command Scanning**
```python
def scan_skill_commands() -> Dict[str, Dict[str, Any]]:
    """Scan ~/.hermes/skills/ and return mapping of /command -> skill info."""
    for skill_md in SKILLS_DIR.rglob("SKILL.md"):
        if any(part in ('.git', '.github', '.hub') for part in skill_md.parts):
            continue
        frontmatter, body = _parse_frontmatter(content)
        if not skill_matches_platform(frontmatter):
            continue
        # ... extract name, description, build command mapping
```

### Technical Debt or Concerns

1. **Circular Import Risk**: `honcho_integration/session.py` imports from `honcho` at type-check time only, but `get_honcho_client()` does runtime import inside function
2. **Honcho Dependency**: The feature requires external `honcho-ai` service - not fully self-contained
3. **Session Key Sanitization**: Uses regex `[a-zA-Z0-9_-]` but session keys may contain colons (`channel:chat_id`)
4. **Memory Migration Complexity**: `_format_migration_transcript()` uses XML wrapping which is non-standard

---

## Feature 2: Multi-Platform Messaging Gateway (10+ platforms)

### Core Implementation Files
- `gateway/run.py` - GatewayRunner (main controller, 262KB)
- `gateway/platforms/base.py` - BasePlatformAdapter (1342 lines)
- `gateway/platforms/telegram.py` - TelegramAdapter (78KB)
- `gateway/platforms/discord.py` - DiscordAdapter (94KB)
- `gateway/stream_consumer.py` - GatewayStreamConsumer (203 lines)
- `gateway/session.py` - SessionStore
- `gateway/config.py` - Platform/GatewayConfig

### How the Feature Works

**1. Architecture Pattern: Base Adapter**

All platform adapters inherit from `BasePlatformAdapter` which defines the interface:

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool:
        """Connect to the platform and start receiving messages."""

    @abstractmethod
    async def disconnect(self) -> None:
        """Disconnect from the platform."""

    @abstractmethod
    async def send(self, chat_id: str, content: str, reply_to: Optional[str] = None,
                  metadata: Optional[Dict[str, Any]] = None) -> SendResult:
        """Send a message to a chat."""

    @abstractmethod
    async def get_chat_info(self, chat_id: str) -> Dict[str, Any]:
        """Get information about a chat/channel."""
```

**2. Message Normalization**

All incoming messages are normalized to `MessageEvent`:

```python
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
    raw_message: Any = None
    message_id: Optional[str] = None
    media_urls: List[str] = field(default_factory=list)
    media_types: List[str] = field(default_factory=list)
    reply_to_message_id: Optional[str] = None
    reply_to_text: Optional[str] = None
    auto_skill: Optional[str] = None
    timestamp: datetime = field(default_factory=datetime.now)
```

Supported message types: TEXT, LOCATION, PHOTO, VIDEO, AUDIO, VOICE, DOCUMENT, STICKER, COMMAND

**3. Streaming: Progressive Message Editing**

The `GatewayStreamConsumer` implements real-time streaming via edit transport:

```python
class GatewayStreamConsumer:
    """Bridges sync agent callbacks to async platform delivery.
    Uses edit transport (send initial message, then editMessageText)."""

    def __init__(self, adapter, chat_id: str, config: StreamConsumerConfig, metadata):
        self._queue: queue.Queue = queue.Queue()
        self._accumulated = ""
        self._message_id: Optional[str] = None
        self._edit_supported = True  # Disabled on first edit failure

    def on_delta(self, text: str) -> None:
        """Thread-safe callback — called from agent's worker thread."""
        if text:
            self._queue.put(text)

    async def run(self) -> None:
        """Async task that drains queue and edits platform message."""
        while True:
            # Drain all available items
            while True:
                item = self._queue.get_nowait()
                if item is _DONE:
                    got_done = True
                    break
                self._accumulated += item

            # Decide whether to flush
            should_edit = (
                got_done
                or (elapsed >= self.cfg.edit_interval and len(self._accumulated) > 0)
                or len(self._accumulated) >= self.cfg.buffer_threshold
            )

            if should_edit and self._accumulated:
                display_text = self._accumulated + (self.cfg.cursor if not got_done else "")
                await self._send_or_edit(display_text)
```

**4. Supported Platforms**

| Platform | File | Key Features |
|----------|------|--------------|
| Telegram | `telegram.py` (78KB) | MarkdownV2 formatting, forum topics, media batching |
| Discord | `discord.py` (94KB) | Slash commands, threads, Opus audio |
| Slack | `slack.py` | Thread support |
| WhatsApp | `whatsapp.py` | - |
| Email | `email.py` | SMTP/IMAP |
| Signal | `signal.py` | - |
| DingTalk | `dingtalk.py` | - |
| Matrix | `matrix.py` | - |
| Mattermost | `mattermost.py` | - |
| SMS | `sms.py` | - |
| Home Assistant | `homeassistant.py` | - |
| Webhook | `webhook.py` | - |

**5. Media Caching**

Images, audio, and documents from platforms are cached locally for vision tool access:

```python
IMAGE_CACHE_DIR = get_hermes_home() / "image_cache"

def cache_image_from_bytes(data: bytes, ext: str = ".jpg") -> str:
    filename = f"img_{uuid.uuid4().hex[:12]}{ext}"
    filepath = IMAGE_CACHE_DIR / filename
    filepath.write_bytes(data)
    return str(filepath)
```

**6. Interrupt Support**

```python
async def handle_message(self, event: MessageEvent) -> None:
    """Process incoming message via background task for interrupt support."""
    if session_key in self._active_sessions:
        # Photo bursts: queue without interrupting
        if event.message_type == MessageType.PHOTO:
            self._pending_messages[session_key] = event
            return
        # Default: interrupt running agent
        self._pending_messages[session_key] = event
        self._active_sessions[session_key].set()  # Signal interrupt
        return

    # Spawn background task to process message
    task = asyncio.create_task(self._process_message_background(event, session_key))
    self._background_tasks.add(task)
```

**7. Gateway Lifecycle Management**

```python
class GatewayRunner:
    def __init__(self, config: Optional[GatewayConfig] = None):
        self.adapters: Dict[Platform, BasePlatformAdapter] = {}
        self.session_store = SessionStore(...)
        self.delivery_router = DeliveryRouter(self.config)
        # AIAgent cache per session for prompt caching
        self._agent_cache: Dict[str, tuple] = {}
        # Persistent Honcho managers keyed by gateway session key
        self._honcho_managers: Dict[str, Any] = {}
```

### Notable Code Patterns

**1. SSL Certificate Auto-Detection**
```python
def _ensure_ssl_certs() -> None:
    """Set SSL_CERT_FILE if system doesn't expose CA certs to Python."""
    # 1. Python's compiled-in defaults
    # 2. certifi (ships Mozilla bundle)
    # 3. Common distro / macOS locations
    for candidate in (
        "/etc/ssl/certs/ca-certificates.crt",  # Debian/Ubuntu
        "/etc/pki/tls/certs/ca-bundle.crt",    # RHEL/CentOS
        "/etc/ssl/cert.pem",                   # Alpine / macOS
        ...
    ):
        if os.path.exists(candidate):
            os.environ["SSL_CERT_FILE"] = candidate
            return
```

**2. MarkdownV2 Escaping (Telegram)**
```python
_MDV2_ESCAPE_RE = re.compile(r'([_*\[\]()~`>#\+\-=|{}.!\\])')

def _escape_mdv2(text: str) -> str:
    """Escape Telegram MarkdownV2 special characters."""
    return _MDV2_ESCAPE_RE.sub(r'\\\1', text)
```

**3. Message Truncation with Code Block Preservation**
```python
@staticmethod
def truncate_message(content: str, max_length: int = 4096) -> List[str]:
    """Split long messages, preserving code block boundaries."""
    # If split falls inside triple-backtick block, close fence at end
    # and reopen (with language tag) at start of next chunk
    while remaining:
        # Find safe split point (prefer newlines, then spaces)
        # Avoid splitting inside inline code span
        if backtick_count % 2 == 1:
            # Find last unescaped backtick and split before it
```

### Technical Debt or Concerns

1. **Large File Sizes**: `gateway/run.py` (262KB), `telegram.py` (78KB), `discord.py` (94KB) suggest complex single-file implementations
2. **Polling Conflict Handling**: Telegram's `_looks_like_polling_conflict()` checks class name and text content - fragile
3. **Platform-Specific Special Cases**: Extensive platform-specific handling in base class (`_is_animation_url`, `_get_human_delay`)
4. **Edit Transport Limitation**: Not all platforms support message editing (Signal, Email, Home Assistant disable streaming)
5. **Security**: Allowlist configuration warning on startup but no enforcement mechanism visible

---

## Feature 3: Terminal TUI with Streaming Output

### Core Implementation Files
- `hermes_cli/curses_ui.py` - Curses checklist UI (141 lines)
- `agent/display.py` - CLI presentation, spinner, tool preview (800+ lines)
- `tools/terminal_tool.py` - Terminal execution backend (500+ lines)
- `hermes_cli/skin_engine.py` - Theming/skin system
- `run_agent.py` - Agent with streaming API calls

### How the Feature Works

**1. Curses-Based Interactive UI**

```python
def curses_checklist(title: str, items: List[str], selected: Set[int], *,
                     cancel_returns: Set[int] | None = None) -> Set[int]:
    """Curses multi-select checklist with keyboard navigation."""
    def _draw(stdscr):
        curses.curs_set(0)
        if curses.has_colors():
            curses.start_color()
            curses.init_pair(1, curses.COLOR_GREEN, -1)
            curses.init_pair(2, curses.COLOR_YELLOW, -1)

        while True:
            stdscr.clear()
            max_y, max_x = stdscr.getmaxyx()

            # Header with navigation hints
            stdscr.addnstr(1, 0,
                "  ↑↓ navigate  SPACE toggle  ENTER confirm  ESC cancel",
                max_x - 1, curses.A_DIM)

            # Scrollable item list
            for draw_i, i in enumerate(range(scroll_offset, visible_rows)):
                check = "✓" if i in chosen else " "
                arrow = "→" if i == cursor else " "
                line = f" {arrow} [{check}] {items[i]}"
                # Bold + color for selected item
```

Fallback for non-curses terminals:
```python
def _numbered_fallback(title, items, selected, cancel_returns) -> Set[int]:
    """Text-based toggle fallback for terminals without curses."""
    chosen = set(selected)
    print(color(f"\n  {title}", Colors.YELLOW))
    while True:
        for i, label in enumerate(items):
            marker = color("[✓]", Colors.GREEN) if i in chosen else "[ ]"
            print(f"  {marker} {i + 1:>2}. {label}")
        val = input(color("  Toggle # (or Enter to confirm): ", Colors.DIM)).strip()
        # ...
```

**2. Agent Streaming API**

```python
def _interruptible_streaming_api_call(self, api_kwargs: dict, *,
                                      on_first_delta: callable = None):
    """Streaming variant with real-time token delivery."""
    # Uses Anthropic SDK streaming for anthropic_messages mode
    # Uses OpenAI streaming for chat_completions mode
    # Uses _run_codex_stream for codex_responses mode

    def _fire_stream_delta(text: str) -> None:
        """Fire all registered stream delta callbacks (display + TTS)."""
        if self._stream_needs_break and text:
            self.stream_delta_callback("\n\n")
            self._stream_needs_break = False
        if text:
            self.stream_delta_callback(text)

    with self._anthropic_client.messages.stream(**api_kwargs) as stream:
        for event in stream:
            if self._interrupt_requested:
                break
            # Handle text deltas
            # Handle tool calls (suppress text streaming)
            # Handle reasoning content
```

**3. Tool Execution with Verbose Output**

```python
def _vprint(self, *args, force: bool = False, **kwargs):
    """Verbose print — suppressed when actively streaming tokens."""
    # Pass force=True for error/warning messages during streaming
    if self._executing_tools:
        # Allow printing during tool execution even with stream consumers
        pass
```

**4. Thinking Spinner**

```python
class KawaiiSpinner:
    """Animated thinking spinner with skin-aware faces."""
    KAWAII_THINKING = ["(・ˍ・)", "(´・ω・`)", "(´〜`)", "ヽ(ˋ▽ˊ)ノ", "(๑•̀ㅂ•́)و✧"]
    THINKING_VERBS = ["thinking", "pondering", "musing", "contemplating", "deliberating"]

    def _update(self):
        """Update display with current face and spin position."""
        # Cycles through frames: face → verb → ... → elapsed time
```

**5. Terminal Tool with Multiple Backends**

```python
def terminal_tool(command: str, background: bool = False, ...) -> str:
    """Execute commands in local, Docker, Modal, SSH, Singularity, Daytona."""
    backend = os.getenv("TERMINAL_ENV", "local")

    if backend == "local":
        return _run_local(command, ...)
    elif backend == "docker":
        return _run_docker(command, ...)
    elif backend == "modal":
        return _run_modal(command, ...)
    # ... other backends
```

**6. Dangerous Command Approval**

```python
def _check_dangerous_command(command: str, env_type: str) -> dict:
    """Check if command requires approval before execution."""
    dangerous_patterns = [
        (r'sudo\s+', 'Privilege escalation'),
        (r'drop\s+database', 'Database deletion'),
        (r'rm\s+-rf\s+/', 'Recursive force deletion'),
        (r'>\s*/dev/', 'Device writing'),
    ]
    # Returns {"dangerous": bool, "reason": str, "pattern_key": str}
```

### Notable Code Patterns

**1. Skin/Theme System**
```python
class Skin:
    """Configurable appearance for CLI output."""
    def get_spinner_list(self, key: str) -> list:
        """Get spinner face list from active skin."""
    def get_tool_prefix(self) -> str:
        """Get tool output prefix character (e.g., "┊")."""
```

**2. Stream Delta Callback Architecture**
```python
# In AIAgent:
self.stream_delta_callback = None  # Set by consumer (CLI or gateway)
self._stream_callback = None       # For TTS pipeline

def _fire_stream_delta(self, text: str) -> None:
    """Fire all registered stream delta callbacks."""
    if self._stream_needs_break and text:
        self.stream_delta_callback("\n\n")  # Paragraph break
        self._stream_needs_break = False
    if text:
        self.stream_delta_callback(text)
    if self._stream_callback:
        self._stream_callback(text)
```

**3. Tool Preview Building**
```python
def build_tool_preview(tool_name: str, args: dict, max_len: int = 40) -> str | None:
    """Build short preview of tool call's primary argument."""
    primary_args = {
        "terminal": "command", "web_search": "query", "web_extract": "urls",
        "read_file": "path", "write_file": "path", "patch": "path",
        "search_files": "pattern", "browser_navigate": "url",
        # ...
    }
    key = primary_args.get(tool_name)
    if not key or key not in args:
        return None
    value = args[key]
    if len(preview) > max_len:
        preview = preview[:max_len - 3] + "..."
    return preview
```

### Technical Debt or Concerns

1. **TUI is Minimal**: `curses_ui.py` is only 141 lines - the "Terminal TUI" is actually just a checklist component, not a full terminal interface
2. **prompt_toolkit Not Used for Main TUI**: Despite references to prompt_toolkit in callbacks and terminal_tool, the actual REPL/input handling appears to be in `cli.py` and `hermes_cli/main.py`
3. **KawaiiSpinner Blocking**: The spinner runs in a background thread but could conflict with streaming output
4. **Stream Callback Race Conditions**: Multiple consumers (display + TTS) fire from same `_fire_stream_delta()` with minimal synchronization
5. **Terminal Backend Complexity**: 6 different backends (local, Docker, SSH, Daytona, Singularity, Modal) each with different failure modes

---

## Summary Table

| Feature | Core Files | Architecture | Key Innovation |
|---------|-----------|--------------|----------------|
| Self-Improving Agent | honcho_integration/session.py, tools/skills_tool.py | Skills + Honcho memory | Async write queue, dynamic reasoning levels |
| Multi-Platform Gateway | gateway/run.py, platforms/base.py | Base adapter pattern | Edit-transport streaming, media caching |
| Terminal TUI | curses_ui.py, display.py, terminal_tool.py | Multiple backends | Curses checklist, skin system, streaming callbacks |
