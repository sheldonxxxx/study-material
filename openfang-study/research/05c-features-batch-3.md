# Features 7-9 Deep Dive: Desktop App, Skills System, WASM Sandbox

**Date:** 2026-03-26
**Features:** 7. Desktop Application (Tauri 2.0), 8. Skills System & Marketplace, 9. WASM Tool Sandbox

---

## Feature 7: Desktop Application (Tauri 2.0)

### Description
Native desktop app wrapping the OpenFang Agent OS kernel with system tray, notifications, global shortcuts, auto-start, and update checking. The TUI dashboard is available separately via CLI.

### Key Files Traced

| File | Purpose |
|------|---------|
| `crates/openfang-desktop/src/lib.rs` | Tauri app entry point, kernel/server boot |
| `crates/openfang-desktop/src/tray.rs` | System tray menu and event handling |
| `crates/openfang-desktop/src/commands.rs` | Tauri IPC command handlers |
| `crates/openfang-cli/src/tui/mod.rs` | Ratatui TUI with 16 tabs (separate binary) |

### Code Flow

**Boot Sequence (`lib.rs`):**
```
run() → server::start_server() → Tauri::Builder
  → setup() → WebviewWindowBuilder (points to embedded HTTP server)
           → tray::setup_tray()
           → spawn notification listener on kernel event bus
           → spawn startup update check
```

