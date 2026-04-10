# openfang Tech Stack Analysis

## Project Overview

**openfang** is an open-source Agent Operating System written in Rust. It provides a daemon-based runtime with CLI and desktop UI, supporting agent execution, memory management, skills/extensions, and multi-channel integrations (Discord, Slack, Email, WhatsApp).

---

## Language Choices

| Language | Role | Files |
|----------|------|-------|
| **Rust** | Core runtime, CLI, API server, agent framework | 14 crates in `crates/` |
| **JavaScript/TypeScript** | SDK for client integrations | `sdk/javascript/` |
| **Python** | SDK for client integrations | `sdk/python/` |
| **Nix** | Declarative package management | `flake.nix` |
| **Shell** | Install scripts | `scripts/` |
| **Dockerfile** | Multi-stage container builds | `Dockerfile` |

---

## Rust Workspace Structure

### Crates (`crates/`)

| Crate | Purpose |
|-------|---------|
| `openfang-types` | Core types, traits, and interfaces |
| `openfang-memory` | SQLite-based memory substrate |
| `openfang-runtime` | Agent execution environment with WASM support |
| `openfang-wire` | OpenFang Protocol (OFP) — agent-to-agent networking |
| `openfang-api` | HTTP/WebSocket API server (Axum) |
| `openfang-kernel` | Core kernel orchestrating all subsystems |
| `openfang-cli` | CLI binary entry point (`openfang` command) |
| `openfang-channels` | Pluggable messaging integrations (Discord, Slack, Email, etc.) |
| `openfang-skills` | Skill registry, loader, marketplace, OpenClaw compatibility |
| `openfang-hands` | Curated autonomous capability packages |
| `openfang-extensions` | Extension system: MCP server setup, credential vault, OAuth2 PKCE |
| `openfang-migrate` | Migration engine for importing from other agent frameworks |
| `openfang-desktop` | Tauri 2.0 native desktop application |
| `xtask` | Build automation tasks |

### Workspace Configuration

```toml
[workspace]
resolver = "2"
members = [
    "crates/openfang-types",
    "crates/openfang-memory",
    "crates/openfang-runtime",
    "crates/openfang-wire",
    "crates/openfang-api",
    "crates/openfang-kernel",
    "crates/openfang-cli",
    "crates/openfang-channels",
    "crates/openfang-migrate",
    "crates/openfang-skills",
    "crates/openfang-desktop",
    "crates/openfang-hands",
    "crates/openfang-extensions",
    "xtask",
]
```

- **MSRV**: Rust 1.75
- **Edition**: 2021
- **Toolchain**: Stable (per `rust-toolchain.toml`)

---

## Key Dependencies

### Async Runtime
- **Tokio** — Full-featured async runtime with `full` feature set
- **Tokio Stream** — Stream utilities for async iteration

### HTTP Server
- **Axum 0.8** — Modular web framework with WebSocket support
- **Tower HTTP 0.6** — Middleware (CORS, tracing, compression)

### Serialization
- **Serde** + **Serde JSON** — JSON serialization
- **TOML** — Config file parsing
- **rmp-serde** — MessagePack for wire protocol

### Database
- **rusqlite 0.31** (bundled) — SQLite with bundled SQLite, JSON support

### Security / Crypto
- **AES-GCM 0.10** — Authenticated encryption
- **Argon2 0.5** — Password hashing
- **Ed25519-dalek 2** — Digital signatures
- **HMAC-SHA2** — Message authentication
- **SHA2 / SHA1** — Hashing
- **subtle 2** — Constant-time operations

### WASM
- **Wasmtime 41** — WASM sandbox runtime for skill isolation

### HTTP Client
- **reqwest 0.12** — HTTP client with JSON, streaming, multipart, TLS (rustls)

### WebSocket
- **tokio-tungstenite 0.24** — WebSocket client for Discord/Slack gateways

### CLI
- **clap 4** — CLI argument parsing
- **ratatui 0.29** — Terminal UI library
- **colored 3** — Terminal colors

### Email
- **lettre 0.11** — SMTP client
- **imap 2** — IMAP client
- **mailparse 0.16** — MIME parsing

### Observability
- **tracing 0.1** — Structured logging and diagnostics
- **tracing-subscriber 0.3** — Log subscriber with JSON/env-filter

---

## Build System

### Cargo Profiles

| Profile | LTO | Codegen Units | Strip | Opt Level |
|---------|-----|---------------|-------|-----------|
| `release` | true (fat) | 1 | true | 3 |
| `release-fast` | thin | 8 | false | 2 |

### Nix Flake

