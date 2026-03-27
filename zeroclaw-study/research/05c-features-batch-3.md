# Feature Deep Dive: Batch 3 (Features 7-9)

> **ZeroClaw** — Personal AI Assistant
> **Date:** 2026-03-26
> **Sources:** `src/gateway/`, `src/sop/`, `src/skills/`, `web/` (React frontend)

---

## Feature 7: Web Dashboard & Gateway API

### Overview

The Gateway is both an HTTP/WebSocket control plane AND a React web dashboard served directly from ZeroClaw. It provides real-time chat, configuration editing, cron management, memory browsing, tool inspection, logs viewing, cost tracking, health diagnostics, integrations status, and pairing management.

### Architecture

```
web/ (React 19 + Vite + Tailwind CSS)
    |
    | compiled static assets
    v
src/gateway/mod.rs (128KB — Gateway core)
    |         |           |
    v         v           v
 api.rs   ws.rs       sse.rs
(74KB)   (20KB)     (5KB)
    |
    v
AppState (central shared state)
```

**Key files:**
- `src/gateway/mod.rs` — Axum router setup, AppState struct, middleware
- `src/gateway/api.rs` — 74KB of REST API handlers
- `src/gateway/ws.rs` — WebSocket chat transport
- `src/gateway/sse.rs` — Server-Sent Events for real-time push
- `src/gateway/api_pairing.rs` — Pairing/token management
- `src/gateway/api_webauthn.rs` — WebAuthn registration/challenge
- `src/gateway/tls.rs` — TLS termination helpers
- `web/` — React dashboard (AgentChat, Dashboard, Cron, Memory, Tools, Config, etc.)

### AppState: Central Dependency Injection

All API handlers receive `AppState` via Axum's `State` extractor. The struct holds all runtime dependencies:

```rust
pub struct AppState {
    pub config: Arc<Mutex<Config>>,           // Live config (clone for read)
    pub provider: Arc<dyn Provider>,          // AI provider abstraction
    pub model: String,
    pub temperature: f64,
    pub mem: Arc<dyn Memory>,                 // Memory backend trait
    pub pairing: Arc<PairingGuard>,           // Auth guard
    pub rate_limiter: Arc<GatewayRateLimiter>,
    pub idempotency_store: Arc<IdempotencyStore>,
    pub whatsapp: Option<Arc<WhatsAppChannel>>,
    pub linq: Option<Arc<LinqChannel>>,
    pub nextcloud_talk: Option<Arc<NextcloudTalkChannel>>,
    pub wati: Option<Arc<WatiChannel>>,
    pub gmail_push: Option<Arc<GmailPushChannel>>,
    pub observer: Arc<dyn Observer>,
    pub tools_registry: Arc<Vec<ToolSpec>>,
    pub cost_tracker: Option<Arc<CostTracker>>,
    pub event_tx: tokio::sync::broadcast::Sender<serde_json::Value>,  // SSE broadcast
    pub shutdown_tx: tokio::sync::watch::Sender<bool>,
    pub node_registry: Arc<nodes::NodeRegistry>,
    pub path_prefix: String,
    pub session_backend: Option<Arc<dyn SessionBackend>>,
    // ... webauthn, canvas_store, etc.
}
```

### Bearer Token Authentication

Every `/api/*` route is protected by `require_auth()`. The pairing model means unauthenticated users must first call `POST /pair` to receive a token:

```rust
// api.rs:26
pub(super) fn require_auth(
    state: &AppState,
    headers: &HeaderMap,
) -> Result<(), (StatusCode, Json<serde_json::Value>)> {
    if !state.pairing.require_pairing() {
        return Ok(());  // Pairing disabled in config
    }
    let token = extract_bearer_token(headers).unwrap_or("");
    if state.pairing.is_authenticated(token) {
        Ok(())
    } else {
        Err((StatusCode::UNAUTHORIZED, Json(serde_json::json!({
            "error": "Unauthorized — pair first via POST /pair"
        }))))
    }
}
```

