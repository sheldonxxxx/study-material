# ZeroClaw Project Topology

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Generated:** 2026-03-26

## Overview

ZeroClaw is a Rust-based AI agent framework and CLI application. The project uses a Cargo workspace structure with multiple crates, a Tauri desktop app, a TypeScript web frontend, embedded firmware, and Python bindings.

## Directory Layout

```
zeroclaw/
├── src/                    # Main Rust application source
│   ├── main.rs            # CLI entry point (zeroclaw binary)
│   ├── lib.rs             # Library root (exports zeroclaw crate)
│   ├── agent/             # Agent loop, personality, context management
│   ├── channels/          # Messaging platform integrations (45 subdirs)
│   ├── providers/         # LLM provider implementations
│   ├── tools/             # Tool implementations (96 subdirs)
│   ├── memory/            # Memory and persistence
│   ├── security/          # Security primitives
│   ├── skills/             # Skill system
│   ├── routines/           # Routine/scripting system
│   └── [other modules]     # auth, config, cron, gateway, hardware, etc.
├── crates/                # Internal Rust crates
│   ├── robot-kit/         # Robot hardware abstraction
│   └── aardvark-sys/      # Aardvark I2C/SPI/GPIO adapter bindings
├── apps/                  # Applications built on zeroclaw
│   └── tauri/             # Tauri desktop app wrapper
├── web/                   # TypeScript web frontend (Vite + React)
│   ├── src/               # React components, pages, hooks
│   └── package.json       # Web dependencies
├── firmware/              # Embedded firmware (Rust)
│   ├── esp32/             # ESP32 firmware
│   ├── esp32-ui/          # ESP32 UI firmware
│   ├── nucleo/            # STM32 Nucleo firmware
│   ├── pico/              # Raspberry Pi Pico firmware
│   └── [other boards]
├── python/                # Python bindings/tools
│   └── zeroclaw_tools/    # Python package
├── tests/                 # Integration and system tests
├── docs/                  # Documentation (79 items)
├── scripts/               # Build and utility scripts
├── tool_descriptions/     # AI tool descriptions
├── dev/                   # Development utilities, CI configs
├── dist/                  # Distribution artifacts
└── benches/               # Benchmarks
```

## Entry Points

### Primary CLI Entry Point
- **`src/main.rs`** (~100KB) - Main binary entry point
  - Defines CLI structure using `clap`
  - Handles subcommands (`onboard`, `agent`, etc.)
  - Initializes logging and tracing

### Library Entry Point
- **`src/lib.rs`** (~19KB) - Core library exports
  - Re-exports major modules
  - Defines public API

### Tauri Desktop App
- **`apps/tauri/src/`** - Tauri application source
  - **`src/main.rs`** - Tauri entry point
  - Desktop wrapper around core zeroclaw functionality

### Web Frontend
- **`web/src/main.tsx`** - React application entry
- **`web/src/App.tsx`** - Main React component
- **`web/index.html`** - HTML shell

### Python Bindings
- **`python/zeroclaw_tools/`** - Python package
  - `pyproject.toml` defines package structure

### Firmware
- Each firmware crate has its own `main.rs`:
  - `firmware/esp32/`
  - `firmware/esp32-ui/`
  - `firmware/nucleo/`
  - etc.

## Workspace Structure

**Root `Cargo.toml`** defines workspace members:
```toml
[workspace]
members = [".", "crates/robot-kit", "crates/aardvark-sys", "apps/tauri"]
```

### Workspace Crates

| Crate | Path | Purpose |
|-------|------|---------|
| `zeroclawlabs` (root) | `./` | Main CLI application + library |
| `robot-kit` | `crates/robot-kit/` | Robot hardware abstraction layer |
| `aardvark-sys` | `crates/aardvark-sys/` | Total Phase Aardvark adapter bindings |
| `tauri` | `apps/tauri/` | Desktop GUI application |

## Key Module Purposes

### `src/` Core Modules

| Module | Files | Purpose |
|--------|-------|---------|
| `agent/` | 17 files | Agent loop, context, evaluation, personality |
| `providers/` | 21 files | LLM provider implementations (OpenAI, Anthropic, Gemini, etc.) |
| `channels/` | 45 files | Messaging integrations (Discord, Slack, Matrix, WhatsApp, etc.) |
| `tools/` | 96 files | Tool implementations (file ops, web fetch, git, etc.) |
| `memory/` | 26 files | Memory system, SQLite persistence |
| `security/` | 25 files | Auth, encryption, trust management |
| `config/` | 6 files | Configuration management |
| `gateway/` | 8 files | HTTP gateway server |

### Provider Implementations (`src/providers/`)
- `openai.rs`, `anthropic.rs`, `bedrock.rs` - Cloud providers
- `ollama.rs`, `lmstudio.rs` - Local providers
- `router.rs`, `reliable.rs` - Routing and reliability layers
- `compatible.rs` - OpenAI-compatible endpoint adapter

### Channel Integrations (`src/channels/`)
- **Chat:** Discord, Slack, Matrix, Telegram, WhatsApp
- **Social:** Twitter, LinkedIn, Reddit, Bluesky
- **Business:** Lark/Feishu, DingTalk, WeCom, Mattermost
- **Communication:** Email, IRC, Signal, Voice calls
- **Other:** Nostr, Notion, JIRA, NextCloud Talk

## Build Configuration

### Binaries Defined
```toml
[[bin]]
name = "zeroclaw"
path = "src/main.rs"
```

### Key Dependencies
- **CLI:** `clap` with derive
- **Async:** `tokio` runtime
- **HTTP:** `reqwest`, `axum` (gateway)
- **Database:** `rusqlite` (bundled SQLite)
- **Serialization:** `serde`, `serde_json`
- **Matrix:** `matrix-sdk` (optional)
- **Web:** `rust-embed` for embedding frontend

### Features
- Default: `observability-prometheus`, `skill-creation`
- Notable: `hardware`, `channel-matrix`, `whatsapp-web`, `voice-wake`, `plugins-wasm`

## Distribution Files

| File | Purpose |
|------|---------|
| `install.sh` | Installation script |
| `setup.bat` | Windows setup |
| `Dockerfile` | Container image |
| `docker-compose.yml` | Local development services |
| `flake.nix` | Nix flake for reproducibility |

## Patterns Observed

1. **Large monolithic src/:** Despite having `crates/`, most code lives in `src/` as modules
2. **Feature-gated integrations:** Most external integrations (WhatsApp, Matrix, etc.) are optional via Cargo features
3. **Tool-heavy architecture:** 96 tool implementations suggest extensive capability surface
4. **Multi-channel messaging:** Strong focus on unified messaging across platforms
5. **Provider abstraction:** Multiple LLM providers with routing/retry layers
6. **Embedded targets:** Firmware for multiple MCU platforms (ESP32, STM32, Pi Pico)
