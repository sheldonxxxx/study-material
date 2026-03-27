# OpenFang Design Patterns

## Pattern Catalog

Patterns observed in OpenFang codebase with evidence and file references.

---

## 1. Trait-Based Abstraction

**Purpose:** Enable multiple implementations without client code knowing details.

**Evidence:**

| Trait | File | Purpose |
|-------|------|---------|
| `LlmDriver` | `openfang-runtime/src/llm_driver.rs` | Multiple LLM providers |
| `KernelHandle` | `openfang-runtime/src/kernel_handle.rs` | Kernel/runtime decoupling |
| `ChannelAdapter` | `openfang-channels/src/types.rs` | Pluggable integrations |
| `EmbeddingDriver` | `openfang-memory/src/substrate.rs` | Vector search providers |

**Implementation - LlmDriver:**
```rust
#[async_trait]
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn stream(&self, request: CompletionRequest, tx: mpsc::Sender<StreamEvent>) -> Result<CompletionResponse, LlmError>;
}
```

**Well-implemented:** `LlmDriver` — clean trait with 7 provider implementations (OpenAI, Anthropic, Gemini, Copilot, Claude Code, Qwen Code, Fallback).

**Could improve:** `ChannelAdapter` has many default no-op implementations forcing callers to check for specific error patterns (`crates/openfang-channels/src/types.rs`).

---

## 2. Event Bus (Observer Pattern)

**Purpose:** Loose coupling between components; event history for debugging.

**File:** `crates/openfang-runtime/src/event_bus.rs`

**Implementation:**
```rust
pub struct EventBus {
    sender: broadcast::Sender<Event>,                              // Global broadcast
    agent_channels: DashMap<AgentId, broadcast::Sender<Event>>,     // Per-agent
    history: Arc<RwLock<VecDeque<Event>>>,                         // Ring buffer (1000 events)
}

pub fn publish(&self, target: EventTarget, event: Event) -> Result<(), String>;
pub fn subscribe(&self, target: EventTarget) -> broadcast::Receiver<Event>;
```

**Event routing:**
- `EventTarget::Agent(id)` — specific agent
- `EventTarget::Broadcast` — all agents
- `EventTarget::System` — system-wide
- `EventTarget::Pattern` — pattern-matched

**Well-implemented:** History ring buffer enables replay and debugging. Per-agent channels enable targeted delivery.

**Note:** This pattern appears in architecture docs but implementation details from `event_bus.rs` not fully visible in batch files.

---

## 3. Actor-like Agent Model

**Purpose:** Independent agent lifecycles with message-driven behavior.

**Evidence:** Each agent has:
- Independent lifecycle (spawn, run, stop, kill)
- Per-agent message serialization via `agent_msg_locks: DashMap<AgentId, Arc<Mutex<()>>>`
- State machine: Idle -> Running -> Waiting -> Stopped

**File:** `crates/openfang-kernel/src/agent_registry.rs` (referenced in architecture)

**Well-implemented:** Per-agent Mutex prevents concurrent messages to same agent. State transitions are explicit.

**Pattern in agent loop (`openfang-runtime/src/agent_loop.rs`):**
```rust
// Max 50 iterations before giving up (prevents infinite loops)
// Max 3 retries with exponential backoff for rate-limited APIs
// Max 5 continuations for max-token truncation
```

---

## 4. Registry Pattern

**Purpose:** Track system state for various subsystems.

**Evidence - Multiple registries:**

| Registry | File | Purpose |
|----------|------|---------|
| `AgentRegistry` | `openfang-kernel/src/agent_registry.rs` | Running agents |
| `SkillRegistry` | `openfang-skills/src/registry.rs` | Installed skills |
| `HandRegistry` | `openfang-hands/src/registry.rs` | Tool packages |
| `PeerRegistry` | `openfang-wire/src/registry.rs` | OFP network peers |

**Implementation (`HandRegistry` as example):**
```rust
pub struct HandRegistry {
    hands: Arc<RwLock<HashMap<String, HandDefinition>>>,
    instances: Arc<RwLock<HashMap<String, HandInstance>>>,
    bundled: HashMap<String, (String, String)>,  // id -> (HAND.toml, SKILL.md)
}
```

