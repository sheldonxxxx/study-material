# Dependencies

## Overview

CrewAI is a monorepo with 4 packages managed via `uv workspace`:

```
lib/
  crewai/           # Core framework (main dependency)
  crewai-tools/     # Tool integrations (70+ tools)
  crewai-files/     # File handling utilities
  devtools/         # Internal release tooling
```

## Core Package (crewai)

### Runtime Dependencies

```toml
[project]
requires-python = ">=3.10, <3.14"
dependencies = [
    # Core AI
    "pydantic~=2.11.9",
    "openai>=1.83.0,<3",
    "instructor>=1.3.3",

    # Text Processing
    "pdfplumber~=0.11.4",
    "regex~=2026.1.15",

    # Telemetry
    "opentelemetry-api~=1.34.0",
    "opentelemetry-sdk~=1.34.0",
    "opentelemetry-exporter-otlp-proto-http~=1.34.0",

    # Data/Vectors
    "chromadb~=1.1.0",
    "tokenizers>=0.21,<1",
    "openpyxl~=3.1.5",
    "lancedb>=0.29.2",

    # Auth/Config
    "python-dotenv~=1.1.1",
    "pyjwt>=2.9.0,<3",

    # TUI
    "textual>=7.5.0",

    # Utilities
    "click~=8.1.7",
    "appdirs~=1.4.4",
    "jsonref~=1.1.0",
    "json-repair~=0.25.2",
    "tomli-w~=1.1.0",
    "tomli~=2.0.2",
    "json5~=0.10.0",
    "portalocker~=2.7.0",
    "pydantic-settings~=2.10.1",
    "httpx~=0.28.1",
    "mcp~=1.26.0",
    "uv~=0.9.13",
    "aiosqlite~=0.21.0",
    "pyyaml~=6.0",
]
```

### Optional Dependencies

| Extra | Packages | Purpose |
|-------|----------|---------|
| `tools` | `crewai-tools` | All integration tools |
| `embeddings` | `tiktoken` | Token counting |
| `pandas` | `pandas` | DataFrame support |
| `openpyxl` | `openpyxl` | Excel file support |
| `mem0` | `mem0ai` | Memory backend |
| `docling` | `docling` | Document parsing |
| `qdrant` | `qdrant-client[fastembed]` | Vector search |
| `aws` | `boto3`, `aiobotocore` | AWS integration |
| `bedrock` | `boto3` | AWS Bedrock |
| `watson` | `ibm-watsonx-ai` | IBM Watson |
| `voyageai` | `voyageai` | Embeddings |
| `litellm` | `litellm` | Multi-LLM gateway |
| `google-genai` | `google-genai` | Google AI models |
| `azure-ai-inference` | `azure-ai-inference` | Azure AI |
| `anthropic` | `anthropic` | Claude models |
| `a2a` | `a2a-sdk`, `httpx-auth`, `httpx-sse`, `aiocache` | Agent-to-Agent |
| `file-processing` | `crewai-files` | File handling |
| `qdrant-edge` | `qdrant-edge-py` | Edge vector search |

## Tools Package (crewai-tools)

### Runtime Dependencies

```toml
[project]
requires-python = ">=3.10, <3.14"
dependencies = [
    "pytube~=15.0.0",
    "requests~=2.32.5",
    "docker~=7.1.0",
    "crewai==1.13.0rc1",
    "tiktoken~=0.8.0",
    "beautifulsoup4~=4.13.4",
    "python-docx~=1.2.0",
    "youtube-transcript-api~=1.2.2",
    "pymupdf~=1.26.6",
]
```

### Optional Tool Dependencies (70+ integrations)

| Category | Tools |
|----------|-------|
| **Search** | `serpapi`, `tavily-python`, `exa-py`, `duckduckgo-search` |
| **Web Scraping** | `scrapfly-sdk`, `firecrawl-py`, `scrapegraph-py`, `hyperbrowser`, `selenium` |
| **Vector DBs** | `qdrant-client`, `weaviate-client`, `chromadb` (via crewai), `singlestoredb` |
| **Cloud Data** | `snowflake-connector-python`, `couchbase`, `pymongo`, `pymysql`, `psycopg2-binary` |
| **LLM Platforms** | AWS Bedrock (`bedrock-agentcore`), IBM Watson, Google GenAI, Azure AI |
| **Browser** | `browserbase`, `playwright` (Bedrock) |
| **Agents** | `composio-core`, `contextual-client`, `stagehand` |
| **RAG** | `python-docx`, `lxml`, `unstructured` |
| **Video** | `pytube`, `youtube-transcript-api` |
| **Git** | `gitpython`, `PyGithub` |
| **MCP** | `mcp`, `mcpadapt` |
| **Databases** | `sqlalchemy`, `databricks-sdk` |

