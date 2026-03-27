# CoPaw Tech Stack

## Overview

CoPaw is a **Python monorepo** with a React/TypeScript frontend, supporting multi-channel messaging and local AI model integration.

## Languages and Versions

| Language | Version | Purpose |
|----------|---------|---------|
| **Python** | 3.10 - 3.13 (<3.14) | Backend runtime |
| **TypeScript** | 5.6.2 - 5.8.3 | Frontend (Console + Website) |
| **Node.js** | 20 | Frontend build tooling |

## Backend Stack

### Core Framework
- **FastAPI/uvicorn** - ASGI web server for the REST API
- **AgentScope** (1.0.17) - Multi-agent runtime framework
- **AgentScope Runtime** (1.1.1) - Runtime environment for agents

### Task Scheduling
- **APScheduler** (3.11.x) - Cron job scheduling for heartbeat and scheduled tasks

### Key Backend Libraries
| Library | Purpose |
|---------|---------|
| `httpx` | HTTP client for API calls |
| `pyyaml` | YAML configuration parsing |
| `python-dotenv` | Environment variable management |
| `packaging` | Package version handling |
| `shortuuid` | UUID generation |
| `json-repair` | JSON repair/parsing |
| `tzdata` | Timezone data |
| `aiofiles` | Async file I/O |
| `questionary` | Interactive CLI prompts |

## Frontend Stack

### Console (Main Web UI)
| Library | Version | Purpose |
|---------|---------|---------|
| **React** | 18 | UI framework |
| **Vite** | 6 | Build tool and dev server |
| **Ant Design** | 5 | Component library |
| **AgentScope Design System** | - | Custom design components |
| **Zustand** | 5 | State management |
| **React Router** | 7 | Client-side routing |
| **i18next** | - | Internationalization |
| **dnd-kit** | - | Drag-and-drop functionality |

### Website (Documentation)
| Library | Purpose |
|---------|---------|
| **React Markdown** | Markdown content rendering |
| **Mermaid** | Diagram generation |
| **Highlight.js** | Syntax highlighting |
| **Motion** | Animations |

## AI/ML Providers

### Cloud APIs
- **google-genai** - Google Gemini models
- **lark-oapi** - Feishu/Lark integration
- **twilio** - SMS/WhatsApp

### Local Models
| Library | Purpose |
|---------|---------|
| `transformers` | Hugging Face transformers |
| `ollama` | Local LLM integration (via ollama SDK) |
| `llama-cpp-python` | Llama model inference |
| `mlx-lm` | Apple MLX acceleration (macOS M-series) |
| `openai-whisper` | Speech recognition |
| `onnxruntime` | ONNX model execution |

## Messaging Channels

### Supported Platforms
| Platform | Library |
|----------|---------|
| Discord | `discord-py` (2.3+) |
| DingTalk | `dingtalk-stream` (0.24.3+) |
| Telegram | `python-telegram-bot` (20.0+) |
| Feishu/Lark | `lark-oapi` (1.5.3+) |
| WeCom | `wecom-aibot-python-sdk` |
| QQ | Custom WebSocket implementation |
| iMessage | Custom implementation |
| Matrix | `matrix-nio` |
| MQTT | `paho-mqtt` |
| SMS | `twilio` |

## Desktop Integration
- **playwright** (1.49+) - Browser automation with Chromium
- **pywebview** (4.0+) - Desktop webview wrapper
- **mss** - Screenshot capture

## Build System

### Python
- **setuptools** - Package building
- **pip/uv** - Package installation

### JavaScript
- **npm/pnpm** - Package management
- **Vite** - Bundler

### Monorepo Structure
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
└── deploy/               # Deployment files (Docker)
```

## Containerization

- **Multi-stage Dockerfile** - Node.js slim base, Debian runtime with Python 3, Chromium, XFCE
- **Docker Compose** - Single service deployment with named volumes
- **Supervisor** - Process management in containers
- **Xvfb** - Headless GUI support

## Code Quality Tools

| Tool | Purpose |
|------|---------|
| **Black** | Code formatting (line-length: 79) |
| **Flake8** | Python linting |
| **Pylint** | Advanced Python linting |
| **Mypy** | Type checking |
| **Prettier** | JS/TS formatting |
| **ESLint** | JS/TS linting |
| **Pre-commit** | Git hook automation |

## Optional Dependency Groups

| Group | Dependencies |
|-------|-------------|
| `dev` | pytest, pytest-asyncio, pre-commit, pytest-cov, hypothesis |
| `local` | huggingface_hub |
| `llamacpp` | llama-cpp-python |
| `mlx` | mlx-lm (macOS only) |
| `ollama` | ollama |
| `whisper` | openai-whisper |
| `full` | All of the above |

## Development Setup

```bash
# Install with dev dependencies
pip install -e ".[dev,full]"

# Install pre-commit hooks
pre-commit install

# Run pre-commit checks
pre-commit run --all-files

# Run tests
pytest
```
