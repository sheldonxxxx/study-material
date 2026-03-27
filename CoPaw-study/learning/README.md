# CoPaw — Learning Reference

## What This Is
Analysis of the [CoPaw](https://github.com/agentscope-ai/CoPaw) codebase for the purpose of informing development of a similar multi-agent AI orchestration platform.

## Key Takeaways
- **AgentScope as foundation** provides ReAct agents, ChatModel abstractions, and FormatterBase — build on top rather than reinventing
- **Registry + Factory patterns** enable extensible channels (12+) and providers (5+) without code changes
- **Defense-in-depth security** with Tool Guard (file access), Skill Scanner (pre-execution), and Rule Guardian (shell commands)
- **Three-tier CLI** via LazyGroup achieves 19 subcommands without heavy import overhead
- **Multi-channel message handling** with debouncing, queue coalescing, and per-channel session resolution is well-architected
- **Technical debt to note**: `discord_/` trailing underscore hack, large files (35KB channels_cmd.py), hardcoded model lists, no API rate limiting

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, security patterns, multi-agent coordination, async/FastAPI patterns
- **Do Not Trust**: README claims about simplicity (verify with code) — the implementation is sophisticated

## At a Glance
- **Language**: Python 3.10-3.13 (backend), TypeScript 5.6-5.8 (frontend)
- **Architecture**: Multi-Layered Event-Driven with Plugin Extensibility (Registry/Factory/Mixin patterns)
- **Key Libraries**: FastAPI, AgentScope, React 18, Ant Design, APScheduler, Playwright, pywebview
- **Notable Patterns**: Registry, Factory, Strategy, Template Method, Observer, Hook, Debounce Queue, Mixin
- **Stars / Activity**: Very high (30+ commits in 3 days as of 2026-03)
- **License**: Apache 2.0

## Repository
`/Users/sheldon/Documents/claw/reference/CoPaw`

## Study Structure
```
CoPaw-study/
├── learning/          # Synthesized learning documents
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/          # Intermediate research from subagents
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-features-batch-1.md
    ├── 05b-features-batch-2.md
    ├── 05c-features-batch-3.md
    ├── 05d-features-batch-4.md
    ├── 05e-features-batch-5.md
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```
