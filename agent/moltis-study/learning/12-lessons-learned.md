# Lessons Learned: Moltis Project Study

A structured assessment of architectural decisions, quality practices, technical debt, and notable findings from the Moltis project study.

---

## Category 1: Architectural Decisions to Emulate

### Lesson 1: Trait-Based Plugin Architecture Scales Well

**Finding:** Every major subsystem -- channels, tools, providers, sandbox -- uses a common trait pattern with `async_trait`, `Arc<dyn Trait>`, and `Send + Sync` bounds.

**Example from `crates/channels/src/plugin.rs`:**
```rust
#[async_trait]
pub trait ChannelPlugin: Send + Sync {
    fn id(&self) -> &str;
    async fn start_account(&mut self, account_id: &str, config: serde_json::Value) -> Result<()>;
    fn outbound(&self) -> Option<&dyn ChannelOutbound>;
}
```

**Example from `crates/memory/src/store.rs`:**
```rust
#[async_trait]
pub trait MemoryStore: Send + Sync {
    async fn vector_search(&self, query_embedding: &[f32], limit: usize) -> anyhow::Result<Vec<SearchResult>>;
    async fn keyword_search(&self, query: &str, limit: usize) -> anyhow::Result<Vec<SearchResult>>;
}
```

**Why it works:** The same pattern across 52 crates means developers only learn one abstraction pattern. Adding a new channel (Discord, Slack, Telegram, etc.) requires implementing the trait, nothing else.

**Verdict:** Adopt this pattern. Define traits early, implement against them.

---

### Lesson 2: Multi-Layer Policy Merging with Deny-Always-Wins

**Finding:** The tools system resolves policy across six layers (global, per-provider, per-agent, per-group, per-sender, per-sandbox) with deny-always-wins semantics.

**Example from `crates/tools/src/policy.rs`:**
```rust
pub fn resolve_effective_policy(config: &serde_json::Value, context: &PolicyContext) -> ToolPolicy {
    // Six-layer merge with deny accumulation
}
```

**Why it works:** Policy is additive in the allow direction but accumulates denials. A single `Deny` at any layer overrides all `Allow` decisions at lower layers. This makes privilege escalation impossible.

**Verdict:** This is a strong pattern for security-sensitive features. Adopt early before building tool systems.

---

### Lesson 3: Service Traits with Noop Defaults for Graceful Degradation

**Finding:** `crates/service-traits` defines ~20 domain service traits, each with a `Noop*` implementation that returns empty/default responses. The gateway starts without all services configured.

**Example:**
```rust
impl AgentService for NoopAgentService {
    async fn run(&self, params: RunParams) -> Result<RunResult> {
        Err(Error::Message("agent service not configured".into()))
    }
}
```

**Why it works:** The system does not require every service to be configured at startup. This enables incremental deployment and testing.

**Verdict:** Design services this way from the start. Retrofitting Noop implementations is painful.

---

### Lesson 4: Vault / DEK-KEK Separation is the Right Model for Secrets

**Finding:** The vault crate implements proper encryption-at-rest with Data Encryption Key (DEK) wrapped by a Key Encryption Key (KEK) derived from a user password via Argon2id. The DEK lives in memory only when the vault is "unsealed."

**Architecture:**
```
Password → Argon2id → KEK
                  ↓
          Random DEK
                  ↓
          XChaCha20-Poly1305 → Encrypted secrets
```

**Additional Authenticated Data (AAD)** binds encrypted values to their usage context (`"env:MY_API_KEY"`). Wrong AAD during decryption causes failure, preventing cross-context misuse.

**Verdict:** This is the correct model. Do not store secrets without a KEK-DEK separation and AAD binding.

---

### Lesson 5: SSRF Protection Belongs in the HTTP Client Layer

**Finding:** SSRF protection in `crates/tools/src/ssrf.rs` blocks private/loopback/link-local IPs at the HTTP client level. It covers IPv4 (10.x, 172.16-31.x, 192.168.x, 100.64.x CGNAT) and IPv6 (fc00::/7, fe80::/10), with CIDR allowlist support for internal networks.

**Example:**
```rust
pub fn is_private_ip(ip: &IpAddr) -> bool {
    match ip {
        IpAddr::V4(v4) => v4.is_loopback() || v4.is_private() || v4.is_link_local() || ...,
        IpAddr::V6(v6) => v6.is_loopback() || ...,
    }
}
```

**Verdict:** Build SSRF protection into the shared HTTP client from day one. Retrofitting it is error-prone.

---