## Dependency Categories

### AI/LLM

| Package | Purpose | Version Constraint |
|---------|---------|-------------------|
| `openai` | OpenAI API client | `>=1.83.0,<3` |
| `instructor` | Structured outputs | `>=1.3.3` |
| `anthropic` | Claude models | `~=0.73.0` |
| `google-genai` | Gemini models | `~=1.65.0` |
| `litellm` | Multi-LLM gateway | `>=1.74.9,<=1.82.6` |
| `tiktoken` | Tokenizer | `~=0.8.0` |

### Vector/Memory

| Package | Purpose |
|---------|---------|
| `chromadb` | Vector store |
| `lancedb` | Vector store |
| `mem0ai` | Memory backend |
| `qdrant-client` | Vector search |

### Data Processing

| Package | Purpose |
|---------|---------|
| `pdfplumber` | PDF extraction |
| `pymupdf` | PDF processing |
| `python-docx` | Word docs |
| `openpyxl` | Excel files |
| `beautifulsoup4` | HTML parsing |

### Observability

| Package | Purpose |
|---------|---------|
| `opentelemetry-api` | Telemetry API |
| `opentelemetry-sdk` | Telemetry SDK |
| `opentelemetry-exporter-otlp-proto-http` | OTLP export |

### Web/HTTP

| Package | Purpose |
|---------|---------|
| `httpx` | Async HTTP client |
| `requests` | HTTP library |
| `mcp` | Model Context Protocol |

### CLI/TUI

| Package | Purpose |
|---------|---------|
| `click` | CLI framework |
| `textual` | Terminal UI |

## Security Practices

### Dependency Pinning

Critical security patches are pinned:

```toml
[tool.uv]
override-dependencies = [
    "langchain-core>=1.2.11,<2",  # CVE-2026-26013 SSRF fix
    "urllib3>=2.6.3",
    "pillow>=12.1.1",
]
```

### Version Constraints

| Package | Constraint | Reason |
|---------|------------|--------|
| `onnxruntime` | `<1.24; python_version < '3.11'` | Python 3.10 wheel compatibility |
| `pillow` | `>=12.1.1` | Security fix |
| `langchain-core` | `>=1.2.11,<2` | CVE-2026-26013 |

### Security Scanning

1. **CodeQL** - GitHub Actions workflow for vulnerability scanning
2. **Bandit** - ruff `S` rules for security issues in code
3. **Dependency Review** - GitHub-native dependency scanning

## Development Dependencies

```toml
[dependency-groups]
dev = [
    "ruff==0.15.1",
    "mypy==1.19.1",
    "pre-commit==4.5.1",
    "bandit==1.9.2",
    "pytest==8.4.2",
    "pytest-asyncio==1.3.0",
    "pytest-subprocess==1.5.3",
    "vcrpy==7.0.0",
    "pytest-recording==0.13.4",
    "pytest-randomly==4.0.1",
    "pytest-timeout==2.4.0",
    "pytest-xdist==3.8.0",
    "pytest-split==0.10.0",
    "types-requests~=2.31.0.6",
    "types-pyyaml==6.0.*",
    "types-regex==2026.1.15.*",
    "types-appdirs==1.4.*",
    "boto3-stubs[bedrock-runtime]==1.42.40",
    "types-psycopg2==2.9.21.20251012",
    "types-pymysql==1.1.0.20250916",
    "types-aiofiles~=25.1.0",
    "commitizen>=4.13.9",
]
```

## Dependency Update Policy

### Release Process

1. Nightly canary releases (`nightly.yml`)
2. Manual release trigger (`publish.yml`)
3. Cross-package pins updated automatically in nightly builds
4. Production releases require manual workflow dispatch

### Constraints

- PyTorch index configuration for Python 3.13 compatibility
- PyTorch nightly for Python >=3.13, stable for <3.13
- Crewai-tools pins `crewai==1.13.0rc1` as runtime dependency

## Lock File

`uv.lock` is committed to the repository and validated by pre-commit hooks:

```bash
uv lock
```

## Installation Variants

```bash
# Core only
uv pip install crewai

# With tools
uv pip install 'crewai[tools]'

# Specific integrations
uv pip install 'crewai[anthropic,google-genai]'
uv pip install 'crewai[qdrant,bedrock]'
```

## Workspace Configuration

```toml
[tool.uv.workspace]
members = [
    "lib/crewai",
    "lib/crewai-tools",
    "lib/devtools",
    "lib/crewai-files",
]

[tool.uv.sources]
crewai = { workspace = true }
crewai-tools = { workspace = true }
crewai-devtools = { workspace = true }
crewai-files = { workspace = true }
```