Uses `flake-parts` with `juspay/rust-flake` for declarative Rust builds:
- Supports: `x86_64-linux`, `aarch64-linux`, `aarch64-darwin`, `x86_64-darwin`
- `openfang-desktop` requires GTK3/WebKit dependencies on Linux

### Cross-Compilation

- **Cross** tool for ARM64 Linux cross-compilation
- `Cross.toml` adds `libssl-dev:$CROSS_DEB_ARCH` for `aarch64-unknown-linux-gnu`

### Docker

Multi-stage build:
1. **Builder stage**: `rust:1-slim-bookworm` with LTO/codeGen tuning
2. **Runtime stage**: `rust:1-slim-bookworm` with Python3, Node.js, npm

---

## CI/CD

### GitHub Actions (`.github/workflows/`)

#### CI (`ci.yml`)
- **Check**: `cargo check --workspace` on Ubuntu, macOS, Windows
- **Test**: `cargo test --workspace` on all 3 platforms
- **Clippy**: Linting with `-D warnings` on Ubuntu
- **Format**: `cargo fmt --check` on Ubuntu
- **Audit**: `cargo audit` for security vulnerabilities
- **Secrets Scan**: TruffleHog filesystem scan
- **Install Smoke**: Shell and PowerShell installer syntax check

#### Release (`release.yml`)
- **Desktop**: Tauri 2.0 builds for:
  - Linux x86_64 (AppImage, .deb)
  - macOS x86_64 and ARM64 (.dmg, .app with code signing + notarization)
  - Windows x86_64 and ARM64 (.msi, .exe)
- **CLI**: 6 targets via `cargo build --release`
  - `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`
  - `x86_64-apple-darwin`, `aarch64-apple-darwin`
  - `x86_64-pc-windows-msvc`, `aarch64-pc-windows-msvc`
- **Docker**: Multi-arch images (`linux/amd64`, `linux/arm64`) pushed to GHCR

---

## Additional Systems

### Agents (`agents/`)
30+ pre-built agent configurations as `.toml` files:
- `analyst`, `architect`, `assistant`, `code-reviewer`, `coder`
- `customer-support`, `data-scientist`, `debugger`, `devops-lead`
- `doc-writer`, `email-assistant`, `health-tracker`, `hello-world`
- `home-automation`, `legal-assistant`, `meeting-assistant`, `ops`
- `orchestrator`, `personal-finance`, `planner`, `recruiter`
- `researcher`, `sales-assistant`, `security-auditor`, `social-media`
- `test-engineer`, `translator`, `travel-planner`, `tutor`, `writer`

### SDKs

| SDK | Path | Language | Entry Point |
|-----|------|----------|-------------|
| JavaScript/TypeScript | `sdk/javascript/` | JS/TS | `index.js`, `index.d.ts` |
| Python | `sdk/python/` | Python | `openfang_sdk.py`, `openfang_client.py` |

### Packages (`packages/`)

| Package | Description |
|---------|-------------|
| `whatsapp-gateway` | WhatsApp integration via `index.js` |

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
│ openfang-kernel │◄───│  openfang-runtime    │───►│ openfang-channels│
│   (orchestrator)│    │ (WASM skill sandbox)│    │ (Discord/Slack)  │
└────────┬────────┘    └─────────────────────┘    └──────────────────┘
         │
  ┌──────┴──────┬──────────────┬──────────────┐
  │             │              │              │
┌─▼────┐  ┌─────▼────┐  ┌─────▼─────┐  ┌─────▼─────┐
│memory│  │  types   │  │   wire    │  │ extensions│
│(SQLite)│ │         │  │   (OFP)   │  │  (MCP)    │
└───────┘  └─────────┘  └───────────┘  └───────────┘
```

---

## Summary

| Dimension | Choice |
|-----------|--------|
| **Core Language** | Rust (1.75+, stable) |
| **Async Runtime** | Tokio |
| **HTTP Server** | Axum + Tower HTTP |
| **Database** | SQLite (rusqlite, bundled) |
| **WASM Runtime** | Wasmtime 41 |
| **HTTP Client** | reqwest (rustls TLS) |
| **CLI Framework** | clap 4 |
| **Desktop UI** | Tauri 2.0 (Rust-first) |
| **Serialization** | Serde (JSON, TOML, MessagePack) |
| **Crypto** | AES-GCM, Argon2, Ed25519, HMAC-SHA2 |
| **Build** | Cargo + Nix flakes |
| **Containers** | Docker multi-stage (rust:1-slim-bookworm) |
| **CI/CD** | GitHub Actions |
| **Client SDKs** | JavaScript/TypeScript, Python |
| **Cross-compile** | Cross for ARM64 Linux |
| **Nix support** | flakes with flake-parts |
