# Feature Batch 3 Deep Dive: Communication Channels, Voice I/O, and MCP

**Date:** 2026-03-26
**Repository:** `/Users/sheldon/Documents/claw/reference/moltis`
**Source Files Analyzed:** `crates/channels/`, `crates/discord/`, `crates/telegram/`, `crates/whatsapp/`, `crates/msteams/`, `crates/slack/`, `crates/voice/`, `crates/mcp/`

---

## Feature 7: Communication Channels

### Overview

The communication channel system provides unified messaging integration across five platforms: **Discord, Telegram, WhatsApp, Microsoft Teams, and Slack**. All channels share a common plugin architecture defined in `crates/channels/`.

### Architecture

#### Core Abstraction: `ChannelPlugin` Trait

**File:** `crates/channels/src/plugin.rs` (lines 508-576)

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

#### Channel Type Enum

**File:** `crates/channels/src/plugin.rs` (lines 10-20)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub enum ChannelType {
    Telegram,
    Whatsapp,
    MsTeams,
    Discord,
    Slack,
}
```

#### Inbound Modes

Each channel has a specific inbound communication pattern:

| Channel | Mode | Implementation |
|---------|------|----------------|
| **Telegram** | `Polling` | Long-polling via `teloxide` library |
| **WhatsApp** | `GatewayLoop` | Persistent WebSocket via `whatsapp-rust` |
| **Discord** | `GatewayLoop` | Persistent WebSocket via `serenity` library |
| **Slack** | `SocketMode` | WebSocket via `slack-morphism` |
| **MS Teams** | `Webhook` | HTTP webhook endpoints with signature verification |

#### Capability Flags

**File:** `crates/channels/src/plugin.rs` (lines 185-198)

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

#### Channel Events

**File:** `crates/channels/src/plugin.rs` (lines 210-277)

Channels emit events for real-time UI updates:

```rust
pub enum ChannelEvent {
    InboundMessage { ... },
    AccountDisabled { ... },
    ReactionChange { ... },
    OtpChallenge { ... },          // OTP issued to non-allowlisted user
    OtpResolved { ... },            // OTP approved, locked out, or expired
    PairingQrCode { ... },          // WhatsApp Linked Devices QR
    PairingComplete { ... },
    PairingFailed { ... },
}
```

### Discord Implementation

**Crate:** `crates/discord/`
**Key Files:** `src/plugin.rs`, `src/handler.rs`

#### Gateway Connection Pattern

```rust
// DiscordPlugin::start_account spawns a serenity Client
let mut client = serenity::Client::builder(&token, required_intents())
    .event_handler(handler)
    .await;

// Required intents
pub fn required_intents() -> GatewayIntents {
    GatewayIntents::GUILD_MESSAGES
        | GatewayIntents::DIRECT_MESSAGES
        | GatewayIntents::MESSAGE_CONTENT
        | GatewayIntents::GUILDS
        | GatewayIntents::GUILD_MESSAGE_REACTIONS
        | GatewayIntents::DIRECT_MESSAGE_REACTIONS
}
```

#### Message Handling

**File:** `crates/discord/src/handler.rs` (lines 218-476)

1. Bot mentions are stripped using `strip_bot_mention()`
2. Access policy checked via `access::check_access()` (Allowlist/Deny based)
3. Slash commands (`/command`) dispatched to `sink.dispatch_command()`
4. Location coordinates extracted from text/URLs via `extract_location_coordinates()`
5. Reactions added as ack indicator during processing

#### Markdown-Aware Message Chunking

**File:** `crates/discord/src/handler.rs` (lines 754-810)

Discord's 2000-character limit is handled with a smart chunker that:
- Avoids splitting inside fenced code blocks (` ``` `)
- Prefers splitting at newlines outside fences
- Falls back to hard limit if no better boundary found

