# Feature Deep-Dive: Batch 3

## Features Analyzed
- Feature 7: Natural Language Cron Scheduling
- Feature 8: Subagent Delegation & Parallelization
- Feature 9: Research-Ready RL Training Tools

---

## Feature 7: Natural Language Cron Scheduling

### Core Implementation Files
| File | Purpose |
|------|---------|
| `cron/jobs.py` | Job storage, schedule parsing, CRUD operations |
| `cron/scheduler.py` | Tick execution, job running, delivery |
| `tools/cronjob_tools.py` | Tool wrapper exposing cron management to agent |

### How It Works

#### Schedule Parsing (`cron/jobs.py`)
The system supports multiple schedule formats that are parsed into a canonical structure:

```python
# Supported formats -> parse_schedule returns:
"30m"              -> {"kind": "once", "run_at": "...", "display": "once in 30m"}
"every 30m"        -> {"kind": "interval", "minutes": 30, "display": "every 30m"}
"0 9 * * *"        -> {"kind": "cron", "expr": "0 9 * * *", "display": "0 9 * * *"}
"2026-02-03T14:00" -> {"kind": "once", "run_at": "...", "display": "once at 2026-02-03 14:00"}
```

Uses `croniter` library for cron expression validation and next-run computation.

#### Job Storage
- Jobs stored in `~/.hermes/cron/jobs.json`
- Output saved to `~/.hermes/cron/output/{job_id}/{timestamp}.md`
- Atomic writes with temp files + fsync for crash safety
- Permissions locked to owner-only (0700/0600)

#### Execution Flow (`cron/scheduler.py`)
1. `tick()` called every 60 seconds (from gateway background thread)
2. File lock prevents concurrent ticks (cross-process safety)
3. `get_due_jobs()` finds jobs where `next_run_at <= now`
4. For each due job:
   - Runs fresh `AIAgent` with the job's prompt
   - Supports skill loading before prompt execution
   - Output saved to markdown file
   - Delivery attempted to configured target (origin, local, or platform-specific)

#### Delivery System
Jobs can deliver output to:
- `local` - saves to file only
- `origin` - back to the chat where job was created
- `telegram`, `discord`, `slack`, `whatsapp`, `signal`, `matrix`, `mattermost`, `email`, `sms`, `homeassistant`, `dingtalk` - platform-specific delivery
- `platform:chat_id` or `platform:chat_id:thread_id` for explicit targeting

The `[SILENT]` marker allows the cron agent to suppress delivery when there's nothing noteworthy to report.

### Notable Code Patterns

#### Grace Period for Missed Jobs
```python
# Daily jobs can catch up if missed by up to 2 hours
# Hourly jobs catch up within 30 minutes
# Frequent jobs (every 5-10 min) fast-forward quickly
grace = max(MIN_GRACE, min(period_seconds // 2, MAX_GRACE))
```
Prevents cron storm on gateway restart after downtime.

#### Security Scanning
`tools/cronjob_tools.py` includes threat pattern detection:
- Prompt injection patterns (`ignore previous instructions`)
- Deception patterns (`do not tell the user`)
- Exfiltration (`curl ... $API_KEY`)
- SSH backdoors (`authorized_keys`)
- Destructive commands (`rm -rf /`)

Also detects invisible unicode characters (U+200B zero-width space, etc.) used in injection attacks.

#### Skill Loading at Runtime
```python
# Skills are loaded fresh when job runs, not when created
# This allows skill updates between scheduling and execution
parts.extend([
    f'[SYSTEM: The user has invoked the "{skill_name}" skill...]',
    "",
    content,
])
```

### Technical Debt / Concerns

1. **Timezone handling**: Naive timestamps from older jobs are interpreted as system-local time and converted. This can cause issues if system timezone changes.

2. **No scheduling of cron jobs from cron**: The tool explicitly warns "cron-run sessions should not recursively schedule more cron jobs" - but this is advisory, not enforced.

