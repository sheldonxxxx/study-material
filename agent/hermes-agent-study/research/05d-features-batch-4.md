# Feature Deep-Dive: Batch 4 (Features 10-12)

## Feature 10: Persistent Memory & User Modeling

### Core Implementation Files
- `tools/memory_tool.py` - File-backed curated memory with bounded stores
- `tools/session_search_tool.py` - FTS5 full-text search + LLM summarization of past sessions
- `honcho_integration/client.py` - Honcho client configuration and session management
- `honcho_integration/session.py` - Honcho-based session manager for user modeling
- `hermes_state.py` - SQLite session store with FTS5 (lines 87-90, 744-793)

### How It Works

#### Local File-Backed Memory (MEMORY.md / USER.md)

The `MemoryStore` class maintains two bounded character-limited stores:

```
~/.hermes/memories/
├── MEMORY.md   # Agent's personal notes (2200 char limit default)
└── USER.md     # User profile and preferences (1375 char limit default)
```

**Key design: frozen snapshot pattern for cache stability**

The system prompt is frozen at session start and never modified mid-session. Mid-session writes persist to disk immediately but do not affect the system prompt. This preserves the Anthropic prompt caching prefix, dramatically reducing costs on long conversations.

```python
# Snapshot captured at load time
self._system_prompt_snapshot = {
    "memory": self._render_block("memory", self.memory_entries),
    "user": self._render_block("user", self.user_entries),
}

# Mid-session writes update files but NOT the snapshot
# Only refreshed on next session start
```

**Entry management:**
- Uses `§` (section sign) as delimiter between entries
- Single `memory` tool with `action` parameter: `add`, `replace`, `remove`
- Substring matching for replace/remove (not full text or IDs)
- Exact duplicate rejection
- Character limit enforcement with usage reporting

**Atomic file writes via temp rename:**
```python
fd, tmp_path = tempfile.mkstemp(dir=path.parent, suffix=".tmp", prefix=".mem_")
with os.fdopen(fd, "w", encoding="utf-8") as f:
    f.write(content)
    f.flush()
    os.fsync(f.fileno())
os.replace(tmp_path, str(path))  # Atomic on same filesystem
```
This prevents readers from seeing truncated files during concurrent writes.

**File locking for read-modify-write safety:**
Uses a separate `.lock` file so memory files can still be atomically replaced. Lock is acquired for the duration of `add`/`replace`/`remove` operations.

**Prompt injection scanning:**
```python
_MEMORY_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    # ... SSH backdoor, exfil via curl/wget patterns
]
_INVISIBLE_CHARS = {'\u200b', '\u200c', '\u200d', '\ufeff', ...}  # Zero-width chars
```
All memory content is scanned before injection into the system prompt.

#### FTS5 Session Search (session_search_tool.py)

Searches past session transcripts stored in SQLite via FTS5:

```
hermes_state.py creates:
  messages_fts FTS5 virtual table linked to messages table
```

**Flow:**
1. FTS5 MATCH query finds matching messages ranked by BM25 relevance
2. Groups by session, deduplicates via delegation parent resolution
3. Truncates each session to ~100k chars centered around matches
4. Sends to Gemini Flash with summarization prompt
5. Returns per-session summaries with timestamps and sources

**Delegation chain resolution:**
```python
def _resolve_to_parent(session_id: str) -> str:
    """Walk delegation chain to find the root parent session ID."""
    visited = set()
    sid = session_id
    while sid and sid not in visited:
        visited.add(sid)
        session = db.get_session(sid)
        parent = session.get("parent_session_id")
        sid = parent if parent else None
    return sid
```
This prevents duplicate results when compression or delegation creates child sessions.

**Truncation around matches:**
```python
def _truncate_around_matches(full_text: str, query: str, max_chars: int = 100_000):
    # Find first occurrence of query term
    # Center window around it with half before, half after
    # Add "...[earlier conversation truncated]..." prefix if needed
```

**Two modes:**
- **Recent sessions** (no query): Returns metadata (titles, previews, timestamps) - zero LLM cost
- **Keyword search**: FTS5 + LLM summarization, limited to 5 sessions

#### Honcho Integration (User Modeling)

The `HonchoSessionManager` wraps Honcho's AI-native memory system:

**Config resolution order:**
1. `$HERMES_HOME/honcho.json` (instance-local)
2. `~/.honcho/config.json` (global)
3. Environment variables

**Session naming strategies:**
- `per-session`: Inherit Hermes session_id
- `per-repo`: Git repo root directory name
- `per-directory`: Working directory basename
- `global`: Single session across all directories

