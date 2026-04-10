# OpenViking Project Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Project Type:** Hybrid Python + Rust with C++ extensions
**Core Purpose:** An Agent-native context database for AI Agents

---

## What is OpenViking?

OpenViking is a **context database** designed specifically for AI Agent workloads. It provides a persistent memory layer that allows AI agents to store, retrieve, and reason over contextual information across sessions and interactions.

### Key Value Proposition

1. **Agent-Native Design** - Built from the ground up for AI agent workflows, not retrofitted from general-purpose databases
2. **Hybrid Architecture** - Python for flexibility and rapid development, Rust for performance-critical CLI operations, C++ for vector computation
3. **Memory Management** - Structured memory extraction, deduplication, and similarity-based retrieval
4. **Multi-Channel Integration** - VikingBot framework supports Telegram, Slack, Lark, and other messaging platforms

---

## Project Structure

```
OpenViking/
|-- crates/ov_cli/          # Rust CLI tool (high-performance terminal interface)
|-- openviking/             # Core Python package (context DB engine)
|-- openviking_cli/          # CLI wrapper package
|-- bot/vikingbot/          # AI Agent framework (multi-channel bot)
|-- src/                    # C++ vector database extensions
|-- tests/                  # Comprehensive test suite
|-- examples/               # Usage examples
|-- third_party/            # AGFS SDK bindings
```

---

## Core Components

### 1. OpenViking Core (`openviking/`)

The context database engine providing:

| Module | Purpose |
|--------|---------|
| `client/` | Client implementations (sync/async) |
| `core/` | Core functionality |
| `crypto/` | Encryption at rest (AES-256-GCM) |
| `eval/` | Evaluation metrics |
| `message/` | Message handling |
| `models/` | Data models |
| `parse/` | Code/document parsing (tree-sitter) |
| `retrieve/` | Retrieval logic |
| `server/` | Server components |
| `session/` | Session management |
| `storage/` | Storage backends (vector DB) |
| `telemetry/` | Observability |
| `utils/` | Utilities |

### 2. Rust CLI (`crates/ov_cli/`)

High-performance terminal interface with:

- **Commands:** chat, session, admin, search subcommands
- **TUI:** Terminal user interface (ratatui)
- **HTTP Client:** reqwest-based API client

### 3. VikingBot (`bot/vikingbot/`)

AI Agent framework with:

| Module | Purpose |
|--------|---------|
| `agent/` | Agent logic core |
| `bus/` | Event bus |
| `channels/` | Platform integrations |
| `providers/` | AI provider abstractions |
| `sandbox/` | Code execution sandboxing |
| `session/` | Session management |

---

## Tech Stack

### Python (Primary Language)
- **Core:** Python 3.10+, pydantic, httpx, fastapi, uvicorn
- **AI Integration:** openai, litellm
- **Parsing:** tree-sitter (multi-language code parsing)
- **CLI:** typer, prompt-toolkit, rich
- **Testing:** pytest, pytest-asyncio, pytest-cov

### Rust (Performance-Critical CLI)
- **CLI Framework:** clap
- **TUI:** ratatui
- **HTTP:** reqwest
- **Error Handling:** thiserror

### C++ (Vector Computation)
- **SIMD Optimizations:** SSE3, AVX2, AVX512
- **Python Bindings:** abi3 stable ABI (Python 3.10+)
- **Build:** CMake

---

## Entry Points

### CLI Commands

| Command | Entry Point | Description |
|---------|-------------|-------------|
| `ov` | `openviking_cli.rust_cli:main` | Rust CLI wrapper |
| `openviking` | `openviking_cli.rust_cli:main` | Alias for ov |
| `openviking-server` | `openviking_cli.server_bootstrap:main` | Server bootstrap |
| `vikingbot` | `vikingbot.cli.commands:app` | Bot CLI (Typer) |

### Module Execution

```bash
# Run vikingbot
python -m vikingbot

# Run server
python -m openviking_cli.server_bootstrap
```

---

## Architecture Patterns

### 1. Wrapper Pattern

The Python package `openviking_cli/rust_cli.py` is a thin wrapper that locates and executes the Rust binary. This allows the Rust CLI to be distributed as a standalone binary while maintaining Python package conventions.

### 2. Hybrid Architecture

```
Python (Core DB) <--> Rust (CLI/TUI) <--> C++ (Vector Engine)
```

- **Python:** Primary development language for business logic
- **Rust:** Performance-critical CLI operations and TUI
- **C++:** SIMD-optimized vector computations

### 3. Multi-Component Workspace

Root-level `pyproject.toml` coordinates multiple packages:
- `openviking/` - Core database
- `bot/` - Agent framework
- `openviking_cli/` - CLI wrappers
- `crates/ov_cli/` - Rust CLI

---

## Security Model

- **Encryption at Rest:** AES-256-GCM with envelope encryption
- **Key Management:** Three providers (LocalFile, Vault, VolcengineKMS)
- **API Key Security:** Argon2id hashing, HMAC constant-time comparison
- **Multi-Tenant Isolation:** Role-based access (ROOT, ADMIN, USER)

---

## Quality Assurance

- **Linting:** ruff (Python), clang-format (C++), no clippy in CI (Rust)
- **Type Checking:** mypy (lenient config)
- **Testing:** Comprehensive pytest suite (39+ test files)
- **CI Gates:** Lint, security scan (CodeQL), lite integration tests
- **PR Automation:** PR-Agent with 15 domain-specific review rules

---

## Summary

OpenViking is a sophisticated, polyglot project that combines Python's flexibility with Rust's performance and C++'s computational power. It is purpose-built as a context database for AI agents, featuring strong encryption, comprehensive testing, and multi-channel integration capabilities. The project follows professional software engineering practices with automated linting, type checking, and PR review workflows.
