# pi-mono — Learning Reference

## What This Is
Analysis of the [pi-mono](https://github.com/badlogic/pi) codebase — an AI coding agent CLI built as a 7-package TypeScript monorepo. Produced to inform development of similar AI agent tooling.

## Key Takeaways

- **Layered architecture works**: Provider → Runtime → Application → Presentation → Integration separation makes each layer testable and replaceable independently
- **Event streaming as backbone**: `EventStream<T>` is the core primitive used everywhere — providers emit deltas, agent emits tool lifecycle events, TUI consumes and renders
- **Registry + lazy loading = good extensibility**: 18+ LLM providers registered lazily; same pattern for tools, extensions, skills
- **Tree-based session persistence is the right model**: JSONL append-only with leaf pointer, parent pointers, and compaction — enables undo, branching, and resume
- **Single-maintainer velocity is sustainable**: 710 commits/month with npm workspaces + Biome (no Turborepo), conventional commits, and a curated contributor pool

## Should Trust / Should Not Trust

- **Trust**: Architecture decisions, provider abstraction patterns, event streaming design, session model, contract-based error handling
- **Do Not Trust**: README simplicity claims (verify with code) — the codebase is more complex than docs suggest
- **Do Not Trust**: `pi-pods` package claims — `pi agent` command throws "Not implemented"

## At a Glance

| | |
|---|---|
| **Language** | TypeScript 5.9, Node.js >= 20 |
| **Architecture** | 5-layer: Provider → Runtime → Application → Presentation → Integration |
| **Packages** | 7 (pi-ai, pi-agent-core, pi-coding-agent, pi-tui, pi-web-ui, pi-mom, pi-pods) |
| **Key Libraries** | @anthropic-ai/sdk, openai, @google/genai, lit, TypeBox, Biome, jiti |
| **Notable Patterns** | EventStream, Registry, Hooks, Tree-based sessions, CSI 2026 terminal sync |
| **Stars / Activity** | 28,137 stars, ~23 commits/day, releases every 2-4 weeks |
| **License** | MIT |
| **Test Quality** | 7/10 — vitest for ai/agent/coding-agent/tui; mom and pods untested |
| **Code Quality** | 8/10 TypeScript, 9/10 Biome, 5/10 logging (ad-hoc console.log usage) |

## Document Map

| File | What It Covers |
|---|---|
| `01-project-overview.md` | Project purpose, scope, package inventory |
| `02-architecture.md` | 5-layer architecture, module responsibilities, communication flows |
| `03-tech-stack.md` | TypeScript, Node, npm workspaces, tooling |
| `04-features-deep-dive.md` | All 11 features with implementation details |
| `05-code-quality.md` | Testing, typing, linting, error handling, logging |
| `06-ci-cd.md` | GitHub Actions workflows, binary builds, release process |
| `07-documentation.md` | README structure, contributing guide, AI agent prompts |
| `08-security.md` | Secrets hierarchy, OAuth flows, input validation |
| `09-dependencies.md` | Full dependency graph across all packages |
| `10-community.md` | 28k stars, 7 open issues, curated contributor model |
| `11-patterns.md` | 13 design patterns with code evidence |
| `12-lessons-learned.md` | What to emulate, avoid, and surprises |
| `13-my-action-items.md` | Prioritized implementation roadmap |