3. **No retry mechanism**: Failed jobs are marked as failed but not retried. If a job fails due to transient error, user must manually trigger.

4. **Delivery is fire-and-forget**: If delivery fails, the error is logged but there's no retry queue.

---

## Feature 8: Subagent Delegation & Parallelization

### Core Implementation Files
| File | Purpose |
|------|---------|
| `tools/delegate_tool.py` | Subagent spawning, credential resolution, execution |

### How It Works

#### Architecture
Each child agent gets:
- Fresh conversation (no parent history)
- Own `task_id` (own terminal session, file ops cache)
- Restricted toolset (blocked tools always stripped)
- Focused system prompt built from goal + context
- Independent iteration budget

The parent context only sees the delegation call and final summary - intermediate tool calls never enter parent context.

#### Blocked Tools
```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # no recursive delegation
    "clarify",         # no user interaction
    "memory",          # no writes to shared MEMORY.md
    "send_message",    # no cross-platform side effects
    "execute_code",    # children should reason step-by-step
])
```

#### Depth Limiting
```python
MAX_DEPTH = 2  # parent (0) -> child (1) -> grandchild rejected (2)
```
Prevents infinite delegation chains.

#### Credential Inheritance
Children can inherit from parent OR use delegation-specific credentials:
```python
# Config in ~/.hermes/config.yaml:
# delegation:
#   provider: openrouter
#   model: anthropic/claude-sonnet-4
```
This enables routing subagents to a different provider:model pair (e.g., cheap/fast model for subagents while parent runs on Nous Portal).

#### Execution Modes
1. **Single task**: `delegate_task(goal="...")` - runs directly on main thread (no thread pool overhead)
2. **Batch mode**: `delegate_task(tasks=[{goal, context, toolsets}, ...])` - up to 3 tasks in parallel via ThreadPoolExecutor

#### Progress Callback
Child tool calls are relayed to parent display via callback:
- CLI: prints tree-view lines above parent's delegation spinner
- Gateway: batches tool names and relays to parent's progress callback

### Notable Code Patterns

#### Global State Mutation Guard
```python
# AIAgent() calls get_tool_definitions() which overwrites
# model_tools._last_resolved_tool_names with child's toolset.
# This breaks parent if not restored.
_parent_tool_names = list(_model_tools._last_resolved_tool_names)
try:
    child = AIAgent(...)  # mutates global
    child._delegate_saved_tool_names = _parent_tool_names
finally:
    _model_tools._last_resolved_tool_names = _parent_tool_names  # restore
```

#### Interrupt Propagation
```python
# Children are registered with parent for interrupt handling
if hasattr(parent_agent, '_active_children'):
    with lock:
        parent_agent._active_children.append(child)
```

#### Tool Call Matching
```python
# Matches parallel tool calls with their results via tool_call_id
trace_by_id: Dict[str, Dict] = {}
for msg in messages:
    if msg["role"] == "assistant":
        for tc in msg.get("tool_calls", []):
            entry = {"tool": tc["function"]["name"], "args_bytes": len(...)}
            trace_by_id[tc["id"]] = entry
    elif msg["role"] == "tool":
        # Match by tool_call_id for parallel calls
        target = trace_by_id.get(msg["tool_call_id"])
```

### Technical Debt / Concerns

1. **MAX_CONCURRENT_CHILDREN = 3**: Hardcoded limit. If user passes 3 tasks, they run in parallel. But if they pass 5 tasks, only 3 run and 2 are silently dropped (line 445: `task_list = tasks[:MAX_CONCURRENT_CHILDREN]`).

2. **No timeout on child agents**: `_run_single_child()` has no explicit timeout. If a child agent hangs, it blocks the thread (single mode) or holds a thread in the pool indefinitely.

3. **Credential resolution is complex**: `_resolve_delegation_credentials()` handles multiple cases (base_url direct, provider resolution, inheritance). The complexity makes it fragile.

4. **Tool trace has no size limit**: In `_run_single_child()`, the `tool_trace` list accumulates all tool calls. For very long subagent runs, this could be large.

