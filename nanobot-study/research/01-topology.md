# Nanobot Project Topology

## Project Overview

**nanobot** is a lightweight personal AI assistant framework written in Python (3.11+). It provides a modular agent system with support for multiple chat channels (Telegram, Discord, Slack, WhatsApp, DingTalk, Feishu, WeChat Work, QQ, Matrix, etc.) and multiple LLM providers (Anthropic, OpenAI, Azure OpenAI, OpenRouter, DeepSeek, Ollama, and more).

The project consists of three main components:
1. **Python package** (`nanobot/`) - Core agent framework
2. **WhatsApp bridge** (`bridge/`) - TypeScript/Baleiys-based WhatsApp connector
3. **Supporting files** - Docker, configs, documentation

---

## Root Directory Structure

```
/Users/sheldon/Documents/claw/reference/nanobot/
|-- pyproject.toml           # Python project configuration (hatch build system)
|-- README.md                # 57KB comprehensive documentation
|-- SECURITY.md             # Security policy
|-- CONTRIBUTING.md         # Contribution guidelines
|-- COMMUNICATION.md        # Communication guidelines
|-- Dockerfile              # Docker container definition
|-- docker-compose.yml      # Multi-container Docker setup
|-- core_agent_lines.sh     # Shell script utility
|-- nanobot_arch.png        # Architecture diagram (643KB)
|-- nanobot_logo.png        # Logo image (191KB)
|--
|-- nanobot/                # Main Python package (AI agent framework)
|-- bridge/                 # TypeScript WhatsApp bridge
|-- case/                   # (empty directory, potential placeholder)
|-- tests/                  # Symlink/copy of nanobot structure for testing
|-- docs/                   # Documentation files
|-- .github/workflows/      # CI/CD GitHub Actions
|-- .git/                   # Git repository
```

---

## Python Package (`nanobot/`)

**Location:** `/Users/sheldon/Documents/claw/reference/nanobot/nanobot/`

**Entry Points:**
- `python -m nanobot` → `__main__.py` → `cli.commands:app`
- `nanobot` CLI command (installed via pyproject.toml scripts)

### Package Structure

