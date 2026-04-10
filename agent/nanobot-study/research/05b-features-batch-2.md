# Feature Deep Dive - Batch 2: Features 4-6

## Features Covered
- **Feature 4**: Built-in Agent Tools (shell, filesystem, web, spawn, MCP, cron)
- **Feature 5**: Persistent Memory System
- **Feature 6**: Cron and Heartbeat

---

## Feature 4: Built-in Agent Tools

### Overview
The built-in tools provide the agent's capability to interact with the outside world. All tools follow a common `Tool` abstract base class pattern and are registered with a `ToolRegistry`.

### Key Files
| File | Purpose |
|------|---------|
| `nanobot/agent/tools/base.py` | Abstract `Tool` base class with param validation/casting |
| `nanobot/agent/tools/registry.py` | `ToolRegistry` for dynamic tool registration and execution |
| `nanobot/agent/tools/shell.py` | `ExecTool` for shell command execution |
| `nanobot/agent/tools/filesystem.py` | `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool` |
| `nanobot/agent/tools/web.py` | `WebSearchTool`, `WebFetchTool` |
| `nanobot/agent/tools/spawn.py` | `SpawnTool` for subagent creation |
| `nanobot/agent/tools/mcp.py` | `MCPToolWrapper` and `connect_mcp_servers()` |
| `nanobot/agent/tools/cron.py` | `CronTool` for scheduling |

### Tool Base Class (`base.py`)

The `Tool` ABC is well-designed with comprehensive parameter validation:

```python
class Tool(ABC):
    @property @abstractmethod def name(self) -> str: pass
    @property @abstractmethod def description(self) -> str: pass
    @property @abstractmethod def parameters(self) -> dict[str, Any]: pass
    @abstractmethod async def execute(self, **kwargs: Any) -> Any: pass
```

**Notable features:**
- **`cast_params()`**: Auto-converts string values to expected types (e.g., `"123"` -> `123` for integer params)
- **`validate_params()`**: Full JSON Schema validation including `enum`, `minimum`, `maximum`, `minLength`, `maxLength`
- **`to_schema()`**: Converts to OpenAI function-calling format
- **Nullable union handling**: `_resolve_type()` handles `["string", "null"]` patterns

**Code snippet** - parameter casting (`base.py` lines 106-109):
```python
if target_type == "integer" and isinstance(val, str):
    try:
        return int(val)
    except ValueError:
        return val
```

### Shell Tool (`shell.py`)

The `ExecTool` has substantial safety guardrails:

**Deny patterns** (regex-based, line 29-39):
```python
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",          # rm -r, rm -rf
    r"\bdel\s+/[fq]\b",              # del /f, del /q
    r"\bdd\s+if=",                   # dd operations
    r">\s*/dev/sd",                  # write to disk devices
    r"\b(shutdown|reboot|poweroff)\b",  # system power
    r":\(\)\s*\{.*\};\s*:",          # fork bomb pattern
    ...
]
```

**Safety features:**
1. **Deny patterns** - blocks dangerous commands via regex
2. **Allow patterns** - optional whitelist mode
3. **Path restriction** - `restrict_to_workspace=True` prevents directory traversal
4. **Internal URL detection** - prevents `curl`/`wget` to private addresses
5. **Path extraction** - extracts absolute paths and validates they're within allowed directory

**Implementation detail** - head+tail truncation for output (lines 139-146):
```python
max_len = self._MAX_OUTPUT  # 10,000 chars
if len(result) > max_len:
    half = max_len // 2
    result = result[:half] + f"\n\n... ({len(result) - max_len:,} chars truncated) ...\n\n" + result[-half:]
```

**Technical debt/quirks:**
- Uses `asyncio.create_subprocess_shell` which is vulnerable to shell injection if deny patterns are bypassed
- No timeout hard-cap beyond `_MAX_TIMEOUT = 600` seconds
- `os.WNOHANG` on Windows is problematic (line 117-121 has a Platform-specific workaround)

### Filesystem Tools (`filesystem.py`)

Four tools implemented: `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool`.

**Path resolution** (`_resolve_path()`):
- Relative paths resolved against workspace
- Enforces `allowed_dir` restriction (prevents escaping workspace)
- `extra_allowed_dirs` for additional permitted paths (used for builtin skills)

**ReadFileTool** (lines 59-151):
- Supports line-based pagination with `offset` and `limit`
- Auto-detects images via `detect_image_mime()` and returns image content blocks
- Returns numbered lines with format `{line_num}| {content}`
- Max output: 128,000 chars with automatic pagination hints
- Binary files rejected with error message

