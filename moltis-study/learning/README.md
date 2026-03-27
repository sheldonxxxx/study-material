# Moltis — Learning Reference

## What This Is
Analysis of the [Moltis](https://github.com/molti-org/moltis) codebase — a Rust-based personal AI gateway — for informing development of similar self-hosted AI infrastructure projects.

## Key Takeaways

- **Trait-based plugin architecture** enables 5 channel integrations (Discord, Slack, Telegram, WhatsApp, MS Teams) and 7+ LLM providers without modifying core logic. Define traits early; implement against them.
- **Vault security design** (Argon2id → KEK → random DEK → XChaCha20-Poly1305) with DEK/KEK separation and 5-min WebAuthn challenge TTL is a strong model for secrets-at-rest.
- **Multi-layer policy approval** for tool execution (6 layers, deny-always-wins) with WASM sandboxing provides defense-in-depth without sacrificing extensibility.
- **Production-grade quality tooling** from day one: pinned nightly toolchain, three-layer testing (unit/integration/E2E via Playwright), `thiserror` enums with Context trait, `secrecy::Secret` wrappers, and comprehensive CI gates.
- **Highly centralized project** (94%+ commits from one maintainer) with exceptional automation compensating for solo sustainability — dated versioning, `just` task runner, 5 GitHub Actions workflows.

## Should Trust / Should Not Trust

- **Trust**: Architectural patterns (layered architecture, plugin traits, service traits with Noop defaults), security design (SSRF protection, WASM fuel metering, SQL parameterized queries), quality practices (tracing instrumentation, metrics facade, three-layer tests)
- **Do Not Trust**: README claims about "simple" setup (verify with code — 52-crate monorepo, complex multi-backend sandbox system); documentation accuracy (audit against actual implementation)
- **Caution**: Single-user auth model baked into `moltis-auth`; error classification via string matching; very large files (runner.rs 226K, chat/lib.rs 457K)

## At a Glance

| | |
|---|---|
| **Language** | Rust (nightly-2025-11-30, edition 2024, 52-crate workspace) |
| **Architecture** | Layered monorepo with trait-based plugin system |
| **Key Libraries** | tokio, axum, sqlx, async-graphql, wasmtime, serenity/teloxide/slack-morphism, argon2, webauthn-rs |
| **Notable Patterns** | Plugin traits (ChannelPlugin, ToolPlugin), Registry pattern, Vault DEK/KEK separation, multi-layer policy merging, ProviderChain circuit breaker |
| **Testing** | Unit + integration + Playwright E2E (3 layers) |
| **Activity** | 11 releases in 3 weeks (March 2026), v0.10.x series |
| **License** | MIT |
| **Maintainer** | Fabien Penso (94.2% of 2,110 commits) |
| **Stars** | ~2.5K (estimate — gh CLI unavailable during study) |

## Repository

```
/Users/sheldon/Documents/claw/reference/moltis/
```

## Study Structure

```
moltis-study/
├── learning/          # 13 synthesized learning documents
│   ├── README.md      # This file
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/          # 12 raw research files (preserved as evidence)
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

## Most Valuable Files for Reference

1. **`12-lessons-learned.md`** — 19 concrete lessons with code examples (what to emulate, what to avoid, surprises)
2. **`13-my-action-items.md`** — 18 prioritized next steps organized in 6 phases
3. **`11-patterns.md`** — 10 design patterns with file references and evidence
4. **`04-features-deep-dive.md`** — 15 features analyzed with implementation details
5. **`08-security.md`** — Security patterns worth directly emulating
