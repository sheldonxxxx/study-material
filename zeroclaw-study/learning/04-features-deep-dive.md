# ZeroClaw Feature Deep Dive

> **Project:** ZeroClaw - Personal AI Assistant
> **Sources:** Feature index + batch deep dives (4 batches covering 12 features)
> **Date:** 2026-03-26
> **Output:** Synthesized narrative per feature, ordered by priority

---

## Introduction

ZeroClaw is a Rust-based personal AI assistant that runs on 26+ messaging platforms, controls hardware peripherals, orchestrates multi-agent workflows, and provides a web dashboard for real-time control. The codebase demonstrates mature Rust async patterns with strong separation of concerns across feature domains.

This document synthesizes four batches of deep-dive research into a coherent narrative per feature, ordered by the priority established in the feature index.

---

## Core Features (Priority 1)

### Feature 1: Agent Loop & Orchestration

**Key file:** `src/agent/loop_.rs` (346KB)

The agent loop is the core orchestration engine implementing a classic LLM agent loop pattern with tool dispatch, prompt construction, message classification, context management, and memory loading.

**Core Loop Structure (`run_tool_call_loop`, line 2783):**

```rust
pub(crate) async fn run_tool_call_loop(
    provider: &dyn Provider,
    history: &mut Vec<ChatMessage>,
    tools_registry: &[Box<dyn Tool>],
    // ... many more parameters
) -> Result<String> {
    let max_iterations = if max_tool_iterations == 0 {
        DEFAULT_MAX_TOOL_ITERATIONS  // 10
    } else {
        max_tool_iterations
    };
    for iteration in 0..max_iterations {
        // Tool dispatch, LLM call, result processing
    }
}
```

**Tool Dispatcher Strategy Pattern:**

Two dispatcher implementations handle LLM response parsing:
- `NativeToolDispatcher` - for providers with native function calling (OpenAI, Anthropic)
- `XmlToolDispatcher` - for providers returning XML-style tool calls

The XML parser supports 6 formats including MiniMax's `<invoke name="shell">` format, enabling provider flexibility.

**Multi-Format XML Tool Call Parsing:**
```rust
const TOOL_CALL_OPEN_TAGS: [&str; 6] = [
    "<tool_call>", "<toolcall>", "<tool-call>",
    "<invoke>", "<minimax:tool_call>", "<minimax:toolcall>",
];
```

**Vision Provider Routing:**
When image inputs are detected but the default provider lacks vision capability, the loop routes to a configured `vision_provider` with validation.

**Credential Scrubbing:**
Tool outputs pass through a regex-based scrubber that redacts API keys, tokens, passwords, and bearer credentials while preserving context (first 4 characters of value).

**Notable Concern:** Global static `MODEL_SWITCH_REQUEST` using `LazyLock` is not ideal for testability. The 346KB file size suggests potential decomposition would aid maintainability.

---

### Feature 2: Multi-Channel Inbox

**Key file:** `src/channels/mod.rs` (415KB)

Unified messaging across 26+ platforms through a shared `Channel` trait. Each platform implements `send`, `listen`, `health_check`, `typing indicators`, `reactions`, and `draft updates`.

**Channel Trait (`src/channels/traits.rs`):**
```rust
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;
    async fn health_check(&self) -> bool { true }
    // Draft update support
    fn supports_draft_updates(&self) -> bool { false };
    async fn send_draft(&self, _message: &SendMessage) -> anyhow::Result<Option<String>> { Ok(None) }
    // Reactions
    async fn add_reaction(&self, _channel_id: &str, _message_id: &str, _emoji: &str) -> anyhow::Result<()>
}
```

**Per-Sender Conversation Isolation:**
```rust
type ConversationHistoryMap = Arc<Mutex<HashMap<String, Vec<ChatMessage>>>>;
const MAX_CHANNEL_HISTORY: usize = 50;
```

Each sender gets a dedicated conversation history with proactive compaction before context window exhaustion.

**Concurrent Message Processing:**
```rust
const CHANNEL_PARALLELISM_PER_CHANNEL: usize = 4;
const CHANNEL_MIN_IN_FLIGHT_MESSAGES: usize = 8;
const CHANNEL_MAX_IN_FLIGHT_MESSAGES: usize = 64;
```

**Notable Channel Implementations:**
- **Telegram** (183KB): Long polling, ACK reactions, streaming draft mode, transcription/TTS
- **Slack** (162KB): Event-based webhooks + web API polling, markdown blocks, threads
- **Discord** (85KB): Gateway bot connection, guild awareness, embeds

