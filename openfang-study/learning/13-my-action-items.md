# My Action Items: Building a Similar Project

## Executive Summary

If I were building an Agent OS or similar complex AI infrastructure project, here are the concrete steps I would take, organized by priority. These are derived from both emulating openfang's strengths and avoiding its weaknesses.

---

## Phase 1: Foundation (Do First)

### 1.1 Establish Clean Architectural Separation

**Why:** OpenFang's kernel/runtime split via `KernelHandle` trait is its most important architectural pattern. It prevents circular dependencies while enabling testability.

**Action:** Define traits for all cross-component communication before writing implementation. The trait is the contract.

```rust
// Define FIRST:
#[async_trait]
pub trait RuntimeHandle: Send + Sync {
    async fn execute_agent(&self, manifest: &AgentManifest) -> Result<ExecutionResult, RuntimeError>;
    async fn list_agents(&self) -> Vec<AgentInfo>;
}

// Implementation SECOND:
pub struct Kernel {
    runtime: Arc<dyn RuntimeHandle>,
    // ...
}
```

**Checklist:**
- [ ] Kernel defines traits, runtime implements them
- [ ] No direct imports of kernel in runtime crate
- [ ] Integration tests can mock either side

### 1.2 Build Security Layers Incrementally and Independently

**Why:** OpenFang's 16 security layers work because each is independently testable. Building them all at once creates untestable complexity.

**Action:** Implement one security layer at a time with dedicated tests:

1. **Layer 1:** Audit log (append-only, hash-chained) -- test chain integrity
2. **Layer 2:** Capability gates -- test allow/deny logic
3. **Layer 3:** Taint tracking -- test label propagation
4. **Layer 4:** WASM sandbox -- test fuel exhaustion, epoch timeout, capability enforcement
5. **Layer 5:** SSRF protection -- test private IP blocking, DNS rebinding defense
6. **Layer 6:** Shell metachar blocking -- test injection vectors

