# OpenFang Project Topology

## Project Type

**Rust Monorepo / Agent Operating System**

OpenFang is an open-source AI agent framework written in Rust, featuring a workspace of 14 crates plus SDKs in JavaScript and Python. It functions as an "Agent OS" — a runtime environment for deploying and orchestrating AI agents with various communication channels (Discord, Slack, WhatsApp, etc.).

---

## Directory Layout

```
openfang/
├── Cargo.toml           # Workspace root (14 crates + xtask)
├── CLAUDE.md            # Project instructions for Claude
├── Cargo.lock
├── rust-toolchain.toml
├── Dockerfile
├── docker-compose.yml
├── flake.nix            # Nix flake for reproducible builds
│
├── crates/              # Rust workspace members (14 crates)
│   ├── openfang-cli/         # Main CLI binary entry point
│   ├── openfang-api/         # HTTP API server (axum)
│   ├── openfang-kernel/      # Core agent orchestration kernel
│   ├── openfang-runtime/     # Agent execution runtime
│   ├── openfang-types/       # Shared type definitions
│   ├── openfang-wire/        # Serialization (MessagePack)
│   ├── openfang-memory/      # Persistent memory layer
│   ├── openfang-channels/    # Channel adapters (Discord, Slack, etc.)
│   ├── openfang-skills/      # Skill system
│   ├── openfang-extensions/  # Extension framework
│   ├── openfang-desktop/     # Desktop UI components
│   ├── openfang-hands/       # Tool execution
│   └── openfang-migrate/     # Database migrations
│
├── agents/              # Agent configurations (25 agents)
│   ├── assistant/             # General purpose assistant
│   ├── coder/                 # Code generation agent
│   ├── planner/               # Planning agent
│   ├── orchestrator/          # Orchestration agent
│   ├── researcher/             # Research agent
│   ├── analyst/                # Data analysis agent
│   ├── architect/              # Software architecture agent
│   ├── debugger/               # Debugging agent
│   ├── code-reviewer/          # Code review agent
│   ├── test-engineer/          # Test generation agent
│   ├── doc-writer/             # Documentation agent
│   ├── translator/             # Translation agent
│   ├── tutor/                  # Teaching agent
│   ├── email-assistant/        # Email handling
│   ├── meeting-assistant/      # Meeting management
│   ├── recruiter/              # Recruitment tasks
│   ├── legal-assistant/        # Legal document assistance
│   ├── health-tracker/        # Health monitoring
│   ├── personal-finance/      # Finance assistant
│   ├── travel-planner/         # Travel planning
│   ├── sales-assistant/        # Sales support
│   ├── customer-support/       # Support agent
│   ├── social-media/           # Social media management
│   ├── home-automation/        # Smart home control
│   ├── devops-lead/            # DevOps automation
│   ├── data-scientist/          # ML/data work
│   ├── ops/                    # Operations agent
│   ├── security-auditor/       # Security analysis
│   └── hello-world/            # Example/template agent
│
├── sdk/                 # Language SDKs
│   ├── javascript/           # JS/TS SDK with examples
│   └── python/               # Python SDK (empty)
│
├── packages/            # Additional packages
│   └── whatsapp-gateway/     # WhatsApp integration
│
├── xtask/               # Build helper tasks
│
├── docs/                # Documentation (17 markdown files)
│   ├── architecture.md       # System architecture
│   ├── api-reference.md      # API documentation
│   ├── cli-reference.md       # CLI commands
│   ├── configuration.md       # Config guide
│   ├── channel-adapters.md   # Channel integrations
│   ├── providers.md           # LLM providers
│   ├── skill-development.md  # Skill creation
│   ├── workflows.md           # Workflow patterns
│   └── ...                   # Various other docs
│
├── deploy/              # Deployment configurations
├── scripts/             # Utility scripts
├── public/              # Static assets
└── .github/             # GitHub workflows
```

---

## Key Entry Points

### Primary Binary
- **`crates/openfang-cli/src/main.rs`** — Main CLI entry point (239KB, very large). Contains the CLI launcher and TUI.

### Library Crates (Internal APIs)
| Crate | Purpose | Key Files |
|-------|---------|-----------|
| `openfang-kernel` | Core orchestration | Agent lifecycle, message routing |
| `openfang-runtime` | Agent execution | Runs agent loops |
| `openfang-api` | HTTP API server | `server.rs` (34KB), `routes.rs` (431KB), `ws.rs` (54KB) |
| `openfang-channels` | External integrations | Discord, Slack, WhatsApp adapters |
| `openfang-types` | Shared types | `Message`, `Agent`, `Task` definitions |
| `openfang-wire` | Serialization | MessagePack encoding |
| `openfang-memory` | Persistence | SQLite-backed storage |
| `openfang-skills` | Skill system | Extensible tool framework |

### SDK Entry Points
- **`sdk/javascript/index.js`** — JavaScript SDK entry (14KB)
- **`sdk/javascript/index.d.ts`** — TypeScript definitions

### Agent Entry Points
- Each agent in `/agents/{name}/agent.toml` — Agent configuration file defining the agent's personality, instructions, and capabilities.

---

## Workspace Members (from Cargo.toml)

```toml
[workspace]
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

---

## Build & Run

- **Build CLI:** `cargo build --release -p openfang-cli`
- **Run daemon:** `target/release/openfang.exe start`
- **Default API URL:** `http://127.0.0.1:4200`
- **Config location:** `~/.openfang/config.toml`

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Rust 1.75+ |
| HTTP Server | Axum 0.8 |
| Serialization | MessagePack (rmp-serde), JSON |
| Database | SQLite (rusqlite) |
| Async Runtime | Tokio |
| WebSocket | tokio-tungstenite |
| CLI UI | Ratatui (TUI) |
| LLM Drivers | HTTP (reqwest) — OpenAI compatible |
| WASM | wasmtime |
| Security | AES-GCM, Argon2, Ed25519 |
| Email | lettre (SMTP), imap |
| TLS | rustls, native-tls, openssl (vendored) |
