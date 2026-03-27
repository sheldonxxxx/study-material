# Tech Stack - Hermes Agent

## Core Technology Stack

### Primary Language
- **Python 3.11+** — Main implementation language for the agent core, CLI, tools, and gateway
- **Node.js 18+** — Optional, required only for browser tools and WhatsApp bridge
- **TypeScript** — Documentation website (Docusaurus)

### Python Package Manager
- **uv** — Fast Python package manager, used throughout the project
  - Referenced via `astral-sh/setup-uv@v5` in CI
  - `uv venv` for virtual environment creation
  - `uv pip install -e ".[all,dev]"` for dependency installation

### Web Framework (Website)
- **Docusaurus 3.9.2** — Static site generator for documentation
  - `@docusaurus/core`, `@docusaurus/preset-classic`
  - `@docusaurus/theme-mermaid` for diagrams
  - `@easyops-cn/docusaurus-search-local` for search
  - React 19.0.0, React-DOM 19.0.0
  - TypeScript 5.6.2

### Core Dependencies

| Category | Package | Version | Purpose |
|----------|---------|---------|---------|
| **LLM Clients** | `openai` | >=2.21.0,<3 | OpenAI-compatible API client |
| | `anthropic` | >=0.39.0,<1 | Anthropic API client |
| **CLI** | `fire` | >=0.7.1,<1 | CLI argument parsing |
| | `prompt_toolkit` | >=3.0.52,<4 | Interactive CLI (TUI) |
| | `rich` | >=14.3.3,<15 | Rich text formatting |
| **HTTP** | `httpx` | >=0.28.1,<1 | Async HTTP client |
| | `requests` | >=2.33.0,<3 | Sync HTTP (CVE-aware) |
| **Data** | `pydantic` | >=2.12.5,<3 | Data validation |
| | `pyyaml` | >=6.0.2,<7 | YAML config parsing |
| | `jinja2` | >=3.1.5,<4 | Template rendering |
| **Utilities** | `tenacity` | >=9.1.4,<10 | Retry logic |
| | `python-dotenv` | >=1.2.1,<2 | Environment variable loading |
| **Web Tools** | `firecrawl-py` | >=4.16.0,<5 | Web scraping |
| | `parallel-web` | >=0.4.2,<1 | Parallel web requests |
| | `fal-client` | >=0.13.1,<1 | Fal.ai API |
| **Audio** | `edge-tts` | >=7.2.7,<8 | Free TTS (no API key) |
| | `faster-whisper` | >=1.0.0,<2 | Whisper transcription |

### Optional Dependencies (Extras)

| Extra | Key Dependencies |
|-------|------------------|
| `messaging` | `python-telegram-bot`, `discord.py[voice]`, `slack-bolt`, `slack-sdk`, `aiohttp` |
| `modal` | `swe-rex[modal]` |
| `daytona` | `daytona` |
| `cron` | `croniter` |
| `honcho` | `honcho-ai` |
| `mcp` | `mcp` |
| `voice` | `sounddevice`, `numpy` |
| `rl` | `atroposlib`, `tinker`, `fastapi`, `uvicorn`, `wandb` |
| `dev` | `pytest`, `pytest-asyncio`, `pytest-xdist`, `mcp` |
| `cli` | `simple-term-menu` |
| `acp` | `agent-client-protocol` |

### Terminal Backends

| Backend | Implementation |
|---------|---------------|
| Local | Native shell execution |
| Docker | Containerized execution |
| SSH | Remote execution via SSH |
| Singularity | HPC container runtime |
| Modal | Serverless platform |
| Daytona | Cloud development platform |

### Database

- **SQLite** — Session storage via `hermes_state.py`
- **FTS5** — Full-text search on session content
- **JSON** — Session logs in `~/.hermes/sessions/`

### Build System

- **setuptools** — `pyproject.toml` with `setuptools.build_meta`
- **Nix** — `flake.nix` with pyproject-nix, uv2nix integration
  - Targets: `x86_64-linux`, `aarch64-linux`, `aarch64-darwin`

### Version Control

- **Git** — Source control
- **GitHub** — Remote hosting, CI/CD, Issues, Discussions

## Architecture Pattern

```
User Message → AIAgent._run_agent_loop()
  ├── Build system prompt (prompt_builder.py)
  ├── Build API kwargs (model, messages, tools, reasoning)
  ├── Call LLM (OpenAI-compatible API)
  ├── If tool_calls → Execute via registry → Loop back
  ├── If text → Persist session → Return response
  └── Context compression if approaching token limit
```

### Key Design Patterns

1. **Self-registering tools** — Each tool file calls `registry.register()` at import time
2. **Toolset grouping** — Tools grouped into toolsets (web, terminal, file, browser)
3. **Session persistence** — SQLite with FTS5 + JSON logs
4. **Provider abstraction** — Any OpenAI-compatible API
5. **Ephemeral injection** — System prompts not persisted to DB/logs

## Project Structure

```
hermes-agent/
├── run_agent.py              # AIAgent class — core conversation loop
├── cli.py                    # HermesCLI — interactive TUI
├── model_tools.py            # Tool orchestration
├── toolsets.py               # Tool groupings
├── hermes_state.py           # SQLite + FTS5 session storage
├── batch_runner.py           # Parallel batch processing
├── agent/                    # Agent internals
│   ├── prompt_builder.py
│   ├── context_compressor.py
│   └── ...
├── hermes_cli/               # CLI commands
│   ├── main.py, config.py, setup.py, auth.py
│   ├── models.py, banner.py, commands.py
│   └── skin_engine.py
├── tools/                    # Tool implementations
│   ├── registry.py, approval.py
│   ├── terminal_tool.py, file_operations.py
│   ├── web_tools.py, vision_tools.py
│   └── environments/
├── gateway/                  # Messaging gateway
│   ├── run.py, config.py, session.py
│   └── platforms/ (telegram, discord, slack, whatsapp...)
├── skills/                   # Bundled skills
├── optional-skills/           # Official optional skills
├── website/                   # Docusaurus documentation
├── scripts/                   # Installers, bridges
└── environments/             # RL training environments
```

## Environment & Configuration

- **Config file**: `~/.hermes/config.yaml`
- **Secrets**: `~/.hermes/.env`
- **Skills**: `~/.hermes/skills/`
- **Memory**: `~/.hermes/memories/`
- **Sessions**: `~/.hermes/state.db`, `~/.hermes/sessions/`
- **Cron**: `~/.hermes/cron/`

## Supported Platforms

- Linux (primary)
- macOS (full support)
- Windows via WSL2 (native Windows not supported)

## Standards Compliance

- **Conventional Commits** — `fix(scope):`, `feat(scope):`, etc.
- **PEP 8** — Python style (practical exceptions for line length)
- **MIT License** — Open source
