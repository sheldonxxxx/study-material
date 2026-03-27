# Features 10-12: MCP & A2A, P2P Wire Protocol, Credential Vault

**Batch:** 4 of N (Features 10-12)
**Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/openfang`

---

## Feature 10: MCP (Model Context Protocol) & A2A (Agent-to-Agent)

**Priority:** Secondary
**Crate:** `crates/openfang-runtime`

### MCP Client (`mcp.rs`) - Connecting to External MCP Servers

**Purpose:** Let OpenFang agents use tools from any MCP server (100+ available: GitHub, filesystem, databases, APIs).

#### Code Flow

```
McpServerConfig (name, transport, timeout_secs, env)
        |
        v
McpConnection::connect()  -->  initialize()  -->  discover_tools()
        |                           |
        |                     sends "initialize" JSON-RPC
        |                     sends "notifications/initialized"
        |
        v
discover_tools()  -->  "tools/list" request  -->  ToolDefinition[]
        |
        v
call_tool(name, arguments)  -->  "tools/call" request  -->  response
```

#### Key Implementation Details

**1. Dual Transport Support**

```rust
pub enum McpTransport {
    Stdio { command: String, args: Vec<String> },
    Sse { url: String },
}
```

- **Stdio:** Spawns subprocess, communicates via stdin/stdout JSON-RPC lines
- **SSE:** HTTP POST to endpoint, receives JSON-RPC response

**2. Tool Namespacing** (`mcp_{server}_{tool}`)

```rust
pub fn format_mcp_tool_name(server: &str, tool: &str) -> String {
    format!("mcp_{}_{}", normalize_name(server), normalize_name(tool))
}
// github + create_issue --> mcp_github_create_issue
// bocha-search + bocha_web_search --> mcp_bocha_search_bocha_web_search
```

**3. Hyphenated Tool Name Preservation**

A subtle issue: MCP servers may return tool names with hyphens (e.g., `list-connections`). The normalized name has underscores, but the server expects the original hyphenated form.

```rust
original_names: HashMap<String, String>,  // mcp_sqlcl_list_connections --> "list-connections"

pub async fn call_tool(&mut self, name: &str, arguments: &serde_json::Value) -> Result<String, String> {
    let raw_name = self.original_names.get(name)
        .map(|s| s.as_str())
        .or_else(|| strip_mcp_prefix(&self.config.name, name))
        .unwrap_or(name);
    // Sends original name to MCP server
}
```

**4. SSRF Protection for SSE Transport**

```rust
async fn connect_sse(url: &str) -> Result<McpTransportHandle, String> {
    let lower = url.to_lowercase();
    if lower.contains("169.254.169.254") || lower.contains("metadata.google") {
        return Err("SSRF: MCP SSE URL targets metadata endpoint".to_string());
    }
    // ...
}
```

**5. Subprocess Sandbox (Security Layer 11)**

```rust
cmd.env_clear();  // Clear ALL env vars first
for var_name in env_whitelist {
    if let Ok(val) = std::env::var(var_name) {
        cmd.env(var_name, val);  // Only pass whitelisted
    }
}
// Always pass PATH for binary resolution
```

Windows special handling for npm/npx .cmd batch wrappers.

**6. Background stderr logging**

```rust
if let Some(stderr) = child.stderr.take() {
    tokio::spawn(async move {
        let reader = tokio::io::BufReader::new(stderr);
        let mut lines = reader.lines();
        while let Ok(Some(line)) = lines.next_line().await {
            tracing::debug!(mcp_server = %cmd_name, "stderr: {line}");
        }
    });
}
```

#### Clever Solutions / Shortcuts

1. **Stateless handler** - `handle_mcp_request()` is a pure function that can be called from any transport layer
2. **Tool execution delegation** - Server doesn't execute tools; just validates requests and delegates to caller
3. **Graceful Windows compat** - Detects .cmd variants on PATH for npm/npx

#### Technical Debt / Concerns

1. **MCP server test coverage** - The `mcp_server.rs` file has a stub for tool execution (`"Tool '{tool_name}' is available. Execution must be wired by the host."`)
2. **No reconnection logic** - If an MCP stdio subprocess dies, no automatic restart
3. **SSE transport is basic** - Assumes single POST/response cycle; MCP spec may support streaming

---

### MCP Server (`mcp_server.rs`) - Exposing OpenFang Tools via MCP

**Purpose:** Allow external MCP clients (Claude Desktop, VS Code) to use OpenFang's built-in tools.

```rust
pub async fn handle_mcp_request(
    request: &serde_json::Value,
    tools: &[ToolDefinition],
) -> serde_json::Value
```

Supported methods:
- `initialize` - Returns protocol version 2024-11-05, tools capability
- `notifications/initialized` - No-op (notification)
- `tools/list` - Returns all available tools
- `tools/call` - **Stub** - Just validates tool exists, returns "Execution must be wired by the host"

#### Technical Debt

The tool execution is explicitly unimplemented:
```rust
// Tool execution is delegated to the caller (kernel/CLI).
// This handler just validates the request format.
// In a full implementation, the caller would wire this to execute_tool().
```

---

### A2A Protocol (`a2a.rs`) - Agent-to-Agent Interoperability

**Purpose:** Google's A2A protocol for cross-framework agent communication via Agent Cards and task-based coordination.

#### Core Types

```rust
pub struct AgentCard {
    pub name: String,
    pub description: String,
    pub url: String,              // e.g., "https://host/a2a"
    pub version: String,
    pub capabilities: AgentCapabilities,
    pub skills: Vec<AgentSkill>,  // Maps tools to A2A skills
}