The error message is instructive — it tells the caller exactly how to fix the auth failure.

### Sensitive Field Masking (The Clever Part)

`GET /api/config` returns TOML but **masks all secrets before serialization**. The masking is bidirectional — outgoing masks, incoming restores:

```rust
// api.rs:792 — outgoing: mask secrets before sending to UI
fn mask_sensitive_fields(config: &Config) -> Config {
    let mut masked = config.clone();
    mask_optional_secret(&mut masked.api_key);
    mask_vec_secrets(&mut masked.reliability.api_keys);
    // ... 50+ fields across all channel configs (Telegram, Discord, Slack,
    //     Mattermost, WhatsApp, Matrix, Email, DingTalk, QQ, Lark, Feishu,
    //     Nostr, WATI, IRC, ClawdTalk, Nextcloud Talk, Linq, Cloudflare,
    //     ngrok, Qdrant, Composio, Browser, WebSearch, model routes)
    masked
}

// api.rs:1062 — incoming: restore masked placeholders from current config
fn restore_masked_sensitive_fields(incoming: &mut Config, current: &Config) {
    restore_optional_secret(&mut incoming.api_key, &current.api_key);
    // ... same 50+ fields
}
```

This allows the UI to GET config, modify non-secret fields, and PUT it back — secrets survive intact as `***MASKED***` placeholders that get restored from the in-memory current config before validation and save.

### Config PUT: Smart Merge

```rust
// api.rs:161
pub async fn handle_api_config_put(...) {
    let incoming: Config = toml::from_str(&body)?;
    let current_config = state.config.lock().clone();
    let new_config = hydrate_config_for_save(incoming, &current_config);
    // Validation happens BEFORE save
    new_config.validate()?;
    new_config.save().await?;
    *state.config.lock() = new_config;  // Atomic swap
}
```

### Idempotency

The `IdempotencyStore` prevents duplicate processing of retried requests. Header: `X-Idempotency-Key`. TTL-based eviction.

### Session Management

Sessions are stored with metadata (created_at, last_activity, message_count, optional name). Gateway sessions are prefixed `gw_` to distinguish from agent sessions. The UI can list, rename, and delete sessions via `GET/PUT/DELETE /api/sessions/{id}`.

### Claude Code Hook Endpoint

```rust
// api.rs:1391 — POST /hooks/claude-code
pub async fn handle_claude_code_hook(
    State(state): State<AppState>,
    Json(payload): Json<ClaudeCodeHookEvent>,
) -> impl IntoResponse {
    // NO auth required — Claude Code subprocesses can't持有 pairing token
    // Events tied to session_id for traceability
    tracing::info!(session_id = %payload.session_id, ...);
    Json(serde_json::json!({ "ok": true }))
}
```

### React Frontend Pages (14 pages)

| Page | File | Purpose |
|------|------|---------|
| Dashboard | `Dashboard.tsx` | System overview, health, channels, integrations |
| AgentChat | `AgentChat.tsx` | Real-time chat via WebSocket |
| Cron | `Cron.tsx` | Cron job management (list, add, edit, delete) |
| Memory | `Memory.tsx` | Memory browser (list + search) |
| Tools | `Tools.tsx` | Tool inspector |
| Config | `Config.tsx` | Config editor with TOML preview |
| Cost | `Cost.tsx` | Cost tracking summary |
| Logs | `Logs.tsx` | Log viewer |
| Doctor | `Doctor.tsx` | Diagnostics runner |
| Pairing | `Pairing.tsx` | Pairing management |
| Integrations | `Integrations.tsx` | Channel status |
| Canvas | `Canvas.tsx` | Canvas/interactive workspace |

### Notable Patterns

1. **Path prefix support** for reverse-proxy deployments (`path_prefix: String` in AppState)
2. **Broadcast channel** (`event_tx`) fans out SSE events to all connected dashboard clients
3. **Health component tracking** (`mark_component_ok`, `mark_component_error`) for observability
4. **Webhook secret hash stored** (SHA-256 hex, never plaintext) for signature verification
5. **Rate limiting** via `GatewayRateLimiter` on the router level

