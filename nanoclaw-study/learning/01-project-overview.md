# NanoClaw Project Overview

## Project Purpose

NanoClaw is a **personal Claude assistant** that runs AI agents securely in isolated Linux containers. It provides multi-channel messaging (WhatsApp, Telegram, Discord, Slack, Gmail) with persistent memory per conversation, scheduled tasks, and true OS-level isolation for agent execution.

**Core philosophy:** The codebase is intentionally small (~20-30 source files) so users can fully understand and customize it. Unlike monolithic frameworks, NanoClaw is designed to be bespoke - each user forks it and modifies the code directly to match their exact needs.

## Project Scope

### What NanoClaw Does

- **Multi-channel messaging**: Connect to WhatsApp, Telegram, Discord, Slack, Gmail, and more via self-contained skills
- **Container-isolated agents**: Claude Code runs inside Linux containers (Apple Container on macOS, Docker on Linux) with filesystem isolation
- **Per-group memory**: Each conversation group has isolated CLAUDE.md memory, filesystem, and container
- **Scheduled tasks**: Recurring or one-time tasks run as full agents with cron/interval/once schedules
- **Credential security**: API keys never enter containers - routed through OneCLI Agent Vault for secret injection

### Architecture

```
Channels --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
```

Single Node.js process with skill-based channel system. Channels self-register at startup via a factory registry pattern.

### Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| No config files | Customization = code changes; code is small enough to modify safely |
| Skills over features | New capabilities submitted as Claude Code skills, not core PRs |
| SQLite over JSON | Enables proper queries, migrations, concurrent access |
| Container isolation | True OS-level sandboxing vs application-level allowlists |
| OneCLI Agent Vault | Secrets never touch the container; injected at request time |

### Technology Stack

| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20+ (ESM modules) |
| Database | SQLite via better-sqlite3 |
| Agent | Claude Agent SDK (@onecli-sh/sdk) |
| Container | Docker (Linux/macOS), Apple Container (macOS native) |
| Logging | Pino with pino-pretty |
| Validation | Zod for schema validation |
| Testing | Vitest |
| Linting | ESLint + TypeScript-ESLint |
| Formatting | Prettier |

### Project Structure

```
nanoclaw/
├── src/                    # Core source (orchestrator, channels, IPC, DB)
│   ├── index.ts           # Main orchestrator
│   ├── channels/          # Channel registry and interface
│   ├── db.ts             # SQLite schema and queries
│   ├── ipc.ts            # IPC watcher for container communication
│   ├── container-runner.ts
│   ├── task-scheduler.ts
│   └── mount-security.ts  # Mount allowlist validation
├── container/             # Agent container Dockerfile and code
├── groups/               # Per-group memory and filesystem
├── .claude/skills/       # Claude Code skills (add-*, setup, debug)
├── store/                # SQLite DB, auth state
└── data/                 # Sessions, IPC, env
```

### Security Model

NanoClaw's security is based on **isolation, not trust**:

1. **Container isolation**: Agents run in Linux VMs with only explicitly mounted directories accessible
2. **Credential proxying**: OneCLI Agent Vault injects secrets at request time - keys never enter containers
3. **Mount allowlist**: Additional directory mounts validated against user-configured allowlist at `~/.config/nanoclaw/mount-allowlist.json`
4. **Blocked patterns**: Sensitive paths (.ssh, .aws, .gpg, credentials) automatically blocked
5. **Group authorization**: IPC commands enforce main/non-main group permissions
6. **Sender allowlist**: Optional per-chat sender restrictions

### Development Model

- **AI-native**: Setup, debugging, and customization handled by asking Claude Code directly
- **Skills taxonomy**: Feature skills (add capabilities), utility skills (code + docs), operational skills (workflows), container skills (loaded at runtime)
- **Small surface area**: Core accepts only security fixes, bug fixes, and clear improvements
