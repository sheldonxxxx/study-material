# Deep Dive: Features Batch 2

**Date:** 2026-03-26
**Features Analyzed:**
- Feature 4: Memory & Context System (moltis-memory, moltis-qmd)
- Feature 5: Tool Execution & Sandboxing (moltis-tools)
- Feature 6: Authentication & Security (moltis-auth, moltis-vault)

---

## Feature 4: Memory & Context System

### Crates Analyzed
- `crates/memory/` - Primary memory backend (SQLite + FTS5 + vector embeddings)
- `crates/qmd/` - Alternative backend using QMD sidecar process

### Core Architecture

#### MemoryStore Trait (`crates/memory/src/store.rs`)
The memory system uses a trait-based abstraction allowing multiple backends:

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
    async fn count_cached_embeddings(&self) -> anyhow::Result<usize>;
    async fn evict_embedding_cache(&self, keep: usize) -> anyhow::Result<usize>;

    // Search
    async fn vector_search(&self, query_embedding: &[f32], limit: usize) -> anyhow::Result<Vec<SearchResult>>;
    async fn keyword_search(&self, query: &str, limit: usize) -> anyhow::Result<Vec<SearchResult>>;
}
```

#### Data Flow
1. **File Discovery** - Walk through configured `memory_dirs` looking for `.md` and `.markdown` files
2. **Hash-based Change Detection** - SHA256 content hash + mtime + size for fast-path skipping
3. **Chunking** - Split content into overlapping chunks by line count (targeting token budget)
4. **Embedding Generation** - Call embedding provider (OpenAI, local, fallback chain) with caching
5. **Search** - Hybrid search combining vector similarity + FTS5 keyword search

#### Key Implementation Details

**Chunking Strategy** (`crates/memory/src/chunker.rs`):
- Line-based splitting targeting `chunk_size` tokens
- `chunk_overlap` tokens of overlap between chunks for context preservation
- 1-based line numbers in output
- Falls back to tree-sitter AST splitting when grammar available (via `splitter.rs`)

**SQLite Storage** (`crates/memory/src/store_sqlite.rs`):
- **FTS5** for full-text search with BM25 ranking
- **Vector similarity** using cosine similarity on BLOBs (f32 little-endian)
- **Embedding cache** with LRU eviction (max 50,000 rows)
- **Streaming top-K** using BinaryHeap for memory-efficient vector search

**Hybrid Search Merge** (`crates/memory/src/search.rs`):
Two merge strategies:
1. **Linear** - Weighted score combination: `score = vector_score * weight + keyword_score * weight`
2. **RRF (Reciprocal Rank Fusion)** - Rank-based fusion that is score-magnitude-agnostic

```rust
// RRF formula
score = weight / (rrf_k + rank + 1)
where rrf_k = limit * 2
```

**Citation Handling**:
- `CitationMode::On` - Always include citations
- `CitationMode::Off` - Never include
- `CitationMode::Auto` - Include only when results span multiple files

#### Embedding Providers

**Trait** (`crates/memory/src/embeddings.rs`):
```rust
#[async_trait]
pub trait EmbeddingProvider: Send + Sync {
    async fn embed(&self, text: &str) -> anyhow::Result<Vec<f32>>;
    async fn embed_batch(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>>;
    fn model_name(&self) -> &str;
    fn dimensions(&self) -> usize;
    fn provider_key(&self) -> &str;
}
```

Implementations:
- `embeddings_openai.rs` - OpenAI API
- `embeddings_local.rs` - Local embeddings via llama-cpp-2 (FFI, requires `allow(unsafe_code)`)
- `embeddings_fallback.rs` - Chain of providers with fallback
- `embeddings_batch.rs` - Batch processing wrapper

#### QMD Alternative Backend

QMD is an external sidecar process (`crates/qmd/`):

**Manager** (`crates/qmd/src/manager.rs`):
- Manages QMD CLI lifecycle
- Supports three search modes: `Keyword` (BM25), `Vector`, `Hybrid` (BM25 + vector + LLM reranking)
- Per-collection indexing with JSON stats
- 30-second default timeout with configurable timeout

**QmdStore** (`crates/qmd/src/store.rs`):
- Implements `MemoryStore` trait
- Delegates file/chunk operations to optional fallback store
- Search operations use QMD with optional built-in memory merging
- QMD unavailable gracefully falls back to built-in memory

#### Memory Writing

The `MemoryManager` implements `MemoryWriter` trait from `moltis_agents`:
- Write to `MEMORY.md` or `memory/*.md` paths
- 50KB maximum content size
- Path traversal protection (validates within `data_dir`)
- Immediate re-indexing after write for instant searchability

#### Clever Solutions

1. **Streaming vector search** - Uses `BinaryHeap` as min-heap, O(limit) memory instead of O(all_chunks)
2. **Batch embedding with cache** - Checks cache first, only embeds misses
3. **FTS5 query sanitization** - Strips special characters that would cause syntax errors (e.g., "37.759" coordinates)
4. **Chunk ID as composite key** - Format `"path:chunk_index"` for easy lookup
5. **Source categorization** - Files containing "MEMORY" in path are "longterm", others are "daily"

#### Technical Debt / Concerns

1. **Embedding dimension mismatch** - No validation that all embeddings have same dimensions
2. **Local embeddings FFI** - `allow(unsafe_code)` in `embeddings_local.rs` due to llama-cpp-2 bindings
3. **QMD availability check** - Spawns subprocess on every `is_available()` call until cached
4. **In-memory embedding blobs** - Uses `bytemuck::try_cast_slice` with fallback to byte-by-byte deserialization

---

## Feature 5: Tool Execution & Sandboxing

### Crate Analyzed
- `crates/tools/` - Tool implementations (21.9K LoC)

### Core Architecture

The tools crate provides ~30 tool implementations and the sandbox execution engine.

#### Key Modules

| Module | Purpose |
|--------|---------|
| `exec.rs` | Shell command execution with sandbox integration |
| `approval.rs` | Command approval workflow with dangerous pattern detection |
| `policy.rs` | Multi-layered allow/deny policy resolution |
| `sandbox/` | Container execution backends (Docker, Podman, Apple Container, WASM) |
| `web_fetch.rs` | HTTP requests with SSRF protection |
| `web_search.rs` | Web search via multiple providers |
| `browser.rs` | Headless browser automation |
| `spawn_agent.rs` | Sub-agent spawning |

#### Exec Tool (`crates/tools/src/exec.rs`)

**ExecTool** struct composes multiple capabilities:
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

**Execution Flow**:
1. Check approval requirement via `ApprovalManager`
2. If approved (or no approval needed), execute via sandbox or direct
3. Apply timeout, capture stdout/stderr, truncate if > `max_output_bytes`
4. Fire completion callback

#### Sandbox Trait (`crates/tools/src/sandbox/types.rs`)

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

**Implementations**:
- `docker.rs` - Docker containers
- `containers.rs` - Generic container management (Docker/Podman)
- `apple.rs` - Apple Container on macOS (44K LOC!)
- `wasm.rs` - WASM sandbox execution
- `platform.rs` - Restricted host execution, cgroup-based isolation
- `host.rs` - Host provisioning with package installation
- `router.rs` - Dynamic sandbox routing with failover

#### Sandbox Configuration (`SandboxConfig`)

```rust
pub struct SandboxConfig {
    pub mode: SandboxMode,           // Off, NonMain, All
    pub scope: SandboxScope,         // Session, Agent, Shared
    pub workspace_mount: WorkspaceMount,  // None, Ro, Rw
    pub home_persistence: HomePersistence,  // Off, Session, Shared
    pub image: Option<String>,       // e.g., "ubuntu:25.10"
    pub backend: String,             // "auto", "docker", "podman", "apple-container", "wasm"
    pub resource_limits: ResourceLimits,  // memory_limit, cpu_quota, pids_max
    pub network: NetworkPolicy,       // Blocked, Trusted, Open
    pub trusted_domains: Vec<String>,
    pub packages: Vec<String>,       // apt-get packages to install
    pub wasm_fuel_limit: Option<u64>,
    pub wasm_epoch_interval_ms: Option<u64>,
}
```

#### Approval System (`crates/tools/src/approval.rs`)

**Three approval modes**:
- `Off` - No approval needed (except dangerous patterns)
- `OnMiss` - Approve safe commands + allowlist + cached approvals
- `Always` - Everything requires approval

**Security levels**:
- `Deny` - All exec denied
- `Allowlist` - Only allowlisted commands proceed
- `Full` - Everything allowed (except dangerous patterns that force approval)

**Dangerous pattern detection** uses RegexSet for efficiency:
- Filesystem destruction: `rm -rf /`, `mkfs`, `dd if=/dev/zero`
- Git destruction: `git reset --hard`, `git push --force`
- DB destruction: `DROP TABLE`, `TRUNCATE`
- Container destruction: `docker system prune`, `kubectl delete namespace`
- Fork bombs: `:() { ... };`

**130+ safe commands** whitelisted: `cat`, `echo`, `grep`, `jq`, `git`, `cargo`, `npm`, etc.

#### Policy Resolution (`crates/tools/src/policy.rs`)

Six-layer merge with **deny-always-wins** semantics:

1. **Global** - `tools.policy`
2. **Per-provider** - `tools.providers.<provider>.policy`
3. **Per-agent** - `agents.list[agent_id].tools.policy`
4. **Per-group** - `channels.<ch>.groups.<gid>.tools.policy`
5. **Per-sender** - `channels.<ch>.groups.<gid>.tools.bySender.<sender>`
6. **Sandbox** - `tools.exec.sandbox.tools` (when `sandboxed: true`)

```rust
pub fn resolve_effective_policy(config: &serde_json::Value, context: &PolicyContext) -> ToolPolicy {
    // Layer merge with deny accumulation
}
```

#### Shared HTTP Client

```rust
static SHARED_CLIENT: std::sync::OnceLock<reqwest::Client> = std::sync::OnceLock::new();

pub fn shared_http_client() -> &'static reqwest::Client {
    SHARED_CLIENT.get_or_init(reqwest::Client::new)
}
```
- Reuses connection pool, DNS resolver, TLS session cache
- Falls back to plain client if `init_shared_http_client()` never called

#### Remote Node Execution

NodeExecProvider trait allows routing exec to remote machines:
```rust
#[async_trait]
pub trait NodeExecProvider: Send + Sync {
    async fn exec_on_node(&self, node_id: &str, command: &str, ...) -> anyhow::Result<ExecResult>;
    async fn resolve_node_id(&self, node_ref: &str) -> Option<String>;
    fn has_connected_nodes(&self) -> bool;
}
```

### Clever Solutions

1. **NoSandbox passthrough** - Default sandbox is `NoSandbox` (pass-through to host) for development
2. **Per-session sandbox routing** - `SandboxRouter` enables dynamic sandbox allocation per session
3. **Command extraction** - Strips env var assignments and path prefixes to identify actual binary
4. **Approval broadcast** - WebSocket-based approval request broadcasting for real-time UI updates
5. **Image hash tagging** - Deterministic image tags from base + packages hash for caching
6. **No-network proxy fallback** - Falls back to blocked network when proxy unavailable

### Technical Debt / Concerns

1. **Apple Container 44K LOC** - Substantial platform-specific code
2. **Complex sandbox router logic** - `router.rs` handles multiple backends with failover
3. **ExecTool builder pattern** - Many optional fields, builder pattern with `with_*` methods
4. **Timeout handling** - `tokio::time::timeout` kills process but doesn't guarantee cleanup
5. **Sandbox cleanup on session end** - Must call `ExecTool::cleanup()` explicitly

---

## Feature 6: Authentication & Security

### Crates Analyzed
- `crates/auth/` - Authentication and credential management
- `crates/vault/` - Encryption-at-rest for secrets

### Auth Architecture

#### CredentialStore (`crates/auth/src/credential_store.rs`)

Single-user credential store backed by SQLite:

```rust
pub struct CredentialStore {
    pool: SqlitePool,
    setup_complete: AtomicBool,
    auth_disabled: AtomicBool,
    vault: Option<Arc<Vault>>,  // For encrypted env vars
}
```

**Tables**:
- `auth_password` - Argon2id hashed passwords (single row, id=1)
- `passkeys` - WebAuthn passkey credentials
- `api_keys` - API keys with scopes
- `env_variables` - Environment variables (optionally encrypted)
- `auth_state` - Global auth state (disabled flag)

#### Auth Methods

```rust
pub enum AuthMethod {
    Password,
    Passkey,
    ApiKey,
    Loopback,
}
```

**Valid API Key Scopes**:
```rust
pub const VALID_SCOPES: &[&str] = &[
    "operator.admin",
    "operator.read",
    "operator.write",
    "operator.approvals",
    "operator.pairing",
];
```

#### WebAuthn/Passkey Support (`crates/auth/src/webauthn.rs`)

**WebAuthnState** manages in-flight challenges with TTL:
```rust
pub struct WebAuthnState {
    webauthn: Webauthn,
    pending_registrations: DashMap<String, PendingRegistration>,
    pending_authentications: DashMap<String, PendingAuthentication>,
}
```

- 5-minute challenge TTL
- Single-user model (fixed user ID)
- Supports multiple RP IDs via **WebAuthnRegistry**

**Host normalization** handles:
- Port stripping
- IPv6 bracket handling
- Case normalization
- Trailing dot stripping

```rust
pub fn normalize_host(host: &str) -> String {
    // Strips port, lowercases, handles IPv6
}
```

#### Vault Architecture (`crates/vault/src/vault.rs`)

**Encryption-at-rest with DEK/KEK separation**:

```
Password → Argon2id → KEK (Key Encryption Key)
                    ↓
          Random DEK (Data Encryption Key)
                    ↓
          XChaCha20-Poly1305 → Encrypted secrets
```

**Three states**:
```rust
pub enum VaultStatus {
    Uninitialized,  // No password set
    Sealed,          // Vault exists, DEK not in memory
    Unsealed,        // DEK held in memory
}
```

**Recovery key** - Generated on initialization, wrapped DEK stored separately for password recovery

#### XChaCha20-Poly1305 Cipher (`crates/vault/src/xchacha20.rs`)

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

**Blob format**: `version_tag (1 byte) || nonce (24 bytes) || ciphertext || poly1305_tag (16 bytes)`

#### Key Derivation (`crates/vault/src/kdf.rs`)

**Argon2id parameters**:
```rust
pub struct KdfParams {
    pub m_cost: u32 = 65536,  // 64 MiB
    pub t_cost: u32 = 3,       // iterations
    pub p_cost: u32 = 1,      // parallelism
}
```

- Memory-hard to resist GPU/ASIC attacks
- 16-byte random salt (base64 encoded)
- 256-bit output (for XChaCha20 key)

#### Vault Operations

| Operation | Description |
|-----------|-------------|
| `initialize(password)` | Generate DEK, wrap with KEK, store metadata, return recovery key |
| `unseal(password)` | Derive KEK, unwrap DEK, hold in memory |
| `seal()` | Clear DEK from memory |
| `change_password(old, new)` | Verify old, derive new KEK, re-wrap DEK |
| `encrypt_string(plaintext, aad)` | AEAD encryption with version tag |
| `decrypt_string(b64, aad)` | Verify version, AEAD decryption, return plaintext |

#### Encryption Features

1. **AAD (Additional Authenticated Data)** - Context-specific (e.g., `"env:MY_KEY"`)
   - Wrong AAD during decryption causes failure
   - Prevents cross-context misuse

2. **Version tag** - Supports cipher migration
   - Currently only `0x01` (XChaCha20-Poly1305)
   - Unknown versions rejected

3. **Zeroizing memory** - `Zeroizing<[u8; 32]>` clears DEK on drop

4. **Recovery key** - Bip39-style phrase for emergency access
   - Stored with separate hash verification
   - Independent of password

#### Auth-Vault Integration

When `CredentialStore` created with vault:
```rust
pub async fn with_vault(
    pool: SqlitePool,
    auth_config: &moltisConfig::AuthConfig,
    vault: Option<Arc<Vault>>,
) -> Self
```

Environment variables can be encrypted:
```rust
pub struct EnvVarEntry {
    pub id: i64,
    pub key: String,
    pub created_at: String,
    pub updated_at: String,
    pub encrypted: bool,  // True if stored in vault
}
```

### Security Features

1. **Argon2id** - Memory-hard KDF (64 MiB default)
2. **XChaCha20-Poly1305** - Modern AEAD, resistant to timing attacks
3. **Random nonces** - 24 bytes of randomness per encryption
4. **AAD context binding** - Encryption tied to usage context
5. **DEK in memory only when unsealed** - Sealed state = no key material
6. **Recovery key** - Out-of-band recovery without password

### Clever Solutions

1. **Single-row password table** - Single-user model, `CHECK (id = 1)`
2. **DashMap for pending challenges** - Concurrent in-flight registration/auth
3. **WebAuthnRegistry by hostname** - Supports multiple hosts (localhost + mDNS)
4. **Trait-based Cipher** - Swap encryption backend without API changes
5. **Recovery key phrase** - Shown exactly once, stored with hash

### Technical Debt / Concerns

1. **Single-user model** - No multi-user support (password, passkeys are for one owner)
2. **No key rotation** - DEK never rotated, only re-wrapped with new KEK
3. **Recovery key storage** - Recovery phrase hash stored, could be brute-forced offline
4. **Vault seal on panic** - DEK stays in memory if process crashes
5. **No IP-based rate limiting** - Only authentication state tracking
6. **Auth disabled flag** - Bypasses all auth (useful for dev, risky for prod)

---

## Cross-Feature Observations

### Shared Patterns

1. **Trait-based abstraction** - MemoryStore, Sandbox, Cipher all use traits for flexibility
2. **Async trait pattern** - `#[async_trait]` throughout
3. **Builder pattern** - ExecTool, ApprovalManager use `with_*` methods
4. **Configuration-driven** - JSON config with pointer-based lookups
5. **Error handling** - `anyhow::Result` at boundaries, `thiserror` for library errors
6. **Streaming patterns** - BinaryHeap for top-K, stream processing for large datasets

### Security Boundaries

1. **Vault** - Encrypts secrets at rest, DEK in memory only when unsealed
2. **ApprovalManager** - Gateway for dangerous commands, forces approval for destructive ops
3. **Policy layers** - Deny-always-wins merging prevents privilege escalation
4. **SSRF protection** - `web_fetch.rs` blocks private/IP ranges
5. **Path traversal protection** - Memory writes validate against `data_dir`
6. **FTS5 sanitization** - Prevents injection via special characters

### Data Flow Integration

```
Agent wants to exec command
    → Policy::resolve_effective_policy()  [multi-layer merge]
    → ApprovalManager::check_command()      [dangerous pattern + approval mode]
    → ExecTool::execute()                  [sandbox or direct]
    → Sandbox::exec()                      [container or WASM]
    → Result returned to agent

Agent searches memory
    → MemoryManager::search()              [hybrid or keyword-only]
    → EmbeddingProvider::embed()            [with cache]
    → MemoryStore::vector_search() + keyword_search() [merge]
    → SearchResult returned

User stores API key
    → CredentialStore with vault
    → Vault::encrypt_string()              [AAD: "env:KEY_NAME"]
    → Encrypted blob stored in SQLite
```
