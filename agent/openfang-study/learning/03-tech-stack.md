# OpenFang Tech Stack

## Language Choices

| Language | Role | Details |
|----------|------|---------|
| **Rust** | Core runtime, CLI, API server, agent framework | 14 crates in `crates/`; MSRV 1.75, Edition 2021 |
| **JavaScript/TypeScript** | SDK for client integrations | `sdk/javascript/` with `index.js`, `index.d.ts` |
| **Python** | SDK for client integrations | `sdk/python/openfang_sdk.py`, `openfang_client.py` |
| **Nix** | Declarative package management | `flake.nix` using `flake-parts` + `juspay/rust-flake` |
| **Shell** | Install scripts | `scripts/` directory |
| **Dockerfile** | Multi-stage container builds | `rust:1-slim-bookworm` base |

---

## Rust Crate Architecture

### Workspace Structure

```
14 crates in `crates/`
MSRV: Rust 1.75 | Edition: 2021 | Toolchain: Stable
Resolver: "2" (workspace resolver v2)
```

### Crate Inventory

| Crate | Purpose |
|-------|---------|
| `openfang-types` | Core types, traits, interfaces, taint tracking, Ed25519 manifest signing, model catalog, MCP/A2A config types |
| `openfang-memory` | SQLite-backed memory substrate with vector embeddings, usage tracking, canonical sessions, JSONL mirroring |
| `openfang-runtime` | Agent execution loop, 3 LLM drivers (Anthropic/Gemini/OpenAI-compat), 38 built-in tools, WASM sandbox, MCP client/server, A2A protocol |
| `openfang-wire` | OpenFang Protocol (OFP) -- TCP P2P networking with HMAC-SHA256 mutual authentication |
| `openfang-api` | HTTP/WebSocket API server using Axum 0.8; 76 endpoints, 14-page SPA dashboard, OpenAI-compatible `/v1/chat/completions` |
| `openfang-kernel` | Core kernel orchestrating all subsystems: workflow engine, RBAC auth, heartbeat monitor, cron scheduler, config hot-reload |
| `openfang-cli` | CLI binary entry point (`openfang` command) with daemon auto-detect (HTTP mode vs. in-process fallback) |
| `openfang-channels` | Pluggable messaging integrations: 40 channel adapters (Telegram, Discord, Slack, WhatsApp, +36 more) |
| `openfang-skills` | Skill registry, loader, marketplace, OpenClaw compatibility, prompt injection scanning |
| `openfang-hands` | Curated autonomous capability packages; 7 bundled hands |
| `openfang-extensions` | Extension system: 25 bundled MCP templates, AES-256-GCM credential vault, OAuth2 PKCE |
| `openfang-migrate` | Migration engine for importing from OpenClaw (and future frameworks) |
| `openfang-desktop` | Tauri 2.0 native desktop application (WebView + system tray + single-instance + notifications) |
| `xtask` | Build automation tasks |

---

## Key Library Versions

### Async Runtime
- **Tokio** `1` (full features) -- async runtime
- **Tokio Stream** `0.1` -- stream utilities for async iteration

### HTTP Server (API Daemon)
- **Axum** `0.8` (with `ws` feature) -- modular web framework with WebSocket support
- **Tower** `0.5` -- middleware foundation
- **Tower HTTP** `0.6` (with `cors`, `trace`, `compression-gzip`, `compression-br`) -- middleware

### HTTP Client (LLM Drivers)
- **reqwest** `0.12` -- HTTP client with `json`, `stream`, `multipart`, `rustls-tls`, `gzip`, `deflate`, `brotli`
- **tokio-tungstenite** `0.24` -- WebSocket client for Discord/Slack gateways

### Serialization
- **Serde** `1` (with `derive`) -- serialization framework
- **Serde JSON** `1` -- JSON serialization
- **TOML** `0.8` -- TOML config parsing
- **rmp-serde** `1` -- MessagePack for wire protocol
- **Serde YAML** `0.9` -- YAML parsing
- **JSON5** `0.4` -- JSON5 parsing

### Database
- **rusqlite** `0.31` (bundled, with `serde_json`) -- SQLite with bundled SQLite

### Security / Crypto
- **aes-gcm** `0.10` -- authenticated encryption (AES-256-GCM)
- **argon2** `0.5` -- password hashing
- **ed25519-dalek** `2` (with `rand_core`) -- digital signatures
- **HMAC-SHA2** `0.12` -- message authentication
- **SHA2** `0.10` -- hashing
- **SHA1** `0.10` -- hashing
- **subtle** `2` -- constant-time operations
- **zeroize** `1` (with `derive`) -- memory zeroing

### WASM Runtime
- **wasmtime** `41` -- WASM sandbox for skill isolation

### CLI
- **clap** `4` (with `derive`) -- CLI argument parsing
- **clap_complete** `4` -- shell completion generation
- **ratatui** `0.29` -- terminal UI library
- **colored** `3` -- terminal colors

### Email Integration
- **lettre** `0.11` -- SMTP client with `tokio1`, `rustls-tls`
- **imap** `2` -- IMAP client
- **mailparse** `0.16` -- MIME parsing
- **native-tls** `0.2` -- TLS support

