# ZeroClaw Feature Index

> **Project:** ZeroClaw — Personal AI Assistant
> **Source:** README.md + src/ directory structure analysis
> **Date:** 2026-03-26

## Feature Overview

ZeroClaw is a Rust-based personal AI assistant that runs on messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, IRC, Email, and 20+ more), with a web dashboard for real-time control, hardware peripheral support (ESP32, STM32, Arduino, Raspberry Pi), and multi-agent orchestration capabilities.

---

## Core Features (Priority 1)

### 1. Agent Loop & Orchestration
**Description:** The core AI agent orchestration engine that handles tool dispatch, prompt construction, message classification, context management, and memory loading.

**Key Files:**
- `src/agent/loop_.rs` (346KB — main loop)
- `src/agent/agent.rs` (62KB — agent struct)
- `src/agent/prompt.rs` (context/prompt construction)
- `src/agent/context_compressor.rs`
- `src/agent/classifier.rs`
- `src/agent/dispatcher.rs`
- `src/agent/memory_loader.rs`
- `src/agent/personality.rs`
- `src/agent/thinking.rs`
- `src/agent/loop_detector.rs`
- `src/agent/history_pruner.rs`
- `src/agent/eval.rs`

**Verification:** Folder exists, `loop_.rs` is 346KB indicating substantial orchestration logic.

---

### 2. Multi-Channel Inbox (20+ Integrations)
**Description:** Unified messaging across WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, IRC, Email, Bluesky, Nostr, Mattermost, Nextcloud Talk, DingTalk, Lark, QQ, Reddit, LinkedIn, Twitter, MQTT, WeChat Work, WATI, Mochat, Linq, Notion, WebSocket, ClawdTalk. Includes voice calling, wake word detection, transcription, TTS, and link enrichment.

**Key Files:**
- `src/channels/mod.rs` (415KB — channel registry)
- `src/channels/telegram.rs` (183KB)
- `src/channels/slack.rs` (162KB)
- `src/channels/whatsapp_web.rs` (60KB)
- `src/channels/whatsapp.rs` (61KB)
- `src/channels/discord.rs` (85KB)
- `src/channels/lark.rs` (124KB)
- `src/channels/matrix.rs` (82KB)
- `src/channels/qq.rs` (63KB)
- `src/channels/voice_call.rs`, `src/channels/voice_wake.rs`, `src/channels/tts.rs`, `src/channels/transcription.rs`
- `src/channels/session_store.rs`, `src/channels/session_sqlite.rs`
- `src/channels/link_enricher.rs`, `src/channels/media_pipeline.rs`

**Verification:** 26 channel implementations confirmed in src/channels/ directory.

---

### 3. Tool System (70+ Tools)
**Description:** First-class tools for shell, file I/O, browser control, git operations, web fetch/search, MCP (Model Context Protocol), Jira, Notion, Google Workspace, Microsoft 365, LinkedIn, Composio, cron scheduling, memory operations, and hardware access.

**Key Files:**
- `src/tools/mod.rs` (50KB — tool registry)
- `src/tools/delegate.rs` (99KB — agent-to-agent delegation)
- `src/tools/swarm.rs` (31KB — swarm orchestration)
- `src/tools/browser.rs` (92KB)
- `src/tools/git_operations.rs`
- `src/tools/file_read.rs`, `src/tools/file_write.rs`, `src/tools/file_edit.rs`
- `src/tools/shell.rs` (26KB)
- `src/tools/web_fetch.rs` (48KB), `src/tools/web_search_tool.rs` (25KB)
- `src/tools/mcp_client.rs`, `src/tools/mcp_tool.rs`, `src/tools/mcp_deferred.rs`
- `src/tools/jira_tool.rs` (56KB), `src/tools/notion_tool.rs`, `src/tools/google_workspace.rs`
- `src/tools/cron_add.rs` (30KB), `src/tools/cron_update.rs`, `src/tools/cron_remove.rs`
- `src/tools/memory_recall.rs`, `src/tools/memory_store.rs`, `src/tools/memory_forget.rs`
- `src/tools/hardware_board_info.rs`, `src/tools/hardware_memory_map.rs`, `src/tools/hardware_memory_read.rs`

