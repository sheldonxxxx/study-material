# Moltis Features Deep Dive

**Date:** 2026-03-26
**Source:** Feature batch research documents (05a-05e)
**Priority Order:** Core features first, then secondary features

---

## Core Features

### 1. AI Gateway (Multi-Provider LLM Support)

**Crate:** `crates/providers/`, `crates/gateway/`
**LoC:** 17.6K (providers), 36.1K (gateway)
**Priority:** CORE

#### What It Does

The AI Gateway provides unified LLM access through a pluggable provider architecture. Multiple providers (OpenAI, Anthropic, GitHub Copilot, Kimi, local models) implement a common `LlmProvider` trait, with automatic failover via `ProviderChain`.

#### How It's Implemented

**Provider Trait** (`moltis_agents::model::LlmProvider`):
```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn name(&self) -> &str;
    fn id(&self) -> &str;
    async fn complete(&self, messages: &[ChatMessage], tools: &[serde_json::Value]) -> anyhow::Result<CompletionResponse>;
    fn stream(&self, messages: Vec<ChatMessage>) -> Pin<Box<dyn Stream<Item = StreamEvent> + Send + '_>>;
    fn stream_with_tools(&self, messages: Vec<ChatMessage>, tools: Vec<serde_json::Value>) -> Pin<Box<dyn Stream<Item = StreamEvent> + Send + '_>>;
    fn supports_tools(&self) -> bool { false }
    fn context_window(&self) -> u32 { 200_000 }
    fn supports_vision(&self) -> bool { false }
    fn tool_mode(&self) -> Option<moltis_config::ToolMode> { None }
    fn reasoning_effort(&self) -> Option<ReasoningEffort> { None }
}
```

**Provider Chain (Circuit Breaker Pattern):**

Error classification determines failover behavior:
```rust
pub enum ProviderErrorKind {
    RateLimit,        // 429 — failover
    AuthError,        // 401/403 — failover
    ServerError,      // 5xx — failover
    BillingExhausted, // Quota — failover
    ContextWindow,    // Token limit — NO failover (trigger compaction)
    InvalidRequest,  // 400 — NO failover (will fail everywhere)
    Unknown,          // Try failover
}
```

Circuit breaker trips after 3 consecutive failures, resets after 60s cooldown.

**Model ID Namespace:** Providers use `provider::model-id` format (e.g., `"anthropic::claude-sonnet-4-20250514"`). Helper functions handle reasoning effort suffixes (`@reasoning-high`).

**Stream Events:**
```rust
pub enum StreamEvent {
    Delta(String),                                    // Text delta
    ProviderRaw(serde_json::Value),                  // Raw for debugging
    ReasoningDelta(String),                           // Extended thinking
    ToolCallStart { id, name, index },               // Tool call begins
    ToolCallArgumentsDelta { index, delta },        // Tool args streaming
    ToolCallComplete { index },                       // Tool args done
    Done(Usage),                                      // Stream finished
    Error(String),
}
```

#### Provider Implementations

| Provider | File | Notes |
|----------|------|-------|
| OpenAI | `providers/src/openai.rs` | Full-featured, streaming, tools |
| Anthropic | `providers/src/anthropic.rs` | Claude models, extended thinking |
| OpenAI Codex | `providers/src/openai_codex.rs` | GitHub Copilot |
| GitHub Copilot | `providers/src/github_copilot.rs` | OAuth discovery support |
| Kimi | `providers/src/kimi_code.rs` | Moonshot partner |
| OpenAI Compatible | `providers/src/openai_compat.rs` | OpenRouter, Ollama, etc. |
| Local GGUF | `providers/src/local_gguf/` | Local quantized models |
| Local LLM | `providers/src/local_llm/` | llama.cpp, ollama |

#### Notable Patterns

1. **Shared HTTP Client:** `providers::shared_http_client()` reuses connection pools
2. **Anthropic Extended Thinking:** Budget tokens configured per effort level (Low: 4096, Medium: 10240, High: 32768)
3. **Retry Logic:** Rate limits use exponential backoff (2s initial, 60s max, 10 retries); server errors fixed 2s delay with 1 retry
4. **Tool Sanitization:** Strips `functions_` prefix (OpenAI legacy) and trailing `_\d+` suffix (Kimi indexing)

#### Technical Debt

- **Large single files:** `providers/src/lib.rs` (145K) — monolithic provider registry
- **Error classification via string matching:** `classify_error()` uses `err.to_string().to_lowercase().contains(...)`, fragile to provider changes
- **Orphan tool handling:** Tool messages filtered by tracking `pending_tool_call_ids`

---

### 2. Chat Engine & Agent Orchestration

**Crate:** `crates/chat/`, `crates/agents/`
**LoC:** 11.5K (chat), 9.6K (agents)
**Priority:** CORE

#### What It Does

The chat engine orchestrates agent runs, managing conversation context, parallel tool execution, sub-agent delegation, and prompt assembly. The core agent loop lives in `moltis-agents::runner.rs`.

#### How It's Implemented

**Core Agent Loop** (`run_agent_loop()` ~500 lines):
1. Builds message list with system prompt + history + user content
2. Calls LLM via `provider.complete()`
3. On tool calls: executes concurrently, handles results
4. On text only: returns result
5. Handles retries, malformation recovery, context window errors

