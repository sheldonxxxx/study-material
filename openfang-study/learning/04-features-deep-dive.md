# OpenFang Feature Deep-Dive

**Project:** OpenFang (Rust Agent OS)
**Synthesized:** 2026-03-26
**Source:** Research batches 1-5 across 14 features
**Repo:** `/Users/sheldon/Documents/claw/reference/openfang`

---

## Introduction

OpenFang is a Rust-native agent runtime that positions itself as an "Agent OS" — a unified kernel that orchestrates LLM agents, multi-channel messaging, autonomous "Hands" workflows, memory management, and security sandboxing. The codebase spans 14 crates with 50+ channel adapters, 27+ LLM providers, 8 bundled autonomous Hands, and 16 independent security layers.

This document synthesizes the five batch deep-dives into a coherent narrative, ordered by feature priority (core first), with cross-feature integration gaps, most impressive implementations, and most concerning technical debt identified.

---

## Core Features

### Feature 1: Autonomous "Hands" System

**Priority:** Core | **Crate:** `crates/openfang-hands`

#### What It Is

A Hand is a pre-packaged autonomous agent that runs on a schedule without user prompting. Unlike chat agents (where you converse), Hands work in the background and deliver results (clips, leads, reports). Eight bundled Hands ship with complete pipelines: Clip (video), Lead (prospecting), Collector (OSINT), Predictor (forecasting), Researcher (reports), Twitter (social), Browser (web automation), and Trader.

#### Architecture

```
HandRegistry
  ├── bundled/ → compile-time embedded HAND.toml + SKILL.md
  ├── user-installed/ → ~/.openfang/hands/
  └── workspace/ → per-workspace overrides

HandDefinition (HAND.toml)
  ├── id, name, description, category
  ├── tools: Vec<String>
  ├── requires: Vec<HandRequirement> (binary/env/api_key)
  ├── settings: Vec<HandSetting> (select/text/toggle)
  └── agent: AgentManifest
```

#### Key Implementation Details

**Requirements Checking:** The registry performs cross-platform binary detection. Notably, Python detection (`check_python3_available()`) actually runs `python3 --version` and inspects stdout AND stderr for "Python 3" — this is specifically to avoid the Windows Store shim that just opens the Microsoft Store without actually running Python.

Chromium detection for the Browser Hand checks: env vars first, then PATH for common binary names, then well-known install paths, then Playwright's cache at `~/.cache/ms-playwright/chromium-*`.

**Dual TOML Format Support:** Hand definitions support both flat TOML and `[hand]` wrapper table formats. The parser tries flat first, falls back to wrapped.

**Issue #402 - Cron Reassignment:** When the kernel restarts, `load_state()` returns `(hand_id, config, old_agent_id)` so cron jobs can be reassigned to newly spawned agents with different PIDs.

**Settings Availability:** Each setting option can declare `provider_env` (an API key to check) and/or `binary` (a command to verify on PATH). The "Ready" badge in the dashboard only shows if the selected option's requirements are met.

#### Technical Debt

- No explicit Hand uninstall operation — only `upsert_from_content` for overwriting
- `bundled_hands()` hardcodes 8 `include_str!` entries — adding a new bundled hand requires code changes
- SKILL.md content embedded at compile time via `include_str!` — updates require recompilation

---

### Feature 2: Agent Runtime Engine

**Priority:** Core | **Crate:** `crates/openfang-runtime`

#### What It Is

The core agent execution loop. Receives a user message, recalls relevant memories via embeddings, builds the system prompt, calls the LLM in a retry loop with fallback chains, executes tool calls through a loop guard, and saves sessions. Handles 50+ tool types with taint checking, capability enforcement, and approval gates.

#### Architecture

```
run_agent_loop(manifest, user_message, session, memory, driver)
  │
  ├─ 1. Memory recall (vector embedding or text fallback)
  ├─ 2. System prompt build (base + recalled memories)
  ├─ 3. Add user message to session history
  ├─ 4. History emergency trim (if > 20 messages)
  │
  ├─ 5. FOR each iteration (max_iterations, default 50):
  │     │
  │     ├─ Context overflow recovery (7-phase pipeline)
  │     ├─ Context guard (compact oversized tool results)
  │     ├─ LLM call_with_retry():
  │     │     - Provider cooldown circuit breaker
  │     │     - Exponential backoff (rate limited / overloaded)
  │     │     - Up to 3 retries
  │     │     - ModelNotFound fallback chain
  │     │     - Error classification via llm_errors::classify_error()
  │     │
  │     ├─ Handle stop_reason:
  │     │     - EndTurn | StopSequence → extract text, save session
  │     │     - ToolUse → execute tools
  │     │     - MaxTokens → continue with "Please continue."
  │     │
  │     └─ Tool execution with loop guard, approval gates, 120s timeout
  │
  └─ 6. Return AgentLoopResult { response, total_usage, iterations, cost_usd }
```

#### Key Safeguard Systems

**Loop Guard (Circuit Breaker):** SHA256-based detection with four mechanisms:
1. Hash-Count: identical `(tool_name | serialized_params)` calls
2. Outcome-Aware: `(tool_call_hash | result_truncated)` pairs
3. Ping-Pong: sliding window of last 30 calls, detects A-B-A-B patterns
4. Poll Tool Handling: `shell_exec` with status/poll/watch gets 3x relaxed thresholds

Graduated responses: `Allow` → `Warn` → `Block` → `CircuitBreak`.

**Phantom Action Detection:** Detects when the LLM claims to have sent/posted/emailed without calling the corresponding tool. Simple keyword matching on "sent " + channel names. If detected, re-prompts with system guidance forcing real tool usage.