```rust
fn find_split_point(text: &str, max_len: usize) -> usize {
    // Tracks whether each newline is inside a fenced code block
    // Returns best split position respecting code fence boundaries
}
```

#### OTP Self-Approval Flow

**File:** `crates/discord/src/handler.rs` (lines 528-686)

For `dm_policy = Allowlist` with `otp_self_approval = true`:
1. First message: Issues 6-digit OTP challenge (code shown only in web UI)
2. Code reply: Verifies against stored code, auto-approves on match
3. Wrong code: Shows remaining attempts
4. Locked out: Too many failures, must wait

**Security:** `OTP_CHALLENGE_MSG` never contains the actual code.

### Telegram Implementation

**Crate:** `crates/telegram/`

Uses `teloxide` library for Telegram Bot API integration. Key file: `src/socket.rs` (WebSocket long-polling).

Features:
- Edit-in-place streaming support
- Markdown/HTML message formatting
- Voice message transcription via STT
- Inline keyboard buttons for interactive messages

### WhatsApp Implementation

**Crate:** `crates/whatsapp/`

Uses `whatsapp-rust` library for WhatsApp Web (Multi-Device API). Key files:
- `src/handler.rs` - Message handling
- `src/outbound.rs` - Message sending
- `src/sled_store.rs` - Local message persistence

Features:
- QR code device pairing ("Link a Device")
- OTP verification flow
- Voice note transcription
- Media message support (photos, videos, documents)

### Microsoft Teams Implementation

**Crate:** `crates/msteams/`

Bot Framework adapter with OAuth client credentials. Key files:
- `src/plugin.rs` - Main plugin logic
- `src/activity.rs` - Teams activity parsing
- `src/outbound.rs` - Message delivery

Uses HTTP webhooks with HMAC signature verification (`channel_webhook_middleware.rs`).

### Slack Implementation

**Crate:** `crates/slack/`

Uses `slack-morphism` library with Socket Mode. Key files:
- `src/handlers.rs` - Event handling
- `src/socket.rs` - WebSocket connection
- `src/outbound.rs` - Message sending

Features:
- Slash commands
- Interactive button callbacks
- Thread context support
- Reaction support

### Channel Registry

**File:** `crates/channels/src/registry.rs`

The registry provides O(1) account-to-plugin routing:

```rust
pub struct ChannelRegistry {
    plugins: HashMap<String, Arc<RwLock<dyn ChannelPlugin>>>,  // channel_type -> plugin
    account_index: StdRwLock<HashMap<String, String>>,         // account_id -> channel_type
}
```

### Access Control

**File:** `crates/channels/src/access.rs`

Channel-level sender allowlisting with DM vs Group policies:
- `DmPolicy::Allowlist` - Only allowlisted users can DM
- `DmPolicy::Anyone` - Anyone can DM
- `GroupPolicy` similar controls for group messages

### Key Design Patterns

1. **Builder Pattern:** Plugins use builder for `with_message_log()`, `with_event_sink()`
2. **Arc<RwLock<HashMap<...>>>:** Thread-safe account state management
3. **Event Sink:** Async channel for dispatching to chat session
4. **CancellationToken:** Graceful shutdown of background tasks
5. **Feature-gated Optional Dependencies:** Many features depend on feature flags

### Technical Debt / Concerns

1. **Secret Management:** Bot tokens stored as `Secret<String>` but serialized config contains redacted values
2. **Error Recovery:** No automatic reconnection logic for gateway disconnections
3. **Rate Limiting:** Not systematically handled across all channels
4. **Message Editing:** Discord supports edit-in-place, but Telegram does not

---

## Feature 8: Voice I/O (STT/TTS)

### Overview

The voice system provides **Speech-to-Text (STT)** and **Text-to-Speech (TTS)** capabilities through a provider-agnostic abstraction.

**Crate:** `crates/voice/`
**Key Modules:** `src/stt/` (9 providers), `src/tts/` (5 providers)

### STT Providers

**File:** `crates/voice/src/stt/mod.rs`

