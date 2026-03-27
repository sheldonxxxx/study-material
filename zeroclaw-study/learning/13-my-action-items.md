# My Action Items: Building a ZeroClaw-Inspired AI Agent

**Derived from:** 12-lessons-learned.md analysis
**Purpose:** Prioritized next steps for building a similar AI agent framework

---

## Phase 1: Architecture & Foundation (Do First)

### A1.1 Define Core Trait Abstractions

**Why:** ZeroClaw's pluggable trait system is its greatest strength. Establish these before any implementation.

**Action:** Design and document these traits in a design document:

```rust
// Core agent traits to define upfront
trait Provider: Send + Sync {
    fn capabilities(&self) -> ProviderCapabilities;
    async fn chat(&self, request: ChatRequest) -> Result<ChatResponse>;
    async fn stream(&self, request: ChatRequest) -> Result<Stream>;
}

trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> Value;
    async fn execute(&self, args: Value) -> Result<ToolResult>;
}

trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, message: &SendMessage) -> Result<()>;
    async fn listen(&self, tx: Sender<ChannelMessage>) -> Result<()>;
}

trait Memory: Send + Sync {
    async fn insert(&self, entry: MemoryEntry) -> Result<()>;
    async fn retrieve(&self, query: &str, limit: usize) -> Result<Vec<MemoryEntry>>;
}
```

**Verify:** Trait definitions are in a single `core/traits.rs` module, documented, and tested with mock implementations.

### A1.2 Implement tokio::task_local for Request-Scoped Context

**Why:** ZeroClaw uses this for cost tracking and provider fallback info. It's the right tool for request-scoped state in async code.

**Action:**

```rust
tokio::task_local! {
    pub static REQUEST_CONTEXT: RefCell<Option<RequestContext>>;
}

pub async fn with_context<F: Future>(ctx: RequestContext, f: F) -> F::Output {
    REQUEST_CONTEXT.scope(RefCell::new(Some(ctx)), f).await
}
```

**Verify:** Any state that must not leak across concurrent requests uses task_local.

### A1.3 Set Up CI/CD Pipeline

**Why:** ZeroClaw's multi-platform builds and automated releases are essential for distribution.

**Action:**
- GitHub Actions with cargo nextest (faster than cargo test)
- Multi-platform builds: Linux x86_64, macOS ARM64, Windows x86_64
- Dependabot for daily dependency updates
- Release automation for package managers (Homebrew, etc.)

**Verify:** PR requires green CI, releases are automated.

### A1.4 Configure Clippy and rustfmt Strictly

**Why:** ZeroClaw's 1,056 clippy suppressions show what happens when you don't enforce standards from day one.

**Action:**
```toml
# clippy.toml
cognitive-complexity-threshold = 20  # Stricter than ZeroClaw's 30
too-many-arguments-threshold = 8       # Stricter than ZeroClaw's 10
too-many-lines-threshold = 150       # Stricter than ZeroClaw's 200
```

**Verify:** Zero clippy allow attributes in initial codebase.

---

## Phase 2: Security Foundation (Do Second)

### A2.1 ChaCha20-Poly1305 Encrypted Secrets

**Why:** ZeroClaw's `secrets.rs` implements proper AEAD encryption. Don't store secrets plaintext.

**Action:**
```rust
pub struct SecretStore {
    key_path: PathBuf,
}

impl SecretStore {
    pub fn encrypt(&self, plaintext: &str) -> Result<String> {
        let key = self.load_or_create_key()?; // 256-bit from OsRng
        let cipher = ChaCha20Poly1305::new(Key::from_slice(&key_bytes));
        let nonce = ChaCha20Poly1305::generate_nonce(&mut OsRng);
        let ciphertext = cipher.encrypt(&nonce, plaintext.as_bytes())?;
        Ok(format!("enc2:{}", hex::encode(nonce + ciphertext)))
    }
}
```

**Verify:** Secrets never appear plaintext in config files or logs.

### A2.2 Security Policy with Shell Parsing

