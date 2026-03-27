# Features Batch 4 Deep Dive

**Date:** 2026-03-26
**Features Covered:** Feature 10 (Scheduling/Cron/CalDAV), Feature 11 (Skills System), Feature 12 (Hook/Plugin Architecture)

---

## Feature 10: Scheduling (Cron/CalDAV)

### Overview

Scheduling is implemented across two crates: `moltis-cron` for cron job scheduling and `moltis-caldav` for CalDAV calendar integration. They work together to provide time-based automation.

### Architecture

```
crates/cron/          crates/caldav/
    lib.rs                lib.rs
    types.rs              client.rs
    service.rs            types.rs
    schedule.rs           ical.rs
    heartbeat.rs          tool.rs
    system_events.rs      discovery.rs
    store.rs              bundled/
    store_file.rs
    store_sqlite.rs
    store_memory.rs
    error.rs
    tool.rs
```

### Crate: `moltis-cron`

**Purpose:** Persistent cron scheduler with support for cron expressions, fixed intervals, and one-shot scheduling.

#### Key Data Structures

**`CronSchedule` (types.rs:19-34)** - Three scheduling modes:
```rust
pub enum CronSchedule {
    At { at_ms: u64 },              // One-shot at epoch ms
    Every { every_ms: u64, anchor_ms: Option<u64> },  // Fixed interval
    Cron { expr: String, tz: Option<String> }, // Cron expression
}
```

**`CronPayload` (types.rs:37-56)** - What a job fires:
```rust
pub enum CronPayload {
    SystemEvent { text: String },   // Inject into main session
    AgentTurn {                    // Isolated agent turn
        message: String,
        model: Option<String>,
        timeout_secs: Option<u64>,
        deliver: bool,             // Send to channel
        channel: Option<String>,
        to: Option<String>,
    },
}
```

**`SessionTarget` (types.rs:59-69)** - Where jobs run:
```rust
pub enum SessionTarget {
    Main,              // Main conversation session
    Isolated,          // Throwaway session (default)
    Named(String),     // Persistent named session (e.g. "heartbeat")
}
```

**`CronJob` (types.rs:99-126)** - Full job definition with sandbox config, wake mode, and runtime state.

**`CronRunRecord` (types.rs:129-145)** - Persisted record of each job execution with input/output tokens.

#### `CronService` (service.rs)

The core scheduler with these key methods:
- `start()` / `stop()` - lifecycle management
- `add()` / `update()` / `remove()` - CRUD operations
- `run()` - force-run a job immediately
- `wake()` - wake the heartbeat job immediately

**Notable Design Decisions:**

1. **Wake Mode** (`CronWakeMode`): Controls whether the heartbeat fires immediately after a job completes:
   - `Now` - trigger immediate heartbeat
   - `NextHeartbeat` - wait for next scheduled tick (default)

2. **Stuck Job Detection** (service.rs:131-132): Jobs running >2 hours are cleared and marked as error.

3. **Rate Limiting** (service.rs:49-101): Sliding window rate limiter (default 10 jobs/minute), system jobs bypass it.

4. **Session + Payload Validation** (service.rs:701-738): `Main` session requires `SystemEvent` payload; `Isolated/Named` requires `AgentTurn`. `deliver=true` requires both `channel` and `to`.

5. **Atomic State Updates**: Job `running_at_ms` is set BEFORE spawning the async task, preventing duplicate runs.

#### Schedule Computation (schedule.rs)

`compute_next_run()` handles:
- **At**: Simple comparison with now_ms
- **Every**: Computes next interval based on anchor
- **Cron**: Uses the `cron` crate; auto-converts 5-field to 6-field expressions by prepending "0 " and appending " *"

#### Persistence (store implementations)

Three backends implementing `CronStore` trait:

1. **`FileStore`** (store_file.rs): JSON jobs file + JSONL per-job run history with atomic writes and `.bak` backup.

2. **`SqliteStore`** (store_sqlite.rs): Stores jobs as JSON blobs, runs as structured rows with upsert.

3. **`InMemoryStore`** (store_memory.rs): For testing only.

#### Heartbeat (heartbeat.rs)

The heartbeat is a special system job (`__heartbeat__`) that periodically prompts the LLM to check for noteworthy items.

**Key Concepts:**
- `HEARTBEAT_OK` sentinel token - LLM responds with this when nothing needs attention
- `StripMode::Exact` vs `StripMode::Trim` for token removal
- `is_within_active_hours()` handles overnight windows (22:00-06:00)
- `resolve_heartbeat_prompt()` precedence: config > `HEARTBEAT.md` file > default

#### System Events Queue (system_events.rs)

