# My Action Items: Building an AI Gateway After Moltis

Prioritized, concrete next steps for someone building a similar AI gateway project. Derived directly from the lessons learned analysis.

---

## Phase 1: Foundation (Do These First)

### Action 1: Write CLAUDE.md Before Writing Code

**Why:** Moltis proved that a 405-line CLAUDE.md enabled 49 commits from AI agents with zero quality regressions. Without it, every AI collaborator makes up its own conventions.

**What to include:**
- Project overview and goal
- Tech stack decisions and why (with library versions)
- Code organization and naming conventions
- Error handling patterns (Context trait, thiserror usage)
- Secrets handling rules (secrecy::Secret always, no .unwrap())
- Testing requirements (unit + integration + E2E)
- CI gates that must pass
- Security practices (SSRF protection, input validation)
- Release workflow

**Deliverable:** `CLAUDE.md` at project root, committed before any production code.

---

### Action 2: Define Plugin Traits Before Implementations

**Why:** Moltis's trait-based architecture worked because the trait came first. Retrofitting traits onto existing implementations is painful.

**What to do:**
1. Define `LlmProvider` trait with `complete()`, `stream()`, `supports_tools()`, `context_window()`
2. Define `ChannelPlugin` trait with `start_account()`, `stop_account()`, `outbound()`, `status()`
3. Define `Sandbox` trait with `exec()`, `ensure_ready()`, `cleanup()`
4. Define `Tool` trait with `name()`, `description()`, `parameters_schema()`, `execute()`
5. Implement Noop versions of every trait for graceful degradation

**Do not:** Implement providers, channels, or tools before the trait exists.

---

### Action 3: Set Up CI/CD Before Code

**Why:** Moltis's pinned nightly toolchain and multi-layer CI prevented formatting drift, linter regressions, and test failures from accumulating. Retrofitting CI onto an existing project is much harder than starting with it.

**What to set up:**
1. `rust-toolchain.toml` pinning a specific nightly version
2. `justfile` with `fmt-check`, `lint`, `test`, `build` recipes matching CI exactly
3. GitHub Actions workflow with: fmt check, clippy (-D warnings), tests, TOML formatting
4. Pre-commit hook running local validation

**Critical:** The local `just` commands and CI commands must be byte-for-byte identical. Moltis's `./scripts/local-validate.sh` enforces this.

---

### Action 4: Implement Vault Before Storing Any Secrets

**Why:** Moltis's vault was added after secrets were already being stored in plaintext. Retrofitting encryption required migration.

**What to implement:**
1. KEK derived from password via Argon2id (64 MiB memory, 3 iterations)
2. Random DEK generated on vault initialization
3. DEK wrapped with KEK, stored encrypted
4. Recovery key generated and shown exactly once
5. AAD binding for each secret type (`"env:API_KEY"`, `"session:SESSION_TOKEN"`)
6. DEK held in `Zeroizing<[u8; 32]>` and cleared on drop

**Do not:** Store any API keys, passwords, or tokens in plaintext or in environment variables that get logged.

---

## Phase 2: Core Architecture

### Action 5: Build the Agent Loop First (Minimal Viable Agent)

**Why:** The agent loop is the core product. Everything else (providers, tools, channels) exists to serve it. Starting with a working agent makes all downstream decisions concrete.

**What to build:**
1. Single `LlmProvider` trait with `complete()` and `stream()` returning typed `ChatMessage` and `StreamEvent`
2. Single OpenAI provider implementation
3. `run_agent_loop()` that: builds prompt -> calls provider -> parses tool calls -> executes tools -> loops
4. Hard-coded iteration limit (25 default)
5. Typed `ToolCall` parsing from LLM output

**Start simple:** No provider chain, no failover, no sub-agents. Just one provider, one tool, one loop.

---

### Action 6: Add Provider Chain Failover Before Adding Second Provider

**Why:** Provider failover is complex (circuit breaker, error classification, streaming failover). Adding it too early, before the basic loop works, creates two complex systems at once.

**What to build:**
1. `ProviderChain` with ordered list of providers
2. Error classification via structured types (NOT string matching):
   ```rust
   pub enum ProviderErrorKind {
       RateLimit { retry_after: Option<u64> },
       AuthError,
       ServerError { status: u16 },
       ContextWindowExceeded { limit: u32 },
       InvalidRequest { message: String },
       Unknown,
   }
   ```
