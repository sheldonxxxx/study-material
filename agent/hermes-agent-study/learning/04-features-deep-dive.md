# Hermes Agent Feature Deep-Dive

## Overview

This document synthesizes detailed analysis of all 13 features across 5 feature batches. Features are ordered by priority as defined in the feature index: core features first (learning loop, gateway, TUI, multi-model, execution backends, skills, cron), followed by secondary features (delegation, RL tools, memory, MCP, context files, migration).

---

## Core Features

### Feature 1: Self-Improving Agent with Learning Loop

**What it does:** Creates skills from experience, persists knowledge across sessions, and improves skills during use. Includes FTS5 session search with LLM summarization for cross-session recall.

**Key Architecture:**

The learning loop operates through three interconnected subsystems:

1. **Skills System (Procedural Memory)** — Markdown files with YAML frontmatter stored in `~/.hermes/skills/`. Progressive disclosure tiers minimize token cost: categories (Tier 0) → names+descriptions (Tier 1) → full content (Tier 2+). Skills follow the agentskills.io open standard.

2. **Honcho Integration (Persistent Memory)** — `HonchoSessionManager` provides cross-session memory via Honcho's AI-native memory system. Supports dialectic queries (LLM-powered queries about users), context prefetching, session migration, and memory file migration.

3. **Session-Based Learning** — The system observes user behavior across sessions and creates conclusions that inform future interactions via `create_conclusion()`.

**Notable Implementation Patterns:**

- **Async Write Queue**: Background daemon thread drains queue with retry logic (warn on first failure, error on retry failure, then drop)
- **Dynamic Reasoning Level Selection**: Query complexity (< 120 chars → default, 120-400 → one level up, > 400 → two levels up, capped at "high")
- **Skill Command Scanning**: `scan_skill_commands()` maps `/command-name` slash commands to skill files

**Technical Debt:**
- Circular import risk with honcho at type-check time
- Session key sanitization uses `[a-zA-Z0-9_-]` but keys may contain colons (`channel:chat_id`)
- Memory migration uses XML wrapping which is non-standard

---

### Feature 2: Multi-Platform Messaging Gateway

**What it does:** Unified gateway delivering conversations to 12+ platforms (Telegram, Discord, Slack, WhatsApp, Signal, DingTalk, Matrix, Mattermost, Email, SMS, Home Assistant, Webhook). Supports voice memo transcription and cross-platform continuity.

**Key Architecture:**

- **Base Adapter Pattern**: All platform adapters inherit from `BasePlatformAdapter` defining `connect()`, `disconnect()`, `send()`, `get_chat_info()`
- **Message Normalization**: All incoming messages normalized to `MessageEvent` dataclass with support for TEXT, LOCATION, PHOTO, VIDEO, AUDIO, VOICE, DOCUMENT, STICKER, COMMAND
- **Streaming via Edit Transport**: `GatewayStreamConsumer` bridges sync agent callbacks to async platform delivery using edit-message primitives

**Implementation Highlights:**

- **SSL Certificate Auto-Detection**: Searches 8 common distro/macOS cert locations before falling back to certifi
- **MarkdownV2 Escaping (Telegram)**: Regex-based escaping of 15 special characters
- **Message Truncation with Code Block Preservation**: Splits long messages while keeping code blocks intact by closing/reopening fences
- **Media Caching**: Images, audio, documents cached locally at `~/.hermes/image_cache/` for vision tool access
- **Interrupt Support**: Photo bursts queue without interrupting; other messages interrupt running agent

**Technical Debt:**
- Large file sizes: `gateway/run.py` (262KB), `telegram.py` (78KB), `discord.py` (94KB)
- Telegram polling conflict detection is fragile (class name + text content heuristic)
- Not all platforms support edit transport (Signal, Email, Home Assistant disable streaming)
- Allowlist configuration has no enforcement mechanism

---

### Feature 3: Terminal TUI with Streaming Output

**What it does:** Full terminal interface with multiline editing, slash-command autocomplete, conversation history, interrupt-and-redirect, and real-time streaming tool output.

**Key Architecture:**

- **Curses Checklist UI**: 141-line curses multi-select component with keyboard navigation and text fallback for non-curses terminals
- **Agent Streaming API**: Three streaming modes (Anthropic messages.stream, OpenAI streaming, Codex Responses streaming) via `_interruptible_streaming_api_call()`
- **KawaiiSpinner**: Animated thinking spinner with skin-aware faces and cycling verbs
- **Skin/Theme System**: Data-driven YAML theming via `Skin` class