**EditFileTool** (lines 225-318):
- **Two-phase matching**: exact match first, then line-trimmed sliding window
- Handles CRLF line-ending normalization transparently
- When multiple matches exist, requires `replace_all=True` or more context
- **Fuzzy matching fallback**: Uses `difflib.SequenceMatcher` to suggest closest match when exact not found
- Unified diff output for best-match display (lines 311-317)

```python
# Fuzzy match example from _not_found_msg()
ratio = difflib.SequenceMatcher(None, old_lines, lines[i : i + window]).ratio()
if best_ratio > 0.5:
    diff = "\n".join(difflib.unified_diff(...))
    return f"Error: old_text not found... Best match ({best_ratio:.0%} similar)..."
```

**ListDirTool** (lines 325-411):
- Auto-ignores common noise directories: `.git`, `node_modules`, `__pycache__`, `.venv`, etc.
- Recursive listing via `Path.rglob("*")`
- Output uses emoji: `📁 ` for directories, `📄 ` for files
- Default max 200 entries with truncation message

### Web Tools (`web.py`)

**WebSearchTool**:
- Multi-provider with automatic fallback chain: `brave` -> `tavily` -> `searxng` -> `jina` -> `duckduckgo`
- Each provider has dedicated API key env var or config
- DuckDuckGo runs via `ddgs` library in thread pool (`asyncio.to_thread`)
- Result format: numbered list with title, URL, and snippet

**WebFetchTool**:
- Three-tier fetch strategy:
  1. **Image detection**: Streams response, checks `content-type`, returns image block if image
  2. **Jina Reader API**: `https://r.jina.ai/{url}` with API key support, rate-limit handling (429 -> fallback)
  3. **Readability fallback**: Local HTML parsing with `readability-lxml`, supports markdown conversion

**Security - SSRF Protection** (`network.py`):
```python
_BLOCKED_NETWORKS = [
    ipaddress.ip_network("0.0.0.0/8"),
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("127.0.0.0/8"),   # loopback
    ipaddress.ip_network("169.254.0.0/16"), # cloud metadata
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ...
]
```

- `validate_url_target()`: Checks scheme, hostname, and resolves + validates all IPs
- `validate_resolved_url()`: For already-fetched URLs (post-redirect), validates only IP
- `contains_internal_url()`: Scans command strings for URLs targeting private addresses

### Spawn Tool (`spawn.py`)

The `SpawnTool` creates background subagents via `SubagentManager`:

**Subagent lifecycle** (`subagent.py` lines 50-80):
1. Generates unique `task_id` (8-char UUID prefix)
2. Creates `asyncio.Task` for background execution
3. Returns immediately: "Subagent started... I'll notify you when it completes."
4. `done_callback` cleans up task from `_running_tasks` dict

**Subagent tools** (lines 93-108):
- Spawned agents get a restricted toolset: `ReadFile`, `WriteFile`, `EditFile`, `ListDir`, `Exec`, `WebSearch`, `WebFetch`
- **Missing**: No `SpawnTool` (prevents recursive spawning), no `MessageTool`
- `exec_config` and `restrict_to_workspace` settings inherited from parent

**Subagent execution** (lines 121-155):
- Max 15 iterations of LLM -> tool execution -> LLM loop
- No memory consolidation (independent, self-contained tasks)
- Result announced via `MessageBus` as `InboundMessage` from `"system"` channel with `"subagent"` sender

**Result announcement** (lines 168-198):
```python
announce_content = f"""[Subagent '{label}' {status_text}]
Task: {task}
Result: {result}
Summarize this naturally for the user. Keep it brief (1-2 sentences). Do not mention technical details..."""
```

### MCP Tool (`mcp.py`)

**Transport types supported** (lines 162-196):
1. **stdio**: Pipes to subprocess stdin/stdout via `StdioServerParameters`
2. **SSE**: Server-Sent Events via `sse_client` with custom `httpx.AsyncClient` factory
3. **streamableHttp**: Full HTTP streaming via `streamable_http_client` with explicit `httpx.AsyncClient`

**Schema normalization** (`_normalize_schema_for_openai()` lines 34-74):
- Handles nullable union types: `["string", "null"]` -> `{"type": "string", "nullable": true}`
- Extracts single non-null branch from `oneOf`/`anyOf` patterns
- Recursively normalizes nested `properties` and `items`

**Tool naming**: `mcp_{server_name}_{original_tool_name}` (e.g., `mcp_github_list_repos`)

