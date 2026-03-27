# CoPaw Tech Stack Analysis

## Project Overview

CoPaw is a **personal AI assistant** that runs in your own environment, supporting multiple communication channels (DingTalk, Feishu, QQ, Discord, iMessage, etc.) and scheduled tasks. It uses a skill-based architecture where capabilities are extended through custom skills.

## Language(s) and Versions

| Language | Version |
|----------|---------|
| Python | 3.10 - 3.13 (requires >=3.10, <3.14) |
| TypeScript | 5.6.2 - 5.8.3 |
| Node.js | 20 (for frontend builds) |

## Backend Frameworks

| Framework | Purpose |
|-----------|---------|
| **FastAPI/uvicorn** | ASGI web server for the API |
| **AgentScope** | Multi-agent runtime framework |
| **AgentScope Runtime** | Runtime environment for agents |
| **APScheduler** | Cron job scheduling |

## Frontend Frameworks

### Console (Main UI)
- **React 18** with TypeScript
- **Vite 6** as build tool
- **Ant Design 5** UI component library
- **AgentScope Design System** (@agentscope-ai/design)
- **Zustand 5** for state management
- **React Router 7** for routing
- **i18next** for internationalization
- **dnd-kit** for drag-and-drop functionality

### Website (Documentation)
- **React 18** with TypeScript
- **Vite 6** as build tool
- **React Markdown** for content rendering
- **Mermaid** for diagrams
- **Highlight.js** for syntax highlighting
- **Motion** for animations

## Key Dependencies

### AI/ML Providers
- **google-genai** - Google Gemini models
- **transformers** - Hugging Face transformers
- **ollama** - Local LLM integration
- **llama-cpp-python** - Llama model inference
- **mlx-lm** - Apple MLX acceleration (macOS)
- **openai-whisper** - Speech recognition

### Messaging Channels
- **discord-py** - Discord integration
- **dingtalk-stream** - DingTalk integration
- **python-telegram-bot** - Telegram integration
- **twilio** - SMS/WhatsApp integration
- **lark-oapi** - Feishu/Lark integration
- **matrix-nio** - Matrix protocol
- **wecom-aibot-python-sdk** - WeCom integration
- **paho-mqtt** - MQTT support

### Infrastructure
- **httpx** - HTTP client
- **playwright** - Browser automation (with Chromium)
- **pywebview** - Desktop webview
- **uvicorn** - ASGI server
- **python-dotenv** - Environment management
- **pyyaml** - YAML configuration

## Build System

| Tool | Usage |
|------|-------|
| **setuptools** | Python packaging |
| **pip** | Python package installer |
| **npm/pnpm** | JavaScript package management |
| **Vite** | Frontend bundler |
| **TypeScript** | Type-safe JavaScript |

## Monorepo Structure

```
CoPaw/
├── src/copaw/           # Python backend package
│   ├── agents/          # Agent implementations
│   ├── app/             # Application logic, channels, routers
│   ├── cli/             # CLI commands
│   ├── config/          # Configuration handling
│   ├── local_models/    # Local model integrations
│   ├── providers/       # AI provider integrations
│   └── ...
├── console/              # Main React UI (TypeScript/Vite)
├── website/              # Documentation site (React/Vite)
├── tests/                # Python tests (unit/integrated)
├── scripts/              # Build and utility scripts
└── deploy/               # Deployment files
```

## Containerization

### Dockerfile (Multi-stage)
- **Base**: Node.js slim image
- **Runtime**: Debian-based with Python 3, Chromium, XFCE
- **Features**: Supervisor for process management, Xvfb for headless GUI

### Docker Compose
- Single service deployment
- Named volumes for data persistence
- Optional auth configuration

## CI/CD Setup

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `tests.yml` | Push/PR | Unit + integrated tests, coverage |
| `docker-release.yml` | Release | Multi-arch Docker builds to DockerHub + Aliyun ACR |
| `desktop-release.yml` | Release | Desktop application builds |
| `deploy-website.yml` | Push | Documentation deployment |
| `publish-pypi.yml` | Release | PyPI package publishing |
| `npm-format.yml` | PR | npm code formatting |
| `pre-commit.yml` | PR | Pre-commit hook validation |
| `issue-welcome.yml` | Issue opened | Automated issue responses |
| `pr-welcome.yml` | PR opened | Automated PR responses |

### Testing Strategy
- **pytest** with asyncio support
- Unit tests + integrated tests
- Multi-platform: Ubuntu, macOS, Windows
- Multi-Python: 3.10, 3.12, 3.13
- Coverage reporting with PR comments

## Code Quality Tools

| Tool | Purpose |
|------|---------|
| **Black** | Code formatting (line-length: 79) |
| **Flake8** | Linting |
| **Pylint** | Advanced linting with custom rules |
| **Mypy** | Type checking |
| **Prettier** | JavaScript/TypeScript formatting |
| **ESLint** | JavaScript/TypeScript linting |
| **Pre-commit** | Git hook automation |

## Development Dependencies

```
[dev]
- pytest>=8.3.5
- pytest-asyncio>=0.23.0
- pre-commit>=4.2.0
- pytest-cov>=6.2.1
- hypothesis>=6.0.0
```

## Optional Dependency Groups

| Group | Additional Dependencies |
|-------|------------------------|
| `local` | huggingface_hub |
| `llamacpp` | llama-cpp-python |
| `mlx` | mlx-lm (macOS only) |
| `ollama` | ollama |
| `whisper` | openai-whisper |
| `full` | All of the above |

## Summary

CoPaw is a well-structured **Python monorepo** with:
- A React/TypeScript frontend built with Vite
- A FastAPI-based Python backend
- Multi-channel messaging support (10+ platforms)
- Local AI model support (Ollama, llama.cpp, MLX, Transformers)
- Comprehensive CI/CD with GitHub Actions
- Docker containerization with multi-arch support
- Full test coverage across platforms