**Implementation Highlights:**

- **Tool Preview Building**: Extracts primary argument from tool definitions to show compact previews
- **Dangerous Command Approval**: Pattern-matching detection for privilege escalation, database deletion, recursive force deletion, device writing
- **Stream Delta Callback Architecture**: `stream_delta_callback` and `_stream_callback` for display and TTS pipelines
- **Terminal Backend**: Routes to local, Docker, SSH, Daytona, Singularity, or Modal based on `TERMINAL_ENV`

**Technical Debt:**
- TUI is minimal (141 lines for curses component, not a full terminal interface)
- prompt_toolkit referenced but actual REPL/input handling is in separate files
- Stream callback race conditions between display and TTS consumers
- 6 backends with different failure modes adds complexity

---

### Feature 4: Multi-Model LLM Support

**What it does:** Use any model provider — Nous Portal, OpenRouter (200+ models), z.ai/GLM, Kimi/Moonshot, MiniMax, OpenAI, Anthropic, or custom endpoints. Switch models with `hermes model` command.

**Key Architecture:**

- **Centralized Provider Router**: `resolve_provider_client()` in `auxiliary_client.py` acts as single resolution chain for all LLM consumers
- **Resolution Order (auto mode)**: OpenRouter → Nous Portal → Custom endpoint → Codex OAuth → Native Anthropic → Direct API-key providers → PROVIBER_REGISTRY fallback

**Implementation Highlights:**

- **Adapter Pattern**: `_AnthropicCompletionsAdapter` and `_CodexCompletionsAdapter` wrap non-OpenAI-compatible APIs to present uniform `.chat.completions.create()` interface
- **Thread-Safe Client Caching**: Cache key includes `(provider, async_mode, base_url, api_key, event_loop_id)` preventing cross-event-loop reuse
- **Per-Task Override System**: `AUXILIARY_{TASK}_PROVIDER/MODEL/BASE_URL/API_KEY` env vars for fine-grained control
- **Dual Token Handling**: `auxiliary_max_tokens_param()` handles `max_tokens` vs `max_completion_tokens` differences
- **OpenRouter Attribution**: Required HTTP headers (`HTTP-Referer`, `X-OpenRouter-Title`, `X-OpenRouter-Categories`)

**Technical Debt:**
- Complex 7+ provider fallback chain makes debugging silent failures difficult
- `max_tokens` vs `max_completion_tokens` retry logic suggests ongoing provider quirks
- Client cache loop-ID-based eviction adds complexity
- Nous Portal detection uses global `auxiliary_is_nous` boolean (not testable)
- Tool definitions route through OpenRouter specifically despite auxiliary client supporting many providers

---

### Feature 5: Multi-Environment Execution Backends

**What it does:** Six terminal backends — local, Docker, SSH, Daytona, Singularity, and Modal. Daytona and Modal offer serverless persistence with hibernation.

**Key Architecture:**

All backends implement `BaseEnvironment` interface with `execute()` and `cleanup()` methods.

| Backend | Key Features |
|---------|-------------|
| Local | `bash -lic` (full user env), dynamic env blocklist from PROVIDER_REGISTRY, Windows Git Bash detection |
| SSH | ControlMaster for connection persistence, file-based IPC for output capture |
| Docker | Hardened security flags, tmpfs mounts, forward-env allowlist, non-blocking cleanup threads |
| Daytona | SDK wrapper, persistent mode (stop on cleanup, resume on create), shell timeout enforcement |
| Modal | SWE-ReX integration, async worker thread pattern, snapshot filesystem persistence |
| Singularity | Apptainer/Singularity auto-detect, host isolation, persistent overlay, SIF building from docker:// |

**Implementation Highlights:**

- **Output Fencing**: Unique fence marker wraps command output to isolate real output from shell init/exit noise
- **Sudo Password Handling**: `_transform_sudo_command` prepends `sudo -S` and pipes password
- **Interrupt Support**: Every backend polls `is_interrupted()` during execution
- **Persistent Shell Mixin**: File-based IPC with exponential backoff polling (10ms → 250ms)