**Error handling** (lines 109-127):
```python
except asyncio.TimeoutError:
    return f"(MCP tool call timed out after {self._tool_timeout}s)"
except asyncio.CancelledError:
    # Re-raise only if externally cancelled (e.g. /stop command)
    task = asyncio.current_task()
    if task is not None and task.cancelling() > 0:
        raise
    return "(MCP tool call was cancelled)"
```

**Tool filtering** (lines 205-232):
- `enabledTools: ["*"]` = all tools allowed
- Otherwise only listed tools are registered
- Warns about unmatched `enabledTools` entries at startup

### Cron Tool (`cron.py`)

The `CronTool` is a thin wrapper around `CronService` with agent-facing UX:

**Actions**: `add`, `list`, `remove`

**Schedule types**:
- `every_seconds`: Interval-based recurring
- `cron_expr`: Standard cron with optional IANA timezone
- `at`: One-time ISO datetime, auto-deletes after run

**Notable implementation detail** - ContextVar for cron context (lines 20, 27-33):
```python
_in_cron_context: ContextVar[bool] = ContextVar("cron_in_context", default=False)

def set_cron_context(self, active: bool):
    return self._in_cron_context.set(active)

def reset_cron_context(self, token) -> None:
    self._in_cron_context.reset(token)
```
This prevents scheduling new cron jobs from within a cron job callback.

**Validation** (lines 36-43, 134-152):
- Timezone validated via `zoneinfo.ZoneInfo`
- `tz` only allowed with `cron_expr`, not `every` or `at`
- Naive ISO datetimes get default timezone appended

### ToolRegistry (`registry.py`)

Simple but effective plugin pattern:

```python
class ToolRegistry:
    def register(self, tool: Tool) -> None
    def unregister(self, name: str) -> None
    def get(self, name: str) -> Tool | None
    def has(self, name: str) -> bool
    def get_definitions(self) -> list[dict[str, Any]]  # OpenAI format
    async def execute(self, name: str, params: dict[str, Any]) -> Any
```

**Error injection** (lines 40, 53, 56):
```python
_HINT = "\n\n[Analyze the error above and try a different approach.]"
if result.startswith("Error"):
    return result + _HINT
```
All error results get a hint appended to encourage the agent to self-correct.

---

## Feature 5: Persistent Memory System

### Overview
The memory system uses a two-layer approach: `MEMORY.md` for long-term facts and `HISTORY.md` for a grep-searchable log. It was redesigned in v0.1.4 for reliability.

### Key Files
| File | Purpose |
|------|---------|
| `nanobot/agent/memory.py` | `MemoryStore`, `MemoryConsolidator` classes |
| `nanobot/utils/helpers.py` | Token estimation functions |

### Two-Layer Architecture (`MemoryStore`)

**Storage** (lines 75-101):
```python
self.memory_dir = workspace / "memory"
self.memory_file = self.memory_dir / "MEMORY.md"  # Long-term facts
self.history_file = self.memory_dir / "HISTORY.md"  # Searchable log
```

**`MEMORY.md`**: Full updated long-term memory as markdown, rewritten on each consolidation
**`HISTORY.md`**: Append-only log with timestamped entries, grep-searchable

### Token-Based Consolidation (`MemoryConsolidator`)

**Budget calculation** (`maybe_consolidate_by_tokens()`, lines 306-330):
```python
budget = context_window_tokens - max_completion_tokens - _SAFETY_BUFFER  # 1024 tokens
target = budget // 2
```
When prompt exceeds budget, consolidation begins. Target is 50% of budget to prevent thrashing.

**Consolidation boundary selection** (`pick_consolidation_boundary()`, lines 258-278):
- Walks from `last_consolidated` offset forward
- Only picks `user` message boundaries (natural conversation turn)
- Accumulates tokens until `tokens_to_remove` threshold met
- Falls back to last available boundary if no single boundary meets threshold

**Consolidation flow** (`consolidate()`, lines 114-199):
1. Formats messages into readable transcript
2. Calls LLM with `save_memory` tool (forced tool_choice)
3. **Fallback on error**: If LLM doesn't call tool after 3 consecutive failures, raw-archives messages
4. **Tool choice fallback**: If `tool_choice: forced` unsupported, retries with `auto`

```python
if response.finish_reason == "error" and _is_tool_choice_unsupported(response.content):
    logger.warning("Forced tool_choice unsupported, retrying with auto")
    response = await provider.chat_with_retry(
        messages=chat_messages,
        tools=_SAVE_MEMORY_TOOL,
        model=model,
        tool_choice="auto",  # Some providers don't support forced tool_choice
    )
```

