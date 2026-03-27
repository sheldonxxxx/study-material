# ZeroClaw Feature Deep Dive: Batch 4

**Features Covered:** Cron & Scheduling, Tunnel & Remote Access, Hardware Peripherals
**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Research Date:** 2026-03-26

---

## Feature 10: Cron & Scheduling

### Overview

ZeroClaw's cron system (`src/cron/`) is a sophisticated persistent scheduler with SQLite-backed storage, supporting multiple schedule types, two job types (Shell and Agent), delivery notifications to messaging channels, and declarative configuration. It is composed of:

- `scheduler.rs` (51KB) — async runtime, job execution, delivery, retry logic
- `store.rs` (60KB) — SQLite persistence, CRUD operations, schema migrations
- `schedule.rs` (11KB) — cron expression parsing, timezone support, weekday normalization
- `types.rs` (7KB) — domain types: `CronJob`, `CronRun`, `Schedule`, `JobType`, `SessionTarget`, `DeliveryConfig`

### Schedule Types

The `Schedule` enum (types.rs:88-101) supports three variants:

```rust
pub enum Schedule {
    Cron { expr: String, tz: Option<String> },  // 5-field or 6-field cron
    At   { at: DateTime<Utc> },                  // One-shot at specific time
    Every { every_ms: u64 },                    // Interval-based
}
```

**Key insight — weekday normalization:** The `cron` crate used internally uses 1=Sunday, 2=Monday...7=Saturday, but standard crontab uses 0=Sunday, 1=Monday...6=Saturday (and 7 as Sunday alias). The `normalize_weekday_field()` function in `schedule.rs` translates between these conventions automatically. This means users can write `0 9 * * 1-5` (standard crontab: Mon-Fri at 9am UTC) and it correctly becomes `0 0 9 * * 2-6` internally (schedule.rs:243-244).

```rust
// schedule.rs:79-86 — weekday translation from crontab to cron-crate semantics
let weekday = fields[4];
let normalized_weekday = normalize_weekday_field(weekday)?;
fields[4] = &normalized_weekday;
Ok(format!("0 {} {} {} {} {}", fields[0], fields[1], fields[2], fields[3], fields[4]))
```

### Job Execution

Two job types (`JobType` enum, types.rs:32-36):

| Type | Execution | Security |
|------|-----------|----------|
| `Shell` | `sh -c <command>` via `tokio::process::Command` | Security policy validation + `allowed_tools` restriction |
| `Agent` | Runs LLM agent with prefixed prompt | `can_act()`, `is_rate_limited()`, `record_action()` checks |

**Execution flow (scheduler.rs):**

1. `execute_job_with_retry()` — retries up to `config.reliability.scheduler_retries` times with exponential backoff + jitter (scheduler.rs:142-175)
2. Deterministic security policy violations are NOT retried — they fail immediately (scheduler.rs:162-165)
3. `persist_job_result()` handles delivery, one-shot deletion, and rescheduling atomically (scheduler.rs:295-359)

**Important execution detail — non-login shell:** The shell command uses `sh -c` (non-login), NOT `sh -lc` (login shell). The comment in `build_cron_shell_command()` explains: "login shells load the full user profile on every invocation which is slow and may cause side effects" (scheduler.rs:716-720).

### SQLite Schema

The cron store (`store.rs`) uses a workspace-relative path: `workspace_dir/cron/jobs.db`.