**Context Overflow Recovery:** Replaces the older `emergency_trim_messages`. Uses a 7-phase pipeline that re-validates `tool_call`/`tool_result` pairing after drains to prevent orphan entries.

**Interim Session Save:** After every tool execution, `memory.save_session_async()` is called fire-and-forget. Prevents data loss on crash.

**Dynamic Truncation:** Tool results are truncated context-aware, not flat `MAX_CHARS`. Uses `truncate_tool_result_dynamic(&result.content, &context_budget)`.

#### LLM Driver System

Factory pattern creating drivers for 27+ providers:

```rust
match provider {
    "anthropic" => AnthropicDriver::new(api_key, base_url),
    "gemini" | "google" => GeminiDriver::new(api_key, base_url),
    "openai" | "groq" | "deepseek" | ... => OpenAIDriver::new(api_key, base_url),
    "claude-code" => ClaudeCodeDriver::new(cli_path, skip_permissions),
    "qwen-code" => QwenCodeDriver::new(cli_path, skip_permissions),
    "github-copilot" | "copilot" => CopilotDriver::new(github_token, base_url),
    _ => { /* OpenAI-compatible fallback */ }
}
```

**ModelNotFound Fallback Chain:** When primary model returns `ModelNotFound`, tries fallback models in order before propagating. Creates fresh driver for each fallback.

#### Technical Debt

- `call_with_retry()` creates new fallback drivers per attempt — adds overhead but acceptable for infrequent fallback use
- No cancellation support — once LLM call starts, only timeout can interrupt
- Phantom action detection is naive keyword matching — could produce false positives/negatives
- MAX_ITERATIONS default of 50 may be wrong for some use cases (made configurable per agent)

---

### Feature 3: Multi-Channel Messaging (40+ Adapters)

**Priority:** Core | **Crate:** `crates/openfang-channels`

#### What It Is

50+ pluggable channel adapters that convert platform-specific messages into unified `ChannelMessage` events. Each adapter supports per-channel model overrides, DM/group policies, rate limiting, and output formatting. The bridge dispatches messages concurrently to the kernel.

#### Architecture

```
BridgeManager
  ├── start_adapter(adapter) → spawns async task
  │     ├── Per-message concurrency (Semaphore(32))
  │     └── Stream dispatch with shutdown coordination
  │
  └── dispatch_message(message)
        ├── Rate limit check
        ├── RBAC authorization
        ├── Agent resolution (by ID or routing rules)
        ├── LLM call via handle.send_message()
        ├── Response formatting
        └── Send via adapter + typing indicator loop
```

**Key insight:** Per-message concurrency is at the dispatch level, not per-agent. This prevents slow LLM calls from blocking the entire stream. Per-agent serialization is handled separately by the kernel's `agent_msg_locks`.

#### Discord Adapter (WebSocket Gateway Pattern)

- Connects to Discord's Gateway via WebSocket
- Exponential backoff: starts at 1s, doubles each failure, caps at 60s
- Handles: DISPATCH (events), HEARTBEAT, RECONNECT, INVALID_SESSION, HELLO
- Session resume supported for brief disconnects
- Message sending via REST API with >2000 char chunking

#### Rate Limiting

Per-channel rate limiter using `DashMap` for concurrent access:

```rust
ChannelRateLimiter::check(channel_type, platform_id, max_per_minute)
  → "channel:platform_id" key
  → retain() evicts entries older than 60s
  → if len >= max_per_minute → Err
```

#### Output Formatting

Each channel has a native format:
- Telegram: HTML (`<b>`, `<i>`, `<code>`)
- Slack: Mrkdwn
- WeCom: Plain text only (no formatting support — special-cased)
- Default: Markdown

#### Technical Debt

- 50+ adapter files but each is large boilerplate — adding new adapters requires significant copy-paste
- No adapter-level health monitoring — if adapter silently fails, only visible through missing messages
- WeCom special-casing leaks into dispatch logic
- Hardcoded 4-second typing refresh interval — works for Telegram (~5s expiry) but may not be optimal for all platforms

---

### Feature 4: 16-Layer Security System

**Priority:** Core | **Crates:** `openfang-runtime`, `openfang-types`, `openfang-extensions`

#### Overview

Defense-in-depth architecture with 16 independently testable security layers spanning:

| Layer | System | Implementation |
|-------|--------|----------------|
| 1 | WASM Dual-Metered Sandbox | Fuel metering + epoch interruption |
| 2 | Merkle Hash-Chain Audit Trail | SHA256 chaining with SQLite persistence |
| 3 | Information Flow Taint Tracking | Lattice-based propagation |
| 4 | Ed25519 Signed Agent Manifests | Cryptographic identity |
| 5 | SSRF Protection | Blocks private IPs, cloud metadata, DNS rebinding |
| 6 | Secret Zeroization | Zeroizing<String> auto-wipes |
| 7 | OFP Mutual Authentication | HMAC-SHA256 nonce-based P2P |
| 8 | Capability Gates | RBAC on agent tool access |
| 9 | Security Headers | CSP, X-Frame-Options, HSTS |
| 10 | Health Endpoint Redaction | Minimal public info |
| 11 | Subprocess Sandbox | env_clear() + selective passthrough |
| 12 | Prompt Injection Scanner | Override/exfiltration detection |
| 13 | Loop Guard | SHA256 circuit breaker |
| 14 | Session Repair | 7-phase message validation |
| 15 | Path Traversal Prevention | Canonicalization + symlink escape |
| 16 | GCRA Rate Limiter | Token bucket with per-IP tracking |

#### WASM Sandbox (Layers 1, 11)

**Dual Metering:**
1. **Fuel Metering:** Compiler-inserted instruction counting. `store.set_fuel(fuel_limit)` sets deterministic budget.
2. **Epoch Interruption:** Wall-clock timeout via watchdog thread calling `engine.increment_epoch()`.

