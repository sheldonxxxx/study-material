# Tech Stack Analysis: nanoclaw

## Overview

NanoClaw is a personal Claude assistant built as a lightweight Node.js application with a skill-based channel system. It uses TypeScript throughout, Docker containers for agent isolation, and SQLite for persistence.

---

## Language & Runtime

| Aspect | Value |
|--------|-------|
| Primary Language | TypeScript |
| JavaScript | ESM modules (`"type": "module"`) |
| Node.js Engine | `>= 20` |
| Target | ES2022 |
| Module System | NodeNext |

### TypeScript Configuration

**Root project** (`tsconfig.json`):
- Module: `NodeNext` with `NodeNext` resolution
- Strict mode enabled
- Declaration maps enabled for debugging
- Source maps enabled
- `resolveJsonModule: true`

**Container agent-runner** (`container/agent-runner/tsconfig.json`):
- Same ES2022 target and NodeNext module system
- Strict mode enabled
- No sourceMap (lighter container image)

---

## Build System

| Tool | Version | Purpose |
|------|---------|---------|
| TypeScript | 5.7.x | Type compilation |
| tsx | 4.19.x | Dev runtime (hot reload) |

**Build scripts:**
- `npm run build` - Compile TypeScript to `dist/`
- `npm run dev` - Run with tsx (hot reload)
- `npm run start` - Run compiled JavaScript from `dist/`

**Container build:**
- Uses `npm run build` inside Docker
- Agent container built via `container/build.sh`

---

## Testing

| Tool | Version | Coverage |
|------|---------|----------|
| Vitest | 4.0.x | @vitest/coverage-v8 (4.0.x) |

**Test configs:**
- `vitest.config.ts` - Tests in `src/**/*.test.ts` and `setup/**/*.test.ts`
- `vitest.skills.config.ts` - Tests in `.claude/skills/**/tests/*.test.ts`

**Test commands:**
- `npm test` / `npm run test:watch` - Run tests with Vitest
- `npm run typecheck` - TypeScript type checking without emit

---

## Code Quality

### ESLint

**Version:** 9.35.x with flat config

**Plugins:**
- `@eslint/js` - Recommended JS rules
- `typescript-eslint` - TypeScript rules
- `eslint-plugin-no-catch-all` - Catch clause enforcement

**Rules applied:**
- `no-catch-all/no-catch-all: warn` - Warns against catch-all error handlers
- `@typescript-eslint/no-unused-vars: error` - Strict unused variable detection
- `@typescript-eslint/no-explicit-any: warn` - Any type warnings
- `preserve-caught-error: error` - Requires catch parameter

**Ignored paths:** `node_modules/`, `dist/`, `container/`, `groups/`

### Prettier

**Version:** 3.8.x

**Config (`.prettierrc`):**
- `singleQuote: true`

**Scripts:**
- `npm run format` / `npm run format:fix` - Format source
- `npm run format:check` - Check formatting

---

## Dependencies

### Runtime Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| @onecli/sh/sdk | 0.2.x | Secret injection / credential proxy |
| better-sqlite3 | 11.10.x | SQLite database |
| pino | 9.6.x | Structured logging |
| pino-pretty | 13.0.x | Log formatting |
| cron-parser | 5.5.x | Cron expression parsing |
| yaml | 2.8.x | YAML parsing |
| zod | 4.3.x | Schema validation |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| @types/node | 22.10.x | Node.js types |
| @types/better-sqlite3 | 7.6.x | SQLite types |
| husky | 9.1.x | Git hooks |
| eslint | 9.35.x | Linting |
| typescript-eslint | 8.35.x | ESLint TypeScript support |
| @vitest/coverage-v8 | 4.0.x | Test coverage |
| vitest | 4.0.x | Test runner |
| prettier | 3.8.x | Formatting |
| tsx | 4.19.x | TypeScript execution |

---

## Container / Docker

### Agent Container (`container/Dockerfile`)

**Base image:** `node:22-slim`

**System dependencies installed:**
- Chromium (browser automation)
- fonts-liberation, fonts-noto-cjk, fonts-noto-color-emoji
- libgbm1, libnss3, libatk-bridge2.0-0, libgtk-3-0, libx11-xcb1
- libxcomposite1, libxdamage1, libxrandr2, libasound2
- libpangocairo-1.0-0, libcups2, libdrm2, libxshmfence1
- curl, git

**Global npm packages:**
- `agent-browser`
- `@anthropic-ai/claude-code`

**Workspace layout:**
- `/workspace/group` - Per-group isolated filesystem
- `/workspace/global` - Shared global state
- `/workspace/extra` - Extra workspace
- `/workspace/ipc/messages`, `/workspace/ipc/tasks`, `/workspace/ipc/input` - IPC directories

### Agent Runner (`container/agent-runner/`)

**Own package.json** with dependencies:

| Package | Version | Purpose |
|---------|---------|---------|
| @anthropic-ai/claude-agent-sdk | 0.2.76 | Claude Agent SDK |
| @modelcontextprotocol/sdk | 1.12.x | MCP protocol |
| cron-parser | 5.0.x | Cron parsing |
| zod | 4.0.x | Validation |

---

## CI/CD

**Platform:** GitHub Actions

**Workflows:**

1. **CI** (`.github/workflows/ci.yml`)
   - Triggered on PR to main
   - Steps: `npm ci` -> format check -> typecheck -> vitest

2. **Label PR** (`.github/workflows/label-pr.yml`)
   - Auto-labeling PRs

3. **Bump Version** (`.github/workflows/bump-version.yml`)
   - Version management

4. **Update Tokens** (`.github/workflows/update-tokens.yml`)
   - Token refresh automation

---

## Monorepo Structure

**Not a monorepo.** No npm workspaces or Turborepo.

Two separate TypeScript projects:
1. **Root project** - Main NanoClaw orchestrator
2. **container/agent-runner** - Sidecar agent runner (independent package.json)

---

## Secrets & Credentials

Secrets managed via **OneCLI Agent Vault** (`@onecli/sh/sdk`):
- API keys, OAuth tokens, auth credentials injected at container request time
- No secrets passed directly to containers
- `.env` credentials migrated to OneCLI vault

---

## Summary Table

| Category | Choice |
|----------|--------|
| Language | TypeScript (strict) |
| Runtime | Node.js 20+ |
| Module | ESM |
| Build | TypeScript compiler |
| Test | Vitest |
| Lint | ESLint 9 flat config |
| Format | Prettier |
| Database | SQLite (better-sqlite3) |
| Logging | Pino |
| Validation | Zod |
| Secrets | OneCLI SDK |
| Container | node:22-slim + Chromium |
| CI | GitHub Actions |
