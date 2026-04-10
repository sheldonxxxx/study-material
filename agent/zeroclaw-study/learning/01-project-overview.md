# ZeroClaw Project Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Study Path:** `/Users/sheldon/Documents/claw/zeroclaw-study/`
**Generated:** 2026-03-26

## Project Purpose

ZeroClaw is a Rust-based AI agent framework and CLI application designed to orchestrate large language model (LLM) interactions across multiple providers and messaging channels. It provides a unified interface for building AI-powered agents that can interact through various chat platforms, execute tools, maintain memory, and operate on hardware devices.

## Core Capabilities

### AI Agent Framework
- **Multi-Provider LLM Support**: OpenAI, Anthropic, Google Gemini, AWS Bedrock, plus local providers (Ollama, LM Studio)
- **Provider Routing**: Built-in load balancing, fallback chains, and OpenAI-compatible endpoint adapter
- **Tool System**: 96 tool implementations covering file operations, web fetching, git, and more
- **Memory System**: SQLite-backed persistence with vector similarity search, FTS5 keyword search, and a two-tier response cache

### Messaging Channels
ZeroClaw integrates with 45 messaging platforms across multiple categories:

| Category | Platforms |
|----------|-----------|
| Chat | Discord, Slack, Matrix, Telegram, WhatsApp |
| Social | Twitter, LinkedIn, Reddit, Bluesky |
| Business | Lark/Feishu, DingTalk, WeCom, Mattermost |
| Communication | Email, IRC, Signal, Voice calls |
| Other | Nostr, Notion, JIRA, NextCloud Talk |

### Hardware and Robotics
- Robot hardware abstraction layer (`robot-kit`)
- Embedded firmware for ESP32, STM32 Nucleo, Raspberry Pi Pico, and other MCUs
- Aardvark I2C/SPI/GPIO adapter bindings via `aardvark-sys`

### Desktop and Web
- Tauri desktop application wrapper
- TypeScript web frontend (Vite + React)

## Architecture

### Workspace Structure

```
zeroclaw/
├── src/                    # Main Rust application (core modules)
├── crates/                 # Internal Rust crates
│   ├── robot-kit/         # Robot hardware abstraction
│   └── aardvark-sys/      # Aardvark adapter bindings
├── apps/tauri/            # Tauri desktop app
├── web/                   # TypeScript React frontend
├── firmware/              # Embedded firmware (ESP32, STM32, Pico)
├── python/                # Python bindings
├── tests/                 # Integration and system tests
└── docs/                  # Documentation (79 items)
```

### Key Modules

| Module | Files | Purpose |
|--------|-------|---------|
| `agent/` | 17 | Agent loop, context, evaluation, personality |
| `providers/` | 21 | LLM provider implementations |
| `channels/` | 45 | Messaging platform integrations |
| `tools/` | 96 | Tool implementations |
| `memory/` | 26 | Memory system, SQLite persistence |
| `security/` | 25 | Auth, encryption, trust management |

### Entry Points

- **CLI**: `src/main.rs` (~100KB) - Defines subcommands (`onboard`, `agent`, etc.)
- **Library**: `src/lib.rs` (~19KB) - Core library exports
- **Tauri**: `apps/tauri/src/` - Desktop GUI wrapper
- **Web**: `web/src/main.tsx` - React application entry

## Technical Stack

### Core Technologies
- **Language**: Rust (workspace with 4 crates)
- **Async Runtime**: Tokio (full feature set)
- **HTTP**: Reqwest, Axum (gateway)
- **Database**: SQLite via Rusqlite (bundled)
- **Serialization**: Serde, Serde JSON
- **CLI**: Clap with derive
- **TLS**: Rustls (no OpenSSL)

### Feature Flags
Notable optional features include:
- `observability-prometheus` - Prometheus metrics
- `hardware` - Hardware support
- `channel-matrix` - Matrix protocol
- `whatsapp-web` - WhatsApp integration
- `voice-wake` - Voice wake word
- `plugins-wasm` - WASM plugin support

## Distribution

### Installation Methods
- `install.sh` - Unix/Linux installation script
- `setup.bat` - Windows setup
- `Dockerfile` - Container image
- `docker-compose.yml` - Local development services
- `flake.nix` - Nix flake for reproducibility

## Patterns and Design Decisions

1. **Large monolithic `src/`**: Despite `crates/` workspace, most code lives in `src/` as modules
2. **Feature-gated integrations**: External integrations are optional via Cargo features
3. **Tool-heavy architecture**: 96 tools suggest extensive capability surface
4. **Multi-channel messaging**: Unified messaging across platforms
5. **Provider abstraction**: Multiple LLM providers with routing/retry layers
6. **Embedded targets**: Firmware for multiple MCU platforms

## Security Architecture

ZeroClaw implements defense-in-depth security:

- **Secrets**: ChaCha20-Poly1305 AEAD encryption with OS-level file permissions
- **Authentication**: One-time pairing codes, WebAuthn/FIDO2 hardware keys
- **Authorization**: Command allowlists, path confinement, risk tiers
- **Leak Detection**: Credential exfiltration prevention via content scanning
- **Prompt Injection Defense**: Multi-pattern detection with scoring
- **Sandboxing**: Landlock, Bubblewrap, Firejail, Seatbelt (platform-dependent)
- **Audit Logging**: Merkle hash chain with optional HMAC signing

See [08-security.md](./08-security.md) for detailed security analysis.

## Code Quality

The project maintains:
- `#![forbid(unsafe_code)]` workspace-wide (except FFI bindings)
- Comprehensive configuration schema validation
- Multi-backend observability (log, Prometheus, OpenTelemetry)
- Component and integration test separation

See [05-code-quality.md](./05-code-quality.md) for detailed code quality analysis.