```rust
let engine_clone = engine.clone();
let _watchdog = std::thread::spawn(move || {
    std::thread::sleep(Duration::from_secs(timeout_secs));
    engine_clone.increment_epoch();
});
```

Watchdog runs on a std thread (not tokio) so it is not affected by async runtime blocking.

**Host ABI with Capability Enforcement:**
- `host_call` — RPC dispatch with capability checks
- `host_log` — logging without capability checks
- Capabilities checked before every dispatch. Deny-by-default.

#### Merkle Hash-Chain Audit (Layer 2)

Append-only log with SHA-256 chaining:

```
entry[i].hash = SHA256(seq | timestamp | agent_id | action | detail | outcome | prev_hash)
```

- Genesis: 64 zero characters
- SQLite backend optional
- `verify_integrity()` recomputes all hashes from genesis

#### Taint Tracking (Layer 3)

Lattice-based propagation with 5 labels: `ExternalNetwork`, `UserInput`, `Pii`, `Secret`, `UntrustedAgent`.

Predefined sinks block specific labels:
- `shell_exec`: blocks ExternalNetwork, UntrustedAgent, UserInput
- `net_fetch`: blocks Secret, Pii
- `agent_message`: blocks Secret

Manual `declassify()` required — no automatic sanitization. Intentional but means misuse is possible.

#### Path Traversal Prevention (Layer 15)

`safe_resolve_parent()` in `workspace_sandbox.rs`:
1. Reject any `..` components outright
2. Join relative paths with workspace_root
3. Canonicalize the candidate (or parent for new files)
4. Verify canonical path starts with canonical workspace root

This prevents: relative traversal, absolute escape, and symlink escape via canonicalize().

#### Technical Debt

- **Watchdog thread spawn per execution:** In `sandbox.rs`, every `execute_sync()` spawns a new watchdog thread. If execution panics, thread is orphaned — harmless but wasteful.
- **Max memory not enforced:** `SandboxConfig::max_memory_bytes` stored but never enforced.
- **Guest alloc is bump allocator:** No free — appropriate for short-lived executions but could cause memory pressure.
- **Audit log growth unbounded:** No pruning or compaction strategy.
- **Taint declassification manual:** Caller must explicitly `declassify()` — easy to forget.

---

### Feature 5: Memory & Knowledge Management

**Priority:** Core | **Crate:** `crates/openfang-memory`

#### What It Is

SQLite-backed persistent memory with structured KV, semantic vector search, knowledge graph, and cross-channel session management. Unified behind `MemorySubstrate` compositing multiple stores.

#### Session Management

**Dual Session Model:**
1. **Regular Sessions:** Per-channel conversation history
2. **Canonical Sessions:** Cross-channel persistent context (one per agent)

```
User on Telegram → append_canonical() → stored in SQLite
User on Discord  → append_canonical() → loads canonical + appends
                → Agent remembers context from Telegram
```

**Compaction:** Threshold of 100 messages. On exceed: older messages summarized (truncated text summaries), recent 50 kept. Summary capped at 4000 chars.

**JSONL Mirror:** Best-effort human-readable export to `~/.openfang/sessions/{session_id}.jsonl`.

#### Knowledge Graph

**Entities and Relations:**
- `Entity`: id, entity_type, name, properties (JSON), timestamps
- `Relation`: source, relation_type, target, confidence (0-1), properties

**Pattern Query:** `GraphPattern { source, relation, target, max_depth }` — Note: `max_depth` field exists but is NOT used in SQL; always returns depth-1 relations.

#### Memory Substrate

```rust
struct MemorySubstrate {
    conn: Arc<Mutex<Connection>>,     // Shared SQLite
    structured: StructuredStore,       // KV pairs
    semantic: SemanticStore,         // Vector embeddings
    knowledge: KnowledgeStore,       // Knowledge graph
    sessions: SessionStore,          // Session management
    consolidation: ConsolidationEngine, // Memory decay
    usage: UsageStore,                // Token/cost tracking
}
```

**SQLite Pragmas:**
```rust
conn.execute_batch("PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;");
```
WAL mode: better concurrent read performance. busy_timeout: 5s wait on lock contention.

#### Technical Debt

- **`import()` returns stub:** `Memory::import()` returns "not yet implemented" error — data import explicitly deferred.
- **`max_depth` unused:** GraphPattern has `max_depth` field but SQL query ignores it.
- **No vector embedding storage visible:** Semantic store exists but embedding generation is in `openfang-runtime/src/embedding.rs`, not substrate.
- **ConsolidationEngine not wired:** Created but its API not visible in substrate public methods.

---

### Feature 6: REST API & WebSocket Server

**Priority:** Core | **Crate:** `crates/openfang-api`

#### What It Is

140+ REST/WS/SSE endpoints built on Axum 0.8. OpenAI-compatible `/v1/chat/completions` drop-in. Serves Alpine.js dashboard SPA. Extensive middleware stack.

#### Middleware Stack (applied in order)

1. Auth middleware (API key or password hash)
2. GCRA Rate limiter (per-IP token bucket)
3. Security headers (CSP, X-Frame-Options, HSTS, X-Content-Type-Options)
4. Request logging
5. Compression (gzip/brotli)
6. Tracing (tower-http TraceLayer)
7. CORS

**CORS Logic:**
- No auth + `api_key` empty: Permissive localhost origins
- Auth enabled: Restricted to localhost + configured origins
- Explicit comment noting `CorsLayer::permissive()` is dangerous

#### Server Boot Sequence

1. Boot kernel
2. Start background agents
3. Spawn config hot-reload watcher (polls every 30s)
4. Build router with all routes
5. Check for existing daemon (PID + health endpoint verification)
6. Write `daemon.json` with 0600 permissions
7. Start HTTP server with graceful shutdown

