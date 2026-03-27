# OpenFang Feature Index

## Core Features (Essential Functionality)

### 1. Autonomous "Hands" System
**Priority: Core**

Pre-built autonomous capability packages that run independently on schedules without user prompting. Each Hand bundles a manifest (HAND.toml), multi-phase system prompt, skill reference (SKILL.md), and approval guardrails.

**Bundled Hands:**
- **Clip** - YouTube video download, moment extraction, vertical shorts generation with captions/thumbnails, AI voice-over, publish to Telegram/WhatsApp. 8-phase FFmpeg + yt-dlp pipeline.
- **Lead** - Daily prospect discovery, web enrichment, 0-100 scoring, ICP profile building, CSV/JSON/Markdown delivery.
- **Collector** - OSINT-grade continuous monitoring, change detection, sentiment tracking, knowledge graph construction, critical alerts.
- **Predictor** - Superforecasting engine with calibrated reasoning, confidence intervals, Brier score tracking, contrarian mode.
- **Researcher** - Cross-referencing multiple sources, CRAAP credibility evaluation, APA-formatted cited reports.
- **Twitter** - Autonomous account management, 7 rotating content formats, engagement optimization, approval queue.
- **Browser** - Web automation via Playwright bridge, multi-step workflows, mandatory purchase approval gate.

**Key Files:**
- `crates/openfang-hands/src/lib.rs` - Hand lifecycle management
- `crates/openfang-hands/src/registry.rs` - Hand registration
- `agents/*/agent.toml` - Individual agent manifests (30 agents in `agents/`)

---

### 2. Agent Runtime Engine
**Priority: Core**

The core agent execution loop that orchestrates LLM calls, tool execution, memory management, and safety systems. Supports 3 native LLM drivers connecting to 27 providers (123+ models).

**Key Files:**
- `crates/openfang-runtime/src/agent_loop.rs` - Main agent loop (183KB)
- `crates/openfang-runtime/src/drivers/` - LLM driver implementations (Anthropic, Claude Code, Copilot, Gemini, OpenAI, Qwen Code)
- `crates/openfang-runtime/src/tool_runner.rs` - Tool execution engine (155KB)
- `crates/openfang-runtime/src/llm_driver.rs` - Driver abstraction layer

---

### 3. Multi-Channel Messaging (40 Adapters)
**Priority: Core**

Connect agents to every platform users are on. Each adapter supports per-channel model overrides, DM/group policies, rate limiting, and output formatting.

**Adapter Categories:**
- **Core:** Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Email (IMAP/SMTP)
- **Enterprise:** Microsoft Teams, Mattermost, Google Chat, Webex, Feishu/Lark, Zulip
- **Social:** LINE, Viber, Facebook Messenger, Mastodon, Bluesky, Reddit, LinkedIn, Twitch
- **Community:** IRC, XMPP, Guilded, Revolt, Keybase, Discourse, Gitter
- **Privacy:** Threema, Nostr, Mumble, Nextcloud Talk, Rocket.Chat, Ntfy, Gotify
- **Workplace:** Pumble, Flock, Twist, DingTalk, Zalo, Webhooks

**Key Files:**
- `crates/openfang-channels/src/bridge.rs` - Channel abstraction layer
- `crates/openfang-channels/src/*.rs` - Individual adapter implementations (50+ adapters)

---

### 4. 16-Layer Security System
**Priority: Core**

Defense-in-depth security architecture with independently testable layers:

| # | System | Implementation |
|---|--------|----------------|
| 1 | WASM Dual-Metered Sandbox | Fuel metering + epoch interruption, watchdog thread |
| 2 | Merkle Hash-Chain Audit Trail | Cryptographic tamper detection |
| 3 | Information Flow Taint Tracking | Secret propagation from source to sink |
| 4 | Ed25519 Signed Agent Manifests | Cryptographic identity and capability signing |
| 5 | SSRF Protection | Blocks private IPs, cloud metadata, DNS rebinding |
| 6 | Secret Zeroization | Zeroizing<String> auto-wipes API keys from memory |
| 7 | OFP Mutual Authentication | HMAC-SHA256 nonce-based P2P verification |
| 8 | Capability Gates | RBAC enforcement on agent tool access |
| 9 | Security Headers | CSP, X-Frame-Options, HSTS, X-Content-Type-Options |
| 10 | Health Endpoint Redaction | Minimal public info, auth required for full diagnostics |
| 11 | Subprocess Sandbox | env_clear() + selective variable passthrough |
| 12 | Prompt Injection Scanner | Override/exfiltration pattern detection |
| 13 | Loop Guard | SHA256-based circuit breaker for tool call loops |
| 14 | Session Repair | 7-phase message history validation and recovery |
| 15 | Path Traversal Prevention | Canonicalization with symlink escape prevention |
| 16 | GCRA Rate Limiter | Cost-aware token bucket with per-IP tracking |