**Builder Pattern:** Each channel uses a fluent builder for optional configurations (`.with_ack_reactions()`, `.with_streaming()`, `.with_transcription()`).

**Notable Concern:** 415KB `mod.rs` is extremely large. Health checking is fire-and-forget without explicit restart logic for unhealthy channels.

---

### Feature 3: Tool System

**Key files:** `src/tools/mod.rs` (50KB), `src/tools/delegate.rs` (99KB), `src/tools/traits.rs`

70+ tools implementing the `Tool` trait with async execution.

**Tool Trait:**
```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
}
```

**Default Tools:** Shell, FileRead, FileWrite, FileEdit, GlobSearch, ContentSearch

**Full Registry:** Plus Browser, Git, Web fetch/search, MCP, Jira, Notion, Google Workspace, Cron, Memory, Hardware access

**DelegateTool (99KB):** Enables multi-agent workflows with three execution modes:
1. **Synchronous:** Blocks until sub-agent completes
2. **Background:** Spawns tokio task, returns task_id immediately
3. **Parallel:** Runs multiple agents concurrently with aggregated results

**Background Task Security:**
```rust
fn validate_task_id(task_id: &str) -> Result<(), String> {
    if uuid::Uuid::parse_str(task_id).is_err() {
        return Err(format!("Invalid task_id '{task_id}': must be a valid UUID"));
    }
    Ok(())
}
```
UUID validation prevents path traversal attacks on result files.

**Cancellation Cascade:**
```rust
pub fn cancel_all_background_tasks(&self) {
    self.cancellation_token.cancel();
}
```
Each background task receives a child token that propagates cancellation through the delegation chain.

**Notable Concern:** Background result files have no TTL or cleanup mechanism. 99KB `delegate.rs` handles many concerns and could benefit from decomposition.

---

### Feature 4: Memory & Knowledge Management

**Key file:** `src/memory/sqlite.rs` (2762 lines)

SQLite-backed brain with hybrid search: vector embeddings + full-text search (FTS5) + BM25 scoring.

**Hybrid Search Pipeline (`recall()`, lines 604-839):**

1. **BM25 keyword search** via FTS5 virtual table
2. **Vector similarity** via cosine similarity on stored BLOB embeddings
3. **Hybrid merge** with configurable weighting (default 70% vector, 30% keyword)

```rust
let keyword_results = Self::fts5_search(&conn, &query, limit * 2)?;
let vector_results = Self::vector_search(&conn, qe, limit * 2, None, None)?;
let merged = vector::hybrid_merge(&vector_results, &keyword_results, vector_weight, keyword_weight, limit)?;
```

**Fallback chain:** If hybrid returns nothing, LIKE-based substring search kicks in.

**Embedding Cache with Stable Hashing:**

`get_or_compute_embedding()` uses SHA-256 truncated to 16 hex chars as deterministic cache key, avoiding `DefaultHasher` instability across Rust versions:

```rust
fn content_hash(text: &str) -> String {
    use sha2::{Digest, Sha256};
    let hash = Sha256::digest(text.as_bytes());
    format!("{:016x}", u64::from_be_bytes(hash[..8].try_into().expect(...)))
}
```

LRU eviction runs in the same `spawn_blocking` call after insert.

**SQLite Performance Configuration:**
```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA mmap_size = 8388608;  -- 8MB
PRAGMA cache_size = -2000;   -- 2MB
PRAGMA temp_store = MEMORY;
```

**Session Isolation:** Memory entries carry `session_id` and `namespace`. `recall()` filters by session when provided.

**Notable Pattern:** Single-query multi-ID fetch (lines 692-705) builds `WHERE id IN (?1, ?2, ...)` with a single prepared statement instead of N round-trips.

---

### Feature 5: Security & Sandboxing

**Key file:** `src/security/policy.rs` (3128 lines)

Defense-in-depth through five sequential gates:

