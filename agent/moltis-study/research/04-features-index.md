# Moltis Feature Index

Prioritized feature list derived from README.md documentation and codebase structure analysis.

## Core Features (5-7)

### 1. AI Gateway (Multi-Provider LLM Support)
**Description:** Unified gateway that routes requests to multiple LLM providers (OpenAI Codex, GitHub Copilot, local models) with streaming responses, model fallback, and cost management.

**Key Files/Locations:**
- `crates/providers/` — Provider implementations (17.6K LoC)
- `crates/agents/` — Agent loop and streaming logic (9.6K LoC)
- `crates/gateway/` — HTTP/WS server, RPC, multi-provider routing (36.1K LoC)
- `crates/provider-setup/` — Provider configuration and key management

**Priority:** CORE

---

### 2. Chat Engine & Agent Orchestration
**Description:** Core chat engine that orchestrates agents, manages conversation context, handles parallel tool execution, sub-agent delegation, and prompt assembly.

**Key Files/Locations:**
- `crates/chat/` — Chat engine and agent orchestration (11.5K LoC)
- `crates/agents/src/runner.rs` — Main agent loop (~5K LoC core)
- `crates/agents/src/model.rs` — Model interaction layer

**Priority:** CORE

---

### 3. Web UI
**Description:** Built-in web interface with real-time chat, agent management, provider configuration, and session history. Includes PWA support with push notifications.

**Key Files/Locations:**
- `crates/web/` — Web UI crate (4.5K LoC)
- `crates/web/ui/` — Frontend assets (Preact, Tailwind CSS)
- `crates/web/ui/e2e/` — Playwright E2E tests
- `website/` — Documentation site

**Priority:** CORE

---

### 4. Session Management & Persistence
**Description:** JSONL-based session persistence with automatic compaction, per-agent memory workspaces, and cross-channel session continuity.

**Key Files/Locations:**
- `crates/sessions/` — Session persistence (3.8K LoC)
- `crates/sessions/migrations/` — SQLite schema for sessions
- `crates/gateway/src/auth.rs` — Auth sessions, API keys

**Priority:** CORE

---

### 5. Memory & Context System
**Description:** Hybrid storage combining SQLite full-text search with vector embeddings for long-term memory. Includes embeddings cache, chunk management, and automatic context injection.

**Key Files/Locations:**
- `crates/memory/` — Memory backend (SQLite + FTS + vector)
- `crates/qmd/` — Query processing and memory document handling
- `crates/memory/migrations/` — Memory schema migrations
- `crates/projects/` — Project-level context management

**Priority:** CORE

---

### 6. Tool Execution & Sandboxed Sandbox
**Description:** Secure tool execution in Docker containers or Apple Container with package management, filesystem isolation, and resource limits. Supports WASM sandbox tools.

**Key Files/Locations:**
- `crates/tools/` — Tool execution engine (21.9K LoC)
- `crates/tools/src/sandbox.rs` — Sandbox abstraction (trait + Docker/Apple Container impls)
- `crates/wasm-tools/` — WASM tool compilation and execution
- `crates/cli/src/sandbox_commands.rs` — CLI sandbox management

**Priority:** CORE

---

### 7. Authentication & Security
**Description:** Multi-layer auth including password + passkey (WebAuthn), API keys, OAuth integration, and encrypted vault for secrets. Rate limiting, SSRF protection, and WebSocket origin validation.

**Key Files/Locations:**
- `crates/auth/` — Authentication logic
- `crates/vault/` — Encrypted secret storage (XChaCha20-Poly1305 + Argon2id)
- `crates/oauth/` — OAuth provider integrations
- `crates/gateway/src/auth_middleware.rs` — Auth middleware
- `crates/gateway/src/web_fetch.rs` — SSRF protection

**Priority:** CORE

---

## Secondary Features (3-5)

### 8. Communication Channels (Telegram, Discord, WhatsApp, MS Teams)
**Description:** Multi-platform messaging integration with unified message handling, sender allowlisting, OTP verification flows, and bot capabilities.