**Key Files:**
- `crates/openfang-runtime/src/sandbox.rs` - WASM sandbox
- `crates/openfang-runtime/src/audit.rs` - Merkle hash-chain
- `crates/openfang-runtime/src/subprocess_sandbox.rs` - Process isolation
- `crates/openfang-runtime/src/loop_guard.rs` - Loop detection
- `crates/openfang-types/src/taint.rs` - Taint tracking
- `crates/openfang-types/src/manifest_signing.rs` - Ed25519 signing
- `crates/openfang-extensions/src/vault.rs` - Credential storage

---

### 5. Memory & Knowledge Management
**Priority: Core**

SQLite persistence with vector embeddings for semantic search, canonical session management, and automatic compaction.

**Key Files:**
- `crates/openfang-memory/src/session.rs` - Session management
- `crates/openfang-memory/src/knowledge.rs` - Knowledge graph construction
- `crates/openfang-memory/src/semantic.rs` - Vector embedding search
- `crates/openfang-memory/src/substrate.rs` - SQLite storage layer
- `crates/openfang-memory/src/compactor.rs` - Memory compaction
- `crates/openfang-runtime/src/embedding.rs` - Embedding generation

---

### 6. REST API & WebSocket Server
**Priority: Core**

140+ REST/WS/SSE endpoints including OpenAI-compatible API drop-in replacement. Serves the Alpine.js dashboard SPA.

**Key Files:**
- `crates/openfang-api/src/routes.rs` - API route definitions (431KB)
- `crates/openfang-api/src/server.rs` - Axum server setup
- `crates/openfang-api/src/openai_compat.rs` - OpenAI-compatible endpoints
- `crates/openfang-api/src/ws.rs` - WebSocket handling
- `crates/openfang-api/src/channel_bridge.rs` - Channel API bridge

---

### 7. Desktop Application (Tauri 2.0)
**Priority: Core**

Native desktop app with system tray, notifications, and global shortcuts. TUI dashboard available via CLI.

**Key Files:**
- `crates/openfang-desktop/src/lib.rs` - Tauri app entry
- `crates/openfang-desktop/src/tray.rs` - System tray integration
- `crates/openfang-desktop/src/commands.rs` - Tauri commands
- `crates/openfang-cli/src/tui/` - CLI TUI implementation

---

## Secondary Features (Important but Non-Core)

### 8. Skills System & Marketplace
**Priority: Secondary**

60 bundled skills (domain expertise reference injected at runtime) with FangHub marketplace integration. SKILL.md parser for agent context enrichment.

**Key Files:**
- `crates/openfang-skills/src/loader.rs` - SKILL.md parsing
- `crates/openfang-skills/src/registry.rs` - Skill registration
- `crates/openfang-skills/src/clawhub.rs` - FangHub marketplace

---

### 9. WASM Tool Sandbox
**Priority: Secondary**

Dual-metered WebAssembly execution environment for running untrusted tool code with fuel metering and epoch interruption.

**Key Files:**
- `crates/openfang-runtime/src/sandbox.rs` - WASM sandbox
- `crates/openfang-runtime/src/workspace_sandbox.rs` - Workspace isolation
- `crates/openfang-runtime/src/host_functions.rs` - Host function definitions

---

### 10. MCP (Model Context Protocol) & A2A (Agent-to-Agent)
**Priority: Secondary**

Standard protocols for tool serving and inter-agent communication.

**Key Files:**
- `crates/openfang-runtime/src/mcp.rs` - MCP server implementation
- `crates/openfang-runtime/src/mcp_server.rs` - MCP protocol handling
- `crates/openfang-runtime/src/a2a.rs` - Agent-to-agent protocol

---

### 11. P2P Wire Protocol (OFP)
**Priority: Secondary**

OpenFang Protocol for peer-to-peer networking with HMAC-SHA256 mutual authentication.

