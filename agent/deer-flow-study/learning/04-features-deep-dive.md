# DeerFlow Feature Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/deer-flow`
**Synthesized:** 2026-03-26
**Source:** Feature batch analyses 05a-05e

---

## Overview

DeerFlow is an open-source **super agent harness** that orchestrates sub-agents, memory, and sandboxes to execute complex tasks. Built on LangGraph and LangChain, version 2.0 is a ground-up rewrite.

**Tech Stack:** Python 3.12+ (backend), Node.js 22+ (frontend), Next.js (UI), LangGraph/LangChain (orchestration), Docker (sandbox)

**Directory Structure:**
```
deer-flow/
├── backend/
│   ├── app/
│   │   ├── channels/           # IM integrations (telegram, slack, feishu)
│   │   └── gateway/            # HTTP Gateway API
│   ├── docs/                   # Architecture and configuration docs
│   ├── packages/harness/deerflow/
│   │   ├── agents/             # Lead agent, memory, checkpointer
│   │   ├── subagents/           # Sub-agent system
│   │   ├── sandbox/            # Sandbox execution (local, docker, k8s)
│   │   ├── skills/             # Skill loading
│   │   ├── tools/               # Built-in tools
│   │   ├── mcp/                 # MCP server support
│   │   ├── models/              # Model providers
│   │   ├── community/           # External integrations (tavily, jina, infoquest)
│   │   ├── guardrails/          # Safety middleware
│   │   └── client.py           # Embedded Python client
│   └── tests/                   # Backend tests
├── frontend/
│   ├── src/
│   │   ├── app/                 # Next.js app router
│   │   ├── components/
│   │   │   ├── workspace/       # Main workspace UI
│   │   │   ├── ai-elements/      # AI-specific elements
│   │   │   ├── ui/               # Shadcn/ui component library
│   │   │   └── landing/         # Landing page
│   │   └── hooks/               # React hooks
│   └── public/                  # Static assets
├── skills/
│   └── public/                  # Built-in skills (16+ skill definitions)
├── docker/                       # Docker and Kubernetes configs
├── scripts/                      # Deployment and utility scripts
└── docs/                         # Additional documentation
```

---

## Part I: Core Features

---

### 1. Skills & Tools Framework

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/skills/`, `backend/packages/harness/deerflow/tools/`, `skills/public/`

#### Overview

DeerFlow's Skills system provides an extensible framework for defining agent workflows, best practices, and resources. Skills are bundled as directories containing a `SKILL.md` file with YAML frontmatter metadata. The system supports both built-in public skills (in `skills/public/`) and custom user skills (in `skills/custom/`). Ships with 16+ built-in skills including research, report-generation, slide-creation, image-generation, video-generation, podcast-generation, ppt-generation, deep-research, github-deep-research, data-analysis, frontend-design, web-design-guidelines, consulting-analysis, chart-visualization, skill-creator, find-skills, surprise-me, vercel-deploy-claimable, and claude-to-deerflow.

#### Skill Data Model

**File:** `backend/packages/harness/deerflow/skills/types.py`

```python
@dataclass
class Skill:
    name: str
    description: str
    license: str | None
    skill_dir: Path
    skill_file: Path
    relative_path: Path  # Relative from category root
    category: str  # 'public' or 'custom'
    enabled: bool = False  # From extensions_config.json
```

Key methods:
- `get_container_path()` - Returns `/mnt/skills/{category}/{skill_path}` for in-container access
- `get_container_file_path()` - Returns full path to `SKILL.md` in container

#### Skill Loading

**File:** `backend/packages/harness/deerflow/skills/loader.py`

The `load_skills()` function:
- Recursively scans `skills/{public,custom}/` directories for `SKILL.md` files
- Uses `os.walk()` with hidden directory filtering (skips dotfiles)
- Parses frontmatter via `parse_skill_file()`
- Reads enabled state from `extensions_config.json` (not cached config, always reads from disk for cross-process sync)
- Returns skills sorted alphabetically by name

Path resolution strategy:
```python
# loader.py is at packages/harness/deerflow/skills/loader.py — 5 parents reaches backend/
backend_dir = Path(__file__).resolve().parent.parent.parent.parent.parent
skills_dir = backend_dir.parent / "skills"  # sibling to backend directory
```

#### Skill Parsing

**File:** `backend/packages/harness/deerflow/skills/parser.py`

Simple regex-based frontmatter extraction (not full YAML parser):
```python
front_matter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n", content, re.DOTALL)
```

Extracts: `name`, `description`, `license`. Returns `Skill` with `enabled=True` as default (actual state comes from config).

#### Skill Validation

**File:** `backend/packages/harness/deerflow/skills/validation.py`

Validates `SKILL.md` frontmatter:
- Allowed properties: `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`, `version`, `author`
- Name rules: hyphen-case (lowercase, digits, hyphens), max 64 chars, no leading/trailing/consecutive hyphens
- Description rules: max 1024 chars, no angle brackets

#### Skill Installation

**File:** `backend/packages/harness/deerflow/skills/installer.py`

Installs skills from `.skill` (ZIP archive) files with security protections:
- **Zip bomb defense**: 512MB max uncompressed size with chunked reading
- **Path traversal prevention**: `PurePosixPath` normalization, rejects `..` in paths
- **Symlink handling**: Skips symlink entries instead of materializing them
- **Archive structure detection**: Supports both single-dir and flat archives (filters `__MACOSX`, `.DS_Store`)

#### Tool System

**File:** `backend/packages/harness/deerflow/tools/tools.py`

`get_available_tools()` assembles tools from multiple sources:

| Source | Description |
|--------|-------------|
| Config-defined tools | Resolved from `config.yaml` via `resolve_variable()` |
| MCP tools | From enabled MCP servers (lazy, mtime-cached) |
| Built-in tools | `present_files`, `ask_clarification`, `view_image` (conditional) |
| Subagent tool | `task` tool (if `subagent_enabled=True`) |
| ACP tools | External ACP-compatible agents |
| Tool search | Deferred MCP tool registry (if `tool_search.enabled`) |

**Key pattern**: `ExtensionsConfig.from_file()` is used instead of cached config to ensure cross-process sync when Gateway API modifies MCP/skills config.

#### Built-in Tool Implementations

**Location:** `backend/packages/harness/deerflow/tools/builtins/`

- `task_tool.py` - Subagent delegation
- `present_file_tool.py` - Makes output files visible to user (only `/mnt/user-data/outputs`)
- `ask_clarification_tool.py` - Requests clarification (intercepted by ClarificationMiddleware)
- `view_image_tool.py` - Reads image as base64 (added only if model supports vision)
- `invoke_acp_agent_tool.py` - Invokes external ACP-compatible agents
- `tool_search.py` - Deferred MCP tool discovery

#### Clever Solutions

1. **Cross-process config sync**: Always reads `extensions_config.json` from disk rather than using cached config, ensuring Gateway API changes are immediately visible to LangGraph Server.

2. **Path preservation style**: `_join_path_preserving_style()` maintains POSIX vs Windows path semantics when joining.

3. **Skill container paths**: Skills resolve to `/mnt/skills/{category}/{relative_path}` inside containers, decoupled from host layout.

#### Technical Debt / Concerns

1. **Simple regex YAML parsing**: `parser.py` uses simple key-value splitting instead of proper YAML parser, which could fail on complex frontmatter.

2. **No skill version/content validation**: Only frontmatter is validated, not the actual skill content/guidance quality.

3. **Global skill state**: Enabled/disabled state is global, not per-thread or per-user.

---

### 2. Sub-Agent Orchestration

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/subagents/`, `backend/packages/harness/deerflow/tools/builtins/task_tool.py`

