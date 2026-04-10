# Moltis Architecture Analysis

## Overview

Moltis is a **Rust monorepo** (~52 crates) implementing a personal AI gateway. It combines an HTTP server, multi-channel messaging (Discord, Slack, Telegram, WhatsApp, MS Teams), LLM provider abstraction, agent orchestration, skills system, and a plugin/sandbox architecture for tool execution.

## Architectural Pattern

### Layered Architecture with Service Traits

Moltis uses a **layered architecture** with clear separation between transport, business logic, and domain concerns:

```
┌─────────────────────────────────────────────────────────┐
│                    Transport Layer                        │
│  apps/macos (Swift)  |  apps/ios (Swift)  |  website/   │
│         ↓                 ↓                    ↓         │
│              crates/httpd (Axum HTTP/WebSocket)          │
│         ↓                 ↓                    ↓         │
│              crates/gateway (Core Logic)                 │
│              crates/agents (Agent Runtime)               │
│              crates/channels (Plugin System)              │
│              crates/providers (LLM Abstraction)          │
│              crates/tools (Execution/Sandbox)            │
│              crates/skills (Skill Discovery)             │
│              crates/service-traits (Trait Interfaces)     │
└─────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **Transport Independence**: HTTP transport (`httpd`) depends on `gateway`, but not vice versa. Non-HTTP consumers (TUI, tests) depend on `gateway` directly.

2. **Dependency Rule**: Low-level crates have no dependencies on high-level ones. `gateway` knows about `agents`, `channels`, `providers`. `agents` knows about `providers`. But `providers` knows nothing about `gateway`.

3. **No Cyclic Dependencies**: Crates are organized in a strict DAG within the workspace.

## Core Components

### 1. Gateway (`crates/gateway`)

The **central hub** of the system. `server.rs` (193KB) is the largest file, handling:

- **Session Management**: `session.rs` - conversation sessions with message history
- **Channel Coordination**: `channel.rs` - routes messages between channels and agents
- **Broadcast System**: `broadcast.rs` - event distribution to connected clients
- **State Management**: `state.rs` - global gateway state including connected clients

```
Gateway State (AppState)
├── auth: Arc<ResolvedAuth>
├── sessions: Arc<SessionStore>
├── services: GatewayServices (bundle of trait objects)
├── channel_registry: ChannelRegistry
├── node_registry: NodeRegistry
├── skill_registry: Arc<dyn SkillRegistry>
└── mcp_hosts: HashMap
```

**Key Pattern**: `GatewayServices` bundle (from `service-traits`) holds `Arc<dyn Trait>` for all domain services, allowing loose coupling.

### 2. HTTP/D Web Server (`crates/httpd`)

Axum-based HTTP server providing:

- **WebSocket Transport**: For real-time client-server communication
- **Auth Middleware**: `auth_middleware.rs` - request authentication
- **RPC Handlers**: JSON-RPC style over WebSocket
- **REST Routes**: For channel webhooks, file uploads, GraphQL

```
HTTP Routes:
├── /ws           → WebSocket upgrade (protocol v3/v4)
├── /api/auth/*   → Authentication endpoints
├── /api/channels/* → Channel webhook endpoints
├── /api/gon      → Server-injected data (gon pattern)
└── /graphql      → GraphQL API (optional)
```

### 3. Channel Plugin System (`crates/channels`)

**Plugin Architecture** for messaging platforms:

```
ChannelPlugin (trait)
├── start_account(account_id, config)
├── stop_account(account_id)
├── outbound() → dyn ChannelOutbound
├── status() → dyn ChannelStatus
└── event_sink → dyn ChannelEventSink

ChannelEventSink (trait)
├── emit(channel_event)
├── dispatch_to_chat(text, reply_to, meta)
├── dispatch_command(command, reply_to)
└── request_disable_account(...)

InboundMode enum
├── Polling      (Telegram - long poll)
├── GatewayLoop  (Discord, WhatsApp - persistent WS)
├── SocketMode   (Slack - SocketMode connection)
└── Webhook      (MS Teams - HTTP callbacks)
```

**Channel Implementations**:
- `crates/discord` - Serenity-based bot with handlers
- `crates/slack` - slack-morphism for Slack API
- `crates/telegram` - teloxide bot framework
- `crates/whatsapp` - WhatsApp web API via waputo
- `crates/msteams` - MS Teams webhook receiver

**Key Design**: Each channel plugin is self-contained with its own bot client. The `ChannelEventSink` bridge allows channels to emit events back to the gateway for routing to agents.

### 4. Agent System (`crates/agents`)

**Agent Runtime** for LLM interaction:

```
Agent Run Flow (runner.rs):
1. Build prompt from system + messages + tools
2. Select provider (provider chain)
3. Stream completion from LLM
4. Parse tool calls from response
5. Execute tools (via tool registry)
6. Loop until no more tool calls or max iterations
7. Return final response
```

**Key Types** (`model.rs`):
```rust
// Typed messages (no LLM-specific fields leak in)
ChatMessage::System { content }
ChatMessage::User { content: UserContent }
ChatMessage::Assistant { content, tool_calls }
ChatMessage::Tool { tool_call_id, content }

// LLM Provider trait
trait LlmProvider {
    fn name(&self) -> &str;
    fn id(&self) -> &str;
    async fn complete(&self, messages, tools) -> CompletionResponse;
    fn stream(&self, messages, tools) -> Stream<StreamEvent>;
    fn supports_tools(&self) -> bool;
    fn supports_vision(&self) -> bool;
}
```

**Agent Submodules**:
- `model.rs` - ChatMessage types, LlmProvider trait
- `runner.rs` - Main agent loop with retry logic
- `prompt.rs` - System prompt construction
- `provider_chain.rs` - Multi-provider fallback
- `tool_registry.rs` - Available tools for agent
- `tool_parsing.rs` - Parse tool calls from LLM output
- `multimodal.rs` - Image/audio handling

### 5. Provider System (`crates/providers`)

**LLM Provider Abstraction** with multiple implementations:

```
Providers (feature-gated)
├── openai.rs          → OpenAI Chat Completions API
├── anthropic.rs       → Anthropic Messages API (Claude)
├── openai_compat.rs   → OpenAI-compatible endpoints
├── github_copilot.rs  → GitHub Copilot
├── kimi_code.rs       → Kimi Code
├── genai_provider.rs  → Google AI (Gemini)
├── async_openai_provider.rs → async-openai wrapper
└── local_gguf/       → llama.cpp bindings
```

**Provider Features**:
- **Streaming**: All providers support streaming responses
- **Tool Calling**: OpenAI and Anthropic native, others via compatibility layers
- **Model Discovery**: Fetch model lists from provider APIs
- **Reasoning Effort**: Claude-specific reasoning@ suffix support

**Key Pattern**: Namespaced model IDs (`provider::model-id`) for uniqueness across providers.

### 6. Tools System (`crates/tools`)

**Tool Execution** with sandbox isolation:

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

**Policy Layers** (`approval.rs`):
1. Global allow/deny (config)
2. Per-agent policy
3. Per-provider policy
4. Per-sender policy
5. Sandbox enforcement

**Key Design**: `ApprovalManager` handles user approval workflows for dangerous operations.

### 7. Skills System (`crates/skills`)

**Agent Skills** following the Agent Skills open standard:

```
Skill = Directory with SKILL.md
├── YAML frontmatter (name, description, triggers)
└── Markdown instructions

SkillDiscovery → SkillRegistry → SkillParser → Prompt injection
```

**Skill Lifecycle**:
1. Discover from filesystem, git repos, or registry
2. Parse SKILL.md frontmatter
3. Install dependencies
4. Load into agent prompt at runtime

### 8. Protocol (`crates/protocol`)

**WebSocket Protocol v4** definitions:

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

**Key Features**:
- Protocol version negotiation (v3/v4 compatible)
- Stream multiplexing via `stream` field
- Channel multiplexing for multi-session support
- Extensions namespace for Moltis-specific data

### 9. Service Traits (`crates/service-traits`)

**Domain Service Interfaces** with Noop defaults:

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

**Pattern**: Each trait has a `Noop*` implementation returning empty/default responses, allowing the gateway to start without all services configured.

## Communication Patterns

### 1. Direct Method Calls

Within the same process, crates call each other via traits:
```rust
// Gateway calls agent
services.agent.run(params).await

// Agent calls provider
provider.complete(messages, tools).await
```

### 2. WebSocket Frames

Client-gateway communication uses JSON frames:
```rust
// Client → Gateway
{ "type": "req", "id": "1", "method": "chat.send", "params": {...} }

// Gateway → Client (response)
{ "type": "res", "id": "1", "ok": true, "payload": {...} }

// Gateway → Client (event)
{ "type": "event", "event": "chat", "payload": {...}, "seq": 1 }
```

### 3. Channel Events (Internal Pub/Sub)

Channels emit events via `ChannelEventSink`:
```rust
ChannelEvent::InboundMessage { channel_type, account_id, peer_id, ... }
ChannelEvent::OtpChallenge { ... }
ChannelEvent::PairingQrCode { ... }
```

### 4. Session Event Bus

Cross-session notifications via `SessionEventBus`:
```rust
SessionEvent::MessageAdded { session_key, message_id }
SessionEvent::RunStarted { session_key, run_id }
SessionEvent::RunCompleted { session_key, run_id, usage }
```

## Design Patterns in Use

### 1. Plugin/Strategy Pattern
Channels, tools, and providers all implement shared traits, allowing runtime discovery and composition.

### 2. Builder Pattern
`ExecOpts`, `ChannelDescriptor`, `ChatMessage` all use builder-style construction with `Default`.

### 3. Registry Pattern
`ChannelRegistry`, `SkillRegistry`, `ToolRegistry` provide centralized lookup and management.

### 4. Noop/Stub Pattern
All service traits have `Noop*` implementations for graceful degradation.

### 5. Event Sourcing Ready
Messages stored in `message_log` table, session history persisted, enabling audit trails.

### 6. Feature Gating
Most provider implementations are feature-gated to avoid compile-time bloat:
```toml
[features]
default = ["provider-openai", "provider-anthropic"]
provider-github-copilot = ["..."]
```

### 7. Static Dispatch Where Hot, Dynamic Where Heterogeneous
- Hot paths use generics (`Arc<T>`)
- Plugin registries use `dyn Trait` for runtime polymorphism

## Module Map

| Crate | Responsibility |
|-------|---------------|
| `gateway` | Core server, session management, channel coordination |
| `httpd` | HTTP/WebSocket transport layer |
| `agents` | Agent runtime, prompt building, tool parsing |
| `providers` | LLM provider implementations |
| `channels` | Channel plugin trait, event types |
| `discord/slack/telegram/etc` | Platform-specific channel implementations |
| `tools` | Tool execution, sandboxing, policy |
| `skills` | Skill discovery, parsing, registry |
| `protocol` | WebSocket protocol frame definitions |
| `service-traits` | Domain service trait interfaces |
| `config` | Configuration schema and validation |
| `sessions` | Session persistence |
| `memory` | Vector embeddings for semantic search |
| `mcp` | Model Context Protocol integration |
| `graphql` | GraphQL API layer |
| `auth` | Password + passkey authentication |
| `vault` | Secrets storage |
| `cron` | Job scheduling |
| `oauth` | OAuth 2.0 flows |

## Key Architectural Decisions

1. **SQLite for Persistence**: All data stored in SQLite via sqlx with per-crate migrations. No external database dependency.

2. **TOML Configuration**: Single `moltis.toml` with schema validation at startup.

3. **No Built-in Auth Provider**: Relies on passkey (WebAuthn) and password. Integration with external identity providers via OAuth.

4. **WASM for Extensibility**: Guest tools compiled to WASM for sandboxed execution across platforms.

5. **SSH for Node Communication**: Remote nodes connected via SSH for command execution, avoiding additional protocol complexity.

6. **Dolt for Issue Tracking**: `.beads/` directory uses Dolt (git-for-data) for issue tracking with SQL queries.
