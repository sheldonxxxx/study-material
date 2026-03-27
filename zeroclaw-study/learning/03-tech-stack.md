# Tech Stack

## Language

**Rust** (edition 2021, rust-version 1.87+)

ZeroClaw is a 100% Rust project optimized for minimal binary size and maximum performance. The release profile uses `-C opt-level=z`, fat LTO, and panic=abort to produce the smallest possible binaries.

## Core Runtime

| Component | Library | Purpose |
|-----------|---------|---------|
| Async runtime | `tokio` 1.50 | Multi-threaded async I/O |
| HTTP server | `axum` 0.8 | Gateway HTTP/1.1 server |
| HTTP client | `reqwest` 0.12 | REST API calls with streaming |
| WebSocket | `tokio-tungstenite` | Real-time messaging channels |

## Data & Serialization

| Library | Version | Purpose |
|---------|---------|---------|
| `serde` | 1.0 | Serialization framework |
| `serde_json` | 1.0 | JSON parsing |
| `prost` | 0.14 | Protocol buffers (Lark, WhatsApp) |
| `rusqlite` | 0.37 | SQLite with bundled mode |
| `chrono` + `chrono-tz` | 0.4 / 0.10 | Time and timezone handling |

## AI/ML Providers

ZeroClaw supports multiple LLM backends via a trait-based pluggable architecture:

- **OpenAI** (GPT-4, GPT-4o)
- **Anthropic** (Claude 3.5, Claude Code)
- **Google Gemini** / **GLM** (Zhipu)
- **Azure OpenAI**
- **Amazon Bedrock**
- **OpenRouter** (meta-router)
- **Ollama** (local models)
- **GitHub Copilot**

## Messaging Channels

Trait-based channels with E2EE support:

- **Discord** - Full bot with history replay
- **Matrix** - E2EE encrypted rooms via `matrix-sdk`
- **Lark/Feishu** - ByteDance suite integration
- **Nostr** - Decentralized social via `nostr-sdk`
- **Email** (IMAP/SMTP) via `async-imap` + `lettre`
- **WhatsApp Web** - Custom `wa-rs` client
- **IRC, Mattermost, DingTalk, MQTT, iMessage** (ACP server)

## CLI & Config

| Library | Purpose |
|---------|---------|
| `clap` 4.5 | CLI argument parsing with derive macros |
| `clap_complete` | Shell completion generation |
| `directories` 6.0 | Platform-specific config/data dirs |
| `toml` 1.0 | Config file parsing |
| `schemars` 1.2 | JSON Schema generation for config export |

## Security

| Library | Purpose |
|---------|---------|
| `chacha20poly1305` | AEAD for secret store encryption |
| `hmac` + `sha2` | Webhook signature verification |
| `ring` | HMAC-SHA256 for JWT auth |
| `rustls` 0.23 | TLS 1.2/1.3 (replaces OpenSSL) |
| `portable-atomic` | Atomic operations for metrics |

## Observability

| Library | Purpose |
|---------|---------|
| `tracing` + `tracing-subscriber` | Structured logging |
| `prometheus` 0.14 | Metrics (64-bit targets only) |
| `opentelemetry` + `opentelemetry-otlp` | Distributed tracing + metrics export |

## Tools & Utilities

| Library | Purpose |
|---------|---------|
| `anyhow` / `thiserror` | Error handling |
| `async-trait` | Async trait methods |
| `futures-util` | Async stream utilities |
| `indicatif` | Progress bars |
| `tempfile` | Temporary file handling |
| `zip` / `tar` / `flate2` | Archive extraction |
| `image` 0.25 | Screenshot processing |
| `qrcode` | Terminal QR rendering |

## Hardware & Peripheral Support

| Library | Platform | Purpose |
|---------|----------|---------|
| `nusb` | Linux/macOS/Windows | USB device enumeration |
| `tokio-serial` | Cross-platform | Serial port communication |
| `probe-rs` | Cross-platform | STM32 memory access |
| `rppal` | Linux only | Raspberry Pi GPIO |
| `landlock` | Linux only | Linux sandboxing |
| `aardvark-sys` | Cross-platform | Total Phase Aardvark I2C/SPI/GPIO |

## WASM & Plugins

| Library | Purpose |
|---------|---------|
| `extism` 1.20 | WASM plugin runtime |

## Browser Automation

| Library | Purpose |
|---------|---------|
| `fantoccini` 0.22 | Selenium-like browser automation (optional) |

## Release Profiles

```toml
[profile.release]
opt-level = "z"      # Size optimization
lto = "fat"          # Maximum cross-crate optimization
codegen-units = 1    # Serialized for low-memory devices
strip = true         # Remove debug symbols
panic = "abort"      # Smaller binary

[profile.release-fast]
inherits = "release"
codegen-units = 8    # Parallel for faster builds

[profile.ci]
inherits = "release"
lto = "thin"         # Faster than fat LTO
codegen-units = 16   # Full parallelism
```

## Workspace Structure

```
zeroclawlabs (root)
├── zeroclaw           # Main CLI binary
├── crates/
│   ├── robot-kit       # Robotics kit
│   └── aardvark-sys    # Total Phase adapter bindings
└── apps/
    └── tauri           # Desktop GUI app
```

## Key Architectural Patterns

- **Trait-based pluggability**: Every subsystem (providers, channels, tools, memory, observability) is swappable via traits
- **Feature flags**: 20+ optional features for minimal default builds
- **Zero-cost abstractions**: Async/await, trait objects, minimal runtime overhead
- **Memory safety**: 100% Rust, no C/C++ dependencies except bundled SQLite