#### Overview

DeerFlow's Sub-Agent system allows the lead agent to decompose complex tasks and delegate to specialized sub-agents running in separate contexts. Sub-agents execute asynchronously in a thread pool, with the lead agent polling for completion.

#### Subagent Types

**Location:** `backend/packages/harness/deerflow/subagents/builtins/`

**general-purpose** - Full-capability agent for complex multi-step tasks:
```python
SubagentConfig(
    name="general-purpose",
    tools=None,  # Inherit all tools
    disallowed_tools=["task", "ask_clarification", "present_files"],
    model="inherit",
    max_turns=50,
    timeout_seconds=900,
)
```

**bash** - Command execution specialist:
```python
SubagentConfig(
    name="bash",
    tools=["bash", "ls", "read_file", "write_file", "str_replace"],  # Sandbox tools only
    disallowed_tools=["task", "ask_clarification", "present_files"],
    model="inherit",
    max_turns=30,
    timeout_seconds=900,
)
```

#### Subagent Executor

**File:** `backend/packages/harness/deerflow/subagents/executor.py`

**Dual thread pool architecture**:
```python
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
```

- `_scheduler_pool`: Accepts new task requests, updates status
- `_execution_pool`: Runs actual subagent execution with timeout enforcement

**Execution flow**:
1. `execute_async()` creates `SubagentResult` with `PENDING` status
2. Submit to `_scheduler_pool` which updates status to `RUNNING`
3. `_scheduler_pool` submits to `_execution_pool` with timeout
4. `asyncio.run()` runs async agent in sync context (required for async MCP tools)
5. Polling every 5 seconds in `task_tool.py` for completion

**Background task storage**:
```python
_background_tasks: dict[str, SubagentResult] = {}  # Global dict with thread lock
```

#### Task Tool

**File:** `backend/packages/harness/deerflow/tools/builtins/task_tool.py`

The main interface for subagent delegation:

```python
@tool("task", parse_docstring=True)
def task_tool(
    description: str,      # Short label for logging
    prompt: str,           # Task description
    subagent_type: Literal["general-purpose", "bash"],
    max_turns: int | None = None,
) -> str:
```

**Key behaviors**:
- Always runs async via `execute_async()` to prevent blocking LLM
- Uses `tool_call_id` as `task_id` for traceability
- Backend polling (not LLM polling) - polls every 5 seconds up to `timeout + 60` seconds
- Sends SSE events: `task_started`, `task_running`, `task_completed`, `task_failed`, `task_timed_out`

**Result extraction logic** (handles complex content types):
```python
# Extracts from final_state["messages"], finds last AIMessage
# Handles str, list of content blocks, with fallback to last message
```

#### Middleware Chain

**File:** `backend/packages/harness/deerflow/agents/lead_agent/agent.py`

Middleware execution order:
1. ThreadDataMiddleware - Creates per-thread directories
2. UploadsMiddleware - Tracks file uploads
3. SandboxMiddleware - Acquires sandbox
4. DanglingToolCallMiddleware - Patches missing tool responses
5. GuardrailMiddleware - Pre-tool-call authorization
6. SummarizationMiddleware - Context reduction
7. TodoListMiddleware - Task tracking
8. TitleMiddleware - Auto-generate title
9. MemoryMiddleware - Queue for memory update
10. ViewImageMiddleware - Inject image data
11. SubagentLimitMiddleware - Truncate excess `task` calls
12. ClarificationMiddleware - Intercept clarification requests

#### Lead Agent Prompt

**File:** `backend/packages/harness/deerflow/agents/lead_agent/prompt.py`

Subagent section with concurrency limit baked into prompt:
```
**HARD CONCURRENCY LIMIT: MAXIMUM 3 `task` CALLS PER RESPONSE**
- If count > 3: Pick 3 most important for this turn
- Multi-batch: Launch 3 now, wait, launch next batch
```

#### Clever Solutions

1. **Async backend polling**: LLM doesn't poll - backend blocks on `time.sleep(5)` and streams SSE events. LLM just waits for `task_completed`.

2. **Tool call ID as task ID**: Uses `tool_call_id` from `@InjectedToolCallId` as `task_id`, enabling traceability from LLM call to background task.

3. **Dual pool design**: Scheduler pool accepts requests quickly, execution pool runs with proper timeout isolation.

4. **Subagent tool filtering**: Disallows `task` tool in subagents to prevent recursive nesting, while still allowing MCP/ACP tools.

5. **Content type handling**: Robust handling of AIMessage content as both `str` and `list` of content blocks.

#### Technical Debt / Concerns

1. **Global task storage**: `_background_tasks` is a module-level dict with a simple lock - could have concurrency issues at scale.

2. **No task priority**: All tasks are equal priority, no preemption.

3. **Timeout as cancellation**: Uses `Future.cancel()` which is best-effort - task may still run to completion.

4. **Result deduplication**: Message deduplication by ID or full dict comparison could miss legitimate duplicates with different IDs.

5. **15-minute hard timeout**: `timeout_seconds=900` is not configurable per-task, only per subagent type globally.

---

### 3. Sandbox Execution Environment

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/sandbox/`, `backend/packages/harness/deerflow/community/aio_sandbox/`

#### Overview

DeerFlow's Sandbox system provides isolated execution environments for agent operations. Supports local execution (singleton), Docker containers, and Kubernetes pods via provisioner service.

#### Sandbox Interface

**File:** `backend/packages/harness/deerflow/sandbox/sandbox.py`

```python
class Sandbox(ABC):
    _id: str

    @abstractmethod
    def execute_command(self, command: str) -> str: ...

    @abstractmethod
    def read_file(self, path: str) -> str: ...

    @abstractmethod
    def list_dir(self, path: str, max_depth=2) -> list[str]: ...

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None: ...

    @abstractmethod
    def update_file(self, path: str, content: bytes) -> None: ...
