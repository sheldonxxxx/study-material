# CoPaw Project Overview

## Project Purpose

CoPaw is a **multi-agent AI orchestration platform** that enables users to deploy and manage AI agents with diverse capabilities. The platform connects to multiple LLM providers (OpenAI, Anthropic, Gemini, Ollama) and supports various messaging channels (DingTalk, Discord, Feishu, Telegram, WeCom, QQ, iMessage, Matrix, etc.) to create a flexible hub-and-spoke architecture where agents can interact across different communication platforms.

## Project Scope

### Core Capabilities

| Capability | Implementation |
|------------|----------------|
| **Agent Framework** | ReAct pattern agents with skills, tools, and memory management |
| **Multi-Provider Support** | OpenAI, Anthropic, Gemini, Ollama with automatic failover |
| **Messaging Channels** | 12+ channel integrations for cross-platform messaging |
| **Desktop GUI** | pywebview-based desktop application (React + TypeScript) |
| **Web API** | FastAPI backend for programmatic agent control |
| **CLI** | Click-based command-line interface with 20+ subcommands |
| **Task Scheduling** | APScheduler-based cron jobs for automated workflows |

### Target Users

- Developers building AI-powered workflows
- Teams needing cross-platform messaging integration with AI
- Organizations deploying internal AI assistants
- Power users managing multiple AI agents

---

## Technology Stack

### Backend

| Layer | Technology |
|-------|------------|
| Language | Python 3.10-3.13 |
| CLI Framework | Click |
| Web Framework | FastAPI + uvicorn |
| Task Scheduling | APScheduler |
| AI/ML | AgentScope, transformers, onnxruntime |

### Frontend

| Layer | Technology |
|-------|------------|
| Language | TypeScript (strict mode) |
| Framework | React 18+ |
| Build Tool | Vite |
| Desktop | pywebview |

### Integrations

| Category | Libraries |
|----------|-----------|
| Messaging | discord.py, dingtalk-stream, python-telegram-bot, lark-oapi, matrix-nio |
| AI Providers | OpenAI SDK, Anthropic SDK, google-generativeai, ollama |

---

## Architecture Overview

```
CoPaw/
|-- src/copaw/           # Core Python package
|   |-- cli/            # CLI commands (20+ subcommands)
|   |-- app/            # FastAPI web application
|   |   |-- routers/    # API endpoints (agents, skills, providers, channels)
|   |   |-- runner/     # Agent execution runtime
|   |   |-- channels/   # 12+ messaging channel integrations
|   |-- agents/         # Agent definitions, skills, tools
|   |-- providers/      # LLM provider abstractions
|   |-- security/       # Tool guard, skill scanner
|-- console/            # Desktop GUI (React + pywebview)
|-- website/            # Documentation site
|-- tests/              # pytest test suite
|-- scripts/            # Build and deployment scripts
```

---

## Entry Points

| Interface | Command | Description |
|-----------|---------|-------------|
| CLI | `copaw` | Main command with 20+ subcommands |
| Web API | `copaw app` | Starts FastAPI server (default 127.0.0.1:8088) |
| Desktop | `copaw desktop` | Launches pywebview GUI |
| Python | `python -m copaw` | Alternative module-based invocation |

---

## Key Features

### Multi-Agent Management

- Create, configure, and run multiple AI agents simultaneously
- Per-agent provider, model, and skill configuration
- Agent context propagation across sessions

### Skill System

- Built-in skills for common tasks
- Custom skill loading via directory scanning
- Security scanning before skill activation

### Tool Guard Architecture

- Path-based file protection (File Guardian)
- Rule-based command injection detection (Rule Guardian)
- Configurable deny/allow lists

### Channel Integration

- Unified message handling across platforms
- Channel-specific rendering and formatting
- Async message processing for scalability

---

## Deployment Options

| Method | Description |
|--------|-------------|
| pip | `pip install copaw` |
| Docker | docker-compose with PostgreSQL, Redis |
| Desktop | Local installation with GUI |
| Source | `pip install -e .` from repository |

---

## Project Maturity Signals

| Signal | Status |
|--------|--------|
| License | Apache 2.0 (permissive) |
| Security Policy | SECURITY.md exists |
| Multi-language Docs | zh, ja localization |
| CI/CD | GitHub Actions (3 platforms) |
| Pre-commit Hooks | 10+ automated checks |
| Code Coverage | PR-based coverage reporting |
| Contributing Guide | Multi-language |
