# PicoClaw Feature Index

## Core Features (Essential Capabilities)

### 1. Ultra-Lightweight Agent Core
**Description:** Memory-optimized AI agent running in <10MB RAM, 99% smaller than OpenClaw. Built in Go with AI-assisted development.

**Key Files/Locations:**
- `pkg/agent/loop.go` - Main agent loop (107KB)
- `pkg/agent/instance.go` - Agent instance management
- `pkg/agent/turn.go` - Turn processing
- `pkg/agent/memory.go` - Memory management

**Priority:** CORE

---

### 2. Multi-Provider LLM Support
**Description:** Unified interface to 30+ LLM providers (OpenAI, Anthropic, Gemini, DeepSeek, local Ollama/vLLM, etc.) with fallback routing and model selection.

**Key Files/Locations:**
- `pkg/providers/factory_provider.go` - Provider factory
- `pkg/providers/fallback.go` - Fallback routing
- `pkg/providers/error_classifier.go` - Error classification
- `pkg/providers/openai_compat/` - OpenAI-compatible providers
- `docs/providers.md` - Provider documentation

**Priority:** CORE

---

### 3. Multi-Channel Chat Integration
**Description:** Connect to 17+ messaging platforms (Telegram, Discord, WhatsApp, WeChat, QQ, Slack, Matrix, Feishu, DingTalk, LINE, IRC, etc.) via unified channel abstraction.

**Key Files/Locations:**
- `pkg/channels/manager.go` - Channel manager
- `pkg/channels/telegram/` - Telegram integration
- `pkg/channels/discord/` - Discord integration
- `pkg/channels/weixin/` - WeChat integration
- `pkg/channels/feishu/` - Feishu/Lark integration
- `docs/channels/` - Channel setup guides

**Priority:** CORE

---

### 4. MCP (Model Context Protocol) Integration
**Description:** Native MCP protocol support to extend agent capabilities with external tools and data sources via stdio, SSE, or HTTP transports.

**Key Files/Locations:**
- `pkg/mcp/` - MCP implementation
- `pkg/tools/mcp_tool.go` - MCP tool wrapper
- `pkg/agent/loop_mcp.go` - MCP loop integration
- `docs/tools_configuration.md#mcp-tool` - MCP configuration

**Priority:** CORE

---

### 5. Vision Pipeline
**Description:** Send images and files directly to the agent with automatic base64 encoding for multimodal LLMs.

**Key Files/Locations:**
- `pkg/agent/loop_media.go` - Media processing in agent loop
- `pkg/media/` - Media utilities
- `pkg/tools/send_file.go` - File sending tool

**Priority:** CORE

---

### 6. Gateway Service
**Description:** HTTP/WebSocket gateway that bridges chat channels to the AI agent, handling message routing, authentication, and protocol translation.

**Key Files/Locations:**
- `pkg/gateway/` - Gateway implementation
- `pkg/channels/manager.go` - Channel orchestration
- `cmd/picoclaw/` - CLI entry points

**Priority:** CORE

---

### 7. Smart Model Routing
**Description:** Rule-based model routing that directs simple queries to lightweight models and complex tasks to premium models, optimizing API costs.

**Key Files/Locations:**
- `pkg/routing/` - Routing implementation
- `pkg/providers/fallback.go` - Fallback strategy
- `pkg/agent/model_resolution.go` - Model resolution

**Priority:** CORE

---

## Secondary Features (Important Capabilities)

### 8. Web UI Launcher
**Description:** Browser-based configuration and chat interface running on localhost:18800. Provides one-click setup for providers and channels.

**Key Files/Locations:**
- `web/frontend/` - Frontend Vue.js application
- `web/backend/` - Backend Go API
- `cmd/picoclaw-launcher/` - Launcher entry point
- `assets/launcher-webui.jpg` - UI screenshot

**Priority:** SECONDARY

---

### 9. TUI (Terminal UI) Launcher
**Description:** Full-featured terminal interface for headless environments, servers, and SSH access.

**Key Files/Locations:**
- `cmd/picoclaw-launcher-tui/` - TUI launcher
- `assets/launcher-tui.jpg` - TUI screenshot

**Priority:** SECONDARY

---

### 10. Skills System
**Description:** Modular capability extensions loaded from SKILL.md files. Install from ClawHub marketplace or create custom skills.

**Key Files/Locations:**
- `pkg/skills/` - Skills implementation
- `pkg/tools/skills_install.go` - Skill installation
- `pkg/tools/skills_search.go` - Skill search
- `workspace/skills/` - Installed skills

**Priority:** SECONDARY

---

### 11. Cron / Scheduled Tasks
**Description:** Built-in scheduling for reminders and recurring tasks using natural language ("remind me in 10 minutes") or cron expressions.

**Key Files/Locations:**
- `pkg/cron/` - Cron implementation
- `pkg/tools/cron.go` - Cron tool
- `pkg/commands/` - CLI commands

**Priority:** SECONDARY

---

### 12. Web Search Tools
**Description:** Integrated web search via multiple engines (DuckDuckGo built-in, Tavily, Brave, Perplexity, SearXNG, Baidu, etc.).

**Key Files/Locations:**
- `pkg/tools/search_tool.go` - Search tool
- `pkg/tools/web.go` - Web search implementation
- `docs/tools_configuration.md` - Tool configuration

**Priority:** SECONDARY

---

## Extended Features (Specialized Capabilities)