Bounded queue (20 events max) that accumulates events between heartbeat ticks:
- Deduplicates consecutive identical events
- FIFO drain on heartbeat tick
- Events prepended to heartbeat prompt with `[reason]` tags

### Crate: `moltis-caldav`

**Purpose:** CalDAV client for calendar CRUD operations via `libdav`.

#### Key Architecture

**`CalDavClient` trait** (client.rs:18-43) - Abstraction allowing mock/test implementations:
```rust
pub trait CalDavClient: Send + Sync {
    async fn list_calendars(&self) -> Result<Vec<CalendarInfo>>;
    async fn list_events(&self, calendar_href: &str, range: Option<TimeRange>) -> Result<Vec<EventSummary>>;
    async fn create_event(&self, calendar_href: &str, event: NewEvent) -> Result<CreatedEvent>;
    async fn update_event(&self, href: &str, etag: &str, updates: UpdateEvent) -> Result<UpdatedEvent>;
    async fn delete_event(&self, href: &str, etag: &str) -> Result<()>;
}
```

**`LibDavCalDavClient`** (client.rs:45-128) - Production implementation using `libdav` with:
- `hyper_rustls` for HTTPS
- `tower_http::auth::AddAuthorization` for Basic auth
- Service discovery to locate CalDAV context path
- Property requests for display name, color, description

#### Provider Support (discovery.rs)

Hardcoded well-known URLs:
- Fastmail: `https://caldav.fastmail.com`
- iCloud: `https://caldav.icloud.com`

#### iCalendar Handling (ical.rs)

- `build_vevent()` - Creates VCALENDAR/VEVENT strings from `NewEvent`
- `parse_events()` - Extracts events from raw iCal data, detects all-day by 8-digit DTSTART
- `merge_updates()` - Partial updates on existing iCal data (preserves unmodified properties)
- `normalise_datetime()` - Converts basic format (20250615T100000) to ISO 8601

#### `CalDavTool` AgentTool (tool.rs)

Lazy client connection on first use. Supports multiple accounts with auto-detection of single-account scenarios.

**Operations:** `list_calendars`, `list_events`, `create_event`, `update_event`, `delete_event`

ETags required for update/delete (optimistic concurrency control).

---

## Feature 11: Skills System

### Overview

The skills system enables agents to acquire new capabilities through structured skill definitions. Skills are directories containing a `SKILL.md` file with YAML frontmatter and markdown instructions.

**Crate:** `moltis-skills`

### Directory Structure

```
crates/skills/
    lib.rs
    types.rs           # Core types (SkillsManifest, SkillMetadata, SkillContent)
    manifest.rs        # Persistent manifest with atomic writes
    discover.rs        # Filesystem discovery of skills
    registry.rs       # In-memory registry with async_trait
    requirements.rs    # Binary requirement checking
    parse.rs           # SKILL.md parsing (15.6K LOC)
    formats.rs         # Plugin format detection (29.4K LOC)
    install.rs         # Git-based installation (17.4K LOC)
    prompt_gen.rs      # Prompt generation from skills
    migration.rs       # Migration from old format
    watcher.rs         # Optional filesystem watcher
```

### Core Types

**`SkillMetadata` (types.rs:130-169)** - Lightweight metadata from frontmatter:
```rust
pub struct SkillMetadata {
    pub name: String,           // Internal name (slug)
    pub slug: Option<String>,
    pub display_name: Option<String>,  // Original human-readable name
    pub description: String,
    pub homepage: Option<String>,
    pub license: Option<String>,
    pub compatibility: Option<String>,
    pub allowed_tools: Vec<String>,
    pub dockerfile: Option<String>,
    pub requires: SkillRequirements,
    pub path: PathBuf,          // Filesystem location
    pub source: Option<SkillSource>,
}

pub enum SkillSource {
    Project,    // <data_dir>/.moltis/skills/
    Personal,   // <data_dir>/skills/
    Plugin,     // Bundled in plugin
    Registry,   // Installed from registry (e.g. skills.sh)
}
```

**`SkillRequirements` (types.rs:173-185)** - Binary/tool dependencies:
```rust
pub struct SkillRequirements {
    pub bins: Vec<String>,          // All must exist
    pub any_bins: Vec<String>,       // At least one must exist
    pub install: Vec<InstallSpec>,   // How to install missing
}

pub enum InstallKind {
    Brew, Npm, Go, Cargo, Uv, Download
}
```

**`SkillEligibility` (types.rs:221-228)** - Requirement check result:
```rust
pub struct SkillEligibility {
    pub eligible: bool,
    pub missing_bins: Vec<String>,
    pub install_options: Vec<InstallSpec>,
}
```

