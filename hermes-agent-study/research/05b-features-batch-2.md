# Feature Deep-Dive: Batch 2

## Feature 4: Multi-Model LLM Support (OpenRouter 200+ models)

### Core Implementation Files

- `agent/auxiliary_client.py` (~1642 lines) — Central provider router and client factory
- `agent/anthropic_adapter.py` — Anthropic API adapter
- `tools/openrouter_client.py` (~34 lines) — Thin OpenRouter client wrapper
- `hermes_cli/models.py` (~1196 lines) — Model CLI commands and provider management
- `hermes_constants.py` — `OPENROUTER_BASE_URL` constant

### How It Works

The multi-model support is built around a **centralized provider router** in `auxiliary_client.py` that acts as a single resolution chain for all LLM consumers (context compression, session search, web extraction, vision analysis, browser vision).

**Resolution Order (auto mode for text tasks):**
1. OpenRouter (via `OPENROUTER_API_KEY`) — 200+ models
2. Nous Portal (via `~/.hermes/auth.json` active provider)
3. Custom endpoint (`OPENAI_BASE_URL` + `OPENAI_API_KEY`)
4. Codex OAuth (Responses API via chatgpt.com with `gpt-5.2-codex`)
5. Native Anthropic
6. Direct API-key providers: z.ai/GLM, Kimi/Moonshot, MiniMax, MiniMax-CN
7. Falls back through `PROVIDER_REGISTRY`

**Key Architecture Patterns:**

**1. Adapter Pattern for Non-Standard APIs:**
The codebase wraps non-OpenAI-compatible APIs to present a uniform `.chat.completions.create()` interface:
- `_AnthropicCompletionsAdapter` — wraps Anthropic Messages API to look like chat.completions
- `_CodexCompletionsAdapter` — wraps Codex Responses API (different format) to look like chat.completions
- Both adapters return `SimpleNamespace` objects that mimic the OpenAI response structure

**2. Provider Router (`resolve_provider_client`):**
```python
def resolve_provider_client(
    provider: str,  # "openrouter", "nous", "openai-codex", "zai", "kimi-coding", "minimax", "custom", "auto"
    model: str = None,
    async_mode: bool = False,
    raw_codex: bool = False,
    explicit_base_url: str = None,
    explicit_api_key: str = None,
) -> Tuple[Optional[Any], Optional[str]]
```

**3. Thread-Safe Client Caching:**
Clients are cached with `(provider, async_mode, base_url, api_key, event_loop_id)` as the key, preventing cross-event-loop reuse issues (which cause `RuntimeError: Event loop is closed` with httpx/AsyncOpenAI).

**4. Per-Task Override System:**
Environment variables allow per-task provider and model overrides:
- `AUXILIARY_{TASK}_PROVIDER` / `CONTEXT_{TASK}_PROVIDER`
- `AUXILIARY_{TASK}_MODEL` / `CONTEXT_{TASK}_MODEL`
- `AUXILIARY_{TASK}_BASE_URL` / `AUXILIARY_{TASK}_API_KEY`

**5. OpenRouter Integration:**
- Base URL: `https://openrouter.ai/api/v1`
- Attribution headers in every request:
```python
_OR_HEADERS = {
    "HTTP-Referer": "https://hermes-agent.nousresearch.com",
    "X-OpenRouter-Title": "Hermes Agent",
    "X-OpenRouter-Categories": "productivity,cli-agent",
}
```

**6. Dual Token Handling:**
The system handles `max_tokens` vs `max_completion_tokens` differences between providers:
```python
def auxiliary_max_tokens_param(value: int) -> dict:
    # Only uses max_completion_tokens for direct OpenAI api.openai.com
    # All others (OpenRouter, Nous, Anthropic) use max_tokens
```

### Notable Code Patterns

- **Centralized LLM Call API** (`call_llm` / `async_call_llm`): Owns the full request lifecycle — resolution, client caching, request formatting, error handling. Every auxiliary consumer should use these, never construct clients ad-hoc.
- **Client shutdown** via `shutdown_cached_clients()`: Prevents `AsyncHttpxClientWrapper.__del__` errors during CLI shutdown
- **Codex JWT Expiry Check**: Access tokens are parsed for JWT expiry before use, preventing silent failures
- ** nous_auth reading**: Reads `~/.hermes/auth.json` for OAuth tokens

### Technical Debt / Concerns