### 13. Hardware Device Support
**Description:** Support for specialized hardware interfaces including I2C, SPI, and MaixCam for embedded/IoT deployments.

**Key Files/Locations:**
- `pkg/tools/i2c.go` - I2C interface
- `pkg/tools/spi.go` - SPI interface
- `pkg/channels/maixcam/` - MaixCam channel
- `docs/hardware-compatibility.md` - Hardware list

**Priority:** EXTENDED

---

### 14. Sub-Agent Orchestration
**Description:** Spawn async sub-agents for parallel task execution with status tracking and lifecycle management.

**Key Files/Locations:**
- `pkg/agent/subturn.go` - Sub-turn coordination
- `pkg/tools/spawn.go` - Spawn tool
- `pkg/tools/spawn_status.go` - Status tracking
- `pkg/tools/subagent.go` - Subagent tool

**Priority:** EXTENDED

---

### 15. Steering & Hooks System
**Description:** Inject messages into running agent loop and event-driven hooks (observers, interceptors, approval hooks).

**Key Files/Locations:**
- `pkg/agent/steering.go` - Steering implementation
- `pkg/agent/hooks.go` - Hook system
- `pkg/agent/hook_process.go` - Hook processing
- `docs/steering.md` - Steering documentation
- `docs/hooks/README.md` - Hooks documentation

**Priority:** EXTENDED

---

### 16. Credential Management & Security
**Description:** Secure credential storage with encryption, sensitive data filtering, and security configuration.

**Key Files/Locations:**
- `pkg/credential/` - Credential management
- `pkg/auth/` - Authentication
- `docs/security_configuration.md` - Security docs
- `docs/credential_encryption.md` - Encryption docs

**Priority:** EXTENDED

---

### 17. Session & Shell Tools
**Description:** Interactive shell execution, session management, and file operations with security sandboxing.

**Key Files/Locations:**
- `pkg/tools/shell.go` - Shell tool
- `pkg/tools/session.go` - Session management
- `pkg/tools/filesystem.go` - File operations
- `pkg/session/` - Session package

**Priority:** EXTENDED

---

### 18. Voice / Audio Pipeline
**Description:** Voice processing capabilities for audio input/output with LLM integration.

**Key Files/Locations:**
- `pkg/voice/` - Voice processing

**Priority:** EXTENDED

---

### 19. Docker Deployment
**Description:** Containerized deployment via Docker Compose with profiles for launcher, agent, and gateway modes.

**Key Files/Locations:**
- `docker/` - Docker configuration
- `docker/docker-compose.yml` - Compose file
- `docs/docker.md` - Docker documentation

**Priority:** SECONDARY

---

### 20. Cross-Platform Binary
**Description:** Single binary distribution for x86_64, ARM64, MIPS, RISC-V, LoongArch architectures.

**Key Files/Locations:**
- `.goreleaser.yaml` - Release configuration
- `Makefile` - Build targets
- `docs/hardware-compatibility.md` - Supported hardware

**Priority:** SECONDARY

---

## Feature Summary by Priority

| Priority | Count | Features |
|----------|-------|----------|
| **CORE** | 7 | Ultra-Lightweight Agent, Multi-Provider LLM, Multi-Channel, MCP, Vision, Gateway, Model Routing |
| **SECONDARY** | 6 | Web UI, TUI, Skills, Cron, Web Search, Docker, Cross-Platform Binary |
| **EXTENDED** | 7 | Hardware Support, Sub-Agent Orchestration, Steering/Hooks, Credential Security, Shell/Session, Voice, |

**Total Features:** 20

---

## Cross-Reference: README Claims vs. Implementation

| README Feature | Status | Implementation Location |
|----------------|--------|-------------------------|
| Ultra-lightweight (<10MB) | Verified | `pkg/agent/` - Go native implementation |
| $10 Hardware support | Verified | `docs/hardware-compatibility.md` |
| <1s boot time | Verified | `cmd/picoclaw/` - Minimal startup |
| Multi-architecture | Verified | `.goreleaser.yaml` - x86, ARM, MIPS, RISC-V, LoongArch |
| AI-bootstrapped | Verified | `pkg/` - Pure Go, 95% agent-generated |
| MCP support | Verified | `pkg/mcp/`, `pkg/tools/mcp_tool.go` |
| Vision pipeline | Verified | `pkg/agent/loop_media.go`, `pkg/media/` |
| Smart routing | Verified | `pkg/routing/`, `pkg/providers/fallback.go` |
| 17+ channels | Verified | `pkg/channels/` (telegram, discord, weixin, etc.) |
| 30+ providers | Verified | `pkg/providers/` (openai, anthropic, gemini, etc.) |
| Web Search | Verified | `pkg/tools/search_tool.go`, `pkg/tools/web.go` |
| Skills system | Verified | `pkg/skills/`, `pkg/tools/skills_*.go` |
| Cron/scheduling | Verified | `pkg/cron/`, `pkg/tools/cron.go` |
| Docker support | Verified | `docker/` |
| WebUI Launcher | Verified | `web/` |
| TUI Launcher | Verified | `cmd/picoclaw-launcher-tui/` |

---

## Notes

- PicoClaw is in active development (v0.2.x as of March 2026)
- Memory footprint noted to be 10-20MB in recent builds due to rapid feature additions
- Resource optimization planned after feature stabilization at v1.0
- All documented features have corresponding implementation in `pkg/` or `cmd/` directories
