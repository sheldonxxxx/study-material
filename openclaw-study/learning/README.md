# OpenClaw — Learning Reference

## What This Is
Analysis of the OpenClaw codebase for the purpose of informing development of a similar multi-channel AI gateway project.

## Key Takeaways
- **Hub-and-spoke architecture**: A local WebSocket gateway (port 18789) acts as the central control plane, with 85+ extension plugins providing channel integrations (Discord, Slack, Telegram, WhatsApp, Matrix, etc.)
- **Plugin SDK boundary is strict**: Extensions must use `openclaw/plugin-sdk/*` imports exclusively; the boundary between core and extensions is enforced via TypeScript path aliases and a dedicated SDK surface
- **ESM-only, strict TypeScript**: No CommonJS, strict mode with no `any` types, path aliases for plugin SDK, tsdown for bundling
- **Pi agent runtime**: Core AI processing via `@mariozechner/pi-agent-core` with tool streaming, block streaming, multi-agent routing, and session isolation
- **Comprehensive CI/CD**: Preflight system dynamically generates test matrices based on changed files, testing across Linux (primary), macOS, Windows, Android, and iOS simultaneously

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, code patterns, test coverage
- **Do Not Trust**: README claims about simplicity (verify with code)

## At a Glance
- Language: TypeScript (ESM-only, strict mode)
- Architecture: Hub-and-spoke multi-channel AI gateway
- Key Libraries: Hono, Express, ws, Vitest, pnpm workspaces
- Notable Patterns: Plugin SDK pattern, ACP RPC protocol, channel abstraction
- License: MIT

## Quick Navigation

| Document | Topic |
|----------|-------|
| `01-project-overview.md` | Project structure, directory layout, entry points |
| `02-detailed-features.md` | Feature deep-dives (pending) |
| `03-architecture.md` | System architecture patterns (pending) |
| `04-code-patterns.md` | Code organization and patterns (pending) |
| `05-security-model.md` | Security approach (pending) |