**Technical Debt:**
- Docker storage-opt probing does dry-run `docker create` on every LocalEnvironment instantiation
- Singularity SIF build can take 10+ minutes with 600s timeout
- Modal async worker uses unusual `run_forever()` loop pattern
- Daytona sandbox resume by name uses legacy fallback suggesting non-atomic migration
- No resource cleanup guarantee if process exits before cleanup runs

---

### Feature 6: Skills System with Procedural Memory

**What it does:** Agent-curated skill creation after complex tasks. Skills self-improve during use. Compatible with agentskills.io open standard. Includes skill management, sync, and guardrails.

**Key Architecture:**

- **Three-Tier Progressive Disclosure**: categories (zero content) → names/descriptions (metadata only) → full SKILL.md + linked files
- **SKILL.md Format**: YAML frontmatter with `name`, `description`, `version`, `platforms`, `prerequisites`, `required_environment_variables`
- **Skill Manager Actions**: create, edit, patch, delete, write_file, remove_file

**Security (Skills Guard):**

Four trust levels with install policy matrix:

| Source | safe | caution | dangerous |
|--------|------|---------|-----------|
| builtin | allow | allow | allow |
| trusted | allow | allow | block |
| community | allow | **block** | block |
| agent-created | allow | allow | ask |

Scanning categories: Exfiltration, Injection, Destructive, Persistence, Network, Obfuscation, Credential Exposure, Jailbreaks

**Implementation Highlights:**

- **Atomic File Writes**: `tempfile.mkstemp` + `os.replace()` ensures never partially written
- **Frontmatter Validation**: `yaml.safe_load` with fallback to simple key:value parsing
- **Path Traversal Protection**: Both skill_view and skill_manager_tool validate and resolve paths
- **LLM Secondary Scan**: `llm_audit_skill()` provides contextual analysis beyond regex patterns

**Technical Debt:**
- YAML fallback parsing is fragile for complex structures
- LLM audit exceptions are caught and swallowed silently
- `agent-created` "ask" policy for dangerous verdicts requires user confirmation
- Skills sync has potential race conditions
- No skill versioning or rollback
- Hub-installed skills treated as community trust (may be overly strict)

---

### Feature 7: Natural Language Cron Scheduling

**What it does:** Built-in cron scheduler with platform delivery. Define daily reports, nightly backups, weekly audits in natural language.

**Key Architecture:**

- **Schedule Parsing**: Supports "30m", "every 30m", "0 9 * * *" (cron expressions), "2026-02-03T14:00" via `croniter`
- **Job Storage**: `~/.hermes/cron/jobs.json` with atomic writes, 0700/0600 permissions
- **Output Storage**: `~/.hermes/cron/output/{job_id}/{timestamp}.md`
- **Delivery Targets**: local, origin, telegram, discord, slack, whatsapp, signal, matrix, mattermost, email, sms, homeassistant, dingtalk

**Implementation Highlights:**

- **Grace Period for Missed Jobs**: Daily jobs catch up within 2 hours, hourly within 30 minutes
- **Security Scanning**: Threat pattern detection for injection, exfiltration, SSH backdoors, destructive commands, invisible unicode
- **Skill Loading at Runtime**: Skills loaded fresh when job runs (not when created) allowing updates between scheduling and execution

**Technical Debt:**
- Naive timestamps from older jobs interpreted as system-local time
- No scheduling of cron jobs from cron (advisory warning only)
- No retry mechanism for failed jobs
- Delivery is fire-and-forget with no retry queue

---

## Secondary Features

### Feature 8: Subagent Delegation & Parallelization

**What it does:** Spawn isolated subagents for parallel workstreams. Zero-context-cost turns via RPC collapse multi-step pipelines.

**Key Architecture:**

- Each child gets: fresh conversation, own task_id, restricted toolset, focused system prompt, independent iteration budget
- Parent only sees delegation call and final summary — intermediate tool calls never enter parent context
- `MAX_CONCURRENT_CHILDREN = 3` via ThreadPoolExecutor

**Blocked Tools:**
```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task", "clarify", "memory",
    "send_message", "execute_code"
])
MAX_DEPTH = 2  # parent -> child -> grandchild rejected
```

**Implementation Highlights:**

- **Global State Mutation Guard**: Saves/restores `_model_tools._last_resolved_tool_names` around child instantiation
- **Credential Inheritance**: Children can inherit parent credentials or use delegation-specific config
- **Interrupt Propagation**: Children registered with parent for collective interrupt handling
- **Tool Call Matching**: Matches parallel tool calls with results via `tool_call_id`