3. Circuit breaker: 3 consecutive failures trips, 60s cooldown
4. Streaming failover: select non-tripped provider before starting stream (cannot failover mid-stream)

**Do not:** Implement retry logic in `runner.rs` AND `provider_chain.rs`. Pick one layer.

---

### Action 7: Build the Sandbox Abstraction Early

**Why:** Tool execution security cannot be bolted on later. Moltis's sandbox abstraction (Docker, WASM, host) was well-designed but the `NoSandbox` passthrough was essential for development.

**What to build:**
1. `Sandbox` trait with `exec()`, `ensure_ready()`, `cleanup()`
2. `NoSandbox` (pass-through to host, default for dev)
3. `DockerSandbox` (ephemeral containers, network isolation)
4. `WasmSandbox` (wasmtime with fuel + epoch + memory limits)
5. Policy resolution: six-layer merge with deny-always-wins

**Start with:** `NoSandbox` + policy enforcement. The real sandbox comes after the agent loop works.

---

### Action 8: Define the Message Protocol Before Building Channels

**Why:** Moltis's WebSocket protocol v4 took multiple iterations. Channels (Discord, Slack, Telegram) are complex enough without protocol churn.

**What to define:**
1. Frame types: `RequestFrame`, `ResponseFrame`, `EventFrame`
2. Protocol version negotiation
3. Stream multiplexing via `stream` field
4. Channel multiplexing for multi-session support
5. JSON serialization with typed messages

**Do not:** Start building channel implementations before the protocol is stable.

---

## Phase 3: Security Hardening

### Action 9: Build SSRF Protection Into the HTTP Client

**Why:** Every `web_fetch` tool is an SSRF vector. Moltis's `ssrf.rs` handles it correctly but was added after the fact.

**What to implement:**
1. Private IP blocklist: 10.x, 172.16-31.x, 192.168.x, 100.64.x (CGNAT), loopback, link-local
2. IPv6 equivalents: fc00::/7, fe80::/10, loopback
3. CIDR allowlist for internal networks
4. Block both async and blocking HTTP variants
5. Reject URLs with LLM garbage patterns before fetching

**Do this:** Before the `web_fetch` tool exists.

---

### Action 10: Implement Rate Limiting on Auth Endpoints

**Why:** Moltis has rate limiting on the general API but not on auth endpoints specifically. Auth brute-force protection needs its own limits.

**What to implement:**
1. Token bucket per IP per scope
2. Auth-specific limits: login (5 req/min), API key auth (120 req/min)
3. WebSocket reconnect storm protection (30 req/min)
4. Cleanup every N requests to prevent memory growth

**Do this:** Before the auth system is exposed to the network.

---

### Action 11: Add Graceful Shutdown with Vault Seal

**Why:** Moltis's vault DEK stays in memory on crash. A clean shutdown should seal the vault.

**What to implement:**
1. `tokio::signal::ctrl_c()` handler
2. On shutdown signal: seal vault (clear DEK from memory), drain pending tool executions, close channels gracefully
3. Consider `mlock()` to prevent DEK from being swapped to disk

---

## Phase 4: Quality Infrastructure

### Action 12: Set Clippy Thresholds Correctly From Day One

**Why:** Moltis's `clippy.toml` disabled `too-many-arguments-threshold` and `type-complexity-threshold` with values of 999. This contributed to massive files.

**What to set:**
```toml
avoid-breaking-exported-api = true
too-many-arguments-threshold = 8
type-complexity-threshold = 500
```

**Enforce in CI:** `-D warnings` must catch all clippy violations. No `#[allow(clippy::unwrap_used)]` in production code (only in tests).

---

### Action 13: Build Three-Layer Testing from the Start

**Why:** Moltis's inline unit tests, integration tests, and Playwright E2E tests each catch different classes of bugs. Starting with E2E alone leaves unit-level bugs undetected.

**What to build:**
1. **Unit tests:** Inline `#[test]`, `#[tokio::test]` in source files
2. **Integration tests:** Per-crate `tests/` directories with full service initialization
3. **E2E tests:** Playwright specs for auth flows, chat, tool execution

**Start with:** Unit tests for core logic (agent loop, tool parsing, policy resolution). Add integration tests when two crates interact. Add E2E when the web UI exists.

---

### Action 14: Add Structured Error Types, Not String Matching

**Why:** Moltis's `classify_error()` uses `to_string().to_lowercase().contains(...)` which is fragile across API version changes.

