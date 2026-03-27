# Agency-Agents — Learning Reference

## What This Is
Analysis of the [agency-agents](https://github.com/msitarzewski/agency-agents) repository — a collection of 144+ AI agent personality definitions for AI coding tools (Claude Code, Cursor, Windsurf, Aider, etc.).

## Key Takeaways
- **Architecture wins**: Single-source markdown → multi-tool conversion pipeline is elegant and extensible
- **Content quality varies wildly**: Agents range from 33 lines to 400+ lines — no minimum standards enforced
- **Ghost files undermine credibility**: Several agents reference non-existent files (qa-playwright-capture.sh, etc.)
- **Schema validation is missing**: YAML frontmatter has no JSON Schema — quality gates are shallow
- **Community is healthy**: 63K stars, active development, but maintainer concentration is a risk (67% commits from one person)

## Should Trust / Should Not Trust
- **Trust**: Conversion pipeline architecture, design patterns, CONTRIBUTING.md quality
- **Do Not Trust**: README simplicity claims (code reveals far more complexity), agent completeness claims (verify references exist)

## At a Glance
- **Language:** Markdown (primary content), Bash (scripts), TypeScript/Python/Solidity (example code)
- **Architecture:** Content publishing system — markdown + YAML → multi-tool converter
- **Key Libraries:** None (zero external dependencies)
- **Notable Patterns:** Visitor/Transformer, Schema+Content Separation, Factory, Strategy, Facade
- **Stars / Activity:** 63,112 stars, 150+ contributors, ~45 commits/day on peak days
- **License:** MIT

## Project Structure
```
agency-agents-study/
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
└── research/          # Raw subagent research (preserved)
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-features-batch-1.md
    ├── 05b-features-batch-2.md
    ├── 05c-features-batch-3.md
    ├── 05d-features-batch-4.md
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

## Source Repository
https://github.com/msitarzewski/agency-agents