**Stale Daemon Detection:**
```rust
if is_process_alive(info.pid) && is_daemon_responding(&info.listen_addr) {
    return Err("Another daemon is already running");
}
// Only removes stale file if process is dead OR not responding
```

#### OpenAI Compatibility

**Agent Resolution:**
1. `openfang:{name}` → registry.find_by_name
2. Valid UUID → registry.get
3. Plain string → registry.find_by_name (fallback)

**Streaming Response:** SSE with `text/event-stream`. Tool calls streamed as `{index, id, type, function{name, arguments}}` chunks. `ContentComplete` with `StopReason::ToolUse` does NOT finish — waits for tool result.

**Missing Features:**
- No `functions` tool calling mode (only `function_call` / `function` parts)
- No `stop` parameter support
- No `presence_penalty` / `frequency_penalty`
- `max_tokens` accepted but may not propagate to all drivers

#### Technical Debt

- GCRA rate limiter on `Arc<GcraLimiter>` — shared globally, no per-tenant or per-endpoint granularity visible
- Import not implemented in memory substrate
- No `functions` tool calling limits OpenAI compat

---

### Feature 7: Desktop Application (Tauri 2.0)

**Priority:** Core | **Crate:** `crates/openfang-desktop`

#### What It Is

Native desktop app wrapping the kernel with system tray, notifications, global shortcuts, auto-start, and update checking. TUI dashboard available separately via CLI.

#### Architecture

**Boot Sequence:**
```
run() → server::start_server() → Tauri::Builder
  → setup() → WebviewWindowBuilder (points to embedded HTTP server)
           → tray::setup_tray()
           → spawn notification listener on kernel event bus
           → spawn startup update check
```