**Technical Debt:**
- `MAX_CONCURRENT_CHILDREN = 3` hardcoded; silently drops tasks beyond limit
- No timeout on child agents
- Credential resolution is complex and fragile
- Tool trace has no size limit

---

### Feature 9: Research-Ready RL Training Tools

**What it does:** Batch trajectory generation, Atropos RL environment integration, trajectory compression for training next-generation tool-calling models.

**Key Architecture:**

- **RL Training Tools**: AST-based environment discovery, locked configuration fields, 3-process training pipeline
- **Batch Runner**: `multiprocessing.Pool` for parallel processing, checkpointing, JSONL output
- **Trajectory Compressor**: Async parallel LLM summarization with semaphore rate limiting

**Trajectory Format:**
```python
{
    "prompt_index": 0,
    "conversations": [...],  # from/value pairs
    "metadata": {...},
    "completed": True,
    "partial": False,
    "api_calls": 15,
    "toolsets_used": ["terminal", "web"],
    "tool_stats": {tool: {"count": N, "success": N, "failure": N}},
}
```

**Compression Strategy:**
- Protect first turns (system, human, first gpt, first tool)
- Protect last N turns (default 4)
- Compress middle starting from 2nd tool response
- Replace compressed region with LLM-generated summary

**Technical Debt:**
- tinker-atropos is a git submodule; initialization failures produce cryptic errors
- No Windows support (fcntl/msvcrt)
- Hardcoded paths (`~/.hermes/cron/`)
- Batch runner has disk I/O bottleneck (sequential processing, one-by-one JSONL writes)
- Trajectory compressor uses blocking LLM calls without explanation
- No validation of HuggingFace dataset format schema

---

### Feature 10: Persistent Memory & User Modeling

**What it does:** Honcho dialectic user modeling. Persistent memory, user profiles across sessions. FTS5 session search with LLM summarization.

**Key Architecture:**

- **MemoryStore**: Bounded character-limited stores (MEMORY.md: 2200 chars, USER.md: 1375 chars) with entry management
- **Frozen Snapshot Pattern**: System prompt frozen at session start for Anthropic prompt caching prefix stability
- **FTS5 Session Search**: SQLite FTS5 with BM25 relevance, delegation chain resolution, truncation around matches
- **HonchoSessionManager**: Dialectic queries, context prefetching, session naming strategies (per-session/per-repo/per-directory/global)

**Implementation Highlights:**

- **Atomic Rename for File Writes**: Temp file + `os.replace()` prevents partial reads during concurrent writes
- **Dedicated Lock File**: Memory files remain atomic-readable while operations are locked
- **FTS5 Query Sanitization**: Strips FTS5-special chars (`+`, `-`, `(`, `)`, `"`)
- **Thread-Safe Background Loops**: Both Honcho async writer and MCP event loop use dedicated daemon threads
- **Context Prefetching**: Dialectic and context results prefetched non-blocking, consumed next turn

**Technical Debt:**
- Character limits vs token limits mismatch
- Honcho requires external service; graceful degradation is just logging warnings
- Memory migration is one-way; no sync mechanism
- Dialectic char cap hard truncation can cut sentences mid-word

---

### Feature 11: MCP (Model Context Protocol) Integration

**What it does:** Connect any MCP server for extended capabilities. Includes OAuth support for MCP authentication.

**Key Architecture:**

- **Transport Types**: Stdio (command + args) and HTTP/StreamableHTTP (url + headers)
- **Connection Lifecycle**: Long-lived asyncio tasks on dedicated `_mcp_loop` daemon thread with reconnection strategy
- **Sampling Handler**: Server-initiated LLM requests with rate limiting and tool loop governance
- **Tool Name Prefixing**: Sanitization of hyphens/dots to underscores, prefixed as `mcp_{server}_{tool}`

**Reconnection Strategy:**
- Exponential backoff: 1s, 2s, 4s, 8s... capped at 60s
- Max 5 retries
- Only reconnects on unexpected drops (not shutdown)

**Security:**

- Environment: Explicit allowlist (`PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TERM`, `SHELL`, `TMPDIR`, `XDG_*`)
- Credential Stripping: Regex-based sanitization of GitHub PATs, OpenAI-style keys, Bearer tokens
- Command Resolution: `shutil.which()` plus fallback paths for node-based tools