```sql
-- cron_jobs table with declarative/imperative source tracking
CREATE TABLE cron_jobs (
    id               TEXT PRIMARY KEY,
    expression       TEXT NOT NULL,
    command          TEXT NOT NULL,
    schedule         TEXT,          -- JSON-encoded Schedule
    job_type         TEXT NOT NULL DEFAULT 'shell',
    prompt           TEXT,
    name             TEXT,
    session_target   TEXT NOT NULL DEFAULT 'isolated',
    model            TEXT,
    enabled          INTEGER NOT NULL DEFAULT 1,
    delivery         TEXT,          -- JSON-encoded DeliveryConfig
    delete_after_run INTEGER NOT NULL DEFAULT 0,
    allowed_tools    TEXT,          -- JSON-encoded Vec<String>
    source           TEXT DEFAULT 'imperative',  -- 'declarative' or 'imperative'
    created_at       TEXT NOT NULL,
    next_run         TEXT NOT NULL,
    last_run         TEXT,
    last_status      TEXT,
    last_output      TEXT
);
-- Index on next_run for efficient due_jobs queries
CREATE INDEX idx_cron_jobs_next_run ON cron_jobs(next_run);

-- cron_runs table for run history (auto-pruned)
CREATE TABLE cron_runs (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    job_id      TEXT NOT NULL,
    started_at  TEXT NOT NULL,
    finished_at TEXT NOT NULL,
    status      TEXT NOT NULL,
    output      TEXT,
    duration_ms INTEGER,
    FOREIGN KEY (job_id) REFERENCES cron_jobs(id) ON DELETE CASCADE
);
```

**Migration-safe schema:** `add_column_if_missing()` tolerates "duplicate column name" errors from concurrent migrations (store.rs:847-875). The `source` column tracks whether a job was created via CLI/API (`imperative`) or from config (`declarative`).

### Declarative Configuration Sync

`sync_declarative_jobs()` (store.rs:584-773) reconciles config-defined jobs with the database:

- Jobs in config but not DB → INSERT
- Jobs in DB and config → UPDATE (preserving runtime state: `next_run`, `last_run`, `last_status`)
- Jobs in DB with `source='declarative'` but not in config → DELETE
- **Imperative jobs are never touched**

The sync is transactional and validates each declaration before touching the DB (store.rs:608-611).

### Delivery / Announcements

Cron jobs can deliver their output to messaging channels after completion (scheduler.rs:401-625). The `DeliveryConfig` struct:

```rust
pub struct DeliveryConfig {
    pub mode: String,       // "none" or "announce"
    pub channel: Option<String>,  // "telegram", "discord", "slack", etc.
    pub to: Option<String>,       // Channel-specific target (chat ID, channel, etc.)
    pub best_effort: bool,       // If true, delivery failure doesn't mark job as failed
}
```

**Security — credential redaction:** Before delivering output to a channel, `scan_and_redact_output()` runs the output through `LeakDetector` to redact any API keys, tokens, or passwords. This prevents accidental credential exfiltration (scheduler.rs:433-449).

### One-Shot Jobs (`At` schedule)

`Schedule::At` jobs have `delete_after_run = true` and are handled specially:

- On success: job is **deleted** from DB (store.rs:324-327)
- On failure: job is **disabled** (not deleted, so last status/output is preserved) (store.rs:338-350)
- After execution, `reschedule_after_run()` disables `At` jobs (not one-shot) to prevent re-execution with past timestamps (store.rs:341-351)

### Startup Catch-Up

If the machine was off or the daemon was restarted, regular polling would only pick up `max_tasks` jobs per cycle. The `catch_up_overdue_jobs()` function runs at startup and queries **all** overdue jobs (ignoring `max_tasks`) to ensure nothing is missed (scheduler.rs:78-88, 108-135).

### Key Files Traced

| File | Purpose |
|------|---------|
| `src/cron/scheduler.rs` | Execution runtime, job dispatch, delivery |
| `src/cron/store.rs` | SQLite persistence, CRUD, declarative sync |
| `src/cron/schedule.rs` | Cron expression parsing, timezone normalization |
| `src/cron/types.rs` | Domain types: CronJob, CronRun, Schedule, DeliveryConfig |
| `src/cron/mod.rs` | Public re-exports and top-level API |
| `src/tools/cron_*.rs` | Tool implementations (add, remove, list, run, etc.) |

### Notable Design Decisions

1. **Weekday normalization** is automatic — users write standard crontab, system translates internally
2. **Non-login shell** for command execution — avoids profile side effects
3. **Declarative/imperative split** allows config to own certain jobs without overriding user-created ones
4. **Credential scanning** before channel delivery prevents accidental secrets leakage
5. **Exponential backoff with jitter** for retries — `backoff_ms * 2^attempt` with `jitter_ms = now % 250`

