# Tech Stack

## Overview

GitClaw is a universal git-native AI agent framework built with TypeScript and ESM modules, designed to run on Node.js 20+.

## Core Technologies

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| Runtime | Node.js | >=20 | Server-side JavaScript execution |
| Language | TypeScript | ^5.7.0 | Type-safe development |
| Module System | ESM | `"type": "module"` | Modern JavaScript modules |
| LLM Abstraction | pi-ai | ^0.55.4 | Multi-provider LLM support |
| Agent Core | pi-agent-core | ^0.55.4 | Agent runtime orchestration |
| Type Validation | TypeBox | ^0.34.41 | Runtime type checking |
| CLI Framework | Google Workspace CLI | ^0.8.1 | CLI argument parsing |
| YAML Processing | yaml, js-yaml | ^2.8.2, ^4.1.0 | Config and tool definitions |
| WebSocket | ws | ^8.19.0 | Real-time communication |
| Scheduling | node-cron | ^3.0.3 | Cron-based task scheduling |
| WhatsApp Integration | Baileys | ^7.0.0-rc.9 | WhatsApp protocol support |
| Testing | Node.js built-in test runner | --experimental-strip-types | Test execution |

## Language & Runtime

- **TypeScript 5.7**: Strict mode enabled, full type coverage
- **ESM Modules**: Native ES module support (`"type": "module"`)
- **Node.js 20+**: Minimum engine requirement, uses native test runner

## LLM Integration

### Multi-Provider Support (via pi-ai)

GitClaw leverages [pi-ai](https://github.com/nicepkg/pi-ai) for provider abstraction:

| Provider | Prefix Example | Status |
|----------|----------------|--------|
| Anthropic | `anthropic:claude-sonnet-4-5-20250929` | Preferred |
| OpenAI | `openai:gpt-4o` | Supported |
| Google | `google:gemini-2.0-flash` | Supported |
| xAI | `xai:*` | Supported |
| Groq | `groq:*` | Supported |
| Mistral | `mistral:*` | Supported |

### Model Configuration

Agents specify models in `agent.yaml`:

```yaml
model:
  preferred: "anthropic:claude-sonnet-4-5-20250929"
  fallback:
    - "openai:gpt-4o"
    - "google:gemini-2.0-flash"
  constraints:
    temperature: 0.7
    max_tokens: 4096
```

## Agent Architecture

### Key Modules

| Module | Purpose |
|--------|---------|
| `src/index.ts` | CLI entry point |
| `src/sdk.ts` | Core SDK (`query` function) |
| `src/exports.ts` | Public API surface |
| `src/loader.ts` | Agent configuration loader |
| `src/tools/` | Built-in tools (cli, read, write, memory) |
| `src/voice/` | Voice mode (OpenAI Realtime adapter) |
| `src/hooks.ts` | Script-based lifecycle hooks |
| `src/sdk-hooks.ts` | Programmatic SDK hooks |
| `src/skills.ts` | Skill expansion system |
| `src/workflows.ts` | Workflow metadata |
| `src/agents.ts` | Sub-agent metadata |
| `src/compliance.ts` | Compliance validation |
| `src/audit.ts` | Audit logging |

### Agent File Structure

```
my-agent/
├── agent.yaml          # Model, tools, runtime config
├── SOUL.md             # Agent identity & personality
├── RULES.md            # Behavioral rules & constraints
├── memory/
│   └── MEMORY.md       # Git-committed agent memory
├── tools/
│   └── *.yaml          # Declarative tool definitions
├── skills/
│   └── <name>/         # Composable skill modules
├── hooks/
│   └── hooks.yaml      # Lifecycle hook scripts
└── plugins/
    └── <name>/         # Local plugins
```

## Build & Development

### Build Pipeline

```bash
npm run build  # tsc && cp src/voice/ui.html dist/voice/
npm run dev    # tsc --watch (development)
npm run start  # node dist/index.js
npm run test   # node --test test/*.test.ts --experimental-strip-types
```

### Output Artifacts

- `dist/exports.js` - Main export entry
- `dist/exports.d.ts` - Type definitions
- `dist/index.js` - CLI entry point
- `dist/voice/ui.html` - Voice UI

### Package Exports

```json
{
  "exports": {
    ".": {
      "import": "./dist/exports.js",
      "types": "./dist/exports.d.ts"
    },
    "./cli": {
      "import": "./dist/index.js"
    }
  }
}
```

## Peer Dependencies

- `gitmachine` (>=0.1.0) - Optional peer dependency for gitmachine integration

## Key Architectural Decisions

1. **Git-Native Design**: Agent identity, memory, and configuration are git-tracked files
2. **In-Process SDK**: No subprocess/IPC - runs directly in-process via `query()`
3. **Declarative + Programmatic**: Supports both YAML-defined tools and TypeScript plugin APIs
4. **Streaming-first**: AsyncGenerator-based message streaming for real-time responses
5. **Provider Agnostic**: Via pi-ai abstraction, supports multiple LLM providers

## Dependencies Summary

| Category | Count | Notable |
|----------|-------|---------|
| Production | 9 | pi-ai, pi-agent-core, TypeBox, Baileys, ws |
| Development | 5 | TypeScript, type definitions |
| Peer | 1 | gitmachine (optional) |
