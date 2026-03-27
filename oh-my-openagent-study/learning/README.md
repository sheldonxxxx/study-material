# oh-my-openagent — Learning Reference

## What This Is
Analysis of the oh-my-openagent codebase (43.7k stars) — an AI agent harness that orchestrates multi-model agents via a hook-based plugin architecture, hash-anchored edits, background task management, and Claude Code compatibility.

## Key Takeaways
- **Hook-based plugin architecture** (76+ hooks) is the core abstraction — compose behavior by registering hooks on OpenCode events
- **Hashline edit tool** is the single highest-impact feature (6.7% → 68.3% success rate on stale-line edits)
- **Preemptive context compaction** at 78% threshold prevents reactive truncation — a reliability differentiator
- **Simple Bun workspaces** suffice for monorepo — no Turborepo/Nx/Lerna needed for this scale
- **No automated linting** (ESLint/Prettier absent) is a notable quality gap
- **Claude Code compatibility layer** absorbs the ecosystem rather than competing — key go-to-market lesson

## Should Trust / Should Not Trust
- **Trust**: Architecture patterns, hook composition model, background task safety (circuit breakers, depth limits), Zod schema validation approach, test isolation strategy
- **Do Not Trust**: README simplicity (code is significantly more complex than docs suggest); README links to factory.ai/news/terminal-bench which appears to be an external reference

## At a Glance
- **Language**: TypeScript 5.7.3 (strict mode), Bun runtime
- **Architecture**: Hook-based event-driven plugin system
- **Key Libraries**: OpenCode SDK, MCP SDK, Zod v4, ast-grep, Commander
- **Notable Patterns**: Factory, Observer, State Machine, Strategy, Registry, Proxy/Middleware
- **Stars / Activity**: 43,738 stars, 64 commits today, 60+ releases in v3.x
- **License**: MIT (CLA.md present)
