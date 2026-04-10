# Moltis Project Overview

## What is Moltis?

Moltis is a **Rust-based personal AI gateway** - a self-hosted platform that connects AI models to various communication channels. It serves as a unified entry point for managing agents, chat sessions, and integrations with external services.

## Purpose

The core purpose is to provide a **private, self-hosted alternative** to commercial AI services. Users run their own gateway that handles:

- **Multi-channel messaging** - Connect Discord, Slack, Telegram, WhatsApp, Microsoft Teams
- **LLM provider integration** - OpenAI, Anthropic, GitHub Copilot, Ollama, and custom providers
- **Agent execution** - Run AI agents with tool access and memory
- **Web UI** - Browser-based interface for configuration and chat

## Architecture

### Rust Monorepo Structure

```
moltis/
├── 52 workspace crates   # Core libraries
├── 3 applications        # CLI, macOS app, iOS app
├── apps/
│   ├── courier/         # APNS push relay
│   ├── ios/             # Swift iOS app
│   └── macos/           # Swift macOS app
├── crates/
│   ├── gateway/         # Core HTTP/WS server
│   ├── cli/            # Main CLI tool
│   ├── auth/           # Password + passkey authentication
│   ├── agents/         # Agent system
│   ├── channels/       # discord, slack, telegram, whatsapp, msteams
│   ├── providers/      # LLM provider integrations
│   ├── memory/         # Vector embeddings for context
│   ├── skills/         # Agent skill definitions
│   ├── tools/          # Tool implementations (sandbox, exec)
│   └── ... (40+ more)
└── website/            # Preact + Tailwind web UI
```

### Tech Stack

| Component | Technology |
|-----------|------------|
| Runtime | Rust (Edition 2024, nightly pinned) |
| HTTP/WS | Axum |
| Database | SQLite via sqlx |
| Auth | webauthn-rs (passkeys), argon2 (passwords) |
| GraphQL | async-graphql |
| Web UI | Preact + Tailwind CSS |

### Entry Points

1. **CLI** (`crates/cli`) - Main command-line interface
2. **Gateway** (`crates/gateway`) - Default server when running `moltis`
3. **macOS App** (`apps/macos`) - Native macOS UI
4. **iOS App** (`apps/ios`) - Native iOS UI

## Who is it For?

Moltis targets:

- **Privacy-conscious users** who want to self-host their AI infrastructure
- **Developers** comfortable with Rust and self-hosted solutions
- **Power users** who need unified AI access across multiple chat platforms
- **Organizations** requiring custom AI workflow integration

## Key Features

| Feature | Description |
|---------|-------------|
| Multi-channel | Single agent connected to Discord, Slack, Telegram, WhatsApp, MS Teams |
| Passkey auth | WebAuthn-based passwordless login |
| WASM sandboxing | Safe tool execution with resource limits |
| Memory system | Vector embeddings for persistent context |
| Skills | Extendable agent capabilities |
| MCP support | Model Context Protocol integration |

## Project Organization

- **52 crates** organized by domain (agents, channels, providers, tools, etc.)
- **Strict separation** of concerns across crates
- **Workspace dependencies** - all versions defined in root `Cargo.toml`
- **Feature-gated optional dependencies** - `default-features = false` pattern
- **Pinned toolchain** - `nightly-2025-11-30` for consistent formatting/linting

## Development Practices

- Conventional commits enforced
- CI gates: fmt, clippy, tests, E2E
- Three testing layers: unit, integration, E2E (Playwright)
- Comprehensive observability with tracing + metrics