---

## Feature 11: Tunnel & Remote Access

### Overview

ZeroClaw's tunnel system (`src/tunnel/`) provides a pluggable abstraction for exposing the local Gateway to the internet. It supports multiple providers via a shared `Tunnel` trait and factory pattern. All tunnels wrap external binaries (`cloudflared`, `tailscale`, `ngrok`, etc.) as child processes.

Key file: `src/tunnel/mod.rs` (15KB)

### Tunnel Trait

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

### Supported Providers

| Provider | Config Section | Key Feature |
|----------|---------------|-------------|
| `cloudflare` | `tunnel.cloudflare.token` | Zero Trust tunnels, TLS |
| `tailscale` | `tunnel.tailscale.funnel` (bool), `hostname` | `tailscale serve` (tailnet) or `funnel` (public) |
| `ngrok` | `tunnel.ngrok.auth_token`, `domain` | Custom domains, auth |
| `openvpn` | `tunnel.openvpn.config_file`, `auth_file` | Full VPN |
| `pinggy` | `tunnel.pinggy.token`, `region` | SSH-based tunnel |
| `custom` | `tunnel.custom.start_command`, `health_url` | Arbitrary tunnel command |
| `none` | — | No tunnel, local only |

### SharedProcess Pattern

All tunnel implementations share a common `TunnelProcess` handle stored in `Arc<Mutex<Option<TunnelProcess>>>`:

```rust
pub(crate) struct TunnelProcess {
    pub child: tokio::process::Child,
    pub public_url: String,
}
pub(crate) type SharedProcess = Arc<Mutex<Option<TunnelProcess>>>;
```

This allows the factory-created tunnel instance to persist the child process reference across `start()` calls, and `kill_shared()` provides a unified stop mechanism (mod.rs:64-72).

### CloudflareTunnel Implementation

The most sophisticated tunnel implementation. Key challenge: extracting the public URL from cloudflared's stderr output, which contains multiple URLs including documentation links.

```rust
// cloudflare.rs:11-31 — filters out documentation URLs
fn extract_tunnel_url(line: &str) -> Option<String> {
    let candidate = line[idx..end];  // extracts URL after "https://"

    let is_tunnel_line = line.contains("Visit it at")
        || line.contains("Route at")
        || line.contains("Registered tunnel connection");
    let is_tunnel_domain = candidate.contains(".trycloudflare.com");
    let is_docs_url = candidate.contains("github.com")
        || candidate.contains("cloudflare.com/docs")
        || candidate.contains("developers.cloudflare.com");

    if is_tunnel_line || is_tunnel_domain || !is_docs_url {
        Some(candidate.to_string())
    } else {
        None
    }
}
```

The `start()` method:
1. Spawns `cloudflared tunnel run --token <token> --url http://localhost:<port>`
2. Reads stderr line-by-line with 5-second timeout per line, up to 30 seconds total
3. Parses URL using `extract_tunnel_url()` which distinguishes real tunnel URLs from warning/docs links
4. Returns the public URL on success

### TailscaleTunnel Implementation

Simpler — uses `tailscale serve` or `tailscale funnel` subcommand. On `start()`:
1. Queries `tailscale status --json` to get the DNS name if no hostname configured
2. Spawns `tailscale <serve|funnel> <port>`
3. Constructs URL as `https://{hostname}:{port}` without waiting for confirmation

On `stop()`: also runs `tailscale <serve|funnel> reset` to clean up the serve/funnel configuration (tailscale.rs:81-86).

### Health Check

All tunnel implementations use the same pattern: check if the child process is still running by testing `tp.child.id().is_some()` under the lock (mod.rs:122-124). The `public_url()` method uses `try_lock()` to avoid blocking on a sync call.

### Error Handling

Factory errors are descriptive:
```
"tunnel.provider = \"cloudflare\" but [tunnel.cloudflare] section is missing"
```

All implementations return `Result<String>` from `start()` with specific error messages for:
- Binary not found
- Token/auth invalid
- Timeout waiting for URL
- Process spawn failure

### Key Files Traced