1. **Complex fallback chain**: The auto-detection chain tries 7+ providers. When a provider fails silently, debugging which one was selected is non-trivial.
2. **Dual `max_tokens` handling**: `max_tokens` vs `max_completion_tokens` retry logic in `call_llm` suggests provider-specific quirks are still being discovered.
3. **Client cache key includes loop ID**: Async client caching is sophisticated but the loop-ID-based eviction logic adds complexity.
4. **Nous Portal detection**: Uses a global `auxiliary_is_nous` boolean flag set as a side effect of `_try_nous()` — not ideal for testability.
5. **OpenRouter-only for tools**: While the auxiliary client routes to many providers, the tool definitions for skills, memory, etc. route through OpenRouter specifically.

---

## Feature 5: Multi-Environment Execution Backends (6 backends)

### Core Implementation Files

- `tools/environments/base.py` (~100 lines) — Abstract `BaseEnvironment` class
- `tools/environments/local.py` (~477 lines) — Local host execution
- `tools/environments/ssh.py` (~233 lines) — SSH remote execution
- `tools/environments/docker.py` (~495 lines) — Docker container execution
- `tools/environments/daytona.py` (~251 lines) — Daytona cloud sandbox
- `tools/environments/modal.py` (~260 lines) — Modal cloud sandbox
- `tools/environments/singularity.py` (~370 lines) — Singularity/Apptainer containers
- `tools/environments/persistent_shell.py` (~278 lines) — Mixin for persistent shell mode

### How It Works

All backends implement a common interface from `BaseEnvironment`:

```python
class BaseEnvironment(ABC):
    @abstractmethod
    def execute(self, command: str, cwd: str = "", *,
                timeout: int | None = None,
                stdin_data: str | None = None) -> dict:
        """Returns {"output": str, "returncode": int}"""
        ...

    @abstractmethod
    def cleanup(self):
        """Release backend resources"""
        ...
```

**Shared Patterns Across Backends:**

**1. Output Fencing (Local, SSH):**
A unique fence marker (`__HERMES_FENCE_a9f7b3__`) wraps command output to isolate real output from shell init/exit noise. Falls back to pattern-based noise stripping when fences are missing.

**2. Sudo Password Handling:**
`_transform_sudo_command` (from `terminal_tool`) prepends sudo -S and pipes the password, keeping it out of process.argv visible to `ps`.

**3. Interrupt Support:**
Every backend polls `is_interrupted()` during execution loops, allowing users to cancel long-running commands.

**4. Persistent Shell Mixin (`PersistentShellMixin`):**
File-based IPC protocol where:
- Each command writes output to temp files (`{prefix}-stdout`, `{prefix}-stderr`, `{prefix}-status`)
- Polling reads status file until command ID appears
- Exponential backoff on poll intervals (10ms -> 250ms) to reduce I/O

---

### Backend-Specific Details

**LocalEnvironment:**
- Uses `bash -lic` (interactive login shell for full user environment)
- Hermes provider env vars are **blocked** from subprocess env (60+ vars blocked, dynamically derived from `PROVIDER_REGISTRY`)
- `_sanitize_subprocess_env()`: Filters out `_HERMES_*`, `OPENAI_*`, API keys, gateway config
- Windows: Uses Git Bash with path searching in common install locations

**SSHEnvironment:**
- ControlMaster for connection persistence (`ControlPath`, `ControlPersist=300`)
- File-based IPC for output capture (reads remote temp files via ControlMaster)
- BatchMode (no password prompts), StrictHostKeyChecking=accept-new

**DockerEnvironment:**
- Hardened security flags: `cap-drop ALL`, `no-new-privileges`, `pids-limit=256`
- tmpfs mounts for `/tmp`, `/var/tmp`, `/run` with size limits
- `forward_env` list: Only explicitly whitelisted env vars are passed into containers
- Auto-detects Docker Desktop on macOS at non-PATH locations
- Storage-opt disk quotas only work on `overlay2` driver with XFS pquota — probes and warns if unavailable
- `cleanup()` runs stop/rm in background threads so cleanup doesn't block

**DaytonaEnvironment:**
- Wraps Daytona Python SDK
- Persistent mode: sandbox.stop() on cleanup (not delete), resumes on next creation
- Legacy fallback: finds sandboxes by labels if name lookup fails (migration from old naming scheme)
- Uses shell `timeout` utility (not SDK timeout) as primary enforcement — SDK timeout was unreliable
- Thread-based execution with polling for interrupt support