**Technical Debt:**
- Sampling offload threading history (previous implementation caused deadlocks)
- Tool name collision detection is runtime only
- No persistent OAuth token refresh handling
- Stdio subprocess env filtering excludes `DISPLAY`, `SSH_AUTH_SOCK` and others

---

### Feature 12: Context Files for Project Shaping

**What it does:** Project context files that shape every conversation. AGENTS.md workspace instructions.

**Key Architecture:**

- **Priority-Based Single-Type Loading**: Only ONE project context type loaded (first found wins)
- **Priority Order**: `.hermes.md` (walks to git root) → `AGENTS.md` (cwd only) → `CLAUDE.md` (cwd only) → `.cursorrules` (cwd only)
- **SOUL.md Independence**: Always loaded separately from project context

| Priority | File(s) | Scope | Purpose |
|----------|---------|-------|---------|
| 1 | `.hermes.md` / `HERMES.md` | Walk to git root | Project-specific instructions |
| 2 | `AGENTS.md` / `agents.md` | CWD only | AI coding assistant instructions |
| 3 | `CLAUDE.md` / `claude.md` | CWD only | Claude-specific instructions |
| 4 | `.cursorrules` + `.cursor/rules/*.mdc` | CWD only | Cursor IDE rules |
| SOUL | `SOUL.md` (in `~/.hermes/`) | Always | Agent identity/persona |

**Implementation Highlights:**

