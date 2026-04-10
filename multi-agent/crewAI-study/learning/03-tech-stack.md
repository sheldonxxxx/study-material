# Tech Stack

## Python Version Support

- **Minimum:** Python 3.10
- **Maximum:** Python 3.13
- **Development Target:** Python 3.12
- **Constraint:** `>=3.10,<3.14` across all packages

## Package Manager

**uv** (Astral's Python package manager) is the sole package management tool:

```bash
# Install crewai
uv pip install crewai

# Install with tools
uv pip install 'crewai[tools]'

# Development setup
uv sync --all-groups --all-extras
```

The project uses `uv.lock` for reproducible dependency resolution and `uv workspace` for monorepo management with members under `lib/`.

## Build System

**hatchling** is the build backend for all packages:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Version is dynamically read from `__init__.py` files via hatchling's path strategy.

## Workspace Structure

```
lib/
  crewai/           # Core framework
  crewai-tools/     # Tool integrations
  crewai-files/     # File handling
  devtools/         # Internal release tooling
```

## Key Dependencies

### Core Framework (crewai)

| Category | Libraries |
|----------|-----------|
| **AI/ML** | `openai>=1.83.0,<3`, `instructor>=1.3.3` |
| **Data Validation** | `pydantic~=2.11.9`, `pydantic-settings~=2.10.1` |
| **Vector Storage** | `chromadb~=1.1.0`, `lancedb>=0.29.2` |
| **Telemetry** | `opentelemetry-api~=1.34.0`, `opentelemetry-sdk~=1.34.0` |
| **TUI** | `textual>=7.5.0` |
| **CLI** | `click~=8.1.7` |
| **HTTP** | `httpx~=0.28.1` |
| **Protocol** | `mcp~=1.26.0` (Model Context Protocol), `uv~=0.9.13` |
| **Database** | `aiosqlite~=0.21.0` |
| **Config** | `pyyaml~=6.0`, `python-dotenv~=1.1.1`, `tomli~=2.0.2` |

### Tool Integrations (crewai-tools)

| Category | Libraries |
|----------|-----------|
| **Web Scraping** | `beautifulsoup4~=4.13.4`, `pytube~=15.0.0`, `requests~=2.32.5` |
| **Document Parsing** | `pdfplumber`, `python-docx~=1.2.0`, `pymupdf~=1.26.6` |
| **Video** | `youtube-transcript-api~=1.2.2` |
| **LLM Embeddings** | `tiktoken~=0.8.0` |
| **Search** | Multiple optional: `serpapi`, `tavily-python`, `exa-py`, `duckduckgo-search` |
| **Vector DBs** | `qdrant-client`, `weaviate-client`, `chromadb` |
| **Cloud APIs** | AWS (`boto3`), Snowflake, Databricks, Couchbase, MongoDB, PostgreSQL |

### Optional Extras

crewai provides extensive optional dependencies:

```toml
[project.optional-dependencies]
tools = ["crewai-tools"]      # All tools
embeddings = ["tiktoken"]     # Tokenization
pandas = ["pandas"]           # Dataframe support
openpyxl = ["openpyxl"]       # Excel support
mem0 = ["mem0ai"]             # Memory backend
docling = ["docling"]          # Document parsing
qdrant = ["qdrant-client"]    # Vector search
aws = ["boto3"]               # AWS integration
bedrock = ["boto3"]           # AWS Bedrock
anthropic = ["anthropic"]     # Claude models
voyageai = ["voyageai"]       # Embeddings
litellm = ["litellm"]         # Multi-LLM gateway
google-genai = ["google-genai"]
azure-ai-inference = ["azure-ai-inference"]
a2a = ["a2a-sdk"]             # Agent-to-Agent protocol
```

## Type System

### mypy Configuration

Strict type checking is enforced with `pydantic.mypy` plugin:

```toml
[tool.mypy]
strict = true
disallow_untyped_defs = true
disallow_any_unimported = true
no_implicit_optional = true
check_untyped_defs = true
warn_return_any = true
python_version = "3.12"
plugins = ["pydantic.mypy"]
```

CI runs mypy on Python 3.10, 3.11, 3.12, and 3.13 for every PR.

### Type Annotations Policy

From CONTRIBUTING.md:
- Use built-in generics (`list[str]`, `dict[str, int]`) not `typing.List`/`typing.Dict`
- Full type annotations on all functions, methods, and classes
- Use `isinstance`, `TypeIs`, or `TypeGuard` for type narrowing
- Avoid bare `dict`/`list` without type parameters

## Code Quality Tools

### Ruff (Linter + Formatter)

```toml
[tool.ruff]
src = ["lib/*"]
target-version = "py310"
fix = true
```

Rules enabled: E, F, B, S, RUF, N, W, I, T, PERF, PIE, TID, ASYNC, RET, UP (all modernizations)

### Pre-commit Hooks

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff
      - id: ruff-format
      - id: mypy
  - repo: astral-sh/uv-pre-commit
    hooks:
      - id: uv-lock
  - repo: commitizen-tools/commitizen
    hooks:
      - id: commitizen
```

## Key Architectural Decisions

1. **No LangChain dependency** - CrewAI is built entirely from scratch, independent of other agent frameworks

2. **Pydantic v2 for validation** - All agent configs, task definitions, and tool schemas use Pydantic models

3. **Instructor for structured outputs** - Enables reliable LLM response parsing with type validation

4. **Dual storage backends** - ChromaDB and LanceDB both supported for vector storage flexibility

5. **OpenTelemetry for observability** - Vendor-neutral telemetry with OTLP export

6. **Textual for CLI TUI** - Terminal user interface for the crewai CLI tool

7. **MCP protocol support** - Native Model Context Protocol integration for agent communication

8. **A2A SDK support** - Agent-to-Agent protocol for inter-agent communication