| File | Purpose |
|------|---------|
| `src/tunnel/mod.rs` | Trait definition, factory, SharedProcess |
| `src/tunnel/cloudflare.rs` | cloudflared binary wrapper with URL extraction |
| `src/tunnel/tailscale.rs` | tailscale serve/funnel wrapper |
| `src/tunnel/ngrok.rs` | ngrok binary wrapper |
| `src/tunnel/openvpn.rs` | OpenVPN config-file-based tunnel |
| `src/tunnel/pinggy.rs` | Pinggy SSH tunnel |
| `src/tunnel/custom.rs` | Arbitrary command tunnel |

### Notable Design Decisions

1. **URL extraction from stderr** — cloudflared prints URLs to stderr, not stdout, and mixes in warning URLs that must be filtered
2. **try_lock() for public_url()** — allows checking URL without blocking when scheduler queries it
3. **Process handle persistence** — `SharedProcess` allows a single tunnel instance to track its child across async operations
4. **Graceful stop** — `stop()` kills the process, waits for it to exit, and resets any provider-specific state (Tailscale)

---

## Feature 12: Hardware Peripherals

### Overview

ZeroClaw's hardware system (`src/hardware/` + `src/peripherals/`) connects to physical devices (microcontrollers, SBCs, debug adapters) and exposes their capabilities as LLM-callable tools. The system is organized around:

- **`src/hardware/`** — Core framework: device registry, transport abstraction, protocol definition, GPIO tools, discovery, UF2 flashing, Aardvark support
- **`src/peripherals/`** — Board-specific implementations: Arduino, STM32 Nucleo, Raspberry Pi GPIO, Uno Q bridge

### Architecture Layers

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

### Device Registry

`DeviceRegistry` (`device.rs:187-553`) assigns stable session aliases to discovered hardware:

```rust
// device.rs:202-244 — alias assignment with counter per prefix
pub fn register(&mut self, board_name: &str, vid: Option<u16>, pid: Option<u16>,
                device_path: Option<String>, architecture: Option<String>) -> String {
    let prefix = alias_prefix(board_name);  // "pico", "arduino", "nucleo", etc.
    let counter = self.alias_counters.entry(prefix.clone()).or_insert(0);
    let alias = format!("{}{}", prefix, counter);
    *counter += 1;
    // ... create Device and store
}
```

Alias prefixes derived from board name:
| Prefix | Board patterns |
|--------|---------------|
| `pico` | raspberry-pi-pico, pico, pico-w |
| `arduino` | arduino-uno, arduino-mega |
| `esp` | esp32, esp32-s3 |
| `nucleo` | nucleo-*, stm32-* |
| `rpi` | rpi-*, raspberry-pi |
| `device` | (fallback) |

### Device Identification via VID

USB Vendor IDs map to device kinds (device.rs:81-93):

```rust
pub fn from_vid(vid: u16) -> Option<Self> {
    match vid {
        0x2e8a => Some(Self::Pico),       // Raspberry Pi Pico
        0x2341 => Some(Self::Arduino),    // Arduino
        0x10c4 => Some(Self::Esp32),      // ESP32 via CP2102
        0x0483 => Some(Self::Nucleo),     // STM32 Nucleo
        0x2b76 => Some(Self::Aardvark),  // Total Phase Aardvark
        _ => None,
    }
}
```

### Transport Trait

The `Transport` trait (`transport.rs:72-81`) decouples tools from wire protocols:

```rust
#[async_trait]
pub trait Transport: Send + Sync {
    async fn send(&self, cmd: &ZcCommand) -> Result<ZcResponse, TransportError>;
    fn kind(&self) -> TransportKind;
    fn is_connected(&self) -> bool;
}
```

`TransportKind` discriminates: `Serial`, `Swd`, `Uf2`, `Native`, `Aardvark`.

### ZeroClaw Serial JSON Protocol

The firmware contract (`protocol.rs`) defines newline-delimited JSON:

```
Host → Device:  {"cmd":"gpio_write","params":{"pin":25,"value":1}}\n
Device → Host:  {"ok":true,"data":{"pin":25,"value":1,"state":"HIGH"}}\n
```

