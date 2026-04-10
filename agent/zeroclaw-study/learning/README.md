# ZeroClaw — Learning Reference

## What This Is
Analysis of the ZeroClaw codebase (Rust-based personal AI assistant with 45+ messaging channel integrations, hardware peripheral support, and multi-agent orchestration) for the purpose of informing development of similar agent framework projects.

## Key Takeaways

- **Trait-based pluggable architecture** is the core design principle — every major subsystem (providers, channels, tools, memory, sandbox) uses `dyn Trait` for swappability. Study `src/providers/traits.rs`, `src/channels/traits.rs`, `src/tools/traits.rs` as the architecture foundation.
- **Defense-in-depth security** with 5 independent shell validation gates, quote-aware parsing, per-segment risk classification, and path canonicalization. The `src/security/policy.rs` (108KB) IAM policy engine is the most sophisticated piece.
- **Single monolithic agent loop** (`src/agent/loop_.rs` — 9,506 lines) is the primary technical debt. It handles too many concerns and uses a global static for model switching — avoid this pattern.
- **3-stage memory retrieval** (LRU hot cache → FTS5 → vector search) with SQLite WAL mode and importance-weighted scoring is a proven pattern for agent memory at scale.
- **Builder pattern for dependency injection** (AgentBuilder) with Arc<dyn Trait> for shared components and Box<dyn Trait> for owned components works well for complex runtime assembly.

## Should Trust / Should Not Trust

- **Trust**: Trait abstractions, security policy validation logic, SQLite patterns, provider failover architecture, sandbox trait implementations
- **Do Not Trust**: README claims about "simple" setup (verify with code — the implementation is complex), CHANGELOG.md (it is empty), line counts of large files as complexity indicators (loop_.rs is 9,506 lines but is actually well-structured within)

## At a Glance

| Aspect | Value |
|--------|-------|
| **Language** | Rust (1.87+, 2021 edition) |
| **Architecture** | Modular monolithic agent framework with trait-based pluggability |
| **Key Libraries** | tokio, axum, reqwest, rusqlite, rustls, serde |
| **AI Providers** | OpenAI, Anthropic, Gemini, Bedrock, Ollama, OpenRouter, Codex + more via traits |
| **Channels** | 45+ (Discord, Slack, Matrix, WhatsApp, Telegram, Signal, iMessage, etc.) |
| **Notable Patterns** | Builder, Registry, Strategy (ToolDispatcher), Wrapper (ReliableProvider), Observer, Hook |
| **Stars / Activity** | Active project with frequent releases (0.6.x), 275 contributors |
| **License** | MIT + Apache-2.0 |
| **Security** | ChaCha20-Poly1305 secrets, WebAuthn, 5-layer shell validation, sandbox isolation (Landlock/Bubblewrap/Firejail/Seatbelt) |
| **CI/CD** | GitHub Actions — lint, test, cross-platform build, Docker, Homebrew/AUR/Scoop publish |

## Study Structure

```
zeroclaw-study/
├── learning/
│   ├── README.md              # This file
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependendencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/                  # PRESERVED — evidence of parallel subagent work
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

## Quick-Start Action Items

From `13-my-action-items.md`:

1. **Define core trait abstractions first** — Provider, Tool, Channel, Memory before implementing any feature
2. **Use AgentBuilder pattern** for dependency injection — avoid globals
3. **Keep agent loop under 500 lines** — delegate to Strategy/Handler modules
4. **Use `tokio::task_local!`** for request-scoped context isolation
5. **SQLite WAL mode + FTS5** for memory persistence
6. **ChaCha20-Poly1305** for secrets, not any other cipher
7. **Landlock** as first sandbox option (works on Linux without root)
8. **Enable `#![forbid(unsafe_code)]`** workspace-wide except FFI bindings
9. **Set up deny.toml** for license and security advisory checking from day one