```

#### Sandbox Provider

**File:** `backend/packages/harness/deerflow/sandbox/sandbox_provider.py`

Abstract provider with lifecycle:
```python
class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str: ...

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None: ...

    @abstractmethod
    def release(self, sandbox_id: str) -> None: ...
```

Singleton pattern with `get_sandbox_provider()`, reset via `reset_sandbox_provider()` or `shutdown_sandbox_provider()`.

#### Local Sandbox

**File:** `backend/packages/harness/deerflow/sandbox/local/local_sandbox.py`

**Path mapping system**:
```python
def _resolve_path(self, path: str) -> str:
    # Longest-prefix-first matching
    for container_path, local_path in sorted(self.path_mappings.items(), ...):
        if path_str == container_path or path_str.startswith(container_path + "/"):
            relative = path_str[len(container_path):].lstrip("/")
            return str(Path(local_path) / relative)
```

**Bidirectional path translation**:
- `_resolve_path()`: Container path -> local path
- `_reverse_resolve_path()`: Local path -> container path
- `_resolve_paths_in_command()`: Translates all paths in a command string
- `_reverse_resolve_paths_in_output()`: Translates local paths back to container in output

**Shell detection**:
```python
@staticmethod
def _get_shell() -> str:
    for shell in ("/bin/zsh", "/bin/bash", "/bin/sh"):
        if os.path.isfile(shell) and os.access(shell, os.X_OK):
            return shell
```

**Command execution**:
```python
subprocess.run(
    resolved_command,
    executable=self._get_shell(),
    shell=True,
    capture_output=True,
    text=True,
    timeout=600,  # 10 minute timeout
)
```

#### Local Sandbox Provider

**File:** `backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py`

**Singleton pattern**:
```python
_singleton: LocalSandbox | None = None

def acquire(self, thread_id: str | None = None) -> str:
    global _singleton
    if _singleton is None:
        _singleton = LocalSandbox("local", path_mappings=self._path_mappings)
    return _singleton.id
```

**Note**: `release()` is intentionally no-op for local sandbox to enable reuse across turns. Docker-based providers clean up on `shutdown()`.

#### Sandbox Tools

**File:** `backend/packages/harness/deerflow/sandbox/tools.py`

**Virtual path mapping**:
```
/mnt/user-data/workspace -> thread_data['workspace_path']
/mnt/user-data/uploads -> thread_data['uploads_path']
/mnt/user-data/outputs -> thread_data['outputs_path']
/mnt/skills -> skills host path
/mnt/acp-workspace -> ACP workspace path
```

**ACP workspace handling**:
- Per-thread workspace: `{base_dir}/threads/{thread_id}/acp-workspace/`
- Global fallback: `{base_dir}/acp-workspace/`
- Path traversal prevention via `is_relative_to()` check

#### Sandbox Middleware

**File:** `backend/packages/harness/deerflow/sandbox/middleware.py`

```python
class SandboxMiddleware(AgentMiddleware):
    def __init__(self, lazy_init: bool = True):
        # lazy_init=True: Defer sandbox acquisition to first tool call
        # lazy_init=False: Acquire in before_agent()