**What to do:**
1. Define structured error enums per provider SDK
2. Map SDK errors to project error types at the boundary, not in business logic
3. Use `thiserror` for library errors, `anyhow::Error` for application errors

**Do not:** Pass raw SDK errors through the call stack without mapping to structured types.

---

## Phase 5: Scaling the Architecture

### Action 15: Split Large Files Before They Exceed 2000 Lines

**Why:** Moltis has files of 457K, 226K, 193K, 145K. At that size, no one can review them safely. Splitting is the only fix.

**What to do:**
1. Enforce a soft limit of 1500 lines per file
2. If a file exceeds 2000 lines, it MUST be split before merging
3. Use `clippy::too_many_lines` as a warning, fail CI at 2000

**How to split:**
- `runner.rs` (226K) → `runner.rs` + `retry.rs` + `tool_parsing.rs` + `prompt.rs`
- `providers/lib.rs` (145K) → per-provider files + `provider_chain.rs` + `types.rs`
- `gateway/server.rs` (193K) → separate route modules per concern

---

### Action 16: Consolidate Retry Logic Into One Layer

**Why:** Moltis has retry logic in both `runner.rs` (within-provider retries) and `provider_chain.rs` (provider failover). The interaction between these is unclear.

**What to do:**
1. Choose one layer for retry: either `runner` retries everything, OR `provider_chain` retries everything
2. If both are needed, document the boundary clearly
3. Consider a `RetryConfig` struct that both layers reference

**Recommendation:** Let `provider_chain` handle provider failover. Let `runner` handle transient errors (network, timeout) with a single retry strategy.

---

### Action 17: Design for Multi-User From Day One

**Why:** Moltis's single-user model (password table `CHECK (id = 1)`, fixed user ID for passkeys) would require significant architectural rework to support multi-user.

**What to do:**
1. Every data model gets a `user_id: String` or `user_id: Option<String>` field from the start
2. API key scoped to user: `user_id` is the first column in every auth query
3. Session ownership verified on every access
4. Rate limits per-user, not just per-IP

**If single-user is truly intended:** Document it explicitly and do not add `user_id` fields that are always `None`.

---

## Phase 6: Observability

### Action 18: Add Tracing and Metrics From the First Feature

**Why:** Moltis has comprehensive observability (tracing + metrics + feature gates) but it was added incrementally. Starting with it avoids the retrofit cost.

**What to do:**
1. Every async function gets `#[instrument]` from day one
2. Metrics facade with `metrics::counter!`, `metrics::histogram!` for every operation
3. Feature-gated metrics (no prometheus feature = no-op recorder)
4. Custom histogram buckets per metric type (HTTP: 1ms-60s, LLM: 100ms-5min)

**Do this:** Before the agent loop ships.

---

## Prioritization Summary

| Priority | Action | Why First |
|----------|--------|-----------|
| 1 | CLAUDE.md | Prevents convention drift from day 1 |
| 2 | Plugin traits | Everything depends on these |
| 3 | CI/CD setup | Prevents quality regressions |
| 4 | Vault | Secrets added early, encryption cannot be retrofitted |
| 5 | Minimal agent loop | Core product, validates architecture |
| 6 | SSRF protection | web_fetch tool implies SSRF vector |
| 7 | Provider chain failover | Second provider implies failover need |
| 8 | Sandbox abstraction | Tool security cannot be bolted on later |
| 9 | Clippy thresholds | Prevents large-file accumulation |
| 10 | Three-layer testing | Catches bugs at right level |
| 11 | Structured errors | String matching breaks across versions |
| 12 | Graceful shutdown + vault seal | DEK in memory on crash is a risk |
| 13 | File size limits | Prevents maintenance hazard |
| 14 | Single-layer retry | Duplicate retry logic is confusing |
| 15 | Multi-user design | Retrofitting is expensive |
| 16 | Tracing + metrics from start | Observability retrofit is costly |

---

## What Not to Do

Based on Moltis surprises and mistakes:

1. **Do not build all 52 crates at once.** Moltis started with a working agent, then added channels, then added tools, then added WASM. Each addition was justified by demonstrated need.

2. **Do not use `static OnceLock` for shared HTTP clients.** Pass `Arc<reqwest::Client>` through the DI container instead.

3. **Do not spawn subprocesses on hot paths.** Cache availability checks for a TTL.

4. **Do not disable clippy thresholds.** They exist to catch design problems early.

5. **Do not store the vault DEK without signal handlers.** A `SIGTERM` that does not seal the vault defeats the purpose of sealing.
