# Nanobot Tech Stack Analysis

**Project:** nanobot-ai (A lightweight personal AI assistant framework)
**Version:** 0.1.4.post5
**Last Updated:** 2026-03-26

---

## Python Version and Runtime Requirements

| Requirement | Value |
|-------------|-------|
| Python Minimum | 3.11 |
| Python Supported | 3.11, 3.12, 3.13 |
| Package Installer | uv (Astral) |
| Build System | hatchling |

**Runtime Base Image (Docker):** `ghcr.io/astral-sh/uv:python3.12-bookworm-slim`

---

## Key Dependencies

### AI/ML Providers

| Library | Version | Purpose |
|---------|---------|---------|
| anthropic | >=0.45.0,<1.0.0 | Anthropic Claude API client |
| openai | >=2.8.0 | OpenAI API client (any version) |
| tiktoken | >=0.12.0,<1.0.0 | Tokenizer for OpenAI models |

### Data Models and Settings

| Library | Version | Purpose |
|---------|---------|---------|
| pydantic | >=2.12.0,<3.0.0 | Data validation using Python type hints |
| pydantic-settings | >=2.12.0,<3.0.0 | Settings management for Pydantic |

### CLI and Terminal UI

| Library | Version | Purpose |
|---------|---------|---------|
| typer | >=0.20.0,<1.0.0 | CLI framework (based on Click) |
| rich | >=14.0.0,<15.0.0 | Rich text and beautiful formatting in terminal |
| prompt-toolkit | >=3.0.50,<4.0.0 | Terminal UI components |
| questionary | >=2.0.0,<3.0.0 | Interactive prompts for CLI |

### Web and Network

| Library | Version | Purpose |
|---------|---------|---------|
| httpx | >=0.28.0,<1.0.0 | HTTP client (sync/async) |
| websockets | >=16.0,<17.0 | WebSocket server/client |
| websocket-client | >=1.9.0,<2.0.0 | WebSocket client |
| python-socketio | >=5.16.0,<6.0.0 | Socket.IO client |

### Search and Web Scraping

| Library | Version | Purpose |
|---------|---------|---------|
| ddgs | >=9.5.5,<10.0.0 | DuckDuckGo search |
| readability-lxml | >=0.8.4,<1.0.0 | Article text extraction |

### Messaging Platform Integrations

| Library | Version | Purpose |
|---------|---------|---------|
| python-telegram-bot | >=22.6,<23.0 | Telegram bot framework |
| dingtalk-stream | >=0.24.0,<1.0.0 | DingTalk streaming API |
| lark-oapi | >=1.5.0,<2.0.0 | Feishu/Lark API |
| slack-sdk | >=3.39.0,<4.0.0 | Slack API |
| slackify-markdown | >=0.2.0,<1.0.0 | Convert markdown to Slack format |
| qq-botpy | >=1.2.0,<2.0.0 | QQ bot framework |
| matrix-nio | >=0.25.2 | Matrix protocol client |
| wecom-aibot-sdk-python | >=0.1.5 | WeCom bot (optional: wecom) |

### SOCKS Proxy Support

| Library | Version | Purpose |
|---------|---------|---------|
| python-socks | >=2.8.0,<3.0.0 | SOCKS protocol client |
| socksio | >=1.0.0,<2.0.0 | SOCKS protocol implementation |

### Utilities

| Library | Version | Purpose |
|---------|---------|---------|
| loguru | >=0.7.3,<1.0.0 | Logging library |
| croniter | >=6.0.0,<7.0.0 | Cron parsing |
| msgpack | >=1.1.0,<2.0.0 | MessagePack serialization |
| json-repair | >=0.57.0,<1.0.0 | Repair invalid JSON |
| chardet | >=3.0.2,<6.0.0 | Character encoding detection |
| oauth-cli-kit | >=0.1.3,<1.0.0 | OAuth utilities |
| mcp | >=1.26.0,<2.0.0 | Model Context Protocol |

### Optional Dependencies

#### WeChat (weixin)
- qrcode[pil] >=8.0
- pycryptodome >=3.20.0

#### Matrix (matrix)
- matrix-nio[e2e] >=0.25.2
- mistune >=3.0.0,<4.0.0
- nh3 >=0.2.17,<1.0.0

#### LangSmith Observability
- langsmith >=0.1.0

### Development Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| pytest | >=9.0.0,<10.0.0 | Testing framework |
| pytest-asyncio | >=1.3.0,<2.0.0 | Async test support |
| pytest-cov | >=6.0.0,<7.0.0 | Coverage plugin |
| ruff | >=0.1.0 | Linter (replaces flake8, isort) |

---

## Build System and Packaging

| Aspect | Details |
|--------|---------|
| Build Backend | hatchling |
| Package Manager | uv (Astral) |
| Python Versions | 3.11, 3.12, 3.13 |
| Entry Point | `nanobot = nanobot.cli.commands:app` |

### Package Build Configuration

