# crewAI Tech Stack Analysis

## Repository Structure: Monorepo

This is a **monorepo** with 4 packages under the `lib/` directory:

```
lib/
├── crewai/           # Core framework
├── crewai-tools/      # Tool integrations
├── crewai-files/     # File handling utilities
└── devtools/         # Development automation (private, not published)
```

### Root Workspace (`pyproject.toml`)
- Workspace members defined via `[tool.uv.workspace]`
- Centralized linting (ruff) and testing (pytest) configuration
- Dependency overrides via `[tool.uv.override-dependencies]`

---

## Python Version Support

| Setting | Value |
|---------|-------|
| Minimum | Python 3.10 |
| Maximum | Python 3.13 |
| `.python-version` | `3.13` |

CI/CD tests against: **3.10, 3.11, 3.12, 3.13**

---

## Build System

**hatchling** (`build-backend = "hatchling.build"`)

Each package uses Hatch's dynamic versioning from `__init__.py`:
```toml
[tool.hatch.version]
path = "src/crewai/__init__.py"
```

---

## Package Manager

**uv** (Astral's fast Python package manager)

- Lock file: `uv.lock` (ensures reproducible installs)
- Workspace-aware: `uv sync --all-groups --all-extras`
- Dependency groups via `[dependency-groups]` in root `pyproject.toml`

---

## Core Dependencies

### crewai (Main Framework)
| Category | Dependencies |
|----------|--------------|
| AI/LLM | `openai>=1.83.0,<3`, `instructor>=1.3.3` |
| Data/Embedding | `chromadb~=1.1.0`, `lancedb>=0.29.2`, `tokenizers>=0.21,<1` |
| Telemetry | `opentelemetry-api~=1.34.0`, `opentelemetry-sdk~=1.34.0` |
| TUI | `textual>=7.5.0` |
| Configuration | `pydantic~=2.11.9`, `pydantic-settings~=2.10.1` |
| CLI | `click~=8.1.7` |
| Document Processing | `pdfplumber~=0.11.4`, `openpyxl~=3.1.5` |
| HTTP | `httpx~=0.28.1` |
| Utils | `pyyaml~=6.0`, `pyjwt>=2.9.0,<3`, `python-dotenv~=1.1.1` |

### crewai-tools
| Category | Dependencies |
|----------|--------------|
| Web Scraping | `beautifulsoup4~=4.13.4`, `requests~=2.32.5` |
| Video | `pytube~=15.0.0`, `youtube-transcript-api~=1.2.2` |
| Document | `python-docx~=1.2.0`, `pymupdf~=1.26.6` |
| Docker | `docker~=7.1.0` |
| Core | `crewai==1.13.0rc1` |

### crewai-files
| Category | Dependencies |
|----------|--------------|
| Image | `Pillow~=12.1.1` |
| PDF | `pypdf~=6.9.1` |
| Audio/Video | `av~=13.0.0` |
| Async I/O | `aiofiles~=24.1.0`, `aiocache~=0.12.3` |
| Metadata | `tinytag~=2.2.1`, `python-magic>=0.4.27` |

### devtools
| Dependencies |
|--------------|
| `click~=8.1.7`, `toml~=0.10.2`, `openai~=1.83.0`, `pygithub~=1.59.1`, `rich>=13.9.4` |

---

## Optional Dependencies (Extras)

### crewai Extras
| Extra | Dependencies |
|-------|--------------|
| `tools` | `crewai-tools==1.13.0rc1` |
| `embeddings` | `tiktoken~=0.8.0` |
| `pandas` | `pandas~=2.2.3` |
| `mem0` | `mem0ai~=0.1.94` |
| `docling` | `docling~=2.75.0` |
| `qdrant` | `qdrant-client[fastembed]~=1.14.3` |
| `aws` | `boto3~=1.40.38`, `aiobotocore~=2.25.2` |
| `bedrock` | `boto3~=1.40.45` |
| `google-genai` | `google-genai~=1.65.0` |
| `anthropic` | `anthropic~=0.73.0` |
| `azure-ai-inference` | `azure-ai-inference~=1.0.0b9` |
| `litellm` | `litellm>=1.74.9,<=1.82.6` |
| `voyageai` | `voyageai~=0.3.5` |
| `watson` | `ibm-watsonx-ai~=1.3.39` |

### crewai-tools Extras
25+ optional extras including: `sqlalchemy`, `selenium`, `firecrawl-py`, `composio-core`, `browserbase`, `weaviate-client`, `snowflake`, `couchbase`, `mcp`, `stagehand`, `github`, `rag`, `xml`, and more.

---

## Development Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| ruff | 0.15.1 | Linting & formatting |
| mypy | 1.19.1 | Type checking |
| pytest | 8.4.2 | Testing |
| pytest-asyncio | 1.3.0 | Async test support |
| pytest-xdist | 3.8.0 | Parallel test execution |
| pytest-split | 0.10.0 | Test duration-based splitting |
| pre-commit | 4.5.1 | Git hooks |
| bandit | 1.9.2 | Security scanning |
| commitizen | >=4.13.9 | Conventional commits |

---

## CI/CD Pipeline (GitHub Actions)

### Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `tests.yml` | PR | Run pytest across Python 3.10-3.13 (8 parallel groups) |
| `linter.yml` | PR | ruff check + ruff format --check |
| `type-checker.yml` | PR | mypy on all Python versions |
| `build-uv-cache.yml` | Push to main, schedule | Pre-build UV cache every 5 days |
| `publish.yml` | Manual dispatch | Build and publish to PyPI |
| `codeql.yml` | Schedule | GitHub security analysis |
| `docs-broken-links.yml` | Schedule | Check documentation links |
| `nightly.yml` | Schedule | Nightly test runs |
| `pr-size.yml` | PR | Report PR size metrics |
| `pr-title.yml` | PR | Enforce PR title format |
| `stale.yml` | Schedule | Mark stale issues/PRs |

### Test Execution Strategy
- **Parallelization**: 8 groups via `pytest-split`
- **Timeout**: 15 minutes per job
- **Cache**: UV global cache restored/saved across runs
- **Test Paths**: `lib/crewai/tests`, `lib/crewai-tools/tests`, `lib/crewai-files/tests`

---

## Docker

**Single Dockerfile** at:
```
lib/crewai-tools/src/crewai_tools/tools/code_interpreter_tool/Dockerfile
```

```dockerfile
FROM python:3.12-alpine
RUN pip install requests beautifulsoup4
WORKDIR /workspace
```

Minimal image for the code interpreter tool (sandboxed Python execution).

---

## Code Quality Tools

### Ruff (Linting & Formatting)
- **Target**: Python 3.10+
- **Extensions**: E, F, B, S, RUF, N, W, I, T, PERF, PIE, TID, ASYNC, RET, UP*
- **Per-file ignores**: Test files allow `S101` (assert), hardcoded passwords, etc.
- **Configured for**: `lib/*`

### MyPy (Type Checking)
- **Strict mode** enabled
- **Plugin**: `pydantic.mypy`
- **Python version**: 3.12 (for type checking)

### pytest Configuration
- **Mode**: `strict` asyncio
- **Fixtures**: `function` loop scope
- **Options**: `-n auto --timeout=60 --dist=loadfile`

---

## Dependency Management Notes

### uv Override Dependencies
```toml
[tool.uv]
override-dependencies = [
    "rich>=13.7.1",
    "onnxruntime<1.24; python_version < '3.11'",
    "pillow>=12.1.1",
    "langchain-core>=1.2.11,<2",
    "urllib3>=2.6.3",
]
```

### Special Index Handling
PyTorch uses custom indices for Python 3.13 compatibility:
```toml
[[tool.uv.index]]
name = "pytorch-nightly"
url = "https://download.pytorch.org/whl/nightly/cpu"

[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cpu"
```

### Dependabot
Weekly UV dependency updates with security grouping.

---

## Key Libraries Summary

| Category | Primary Choice |
|----------|---------------|
| LLM Integration | OpenAI SDK + Instructor |
| Vector Stores | ChromaDB, LanceDB |
| Type System | Pydantic v2 |
| CLI Framework | Click |
| TUI | Textual |
| Async HTTP | httpx |
| Telemetry | OpenTelemetry |
| Testing | pytest + pytest-asyncio + pytest-xdist |
| Linting | Ruff |
| Type Checking | MyPy |
| Package Manager | uv |
| Build System | Hatchling |
| CI/CD | GitHub Actions |