1. **Autonomy gate** - `ReadOnly` blocks all commands immediately
2. **Subshell/expansion blocker** - blocks `` ` ``, `$(`, `${`, `<(`, `>(`
3. **Redirection blocker** - blocks unquoted `<` and `>`
4. **Tee blocker** - prevents `echo x | tee /file`
5. **Background chain blocker** - blocks bare `&` (but allows `&&`)
6. **Per-segment allowlist** - each sub-command validated independently

**Quote-Aware Shell Lexer (`split_unquoted_segments()`, lines 325-412):**
```rust
enum QuoteState { None, Single, Double }
// `sqlite3 db "SELECT 1; SELECT 2;"` → single segment
// `ls | grep foo` → two segments
// `echo '$(rm -rf /)'` → single segment (literal)
```

This prevents attacks like `echo '$(rm -rf /)'` passing as an allowed `echo` command.

**Command Risk Classification (lines 711-832):**

High-risk: `rm`, `curl`, `wget`, `shutdown`, `sudo`, `reboot`, PowerShell, fork bombs

Medium-risk: `git commit/push/reset/clean/rebase/merge`, `npm install/add/remove/publish`, `cargo add/remove/install/clean`

**Explicit Allowlist Exemption:**

The wildcard `"*"` in `allowed_commands` does NOT exempt from `block_high_risk_commands`. Explicit entries are required:

```rust
let has_wildcard = self.allowed_commands.iter().any(|c| c.trim() == "*");
if has_wildcard && !self.block_high_risk_commands {
    return Ok(risk); // Skip all risk gates
}
// But wildcard alone does NOT bypass high-risk command checks
```

**Rate Limiting (`ActionTracker`, lines 36-78):**
```rust
pub fn record(&self) -> usize {
    let mut actions = self.actions.lock();
    let cutoff = Instant::now().checked_sub(Duration::from_secs(3600)).unwrap();
    actions.retain(|t| *t > cutoff);
    actions.push(Instant::now());
    actions.len()
}
```

**Path Validation (`is_resolved_path_allowed()`, lines 1189-1226):** Canonicalizes paths and checks after symlink resolution to prevent symlink escapes.

**Notable Pattern:** `prompt_summary()` (lines 1406-1485) renders human-readable security constraints for LLM system prompt injection.

---

### Feature 6: AI Provider Abstraction & Routing

**Key files:** `src/providers/mod.rs` (129KB), `src/providers/reliable.rs` (111KB)

Factory pattern with canonical string keys and regional/provider aliases.

**ReliableProvider Three-Level Failover (lines 429-567):**

```rust
// Outer: model fallback chain
for current_model in &models {
    // Middle: provider priority list
    for (provider_name, provider) in &self.providers {
        // Inner: retry with exponential backoff
        for attempt in 0..=self.max_retries {
            match provider.chat_with_history(&messages, current_model, temp).await {
                Ok(resp) => return Ok(resp),
                Err(e) => {
                    if is_non_retryable(&e) { break; }
                    tokio::time::sleep(Duration::from_millis(wait)).await;
                    backoff_ms = (backoff_ms * 2).min(10_000);
                }
            }
        }
    }
}
```

**Error Classification:**

- **Retryable:** 5xx, 408, 429, connection errors, model overload
- **Non-retryable:** 4xx except 429/408, auth failures, unknown models
- **Context window errors:** NOT non-retryable - trigger history truncation (drop oldest half of non-system messages) and retry

**Context Truncation (`truncate_for_context()`, lines 288-312):**
```rust
fn truncate_for_context(messages: &mut Vec<ChatMessage>) -> usize {
    let non_system: Vec<usize> = messages.iter().enumerate()
        .filter(|(_, m)| m.role != "system")
        .map(|(i, _)| i).collect();
    if non_system.len() <= 1 { return 0; }
    let drop_count = non_system.len() / 2;
    for &idx in indices_to_remove.iter().rev() {
        messages.remove(idx);
    }
    drop_count
}
```

**Task-Local Fallback Info:**
```rust
tokio::task_local! {
    static PROVIDER_FALLBACK: RefCell<Option<ProviderFallbackInfo>>;
}
```
When a fallback occurs, `record_provider_fallback()` stores info accessible after the request to notify users.

**API Key Rotation:**
```rust
fn rotate_key(&self) -> Option<&str> {
    let idx = self.key_index.fetch_add(1, Ordering::Relaxed) % self.api_keys.len();
    Some(&self.api_keys[idx])
}
```
Note: The warning at line 512-516 notes `Provider` trait has no `set_api_key` method, so key rotation cannot actually be applied.

---

### Feature 7: Web Dashboard & Gateway API

**Key files:** `src/gateway/mod.rs` (128KB), `src/gateway/api.rs` (74KB), `web/` (React frontend)

Axum-based HTTP/WebSocket control plane with React 19 + Vite + Tailwind CSS dashboard served directly from ZeroClaw.

**AppState: Central Dependency Injection:**
```rust
pub struct AppState {
    pub config: Arc<Mutex<Config>>,
    pub provider: Arc<dyn Provider>,
    pub mem: Arc<dyn Memory>,
    pub pairing: Arc<PairingGuard>,
    pub rate_limiter: Arc<GatewayRateLimiter>,
    pub idempotency_store: Arc<IdempotencyStore>,
    pub event_tx: tokio::sync::broadcast::Sender<serde_json::Value>,  // SSE broadcast
    pub shutdown_tx: tokio::sync::watch::Sender<bool>,
    // ... channel options, webauthn, canvas_store, etc.
}
```

**Sensitive Field Masking (The Clever Part):**

`GET /api/config` returns TOML but masks all secrets before serialization. Incoming restores masked placeholders from current config:

```rust
// Outgoing: mask secrets before sending to UI
fn mask_sensitive_fields(config: &Config) -> Config { ... }
// Incoming: restore masked placeholders from current config
fn restore_masked_sensitive_fields(incoming: &mut Config, current: &Config) { ... }
```

This allows UI to GET config, modify non-secret fields, and PUT it back - secrets survive intact as `***MASKED***` placeholders.

**React Frontend Pages (14 pages):**

Dashboard, AgentChat, Cron, Memory, Tools, Config, Cost, Logs, Doctor, Pairing, Integrations, Canvas

**Notable Pattern:** Path prefix support for reverse-proxy deployments via `path_prefix: String` in AppState.

**Technical Debt:** 50+ field masking function is repetitive. `Config` cloning on every API request.

---

### Feature 8: SOPs (Standard Operating Procedures)

**Key files:** `src/sop/mod.rs` (73KB), `src/sop/engine.rs` (73KB), `src/sop/dispatch.rs` (27KB)

Event-driven workflow automation with MQTT, webhook, cron, and peripheral triggers.

**File-Based SOP Structure:**
```
<workspace>/sops/<sop-name>/
  SOP.toml   — metadata, triggers, priority, execution mode
  SOP.md     — step procedures
```

**Trigger Types:**

| Trigger | Match Logic | Condition |
|---------|-------------|-----------|
| `MQTT` | Topic wildcard matching (+, #) | JSON path or direct comparison |
| `Webhook` | Exact path match | None |
| `Cron` | Expression match against scheduler | None |
| `Peripheral` | `board/signal` exact match | Direct comparison |
| `Manual` | Always matches | None |

**Execution Modes:**
- `Auto` - Execute all steps without approval
- `Supervised` - Approval before step 1 only
- `StepByStep` - Approval before every step
- `PriorityBased` - Critical/High to Auto; Normal/Low to Supervised
- `Deterministic` - No LLM calls; step outputs pipe to next step inputs

**Deterministic Execution (The Clever Part):**

Steps run without LLM round-trips. Each step's output pipes as JSON input to the next:

```rust
// engine.rs:308 — start_deterministic_run()
let run_id = format!("det-{epoch_ms}-{:04}", self.run_counter);
run.llm_calls_saved += 1;  // Each step saves an LLM call
```

**Checkpoint State Persistence:**
```rust
// engine.rs:486-522
let state_file = dir.join(format!("{run_id}.state.json"));
std::fs::write(&state_file, json)?;  // Persist to SOP location directory
```

**Two-Phase Lock Pattern (dispatch.rs:69):**
```rust
// Phase 1: Lock → match_trigger → drop lock
let matched_names = { engine.lock().match_trigger(&event)... };
// Phase 2: Lock → start_run → collect results → drop lock
// Phase 3: Async (no lock): audit each started run
```

**Technical Debt:** Custom ISO-8601 in engine.rs while dispatch.rs uses chrono. Step numbering gap detection is purely advisory (warning, not error).

---

## Secondary Features (Priority 2)

### Feature 9: Skills Platform

**Key files:** `src/skills/mod.rs` (72KB), `src/skills/audit.rs` (29KB), `src/skills/creator.rs` (31KB)

Bundled, community, and workspace skills with SKILL.md/SKILL.toml format, security auditing, and git-based distribution.

**Skill Manifests:**

**SKILL.toml** (structured):
```toml
[skill]
name = "pdf-extract"
[[tools]]
name = "extract"
kind = "shell"
command = "python3 scripts/extract.py {{file}}"
```

**SKILL.md** (markdown-first with frontmatter):
```markdown
---
name: my-skill
description: What this skill does
---
# My Skill
Instructions for the agent...
```

**Security Audit (10 checks):**

1. Manifest required (SKILL.md or SKILL.toml)
2. Symlinks blocked
3. Script files blocked by default (.sh, .bash, .zsh, .fish, .ps1, .bat, .cmd)
4. Shebang detection
5. File size limit (>512KB = rejection)
6. Markdown link validation (remote .md links blocked, cross-skill references allowed)
7. TOML validation
8. High-risk pattern detection (curl | sh, invoke-expression, rm -rf /, fork bombs, etc.)
9. Shell chaining blocked (&&, ||, ;, $(), backticks)
10. Empty command rejection

**Double Audit:** Once on source, once on installed copy (after copy for local installs).

**Open-Skills Sync:**
```rust
const OPEN_SKILLS_REPO_URL = "https://github.com/besoeasy/open-skills";
const OPEN_SKILLS_SYNC_INTERVAL_SECS = 60 * 60 * 24 * 7;  // Weekly
// Clone --depth 1 on first run
// Pull --ff-only on subsequent runs
```

**Technical Debt:** open-skills repo sync on every startup if marker is old - no jitter to avoid GitHub thundering herd.

---

### Feature 10: Cron & Scheduling

**Key files:** `src/cron/scheduler.rs` (51KB), `src/cron/store.rs` (60KB), `src/cron/schedule.rs` (11KB)

Persistent SQLite-backed scheduler with multiple schedule types, two job types, delivery notifications, and declarative configuration sync.

**Schedule Types:**
```rust
pub enum Schedule {
    Cron { expr: String, tz: Option<String> },  // 5-field or 6-field cron
    At   { at: DateTime<Utc> },                  // One-shot at specific time
    Every { every_ms: u64 },                    // Interval-based
}
```

**Weekday Normalization (The Clever Part):**

The `cron` crate uses 1=Sunday, but standard crontab uses 0=Sunday. `normalize_weekday_field()` translates automatically:

```rust
// schedule.rs:243-244
let normalized_weekday = normalize_weekday_field(weekday)?;
fields[4] = &normalized_weekday;
Ok(format!("0 {} {} {} {} {}", fields[0], fields[1], fields[2], fields[3], fields[4]))
```

**Job Execution:**

| Type | Execution | Security |
|------|-----------|----------|
| `Shell` | `sh -c <command>` via `tokio::process::Command` | Policy validation + `allowed_tools` restriction |
| `Agent` | Runs LLM agent with prefixed prompt | `can_act()`, `is_rate_limited()`, `record_action()` checks |

**Non-login Shell:** Uses `sh -c` (non-login), NOT `sh -lc` (login shell). Comment explains: "login shells load the full user profile on every invocation which is slow and may cause side effects."

**Declarative/Imperative Split:**

- Jobs in config but not DB: INSERT
- Jobs in DB and config: UPDATE (preserving runtime state)
- Jobs in DB with `source='declarative'` but not in config: DELETE
- **Imperative jobs are never touched**

**Credential Redaction:** Before delivering output to channels, `scan_and_redact_output()` runs output through `LeakDetector` to redact API keys, tokens, passwords.

---

### Feature 11: Tunnel & Remote Access

**Key files:** `src/tunnel/mod.rs` (15KB), `src/tunnel/cloudflare.rs`, `src/tunnel/tailscale.rs`

Pluggable abstraction for exposing the local Gateway to the internet via `Tunnel` trait.

**Tunnel Trait:**
```rust
#[async_trait::async_trait]
pub trait Tunnel: Send + Sync {
    fn name(&self) -> &str;
    async fn start(&self, local_host: &str, local_port: u16) -> Result<String>;  // Returns public URL
    async fn stop(&self) -> Result<()>;
    async fn health_check(&self) -> bool;
    fn public_url(&self) -> Option<String>;
}
```

**Supported Providers:** cloudflare, tailscale, ngrok, openvpn, pinggy, custom, none

**SharedProcess Pattern:**
```rust
pub(crate) struct TunnelProcess {
    pub child: tokio::process::Child,
    pub public_url: String,
}
pub(crate) type SharedProcess = Arc<Mutex<Option<TunnelProcess>>>;
```

**CloudflareTunnel URL Extraction (The Clever Part):**

cloudflared prints URLs to stderr mixed with warning/docs links. `extract_tunnel_url()` filters:

```rust
fn extract_tunnel_url(line: &str) -> Option<String> {
    let is_tunnel_line = line.contains("Visit it at")
        || line.contains("Route at")
        || line.contains("Registered tunnel connection");
    let is_tunnel_domain = candidate.contains(".trycloudflare.com");
    let is_docs_url = candidate.contains("github.com")
        || candidate.contains("cloudflare.com/docs");
    if is_tunnel_line || is_tunnel_domain || !is_docs_url {
        Some(candidate.to_string())
    } else {
        None
    }
}
```

---

### Feature 12: Hardware Peripherals

**Key files:** `src/hardware/mod.rs` (27KB), `src/hardware/device.rs` (30KB), `src/hardware/gpio.rs` (22KB), `src/peripherals/`

Connects to physical devices (microcontrollers, SBCs, debug adapters) and exposes capabilities as LLM-callable tools.

**Architecture:**
```
LLM Agent Tool Calls
        |
        v
GPIO tools (gpio_read, gpio_write) + Board tools (flash, upload, memory_*)
        |
        v
Transport Trait (serial, SWD, UF2, Native, Aardvark)
        |
        v
Physical Device (Pico, Arduino, Nucleo, RPi, Aardvark adapter)
```

**Device Registry with Aliases:**

```rust
pub fn register(&mut self, board_name: &str, vid: Option<u16>, pid: Option<u16>,
                device_path: Option<String>, architecture: Option<String>) -> String {
    let prefix = alias_prefix(board_name);  // "pico", "arduino", "nucleo", etc.
    let counter = self.alias_counters.entry(prefix.clone()).or_insert(0);
    let alias = format!("{}{}", prefix, counter);
}
```

LLM never sees `/dev/ttyACM0`, only `pico0`, `arduino0`.

**VID-Based Device Identification:**
```rust
pub fn from_vid(vid: u16) -> Option<Self> {
    match vid {
        0x2e8a => Some(Self::Pico),       // Raspberry Pi Pico
        0x2341 => Some(Self::Arduino),     // Arduino
        0x10c4 => Some(Self::Esp32),       // ESP32 via CP2102
        0x0483 => Some(Self::Nucleo),      // STM32 Nucleo
        0x2b76 => Some(Self::Aardvark),   // Total Phase Aardvark
        _ => None,
    }
}
```

**Ping Handshake for Unknown VIDs:**

For unknown VID (0), runs `ping_handshake()` - sends `{"cmd":"ping"}` and waits for response. Only registers if ZeroClaw firmware responds. Prevents registering random USB-serial adapters.

**Registry Lock Not Held Across Async I/O (The Clever Part):**
```rust
let (device_alias, ctx) = {
    let registry = self.registry.read().await;
    match registry.resolve_gpio_device(&args) {
        Ok(resolved) => resolved,
        Err(msg) => return Err(ToolResult { success: false, error: Some(msg), .. }),
    }
}; // registry read guard dropped here before async I/O
```

**Technical Debt:** Dual codebase (hardware/ vs peripherals/) with some duplication. Ping handshake assumes ZeroClaw firmware - generic devices without it are rejected.

---

## Cross-Cutting Observations

### Shared Patterns

**Error Handling:** All features use `anyhow::Result` for ergonomic error handling with typed errors for specific failure modes.

**Async Patterns:**
- `async_trait` for trait-based async execution
- `tokio::spawn` for concurrent operations
- `CancellationToken` for graceful shutdown
- `tokio::sync::mpsc` for message passing

**tokio::task_local for Request-Scoped State:**
Used in ReliableProvider for fallback info that must not leak across concurrent requests.

**Defensive Lock Patterns:**
- Two-phase locks in SOP dispatch (minimize lock hold time)
- `try_lock()` for non-blocking public_url() queries
- Registry locks not held across async I/O

**Idempotent Schema Migrations:**
Present in both memory (`schema_sql.contains()` before ALTER) and cron (tolerates "duplicate column name" errors).

### Security Integration

| Feature | Security Mechanism |
|---------|-------------------|
| Cron | Security policy checks before shell/agent execution |
| SOPs | Fail-closed conditions, approval gates, deterministic audit |
| Skills | 10-check audit before load, script blocking, path traversal protection |
| Gateway | Bearer token auth, secret masking, idempotency |
| Hardware | Tools gated behind registry; raw device paths never exposed to LLM |
| Tools | UUID validation for task IDs, credential scrubbing |

### Configuration Patterns

| Feature | Config Location | Format |
|---------|-----------------|--------|
| Cron | `[cron.jobs]` in config.toml | Sync to SQLite via `cron sync` |
| Tunnel | `[tunnel]` section | Provider + provider-specific subsections |
| Hardware | `[peripherals]` section | `[[peripherals.boards]]` entries |
| Skills | `~/.zeroclaw/open-skills/` | Git-synced directory |
| SOPs | `<workspace>/sops/<name>/` | SOP.toml + SOP.md |

### Testing Culture

| Feature | Inline Tests |
|---------|--------------|
| Memory | Schema, recall, embedding cache, hybrid search |
| Security | Policy parsing, quote-aware lexer, path validation |
| Provider | Retry logic, error classification, context truncation |
| Cron | 60+ tests covering parsing, CRUD, retry, delivery, one-shot |
| Tunnel | ~25 tests covering factory, URL extraction, health, lifecycle |
| Hardware | 40+ tests covering alias, VID mapping, registry, GPIO, protocol |
| SOPs | Engine state machine, condition evaluation, dispatch |
| Skills | Audit checks, manifest parsing, installation |

---

## Architecture Risks & Technical Debt

### File Size Concerns

| File | Size | Concern |
|------|------|---------|
| `src/agent/loop_.rs` | 346KB | Decomposition would aid maintainability |
| `src/channels/mod.rs` | 415KB | Extremely large for a single module |
| `src/tools/delegate.rs` | 99KB | Handles many concerns (delegation, parallel, background) |
| `src/providers/reliable.rs` | 111KB | Complex nested retry/failover logic |
| `src/providers/mod.rs` | 129KB | Large factory with many provider constructions |

### Global State

- `MODEL_SWITCH_REQUEST` static in agent loop - not ideal for testability
- `ActionTracker` cloned trackers are independent (each gets own `Mutex<Vec<Instant>>`)

### Inconsistencies

- Custom ISO-8601 in `engine.rs` while `dispatch.rs` uses `chrono::Utc::now()`
- API key rotation code exists but cannot actually apply keys (trait lacks `set_api_key`)
- Dual codebase for hardware (hardware/ vs peripherals/)

### Missing Cleanup

- Background delegate result files have no TTL or cleanup mechanism
- open-skills repo sync lacks jitter to avoid GitHub thundering herd

### Feature-Gate Complexity

Matrix, Lark, Nostr require `channel-matrix`, `channel-lark`, `channel-nostr` features. RPi tools require both `hardware` feature AND Linux OS AND `peripheral-rpi` feature.

---

## Files Analyzed Summary

| Feature | Primary Files | Total Size |
|---------|--------------|------------|
| Agent Loop | loop_.rs, agent.rs, dispatcher.rs, prompt.rs, context_compressor.rs | ~500KB |
| Multi-Channel | mod.rs, traits.rs, 26 channel implementations | ~700KB |
| Tool System | mod.rs, delegate.rs, traits.rs, 70+ tool implementations | ~300KB |
| Memory | sqlite.rs, knowledge_graph.rs, embeddings.rs, qdrant.rs, hygiene.rs | ~350KB |
| Security | policy.rs, webauthn.rs, audit.rs, pairing.rs, iam_policy.rs, secrets.rs | ~400KB |
| Provider | mod.rs, reliable.rs, router.rs, 10+ provider implementations | ~600KB |
| Gateway | mod.rs, api.rs, ws.rs, sse.rs, TLS, 14 React pages | ~300KB |
| SOPs | mod.rs, engine.rs, dispatch.rs, condition.rs, metrics.rs | ~300KB |
| Skills | mod.rs, audit.rs, creator.rs, improver.rs | ~150KB |
| Cron | scheduler.rs, store.rs, schedule.rs, types.rs, 7 cron tools | ~180KB |
| Tunnel | mod.rs, cloudflare.rs, tailscale.rs, ngrok.rs, openvpn.rs, pinggy.rs, custom.rs | ~60KB |
| Hardware | mod.rs, device.rs, gpio.rs, transport.rs, protocol.rs, peripherals/ | ~150KB |

---

*Synthesized from four batches of feature deep-dive research covering all 12 features documented in the ZeroClaw feature index.*