**Desktop Window:** Built programmatically in `setup()`, NOT via `tauri.conf.json`. The comment explains why: Tauri would load `index.html` from embedded assets (which don't exist), causing an `AssetNotFound` race condition. The window points directly at `http://127.0.0.1:{port}`.

**System Tray (`tray.rs`):**
- Menu items: Show Window, Open in Browser, Agents: N running, Status: Running (uptime), Launch at Login, Check for Updates, Open Config, Quit
- Left-click on tray icon: shows and focuses main window
- Menu actions handled via `on_menu_event` match statement

**Notification Forwarding:** The desktop app subscribes to the kernel event bus and forwards only critical events as native OS notifications:
- `LifecycleEvent::Crashed { agent_id, error }`
- `SystemEvent::KernelStopping`
- `SystemEvent::QuotaEnforced { agent_id, spent, limit }`
- Health checks and quota warnings are intentionally noisy and skipped.

**Single Instance:** Uses `tauri_plugin_single_instance` - if a second instance launches, it focuses the existing window instead of booting a second kernel.

**Auto-Start:** Uses `tauri_plugin_autostart` with `--minimized` args. Toggle via tray menu item or settings.

**Update Flow:** Check via `tauri_plugin_updater` → if available, show "Installing Update..." notification → download and install → app restarts automatically.

### TUI (CLI, separate from desktop)

The TUI is in `crates/openfang-cli/src/tui/` and is compiled into the CLI binary (`openfang`), not the desktop app. It uses Ratatui with 19 tabs:

```
Dashboard, Agents, Chat, Sessions, Workflows, Triggers, Memory,
Channels, Skills, Hands, Extensions, Templates, Peers, Comms,
Security, Audit, Usage, Settings, Logs
```

Two-phase navigation: `Phase::Boot` (Welcome/Wizard) → `Phase::Main` with tab switching.

### Clever Solutions

1. **Window creation workaround:** Building the WebviewWindow programmatically after server boot avoids the asset-not-found race condition that would occur if defined in `tauri.conf.json`.

2. **Notification filtering:** Only critical events become desktop notifications — the developers recognized that health checks and quota warnings would be too noisy for native notifications.

3. **Single instance as window focuser:** Rather than refusing to launch, the second instance politely brings the existing window to focus.

4. **Uptime formatting utility:** `format_uptime(secs)` converts elapsed seconds to `Xh Xm Xs` / `Xm Xs` / `Xs` automatically.

### Technical Debt / Shortcuts

1. **Two separate UIs:** Desktop app (WebView) and CLI (Ratatui TUI) are separate binaries, not one app with two modes. Maintenance burden is doubled.

2. **Tauri context in Rust:** Uses `include_bytes!("../icons/32x32.png")` for tray icon — embedded at compile time. If the icon file moves, this breaks silently (no build error, just missing icon at runtime).

3. **Blocking file pickers:** `blocking_pick_file()` in commands.rs blocks the Tauri command thread. For large file imports, this could freeze the UI.

4. **No window state persistence:** Window size/position is hardcoded in `setup()`. User cannot resize once and have it persist.

---

## Feature 8: Skills System & Marketplace

### Description
60 bundled skills with domain expertise reference injected at runtime. FangHub marketplace integration for browsing and installing community skills. SKILL.md parser for agent context enrichment.

### Key Files Traced

| File | Purpose |
|------|---------|
| `crates/openfang-skills/src/loader.rs` | Skill execution (Python/Node/Shell/PromptOnly) |
| `crates/openfang-skills/src/registry.rs` | Skill registry, bundling, loading |
| `crates/openfang-skills/src/clawhub.rs` | ClawHub marketplace API client |

### Code Flow

**Registry Loading (`registry.rs`):**
```
new(skills_dir)
  → load_bundled()     // compile-time embedded skills
  → load_all()         // user-installed from ~/.openfang/skills/
  → load_workspace_skills()  // per-workspace overrides
```

Bundled skills are parse at compile time via `bundled::bundled_skills()`. User skills auto-convert: if `SKILL.md` exists but no `skill.toml`, the registry runs `openclaw_compat::convert_skillmd()` and writes the new format.

**Skill Execution (`loader.rs`):**
```
execute_skill_tool(manifest, skill_dir, tool_name, input)
  → match manifest.runtime.runtime_type
      Python  → execute_python()
      Node    → execute_node()
      Shell   → execute_shell()
      Wasm    → RuntimeNotAvailable error
      Builtin → RuntimeNotAvailable error
      PromptOnly → returns note to use built-in tools
```

Each subprocess runtime (Python/Node/Shell):
1. Finds binary via `find_python()` / `find_node()` / `find_shell()`
2. Spawns with `stdin(Stdio::piped)`, `stdout(Stdio::piped)`, `stderr(Stdio::piped)`
3. **SECURITY: `cmd.env_clear()`** — strips ALL environment variables including API keys
4. Re-adds only `PATH`, `HOME`, platform essentials (`SYSTEMROOT`, `TEMP` on Windows), `PYTHONIOENCODING=utf-8` / `NODE_NO_WARNINGS=1`
5. Writes JSON payload `{"tool": tool_name, "input": input}` to stdin
6. Parses stdout as JSON result or treats as plain text if JSON parse fails

**ClawHub Client (`clawhub.rs`):**

API endpoints (verified against actual clawhub.ai API v1):
- `GET /api/v1/search?q=...&limit=N` → `ClawHubSearchResponse` (root key: `results`)
- `GET /api/v1/skills?limit=N&sort=trending` → `ClawHubBrowseResponse` (root key: `items`)
- `GET /api/v1/skills/{slug}` → detail with nested `skill`, `latestVersion`, `owner`
- `GET /api/v1/download?slug=...` → zip download
- `GET /api/v1/skills/{slug}/file?path=SKILL.md` → individual file

**Retry Logic:** 5 attempts with exponential backoff + jitter. Respects `Retry-After` header when present, capped at 30s.

**Install Pipeline (`install()`):**
1. Download zip, compute SHA256
2. Detect format: SKILL.md (frontmatter) vs zip vs package.json
3. Extract zip entries with `enclosed_name()` path safety check
4. Convert to OpenFang format
5. **Prompt injection scan** — blocks critical severity, warns on lower
6. Binary dependency check (`which` lookup)
7. Write `skill.toml` with `verified: false`

### Security: Prompt Injection Scanning

The registry scans ALL skill prompt content (even bundled) before loading. The `SkillVerifier::scan_prompt_content()` detects override/exfiltration patterns. Critical severity blocks the skill entirely. The comment notes that 341 malicious skills were found on ClawHub during development.

### Skills Freeze Mechanism

Registry can be `freeze()`d into "Stable mode" after initial boot. After freezing:
- `load_skill()` returns `SkillError::NotFound("Skill registry is frozen")`
- `load_workspace_skills()` also blocked
- `load_bundled()` still works (called before freeze)

This prevents runtime introduction of new skills while preserving bundled ones.

### Clever Solutions

1. **Env isolation by default:** Every skill subprocess gets `env_clear()` — API keys never leak to third-party skill code. Only explicitly needed vars (PATH, HOME) are re-added.

2. **Format auto-detection:** If a user drops a SKILL.md file into the skills directory, the registry auto-converts it to `skill.toml` + `prompt_context.md` on first load.

3. **Workspace skill overrides:** Skills in `workspace_skills_dir` shadow global/bundled skills with the same name. Same security scanning applies.

4. **Graceful stdout fallback:** If skill script output isn't valid JSON, wraps it as `{"result": stdout.trim()}` instead of erroring. Prevents skill bugs from crashing the agent.

5. **Retry with jitter:** Uses `SystemTime::now().duration_since(UNIX_EPOCH).subsec_nanos()` for deterministic-ish jitter without a random number generator dependency.

6. **URL-encoded search:** Custom `urlencoded()` function with RFC 3986 compliance (unreserved chars pass through, space → `+`).

### Technical Debt / Shortcuts

1. **WASM runtime not implemented:** `SkillRuntime::Wasm` and `SkillRuntime::Builtin` both return `RuntimeNotAvailable` errors. The WASM sandbox (Feature 9) exists but isn't wired into the skills system.

2. **Python/Node/Shell binary detection is naive:** `find_python()` tries `python3` then `python`, no version check. Skills requiring Python 3.8+ won't get a clear error if an older version is found.

3. **Skills run synchronously:** `execute_python()` etc. are `async` but spawn `tokio::process::Command` which is awaitable. However, the skill tool interface doesn't support streaming output — stdout is captured only after completion.

4. **No skill update checking:** The ClawHub client can install but doesn't check for updates to already-installed skills.

5. **Tool name collisions:** If two installed skills provide a tool with the same name, the last-loaded one wins (HashMap insert semantics). No conflict detection.

---

## Feature 9: WASM Tool Sandbox

### Description
Dual-metered WebAssembly execution environment using Wasmtime for running untrusted tool code with fuel metering (CPU instruction budget) and epoch interruption (wall-clock timeout).

### Key Files Traced

| File | Purpose |
|------|---------|
| `crates/openfang-runtime/src/sandbox.rs` | WASM sandbox engine (Wasmtime) |
| `crates/openfang-runtime/src/workspace_sandbox.rs` | Workspace filesystem sandboxing |
| `crates/openfang-runtime/src/host_functions.rs` | Host ABI implementations |

### Guest ABI

WASM modules must export:
- `memory` — linear memory
- `alloc(size: i32) -> i32` — allocate bytes, return pointer
- `execute(input_ptr: i32, input_len: i32) -> i64` — main entry, returns `(result_ptr << 32) | result_len`

### Host ABI

Imported from `"openfang"` module:
- `host_call(request_ptr, request_len) -> i64` — RPC dispatch
- `host_log(level, msg_ptr, msg_len)` — logging (no capability check)

### Code Flow

**Sandbox Creation (`sandbox.rs`):**
```
WasmSandbox::new()
  → Config::new()
  → config.consume_fuel(true)      // CPU instruction metering
  → config.epoch_interruption(true) // wall-clock timeout
  → Engine::new(&config)
```

**Execution (`execute()` async wrapper):**
```
spawn_blocking() → execute_sync()
  → Module::new(engine, wasm_bytes)   // compile (accepts .wasm or .wat)
  → Store::new(engine, GuestState)
  → store.set_fuel(fuel_limit)        // deterministic metering
  → store.set_epoch_deadline(1)
  → spawn watchdog thread (timeout_secs → engine.increment_epoch())
  → Linker::new(engine)
  → register_host_functions(linker)   // openfang::host_call, openfang::host_log
  → linker.instantiate(&mut store, &module)
  → get memory, alloc, execute exports
  → alloc(input_len) → write input_bytes to guest memory
  → execute_fn.call(input_ptr, input_len)
  → unpack i64 result: (ptr << 32) | len
  → read output from guest memory
  → return ExecutionResult { output, fuel_consumed }
```

**Dual Metering:**
1. **Fuel (deterministic):** Set via `store.set_fuel()`. Each WASM instruction costs 1 fuel unit. `Trap::OutOfFuel` caught explicitly.
2. **Epoch (wall-clock):** Set via `store.set_epoch_deadline(1)`. A watchdog thread calls `engine.increment_epoch()` after `timeout_secs`. `Trap::Interrupt` caught explicitly.

**Host Call Dispatch (`host_functions.rs`):**
```
dispatch(state, method, params) → match method
  time_now        → always allowed (no capability check)
  fs_read         → requires FileRead capability
  fs_write        → requires FileWrite capability
  fs_list         → requires FileRead capability
  net_fetch       → requires NetConnect capability + SSRF check
  shell_exec      → requires ShellExec capability
  env_read        → requires EnvRead capability
  kv_get          → requires MemoryRead capability
  kv_set          → requires MemoryWrite capability
  agent_send      → requires AgentMessage capability
  agent_spawn     → requires AgentSpawn capability
```

### Workspace Sandbox (`workspace_sandbox.rs`)

`resolve_sandbox_path(user_path, workspace_root)`:
1. Reject any `..` components outright
2. Join relative paths with workspace_root
3. Canonicalize the candidate (or parent for new files)
4. Verify canonical path starts with canonical workspace root

This prevents:
- `/tmp/../../etc/passwd` (relative traversal)
- `/etc/passwd` when workspace is `/home/user/project` (absolute escape)
- `symlink_to_outside/secret.txt` (symlink escape via canonicalize())

### SSRF Protection

`is_ssrf_target(url)` in `host_functions.rs`:
1. Block non-http/https schemes (`file://`, `gopher://`, `ftp://`)
2. Block hostname blocklist: `localhost`, `169.254.169.254`, `metadata.google.internal`, `metadata.aws.internal`, `instance-data`
3. DNSresolve the hostname and check every returned IP against private ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x)