Both `ZcCommand` and `ZcResponse` are simple structs with `cmd`/`params` and `ok`/`data`/`error` fields respectively.

### Device Discovery

`DeviceRegistry::discover()` (`device.rs:489-552`) performs USB enumeration:

1. Scans serial ports via `scan_serial_devices()`
2. For known VIDs: registers immediately
3. For unknown VID (0): runs `ping_handshake()` — sends `{"cmd":"ping"}` and waits for response. Only registers if ZeroClaw firmware responds. This prevents registering random USB-serial adapters.

### GPIO Tools

`gpio_write` and `gpio_read` tools (`gpio.rs`) use the registry to resolve device aliases:

```rust
// gpio.rs:108-121 — resolves device before dropping registry lock
let (device_alias, ctx) = {
    let registry = self.registry.read().await;
    match registry.resolve_gpio_device(&args) {
        Ok(resolved) => resolved,
        Err(msg) => return Err(ToolResult { success: false, error: Some(msg), .. }),
    }
}; // registry read guard dropped here before async I/O
```

Key pattern: the registry lock is **not held** during the async `transport.send()` call, preventing lock contention.

### Boot Process

`hardware::boot()` (`mod.rs:239-362`) initializes the hardware subsystem:

1. Discovers USB-serial devices and registers them
2. Pre-registers config-specified serial boards (lazy — port need not exist at startup)
3. Detects Pico in BOOTSEL mode (RPI-RP2 drive present) and warns
4. Registers Aardvark adapters with full I2C/SPI/GPIO capabilities
5. Loads `ToolRegistry` from discovered devices
6. Injects RPi self-discovery tools if on Linux with `peripheral-rpi` feature
7. Loads hardware context files from `~/.zeroclaw/hardware/` for LLM system prompt

### Hardware Context Files

The system prompt gets hardware context from `~/.zeroclaw/hardware/` (`mod.rs:124-180`):

```
~/.zeroclaw/hardware/HARDWARE.md          — global hardware description
~/.zeroclaw/hardware/devices/<alias>.md  — per-device profiles
~/.zeroclaw/hardware/skills/*.md          — hardware skills (sorted alphabetically)
```

These are concatenated and injected into the LLM's context so it knows how to use the connected devices.

### RPi Self-Discovery

On Linux with `peripheral-rpi` feature, the system introspects the device tree to detect Raspberry Pi model and adds built-in GPIO tools (`gpio_rpi_write`, `gpio_rpi_read`, `gpio_rpi_blink`, `rpi_system_info`) automatically (mod.rs:189-227).

### Aardvark Support

Total Phase Aardvark USB adapters are first-class citizens: registered during `boot()` with `AardvarkTransport` and full I2C/SPI/GPIO capabilities (mod.rs:296-329). The `aardvark_tools.rs` module provides I2C scan, read, write, and SPI transfer tools.

### Peripherals Module

`src/peripherals/` provides higher-level board-specific functionality:

| File | Purpose |
|------|---------|
| `traits.rs` | `Peripheral` trait — connect/disconnect/health_check/tools |
| `serial.rs` | `SerialPeripheral` — wraps serial transport with Peripheral trait |
| `arduino_flash.rs` | Flash Arduino firmware via arduino-cli |
| `arduino_upload.rs` | Upload sketches to Arduino |
| `nucleo_flash.rs` | Flash STM32 Nucleo firmware |
| `uno_q_bridge.rs` | Bridge GPIO for Arduino Uno Q |
| `uno_q_setup.rs` | Setup script for Uno Q bridge |
| `capabilities_tool.rs` | Query device capabilities over serial |
| `rpi.rs` | Native RPi GPIO peripheral (Linux only) |

### Key Design Patterns

1. **Alias-based device naming** — LLM never sees `/dev/ttyACM0`, only `pico0`, `arduino0`
2. **Lazy serial transport** — serial port opens on first use, not at registration
3. **Ping handshake for unknown VIDs** — prevents registering random USB-serial devices
4. **Registry lock not held across async I/O** — prevents deadlocks and lock contention
5. **Boot-time BOOTSEL detection** — warns user if Pico is waiting to be flashed
6. **Context files in system prompt** — LLM knows about connected hardware without explicit tool calls
7. **DeviceRuntime from DeviceKind** — determines execution environment (MicroPython, Arduino, Nucleus, etc.)