pub struct AgentCapabilities {
    pub streaming: bool,
    pub push_notifications: bool,
    pub state_transition_history: bool,
}

pub enum A2aTaskStatus {
    Submitted, Working, InputRequired, Completed, Cancelled, Failed
}
```

**`A2aTaskStatusWrapper` - Handling Dual Serialization Forms**

A clever untagged enum handling:

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

This allows both `"completed"` and `{"state": "completed", "message": null}` forms.

**`A2aTaskStore` - Bounded Task Registry**

```rust
pub struct A2aTaskStore {
    tasks: Mutex<HashMap<String, A2aTask>>,
    max_tasks: usize,  // FIFO eviction when full
}

pub fn insert(&self, task: A2aTask) {
    // Evicts oldest completed/failed/cancelled tasks if at capacity
}
```

#### A2A Client Operations

```rust
pub async fn discover(&self, url: &str) -> Result<AgentCard, String>  // GET /.well-known/agent.json
pub async fn send_task(&self, url: &str, message: &str, session_id: Option<&str>) -> Result<A2aTask, String>
pub async fn get_task(&self, url: &str, task_id: &str) -> Result<A2aTask, String>
```

#### Agent Discovery at Boot

```rust
pub async fn discover_external_agents(
    agents: &[openfang_types::config::ExternalAgent],
) -> Vec<(String, AgentCard)> {
    // Called during kernel boot to populate known external agents
}
```

#### Tool-to-Skill Mapping

```rust
pub fn build_agent_card(manifest: &AgentManifest, base_url: &str) -> AgentCard {
    let tools: Vec<String> = manifest.capabilities.tools.clone();
    let skills: Vec<AgentSkill> = tools.iter().map(|tool| AgentSkill {
        id: tool.clone(),
        name: tool.replace('_', " "),
        description: format!("Can use the {tool} tool"),
        tags: vec!["tool".to_string()],
        examples: vec![],
    }).collect();
    // ...
}
```

---

## Feature 11: P2P Wire Protocol (OFP)

**Priority:** Secondary
**Crate:** `crates/openfang-wire`
**Files:** `lib.rs`, `message.rs`, `peer.rs`, `registry.rs`

### Protocol Design

**Message Framing:** 4-byte big-endian length + JSON body
```rust
pub fn encode_message(msg: &WireMessage) -> Result<Vec<u8>, serde_json::Error> {
    let json = serde_json::to_vec(msg)?;
    let len = json.len() as u32;
    let mut bytes = Vec::with_capacity(4 + json.len());
    bytes.extend_from_slice(&len.to_be_bytes());
    bytes.extend_from_slice(&json);
    Ok(bytes)
}
```

**Message Kinds:**
```rust
pub enum WireMessageKind {
    Request(WireRequest),      // { "type": "request", "method": "handshake", ... }
    Response(WireResponse),    // { "type": "response", "method": "handshake_ack", ... }
    Notification(WireNotification),  // { "type": "notification", "event": "agent_spawned", ... }
}
```

### Security Architecture (Security Layer 7: OFP Mutual Authentication)

**HMAC-SHA256 nonce-based mutual authentication.**

#### Handshake Flow

```
Client                              Server
  |                                   |
  |--- Handshake (nonce_a, HMAC) ---->|
  |    auth_hmac = HMAC(secret,      |
  |      nonce_a + node_id_a)         |
  |                                   |  [Verify HMAC, check nonce replay]
  |                                   |  [Generate nonce_b, HMAC]
  |<-- HandshakeAck (nonce_b, HMAC) --|
  |    auth_hmac = HMAC(secret,       |
  |      nonce_b + node_id_b)         |
  |                                   |
  |  [Verify HMAC, check nonce_b     |
  |   replay, derive session_key]    |
  |                                   |
  |====== Authenticated Loop ========|
  |  session_key = HMAC(secret,       |
  |    nonce_a + nonce_b)             |