**Tool Execution Flow:**
```rust
let tool_futures: Vec<_> = response.tool_calls.iter().map(|tc| {
    async move {
        // BeforeToolCall hook
        // Execute tool
        // AfterToolCall hook
    }
}).collect();
```

**Tool Registry:**
```rust
pub trait AgentTool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, params: serde_json::Value) -> Result<serde_json::Value>;
}
```

Tool sources tracked for filtering: `Builtin`, `Mcp { server: String }`, `Wasm { component_hash: [u8; 32] }`.

**Hook System:**
```rust
HookPayload::BeforeLLMCall { session_key, provider, model, messages, tool_count, iteration }
HookPayload::AfterLLMCall { session_key, provider, model, text, tool_calls, ... }
HookPayload::BeforeToolCall { session_key, tool_name, arguments }
HookAction::Block(reason), ModifyPayload(new_value), Continue
```

#### Notable Patterns

1. **Iteration limit:** Default 25, lazy mode 75 (3x for tool_search discovery)
2. **Parallel tool execution:** All tools in a turn execute concurrently
3. **Tool result sanitization:** Strips base64 blobs (>= 200 chars), hex blobs (>= 200 hex chars), truncates with char boundary awareness
4. **Vision support:** When provider `supports_vision() == true`, tool results with images become multimodal content
5. **Explicit shell commands:** Detects `/sh ...` commands for forced execution

#### Runner Events (Callbacks)

```rust
pub enum RunnerEvent {
    Thinking, ThinkingDone,
    ToolCallStart { id, name, arguments },
    ToolCallEnd { id, name, success, error, result },
    ThinkingText(String), TextDelta(String),
    Iteration(usize),
    SubAgentStart { task, model, depth },
    SubAgentEnd { task, model, depth, iterations, tool_calls_made },
    RetryingAfterError { error, delay_ms },
}
```

#### Technical Debt

- **runner.rs (226K):** Monolithic agent loop file
- **Retry logic duplication:** `runner.rs` and `provider_chain.rs` both have retry logic with different strategies
- **Message filtering:** Skips orphan tool messages by tracking `pending_tool_call_ids`

---

### 3. Session Management & Persistence

**Crate:** `crates/sessions/`
**LoC:** 3.8K
**Priority:** CORE

#### What It Does

JSONL-based session persistence with SQLite-backed metadata, full-text search, and per-session state storage.

#### How It's Implemented

**Storage Architecture:**

Sessions stored at: `<data_dir>/agents/<agentId>/sessions/<sessionKey>.jsonl`

```rust
pub struct SessionStore {
    pub base_dir: PathBuf,
}
```

**Key Operations:**
```rust
impl SessionStore {
    pub async fn append(&self, key: &str, message: &serde_json::Value) -> Result<()>
    pub async fn read(&self, key: &str) -> Result<Vec<serde_json::Value>>
    pub async fn read_by_run_id(&self, key: &str, run_id: &str) -> Result<Vec<serde_json::Value>>
    pub async fn read_last_n(&self, key: &str, n: usize) -> Result<Vec<serde_json::Value>>
    pub async fn clear(&self, key: &str) -> Result<()>
    pub async fn replace_history(&self, key: &str, messages: Vec<serde_json::Value>) -> Result<()>
    pub async fn search(&self, query: &str, max_results: usize) -> Result<Vec<SearchResult>>
}
```

**Message Format:**
```rust
#[serde(tag = "role", rename_all = "lowercase")]
pub enum PersistedMessage {
    System { content: String, created_at: Option<u64> },
    Notice { content: String, created_at: Option<u64> },  // UI-only
    User { content: MessageContent, audio: Option<String>, channel: Option<serde_json::Value>, ... },
    Assistant { content: String, model: Option<String>, provider: Option<String>, tool_calls: Option<Vec<PersistedToolCall>>, reasoning: Option<String>, ... },
    Tool { tool_call_id: String, content: String, created_at: Option<u64> },
    ToolResult { tool_call_id: String, tool_name: String, arguments: Option<serde_json::Value>, success: bool, result: Option<serde_json::Value>, ... },
}
```

**Session State Store (KV):**

SQLite-backed key-value store scoped to `(session_key, namespace, key)`:
```rust
pub struct SessionStateStore {
    pool: sqlx::SqlitePool,
}
```

#### Notable Patterns

1. **Append-only JSONL:** Messages only appended, never modified (except `replace_history` for compaction)
2. **File locking:** `RwLock` from `fd_lock` crate for safe concurrent reads/writes
3. **Run ID filtering:** Can read messages for specific agent run
4. **Sequence numbers:** Client-assigned `seq` for ordering diagnostics
5. **Notice messages:** UI-only info not sent to LLM

#### Technical Debt

- **Simple substring search:** Not semantic/vector; one hit per session limitation
- **Orphan tool handling:** Tool messages without matching assistant tool_call_id skipped on replay

---

### 4. Memory & Context System

**Crate:** `crates/memory/`, `crates/qmd/`
**Priority:** CORE

#### What It Does

Hybrid storage combining SQLite full-text search with vector embeddings for long-term memory. Includes embeddings cache, chunk management, and automatic context injection.

#### How It's Implemented