**Verification:** 96 files in src/tools/ directory, confirming extensive tool ecosystem.

---

### 4. Memory & Knowledge Management
**Description:** Multi-strategy memory system with SQLite persistence, vector embeddings (Qdrant, local), knowledge graph, retrieval with importance scoring, decay algorithms, hygiene maintenance, response caching, and snapshot capabilities.

**Key Files:**
- `src/memory/sqlite.rs` (96KB — core storage)
- `src/memory/knowledge_graph.rs` (29KB)
- `src/memory/embeddings.rs`
- `src/memory/qdrant.rs` (20KB)
- `src/memory/vector.rs`
- `src/memory/retrieval.rs`
- `src/memory/lucid.rs` (21KB — memory hygiene)
- `src/memory/hygiene.rs` (17KB)
- `src/memory/decay.rs`
- `src/memory/importance.rs`
- `src/memory/response_cache.rs` (17KB)
- `src/memory/snapshot.rs`
- `src/memory/consolidation.rs`
- `src/memory/chunker.rs`

**Verification:** 26 files in src/memory/ confirming comprehensive memory architecture.

---

### 5. Security & Sandboxing
**Description:** Defense-in-depth security including pairing-based DM approval, WebAuthn authentication, IAM policy engine, audit logging, sandboxing (Bubblewrap, Firejail, Landlock, Seatbelt), secrets management, leak detection, vulnerability scanning, prompt injection guard, emergency stop, and workspace isolation.

**Key Files:**
- `src/security/policy.rs` (108KB — IAM policy engine)
- `src/security/webauthn.rs` (49KB)
- `src/security/audit.rs` (41KB — security audit)
- `src/security/pairing.rs` (27KB)
- `src/security/iam_policy.rs` (15KB)
- `src/security/secrets.rs` (33KB)
- `src/security/leak_detector.rs` (21KB)
- `src/security/estop.rs` (13KB — emergency stop)
- `src/security/prompt_guard.rs` (13KB)
- `src/security/bubblewrap.rs`, `src/security/firejail.rs`, `src/security/landlock.rs`
- `src/security/workspace_boundary.rs`
- `src/security/vulnerability.rs`
- `src/security/domain_matcher.rs`

**Verification:** 25 files in src/security/ confirming extensive security infrastructure. README mentions 129+ automated security tests.

---

### 6. AI Provider Abstraction & Routing
**Description:** Swappable AI provider layer supporting Anthropic, OpenAI, OpenAI Codex, Google Gemini, Azure OpenAI, AWS Bedrock, Ollama, OpenRouter, Claude Code, Groq, Cohere, and custom endpoints. Includes model routing, failover, retry logic, and reliable request wrapper.

**Key Files:**
- `src/providers/mod.rs` (129KB — provider registry)
- `src/providers/reliable.rs` (111KB — resilient wrapper with retry/failover)
- `src/providers/router.rs` (38KB — model routing)
- `src/providers/anthropic.rs` (72KB)
- `src/providers/openai.rs` (34KB)
- `src/providers/gemini.rs` (82KB)
- `src/providers/bedrock.rs` (66KB)
- `src/providers/openrouter.rs` (43KB)
- `src/providers/openai_codex.rs` (37KB)
- `src/providers/ollama.rs` (51KB)
- `src/providers/compatible.rs` (139KB — OpenAI-compatible endpoints)
- `src/providers/traits.rs` (34KB)

**Verification:** 21 files in src/providers/ confirming multi-provider architecture.

---

### 7. Web Dashboard & Gateway API
**Description:** React 19 + Vite + Tailwind CSS web dashboard served directly from the Gateway with real-time chat, memory browser, config editor, cron manager, tool inspector, logs viewer, cost tracking, health diagnostics, integrations status, and pairing management. Gateway provides HTTP/WebSocket/SSE control plane.

