# OpenCode — Learning Reference

## What This Is
Analysis of the [OpenCode](https://github.com/anomalyco/anomalyco) codebase for the purpose of informing development of a similar AI coding agent project.

## Key Takeaways
- **Effect framework is the architectural backbone** — dependency injection, service layers, and structured concurrency unified under one functional paradigm
- **Multi-provider abstraction via Vercel AI SDK** — 20+ providers handled with custom loaders for quirks, model discovery from models.dev with local fallback
- **Tool system with Zod validation** — 47+ tools defined via `Tool.define()` factory, parameterized with typed Zod schemas
- **Client-server is a first-class concern** — embedded mode (same process) and server mode (network) share all logic via HTTP
- **Honest about tradeoffs** — permissions are UX not security, beta deps are acceptable, iterate on real usage over over-engineering

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions (Effect-based services, event bus), code patterns (Tool.define factory), test coverage in core packages
- **Do Not Trust**: README claims about simplicity — verify with code; issue/PR templates are strict but community is welcoming
- **Caution**: Some `@ts-ignore` TODOs in provider code, JSON storage remnants, inconsistent TypeScript strictness across packages

## At a Glance
- **Language:** TypeScript (primary), Rust (desktop backend), Go (infrastructure)
- **Architecture:** Layered service-oriented with Effect DI + event bus + HTTP client-server
- **Key Libraries:** Effect, Vercel AI SDK, Hono, SolidJS, Drizzle ORM, Tailwind CSS, Tauri v2
- **Notable Patterns:** ServiceMap DI, Bus pub/sub, Tool.define factory, Provider adapter, Repository pattern, Strategy (agent types)
- **Stars / Activity:** 10M+ downloads (Jan 2026), releases every 1-3 days, `dev` as default branch
- **License:** Proprietary (Anomalyco)
- **Monorepo:** 19+ packages via Turborepo + Bun workspaces

## Document Map
| File | Content |
|------|---------|
| `01-project-overview.md` | High-level purpose, scope, and structure |
| `02-architecture.md` | Module responsibilities, communication patterns |
| `03-tech-stack.md` | Runtime, frameworks, key libraries |
| `04-features-deep-dive.md` | 18 features analyzed with implementation details |
| `05-code-quality.md` | Testing, TypeScript, linting, error handling |
| `06-ci-cd.md` | GitHub Actions, Docker, multi-platform builds |
| `07-documentation.md` | README system, AGENTS.md, multi-language support |
| `08-security.md` | Secrets, auth, permissions, CSP/CORS |
| `09-dependencies.md` | Key deps, catalog system, workspace management |
| `10-community.md` | Governance, release cadence, contribution policies |
| `11-patterns.md` | Design patterns with file references |
| `12-lessons-learned.md` | What to emulate, avoid, and surprises |
| `13-my-action-items.md` | Prioritized next steps |