**`SkillsManifest` (types.rs:10-62)** - Top-level tracking of installed repos:
```rust
pub struct SkillsManifest {
    pub version: u32,
    pub repos: Vec<RepoEntry>,
}

pub struct RepoEntry {
    pub source: String,        // "owner/repo"
    pub repo_name: String,
    pub installed_at_ms: u64,
    pub commit_sha: Option<String>,
    pub format: PluginFormat,  // Skill, ClaudeCode, etc.
    pub skills: Vec<SkillState>, // Per-skill enabled/trusted
}
```

### Discovery (`discover.rs`)

**Search Paths (priority order):**
1. `~/.moltis/skills/` - Project-local
2. `~/.moltis/skills/` (personal)
3. `~/.moltis/installed-skills/` - Registry-installed
4. `~/.moltis/installed-plugins/` - Plugin skills

**Discovery Logic:**
- **Project/Personal**: Flat scan of directories for `SKILL.md` files
- **Registry**: Uses manifest for enabled filtering; parses `SKILL.md` for full metadata
- **Plugin**: Uses plugins manifest; non-SKILL.md formats get stub metadata

### Registry (`registry.rs`)

**`SkillRegistry` trait** - Abstraction for skill management:
```rust
pub trait SkillRegistry: Send + Sync {
    async fn list_skills(&self) -> anyhow::Result<Vec<SkillMetadata>>;
    async fn load_skill(&self, name: &str) -> anyhow::Result<SkillContent>;
    async fn install_skill(&self, source: &str) -> anyhow::Result<SkillMetadata>;
    async fn remove_skill(&self, name: &str) -> anyhow::Result<()>;
}
```

Only registry-installed skills can be removed (enforced).

### Requirements Checking (`requirements.rs`)

- `check_bin()` - Uses `std::env::split_paths` + `is_executable` (Unix permissions check)
- `check_requirements()` - Returns `SkillEligibility` with missing bins and OS-filtered install options
- `install_program_and_args()` - Maps `InstallKind` to commands (brew, npm, go, cargo, uv)
- **Security Note**: npm installs use `--ignore-scripts` flag to prevent supply chain attacks

### Manifest Persistence (`manifest.rs`)

Atomic writes via temp file + rename:
```rust
pub fn save(&self, manifest: &SkillsManifest) -> anyhow::Result<()> {
    let tmp = self.path.with_extension("json.tmp");
    std::fs::write(&tmp, data)?;
    std::fs::rename(&tmp, &self.path)?;  // Atomic on POSIX
}
```

---

## Feature 12: Hook/Plugin Architecture

### Overview

Event-driven hook system with 17 lifecycle event types, circuit breaker pattern, and support for both native Rust handlers and shell-based external commands.

**Crate:** `moltis-plugins`

### Directory Structure

```
crates/plugins/
    lib.rs
    error.rs
    hooks.rs          # Re-exports from moltis-common + config types
    hook_discovery.rs # Filesystem discovery of HOOK.md files
    hook_metadata.rs # TOML frontmatter parsing
    hook_eligibility.rs # OS/bin/env requirement checks
    shell_hook.rs     # External command execution handler
    bundled/
        mod.rs
        boot_md.rs      # Injects BOOT.md on GatewayStart
        command_logger.rs # Logs Command events to JSONL
        session_memory.rs # Saves session to markdown on new/reset
```

### Core Hook Types (`moltis-common/src/hooks.rs`)

**`HookEvent` (17 types):**
```rust
pub enum HookEvent {
    BeforeAgentStart, AgentEnd,
    BeforeLLMCall, AfterLLMCall,
    BeforeCompaction, AfterCompaction,
    MessageReceived, MessageSending, MessageSent,
    BeforeToolCall, AfterToolCall, ToolResultPersist,
    SessionStart, SessionEnd,
    GatewayStart, GatewayStop,
    Command,
}
```

**`HookPayload`** - Typed payloads per event (e.g., `BeforeToolCall` includes `session_key`, `tool_name`, `arguments`).

**`HookAction`:**
```rust
pub enum HookAction {
    Continue,                    // Proceed normally
    ModifyPayload(Value),         // Replace/modify data
    Block(String),               // Block with reason
}
```

### `HookHandler` Trait

```rust
pub trait HookHandler: Send + Sync {
    fn name(&self) -> &str;
    fn events(&self) -> &[HookEvent];
    fn priority(&self) -> i32 { 0 }  // Higher runs first
    async fn handle(&self, event: HookEvent, payload: &HookPayload) -> Result<HookAction>;
    fn handle_sync(&self, ...) { ... }  // Hot-path optimization
}
```

### `HookRegistry` (`moltis-common`)

