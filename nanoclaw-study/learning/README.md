# nanoclaw — Learning Reference

## What This Is
Analysis of the nanoclaw codebase — a personal Claude assistant daemon that runs in isolated Docker/Apple Container sandboxes, connects to multiple messaging channels (Telegram, WhatsApp, Discord, Slack, Gmail), and provides persistent memory per conversation group.

## Key Takeaways
- **"Never trust the container"** — API keys never enter containers; OneCLI Agent Vault intercepts HTTPS and injects credentials at request time
- **Filesystem IPC over network** — Container↔Host communication uses atomic file renames (`.tmp` + `renameSync`), not pipes or WebSockets
- **Self-registering channels** — Side-effect imports in `channels/index.ts` auto-register channels; new channels are skills that modify this file
- **Context accumulation** — Non-trigger messages accumulate in SQLite and are included as context when trigger word arrives
- **Skill-based architecture** — Core is intentionally minimal (~6000 lines); features live in `.claude/skills/` and are distributed via git branches
- **XML context, not JSON** — Messages formatted as XML context for Claude (with proper XSS escaping), not JSON arrays

## Should Trust / Should Not Trust
- **Trust**: Security model (credential vault, mount allowlist, IPC auth via directory path), SQLite migration patterns, channel registry architecture
- **Do Not Trust**: README claims about "simple setup" — actual setup involves Docker, OneCLI, browser deps, and env migration

## At a Glance
- **Language**: TypeScript (strict mode, NodeNext ESM)
- **Architecture**: Single-process event loop + filesystem IPC + SQLite persistence + container pool
- **Key Libraries**: `better-sqlite3`, `pino`, `zod`, `@onecli/sh/sdk`, `@anthropic-ai/claude-agent-sdk`
- **Notable Patterns**: Registry/Factory (channels), Observer (GroupQueue), Circuit Breaker (exponential backoff), Leased Concurrency (bounded container pool)
- **Stars / Activity**: 657 commits, extremely active daily releases, 2 core maintainers + 8+ contributors
- **License**: MIT

## Repository
- **Source**: https://github.com/qwibitai/nanoclaw
- **Docs**: https://docs.nanoclaw.dev
- **Discord**: https://discord.gg/VDdww8qS42

## Study Structure
```
research/          # Raw research from parallel subagents (preserved)
  01-topology.md
  02-tech-stack.md
  03-community.md
  04-features-index.md
  05a-features-batch-1.md   # Features 1-3
  05b-features-batch-2.md   # Features 4-6
  05c-features-batch-3.md   # Features 7-9
  05d-features-batch-4.md   # Features 10-12
  05e-features-batch-5.md   # Features 13-14
  06-architecture.md
  07-code-quality.md
  08-security-perf.md

learning/          # Synthesized learning documents
  01-project-overview.md
  02-architecture.md
  03-tech-stack.md
  04-features-deep-dive.md
  05-code-quality.md
  06-ci-cd.md
  07-documentation.md
  08-security.md
  09-dependencies.md
  10-community.md
  11-patterns.md
  12-lessons-learned.md
  13-my-action-items.md
```
