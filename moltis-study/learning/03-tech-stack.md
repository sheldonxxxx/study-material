# Tech Stack

## Language & Toolchain

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Language** | Rust (nightly) | Systems programming with memory safety; ideal for AI gateway with sandboxed tool execution |
| **Edition** | 2024 | Latest Rust edition with improved async capabilities |
| **Toolchain** | `nightly-2025-11-30` (pinned) | Stable nightly features (edition 2024, WASM) with reproducible builds |
| **Minimum Rust** | 1.91 | Enables latest language features while maintaining broad compatibility |
| **WASM Target** | `wasm32-wasip2` | WASI Preview 2 for guest tool sandboxing |
| **Resolver** | version 2 | Required for complex workspace with 52 crates |

## Why Rust?

Moltis is an AI gateway that handles:
- **Sandboxed tool execution**: Rust's memory safety + WASM runtime isolate untrusted code
- **50+ concurrent channel connections**: Tokio async runtime handles thousands of simultaneous connections
- **Plugin system**: Rust's trait system and zero-cost abstractions enable safe plugin interfaces
- **Performance**: LLM interactions are I/O-bound; Rust's async handles latency without heavy threads

## Core Framework Versions

| Library | Version | Purpose |
|---------|---------|---------|
| **tokio** | full features | Async runtime for all async operations |
| **axum** | 0.8 | HTTP server with WebSocket support for gateway |
| **axum-extra** | 0.10 | Cookie support for session management |
| **async-graphql** | 7 | GraphQL API layer (used for iOS app sync) |
| **sqlx** | 0.8 | Async SQLite with compile-time query validation |
| **serde** / **serde_json** | latest | Serialization for config, API, persistence |

## Key Library Choices

### HTTP Stack
- **axum + tower-http**: Modular middleware (auth, logging, rate limiting)
- **reqwest**: HTTP client for LLM provider APIs
- **askama**: Server-side HTML templates for web UI (no JS framework overhead)

### LLM Integration
- **async-openai**: OpenAI API client (GPT-4, o3, etc.)
- **genai**: Generic AI abstraction layer for multi-provider support
- **llama-cpp-2**: Local model inference (llama.cpp bindings)

### Messaging Channels
| Channel | Library | Notes |
|---------|---------|-------|
| Discord | serenity 0.12 | Mature Rust Discord library |
| Slack | slack-morphism 2.6 | Slack API client |
| Telegram | teloxide 0.13 | Ergonomic Telegram bot framework |
| WhatsApp | waprot o + whatsapp-rust | Protocol implementations |
| MS Teams | Custom (msteams crate) | Direct Bot Framework integration |

### Security
| Library | Purpose |
|---------|---------|
| **argon2** | Password hashing (not MD5/SHA) |
| **webauthn-rs** | Passkey/WebAuthn authentication |
| **chacha20poly1305** | Symmetric encryption for secrets |
| **rustls** | TLS (ring backend, faster than OpenSSL) |
| **secrecy** | Wraps sensitive values to prevent accidental logging |

### WASM Runtime
- **wasmtime 36**: Production-grade WASM runtime with component model support
- **wit-bindgen 0.41**: Generates bindings from WIT interface definitions
- **pre-compiled `.cwasm`**: Guest tools pre-compiled to AOT for faster startup

## Web UI Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Framework** | Preact | Lightweight (3KB) alternative to React; works without build step |
| **Styling** | Tailwind CSS v4 | Utility-first; compiled via `tailwindcss` CLI |
| **Build** | Vite (via Biome/Node) | Fast HMR during development |
| **Server** | Axum + Askama | Serves pre-rendered HTML + static assets |
| **Testing** | Playwright | E2E tests for critical user flows |

## Architecture Patterns

### Monorepo Structure (52 crates)
```
apps/           # Binary applications
  courier/      # APNS push notification relay
crates/         # Core libraries
  agents/       # Agent orchestration
  channels/     # Unified channel trait
  providers/    # LLM provider implementations
  plugins/      # Plugin system
  tools/        # Tool execution + sandboxing
  wasm-tools/   # WASM guest tools
```

### Why Monorepo?
- Shared `crates/` prevents duplicated channel/provider code
- Single `Cargo.lock` ensures compatible versions across all crates
- Easier cross-crate refactoring (e.g., auth changes propagate compile-time)

## Notable Technical Decisions

| Decision | Tradeoff | Rationale |
|----------|----------|-----------|
| **nightly Rust** | Instability risk | Edition 2024 features, WASM component model unavailable on stable |
| **SQLite over Postgres** | Limited concurrency | Single-file simplicity; sqlx async handles most use cases |
| **TOML config** | Human-editable | User-friendly over YAML/JSON; validated via `crates/config` |
| **Date versioning** | Non-semver | Reflects fast iteration pace; `YYYYMMDD.NN` shows activity |
| **Docker socket mount** | Container escape risk | Allows tool sandboxing; documented SSRF protection mitigates risk |
