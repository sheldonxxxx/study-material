# Dependencies

## Overview

GitClaw has a focused dependency footprint with 9 production dependencies, 5 dev dependencies, and 1 optional peer dependency. The project emphasizes minimal external dependencies while leveraging battle-tested libraries for complex functionality.

## Production Dependencies

### Core Agent Runtime

| Package | Version | Purpose |
|---------|---------|---------|
| `@mariozechner/pi-agent-core` | ^0.55.4 | Agent orchestration engine |
| `@mariozechner/pi-ai` | ^0.55.4 | Multi-provider LLM abstraction |

### Type & Schema

| Package | Version | Purpose |
|---------|---------|---------|
| `@sinclair/typebox` | ^0.34.41 | Runtime type validation with TypeScript support |

### CLI & Input

| Package | Version | Purpose |
|---------|---------|---------|
| `@googleworkspace/cli` | ^0.8.1 | CLI argument parsing framework |

### Communication

| Package | Version | Purpose |
|---------|---------|---------|
| `ws` | ^8.19.0 | WebSocket support for real-time features |

### Data Formats

| Package | Version | Purpose |
|---------|---------|---------|
| `yaml` | ^2.8.2 | YAML parsing (config files) |
| `js-yaml` | ^4.1.0 | Alternative YAML support (tool definitions) |

### Scheduling

| Package | Version | Purpose |
|---------|---------|---------|
| `node-cron` | ^3.0.3 | Cron-based task scheduling |

### Protocol Integration

| Package | Version | Purpose |
|---------|---------|---------|
| `baileys` | ^7.0.0-rc.9 | WhatsApp protocol implementation |

## Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.7.0 | TypeScript compiler |
| `@types/node` | ^22.0.0 | Node.js type definitions |
| `@types/ws` | ^8.18.1 | WebSocket type definitions |
| `@types/js-yaml` | ^4.0.9 | js-yaml type definitions |
| `@types/node-cron` | ^3.0.11 | node-cron type definitions |

## Peer Dependencies

| Package | Constraint | Purpose | Status |
|---------|------------|---------|--------|
| `gitmachine` | >=0.1.0 | Git machine integration | Optional |

## Dependency Graph

```
gitclaw
├── pi-agent-core (^0.55.4)
│   └── (agent orchestration)
├── pi-ai (^0.55.4)
│   └── (LLM provider abstraction)
├── TypeBox (^0.34.41)
│   └── (runtime types)
├── @googleworkspace/cli (^0.8.1)
│   └── (CLI parsing)
├── ws (^8.19.0)
│   └── (WebSocket)
├── yaml (^2.8.2)
│   └── (YAML parsing)
├── js-yaml (^4.1.0)
│   └── (YAML support)
├── node-cron (^3.0.3)
│   └── (scheduling)
└── baileys (^7.0.0-rc.9)
    └── (WhatsApp)
```

## Notable Dependency Choices

### pi-ai Over Direct SDK Calls

GitClaw delegates LLM provider handling to `pi-ai` rather than using provider SDKs directly. This provides:
- Unified interface across providers
- Automatic retry and fallback logic
- Provider-specific prompt engineering

### Baileys for WhatsApp

Uses Baileys library (popular Node.js WhatsApp implementation) rather than official WhatsApp Business API for flexibility.

### Dual YAML Libraries

Two YAML libraries present (`yaml` and `js-yaml`) suggest gradual migration or different use cases within the codebase.

## Dependency Health

### Version Currency

| Category | Assessment |
|----------|-------------|
| Core (pi-*) | v0.55.4 - actively maintained |
| TypeBox | v0.34.41 - stable release |
| TypeScript | v5.7.0 - latest major |
| ws | v8.19.0 - stable |
| baileys | v7.0.0-rc.9 - release candidate |

### Maintenance Signals

- Regular npm releases (1.1.6 as of analysis)
- Active GitHub repository
- TypeScript strict mode compliance
- ESM-first development

## What Could Be Removed

| Dependency | Potential Replacement | Risk |
|------------|----------------------|------|
| `baileys` | Optional feature flag | Medium - breaking for WhatsApp users |
| `node-cron` | Native node:timers or built-in | Low - simple scheduling |
| `js-yaml` | Consolidate on `yaml` | Low - code deduplication |

## Security Considerations

- **ws** - WebSocket library, potential XSS/DoS vector
- **baileys** - WhatsApp protocol, account ban risk
- **js-yaml** - YAML parsing, potential deserialization issues

Recommend auditing `npm audit` output for known vulnerabilities.

## Bundle Impact

The `files` field in `package.json` restricts npm distribution to:

```json
{
  "files": ["dist", "README.md"]
}
```

This keeps the published package lean by excluding source, tests, and config files.

## Dependency Installation

```bash
npm install          # Production + dev dependencies
npm install --omit=dev  # Production only
npm ci               # Clean install from lockfile
```
