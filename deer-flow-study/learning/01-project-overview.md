# Project Overview: DeerFlow 2.0

## Project Purpose

DeerFlow (**D**eep **E**xploration and **E**fficient **R**esearch **Flow**) is an open-source **super agent harness** developed by ByteDance Volcengine that orchestrates sub-agents, memory, and sandboxes to accomplish complex tasks. Built as a ground-up rewrite in version 2.0, it provides a production-ready runtime giving AI agents the infrastructure to actually execute tasks — not just discuss them.

**Key differentiation**: Unlike a chatbot with tool access, DeerFlow gives agents an actual execution environment with filesystem access, sandboxed command execution, and the ability to spawn parallel sub-agents for complex multi-step tasks.

## Architecture Overview

### Service Topology

```
                                    ┌─────────────────────┐
                                    │   Nginx (port 2026) │◄── User Access
                                    │   Reverse Proxy     │
                                    └──────────┬──────────┘
                                               │
                        ┌──────────────────────┼──────────────────────┐
                        │                      │                      │
                        ▼                      ▼                      ▼
              ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
              │ Frontend (3000) │    │ LangGraph (2024)│    │  Gateway (8001) │
              │  Next.js 16     │    │  Agent Runtime  │    │   FastAPI API   │
              └─────────────────┘    └─────────────────┘    └─────────────────┘
                                                                        │
                                    ┌────────────────────────────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │ Provisioner     │
                          │ (8002, K8s only)│
                          └─────────────────┘
```

### Core Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Frontend** | Next.js 16, React 19, TypeScript | Web UI for chat interface |
| **LangGraph Server** | Python, LangGraph | Agent orchestration, workflow execution |
| **Gateway API** | FastAPI | REST API for models, skills, memory, uploads, artifacts |
| **Nginx** | Reverse proxy | Unified entry point, routing |
| **Provisioner** | Python | Kubernetes pod management for sandbox mode |

### Backend Package Structure

```
backend/
├── packages/harness/deerflow/      # Publishable agent framework
│   ├── agents/                      # Lead agent, middlewares, memory
│   ├── sandbox/                     # Local/Docker sandbox execution
│   ├── subagents/                   # Subagent delegation system
│   ├── tools/                       # Built-in tools (file ops, bash)
│   ├── mcp/                         # MCP integration
│   ├── models/                      # Model factory (OpenAI, Claude, etc.)
│   ├── skills/                      # Skills discovery and loading
│   ├── config/                      # Configuration system
│   ├── guardrails/                  # Tool call authorization
│   └── client.py                    # Embedded Python client
└── app/
    ├── gateway/                     # FastAPI Gateway API
    └── channels/                    # IM integrations (Slack, Telegram, Feishu)
```

### Frontend Source Structure

```
frontend/src/
├── app/                      # Next.js App Router
├── components/              # UI, workspace, landing components
├── core/                    # Business logic (threads, api, artifacts)
├── hooks/                   # Shared React hooks
└── lib/                     # Utilities (cn, env validation)
```

## Key Features

### 1. Skills & Tools System

Skills are structured Markdown modules defining workflows and best practices. DeerFlow ships with built-in skills for research, report generation, slide creation, web pages, and image/video generation. The system supports:
- Progressive loading (only needed skills loaded)
- Custom skill installation via `.skill` archives
- Tool extensibility via MCP servers

### 2. Sub-Agent Delegation

Complex tasks are decomposed across parallel sub-agents:
- Each sub-agent runs in isolated context
- Max 3 concurrent sub-agents (configurable)
- Built-in agent types: `general-purpose` and `bash`
- 15-minute timeout per sub-agent task

### 3. Sandbox Execution

Three execution modes with increasing isolation:
- **Local**: Direct host execution (fastest, least isolated)
- **Docker**: Containerized execution (good isolation)
- **Kubernetes**: Pod-based execution via provisioner (strongest isolation)

Virtual path system maps agent paths (`/mnt/user-data/`) to physical paths with thread isolation.

### 4. Context Engineering

- **Isolated sub-agent contexts**: Sub-agents cannot see main agent context
- **Summarization**: Aggressive context reduction for long tasks
- **Memory system**: Persistent cross-session memory with fact extraction

### 5. Long-Term Memory

Stores user profile, preferences, and accumulated knowledge in `memory.json`:
- Fact extraction with confidence scores
- Debounced async updates
- Context injection into agent prompts (top 15 facts)

## Supported Integrations

### Models
Model-agnostic via LangChain — any OpenAI-compatible API works:
- OpenAI (including Responses API)
- Anthropic Claude (including Claude Code OAuth)
- Google Gemini via OpenRouter
- OpenRouter aggregated models
- Local models via LiteLLM

### IM Channels
- **Telegram**: Bot API with long-polling
- **Slack**: Socket Mode
- **Feishu/Lark**: WebSocket

### External Services
- Tavily (web search/fetch)
- Jina AI (web fetch with readability)
- Firecrawl (web scraping)
- InfoQuest (BytePlus search)

## Technology Stack

| Layer | Technology |
|-------|------------|
| **Backend Runtime** | Python 3.12+ |
| **API Framework** | FastAPI |
| **Agent Framework** | LangGraph, LangChain |
| **Frontend Framework** | Next.js 16 |
| **UI Library** | React 19, Radix UI, Tailwind CSS 4 |
| **Type System** | TypeScript 5.8, Zod |
| **Package Management** | pnpm (frontend), uv (backend) |
| **Testing** | pytest (backend), no frontend test framework |

## Configuration

Central `config.yaml` manages all settings:
- Model configurations
- Tool definitions and groups
- Sandbox provider selection
- Skills paths
- Memory settings
- Guardrails policies

Environment variable substitution supported (`$API_KEY`).

## Development Status

- **Version**: 2.0 (ground-up rewrite from v1)
- **Release Status**: Pre-1.0, latest commit is the only supported version
- **Community**: Active development, #1 GitHub Trending Feb 28, 2026
- **License**: MIT

## Quick Start

```bash
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
make config          # Generate config.yaml from template
# Edit config.yaml with API keys
make docker-start    # Start all services
# Access at http://localhost:2026
```

## Documentation

- [Main README](https://github.com/bytedance/deer-flow#deer-flow---20)
- [Backend Architecture](../reference/deer-flow/backend/CLAUDE.md)
- [Frontend Architecture](../reference/deer-flow/frontend/CLAUDE.md)
- [Configuration Guide](../reference/deer-flow/backend/docs/CONFIGURATION.md)
- [Guardrails System](../reference/deer-flow/backend/docs/GUARDRAILS.md)