- **Git Root Walking**: `.hermes.md` applies to entire repository regardless of subdirectory
- **Threat Scanning**: Content matched against injection/hijack/deception patterns; fail-open approach (blocks injection but doesn't error)
- **Head+Tail Truncation**: 40% head + 50% tail preserved, 10% middle replaced with marker (preserves setup/installation at start and conclusions at end)
- **YAML Frontmatter Stripping**: Removes `---` delimited metadata before injection

**Technical Debt:**
- Fail-open threat scanning means injection attempts go unnoticed
- Character-based truncation may be very different in actual tokens
- Only one project context type loaded; no conflict detection
- `skip_context_files` bypasses ALL context files (coarse-grained)
- No file watching; context files loaded once at session start

---

### Feature 13: OpenClaw Migration Path

**What it does:** Automatic migration from OpenClaw including SOUL.md persona, memories, skills, command allowlist, messaging settings, API keys, and TTS assets.

**Key Architecture:**

- **Two-Layer System**: CLI wrapper (`hermes_cli/claw.py`) + migration engine (`openclaw_to_hermes.py`)
- **Preset System**: `user-data` (no secrets) and `full` (opt-in secrets via `--migrate-secrets`)
- **Secret Allowlist**: Only 6 keys migrated (`TELEGRAM_BOT_TOKEN`, `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `ELEVENLABS_API_KEY`, `VOICE_TOOLS_OPENAI_KEY`)

**Migration Categories:**
- Direct: SOUL.md, MEMORY.md/USER.md, skills, TTS, exec-approvals, MCP servers, model config, TTS config
- Archived (no mapping): IDENTITY.md, TOOLS.md, HEARTBEAT.md, plugin configs, hook configs, cron configs

**Implementation Highlights:**

- **Dynamic Module Loading**: `importlib.util` loads migration script from repo or installed skill location
- **Memory Entry Deduplication**: Normalizes text, skips duplicates, enforces char limits with overflow handling
- **Skill Conflict Resolution**: Three modes (skip, overwrite, rename with suffix)
- **Channel Config Mapping**: Extensive mapping of OpenClaw channel config to Hermes env vars
- **Backup Before Overwrite**: Every destructive operation creates backup in `~/.hermes/migration/openclaw/<timestamp>/backups/`
- **Dry-Run Mode**: Simulates all changes without modification
- **Run-If-Selected Pattern**: Clean separation between selection and execution

**Technical Debt:**
- 2500-line single `Migrator` class handles 35+ migration types (should be split)
- Memory overflow saves to file but user has no notification or UI to review
- Archived items are effectively dead (no follow-up system)
- Workspace detection heuristic assumes single primary workspace
- PyYAML dependency for config migration; absence causes hard failure
- API key matching by provider name/URL is fragile
- No transactional guarantees; partial migration leaves inconsistent state
- Skill discovery relies on `SKILL.md` which may not match OpenClaw structure

---

## Cross-Cutting Themes

### 1. Security as a First-Class Concern

Every feature that handles external input or executes code has threat scanning:

- **Skills Guard**: Multi-level trust system with regex + LLM secondary scan
- **Cron Security**: Pattern detection for injection, exfiltration, backdoors
- **Memory Threat Scanning**: Prompt injection patterns, invisible unicode detection
- **Context File Scanning**: Same injection/hijack/deception patterns as memory
- **MCP Credential Stripping**: Regex sanitization of error messages
- **Docker Security**: Hardened flags, tmpfs mounts, env allowlist
- **Secret Migration**: Opt-in with allowlist, not blind migration

### 2. Atomic Operations for Crash Safety

Multiple systems use the same pattern:
- Temp file creation → write → fsync → atomic rename

Examples: Memory writes, Cron job storage, Batch runner checkpointing, Skill file writes

### 3. Progressive Disclosure for Token Efficiency

Features minimize token cost at discovery time:
- Skills: categories → names → full content
- Session search: metadata only → LLM summarization on demand
- Context files: single-type loaded, head+tail truncation

### 4. Multi-Provider Abstraction

Multi-model support and MCP integration both use adapter/provider patterns to normalize different APIs to a common interface.

### 5. Isolation vs. Sharing Tension

Several features make different tradeoffs:
- **Delegation**: Isolated context, inherited credentials
- **Cron**: Fresh credentials, isolated execution
- **Honcho**: Uploads local memories to external service (one-way sync)
- **Context Files**: Git-root walking applies project context broadly

### 6. Background Threading for Non-Blocking Operations

Common patterns:
- Async write queue (Honcho, memory)
- Prefetch threads (Honcho context)
- Daemon threads for MCP event loop
- Background threads for subprocess management

---

## Key Innovations

1. **Frozen Snapshot Pattern**: Memory writes mid-session persist to disk but don't invalidate the prompt cache, dramatically reducing costs on long conversations

2. **Edit-Transport Streaming**: Gateway bridges sync agent callbacks to async platform delivery using message editing primitives

3. **Dynamic Reasoning Level Selection**: Honcho dialectic queries scale reasoning effort based on query complexity

4. **Grace Period for Missed Cron Jobs**: Prevents cron storm on gateway restart while maintaining schedule intent

5. **Trajectory Compression with Protection**: Preserves first turns (system setup) and last turns (conclusions) while compressing middle

6. **Thread-Safe Event Loop Dispatch**: MCP uses `asyncio.run_coroutine_threadsafe()` for safe cross-thread coroutine scheduling

7. **Output Fencing**: Solves shell init/exit noise problem in terminal output without relying on fragile patterns

---

## Recurring Technical Debt

### High Severity (Impact Multiple Features)

1. **No Transactional Guarantees**: Cron, memory, migration all have partial-failure risks
2. **Character Limits vs Token Limits**: Memory (2200 chars), context files (20k chars) use rough approximations
3. **External Service Dependencies**: Honcho, tinker-atropos (submodule) introduce availability/initialization risks

### Medium Severity (Feature-Specific)

1. **Large Single-File Implementations**: Gateway files (262KB, 78KB, 94KB) suggest monolithic code
2. **No Retry Mechanisms**: Cron delivery, RL training have fire-and-forget patterns
3. **Manual Sync of Blocklists**: Delegation blocks, cron disables must be kept in sync manually
4. **Hardcoded Limits**: MAX_CONCURRENT_CHILDREN=3, MAX_DEPTH=2, MAX_RECONNECT_RETRIES=5

### Low Severity (Known Limitations)

1. **Fail-Open Security Scanning**: Context/memory injection attempts silently blocked
2. **One-Way Sync**: Honcho migration uploads but doesn't sync back
3. **No File Watching**: Context files/memories loaded once at session start
4. **Windows Support Gaps**: fcntl vs msvcrt, Git Bash detection

---

## Patterns Established

1. **Always use atomic file operations** (temp + rename + fsync)
2. **Always use dedicated lock files** for read-modify-write safety
3. **Always scan external content** before injection (threat patterns + invisible unicode)
4. **Always provide dry-run/dry-run mode** for destructive operations
5. **Always backup before overwrite** for migration and configuration changes
6. **Always use exponential backoff** for reconnection logic
7. **Always use allowlist over denylist** for security-sensitive env var passing
