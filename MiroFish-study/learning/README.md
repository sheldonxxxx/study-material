# MiroFish — Learning Reference

## What This Is
Analysis of the MiroFish codebase — an AI-powered swarm intelligence simulation engine for predictive analytics. This study was conducted to inform development of similar projects in the multi-agent simulation / GraphRAG space.

## Key Takeaways
- **Architecture**: Vue 3 + Flask monolithic app with file-based IPC; service-layer separation; daemon threads for background tasks; polling-based real-time updates
- **AI Stack**: CAMEL-OASIS for simulation, Zep Cloud for knowledge graph/memory, OpenAI for LLM; ReACT pattern for tool-augmented agents
- **Security is an afterthought**: No auth on any endpoint, path traversal vulnerability, full tracebacks exposed, hardcoded SECRET_KEY — do NOT replicate these
- **Quality infrastructure is missing**: No tests, no linting, no type enforcement, no CI quality gates — monolithic files (simulation.py at 2,712 lines)
- **Single maintainer** (99.5% of commits) with 43K stars suggests a project in maintenance mode, not active development

## Should Trust / Should Not Trust
- **Trust**: Architectural patterns (service layer, retry with backoff, progress callbacks), error handling patterns, dual logging system
- **Do Not Trust**: README claims about simplicity (verify with code — monolithic files contradict this), security practices

## At a Glance
- **Language**: Python 3.11 (backend) + Vue 3 + TypeScript (frontend)
- **Architecture**: Layered monolithic (Flask + Vue SPA, file-based IPC, no message queue)
- **Key Libraries**: Flask, Pydantic, camel-oasis, camel-ai, zep-cloud, Vue 3, D3.js
- **Notable Patterns**: Service Layer, Manager/Singleton, Factory (LLMClient), Observer (progress callbacks), Retry with backoff, File-based IPC
- **Stars / Activity**: 43,344 stars · 5,992 forks · 110 open issues · 87 open PRs · ~monthly releases
- **License**: AGPL-3.0

## Study Structure
```
MiroFish-study/
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
└── research/          # Preserved evidence from parallel subagents
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
