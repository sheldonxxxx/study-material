# Feature Deep Dive - Batch 3: Scheduled Tasks, MCP Support, Security Features

## Table of Contents
1. [Scheduled Tasks (Heartbeat/Cron)](#1-scheduled-tasks-heartbeatcron)
2. [MCP (Model Context Protocol) Support](#2-mcp-model-context-protocol-support)
3. [Security Features (Tool Guard)](#3-security-features-tool-guard)

---

## 1. Scheduled Tasks (Heartbeat/Cron)

### Implementation Overview

The cron system uses **APScheduler** (`apscheduler.schedulers.asyncio.AsyncIOScheduler`) for scheduling, with a manager/executor pattern.

**Key Files:**
- `src/copaw/app/crons/manager.py` - CronManager orchestration
- `src/copaw/app/crons/heartbeat.py` - Heartbeat implementation
- `src/copaw/app/crons/executor.py` - Job execution
- `src/copaw/app/crons/models.py` - Pydantic data models
- `src/copaw/app/crons/api.py` - FastAPI routes
- `src/copaw/app/crons/repo/base.py` - Repository interface
- `src/copaw/app/crons/repo/json_repo.py` - JSON file persistence

### Architecture

```
CronManager
├── AsyncIOScheduler (timezone-aware)
├── CronExecutor (runs jobs)
├── JsonJobRepository (persistence)
└── Job state tracking (_states dict)
```

### Cron Expression Handling

**Notable:** Day-of-week conversion between crontab and APScheduler formats.

```python
# From models.py (lines 25-55)
_CRONTAB_NUM_TO_NAME: dict[str, str] = {
    "0": "sun", "1": "mon", "2": "tue", "3": "wed",
    "4": "thu", "5": "fri", "6": "sat", "7": "sun",
}

def _crontab_dow_to_name(field: str) -> str:
    """Convert the day-of-week field from crontab numbers to abbreviations.
    Handles: *, single values, comma-separated lists, and ranges.
    """
```

**Key insight:** APScheduler v3 uses ISO 8601 weekday numbering (0=Mon) while standard crontab uses (0=Sun). The code normalizes to three-letter abbreviations which are unambiguous in both systems.

### Interval Parsing

From `heartbeat.py` (lines 27-77):

```python
_EVERY_PATTERN = re.compile(
    r"^(?:(?P<hours>\d+)h)?(?:(?P<minutes>\d+)m)?(?:(?P<seconds>\d+)s)?$",
    re.IGNORECASE,
)

def parse_heartbeat_every(every: str) -> int:
    """Parse interval string (e.g. '30m', '1h') to total seconds."""
    # Supports: "30m", "1h", "2h30m", "90s"
    # Default: 30 minutes if invalid
```

### Heartbeat Dispatch

The heartbeat can send results to the "last active channel":

```python
# From heartbeat.py (lines 182-201)
target = (hb.target or "").strip().lower()
if target == HEARTBEAT_TARGET_LAST and last_dispatch:
    ld = last_dispatch
    if ld.channel and (ld.user_id or ld.session_id):
        async def _run_and_dispatch() -> None:
            async for event in runner.stream_query(req):
                await channel_manager.send_event(
                    channel=ld.channel,
                    user_id=ld.user_id,
                    session_id=ld.session_id,
                    event=event,
                    meta={},
                )
        await asyncio.wait_for(_run_and_dispatch(), timeout=120)
```

### Job Runtime Controls

```python
# From models.py (lines 101-104)
class JobRuntimeSpec(BaseModel):
    max_concurrency: int = Field(default=1, ge=1)
    timeout_seconds: int = Field(default=120, ge=1)
    misfire_grace_seconds: int = Field(default=60, ge=0)
```

### Active Hours Check

From `heartbeat.py` (lines 80-115):

```python
def _in_active_hours(active_hours: Any) -> bool:
    """Return True if the current time in user timezone is within [start, end]."""
    # Supports overnight ranges (e.g., 22:00-06:00)
    if start_t <= end_t:
        return start_t <= now <= end_t
    return now >= start_t or now <= end_t
```

### Notable Patterns

1. **Fire-and-forget async execution** - `run_job()` creates task with callback:
   ```python
   task = asyncio.create_task(self._execute_once(job), name=f"cron-run-{job_id}")
   task.add_done_callback(lambda t: self._task_done_cb(t, job))
   ```

2. **Auto-disable invalid jobs** - On startup, invalid jobs are automatically disabled and persisted:
   ```python
   if job.enabled:
       disabled_job = job.model_copy(update={"enabled": False})
       await self._repo.upsert_job(disabled_job)
   ```

3. **Error propagation to console** - Errors are pushed to the console push store:
   ```python
   error_text = f"Cron job [{job.name}] failed: {exc}"
   asyncio.ensure_future(push_store_append(session_id, error_text))
   ```

### Technical Debt / Concerns

1. **Repository loads entire file on every operation** - `upsert_job` calls `load()` then `save()` every time, no batching
2. **No locking for concurrent access** - JsonJobRepository notes "Single-machine, no cross-process lock"
3. **Heartbeat uses hardcoded 120s timeout** - Should be configurable

---

## 2. MCP (Model Context Protocol) Support

### Implementation Overview

MCP client management with hot-reload support. Uses `agentscope.mcp` library for actual MCP protocol handling.

**Key Files:**
- `src/copaw/app/mcp/manager.py` - MCPClientManager
- `src/copaw/app/mcp/watcher.py` - MCPConfigWatcher
- `src/copaw/app/routers/mcp.py` - FastAPI routes

### Architecture

```
MCPClientManager
├── _clients: Dict[str, Any]  (StdIOStatefulClient or HttpStatefulClient)
├── _lock: asyncio.Lock
└── Hot-reload via MCPConfigWatcher

MCPConfigWatcher
├── Polls for config changes (mtime + hash)
├── Per-client retry tracking
└── Non-blocking reload task
```

### Client Types Supported

```python
# From manager.py (lines 232-254)
if client_config.transport == "stdio":
    client = StdIOStatefulClient(
        name=client_config.name,
        command=client_config.command,
        args=client_config.args,
        env=client_config.env,
        cwd=client_config.cwd or None,
    )
else:  # streamable_http, sse
    client = HttpStatefulClient(
        name=client_config.name,
        transport=client_config.transport,
        url=client_config.url,
        headers=headers or None,
    )
```

### Hot-Reload Strategy

From `watcher.py` (lines 78-120):

```python
async def replace_client(self, key, client_config, timeout=60.0):
    """Replace or add a client with new configuration.

    Flow: connect new (outside lock) -> swap + close old (inside lock).
    This ensures minimal lock holding time.
    """
    # 1. Create and connect new client outside lock (may be slow)
    new_client = self._build_client(client_config)
    await asyncio.wait_for(new_client.connect(), timeout=timeout)

    # 2. Swap and close old client inside lock
    async with self._lock:
        old_client = self._clients.get(key)
        self._clients[key] = new_client
        if old_client is not None:
            await old_client.close()
```

### Retry Tracking

From `watcher.py` (lines 262-318):

```python
_client_failures: Dict[str, tuple[int, int]]  # {client_key: (retry_count, last_config_hash)}

def _should_skip_client(self, key, client_hash) -> bool:
    if key in self._client_failures:
        retry_count, last_hash = self._client_failures[key]
        if last_hash == client_hash and retry_count >= self._max_retries:
            return True  # Stop retrying same config
```

### Force Cleanup for Failed Connections

```python
# From manager.py (lines 179-216)
@staticmethod
async def _force_cleanup_client(client: Any) -> None:
    """Force-close a client whose connect() was interrupted.

    StatefulClientBase.close() refuses to run when is_connected is still False.
    We bypass that guard by closing the AsyncExitStack directly.
    """
    stack = getattr(client, "stack", None)
    if stack is None:
        return
    await stack.aclose()  # Triggers stdio_client finally-block
```

### Env Variable Expansion

```python
# From manager.py (lines 243-245)
headers = client_config.headers
if headers:
    headers = {k: os.path.expandvars(v) for k, v in headers.items()}
```

### Security: Env Value Masking

From `routers/mcp.py` (lines 132-160):

```python
def _mask_env_value(value: str) -> str:
    """Mask environment variable showing first 2-3 chars and last 4 chars."""
    # Example: sk-proj-1234567890abcdefghij1234 -> sk-****************************1234
    # Short values (<=8 chars): fully masked
```

### Notable Patterns

1. **Lazy singleton pattern** - MCPClientManager instantiated per-agent via context
2. **Non-blocking reload** - Config watcher triggers reload in background task, doesn't block polling
3. **Client rebuild info stored** - `_copaw_rebuild_info` attribute preserves config for hot-reload

### Technical Debt / Concerns

1. **No connection pooling** - Each client maintains its own connection
2. **60s default timeout** - Can be long for HTTP clients
3. **Watcher uses polling** - No filesystem watching (inotify/FSEvents), just 2-second polling interval

---

## 3. Security Features (Tool Guard)

### Implementation Overview

Two-layer security system:
1. **Tool Guard** - Pre-tool-call parameter validation
2. **Skill Scanner** - Pre-skill-install security scanning

**Tool Guard Key Files:**
- `src/copaw/security/tool_guard/engine.py` - ToolGuardEngine orchestrator
- `src/copaw/security/tool_guard/models.py` - Data models
- `src/copaw/security/tool_guard/guardians/file_guardian.py` - Path-based protection
- `src/copaw/security/tool_guard/guardians/rule_guardian.py` - Regex-based detection
- `src/copaw/security/tool_guard/utils.py` - Config resolution

**Skill Scanner Key Files:**
- `src/copaw/security/skill_scanner/scanner.py` - SkillScanner orchestrator
- `src/copaw/security/skill_scanner/models.py` - Data models
- `src/copaw/security/skill_scanner/scan_policy.py` - Policy management
- `src/copaw/security/skill_scanner/analyzers/pattern_analyzer.py` - YAML regex scanner

### Tool Guard Architecture

```
ToolGuardEngine (singleton)
├── FilePathToolGuardian (always_run=True)
│   └── Blocks access to sensitive files/directories
├── RuleBasedToolGuardian
│   └── Matches params against YAML regex rules
└── Scoped to configured tools (default: high-risk tools only)
```

### Guard Severity Levels

```python
# From models.py (lines 25-33)
class GuardSeverity(str, Enum):
    CRITICAL = "CRITICAL"
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"
    INFO = "INFO"
    SAFE = "SAFE"
```

### Threat Categories

```python
# From models.py (lines 36-52)
class GuardThreatCategory(str, Enum):
    COMMAND_INJECTION = "command_injection"
    DATA_EXFILTRATION = "data_exfiltration"
    PATH_TRAVERSAL = "path_traversal"
    SENSITIVE_FILE_ACCESS = "sensitive_file_access"
    NETWORK_ABUSE = "network_abuse"
    CREDENTIAL_EXPOSURE = "credential_exposure"
    RESOURCE_ABUSE = "resource_abuse"
    PROMPT_INJECTION = "prompt_injection"
    CODE_EXECUTION = "code_execution"
    PRIVILEGE_ESCALATION = "privilege_escalation"
```

### File Guardian - Path Extraction from Shell Commands

From `guardians/file_guardian.py` (lines 111-158):

```python
def _extract_paths_from_shell_command(command: str) -> list[str]:
    """Extract candidate file paths from a shell command string."""
    tokens = shlex.split(command, posix=True)

    # Handles: separated redirectors (cat a > out.txt)
    #          attached redirectors (>out.txt, 2>err.log)
    #          path tokens (looks_like_path_token heuristic)
```

### Path Normalization

```python
# From guardians/file_guardian.py (lines 46-51)
def _normalize_path(raw_path: str) -> str:
    """Normalize raw_path to canonical absolute path string."""
    p = Path(raw_path).expanduser()
    if not p.is_absolute():
        p = _workspace_root() / p
    return str(p.resolve(strict=False))
```

### Tool-Guard Config Resolution Priority

From `utils.py` (lines 63-95):

```python
def resolve_guarded_tools(user_defined=None) -> set[str] | None:
    """Priority:
    1) constructor-provided user_defined
    2) COPAW_TOOL_GUARD_TOOLS env var
    3) config.json -> security.tool_guard.guarded_tools
    4) built-in high-risk default set
    """
```

### Default Guarded Tools

```python
# From utils.py (lines 18-29)
_DEFAULT_GUARDED_TOOLS = frozenset({
    "execute_shell_command",
    "read_file",
    "write_file",
    "edit_file",
    "append_file",
    "send_file_to_user",
    "view_text_file",
    "write_text_file",
})
```

### Danger Shell Command Rules

From `rules/dangerous_shell_commands.yaml`:

```yaml
- id: TOOL_CMD_FS_DESTRUCTION
  severity: CRITICAL
  patterns:
    - "\\bmkfs(\\.[a-zA-Z0-9_]+)?\\b"
    - "\\bmke2fs\\b"
    - "\\bdd\\s+.*of=\\/dev\\/"
    - ">\\s*\\/dev\\/(sd[a-z][0-9]*|vd[a-z][0-9]*)"

- id: TOOL_CMD_PIPE_TO_SHELL
  severity: CRITICAL
  patterns:
    - "\\b(curl|wget)\\b\\s+.*\\|.*\\b(bash|sh|zsh|ash|dash)\\b"

- id: TOOL_CMD_REVERSE_SHELL
  severity: CRITICAL
  patterns:
    - "\\/dev\\/(tcp|udp)\\/"
    - "\\bnc\\s+.*-e\\s*\\S+"
```

### Skill Scanner - File Discovery Security

From `scanner.py` (lines 257-278):

```python
# Symlink traversal prevention
if p.is_symlink():
    logger.warning("Skipping symlink %s", p)
    continue

# Verify resolved path stays within skill directory
real = p.resolve(strict=True)
if not real.is_relative_to(skill_dir):
    logger.warning("Skipping file outside skill directory: %s -> %s", p, real)
    continue
```

### Skill Scanner - Result Caching

From `scanner.py` (lines 325-379):

```python
_scan_cache: dict[str, tuple[float, ScanResult]] = {}  # LRU cache (max 64 entries)

def _get_cached_result(skill_dir: Path) -> ScanResult | None:
    """Return cached ScanResult if directory hasn't changed (mtime-based)."""

def _store_cached_result(skill_dir: Path, result: ScanResult) -> None:
    """LRU eviction when exceeding _MAX_CACHE_ENTRIES (64)."""
```

### Skill Scanner - Scan Modes

From `scanner.py` (lines 82-107):

```python
_VALID_MODES = {"block", "warn", "off"}

def _get_scan_mode(cfg=None) -> str:
    """Priority: env COPAW_SKILL_SCAN_MODE > config > default 'warn'"""
```

### Skill Scanner - Policy System

From `scan_policy.py` - Comprehensive organizational policy support:

```python
@dataclass
class ScanPolicy:
    hidden_files: HiddenFilePolicy
    rule_scoping: RuleScopingPolicy      # Which rules fire on which files
    credentials: CredentialPolicy         # Suppress test credentials
    file_classification: FileClassificationPolicy
    file_limits: FileLimitsPolicy         # max_file_count, max_file_size
    analysis_thresholds: AnalysisThresholdsPolicy
    severity_overrides: list[SeverityOverride]
    disabled_rules: set[str]
```

### Skill Scanner - Rule Format

YAML-based rules in `rules/signatures/`:

```yaml
# Example from command_injection.yaml
- id: SHELL_PIPE_TO_BASH
  category: command_injection
  severity: HIGH
  patterns:
    - "\\|\\s*bash"
    - "\\|\\s*sh\\b"
  exclude_patterns:
    - "^#"  # Comments
  description: "Shell pipe to bash interpreter"
  remediation: "Inspect script before execution"
```

### Notable Patterns

1. **Guardian always_run flag** - FilePathToolGuardian runs even on non-guarded tools for path-level checks
2. **Tool rebuild info preservation** - `_copaw_rebuild_info` on clients enables hot-reload without full re-read
3. **Policy merge on load** - Custom policies deep-merge with defaults, only override needed sections
4. **Credential filtering** - Well-known test credentials (`sk-test-*`, `AKIA...`) are auto-suppressed

### Technical Debt / Concerns

1. **Tool guard doesn't block execution** - It only returns findings; blocking requires separate approval workflow
2. **No LLM-based analysis** - Only regex patterns (extensible interface exists for future)
3. **Skill scanner uses ThreadPoolExecutor** - Single-threaded scan in background, but still blocks if timeout is hit
4. **No rule hot-reload** - Tool guard rules require engine restart (though config can add custom rules)
5. **Whitelist uses content hash** - But hash is over all files, so any file change invalidates whitelist

---

## Summary Table

| Feature | Core Component | Scheduler/Library | Persistence | Hot-Reload |
|---------|---------------|-------------------|-------------|------------|
| Scheduled Tasks | CronManager | APScheduler | JSON file | Config-based |
| MCP Support | MCPClientManager | agentscope.mcp | Config file | Polling + hash |
| Tool Guard | ToolGuardEngine | N/A | Config only | No |
| Skill Scanner | SkillScanner | N/A | Config + cache | No |
