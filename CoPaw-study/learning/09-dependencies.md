# CoPaw Dependencies

## Overview

CoPaw is a **Python-first monorepo** with significant JavaScript/TypeScript frontend components. Dependencies are managed via `pyproject.toml` (Python) and `package.json` (npm).

## Python Dependencies

### Core Dependencies (Required)

| Package | Version | Purpose |
|---------|---------|---------|
| `agentscope` | 1.0.17 | Multi-agent runtime |
| `agentscope-runtime` | 1.1.1 | Runtime environment |
| `httpx` | >=0.27.0 | HTTP client |
| `packaging` | >=24.0 | Version parsing |
| `discord-py` | >=2.3 | Discord integration |
| `dingtalk-stream` | >=0.24.3 | DingTalk integration |
| `uvicorn` | >=0.40.0 | ASGI server |
| `apscheduler` | >=3.11.2,<4 | Task scheduling |
| `playwright` | >=1.49.0 | Browser automation |
| `questionary` | >=2.1.1 | CLI prompts |
| `mss` | >=9.0.0 | Screenshot capture |
| `reme-ai` | 0.3.1.3 | Memory system |
| `transformers` | >=4.30.0 | Hugging Face models |
| `python-dotenv` | >=1.0.0 | Env vars |
| `python-socks` | >=2.5.3 | SOCKS proxy |
| `onnxruntime` | <1.24 | ONNX inference |
| `lark-oapi` | >=1.5.3 | Feishu/Lark API |
| `python-telegram-bot` | >=20.0 | Telegram |
| `twilio` | >=9.10.2 | SMS/WhatsApp |
| `pywebview` | >=4.0 | Desktop webview |
| `aiofiles` | >=24.1.0 | Async file I/O |
| `paho-mqtt` | >=2.0.0 | MQTT protocol |
| `wecom-aibot-python-sdk` | 1.0.1 | WeCom integration |
| `matrix-nio` | >=0.24.0 | Matrix protocol |
| `shortuuid` | >=1.0.0 | UUID generation |
| `google-genai` | >=1.67.0 | Google AI |
| `tzdata` | >=2024.1 | Timezones |
| `pyyaml` | >=6.0 | YAML parsing |
| `json-repair` | >=0.30.0 | JSON repair |

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | >=8.3.5 | Testing |
| `pytest-asyncio` | >=0.23.0 | Async test support |
| `pre-commit` | >=4.2.0 | Git hooks |
| `pytest-cov` | >=6.2.1 | Coverage |
| `hypothesis` | >=6.0.0 | Property-based testing |

### Optional Dependencies

| Group | Packages |
|-------|----------|
| `local` | `huggingface_hub>=0.20.0` |
| `llamacpp` | `llama-cpp-python>=0.3.0` |
| `mlx` | `mlx-lm>=0.10.0` (macOS only) |
| `ollama` | `ollama>=0.6.1` |
| `whisper` | `openai-whisper>=20231117` |
| `full` | All optional groups combined |

### Dependency Constraints

- `apscheduler`: Strict upper bound `<4` (breaking API changes in v4)
- `onnxruntime`: Strict upper bound `<1.24` (compatibility)
- `mlx-lm`: Platform-gated to `sys_platform == 'darwin'`

## JavaScript Dependencies

### Console (package.json in `console/`)

| Package | Purpose |
|---------|---------|
| `react` 18 | UI framework |
| `react-dom` | React DOM renderer |
| `react-router-dom` | Routing |
| `antd` 5 | Component library |
| `@agentscope-ai/design` | Design system |
| `zustand` 5 | State management |
| `i18next` | i18n |
| `dnd-kit` | Drag and drop |
| `vite` 6 | Build tool |
| `typescript` | Type system |
| `@typescript-eslint/*` | TS linting |

### Website (package.json in `website/`)

| Package | Purpose |
|---------|---------|
| `react` 18 | UI framework |
| `react-markdown` | Markdown rendering |
| `mermaid` | Diagrams |
| `highlight.js` | Syntax highlighting |
| `motion` | Animations |

## Version Management

### Python Version Policy
- **Minimum**: Python 3.10
- **Maximum**: Python 3.13 (<3.14)
- **Tested versions**: 3.10, 3.12, 3.13

### Dependency Update Strategy
- Lock files for npm (`package-lock.json`)
- pip cache in CI
- Regular updates in development

### Security Considerations

| Concern | Mitigation |
|---------|------------|
| `onnxruntime` version cap | Known compatibility issues with newer versions |
| `discord-py` v2.3+ required | API changes in v2 |
| `python-telegram-bot` v20+ | API changes |
| `llama-cpp-python` | Prebuilt wheels not always available |

## Installing Dependencies

```bash
# Core only
pip install copaw

# With dev tools
pip install -e ".[dev]"

# With all optional (full)
pip install -e ".[full]"

# Specific extras
pip install -e ".[ollama,llamacpp]"
```

## CI Dependency Handling

In GitHub Actions (tests.yml):

```bash
# Ubuntu (all extras)
pip install -e ".[dev,full]"

# macOS (special handling)
pip install -e ".[dev,local,ollama]"
pip install 'mlx-lm>=0.10.0'  # macOS only
pip install --only-binary=llama-cpp-python 'llama-cpp-python>=0.3.0' || echo "skipped"
```

## Notable External Services

| Service | SDK/Integration |
|---------|-----------------|
| DashScope | Environment variable `DASHSCOPE_API_KEY` |
| ModelScope | `modelscope` SDK via agentscope |
| Ollama | REST API (local) |
| Google Gemini | `google-genai` |
| Docker Hub | Container registry |
| Aliyun ACR | `agentscope-registry.ap-southeast-1.cr.aliyuncs.com` |

## Package Data

Setuptools includes non-code files:

```toml
[tool.setuptools.package-data]
"copaw" = [
    "console/**",              # Built frontend
    "agents/md_files/**",      # Agent prompts
    "agents/skills/**",        # Built-in skills
    "tokenizer/**",            # Tokenizers
    "security/tool_guard/rules/**",
    "security/skill_scanner/rules/**",
    "security/skill_scanner/data/**",
]
```