**Key features:**
- **Circuit Breaker**: Auto-disables handlers after 3 consecutive failures; re-enables after 60s cooldown
- **Priority Ordering**: Higher priority handlers run first
- **Parallel vs Sequential**: Read-only events (`SessionStart`, `AgentEnd`, etc.) dispatch in parallel; modifying events (`BeforeToolCall`, `BeforeLLMCall`) dispatch sequentially
- **Dry Run Mode**: Block/Modify actions are logged but not applied
- **Per-handler Stats**: `HookStats` tracks call count, failures, latency

**Dispatch Logic:**
- Read-only events: Parallel dispatch, Block/Modify ignored
- Modifying events: Sequential, first Block short-circuits, last Modify wins

### Hook Discovery (`hook_discovery.rs`)

**Search Paths:**
- `~/.moltis/hooks/` - Project
- `~/.moltis/hooks/` (user)

**Discovery:** Scans directories for `HOOK.md` files, parses TOML frontmatter.

### Hook Metadata (`hook_metadata.rs`)

TOML frontmatter in `HOOK.md`:
```toml
+++
name = "my-hook"
description = "What it does"
emoji = "🔧"
events = ["BeforeToolCall", "AfterToolCall"]
command = "./handler.sh"
timeout = 5
priority = 10

[requires]
os = ["darwin", "linux"]
bins = ["jq"]
env = ["SLACK_WEBHOOK_URL"]
+++
```

### Hook Eligibility (`hook_eligibility.rs`)

```rust
pub fn check_hook_eligibility(metadata: &HookMetadata) -> EligibilityResult {
    // Checks: OS match, bins exist, env vars set
    // Note: config requirements skipped (requires MoltisConfig access)
}
```

Uses `which` command for binary existence check.

### Shell Hook Handler (`shell_hook.rs`)

Spawns `sh -c "<command>"` with payload as JSON on stdin.

**Protocol:**
- Exit 0, no stdout -> `Continue`
- Exit 0, stdout `{"action": "modify", "data": {...}}` -> `ModifyPayload`
- Exit 1 -> `Block` (stderr is reason)
- Exit other -> Error (non-fatal, logged)
- Timeout -> Error

**Features:**
- Configurable timeout
- Environment variable injection
- Working directory support
- Broken pipe handling (ignores if hook doesn't read stdin)

### Bundled Hooks

**`boot-md`** - On `GatewayStart`, reads `BOOT.md` from workspace and returns as `ModifyPayload` for startup injection. Strips leading HTML comments.

**`command-logger`** - Appends `Command` events to JSONL file at `~/.moltis/logs/commands.log`. Has async and sync handlers.

**`session-memory`** - On `Command` with `new`/`reset` action, saves session conversation to `memory/session-<key>-<date>.md`. Truncates long messages and limits to 50 most recent.

### Hook Configuration (`hooks.rs` in plugins)

```rust
pub struct ShellHookConfig {
    pub name: String,
    pub command: String,
    pub events: Vec<HookEvent>,
    pub timeout: u64,
    pub env: HashMap<String, String>,
}
```

Allows config-file-based hooks (not requiring HOOK.md discovery).

---

## Cross-Cutting Observations

### Strengths

1. **Excellent Test Coverage**: All three crates have comprehensive test suites embedded in the source files
2. **Trait-based Abstractions**: `CronStore`, `CalDavClient`, `SkillRegistry`, `HookHandler` all use `async_trait` for clear interfaces
3. **Security Considerations**: npm installs use `--ignore-scripts`, shell hooks have timeout protection, hook eligibility checks OS/bins/env
4. **Persistence Patterns**: Atomic writes via temp+rename, JSONL for append-only logs
5. **Circuit Breaker**: Hook system has resilience pattern for failing external handlers

### Technical Debt / Concerns

1. **`__heartbeat__` Magic Name**: Heartbeat job ID is hardcoded string in two places (service.rs, heartbeat.rs). Should be a constant.
2. **Config Requirement Check Skipped**: `hook_eligibility.rs:56` notes config check is deferred - this is a gap
3. **libdav Dependency**: `CalDavTool` depends on external `libdav` crate; no embedded mock for unit tests
4. **Error Handling in Shell Hooks**: Exit codes other than 0/1 treated as errors; could be more granular
5. **Bin Existence Check**: Uses `which` command rather than `std::env::split_paths` for cross-platform consistency

### Notable Code Patterns

1. **Async trait pattern** with `Arc<dyn Trait + Send + Sync>` throughout
2. **`tracing` for structured logging** with `tracing::warn!` for expected "not configured" states
3. **`serde` serialization** with `#[serde(skip_serializing_if = "Option::is_none")]` for optional fields
4. **Bounded queues** with FIFO eviction (SystemEventsQueue, MAX_EVENTS=20)
5. **Dedup via consecutive comparison** (SystemEventsQueue: `events.back().is_some_and(|last| last.text == text)`)