### Observability
- **tracing** `0.1` -- structured logging and diagnostics
- **tracing-subscriber** `0.3` (with `env-filter`, `json`) -- log subscriber

### Concurrency & Utilities
- **dashmap** `6` -- concurrent HashMap
- **crossbeam** `0.8` -- parallelism primitives
- **futures** `0.3` -- async abstractions
- **thiserror** `2` -- error types
- **anyhow** `1` -- error handling
- **uuid** `1` (with `v4`, `v5`, `serde`) -- ID generation
- **chrono** `0.4` (with `serde`) -- time handling
- **chrono-tz** `0.10` -- timezone support
- **walkdir** `2` -- directory walking
- **url** `2` -- URL parsing
- **base64** `0.22` -- encoding
- **bytes** `1` -- byte handling
- **hex** `0.4` -- hex encoding
- **rand** `0.8` -- randomness
- **governor** `0.8` -- rate limiting
- **regex-lite** `0.1` -- regex
- **socket2** `0.5` -- socket options
- **zip** `4` -- archive extraction
- **dirs** `6` -- home directory resolution
- **html-escape** `0.2` -- HTML entity decoding
- **openssl** `0.10` (vendored) -- OpenSSL with vendored crypto

### Desktop
- **Tauri 2.0** -- native desktop framework

---

## Build System

### Cargo Profiles

| Profile | LTO | Codegen Units | Strip | Opt Level |
|---------|-----|---------------|-------|-----------|
| `release` | fat | 1 | true | 3 |
| `release-fast` | thin | 8 | false | 2 |

### Nix Flake Support

- Uses `flake-parts` with `juspay/rust-flake` for declarative Rust builds
- Supported platforms: `x86_64-linux`, `aarch64-linux`, `aarch64-darwin`, `x86_64-darwin`
- `openfang-desktop` requires GTK3/WebKit dependencies on Linux

### Cross-Compilation

- **Cross** tool for ARM64 Linux cross-compilation
- `Cross.toml` adds `libssl-dev:$CROSS_DEB_ARCH` for `aarch64-unknown-linux-gnu`

### Docker

Multi-stage build:
1. **Builder stage**: `rust:1-slim-bookworm` with LTO/codeGen tuning
2. **Runtime stage**: `rust:1-slim-bookworm` with Python3, Node.js, npm

---

## Runtime Architecture

```
                    ┌─────────────────────────────┐
                    │        openfang-cli         │
                    │    (CLI binary entrypoint)   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │       openfang-api          │
                    │  (Axum HTTP/WebSocket daemon) │
                    └──────────────┬──────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
┌────────▼────────┐    ┌──────────▼──────────┐    ┌────────▼────────┐
│ openfang-kernel │◄───│  openfang-runtime   │───►│ openfang-channels│
│   (orchestrator)│    │ (WASM skill sandbox)│    │ (Discord/Slack)  │
└────────┬────────┘    └─────────────────────┘    └──────────────────┘
         │
  ┌──────┴──────┬──────────────┬──────────────┐
  │             │              │              │
┌─▼────┐  ┌─────▼────┐  ┌─────▼─────┐  ┌─────▼─────┐
│memory│  │  types   │  │   wire    │  │ extensions│
│(SQLite)│ │          │  │   (OFP)   │  │  (MCP)    │
└───────┘  └─────────┘  └───────────┘  └───────────┘
```

### Key Architectural Patterns

- **`KernelHandle` trait**: Defined in `openfang-runtime`, implemented on `OpenFangKernel` in `openfang-kernel`. Avoids circular crate dependencies while enabling inter-agent tools.
- **Shared memory**: Fixed UUID (`AgentId(Uuid::from_bytes([0..0, 0x01]))`) provides cross-agent KV namespace.
- **Daemon detection**: CLI checks `~/.openfang/daemon.json` and pings health endpoint. If daemon running, commands use HTTP; otherwise, boots in-process kernel.
- **Capability-based security**: Every agent operation checked against granted capabilities before execution.

---

## SDKs

| SDK | Path | Language | Entry Point |
|-----|------|----------|-------------|
| JavaScript/TypeScript | `sdk/javascript/` | JS/TS | `index.js`, `index.d.ts` |
| Python | `sdk/python/` | Python | `openfang_sdk.py`, `openfang_client.py` |

---

## Summary Table

| Dimension | Choice |
|-----------|--------|
| **Core Language** | Rust (1.75+, stable) |
| **Async Runtime** | Tokio |
| **HTTP Server** | Axum 0.8 + Tower HTTP 0.6 |
| **Database** | SQLite (rusqlite 0.31, bundled) |
| **WASM Runtime** | Wasmtime 41 |
| **HTTP Client** | reqwest 0.12 (rustls TLS) |
| **CLI Framework** | clap 4 |
| **Desktop UI** | Tauri 2.0 (Rust-first) |
| **Serialization** | Serde (JSON, TOML, MessagePack, YAML, JSON5) |
| **Crypto** | AES-GCM 0.10, Argon2 0.5, Ed25519-dalek 2, HMAC-SHA2 |
| **Build** | Cargo + Nix flakes |
| **Containers** | Docker multi-stage (rust:1-slim-bookworm) |
| **Cross-compile** | Cross for ARM64 Linux |
| **Client SDKs** | JavaScript/TypeScript, Python |
