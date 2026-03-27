# Feature Batch 1 Deep Dive

**Features:** Skills & Tools framework, Sub-Agent Orchestration, Sandbox Execution
**Repository:** `/Users/sheldon/Documents/claw/reference/deer-flow`
**Date:** 2026-03-26

---

## Feature 1: Skills & Tools Framework

### Overview

DeerFlow's Skills system provides an extensible framework for defining agent workflows, best practices, and resources. Skills are bundled as directories containing a `SKILL.md` file with YAML frontmatter metadata. The system supports both built-in public skills (in `skills/public/`) and custom user skills (in `skills/custom/`).

### Core Implementation

#### 1. Skill Data Model (`backend/packages/harness/deerflow/skills/types.py`)

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

#### 2. Skill Loading (`backend/packages/harness/deerflow/skills/loader.py`)

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

#### 3. Skill Parsing (`backend/packages/harness/deerflow/skills/parser.py`)

Simple regex-based frontmatter extraction (not full YAML parser):
```python
front_matter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n", content, re.DOTALL)
```

Extracts: `name`, `description`, `license`. Returns `Skill` with `enabled=True` as default (actual state comes from config).

#### 4. Skill Validation (`backend/packages/harness/deerflow/skills/validation.py`)

Validates `SKILL.md` frontmatter:
- Allowed properties: `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`, `version`, `author`
- Name rules: hyphen-case (lowercase, digits, hyphens), max 64 chars, no leading/trailing/consecutive hyphens
- Description rules: max 1024 chars, no angle brackets

#### 5. Skill Installation (`backend/packages/harness/deerflow/skills/installer.py`)

Installs skills from `.skill` (ZIP archive) files with security protections:
- **Zip bomb defense**: 512MB max uncompressed size with chunked reading
- **Path traversal prevention**: `PurePosixPath` normalization, rejects `..` in paths
- **Symlink handling**: Skips symlink entries instead of materializing them
- **Archive structure detection**: Supports both single-dir and flat archives (filters `__MACOSX`, `.DS_Store`)

```python
def safe_extract_skill_archive(zip_ref, dest_path, max_total_size=512*1024*1024):
    # Validates each entry, skips symlinks, enforces size limit
```

#### 6. Tool System (`backend/packages/harness/deerflow/tools/tools.py`)

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

#### 7. Built-in Tool Implementations (`backend/packages/harness/deerflow/tools/builtins/`)

- `task_tool.py` - Subagent delegation (see Subagent Orchestration)
- `present_file_tool.py` - Makes output files visible to user (only `/mnt/user-data/outputs`)
- `ask_clarification_tool.py` - Requests clarification (intercepted by ClarificationMiddleware)
- `view_image_tool.py` - Reads image as base64 (added only if model supports vision)
- `invoke_acp_agent_tool.py` - Invokes external ACP-compatible agents
- `tool_search.py` - Deferred MCP tool discovery

### Clever Solutions

1. **Cross-process config sync**: Always reads `extensions_config.json` from disk rather than using cached config, ensuring Gateway API changes are immediately visible to LangGraph Server.

2. **Path preservation style**: `_join_path_preserving_style()` maintains POSIX vs Windows path semantics when joining.

3. **Skill container paths**: Skills resolve to `/mnt/skills/{category}/{relative_path}` inside containers, decoupled from host layout.

### Technical Debt / Concerns

1. **Simple regex YAML parsing**: `parser.py` uses simple key-value splitting instead of proper YAML parser, which could fail on complex frontmatter.

2. **No skill version/content validation**: Only frontmatter is validated, not the actual skill content/guidance quality.

3. **Global skill state**: Enabled/disabled state is global, not per-thread or per-user.

### Error Handling

- Skill loading failures are logged but don't crash (returns empty list)
- Invalid frontmatter returns `None` from parser
- Missing skills directory returns empty list gracefully

---

## Feature 2: Sub-Agent Orchestration

### Overview

DeerFlow's Sub-Agent system allows the lead agent to decompose complex tasks and delegate to specialized sub-agents running in separate contexts. Sub-agents execute asynchronously in a thread pool, with the lead agent polling for completion.

### Core Implementation

#### 1. Subagent Types (`backend/packages/harness/deerflow/subagents/builtins/`)

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

#### 2. Subagent Executor (`backend/packages/harness/deerflow/subagents/executor.py`)

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

#### 3. Task Tool (`backend/packages/harness/deerflow/tools/builtins/task_tool.py`)

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

#### 4. Tool Filtering (`backend/packages/harness/deerflow/subagents/executor.py`)

```python
def _filter_tools(all_tools, allowed, disallowed):
    # Apply allowlist first, then denylist
    # "task" is always disallowed to prevent recursive nesting
```

#### 5. Middleware Chain (`backend/packages/harness/deerflow/agents/lead_agent/agent.py`)