**Well-implemented:** Concurrent access via `Arc<RwLock<HashMap<...>>>`. Bundled content loaded at compile time via `include_str!()`.

---

## 5. Builder Pattern

**Purpose:** Construct complex objects with many optional parameters.

**Evidence:**

```rust
// AgentManifest builder
AgentManifest::builder()
    .name("my-agent")
    .provider("openai")
    .model("gpt-4")
    .system_prompt("You are helpful")
    .build()

// CompletionRequest builder
CompletionRequest::builder()
    .model("gpt-4")
    .messages(messages)
    .max_tokens(1000)
    .temperature(0.7)
    .build()

// Router builder (Axum)
Router::builder()
    .route("/api/agents", post(spawn_agent))
    .route("/api/agents/:id", get(get_agent))
    .build()
```

**Files:** `openfang-types/src/manifest.rs`, `openfang-runtime/src/drivers/mod.rs`, `openfang-api/src/routes.rs`

**Well-implemented:** Consistent builder chain across all complex types.

---

## 6. Strategy Pattern

**Purpose:** Select algorithm at runtime based on configuration.

**Evidence - LLM driver selection:**
```rust
pub fn create_driver(config: &DriverConfig) -> Result<Arc<dyn LlmDriver>, LlmError> {
    match provider {
        "anthropic" => Arc::new(anthropic::AnthropicDriver::new(api_key, base_url)),
        "gemini" | "google" => Arc::new(gemini::GeminiDriver::new(api_key, base_url)),
        "openai" | "groq" | "deepseek" | ... => Arc::new(openai::OpenAIDriver::new(api_key, base_url)),
        // ...
    }
}
```

**File:** `openfang-runtime/src/drivers/mod.rs`

**Well-implemented:** 27+ providers via OpenAI-compatible format. Clean match-based selection.

---

## 7. Plugin Architecture

**Purpose:** Extensibility via external packages.

**Evidence:**

| System | File | Extension Mechanism |
|--------|------|---------------------|
| Skills | `openfang-skills/src/loader.rs` | Python/Node/Shell/Prompt skill packages |
| Extensions | `openfang-extensions/src/lib.rs` | Generic extension framework |
| Hands | `openfang-hands/src/bundled.rs` | Autonomous agent configurations |
| Channels | `openfang-channels/src/types.rs` | `ChannelAdapter` trait |

**Skills loading (`loader.rs`):**
```rust
pub enum SkillRuntime {
    Python,
    Node,
    Shell,
    Wasm,        // RuntimeNotAvailable - not implemented
    Builtin,     // RuntimeNotAvailable - not implemented
    PromptOnly,
}

pub async fn execute_skill_tool(manifest, skill_dir, tool_name, input) {
    match manifest.runtime.runtime_type {
        Python => execute_python(),
        Node => execute_node(),
        Shell => execute_shell(),
        // ...
    }
}
```

**Well-implemented:** Security through `env_clear()` stripping ALL environment variables before spawning skill subprocesses.

**Could improve:** WASM runtime in skills system returns `RuntimeNotAvailable` — sandbox exists but not wired in.

---

## 8. Circuit Breaker Pattern

**Purpose:** Prevent cascading failures from repeated requests to failing services.

**Evidence:**

| Component | File | Implementation |
|-----------|------|----------------|
| Loop Guard | `openfang-runtime/src/loop_guard.rs` | Global circuit breaker (30 total calls) |
| Provider Cooldown | `openfang-runtime/src/provider_cooldown.rs` | Per-provider rate limit tracking |
| Auth Cooldown | `openfang-api/src/middleware/auth_cooldown.rs` | API auth failure tracking |

**Loop Guard (`loop_guard.rs`):**
```rust
pub enum LoopGuardVerdict {
    Allow,                    // Below thresholds
    Warn(String),             // Above warn_threshold
    Block(String),            // Above block_threshold
    CircuitBreak(String),    // Global threshold exceeded (30 total calls)
}

pub fn check(&mut self, tool_name: &str, params: &Value) -> LoopGuardVerdict;
```