**Peer observation model:**
- Both user and AI peers observe each other (`observe_me=True, observe_others=True`)
- This enables identity formation over time

**Write frequency modes:**
```python
"async"   # Background thread, zero blocking
"turn"    # Flush synchronously every turn
"session" # Defer until explicit flush_all()
N (int)   # Flush every N turns
```

**Dialectic queries:**
Honcho's LLM reasoning about a peer's representation. Dynamic reasoning level based on query complexity:
```python
# < 120 chars → default (low)
# 120-400 chars → one level above default
# > 400 chars → two levels above default
# Cap at "high" for auto-selection
```

**Prefetch pattern for non-blocking context:**
```python
def prefetch_context(self, session_key: str, user_message: str = None):
    def _run():
        result = self.get_prefetch_context(session_key, user_message)
        self.set_context_result(session_key, result)
    t = threading.Thread(target=_run, daemon=True)
    t.start()
# Result consumed next turn via pop_context_result()
```

**Memory migration on Honcho activation:**
- `migrate_local_history()`: Uploads prior conversation as XML transcript
- `migrate_memory_files()`: Uploads MEMORY.md, USER.md, SOUL.md as structured files

### Notable Code Patterns

1. **Frozen snapshot for cache stability**: Mid-session memory writes do not invalidate the prefix cache
2. **Atomic rename for file writes**: Temp file + `os.replace()` prevents partial reads
3. **Dedicated lock file**: Memory files themselves remain atomic-readable while operations are locked
4. **FTS5 query sanitization**: Strips FTS5-special chars (`+`, `-`, `()`, `"`) while preserving quoted phrases
5. **Thread-safe background loops**: Both Honcho async writer and MCP event loop use dedicated daemon threads
6. **Context prefetching**: Dialectic and context results prefetched non-blocking, consumed next turn

### Technical Debt / Concerns

1. **Character limits vs token limits**: Memory uses character counts (`len()`) rather than token counts. Since different models have different tokenizers, a 2200-char limit may be significantly different in actual tokens depending on content. The comment says "model-independent" but this is a rough approximation.

2. **Honcho is a separate service**: Requires external `honcho-ai` package and Honcho API access. If Honcho is unavailable or rate-limited, the graceful degradation is just logging warnings.

3. **Memory migration is one-way**: When Honcho activates mid-conversation, existing local memories are uploaded to Honcho but stay in local files too. No sync mechanism if both are modified.

4. **Dialectic char cap**: Hard truncation at `_dialectic_max_chars` with word-boundary split (`rsplit(" ", 1)[0]`) can cut sentences mid-word awkwardly.

---

## Feature 11: MCP (Model Context Protocol) Integration

### Core Implementation Files
- `tools/mcp_tool.py` (~1860 lines) - Main MCP client implementation
- `hermes_cli/mcp_config.py` - CLI commands for `hermes mcp` subcommands

### How It Works

#### Architecture Overview

```
Hermes Agent
    │
    ├── MCP event loop (daemon thread)
    │       │
    │       ├── MCPServerTask (stdio or HTTP)
    │       │       └── ClientSession (mcp SDK)
    │       ├── MCPServerTask (HTTP)
    │       │       └── ClientSession (mcp SDK)
    │       └── ...
    │
    └── Tool Registry
            ├── mcp_<server>_<tool_name>  (discovered tools)
            ├── mcp_<server>_list_resources
            ├── mcp_<server>_read_resource
            ├── mcp_<server>_list_prompts
            └── mcp_<server>_get_prompt
```

#### Transport Types

**Stdio transport** (command + args):
```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    env: {}
```

**HTTP/StreamableHTTP transport** (url + optional headers):
```yaml
mcp_servers:
  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer ${MCP_REMOTE_API_API_KEY}"
```

#### Connection Lifecycle

Each MCP server runs as a long-lived `asyncio.Task` on the dedicated `_mcp_loop`:

```python
async def run(self, config: dict):
    while True:
        try:
            if self._is_http():
                await self._run_http(config)
            else:
                await self._run_stdio(config)
            break  # Normal shutdown
        except Exception as exc:
            if not self._ready.is_set():
                self._error = exc
                return  # First connect fail
            if self._shutdown_event.is_set():
                return  # Requested shutdown
            # Reconnect with exponential backoff
            retries += 1
            if retries > _MAX_RECONNECT_RETRIES:
                return
```

#### Reconnection Strategy