```
nanobot/
|-- __init__.py              # Package init, exports __version__ = "0.1.4.post5"
|-- __main__.py              # Module entry point (python -m nanobot)
|--
|-- agent/                   # Core agent processing engine
|   |-- __init__.py
|   |-- loop.py              # AgentLoop: main processing engine (45KB)
|   |-- context.py            # ContextBuilder for prompt construction (8KB)
|   |-- memory.py             # MemoryConsolidator (14KB)
|   |-- skills.py             # Skill loading and management (8KB)
|   |-- subagent.py           # SubagentManager for spawning sub-agents (9KB)
|   |-- tools/                # Built-in agent tools
|   |   |-- README.md
|   |   |-- cron/             # Cron scheduling tool
|   |   |-- filesystem.py      # File read/write/edit tools
|   |   |-- web.py            # Web fetch and search tools
|   |   |-- shell.py          # Shell execution tool
|   |   |-- spawn.py          # Process spawning tool
|   |   |-- message.py        # Message tool
|   |   |-- mcp.py            # MCP (Model Context Protocol) tool
|   |   |-- registry.py       # Tool registry
|   |   |-- skill-creator/    # Skill creation tool
|   |   |-- memory/           # Memory-related skills
|   |   |-- github/           # GitHub integration skills
|   |   |-- tmux/             # Tmux integration skills
|   |   |-- summarize/        # Summarization skill
|   |   |-- weather/          # Weather query skill
|   |   |-- clawhub/          # Clawhub integration
|   |
|-- bus/                     # Message bus (event system)
|   |-- __init__.py
|   |-- events.py             # InboundMessage, OutboundMessage dataclasses
|   |-- queue.py              # MessageBus queue implementation
|
|-- channels/                 # Chat platform integrations
|   |-- __init__.py
|   |-- base.py               # Channel base class
|   |-- manager.py            # ChannelManager for routing
|   |-- registry.py           # Channel registry
|   |-- telegram.py            # Telegram integration (37KB)
|   |-- discord.py            # Discord integration (15KB)
|   |-- slack.py               # Slack integration (12KB)
|   |-- dingtalk.py           # DingTalk integration (23KB)
|   |-- feishu.py             # Feishu/Lark integration (49KB)
|   |-- wecom.py              # WeChat Work integration (13KB)
|   |-- weixin.py             # WeChat integration (39KB)
|   |-- qq.py                 # QQ bot integration (22KB)
|   |-- matrix.py             # Matrix protocol integration (30KB)
|   |-- email.py              # Email channel integration (17KB)
|   |-- mochat.py            # MoChat integration (37KB)
|   |-- whatsapp.py           # WhatsApp integration (10KB)
|
|-- cli/                      # Command-line interface
|   |-- __init__.py
|   |-- commands.py           # Typer CLI commands (45KB)
|   |-- models.py             # CLI data models
|   |-- onboard.py            # Onboarding flow (35KB)
|   |-- stream.py             # Streaming output handlers
|   |-- events.py             # CLI event handling
|
|-- command/                  # Slash command system
|   |-- __init__.py
|   |-- router.py             # CommandRouter
|   |-- builtin.py            # Built-in command handlers
|
|-- config/                   # Configuration management
|   |-- __init__.py
|   |-- schema.py             # Pydantic configuration schemas (11KB)
|   |-- service.py            # Config loading service (14KB)
|   |-- loader.py             # Config file loader
|   |-- paths.py              # Path utilities
|
|-- providers/                # LLM provider integrations
|   |-- __init__.py
|   |-- base.py               # LLMProvider abstract base class
|   |-- registry.py           # Provider registry
|   |-- anthropic_provider.py # Anthropic/Claude provider (16KB)
|   |-- openai_compat_provider.py # OpenAI-compatible provider (21KB)
|   |-- openai_codex_provider.py # OpenAI Codex provider (12KB)
|   |-- azure_openai_provider.py # Azure OpenAI provider (11KB)
|   |-- transcription.py      # Speech-to-text transcription
|
|-- session/                  # Conversation session management
|   |-- __init__.py
|   |-- manager.py            # SessionManager (9KB)
|
|-- cron/                     # Cron scheduling service
|   |-- __init__.py
|   |-- service.py            # CronService (6KB)
|   |-- types.py              # Cron type definitions
|
|-- heartbeat/                # Heartbeat/monitoring service
|   |-- __init__.py
|   |-- service.py            # HeartbeatService (6KB)
|
|-- skills/                   # Agent skills/templates
|   |-- __init__.py
|   |-- base.py               # Skill base class
|   |-- registry.py           # Skill registry
|   |-- cron.py               # Cron skill
|   |-- filesystem.py         # Filesystem skill
|   |-- mcp.py                # MCP skill
|   |-- message.py            # Message skill
|   |-- shell.py              # Shell skill
|   |-- spawn.py              # Spawn skill
|   |-- web.py                # Web skill
|   |-- templates/            # Skill templates
|   |   |-- AGENTS.md
|   |   |-- HEARTBEAT.md
|   |   |-- SOUL.md
|   |   |-- TOOLS.md
|   |   |-- USER.md
|   |   |-- memory/
|
|-- utils/                    # Utility modules
|   |-- __init__.py
|   |-- helpers.py            # Helper functions (10KB)
|   |-- evaluator.py          # Expression evaluator
|
|-- security/                 # Security utilities
|   |-- __init__.py
|   |-- network.py            # Network security utilities (3KB)
|
|-- templates/               # Prompt templates (markdown)
    |-- AGENTS.md
    |-- HEARTBEAT.md
    |-- SOUL.md
    |-- TOOLS.md
    |-- USER.md
    |-- memory/
```

---

## WhatsApp Bridge (`bridge/`)

**Location:** `/Users/sheldon/Documents/claw/reference/nanobot/bridge/`

**Purpose:** WhatsApp Web bridge using Baileys library, connects to Python backend via WebSocket.

### Structure

```
bridge/
|-- package.json              # Node.js project config (version 0.1.0)
|-- tsconfig.json             # TypeScript configuration
|-- src/
|   |-- index.ts              # Entry point, starts BridgeServer
|   |-- server.ts             # WebSocket server for Python bridge communication
|   |-- whatsapp.ts           # WhatsApp client using Baileys (9KB)
|   |-- types.d.ts           # TypeScript type definitions
```

**Key Technologies:**
- `@whiskeysockets/baileys` v7.0.0-rc.9
- `ws` WebSocket library
- `pino` logging
- TypeScript 5.4+

**Note:** The bridge is bundled into the Python wheel via `pyproject.toml`:
```toml
[tool.hatch.build.targets.wheel.force-include]
"bridge" = "nanobot/bridge"
```

---

## Configuration Schema (`nanobot/config/schema.py`)

Key configuration classes:

