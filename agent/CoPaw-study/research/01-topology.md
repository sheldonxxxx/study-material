# CoPaw Project Topology

## Root-Level File Inventory

| File | Purpose |
|------|---------|
| `README.md` | Main documentation (multiple languages: zh, ja) |
| `README_zh.md`, `README_ja.md` | Localized documentation |
| `pyproject.toml` | Python project configuration, dependencies, entry points |
| `setup.py` | Legacy setup file |
| `docker-compose.yml` | Docker orchestration |
| `.flake8` | Python linting config |
| `.pre-commit-config.yaml` | Pre-commit hooks |
| `.python-version` | Python version specification |
| `CONTRIBUTING.md`, `CONTRIBUTING_zh.md` | Contribution guidelines |
| `LICENSE` | Apache 2.0 license |
| `SECURITY.md` | Security policy |
| `.gitignore`, `.gitattributes`, `.dockerignore` | VCS/deployment ignore files |

---

## Directory Tree Overview

```
CoPaw/
|-- .github/                  # GitHub workflows and issue templates
|   |-- workflows/
|   |-- ISSUE_TEMPLATE/
|
|-- .git/                     # Git repository
|
|-- src/copaw/                # Core Python package
|   |-- __init__.py
|   |-- __main__.py           # Entry: python -m copaw
|   |-- __version__.py
|   |-- cli/                  # CLI commands (Click-based)
|   |   |-- main.py           # CLI entry point (copaw command)
|   |   |-- app_cmd.py
|   |   |-- agents_cmd.py
|   |   |-- channels_cmd.py
|   |   |-- chats_cmd.py
|   |   |-- cron_cmd.py
|   |   |-- daemon_cmd.py
|   |   |-- desktop_cmd.py
|   |   |-- env_cmd.py
|   |   |-- init_cmd.py
|   |   |-- providers_cmd.py
|   |   |-- skills_cmd.py
|   |   |-- update_cmd.py
|   |   |-- shutdown_cmd.py
|   |   |-- uninstall_cmd.py
|   |   |-- auth_cmd.py
|   |   |-- clean_cmd.py
|   |   |-- process_utils.py
|   |   |-- utils.py
|   |
|   |-- app/                  # Web API application (FastAPI/uvicorn)
|   |   |-- auth.py
|   |   |-- migration.py
|   |   |-- multi_agent_manager.py
|   |   |-- agent_config_watcher.py
|   |   |-- agent_context.py
|   |   |-- console_push_store.py
|   |   |-- download_task_store.py
|   |   |-- utils.py
|   |   |-- routers/          # API route handlers
|   |   |   |-- agent.py
|   |   |   |-- agents.py
|   |   |   |-- agent_scoped.py
|   |   |   |-- auth.py
|   |   |   |-- config.py
|   |   |   |-- console.py
|   |   |   |-- envs.py
|   |   |   |-- files.py
|   |   |   |-- local_models.py
|   |   |   |-- messages.py
|   |   |   |-- mcp.py
|   |   |   |-- ollama_models.py
|   |   |   |-- providers.py
|   |   |   |-- schemas_config.py
|   |   |   |-- skills.py
|   |   |   |-- skills_stream.py
|   |   |   |-- token_usage.py
|   |   |   |-- tools.py
|   |   |   |-- voice.py
|   |   |   |-- workspace.py
|   |   |-- channels/         # Messaging channel integrations
|   |   |   |-- base.py
|   |   |   |-- manager.py
|   |   |   |-- renderer.py
|   |   |   |-- registry.py
|   |   |   |-- schema.py
|   |   |   |-- utils.py
|   |   |   |-- console/
|   |   |   |-- dingtalk/
|   |   |   |-- discord_/
|   |   |   |-- feishu/
|   |   |   |-- imessage/
|   |   |   |-- matrix/
|   |   |   |-- mattermost/
|   |   |   |-- mqtt/
|   |   |   |-- qq/
|   |   |   |-- telegram/
|   |   |   |-- voice/
|   |   |   |-- wecom/
|   |   |   |-- xiaoyi/
|   |   |-- runner/           # Agent execution runtime
|   |   |   |-- runner.py
|   |   |   |-- api.py
|   |   |   |-- models.py
|   |   |   |-- session.py
|   |   |   |-- utils.py
|   |   |-- workspace/
|   |   |-- crons/
|   |   |-- approvals/
|   |   |-- mcp/
|   |
|   |-- agents/               # Agent definitions and skills
|   |   |-- __init__.py
|   |   |-- react_agent.py    # ReAct agent implementation
|   |   |-- model_factory.py
|   |   |-- prompt.py
|   |   |-- command_handler.py
|   |   |-- skills_manager.py
|   |   |-- skills_hub.py
|   |   |-- tool_guard_mixin.py
|   |   |-- routing_chat_model.py
|   |   |-- schema.py
|   |   |-- skills/           # Built-in skills
|   |   |-- tools/            # Agent tools
|   |   |-- md_files/         # Markdown documentation for agents
|   |   |-- memory/
|   |   |-- hooks/
|   |   |-- utils/
|   |
|   |-- providers/            # LLM provider integrations
|   |   |-- provider.py
|   |   |-- provider_manager.py
|   |   |-- openai_provider.py
|   |   |-- anthropic_provider.py
|   |   |-- gemini_provider.py
|   |   |-- ollama_provider.py
|   |   |-- ollama_manager.py
|   |   |-- openai_chat_model_compat.py
|   |   |-- retry_chat_model.py
|   |   |-- capability_baseline.py
|   |   |-- multimodal_prober.py
|   |   |-- models.py
|   |
|   |-- config/               # Configuration management
|   |-- security/             # Security (tool guard, skill scanning)
|   |-- tokenizer/            # Tokenization utilities
|   |-- tunnel/               # Tunneling utilities
|   |-- local_models/         # Local model support
|   |-- envs/                 # Environment management
|   |-- token_usage/          # Token usage tracking
|   |-- utils/                # General utilities
|   |
|-- console/                 # Desktop GUI (webview-based)
|   |-- src/
|   |   |-- App.tsx
|   |   |-- main.tsx
|   |   |-- api/
|   |   |-- components/
|   |   |-- layouts/
|   |   |-- pages/
|   |   |-- stores/
|   |   |-- i18n.ts
|   |-- public/
|   |-- package.json
|   |-- tsconfig.json
|   |-- vite.config.ts
|   |-- index.html
|   |
|-- website/                 # Documentation/marketing website
|   |-- src/
|   |   |-- App.tsx
|   |   |-- main.tsx
|   |   |-- components/
|   |   |-- pages/
|   |   |-- layouts/
|   |   |-- i18n.ts
|   |-- public/
|   |-- package.json
|   |-- tsconfig.json
|   |-- vite.config.ts
|   |-- index.html
|   |
|-- tests/                   # Test suite
|   |-- unit/
|   |-- integrated/
|   |
|-- scripts/                 # Build and deployment scripts
|   |-- install.sh
|   |-- install.ps1
|   |-- install.bat
|   |-- docker_build.sh
|   |-- docker_sync_latest.sh
|   |-- wheel_build.sh
|   |-- wheel_build.ps1
|   |-- website_build.sh
|   |-- run_tests.py
|   |-- pack/
|   |
|-- deploy/                  # Deployment configuration
|   |-- Dockerfile
|   |-- docker-compose.yml
|   |-- entrypoint.sh
|   |-- config/
```

