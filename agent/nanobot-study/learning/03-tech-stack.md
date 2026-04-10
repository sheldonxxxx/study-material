# Tech Stack

## Runtime Requirements

| Requirement | Value |
|-------------|-------|
| Python Minimum | 3.11 |
| Python Supported | 3.11, 3.12, 3.13 |
| Package Installer | uv (Astral) |
| Build System | hatchling |

## Core Dependencies

### AI/ML Providers

| Library | Version | Purpose |
|---------|---------|---------|
| anthropic | >=0.45.0,<1.0.0 | Anthropic Claude API client |
| openai | >=2.8.0 | OpenAI API client |
| tiktoken | >=0.12.0,<1.0.0 | Tokenizer for OpenAI models |

### Data Models and Settings

| Library | Version | Purpose |
|---------|---------|---------|
| pydantic | >=2.12.0,<3.0.0 | Data validation |
| pydantic-settings | >=2.12.0,<3.0.0 | Settings management |

### CLI and Terminal UI

| Library | Version | Purpose |
|---------|---------|---------|
| typer | >=0.20.0,<1.0.0 | CLI framework (based on Click) |
| rich | >=14.0.0,<15.0.0 | Rich text formatting |
| prompt-toolkit | >=3.0.50,<4.0.0 | Terminal UI components |
| questionary | >=2.0.0,<3.0.0 | Interactive prompts |

### Web and Network

| Library | Version | Purpose |
|---------|---------|---------|
| httpx | >=0.28.0,<1.0.0 | HTTP client (sync/async) |
| websockets | >=16.0,<17.0 | WebSocket server/client |
| websocket-client | >=1.9.0,<2.0.0 | WebSocket client |
| python-socketio | >=5.16.0,<6.0.0 | Socket.IO client |

### Messaging Platform SDKs

| Library | Version | Platform |
|---------|---------|----------|
| python-telegram-bot | >=22.6,<23.0 | Telegram |
| slack-sdk | >=3.39.0,<4.0.0 | Slack |
| lark-oapi | >=1.5.0,<2.0.0 | Feishu/Lark |
| dingtalk-stream | >=0.24.0,<1.0.0 | DingTalk |
| qq-botpy | >=1.2.0,<2.0.0 | QQ |
| matrix-nio | >=0.25.2 | Matrix |
| wecom-aibot-sdk-python | >=0.1.5 | WeCom |

### Utilities

| Library | Version | Purpose |
|---------|---------|---------|
| loguru | >=0.7.3,<1.0.0 | Logging |
| croniter | >=6.0.0,<7.0.0 | Cron parsing |
| msgpack | >=1.1.0,<2.0.0 | Serialization |
| json-repair | >=0.57.0,<1.0.0 | JSON repair |
| mcp | >=1.26.0,<2.0.0 | Model Context Protocol |

### Development Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| pytest | >=9.0.0,<10.0.0 | Testing framework |
| pytest-asyncio | >=1.3.0,<2.0.0 | Async test support |
| pytest-cov | >=6.0.0,<7.0.0 | Coverage plugin |
| ruff | >=0.1.0 | Linter |

## WhatsApp Bridge (Node.js)

**Runtime:** Node.js >=20.0.0

| Package | Version | Purpose |
|---------|---------|---------|
| @whiskeysockets/baileys | 7.0.0-rc.9 | WhatsApp Web API |
| ws | ^8.17.1 | WebSocket |
| qrcode-terminal | ^0.12.0 | QR code display |
| pino | ^9.0.0 | Logging |

## Build System

```toml
[tool.hatch.build.targets.wheel.force-include]
"bridge" = "nanobot/bridge"  # TypeScript bundled into Python wheel
```

## Docker Configuration

**Base Image:** `ghcr.io/astral-sh/uv:python3.12-bookworm-slim`

Multi-service setup with Python 3.12 + Node.js 20 for WhatsApp bridge.

## Code Quality Tools

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]
ignore = ["E501"]
```

## LLM Providers Supported (20+)

| Provider | Backend | Default API Base |
|----------|---------|------------------|
| OpenAI | openai_compat | SDK default |
| Anthropic | anthropic | - |
| OpenRouter | openai_compat | openrouter.ai/api/v1 |
| DeepSeek | openai_compat | api.deepseek.com |
| Gemini | openai_compat | generativelanguage.googleapis.com |
| Azure OpenAI | azure_openai | - |
| Ollama | openai_compat | localhost:11434/v1 |
| vLLM | openai_compat | - |
| Groq | openai_compat | api.groq.com/openai/v1 |
| And 10+ more | openai_compat | Various |

## Channels Supported (12)

Telegram, Discord, Slack, DingTalk, Feishu/Lark, WeChat Work (Wecom), WeChat, QQ, Matrix, Email, MoChat, WhatsApp (via TS bridge)