**MemoryStore Trait:**
```rust
#[async_trait]
pub trait MemoryStore: Send + Sync {
    // Files
    async fn upsert_file(&self, file: &FileRow) -> anyhow::Result<()>;
    async fn get_file(&self, path: &str) -> anyhow::Result<Option<FileRow>>;
    async fn delete_file(&self, path: &str) -> anyhow::Result<()>;
    async fn list_files(&self) -> anyhow::Result<Vec<FileRow>>;

    // Chunks
    async fn upsert_chunks(&self, chunks: &[ChunkRow]) -> anyhow::Result<()>;
    async fn get_chunks_for_file(&self, path: &str) -> anyhow::Result<Vec<ChunkRow>>;
    async fn delete_chunks_for_file(&self, path: &str) -> anyhow::Result<()>;
    async fn get_chunk_by_id(&self, id: &str) -> anyhow::Result<Option<ChunkRow>>;

    // Embedding cache
    async fn get_cached_embedding(&self, provider: &str, model: &str, hash: &str) -> anyhow::Result<Option<Vec<f32>>>;
    async fn put_cached_embedding(&self, ...) -> anyhow::Result<()>;
    async fn put_cached_embeddings_batch(&self, entries: &[CacheEntry<'_>]) -> anyhow::Result<()>;
    async fn evict_embedding_cache(&self, keep: usize) -> anyhow::Result<usize>;

    // Search
    async fn vector_search(&self, query_embedding: &[f32], limit: usize) -> anyhow::Result<Vec<SearchResult>>;
    async fn keyword_search(&self, query: &str, limit: usize) -> anyhow::Result<Vec<SearchResult>>;
}
```

**Data Flow:**
1. File Discovery — Walk through configured `memory_dirs` for `.md` and `.markdown` files
2. Hash-based Change Detection — SHA256 content hash + mtime + size for fast-path skipping
3. Chunking — Split content into overlapping chunks by line count
4. Embedding Generation — Call embedding provider with caching
5. Search — Hybrid search combining vector similarity + FTS5 keyword search

**SQLite Storage:**
- **FTS5** for full-text search with BM25 ranking
- **Vector similarity** using cosine similarity on BLOBs (f32 little-endian)
- **Embedding cache** with LRU eviction (max 50,000 rows)
- **Streaming top-K** using BinaryHeap for memory-efficient vector search

**Hybrid Search Merge:**
1. **Linear** — Weighted score combination
2. **RRF (Reciprocal Rank Fusion)** — Rank-based fusion: `score = weight / (rrf_k + rank + 1)`

**QMD Alternative Backend:**
`crates/qmd/` — External sidecar process supporting three search modes: `Keyword` (BM25), `Vector`, `Hybrid` (BM25 + vector + LLM reranking).

#### Clever Solutions

1. **Streaming vector search:** Uses `BinaryHeap` as min-heap, O(limit) memory instead of O(all_chunks)
2. **Batch embedding with cache:** Checks cache first, only embeds misses
3. **FTS5 query sanitization:** Strips special characters that would cause syntax errors
4. **Chunk ID as composite key:** Format `"path:chunk_index"` for easy lookup
5. **Source categorization:** Files containing "MEMORY" in path are "longterm", others are "daily"

#### Technical Debt

1. **Embedding dimension mismatch:** No validation that all embeddings have same dimensions
2. **Local embeddings FFI:** `allow(unsafe_code)` in `embeddings_local.rs` due to llama-cpp-2 bindings
3. **QMD availability check:** Spawns subprocess on every `is_available()` call until cached

---

### 5. Tool Execution & Sandboxed Sandbox

**Crate:** `crates/tools/`
**LoC:** 21.9K
**Priority:** CORE

#### What It Does

Secure tool execution in Docker containers, Apple Container, or WASM sandbox with package management, filesystem isolation, and resource limits.

#### How It's Implemented

**ExecTool struct:**
```rust
pub struct ExecTool {
    pub default_timeout: Duration,
    pub max_output_bytes: usize,        // 200KB default
    pub working_dir: Option<PathBuf>,
    approval_manager: Option<Arc<ApprovalManager>>,
    broadcaster: Option<Arc<dyn ApprovalBroadcaster>>,
    sandbox: Arc<dyn Sandbox>,          // NoSandbox by default
    sandbox_id: Option<SandboxId>,
    sandbox_router: Option<Arc<SandboxRouter>>,  // For dynamic per-session sandbox
    env_provider: Option<Arc<dyn EnvVarProvider>>,
    completion_callback: Option<ExecCompletionFn>,
    node_provider: Option<Arc<dyn NodeExecProvider>>,  // Remote node execution
    default_node: Option<String>,
}
```

**Sandbox Trait:**
```rust
#[async_trait]
pub trait Sandbox: Send + Sync {
    fn backend_name(&self) -> &'static str;
    async fn ensure_ready(&self, id: &SandboxId, image_override: Option<&str>) -> Result<()>;
    async fn exec(&self, id: &SandboxId, command: &str, opts: &ExecOpts) -> Result<ExecResult>;
    async fn cleanup(&self, id: &SandboxId) -> Result<()>;
    fn is_real(&self) -> bool { true }  // false for NoSandbox
    async fn build_image(&self, base: &str, packages: &[String]) -> Result<Option<BuildImageResult>>;
}
```

**Implementations:** Docker, Podman, Apple Container (44K LOC!), WASM, Platform (cgroup-based), Host, Router.