```

#### Nonce Replay Protection (`NonceTracker`)

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

Uses `DashMap` (from `dashmap` crate) for concurrent HashMap access.

#### Per-Message HMAC

```rust
// After handshake, all messages use session_key HMAC
// Format: [4-byte length][JSON body][64-hex HMAC]

pub async fn write_message_authenticated(writer, msg, session_key) {
    let json_bytes = serde_json::to_vec(msg)?;
    let mac = hmac_sign(session_key, &json_bytes);  // 64 hex chars
    let total_len = json_bytes.len() + mac_bytes.len();
    writer.write_all(&(total_len as u32).to_be_bytes()).await?;
    writer.write_all(&json_bytes).await?;
    writer.write_all(mac_bytes).await?;  // HMAC after JSON
}
```

#### Session Key Derivation

```rust
pub fn derive_session_key(shared_secret: &str, our_nonce: &str, their_nonce: &str) -> String {
    let data = format!("{}{}", our_nonce, their_nonce);
    hmac_sign(shared_secret, data.as_bytes())
}
// Note: Order matters! derive(secret, a, b) != derive(secret, b, a)
```

#### Mandatory Handshake Rejection

```rust
// SECURITY: Reject ALL non-Handshake messages before authentication
WireMessageKind::Request(WireRequest::Handshake { .. }) => { /* process */ }
_ => {
    warn!("OFP: rejected unauthenticated message from {}", addr);
    write_error(&mut writer, 401, "Authentication required: complete HMAC handshake first");
    return Err(WireError::HandshakeFailed("Rejected unauthenticated request".into()));
}
```

### PeerNode Architecture

```rust
pub struct PeerNode {
    config: PeerConfig,
    registry: PeerRegistry,
    local_addr: SocketAddr,
    start_time: Instant,
    nonce_tracker: NonceTracker,
    session_key: Mutex<Option<String>>,  // Derived after handshake
}
```

**Accept Loop Pattern:**
```rust
async fn accept_loop(listener: TcpListener, node: Arc<PeerNode>, registry, handle) {
    loop {
        match listener.accept().await {
            Ok((stream, addr)) => {
                tokio::spawn(async move {
                    Self::handle_inbound(stream, addr, &node, &registry, &*handle).await
                });
            }
            Err(e) => {
                error!("OFP: accept error: {}", e);
                tokio::time::sleep(Duration::from_secs(1)).await;
            }
        }
    }
}
```

### PeerRegistry

```rust
pub struct PeerRegistry {
    peers: Arc<RwLock<HashMap<String, PeerEntry>>>,
}