---

## Entry Points Identified

### CLI Entry Point
- **File:** `src/copaw/cli/main.py`
- **Command:** `copaw` (installed via pyproject.toml `[project.scripts]`)
- **Also callable via:** `python -m copaw` (via `src/copaw/__main__.py`)
- **Subcommands (lazy-loaded):** app, channels, channel, daemon, chats, chat, clean, cron, env, init, models, skills, uninstall, desktop, update, shutdown, auth, agents, agent

### Web API Entry Point
- **File:** `src/copaw/app/` (FastAPI/uvicorn application)
- **Started via:** `copaw app` CLI command
- **Default host/port:** `127.0.0.1:8088`

### Desktop GUI Entry Point
- **File:** `console/` (React + pywebview)
- **Built with:** React, TypeScript, Vite
- **Started via:** `copaw desktop` CLI command

### Website Entry Point
- **File:** `website/` (React documentation site)
- **Built with:** React, TypeScript, Vite
- **Started via:** `scripts/website_build.sh`

---

## Directory Purposes

| Directory | Purpose |
|-----------|---------|
| `src/copaw/` | Core Python package containing all business logic |
| `src/copaw/cli/` | Click-based CLI command implementations |
| `src/copaw/app/` | FastAPI web application with routers for agents, skills, providers, channels, etc. |
| `src/copaw/agents/` | Agent implementations (ReAct pattern), skills management, tool definitions |
| `src/copaw/providers/` | LLM provider abstractions (OpenAI, Anthropic, Gemini, Ollama) |
| `src/copaw/app/channels/` | Messaging channel integrations (DingTalk, Discord, Feishu, Telegram, WeCom, QQ, iMessage, Matrix, etc.) |
| `src/copaw/app/runner/` | Agent execution runtime and session management |
| `src/copaw/config/` | Configuration file handling |
| `src/copaw/security/` | Tool guard and skill scanner for safety |
| `console/` | Desktop GUI application (Electron-like via pywebview) |
| `website/` | Documentation/marketing website |
| `tests/` | Unit and integration tests |
| `scripts/` | Installation, build, and deployment scripts |
| `deploy/` | Docker and deployment configuration |
| `.github/` | GitHub Actions CI/CD workflows |

---

## Technology Stack Summary

- **Language:** Python 3.10-3.13
- **CLI Framework:** Click
- **Web Framework:** FastAPI + uvicorn
- **Desktop:** pywebview (webview-based desktop app)
- **Frontend (Console/Website):** React + TypeScript + Vite
- **AI/ML:** AgentScope, transformers, onnxruntime
- **Messaging Channels:** discord.py, dingtalk-stream, python-telegram-bot, lark-oapi, matrix-nio, etc.
- **Task Scheduling:** APScheduler
- **Testing:** pytest, pytest-asyncio