#### Trait Definition

```rust
#[async_trait]
pub trait SttProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn name(&self) -> &'static str;
    fn is_configured(&self) -> bool;
    async fn transcribe(&self, request: TranscribeRequest) -> Result<Transcript>;
}
```

#### Supported Providers

| Provider | File | Notes |
|----------|------|-------|
| **Whisper** | `stt/whisper.rs` | OpenAI's Whisper API |
| **Whisper CLI** | `stt/whisper_cli.rs` | Local `whisper.cpp` binary |
| **Deepgram** | `stt/deepgram.rs` | Cloud STT with punctuation |
| **ElevenLabs STT** | `stt/elevenlabs.rs` | Low-latency option |
| **Google STT** | `stt/google.rs` | Google Cloud Speech-to-Text |
| **Groq STT** | `stt/groq.rs` | Fast inference provider |
| **Mistral STT** | `stt/mistral.rs` | Mistral's STT offering |
| **SherpaOnnx** | `stt/sherpa_onnx.rs` | On-device/offline STT |
| **Voxtral Local** | `stt/voxtral_local.rs` | Local streaming STT |

#### Request/Response Types

```rust
pub struct TranscribeRequest {
    pub audio: Bytes,
    pub format: AudioFormat,
    pub language: Option<String>,
    pub prompt: Option<String>,  // Context for better transcription
}

pub struct Transcript {
    pub text: String,
    pub language: Option<String>,
    pub confidence: Option<f32>,
    pub duration_seconds: Option<f32>,
    pub words: Option<Vec<Word>>,  // Word-level timestamps
}
```

### TTS Providers

**File:** `crates/voice/src/tts/mod.rs`

#### Trait Definition

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

#### Supported Providers

| Provider | File | Notes |
|----------|------|-------|
| **ElevenLabs** | `tts/elevenlabs.rs` | High-quality, low-latency, voice cloning |
| **OpenAI TTS** | `tts/openai.rs` | TTS-1 and TTS-1-HD models |
| **Google TTS** | `tts/google.rs` | WaveNet and Neural2 voices |
| **Coqui** | `tts/coqui.rs` | Open-source TTS |
| **Piper** | `tts/piper.rs` | On-device TTS |

#### Audio Format Enum

```rust
pub enum AudioFormat {
    Mp3,    // Default, widely compatible
    Opus,   // Good for Telegram voice notes
    Aac,
    Pcm,
    Webm,
}
```

### Text Sanitization for TTS

**File:** `crates/voice/src/tts/mod.rs` (lines 201-538)

LLM output contains markdown that TTS should not read literally. The `sanitize_text_for_tts()` function:

1. Removes fenced code blocks
2. Removes URLs
3. Strips markdown headers
4. Strips bold/italic markers
5. Strips inline code backticks
6. Strips bullet/list prefixes
7. Removes markdown tables
8. Strips SSML `<break>` tags
9. Collapses whitespace

```rust
pub fn sanitize_text_for_tts(text: &str) -> Cow<'_, str> {
    // Zero-allocation fast path when no sanitization needed
    if !needs_sanitization(text) {
        return Cow::Borrowed(text);
    }
    // Process line by line...
}
```

**Example:**
```rust
// Input
"## Getting Started\n\n1. Install from https://rustup.rs/\n2. Run `cargo build`"

// Output
"Getting Started\n\nInstall from\nRun cargo build"
```

### SSML Handling

```rust
pub fn contains_ssml(text: &str) -> bool {
    text.contains("<break")
}

pub fn strip_ssml_tags(text: &str) -> Cow<'_, str> {
    // Removes <break time="0.5s"/> style tags
}
```

Providers declare `supports_ssml()` - if false, tags are stripped before synthesis.

### Configuration

**File:** `crates/voice/src/config.rs`