pub enum PeerState {
    Connected,
    Disconnected,  // Lost but not removed (eligible for reconnect)
}
```

**Key Methods:**
- `find_agents(query)` - Search by name, description, tags
- `all_remote_agents()` - All agents from all connected peers
- `add_agent()` / `remove_agent()` - Dynamic agent list updates

### Clever Solutions

1. **4-byte length prefix** - Simple framing, avoids scroll attacks
2. **OFV1 magic header** (in vault, but same pattern) - Format version detection
3. **Constant-time HMAC comparison** via `subtle::ConstantTimeEq`
4. **Peer state Disconnected vs Removed** - Allows reconnect without re-registration
5. **Notification broadcasting** with fresh per-message HMAC

### Technical Debt / Concerns

1. **`consolidation.rs` listed in feature index but doesn't exist** - Either planned but not implemented, or misnamed in docs
2. **No automatic peer rediscovery** - If peers go down, no automatic reconnect attempts shown
3. **Session key stored in `Mutex<Option<String>>`** - Not used after initial derivation (marked `#[allow(dead_code)]`)
4. **Broadcast notifications open new connections** - Each broadcast creates a fresh TCP connection per peer, rather than reusing established connections

---

## Feature 12: Credential Vault & OAuth2 PKCE

**Priority:** Secondary
**Crate:** `crates/openfang-extensions`
**Files:** `vault.rs`, `oauth.rs`, `credentials.rs`

### Credential Vault (`vault.rs`)

**Purpose:** AES-256-GCM encrypted secret storage at `~/.openfang/vault.enc`

#### On-Disk Format

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

#### Key Derivation

```rust
fn derive_key(master_key: &[u8; 32], salt: &[u8]) -> ExtensionResult<Zeroizing<[u8; 32]>> {
    let mut derived = Zeroizing::new([0u8; 32]);
    Argon2::default()
        .hash_password_into(master_key, salt, derived.as_mut())
        .map_err(|e| ExtensionError::Vault(format!("Key derivation failed: {e}")))?;
    Ok(derived)
}
```

- **Master Key:** 32 bytes, stored in OS keyring or `OPENFANG_VAULT_KEY` env var
- **Derived Key:** Argon2id(master_key, salt) -> 32 bytes for AES-256-GCM
- **Each save:** Fresh random salt + nonce (SALT_LEN=16, NONCE_LEN=12)

#### OS Keyring Implementation (File-Based Fallback)

The code documents that true OS keyring was not implemented:

```rust
// In production, we'd use the `keyring` crate. Since it's an optional
// heavy dependency, we use a file-based fallback that's still better
// than plaintext env vars.

let keyring_path = dirs::data_local_dir()
    .unwrap_or_else(std::env::temp_dir)
    .join("openfang")
    .join(".keyring");

// Store encrypted with machine-specific key (XOR obfuscation with SHA256 mask)
let machine_id = machine_fingerprint();  // USER + HOSTNAME + "openfang-vault-v1"
let mask: Vec<u8> = hasher.finalize().to_vec();
let obfuscated: Vec<u8> = key_bytes.iter()
    .enumerate()
    .map(|(i, b)| b ^ mask[i % mask.len()])
    .collect();
```

**Technical Debt:** The XOR obfuscation with SHA256 mask is reversible with machine fingerprint knowledge. This is acknowledged as "better than plaintext env vars" but not true keyring security.

#### Vault Initialization Flow

