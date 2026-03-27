# Dependencies

## Dependency Philosophy

NanoClaw maintains a deliberately minimal dependency footprint. The project philosophy states: "Source code changes should only be things 90%+ of users need." Additional capabilities are delivered via skills, not core dependencies.

## Host Application Dependencies

### Production Dependencies (7 packages)

| Package | Version | Size Impact | Purpose |
|---------|---------|-------------|---------|
| `@onecli-sh/sdk` | ^0.2.0 | External | Agent Vault - credential proxy injection |
| `better-sqlite3` | 11.10.0 | Native | Embedded SQLite database |
| `cron-parser` | 5.5.0 | ~50KB | Cron expression parsing |
| `pino` | ^9.6.0 | ~100KB | Structured JSON logging |
| `pino-pretty` | ^13.0.0 | ~50KB | Human-readable log formatting |
| `yaml` | ^2.8.2 | ~200KB | YAML configuration parsing |
| `zod` | ^4.3.6 | ~300KB | Schema validation |

**Total production dependency footprint**: ~700KB+ (excluding native module)

### Development Dependencies (14 packages)

| Package | Version | Purpose |
|---------|---------|---------|
| `eslint` | ^9.35.0 | Linting |
| `typescript-eslint` | ^8.35.0 | TypeScript ESLint support |
| `@eslint/js` | ^9.35.0 | ESLint base config |
| `eslint-plugin-no-catch-all` | ^1.1.0 | Error handling enforcement |
| `prettier` | ^3.8.1 | Code formatting |
| `vitest` | ^4.0.18 | Unit testing |
| `@vitest/coverage-v8` | ^4.0.18 | Coverage reports |
| `tsx` | ^4.19.0 | TypeScript execution |
| `husky` | ^9.1.7 | Git hooks |
| `globals` | ^15.12.0 | ESLint Node globals |
| `@types/node` | ^22.10.0 | TypeScript types |
| `@types/better-sqlite3` | ^7.6.12 | SQLite type definitions |
| `typescript` | ^5.7.0 | TypeScript compiler |

## Agent Container Dependencies

### Production Dependencies (4 packages)

| Package | Version | Purpose |
|---------|---------|---------|
| `@anthropic-ai/claude-agent-sdk` | ^0.2.76 | Claude Agent SDK |
| `@modelcontextprotocol/sdk` | ^1.12.1 | MCP protocol |
| `cron-parser` | ^5.0.0 | Scheduling |
| `zod` | ^4.0.0 | Validation |

### Development Dependencies (2 packages)

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.7.3 | TypeScript compiler |
| `@types/node` | ^22.10.7 | TypeScript types |

### Container Base Image

- **Base**: `node:22-slim` (Debian-based)
- **Additional packages**: Chromium browser, fonts (Liberation, Noto CJK, Noto Color Emoji), X11 libraries, GTK3, sound libraries

## Dependency Analysis

### why-lib Report

```
@onecli-sh/sdk          Agent Vault - secret injection proxy
better-sqlite3           Embedded DB - messages, groups, sessions, state
cron-parser              Scheduled task parsing
pino / pino-pretty       Structured logging
yaml                     Mount config and group config parsing
zod                      Config validation and IPC schema validation
```

### No Major Framework

NanoClaw does NOT use:
- Express or other HTTP frameworks (single Node process)
- ORMs (raw SQLite via better-sqlite3)
- Message queue systems (file-based IPC)
- Docker SDK (process spawning via `child_process`)

## Engine Requirements

```json
"engines": {
  "node": ">=20"
}
```

Node 20+ required for ES Modules, native fetch, and modern TypeScript features.

## Lock Files

- `package-lock.json` (134KB) - Host application locks
- `container/agent-runner/package-lock.json` - Container locks

## Skills Dependency Model

Skills can add their own dependencies. Feature skills typically modify `package.json` and create their own branches. Utility skills ship code files alongside SKILL.md and may have their own `node_modules`.

### Example Skill Dependencies

Skills that add channels (WhatsApp, Telegram, Slack, Discord) bring their own SDK dependencies but these are not in the core `package.json`.

## Sensitive Dependencies

`@onecli-sh/sdk` handles all credential management. API keys never enter the container directly - they're injected by the OneCLI proxy at request time. This keeps credentials out of:
- Container filesystem
- Environment variables passed to containers
- Git history (if credentials were accidentally committed)

## Build Output

```
npm run build  -->  dist/  (JavaScript + source maps + declarations)
```

TypeScript compilation target: ES2022, NodeNext modules.
