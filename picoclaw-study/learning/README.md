# PicoClaw — Learning Reference

## What This Is
Analysis of the [PicoClaw](https://github.com/sipeed/picoclaw) codebase (v0.2.4, March 2026) for informing development of a similar AI agent platform.

## Key Takeaways
- **Event-driven architecture** with central MessageBus is the core innovation — enables clean separation between 17+ chat channels and the agent core
- **Factory registry pattern** (self-registering via init()) for channels, providers, and tools is the primary extension mechanism
- **Fallback chain** with cooldown tracking and 8-type error classification enables graceful degradation across 30+ LLM providers
- **Two-phase cleanup** (lock-free map modification + lock-free file deletion) minimizes contention in the media pipeline
- **Secrets in separate .security.yml** with AES-256-GCM encryption and SSH key as second factor is a robust credential pattern

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, error classification system, channel work queue pattern, hook system design
- **Do Not Trust**: README claims about simplicity (107KB AgentLoop is not simple); "ultra-lightweight" branding (10-20MB actual RAM)

## At a Glance
- **Language:** Go 1.25 (backend), TypeScript/React 19 (frontend)
- **Architecture:** Event-driven, channel-based multi-agent with pub/sub MessageBus
- **Key Libraries:** mautrix (Matrix), whatsmeow (WhatsApp), standard net/http (no Gin/Echo), modernc.org/sqlite, anthropic/OpenAI SDKs
- **Notable Patterns:** Factory registry, circuit breaker, interceptor chain, TTL cache, work queue
- **Stars / Activity:** 26,175 stars, very active (100+ commits in 2026, bi-weekly releases)
- **License:** Apache 2.0

## Study Structure
```
learning/
├── 01-project-overview.md   # High-level purpose and scope
├── 02-architecture.md      # Architectural decisions
├── 03-tech-stack.md        # Languages, frameworks, key libraries
├── 04-features-deep-dive.md # All 20 features synthesized
├── 05-code-quality.md      # Quality practices, linting, testing
├── 06-ci-cd.md             # GitHub Actions, GoReleaser, Makefile
├── 07-documentation.md    # Docs structure and community resources
├── 08-security.md          # Secrets, auth, encryption, SSRF
├── 09-dependencies.md      # Dependency analysis by category
├── 10-community.md         # Star history, contribution guidelines
├── 11-patterns.md          # 15 design patterns with code evidence
├── 12-lessons-learned.md   # What to emulate, avoid, surprises
└── 13-my-action-items.md   # Prioritized build plan (5 phases)

research/                   # PRESERVED — evidence from parallel subagents
├── 01-topology.md
├── 02-tech-stack.md
├── 03-community.md
├── 04-features-index.md
├── 05a-features-batch-1.md  (through 05g-features-batch-7.md)
├── 06-architecture.md
├── 07-code-quality.md
└── 08-security-perf.md
```