### Lesson 6: WASM WIT Definitions Provide Stable Host-Guest Contracts

**Finding:** The `wit/` directory defines WIT (WebAssembly Interface Type) interfaces for WASM guest tools. Guest tools (calc, web-fetch, web-search) are compiled with `wasm32-wasip2` and communicate with the host through typed WIT interfaces.

**Verdict:** WIT definitions are a clean contract mechanism. Use them for any WASM extensibility.

---

### Lesson 7: CLAUDE.md as a First-Class Project Artifact

**Finding:** The project has a 405-line `CLAUDE.md` that serves as a comprehensive engineering guide covering Rust idioms, testing requirements, E2E testing with Playwright, security practices, database migration patterns, release workflow, and build/validation commands. This enabled 49 commits from AI agents.

**Verdict:** Invest in CLAUDE.md early. It is not documentation -- it is the project operating manual for AI collaborators.

---

## Category 2: Quality Practices to Emulate

### Lesson 8: Pinned Nightly Toolchain Eliminates Drift

**Finding:** `rust-toolchain.toml` pins to `nightly-2025-11-30`. All formatting, linting, and testing use this exact version. CI enforces exact match.

**Why it matters:** Rustfmt output differs between versions. A PR that passes CI might fail locally if the developer has a different nightly. Pinned toolchain eliminates this class of problem entirely.

**Verdict:** Pin your toolchain. Do not use `rustup default stable` or `nightly`.

---

### Lesson 9: Three-Layer Testing with Platform Awareness

**Finding:** Tests exist at three layers: inline unit tests (`#[test]`, `#[tokio::test]`), integration tests in per-crate `tests/` directories, and Playwright E2E tests. CI runs OS-aware test splits (macOS excludes metal/CUDA features).

**Example from `justfile`:**
```bash
test:
    if [ "$(uname -s)" = "Darwin" ]; then
        cargo +nightly-2025-11-30 nextest run --workspace --all-features --exclude moltis-providers --exclude moltis-gateway
        cargo +nightly-2025-11-30 nextest run -p moltis-providers --features local-llm-metal
    else
        cargo +nightly-2025-11-30 nextest run --workspace --all-features
    fi
```

**Verdict:** Three-layer testing with platform awareness catches more bugs and reduces CI flakiness.

---

### Lesson 10: Structured Error Enrichment via Context Trait

**Finding:** `crates/common/src/error.rs` defines a shared `Context` trait that enables `.context()` and `.with_context()` on `Result` and `Option`. This attaches location-specific context to errors without changing their type.

**Example:**
```rust
file_contents.context("reading config file")?;
option.ok_or_else(|| ...).with_context(|| "missing field")?;
```

**Why it matters:** When an error bubbles up through a 10-call stack, `.context()` makes it immediately clear where it originated.

**Verdict:** Adopt this pattern. It costs almost nothing and makes debugging dramatically faster.

---

## Category 3: Patterns to Avoid

### Lesson 11: Extremely Large Single Files Are a Maintenance Hazard

**Finding:** Several files are alarmingly large:
- `crates/chat/src/lib.rs`: 457K
- `crates/providers/src/lib.rs`: 145K
- `crates/agents/src/runner.rs`: 226K
- `crates/gateway/src/server.rs`: 193K

**Problem:** These files are nearly impossible to review, navigate, or modify safely. A change to one section risks breaking another. Onboarding new contributors is significantly harder.

**Root cause:** The project grew organically without splitting. No enforced module size limits in `clippy.toml` (the `type-complexity-threshold = 999` effectively disables this check).

**Verdict:** Set reasonable limits. Enforce module size checks in CI. Split files before they exceed ~2000 lines.

---

### Lesson 12: Error Classification via String Matching is Fragile

**Finding:** `classify_error()` in `crates/agents/src/provider_chain.rs` uses `err.to_string().to_lowercase().contains(...)` to classify provider errors into `RateLimit`, `AuthError`, `ServerError`, etc.

**Example:**
```rust
pub fn classify_error(err: &Error) -> ProviderErrorKind {
    let s = err.to_string().to_lowercase();
    if s.contains("429") || s.contains("rate limit") { RateLimit }
    else if s.contains("401") || s.contains("403") { AuthError }
    // ...
}
```

**Problem:** Provider error messages change between API versions. A provider that starts returning `"too many requests"` instead of `"rate limit"` would silently misclassify, potentially triggering wrong retry behavior.

**Verdict:** Use structured error types with explicit variant matching. Provider SDKs should return typed errors, not strings.

---

### Lesson 13: Retry Logic Duplication Across Layers

