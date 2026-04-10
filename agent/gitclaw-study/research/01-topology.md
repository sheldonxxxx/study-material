# GitClaw Project Topology

## Overview

**Project:** GitClaw - A universal git-native multimodal always learning AI Agent (TinyHuman)
**Type:** Node.js/TypeScript CLI + SDK package
**Entry Points:** CLI (`./dist/index.js`), SDK (`./dist/exports.js`)

## Directory Structure

```
gitclaw/
├── .git/                          # Git repository data
├── .github/
│   └── workflows/
│       └── publish.yml            # CI/CD workflow
├── agents/                        # Agent definitions (example-skill, gmail-email)
├── examples/                      # Example scripts and configurations
│   ├── adapter.ts
│   ├── chat-history.ts
│   ├── cli.ts
│   ├── index.ts
│   ├── memory.ts
│   ├── read.ts
│   ├── sandbox-cli.ts
│   ├── sandbox-memory.ts
│   ├── sandbox-read.ts
│   ├── sandbox-write.ts
│   ├── shared.ts
│   ├── skill-learner.ts
│   ├── task-tracker.ts
│   └── write.ts
├── memory/                        # Git-committed memory storage
│   └── MEMORY.md
├── skills/                        # Composable skill modules
│   └── assistant/                # Assistant skill package
├── src/                           # Main TypeScript source
│   ├── index.ts                  # CLI ENTRY POINT - main readline interface
│   ├── exports.ts                # SDK exports (main entry for consumers)
│   ├── sdk.ts                    # SDK implementation
│   ├── sdk-types.ts             # SDK type definitions
│   ├── sdk-hooks.ts             # SDK hook integrations
│   ├── agents.ts                # Agent configuration loading
│   ├── audit.ts                 # Audit logging
│   ├── compliance.ts            # Compliance warnings
│   ├── config.ts                # Config management
│   ├── context.ts               # Context management
│   ├── hooks.ts                 # Lifecycle hooks
│   ├── loader.ts                # Agent/tool loading
│   ├── plugin-cli.ts            # Plugin CLI commands
│   ├── plugin-sdk.ts            # Plugin SDK
│   ├── plugin-types.ts          # Plugin type definitions
│   ├── plugins.ts               # Plugin management
│   ├── schedules.ts             # Scheduling
│   ├── schedule-runner.ts       # Schedule execution
│   ├── sandbox.ts               # Sandbox context
│   ├── session.ts               # Session management
│   ├── skills.ts                # Skill loading/management
│   ├── tool-loader.ts           # Declarative tool loading
│   ├── tool-utils.ts            # Tool utilities
│   ├── workflows.ts            # Workflow management
│   ├── composio/                # Composio integration
│   │   ├── local-repo.ts
│   │   └── sdk-demo.ts
│   ├── learning/                # Learning modules
│   │   └── reinforcement.ts
│   ├── tools/                   # Built-in tool adapters
│   │   ├── adapter.ts
│   │   ├── client.ts
│   │   └── index.ts
│   └── voice/                   # Voice server & UI
│       ├── adapter.ts
│       ├── chat-history.ts
│       ├── gemini-live.ts
│       ├── openai-realtime.ts
│       ├── server.ts            # Voice WebSocket server
│       ├── ui.html              # Voice UI (115KB)
│       └── workflows/
├── test/                         # Test suite
│   └── sdk.test.ts
├── agent.yaml                    # Agent specification (tools, model, runtime)
├── package.json                  # npm package definition
├── tsconfig.json                  # TypeScript configuration
├── install.sh                     # One-command installer
├── README.md                      # Documentation
├── SOUL.md                       # Personality/identity
├── RULES.md                      # Behavioral constraints
├── CONTRIBUTING.md               # Contribution guidelines
└── LICENSE                       # MIT license
```

## Key Entry Points

### CLI Entry (Primary)
- **File:** `./src/index.ts`
- **Package bin:** `gitclaw` -> `./dist/index.js`
- **Purpose:** Main readline-based CLI interface with voice server support
- **Invokes:** `Agent` from `@mariozechner/pi-agent-core`, loads tools, skills, hooks

### SDK Entry
- **File:** `./src/exports.ts`
- **Package main:** `./dist/exports.js`
- **Purpose:** Public API for consuming gitclaw as a library
- **Types:** `./dist/exports.d.ts`

### SDK Implementation
- **File:** `./src/sdk.ts` - Main SDK implementation
- **File:** `./src/sdk-types.ts` - Type definitions
- **File:** `./src/sdk-hooks.ts` - Hook integrations

## Core Modules

| Module | File | Purpose |
|--------|------|---------|
| Agent Loading | `src/loader.ts` | Loads agent configuration from agent.yaml |
| Tools | `src/tools/` | Built-in tool adapters (cli, read, write, memory, etc.) |
| Skills | `src/skills.ts` | Skill loading and management |
| Hooks | `src/hooks.ts` | Lifecycle hook system |
| Plugins | `src/plugins.ts` | Plugin management |
| Sandbox | `src/sandbox.ts` | Isolated execution context |
| Voice | `src/voice/` | WebSocket voice server with OpenAI/Gemini realtime |

## Configuration Files

| File | Purpose |
|------|---------|
| `agent.yaml` | Agent spec: model, tools, runtime config |
| `SOUL.md` | Agent personality and identity |
| `RULES.md` | Behavioral constraints |
| `package.json` | npm package with dependencies |

## Build Output

- **Dist folder:** `./dist/` (compiled from `src/`)
- **Build command:** `npm run build` (tsc + copy ui.html)
- **Dev command:** `npm run dev` (TypeScript watch mode)

## Dependencies

**Key runtime dependencies:**
- `@mariozechner/pi-agent-core` - Agent framework
- `@mariozechner/pi-ai` - AI integrations
- `baileys` - WhatsApp protocol (optional)
- `ws` - WebSocket server
- `node-cron` - Scheduling
- `yaml` / `js-yaml` - YAML parsing

**Peer dependency:**
- `gitmachine` (optional)

## TypeScript Configuration

- Target: ES2022+ (inferred from dependencies)
- Module: ESNext
- Strict mode enabled
- Node 20+ required
