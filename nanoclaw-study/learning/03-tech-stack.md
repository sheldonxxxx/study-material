# Tech Stack

## Runtime & Language

| Component | Technology | Version |
|-----------|------------|---------|
| Runtime | Node.js | >= 20 (declared in `engines`) |
| Node Version File | `.nvmrc` | `22` |
| Language | TypeScript | ^5.7.0 |
| Module System | ES Modules (ESM) | `"type": "module"` in package.json |
| Target | ES2022 | Configured in tsconfig.json |

**Key Configuration** (`tsconfig.json`):
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "skipLibCheck": true
  }
}
```

## Core Dependencies

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@onecli-sh/sdk` | ^0.2.0 | Agent Vault - credential proxy for secret injection |
| `better-sqlite3` | 11.10.0 | SQLite database for messages, groups, sessions, state |
| `cron-parser` | 5.5.0 | Cron expression parsing for scheduled tasks |
| `pino` | ^9.6.0 | Structured JSON logging |
| `pino-pretty` | ^13.0.0 | Human-readable log formatting |
| `yaml` | ^2.8.2 | YAML parsing for config and mounts |
| `zod` | ^4.3.6 | Schema validation for config and IPC |

### Agent Container Dependencies

The container runtime (`container/agent-runner/`) has its own dependencies:

| Package | Version | Purpose |
|---------|---------|---------|
| `@anthropic-ai/claude-agent-sdk` | ^0.2.76 | Claude Agent SDK for running agents |
| `@modelcontextprotocol/sdk` | ^1.12.1 | MCP protocol for tool communication |
| `cron-parser` | ^5.0.0 | Scheduling support in container |
| `zod` | ^4.0.0 | Validation |

**Container Base Image**: `node:22-slim` with Chromium browser installed for web automation.

## Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `eslint` | ^9.35.0 | Linting with TypeScript support |
| `typescript-eslint` | ^8.35.0 | ESLint TypeScript rules |
| `@eslint/js` | ^9.35.0 | ESLint base config |
| `eslint-plugin-no-catch-all` | ^1.1.0 | Enforces specific error handling |
| `prettier` | ^3.8.1 | Code formatting |
| `vitest` | ^4.0.18 | Unit testing framework |
| `@vitest/coverage-v8` | ^4.0.18 | V8 coverage provider |
| `tsx` | ^4.19.0 | TypeScript execution for dev |
| `husky` | ^9.1.7 | Git hooks (pre-commit) |
| `globals` | ^15.12.0 | ESLint Node globals |
| `@types/node` | ^22.10.0 | Node.js type definitions |
| `@types/better-sqlite3` | ^7.6.12 | SQLite type definitions |

## Architecture Pattern

Single Node.js process with skill-based channel registration. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) self-register at startup via skills. Messages route to Claude Agent SDK running in isolated Linux containers.

```
Channels --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
```

## Container Runtime Options

| Runtime | Platform | Notes |
|---------|----------|-------|
| Docker | macOS/Linux/Windows (WSL2) | Default, cross-platform |
| Apple Container | macOS only | Lighter weight, native |
| Docker Sandboxes | Optional | Micro VM isolation layer |

## Key Source Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations (messages, groups, sessions, state) |
| `groups/*/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/` | Skills loaded inside agent containers |

## Secrets Management

API keys and credentials are NOT stored in containers. Outbound requests route through [OneCLI's Agent Vault](https://github.com/onecli/onecli), which injects credentials at request time and enforces per-agent policies and rate limits.

## Supported Model Endpoints

NanoClaw supports any Claude API-compatible endpoint via environment variables:
- `ANTHROPIC_BASE_URL` - Custom API endpoint
- `ANTHROPIC_AUTH_TOKEN` - Authentication token

Supports local models via Ollama, Together AI, Fireworks, and other Claude-compatible providers.
