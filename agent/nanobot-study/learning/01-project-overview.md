# Project Overview

## Repository Health

| Metric | Value |
|--------|-------|
| **Stars** | 36,324 |
| **Forks** | 6,220 |
| **Open Issues** | 817 |
| **Unique Contributors** | 200+ |
| **Repository URL** | https://github.com/HKUDS/nanobot |
| **Created** | 2026-02-01 |
| **Last Push** | 2026-03-26 |

## Maintainership

| Maintainer | Role | Branch |
|------------|------|--------|
| @re-bin | Project Lead | `main` branch |
| @chengyongru | Experimental Lead | `nightly` branch |

**Branching Strategy:** Two-branch model with weekly cherry-pick workflow from `nightly` to `main`.

## Top Contributors (by commits)

1. Re-bin: 677
2. Xubin Ren: 197
3. chengyongru: 66
4. Alexander Minges: 47
5. chaohuang-ai: 21

## Component Structure

```
nanobot/
├── agent/          # Core AI agent (loop, context, memory, skills, tools)
├── bus/           # Async message bus for inter-component communication
├── channels/      # 12 chat platform adapters (Telegram, Discord, Slack, etc.)
├── cli/           # Typer-based CLI commands
├── config/        # Pydantic-based configuration schema
├── cron/          # Cron scheduling service
├── heartbeat/     # Periodic health-check/wake-up service
├── providers/     # LLM provider abstraction (20+ providers)
├── security/      # Network security utilities (SSRF protection)
├── session/       # Conversation session management (JSONL storage)
├── skills/        # Agent skill templates (GitHub, Weather, Tmux, etc.)
├── utils/         # Helper functions
└── bridge/        # TypeScript WhatsApp bridge (Node.js 20)

tests/             # 50 test files across 8 directories
docs/              # Documentation files
.github/workflows/ # CI/CD with GitHub Actions
```

## Core Processing Flow

```
Chat Platform → Channel Adapter → MessageBus → AgentLoop
                                                 │
                                    ┌────────────┼────────────┐
                                    │            │            │
                              ContextBuilder  ToolRegistry  SessionMgr
                                    │            │            │
                              LLMProvider ←─────────────→ Tools
                                    │
                              OutboundMessage → MessageBus → ChannelAdapter → Chat Platform
```

## Community Channels

- **Discord**: https://discord.gg/MnCvHqpUGB
- **Email**: xubinrencs@gmail.com
- **WeChat/Feishu**: QR codes in COMMUNICATION.md

## Release Cadence

Weekly post-releases with semantic versioning. Recent: v0.1.4.post5 (2026-03-16).

## Development Standards

- **Python**: 3.11+
- **Linter**: ruff (rules E, F, I, N, W; E501 ignored)
- **Line length**: 100 characters
- **Async**: asyncio throughout
- **Testing**: pytest with `asyncio_mode = "auto"`
- **Commit format**: `type(scope): description` (e.g., `fix(telegram): fix streaming`)