**Detection mechanisms:**
1. Hash-Count Detection — SHA-256(tool_name | serialized_params)
2. Outcome-Aware Detection — hashes (tool_call_hash | result_truncated) pairs
3. Ping-Pong Detection — sliding window of last 30 calls

**Well-implemented:** Comprehensive test suite (30+ tests covering all detection modes).

---

## 9. Sandbox Pattern (WASM)

**Purpose:** Run untrusted code in isolated environment with resource limits.

**File:** `crates/openfang-runtime/src/sandbox.rs`

**Dual Metering:**
1. **Fuel Metering (CPU)** — deterministic instruction counting, default 1M fuel units
2. **Epoch Interruption (wall-clock)** — watchdog thread calls `engine.increment_epoch()` after timeout (default 30s)

**Guest ABI Contract:**
```rust
// Exports required from WASM module:
memory: LinearMemory      // Guest linear memory
alloc(size: i32) -> i32   // Allocate in guest memory
execute(input_ptr, input_len) -> i64  // Packed (ptr << 32) | len
```

**Host ABI (imports available to guest):**
- `host_call(request_ptr, request_len) -> i64` — RPC dispatch with capability checks
- `host_log(level, msg_ptr, msg_len)` — Logging without capability checks

**Capability System:**
```rust
pub struct GuestState {
    capabilities: Vec<Capability>,  // Deny-by-default
}

pub enum Capability {
    FileRead(String),   // Glob pattern
    FileWrite(String),
    NetConnect,
    ShellExec,
    EnvRead,
    MemoryRead,
    MemoryWrite,
    AgentMessage,
    AgentSpawn,
}
```

**Workspace sandbox (`workspace_sandbox.rs`):**
```rust
pub fn resolve_sandbox_path(user_path: &Path, workspace_root: &Path) -> Result<PathBuf, String> {
    // 1. Reject any `..` components outright
    // 2. Join relative paths with workspace_root
    // 3. Canonicalize the candidate (or parent for new files)
    // 4. Verify canonical path starts with canonical workspace root
}
```

**SSRF Protection (`host_functions.rs`):**
```rust
pub fn is_ssrf_target(url: &Url) -> bool {
    // Block non-http/https schemes
    // Block hostname blocklist: localhost, 169.254.169.254, metadata.google.internal
    // DNS-resolve and check every IP against private ranges
}
```

**Well-implemented:** Dual metering, capability inheritance on spawn, comprehensive path validation.

**Could improve:** `max_memory_bytes` config field exists but not enforced.

---

## 10. Env Clear / Least Privilege

**Purpose:** Prevent secret leakage to third-party subprocess code.

**File:** `crates/openfang-skills/src/loader.rs`

**Implementation:**
```rust
cmd.env_clear();  // Strip ALL environment variables first

// Re-add only essentials:
cmd.env("PATH", std::env::var("PATH").unwrap_or_default());
cmd.env("HOME", std::env::var("HOME").unwrap_or_default());
// Platform-specific: SYSTEMROOT, TEMP on Windows
cmd.env("PYTHONIOENCODING", "utf-8");  // Python
cmd.env("NODE_NO_WARNINGS", "1");     // Node
```

**Also used in:** MCP subprocess spawning (`openfang-runtime/src/mcp.rs`)

**Well-implemented:** Every skill subprocess gets `env_clear()`. API keys never leak to third-party skill code.

---

## 11. Memory Zeroization

**Purpose:** Ensure sensitive data is wiped from memory when no longer needed.

**Evidence:** All sensitive strings use `Zeroizing<String>` (from `zeroize` crate):

| Location | Data |
|----------|------|
| `openfang-extensions/src/vault.rs` | Vault entries, cached key |
| `openfang-extensions/src/oauth.rs` | OAuth tokens (access + refresh) |
| `openfang-channels/src/discord.rs` | Bot token |

