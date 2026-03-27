# Tech Stack & Architecture

## Python Version

| Setting | Value |
|---------|-------|
| Required Version | `>=3.10,<4.0` |
| Docker Base Image | `python:3.11-slim-bullseye` |
| CI Test Matrix | Python 3.10, 3.11, 3.12 |

The project targets Python 3.10+ with Docker deployments using 3.11 for stability.

## Build System

| Component | Choice |
|-----------|--------|
| Build Backend | `poetry-core>=1.0.0` |
| Package Manager | Poetry |
| Config File | `pyproject.toml` |

## Core Dependencies

### AI/LLM Integration
- **litellm** (1.76.1) - Unified LLM API interface for 50+ providers
- **pydantic** - Data validation and settings management
- **mcp** - Model Context Protocol integration

### Async Runtime
- **asyncio** (>=3.4.3,<5.0) - Async/await concurrency
- **aiofiles**, **aiohttp**, **httpx** - Async I/O operations
- **uvloop** (Linux/Mac) / **winloop** (Windows) - Fast asyncio event loop

### Data Processing
- **numpy** - Numerical computing
- **networkx** - Graph data structures for agent interactions
- **pypdf** (5.1.0) - PDF text extraction

### Reliability Patterns
- **tenacity** - Retry logic
- **ratelimit** (2.2.1) - API rate limiting

### CLI & Output
- **loguru** - Structured logging
- **rich** - Terminal formatting/colors
- **schedule** - Job scheduling

### Configuration
- **python-dotenv** - .env file loading
- **PyYAML**, **toml** - Config file parsing

## Project Structure

```
swarms/
├── agents/          # Multi-agent implementations (46 files)
├── artifacts/       # Output/artifacts handling
├── cli/             # Command-line interface
├── prompts/         # Prompt templates (69 files)
├── schemas/         # Data schemas (13 files)
├── sims/            # Simulation modules
├── structs/         # Core data structures (65 files)
├── telemetry/       # Bootup and telemetry
├── tools/           # Tool integrations (20 files)
└── utils/           # Utility functions (29 files)
```

## Code Quality Tools

| Tool | Purpose |
|------|---------|
| **ruff** | Fast linting (line-length: 70) |
| **black** | Code formatting (target-version: py38, line-length: 70) |
| **pytest** | Testing (>=8.1.1,<10.0.0) |
| **mypy-protobuf** | Type checking |

## Architectural Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Package Manager | Poetry | Modern, lockfile-based dependency management |
| Python Constraint | 3.10 - 4.0 | Modern async syntax, broad compatibility |
| Async Runtime | asyncio + uvloop | Performance on Linux/Mac, Windows fallback |
| LLM Abstraction | litellm | Unified API for 50+ LLM providers |
| Data Validation | pydantic | Type safety, settings management |
| Graph Structures | networkx | Agent interaction patterns |
| Logging | loguru | Simpler API than standard logging module |
| CLI Output | rich | Pretty terminal output |
| PDF Processing | pypdf | Lightweight PDF text extraction |
