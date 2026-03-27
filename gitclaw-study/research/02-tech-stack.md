# Tech Stack Analysis: GitClaw

## Project Overview

**GitClaw** is a universal git-native multimodal AI agent framework. The agent lives inside a git repository, with identity, rules, memory, tools, and skills all version-controlled.

- **Repository:** https://github.com/open-gitagent/gitclaw
- **Version:** 1.1.6
- **License:** MIT
- **Author:** shreyaskapale (Lyzr Research Labs)

---

## Language & Runtime

| Aspect | Choice |
|--------|--------|
| **Primary Language** | TypeScript 5.7 |
| **Runtime** | Node.js 20+ (engine requirement) |
| **Module System** | ESM (`"type": "module"`) |
| **Target** | ES2022 |

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "strict": true,
    "declaration": true,
    "skipLibCheck": true
  }
}
```

---

## Package Manager & Build

| Aspect | Choice |
|--------|--------|
| **Package Manager** | npm |
| **Lock File** | package-lock.json |
| **Build Tool** | TypeScript compiler (`tsc`) |
| **Build Command** | `npm run build` → `tsc && cp src/voice/ui.html dist/voice/` |
| **Dev Command** | `tsc --watch` |
| **Test Command** | `node --test test/*.test.ts --experimental-strip-types` |

### Output Targets

- **CLI Entry:** `./dist/index.js` (bin: `gitclaw`)
- **SDK Entry:** `./dist/exports.js`
- **Types:** `./dist/exports.d.ts`
- **Files Distributed:** `dist/`, `README.md`

---

## Core Dependencies

### Agent Framework

| Package | Version | Purpose |
|---------|---------|---------|
| `@mariozechner/pi-agent-core` | ^0.55.4 | Core agent runtime |
| `@mariozechner/pi-ai` | ^0.55.4 | LLM adapter layer (multi-provider) |
| `@sinclair/typebox` | ^0.34.41 | TypeScript-first schema validation |

### Communication / Realtime

| Package | Version | Purpose |
|---------|---------|---------|
| `ws` | ^8.19.0 | WebSocket server (voice/realtime) |
| `baileys` | ^7.0.0-rc.9 | WhatsApp protocol (messaging) |
| `@googleworkspace/cli` | ^0.8.1 | Gmail/Google Workspace integration |

### Scheduling & Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `node-cron` | ^3.0.3 | Cron-style scheduling |
| `js-yaml` | ^4.1.0 | YAML parsing |
| `yaml` | ^2.8.2 | YAML serialization |

### Optional Peer Dependencies

| Package | Purpose |
|---------|---------|
| `gitmachine` | Optional git integration (>=0.1.0) |

---

## Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.7.0 | TypeScript compiler |
| `@types/node` | ^22.0.0 | Node.js type definitions |
| `@types/ws` | ^8.18.1 | WebSocket type definitions |
| `@types/js-yaml` | ^4.0.9 | js-yaml type definitions |
| `@types/node-cron` | ^3.0.11 | node-cron type definitions |

---

## Supported LLM Providers

Via `@mariozechner/pi-ai` (based on README):

- **Anthropic** (Claude models) — primary/preferred
- **OpenAI** (GPT-4o, GPT-4o-mini)
- **Google** (Gemini 2.0 Flash)
- **xAI** (Grok)
- **Groq**
- **Mistral**

Model selection is configured in `agent.yaml`:

```yaml
model:
  preferred: "anthropic:claude-sonnet-4-5-20250929"
  fallback: ["openai:gpt-4o"]
```

---

## CI/CD

### GitHub Actions Workflow (`.github/workflows/publish.yml`)

| Aspect | Configuration |
|--------|---------------|
| **Trigger** | Push to any tag matching `v*` |
| **Runner** | ubuntu-latest |
| **Node Version** | 20 |
| **Steps** | `npm ci` → `npm run build` → `npm publish --provenance --access public` |
| **Registry** | https://registry.npmjs.org |
| **Auth** | `NODE_AUTH_TOKEN` via `secrets.NPM_TOKEN` |
| **Permissions** | `contents: read`, `id-token: write` |

---

## Project Structure

```
gitclaw/
├── src/
│   ├── index.ts           # CLI entry point
│   ├── exports.ts         # SDK exports
│   ├── sdk.ts             # Main SDK implementation
│   ├── sdk-types.ts       # SDK type definitions
│   ├── config.ts          # Configuration loading
│   ├── loader.ts          # Agent/project loader
│   ├── session.ts         # Session management
│   ├── context.ts         # Context management
│   ├── agents.ts          # Agent orchestration
│   ├── plugins.ts         # Plugin system
│   ├── skills.ts          # Skills system
│   ├── hooks.ts           # Lifecycle hooks
│   ├── schedules.ts       # Schedule management
│   ├── workflows.ts       # Workflow engine
│   ├── audit.ts           # Audit logging
│   ├── compliance.ts      # Compliance checking
│   ├── knowledge.ts       # Knowledge base
│   ├── sandbox.ts         # Sandbox VM
│   ├── tool-loader.ts     # Declarative tool loader
│   ├── tool-utils.ts      # Tool utilities
│   ├── tools/             # Built-in tools (cli, read, write, memory, etc.)
│   ├── voice/             # Voice/realtime adapters (OpenAI Realtime, Gemini Live)
│   ├── composio/          # Composio integration
│   └── learning/          # Reinforcement learning
├── test/
│   └── sdk.test.ts        # SDK tests
├── examples/
│   ├── local-repo.ts
│   └── sdk-demo.ts
├── skills/                # Example skills
├── agents/                # Agent definitions
├── install.sh             # Interactive installer script
├── agent.yaml             # Agent manifest template
├── package.json
├── tsconfig.json
└── README.md
```

---

## Monorepo Tooling

**None detected.** This is a single-package TypeScript project. No Nx, Turborepo, Lerna, or pnpm workspaces are used.

---

## Install & Distribution

### Installation Methods

1. **One-command install:**
   ```bash
   bash <(curl -fsSL "https://raw.githubusercontent.com/open-gitagent/gitclaw/main/install.sh")
   ```

2. **npm global install:**
   ```bash
   npm install -g gitclaw
   ```

### Global Installation Behavior

The `install.sh` script:
- Validates Node 18+, npm, git prerequisites
- Installs/updates via `npm install -g gitclaw@latest`
- Supports Quick Setup (OpenAI voice + Claude agent) or Advanced Setup
- Auto-creates `~/assistant` project directory with `agent.yaml`, `SOUL.md`, `RULES.md`, `memory/`
- Supports Telegram bot, Composio integrations
- Launches voice UI at `http://localhost:3333`

---

## Key Technology Decisions

| Decision | Rationale |
|----------|-----------|
| TypeScript 5.7 | Modern type safety with strict mode |
| Node.js 20+ | ESM support, native test runner, WebSocket improvements |
| pi-agent-core + pi-ai | Clean separation of agent runtime from LLM providers |
| TypeBox over Zod | Lighter weight, TSOA-compatible schema validation |
| WebSockets (ws) | Voice realtime communication |
| Baileys | WhatsApp protocol support for messaging |
| node-cron | Lightweight scheduling without external services |
| Git-native config | Agent config as version-controlled files |

---

## Runtime Requirements

- **Node.js:** >=20
- **npm:** any recent version
- **git:** required for git-native features
- **API Keys (runtime):**
  - `ANTHROPIC_API_KEY` (required for agent)
  - `OPENAI_API_KEY` or `GEMINI_API_KEY` (for voice)
  - `COMPOSIO_API_KEY` (optional, for Gmail/Calendar/Slack/GitHub)
  - `TELEGRAM_BOT_TOKEN` (optional, for Telegram messaging)
  - `GITHUB_TOKEN` (optional, for repo operations)