**Sandbox Configuration:**
```rust
pub struct SandboxConfig {
    pub mode: SandboxMode,           // Off, NonMain, All
    pub scope: SandboxScope,         // Session, Agent, Shared
    pub workspace_mount: WorkspaceMount,  // None, Ro, Rw
    pub home_persistence: HomePersistence,  // Off, Session, Shared
    pub image: Option<String>,        // e.g., "ubuntu:25.10"
    pub backend: String,             // "auto", "docker", "podman", "apple-container", "wasm"
    pub resource_limits: ResourceLimits,  // memory_limit, cpu_quota, pids_max
    pub network: NetworkPolicy,       // Blocked, Trusted, Open
    pub trusted_domains: Vec<String>,
    pub packages: Vec<String>,       // apt-get packages to install
}
```

**Approval System:**
- Three modes: `Off`, `OnMiss` (approve safe + cached), `Always`
- Security levels: `Deny`, `Allowlist`, `Full`
- **130+ safe commands** whitelisted: `cat`, `echo`, `grep`, `jq`, `git`, `cargo`, `npm`, etc.
- **Dangerous pattern detection** via RegexSet: `rm -rf /`, `mkfs`, `dd if=/dev/zero`, `git reset --hard`, `DROP TABLE`, etc.

**Policy Resolution (6 layers, deny-always-wins):**
1. Global → 2. Per-provider → 3. Per-agent → 4. Per-group → 5. Per-sender → 6. Sandbox

#### Clever Solutions

1. **NoSandbox passthrough:** Default sandbox is `NoSandbox` for development
2. **Per-session sandbox routing:** `SandboxRouter` enables dynamic sandbox allocation per session
3. **Approval broadcast:** WebSocket-based approval request broadcasting for real-time UI updates
4. **Image hash tagging:** Deterministic image tags from base + packages hash for caching
5. **No-network proxy fallback:** Falls back to blocked network when proxy unavailable

#### Technical Debt

1. **Apple Container 44K LOC:** Substantial platform-specific code
2. **Complex sandbox router logic:** `router.rs` handles multiple backends with failover
3. **Timeout handling:** `tokio::time::timeout` kills process but doesn't guarantee cleanup
4. **Explicit cleanup required:** Must call `ExecTool::cleanup()` on session end

---

### 6. Authentication & Security

**Crate:** `crates/auth/`, `crates/vault/`
**Priority:** CORE

#### What It Does

Multi-layer auth including password + passkey (WebAuthn), API keys, OAuth integration, and encrypted vault for secrets.

#### How It's Implemented

**CredentialStore:**
```rust
pub struct CredentialStore {
    pool: SqlitePool,
    setup_complete: AtomicBool,
    auth_disabled: AtomicBool,
    vault: Option<Arc<Vault>>,  // For encrypted env vars
}
```

**Auth Methods:** Password (Argon2id), Passkey (WebAuthn), ApiKey, Loopback.

**Valid API Key Scopes:**
```rust
pub const VALID_SCOPES: &[&str] = &[
    "operator.admin", "operator.read", "operator.write",
    "operator.approvals", "operator.pairing",
];
```

**Vault Architecture (Encryption-at-rest with DEK/KEK separation):**
```
Password → Argon2id → KEK (Key Encryption Key)
                    ↓
          Random DEK (Data Encryption Key)
                    ↓
          XChaCha20-Poly1305 → Encrypted secrets
```

**Three vault states:**
```rust
pub enum VaultStatus {
    Uninitialized,  // No password set
    Sealed,         // Vault exists, DEK not in memory
    Unsealed,       // DEK held in memory
}
```

**XChaCha20-Poly1305:**
```rust
impl Cipher for XChaCha20Poly1305Cipher {
    fn version_tag(&self) -> u8 { 0x01 }
    fn encrypt(&self, key: &[u8; 32], plaintext: &[u8], aad: &[u8]) -> Result<Vec<u8>, VaultError> {
        // Nonce: 24 bytes (random)
        // AEAD: XChaCha20-Poly1305 with AAD
        // Output: [nonce: 24][ciphertext + tag: N+16]
    }
}
```

**Argon2id parameters:**
```rust
pub struct KdfParams {
    pub m_cost: u32 = 65536,  // 64 MiB
    pub t_cost: u32 = 3,       // iterations
    pub p_cost: u32 = 1,      // parallelism
}
```

#### Security Features

1. **Argon2id:** Memory-hard KDF (64 MiB default)
2. **XChaCha20-Poly1305:** Modern AEAD, resistant to timing attacks
3. **Random nonces:** 24 bytes of randomness per encryption
4. **AAD context binding:** Encryption tied to usage context (e.g., `"env:MY_KEY"`)
5. **DEK in memory only when unsealed:** Sealed state = no key material
6. **Recovery key:** Bip39-style phrase for emergency access, shown exactly once

#### Technical Debt

1. **Single-user model:** No multi-user support
2. **No key rotation:** DEK never rotated, only re-wrapped with new KEK
3. **Recovery key storage:** Recovery phrase hash stored, could be brute-forced offline
4. **Vault seal on panic:** DEK stays in memory if process crashes
5. **No IP-based rate limiting:** Only authentication state tracking
6. **Auth disabled flag:** Bypasses all auth (useful for dev, risky for prod)