**Why:** ZeroClaw's quote-aware shell parsing prevents injection attacks.

**Action:** Implement layered validation:
1. Autonomy gate (ReadOnly blocks all commands)
2. Subshell blocking (`$()`, backticks, process substitution)
3. Redirection blocking (`>`, `<` unquoted)
4. Per-segment command allowlist

```rust
enum QuoteState { None, Single, Double }

fn split_unquoted_segments(input: &str) -> Vec<String> {
    // Parse respecting quote state
    // `echo '$(rm -rf /)'` → single segment (literal)
}
```

**Verify:** Test injection attempts are blocked.

### A2.3 Credential Leak Detection

**Why:** Prevent accidental exfiltration of API keys.

**Action:** Implement high-entropy detection and API key pattern matching:
```rust
fn scan_for_secrets(content: &str) -> Vec<RedactedSegment> {
    // Stripe, OpenAI, GitHub, AWS patterns
    // High-entropy detection (Shannon > 4.375)
}
```

**Verify:** Leaked credentials in tool output are redacted before LLM consumption.

### A2.4 Workspace Isolation

**Why:** The agent should only access intended files.

**Action:**
- Canonicalize paths before checking
- Block `..` traversal
- Allowlist specific roots
- Protect config files

**Verify:** Agent cannot access `/etc/passwd` or `~/.ssh/`.

---

## Phase 3: Core Agent Loop (Do Third)

### A3.1 Build Minimal Agent Loop First

**Why:** ZeroClaw's 9,506-line loop_.rs is a modularity failure. Keep it small.

**Target:** Initial loop should be < 500 lines.

**Action:**
```
agent/
├── mod.rs          # Thin re-exports
├── loop.rs         # Main orchestration (< 500 lines)
├── dispatcher.rs   # Tool call parsing
├── prompt.rs       # System prompt construction
└── context.rs      # History management
```

**Verify:** Each module has < 200 lines.

### A3.2 Builder Pattern for Agent Construction

**Why:** Agent construction has many optional components.

**Action:**
```rust
let agent = AgentBuilder::new()
    .provider(provider)
    .tools(tools)
    .memory(memory)
    .security(security_policy)
    .build()?;
```

**Verify:** Required fields are validated in `build()`, optional fields have defaults.

### A3.3 Implement CancellationToken Propagation

**Why:** Long-running operations must be interruptible.

**Action:**
- All async operations accept `CancellationToken`
- Streaming responses check token on each chunk
- Tool execution respects cancellation

**Verify:** Ctrl-C stops in-progress streaming within 1 second.

---

## Phase 4: Data & Storage (Do Fourth)

### A4.1 SQLite with WAL Mode

**Why:** ZeroClaw's consistent SQLite patterns work well for embedded storage.

**Action:**
```rust
fn open_connection(path: &Path) -> Result<Connection> {
    let conn = Connection::open(path)?;
    conn.execute_batch(
        "PRAGMA journal_mode = WAL;
         PRAGMA synchronous = NORMAL;
         PRAGMA temp_store = MEMORY;"
    )?;
    Ok(conn)
}
```

**Verify:** Concurrent reads don't block writes.

### A4.2 Memory Retrieval Pipeline

**Why:** Three-stage retrieval (cache -> FTS -> vector) is effective.

**Action:**
```
cache (in-memory LRU, 256 entries, 5min TTL)
    ↓ miss
keyword search (SQLite FTS5)
    ↓ if score < 0.85
vector similarity + hybrid merge
```

**Verify:** Repeated queries hit cache.

### A4.3 Response Cache for LLM Calls

**Why:** Avoid burning tokens on repeated prompts.

**Action:**
```rust
// SHA-256(model || system_prompt || user_prompt) → cached response
// Two-tier: hot (LRU in-memory) + warm (SQLite)
```

**Verify:** Identical prompts return cached responses.

---

## Phase 5: Provider Abstraction (Do Fifth)

### A5.1 Implement OpenAI Provider First

**Why:** It's the most common backend. Use it to validate the Provider trait.