---

## Feature 9: Research-Ready RL Training Tools

### Core Implementation Files
| File | Purpose |
|------|---------|
| `rl_cli.py` | Dedicated CLI runner for RL workflows |
| `batch_runner.py` | Parallel batch trajectory generation |
| `trajectory_compressor.py` | Post-processing compression for training |
| `tools/rl_training_tool.py` | Tinker-Atropos integration tools |

### How It Works

#### RL Training Tools (`tools/rl_training_tool.py`)
Integrates with Tinker-Atropos for RL training:

**Environment Discovery** (AST-based scanning):
```python
# Scans tinker-atropos/tinker_atropos/environments/
# for classes inheriting from BaseEnv
for node in ast.walk(tree):
    if isinstance(node, ast.ClassDef):
        for base in node.bases:
            if base_name == "BaseEnv":
                # Extract name, description, config_class
```

**Training Pipeline** (3-process architecture):
1. `run-api` - Atropos API server
2. `launch_training.py` - Tinker trainer + inference server
3. `environment.py serve` - The RL environment

**Locked Configuration**:
```python
LOCKED_FIELDS = {
    "env": {
        "tokenizer_name": "Qwen/Qwen3-8B",
        "rollout_server_url": "http://localhost:8000",
        "total_steps": 2500,
        "steps_per_eval": 25,
        # ... infrastructure-tuned values
    },
    "tinker": {
        "lora_rank": 32,
        "learning_rate": 0.00004,
    }
}
```
Infrastructure fields cannot be changed by the model.

**Toolset**:
- `rl_list_environments` - AST scan for BaseEnv subclasses
- `rl_select_environment` - Load environment config
- `rl_get_current_config` / `rl_edit_config` - Modify non-locked fields
- `rl_start_training` - Spawn 3-process training pipeline
- `rl_check_status` - Rate-limited status (30 min between checks)
- `rl_test_inference` - Quick validation before training
- `rl_stop_training` / `rl_get_results` / `rl_list_runs`

#### Batch Runner (`batch_runner.py`)
Parallel batch processing of agent trajectories:

**Architecture**:
- `multiprocessing.Pool` for parallel batch processing
- Checkpointing for fault tolerance/resume
- JSONL output format

**Trajectory Format**:
```python
{
    "prompt_index": 0,
    "conversations": [...],  # from/value pairs
    "metadata": {"batch_num": 0, "timestamp": "...", "model": "..."},
    "completed": True,
    "partial": False,  # True if stopped due to invalid tool calls
    "api_calls": 15,
    "toolsets_used": ["terminal", "web"],
    "tool_stats": {tool: {"count": N, "success": N, "failure": N}},
    "tool_error_counts": {tool: failure_count}
}
```

**Tool Statistics Normalization**:
```python
# ALL_POSSIBLE_TOOLS auto-derived from model_tools.py
# Ensures consistent schema in HuggingFace datasets
# Tools not used get zero counts
for tool in ALL_POSSIBLE_TOOLS:
    normalized[tool] = tool_stats.get(tool, DEFAULT_TOOL_STATS)
```

**Smart Resume**:
```python
# Scans batch files by PROMPT TEXT content matching
# Allows recovery even if indices don't match
completed_prompts = set()
for entry in batch_file:
    if entry.get("failed"): continue
    for msg in entry["conversations"]:
        if msg["from"] == "human":
            completed_prompts.add(msg["value"])
```

**Corruption Filtering**:
```python
# Filters entries where model generated invalid tool names (hallucinations)
VALID_TOOLS = ALL_POSSIBLE_TOOLS
invalid_tools = [k for k in tool_stats.keys() if k not in VALID_TOOLS]
if invalid_tools:
    continue  # Skip corrupted entry
```

#### Trajectory Compressor (`trajectory_compressor.py`)
Post-processes trajectories to fit token budget:

**Compression Strategy**:
1. Protect first turns: system, human, first gpt, first tool
2. Protect last N turns (configurable, default 4)
3. Compress middle turns starting from 2nd tool response
4. Replace compressed region with LLM-generated summary
5. Keep remaining middle turns intact

