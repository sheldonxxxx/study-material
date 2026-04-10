# OpenViking Tech Stack Analysis

## Project Overview

OpenViking is an **Agent-native context database** developed by ByteDance. It is a **hybrid Python/Rust/C++/Go polyglot project** with native extensions.

## Languages and Runtimes

| Language | Purpose | Version |
|----------|---------|---------|
| Python | Primary language, core library | 3.10 - 3.14 |
| Rust | CLI binary (`ov`) | 1.88+ |
| C++ | Vector database engine extensions | C++17 |
| Go | AGFS server component | 1.22+ |
| JavaScript/TypeScript | Bot framework (vikingbot) | Node.js |

## Build System

### Python Package Management

- **Primary tool**: `uv` (Astral's fast Python package manager)
- **Lock file**: `uv.lock`
- **Build backend**: setuptools with `setuptools.build_meta`
- **Version management**: setuptools-scm (git tag-based versioning)
- **Build entry point**: `setup.py` with custom `build_ext` command

### Rust Build

- **Workspace**: Cargo workspace with single member `crates/ov_cli`
- **Lock file**: `Cargo.lock`
- **Profile**: Release builds with LTO and strip enabled

### C++ Build

- **Build system**: CMake
- **Minimum version**: 3.12
- **Extension type**: Python stable ABI (abi3) extensions
- **Source location**: `src/` directory

### Go Build

- **Purpose**: AGFS (Agent File System) server
- **Location**: `third_party/agfs/agfs-server/`
- **Minimum version**: 1.22

## Project Structure

```
OpenViking/
├── Cargo.toml          # Rust workspace
├── Cargo.lock
├── pyproject.toml      # Python package config
├── uv.lock
├── setup.py            # Custom build script
├── Makefile            # Build orchestration
├── Dockerfile          # Multi-stage Docker build
├── docker-compose.yml
├── .github/
│   └── workflows/      # CI/CD pipelines
├── bot/                # Vikingbot (Node.js/TypeScript)
│   ├── package.json
│   ├── vikingbot/      # Bot implementation
│   └── bridge/         # Bot bridge components
├── crates/
│   └── ov_cli/         # Rust CLI (`ov` binary)
├── openviking/         # Core Python library
├── openviking_cli/      # Python CLI bootstrap
├── src/                # C++ extension source
├── third_party/
│   ├── agfs/           # AGFS SDK and server (Go)
│   ├── leveldb-1.23/   # LevelDB fork
│   ├── rapidjson/      # JSON library
│   └── spdlog-1.14.1/  # Logging library
└── examples/           # Usage examples
```

## Key Dependencies

### Core Python Dependencies

| Package | Purpose |
|---------|---------|
| `pydantic>=2.0` | Data validation |
| `fastapi>=0.128` | HTTP API framework |
| `uvicorn>=0.39` | ASGI server |
| `httpx>=0.25` | HTTP client |
| `openai>=1.0` | OpenAI API client |
| `litellm>=1.0` | LLM abstraction layer |
| `volcengine>=1.0` | Volcengine cloud SDK |
| `volcengine-python-sdk[ark]>=5.0` | ARK platform SDK |
| `tree-sitter>=0.23` | AST parsing (multiple languages) |
| `protobuf>=6.33` | Protocol buffers |

### Optional Dependency Groups

| Group | Purpose |
|-------|---------|
| `test` | Pytest, boto3, ragas, pandas |
| `dev` | mypy, ruff |
| `doc` | Sphinx documentation |
| `eval` | RAG evaluation |
| `gemini` | Google Gemini support |
| `bot` | Vikingbot core dependencies |
| `bot-langfuse` | Langfuse observability |
| `bot-telegram` | Telegram integration |
| `bot-feishu` | Feishu/Lark integration |
| `bot-dingtalk` | DingTalk integration |
| `bot-slack` | Slack integration |
| `bot-qq` | QQ bot support |
| `bot-sandbox` | Code sandbox execution |
| `bot-fuse` | FUSE filesystem support |
| `bot-opencode` | OpenCode AI integration |

### Rust CLI Dependencies

| Package | Purpose |
|---------|---------|
| `clap` | CLI argument parsing |
| `reqwest` | HTTP client |
| `tokio` | Async runtime |
| `ratatui` | Terminal UI |
| `crossterm` | Terminal capabilities |
| `serde` | Serialization |
| `anyhow` | Error handling |

## CI/CD Pipeline

### Workflows (GitHub Actions)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | Push to main | Main branch checks |
| `pr.yml` | Pull request | PR validation |
| `_build.yml` | workflow_call | Python package builds |
| `_lint.yml` | workflow_call | Code linting |
| `_test_full.yml` | workflow_call | Full test suite |
| `_test_lite.yml` | workflow_call | Quick tests |
| `_codeql.yml` | workflow_call | Security scanning |
| `rust-cli.yml` | Push to main, PR | Rust CLI builds (5 platforms) |
| `build-docker-image.yml` | Push | Docker image builds |
| `release.yml` | Tag | PyPI releases |
| `schedule.yml` | Cron | Scheduled tasks |
| `api_test.yml` | Manual | API integration tests |

### Build Platforms

Rust CLI builds for:
- Linux x86_64 (glibc 2.31)
- Linux ARM64 (aarch64)
- macOS x86_64 (Intel)
- macOS ARM64 (Apple Silicon)
- Windows x86_64

Python wheels built for:
- Linux x86_64, aarch64
- macOS x86_64, arm64
- Windows x86_64
- Python 3.10 - 3.14

## Docker

### Multi-Stage Dockerfile

1. **go-toolchain**: Go 1.26 for AGFS server builds
2. **rust-toolchain**: Rust 1.88 for CLI builds
3. **py-builder**: Python 3.13 with uv, builds all artifacts
4. **runtime**: Python 3.13 slim runtime image

### Docker Compose

Single service `openviking` exposing port 1933 with health checks.

## Development Tools

| Tool | Purpose |
|------|---------|
| `ruff` | Linting (replaces flake8, isort, black) |
| `mypy` | Type checking |
| `pytest` | Testing with asyncio support |
| `pre-commit` | Git hooks configuration |

## Monorepo Characteristics

- **Python packages**: `openviking`, `vikingbot` (both in pyproject.toml)
- **Rust workspace**: `crates/ov_cli`
- **Go components**: `third_party/agfs/`
- **Node.js components**: `bot/` (separate package.json)
- **No formal monorepo tool** (Nx, Turborepo, etc.) - uses raw structure

## Platform Requirements

| Tool | Minimum Version |
|------|-----------------|
| Python | 3.10 |
| Go | 1.22 |
| CMake | 3.12 |
| Rust | 1.88 |
| GCC/Clang | GCC 9 / Clang 11 |

## Notable Architectural Patterns

1. **Python extensions**: C++ vector database engine built via CMake, loaded as abi3 extensions
2. **Rust CLI wrapper**: Python CLI (`openviking`) is a thin wrapper around Rust binary (`ov`)
3. **Embedded Go server**: AGFS server compiled and bundled within Python package
4. **Bot framework**: Separate Node.js/TypeScript service communicating via WebSocket/HTTP
5. **Tree-sitter integration**: Multi-language code parsing (Python, JS, TS, Java, C++, Rust, Go, C#)