```rust
pub fn init(&mut self) -> ExtensionResult<()> {
    // 1. Check for existing master key (env or keyring)
    let key_bytes = if let Ok(existing_b64) = std::env::var(VAULT_KEY_ENV) {
        decode_master_key(&existing_b64)?
    } else if let Ok(existing_b64) = load_keyring_key() {
        decode_master_key(&existing_b64)?
    } else {
        // Generate new key, store in keyring, print to stderr
        let mut kb = Zeroizing::new([0u8; 32]);
        OsRng.fill_bytes(kb.as_mut());
        // ...
        eprintln!("Vault key (save this as {}): {}", VAULT_KEY_ENV, key_b64.as_str());
        kb
    };
    // 2. Create empty vault file
    self.save(&key_bytes)?;
}
```

#### Memory Safety

```rust
pub struct CredentialVault {
    entries: HashMap<String, Zeroizing<String>>,  // Zeroized on drop
    cached_key: Option<Zeroizing<[u8; 32]>>,     // Zeroized on drop
}

impl Drop for CredentialVault {
    fn drop(&mut self) {
        self.entries.clear();
        self.cached_key = None;
        self.unlocked = false;
    }
}
```

#### Legacy Format Support

```rust
fn load(&mut self, master_key: &[u8; 32]) -> ExtensionResult<()> {
    let raw = std::fs::read(&self.path)?;
    let content = if raw.starts_with(VAULT_MAGIC) {  // "OFV1"
        std::str::from_utf8(&raw[VAULT_MAGIC.len()..])?
    } else if raw.first() == Some(&b'{') {
        // Legacy JSON vault (no magic header)
        std::str::from_utf8(&raw)?
    } else {
        return Err(ExtensionError::Vault("Unrecognized vault file format".to_string()));
    };
    // ...
}
```

### Credential Resolver (`credentials.rs`)

**Purpose:** Unified credential resolution chain

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

**Key Methods:**
- `resolve()` - Single credential
- `resolve_all()` - Multiple credentials at once
- `missing_credentials()` - Find missing keys
- `store_in_vault()` / `remove_from_vault()`
- `clear_dotenv_cache()` - For dashboard-driven deletions

#### Dotenv Parser

Custom dotenv parser handles:
- Comments (`# ...`)
- Empty lines
- Quoted values (both `"` and `'`)
- Key = value pairs

### OAuth2 PKCE (`oauth.rs`)

**Purpose:** Localhost callback OAuth2 flows for Google, GitHub, Microsoft, Slack

#### PKCE Flow Implementation

```rust
fn generate_pkce() -> PkcePair {
    let mut bytes = [0u8; 32];
    rand::rngs::OsRng.fill_bytes(&mut bytes);
    let verifier = Zeroizing::new(base64_url_encode(&bytes));
    let challenge = {
        let mut hasher = Sha256::new();
        hasher.update(verifier.as_bytes());
        base64_url_encode(&hasher.finalize())
    };
    PkcePair { verifier, challenge }
}
// S256 method: challenge = base64url(SHA256(verifier))
```

**State Parameter (CSRF Protection):**
```rust
fn generate_state() -> String {
    let mut bytes = [0u8; 16];
    rand::rngs::OsRng.fill_bytes(&mut bytes);
    base64_url_encode(&bytes)
}
```

#### Authorization URL Construction

```rust
let auth_url = format!(
    "{}?client_id={}&redirect_uri={}&response_type=code&scope={}&state={}&code_challenge={}&code_challenge_method=S256",
    oauth.auth_url,
    urlencoding_encode(client_id),
    urlencoding_encode(&redirect_uri),
    urlencoding_encode(&scopes),
    urlencoding_encode(&state),
    urlencoding_encode(&pkce.challenge),
);
```

#### Localhost Callback Server

```rust
// Random port binding
let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await?;
let port = listener.local_addr()?.port();

// Axum router for /callback
axum::Router::new().route("/callback", axum::routing::get(move |query: Query<CallbackParams>| {
    async move {
        if query.state != expected_state {
            return Html("<h1>Error</h1><p>Invalid state parameter. Possible CSRF attack.</p>");
        }
        if let Some(code) = query.code {
            let _ = code_tx.send(code);
        }
        Html("<h1>Success!</h1><script>window.close()</script>")
    }
}));

// 5-minute timeout
let code = tokio::time::timeout(Duration::from_secs(300), code_rx).await?;
```

