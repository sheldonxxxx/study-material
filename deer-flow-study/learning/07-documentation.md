# Documentation

## Overview

DeerFlow maintains comprehensive documentation across multiple files and locations, covering user-facing guides, developer references, and architectural documentation.

## Repository Documentation

### Root Level

| File | Purpose |
|------|---------|
| `README.md` | Main project documentation with quick start, features, configuration |
| `README_zh.md` | Chinese translation |
| `README_ja.md` | Japanese translation |
| `README_fr.md` | French translation |
| `README_ru.md` | Russian translation |
| `CONTRIBUTING.md` | Development setup, workflow, testing guidelines |
| `SECURITY.md` | Security notices and deployment recommendations |
| `LICENSE` | MIT License |
| `.github/copilot-instructions.md` | Claude Code integration instructions |

### Backend Documentation (`backend/docs/`)

| File | Purpose |
|------|---------|
| `README.md` | Backend documentation index |
| `ARCHITECTURE.md` | System architecture overview |
| `API.md` | Complete API reference |
| `CONFIGURATION.md` | Configuration options and examples |
| `SETUP.md` | Backend setup instructions |
| `FILE_UPLOAD.md` | File upload functionality |
| `PATH_EXAMPLES.md` | Path types and usage examples |
| `summarization.md` | Context summarization feature |
| `plan_mode_usage.md` | Plan mode with TodoList |
| `AUTO_TITLE_GENERATION.md` | Automatic title generation |
| `TITLE_GENERATION_IMPLEMENTATION.md` | Title implementation details |
| `MCP_SERVER.md` | MCP server configuration |
| `GUARDRAILS.md` | Guardrail providers setup |
| `MEMORY_IMPROVEMENTS.md` | Memory system improvements |
| `TODO.md` | Planned features and known issues |

### Frontend Documentation (`frontend/`)

| File | Purpose |
|------|---------|
| `README.md` | Frontend overview and commands |
| `CLAUDE.md` | Claude Code guidance for frontend development |
| `AGENTS.md` | Agent system integration |

### Backend Root Documentation

| File | Purpose |
|------|---------|
| `README.md` | Backend architecture and API reference |
| `CLAUDE.md` | Claude Code guidance for backend development |
| `CONTRIBUTING.md` | Backend-specific contribution guidelines |

## Key Documentation Content

### Quick Start Guide
1. Clone repository
2. Run `make config` to generate configuration files
3. Edit `config.yaml` with model settings and API keys
4. Choose deployment: Docker (recommended) or local development
5. Access at http://localhost:2026

### Configuration Guide
- **Models**: Configure LLM providers (OpenAI, OpenRouter, Claude, DeepSeek, etc.)
- **Sandbox**: Local, Docker, or Kubernetes execution modes
- **Channels**: Telegram, Slack, Feishu integrations
- **Memory**: Long-term memory with fact extraction
- **Skills**: MCP servers and skill loading

### Architecture Documentation
- **Harness/App Split**: Strict architectural boundary between `packages/harness/deerflow/` (publishable) and `app/` (application)
- **Agent System**: Lead agent with middleware chain, subagent delegation
- **Sandbox System**: Virtual path translation, containerized execution
- **IM Channels**: Message bridge for external platforms

## Documentation Standards

Per project guidelines:

1. **Update Policy**: README.md and CLAUDE.md must be updated after every code change
2. **TDD Requirement**: Every feature or bug fix must include unit tests
3. **CI Enforcement**: Regression tests run on every PR
4. **CLAUDE.md Synchronization**: Keep development docs aligned with codebase

## External Resources

- [Official Website](https://deerflow.tech) - Live demos and more
- [GitHub Issues](https://github.com/bytedance/deer-flow/issues) - Bug reports and feature requests
- [GitHub Discussions](https://github.com/bytedance/deer-flow/discussions) - Community Q&A
- [BytePlus Docs](https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest) - InfoQuest tool documentation

## Documentation Structure

```
deer-flow/
├── README.md                    # Main entry point
├── CONTRIBUTING.md              # Development guide
├── backend/
│   ├── README.md               # Backend overview
│   ├── CLAUDE.md               # Backend dev guidance
│   └── docs/                   # Detailed backend docs
│       ├── ARCHITECTURE.md     # System architecture
│       ├── API.md              # API reference
│       ├── CONFIGURATION.md    # Configuration guide
│       └── ...                 # Feature documentation
└── frontend/
    ├── README.md               # Frontend overview
    └── CLAUDE.md               # Frontend dev guidance
```