**Finding:** `runner.rs` and `provider_chain.rs` both implement retry logic with different strategies. `ProviderChain` handles provider-level failover (circuit breaker, 3 failures = trip). `runner` handles within-provider retries (exponential backoff, 10 max retries).

**Why this is a problem:** When a rate-limited request fails, does `runner` retry it while `provider_chain` is also trying the next provider? The interaction between these two retry systems is unclear from the code.

**Verdict:** Consolidate retry logic into a single layer. Failover (provider-level) and retry (request-level) should be clearly separated.

---

### Lesson 14: Single-User Model Limits Real-World Deployment

**Finding:** Authentication is designed for a single owner. The password table has `CHECK (id = 1)`. Passkeys are tied to a single user ID. API keys use a flat scope system without user isolation.

**Problem:** Any multi-user scenario (family sharing, team usage) requires significant architectural rework.

**Verdict:** If multi-user is a possibility, design for it from day one. User ID should be in every data model from the start.

---

### Lesson 15: No Structured Fuzzing or Property-Based Testing

**Finding:** Security analysis found no dedicated fuzzing suite, no `proptest` usage, and no penetration testing tooling. Security relies on unit tests and code review.

**Example missed:** Fuzzing the TOML config parser could find schema validation bypasses. Fuzzing the chat message parser could find injection vulnerabilities.

**Verdict:** Add at least basic fuzzing for public-facing parsers (config, message formats, tool parameters).

---

### Lesson 16: Clippy Thresholds are Disabled

**Finding:** `clippy.toml` sets `too-many-arguments-threshold = 999` and `type-complexity-threshold = 999`. Combined with `avoid-breaking-exported-api = false`, these effectively disable checks that normally catch design problems.

**Problem:** Large structs with many fields and complex types are not flagged. This contributed to the large-file problem.

**Verdict:** Use reasonable thresholds. `too-many-arguments-threshold = 8` is a common starting point. `avoid-breaking-exported-api = true` prevents accidental API regressions.

---

### Lesson 17: Global Static HTTP Client with OnceLock

**Finding:** Several crates use `static SHARED_CLIENT: OnceLock<reqwest::Client>` for connection pooling.

**Problem:** `OnceLock` panics if initialized twice. If any crate's initialization races, the result is a panic. The pattern also makes testing harder (shared state across tests).

**Verdict:** Pass HTTP clients as explicit dependencies rather than using global statics. Use `Arc<reqwest::Client>` through a DI container.

---

### Lesson 18: QMD Availability Check Spawns Subprocess Repeatedly

**Finding:** In `crates/memory/src/embeddings.rs`, `QMD::is_available()` spawns a subprocess on every call until the result is cached. Every memory search that checks QMD availability triggers this.

**Verdict:** Cache availability state for a reasonable TTL. Do not spawn processes on hot paths.

---

### Lesson 19: Vault Does Not Seal on Panic

**Finding:** The vault DEK stays in memory if the process crashes. There is no signal handler that seals the vault on `SIGTERM` or `SIGINT`.

**Problem:** A hard kill leaves the DEK readable in memory until the OS reclaims it.

**Verdict:** Add signal handlers for graceful shutdown that seal the vault. Consider `mlock()` to prevent the DEK from being swapped to disk.

---

## Category 4: Unexpected Findings

### Surprise 1: Date-Based Versioning with Static Cargo.toml

**Finding:** `Cargo.toml` has `version = "0.1.0"` permanently. Real versions are injected via `MOLTIS_VERSION` environment variable at build time. Releases use `YYYYMMDD.NN` pattern (e.g., `v0.10.18`).

**Implication:** The versioning is entirely decoupled from the crate registry. This means `crates.io` releases always show `0.1.0` while actual releases are managed through GitHub tags.

**Verdict:** Understand this trade-off. It works for a project that distributes through multiple channels (Homebrew, Docker, deb, rpm) rather than relying on `crates.io` for version reporting.

---

### Surprise 2: Single Maintainer with 94% of Commits is Sustainable with Automation

**Finding:** Fabien Penso accounts for 94.2% of commits. Yet the project has comprehensive CI/CD, multi-platform releases, E2E testing, and a CLAUDE.md that enabled 49 AI-agent commits.

**Why it works:** The automation compensates for the lack of human code reviewers. AI agents can safely make changes because the CLAUDE.md encodes all conventions.

**Implication:** A solo developer with good automation can maintain a large monorepo. The limiting factor is review bandwidth, not commit bandwidth.

---