**Key Files:**
- `src/gateway/mod.rs` (128KB — gateway core)
- `src/gateway/api.rs` (74KB — REST API)
- `src/gateway/ws.rs` (20KB — WebSocket)
- `src/gateway/sse.rs` (5KB — Server-Sent Events)
- `src/gateway/api_pairing.rs`
- `src/gateway/api_webauthn.rs`
- `src/gateway/canvas.rs`
- `src/gateway/tls.rs` (17KB)
- `src/gateway/nodes.rs` (20KB)
- `src/gateway/static_files.rs`
- `src/gateway/hardware_context.rs`

**Verification:** 14 files in src/gateway/ confirming web service layer.

---

### 8. SOPs (Standard Operating Procedures)
**Description:** Event-driven workflow automation engine with MQTT, webhook, cron, and peripheral triggers. SOPs can chain actions, manage state, track metrics, and provide audit trails.

**Key Files:**
- `src/sop/mod.rs` (73KB — SOP engine)
- `src/sop/engine.rs` (73KB)
- `src/sop/dispatch.rs` (27KB)
- `src/sop/condition.rs` (13KB)
- `src/sop/types.rs`
- `src/sop/metrics.rs` (48KB)
- `src/sop/audit.rs`
- `src/sop/creator.rs`
- `src/sop/improver.rs`
- `src/tools/sop_*.rs` (sop_execute, sop_approve, sop_advance, sop_list, sop_status)

**Verification:** 9 files in src/sop/ plus 5 sop_* tools confirming workflow automation.

---

## Secondary Features (Priority 2)

### 9. Skills Platform
**Description:** Bundled, community, and workspace skills with SKILL.md/SKILL.toml format, security auditing before install, and git-based distribution.

**Key Files:**
- `src/skills/mod.rs` (72KB — skill engine)
- `src/skills/creator.rs` (31KB)
- `src/skills/audit.rs` (29KB)
- `src/skills/improver.rs` (14KB)
- `src/skills/testing.rs`
- `src/tools/read_skill.rs`, `src/tools/skill_tool.rs`
- `src/skillforge/` (skill development tools)

**Verification:** 8 files in src/skills/ confirming skills platform.

---

### 10. Cron & Scheduling
**Description:** Persistent cron scheduler with SQLite store, sophisticated scheduling logic, multiple schedule types, and cron tool integration for creating, updating, removing, and running scheduled jobs.

**Key Files:**
- `src/cron/scheduler.rs` (51KB)
- `src/cron/store.rs` (59KB)
- `src/cron/schedule.rs` (11KB)
- `src/cron/mod.rs` (33KB)
- `src/cron/types.rs`
- `src/cron/error.rs`
- `src/tools/cron_add.rs`, `src/tools/cron_update.rs`, `src/tools/cron_remove.rs`, `src/tools/cron_run.rs`, `src/tools/cron_list.rs`, `src/tools/cron_runs.rs`
- `src/tools/schedule.rs`

**Verification:** 7 files in src/cron/ plus 7 cron-related tools.

---

### 11. Tunnel & Remote Access
**Description:** Tunnel providers for Cloudflare, Tailscale, ngrok, OpenVPN, Pinggy, and custom command for exposing local Gateway to the internet with TLS.

**Key Files:**
- `src/tunnel/mod.rs` (14KB)
- `src/tunnel/cloudflare.rs`
- `src/tunnel/tailscale.rs`
- `src/tunnel/ngrok.rs`
- `src/tunnel/openvpn.rs`
- `src/tunnel/pinggy.rs`
- `src/tunnel/custom.rs`
- `src/tunnel/none.rs`

**Verification:** 10 files in src/tunnel/ confirming multi-tunnel strategy.

---

### 12. Hardware Peripherals
**Description:** Peripheral trait and implementations for ESP32, ESP32-UI, STM32 Nucleo, Arduino, and Raspberry Pi GPIO. Supports board info, memory mapping, and memory read operations.

