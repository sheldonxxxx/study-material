# Lessons Learned: Building Hermes Agent

**Assessed:** NousResearch/hermes-agent (as of 2026-03-27)
**Assessment Type:** Reverse engineering from codebase + contributing docs
**Purpose:** Derive actionable insights for building a similar project

---

## What to Emulate

### 1. Documentation-First Culture

**The CONTRIBUTING.md is 27KB of pure gold.** It includes:
- Architecture diagram with core loop
- Skill vs Tool decision framework (when to add which)
- Security considerations (shell injection, sudo, cron prompt injection)
- Cross-platform compatibility rules
- PR checklist that covers code, docs, config, architecture, and testing

**Action for my project:** Write CONTRIBUTING.md before writing code. It forces clarity about architecture decisions.

---

### 2. Registry/Plugin Pattern for Extensibility

Tools register themselves at import time via `registry.register()`. No code changes needed elsewhere to add a new tool.

```python
# tools/file_tools.py
registry.register(
    name="read_file",
    toolset="file",
    schema=READ_FILE_SCHEMA,
    handler=_handle_read_file,
    check_fn=_check_file_reqs,
    emoji="📖"
)
```

**Why it works:** New capabilities are self-registering. The agent discovers them at startup by importing all tool modules.

**Action for my project:** Use a registry pattern for all pluggable components (tools, platform adapters, skills).

---

### 3. Base Adapter Pattern for Multi-Platform Support

All 12 messaging platforms (Discord, Telegram, Slack, WhatsApp, Signal, Matrix, etc.) inherit from `BasePlatformAdapter`:

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool: ...
    @abstractmethod
    async def disconnect(self) -> None: ...
    @abstractmethod
    async def send(self, chat_id: str, content: str, ...) -> SendResult: ...
    @abstractmethod
    async def get_chat_info(self, chat_id: str) -> Dict[str, Any]: ...
```

**Action for my project:** Define the interface first, implement adapters second. Never duplicate platform-specific logic.

---

### 4. Security Scanning at Every Layer

**Multiple security checkpoints:**
- `skills_guard.py`: LLM secondary scan + regex patterns for injection, exfil, persistence
- `tirith_security.py`: Binary pre-execution scanning with auto-install + Cosign verification
- `supply-chain-audit.yml`: CI workflow scanning PRs for .pth files, base64+exec, marshal/pickle
- Memory/prompt injection scanning: `_MEMORY_THREAT_PATTERNS` regexes
- Context file threat scanning: blocks silent injection attempts

**Action for my project:** Security is not an afterthought. Build scanning into: skill installation, command execution, context file loading, memory writes, and CI.

---

### 5. Frozen Snapshot Pattern for Cache Stability

Memory writes during a session do NOT invalidate the cached system prompt:

```python
# Snapshot captured at load time
self._system_prompt_snapshot = {
    "memory": self._render_block("memory", self.memory_entries),
    "user": self._render_block("user", self.user_entries),
}

# Mid-session writes update files but NOT the snapshot
# Only refreshed on next session start
```

**Why it matters:** Anthropic prompt caching (prefix caching) depends on stable content. If you mutate the prefix mid-session, you lose cache benefits.

**Action for my project:** Any cached content must be snapshotted at session start, never mutated during the session.

---

### 6. Atomic File Operations

Every write uses temp file + `os.replace()`:

```python
fd, tmp_path = tempfile.mkstemp(dir=path.parent, suffix=".tmp", prefix=".mem_")
with os.fdopen(fd, "w", encoding="utf-8") as f:
    f.write(content)
    f.flush()
    os.fsync(f.fileno())
os.replace(tmp_path, str(path))  # Atomic on same filesystem
```

**Used in:** Memory stores, cron job files, config writes, batch runner output.

**Action for my project:** Never write directly to a file that might be read concurrently. Use atomic rename pattern for all persistent state.

---

### 7. Async Write Queue for Non-Blocking I/O

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
```

**Action for my project:** Offload non-critical I/O (analytics, memory writes, logging) to background threads with retry logic.

---

### 8. Progressive Disclosure for Expensive Operations

