# OpenViking Tech Stack

## Overview

OpenViking is a **hybrid Python/Rust/C++/Go polyglot project** developed by ByteDance (volcengine organization). It is designed as an Agent-native context database with native extensions for performance-critical components.

## Language Breakdown

| Language | Purpose | Version | LOC (approx) |
|----------|---------|---------|--------------|
| Python | Primary language, core library | 3.10 - 3.14 | 79% |
| C++ | Vector database engine extensions | C++17 | 6% |
| Rust | CLI binary (`ov`) | 1.88+ | 3% |
| Go | AGFS server component | 1.22+ | <5% |
| JavaScript/TypeScript | Bot framework (vikingbot) | Node.js | 1% |

## Build System Architecture

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

- **Build system**: CMake (minimum 3.12)
- **Extension type**: Python stable ABI (abi3) extensions
- **Source location**: `src/` directory

### Go Build

- **Purpose**: AGFS (Agent File System) server
- **Location**: `third_party/agfs/agfs-server/`
- **Minimum version**: 1.22

## Core Python Dependencies

### Data Validation & API

| Package | Purpose |
|---------|---------|
| `pydantic>=2.0` | Data validation and serialization |
| `fastapi>=0.128` | HTTP API framework |
| `uvicorn>=0.39` | ASGI server |

### AI/ML Integration

| Package | Purpose |
|---------|---------|
| `openai>=1.0` | OpenAI API client |
| `litellm>=1.0` | LLM abstraction layer (supports Claude, DeepSeek, Gemini, Qwen, vLLM, Ollama) |
| `volcengine>=1.0` | Volcengine cloud SDK |
| `volcengine-python-sdk[ark]>=5.0` | ARK platform SDK |

### Code Parsing

| Package | Purpose |
|---------|---------|
| `tree-sitter>=0.23` | AST parsing |
| `tree-sitter-python` | Python grammar |
| `tree-sitter-javascript` | JavaScript grammar |
| `tree-sitter-typescript` | TypeScript grammar |
| `tree-sitter-cpp` | C++ grammar |
| `tree-sitter-rust` | Rust grammar |
| `tree-sitter-go` | Go grammar |
| `tree-sitter-java` | Java grammar |
| `tree-sitter-c-sharp` | C# grammar |

### Document Processing

| Package | Purpose |
|---------|---------|
| `pdfplumber>=0.10.0` | PDF text extraction |
| `pdfminer-six>=20251230` | PDF parsing |
| `python-docx>=1.0.0` | Word document processing |
| `openpyxl>=3.0.0` | Excel file processing |
| `python-pptx>=1.0.0` | PowerPoint processing |
| `ebooklib>=0.18.0` | E-book format support |
| `markdownify>=0.11.0` | HTML to Markdown conversion |
| `readabilipy>=0.2.0` | Readability extraction |

### Storage & Serialization

| Package | Purpose |
|---------|---------|
| `protobuf>=6.33` | Protocol buffers |
| `xxhash>=3.0.0` | Fast hashing |
| `pyyaml>=6.0` | YAML configuration |

### Utilities

| Package | Purpose |
|---------|---------|
| `httpx>=0.25` | HTTP client |
| `requests>=2.31.0` | HTTP library |
| `loguru>=0.7.3` | Logging |
| `jinja2>=3.1.6` | Templating |
| `tabulate>=0.9.0` | Table formatting |
| `cryptography>=42.0.0` | Encryption |
| `argon2-cffi>=23.0.0` | Password hashing |

## Rust CLI Dependencies

| Package | Purpose |
|---------|---------|
| `clap` | CLI argument parsing |
| `reqwest` | HTTP client |
| `tokio` | Async runtime |
| `ratatui` | Terminal UI |
| `crossterm` | Terminal capabilities |
| `serde` | Serialization |
| `anyhow` | Error handling |

## Optional Dependency Groups

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

## Platform Requirements

| Tool | Minimum Version |
|------|-----------------|
| Python | 3.10 |
| Go | 1.22 |
| CMake | 3.12 |
| Rust | 1.88 |
| GCC/Clang | GCC 9 / Clang 11 |

## Supported Platforms (Pre-compiled Wheels)

- **Windows**: x86_64
- **macOS**: x86_64 (Intel), arm64 (Apple Silicon)
- **Linux**: x86_64, aarch64 (manylinux)

## Notable Architectural Patterns

1. **Python extensions**: C++ vector database engine built via CMake, loaded as abi3 extensions
2. **Rust CLI wrapper**: Python CLI (`openviking`) is a thin wrapper around Rust binary (`ov`)
3. **Embedded Go server**: AGFS server compiled and bundled within Python package
4. **Bot framework**: Separate Node.js/TypeScript service communicating via WebSocket/HTTP
5. **Tree-sitter integration**: Multi-language code parsing for 8+ languages

## Development Tools

| Tool | Purpose |
|------|---------|
| `ruff` | Linting (replaces flake8, isort, black) |
| `mypy` | Type checking |
| `pytest` | Testing with asyncio support |
| `pre-commit` | Git hooks configuration |

## Project Structure Summary

```
OpenViking/
├── pyproject.toml          # Python package config
├── Cargo.toml              # Rust workspace
├── setup.py                # Custom build script
├── Makefile                # Build orchestration
├── Dockerfile              # Multi-stage Docker build
├── .github/workflows/      # CI/CD pipelines
├── bot/                    # Vikingbot (Node.js/TypeScript)
├── crates/ov_cli/          # Rust CLI
├── openviking/             # Core Python library
├── src/                    # C++ extension source
└── third_party/            # Embedded dependencies (AGFS, LevelDB, rapidjson, spdlog)
```
