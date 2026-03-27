# OpenViking Project Topology

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Project Type:** Hybrid Python + Rust (Python-primary with Rust CLI)
**Description:** An Agent-native context database for AI Agents

---

## Directory Structure

```
OpenViking/
|-- Root Configuration Files
|   |-- pyproject.toml          # Python package config (setuptools)
|   |-- setup.py                # Legacy setup.py (setuptools build)
|   |-- Cargo.toml              # Rust workspace config
|   |-- Cargo.lock
|   |-- uv.lock
|   |-- Makefile
|   |-- Dockerfile
|   |-- docker-compose.yml
|
|-- crates/                     # Rust components
|   |-- ov_cli/                # Rust CLI tool
|       |-- src/
|       |   |-- main.rs         # Rust CLI entry point
|       |   |-- client.rs       # HTTP client
|       |   |-- commands/       # CLI subcommands
|       |   |-- tui/            # Terminal UI
|       |   |-- config.rs
|       |   |-- error.rs
|       |   |-- output.rs
|       |-- Cargo.toml
|       |-- build.rs
|       |-- install.sh          # Installation script
|
|-- openviking/                # Main Python package (context DB core)
|   |-- __init__.py
|   |-- doctor.py               # Diagnostics tool
|   |-- rust_cli.py             # Python wrapper for Rust CLI
|   |-- exceptions.py
|   |-- sync_client.py          # Synchronous client
|   |-- client/                 # Client modules
|   |-- core/                   # Core functionality
|   |-- crypto/                 # Cryptography
|   |-- eval/                   # Evaluation
|   |-- message/                # Message handling
|   |-- models/                 # Data models
|   |-- parse/                  # Parsing (code, docs)
|   |-- prompts/                # Prompt templates
|   |-- pyagfs/                 # AGFS bindings
|   |-- resource/               # Resource management
|   |-- retrieve/               # Retrieval logic
|   |-- server/                 # Server components
|   |-- service/                # Services
|   |-- session/                # Session management
|   |-- storage/                # Storage backends
|   |-- telemetry/              # Telemetry/monitoring
|   |-- utils/                  # Utilities
|
|-- openviking_cli/            # CLI package (thin wrapper)
|   |-- rust_cli.py             # Entry: finds & execs Rust binary
|   |-- server_bootstrap.py     # Entry: openviking-server
|   |-- doctor.py
|   |-- exceptions.py
|   |-- client/
|   |-- retrieve/
|   |-- session/
|   |-- utils/
|
|-- bot/                        # VikingBot (AI Agent framework)
|   |-- vikingbot/              # Bot package
|       |-- __init__.py
|       |-- __main__.py         # Entry: python -m vikingbot
|       |-- cli/
|       |   |-- commands.py     # Entry: vikingbot CLI app
|       |-- agent/              # Agent logic
|       |-- bus/                # Event bus
|       |-- channels/           # Channel integrations
|       |-- config/             # Configuration
|       |-- console/            # Console UI
|       |-- cron/               # Scheduling
|       |-- hooks/              # Hooks system
|       |-- integrations/        # Third-party integrations
|       |-- openviking_mount/   # OpenViking integration
|       |-- providers/           # AI providers
|       |-- sandbox/            # Sandboxing
|       |-- session/            # Session management
|
|-- src/                        # Additional Python source (workspace)
|   |-- bridge/                 # Bridge components
|   |-- deploy/                 # Deployment configs
|   |-- docs/                   # Documentation
|   |-- eval/                   # Evaluation
|   |-- license/                # License files
|   |-- scripts/                # Build/utility scripts
|   |-- tests/                  # Test suite
|   |-- vikingbot/              # Bot source (symlink to bot/vikingbot?)
|   |-- workspace/              # Workspace configs
|
|-- tests/                      # Integration & unit tests
|   |-- api_test/
|   |-- cli/
|   |-- client/
|   |-- engine/
|   |-- integration/
|   |-- misc/
|   |-- parse/
|   |-- server/
|   |-- session/
|   |-- storage/
|   |-- unit/
|   |-- vectordb/
|
|-- examples/                   # Example code & configs
|   |-- basic-usage/
|   |-- cloud/
|   |-- k8s-helm/
|   |-- openclaw-plugin/
|   |-- server_client/
|   |-- skills/
|   |-- ov.conf.example
|   |-- ovcli.conf.example
|
|-- build_support/              # Build support files
|-- deploy/                     # Deployment configs
|-- docs/                       # Documentation
|-- third_party/                # Third-party code (AGFS SDK)
|
|-- Configuration
|   |-- .clang-format
|   |-- .gitignore
|   |-- .pre-commit-config.yaml
|   |-- .pr_agent.toml
|   |-- CONTRIBUTING.md
|   |-- CONTRIBUTING_CN.md
|   |-- CONTRIBUTING_JA.md
|   |-- LICENSE
|   |-- MANIFEST.in
|   |-- README.md
```

