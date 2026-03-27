# Dependencies - Hermes Agent

## Dependency Philosophy

> "pinned to known-good ranges to limit supply chain attack surface"

Core dependencies use upper-bound version constraints (`<next_major>`) to allow patch updates while preventing breaking changes.

## Core Dependencies

### LLM Clients

| Package | Version | Purpose |
|---------|---------|---------|
| `openai` | >=2.21.0,<3 | OpenAI-compatible API (OpenRouter, custom endpoints) |
| `anthropic` | >=0.39.0,<1 | Anthropic Claude API |

### CLI & UI

| Package | Version | Purpose |
|---------|---------|---------|
| `fire` | >=0.7.1,<1 | CLI argument parsing |
| `prompt_toolkit` | >=3.0.52,<4 | Interactive TUI (used directly by cli.py) |
| `rich` | >=14.3.3,<15 | Rich text formatting in terminal |
| `simple-term-menu` | >=1.0,<2 | Terminal menu (optional, CLI extra) |

### HTTP & Networking

| Package | Version | Purpose |
|---------|---------|---------|
| `httpx` | >=0.28.1,<1 | Async HTTP client |
| `requests` | >=2.33.0,<3 | Sync HTTP (pinned for CVE-2026-25645) |

### Data & Config

| Package | Version | Purpose |
|---------|---------|---------|
| `pydantic` | >=2.12.5,<3 | Data validation and serialization |
| `pyyaml` | >=6.0.2,<7 | YAML config parsing |
| `jinja2` | >=3.1.5,<4 | Template rendering |
| `python-dotenv` | >=1.2.1,<2 | Environment variable loading |

### Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `tenacity` | >=9.1.4,<10 | Retry logic with backoff |

### Web Tools

| Package | Version | Purpose |
|---------|---------|---------|
| `firecrawl-py` | >=4.16.0,<5 | Web scraping and extraction |
| `parallel-web` | >=0.4.2,<1 | Parallel web requests |
| `fal-client` | >=0.13.1,<1 | Fal.ai API (image generation) |

### Audio

| Package | Version | Purpose |
|---------|---------|---------|
| `edge-tts` | >=7.2.7,<8 | Free Microsoft Edge TTS (no API key) |
| `faster-whisper` | >=1.0.0,<2 | Fast Whisper transcription |

### Skills Hub

| Package | Version | Purpose |
|---------|---------|---------|
| `PyJWT[crypto]` | >=2.12.0,<3 | JWT auth for GitHub App (Skills Hub bot identity) |

## Optional Dependencies

### Messaging Platforms

```python
messaging = [
    "python-telegram-bot>=22.6,<23",
    "discord.py[voice]>=2.7.1,<3",
    "aiohttp>=3.13.3,<4",
    "slack-bolt>=1.18.0,<2",
    "slack-sdk>=3.27.0,<4"
]
```

### Cloud Platforms

```python
modal = ["swe-rex[modal]>=1.4.0,<2"]
daytona = ["daytona>=0.148.0,<1"]
```

### Scheduling

```python
cron = ["croniter>=6.0.0,<7"]
```

### Voice & Audio

```python
voice = ["sounddevice>=0.4.6,<1", "numpy>=1.24.0,<3"]
tts-premium = ["elevenlabs>=1.0,<2"]
```

### Development

```python
dev = [
    "pytest>=9.0.2,<10",
    "pytest-asyncio>=1.3.0,<2",
    "pytest-xdist>=3.0,<4",
    "mcp>=1.2.0,<2"
]
```

### Skills & Plugins

```python
honcho = ["honcho-ai>=2.0.1,<3"]
mcp = ["mcp>=1.2.0,<2"]
homeassistant = ["aiohttp>=3.9.0,<4"]
sms = ["aiohttp>=3.9.0,<4"]
acp = ["agent-client-protocol>=0.8.1,<1.0"]
dingtalk = ["dingtalk-stream>=0.1.0,<1"]
```

### PTY & Terminal

```python
pty = [
    "ptyprocess>=0.7.0,<1; sys_platform != 'win32'",
    "pywinpty>=2.0.0,<3; sys_platform == 'win32'"
]
```

### RL Training