This defeats DNS rebinding attacks because the check happens on the **resolved** address, not the hostname.

### Capability Matching

`capability_matches(granted, required)` — capabilities use glob/wildcard matching. `Capability::FileRead("*.txt")` matches `Capability::FileRead("data.txt")`. Allows granular grant-with-wildcard patterns.

### Clever Solutions

1. **Packed i64 return value:** Guest returns `(ptr << 32) | len` as a single i64. Avoids needing a second guest export for the result length.

2. **Watchdog thread with epoch:** Using `store.set_epoch_deadline(1)` with a separate thread calling `engine.increment_epoch()` is cleaner than a timer interrupt — Wasmtime handles the trap delivery atomically.

3. **Bump allocator in guest:** The echo WAT in tests uses a simple global `$bump` counter for `alloc`. No free — single-use only. Appropriate for sandboxed tool execution.

4. **Capability inheritance on spawn:** When `host_agent_spawn()` creates a child agent, it passes `state.capabilities` so the child cannot exceed the parent's capabilities. Comment explicitly notes this as a security feature.

5. **Belt-and-suspenders path validation:** `safe_resolve_parent()` checks `..` components AND validates the filename doesn't contain `..` after canonicalizing the parent. Two layers of traversal prevention.

### Technical Debt / Shortcuts