```rust
pub struct VoiceConfig {
    pub stt_provider: SttProviderId,
    pub tts_provider: TtsProviderId,
    pub tts_auto_mode: TtsAutoMode,
    // Provider-specific configs...
}
```

### Integration with Channels

Channels call `sink.transcribe_voice()` for voice messages:
```rust
async fn transcribe_voice(&self, audio_data: &[u8], format: &str) -> Result<String>;
```

### Key Design Patterns

1. **Zero-Copy Fast Path:** `Cow<'_, str>` for text sanitization
2. **Provider Agnostic:** `SttProvider`/`TtsProvider` traits allow swapping implementations
3. **Audio Format Abstraction:** `AudioFormat` with MIME type/extension mapping
4. **Streaming Support:** Voxtral local provider supports streaming transcription

### Technical Debt / Concerns

1. **No Audio Format Negotiation:** Channel sends format, STT must accept
2. **Provider-Specific Features:** Some features (SSML, voice cloning) only available on certain providers
3. **Local vs Cloud:** SherpaOnnx and Piper are on-device but may have quality/feature gaps

---

## Feature 9: MCP Server Support

### Overview

Model Context Protocol (MCP) client support enabling Moltis to connect to MCP-compatible tools and services.

**Crate:** `crates/mcp/`
**Key Files:** `src/lib.rs`, `src/traits.rs`, `src/transport.rs`, `src/sse_transport.rs`, `src/registry.rs`, `src/manager.rs`, `src/client.rs`

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      McpManager                         │
│  (lifecycle, registry, tool aggregation)               │
└───────────────────────┬─────────────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
         ▼              ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ McpClient│  │ McpClient│  │ McpClient│  (one per server)
   │ (stdio)  │  │  (SSE)   │  │  (SSE+OAuth)│
   └────┬─────┘  └────┬─────┘  └─────┬──────┘
        │             │              │
        ▼             ▼              ▼
   StdioTransport  SseTransport   SseTransport
   (child process) (HTTP/SSE)     + Auth
```

### Transport Layer

**File:** `crates/mcp/src/traits.rs` (lines 14-31)

```rust
#[async_trait]
pub trait McpTransport: Send + Sync {
    async fn request(&self, method: &str, params: Option<Value>) -> Result<JsonRpcResponse>;
    async fn notify(&self, method: &str, params: Option<Value>) -> Result<()>;
    async fn is_alive(&self) -> bool;
    async fn kill(&self);
}
```

#### StdioTransport

**File:** `crates/mcp/src/transport.rs`

Spawns a child process and communicates via JSON-RPC over stdin/stdout:

```rust
pub struct StdioTransport {
    child: Mutex<Child>,
    stdin: Mutex<tokio::process::ChildStdin>,
    pending: Arc<Mutex<HashMap<String, oneshot::Sender<JsonRpcResponse>>>>,
    next_id: AtomicU64,
    request_timeout: Duration,
    reader_handle: Mutex<Option<tokio::task::JoinHandle<()>>>,
}
```

**Key Implementation Details:**
- Lines 170-206: Pending request tracking with `oneshot` channels
- Lines 112-156: Background task reads stdout, dispatches responses by ID
- Lines 91-109: Stderr reader logs server errors
- Timeout on each request (configurable, default 30s)

#### SseTransport (HTTP/SSE)

**File:** `crates/mcp/src/sse_transport.rs`

HTTP/SSE transport for remote MCP servers implementing Streamable HTTP:

```rust
pub struct SseTransport {
    client: Client,
    request_url: Secret<String>,
    display_url: String,
    default_headers: HeaderMap,
    next_id: AtomicU64,
    auth: Option<SharedAuthProvider>,
    session_id: RwLock<Option<String>>,
}
```

**Key Implementation Details:**
- Session ID tracking for Streamable HTTP protocol (lines 155-176)
- Bearer token injection from auth provider (lines 145-153)
- 401 retry with OAuth re-authentication (lines 248-383)
- Event stream parsing for SSE responses (lines 200-230)
- `kill()` sends HTTP DELETE to close session (lines 497-530)

### Client Layer

**File:** `crates/mcp/src/client.rs`

```rust
pub struct McpClient {
    server_name: String,
    transport: Arc<dyn McpTransport>,
    state: McpClientState,
    server_info: Option<InitializeResult>,
    tools: Vec<McpToolDef>,
}

