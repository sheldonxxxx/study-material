# CoPaw Feature Index

A prioritized list of features extracted from the CoPaw codebase at `/Users/sheldon/Documents/claw/reference/CoPaw`.

## Priority Tier: Core

These are essential features without which CoPaw would not function as a personal AI assistant.

---

### 1. Multi-Channel Communication

**Description:** CoPaw connects to multiple chat platforms simultaneously, allowing users to interact with their AI assistant through different channels. Supported channels include DingTalk, Feishu, QQ, Discord, iMessage, Telegram, WeCom, Matrix, Mattermost, MQTT, and XiaoYi.

**Key Files/Locations:**
- `src/copaw/app/channels/` - Channel implementations
- `src/copaw/app/channels/manager.py` - Channel orchestration
- `src/copaw/app/channels/dingtalk/` - DingTalk integration
- `src/copaw/app/channels/feishu/` - Feishu/Lark integration
- `src/copaw/app/channels/qq/` - QQ integration
- `src/copaw/app/channels/discord_/` - Discord integration
- `src/copaw/app/channels/telegram/` - Telegram integration
- `src/copaw/app/channels/wecom/` - WeCom integration
- `src/copaw/app/channels/imessage/` - iMessage via Twilio
- `src/copaw/app/channels/matrix/` - Matrix protocol
- `src/copaw/app/channels/mattermost/` - Mattermost
- `src/copaw/app/channels/mqtt/` - MQTT
- `src/copaw/app/channels/xiaoyi/` - XiaoYi
- `src/copaw/app/channels/voice/` - Voice channel support
- `src/copaw/cli/channels_cmd.py` - Channel CLI management

**Priority:** Core

---

### 2. Web Console (User Interface)

**Description:** Browser-based chat interface and administration dashboard for configuring agents, channels, skills, models, and memory. The console provides real-time interaction with CoPaw and exposes configuration options.

**Key Files/Locations:**
- `console/` - React frontend application
- `console/src/App.tsx` - Main application component
- `console/src/pages/Chat/` - Chat interface pages
- `console/src/pages/Settings/` - Configuration pages
- `console/src/pages/Agent/` - Agent management pages
- `console/src/components/` - Reusable UI components
- `console/src/api/` - Frontend API client
- `console/src/stores/` - State management
- `src/copaw/app/routers/console.py` - Console API routes
- `website/public/docs/console.en.md` - Console documentation

**Priority:** Core

---

### 3. LLM Integration (Model Providers)

**Description:** Flexible model provider system supporting multiple LLM backends including OpenAI, Anthropic, Google Gemini, Ollama, and custom OpenAI-compatible APIs. Supports multimodal capabilities, auto-retry on failures, and model routing.

**Key Files/Locations:**
- `src/copaw/providers/` - Provider implementations
- `src/copaw/providers/provider_manager.py` - Multi-provider orchestration
- `src/copaw/providers/openai_provider.py` - OpenAI/API compatible
- `src/copaw/providers/anthropic_provider.py` - Anthropic/Claude
- `src/copaw/providers/gemini_provider.py` - Google Gemini
- `src/copaw/providers/ollama_provider.py` - Ollama local models
- `src/copaw/providers/retry_chat_model.py` - Retry logic
- `src/copaw/providers/multimodal_prober.py` - Capability detection
- `src/copaw/providers/capability_baseline.py` - Model capability tracking
- `src/copaw/app/routers/providers.py` - Provider API routes
- `src/copaw/app/model_factory.py` - Model factory
- `src/copaw/cli/providers_cmd.py` - Provider CLI management

**Priority:** Core

---

### 4. Skills System

**Description:** Extensible plugin architecture for adding capabilities. Built-in skills include cron scheduling, PDF handling, Word/Excel/PowerPoint documents, file reading, web browsing, news digest, and channel message handling. Skills can be auto-loaded from workspace or the Skills Hub marketplace.

**Key Files/Locations:**
- `src/copaw/agents/skills/` - Skill implementations
- `src/copaw/app/skills_manager.py` - Skill orchestration
- `src/copaw/app/skills/skills_hub.py` - Skills Hub integration
- `src/copaw/agents/skills/cron/` - Cron scheduling skill
- `src/copaw/agents/skills/pdf/` - PDF processing
- `src/copaw/agents/skills/docx/` - Word document handling
- `src/copaw/agents/skills/pptx/` - PowerPoint handling
- `src/copaw/agents/skills/xlsx/` - Excel handling
- `src/copaw/agents/skills/file_reader/` - File reading
- `src/copaw/agents/skills/news/` - News digest
- `src/copaw/agents/skills/browser_visible/` - Web browsing
- `src/copaw/agents/skills/browser_cdp/` - Browser automation
- `src/copaw/agents/skills/channel_message/` - Channel message handling
- `src/copaw/agents/skills/multi_agent_collaboration/` - Inter-agent collaboration
- `src/copaw/app/routers/skills.py` - Skills API routes
- `src/copaw/cli/skills_cmd.py` - Skills CLI management