---

### 7. Web UI

**Crate:** `crates/web/`
**LoC:** 4.5K
**Priority:** CORE

#### What It Does

Built-in web interface with real-time chat, agent management, provider configuration, and session history. Includes PWA support with push notifications.

#### Implementation

Frontend assets in `crates/web/ui/` using Preact and Tailwind CSS. E2E tests in `crates/web/ui/e2e/` using Playwright.

The web UI communicates with the gateway via WebSocket for real-time updates and REST API for configuration.

---

## Secondary Features

### 8. Communication Channels (Telegram, Discord, WhatsApp, MS Teams, Slack)

**Crates:** `crates/channels/`, `crates/discord/`, `crates/telegram/`, `crates/whatsapp/`, `crates/msteams/`, `crates/slack/`
**Priority:** SECONDARY

#### What It Does

Multi-platform messaging integration with unified message handling, sender allowlisting, OTP verification flows, and bot capabilities.

#### How It's Implemented

**ChannelPlugin Trait:**
```rust
#[async_trait]
pub trait ChannelPlugin: Send + Sync {
    fn id(&self) -> &str;           // e.g. "telegram", "discord"
    fn name(&self) -> &str;          // Human-readable
    async fn start_account(&mut self, account_id: &str, config: serde_json::Value) -> Result<()>;
    async fn stop_account(&mut self, account_id: &str) -> Result<()>;
    fn outbound(&self) -> Option<&dyn ChannelOutbound>;
    fn status(&self) -> Option<&dyn ChannelStatus>;
    fn has_account(&self, account_id: &str) -> bool;
    fn account_ids(&self) -> Vec<String>;
    fn account_config(&self, account_id: &str) -> Option<Box<dyn ChannelConfigView>>;
    fn shared_outbound(&self) -> Arc<dyn ChannelOutbound>;
    fn shared_stream_outbound(&self) -> Arc<dyn ChannelStreamOutbound>;
}
```

**Inbound Modes:**

| Channel | Mode | Implementation |
|---------|------|----------------|
| Telegram | Polling | Long-polling via `teloxide` library |
| WhatsApp | GatewayLoop | Persistent WebSocket via `whatsapp-rust` |
| Discord | GatewayLoop | Persistent WebSocket via `serenity` library |
| Slack | SocketMode | WebSocket via `slack-morphism` |
| MS Teams | Webhook | HTTP webhook endpoints with signature verification |

**Capability Flags:**
```rust
pub struct ChannelCapabilities {
    pub inbound_mode: InboundMode,
    pub supports_outbound: bool,
    pub supports_streaming: bool,      // Edit-in-place message updates
    pub supports_interactive: bool,    // Button/menu interactions
    pub supports_threads: bool,         // Thread message context
    pub supports_voice_ingest: bool,   // Voice message handling
    pub supports_pairing: bool,        // WhatsApp QR code pairing
    pub supports_otp: bool,             // One-time password verification
    pub supports_reactions: bool,
    pub supports_location: bool,
}
```

**Discord OTP Self-Approval:**
For `dm_policy = Allowlist` with `otp_self_approval = true`:
1. First message: Issues 6-digit OTP challenge (code shown only in web UI)
2. Code reply: Verifies against stored code, auto-approves on match
3. Wrong code: Shows remaining attempts
4. Locked out: Too many failures, must wait

**Markdown-Aware Message Chunking:**
Discord's 2000-character limit handled with smart chunker that avoids splitting inside fenced code blocks.

#### Technical Debt

1. **Secret management:** Bot tokens stored as `Secret<String>` but serialized config contains redacted values
2. **Error recovery:** No automatic reconnection logic for gateway disconnections
3. **Rate limiting:** Not systematically handled across all channels
4. **Message editing:** Discord supports edit-in-place, but Telegram does not

---

### 9. Voice I/O (STT/TTS)

**Crate:** `crates/voice/`
**LoC:** 6.0K
**Priority:** SECONDARY

#### What It Does

Voice input/output with support for 9 speech-to-text and 5 text-to-speech providers. Audio transcription and synthesis integrated into chat flow.

#### STT Providers

| Provider | File | Notes |
|----------|------|-------|
| Whisper | `stt/whisper.rs` | OpenAI's Whisper API |
| Whisper CLI | `stt/whisper_cli.rs` | Local `whisper.cpp` binary |
| Deepgram | `stt/deepgram.rs` | Cloud STT with punctuation |
| ElevenLabs STT | `stt/elevenlabs.rs` | Low-latency option |
| Google STT | `stt/google.rs` | Google Cloud Speech-to-Text |
| Groq STT | `stt/groq.rs` | Fast inference provider |
| Mistral STT | `stt/mistral.rs` | Mistral's STT offering |
| SherpaOnnx | `stt/sherpa_onnx.rs` | On-device/offline STT |
| Voxtral Local | `stt/voxtral_local.rs` | Local streaming STT |

**STT Trait:**
```rust
#[async_trait]
pub trait SttProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn name(&self) -> &'static str;
    fn is_configured(&self) -> bool;
    async fn transcribe(&self, request: TranscribeRequest) -> Result<Transcript>;
}
```

#### TTS Providers