```

**Thread-local sandbox reuse**: Sandbox is NOT released after each turn to avoid wasteful recreation.

#### Kubernetes Sandbox

**Location:** `backend/packages/harness/deerflow/community/aio_sandbox/`

**Architecture**:
```
AioSandboxProvider (orchestration)
├── LocalContainerBackend (Docker mode)
└── RemoteSandboxBackend (K8s mode)
```

**Idle timeout with LRU eviction**:
```python
# Warm pool: released sandboxes whose containers are still running
# When replicas exceeded: evict least-recently-used
```

#### Clever Solutions

1. **Virtual path abstraction**: Agent always sees `/mnt/user-data/*` and `/mnt/skills`, never host paths. Enables same agent code to run in local, Docker, or K8s environments.

2. **Bidirectional path translation**: Commands execute with translated paths, but output has paths re-translated back to virtual paths. User never sees host layout.

3. **Lazy sandbox acquisition**: `lazy_init=True` defers sandbox creation until first tool call, avoiding waste for simple queries.

4. **ACP workspace per-thread isolation**: Each thread gets its own ACP workspace at `{base_dir}/threads/{thread_id}/acp-workspace/`.

5. **LRU warm pool**: Released sandboxes stay warm for `idle_timeout` seconds, allowing fast reuse while still cleaning up eventually.

#### Technical Debt / Concerns

1. **10-minute command timeout**: Hard-coded `timeout=600` in `execute_command()` - not configurable.

2. **Singleton local sandbox**: No true isolation between threads in local mode - all share same filesystem.

3. **Path mapping at construction**: `_path_mappings` is set once at provider creation, not dynamically for thread.

4. **No network isolation**: Local mode has no network namespace isolation.

5. **Process shell execution**: Uses `shell=True` which has security implications (though path translation provides some protection).

---

### 4. Context Engineering

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/agents/memory/`, `backend/packages/harness/deerflow/agents/middlewares/`

#### Overview

Context Engineering is implemented through a middleware chain and memory system that aggressively manages context to prevent window blowout.

#### Key Components

**UploadsMiddleware** (`uploads_middleware.py`)

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

**Edge case handled:** Validates file existence before listing. Skips entries where `filename` is not just a name (guards against path injection).

**MemoryMiddleware** (`memory_middleware.py`)

Queues conversations for async memory update after agent execution:

```python
def _filter_messages_for_memory(messages: list[Any]) -> list[Any]:
    # Strips <uploaded_files> blocks from human messages
    # Filters to: human messages (with uploads stripped) + AI messages without tool_calls
    # Drop entire human+AI pair if the human was upload-only
```

**Key insight:** Uses `skip_next_ai` flag to handle upload-only messages.

**Memory Queue** (`queue.py`)

Debounced update queue with per-thread deduplication:

```python
# Each thread dedupes itself - newer context replaces older pending context
self._queue = [c for c in self._queue if c.thread_id != thread_id]
self._queue.append(context)
```

**Design pattern:** Uses `threading.Timer` for debouncing. Processes all queued contexts in batch after debounce period.

**Memory Updater** (`updater.py`)

LLM-based memory extraction with atomic file I/O:

```python
# Atomic write via temp file + rename
temp_path = file_path.with_suffix(".tmp")
with open(temp_path, "w", encoding="utf-8") as f:
    json.dump(memory_data, f, indent=2, ensure_ascii=False)
temp_path.replace(file_path)  # Atomic on most systems
```

**Important:** File-upload mentions are stripped from summaries before saving using a carefully narrow regex to avoid removing legitimate facts.

**Token-Aware Injection** (`prompt.py`)

```python
def format_memory_for_injection(memory_data: dict[str, Any], max_tokens: int = 2000) -> str:
    # Sorts facts by confidence (descending)
    # Accumulates tokens incrementally to avoid full-string re-tokenization
    # Truncates at token budget with "..." suffix
```

#### Data Flow

```
User Message → UploadsMiddleware (injects file context)
             → Agent executes
             → MemoryMiddleware (after_agent hook)
             → MemoryQueue.add() [debounced 30s]
             → MemoryUpdater.update_memory() [LLM extraction]
             → Atomic file save to memory.json
             → Next interaction: format_memory_for_injection() → system prompt
```

#### Clever Solutions

1. **Per-thread queue deduplication**: Same thread's pending update is replaced by newer context rather than accumulating
2. **Whitespace-normalized fact dedup**: Trims leading/trailing whitespace before comparing fact content
3. **Atomic file I/O**: Temp file + rename pattern prevents corrupted memory.json on crash
4. **Upload-mention scrubbing**: Regex carefully excludes legitimate facts like "user works with CSV files"

#### Technical Debt / Shortcuts

1. **Global singleton queue**: `get_memory_queue()` returns a module-level singleton. Tests must call `reset_memory_queue()` to isolate.
2. **No fact expiration**: Facts accumulate until `max_facts` (100) is hit, then lowest-confidence are evicted. No time-based expiration.
3. **Planned TF-IDF retrieval not implemented**: `MEMORY_IMPROVEMENTS.md` documents planned similarity-based retrieval but it's not shipped yet.
4. **`print()` for logging in queue**: Uses `print()` instead of proper logging.

---

### 5. Long-Term Memory

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/agents/memory/`, `backend/docs/MEMORY_IMPROVEMENTS.md`

#### Overview

Long-term memory persists across sessions through a JSON file with structured data.

#### Data Structure

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

#### Memory Cache with File Mtime Invalidation

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

#### Missing Planned Features

From `MEMORY_IMPROVEMENTS.md`:
- **TF-IDF similarity-based fact retrieval** - Not implemented
- **Context-aware scoring** with `current_context` input - Not implemented
- **Configurable similarity/confidence weights** - Not implemented
- **Middleware/runtime wiring for context-aware retrieval** - Not implemented

Current behavior: Facts ranked purely by confidence, injected until token budget exhausted.

---

### 6. Multi-Channel IM Integration

**Priority:** CORE
**Location:** `backend/app/channels/`

#### Overview

Receive tasks from messaging apps (Telegram, Slack, Feishu/Lark) via WebSocket/bot APIs. Channels auto-start when configured; no public IP required.

#### Architecture Overview

```
External Platform → Channel (telegram.py/slack.py/feishu.py)
                  → MessageBus.publish_inbound()
                  → ChannelManager._dispatch_loop()
                  → LangGraph Server (thread create/invoke)
                  → MessageBus.publish_outbound()
                  → Channel.send()
                  → External Platform reply
```

#### Core Components

**MessageBus** (`message_bus.py`)

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

**ChannelStore** (`store.py`)

JSON-file persistence mapping `channel:chat_id[:topic_id]` to `thread_id`:

```python
# Atomic save via temp file
fd = tempfile.NamedTemporaryFile(mode="w", dir=self._path.parent, suffix=".tmp", delete=False)
json.dump(self._data, fd, fd.close())
Path(fd.name).replace(self._path)
```

**ChannelManager** (`manager.py`)

Core dispatcher bridging IM to LangGraph:

```python
class ChannelManager:
    async def _handle_chat(self, msg: InboundMessage) -> None:
        # Look up or create thread
        thread_id = self.store.get_thread_id(msg.channel_name, msg.chat_id, topic_id=msg.topic_id)
        if thread_id is None:
            thread_id = await self._create_thread(client, msg)
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

#### Platform-Specific Implementations

**Telegram** (`telegram.py`)
- **Transport:** Long-polling via `python-telegram-bot`
- **Threading model:** `topic_id = None` for private chats, `topic_id = reply_to.message_id` for group chats
- **Retry:** 3 retries with exponential backoff (1s, 2s)
- **Running reply:** Replies "Working on it..." to user message before publishing inbound
- **File limits:** 10MB photos, 50MB documents

**Slack** (`slack.py`)
- **Transport:** Socket Mode (WebSocket, no public IP needed)
- **SDK:** `slack-sdk` SocketModeClient
- **Threading model:** Uses `thread_ts` as `topic_id`
- **Reactions:** Adds `eyes` reaction on receive, `white_check_mark` on success, `x` on failure
- **Markdown conversion:** Uses `markdown_to_mrkdwn` for output formatting

**Feishu** (`feishu.py`)
- **Transport:** WebSocket long-connection via `lark-oapi`
- **Threading workaround:** Runs in dedicated thread with plain asyncio loop (not uvloop) because lark-oapi captures event loop at import time
- **Running card pattern:** Creates interactive card, patches same card for each update (uses `update_multi: true`)
- **File limits:** 10MB images, 30MB files

#### Command Handling

All channels support slash commands routed via `InboundMessageType.COMMAND`:

```
/bootstrap — Initialize workspace (creates thread with is_bootstrap context)
/new — Start new conversation (creates new thread)
/status — Show current thread
/models — List available models (queries Gateway API)
/memory — Show memory status (queries Gateway API)
/help — Show available commands
```

#### Clever Patterns

1. **Thread reuse:** `topic_id` maps to same LangGraph thread. Messages in same Feishu thread/Slack thread share conversation.
2. **Streaming debouncing:** `STREAM_UPDATE_MIN_INTERVAL_SECONDS = 0.35` prevents message flooding
3. **Background task tracking:** Feishu keeps strong references to fire-and-forget tasks to surface errors
4. **Retry with emoji feedback:** Slack adds checkmark on success, X on failure after all retries exhausted

#### Technical Debt / Issues

1. **uvloop conflict in Feishu:** Requires dedicated thread workaround due to lark-oapi capturing module-level event loop
2. **No graceful degradation:** If LangGraph Server is down, all channel messages fail with no queue/retry
3. **JSON store limitations:** Comment admits "for production workloads with high concurrency, this can be swapped for a proper database backend"
4. **`print()` in queue.py:** Uses `print()` instead of logging (shared technical debt with memory system)
5. **No message retry queue:** Failed dispatches are not retried, just logged

---

### 7. Model Agnostic Architecture

**Priority:** CORE
**Location:** `backend/packages/harness/deerflow/models/`

#### Overview

DeerFlow's model layer provides a pluggable architecture supporting any LLM with an OpenAI-compatible API. The system includes first-class support for Claude Code OAuth tokens and Codex CLI credentials, enabling users to run DeerFlow without managing separate API keys.

#### Architecture

**Model Factory** (`factory.py`)

The `create_chat_model()` function is the single entry point for model instantiation:

```python
def create_chat_model(name: str | None = None, thinking_enabled: bool = False, **kwargs) -> BaseChatModel:
```

**Key behaviors:**
1. **Config-driven instantiation**: Uses `resolve_class(model_config.use, BaseChatModel)` to dynamically load any LangChain-compatible chat model class
2. **Thinking mode handling**: Complex logic to merge `when_thinking_enabled` config with `thinking` shortcut field
3. **Provider-specific hacks**: Hard-coded awareness of `CodexChatModel` to handle `reasoning_effort` mapping
4. **Tracing integration**: Optionally attaches LangChain tracer if tracing is enabled

**Credential Loader** (`credential_loader.py`)

**Claude Code OAuth credential resolution order:**
1. `$CLAUDE_CODE_OAUTH_TOKEN` or `$ANTHROPIC_AUTH_TOKEN`
2. `$CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`
3. `$CLAUDE_CODE_CREDENTIALS_PATH`
4. `~/.claude/.credentials.json`

**Token format detection:**
```python
def is_oauth_token(token: str) -> bool:
    return isinstance(token, str) and "sk-ant-oat" in token
```

**Claude Provider** (`claude_provider.py`)

`ClaudeChatModel` extends `ChatAnthropic` with OAuth and prompt caching:

**OAuth Bearer auth implementation:**
```python
def _patch_client_oauth(self, client: Any) -> None:
    if hasattr(client, "api_key") and hasattr(client, "auth_token"):
        client.api_key = None
        client.auth_token = self._oauth_access_token
```

**Auto-thinking budget:**
```python
THINKING_BUDGET_RATIO = 0.8  # 80% of max_tokens
```

If `thinking.type: enabled` but no `budget_tokens` specified, automatically allocates 80% of `max_tokens` as thinking budget.

**Codex Provider** (`openai_codex_provider.py`)

Custom LangChain chat model using ChatGPT Codex Responses API:
- **Endpoint:** `https://chatgpt.com/backend-api/codex/responses`
- Uses `reasoning.effort` instead of thinking (options: `none`, `low`, `medium`, `high`, `xhigh`)
- Requires streaming (API enforces this)

#### Configuration Example

```yaml
models:
  # Claude with OAuth
  - name: claude-sonnet-4.6
    use: deerflow.models.claude_provider:ClaudeChatModel
    model: claude-sonnet-4-6
    max_tokens: 16384
    enable_prompt_caching: true

  # Codex CLI
  - name: gpt-5.4
    use: deerflow.models.openai_codex_provider:CodexChatModel
    model: gpt-5.4
    reasoning_effort: medium

  # Standard OpenAI-compatible
  - name: gpt-4
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
```

#### Technical Debt / Issues

1. **Hard-coded Codex awareness**: `factory.py` has explicit `issubclass(model_class, CodexChatModel)` check - violates open/closed principle
2. **Credential loading is imperative**: No lazy loading; credentials are loaded at model creation time
3. **No credential refresh**: OAuth refresh tokens are loaded but never used for refresh
4. **Azure OpenAI not explicitly supported**: Would work via OpenAI-compatible but no examples in config

---

## Part II: Secondary Features

---

### 8. Claude Code CLI Integration

**Priority:** SECONDARY
**Location:** `skills/public/claude-to-deerflow/`, `frontend/src/components/ui/terminal.tsx`

#### Overview

The `claude-to-deerflow` skill enables terminal-based interaction with DeerFlow from Claude Code. Users can send research tasks, manage threads, upload files, and query system state via HTTP API calls from shell scripts.

#### Skill Definition

**File:** `skills/public/claude-to-deerflow/SKILL.md`

**Two API endpoints:**

| Service | Port | Via Proxy |
|---------|------|-----------|
| Gateway API | 8001 | `$DEERFLOW_GATEWAY_URL` |
| LangGraph API | 2024 | `$DEERFLOW_LANGGRAPH_URL` |

**Context modes (progressive capability):**

| Mode | thinking_enabled | is_plan_mode | subagent_enabled |
|------|-----------------|--------------|------------------|
| flash | false | false | false |
| standard | true | false | false |
| pro | true | true | false |
| ultra | true | true | true |

#### Chat Script

**File:** `skills/public/claude-to-deerflow/scripts/chat.sh`

**Workflow:**
1. Health check against Gateway API
2. Create thread (or reuse existing thread_id)
3. Build context based on mode (flash/standard/pro/ultra)
4. Stream run via SSE
5. Parse SSE output to extract final AI response

**SSE parsing approach:**
- Collects full SSE stream to temp file
- Uses Python to parse `event: <type>` and `data: <json>` lines
- Extracts last `values` event to get final state

#### Key Design Patterns

1. **Shell-first API**: All operations use `curl` - no SDK dependency
2. **Thread persistence**: Thread IDs allow conversation continuation
3. **Streaming by default**: Uses `runs/stream` for real-time feedback
4. **Mode abstraction**: Users select capability level, not individual flags

---

### 9. Embedded Python Client (DeerFlowClient)

**Priority:** SECONDARY
**Location:** `backend/packages/harness/deerflow/client.py`

#### Overview

`DeerFlowClient` provides in-process access to DeerFlow's agent capabilities without requiring LangGraph Server or Gateway API HTTP services. The client imports and uses the same internal modules as the server-side components.

#### Architecture

```python
class DeerFlowClient:
    def __init__(
        self,
        config_path: str | None = None,
        checkpointer=None,
        *,
        model_name: str | None = None,
        thinking_enabled: bool = True,
        subagent_enabled: bool = False,
        plan_mode: bool = False,
        agent_name: str | None = None,
    ):
```

**Key design decisions:**
- **Lazy agent creation**: Agent created on first `stream()`/`chat()` call, not at init
- **Config caching with invalidation**: Agent recreated when config-dependent params change
- **No FastAPI dependency**: Pure harness imports only
- **Checkpointer optional**: Without one, each call is stateless

#### Streaming Implementation

```python
def stream(self, message: str, *, thread_id: str | None = None, **kwargs) -> Generator[StreamEvent, None, None]:
```

**Event types align with LangGraph SSE protocol:**

| Event Type | Data | Description |
|------------|------|-------------|
| `values` | `{title, messages, artifacts}` | Full state snapshot |
| `messages-tuple` | `{type, content, id, ...}` | Per-message update |
| `end` | `{usage}` | Stream finished |

#### Gateway Conformance

The client aims to return types matching Gateway API Pydantic models. `TestGatewayConformance` class parses client output through Gateway Pydantic models, ensuring client and Gateway stay in sync.

#### Clever Solutions

1. **Lazy import pattern**: `_get_tools()` uses lazy import inside method to avoid circular dependency at module level
2. **Content extraction**: `_extract_text()` handles mixed str/list/dict content blocks intelligently
3. **Message serialization**: `_serialize_message()` converts LangChain messages to plain dicts for `values` events
4. **Config hot-reload**: `update_mcp_config()` and `update_skill()` invalidate agent cache, forcing recreation on next call

#### Technical Debt / Issues

1. **No thread cleanup method**: Gateway has `/api/threads/{id}` DELETE for cleanup, but client doesn't
2. **Agent recreation on every config change**: Could use more granular invalidation
3. **Blocking in async context**: `upload_files()` uses `asyncio.run()` in thread pool which could cause issues in some async contexts
4. **No streaming uploads**: File upload is batch, not streaming

---

### 10. Guardrails & Safety System

**Priority:** SECONDARY
**Location:** `backend/packages/harness/deerflow/guardrails/`

#### Overview

Guardrails is a middleware-based pre-tool-call authorization system that evaluates every tool call against a configurable policy before execution. It provides deterministic, policy-driven authorization without requiring human intervention.

#### Architecture

```
GuardrailMiddleware (position 5 in middleware chain)
         │
         ▼
┌─────────────────────────┐
│   GuardrailProvider     │  ← Pluggable via config
│   (Protocol)            │
└─────────┬───────────────┘
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
Built-in AllowlistProvider   OAP Passport Provider   Custom Provider
(zero deps)                  (open standard)         (user code)
```

#### Data Structures

```python
@dataclass
class GuardrailRequest:
    tool_name: str
    tool_input: dict[str, Any]
    agent_id: str | None = None
    thread_id: str | None = None
    is_subagent: bool = False
    timestamp: str = ""

@dataclass
class GuardrailDecision:
    allow: bool
    reasons: list[GuardrailReason] = field(default_factory=list)
    policy_id: str | None = None
    metadata: dict[str, Any] = field(default_factory=dict)
```

The system uses OAP (Open Agent Passport) reason codes for standardization.

#### GuardrailMiddleware

Key design decisions:
1. **Fail-closed by default** - If provider raises exception, tool call is blocked
2. **GraphBubbleUp propagation** - LangGraph control signals always propagate through
3. **Denied calls return error ToolMessage** - Agent sees denial reason and can adapt

#### Built-in AllowlistProvider

```python
class AllowlistProvider:
    def __init__(self, *, allowed_tools: list[str] | None = None, denied_tools: list[str] | None = None):
        self._allowed = set(allowed_tools) if allowed_tools else None
        self._denied = set(denied_tools) if denied_tools else set()
```

**Decision logic:**
- If `_allowed` is set and tool not in allowlist → DENY
- If tool in `_denied` → DENY
- Otherwise → ALLOW

#### Clever Solutions

1. **Protocol-based provider interface** - Uses `@runtime_checkable Protocol` instead of abstract base class
2. **Framework injection** - Provider `__init__` receives `framework="deerflow"` kwarg
3. **Config caching** - GuardrailConfig singleton with reset capability for clean reloading
4. **Async-first design** - Both sync `evaluate` and async `aevaluate` methods

#### Technical Debt / Concerns

1. **Provider loading via string path** - Uses `resolve_variable()` which imports by class path string
2. **No built-in rate limiting** - Only authorization, no throttling of repeated tool calls
3. **Limited audit trail** - Only logs warnings on denial, no persistent audit log

---

### 11. Gateway HTTP API

**Priority:** SECONDARY
**Location:** `backend/app/gateway/`

#### Overview

The Gateway API is a FastAPI application (port 8001) providing REST endpoints for models, MCP, skills, memory, artifacts, uploads, threads, agents, suggestions, and IM channels. It complements the LangGraph API by handling management operations separate from agent runtime.

#### Router Architecture

| Router | Prefix | Purpose |
|--------|--------|---------|
| `models.py` | `/api/models` | List/query available LLM models |
| `mcp.py` | `/api/mcp` | MCP server configuration |
| `skills.py` | `/api/skills` | Skill listing, enable/disable, install |
| `memory.py` | `/api/memory` | Global memory data |
| `uploads.py` | `/api/threads/{id}/uploads` | File upload management |
| `artifacts.py` | `/api/threads/{id}/artifacts` | Artifact file serving |
| `threads.py` | `/api/threads/{id}` | Thread cleanup |
| `agents.py` | `/api/agents` | Custom agent CRUD |
| `suggestions.py` | `/api/threads/{id}/suggestions` | Follow-up question generation |
| `channels.py` | `/api/channels` | IM channel status |

#### Key Design Patterns

1. **Routed middleware inclusion** - Clean separation via `include_router()` calls with tags
2. **Virtual path abstraction** - Sandbox paths abstracted from actual filesystem layout
3. **Thread-isolated uploads** - Each thread has isolated uploads directory
4. **Skill archive handling** - `.skill` files treated as ZIP with internal path resolution
5. **Lifespan context manager** - Clean startup/shutdown for channel services

#### Technical Debt / Concerns

1. **No authentication** - All endpoints publicly accessible (documented limitation)
2. **CORS handled by nginx** - FastAPI CORS config not actually used
3. **MCP tools split responsibility** - Gateway manages MCP config, LangGraph manages MCP tools
4. **Error handling inconsistency** - Some endpoints return `500` with generic message, others return detailed errors

---

### 12. Artifact System

**Priority:** SECONDARY
**Location:** `frontend/src/components/workspace/artifacts/`, `backend/app/gateway/routers/artifacts.py`

#### Overview

The Artifact System provides rich content rendering for code blocks, web previews, markdown, HTML, and visualizations. It spans both backend (artifact file serving) and frontend (rendering components).

#### Backend Implementation

**Endpoint:** `GET /api/threads/{thread_id}/artifacts/{path}`

**Features:**
1. Virtual path resolution with traversal protection
2. Active content (HTML/XHTML/SVG) always served as download
3. ZIP extraction for `.skill` files
4. Text vs binary content type detection
5. Cache headers for ZIP extraction (5 minutes)

**Active content types (always downloaded):**
```python
ACTIVE_CONTENT_MIME_TYPES = {
    "text/html",
    "application/xhtml+xml",
    "image/svg+xml",
}
```

#### Frontend Component Architecture

```
ArtifactFileList (artifact trigger)
         │
         ▼
ArtifactFileDetail (main viewer)
    │
    ├── CodeEditor (CodeMirror)
    │     - Syntax highlighting via @uiw/react-codemirror
    │     - Multiple language support (Python, JS, HTML, CSS, JSON, Markdown)
    │
    ├── ArtifactFilePreview (content renderer)
    │     │
    │     ├── Markdown → Streamdown (markdown renderer)
    │     └── HTML → sandboxed iframe (sandbox="allow-scripts allow-forms")
    │
    └── ArtifactActions (toolbar)
          - Copy to clipboard
          - Download
          - Open in new window
          - Install skill (.skill files only)
          - Close
```

#### CodeBlock

```typescript
highlightCode(code, language, showLineNumbers)
// Uses Shiki for syntax highlighting
// Dual theme support: light (one-light) + dark (one-dark-pro)
```

Features:
- Dual-theme (light/dark) with CSS toggle
- Line numbers via custom Shiki transformer
- Copy button with success feedback

#### Clever Solutions

1. **Dual-theme code highlighting** - Pre-renders both light/dark HTML, CSS controls which is visible (no flash)
2. **Streamdown for Markdown** - Custom markdown renderer with `ArtifactLink` component for internal links
3. **Write-file virtual URL handling** - `write-file:{path}?tool_call_id=...&message_id=...` pattern extracts content directly from tool call
4. **Lazy CodeMirror** - Falls back to textarea during stream loading to avoid rendering glitches

#### Technical Debt / Concerns

1. **Limited preview types** - Only HTML and Markdown have preview; other files show code view
2. **Artifact list in thread state** - `thread.values.artifacts` is array of strings, no metadata (type, size, etc.)
3. **No versioning** - Artifacts are static once created, no revision history
4. **Memory consideration** - Pre-rendered dual-theme HTML could be memory-intensive for large files

---

### 13. InfoQuest Integration

**Priority:** SECONDARY
**Location:** `backend/packages/harness/deerflow/community/infoquest/`

#### Overview

InfoQuest is BytePlus's web search and content extraction service integrated into DeerFlow as community tools. It provides three LangChain tools: `web_search`, `web_fetch`, and `image_search`.

#### Architecture

```
infoquest_client.py (InfoQuestClient)
    ├── fetch()              → /crawl endpoint (reader.infoquest.bytepluses.com)
    ├── web_search()         → /search endpoint (search.infoquest.bytepluses.com)
    ├── image_search()       → /search endpoint with search_type=Images
    └── clean_results()      → Result deduplication and normalization
```

#### Three LangChain Tools

```python
@tool("web_search", parse_docstring=True)
def web_search_tool(query: str) -> str

@tool("web_fetch", parse_docstring=True)
def web_fetch_tool(url: str) -> str  # Returns markdown via ReadabilityExtractor

@tool("image_search", parse_docstring=True)
def image_search_tool(query: str) -> str
```

#### Configuration Schema

```yaml
tools:
  - name: web_search
    group: web
    use: deerflow.community.infoquest.tools:web_search_tool
    search_time_range: 10  # days, -1 to disable

  - name: web_fetch
    use: deerflow.community.infoquest.tools:web_fetch_tool
    timeout: 10            # Overall crawling timeout (seconds)
    fetch_time: 10         # Wait after page load (seconds)
    navigation_timeout: 10 # Navigation timeout (seconds)

  - name: image_search
    use: deerflow.community.infoquest.tools:image_search_tool
    image_search_time_range: 10  # days, -1 to disable
    image_size: "i"  # "l" (large), "m" (medium), "i" (icon)
```

#### Interesting Patterns / Technical Debt

1. **Verbose Logging with Emoji**: Debug logs use emoji for configuration status
2. **Error String Returns**: The client returns error strings instead of raising exceptions, allowing errors as part of agent output
3. **No Rate Limiting**: No evidence of rate limiting or retry logic
4. **API Key Optional**: The API key is optional (`INFOQUEST_API_KEY` env var)
5. **ReadabilityExtractor Coupling**: `web_fetch_tool` uses `ReadabilityExtractor` to convert HTML to markdown (line 73-74). Content is truncated to 4096 characters.

---

### 14. Frontend UI/UX

**Priority:** SECONDARY
**Location:** `frontend/src/app/workspace/`, `frontend/src/components/workspace/`

#### Overview

The DeerFlow frontend is a Next.js 16 application with React 19 that provides the workspace interface for interacting with the AI agent. The UI centers around thread-based conversations with streaming responses, artifact rendering, file uploads, and multi-mode agent interaction.

#### Architecture

```
/workspace
├── page.tsx                 → Redirects to /workspace/chats/new or /workspace/chats/[thread_id]
├── layout.tsx               → WorkspaceLayout with sidebar, QueryClient, Toaster
└── chats/
    ├── page.tsx             → Chat list with search
    └── [thread_id]/
        └── page.tsx         → Individual chat thread

/components/workspace/
├── workspace-container.tsx  → WorkspaceContainer, WorkspaceHeader, WorkspaceBody
├── workspace-sidebar.tsx    → Sidebar with nav, recent chats, menu
├── input-box.tsx            → Message input with mode selector
├── chats/                   → ChatBox, useThreadChat, useChatMode
├── messages/                → MessageList, MessageListItem, MessageGroup
├── artifacts/               → ArtifactTrigger, ArtifactFileList, ArtifactFileDetail
├── welcome.tsx              → Welcome banner with mode-specific styling
├── todo-list.tsx            → Todo list rendering
├── recent-chat-list.tsx     → Recent chats sidebar section
└── settings/                → Settings components
```

#### Multi-Mode System

| Mode | thinking_enabled | is_plan_mode | subagent_enabled | reasoning_effort |
|------|------------------|--------------|-----------------|------------------|
| flash | false | false | false | minimal |
| thinking | true | false | false | low |
| pro | true | true | false | medium |
| ultra | true | true | true | high |

#### useThreadStream Hook

The core streaming hook (hooks.ts:58-411):
- Creates/manages thread lifecycle via LangGraph SDK
- Handles optimistic message updates (immediate UI feedback before server response)
- Manages file uploads before message submission
- Streams events from the backend
- Handles errors with toast notifications

#### InputBox Component

At 915 lines, `input-box.tsx` handles:
- Mode selection (flash, thinking, pro, ultra)
- Reasoning effort selector (minimal, low, medium, high)
- Model selector with search
- File attachments
- Follow-up suggestions
- Submit/stop button with status
- Confirm dialog for replacing/appending suggestions

#### Technical Debt / Concerns

1. **Large InputBox Component**: At 915 lines, `input-box.tsx` is doing too much. Consider splitting.
2. **Optimistic Message IDs**: Uses `Date.now()` for optimistic message IDs which could theoretically collide
3. **History API for Navigation**: Instead of Next.js router (which would cause re-mount), uses native history API for thread creation
4. **Dual Thread ID Tracking**: Both `threadIdRef` and `onStreamThreadId` track thread IDs, with complex synchronization
5. **File Upload in Message Flow**: Files are uploaded before message is sent. If thread creation fails after upload, there's no cleanup.
6. **No Error Boundary**: No React error boundary visible, so any render error would crash the entire workspace.

---

## Cross-Cutting Observations

### Shared Architectural Patterns

1. **Atomic file operations**: Both memory system and channel store use temp file + rename pattern for safe writes
2. **Async pub/sub decoupling**: MessageBus pattern used for channel-agent communication
3. **Virtual path abstraction**: `/mnt/user-data/{workspace,uploads,outputs}` pattern used consistently across sandbox, skills, and artifacts
4. **Thread-isolated resources**: Uploads, artifacts, and sandbox all use thread ID for isolation
5. **Configuration-driven**: All features heavily configured via `config.yaml`
6. **Middleware chain pattern**: Context management composable via middleware (12 middleware in lead agent)
7. **Provider/plugin architecture**: Guardrails, MCP, and models all use provider protocol patterns
8. **Lazy initialization**: Model clients, LangGraph clients all lazy-created

### Shared Technical Debt

1. **`print()` for logging** in `queue.py` (memory system) - uses `print()` instead of proper logging
2. **No structured logging** - uses basic `logger.info/warning/error` without contextual fields
3. **Singleton patterns** - `get_memory_queue()`, `get_channel_service()` - harder to test
4. **No graceful degradation** - If LangGraph Server is down, all channel messages fail with no queue/retry
5. **Global state** - Module-level dicts with simple locks for background tasks

### Integration Points

1. **Skills + Subagents**: Subagent prompt includes skills section via `get_skills_prompt_section()`
2. **Skills + Sandbox**: Skills mounted at `/mnt/skills/{category}/{skill_name}` in container
3. **Subagents + Sandbox**: Subagents inherit parent's `sandbox_state` and `thread_data`
4. **Guardrails evaluates tool calls** - Including `write_file` which creates artifacts
5. **Artifacts served via Gateway** - Same API surface as uploads and other thread resources
6. **Skills can be artifacts** - `.skill` files are ZIPs that can be installed from artifact list
7. **DeerFlowClient uses same agent building blocks**: `_build_middlewares()`, `apply_prompt_template()`, etc.

### Architecture Strengths

1. **Clean separation**: Harness (`deerflow.*`) vs App (`app.*`) boundary enforced
2. **Plugin architecture**: New channels, tools, and models addable via registry + config
3. **Middleware chain**: Context management composable via middleware pattern
4. **Model factory pattern**: Single entry point for all model instantiation
5. **Virtual path abstraction**: Enables same agent code to run in local, Docker, or K8s environments

---

## File Reference by Feature

| Feature | Primary Files |
|---------|--------------|
| Skills & Tools | `skills/types.py`, `skills/loader.py`, `skills/parser.py`, `skills/validation.py`, `skills/installer.py`, `tools/tools.py`, `tools/builtins/task_tool.py` |
| Sub-Agent Orchestration | `subagents/executor.py`, `subagents/config.py`, `subagents/registry.py`, `subagents/builtins/general_purpose.py`, `subagents/builtins/bash_agent.py`, `agents/lead_agent/agent.py` |
| Sandbox | `sandbox/sandbox.py`, `sandbox/sandbox_provider.py`, `sandbox/local/local_sandbox.py`, `sandbox/local/local_sandbox_provider.py`, `sandbox/middleware.py`, `sandbox/tools.py`, `community/aio_sandbox/aio_sandbox_provider.py` |
| Context Engineering | `agents/memory/prompt.py`, `agents/memory/updater.py`, `agents/memory/queue.py`, `agents/middlewares/memory_middleware.py`, `agents/middlewares/uploads_middleware.py` |
| Long-Term Memory | `agents/memory/updater.py`, `docs/MEMORY_IMPROVEMENTS.md`, `docs/MEMORY_IMPROVEMENTS_SUMMARY.md` |
| Multi-Channel IM | `channels/base.py`, `channels/message_bus.py`, `channels/manager.py`, `channels/store.py`, `channels/service.py`, `channels/telegram.py`, `channels/slack.py`, `channels/feishu.py` |
| Model Agnostic | `models/factory.py`, `models/claude_provider.py`, `models/credential_loader.py`, `models/openai_codex_provider.py` |
| Claude Code CLI | `skills/public/claude-to-deerflow/SKILL.md`, `skills/public/claude-to-deerflow/scripts/chat.sh`, `frontend/src/components/ui/terminal.tsx` |
| Python Client | `client.py`, `tests/test_client.py`, `tests/test_client_live.py` |
| Guardrails | `guardrails/provider.py`, `guardrails/middleware.py`, `guardrails/builtin.py`, `docs/GUARDRAILS.md` |
| Gateway API | `gateway/app.py`, `gateway/routers/` (models.py, mcp.py, skills.py, memory.py, uploads.py, artifacts.py, threads.py, agents.py, suggestions.py, channels.py), `docs/API.md` |
| Artifact System | `gateway/routers/artifacts.py`, `frontend/src/components/workspace/artifacts/`, `frontend/src/components/ai-elements/code-block.tsx`, `frontend/src/components/ai-elements/web-preview.tsx`, `frontend/src/components/workspace/code-editor.tsx` |
| InfoQuest | `community/infoquest/infoquest_client.py`, `community/infoquest/tools.py`, `tests/test_infoquest_client.py` |
| Frontend UI | `frontend/src/app/workspace/`, `frontend/src/components/workspace/`, `frontend/src/core/threads/hooks.ts`, `frontend/src/core/api/` |