**Implementation:**
```rust
pub struct CredentialVault {
    entries: HashMap<String, Zeroizing<String>>,  // Zeroized on drop
    cached_key: Option<Zeroizing<[u8; 32]>>,
}

impl Drop for CredentialVault {
    fn drop(&mut self) {
        self.entries.clear();
        self.cached_key = None;
        self.unlocked = false;
    }
}
```

**Discord adapter:**
```rust
pub struct DiscordAdapter {
    token: Zeroizing<String>,  // Auto-wipes on drop
    // ...
}
```

**Well-implemented:** Consistent use of `Zeroizing` for all secrets.

---

## 12. HMAC Nonce-based Authentication

**Purpose:** Mutual authentication for P2P OFP protocol.

**File:** `crates/openfang-wire/src/peer.rs`

**Handshake Flow:**
```
Client                              Server
  |                                   |
  |-- Handshake (nonce_a, HMAC) ----->|
  |    auth_hmac = HMAC(secret,      |
  |      nonce_a + node_id_a)         |
  |                                   | [Verify HMAC, check nonce replay]
  |                                   | [Generate nonce_b, HMAC]
  |<-- HandshakeAck (nonce_b, HMAC)--|
  |    auth_hmac = HMAC(secret,       |
  |      nonce_b + node_id_b)         |
  |                                   | [Verify HMAC, check nonce_b replay]
  |====== Authenticated Loop =========|
  |  session_key = HMAC(secret,       |
  |    nonce_a + nonce_b)             |
```

**Nonce Replay Protection:**
```rust
pub struct NonceTracker {
    seen: Arc<DashMap<String, Instant>>,  // nonce -> timestamp
    window: Duration,                      // 5 minutes
}

pub fn check_and_record(&self, nonce: &str) -> Result<(), String> {
    let now = Instant::now();
    self.seen.retain(|_, ts| now.duration_since(*ts) < self.window);  // GC old
    if self.seen.contains_key(nonce) {
        return Err(format!("Nonce replay detected: {}", nonce));
    }
    self.seen.insert(nonce.to_string(), now);
    Ok(())
}
```

**Session Key Derivation:**
```rust
pub fn derive_session_key(shared_secret: &str, our_nonce: &str, their_nonce: &str) -> String {
    let data = format!("{}{}", our_nonce, their_nonce);
    hmac_sign(shared_secret, data.as_bytes())
}
```

**Well-implemented:** Nonce replay protection with TTL, per-message HMAC after handshake, mandatory handshake rejection for unauthenticated messages.

---

## 13. Dual TOML Format Support

**Purpose:** Backward compatibility with multiple config file formats.

**File:** `crates/openfang-hands/src/lib.rs`

**Implementation:**
```rust
pub fn parse_hand_toml(content: &str) -> Result<HandDefinition, toml::de::Error> {
    // Try flat format first
    if let Ok(def) = toml::from_str::<HandDefinition>(content) {
        return Ok(def);
    }
    // Fall back to [hand] wrapper table format
    let wrapper: HandTomlWrapper = toml::from_str(content)?;
    Ok(wrapper.hand)
}
```

**Also in:** `openfang-types/src/manifest.rs` (likely similar pattern)

**Well-implemented:** Graceful fallback without complex format detection logic.

---

## 14. Bounded Storage with Eviction

**Purpose:** Prevent unbounded memory growth.

**Evidence:**

| Component | File | Bounded By |
|-----------|------|------------|
| `A2aTaskStore` | `openfang-runtime/src/a2a.rs` | `max_tasks: usize` (FIFO eviction) |
| `EventBus` history | `openfang-runtime/src/event_bus.rs` | `VecDeque<Event>` (1000 events) |
| `DeliveryTracker` | `openfang-kernel/src/delivery.rs` | LRU cache |
| `NonceTracker` | `openfang-wire/src/peer.rs` | 5-minute window |