Skills load in tiers, not all at once:
- Tier 0: Category names + descriptions (zero token cost for discovery)
- Tier 1: Name + description per skill (still metadata only)
- Tier 2+: Full SKILL.md content loaded on demand

**Action for my project:** Never load expensive content (full skill docs, full memory, full context) until explicitly needed.

---

### 9. Context Files for Project Shaping

`.hermes.md` / `AGENTS.md` / `CLAUDE.md` files injected into system prompt to shape agent behavior per workspace.

Priority resolution: `.hermes.md` (git-root walk) > `AGENTS.md` (cwd) > `CLAUDE.md` (cwd) > `.cursorrules` (cwd)

**Action for my project:** Support context files that travel with the project (git-root level) and auto-load when agent operates in that directory.

---

### 10. Comprehensive Test Isolation

```python
# tests/conftest.py
@pytest.fixture(autouse=True)
def _isolate_hermes_home(tmp_path, monkeypatch):
    """Redirect HERMES_HOME to a temp dir so tests never write to ~/.hermes/."""

@pytest.fixture(autouse=True)
def _enforce_test_timeout():
    """Kill any individual test that takes longer than 30 seconds."""
    signal.alarm(30)
```

**Action for my project:** Every test fixture should redirect all filesystem/stateful operations to temp directories. Hard 30s timeout per test.

---

### 11. Skills as Markdown + Frontmatter

Skills are self-contained directories with SKILL.md + optional references/, templates/, scripts/, assets/.

Frontmatter format (agentskills.io compatible):
```yaml
---
name: skill-name
description: Brief description
platforms: [macos, linux]
prerequisites:
  env_vars: [API_KEY]
  commands: [curl, jq]
---
```

**Why it works:** Human-readable, git-friendly, easy to version control, no database needed.

**Action for my project:** Store skills as markdown files with YAML frontmatter, not in a database.

---

### 12. Multi-Backend Execution Strategy

Six terminal backends: local, Docker, SSH, Daytona, Singularity, Modal.

All implement the same interface:
```python
class BaseEnvironment(ABC):
    @abstractmethod
    def execute(self, command: str, cwd: str = "", *,
                timeout: int | None = None,
                stdin_data: str | None = None) -> dict: ...
```

**Action for my project:** Abstract execution environments behind a common interface. Swap backends without changing calling code.

---

## What to Avoid

### 1. No Linting/Formatting Tooling

The codebase has NO Ruff, Black, MyPy, or pre-commit hooks. The comment says "Code style is not enforced automatically; relies on developer discipline."

**Consequence:** Inconsistent formatting, type errors go undetected until runtime, no automated quality gates.

**Fix for my project:** Configure Ruff (linter + formatter) + MyPy from day one. Add pre-commit hooks.

---

### 2. Large Monolith Files

| File | Size |
|------|------|
| `run_agent.py` | 387KB |
| `cli.py` | 329KB |
| `gateway/run.py` | 262KB |
| `hermes_cli/main.py` | 165KB |
| `hermes_cli/setup.py` | 139KB |

**Consequence:** Impossible to navigate without IDE, hard to review PRs, single maintainer bottleneck.

**Fix for my project:** Hard limit: no file over 500 lines. Enforce with CI.

---

### 3. Some `print()` Calls in Production Code

```python
# Anti-pattern observed
except Exception as e:
    print(f"[{self.name}] Error handling message: {e}")
    import traceback
    traceback.print_exc()
```

**Fix for my project:** Audit for `print()` calls. Use `logger.exception()` instead of `traceback.print_exc()`.

---

### 4. Fail-Open Threat Scanning

If a context file matches threat patterns, content is simply NOT injected rather than raising an error or warning:

```python
# Context file injection
if threat_detected:
    pass  # Silently skips injection
```

**Consequence:** Prompt injection attempts go unnoticed.

**Fix for my project:** Fail-closed for security-critical scans. Log and notify user, not silently skip.

---

### 5. Character-Based Limits Instead of Token-Based

Memory uses `len()` (character count) not token count. The comment says "model-independent" but different models have different tokenizers.

```python
# From memory_tool.py
self._memory_limit = 2200  # chars, not tokens
```

**Fix for my project:** Use token-based limits with a tokenizer appropriate for the models being used.