**Action:**
- Implement `Provider` trait for OpenAI API
- Support both chat and streaming
- Tool calling via function_call format

**Verify:** Agent can converse using OpenAI.

### A5.2 Implement Anthropic Provider

**Why:** Claude's tool use is different from OpenAI. Validates trait generality.

**Action:**
- Implement `Provider` trait for Anthropic
- Handle their specific tool_call format

**Verify:** Same agent code works with both providers.

### A5.3 Build ReliableProvider Wrapper

**Why:** Three-level failover (model -> provider -> retry) is essential for production.

**Action:**
```rust
struct ReliableProvider {
    providers: Vec<(String, Box<dyn Provider>)>,  // priority order
    model_fallbacks: HashMap<String, Vec<String>>,
    max_retries: u32,
}
```

**Verify:** Fallback to secondary provider when primary fails.

---

## Phase 6: Extensibility (Do Sixth)

### A6.1 Feature Flags for Heavy Dependencies

**Why:** ZeroClaw's feature-gated integrations keep default binary small.

**Action:**
```toml
[features]
default = ["observability-prometheus"]
channel-matrix = []
channel-discord = []
plugins-wasm = []
```

**Verify:** `cargo build` (no features) succeeds with minimal deps.

### A6.2 Pluggable Observer for Observability

**Why:** Different deployments need different observability backends.

**Action:**
```rust
pub trait Observer: Send + Sync {
    fn record_event(&self, event: Event);
}

enum Backend { Log, Prometheus, Otel, Noop }
```

**Verify:** Change observability backend via config.

---

## Phase 7: Testing (Do Ongoing)

### A7.1 Mock Implementations for All Traits

**Why:** Integration tests should not require real API keys.

**Action:**
```rust
struct MockProvider {
    responses: Vec<ChatResponse>,
}

impl Provider for MockProvider { ... }
```

**Verify:** Full agent loop testable without network.

### A7.2 Component Tests for Security Policy

**Why:** Security bugs are critical. Test all attack vectors.

**Action:**
- Test quote-aware parsing
- Test path traversal blocking
- Test credential scrubbing
- Test injection prevention

**Verify:** 100% of identified attack vectors have tests.

---

## What NOT to Do (Based on ZeroClaw's Mistakes)

### DO NOT: Create 9,506 Line Files

Split early, split often. If a module exceeds 500 lines, split it.

### DO NOT: Accumulate 1,056 Clippy Suppressions

Fix the underlying design issue or rewrite the code. Suppressions are technical debt.

### DO NOT: Leave CHANGELOG Empty

Use conventional-changelog or similar. Every release should have release notes.

### DO NOT: Use Global Statics for Request State

Use `tokio::task_local` or `Arc<Mutex<...>>` passed through call chains.

### DO NOT: Ship Without a Public Roadmap

Users and contributors need to see where the project is going.

### DO NOT: Have 3 People Own Everything

Distribute ownership. Bus factor of 3 is a risk.

---

## Priority Summary

| Priority | Action | Why |
|----------|--------|-----|
| P0 | Define core traits | Everything depends on this |
| P0 | Security policy (shell parsing) | Agent safety depends on it |
| P0 | CI/CD pipeline | Distribution depends on it |
| P1 | ChaCha20-Poly1305 secrets | Credential safety |
| P1 | Builder pattern for Agent | Complexity management |
| P1 | SQLite with WAL | Data safety |
| P2 | tokio::task_local patterns | Request-scoped context |
| P2 | Response cache | Cost savings |
| P2 | ReliableProvider wrapper | Production resilience |
| P3 | Feature flags | Binary size |
| P3 | Pluggable observer | Observability |
| Ongoing | Testing | Quality |

---

## Monitoring Checklist

- [ ] Trait implementations have mock variants for testing
- [ ] Security policy tested against known attack vectors
- [ ] No file exceeds 500 lines
- [ ] Clippy warnings are zero (no suppressions needed)
- [ ] CHANGELOG.md updated on every release
- [ ] Roadmap publicly visible
- [ ] Bus factor > 3 for core modules
