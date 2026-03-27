# OpenFang — Learning Reference

## What This Is
Analysis of the OpenFang codebase — an "Agent Operating System" built in Rust — for the purpose of informing development of similar agent framework projects.

## Key Takeaways
- **16-layer security architecture** is the most sophisticated in the agent framework landscape — WASM dual-metering, Merkle hash-chain audit, taint tracking, and SSRF protection with DNS rebinding defense
- **Kernel/Runtime trait separation** via `KernelHandle` trait inversion is a clean pattern that prevents circular dependencies between orchestration and execution
- **Operational resilience patterns** (loop guard, circuit breaker, phantom action detection, empty response retry) are deeply integrated throughout the runtime
- **1866 `.unwrap()` occurrences** is the most concerning technical debt — high density of panic paths in a production system
- **Integration gaps** exist between marketed features: WASM sandbox is not wired to the skills system, MCP tool execution is stubbed, OFP consolidation module doesn't exist

## Should Trust / Should Not Trust
- **Trust**: Architectural decisions (kernel/runtime split, event bus, capability gates), security implementations (SSRF, loop guard, zeroization), multi-provider LLM abstraction, channel adapter trait
- **Do Not Trust**: README claims about "137K LOC, 1,767+ tests, zero clippy warnings" (not independently verified); Feature list descriptions vs. actual code (e.g., MCP server tool execution is stubbed); WASM runtime in skills (returns `RuntimeNotAvailable`)

## At a Glance
- **Language**: Rust (1.75 MSRV) — 14 workspace crates
- **Architecture**: Layered Agent OS with Kernel/Runtime separation, Event Bus pub/sub
- **Key Libraries**: Tokio (async), Axum 0.8 (HTTP), Wasmtime 41 (WASM), Tauri 2.0 (desktop), rusqlite (SQLite), AES-GCM/Argon2/Ed25519 (crypto)
- **Notable Patterns**: KernelHandle trait inversion, Event Bus, WASM dual-metered sandbox, HMAC mutual auth, Merkle hash-chain audit, taint tracking lattice, GCRA rate limiting, circuit breaker
- **Stars / Activity**: 150 stars, very high commit activity (daily), v0.5.2 released 2026-03-26
- **License**: MIT + Apache-2.0 (dual)
- **Bus Factor Risk**: High — single dominant maintainer (jaberjaber23) responsible for 75% of commits

## Document Map
| File | Content |
|------|---------|
| `01-project-overview.md` | High-level summary, purpose, competitive positioning |
| `02-architecture.md` | Architectural decisions, module responsibilities, communication patterns |
| `03-tech-stack.md` | Language choices, frameworks, runtime architecture |
| `04-features-deep-dive.md` | 14 feature analyses with implementation details |
| `05-code-quality.md` | Testing, types, linting, error handling, technical debt |
| `06-ci-cd.md` | Build pipeline, release process, test strategy |
| `07-documentation.md` | Doc coverage, CONTRIBUTING guide quality |
| `08-security.md` | Security posture, credential management, protection matrix |
| `09-dependencies.md` | Dependency health, vulnerability posture |
| `10-community.md` | Activity, governance, contributor diversity |
| `11-patterns.md` | 20 design patterns with code evidence |
| `12-lessons-learned.md` | What to emulate, what to avoid, surprises |
| `13-my-action-items.md` | Prioritized next steps for building a similar project |

## Most Implemented Security Layers (verified)
1. WASM dual-metered sandbox (fuel + epoch)
2. Merkle hash-chain audit trail with SQLite persistence
3. Information flow taint tracking (5-label lattice)
4. SSRF protection with DNS rebinding defense
5. Secret zeroization via `Zeroizing<String>`
6. HMAC-SHA256 OFP mutual authentication
7. Loop guard with SHA256 call hashing + outcome awareness
8. GCRA rate limiting (500 tokens/min/IP)
9. Subprocess sandbox with `env_clear()`
10. Path canonicalization with symlink escape prevention

## Biggest Technical Debt
1. 1866 `.unwrap()`/`.expect()` across 162 files
2. Unbounded audit log growth (no pruning)
3. Vault keyring is XOR obfuscation, not true OS keyring
4. WASM sandbox not wired to skills system
5. MCP server tool execution is stubbed
6. OFP consolidation module listed in docs but doesn't exist