**ModalEnvironment:**
- Uses `SWE-ReX` (`ModalDeployment`) directly
- `_AsyncWorker`: Background thread with its own event loop for async-safe swe-rex calls
- Persistent mode: snapshots filesystem via `sandbox.snapshot_filesystem.aio()`, stores snapshot ID in `modal_snapshots.json`
- One-time SIF building from docker:// URLs with caching in `APPTAINER_CACHEDIR`

**SingularityEnvironment:**
- Uses `apptainer` or `singularity` (auto-detected)
- `--containall --no-home` for full host isolation
- Persistent overlay stored in `{scratch}/hermes-overlays/overlay-{task_id}`
- Falls back to `writable-tmpfs` if instance start fails

### Notable Code Patterns

1. **Dynamic env blocklist**: `LocalEnvironment` builds its blocklist from `PROVIDER_REGISTRY` at runtime, so new providers are automatically covered without manual updates.
2. **Sandbox directory abstraction**: `get_sandbox_dir()` from `base.py` returns `{HERMES_HOME}/sandboxes/` by default, configurable via `TERMINAL_SANDBOX_DIR`.
3. **Windows Git Bash detection**: Searches 3 common install paths for `bash.exe`.
4. **Docker non-blocking cleanup**: Stop and remove commands run in background threads.
5. **Daytona SDK timeout workaround**: The SDK's timeout is not enforced server-side, so commands are wrapped with shell `timeout` utility.

### Technical Debt / Concerns

1. **Docker storage-opt probing**: The per-container disk quota check does a dry-run `docker create` on every `LocalEnvironment` instantiation — cached but still a subprocess call.
2. **Singularity SIF build**: One-time SIF build can take 10+ minutes with 600s timeout. Fallback to `docker://` URL if build fails, but that may not work with `apptainer exec`.
3. **Modal async worker thread**: `_AsyncWorker` uses `asyncio.run_coroutine_threadsafe` which is correct but complex; the `run_forever()` loop pattern is unusual.
4. **Daytona sandbox resume by name**: The legacy fallback (page.list with labels) suggests the naming migration wasn't fully atomic — old sandboxes may still exist.
5. **No resource cleanup guarantee**: `cleanup()` uses background threads/subprocess.Popen for non-blocking removal; if the process exits before cleanup runs, containers/images may be orphaned.

---

## Feature 6: Skills System with Procedural Memory

### Core Implementation Files

- `tools/skills_tool.py` (~1343 lines) — Skills list/view tools with progressive disclosure
- `tools/skill_manager_tool.py` (~666 lines) — Agent-managed skill create/edit/delete/patch
- `tools/skills_guard.py` (~1088 lines) — Security scanner for external skills
- `tools/skills_sync.py` — Skills sync functionality
- `tools/skills_hub.py` — Skills Hub integration
- `hermes_cli/skills_config.py` — Skills configuration management

### How It Works

**Three-Tier Progressive Disclosure:**

1. **Tier 0** (`skills_categories`): Category names + descriptions + skill counts — minimal token cost for discovery
2. **Tier 1** (`skills_list`): Name + description per skill — still metadata only
3. **Tier 2+** (`skill_view`): Full `SKILL.md` content, then linked files on demand

**SKILL.md Format (agentskills.io compatible):**
```yaml
---
name: skill-name              # Required, max 64 chars
description: Brief description # Required, max 1024 chars
version: 1.0.0               # Optional
platforms: [macos, linux]    # Optional — restricts skill to specific OS
prerequisites:                # Optional
  env_vars: [API_KEY]        #   Legacy, normalized to required_environment_variables
  commands: [curl, jq]
required_environment_variables:  # agentskills.io standard
  - name: OPENAI_API_KEY
    prompt: Enter your API key
    help: Get from platform.example.com
compatibility: Requires X    # Optional
metadata:
  hermes:
    tags: [fine-tuning, llm]
    related_skills: [peft, lora]
---

# Skill Title

Full instructions and content here...
```

**Directory Structure:**
```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # Main instructions (required)
│   ├── references/            # Supporting documentation
│   ├── templates/             # Output format templates
│   ├── scripts/               # Executable helpers
│   └── assets/               # Supplementary files (agentskills.io standard)
└── category/
    └── another-skill/
        └── SKILL.md
```

### Skill Manager Actions

