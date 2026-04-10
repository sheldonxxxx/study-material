# nanobot Feature Index

## Project Overview

**nanobot** is an ultra-lightweight personal AI assistant (99% smaller than OpenClaw). It connects LLM providers to multiple chat platforms with built-in agent capabilities including memory, skills, cron scheduling, and MCP tool support.

---

## Core Features (Priority 1)

### 1. Agent Core Loop
**Description:** The central LLM-driven agent loop that orchestrates tool execution, memory management, and response generation.

**Key Files:**
- `nanobot/agent/loop.py` — Main agent loop (LLM <-> tool execution)
- `nanobot/agent/context.py` — Prompt builder
- `nanobot/agent/memory.py` — Persistent memory system
- `nanobot/agent/skills.py` — Skills loader
- `nanobot/agent/subagent.py` — Background task execution

**Priority:** CORE

---

### 2. Multi-Channel Chat Integration
**Description:** Connect to 12 different chat platforms simultaneously (Telegram, Discord, WhatsApp, WeChat, Feishu, DingTalk, Slack, Matrix, Email, QQ, Wecom, Mochat).

**Key Files:**
- `nanobot/channels/` — All channel implementations
- `nanobot/channels/manager.py` — Channel registry and routing
- `nanobot/channels/registry.py` — Plugin-based channel loader
- `nanobot/channels/telegram.py`, `discord.py`, `feishu.py`, `whatsapp.py`, etc.

**Priority:** CORE

---

### 3. Multi-Provider LLM Support
**Description:** Unified interface to 20+ LLM providers including OpenRouter, Anthropic, OpenAI, DeepSeek, Groq, MiniMax, Gemini, Ollama, vLLM, Azure OpenAI, Moonshot, Zhipu, Dashscope, StepFun, Mistral, SiliconFlow, AiHubMix, OpenAI Codex (OAuth), GitHub Copilot (OAuth).

**Key Files:**
- `nanobot/providers/registry.py` — Provider registry (2-step addition pattern)
- `nanobot/providers/base.py` — Base provider interface
- `nanobot/providers/openai_compat_provider.py` — OpenAI-compatible provider
- `nanobot/providers/azure_openai_provider.py`, `anthropic_provider.py`, etc.

**Priority:** CORE

---

### 4. Built-in Agent Tools
**Description:** Core tools available to the agent: shell execution, file read/write/edit, web search, web fetch, message sending, MCP tools, cron scheduling, and spawn subagents.

**Key Files:**
- `nanobot/agent/tools/shell.py` — Shell command execution
- `nanobot/agent/tools/filesystem.py` — File read/write/edit operations
- `nanobot/agent/tools/web.py` — Web search and fetch
- `nanobot/agent/tools/spawn.py` — Spawn subagents
- `nanobot/agent/tools/mcp.py` — MCP tool integration
- `nanobot/agent/tools/cron.py` — Cron scheduling

**Priority:** CORE

---

### 5. Persistent Memory System
**Description:** Token-based memory that maintains conversation history and context across sessions. Redesigned in v0.1.4 for reliability.

**Key Files:**
- `nanobot/agent/memory.py` — Memory management
- `nanobot/templates/memory/` — Memory templates
- `nanobot/skills/memory/` — Memory-related skills

**Priority:** CORE

---

### 6. Cron and Scheduled Tasks
**Description:** Schedule tasks using cron expressions with timezone support. Heartbeat system runs periodic tasks every 30 minutes.

**Key Files:**
- `nanobot/cron/` — Cron scheduler
- `nanobot/heartbeat/` — Periodic wake-up and task execution
- `nanobot/skills/cron/` — Cron-related skills

**Priority:** CORE

---

### 7. Skills System
**Description:** Extensible skill plugins including bundled skills for GitHub, Weather, Tmux, Cron, Summarize, Memory, ClawHub (skill marketplace), and Skill Creator.

**Key Files:**
- `nanobot/skills/` — Skills directory
- `nanobot/skills/github/`, `weather/`, `tmux/`, `cron/`, `summarize/`, `memory/`, `clawhub/`, `skill-creator/`

**Priority:** CORE

---

## Secondary Features (Priority 2)

### 8. MCP (Model Context Protocol) Support
**Description:** Connect external MCP tool servers (stdio or HTTP/SSE transport) as native agent tools. Auto-discovers and registers tools on startup.

**Key Files:**
- `nanobot/agent/tools/mcp.py` — MCP integration

**Priority:** SECONDARY

---

### 9. Web Search
**Description:** Multi-provider web search (Brave, Tavily, Jina, SearXNG, DuckDuckGo) with automatic fallback.

**Key Files:**
- `nanobot/agent/tools/web.py` — Web search and fetch

**Priority:** SECONDARY

---

### 10. Agent Social Network
**Description:** Connect to agent social networks (Moltbook, ClawdChat) via skill installation.

**Key Files:**
- `nanobot/skills/clawhub/` — ClawHub skill marketplace integration

**Priority:** SECONDARY

---

### 11. Multi-Instance Support
**Description:** Run multiple nanobot instances with separate configs, workspaces, and ports simultaneously.

**Key Files:**
- `nanobot/config/` — Configuration schema
- `nanobot/cli/` — CLI commands including `--config` flag

**Priority:** SECONDARY

---

### 12. Docker and Linux Service Support
**Description:** Docker deployment via docker-compose or standalone Dockerfile, plus systemd user service for auto-start.

**Key Files:**
- `Dockerfile`, `docker-compose.yml` — Docker support
- `.github/` — GitHub Actions workflows

**Priority:** SECONDARY

---

## Feature Summary Table

| Feature | Priority | Key Directory | Verification |
|---------|----------|---------------|--------------|
| Agent Core Loop | CORE | `nanobot/agent/` | Code verified |
| Multi-Channel (12 platforms) | CORE | `nanobot/channels/` | Code verified |
| Multi-Provider (20+ LLMs) | CORE | `nanobot/providers/` | Code verified |
| Built-in Tools | CORE | `nanobot/agent/tools/` | Code verified |
| Memory System | CORE | `nanobot/agent/memory.py` | Code verified |
| Cron/Heartbeat | CORE | `nanobot/cron/`, `nanobot/heartbeat/` | Code verified |
| Skills System | CORE | `nanobot/skills/` | Code verified |
| MCP Support | SECONDARY | `nanobot/agent/tools/mcp.py` | Code verified |
| Web Search | SECONDARY | `nanobot/agent/tools/web.py` | Code verified |
| Agent Social Network | SECONDARY | `nanobot/skills/clawhub/` | Code verified |
| Multi-Instance | SECONDARY | `nanobot/config/` | Code verified |
| Docker/Service | SECONDARY | `Dockerfile`, `docker-compose.yml` | Code verified |

---

## Code vs Documentation Discrepancies

None observed. All features documented in README.md have corresponding implementation in the codebase.

---

## Implementation Status

All listed features are implemented in code (not just documented). The project uses a clean plugin architecture:
- Providers: `nanobot/providers/registry.py` with ProviderSpec entries
- Channels: `nanobot/channels/registry.py` with channel plugins
- Skills: `nanobot/skills/` with per-skill directories