**Key Files/Locations:**
- `crates/telegram/` — Telegram bot integration
- `crates/discord/` — Discord bot integration
- `crates/whatsapp/` — WhatsApp integration
- `crates/msteams/` — Microsoft Teams integration
- `crates/channels/` — Shared channel traits and registry

**Priority:** SECONDARY

---

### 9. Voice I/O (STT/TTS)
**Description:** Voice input/output with support for 7+ speech-to-text and 8+ text-to-speech providers. Audio transcription and synthesis integrated into chat flow.

**Key Files/Locations:**
- `crates/voice/` — Voice I/O providers (6.0K LoC)

**Priority:** SECONDARY

---

### 10. MCP Server Support
**Description:** Model Context Protocol server integration supporting both stdio and HTTP/SSE transport modes. Allows Moltis to connect to external MCP-compatible tools and services.

**Key Files/Locations:**
- `crates/mcp/` — MCP client/server implementation

**Priority:** SECONDARY

---

### 11. Scheduling (Cron/CalDAV)
**Description:** Time-based job scheduling with cron expression support and CalDAV calendar integration for event-driven automation.

**Key Files/Locations:**
- `crates/cron/` — Cron job scheduler (5.2K combined)
- `crates/caldav/` — CalDAV calendar integration

**Priority:** SECONDARY

---

### 12. Skills System
**Description:** Reusable skill packages (similar to OpenClaw store) enabling agents to acquire new capabilities through structured skill definitions.

**Key Files/Locations:**
- `crates/skills/` — Skill loading and execution
- `crates/skills/ui/` — Skill store UI components

**Priority:** SECONDARY

---

### 13. Hook System / Plugin Architecture
**Description:** Event-driven hooks with 15 lifecycle event types, circuit breaker pattern, and BeforeToolCall gating for security inspection.

**Key Files/Locations:**
- `crates/plugins/` — Hook dispatch and plugin formats (1.9K LoC)
- `crates/plugins/src/hooks.rs` — Hook event definitions

**Priority:** SECONDARY

---

### 14. Browser Automation
**Description:** Headless browser control for web scraping, automated testing, and web-based tool execution.

**Key Files/Locations:**
- `crates/browser/` — Browser automation (5.1K LoC)

**Priority:** SECONDARY

---

### 15. Extensibility (WASM, GraphQL, Metrics)
**Description:** Additional extensibility layers including WASM tool support, GraphQL API exposure, OpenTelemetry tracing, and Prometheus metrics.

**Key Files/Locations:**
- `crates/wasm-tools/` — WASM tool compilation
- `crates/graphql/` — GraphQL API (4.8K LoC)
- `crates/metrics/` — Prometheus metrics endpoint
- `crates/httpd/` — Additional HTTP server capabilities

**Priority:** SECONDARY

---

## Summary Table

| Feature | Priority | Key Crate(s) |
|---------|----------|--------------|
| AI Gateway | CORE | `moltis-providers`, `moltis-gateway` |
| Chat Engine | CORE | `moltis-chat`, `moltis-agents` |
| Web UI | CORE | `moltis-web` |
| Session Management | CORE | `moltis-sessions` |
| Memory System | CORE | `moltis-memory`, `moltis-qmd` |
| Tool Execution/Sandbox | CORE | `moltis-tools` |
| Authentication/Security | CORE | `moltis-auth`, `moltis-vault` |
| Communication Channels | SECONDARY | `moltis-telegram`, `moltis-discord`, etc. |
| Voice I/O | SECONDARY | `moltis-voice` |
| MCP Server Support | SECONDARY | `moltis-mcp` |
| Scheduling | SECONDARY | `moltis-cron`, `moltis-caldav` |
| Skills System | SECONDARY | `moltis-skills` |
| Hook/Plugin System | SECONDARY | `moltis-plugins` |
| Browser Automation | SECONDARY | `moltis-browser` |
| Extensibility (WASM/GraphQL) | SECONDARY | `moltis-wasm-tools`, `moltis-graphql` |

---

*Generated: 2026-03-26*
*Source: README.md, crates/ directory structure, CLAUDE.md*
