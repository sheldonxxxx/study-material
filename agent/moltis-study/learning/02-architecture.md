# Moltis Architecture

## Overview

Moltis is a personal AI gateway implemented as a **Rust monorepo** with 52 workspace crates and 3 applications. The system combines an HTTP/WebSocket server, multi-channel messaging integration, LLM provider abstraction, agent orchestration, skills system, and sandboxed tool execution.

```
moltis/
├── Cargo.toml           # Workspace definition
├── apps/                 # Applications (macOS, iOS, Courier)
├── crates/               # 52 library crates
├── website/              # Web UI (Preact + Tailwind)
├── wit/                  # WASM Interface Type definitions
└── .beads/               # Dolt issue tracking
```

## Architectural Pattern: Layered Architecture with Service Traits

```
┌─────────────────────────────────────────────────────────────┐
│                      Transport Layer                          │
│   apps/macos    │    apps/ios    │    website/    │  CLI   │
└────────┬────────┴────────┬────────┴────────┬──────┴────────┘
         │                 │                  │
         └──────────────────┼──────────────────┘
                            ↓
              ┌─────────────────────────────┐
              │     crates/httpd (Axum)      │
              │   HTTP Server + WebSocket    │
              └──────────────┬────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│                     Gateway Layer                            │
│  Session │ Channels │ Agents │ Providers │ Skills │ MCP   │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Rule

```
Lower-level crates have zero dependencies on higher-level crates:

gateway ──────────────→ agents ──────────────→ providers
      ──────────────→ channels ──────────────→ (nothing above)
      ──────────────→ skills
      ──────────────→ tools

