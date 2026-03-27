# Moltis Tech Stack Analysis

## Project Overview

**Moltis** is a Rust-based personal AI gateway (inspired by OpenClaw). It is a monorepo workspace with 50+ crates, shipping as both a CLI binary (`moltis`) and a gateway web service. It supports numerous chat channels (Discord, Slack, Telegram, WhatsApp, MS Teams, etc.), LLM providers (OpenAI, GitHub Copilot, local Ollama/llama.cpp), and a plugin/sandbox system for tool execution.

## Rust Toolchain

| Field | Value |
|-------|-------|
| Channel | `nightly-2025-11-30` (pinned) |
| Edition | 2024 |
| Minimum Rust | 1.91 |
| WASM target | `wasm32-wasip2` (for WASM guest tools) |
| Components | clippy, rustfmt |
| Resolver | version 2 (workspace) |

## Workspace Structure

The Cargo workspace contains **52 members**:

**Apps**
- `apps/courier` — APNS push notification relay service
- `apps/courier/Cargo.toml`

**Core Crates (alphabetical)**
- `crates/agents` — Agent orchestration
- `crates/auth` — Password + passkey (WebAuthn) authentication
- `crates/auto-reply` — Auto-reply rules engine
- `crates/benchmarks` — Performance benchmarks
- `crates/browser` — Browser automation / headless Chrome
- `crates/caldav` — CalDAV calendar integration
- `crates/canvas` — Canvas LMS integration
- `crates/channels` — Unified channel abstraction
- `crates/chat` — Chat session management
- `crates/cli` — Main CLI binary (`moltis`)
- `crates/common` — Shared utilities
- `crates/config` — Configuration schema and validation
- `crates/cron` — Cron job scheduler
- `crates/discord` — Discord bot channel
- `crates/gateway` — HTTP gateway server (the main runtime)
- `crates/graphql` — GraphQL API layer
- `crates/httpd` — HTTP server (Axum-based)
- `crates/mcp` — Model Context Protocol integration
- `crates/media` — Media processing
- `crates/memory` — Vector memory / embeddings
- `crates/metrics` — Prometheus metrics
- `crates/msteams` — Microsoft Teams channel
- `crates/network-filter` — SSRF protection / trusted networks
- `crates/node-host` — Node.js MCP server host
- `crates/oauth` — OAuth 2.0 flows
- `crates/onboarding` — First-run onboarding
- `crates/openclaw-import` — Import from OpenClaw
- `crates/plugins` — Plugin system
- `crates/projects` — Project management
- `crates/protocol` — Core messaging protocol
- `crates/provider-setup` — LLM provider configuration
- `crates/providers` — LLM provider implementations (OpenAI, Copilot, Ollama, etc.)
- `crates/qmd` — Quoted Markdown format handling
- `crates/routing` — Message routing
- `crates/schema-export` — GraphQL schema exporter (for iOS)
- `crates/service-traits` — Shared service trait definitions
- `crates/sessions` — Session persistence
- `crates/skills` — Skills / capability registry
- `crates/slack` — Slack channel
- `crates/swift-bridge` — Rust FFI for Swift macOS app
- `crates/tailscale` — Tailscale integration
- `crates/telegram` — Telegram channel
- `crates/tls` — TLS certificate generation
- `crates/tools` — Tool execution and sandboxing
- `crates/vault` — Secrets vault
- `crates/voice` — Voice / audio processing
- `crates/wasm-precompile` — Pre-compiles WASM to AOT `.cwasm`
- `crates/wasm-tools/calc` — WASM guest: calculator
- `crates/wasm-tools/web-fetch` — WASM guest: HTTP fetch
- `crates/wasm-tools/web-search` — WASM guest: web search
- `crates/web` — Web UI server (Axum + Askama templates)
- `crates/whatsapp` — WhatsApp channel

## Key Dependencies

### Async Runtime
- **tokio** (full features) — async runtime

