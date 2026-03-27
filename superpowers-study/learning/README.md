# Superpowers — Learning Reference

## What This Is
Analysis of the Superpowers codebase (v5.0.6) — a multi-platform AI coding agent plugin providing a complete software development workflow via composable skills. Study conducted to inform development of similar workflow frameworks.

## Key Takeaways
- **Zero-dependency philosophy** eliminates supply chain risk — the WebSocket brainstorm server is a complete RFC 6455 implementation using only Node.js built-ins
- **Iron Law enforcement pattern** — explicit "NO X WITHOUT Y" prohibitions are the primary discipline mechanism (no CI enforcement, purely prompt-based)
- **Skills as Markdown with YAML frontmatter** — discovery via description field, no registry, flat namespace
- **Context construction over inheritance** — subagents get fresh, precisely-constructed context rather than inheriting parent state
- **Rationalization resistance is a core design pattern** — tables explicitly naming common excuses ("You're right, I'll add that later") appear throughout

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, zero-dependency implementation, RFC-compliant protocol handling, Graphviz flowchart-driven process control
- **Do Not Trust**: README claims about simplicity (verify with code) — the Visual Companion server alone is 355 lines with sophisticated state management

## At a Glance
- **Language**: Markdown (skills) + JavaScript/ESM (server) + Shell (hooks)
- **Architecture**: Hub-and-spoke (platform plugins → skills hub) with layered module structure
- **Key Libraries**: None (zero runtime dependencies)
- **Notable Patterns**: Iron Law enforcement, rationalization tables, two-stage review (spec→quality), TDD applied to documentation, fresh-context subagents
- **Stars / Activity**: ~5 months active development (Oct 2025–Mar 2026), ~400 commits, single-maintainer (Jesse Vincent @obra)
- **License**: MIT

## Project Structure
```
superpowers-study/
├── learning/          # 13 synthesized learning documents
│   ├── README.md      # This file
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
└── research/          # Preserved intermediate research (8 files)
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-e-features-batch-*.md (5 batch files)
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

## Most Actionable Insights
1. **Start with verification and debugging discipline** — everything else is compromised without it
2. **Use Iron Law formulations** with explicit anti-rationalization tables for any enforcement mechanism
3. **Fine-grained tasks (2-5 min)** with exact file paths and code examples prevent scope creep
4. **Two-stage review**: spec compliance BEFORE code quality — catch direction errors early
5. **Zero-dependency architecture** is achievable and worth the implementation cost for critical paths