pub enum McpClientState {
    Connected,      // Transport spawned
    Ready,          // Initialize handshake complete
    Authenticating, // OAuth in progress
    Closed,
}
```

**Connection Methods:**
- `connect()` - Stdio transport with child process
- `connect_sse()` - Remote HTTP/SSE without auth
- `connect_sse_with_auth()` - Remote with OAuth

### MCP Protocol Types

**File:** `crates/mcp/src/types.rs`

```rust
// JSON-RPC 2.0 base types
pub struct JsonRpcRequest { jsonrpc, id, method, params }
pub struct JsonRpcResponse { jsonrpc, id, result?, error? }
pub struct JsonRpcNotification { jsonrpc, method, params }

// MCP protocol
pub struct InitializeParams { protocol_version, capabilities, client_info }
pub struct InitializeResult { protocol_version, capabilities, server_info }
pub struct McpToolDef { name, description?, input_schema }
pub struct ToolsCallResult { content: Vec<ToolContent>, is_error }
pub const PROTOCOL_VERSION: &str = "2024-11-05";
```

### Registry

**File:** `crates/mcp/src/registry.rs`

Persisted MCP server configuration:

```rust
pub struct McpServerConfig {
    pub command: String,
    pub args: Vec<String>,
    pub env: HashMap<String, String>,
    pub enabled: bool,
    pub request_timeout_secs: Option<u64>,
    pub transport: TransportType,  // Stdio or Sse
    pub url: Option<Secret<String>>,           // For SSE transport
    pub headers: HashMap<String, Secret<String>>,  // Custom headers
    pub oauth: Option<McpOAuthConfig>,
    pub display_name: Option<String>,
}

pub struct McpRegistry {
    pub servers: HashMap<String, McpServerConfig>,
}
```

**Secret Handling:** URL and header values use `Secret<String>` type with custom serde to redact in logs/serialization.

### Manager

**File:** `crates/mcp/src/manager.rs`

Lifecycle management for multiple MCP servers:

```rust
pub struct McpManager {
    inner: RwLock<McpManagerInner>,
    request_timeout_secs: AtomicU64,
}

pub struct McpManagerInner {
    clients: HashMap<String, Arc<RwLock<dyn McpClientTrait>>>,
    tools: HashMap<String, Vec<McpToolDef>>,
    registry: McpRegistry,
    auth_providers: HashMap<String, SharedAuthProvider>,
    env_overrides: HashMap<String, String>,
}
```

**Key Methods:**
- `start_enabled()` - Start all enabled servers from registry
- `start_server()` - Start single server with auth handling
- OAuth flow support for SSE servers

### OAuth Support

**File:** `crates/mcp/src/auth.rs`

```rust
#[async_trait]
pub trait McpAuthProvider: Send + Sync {
    async fn access_token(&self) -> Result<Option<Secret<String>>>;
    async fn handle_unauthorized(&self, www_authenticate: Option<&str>) -> Result<bool>;
    async fn start_oauth(&self, redirect_uri: &str, www_authenticate: Option<&str>) -> Result<Option<String>>;
    async fn complete_oauth(&self, state: &str, code: &str) -> Result<bool>;
    fn pending_auth_url(&self) -> Option<String>;
    fn auth_state(&self) -> McpAuthState;
}
```

### Tool Bridge

**File:** `crates/mcp/src/tool_bridge.rs`

Adapts MCP tools to the agent tool interface:

```rust
pub struct McpToolBridge {
    // Converts MCP tool definitions to agent tools
}