| Provider | File | Notes |
|----------|------|-------|
| ElevenLabs | `tts/elevenlabs.rs` | High-quality, low-latency, voice cloning |
| OpenAI TTS | `tts/openai.rs` | TTS-1 and TTS-1-HD models |
| Google TTS | `tts/google.rs` | WaveNet and Neural2 voices |
| Coqui | `tts/coqui.rs` | Open-source TTS |
| Piper | `tts/piper.rs` | On-device TTS |

**TTS Trait:**
```rust
#[async_trait]
pub trait TtsProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn name(&self) -> &'static str;
    fn is_configured(&self) -> bool;
    fn supports_ssml(&self) -> bool { false }
    async fn voices(&self) -> Result<Vec<Voice>>;
    async fn synthesize(&self, request: SynthesizeRequest) -> Result<AudioOutput>;
}
```

**Text Sanitization for TTS:**
LLM output contains markdown that TTS should not read literally. The `sanitize_text_for_tts()` function removes code blocks, URLs, headers, bold/italic markers, bullet lists, tables, SSML tags.

**Example:**
```
Input: "## Getting Started\n\n1. Install from https://rustup.rs/\n2. Run `cargo build`"
Output: "Getting Started\n\nInstall from\nRun cargo build"
```

#### Technical Debt

1. **No audio format negotiation:** Channel sends format, STT must accept
2. **Provider-specific features:** Some features (SSML, voice cloning) only available on certain providers
3. **Local vs cloud:** SherpaOnnx and Piper are on-device but may have quality/feature gaps

---

### 10. MCP Server Support

**Crate:** `crates/mcp/`
**Priority:** SECONDARY

#### What It Does

Model Context Protocol client integration supporting both stdio and HTTP/SSE transport modes.

#### How It's Implemented

**Architecture:**
```
McpManager
    │
    ├── McpClient (stdio) ── StdioTransport (child process)
    ├── McpClient (SSE) ──── SseTransport (HTTP/SSE)
    └── McpClient (SSE+OAuth) ── SseTransport + Auth
```

**Transport Trait:**
```rust
#[async_trait]
pub trait McpTransport: Send + Sync {
    async fn request(&self, method: &str, params: Option<Value>) -> Result<JsonRpcResponse>;
    async fn notify(&self, method: &str, params: Option<Value>) -> Result<()>;
    async fn is_alive(&self) -> bool;
    async fn kill(&self);
}
```

**StdioTransport:** Spawns child process, communicates via JSON-RPC over stdin/stdout. Pending requests tracked with `oneshot::Sender` for response routing.

**SseTransport:** HTTP/SSE transport for remote MCP servers with session ID tracking, bearer token injection, and 401 retry with OAuth re-authentication.

**Client State:**
```rust
pub enum McpClientState {
    Connected,      // Transport spawned
    Ready,          // Initialize handshake complete
    Authenticating, // OAuth in progress
    Closed,
}
```

**MCP Protocol Types:**
```rust
pub const PROTOCOL_VERSION: &str = "2024-11-05";
pub struct InitializeParams { protocol_version, capabilities, client_info }
pub struct McpToolDef { name, description?, input_schema }
pub struct ToolsCallResult { content: Vec<ToolContent>, is_error }
```

#### Technical Debt

1. **No tool change notifications:** MCP servers can signal tool list changes, not currently handled
2. **Limited resource support:** Resources and prompts MCP capabilities not implemented
3. **OAuth flow complexity:** Interactive OAuth requires browser, not suitable for headless servers

---

### 11. Scheduling (Cron/CalDAV)

**Crates:** `crates/cron/`, `crates/caldav/`
**LoC:** 5.2K combined
**Priority:** SECONDARY

#### What It Does

Time-based job scheduling with cron expression support and CalDAV calendar integration.

#### How It's Implemented

**CronSchedule (3 modes):**
```rust
pub enum CronSchedule {
    At { at_ms: u64 },              // One-shot at epoch ms
    Every { every_ms: u64, anchor_ms: Option<u64> },  // Fixed interval
    Cron { expr: String, tz: Option<String> }, // Cron expression
}
```

**CronPayload:**
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

**SessionTarget:**
```rust
pub enum SessionTarget {
    Main,              // Main conversation session
    Isolated,          // Throwaway session (default)
    Named(String),     // Persistent named session (e.g. "heartbeat")
}
```

**Heartbeat:**
Special system job (`__heartbeat__`) that periodically prompts the LLM to check for noteworthy items:
- `HEARTBEAT_OK` sentinel token — LLM responds when nothing needs attention
- `is_within_active_hours()` handles overnight windows (22:00-06:00)
- System events queue (20 events max) accumulates between ticks

**CalDAV Integration:**
`libdav`-based client for calendar CRUD operations:
- Hardcoded provider support: Fastmail, iCloud
- ETag-based optimistic concurrency for update/delete
- iCalendar building/parsing for event data

#### Notable Patterns

1. **Stuck job detection:** Jobs running >2 hours cleared and marked error
2. **Rate limiting:** Sliding window (default 10 jobs/minute), system jobs bypass
3. **Atomic state updates:** `running_at_ms` set BEFORE spawning async task, prevents duplicate runs

#### Technical Debt

1. **`__heartbeat__` magic name:** Hardcoded string in two places, should be a constant
2. **libdav dependency:** No embedded mock for unit tests

---

