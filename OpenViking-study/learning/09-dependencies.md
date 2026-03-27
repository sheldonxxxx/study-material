# OpenViking Dependencies

## Overview

OpenViking is a polyglot project with dependencies spanning Python, Rust, Go, C++, and JavaScript ecosystems. The project uses `uv` for Python package management and maintains dependencies across multiple lock files.

## Python Dependencies

### Core Dependencies

#### Data Validation & Serialization

| Package | Version | Purpose |
|---------|---------|---------|
| `pydantic` | >=2.0 | Data validation using Python type hints |
| `protobuf` | >=6.33 | Protocol buffer serialization |
| `pyyaml` | >=6.0 | YAML configuration parsing |

#### Web Framework & Server

| Package | Version | Purpose |
|---------|---------|---------|
| `fastapi` | >=0.128 | Modern Python web framework |
| `uvicorn` | >=0.39 | ASGI server implementation |
| `python-multipart` | >=0.0.22 | Multipart form parsing |

#### HTTP & Networking

| Package | Version | Purpose |
|---------|---------|---------|
| `httpx` | >=0.25 | HTTP client with async support |
| `requests` | >=2.31 | HTTP library |
| `urllib3` | >=2.6 | URL handling library |

#### AI/ML Integration

| Package | Version | Purpose |
|---------|---------|---------|
| `openai` | >=1.0 | OpenAI API client |
| `litellm` | >=1.0,<1.82.6 | Unified LLM interface (Claude, DeepSeek, Gemini, Qwen, vLLM, Ollama) |
| `volcengine` | >=1.0.216 | Volcengine cloud SDK |
| `volcengine-python-sdk[ark]` | >=5.0.3 | ARK platform SDK for Volcengine |

#### Code Parsing (Tree-sitter)

| Package | Version | Purpose |
|---------|---------|---------|
| `tree-sitter` | >=0.23 | AST parsing framework |
| `tree-sitter-python` | >=0.23 | Python grammar |
| `tree-sitter-javascript` | >=0.23 | JavaScript grammar |
| `tree-sitter-typescript` | >=0.23 | TypeScript grammar |
| `tree-sitter-java` | >=0.23 | Java grammar |
| `tree-sitter-cpp` | >=0.23 | C++ grammar |
| `tree-sitter-rust` | >=0.23 | Rust grammar |
| `tree-sitter-go` | >=0.23 | Go grammar |
| `tree-sitter-c-sharp` | >=0.23 | C# grammar |

#### Document Processing

| Package | Version | Purpose |
|---------|---------|---------|
| `pdfplumber` | >=0.10 | PDF text extraction |
| `pdfminer-six` | >=20251230 | PDF parsing |
| `python-docx` | >=1.0 | Word document (.docx) processing |
| `python-pptx` | >=1.0 | PowerPoint (.pptx) processing |
| `openpyxl` | >=3.0 | Excel (.xlsx) processing |
| `xlrd` | >=2.0 | Excel (.xls) reading |
| `ebooklib` | >=0.18 | E-book format support |
| `markdownify` | >=0.11 | HTML to Markdown conversion |
| `readabilipy` | >=0.2 | Readability extraction |
| `olefile` | >=0.47 | OLE compound documents |

#### Storage & Databases

| Package | Version | Purpose |
|---------|---------|---------|
| `xxhash` | >=3.0 | Fast non-cryptographic hashing |

#### Logging & Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `loguru` | >=0.7.3 | Logging library |
| `jinja2` | >=3.1.6 | Template engine |
| `tabulate` | >=0.9 | Table formatting |
| `cryptography` | >=42.0 | Encryption library |
| `argon2-cffi` | >=23.0 | Password hashing |
| `typing-extensions` | >=4.5 | Backported typing features |
| `apscheduler` | >=3.11 | Job scheduling |
| `json-repair` | >=0.25 | JSON repair/parsing |
| `typer` | >=0.12 | CLI application library |

### Optional Dependencies

#### Testing

| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | >=7.0 | Testing framework |
| `pytest-asyncio` | >=0.21 | Async test support |
| `pytest-cov` | >=4.0 | Coverage reporting |
| `boto3` | >=1.42 | AWS SDK |
| `ragas` | >=0.1 | RAG evaluation |
| `datasets` | >=2.0 | Dataset handling |
| `pandas` | >=2.0 | Data analysis |

#### Development

| Package | Version | Purpose |
|---------|---------|---------|
| `mypy` | >=1.0 | Static type checking |
| `ruff` | >=0.1 | Linting and formatting |

#### Documentation

| Package | Version | Purpose |
|---------|---------|---------|
| `sphinx` | >=7.0 | Documentation generator |
| `sphinx-rtd-theme` | >=1.3 | Read the Docs theme |
| `myst-parser` | >=2.0 | Markdown parser for Sphinx |