**Key Files:**
- `crates/openfang-wire/src/peer.rs` - Peer connection management
- `crates/openfang-wire/src/registry.rs` - Peer discovery
- `crates/openfang-wire/src/consolidation.rs` - Network consolidation

---

### 12. Credential Vault & OAuth2 PKCE
**Priority: Secondary**

AES-256-GCM encrypted credential storage with OAuth2 PKCE flow support for integrations.

**Key Files:**
- `crates/openfang-extensions/src/vault.rs` - Encrypted credential storage
- `crates/openfang-extensions/src/oauth.rs` - OAuth2 PKCE implementation
- `crates/openfang-extensions/src/credentials.rs` - Credential management

---

### 13. Migration Engine
**Priority: Secondary**

One-command migration from OpenClaw, LangChain, and AutoGPT with automatic agent, memory, skills, and config import.

**Key Files:**
- `crates/openfang-migrate/src/openclaw.rs` - OpenClaw migration (159KB)
- `crates/openfang-migrate/src/migration.rs` - Migration framework
- `crates/openfang-migrate/src/report.rs` - Migration reporting

---

### 14. WhatsApp Web Gateway
**Priority: Secondary**

QR-code-based WhatsApp connection (like WhatsApp Web) without Meta Business account requirement.

**Key Files:**
- `packages/whatsapp-gateway/index.js` - Node.js gateway server

---

## Feature Summary Table

| Feature | Priority | Crate/Location |
|---------|----------|----------------|
| Autonomous Hands System | Core | `crates/openfang-hands`, `agents/` |
| Agent Runtime Engine | Core | `crates/openfang-runtime` |
| Multi-Channel Messaging | Core | `crates/openfang-channels` |
| 16-Layer Security | Core | `crates/openfang-{runtime,types,extensions}` |
| Memory & Knowledge | Core | `crates/openfang-memory` |
| REST API & WebSocket | Core | `crates/openfang-api` |
| Desktop App (Tauri 2.0) | Core | `crates/openfang-desktop` |
| Skills System | Secondary | `crates/openfang-skills` |
| WASM Tool Sandbox | Secondary | `crates/openfang-runtime` |
| MCP & A2A Protocols | Secondary | `crates/openfang-runtime` |
| P2P Wire Protocol | Secondary | `crates/openfang-wire` |
| Credential Vault | Secondary | `crates/openfang-extensions` |
| Migration Engine | Secondary | `crates/openfang-migrate` |
| WhatsApp Web Gateway | Secondary | `packages/whatsapp-gateway` |

---

## Verification: README Claims vs. Actual Implementation

| README Claim | Status | Evidence |
|-------------|--------|----------|
| 7 Bundled Hands | Verified | `crates/openfang-hands/src/registry.rs`, `agents/*/agent.toml` |
| 14 crates | Verified | `Cargo.toml` workspace members |
| 137K LOC | Unverified | README claim |
| 1,767+ tests | Unverified | README claim |
| Zero clippy warnings | Unverified | README claim |
| 40 channel adapters | Verified | `crates/openfang-channels/src/*.rs` (50+ files) |
| 27 LLM providers | Verified | `crates/openfang-runtime/src/drivers/`, `model_catalog.rs` |
| 53+ tools | Unverified | `tool_runner.rs` + `drivers/` suggests this |
| 60 bundled skills | Unverified | `crates/openfang-skills/src/` suggests this |
| 16 security layers | Verified | Multiple security implementations across crates |
| WASM dual-metered sandbox | Verified | `sandbox.rs`, `workspace_sandbox.rs` |
| Merkle hash-chain audit | Verified | `audit.rs` |
| Taint tracking | Verified | `crates/openfang-types/src/taint.rs` |
| 140+ API endpoints | Verified | `routes.rs` (431KB) |
| OpenAI-compatible API | Verified | `openai_compat.rs` |
| Tauri 2.0 desktop | Verified | `crates/openfang-desktop/` |
| CLI with TUI | Verified | `crates/openfang-cli/src/tui/` |
| OFP P2P protocol | Verified | `crates/openfang-wire/` |
| MCP + A2A | Verified | `mcp.rs`, `a2a.rs` |
| Migration engine | Verified | `crates/openfang-migrate/` |
| WhatsApp Web Gateway | Verified | `packages/whatsapp-gateway/` |