- Exponential backoff: 1s, 2s, 4s, 8s... capped at 60s
- Max 5 retries before giving up
- Only reconnects if connection drops unexpectedly (not on shutdown)
- Connection errors on first attempt are reported immediately (no retry)

#### Environment Security

```python
_SAFE_ENV_KEYS = frozenset({
    "PATH", "HOME", "USER", "LANG", "LC_ALL", "TERM", "SHELL", "TMPDIR",
})
# Plus any XDG_* variables

def _build_safe_env(user_env: Optional[dict]) -> dict:
    env = {}
    for key, value in os.environ.items():
        if key in _SAFE_ENV_KEYS or key.startswith("XDG_"):
            env[key] = value
    if user_env:
        env.update(user_env)  # User-specified vars override
    return env
```

**Command resolution for stdio:**
```python
# Bare "npx" is resolved to absolute path before subprocess spawn
# Also prepends command's directory to PATH for npm-installed tools
```

#### Credential Stripping

All error messages are sanitized before returning to LLM:
```python
_CREDENTIAL_PATTERN = re.compile(
    r"(?:"
    r"ghp_[A-Za-z0-9_]{1,255}"           # GitHub PAT
    r"|sk-[A-Za-z0-9_]{1,255}"           # OpenAI-style key
    r"|Bearer\s+\S+"                      # Bearer token
    r"|token=[^\s&,;\"']{1,255}"         # token=...
    # ... more patterns
    r")",
    re.IGNORECASE,
)
```

#### Sampling (Server-initiated LLM Requests)

MCP servers can request LLM completions via `sampling/createMessage`. The `SamplingHandler`:

```python
class SamplingHandler:
    def __init__(self, server_name: str, config: dict):
        self.max_rpm = 10  # Sliding window rate limiter
        self.timeout = 30
        self.max_tokens_cap = 4096
        self.max_tool_rounds = 5  # Tool loop limit
        self.allowed_models = []   # Whitelist
```

**Tool loop governance:**
- `max_tool_rounds=0`: Tool loops disabled
- `max_tool_rounds=N`: After N tool rounds in one sampling call, returns error
- Prevents runaway recursive tool calling from MCP servers

**Conversion of MCP messages to OpenAI format:**
```python
# Tool results → role: tool with tool_call_id
# Tool uses → role: assistant with tool_calls array
# Text blocks → role: user/assistant with content
# Image blocks → base64 data URLs
```

#### Configuration Schema

```yaml
mcp_servers:
  <name>:
    command: "npx"                    # stdio transport
    args: ["-y", "@modelcontextprotocol/server-github"]
    env: {}                           # Env vars for subprocess
    timeout: 120                      # Per-tool-call timeout (default: 120s)
    connect_timeout: 60              # Initial connection timeout
    url: "https://..."                # HTTP transport (overrides command)
    headers: {}                       # HTTP headers (supports ${ENV_VAR})
    auth: "oauth"                     # OAuth 2.1 PKCE
    enabled: true                     # Enable/disable without removing config
    tools:
      include: []                     # Whitelist of tool names
      exclude: []                     # Blacklist of tool names
      resources: true                 # Enable list_resources, read_resource
      prompts: true                  # Enable list_prompts, get_prompt
    sampling:                         # Server-initiated LLM requests
      enabled: true
      model: "gemini-3-flash"         # Override default model
      max_tokens_cap: 4096
      timeout: 30
      max_rpm: 10
      allowed_models: []             # Whitelist (empty = all)
      max_tool_rounds: 5
      log_level: "info"
```

#### CLI Integration (`hermes mcp`)

```bash
hermes mcp add <name> --url <endpoint>        # Add HTTP server
hermes mcp add <name> --command <cmd>         # Add stdio server
hermes mcp remove <name>                      # Remove server
hermes mcp list                              # Show all servers
hermes mcp test <name>                       # Test connection
hermes mcp configure <name>                  # Toggle tools
```

The `hermes mcp add` command:
1. Accepts transport config (url OR command)
2. Handles OAuth flow if `auth=oauth`
3. Prompts for API key if needed
4. **Discovery-first**: Temporarily connects, lists tools, shows them
5. Asks user to select which tools to enable
6. Saves to `~/.hermes/config.yaml`

#### Tool Name Prefixing

```python
# MCP tool names can contain hyphens/dots which conflict with Python/OpenAI
# Tool names are sanitized: hyphens/dots → underscores

safe_tool_name = mcp_tool.name.replace("-", "_").replace(".", "_")
safe_server_name = server_name.replace("-", "_").replace(".", "_")
prefixed_name = f"mcp_{safe_server_name}_{safe_tool_name}"
# e.g., "github/list_pull_requests" → "mcp_github_list_pull_requests"
```