**Window Creation Workaround:** The WebviewWindow is built programmatically in `setup()`, NOT via `tauri.conf.json`. This avoids an `AssetNotFound` race condition that would occur if Tauri loaded `index.html` from embedded assets (which don't exist) before the HTTP server started listening.

**System Tray Menu:**
Show Window, Open in Browser, Agents: N running, Status: Running (uptime), Launch at Login, Check for Updates, Open Config, Quit

**Notification Filtering:** Only critical events become native OS notifications:
- `LifecycleEvent::Crashed { agent_id, error }`
- `SystemEvent::KernelStopping`
- `SystemEvent::QuotaEnforced { agent_id, spent, limit }`

Health checks and quota warnings are intentionally skipped — too noisy for native notifications.

**Single Instance:** Uses `tauri_plugin_single_instance` — if second instance launches, it focuses existing window instead of booting second kernel.

#### TUI (Separate Binary)

Located at `crates/openfang-cli/src/tui/` — compiled into the CLI (`openfang`), not the desktop app. Uses Ratatui with 19 tabs:
```
Dashboard, Agents, Chat, Sessions, Workflows, Triggers, Memory,
Channels, Skills, Hands, Extensions, Templates, Peers, Comms,
Security, Audit, Usage, Settings, Logs
```

#### Technical Debt

- **Two separate UIs:** Desktop app (WebView) and CLI (Ratatui TUI) are separate binaries — maintenance burden doubled
- **Icon embedded via include_bytes!:** `include_bytes!("../icons/32x32.png")` — if icon file moves, breaks silently (no build error)
- **Blocking file pickers:** `blocking_pick_file()` blocks the Tauri command thread — could freeze UI on large imports
- **No window state persistence:** Size/position hardcoded in `setup()`, not persisted across sessions

---

## Secondary Features

### Feature 8: Skills System & Marketplace

**Priority:** Secondary | **Crate:** `crates/openfang-skills`

#### What It Is

60 bundled skills (domain expertise reference injected at runtime) with FangHub marketplace integration. SKILL.md parser for agent context enrichment.

#### Skill Execution

```rust
execute_skill_tool(manifest, skill_dir, tool_name, input)
  → match manifest.runtime.runtime_type
      Python  → execute_python()
      Node    → execute_node()
      Shell   → execute_shell()
      Wasm    → RuntimeNotAvailable error
      Builtin → RuntimeNotAvailable error
      PromptOnly → returns note to use built-in tools
```

**Security - Env Isolation:**
```rust
cmd.env_clear();  // Strip ALL environment variables first
// Then re-add only: PATH, HOME, platform essentials
```

**Prompt Injection Scanning:** ALL skill prompt content scanned before loading. Critical severity blocks the skill entirely. Comment notes 341 malicious skills found on ClawHub during development.

#### ClawHub Marketplace

API endpoints verified against actual `clawhub.ai` API v1:
- `GET /api/v1/search` → `results[]`
- `GET /api/v1/skills?sort=trending` → `items[]`
- `GET /api/v1/skills/{slug}` → nested `skill`, `latestVersion`, `owner`
- `GET /api/v1/download` → zip download

**Retry Logic:** 5 attempts with exponential backoff + jitter. Respects `Retry-After` header, capped at 30s.

#### Technical Debt

- **WASM runtime not implemented:** `SkillRuntime::Wasm` and `Builtin` both return `RuntimeNotAvailable` — the WASM sandbox (Feature 9) exists but isn't wired in
- **Python/Node binary detection is naive:** `find_python()` tries `python3` then `python`, no version check — skills requiring Python 3.8+ won't get clear error
- **No skill update checking:** Can install but doesn't check for updates to installed skills
- **Tool name collisions:** Last-loaded wins on HashMap insert — no conflict detection

---

### Feature 9: WASM Tool Sandbox

**Priority:** Secondary | **Crate:** `crates/openfang-runtime`

#### What It Is

Dual-metered WebAssembly execution environment using Wasmtime. Fuel metering (CPU instruction budget) and epoch interruption (wall-clock timeout) for running untrusted tool code.

#### Guest ABI

WASM modules must export:
- `memory` — linear memory
- `alloc(size: i32) -> i32` — allocate bytes
- `execute(input_ptr, input_len) -> i64` — returns `(ptr << 32) | len`

#### Host ABI (imported from `"openfang"`)

- `host_call(request_ptr, request_len) -> i64` — RPC dispatch with capability checks
- `host_log(level, msg_ptr, msg_len)` — logging without capability checks

#### SSRF Protection

`is_ssrf_target(url)` in `host_functions.rs`:
1. Block non-http/https schemes (`file://`, `gopher://`, `ftp://`)
2. Block hostname blocklist: `localhost`, `169.254.169.254`, `metadata.google.internal`, etc.
3. DNS-resolve hostname and check every returned IP against private ranges

**Critical:** Check happens on **resolved** address, not hostname — defeats DNS rebinding attacks.

#### Capability Matching

`capability_matches(granted, required)` — glob/wildcard matching:
```rust
Capability::FileRead("*.txt") matches Capability::FileRead("data.txt")
```

#### Technical Debt

- **Max memory not enforced:** `max_memory_bytes` stored but never enforced — "reserved for future enforcement"
- **Not wired to Skills:** WASM sandbox exists but `SkillRuntime::Wasm` returns `RuntimeNotAvailable` — two systems built independently, not integrated
- **Shell exec uses `Command::new()` not `Command::new("sh")`:** Safer (no shell injection) but compound commands like `ls | grep foo` require explicit `sh -c`

---

### Feature 10: MCP (Model Context Protocol) & A2A (Agent-to-Agent)

**Priority:** Secondary | **Crate:** `crates/openfang-runtime`

#### MCP Client (Connecting to External MCP Servers)

**Dual Transport Support:**
```rust
pub enum McpTransport {
    Stdio { command: String, args: Vec<String> },  // Subprocess
    Sse { url: String },                           // HTTP POST/response
}
```

**Tool Namespacing:** `mcp_{server}_{tool}` format (e.g., `mcp_github_create_issue`). Original names preserved in `original_names: HashMap<String, String>` for servers that expect hyphenated names.

**SSRF Protection for SSE:** Explicitly blocks metadata URLs (`169.254.169.254`, `metadata.google`).

**Subprocess Sandbox (Security Layer 11):**
```rust
cmd.env_clear();  // Clear ALL env vars
// Re-add only whitelisted vars
```

#### MCP Server (Exposing OpenFang Tools via MCP)

**Status: Stub.** `handle_mcp_request()` validates requests but tool execution returns: `"Tool '{tool_name}' is available. Execution must be wired by the host."`

#### A2A Protocol (Agent-to-Agent)

**AgentCard structure:**
```rust
pub struct AgentCard {
    name, description, url, version,
    capabilities: AgentCapabilities,
    skills: Vec<AgentSkill>,  // Maps tools to A2A skills
}
```

**A2aTaskStore - Bounded Task Registry:** FIFO eviction when at capacity. Evicts oldest completed/failed/cancelled tasks.

**Untagged Enum for Dual Serialization:**
```rust
#[serde(untagged)]
pub enum A2aTaskStatusWrapper {
    Object { state: A2aTaskStatus, message: Option<serde_json::Value> },
    Enum(A2aTaskStatus),
}
```
Allows both `"completed"` and `{"state": "completed", "message": null}` forms.

#### Technical Debt

- **MCP server tool execution stubbed:** `mcp_server.rs` tool handler is unimplemented
- **No reconnection logic for MCP stdio:** If subprocess dies, no automatic restart
- **SSE transport is basic:** Assumes single POST/response cycle; MCP spec may support streaming

---

### Feature 11: P2P Wire Protocol (OFP)

**Priority:** Secondary | **Crate:** `crates/openfang-wire`

#### What It Is

OpenFang Protocol for peer-to-peer networking with HMAC-SHA256 mutual authentication.

#### Protocol Design

**Message Framing:** 4-byte big-endian length + JSON body
```rust
let json = serde_json::to_vec(msg)?;
let len = json.len() as u32;
let mut bytes = Vec::with_capacity(4 + json.len());
bytes.extend_from_slice(&len.to_be_bytes());
bytes.extend_from_slice(&json);
```

#### HMAC-SHA256 Mutual Authentication

**Handshake Flow:**
```
Client                              Server
  |--- Handshake (nonce_a, HMAC) ---->|
  |    auth_hmac = HMAC(secret, nonce_a + node_id_a)
  |<-- HandshakeAck (nonce_b, HMAC) --|
  |    auth_hmac = HMAC(secret, nonce_b + node_id_b)
  |====== Authenticated Loop =========|
  session_key = HMAC(secret, nonce_a + nonce_b)
```

**Nonce Replay Protection:**
```rust
pub struct NonceTracker {
    seen: Arc<DashMap<String, Instant>>,  // nonce -> timestamp
    window: Duration,                      // 5 minutes
}

pub fn check_and_record(&self, nonce: &str) -> Result<(), String> {
    self.seen.retain(|_, ts| now.duration_since(*ts) < self.window);  // GC old
    if self.seen.contains_key(nonce) {
        return Err("Nonce replay detected".into());
    }
    self.seen.insert(nonce.to_string(), now);
    Ok(())
}
```

#### Per-Message HMAC

After handshake, all messages use session_key HMAC appended after the JSON body:
```rust
// Format: [4-byte length][JSON body][64-hex HMAC]
```

#### Technical Debt

- **`consolidation.rs` listed in feature index but doesn't exist** — either planned but not implemented, or misnamed
- **No automatic peer rediscovery** — if peers go down, no automatic reconnect shown
- **Session key stored but unused:** `session_key: Mutex<Option<String>>` marked `#[allow(dead_code)]` after initial derivation
- **Broadcast notifications open new connections** — each broadcast creates fresh TCP connection per peer, rather than reusing established connections

---

### Feature 12: Credential Vault & OAuth2 PKCE

**Priority:** Secondary | **Crate:** `crates/openfang-extensions`

#### Credential Vault

**On-Disk Format:**
```
[4-byte "OFV1" magic][JSON body]
```

```rust
struct VaultFile {
    version: u8,        // 1
    salt: String,       // base64, 16 bytes, for Argon2
    nonce: String,      // base64, 12 bytes, for AES-GCM
    ciphertext: String, // base64, encrypted
}
```

**Key Derivation:**
- Master key: 32 bytes from OS keyring or `OPENFANG_VAULT_KEY` env var
- Derived key: Argon2id(master_key, salt) → 32 bytes for AES-256-GCM
- Each save: Fresh random salt + nonce

**Memory Safety:** `HashMap<String, Zeroizing<String>>` — auto-zeroized on drop.

**OS Keyring — File-Based Fallback:**
```rust
// XOR obfuscation with SHA256 mask
let machine_id = machine_fingerprint();  // USER + HOSTNAME + "openfang-vault-v1"
let mask = hasher.finalize().to_vec();
let obfuscated: Vec<u8> = key_bytes.iter()
    .enumerate()
    .map(|(i, b)| b ^ mask[i % mask.len()])
    .collect();
```

**Technical Debt:** XOR obfuscation with SHA256 mask is reversible with machine fingerprint knowledge. Acknowledged as "better than plaintext env vars" but not true keyring security.

#### Credential Resolver

Priority chain:
1. Vault (if unlocked)
2. Dotenv file (`~/.openfang/.env`)
3. Environment variable
4. Interactive prompt (CLI only)

#### OAuth2 PKCE

**Flow:** S256 method, localhost callback on random port.

```rust
fn generate_pkce() -> PkcePair {
    let verifier = Zeroizing::new(base64_url_encode(&OsRng::fill_bytes(&mut bytes)));
    let challenge = base64_url_encode(&Sha256::finalize(...));  // SHA256(verifier)
    PkcePair { verifier, challenge }
}
```

**Localhost Callback Server:** 5-minute timeout, state parameter for CSRF protection, auto-closes browser tab on success.

#### Technical Debt

- **Keyring is file-based XOR, not true OS keyring** — acknowledged in comments
- **No token refresh logic** — OAuth flow returns tokens but no automatic refresh
- **Default client IDs are obvious placeholders** — "openfang-*-client-id" values

---

### Feature 13: Migration Engine

**Priority:** Secondary | **Crate:** `crates/openfang-migrate`

#### What It Is

One-command migration from OpenClaw with automatic import of agents, memory, skills, sessions, channel configs, and credentials.

#### Supported Migration Items

| Item | Source | Target | Notes |
|------|--------|--------|-------|
| Config | `openclaw.json` / `config.yaml` | `config.toml` | TOML serialization |
| Agents | `agents.list[]` / `agents/{name}/agent.yaml` | `agents/{name}/agent.toml` | Full manifest conversion |
| Memory | `memory/{agent}/MEMORY.md` | `agents/{agent}/imported_memory.md` | Both modern and legacy layouts |
| Workspaces | `workspaces/{agent}/` | `agents/{agent}/workspace/` | Directory tree copy |
| Sessions | `sessions/*.jsonl` | `imported_sessions/*.jsonl` | JSONL conversation logs |
| Channel Tokens | `channels.{name}.*` | `secrets.env` | Bot tokens with 0o600 perms |
| WhatsApp Creds | `channels.whatsapp.auth_dir/` | `credentials/whatsapp/` | Baileys auth dir copy |

**OpenClaw Home Detection:** Checks `OPENCLAW_STATE_DIR`, `~/.openclaw`, `~/.clawdbot`, `~/.moldbot`, `~/.moltbot`, Windows APPDATA.

**Provider Mapping:** Maps 20+ providers (anthropic, openai, groq, ollama, openrouter, deepseek, together, mistral, fireworks, google, gemini, qwen, dashscope, moonshot, kimi, minimax, zhipu, glm, qianfan, baidu, xai, grok, cerebras, sambanova, perplexity, cohere, ai21, huggingface, replicate, github-copilot, copilot, vllm, lmstudio) to canonical names.

#### Skipped Items

| Item | Reason |
|------|--------|
| `auth-profiles.json` | Security: credentials must be env vars |
| `memory-search/index.db` | SQLite vector index not portable |
| `cron-store.json` | Cron run state not portable |
| `hooks` | Different event system in OpenFang |
| iMessage channel | macOS-only |
| BlueBubbles channel | No OpenFang adapter |
| Skills | Must be reinstalled via `openfang skill install` |
| LangChain / AutoGPT | Not yet implemented |

#### Key Technical Decisions

- **Dry-run by default for secrets:** Warns about credentials, gives users chance to review
- **Content-hash change detection:** `write_if_changed()` uses FNV hash to avoid unnecessary file writes and npm reinstalls
- **Dual-layout memory support:** Checks both `memory/{agent}/MEMORY.md` and `agents/{agent}/MEMORY.md`
- **Tool compatibility delegation:** `is_known_openfang_tool()` and `map_tool_name()` live in `openfang_types::tool_compat` — migration and kernel share identical definitions

#### Technical Debt

- **WhatsApp re-auth warning:** Baileys auth_dir copied but re-authentication may be needed
- **Silent tool skipping:** Unknown tools silently skipped with warnings rather than failing
- **Agent ID as name fallback:** If `name` field missing, uses `id` as agent name

---

### Feature 14: WhatsApp Web Gateway

**Priority:** Secondary | **Location:** `packages/whatsapp-gateway/` (Node.js)

#### What It Is

QR-code-based WhatsApp connection (like WhatsApp Web) without Meta Business account. Implemented as standalone Node.js service managed by the Rust kernel.

#### Architecture

```
OpenFang Kernel (Rust)
    │
    ├─ start_whatsapp_gateway()
    │   ├─ node_available() → checks Node.js >= 18
    │   ├─ ensure_gateway_installed()
    │   │   ├─ write_if_changed(index.js) [from include_str!]
    │   │   ├─ npm install --production
    │   └─ spawn node index.js as child process
    │       ├─ WHATSAPP_GATEWAY_PORT=3009
    │       └─ OPENFANG_URL=http://127.0.0.1:4200
    │
    └─ crash monitoring loop (5s/10s/20s backoff, max 3 restarts)

packages/whatsapp-gateway/index.js (Node.js)
    │
    ├─ Baileys socket (makeWASocket)
    │   ├─ useMultiFileAuthState('./auth_store')
    │   └─ printQRInTerminal: true
    │
    ├─ Connection state machine
    │   ├─ QR code → data:image/png;base64 via qrcode
    │   ├─ connected → clear QR, track session
    │   └─ disconnected → delete auth_store + stop OR exponential backoff reconnect
    │
    └─ HTTP server
        ├─ POST /login/start
        ├─ GET /login/status
        ├─ POST /message/send
        └─ GET /health
```

**Auth State Persistence:** Baileys stores in `./auth_store/`: `creds.json`, `app-state-sync.json`, `pre-keys.json`. Checks for existing `creds.json` on startup — auto-connects if found, avoids re-scan.

#### Key Design Patterns

1. **QR as Data URL:** Converted to `data:image/png;base64,...` for HTTP polling — avoids WebSocket complexity
2. **Kernel-embedded JS:** Gateway JS embedded at compile time via `include_str!` — travels with binary
3. **Content-hash updates:** `write_if_changed()` with FNV hash avoids unnecessary `npm install`
4. **PID tracking:** Kernel stores gateway PID in `Arc<Mutex<Option<u32>>>` for clean shutdown
5. **Graceful shutdown:** SIGINT/SIGTERM handlers call `sock.end()` before exit

#### Technical Debt

- **Text-only messages:** Only forwards text content — media files not uploaded or forwarded
- **Single agent routing:** All WhatsApp messages go to `DEFAULT_AGENT` — no per-sender routing
- **No message threading:** Simple JID construction, no conversation threading awareness
- **HTTP polling for QR:** No WebSocket or SSE for real-time QR delivery
- **Node.js as external process:** Rust kernel shells out to Node.js — adds process management complexity, not a true Rust-native solution

---

## Integration Issues Found

### 1. WASM Sandbox Not Connected to Skills

The WASM sandbox (`Feature 9`) exists with full capability checking, dual metering, and SSRF protection. However, `SkillRuntime::Wasm` in the skills loader (`Feature 8`) returns `RuntimeNotAvailable`. The two security systems were built independently and not integrated.

**Impact:** Third-party skills cannot use the WASM sandbox for isolation. The WASM sandbox is available for direct tool execution but not for skill runtime.

### 2. Desktop App vs CLI Startup Paths

The desktop app (`Feature 7`) boots its own kernel + HTTP server internally. The CLI TUI connects to a running server. These are separate code paths for startup — desktop doesn't connect to an existing CLI-started server.

**Impact:** Two different startup paths to maintain. However, single-instance enforcement via `tauri_plugin_single_instance` means desktop focuses existing window if server is already running.

### 3. Skills Env Isolation vs Subprocess Sandbox

The skills system (`Feature 8`) uses `env_clear()` for environment isolation. The runtime crate has a separate `subprocess_sandbox.rs` file. It's unclear if these are redundant, complementary, or one is legacy.

**Impact:** Potential confusion about which isolation mechanism to use. Possible code duplication.

### 4. Memory Import Not Implemented

`MemorySubstrate::import()` returns a stub error with "not yet implemented." Data import is explicitly deferred.

**Impact:** Users cannot migrate memory data from external sources.

### 5. Knowledge Graph Max_Depth Unused

`GraphPattern` has a `max_depth` field but the SQL query hardcodes `LIMIT 100` and always returns depth-1 relations only.

**Impact:** The API suggests configurable traversal depth but it doesn't actually work.

### 6. OFP Consolidation Not Implemented

`crates/openfang-wire/consolidation.rs` is listed in the feature index but doesn't exist as a file. Either planned but not implemented, or misnamed in documentation.

**Impact:** P2P network consolidation features are not available.

### 7. MCP Server Tool Execution Stubbed

`mcp_server.rs` tool handler validates requests but explicitly returns that execution "must be wired by the host." The MCP server can advertise tools but cannot execute them.

**Impact:** External MCP clients (Claude Desktop, VS Code) can discover OpenFang tools but cannot actually call them.

### 8. WhatsApp Bridging Separation

The migration engine (`Feature 13`) handles WhatsApp token/secrets migration, while the gateway (`Feature 14`) handles actual WhatsApp connectivity. These are complementary but separate systems with no visible integration.

**Impact:** WhatsApp setup requires understanding two different subsystems.

---

## Most Impressive Implementations

### 1. Dual-Metered WASM Sandbox

The combination of fuel metering (deterministic instruction counting) with epoch interruption (wall-clock watchdog thread) is sophisticated. Using `store.set_epoch_deadline(1)` with a separate thread calling `engine.increment_epoch()` is elegant — Wasmtime handles trap delivery atomically. The capability inheritance on child agent spawn (`host_agent_spawn()` passes `state.capabilities`) is a well-thought security feature.

### 2. Path Traversal Prevention

The `safe_resolve_parent()` function in `workspace_sandbox.rs` handles all three attack vectors: relative traversal (`..`), absolute escape, and symlink escape. The two-stage check (reject `..` outright, then canonicalize and verify prefix) is belt-and-suspenders done right.

### 3. HMAC-SHA256 OFP Mutual Authentication

The nonce-based mutual authentication with DashMap for concurrent nonce tracking, 5-minute sliding window for replay protection, per-message HMAC after handshake, and constant-time HMAC comparison via `subtle::ConstantTimeEq` is a complete and correct cryptographic protocol implementation.

### 4. Loop Guard with Multiple Detection Mechanisms

Four distinct detection mechanisms (hash-count, outcome-aware, ping-pong, poll-relaxed) with graduated responses (Allow → Warn → Block → CircuitBreak) and a warning bucket to prevent warning spam. The implementation is thorough and production-minded.

### 5. Migration Engine Content-Hash Updates

Using FNV hash in `write_if_changed()` to avoid unnecessary file writes and npm reinstalls is efficient. The dual-layout memory support (checking both modern and legacy paths) and comprehensive channel/token mapping (20+ providers, 13 channels) shows attention to real-world migration complexity.

### 6. Discord WebSocket Gateway with Session Resume

Full Discord Gateway protocol implementation with exponential backoff, heartbeat, session resume for brief disconnects, message chunking for >2000 chars, and proper opcode handling. The implementation handles edge cases like INVALID_SESSION and RECONNECT.

### 7. A2A Untagged Enum Handling

The `A2aTaskStatusWrapper` with `#[serde(untagged)]` allowing both `"completed"` and `{"state": "completed", "message": null}` forms is a pragmatic solution to API inconsistency. The bounded A2aTaskStore with FIFO eviction is also well-designed.

### 8. WhatsApp QR as Data URL

Converting QR codes to `data:image/png;base64,...` format for HTTP polling is a simple but effective solution that avoids WebSocket complexity while still enabling real-time QR delivery to clients.

---

## Most Concerning Technical Debt

### 1. Audit Log Growth Unbounded

The Merkle hash-chain audit trail (`audit.rs`) grows indefinitely with no pruning or compaction strategy. Every agent action, tool call, file access, and network request is logged. In a production system with heavy usage, this will eventually consume significant disk space.

**Severity:** Medium — will cause operational issues over time.

### 2. WASM Sandbox Max Memory Not Enforced

`SandboxConfig::max_memory_bytes` is stored in the config struct but is never actually enforced. The comment says "reserved for future enforcement." A malicious or buggy WASM module could consume unbounded memory.

**Severity:** Medium — security feature incomplete.

### 3. Vault OS Keyring is File-Based XOR

The credential vault acknowledges it uses file-based XOR obfuscation rather than true OS keyring (like macOS Keychain or Windows Credential Manager). The code comments explicitly note this. The XOR with SHA256 mask is reversible with machine fingerprint knowledge.

**Severity:** Medium — better than plaintext but not true keyring security.

### 4. Two Separate UI Implementations

Desktop app (WebView via Tauri) and CLI TUI (Ratatui) are separate binaries with separate codebases. This doubles the maintenance burden and could lead to feature drift between the two interfaces.

**Severity:** Low-Medium — maintainability concern.

### 5. WhatsApp Text-Only Messages

The WhatsApp gateway only forwards text content. Images, videos, stickers, and documents are detected but not uploaded or forwarded to OpenFang. Media-rich conversations will be degraded.

**Severity:** Low — feature incomplete but gateway still functional.

### 6. Taint Declassification Manual

The taint tracking system requires callers to explicitly `declassify()` before passing tainted data to sensitive sinks. There is no automatic sanitization. A careless caller can forget to sanitize, potentially causing security issues.

**Severity:** Low-Medium — relies on correct usage, easy to misuse.

### 7. No Skill Update Checking

The ClawHub marketplace client can install skills but never checks for updates to already-installed skills. Users could be running outdated skills with known vulnerabilities.

**Severity:** Low — operational security gap.

### 8. MCP Server Tool Execution Stubbed

The MCP server (`mcp_server.rs`) advertises tools to external clients but cannot execute them. The handler explicitly returns "Execution must be wired by the host." This means MCP protocol support is incomplete.

**Severity:** Medium — MCP integration is half-implemented.

### 9. Knowledge Graph Max_Depth Unused

The API suggests configurable relation traversal depth but the SQL always returns depth-1 only. The `max_depth` field exists in `GraphPattern` but is ignored.

**Severity:** Low — API mismatch, not a runtime bug.

### 10. OFP Session Key Unused After Derivation

The `session_key: Mutex<Option<String>>` in `PeerNode` is derived during handshake but then marked `#[allow(dead_code)]` and never used for subsequent message authentication.

**Severity:** Low — security mechanism incomplete.

---

## Summary

OpenFang is an ambitious Rust-native agent runtime with a sophisticated multi-layer security architecture, comprehensive multi-channel messaging, and well-designed autonomous agent workflows. The codebase shows thoughtful engineering in areas like the WASM sandbox, P2P protocol, and migration engine.

The most significant integration gaps are between the WASM sandbox and skills system, the MCP server's stubbed tool execution, and the separate WhatsApp migration/gateway subsystems. The most pressing technical debt is the unbounded audit log growth, unenforced WASM memory limits, and file-based vault keyring.

Overall, the architecture is sound and production-minded with defense-in-depth at its core. The technical debt items are mostly features-incomplete rather than architectural mistakes, and none are blockers for the system's core functionality.