### HTTP / Web
- **axum** (0.8, with `ws`) — HTTP server + WebSocket support
- **axum-extra** (0.10, cookie support) — axum extensions
- **hyper** (1.x) — low-level HTTP
- **hyper-util** — hyper utilities
- **tower** / **tower-http** — middleware
- **reqwest** (0.12) — HTTP client
- **tokio-tungstenite** (0.26) — WebSocket client
- **askama** (0.15) — template engine (for server-rendered HTML)

### Serialization
- **serde** / **serde_json** — JSON serialization
- **serde_yaml** — YAML config
- **postcard** (alloc) — binary serialization for WASM
- **toml** / **toml_edit** — TOML config

### Database
- **sqlx** (0.8, sqlite) — async SQLite with migrations

### LLM / AI
- **async-openai** (0.32) — OpenAI API client
- **genai** (0.5) — generic AI provider abstraction
- **llama-cpp-2** (0.1) — llama.cpp bindings
- **slack-morphism** (2.6) — Slack API
- **teloxide** (0.13) — Telegram bot framework
- **serenity** (0.12, rustls) — Discord bot
- **wacore** / **waproto** / **whatsapp-rust** — WhatsApp protocol

### GraphQL
- **async-graphql** (7) — GraphQL server
- **async-graphql-axum** (7) — Axum integration

### WASM
- **wasmtime** (36, component-model) — WASM runtime
- **wit-bindgen** (0.41) — WIT interface bindings

### Security / Crypto
- **argon2** — password hashing
- **chacha20poly1305** — symmetric encryption
- **p256** (0.13) — ECDH
- **webauthn-rs** (0.5) — passkey / WebAuthn
- **secrecy** (0.8) — secret-handling wrapper
- **rustls** (0.23, ring) — TLS
- **openssl** (0.10, vendored) — OpenSSL fallback

### Observability
- **tracing** / **tracing-subscriber** (env-filter, json) — structured logging + OpenTelemetry
- **opentelemetry** / **opentelemetry-otlp** / **opentelemetry_sdk** — OTLP export
- **tracing-opentelemetry** — tracing <-> OTLP bridge
- **metrics** / **metrics-exporter-prometheus** / **metrics-tracing-context** — metrics

### Code Analysis / Parsing
- **tree-sitter** (multiple languages: bash, c, cpp, css, go, html, java, javascript, json, md, python, ruby, rust, toml, typescript)
- **text-splitter** (0.29) — code chunking for embeddings
- **gix** (0.78) — Git operations

### Other Notable
- **chrono** / **chrono-tz** — date/time
- **uuid** (v4) — UUID generation
- **directories** (6) — platform directories (config/data)
- **qrcode** (0.14) — QR code rendering
- **image** (jpeg, png, webp) — image processing
- **pulldown-cmark** — Markdown parsing
- **html2text** — HTML to plain text
- **icalendar** (0.17) — iCalendar parsing
- **libdav** (0.10) — CalDAV client
- **sled** (0.34) — embedded KV store
- **tikv-jemallocator** (0.6) — jemalloc for reduced memory fragmentation
- **wasmtime-wasi** (36) — WASI support
- **wit-bindgen-rt** — WIT runtime support
- **clap** (4, derive + env) — CLI argument parsing

## Build System

### Task Runner: `just`
The `justfile` provides ~50 recipes covering:
- `build` / `build-release` — standard Rust builds
- `build-css` — Tailwind CSS compilation for web UI
- `build-wasm-artifacts` — WASM guest tools build + precompilation
- `format` / `format-check` — nightly rustfmt (pinned to `nightly-2025-11-30`)
- `lint` — clippy with OS-aware CUDA/metal feature gates
- `test` — cargo nextest (OS-aware, macOS splits out metal features)
- `ci` — full CI pipeline: fmt + lint + i18n-check + build-css + build + test
- `build-test` — parallel Rust tests + E2E tests
- `ui-e2e` / `ui-e2e-headed` — Playwright E2E tests
- `deb` / `deb-amd64` / `deb-arm64` — Debian packages
- `arch-pkg*` — Arch Linux packages
- `rpm*` — RPM packages
- `appimage` — AppImage
- `swift-*` — Swift macOS/iOS app build
- `ios-*` — iOS app build
- `courier-*` — APNS relay build/deploy
- `release-workflow*` — GitHub Actions release dispatch
- `changelog*` — git-cliff changelog generation
- `ship` — automated PR workflow