**Raw archive fallback** (`_fail_or_raw_archive()`, lines 201-219):
```python
_consecutive_failures += 1
if _consecutive_failures < _MAX_FAILURES_BEFORE_RAW_ARCHIVE:  # 3
    return False
_raw_archive(messages)  # Dump raw messages without LLM summarization
return True  # "Success" even though we degraded
```

### Token Estimation (`helpers.py`)

**`estimate_message_tokens()`** (lines 179-214):
```python
def estimate_message_tokens(message: dict[str, Any]) -> int:
    parts = []
    if isinstance(content, str):
        parts.append(content)
    elif isinstance(content, list):
        for part in content:
            if part.get("type") == "text":
                parts.append(part.get("text", ""))
            else:
                parts.append(json.dumps(part))  # image, etc.
    # Also includes: name, tool_call_id, reasoning_content
    payload = "\n".join(parts)
    enc = tiktoken.get_encoding("cl100k_base")
    return max(4, len(enc.encode(payload)) + 4)
```

**`estimate_prompt_tokens_chain()`** (lines 217-236):
- First tries provider's own counter (`provider.estimate_prompt_tokens`)
- Falls back to `tiktoken` with full message chain + tool definitions
- Returns `(tokens, source)` tuple for debugging

### Session Integration

**`archive_messages()`** (lines 297-304):
- Retries consolidation up to `_MAX_FAILURES_BEFORE_RAW_ARCHIVE` times
- **Always returns `True`**: After max retries, raw-archives and considers it done
- This prevents infinite loops but means memory may degrade without user notification

### Key Insight: No Delete Operation
The memory system has no delete/redact capability. Facts persist indefinitely in `MEMORY.md` once written. This could be a limitation for privacy-sensitive data.

---

## Feature 6: Cron and Heartbeat

### Overview
Cron and Heartbeat are separate services sharing a similar pattern: time-based scheduling with file-based state persistence.

### Key Files
| File | Purpose |
|------|---------|
| `nanobot/cron/service.py` | `CronService` for scheduled jobs |
| `nanobot/cron/types.py` | `CronJob`, `CronSchedule`, `CronPayload`, etc. |
| `nanobot/heartbeat/service.py` | `HeartbeatService` for periodic wake-ups |
| `nanobot/utils/evaluator.py` | Post-run notification evaluation |

### Cron Service (`cron/service.py`)

**Schedule types** (`types.py`):
```python
@dataclass
class CronSchedule:
    kind: Literal["at", "every", "cron"]
    at_ms: int | None = None        # For "at": timestamp ms
    every_ms: int | None = None     # For "every": interval ms
    expr: str | None = None          # For "cron": cron expression
    tz: str | None = None            # Timezone for cron expressions
```

**Job state** (`CronJobState`):
```python
next_run_at_ms: int | None
last_run_at_ms: int | None
last_status: Literal["ok", "error", "skipped"] | None
last_error: str | None
run_history: list[CronRunRecord]  # Last 20 runs
```

**Timer mechanism** (lines 228-245):
```python
def _arm_timer(self) -> None:
    next_wake = self._get_next_wake_ms()  # Earliest next run
    if not next_wake or not self._running:
        return
    delay_ms = max(0, next_wake - _now_ms())
    async def tick():
        await asyncio.sleep(delay_s)
        if self._running:
            await self._on_timer()
    self._timer_task = asyncio.create_task(tick())
```
- **Single timer**: Only one `asyncio.Task` active at a time
- After execution, timer is re-armed for next job
- **On-timer approach** vs interval: Sleeps until next job due, then processes all due jobs