pub struct McpAgentTool {
    // Wraps an MCP tool for agent execution
}
```

### Error Handling

**File:** `crates/mcp/src/types.rs` (lines 167-208)

```rust
pub enum McpManagerError {
    OAuthRequired { server: String },
    OAuthStateNotFound,
    NotSseTransport { server: String },
    MissingSseUrl { server: String },
    ServerNotFound { server: String },
}

pub enum McpTransportError {
    Unauthorized { www_authenticate: Option<String> },
    HttpError { status: u16, body: String },
    Other { source: Box<dyn StdError + Send + Sync> },
}
```

### Key Design Patterns

1. **Trait Objects for Transport:** `Arc<dyn McpTransport>` allows swapping implementations
2. **Request ID Tracking:** Each JSON-RPC request has a unique ID with `oneshot::Sender` for response routing
3. **Secret Sanitization:** URL and headers use `Secret<String>` with custom serde
4. **Session ID Tracking:** For Streamable HTTP protocol compliance
5. **Graceful Shutdown:** `CancellationToken` pattern for stopping servers

### Testing

The transports have comprehensive tests:
- `StdioTransport`: Spawn/kill, timeout, request/response
- `SseTransport`: 401 handling, session propagation, header injection, OAuth override

### Technical Debt / Concerns

1. **No Tool Change Notifications:** MCP servers can signal tool list changes, not currently handled
2. **Limited Resource Support:** Resources and prompts MCP capabilities not implemented
3. **OAuth Flow Complexity:** Interactive OAuth requires browser, not suitable for headless servers

---

## Cross-Feature Observations

### Shared Patterns

1. **Error Enum Pattern:** Each crate defines `Error` enum with `thiserror`
2. **Async Trait Pattern:** `#[async_trait]` used extensively for `async` methods in traits
3. **Arc<RwLock<...>>:** Thread-safe shared state
4. **Builder Pattern:** For complex initialization (plugin configuration)
5. **Secret Handling:** `secrecy::Secret<String>` for sensitive data

### Integration Points

- **Channels -> Voice:** `sink.transcribe_voice()` for voice messages
- **Channels -> MCP:** Via tools - MCP tools can be called as channel tools
- **Voice -> Channels:** TTS output sent via `ChannelOutbound::send_media()`

### Security Considerations

1. **OTP codes never in messages:** Challenge messages don't contain codes
2. **Secret redaction:** URLs with API keys redacted in logs/serialization
3. **Webhook signature verification:** MS Teams uses HMAC
4. **SSRF protection:** HTTP client blocks private IP ranges

---

## File Summary

| File | Lines | Purpose |
|------|-------|---------|
| `crates/channels/src/plugin.rs` | 1047 | Channel trait definitions |
| `crates/channels/src/registry.rs` | 965 | Channel registry implementation |
| `crates/discord/src/handler.rs` | 1086 | Discord message handling |
| `crates/discord/src/plugin.rs` | 336 | Discord plugin implementation |
| `crates/mcp/src/transport.rs` | 283 | Stdio transport for MCP |
| `crates/mcp/src/sse_transport.rs` | 931 | HTTP/SSE transport for MCP |
| `crates/mcp/src/client.rs` | 500+ | MCP client implementation |
| `crates/mcp/src/registry.rs` | 348 | MCP server registry |
| `crates/mcp/src/manager.rs` | 400+ | MCP lifecycle manager |
| `crates/mcp/src/types.rs` | 275 | MCP protocol types |
| `crates/voice/src/stt/mod.rs` | 134 | STT trait and types |
| `crates/voice/src/tts/mod.rs` | 841 | TTS trait, types, sanitization |
| `crates/msteams/src/plugin.rs` | 400+ | MS Teams plugin |
| `crates/slack/src/handlers.rs` | 1000+ | Slack event handlers |
| `crates/whatsapp/src/handler.rs` | 900+ | WhatsApp message handling |