---

### 6. No CODEOWNERS / MAINTAINERS

Single maintainer (teknium1) with 2,000+ commits/month. No formal governance.

**Consequence:** Bus factor risk, no secondary reviewers, hard to onboard contributors.

**Fix for my project:** Add CODEOWNERS file. Identify and nurture secondary maintainers.

---

### 7. Circular Import Risks

```python
# From honcho_integration/session.py
# Imports from `honcho` at type-check time only
if TYPE_CHECKING:
    from honcho import Honcho
# But get_honcho_client() does runtime import inside function
```

**Fix for my project:** Avoid circular imports by deferring runtime imports inside functions, not at module level.

---

## Surprises

### 1. Dataclasses Instead of Pydantic for Core Data

Despite Pydantic being a dependency, core gateway/agent code uses `dataclasses`:

```python
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
```

**Pydantic is only used in:** Environment modules, documentation examples.

**Insight:** Pydantic's validation overhead may not be needed for internal data structures where you control the source.

---

### 2. uv Instead of pip

The project uses [Astral's uv](https://github.com/astral-sh/uv) as the package manager, not pip.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Insight:** uv is 10-100x faster than pip. Worth adopting for large projects.

---

### 3. Skills Stored as Markdown Files

Not a database, not a registry service. Just files on disk:

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # Required
│   ├── references/           # Optional
│   ├── templates/           # Optional
│   └── assets/              # Optional
```

**Insight:** Simplest solution that works. Git-friendly, no backend needed.

---

### 4. Honcho is an External Service

The "persistent memory" feature requires an external `honcho-ai` service running:

```python
class HonchoSessionManager:
    def __init__(self, honcho: Honcho, context_tokens: int | None, config: HonchoClientConfig):
```

**Insight:** If Honcho is unavailable, graceful degradation means just logging warnings. The feature is additive, not core.

---

### 5. MCP Has Sophisticated Integration

- OAuth 2.1 PKCE support
- Server-initiated LLM requests (sampling/createMessage)
- Tool loop governance with `max_tool_rounds`
- Parallel server discovery via `asyncio.gather()`

**Insight:** MCP is more than a JSON-RPC wrapper. Study the full protocol before implementing.

---

### 6. Trajectory Compression for RL Training

The project generates training data for reinforcement learning:

```python
# trajectory_compressor.py
# Protect first turns: system, human, first gpt, first tool
# Protect last N turns (configurable, default 4)
# Compress middle turns starting from 2nd tool response
# Replace compressed region with LLM-generated summary
```

**Insight:** If you're building agent infrastructure, consider RL-training-ready data formats from the start.

---

### 7. Supply Chain Audit is a Dedicated CI Workflow

`.github/workflows/supply-chain-audit.yml` scans every PR for:
- `.pth` files (auto-execute on Python startup)
- base64 decode + exec/eval patterns
- marshal/pickle/compile usage

**Insight:** Supply chain attacks are real. Automated scanning is essential for any project handling external code.

---

### 8. Daytona/Modal for Serverless Persistence

Cloud sandboxes that hibernate when idle, resume on next request. No always-on servers.

**Insight:** For agent workloads, serverless containers with filesystem persistence are more cost-effective than always-on VMs.

---

### 9. Nix Flake for Reproducible Builds

```nix
# flake.nix
description = "Hermes Agent - Development environment"
```

**Insight:** Even if you don't use Nix for deployment, a flake.nix provides reproducible dev environments.

---

### 10. No Pre-Commit Hooks

Despite comprehensive contributing docs and CI workflows, there are no pre-commit hooks for:
- Code formatting before commit
- Linting before commit
- Running tests before commit

**Insight:** CONTRIBUTING.md documents the ideal, but CI is the only enforcement. Pre-commit hooks would catch issues earlier.

---

## Summary

| Category | Count |
|----------|-------|
| Emulation targets | 12 |
| Anti-patterns to avoid | 7 |
| Surprises | 10 |

**Top 3 priorities for my project:**
1. Set up Ruff + MyPy + pre-commit hooks before writing any code
2. Enforce 500-line file limit via CI
3. Build security scanning into skill installation, command execution, and CI