### Notable Code Patterns

1. **Thread-safe event loop**: `_lock` protects `_servers`, `_mcp_loop`, `_mcp_thread` from concurrent access
2. **Coroutine threadsafe dispatch**: `asyncio.run_coroutine_threadsafe()` schedules work onto the MCP loop from any thread
3. **Graceful SDK degradation**: `_MCP_AVAILABLE`, `_MCP_HTTP_AVAILABLE`, `_MCP_SAMPLING_TYPES` flags handle optional SDK components
4. **anyio TaskGroup error unwrapping**: MCP SDK uses anyio which wraps exceptions in `BaseExceptionGroup`
5. **Command resolution**: `shutil.which()` plus fallback paths for node-based tools
6. **Parallel server discovery**: All configured servers are connected in parallel via `asyncio.gather()`

### Technical Debt / Concerns

1. **Sampling offload to thread**: The synchronous LLM call (`call_llm`) is offloaded via `asyncio.to_thread()`. This is the right pattern, but the previous implementation used `asyncio.run()` in a `ThreadPoolExecutor` which created disposable event loops conflicting with cached httpx clients, causing deadlocks (#2681).

2. **Tool name collision detection is runtime only**: If a built-in tool and MCP tool collide, the built-in wins (logged as warning). But this only happens at discovery time, not at config authoring time.

3. **No persistent OAuth token refresh**: OAuth tokens are acquired at first connection but there's no explicit refresh handling visible in the code. The MCP SDK's `httpx.Auth` handler should handle this, but it's not explicitly tested.

4. **`_existing_tool_names()` iterates `_servers` directly**: This function accesses `server._tools` and `server._registered_tool_names` without holding `_lock` in some code paths, though it's called within `discover_mcp_tools()` which does hold the lock.

5. **Stdio subprocess env filtering**: While safe env vars are explicitly allowlisted, `XDG_*` is the only namespace pattern allowed. Other common safe vars like `DISPLAY` (for GUI tools) or `SSH_AUTH_SOCK` would be filtered out.

---

## Feature 12: Context Files for Project Shaping

### Core Implementation Files
- `agent/prompt_builder.py` - Context file loading and scanning (lines ~17-85 threat patterns, 425-580 context file functions)
- `AGENTS.md` (root of repo) - Development guide for AI coding assistants
- `CONTRIBUTING.md` (root of repo) - Contributor guide

### How It Works

Context files shape every conversation by injecting project-specific instructions into the system prompt. The system uses a **priority-based single-type loading** where only ONE project context type is loaded (not all found).

#### Supported Context File Types

| Priority | File(s) | Scope | Purpose |
|----------|---------|-------|---------|
| 1 | `.hermes.md` / `HERMES.md` | Walk to git root | Project-specific instructions |
| 2 | `AGENTS.md` / `agents.md` | CWD only | AI coding assistant instructions |
| 3 | `CLAUDE.md` / `claude.md` | CWD only | Claude-specific instructions |
| 4 | `.cursorrules` + `.cursor/rules/*.mdc` | CWD only | Cursor IDE rules |
| SOUL | `SOUL.md` (in `~/.hermes/`) | Always | Agent identity/persona |

**First found wins - only ONE project context type is loaded.** This prevents conflicting instructions from multiple sources.

#### Priority Resolution

```python
def build_context_files_prompt(cwd: Optional[str] = None, skip_soul: bool = False):
    # 1. .hermes.md / HERMES.md (walk to git root)
    hermes_md = _load_hermes_md(cwd_path)
    if hermes_md:
        return hermes_md

    # 2. AGENTS.md / agents.md (cwd only)
    agents_md = _load_agents_md(cwd_path)
    if agents_md:
        return agents_md

    # 3. CLAUDE.md / claude.md (cwd only)
    claude_md = _load_claude_md(cwd_path)
    if claude_md:
        return claude_md

    # 4. .cursorrules + .cursor/rules/*.mdc (cwd only)
    cursorrules = _load_cursorrules(cwd_path)
    if cursorrules:
        return cursorrules

    # SOUL.md is independent - always included if present
    # (unless skip_soul=True because it was already loaded separately)
```

#### Git Root Walking (.hermes.md)

```python
def _find_hermes_md(cwd_path: Path) -> Optional[Path]:
    """Walk from CWD to git root looking for .hermes.md or HERMES.md."""
    current = cwd_path.resolve()
    for parent in [current] + list(current.parents):
        git_dir = parent / ".git"
        if git_dir.exists():
            for name in [".hermes.md", "HERMES.md"]:
                candidate = parent / name
                if candidate.exists():
                    return candidate
        if parent == parent.parent:  # Reached filesystem root
            break
    return None
```

This allows `.hermes.md` to apply to the entire git repository while the agent operates in any subdirectory.

#### Threat Scanning

All context file content is scanned before injection:

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    # ... more patterns
]
# Plus invisible unicode detection
_INVISIBLE_CHARS = {'\u200b', '\u200c', '\u200d', '\ufeff', ...}
```

The `_scan_context_content()` function blocks injection attempts but does NOT block the file from being loaded - it just doesn't inject that file's content. This is a "fail open" approach.

#### Content Truncation

```python
CONTEXT_FILE_MAX_CHARS = 20_000
CONTEXT_TRUNCATE_HEAD_RATIO = 0.4  # 40% head
CONTEXT_TRUNCATE_TAIL_RATIO = 0.5   # 50% tail
# Middle 10% is replaced with truncation marker

