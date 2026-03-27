# Moltis Repository Topology

## Repository Type

**Rust Monorepo** with Cargo workspace (resolver 2)

- **Root**: `Cargo.toml` defines workspace members
- **52 workspace crates** in `crates/` directory
- **3 applications** in `apps/` directory
- **Edition**: Rust 2024, Minimum version: 1.91

## Directory Structure

```
moltis/
├── Cargo.toml              # Workspace definition
├── CLAUDE.md               # Project guidelines (16957 lines)
├── apps/                   # Application entry points
│   ├── courier/            # Rust binary (main.rs)
│   ├── ios/                # Swift iOS app
│   └── macos/              # Swift macOS app
├── crates/                 # 52 workspace library crates
│   ├── cli/                 # Main CLI tool (moltis binary)
│   ├── gateway/             # Core server/gateway
│   ├── agents/              # Agent system
│   ├── auth/                # Authentication (password + passkey)
│   ├── web/                 # Web UI server (Axum)
│   ├── providers/           # LLM provider integrations
│   ├── channels/            # Channel integrations
│   │   ├── discord/
│   │   ├── slack/
│   │   ├── telegram/
│   │   ├── whatsapp/
│   │   └── msteams/
│   ├── memory/              # Memory/embedding system
│   ├── mcp/                 # Model Context Protocol
│   ├── skills/              # Agent skills
│   ├── graph/               # GraphQL API
│   └── ... (40+ more)
├── website/                 # Web UI assets (Preact + Tailwind)
├── docs/                    # mdBook documentation
├── examples/                # Example code
├── scripts/                 # Build/deployment scripts
├── wit/                     # WebAssembly Interface Type definitions
├── plans/                   # GSD planning documents
├── prompts/                 # Session prompts
└── .beads/                  # Issue tracking (Dolt database)
```

## Entry Points

### Primary Entry Points

| Entry Point | Path | Type | Purpose |
|-------------|------|------|---------|
| **CLI** | `crates/cli/src/main.rs` | Rust binary | Main CLI tool `moltis` |
| **Gateway** | `crates/gateway/` | Rust lib + binary | Core server (default when running `moltis` without subcommand) |
| **Courier** | `apps/courier/src/main.rs` | Rust binary | Secondary application |
| **macOS App** | `apps/macos/` | Swift app | Native macOS UI |
| **iOS App** | `apps/ios/` | Swift app | Native iOS UI |

### CLI Entry Point (`crates/cli/src/main.rs`)

- Uses `clap` for CLI parsing
- Commands: `gateway`, `agent`, `auth`, `config`, `channel`, `db`, `doctor`, `hooks`, `import`, `memory`, `node`, `sandbox`, `service`, `tailscale`
- Default command: `Gateway` (starts the server)
- Global flags: `--log-level`, `--json-logs`, `--bind`, `--port`, `--config-dir`, `--data-dir`, `--share-dir`

### Gateway Entry Point

The gateway is the core server that:
- Serves the web UI (`crates/web`)
- Handles authentication (`crates/auth`)
- Manages agents (`crates/agents`)
- Routes messages through channels (`crates/channels`)
- Provides GraphQL API (`crates/graphql`)

## Key Crate Architecture

### Core Infrastructure
| Crate | Purpose |
|-------|---------|
| `crates/gateway` | HTTP/WS server, auth middleware, session management |
| `crates/cli` | CLI interface |
| `crates/common` | Shared types/utilities |
| `crates/config` | Configuration schema and validation |
| `crates/protocol` | Message protocol definitions |
| `crates/routing` | Message routing logic |

### Agent System
| Crate | Purpose |
|-------|---------|
| `crates/agents` | Agent implementation |
| `crates/skills` | Agent skill definitions |
| `crates/tools` | Tool implementations |
| `crates/mcp` | MCP protocol support |
| `crates/providers` | LLM provider integrations (OpenAI, GitHub Copilot, Kimi, etc.) |

### Channels (Messaging Platforms)
| Crate | Platform |
|-------|----------|
| `crates/discord` | Discord |
| `crates/slack` | Slack |
| `crates/telegram` | Telegram |
| `crates/whatsapp` | WhatsApp |
| `crates/msteams` | Microsoft Teams |
| `crates/channels` | Channel trait definitions |

### Data & Storage
| Crate | Purpose |
|-------|---------|
| `crates/memory` | Vector memory/embeddings |
| `crates/sessions` | Session storage |
| `crates/projects` | Project data |
| `crates/vault` | Secrets storage |
| `crates/cron` | Scheduled jobs |

### Platform Integration
| Crate | Purpose |
|-------|---------|
| `crates/swift-bridge` | Swift/Rust interop |
| `crates/browser` | Browser automation |
| `crates/voice` | Voice processing |
| `crates/tailscale` | Tailscale integration |
| `crates/tls` | TLS certificate generation |

## Workspace Dependencies

Key dependencies (from `Cargo.toml`):
- **Async runtime**: `tokio` (full features)
- **HTTP/WS server**: `axum` (with WebSocket support)
- **Database**: `sqlx` (SQLite)
- **Serialization**: `serde`, `serde_json`
- **Authentication**: `webauthn-rs` (passkeys), `argon2` (passwords)
- **LLM providers**: `async-openai`, `genai`
- **GraphQL**: `async-graphql`, `async-graphql-axum`

## Build Configuration

- **Rust toolchain**: `rust-toolchain.toml` (pinned nightly)
- **Formatter**: `rustfmt.toml` (nightly, pinned)
- **Linter**: `clippy.toml`
- **TOML formatter**: `taplo`
- **JS linter**: `biome.json`

## Notable Patterns

1. **Feature-gated optional dependencies**: Many crates have `default-features = false`
2. **Workspace dependencies**: All crate versions defined in root `Cargo.toml`
3. **SSSS pattern**: Strict separation of concerns across crates
4. **WIT for WASM**: WebAssembly Interface Type definitions in `wit/`
5. **BD issue tracking**: Dolt-powered issue tracking in `.beads/`