**Priority:** Core

---

### 5. Multi-Agent Support

**Description:** Create and manage multiple independent AI agents, each with their own personality, skills, and configuration. Agents can collaborate through a dedicated collaboration skill, enabling complex workflows and delegation.

**Key Files/Locations:**
- `src/copaw/agents/multi_agent_manager.py` - Multi-agent orchestration
- `src/copaw/agents/skills/multi_agent_collaboration/` - Inter-agent communication
- `src/copaw/agents/_app.py` - Agent application logic
- `src/copaw/agents/react_agent.py` - ReAct agent implementation
- `src/copaw/agents/agent_context.py` - Agent context management
- `src/copaw/app/routers/agents.py` - Agent API routes
- `src/copaw/app/routers/agent.py` - Single agent routes
- `src/copaw/cli/agents_cmd.py` - Agent CLI management
- `website/public/docs/multi-agent.en.md` - Multi-agent documentation

**Priority:** Core

---

### 6. Memory System

**Description:** Long-term memory system that preserves conversation history and learned information across sessions. Supports multiple memory backends and provides context-aware retrieval for agent conversations.

**Key Files/Locations:**
- `src/copaw/agents/memory/` - Memory implementations
- `src/copaw/app/memory/memory_manager.py` - Memory orchestration
- `src/copaw/app/memory/agent_md_manager.py` - Markdown memory storage
- `src/copaw/local_models/backends/` - Local storage backends
- `src/copaw/app/routers/messages.py` - Message history API
- `website/public/docs/memory.en.md` - Memory documentation

**Priority:** Core

---

### 7. Scheduled Tasks (Heartbeat/Cron)

**Description:** Built-in cron skill enables scheduled check-ins, periodic digests, and proactive notifications. The heartbeat system provides regular agent health checks and can trigger scheduled content delivery to any channel.

**Key Files/Locations:**
- `src/copaw/app/crons/` - Cron system
- `src/copaw/app/crons/manager.py` - Cron orchestration
- `src/copaw/app/crons/heartbeat.py` - Heartbeat implementation
- `src/copaw/app/crons/executor.py` - Cron execution
- `src/copaw/app/crons/models.py` - Cron models
- `src/copaw/agents/skills/cron/` - Cron skill
- `src/copaw/cli/cron_cmd.py` - Cron CLI management
- `website/public/docs/heartbeat.en.md` - Heartbeat documentation

**Priority:** Core

---

## Priority Tier: Secondary

Important features that enhance CoPaw but are not strictly required for basic operation.

---

### 8. MCP (Model Context Protocol) Support

**Description:** Integration with the Model Context Protocol standard, allowing CoPaw to connect to MCP clients and servers for extended capabilities.

**Key Files/Locations:**
- `src/copaw/app/mcp/` - MCP implementation
- `src/copaw/app/mcp/manager.py` - MCP orchestration
- `src/copaw/app/mcp/watcher.py` - MCP file watcher
- `src/copaw/app/routers/mcp.py` - MCP API routes
- `website/public/docs/mcp.en.md` - MCP documentation

**Priority:** Secondary

---

### 9. Security Features (Tool Guard)

**Description:** File access guard system that prevents skills and agents from accessing sensitive paths. Skill scanner validates skills for security issues before execution. Provides sandboxing for agent operations.

**Key Files/Locations:**
- `src/copaw/security/` - Security implementations
- `src/copaw/security/tool_guard/` - File access control
- `src/copaw/security/skill_scanner/` - Skill validation
- `src/copaw/app/tool_guard_mixin.py` - Tool guard logic
- `website/public/docs/security.en.md` - Security documentation

**Priority:** Secondary

---

### 10. Local Model Support

**Description:** Run language models entirely on-premises without cloud API dependencies. Support for llama.cpp (cross-platform), MLX (Apple Silicon), and Ollama. Includes model download and management tools.