The `pyproject.toml` configures hatchling to:
- Include Python files from `nanobot/`
- Include templates from `nanobot/templates/**/*.md`
- Include skills from `nanobot/skills/**/*.md` and `*.sh`
- Force-include `bridge/` directory as `nanobot/bridge` (Node.js WhatsApp bridge)

### Code Quality Tools

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]
ignore = ["E501"]
```

Ruff is configured with:
- Target Python 3.11
- 100 character line length
- Enabled rules: E (errors), F (pyflakes), I (isort), N (naming), W (warnings)
- Ignored: E501 (line too long)

### Testing Configuration

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

---

## Project Structure

```
nanobot/
├── agent/              # AI agent core (loop, context, memory, skills)
├── bus/                # Event bus for inter-component communication
├── channels/           # Messaging platform integrations
│   ├── anthropic_provider.py
│   ├── openai_compat_provider.py
│   ├── azure_openai_provider.py
│   ├── openai_codex_provider.py
│   ├── telegram.py, slack.py, discord.py, dingtalk.py, feishu.py
│   ├── weixin.py, wecom.py, qq.py, matrix.py, whatsapp.py
├── cli/                # CLI command implementation
├── command/            # Command handlers
├── config/             # Configuration management
├── cron/               # Scheduled task support
├── heartbeat/          # Health check system
├── providers/          # LLM provider abstraction
├── security/           # Security utilities
├── session/            # Session management
├── skills/             # Agent skill definitions
├── templates/          # Response templates
└── utils/              # Utility functions

bridge/                 # Node.js WhatsApp bridge
├── package.json        # Node dependencies
├── src/                # TypeScript source
└── dist/               # Compiled JavaScript
```

---

## Docker Configuration

### Base Image
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim
```

### Multi-Service Setup
- **Python 3.12** in slim Debian Bookworm
- **Node.js 20** for WhatsApp bridge
- Installed via NodeSource apt repository

### Resource Limits
```yaml
deploy:
  resources:
    limits:
      cpus: '1'
      memory: 1G
    reservations:
      cpus: '0.25'
      memory: 256M
```

### Exposed Ports
- **18790** - Gateway default port

### Volume Mounts
- `~/.nanobot:/root/.nanobot` - Persistent configuration

---

## CI/CD Pipeline

**File:** `.github/workflows/ci.yml`

### Trigger Conditions
- Push to `main` or `nightly` branches
- Pull requests to `main` or `nightly` branches

### Test Matrix
```yaml
python-version: ["3.11", "3.12", "3.13"]
```

### Pipeline Steps

1. **Checkout** - `actions/checkout@v4`
2. **Setup Python** - `actions/setup-python@v5` with specified version
3. **Install uv** - `astral-sh/setup-uv@v4`
4. **System Dependencies** - `libolm-dev` and `build-essential`
5. **Install Dependencies** - `uv sync --all-extras`
6. **Run Tests** - `uv run pytest tests/`

---

## WhatsApp Bridge (Node.js)

**Location:** `bridge/`

### Runtime Requirements
- **Node.js:** >=20.0.0
- **Package Manager:** npm

### Key Dependencies
| Package | Version | Purpose |
|---------|---------|---------|
| @whiskeysockets/baileys | 7.0.0-rc.9 | WhatsApp Web API |
| ws | ^8.17.1 | WebSocket |
| qrcode-terminal | ^0.12.0 | QR code display |
| pino | ^9.0.0 | Logging |

### Build Process
```dockerfile
WORKDIR /app/bridge
RUN npm install && npm run build
```

### TypeScript Configuration
- TypeScript ^5.4.0
- Compiled to JavaScript in `dist/`
- ES modules (`"type": "module"`)

---

## Key Architectural Patterns

### Multi-Provider LLM Support
The project supports multiple LLM providers through a provider registry:
- Anthropic (Claude)
- OpenAI (GPT models)
- OpenAI Compatible APIs
- Azure OpenAI
- OpenAI Codex

### Multi-Channel Messaging
Unified interface for multiple messaging platforms:
- Telegram, Slack, Discord
- DingTalk, Feishu/Lark, WeCom
- WeChat, QQ
- WhatsApp (via Node.js bridge)
- Email
- Matrix

### Event-Driven Architecture
- Event bus for component communication
- Session management for conversation state
- Cron-based scheduled tasks

### Configuration Management
- Pydantic Settings for type-safe configuration
- Environment variable support
- Persistent config in `~/.nanobot`

---

## Summary

Nanobot is a **Python-first, multi-channel AI assistant framework** with:

- **Python 3.11+** with modern tooling (uv, hatchling, ruff)
- **Pydantic v2** for data validation
- **Rich CLI** with Typer-based commands
- **Multiple LLM providers** (Anthropic, OpenAI, Azure)
- **12+ messaging platform integrations**
- **TypeScript WhatsApp bridge** running on Node.js 20
- **Docker-native** with resource limits and multi-service compose
- **Comprehensive CI** testing across Python 3.11-3.13