Subagent limit enforcement via `SubagentLimitMiddleware`:
```python
if subagent_enabled:
    max_concurrent_subagents = config.get("configurable", {}).get("max_concurrent_subagents", 3)
    middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))
```

Middleware execution order (from `agent.py` comment):
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

#### 6. Lead Agent Prompt (`backend/packages/harness/deerflow/agents/lead_agent/prompt.py`)

Subagent section with concurrency limit baked into prompt:
```
**HARD CONCURRENCY LIMIT: MAXIMUM 3 `task` CALLS PER RESPONSE**
- If count > 3: Pick 3 most important for this turn
- Multi-batch: Launch 3 now, wait, launch next batch
```

### Clever Solutions

1. **Async backend polling**: LLM doesn't poll - backend blocks on `time.sleep(5)` and streams SSE events. LLM just waits for `task_completed`.

2. **Tool call ID as task ID**: Uses `tool_call_id` from `@InjectedToolCallId` as `task_id`, enabling traceability from LLM call to background task.

3. **Dual pool design**: Scheduler pool accepts requests quickly, execution pool runs with proper timeout isolation.

4. **Subagent tool filtering**: Disallows `task` tool in subagents to prevent recursive nesting, while still allowing MCP/ACP tools.

5. **Content type handling**: Robust handling of AIMessage content as both `str` and `list` of content blocks.

### Technical Debt / Concerns

1. **Global task storage**: `_background_tasks` is a module-level dict with a simple lock - could have concurrency issues at scale.

2. **No task priority**: All tasks are equal priority, no preemption.

3. **Timeout as cancellation**: Uses `Future.cancel()` which is best-effort - task may still run to completion.

4. **Result deduplication**: Message deduplication by ID or full dict comparison could miss legitimate duplicates with different IDs.

5. **15-minute hard timeout**: `timeout_seconds=900` is not configurable per-task, only per subagent type globally.

### Error Handling

- Task not found: Returns error message
- Unknown subagent type: Returns error with available types
- Execution timeout: `FuturesTimeoutError` caught, status set to `TIMED_OUT`
- Exceptions in `_aexecute()`: Caught and returned as `FAILED` status
- `asyncio.run()` failures: Caught at outer level with fallback result

---

## Feature 3: Sandbox Execution Environment

### Overview

DeerFlow's Sandbox system provides isolated execution environments for agent operations. Supports local execution (singleton), Docker containers, and Kubernetes pods via provisioner service.

### Core Implementation

#### 1. Sandbox Interface (`backend/packages/harness/deerflow/sandbox/sandbox.py`)

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

#### 2. Sandbox Provider (`backend/packages/harness/deerflow/sandbox/sandbox_provider.py`)

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

#### 3. Local Sandbox (`backend/packages/harness/deerflow/sandbox/local/local_sandbox.py`)

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

#### 4. Local Sandbox Provider (`backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py`)

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

#### 5. Sandbox Tools (`backend/packages/harness/deerflow/sandbox/tools.py`)

**Path utilities**:
- `replace_virtual_path()`: Translates `/mnt/user-data/*` to actual thread paths
- `mask_local_paths_in_output()`: Hides host paths from user-visible output

**Virtual path mapping**:
```
/mnt/user-data/workspace -> thread_data['workspace_path']
/mnt/user-data/uploads -> thread_data['uploads_path']
/mnt/user-data/outputs -> thread_data['outputs_path']
/mnt/skills -> skills host path
/mnt/acp-workspace -> ACP workspace path
```

**Skills path resolution**:
```python
def _resolve_skills_path(path: str) -> str:
    # /mnt/skills/public/deep-research/SKILL.md -> /path/to/skills/public/deep-research/SKILL.md
```

**ACP workspace handling**:
- Per-thread workspace: `{base_dir}/threads/{thread_id}/acp-workspace/`
- Global fallback: `{base_dir}/acp-workspace/`
- Path traversal prevention via `is_relative_to()` check

#### 6. Sandbox Middleware (`backend/packages/harness/deerflow/sandbox/middleware.py`)

Lifecycle management:
```python
class SandboxMiddleware(AgentMiddleware):
    def __init__(self, lazy_init: bool = True):
        # lazy_init=True: Defer sandbox acquisition to first tool call
        # lazy_init=False: Acquire in before_agent()

    def after_agent(self, state, runtime):
        # Release sandbox after agent call
        # Note: LocalSandbox doesn't actually release (no-op)
```

**Thread-local sandbox reuse**: Sandbox is NOT released after each turn to avoid wasteful recreation.

#### 7. Kubernetes Sandbox (`backend/packages/harness/deerflow/community/aio_sandbox/`)

**Architecture**:
```
AioSandboxProvider (orchestration)
├── LocalContainerBackend (Docker mode)
└── RemoteSandboxBackend (K8s mode)
```

**Configuration**:
```yaml
sandbox:
  use: deerflow.community.aio_sandbox:AioSandboxProvider
  image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
  port: 8080
  replicas: 3
  container_prefix: deer-flow-sandbox
  idle_timeout: 600  # 10 minutes
  mounts:
    - host_path: /path/on/host
      container_path: /path/in/container
  environment:
    NODE_ENV: production
    API_KEY: $MY_API_KEY  # Resolved from host env
```