**A2aTaskStore eviction:**
```rust
pub fn insert(&self, task: A2aTask) {
    // Evicts oldest completed/failed/cancelled tasks if at capacity
    if self.tasks.len() >= self.max_tasks {
        if let Some(oldest) = self.tasks.iter()
            .filter(|(_, t)| matches!(t.status.state(), TaskState::Completed | TaskState::Failed | TaskState::Cancelled))
            .min_by_key(|(_, t)| t.status.inner.timestamp())
        {
            self.tasks.remove(&oldest.0);
        }
    }
}
```

**Well-implemented:** FIFO eviction for completed tasks, time-bounded nonce tracking.

---

## 15. Exponential Backoff with Jitter

**Purpose:** Resilient reconnection without thundering herd.

**Evidence (multiple files):**

| File | Context | Initial | Max |
|------|---------|---------|-----|
| `openfang-channels/src/discord.rs` | WebSocket reconnect | 1s | 60s |
| `openfang-runtime/src/agent_loop.rs` | LLM retry | varies | varies |
| `openfang-runtime/src/mcp.rs` | ClawHub API retry | 1s | 30s |
| `packages/whatsapp-gateway/index.js` | Baileys reconnect | 1s | 60s |

**Discord implementation:**
```rust
let mut backoff = INITIAL_BACKOFF;  // 1s
loop {
    match ws_stream.connect_async(&connect_url).await {
        Ok(_) => backoff = INITIAL_BACKOFF,
        Err(_) => {
            backoff = (backoff * 2).min(MAX_BACKOFF);  // Double, cap at 60s
        }
    }
}
```

**ClawHub retry with jitter:**
```rust
let jitter = SystemTime::now().duration_since(UNIX_EPOCH).subsec_nanos() % 1000;
let delay = base_delay + Duration::from_millis(jitter);
```

**Well-implemented:** Consistent pattern across network operations.

---

## 16. Compile-Time Embedding

**Purpose:** Zero-runtime-I/O for core content.

**Evidence:**

| Content | File | Mechanism |
|---------|------|-----------|
| Bundled hands | `openfang-hands/src/bundled.rs` | `include_str!()` |
| Bundled skills | `openfang-skills/src/loader.rs` | `include_str!()` |
| WhatsApp gateway | `openfang-kernel/src/whatsapp_gateway.rs` | `include_str!()` |
| Tray icon | `openfang-desktop/src/lib.rs` | `include_bytes!()` |

**Example (`bundled.rs`):**
```rust
pub fn bundled_hands() -> Vec<(&'static str, &'static str, &'static str)> {
    vec![
        ("clip", include_str!("../bundled/clip/HAND.toml"), include_str!("../bundled/clip/SKILL.md")),
        ("lead", include_str!("../bundled/lead/HAND.toml"), include_str!("../bundled/lead/SKILL.md")),
        // ... 8 bundled hands
    ]
}
```

**WhatsApp gateway JS embedding:**
```rust
const INDEX_JS: &str = include_str!("../../../packages/whatsapp-gateway/index.js");
const PACKAGE_JSON: &str = include_str!("../../../packages/whatsapp-gateway/package.json");
```

**Well-implemented:** Eliminates runtime file I/O for bundled content.

**Could improve:** If embedded content needs updating, must recompile.

---

## 17. Credential Resolution Chain

**Purpose:** Flexible credential sourcing with priority order.

**File:** `crates/openfang-extensions/src/credentials.rs`

**Implementation:**
```rust
pub struct CredentialResolver {
    vault: Option<CredentialVault>,
    dotenv: HashMap<String, String>,  // Loaded from ~/.openfang/.env
    interactive: bool,
}

pub fn resolve(&self, key: &str) -> Option<Zeroizing<String>> {
    // Priority order:
    // 1. Vault (if unlocked)
    // 2. Dotenv file
    // 3. Environment variable
    // 4. Interactive prompt (CLI only)
}
```

**Well-implemented:** Clear priority, `Zeroizing` for memory safety, `clear_dotenv_cache()` for dashboard-driven deletions.

---

## 18. Workspace Pattern

**Purpose:** Isolated filesystem per agent for tool execution.

