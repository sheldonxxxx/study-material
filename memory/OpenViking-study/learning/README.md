# OpenViking — Learning Reference

## What This Is
Analysis of the [OpenViking](https://github.com/volcengine/OpenViking) codebase — an Agent-native context database with virtual filesystem paradigm — for informing development of similar projects.

## Key Takeaways
- **Virtual Filesystem (`viking://`)** is the core abstraction unifying all context; design URI schemes early
- **Tiered context (L0/L1/L2)** solves token cost problems elegantly — build this from day one
- **Polyglot is worth it**: Python for flexibility, Rust for CLI performance, C++ for SIMD vector ops
- **Security is mature**: Argon2id API keys, envelope encryption, multi-tenant RBAC, timing-safe comparisons
- **Quality gaps exist**: Full test suite disabled, TypeScript path has zero tooling, mypy non-blocking in CI

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions (adapter pattern, service composition, dual client modes), security implementation (Argon2id, envelope encryption, HMAC), async-first design
- **Do Not Trust**: README simplicity claims (code is complex); "full test suite" (actually disabled); TypeScript quality (no ESLint/Prettier/tsc at all)

## At a Glance
- **Language:** Python (79%), C++ (6%), Rust (3%), Go, TypeScript
- **Architecture:** Hybrid Python+Rust+C++ — Facade/Adapter/Strategy/Registry patterns
- **Key Libraries:** pydantic v2, FastAPI, tree-sitter, asyncio, ratatui, LevelDB, LiteLLM
- **Notable Patterns:** Virtual filesystem (viking://), L0/L1/L2 tiered context, score propagation retrieval, envelope encryption, circuit breaker, async-first with zero-overhead telemetry
- **Stars / Activity:** ~19,271 stars, 560 commits in ~3 months, accelerating trajectory
- **License:** Apache-2.0

## Study Structure
```
OpenViking-study/
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
└── research/          # Intermediate research (preserved)
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