### Package Manager (Web UI)
- **npm** (Node.js 22 LTS in Docker) — web UI JS dependencies
- **Tailwind CSS v4** — styling (built via `tailwindcss` CLI)
- **Playwright** — E2E testing
- **Biome** — JS/TS linting and formatting (`biome.json` config present)

### Docker
- Multi-stage Dockerfile (`debian:bookworm-slim` runtime)
- Installs: Chromium, Node.js 22, Docker CLI, `sudo`
- Exposes ports: 13131 (HTTPS gateway), 13132 (HTTP for CA cert), 1455 (OAuth callback)
- Non-root user `moltis` with Docker socket access
- Volume mounts: config, data, npm cache, Docker socket

### CI/CD
GitHub Actions workflows:
- `ci.yml` — fmt + lint + test
- `e2e.yml` — end-to-end web UI tests
- `release.yml` — release packaging (deb, rpm, arch, appimage)
- `docs.yml` — mdBook documentation deploy
- `codspeed.yml` — benchmarking
- `homebrew.yml` — Homebrew tap publish

### Code Quality Gates
- **rustfmt** — pinned nightly, enforced in CI
- **clippy** — deny `expect_used`, `unwrap_used`; deny all warnings
- **taplo** — TOML formatting
- **Biome** — JS formatting/linting
- **Conventional commits** — enforced via commit message hooks
- **Local validation script** (`./scripts/local-validate.sh`) — must pass before PR merge

### Packaging
- **Debian** (.deb) via `cargo deb`
- **RPM** via `cargo generate-rpm`
- **Arch Linux** (.pkg.tar.zst) via custom `fakeroot tar`
- **AppImage** via `appimagetool`
- **macOS** via Swift/Xcode (`swift-build`, `ios-build`)
- **Flatpak** via flatpak-builder
- **Snap** via snapcraft

## Project Type Summary

| Aspect | Value |
|--------|-------|
| **Project type** | AI gateway / personal LLM hub |
| **Architecture** | Monorepo (52 Rust crates + web UI) |
| **Primary binary** | `moltis` CLI → gateway HTTP server |
| **Secondary app** | `moltis-courier` (APNS relay) |
| **macOS app** | Swift app in `apps/macos/` (via `crates/swift-bridge`) |
| **iOS app** | Swift app in `apps/ios/` (GraphQL client) |
| **Web UI** | Preact + Tailwind CSS served from Rust (Axum + Askama) |
| **WASM** | WASI guest tools (calc, web-fetch, web-search) pre-compiled to `.cwasm` |
| **Database** | SQLite via sqlx (workspace-local, per-crate migrations) |
| **Config format** | TOML (`moltis.toml`) |
| **Secrets** | `~/.moltis/credentials.json` via `KeyStore` |
| **Channels** | Discord, Slack, Telegram, WhatsApp, MS Teams, CalDAV, email, Canvas LMS, etc. |
| **LLM providers** | OpenAI, GitHub Copilot, Ollama, llama.cpp, OpenRouter, Kimi Code, and extensible |
| **Sandboxing** | Docker containers (via mounted Docker socket) |
| **Release model** | Date-based versioning (`YYYYMMDD.NN`), Cargo.toml stays at `0.1.0` |
| **Version injection** | Via `MOLTIS_VERSION` env var at build time |