#### Evaluation

| Package | Version | Purpose |
|---------|---------|---------|
| `ragas` | >=0.1 | RAG evaluation framework |
| `datasets` | >=2.0 | Dataset utilities |
| `pandas` | >=2.0 | Data manipulation |

#### Google Gemini

| Package | Version | Purpose |
|---------|---------|---------|
| `google-genai` | >=1.0 | Google Gemini API client |

#### VikingBot

| Package | Version | Purpose |
|---------|---------|---------|
| `pydantic-settings` | >=2.0 | Pydantic settings management |
| `websockets` | >=12.0 | WebSocket client/server |
| `websocket-client` | >=1.6 | WebSocket client |
| `httpx[socks]` | >=0.25 | HTTP with SOCKS support |
| `readability-lxml` | >=0.8 | Readability extraction |
| `rich` | >=13.0 | Rich terminal output |
| `croniter` | >=2.0 | Cron parsing |
| `socksio` | >=1.0 | SOCKS protocol |
| `python-socketio` | >=5.11 | Socket.IO client |
| `msgpack` | >=1.0.8 | MessagePack serialization |
| `python-socks[asyncio]` | >=2.4 | Async SOCKS client |
| `prompt-toolkit` | >=3.0 | Interactive prompts |
| `pygments` | >=2.16 | Syntax highlighting |
| `html2text` | >=2020.1 | HTML to text |
| `beautifulsoup4` | >=4.12 | HTML parsing |
| `ddgs` | >=9.0 | DuckDuckGo search |
| `tavily-python` | >=0.5 | Tavily search API |
| `gradio` | >=6.6 | Gradio UI |
| `py-machineid` | >=1.0 | Machine ID extraction |

#### Bot Platform Integrations

| Package | Purpose |
|---------|---------|
| `langfuse` | Observability (bot-langfuse) |
| `python-telegram-bot[socks]` | Telegram (bot-telegram) |
| `lark-oapi` | Feishu/Lark (bot-feishu) |
| `dingtalk-stream` | DingTalk (bot-dingtalk) |
| `slack-sdk` | Slack (bot-slack) |
| `qq-botpy` | QQ (bot-qq) |
| `opensandbox`, `opensandbox-server`, `agent-sandbox` | Code sandbox (bot-sandbox) |
| `fusepy` | FUSE filesystem (bot-fuse) |
| `opencode-ai` | OpenCode AI (bot-opencode) |

## Rust Dependencies

Located in `crates/ov_cli/Cargo.toml`:

| Package | Purpose |
|---------|---------|
| `clap` | CLI argument parsing |
| `reqwest` | HTTP client |
| `tokio` | Async runtime |
| `ratatui` | Terminal UI library |
| `crossterm` | Terminal capabilities |
| `serde` | Serialization |
| `anyhow` | Error handling |

## Go Dependencies

Located in `third_party/agfs/`:
- AGFS SDK (embedded Python binding)
- Standard Go libraries

## C++ Dependencies

Located in `third_party/`:

| Library | Version | Purpose |
|---------|---------|---------|
| `leveldb` | 1.23 (fork) | Key-value storage |
| `rapidjson` | (bundled) | JSON parsing |
| `spdlog` | 1.14.1 | Logging library |

## Embedded Binaries

The project bundles pre-built binaries:

| Binary | Platform | Purpose |
|--------|----------|---------|
| `agfs-server` | Linux/macOS/Windows | Agent File System server |
| `ov` | Linux/macOS/Windows | Rust CLI binary |
| `libagfsbinding` | Linux/macOS/Windows | Go-Python binding library |

## Build Dependencies

| Package | Purpose |
|---------|---------|
| `setuptools` | Python packaging |
| `setuptools-scm` | Version from git tags |
| `cmake` | C++ build system |
| `wheel` | Wheel building |

## Dependency Management Files

| File | Manager | Purpose |
|------|---------|---------|
| `pyproject.toml` | - | Python package configuration |
| `uv.lock` | uv | Python dependency lock |
| `Cargo.lock` | Cargo | Rust dependency lock |
| `third_party/` | - | Bundled C++/Go libraries |

## Third-Party Libraries (Bundled)

| Library | Location | License | Purpose |
|---------|----------|---------|---------|
| LevelDB | third_party/leveldb-1.23 | BSD | Embedded KV store |
| RapidJSON | third_party/rapidjson | MIT | Fast JSON parsing |
| spdlog | third_party/spdlog-1.14.1 | BSD | Logging library |

## Dependency Groups

The project uses optional dependency groups for modular installation:

```bash
# Core only
pip install openviking

# With testing
pip install openviking[test]

# With bot framework
pip install openviking[bot]

# With all features
pip install openviking[bot-full]
```