**Checklist:**
- [ ] Each security layer has its own test file
- [ ] Tests cover both positive (should pass) and negative (should block) cases
- [ ] Security tests are CI-gated (don't allow security bugs in main)

### 1.3 Design Error Taxonomy Before Integration

**Why:** OpenFang's `llm_errors::classify_error()` shows that external service errors need normalization. Building this after the fact requires retrofitting.

**Action:** Define error classification schema early:

```rust
// Define error taxonomy with retry guidance
pub enum ErrorKind {
    Retryable { delay_ms: u64 },     // Rate limit, timeout, overload
    Billing,                          // Payment required, don't retry
    Auth,                            // Invalid credentials, won't succeed
    NotFound,                        // Resource doesn't exist
    InvalidInput,                    // Bad request, won't succeed
    Internal,                        // Our bug, should panic
}

pub struct ClassifiedError {
    pub kind: ErrorKind,
    pub sanitized_message: String,   // No internal details exposed
    pub source: String,              // "openai", "anthropic", etc.
}
```

**Checklist:**
- [ ] Error taxonomy defined in shared types crate
- [ ] All external calls use taxonomy
- [ ] Retry logic consults taxonomy, not raw errors

### 1.4 Set Up Multi-Platform CI from Day One

**Why:** OpenFang's clippy/format only runs on Ubuntu. Platform-specific bugs take longer to find.

**Action:**
```yaml
# CI matrix: Ubuntu + macOS + Windows for check + test
# Clippy + format on all platforms
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - name: Run clippy
        run: cargo clippy --workspace -- -D warnings
      - name: Run format check
        run: cargo fmt --check
```

**Checklist:**
- [ ] CI runs on all target platforms
- [ ] Clippy with `-D warnings` on all platforms
- [ ] Format check on all platforms
- [ ] Platform-specific issues filed as bugs, not deferred

---

## Phase 2: Core Infrastructure

### 2.1 Implement Graceful Shutdown First

**Why:** OpenFang has PID tracking, daemon.json cleanup, signal handlers. This is operational necessity.

**Action:**
```rust
// Implement shutdown sequence BEFORE adding features
async fn shutdown(&self) {
    // 1. Stop accepting new requests
    // 2. Wait for in-flight requests (with timeout)
    // 3. Flush state to disk
    // 4. Close database connections
    // 5. Kill child processes
    // 6. Remove PID files
}
```

**Checklist:**
- [ ] SIGINT and SIGTERM handled
- [ ] In-flight requests complete or timeout
- [ ] State persisted before exit
- [ ] Child processes cleaned up
- [ ] PID files removed

### 2.2 Build Observability Infrastructure Early

**Why:** OpenFang's structured logging via `tracing` enables operational visibility. Retrofitting is painful.

**Action:**
```rust
// Structured logging from start
tracing::info!(
    agent_id = %agent.id,
    iteration = iteration_count,
    tool_calls = tool_call_count,
    "Agent iteration started"
);

tracing::warn!(
    error = %error,
    retry_count = retry_count,
    "LLM call failed, retrying"
);
```

**Checklist:**
- [ ] JSON logging in production
- [ ] Span-based tracing for async operations
- [ ] Log level configurable at runtime
- [ ] Key fields are structured (not interpolated strings)

### 2.3 Implement Configuration Schema Validation

**Why:** OpenFang's config supports includes, deep merge, path traversal protection. This prevents security issues.

**Action:**
```rust
pub struct Config {
    #[serde(default)]
    pub agents: Vec<AgentConfig>,

    #[serde(default = "default_log_level")]
    pub log_level: String,

    #[serde(default)]
    pub channels: ChannelConfig,

    #[serde(default)]
    pub security: SecurityConfig,
}

impl Config {
    pub fn validate(&self) -> Result<(), ConfigError> {
        // Check for path traversal in includes
        // Check for circular includes
        // Validate required fields present
        // Validate value ranges (ports, timeouts)
    }
}
```

**Checklist:**
- [ ] Config schema documented
- [ ] Validation on load (fail fast)
- [ ] Hot reload with validation
- [ ] Config includes sandboxed (no absolute paths, no traversal)

### 2.4 Design for Testability

**Why:** OpenFang's integration tests boot real kernel + server. This requires upfront design.

**Action:**
```rust
// Boot for testing
pub async fn boot_for_testing() -> (Kernel, TestServer) {
    let config = Config::test();
    let kernel = Kernel::boot(config).await;
    let server = TestServer::new(kernel.clone()).await;
    (kernel, server)
}

// Test server uses random port
async fn new(kernel: Arc<Kernel>) -> Self {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let port = listener.local_addr().unwrap().port();
    // Spawn server...
}
```

**Checklist:**
- [ ] Kernel can boot with test config
- [ ] HTTP server binds to random port for testing
- [ ] Tests don't share state (fresh temp dirs)
- [ ] Integration tests cover API routes end-to-end

---

## Phase 3: Security Implementation

### 3.1 Implement Audit Log with Retention Policy

**Why:** OpenFang's audit log grows indefinitely. This is a known issue.

**Action:**
```rust
pub struct AuditLog {
    entries: Vec<AuditEntry>,
    max_entries: usize,  // e.g., 100,000
    retention_days: u32, // e.g., 90
}

impl AuditLog {
    pub fn prune(&mut self) {
        // Remove entries older than retention_days
        // If still over max_entries, remove oldest first
    }
}
```

**Checklist:**
- [ ] Retention policy configurable
- [ ] Pruning runs on schedule (not just at startup)
- [ ] Tests verify pruning logic
- [ ] Chain integrity verified after pruning

### 3.2 Implement Vault with True OS Keyring

**Why:** OpenFang's XOR obfuscation fallback is acknowledged as weak.

**Action:** Use the `keyring` crate with file-based fallback that **also** uses proper encryption (not XOR):

```rust
use keyring::Entry;

pub struct SecureVault {
    entry: Entry,
    cipher: AesGcm<Cipther Aes256Gcm>,
}

impl SecureVault {
    pub fn get(&self, service: &str) -> Result<Option<String>, VaultError> {
        // Try OS keyring first
        match self.entry.get_password() {
            Ok creds) => Ok(Some(creds)),
            Err(keyring::Error::NoEntry) => Ok(None),
            Err(_) => self.decrypt_fallback(service), // Proper decryption
        }
    }
}
```

**Checklist:**
- [ ] OS keyring (Keychain/Credential Manager/Secret Service)
- [ ] Fallback uses proper encryption (not XOR)
- [ ] Fallback key derived from machine fingerprint + user password
- [ ] Memory zeroized on drop (`Zeroizing`)

### 3.3 Implement Complete OAuth Flow with Token Refresh

**Why:** OpenFang's OAuth implementation lacks token refresh.

**Action:**
```rust
pub struct OAuthFlow {
    client_id: String,
    client_secret: Option<String>, // PKCE doesn't require
    redirect_uri: String,
}

impl OAuthFlow {
    pub async fn refresh_token(&self, refresh_token: &str) -> Result<Tokens, OAuthError> {
        // Exchange refresh token for new access token
        // Update refresh token
        // Store new tokens
    }
}
```

**Checklist:**
- [ ] Token refresh implemented
- [ ] Refresh token rotation supported
- [ ] Tokens stored in vault (not plain text)
- [ ] Expired token handling in API clients

---

## Phase 4: Operational Excellence

### 4.1 Add Coverage Reporting to CI

**Why:** OpenFang has no coverage reporting. Unknown blind spots.

**Action:**
```yaml
- name: Coverage
  run: |
    cargo install cargo-llvm-cov
    cargo llvm-cov --lcov --output-path lcov.info
- name: Coveralls
  uses: coverallsapp/github-action@v2
  with:
    flags: unittests
    format: lcov
```

**Checklist:**
- [ ] Coverage runs in CI
- [ ] Minimum coverage threshold (e.g., 70%)
- [ ] Coverage reports tracked over time
- [ ] New code requires coverage

### 4.2 Implement Structured TODO Tracking

**Why:** OpenFang's TODOs are scattered. No systematic tracking.

**Action:**
```rust
// Use TODO severity annotation
// TODO(high): Remove unwrap in production path (security risk)
// TODO(medium): Implement token refresh for OAuth
// TODO(low): Add coverage for error paths

// Or use issue tracking with labels
// github.com/project/issues?q=label%3Atech-debt
```

**Checklist:**
- [ ] TODOs annotated with severity
- [ ] Security TODOs treated as high priority
- [ ] TODOs converted to issues for tracking
- [ ] Tech debt tracked separately

### 4.3 Add Codeowners and Maintainers Files

**Why:** OpenFang's single-contributor dominance is a bus factor risk.

**Action:**
```markdown
# CODEOWNERS
# Default owners for everything
* @primary-maintainer

# Crate ownership
crates/security/ @security-team
crates/api/ @api-team

# CODEOWNERS requires @ mention
```

**Checklist:**
- [ ] CODEOWNERS file
- [ ] MAINTAINERS file with contact info
- [ ] AUTHORS file for contributors
- [ ] Secondary maintainers for each crate

### 4.4 Document Architecture Decisions

**Why:** OpenFang has extensive code comments but no ADR (Architecture Decision Record) folder.

**Action:**
```
docs/adr/
  0001-use-trait-based-dependency-injection.md
  0002-use-sqlite-for-persistence.md
  0003-use-tokio-for-async-runtime.md
  0004-security-layers-are-independent.md
```

**Checklist:**
- [ ] ADR folder exists
- [ ] Key decisions documented with context
- [ ] Consequences (pros/cons) documented
- [ ] ADRs updated when decisions change

---

## Phase 5: Feature Implementation

### 5.1 Integrate WASM Sandbox with Skills System

**Why:** OpenFang has two separate sandboxing systems (WASM and subprocess) not integrated.

**Action:**
```rust
// Skills should support WASM runtime
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SkillRuntime {
    Python,
    Node,
    Shell,
    Wasm {
        fuel_limit: u64,
        timeout_secs: u64,
        capabilities: Vec<Capability>,
    },
}
```

**Checklist:**
- [ ] WASM runtime implemented in skills loader
- [ ] Skills marketplace filters by runtime capability
- [ ] WASM skills can be installed and executed
- [ ] Capability grants validated at runtime

### 5.2 Split Large Files

**Why:** OpenFang's 431KB routes.rs and 183KB agent_loop.rs are hard to maintain.

**Action:**
```
crates/openfang-api/src/routes/
  mod.rs              // Router aggregation
  agents.rs           // Agent CRUD routes
  sessions.rs         // Session routes
  memory.rs           // Memory routes
  channels.rs         // Channel routes
  skills.rs           // Skills routes
  mcp.rs              // MCP routes
```

**Checklist:**
- [ ] Files under 500 lines
- [ ] Natural boundaries for splitting
- [ ] Module structure documented
- [ ] Tests still pass after split

### 5.3 Implement Real-time Metrics Dashboard

**Why:** OpenFang's dashboard serves Alpine.js SPA but metrics are not documented.

**Action:**
```rust
// Expose Prometheus-format metrics
metrics::counter!("agent_loops_total", "agent_id" => agent.name());
metrics::histogram!("llm_latency_seconds", "provider" => provider);
metrics::gauge!("active_agents", current_agent_count);
```

**Checklist:**
- [ ] `/metrics` endpoint in format Prometheus can scrape
- [ ] Key business metrics tracked
- [ ] Grafana dashboard defined
- [ ] Alerts configured for thresholds

---

## What to Borrow from OpenFang

### 1. The KernelHandle Pattern (Dependency Inversion)

```rust
// Consumer holds Arc<dyn Trait>, producer implements trait
// Breaks circular dependencies at architectural scale
```

### 2. Event Bus for Loose Coupling

```rust
pub struct EventBus {
    sender: broadcast::Sender<Event>,
    agent_channels: DashMap<AgentId, broadcast::Sender<Event>>,
    history: Arc<RwLock<VecDeque<Event>>>, // Ring buffer
}
```

### 3. Capability-Based Security

```rust
// Deny-by-default, glob matching for wildcards
capability_matches(granted: &Capability, required: &Capability) -> bool
```

### 4. Multi-Provider LLM Abstraction

```rust
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn stream(&self, request: CompletionRequest, tx: mpsc::Sender<StreamEvent>) -> Result<CompletionResponse, LlmError>;
}
```

### 5. Loop Guard with Outcome-Aware Detection

```rust
// Hash (tool_call + result_truncated) pairs
// Escalates if same call keeps returning identical results
```

### 6. Session Compaction Strategy

```rust
// Threshold: 100 messages
// Older messages summarized (truncated text summaries)
// Recent 50 kept
// Summary capped at 4000 chars
```

### 7. Hot Config Reload

```rust
// Poll config file every 30s
// Validate before applying
// Rollback on failure
```

---

## What to Avoid or Fix

### 1. Don't Use `.unwrap()` in Production

```rust
// BAD:
let path = config_path.parent().unwrap();

// GOOD:
let path = config_path.parent().ok_or_else(|| ConfigError::InvalidPath)?;
```

**Systematic fix:** Run clippy `unwrap` rule, replace all with proper error handling.

### 2. Don't Ship Partial Feature Implementations

Mark incomplete features as "experimental" or behind feature flags. Don't document them as shipped features if they return `RuntimeNotAvailable`.

### 3. Don't Use XOR for Security-Critical Obfuscation

If OS keyring isn't available, use Argon2id + AES-256-GCM with a key derived from user password, not reversible XOR with machine fingerprint.

### 4. Don't Run CI on Single Platform Only

At minimum, run check + test on all supported platforms. Lint on all platforms too.

### 5. Don't Let Audit Logs Grow Unbounded

Implement retention policy from day one. Unbounded logs will eventually cause storage issues.

### 6. Don't Duplicate Security Mechanisms

If you have `env_clear()` for subprocess sandboxing AND `subprocess_sandbox.rs` for isolation, consolidate them. Duplication creates inconsistency.

### 7. Don't Ignore Bus Factor

Single dominant contributor is a project risk. Actively cultivate secondary maintainers and document decisions in ADRs.

---

## Questions to Answer Before Starting

### Architecture

1. **Kernel/Runtime Split:** Is the separation necessary for our scale? OpenFang does it to avoid circular deps and enable testing. A simpler project might not need it.

2. **Trait-Based Polymorphism:** How many implementations do we expect? OpenFang uses traits for LLM drivers (7+), channels (50+), embedding providers (multiple). If we have few implementations, traits may be over-engineering.

3. **Event Bus:** Is pub/sub necessary? OpenFang uses it for agent events and system notifications. Simpler projects might use direct calls.

### Security

4. **Trust Boundaries:** What do we consider trusted vs untrusted? OpenFang treats all external input (user messages, skill code, MCP tools) as potentially malicious.

5. **Capability Model:** Is deny-by-default appropriate? OpenFang's capability system adds complexity but prevents accidental tool misuse.

6. **Audit Requirements:** What must we log? Regulatory requirements may mandate specific audit trails.

### Operations

7. **Persistence:** SQLite is sufficient for single-machine OpenFang. Would distributed deployment need different storage?

8. **Observability:** What metrics are essential? Define before building, not after.

9. **Multi-Tenancy:** Is multi-tenant operation needed? OpenFang is single-tenant.

### Scope

10. **Feature Scope:** OpenFang has 14 crates, 50+ channels, 16 security layers. What is our minimum viable feature set?

11. **Platform Support:** Which platforms (Linux, macOS, Windows)? Which architectures (x86, ARM)? OpenFang supports all three OSes and two architectures.

12. **SDK Requirements:** OpenFang has JS and Python SDKs. Are language SDKs necessary from day one?