**Idle timeout with LRU eviction**:
```python
# Warm pool: released sandboxes whose containers are still running
# When replicas exceeded: evict least-recently-used
```

**Signal handling**: Registers `atexit` and signal handlers for graceful shutdown.

#### 8. Thread State (`backend/packages/harness/deerflow/agents/thread_state.py`)

```python
class SandboxState(TypedDict):
    sandbox_id: NotRequired[str | None]

class ThreadDataState(TypedDict):
    workspace_path: NotRequired[str | None]
    uploads_path: NotRequired[str | None]
    outputs_path: NotRequired[str | None]

class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]  # Deduplicates
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]  # Clears on {}
```

### Clever Solutions

1. **Virtual path abstraction**: Agent always sees `/mnt/user-data/*` and `/mnt/skills`, never host paths. Enables same agent code to run in local, Docker, or K8s environments.

2. **Bidirectional path translation**: Commands execute with translated paths, but output has paths re-translated back to virtual paths. User never sees host layout.

3. **Lazy sandbox acquisition**: `lazy_init=True` defers sandbox creation until first tool call, avoiding waste for simple queries.

4. **ACP workspace per-thread isolation**: Each thread gets its own ACP workspace at `{base_dir}/threads/{thread_id}/acp-workspace/`.

5. **Path traversal prevention**: Uses `PurePosixPath` normalization and `is_relative_to()` checks to prevent directory escapes.

6. **LRU warm pool**: Released sandboxes stay warm for `idle_timeout` seconds, allowing fast reuse while still cleaning up eventually.

### Technical Debt / Concerns

1. **10-minute command timeout**: Hard-coded `timeout=600` in `execute_command()` - not configurable.

2. **Singleton local sandbox**: No true isolation between threads in local mode - all share same filesystem.

3. **Path mapping at construction**: `_path_mappings` is set once at provider creation, not dynamically for thread.

4. **No network isolation**: Local mode has no network namespace isolation.

5. **Process shell execution**: Uses `shell=True` which has security implications (though path translation provides some protection).

6. **Global ACP workspace cache**: `_get_acp_workspace_host_path()` caches global workspace path - could be stale if directory created after cache set.

### Error Handling

- File not found: `OSError` re-raised with original path (not resolved path)
- Path traversal detected: `PermissionError` raised
- Sandbox not found: Returns `None` from `get()`
- Command timeout: `subprocess.run()` raises `TimeoutExpired`
- Shell not found: `RuntimeError` with clear message

---

## Cross-Feature Integration

### Skills + Subagents
- Subagent prompt includes skills section via `get_skills_prompt_section()`
- Skills provide domain-specific guidance that subagents inherit

### Skills + Sandbox
- Skills mounted at `/mnt/skills/{category}/{skill_name}` in container
- Path mapping established at sandbox creation

### Subagents + Sandbox
- Subagents inherit parent's `sandbox_state` and `thread_data`
- Share same filesystem paths (`/mnt/user-data/*`)
- Each gets isolated ACP workspace

---

## Key Files Reference

| Feature | File | Purpose |
|---------|------|---------|
| Skills | `skills/types.py` | `Skill` dataclass |
| Skills | `skills/loader.py` | `load_skills()` function |
| Skills | `skills/parser.py` | Frontmatter parsing |
| Skills | `skills/validation.py` | Frontmatter validation |
| Skills | `skills/installer.py` | `.skill` archive installation |
| Tools | `tools/tools.py` | `get_available_tools()` |
| Tools | `tools/builtins/task_tool.py` | `task` tool implementation |
| Subagents | `subagents/executor.py` | `SubagentExecutor` with thread pools |
| Subagents | `subagents/config.py` | `SubagentConfig` dataclass |
| Subagents | `subagents/registry.py` | `get_subagent_config()` |
| Subagents | `subagents/builtins/general_purpose.py` | General-purpose config |
| Subagents | `subagents/builtins/bash_agent.py` | Bash agent config |
| Sandbox | `sandbox/sandbox.py` | Abstract `Sandbox` interface |
| Sandbox | `sandbox/sandbox_provider.py` | Abstract `SandboxProvider` |
| Sandbox | `sandbox/local/local_sandbox.py` | Local filesystem implementation |
| Sandbox | `sandbox/local/local_sandbox_provider.py` | Local singleton provider |
| Sandbox | `sandbox/middleware.py` | Lifecycle middleware |
| Sandbox | `sandbox/tools.py` | Path translation utilities |
| Sandbox | `community/aio_sandbox/aio_sandbox_provider.py` | K8s/Docker provider |
| Agent | `agents/lead_agent/agent.py` | `make_lead_agent()` factory |
| Agent | `agents/thread_state.py` | `ThreadState` schema |