The `skill_manage` tool supports 6 actions:
- `create` — New skill with full `SKILL.md` + optional category
- `edit` — Full SKILL.md rewrite
- `patch` — Targeted find-and-replace (unique match required unless `replace_all=true`)
- `delete` — Remove skill + clean up empty category dirs
- `write_file` — Add supporting file under `references/`, `templates/`, `scripts/`, or `assets/`
- `remove_file` — Remove a supporting file

### Security: Skills Guard (`skills_guard.py`)

**Trust Levels:**
- `builtin`: Ships with Hermes — never scanned, always trusted
- `trusted`: `openai/skills`, `anthropics/skills` — caution verdict allowed
- `community`: Everything else — any findings = blocked
- `agent-created`: Created by the agent itself — permissive policy

**Install Policy Matrix:**
| Source | safe | caution | dangerous |
|--------|------|---------|-----------|
| builtin | allow | allow | allow |
| trusted | allow | allow | block |
| community | allow | **block** | block |
| agent-created | allow | allow | ask |

**Scanning Categories:**
- **Exfiltration**: curl/wget fetching with secret env vars, credential file access (`~/.ssh`, `~/.aws`, `~/.hermes/.env`), DNS exfiltration, `os.environ` dumps
- **Injection**: `ignore previous instructions`, role hijacking, `disregard your rules`, hidden HTML (`<div style="display:none">`), invisible unicode (U+200B, etc.)
- **Destructive**: `rm -rf /`, `mkfs`, `dd` to `/dev/`
- **Persistence**: crontab, `.bashrc`, `authorized_keys`, systemd services, sudoers
- **Network**: reverse shells (`nc -l`, `socat`), tunneling services (ngrok, localtunnel), hardcoded IPs
- **Obfuscation**: base64 decode pipes, `eval()` strings, hex-encoded strings, `chr()` building
- **Credential Exposure**: hardcoded API keys, private keys, GitHub tokens
- **Jailbreaks**: DAN mode, developer mode, "for educational purposes only"

**Structural Checks:**
- Max 50 files per skill
- Max 1MB total size
- Max 256KB per single file
- Binary files (`*.exe`, `*.dll`, etc.) are flagged
- Symlinks pointing outside skill directory are blocked

**LLM Secondary Scan:**
After regex scanning, `llm_audit_skill()` runs the skill content through an LLM for contextual analysis that regexes miss. The LLM verdict can only raise severity, never lower it.

### Notable Code Patterns

1. **Atomic file writes**: `_atomic_write_text()` uses `tempfile.mkstemp` + `os.replace()` to ensure files are never partially written on crash.
2. **Frontmatter validation**: `yaml.safe_load` with fallback to simple key: value parsing for malformed YAML.
3. **Path traversal protection**: Both `skill_view` (for linked files) and `skill_manager_tool` (for `write_file`/`remove_file`) validate paths and resolve to ensure they don't escape the skill directory.
4. **Secret capture callback**: Skills requiring API keys can prompt the user via `_secret_capture_callback` (settable globally), with special handling for gateway surfaces (which can't capture secrets interactively).
5. **Remote env var passthrough**: When a skill has required env vars, `register_env_passthrough()` makes them available inside sandboxed execution (Docker, Modal, etc.).
6. **Platform filtering**: Skills declare `platforms: [macos, linux]` in frontmatter; `skill_matches_platform()` checks `sys.platform` prefix.

### Technical Debt / Concerns

1. **YAML frontmatter parsing**: The fallback to simple `key: value` splitting is fragile for complex YAML (lists, nested dicts). Works for simple skills but could silently drop structured data.
2. **Security scanner is regex-only with LLM backup**: The regex patterns are extensive but LLM audit is a best-effort secondary check. LLM audit failing silently (exception caught and swallowed) means some complex injections may get through.
3. **`agent-created` trust level**: The permissive "ask" policy for dangerous verdicts means agent-created skills with critical findings return `None` from `should_allow_install()`, requiring user confirmation before install. This is the intended design but could create confusing UX.
4. **Skills sync race conditions**: Multiple concurrent skill operations (create, edit, delete) could conflict if they race on the same skill directory.
5. **No skill versioning**: Skills are overwritten in place. No history/rollback mechanism.
6. **Skills installed from hub are community trust level**: Even popular hub skills are community-trusted, so any "caution" finding blocks them. This may be overly strict for well-maintained community skills.