#### Token Exchange

```rust
let client = reqwest::Client::new();
let mut params = HashMap::new();
params.insert("grant_type", "authorization_code");
params.insert("code", &code);
params.insert("redirect_uri", &redirect_uri);
params.insert("client_id", client_id);
params.insert("code_verifier", &verifier_str);  // Original verifier, not challenge

let resp = client.post(&oauth.token_url).form(&params).send().await?;
```

#### Client ID Resolution

```rust
pub fn resolve_client_ids(config: &OAuthConfig) -> HashMap<String, String> {
    let defaults = default_client_ids();  // Placeholder "openfang-*-client-id" values
    // Override with config values if provided
    if let Some(ref id) = config.google_client_id {
        resolved.insert("google".into(), id.clone());
    }
    // ...
}
```

#### Clever Solutions

1. **Zeroizing for all tokens** - Both access_token and refresh_token use `Zeroizing<String>`
2. **Configurable client IDs** - Defaults are placeholders, real IDs come from config
3. **Browser fallback** - If browser can't open, prints URL to stderr
4. **Graceful timeout** - 5 minutes足够用户完成OAuth

#### Technical Debt / Concerns

1. **Keyring is file-based XOR, not true OS keyring** - Acknowledged in code comments
2. **No token refresh logic** - OAuth flow returns tokens but no automatic refresh
3. **No PKCE code verifier validation library** - Custom URL encoding may have edge cases
4. **Default client IDs are obvious placeholders** - Could confuse users

---

## Cross-Feature Analysis

### Security Integration Points

| Layer | Feature | Mechanism |
|-------|---------|-----------|
| Security Layer 5 | MCP SSE | SSRF check (rejects metadata URLs) |
| Security Layer 6 | Vault | Zeroizing<String> for all secrets |
| Security Layer 7 | OFP | HMAC-SHA256 nonce mutual auth |
| Security Layer 7 | OFP | Per-message HMAC after handshake |
| Security Layer 11 | MCP Stdio | env_clear() + selective passthrough |

### Shared Patterns

1. **Zeroizing for sensitive data** - Vault and OAuth both use `Zeroizing<String>`
2. **Bounded storage with eviction** - A2aTaskStore (1000 max) and PeerRegistry (unbounded but marks disconnected)
3. **Platform-specific fallbacks** - OAuth browser open (Windows cmd, macOS open, Linux xdg-open)

### Feature Interactions

- MCP tools -> Agent execution -> A2A communication (cross-agent coordination)
- Credential Vault -> MCP server env vars -> MCP subprocess
- OFP peer registry -> A2A agent discovery -> External A2A agents
- OAuth tokens -> Credential vault storage -> Integration auth

---

## Verification: Feature Index Claims vs. Actual Implementation

| Claim | Status | Evidence |
|-------|--------|----------|
| MCP server implementation | Partial | `mcp_server.rs` - tool execution is stubbed |
| MCP client (connect to MCP servers) | Implemented | `mcp.rs` - full stdio + SSE transport |
| A2A Agent Cards | Implemented | `a2a.rs` - full protocol types |
| A2A task lifecycle | Implemented | `A2aTaskStore` with status transitions |
| OFP HMAC-SHA256 auth | Implemented | `peer.rs` with nonce replay protection |
| OFP per-message HMAC | Implemented | `write_message_authenticated()` |
| OFP PeerRegistry | Implemented | `registry.rs` - thread-safe HashMap |
| Credential Vault AES-256-GCM | Implemented | `vault.rs` - Argon2id key derivation |
| Vault OS keyring | Partial | File-based XOR obfuscation (not true keyring) |
| OAuth2 PKCE | Implemented | `oauth.rs` - S256 method, localhost callback |
| Credential resolution chain | Implemented | `credentials.rs` - vault > dotenv > env > prompt |