**Token Counting**:
```python
# Uses HuggingFace tokenizer
tokenizer = AutoTokenizer.from_pretrained("moonshotai/Kimi-K2-Thinking")
count_tokens(text) -> len(tokenizer.encode(text))
```

**Compression Algorithm**:
```python
# Need net_savings >= tokens_to_save
# net_savings = (sum of N turns) - summary_target_tokens
target_tokens_to_compress = tokens_to_save + summary_target_tokens

# Accumulate turns until savings target met
for i in range(compress_start, compress_end):
    accumulated_tokens += turn_tokens[i]
    if accumulated_tokens >= target_tokens_to_compress:
        break
```

**Async Parallel Processing**:
```python
# Semaphore for rate limiting
semaphore = asyncio.Semaphore(max_concurrent_requests)

# Per-trajectory timeout
await asyncio.wait_for(
    self.process_entry_async(entry),
    timeout=per_trajectory_timeout  # default 5 min
)
```

### Notable Code Patterns

#### Rate Limiting for Status Checks
```python
MIN_STATUS_CHECK_INTERVAL = 30 * 60  # 30 minutes
# Prevents spamming expensive status checks
```

#### Ephemeral System Prompt
```python
# System prompt used during agent execution but NOT saved to trajectories
ephemeral_system_prompt=RL_SYSTEM_PROMPT
# Prevents RL training data pollution from prompt engineering
```

#### Reasoning Coverage Tracking
```python
# Tracks how many assistant turns have reasoning
has_scratchpad = "<REASONING_SCRATCHPAD>" in content
has_native_reasoning = bool(msg.get("reasoning", "").strip())
# Discards samples with zero reasoning (not useful for RL)
```

### Technical Debt / Concerns

1. **tinker-atropos is a git submodule**: The code references `TINKER_ATROPOS_ROOT = HERMES_ROOT / "tinker-atropos"` as a submodule. If submodule not initialized, tools fail with cryptic errors.

2. **No Windows support**: File locking uses `fcntl` (Unix-only) with `msvcrt` fallback (untested).

3. **Hardcoded paths**: `~/.hermes/cron/jobs.json`, `~/.hermes/cron/output/` are hardcoded. No environment variable override.

4. **RL tools require Python 3.11+**: `check_rl_python_version()` enforces this, but error message could be clearer.

5. **Batch runner disk I/O bottleneck**: Processing prompts sequentially within batch (line 429: `for prompt_index, prompt_data in prompts_to_process:`), then writing trajectories one-by-one to append-only JSONL.

6. **Trajectory compressor uses blocking LLM calls**: `_generate_summary()` is synchronous. There's an async version but sync version is used in `compress_trajectory()` without explanation.

7. **No validation of HuggingFace dataset format**: `trajectory_compressor.py` assumes `from`/`value` fields but doesn't validate or document this schema anywhere.

---

## Cross-Feature Insights

### Shared Patterns
1. **Atomic file operations**: Both cron (`save_jobs()`) and batch runner use temp-file + `os.replace()` for crash-safe writes
2. **Permission hardening**: Cron sets 0700/0600 on its directories and files
3. **Environment variable injection**: Both cron and delegation inject `HERMES_SESSION_*` env vars to pass context
4. **Checkpoint/resume architecture**: Batch runner and RL training both use checkpointing for fault tolerance

### Tension Points
1. **Context isolation vs. credential sharing**: Delegation inherits API keys from parent but runs in isolated context. Cron jobs run with fresh credentials. These represent two different security/trust models.

2. **Tool blocking**: Delegation blocks 5 tools, cron disables `cronjob`, `messaging`, `clarify` toolsets. But these blocklists must be kept in sync manually.

3. **RL training is heavyweight**: Requires 3 external processes, tinker-atropos submodule, TINKER_API_KEY, WANDB_API_KEY. The complexity means it will only work in specific environments.