def _truncate_content(content: str, filename: str, max_chars: int = 20_000):
    if len(content) <= max_chars:
        return content
    head = content[:head_chars]
    tail = content[-tail_chars:]
    marker = f"\n\n[...truncated {filename}: kept {head_chars}+{tail_chars} of {len(content)} chars. Use file tools to read the full file.]\n\n"
    return head + marker + tail
```

This preserves context near both the beginning (often setup/installation) and end (often the most recent/conclusions) of long files.

#### YAML Frontmatter Stripping

```python
def _strip_yaml_frontmatter(content: str) -> str:
    """Remove --- delimited YAML frontmatter from context files."""
    if content.startswith("---"):
        end = content.find("\n---\n", 4)
        if end != -1:
            return content[end + 5:].strip()
    return content
```

This allows context files to include metadata (like the `platforms: [macos, linux]` field in skills) without it being injected into the prompt.

### AGENTS.md - The Repo-Level Context File

The repository's `AGENTS.md` at root is itself a context file that serves as a **development guide for AI coding assistants**. Key sections:

- **Development Environment**: Python venv activation required
- **Project Structure**: Detailed directory tree with file purposes
- **File Dependency Chain**: How tools/registry.py → tools/*.py → model_tools.py → run_agent.py/cli.py
- **AIAgent Class**: Core loop architecture, parameter documentation
- **CLI Architecture**: Rich + prompt_toolkit, slash command registry, skin engine
- **Adding Tools**: 3-file pattern (create, import in model_tools, add to toolsets.py)
- **Adding Configuration**: config.yaml options and .env variables
- **Skin/Theme System**: Data-driven YAML-based theming
- **Important Policies**: Prompt caching stability, working directory behavior
- **Known Pitfalls**: Terminal menu rendering bugs, ANSI escape sequences, global state in delegate_tool, cross-tool references
- **Testing**: Full pytest suite

This file is read by AI coding assistants working in the repository and shapes how they understand and modify the codebase.

### Notable Code Patterns

1. **Fail-open threat scanning**: If content matches threat patterns, the content is simply not injected rather than raising an error or warning the user
2. **Head+tail truncation preserves edges**: Keeps beginning (setup) and end (recent work) while dropping middle of long files
3. **Git-root walking for project-level context**: `.hermes.md` applies to entire repo regardless of subdirectory
4. **Single-type priority loading**: Prevents conflicting instructions from multiple context files
5. **SOUL.md independence**: Agent persona is always loaded separately from project context

### Technical Debt / Concerns

1. **Fail-open threat scanning**: Blocking content silently skips injection means injection attempts go unnoticed. A more robust approach might warn the user or quarantine the file.

2. **Character-based truncation**: The 20,000 char limit is model-independent but may be very different in actual tokens depending on content (code vs prose vs mixed). No consideration for token budgeting.

3. **Only one project context type**: The decision to load only one (first-found) is intentional to avoid conflicts, but means if a user has both `.hermes.md` and `AGENTS.md`, whichever is found first wins. No conflict detection or user notification.

4. **`skip_context_files` flag bypasses all context files**: The `skip_context_files` parameter in AIAgent constructor skips ALL context file loading, not just specific files. This is coarse-grained.

5. **No file watching**: Context files are loaded once at session start. If they change during a long session, the agent won't see updates without restarting.