### Notable Technical Debt / Complexity

1. **Dual codebase**: `hardware/` (new registry-based) and `peripherals/` (older Peripheral trait) coexist, with some duplication of concern
2. **Feature-gated modules**: `discover` and `introspect` require `hardware` feature AND supported OS; RPi tools require `peripheral-rpi` AND Linux
3. **Ping handshake assumes ZeroClaw firmware** — generic devices without ZeroClaw firmware are rejected even if they're valid microcontrollers

---

## Cross-Feature Observations

### Security Integration

All three features integrate with the security subsystem:
- **Cron**: `SecurityPolicy` checks (`can_act()`, `is_rate_limited()`, `record_action()`, `allowed_commands`, `forbidden_path_argument()`) run before every shell command and agent job execution
- **Tunnel**: Tunnel process management doesn't have explicit security hooks (process is spawned by privileged binary)
- **Hardware**: Tools are gated behind the hardware feature and device registry; raw device paths are never exposed to LLM

### Testing Culture

All three features have extensive inline tests:
- **Cron**: 60+ tests covering schedule parsing, weekday normalization, job CRUD, retry logic, delivery, one-shot behavior, declarative sync, output truncation
- **Tunnel**: ~25 tests covering factory errors, URL extraction, health checks, process lifecycle
- **Hardware**: 40+ tests covering alias assignment, VID mapping, registry operations, GPIO tool execution, protocol serialization

### Error Handling Patterns

1. **Cron**: Errors return `Result` with context (`anyhow::Context`), security blocks return `(false, "blocked by...")` tuples
2. **Tunnel**: `bail!()` for early errors, `Result<String>` for `start()` (returns URL or error)
3. **Hardware**: `TransportError` enum with variants for timeout, disconnection, protocol errors; tools return `ToolResult` with `success: bool` + `error: Option<String>`

### Configuration Schema

All three features support declarative configuration:
- **Cron**: `[cron.jobs]` array in config.toml with `cron sync` reconciling to SQLite
- **Tunnel**: `[tunnel]` section with provider + provider-specific subsections
- **Hardware**: `[peripherals]` section with `[[peripherals.boards]]` entries

---

## Files Analyzed

### Cron & Scheduling
- `src/cron/scheduler.rs` — 51KB, execution runtime
- `src/cron/store.rs` — 60KB, SQLite persistence
- `src/cron/schedule.rs` — 11KB, cron parsing
- `src/cron/types.rs` — 7KB, domain types
- `src/cron/mod.rs` — 33KB, public API

### Tunnel & Remote Access
- `src/tunnel/mod.rs` — 15KB, trait + factory
- `src/tunnel/cloudflare.rs` — 7KB, cloudflared wrapper
- `src/tunnel/tailscale.rs` — 4KB, tailscale wrapper
- `src/tunnel/ngrok.rs` — 5KB, ngrok wrapper
- `src/tunnel/openvpn.rs` — 8KB, OpenVPN wrapper
- `src/tunnel/pinggy.rs` — 7KB, Pinggy wrapper
- `src/tunnel/custom.rs` — 7KB, custom command wrapper
- `src/tunnel/none.rs` — 1KB, no-op tunnel

### Hardware Peripherals
- `src/hardware/mod.rs` — 27KB, boot + discovery
- `src/hardware/device.rs` — 30KB, registry
- `src/hardware/transport.rs` — 5KB, transport trait
- `src/hardware/protocol.rs` — 5KB, serial JSON protocol
- `src/hardware/gpio.rs` — 22KB, GPIO tools
- `src/peripherals/mod.rs` — 14KB, peripheral factory
- `src/peripherals/traits.rs` — 75 lines, Peripheral trait
- `src/peripherals/serial.rs` — serial peripheral implementation
- `src/peripherals/arduino_flash.rs`, `nucleo_flash.rs` — firmware flashing