### 12. Skills System

**Crate:** `crates/skills/`
**Priority:** SECONDARY

#### What It Does

Reusable skill packages enabling agents to acquire new capabilities through structured skill definitions (directories with `SKILL.md` file containing YAML frontmatter and markdown instructions).

#### How It's Implemented

**Core Types:**

```rust
pub struct SkillMetadata {
    pub name: String,           // Internal name (slug)
    pub display_name: Option<String>,  // Original human-readable name
    pub description: String,
    pub allowed_tools: Vec<String>,
    pub dockerfile: Option<String>,
    pub requires: SkillRequirements,
    pub path: PathBuf,
    pub source: Option<SkillSource>,
}

pub enum SkillSource {
    Project,    // <data_dir>/.moltis/skills/
    Personal,   // <data_dir>/skills/
    Plugin,     // Bundled in plugin
    Registry,   // Installed from registry (e.g. skills.sh)
}
```

**Requirements Checking:**
- `check_bin()` — Uses `std::env::split_paths` + executable check
- npm installs use `--ignore-scripts` flag to prevent supply chain attacks

**Discovery (priority order):**
1. `~/.moltis/skills/` — Project-local
2. `~/.moltis/skills/` — Personal
3. `~/.moltis/installed-skills/` — Registry-installed
4. `~/.moltis/installed-plugins/` — Plugin skills

#### Technical Debt

1. **Manifest persistence:** Atomic writes via temp file + rename (POSIX atomic)
2. **Registry removal enforcement:** Only registry-installed skills can be removed

---

### 13. Hook System / Plugin Architecture

**Crate:** `crates/plugins/`
**LoC:** 1.9K
**Priority:** SECONDARY

#### What It Does

Event-driven hooks with 17 lifecycle event types, circuit breaker pattern, and BeforeToolCall gating for security inspection.

#### How It's Implemented

**HookEvent (17 types):**
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

**HookAction:**
```rust
pub enum HookAction {
    Continue,                    // Proceed normally
    ModifyPayload(Value),         // Replace/modify data
    Block(String),               // Block with reason
}
```

**HookHandler Trait:**
```rust
pub trait HookHandler: Send + Sync {
    fn name(&self) -> &str;
    fn events(&self) -> &[HookEvent];
    fn priority(&self) -> i32 { 0 }  // Higher runs first
    async fn handle(&self, event: HookEvent, payload: &HookPayload) -> Result<HookAction>;
}
```

**Circuit Breaker:** Auto-disables handlers after 3 consecutive failures; re-enables after 60s cooldown.

**Dispatch Logic:**
- Read-only events (`SessionStart`, `AgentEnd`): Parallel dispatch, Block/Modify ignored
- Modifying events (`BeforeToolCall`, `BeforeLLMCall`): Sequential, first Block short-circuits, last Modify wins

**Bundled Hooks:**
- `boot-md`: On `GatewayStart`, reads `BOOT.md` for startup injection
- `command-logger`: Appends `Command` events to JSONL
- `session-memory`: Saves session conversation to markdown on new/reset

#### Technical Debt

1. **Config requirement check skipped:** `hook_eligibility.rs` notes config check is deferred
2. **Bin existence check:** Uses `which` command rather than `std::env::split_paths` for cross-platform consistency

---

### 14. Browser Automation

**Crate:** `crates/browser/`
**LoC:** 5.1K
**Priority:** SECONDARY

#### What It Does

Headless browser control for web scraping, automated testing, and web-based tool execution via Chrome DevTools Protocol (CDP).

#### How It's Implemented

**Architecture:**
```
BrowserManager (manager.rs)
    │
    +-- BrowserPool (pool.rs) - manages multiple browser instances
    |       │
    |       +-- BrowserInstance (pool.rs) - single browser with pages
    |               │
    |               +-- chromiumoxide::Browser (CDP connection)
    |               +-- Page (CDP page handle)
    |               +-- BrowserContainer (container.rs, sandboxed only)
    │
    +-- BrowserAction handlers (manager.rs)
            +-- Navigate, Screenshot, Snapshot, Click, Type, Scroll, Evaluate, Wait, etc.
```

**BrowserPool Management:**
- Instance reuse: Sessions map to browser instances
- Memory limits: `memory_limit_percent` (default 90%) and `max_instances`
- Idle cleanup: Closes instances after `idle_timeout_secs` (default 300s)
- Hard TTL: Maximum 30-minute lifetime per instance to prevent Chromium memory leaks
- Low-memory mode: Auto-injects `--single-process`, `--renderer-process-limit=1`

**Container Support:**
Three backends with auto-detection priority: Apple Container (macOS) > Podman > Docker.

**DOM Snapshot System:**
Extracts interactive elements with numeric references (`data-moltis-ref` attribute):
- Tags: `a`, `button`, `input`, `select`, `textarea`, `[role="*"]`, `[onclick]`, `[tabindex]`
- Security: No CSS selectors exposed to model — elements identified by ref number only

**Browser Detection:**
Auto-detects Chrome, Chromium, Edge, Brave, Opera, Vivaldi, Arc across platforms with platform-specific install support.

#### Notable Patterns