providers knows nothing about gateway, agents, or channels
agents knows about providers, skills, tools
gateway knows about agents, channels, providers, skills, tools
```

## Core Module Responsibilities

### Gateway (`crates/gateway`)

The **central hub** maintaining application state and coordinating all subsystems.

```rust
// Gateway State (AppState)
pub struct AppState {
    pub auth: Arc<ResolvedAuth>,              // Authentication resolver
    pub sessions: Arc<SessionStore>,           // Conversation sessions
    pub services: GatewayServices,            // Trait object bundle
    pub channel_registry: ChannelRegistry,    // Active channel plugins
    pub node_registry: NodeRegistry,           // Remote node connections
    pub skill_registry: Arc<dyn SkillRegistry>, // Discovered skills
    pub mcp_hosts: HashMap<McpHostId, McpHost>, // MCP server hosts
}
```

**Key files:**
- `server.rs` (193KB) - Main server implementation
- `session.rs` - Session management with message history
- `channel.rs` - Routes messages between channels and agents
- `broadcast.rs` - Event distribution to connected clients
- `state.rs` - Global gateway state

### HTTP Daemon (`crates/httpd`)

Axum-based HTTP/WebSocket server providing transport independence.

```
Routes:
├── /ws           → WebSocket upgrade (protocol v3/v4)
├── /api/auth/*   → Authentication endpoints
├── /api/channels/* → Channel webhook endpoints
├── /api/gon      → Server-injected data (gon pattern)
└── /graphql      → GraphQL API (optional)
```

**Key files:**
- `router.rs` - Route definitions
- `auth_middleware.rs` - Request authentication
- `websocket.rs` - WebSocket protocol handlers

### Channel Plugin System (`crates/channels`)

Plugin architecture for messaging platforms with self-contained implementations.

```
ChannelPlugin (trait)
├── start_account(account_id, config)
├── stop_account(account_id)
├── outbound() → Arc<dyn ChannelOutbound>
├── status() → Arc<dyn ChannelStatus>
└── event_sink → ChannelEventSink

InboundMode enum:
├── Polling      (Telegram - long poll)
├── GatewayLoop  (Discord, WhatsApp - persistent WS)
├── SocketMode   (Slack - SocketMode connection)
└── Webhook      (MS Teams - HTTP callbacks)
```

**Channel implementations:**
| Crate | Platform | Framework |
|-------|----------|-----------|
| `crates/discord` | Discord | Serenity |
| `crates/slack` | Slack | slack-morphism |
| `crates/telegram` | Telegram | teloxide |
| `crates/whatsapp` | WhatsApp | waputo |
| `crates/msteams` | MS Teams | webhook receiver |

### Agent System (`crates/agents`)

Agent runtime orchestrating LLM interactions with tool execution loops.

```
Agent Run Flow (runner.rs):
1. Build prompt from system + messages + tools
2. Select provider via provider chain
3. Stream completion from LLM
4. Parse tool calls from response
5. Execute tools via tool registry
6. Loop until no more tool calls or max iterations
7. Return final response
```

**Key types (`model.rs`):**
```rust
// Typed messages (no LLM-specific fields leak through)
ChatMessage::System { content: String }
ChatMessage::User { content: UserContent }
ChatMessage::Assistant { content: String, tool_calls: Option<Vec<ToolCall>> }
ChatMessage::Tool { tool_call_id: String, content: String }

// Provider trait
trait LlmProvider {
    fn name(&self) -> &str;
    fn id(&self) -> &str;
    async fn complete(&self, messages: &[ChatMessage], tools: &[Tool]) -> CompletionResult;
    fn stream(&self, messages: &[ChatMessage], tools: &[Tool]) -> Stream<StreamEvent>;
    fn supports_tools(&self) -> bool;
    fn supports_vision(&self) -> bool;
}
```

### Provider System (`crates/providers`)

LLM provider abstraction with feature-gated implementations.

```
Feature-gated providers:
├── openai.rs              → OpenAI Chat Completions API
├── anthropic.rs           → Anthropic Messages API (Claude)
├── openai_compat.rs       → OpenAI-compatible endpoints
├── github_copilot.rs      → GitHub Copilot
├── kimi_code.rs           → Kimi Code
├── genai_provider.rs      → Google AI (Gemini)
├── async_openai_provider.rs → async-openai wrapper
└── local_gguf/           → llama.cpp bindings
```

**Key design**: Namespaced model IDs (`provider::model-id`) ensure uniqueness across providers.

### Tools System (`crates/tools`)

Tool execution with sandbox isolation and multi-layer approval.

```
Tool Types:
├── ExecTool     → Shell command execution
├── WebFetchTool → HTTP requests (SSRF protected)
├── WebSearchTool→ Web search capability
├── BrowserTool  → Headless Chrome automation
├── CronTool     → Scheduled job execution
├── WasmTool     → WASM component execution
└── SkillTool    → Skill invocation

Sandbox Implementations:
├── Docker sandbox    → containers.rs
├── Apple sandbox     → apple.rs (macOS/iOS)
├── Host sandbox      → Direct exec (with policy)
└── WASM runtime      → wasmtime component model
```

**Approval layers (`approval.rs`):**
1. Global allow/deny (config)
2. Per-agent policy
3. Per-provider policy
4. Per-sender policy
5. Sandbox enforcement

### Skills System (`crates/skills`)

Agent skills following the Agent Skills open standard.

```
Skill = Directory with SKILL.md
├── YAML frontmatter (name, description, triggers)
└── Markdown instructions

Skill Lifecycle:
1. Discover from filesystem, git repos, or registry
2. Parse SKILL.md frontmatter
3. Install dependencies
4. Load into agent prompt at runtime
```

### Protocol (`crates/protocol`)

WebSocket protocol v4 frame definitions.

```
Frame Types:
├── RequestFrame  { type: "req", id, method, params, channel }
├── ResponseFrame { type: "res", id, ok, payload/error }
└── EventFrame    { type: "event", event, payload, seq, stream, done }

Protocol Flow:
1. Client connects with ConnectParams (protocol version negotiation)
2. Gateway responds with HelloOk (server info, features, policy)
3. Bidirectional JSON frames over WebSocket
4. Events for server-push (tick, chat, presence, health)
```

### Service Traits (`crates/service-traits`)

Domain service interfaces with Noop defaults for graceful degradation.

```rust
Services {  // Bundle of all domain services
    agent: Arc<dyn AgentService>,
    session: Arc<dyn SessionService>,
    channel: Arc<dyn ChannelService>,
    config: Arc<dyn ConfigService>,
    chat: Arc<dyn ChatService>,
    skills: Arc<dyn SkillsService>,
    mcp: Arc<dyn McpService>,
    browser: Arc<dyn BrowserService>,
    // ... 20+ service traits
}
```

## Communication Patterns

### 1. Direct Method Calls (Same Process)

Crate-to-crate communication via traits:

```rust
// Gateway calls agent service
services.agent.run(params).await

// Agent calls provider
provider.complete(messages, tools).await

// Gateway calls channel
services.channel.send_message(msg).await
```

### 2. WebSocket Frames (Client-Gateway)

```json
// Client → Gateway
{ "type": "req", "id": "1", "method": "chat.send", "params": {...} }

// Gateway → Client (response)
{ "type": "res", "id": "1", "ok": true, "payload": {...} }

// Gateway → Client (event push)
{ "type": "event", "event": "chat", "payload": {...}, "seq": 1 }
```

### 3. Channel Events (Internal Pub/Sub)

Channels emit events via `ChannelEventSink`:

```rust
enum ChannelEvent {
    InboundMessage { channel_type, account_id, peer_id, content, timestamp },
    OtpChallenge { channel_type, account_id, phone, expires_at },
    PairingQrCode { channel_type, account_id, qr_code_data },
    AccountStatusChanged { channel_type, account_id, status },
}
```

### 4. Session Event Bus

Cross-session notifications:

```rust
enum SessionEvent {
    MessageAdded { session_key, message_id },
    RunStarted { session_key, run_id },
    RunCompleted { session_key, run_id, usage },
    PresenceChanged { session_key, presence },
}
```

## Key Architectural Decisions

### 1. SQLite for Persistence

All data stored in SQLite via sqlx with per-crate migrations.

```rust
// Per-crate migration pattern
sqlx::migrate!("./migrations")
```

**Rationale**: Zero external database dependency, single-file storage, sufficient for personal use scale.

### 2. TOML Configuration

Single `moltis.toml` with schema validation at startup.

```toml
[gateway]
host = "0.0.0.0"
port = 3030

[[channel]]
type = "discord"
token = "env:DISCORD_TOKEN"
```

### 3. No Built-in Auth Provider

Relies on passkey (WebAuthn) and password. External identity providers via OAuth.

```rust
// crates/auth
pub enum AuthMethod {
    Password { hash: String },
    Passkey { credential_id: Vec<u8>, authenticator_data: Vec<u8> },
    OAuth { provider: String, access_token: String },
}
```

### 4. WASM for Extensibility

Guest tools compiled to WASM for sandboxed execution across platforms.

```
wit/
├── exec.wit    → Tool execution interface
├── http.wit    → HTTP fetch interface
└── wasi.wit    → WASI interfaces
```

### 5. SSH for Node Communication

Remote nodes connected via SSH for command execution.

### 6. Dolt for Issue Tracking

`.beads/` directory uses Dolt (git-for-data) for issue tracking with SQL queries.

## Module Dependency Map

```
                    ┌─────────────────────────────────────────┐
                    │              apps/macos, ios           │
                    └────────────────────┬────────────────────┘
                                         │ swift-bridge
                    ┌────────────────────▼────────────────────┐
                    │              crates/httpd                │
                    │   (Axum HTTP/WebSocket transport)       │
                    └────────────────────┬────────────────────┘
                                         │ direct call
                    ┌────────────────────▼────────────────────┐
                    │              crates/gateway              │
                    │   (Central hub: sessions, routing)      │
                    └────────────────────┬────────────────────┘
                                         │
           ┌─────────────────────────────┼─────────────────────────────┐
           │                             │                             │
           ↓                             ↓                             ↓
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│    crates/agents    │   │   crates/channels   │   │   crates/providers  │
│  (Agent runtime)    │   │  (Channel trait)    │   │ (LLM abstraction)   │
└─────────┬───────────┘   └─────────┬───────────┘   └─────────┬───────────┘
          │                         │                         │
          ↓                         ↓                         │
┌─────────────────────┐   ┌─────────────────────┐            │
│    crates/skills    │   │  discord/slack/etc   │            │
│  (Skill discovery)  │   │ (Plugin impls)       │            │
└─────────────────────┘   └─────────────────────┘            │
          │                             │                    │
          └─────────────────────────────┼────────────────────┘
                                        ↓
                              ┌─────────────────────┐
                              │    crates/tools     │
                              │ (Sandboxed exec)    │
                              └─────────────────────┘
```
