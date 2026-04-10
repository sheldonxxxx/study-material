# Project Overview: Hermes Agent

**Repository:** `/Users/sheldon/Documents/claw/reference/hermes-agent`
**Project:** Self-improving AI agent built by Nous Research
**Language:** Python (primary), TypeScript (website/docs)

---

## What Is Hermes Agent?

Hermes Agent is a CLI-first AI agent that combines LLM capabilities with extensive tool use and multi-platform messaging integration. The agent can execute commands, browse the web, manage files, and communicate across platforms like Discord, Telegram, and Slack.

---

## Core Architecture

### Primary Components

| Component | Purpose |
|-----------|---------|
| `agent/` | Core agent logic - Anthropic API integration, context compression, prompt building, display rendering |
| `hermes_cli/` | CLI implementation (~35 modules) - commands, configuration, tools, browser automation |
| `gateway/` | Multi-platform messaging gateway - unified bot interface across 15+ platforms |
| `skills/` | Skill definitions organized by domain (35 categories) |
| `tools/` | Tool implementations (~50 tools) - file ops, web, code execution |
| `environments/` | Agent training/testing environments for RL |
| `acp_adapter/` | Agent Client Protocol implementation |
| `tests/` | Comprehensive test suite |

### Entry Points

| Entry Point | Purpose | Size |
|-------------|---------|------|
| `cli.py` | Main CLI entry | ~329KB |
| `run_agent.py` | Agent execution | ~387KB |
| `hermes_cli/main.py` | Full CLI implementation | ~165KB |
| `gateway/run.py` | Gateway runner | ~262KB |

---

## Key Features

### 1. Multi-Platform Messaging Gateway
Unified messaging across Discord, Telegram, Slack, WhatsApp, Signal, Email, Matrix, Mattermost, DingTalk, SMS, HomeAssistant, and webhooks.

### 2. Extensive Tool System
- Browser automation (Playwright)
- File operations
- Terminal execution
- Code execution
- Skills management
- Memory management
- Vision/image processing
- Voice/TTS tools
- MCP server integration
- Web research

### 3. Skill System
35 skill categories including GitHub integration, MLOps, data science, creative tools, productivity, and autonomous AI agents.

### 4. RL-Ready Architecture
- Batch runners for trajectory processing
- Trajectory compression for training
- Multiple benchmark environments
- Support for reinforcement learning workflows

### 5. Environment Abstraction
Multiple execution backends: Local, Docker, Singularity, SSH, Daytona, Modal

---

## Directory Structure

```
hermes-agent/
├── agent/                    # Core agent logic
├── hermes_cli/               # CLI implementation (primary interface)
├── gateway/                  # Messaging gateway
│   └── platforms/            # Platform integrations
├── skills/                   # Skill definitions (35 categories)
├── tools/                    # Tool implementations (~50 tools)
├── environments/             # Agent training/testing environments
├── tests/                    # Test suite
├── acp_adapter/              # Adapter protocol
├── acp_registry/             # Skill registry
├── cron/                     # Cron scheduling
├── scripts/                  # Utility scripts
├── docs/                     # Documentation specs
├── website/                  # Public website
├── landingpage/              # Docusaurus documentation
└── .plans/                   # Planning documents
```

---

## Architecture Patterns

- **CLI-first design**: Primary interface via `hermes_cli/`, with gateway as secondary
- **Multi-platform gateway**: Unified messaging across 15+ platforms
- **Tool-based execution**: Extensive tool system in `hermes_cli/` and `tools/`
- **Skill system**: Procedural memory and skill creation in `skills/`
- **Environment abstraction**: Multiple execution backends (local, Docker, SSH, Daytona, Modal)
- **RL-ready**: Batch runners and trajectory compression for training

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Python >= 3.11 |
| CLI Framework | fire.Fire |
| Validation | Pydantic >= 2.12.5 |
| Async | asyncio |
| API Clients | Anthropic, OpenRouter, OpenAI |
| Browser | Playwright |
| Container | Docker, Singularity, Modal |
| Testing | pytest, pytest-asyncio, pytest-xdist |