**Job execution** (lines 265-304):
1. Calls `on_job(job)` callback (main agent's handler)
2. Records start time, duration, status, error
3. **One-shot handling** (`at` kind): Disables job or deletes if `delete_after_run`
4. **Recurring**: Computes next run time and re-schedules

**Persistence** (lines 80-193):
- JSON file at configured `store_path`
- Auto-reloads if file modified externally (mtime check)
- Full job state saved including run history (last 20 entries)

**Public API**:
- `list_jobs(include_disabled=False)`
- `add_job(name, schedule, message, deliver, channel, to, delete_after_run)`
- `remove_job(job_id)`
- `enable_job(job_id, enabled)`
- `run_job(job_id, force=False)` - Manual execution
- `get_job(job_id)`
- `status()`

### Cron Next-Run Computation (`_compute_next_run()`)

```python
def _compute_next_run(schedule: CronSchedule, now_ms: int) -> int | None:
    if schedule.kind == "at":
        return schedule.at_ms if schedule.at_ms > now_ms else None
    if schedule.kind == "every":
        return now_ms + schedule.every_ms
    if schedule.kind == "cron":
        # Uses croniter library
        base_dt = datetime.fromtimestamp(now_ms / 1000, tz=ZoneInfo(tz))
        cron = croniter(schedule.expr, base_dt)
        next_dt = cron.get_next(datetime)
        return int(next_dt.timestamp() * 1000)
```

Uses `croniter` for cron expression parsing with timezone support.

### Heartbeat Service (`heartbeat/service.py`)

**Design philosophy** (docstring lines 40-51):
```
Phase 1 (decision): reads HEARTBEAT.md and asks the LLM — via a virtual
tool call — whether there are active tasks. This avoids free-text parsing
and the unreliable HEARTBEAT_OK token.

Phase 2 (execution): only triggered when Phase 1 returns ``run``.
```

**Two-phase execution**:

**Phase 1 - Decision** (`_decide()`, lines 87-111):
```python
async def _decide(self, content: str) -> tuple[str, str]:
    response = await self.provider.chat_with_retry(
        messages=[
            {"role": "system", "content": "You are a heartbeat agent..."},
            {"role": "user", "content": f"Current Time: {current_time_str(self.timezone)}\n\nReview HEARTBEAT.md..."},
        ],
        tools=_HEARTBEAT_TOOL,  # {name: "heartbeat", action: "skip"|"run", tasks: str}
        model=self.model,
    )
    if not response.has_tool_calls:
        return "skip", ""
    return args.get("action", "skip"), args.get("tasks", "")
```

**Phase 2 - Execution** (`_tick()`, lines 145-177):
```python
async def _tick(self) -> None:
    content = self._read_heartbeat_file()
    action, tasks = await self._decide(content)

    if action != "run":
        logger.info("Heartbeat: OK (nothing to report)")
        return

    if self.on_execute:
        response = await self.on_execute(tasks)
        if response:
            should_notify = await evaluate_response(response, tasks, ...)
            if should_notify and self.on_notify:
                await self.on_notify(response)
```

**Notification gate** (`evaluate_response()` in `evaluator.py`):
```python
_EVALUATE_TOOL = {
    "name": "evaluate_notification",
    "parameters": {
        "properties": {
            "should_notify": {"type": "boolean"},
            "reason": {"type": "string"}
        },
        "required": ["should_notify"]
    }
}
```
Uses LLM to decide if result warrants user notification. Falls back to `True` (notify) on any error.

**Default interval**: 30 minutes (`30 * 60 = 1800 seconds`)

### Notification Silencing Pattern

Both cron and heartbeat use the same `evaluate_response()` pattern:
1. Agent executes task and returns result
2. LLM is asked: "Should user be notified?"
3. Only notify if `should_notify=True`

This prevents notification spam for routine/empty results.

---

## Cross-Cutting Observations

### Error Handling Patterns

1. **Graceful degradation**: Memory falls back to raw-archive after 3 failures; heartbeat returns `True` (notify) on errors
2. **Hint injection**: Tool errors get "try a different approach" hint appended
3. **Always return something**: Tool execute never raises; returns error string

### Security Considerations

1. **Shell injection**: `ExecTool` uses `create_subprocess_shell` which is inherently vulnerable to injection if deny patterns are bypassed
2. **SSRF protection**: Comprehensive blocked network list for URLs and resolved IPs
3. **Path traversal**: Filesystem tools enforce `allowed_dir` restrictions
4. **Cron job spawning**: Cannot schedule new jobs from within job execution (ContextVar guard)

### Technical Debt / Quirks

1. **Memory has no delete**: No mechanism to remove/redact facts from `MEMORY.md`
2. **Single timer for all cron jobs**: Inefficient if many jobs with varied schedules
3. **MCP CancelledError handling**: Complex logic to distinguish external cancellation vs SDK cancellation
4. **ToolRegistry.execute error concatenation**: Hints are appended multiple times if errors chain
5. **Heartbeat always reads file on tick**: Even when heartbeat is disabled or file missing (just logs debug and returns)
6. **No deduplication**: Two cron jobs with same schedule both run; no idempotency mechanism

### Code vs Documentation

The feature index claims:
- "Built-in Agent Tools: shell execution, file read/write/edit, web search, web fetch, message sending, MCP tools, cron scheduling, and spawn subagents"

**Verification**: All present. However, "message sending" is not a standalone tool - the message functionality is handled through the bus system, not a tool.

The feature index also claims memory is "Redesigned in v0.1.4 for reliability" - the two-layer architecture with raw-archive fallback confirms this redesign.