1. **Element references over selectors:** Stable numeric refs prevent breakage from page updates
2. **Data URI for screenshots:** `data:image/png;base64,...` format allows UI display while sanitizer can strip for LLM context
3. **Hard TTL on instances:** 30-minute max prevents Chromium memory leak accumulation
4. **Navigate retry logic:** Automatic retry with fresh session on connection death
5. **URL validation:** Rejects URLs with suspicious LLM garbage patterns

---

### 15. Extensibility (WASM, GraphQL, Metrics)

#### 15A: WASM Tools

**Crate:** `crates/wasm-tools/`, `wit/`

**WIT World:** `pure-tool` and `http-tool` define WASM component interfaces.

**Implementations:**
- **Calc Tool:** Pure arithmetic expression evaluator with safety limits (MAX_EXPRESSION_CHARS: 512, MAX_TOKENS: 256, MAX_AST_DEPTH: 64, MAX_OPERATIONS: 512)
- **Web Fetch Tool:** HTTP fetch with content extraction (markdown/text), redirect following, 10s timeout, 2MB max
- **Web Search Tool:** Brave Search API integration

#### 15B: GraphQL API

**Crate:** `crates/graphql/`
**LoC:** 4.8K

**Schema Building:**
```rust
pub type MoltisSchema = Schema<QueryRoot, MutationRoot, SubscriptionRoot>;

pub fn build_schema(
    services: Arc<Services>,
    broadcast_tx: broadcast::Sender<(String, Value)>,
) -> MoltisSchema
```

**Context Structure:**
```rust
pub struct GqlContext {
    pub broadcast_tx: broadcast::Sender<(String, Value)>,  // For subscriptions
    pub services: Arc<Services>,  // Direct service access, no RPC indirection
}
```

**Key Design:**
1. **Direct service access:** Resolvers call domain services directly via `services!` macro, no RPC string dispatch
2. **Shared types:** Types use `#[derive(SimpleObject)]` for GraphQL and `#[derive(Deserialize)]` for service JSON
3. **Broadcast for subscriptions:** Real-time events via tokio broadcast channel
4. **Json scalar:** Dynamic JSON wrapped in custom scalar type

#### 15C: Metrics System

**Crate:** `crates/metrics/`

**Architecture:**
- **Facade pattern:** Uses `metrics` crate facade (allows switching backends)
- **Prometheus export:** When `prometheus` feature enabled, exports Prometheus format
- **Feature-gated:** No prometheus feature = no-op recorder
- **Histogram buckets:** Custom buckets per metric type

**Coverage:** HTTP, WebSocket, LLM (completions, tokens, cache, time-to-first-token), Session, Tool, Sandbox, MCP, Channel, Memory, Plugin, Browser, Cron, Auth, System.

---

## Cross-Cutting Observations

### Shared Patterns

1. **Trait-based abstraction:** MemoryStore, Sandbox, Cipher, ChannelPlugin all use traits for flexibility
2. **Async trait pattern:** `#[async_trait]` throughout all crates
3. **Builder pattern:** For complex initialization (ExecTool, plugin configuration)
4. **Configuration-driven:** JSON config with pointer-based lookups
5. **Error handling:** `anyhow::Result` at boundaries, `thiserror` for library errors
6. **Arc<RwLock<...>>:** Thread-safe shared state throughout
7. **Secret handling:** `secrecy::Secret<String>` for sensitive data with custom serde redaction

### Security Boundaries

1. **Vault:** Encrypts secrets at rest, DEK in memory only when unsealed
2. **ApprovalManager:** Gateway for dangerous commands, forces approval for destructive ops
3. **Policy layers:** Deny-always-wins merging prevents privilege escalation
4. **SSRF protection:** `web_fetch.rs` blocks private/IP ranges
5. **Path traversal protection:** Memory writes validate against `data_dir`
6. **FTS5 sanitization:** Prevents injection via special characters

### Integration Flows

**Agent exec command:**
```
Agent → Policy::resolve_effective_policy() → ApprovalManager::check_command() → ExecTool::execute() → Sandbox::exec()
```

**Agent searches memory:**
```
Agent → MemoryManager::search() → EmbeddingProvider::embed() (with cache) → MemoryStore::vector_search() + keyword_search() → SearchResult
```

**User stores API key:**
```
User → CredentialStore with vault → Vault::encrypt_string() (AAD: "env:KEY_NAME") → Encrypted blob in SQLite
```

---

## Technical Debt Summary

| Area | Issue | Severity |
|------|-------|----------|
| AI Gateway | Error classification via string matching | Medium |
| AI Gateway | Large single file (providers/lib.rs 145K) | Medium |
| Chat Engine | Monolithic runner.rs (226K) | High |
| Chat Engine | Retry logic duplication | Low |
| Sessions | Simple substring search, not semantic | Medium |
| Memory | No embedding dimension validation | Low |
| Memory | Local embeddings FFI (`allow(unsafe_code)`) | Medium |
| Tools | Apple Container 44K LOC | High |
| Tools | Explicit cleanup required | Medium |
| Auth | Single-user model only | Medium |
| Auth | Vault seal on panic | Low |
| Browser | Hard TTL prevents memory leak accumulation | N/A (by design) |
| MCP | No tool change notifications | Low |
| Scheduling | `__heartbeat__` magic string | Low |
| Hooks | Config requirement check deferred | Low |

---

*Document synthesized from feature batch research documents (05a-05e)*
*Generated: 2026-03-26*