---

## Key Entry Points

### CLI Commands (Python Entry Points via pyproject.toml)

| Command | Entry | Description |
|---------|-------|-------------|
| `ov` | `openviking_cli.rust_cli:main` | Rust CLI wrapper (finds & execs binary) |
| `openviking` | `openviking_cli.rust_cli:main` | Alias for `ov` |
| `openviking-server` | `openviking_cli.server_bootstrap:main` | Server bootstrap |
| `vikingbot` | `vikingbot.cli.commands:app` | Bot CLI (Typer app) |

### Direct Module Execution

```bash
# Run vikingbot as module
python -m vikingbot

# Run openviking-server
python -m openviking_cli.server_bootstrap
```

### Rust CLI

- **Binary:** `crates/ov_cli/src/main.rs`
- **Commands:** `crates/ov_cli/src/commands/` (chat, session, admin, search, etc.)
- **TUI:** `crates/ov_cli/src/tui/`
- **Installation:** `crates/ov_cli/install.sh`

---

## Key Files

### Configuration

| File | Purpose |
|------|---------|
| `pyproject.toml` | Python package metadata, dependencies, entry points |
| `Cargo.toml` | Rust workspace (ov_cli) |
| `setup.py` | Legacy setuptools build |

### Core Python Package (`openviking/`)

| File/Dir | Purpose |
|----------|---------|
| `client.py`, `sync_client.py`, `async_client.py` | Client implementations |
| `session/` | Session management |
| `storage/` | Storage backends (vectordb, etc.) |
| `retrieve/` | Retrieval logic |
| `parse/` | Code/document parsing (tree-sitter) |
| `server/` | Server components |
| `doctor.py` | Diagnostics |

### Bot Framework (`bot/vikingbot/`)

| File/Dir | Purpose |
|----------|---------|
| `cli/commands.py` | CLI app entry (Typer) |
| `agent/` | Agent logic |
| `channels/` | Channel integrations (Telegram, Slack, Lark, etc.) |
| `providers/` | AI provider integrations |
| `sandbox/` | Sandboxing for code execution |

---

## Tech Stack

### Python
- **Core:** Python 3.10+ (pydantic, httpx, fastapi, uvicorn)
- **AI:** openai, litellm
- **Parsing:** tree-sitter (multiple languages)
- **Storage:** Vector DB, various backends
- **CLI:** typer, prompt-toolkit, rich

### Rust
- **CLI:** clap (argument parsing), ratatui (TUI)
- **HTTP:** reqwest

### Build
- **Python:** setuptools, setuptools-scm
- **Rust:** Cargo
- **C++:** CMake for native extensions

---

## Patterns Observed

1. **Hybrid Architecture:** Python primary with Rust CLI for performance-critical operations
2. **Wrapper Pattern:** `rust_cli.py` is a thin Python wrapper that locates and execs the Rust binary
3. **Workspace Layout:** Root-level `pyproject.toml` with packages in `openviking/`, `bot/`, `crates/`
4. **Multi-Component:** Separate packages for core DB, CLI, and bot framework