| Class | Purpose |
|-------|---------|
| `ChannelsConfig` | Chat channel settings (streaming, retries) |
| `AgentDefaults` | Default agent settings (model, provider, tokens) |
| `ProvidersConfig` | LLM provider configs (api_key, api_base for multiple providers) |
| `Base` | Pydantic base with camelCase/snake_case aliasing |

**Supported LLM Providers:**
- `anthropic`, `openai`, `openrouter`, `deepseek`, `groq`
- `azure_openai`, `zhipu`, `dashscope`, `vllm`, `ollama`
- `ovms`, `gemini`, `moonshot`, `minimax`, `mistral`, `stepfun`
- `custom` (any OpenAI-compatible endpoint)

---

## Entry Points

### 1. CLI Entry Point
**Command:** `nanobot` (installed via pyproject.toml scripts)

```python
# Entry: nanobot.cli.commands:app
# Framework: Typer
```

### 2. Module Entry Point
**Command:** `python -m nanobot`

```python
# Entry: nanobot/__main__.py
# Imports: from nanobot.cli.commands import app
```

### 3. WhatsApp Bridge Entry
**Command:** `npm start` in `bridge/` directory

```typescript
// Entry: bridge/src/index.ts
// Framework: Baileys + ws WebSocket
```

---

## Directory Organization Patterns

| Pattern | Directory | Purpose |
|---------|-----------|---------|
| **`*/__init__.py`** | All Python packages | Package exports, lazy imports |
| **`*/tools/`** | `agent/tools/` | Plugin-like tool implementations |
| **`*/skills/`** | `agent/skills/`, `nanobot/skills/` | Agent capabilities |
| **`*/templates/`** | `nanobot/templates/` | Markdown prompt templates |
| **`base.py`** | channels/, providers/ | Abstract base class pattern |
| **`registry.py`** | channels/, providers/, skills/, tools/ | Plugin registration |
| **`manager.py`** | session/, channels/ | Central management |

---

## Key Data Flow

```
Chat Channel (Telegram/Slack/Discord/etc.)
    |
    v
Channel Adapter (channels/*.py)
    |
    v
InboundMessage --> MessageBus (bus/queue.py)
    |
    v
AgentLoop (agent/loop.py)
    |-- ContextBuilder (builds prompt with history/memory)
    |-- SessionManager (conversation history)
    |-- ToolRegistry (executes tools)
    |-- LLMProvider (calls LLM)
    |
    v
OutboundMessage --> MessageBus --> Channel Adapter
    |
    v
Chat Channel Response
```

---

## Key Classes and Responsibilities

| Class | Module | Responsibility |
|-------|--------|----------------|
| `AgentLoop` | `agent/loop.py` | Core processing engine, tool execution loop |
| `ContextBuilder` | `agent/context.py` | Constructs prompts with history, memory, skills |
| `MemoryConsolidator` | `agent/memory.py` | Consolidates conversation history |
| `SessionManager` | `session/manager.py` | Manages conversation sessions (JSONL storage) |
| `Session` | `session/manager.py` | Single conversation session |
| `ToolRegistry` | `agent/tools/registry.py` | Registers and executes tools |
| `LLMProvider` | `providers/base.py` | Abstract base for LLM integrations |
| `ChannelManager` | `channels/manager.py` | Routes messages to/from channels |
| `CommandRouter` | `command/router.py` | Routes slash commands |
| `MessageBus` | `bus/queue.py` | Async message queue between channels and agent |
| `BridgeServer` | `bridge/server.ts` | WebSocket bridge for WhatsApp |

---

## Dependencies Summary

**Python Version:** >= 3.11

**Key Python Dependencies:**
- `typer` - CLI framework
- `anthropic` - Claude API
- `pydantic` / `pydantic-settings` - Data validation
- `loguru` - Logging
- `rich` - Terminal formatting
- `httpx` - HTTP client
- `websockets` / `websocket-client` - WebSocket support
- `python-socketio` - Socket.IO protocol
- `croniter` - Cron parsing
- `tiktoken` - Token counting
- `mcp` - Model Context Protocol
- Channel SDKs: `python-telegram-bot`, `slack-sdk`, `dingtalk-stream`, `lark-oapi`, `qq-botpy`

**Build System:** hatchling

---

## Testing Structure

Tests directory contains channel plugin tests:
- `tests/agent/` - Agent loop tests
- `tests/channels/` - Channel adapter tests
- `tests/cli/` - CLI tests
- `tests/providers/` - Provider tests
- `tests/tools/` - Tool tests
- `tests/test_docker.sh` - Docker integration test

---

## Output File

OUTPUT_FILE: /Users/sheldon/Documents/claw/nanobot-study/research/01-topology.md