### Surprise 3: Apple Container Sandbox is 44K LOC

**Finding:** `crates/tools/src/sandbox/apple.rs` is ~44,000 lines of macOS-specific container implementation. This is nearly as large as the entire WASM tools crate.

**Why it matters:** This represents a massive platform-specific maintenance burden. It also suggests the platform-specific code could benefit from better abstraction.

**Verdict:** Platform-specific code this large needs its own crate and clear interface boundaries. Do not let platform-specific code grow this large without abstraction.

---

### Surprise 4: Browser Element References are Numeric, Not CSS Selectors

**Finding:** The browser automation system (`crates/browser/`) identifies DOM elements by stable numeric references (e.g., `ref_42`) assigned at snapshot time, not by CSS selectors.

```javascript
// Each element gets: ref_, tag, role, text, href, placeholder, value, aria_label, bounds
// Stored as data-moltis-ref attribute for later retrieval
```

**Why this is smart:** CSS selectors break when the page changes. Numeric refs assigned at snapshot time are stable for that snapshot's duration. The model cannot accidentally query arbitrary DOM elements.

**Verdict:** Numeric refs are a better security boundary than CSS selectors for browser automation by LLMs.

---

### Surprise 5: Error Messages Prevented Claude from Making Security Mistakes

**Finding:** The error in `crates/mcp/src/sse_transport.rs` included this comment:

> "Note: we have to expose the display URL in the error because it's otherwise impossible to debug. This leaks the host name in error messages. We should fix this when we have proper secret.redact() support in place."

Claude immediately flagged this as a security concern without prompting. The human subsequently marked it as a known issue.

**Implication:** Even well-intentioned developers make secret-leaking mistakes. Automated secret detection in error paths is important.

---

### Surprise 6: Claude Agent Commits Follow the Same Patterns as Human Commits

**Finding:** The 49 AI-agent commits followed the same conventional commit format (`feat:`, `fix:`, `docs:`, etc.) and passed the same CI gates as human commits. The CLAUDE.md was sufficient to enforce consistency.

**Verdict:** AI agents can maintain code quality standards when given explicit guidance. The CLAUDE.md is not just documentation -- it is a contract that enforces quality.

---

## Category 5: Security Observations

### Strength: Defense in Depth

The security architecture has multiple independent layers:
1. **Vault** encrypts secrets at rest
2. **ApprovalManager** gates dangerous commands
3. **Policy layers** prevent privilege escalation
4. **SSRF protection** blocks network attacks
5. **WASM sandbox** limits what guest code can do
6. **Rate limiting** prevents brute-force attacks
7. **WebAuthn** with challenge TTL prevents replay

### Weakness: No IP-Based Rate Limiting on Auth

The rate limiter in `crates/httpd/src/request_throttle.rs` is per-IP, but the auth system does not have IP-based lockout after failed attempts. A determined attacker could try many passwords against a single IP.

### Weakness: Recovery Key Offline Attack Surface

The vault recovery key is stored with a hash verification. If an attacker gets read access to the vault file, they could brute-force the recovery phrase offline (it uses Bip39-style words, not high-entropy random).

---

## Summary Table

| Lesson | Category | Verdict |
|--------|----------|---------|
| Trait-based plugin architecture | Architect | Emulate |
| Multi-layer deny-always-wins policy | Architect | Emulate |
| Service traits with Noop defaults | Architect | Emulate |
| Vault DEK-KEK separation + AAD | Architect | Emulate |
| SSRF protection in HTTP client | Architect | Emulate |
| WIT definitions for WASM contracts | Architect | Emulate |
| CLAUDE.md as first-class artifact | Quality | Emulate |
| Pinned nightly toolchain | Quality | Emulate |
| Three-layer testing with platform awareness | Quality | Emulate |
| Context trait for error enrichment | Quality | Emulate |
| 44K-line single files | Debt | Avoid |
| String-matching error classification | Debt | Avoid |
| Retry logic duplication | Debt | Avoid |
| Single-user model from day one | Debt | Avoid |
| No fuzzing/property testing | Debt | Avoid |
| Clippy thresholds disabled | Debt | Avoid |
| Global static OnceLock HTTP client | Debt | Avoid |
| Subprocess spawn on hot path | Debt | Avoid |
| No vault seal on panic | Debt | Avoid |
| Date-based versioning tradeoff | Neutral | Understand |
| Solo maintainer with automation | Neutral | Understand |
| 44K LOC Apple sandbox | Warning | Flag |
| Numeric browser element refs | Positive | Note |
| Error message secret leaks | Warning | Flag |