**Key Files/Locations:**
- `src/copaw/local_models/` - Local model support
- `src/copaw/local_models/manager.py` - Local model orchestration
- `src/copaw/local_models/chat_model.py` - Local chat interface
- `src/copaw/local_models/factory.py` - Local model factory
- `src/copaw/local_models/backends/` - Backend implementations
- `src/copaw/local_models/tag_parser.py` - Model tag parsing
- `src/copaw/app/routers/local_models.py` - Local models API
- `src/copaw/app/ollama_models.py` - Ollama model management
- `website/public/docs/models.en.md` - Models documentation (includes local)

**Priority:** Secondary

---

### 11. Desktop Application

**Description:** Cross-platform desktop application (Windows/macOS) with zero-configuration setup. Provides a self-contained package that does not require Python installation or command-line usage.

**Key Files/Locations:**
- `src/copaw/cli/desktop_cmd.py` - Desktop application logic
- `scripts/pack/` - Desktop app packaging scripts
- `website/public/docs/desktop.en.md` - Desktop documentation
- GitHub Releases for `CoPaw-Setup-*.exe` and `CoPaw-*-macOS.zip`

**Priority:** Secondary

---

### 12. CLI (Command-Line Interface)

**Description:** Comprehensive command-line tool `copaw` for initialization, agent management, channel configuration, skill management, cron jobs, and system maintenance. Provides alternative to web console for power users.

**Key Files/Locations:**
- `src/copaw/cli/` - CLI implementations
- `src/copaw/cli/main.py` - CLI entry point
- `src/copaw/cli/init_cmd.py` - Initialization
- `src/copaw/cli/agents_cmd.py` - Agent management
- `src/copaw/cli/channels_cmd.py` - Channel management
- `src/copaw/cli/skills_cmd.py` - Skill management
- `src/copaw/cli/cron_cmd.py` - Cron management
- `src/copaw/cli/providers_cmd.py` - Provider management
- `src/copaw/cli/update_cmd.py` - Updates
- `src/copaw/cli/shutdown_cmd.py` - Shutdown
- `website/public/docs/cli.en.md` - CLI documentation

**Priority:** Secondary

---

### 13. Docker Deployment

**Description:** Containerized deployment via Docker Hub. Supports volume mounts for data persistence and secrets management. Can connect to host services (like Ollama) via host networking.

**Key Files/Locations:**
- `docker-compose.yml` - Docker Compose configuration
- `Dockerfile` (in repo root or scripts/) - Image definition
- `scripts/docker_build.sh` - Docker build script
- `scripts/docker_sync_latest.sh` - Image sync script
- `website/public/docs/quickstart.en.md` - Deployment docs

**Priority:** Secondary

---

### 14. Voice/Audio Support

**Description:** Audio and video input capabilities in console chat, voice channel support, and speech processing. Includes integration for handling voice messages across channels.

**Key Files/Locations:**
- `src/copaw/app/channels/voice/` - Voice channel implementations
- `src/copaw/app/routers/voice.py` - Voice API routes
- `src/copaw/cli/channels_cmd.py` - Voice channel configuration

**Priority:** Secondary

---

## Summary

| Priority | Feature | Key Distinction |
|----------|---------|-----------------|
| Core | Multi-Channel Communication | Multi-platform messaging |
| Core | Web Console | User interface |
| Core | LLM Integration | AI model support |
| Core | Skills System | Extensibility |
| Core | Multi-Agent Support | Multiple agents |
| Core | Memory System | Persistence |
| Core | Scheduled Tasks | Automation |
| Secondary | MCP Support | Protocol integration |
| Secondary | Security Features | Safety |
| Secondary | Local Models | Privacy |
| Secondary | Desktop App | Ease of use |
| Secondary | CLI | Power users |
| Secondary | Docker | Deployment |
| Secondary | Voice/Audio | Multimodal |

## File Structure Overview

```
src/copaw/
  agents/           # Agent runtime, multi-agent, memory, channels, crons, skills, MCP
  app/              # Web API, routers, channels, crons, memory, skills, tools, MCP
  cli/              # Command-line interface
  config/           # Configuration management
  local_models/     # llama.cpp, MLX, Ollama support
  providers/        # OpenAI, Anthropic, Gemini, Ollama providers
  security/         # Tool guard, skill scanner
  token_usage/      # Token tracking
  tokenizer/        # Token counting
  tunnel/           # Tunneling support
  utils/            # Utilities

console/           # React web UI (TypeScript)
website/           # Documentation site
scripts/           # Build and deployment scripts
tests/             # Test suite
```