**Key Files:**
- `src/hardware/` (22 subdirectories)
- `src/peripherals/`
- `src/tools/hardware_board_info.rs`
- `src/tools/hardware_memory_map.rs`
- `src/tools/hardware_memory_read.rs`
- `docs/hardware/` (hardware documentation)

**Verification:** `src/hardware/` and `src/peripherals/` directories present.

---

## Summary Table

| Priority | Feature | Key Files | Size Indicator |
|----------|---------|-----------|----------------|
| 1 | Agent Loop & Orchestration | `src/agent/loop_.rs` | 346KB |
| 1 | Multi-Channel Inbox | `src/channels/mod.rs` | 415KB |
| 1 | Tool System (70+) | `src/tools/mod.rs` | 50KB |
| 1 | Memory & Knowledge | `src/memory/sqlite.rs` | 96KB |
| 1 | Security & Sandboxing | `src/security/policy.rs` | 108KB |
| 1 | Provider Abstraction | `src/providers/mod.rs` | 129KB |
| 1 | Web Dashboard & Gateway | `src/gateway/mod.rs` | 128KB |
| 1 | SOPs & Automation | `src/sop/mod.rs` | 73KB |
| 2 | Skills Platform | `src/skills/mod.rs` | 72KB |
| 2 | Cron & Scheduling | `src/cron/scheduler.rs` | 51KB |
| 2 | Tunnel & Remote Access | `src/tunnel/mod.rs` | 14KB |
| 2 | Hardware Peripherals | `src/hardware/`, `src/peripherals/` | 22 dirs |

---

## README Claims vs. File Existence Cross-Reference

| README Claim | Directory/File | Verified |
|--------------|----------------|----------|
| Multi-channel (WhatsApp, Telegram, Slack, Discord, etc.) | `src/channels/` | Yes |
| Agent orchestration loop | `src/agent/loop_.rs` | Yes |
| Tool system (70+ tools) | `src/tools/` (96 files) | Yes |
| Memory with SQLite, embeddings, Qdrant | `src/memory/` | Yes |
| Security: pairing, sandboxing, WebAuthn | `src/security/` (25 files) | Yes |
| 20+ LLM provider backends | `src/providers/` (21 files) | Yes |
| Web dashboard (React + Vite) | `src/gateway/` | Yes |
| SOPs with MQTT, webhook, cron triggers | `src/sop/`, `src/cron/` | Yes |
| Skills platform | `src/skills/` | Yes |
| Tunnel: Cloudflare, Tailscale, ngrok, OpenVPN | `src/tunnel/` | Yes |
| Hardware: ESP32, STM32, Arduino, Raspberry Pi | `src/hardware/`, `src/peripherals/` | Yes |
| Lifecycle hooks | `src/hooks/` | Yes |
| Daemon mode | `src/daemon/`, `zeroclaw daemon` | Yes |
| Onboarding wizard | `src/onboard/` | Yes |
| Migration from OpenClaw | `src/migration.rs` | Yes |
| Docker runtime support | `src/security/docker.rs` | Yes |

---

## Architecture Patterns Observed

1. **Trait-based swappability:** Providers, channels, tools, memory, tunnels all follow trait patterns (e.g., `src/providers/traits.rs`, `src/channels/traits.rs`, `src/tools/traits.rs`)

2. **Feature-gated modules:** Matrix, Lark, Nostr require `channel-matrix`, `channel-lark`, `channel-nostr` features

3. **Multi-layered storage:** SQLite core + optional Qdrant vector store + optional external services

4. **Resilient wrappers:** `src/providers/reliable.rs` provides retry/failover for provider calls

5. **WASM plugin support:** `src/plugins/wasm_tool.rs`, `src/plugins/wasm_channel.rs`

6. **Daemon + Gateway separation:** Daemon runs full autonomous runtime; Gateway is the webhook/API control plane

---

*Generated from zeroclaw repository analysis — README.md + src/ directory structure*