```python
rl = [
    "atroposlib @ git+https://github.com/NousResearch/atropos.git",
    "tinker @ git+https://github.com/thinking-machines-lab/tinker.git",
    "fastapi>=0.104.0,<1",
    "uvicorn[standard]>=0.24.0,<1",
    "wandb>=0.15.0,<1"
]
```

### Benchmarks

```python
yc-bench = ["yc-bench @ git+https://github.com/collinear-ai/yc-bench.git ; python_version >= '3.12'"]
```

## All Dependencies Combined

```python
all = [
    "hermes-agent[modal]",
    "hermes-agent[daytona]",
    "hermes-agent[messaging]",
    "hermes-agent[cron]",
    "hermes-agent[cli]",
    "hermes-agent[dev]",
    "hermes-agent[tts-premium]",
    "hermes-agent[slack]",
    "hermes-agent[pty]",
    "hermes-agent[honcho]",
    "hermes-agent[mcp]",
    "hermes-agent[homeassistant]",
    "hermes-agent[sms]",
    "hermes-agent[acp]",
    "hermes-agent[voice]",
    "hermes-agent[dingtalk]"
]
```

## NPM Dependencies

### Website (Docusaurus)

| Package | Version | Purpose |
|---------|---------|---------|
| `@docusaurus/core` | 3.9.2 | Docusaurus framework |
| `@docusaurus/preset-classic` | 3.9.2 | Classic preset with docs, blog, pages |
| `@docusaurus/theme-mermaid` | ^3.9.2 | Mermaid diagram support |
| `@easyops-cn/docusaurus-search-local` | ^0.55.1 | Local search |
| `@mdx-js/react` | ^3.0.0 | MDX React integration |
| `clsx` | ^2.0.0 | Class name utility |
| `prism-react-renderer` | ^2.3.0 | Syntax highlighting |
| `react` | ^19.0.0 | UI framework |
| `react-dom` | ^19.0.0 | React DOM |
| `typescript` | ~5.6.2 | TypeScript (dev) |

### Browser Tools (Root)

| Package | Version | Purpose |
|---------|---------|---------|
| `agent-browser` | ^0.13.0 | Browser automation |

## Dependency Security

### Known CVE Tracking

| Package | Issue | Constraint |
|---------|-------|------------|
| `requests` | CVE-2026-25645 | >=2.33.0,<3 |
| `PyJWT` | CVE-2026-32597 | >=2.12.0,<3 |

### Supply Chain Audit

The `supply-chain-audit.yml` workflow scans every PR for:
- `.pth` files (auto-execute on Python startup)
- `base64 + exec/eval` patterns (litellm attack)
- `subprocess` with encoded arguments
- `marshal/pickle/compile` usage
- Network POST/PUT calls

## Version Constraints Strategy

| Type | Constraint | Example | Purpose |
|------|------------|---------|---------|
| Core | `>=min,<next_major` | `>=2.21.0,<3` | Allow patches, prevent breaking |
| Pin | `~=exact` | (not used) | Would prevent all updates |
| CVE | Hard lower bound | `>=2.33.0` | Force secure version |

## Python Version Requirement

- **Minimum**: Python 3.11
- **Enforced in**: `pyproject.toml` (`requires-python = ">=3.11"`)
- **Tested in CI**: Python 3.11 on ubuntu-latest

## Project Entrypoints

```python
[project.scripts]
hermes = "hermes_cli.main:main"
hermes-agent = "run_agent:main"
hermes-acp = "acp_adapter.entry:main"
```

## Package Discovery

```toml
[tool.setuptools]
py-modules = [
    "run_agent", "model_tools", "toolsets",
    "batch_runner", "trajectory_compressor", "toolset_distributions",
    "cli", "hermes_constants", "hermes_state", "hermes_time",
    "rl_cli", "utils"
]

[tool.setuptools.packages.find]
include = [
    "agent", "tools", "tools.*",
    "hermes_cli", "gateway", "gateway.*",
    "cron", "honcho_integration", "acp_adapter"
]
```

## Nix Integration

The project uses `pyproject-nix` and `uv2nix` for Nix package management, with `flake.lock` for reproducible builds.

## External Git Dependencies

Used for cutting-edge or proprietary integrations:
- `atroposlib` — git+ from NousResearch/atropos
- `tinker` — git+ from thinking-machines-lab/tinker
- `yc-bench` — git+ from collinear-ai/yc-bench (Python 3.12+ only)