1. **Max memory not enforced:** `SandboxConfig::max_memory_bytes` is stored but never enforced. The comment says "reserved for future enforcement."

2. **No WASM → skill integration:** The WASM sandbox exists but `SkillRuntime::Wasm` in `loader.rs` returns `RuntimeNotAvailable`. The two features are not wired together.

3. **Blocking host calls in async context:** `host_shell_exec()` and `host_net_fetch()` use `std::process::Command::new()` synchronously, not `tokio::process::Command`. The `tokio_handle.block_on()` is only used for truly async operations like `net_fetch` (which does use reqwest).

4. **Shell exec uses `Command::new()` not `Command::new("sh")`:** This is actually safer (no shell injection), but it means compound commands like `ls | grep foo` won't work unless the skill explicitly calls `sh -c "ls | grep foo"`.

5. **`subprocess_sandbox.rs` is separate:** The skills system (Feature 8) uses `env_clear()` for environment isolation, but there's also a `subprocess_sandbox.rs` in the runtime crate. It's unclear if/when each is used vs the other.

---

## Cross-Feature Observations

### Integration Gaps

1. **WASM Sandbox not connected to Skills:** The WASM sandbox (`sandbox.rs`) exists with full capability checking, but `SkillRuntime::Wasm` in the skills loader returns `RuntimeNotAvailable`. The two systems were built independently and not integrated.

2. **Desktop app embeds HTTP server:** Rather than the CLI starting the server and the desktop app connecting to it as a client, the desktop app boots its own kernel + server. This is likely correct (single-instance enforcement) but means two code paths for startup.

3. **Skills env isolation vs subprocess sandbox:** `loader.rs` uses `env_clear()` for skills isolation, while `openfang-runtime/src/subprocess_sandbox.rs` exists separately. It's unclear if these are redundant, complementary, or one is legacy.

### Security Architecture

Both Features 8 and 9 have defense-in-depth security models:

**Skills (Feature 8):**
- `env_clear()` strips secrets before spawning third-party code
- Prompt injection scanning on load
- Binary dependency checking

**WASM Sandbox (Feature 9):**
- Dual metering (fuel + epoch)
- Capability-based permissions (deny-by-default)
- Path canonicalization with symlink resolution
- SSRF protection with DNS resolution verification
- Capability inheritance on child agent spawn

### Testing Coverage

Both features have inline tests:
- `loader.rs`: `test_find_python`, `test_find_node`, `test_prompt_only_execution`
- `sandbox.rs`: `test_sandbox_config_default`, `test_echo_module`, `test_fuel_exhaustion`, `test_host_call_capability_denied`, etc.
- `host_functions.rs`: Comprehensive tests for each host function and SSRF protection
- `workspace_sandbox.rs`: Tests for each traversal vector (relative, absolute, symlink, `..` components)
- `clawhub.rs`: Serde round-trip tests for each API response type

---

## Summary

| Aspect | Feature 7 (Desktop) | Feature 8 (Skills) | Feature 9 (WASM) |
|--------|---------------------|---------------------|------------------|
| **Maturity** | Solid, production | Solid, production | Built but not integrated |
| **Security Model** | Single-instance + autostart | env_clear() + prompt scan | Capability-based + metering |
| **Key Strength** | Clean boot/window separation | Format auto-conversion | Dual-metering with epoch |
| **Key Gap** | No window state persistence | WASM runtime unimplemented | Not wired into skills |
| **Lines (approx)** | ~300 (lib+tray+commands) | ~900 (loader+registry+clawhub) | ~900 (sandbox+workspace+hostfn) |