**File:** `crates/openfang-runtime/src/workspace_sandbox.rs`

**Implementation:**
```rust
pub struct Workspace {
    root: PathBuf,
    agent_id: String,
}

pub fn resolve_sandbox_path(&self, user_path: &Path) -> Result<PathBuf, String> {
    // Prevents:
    // - /tmp/../../etc/passwd (relative traversal)
    // - /etc/passwd when workspace is /home/user/project (absolute escape)
    // - symlink_to_outside/secret.txt (symlink escape)
}
```

**Well-implemented:** Symlink resolution via `canonicalize()`, explicit traversal prevention.

---

## 19. Session Compaction

**Purpose:** Manage unbounded conversation history.

**File:** `crates/openfang-memory/src/session.rs`

**Implementation:**
```rust
// Threshold: 100 messages (configurable)
// On exceeding threshold:
// - Older messages summarized (truncated text summaries)
// - Recent 50 kept
// - Summary capped at 4000 chars
// - compaction_cursor tracks progress for incremental compaction

pub fn append_canonical(agent_id, messages, threshold) -> CanonicalSession {
    if messages.len() > threshold {
        let summary = summarize_old_messages(old_messages)?;
        // Keep recent 50, replace old with summary
    }
}
```

**Well-implemented:** Incremental compaction with cursor, UTF-8 safe truncation.

---

## 20. Request/Response Wrapper Types

**Purpose:** Handle multiple serialization forms gracefully.

**Evidence (A2A protocol):**
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum A2aTaskStatusWrapper {
    Object { state: A2aTaskStatus, message: Option<serde_json::Value> },
    Enum(A2aTaskStatus),
}

impl A2aTaskStatusWrapper {
    pub fn state(&self) -> &A2aTaskStatus {
        match self {
            Self::Object { state, .. } => state,
            Self::Enum(s) => s,
        }
    }
}
```

**File:** `crates/openfang-runtime/src/a2a.rs`

**Well-implemented:** `#[serde(untagged)]` handles both `"completed"` and `{"state": "completed", "message": null}` forms.

---

## Pattern Summary

| Pattern | Files | Implementation Quality |
|---------|-------|----------------------|
| Trait-Based Abstraction | `llm_driver.rs`, `kernel_handle.rs`, `types.rs` | Good - clean traits, multiple implementations |
| Event Bus | `event_bus.rs` | Good - history buffer, routing |
| Actor-like Agents | `agent_loop.rs`, `agent_registry.rs` | Good - per-agent serialization |
| Registry | Multiple registry files | Good - concurrent access via RwLock |
| Builder | `manifest.rs`, `drivers/mod.rs` | Good - consistent chains |
| Strategy | `drivers/mod.rs` | Good - 27+ providers |
| Plugin Architecture | `skills/`, `extensions/`, `channels/` | Partial - WASM not wired |
| Circuit Breaker | `loop_guard.rs`, `provider_cooldown.rs` | Good - comprehensive detection |
| Sandbox (WASM) | `sandbox.rs`, `host_functions.rs` | Good - dual metering, SSRF protection |
| Env Clear | `loader.rs`, `mcp.rs` | Good - complete stripping |
| Memory Zeroization | `vault.rs`, `oauth.rs`, `discord.rs` | Good - consistent Zeroizing |
| HMAC Auth | `peer.rs` | Good - nonce replay protection |
| Dual TOML Format | `lib.rs` (hands) | Good - graceful fallback |
| Bounded Storage | `a2a.rs`, `event_bus.rs`, `peer.rs` | Good - eviction policies |
| Exponential Backoff | Multiple files | Good - consistent across network ops |
| Compile-Time Embedding | `bundled.rs`, `whatsapp_gateway.rs` | Good - zero I/O |
| Credential Resolution | `credentials.rs` | Good - clear priority chain |
| Workspace | `workspace_sandbox.rs` | Good - symlink resolution |
| Session Compaction | `session.rs` | Good - incremental with cursor |
| Request/Response Wrappers | `a2a.rs` | Good - untagged enum handling |