### Technical Debt / Rough Edges

- The `api_pairing.rs` and `api_webauthn.rs` files are referenced but their content wasn't fully reviewed — they likely contain the pairing initiation and WebAuthn flow handlers
- 50+ field masking function is repetitive — could use a macro or derive-based approach
- `Config` cloning on every API request (`state.config.lock().clone()`) — lightweight but worth noting

---

## Feature 8: SOPs (Standard Operating Procedures)

### Overview

SOPs are event-driven workflow automation with MQTT, webhook, cron, and peripheral triggers. They chain actions with state management, metrics, and audit trails. The key differentiator is **deterministic execution** — SOPs that can run without LLM round-trips between steps.

### Architecture

```
SOP.toml + SOP.md (file-based SOP definitions)
        |
        v
  sop/mod.rs — loading, parsing, CLI
  sop/engine.rs — 73KB core state machine
  sop/dispatch.rs — event routing
  sop/condition.rs — trigger condition evaluator
  sop/types.rs — all data types
  sop/audit.rs — audit logger
  sop/metrics.rs — cost/savings tracking
        |
        v
 sop tools (sop_execute, sop_approve, sop_advance, sop_list, sop_status)
```

### File-Based SOP Structure

```
<workspace>/sops/<sop-name>/
  SOP.toml   — metadata, triggers, priority, execution mode
  SOP.md     — step procedures
```

**SOP.toml example:**
```toml
[sop]
name = "pump-shutdown"
description = "Emergency shutdown for faulty pump"
version = "1.0.0"
priority = "critical"
execution_mode = "auto"
cooldown_secs = 60
max_concurrent = 1

[[triggers]]
type = "mqtt"
topic = "sensors/pressure"
condition = "$.value > 90"

[[triggers]]
type = "webhook"
path = "/sop/emergency"
```

**SOP.md step parsing** (`mod.rs:152-339`):

Steps are parsed from `## Steps` markdown section. Each numbered item extracts:
- **Bold text** (`**title**`) as step title
- **Em dash/body** as step body
- Sub-bullets: `tools:`, `requires_confirmation:`, `kind:` (execute/checkpoint)

```rust
// mod.rs:157 — parse_steps() handles:
1. "## Steps" heading detection
2. Numbered items "1.", "2." etc.
3. Bold title extraction via extract_bold_title()
4. Sub-bullet parsing: tools, requires_confirmation, kind
5. Multiline body continuation
```

### Trigger Types

Five trigger sources, each with pattern-matching logic:

| Trigger | Match Logic | Condition |
|---------|-------------|-----------|
| `MQTT` | Topic wildcard matching (+, #) | JSON path or direct comparison |
| `Webhook` | Exact path match | None |
| `Cron` | Expression match against scheduler | None |
| `Peripheral` | `board/signal` exact match | Direct comparison |
| `Manual` | Always matches | None |

**MQTT topic wildcard matching** (`engine.rs:730-758`):
```rust
fn mqtt_topic_matches(pattern: &str, topic: &str) -> bool {
    // "+" matches single path segment
    // "#" matches zero or more segments (multi-level wildcard)
    // No escape sequences; # must be at end
}
```

### Condition Evaluator

`condition.rs` implements two condition syntaxes:

**JSON path comparison** (MQTT, Peripheral with payload):
```
$.value > 85
$.data.sensor.temp >= 100
$.status == "critical"
```

**Direct comparison** (peripheral signals):
```
> 0
>= 5
== 42
```

The evaluator is fail-closed: missing payload, invalid JSON, or unresolvable path all return `false`.

```rust
// condition.rs:16 — public API
pub fn evaluate_condition(condition: &str, payload: Option<&str>) -> bool {
    // Empty condition = unconditional match
    // Missing payload = false (fail-closed)
    // JSON path: resolve path, compare with parsed JSON value
    // Direct: parse payload as f64, compare
}
```

### Execution Modes

| Mode | Behavior |
|------|----------|
| `Auto` | Execute all steps without approval |
| `Supervised` | Approval before step 1 only |
| `StepByStep` | Approval before every step |
| `PriorityBased` | Critical/High → Auto; Normal/Low → Supervised |
| `Deterministic` | No LLM calls; step outputs pipe to next step inputs |

**`requires_confirmation` on individual steps** overrides execution mode — even `Auto` mode waits for approval on those steps.

### Deterministic Execution (The Clever Part)

Deterministic mode runs steps without LLM round-trips. Each step's output is piped as JSON input to the next step:

```rust
// engine.rs:308 — start_deterministic_run()
let run_id = format!("det-{epoch_ms}-{:04}", self.run_counter);
// Run is started with status Running, llm_calls_saved = 0

// engine.rs:374 — advance_deterministic_step()
run.llm_calls_saved += 1;  // Each step saves an LLM call
let next_step_num = run.current_step + 1;
if next_step_num > run.total_steps {
    // On completion: accumulate to deterministic_savings
    self.deterministic_savings.total_llm_calls_saved += saved;
}
```

**Checkpoint pauses** (`engine.rs:486-522`):
- When `step.kind == Checkpoint`, run enters `PausedCheckpoint` status
- State is persisted to JSON file `{run_id}.state.json`
- Approval resumes by loading state and continuing from next step

```rust
// Persist to SOP location directory (or system temp)
let dir = sop.location.as_deref().unwrap_or_else(|| temp_dir.as_path());
let state_file = dir.join(format!("{run_id}.state.json"));
std::fs::write(&state_file, json)?;
```

### Approval Timeout Auto-Approve

```rust
// engine.rs:575 — check_approval_timeouts()
// Critical/High priority SOPs: auto-approve after timeout
// Normal/Low priority: wait indefinitely
if timeout_secs > 0 {
    // Backdate waiting_since in test to simulate timeout
    for (run_id, is_critical) in timed_out {
        if is_critical {
            self.approve_step(&run_id)  // Auto-approve
        }
    }
}
```

### Dispatch: The Two-Phase Lock Pattern

```rust
// dispatch.rs:69 — dispatch_sop_event()
// Phase 1: Lock → match_trigger → drop lock
let matched_names = { engine.lock().match_trigger(&event)... };

// Phase 2: Lock → for each: start_run → collect results → drop lock
// Phase 3: Async (no lock): audit each started run
```

This minimizes lock hold time — no lock is held while starting multiple runs or during async audit logging.

### Cooldown and Concurrency

```rust
// engine.rs:89 — can_start()
1. Per-SOP concurrency: count active_runs for this SOP < max_concurrent
2. Global concurrency: active_runs.len() < max_concurrent_total
3. Cooldown: last_finished_run.completed_at + cooldown_secs < now
```

Cooldown uses ISO-8601 timestamp parsing — custom implementation without chrono dependency (though chrono is used elsewhere in dispatch).

### Run Eviction

```rust
// engine.rs:654 — finish_run()
let max = self.config.max_finished_runs;
if max > 0 && self.finished_runs.len() > max {
    self.finished_runs.drain(..excess);  // Evict oldest
}
```

Zero means unlimited (default).

### Step Context Formatting

When injecting step into agent:

```rust
// engine.rs:809 — format_step_context()
format!(
    "[SOP: {} (run {}) — Step {} of {}]\n\n\
     Trigger: {} {}\n\
     Payload: {}\n\
     Previous: Step {} {} — {}\n\
     Current step: **{}**\n\
     {}\n\
     Suggested tools: {}\n",
    sop.name, run.run_id, step.number, run.total_steps,
    trigger_source, topic_or_none,
    payload_or_none,
    prev_step_num, prev_status, prev_output,
    step.title, step.body,
    step.suggested_tools.join(", ")
)
```

### Notable Patterns

1. **No agent loop dependency** — dispatch is fully async; headless callers (MQTT, webhook, cron) can trigger SOPs, with agent loop consuming the `SopRunAction`
2. **Cron cache** (`dispatch.rs:246`) — parsed cron expressions cached at startup to avoid re-parsing on every scheduler tick
3. **Batch lock pattern** — exactly 2 lock acquisitions per dispatch
4. **Condition fail-closed** — bad data never triggers automation
5. **Custom ISO-8601** — no chrono dependency in engine.rs; custom `days_to_ymd` and `parse_iso8601_secs`

### Technical Debt / Rough Edges

- `now_iso8601()` in engine.rs is custom implementation while `dispatch.rs` uses `chrono::Utc::now()` — inconsistency
- Step numbering gap detection in `validate_sop()` is purely advisory (warning, not error)
- Deterministic mode output piping treats all outputs as JSON strings or JSON values — ambiguity if step returns non-JSON text

---

## Feature 9: Skills Platform

### Overview

Skills are bundled, community, or workspace extensions with SKILL.md/SKILL.toml format. The platform includes security auditing before install, git-based distribution, and two loading modes: workspace skills (user-owned) and open-skills (community, synced from GitHub).

### Architecture

```
~/.zeroclaw/workspace/skills/<name>/  — user/workspace skills
~/.zeroclaw/open-skills/             — community skills (git-synced)
~/.zeroclaw/workspace/open-skills/    — alternative open-skills location

src/skills/mod.rs (72KB) — loading, installation, prompt generation
src/skills/audit.rs (29KB) — security auditing
src/skills/creator.rs (31KB) — SKILL.md/SKILL.toml scaffolding
src/skills/improver.rs (14KB) — skill self-improvement
src/skills/testing.rs — TEST.sh runner
```

### Skill Manifests

**SKILL.toml** (structured):
```toml
[skill]
name = "pdf-extract"
description = "Extract text from PDF files"
version = "1.0.0"
author = "community"
tags = ["docs", "parser"]

[[tools]]
name = "extract"
description = "Run pdf-extract on a file"
kind = "shell"
command = "python3 scripts/extract.py {{file}}"

[[prompts]]
# Additional LLM context/instructions
```

**SKILL.md** (simple, markdown-first):
```markdown
---
name: my-skill
description: What this skill does
version: 1.0.0
author: your-name
tags:
  - productivity
---
# My Skill

Instructions for the agent to follow when this skill is active.
```

Frontmatter parsing is **custom** — no serde_yaml dependency. A simple line-by-line parser handles `key: value`, inline arrays `[a, b]`, and YAML block lists.

```rust
// mod.rs:636 — parse_simple_frontmatter()
// Custom YAML-like parser: "name: pdf" → meta.name = Some("pdf")
// Handles: tags: [a, b], tags:\n  - a\n  - b
// Trimmed of surrounding quotes
```

### Loading Pipeline

```rust
// mod.rs:154 — load_skills_with_open_skills_config()
1. If open_skills_enabled:
   - Clone/sync open-skills repo (git pull --ff-only weekly)
   - load_open_skills() — nested `skills/<name>/SKILL.md` layout preferred
   - Falls back to flat .md files (excluding README.md)
2. load_workspace_skills() — `<workspace>/skills/<name>/SKILL.*`
3. Each skill passes through security audit BEFORE loading
4. Failed audit → skill skipped (with warning including remediation hint)
```

### Security Audit (The Critical Part)

`audit.rs` implements defense-in-depth for untrusted skill code:

```rust
// audit.rs:30 — audit_skill_directory()
pub fn audit_skill_directory(skill_dir: &Path) -> Result<SkillAuditReport>
```

**Audit checks:**

1. **Manifest required** — Must have `SKILL.md` or `SKILL.toml`
2. **Symlinks blocked** — Any symlink in skill tree = rejection
3. **Script files blocked** — `.sh`, `.bash`, `.zsh`, `.fish`, `.ps1`, `.bat`, `.cmd` by default (configurable via `allow_scripts`)
4. **Shebang detection** — Files starting with `#!/bin/sh`, `#!/usr/bin/env python3`, etc. treated as scripts
5. **File size limit** — >512KB for markdown/TOML = rejection
6. **Markdown link validation** — All `[]()` links checked:
   - Remote links (http/https) blocked if pointing to `.md` files
   - Absolute paths blocked
   - Script suffixes blocked
   - `..` escapes blocked
   - Cross-skill references (`../other-skill/SKILL.md`) allowed if target stays within skills root
7. **TOML manifest validation** — Invalid TOML = rejection
8. **High-risk pattern detection** — Regex patterns for:
   - `curl ... | sh` / `wget ... | sh`
   - `invoke-expression` / `iex` (PowerShell)
   - `rm -rf /` (destructive)
   - `nc ... -e` (netcat remote exec)
   - `dd if=` (disk overwrite)
   - `mkfs` (filesystem format)
   - Fork bombs `:(){ :|:& };:`
9. **Shell chaining blocked** in tool commands — `&&`, `||`, `;`, `$()`, backticks
10. **Empty command rejection** — `kind = "shell"` with empty `command` = rejection

```rust
// audit.rs:548 — contains_shell_chaining()
fn contains_shell_chaining(command: &str) -> bool {
    ["&&", "||", ";", "\n", "\r", "`", "$("]
        .iter().any(|needle| command.contains(needle))
}
```

**Cross-skill references** are a clever exception:
- `../skill-b/SKILL.md` (parent traversal) — allowed even if target doesn't exist yet
- `other-skill.md` (bare filename) — treated as cross-skill, missing allowed
- `docs/guide.md` (subdirectory) — MUST exist locally or fails

### Installation Sources

```rust
// mod.rs:1434 — install_git_skill_source()
// Git clone --depth 1, then remove .git directory
// SECURITY: audit BEFORE accept, audit AFTER copy

// mod.rs:1257 — install_clawhub_skill_source()
// Downloads from clawhub.ai/api/v1/download?slug=<slug>
// ZIP extracted with path sanitization (no .., no absolute, no special chars)
// Max 50MB
// Has 429 (rate limit) handling

// mod.rs:1194 — install_local_skill_source()
// Copy with symlink check and second audit pass
```

**ClawHub detection** (`mod.rs:967`):
```rust
fn is_clawhub_source(source: &str) -> bool {
    source.starts_with("clawhub:") ||
    parse_clawhub_url(source).is_some()
}

fn clawhub_download_url(source: &str) -> Result<String> {
    // clawhub:skill-slug → https://clawhub.ai/api/v1/download?slug=skill-slug
    // https://clawhub.ai/owner/slug → same
    // https://www.clawhub.ai/slug → same
}
```

### Open-Skills Sync

```rust
const OPEN_SKILLS_REPO_URL = "https://github.com/besoeasy/open-skills";
const OPEN_SKILLS_SYNC_MARKER = ".zeroclaw-open-skills-sync";
const OPEN_SKILLS_SYNC_INTERVAL_SECS = 60 * 60 * 24 * 7;  // Weekly

// Clone --depth 1 on first run
// Pull --ff-only on subsequent runs (fails safely if non-git dir)
// Sync marker file age checked weekly
```

### Skills to System Prompt

```rust
// mod.rs:752 — skills_to_prompt()
pub fn skills_to_prompt(skills: &[Skill], workspace_dir: &Path) -> String

// Two modes via config:
SkillsPromptInjectionMode::Full    — inline all prompts + tools
SkillsPromptInjectionMode::Compact — skip prompts, load via read_skill() on demand
```

Output format is XML (for LLM parsing):
```xml
<available_skills>
  <skill>
    <name>pdf-extract</name>
    <description>Extract text from PDF files</description>
    <location>skills/pdf-extract/SKILL.md</location>
    <instructions>
      <instruction>Do the thing.</instruction>
    </instructions>
    <callable_tools hint="Invoke by name instead of using shell.">
      <tool>
        <name>pdf-extract.extract</name>
        <description>Run pdf-extract on a file</description>
      </tool>
    </callable_tools>
  </skill>
</available_skills>
```

**Tool registration** — `skills_to_tools()` converts skill tools into actual `Tool` trait objects:
- `shell`/`script` kind → `SkillShellTool`
- `http` kind → `SkillHttpTool`

Registered tools can be invoked via function calling without going through shell.

### XML Escaping

```rust
// mod.rs:706 — append_xml_escaped()
for ch in text.chars() {
    match ch {
        '&' => "&amp;", '<' => "&lt;", '>' => "&gt;",
        '"' => "&quot;", '\'' => "&apos;",
        _ => out.push(ch),
    }
}
```

All skill content in XML output is escaped. Test verifies: `<name>xml&lt;skill&gt;</name>` for a skill literally named `xml<skill>`.

### Open-Skills Tag Injection

```rust
// mod.rs:232 — finalize_open_skill()
if !skill.tags.contains("open-skills") {
    skill.tags.push("open-skills");
}
if skill.author.is_none() {
    skill.author = Some("besoeasy/open-skills");
}
```

Community skills are tagged `open-skills` and attributed to the repo author.

### Notable Patterns

1. **Double audit** — once on source, once on installed copy (after copy for local installs)
2. **Path traversal protection** — skill name validated for `..`, `/`, `\` before any filesystem operation
3. **Custom YAML parser** — avoids pulling in serde_yaml (~30KB saved)
4. **No git metadata** — `.git` directory stripped after clone (saves space, prevents accidental git operations inside skills)
5. **Git source detection** is broad but careful — excludes local paths, handles SCP-style (`git@host:path.git`) and URL-style schemes
6. **Installed directory detection** — snapshots before/after clone to determine which directory was created

### Technical Debt / Rough Edges

- `allow_scripts` is a global flag — per-skill override not supported (but warning message guides user to set the config)
- Skill tests (`TEST.sh`) are shell scripts run via `std::process::Command` — security auditor allows scripts in installed skills after `allow_scripts = true` is set, but `TEST.sh` could theoretically execute anything
- open-skills repo sync on every startup if marker is old — no jitter/randomization to avoid thundering herd on GitHub
- Skill installation removes `.git` but leaves other git artifacts (e.g., `gitignore`) — minor but inconsistent with intent

---

## Cross-Feature Observations

### Shared Security Posture

All three features share a security-first mindset:
- **Gateway**: bearer token auth, secret masking, idempotency
- **SOPs**: fail-closed conditions, approval gates, deterministic audit
- **Skills**: comprehensive audit before load, script blocking, path traversal protection

### Event-Driven Architecture

SOPs and the Gateway both use broadcast channels for event distribution:
- Gateway: `event_tx: broadcast::Sender<serde_json::Value>` for SSE
- SOPs: dispatch returns `Vec<DispatchResult>`, headless callers use `process_headless_results()`

### File-Based Configuration

Both SOPs and Skills prefer file-based definitions over database schemas:
- SOPs: `SOP.toml` + `SOP.md` in `<workspace>/sops/<name>/`
- Skills: `SKILL.toml` or `SKILL.md` in `<workspace>/skills/<name>/`

This makes them git-friendly and easy to version control.

### Custom ISO-8601 Implementation

Both `engine.rs` (SOP) and other modules implement ISO-8601 timestamps without chrono for specific subsets. This suggests chrono might be available as a dependency but isn't uniformly used — or these are intentionally isolated to reduce compile times.

### No Dependency on Agent Loop

SOP dispatch and Skills loading are fully independent of the agent loop. They can operate in headless mode (MQTT fan-in, cron triggers) while the agent loop independently processes interactive sessions.
