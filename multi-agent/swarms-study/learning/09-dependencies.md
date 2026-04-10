# Dependencies

## Core Dependencies

### AI/LLM Integration

| Dependency | Version | Purpose |
|------------|---------|---------|
| litellm | 1.76.1 | Unified LLM API interface for 50+ providers |
| pydantic | * | Data validation and settings management |
| mcp | * | Model Context Protocol integration |

### Async Runtime

| Dependency | Version | Purpose |
|------------|---------|---------|
| asyncio | >=3.4.3,<5.0 | Async/await concurrency |
| aiofiles | * | Async file I/O |
| aiohttp | * | Async HTTP client/server |
| httpx | * | Sync/async HTTP client |
| uvloop | * (linux/darwin) | Fast asyncio event loop |
| winloop | * (windows) | Fast asyncio event loop for Windows |

### Data Processing & Utilities

| Dependency | Version | Purpose |
|------------|---------|---------|
| pypdf | 5.1.0 | PDF text extraction |
| numpy | * | Numerical computing |
| networkx | * | Graph data structures for agent interactions |
| psutil | * | System/resource monitoring |

### Logging & Output

| Dependency | Version | Purpose |
|------------|---------|---------|
| loguru | * | Structured logging |
| rich | * | Terminal formatting/colors |

### Reliability Patterns

| Dependency | Version | Purpose |
|------------|---------|---------|
| tenacity | * | Retry logic |
| ratelimit | 2.2.1 | API rate limiting |

### Configuration & Environment

| Dependency | Version | Purpose |
|------------|---------|---------|
| python-dotenv | * | .env file loading |
| PyYAML | * | YAML config parsing |
| toml | * | TOML parsing |

### CLI & Scheduling

| Dependency | Version | Purpose |
|------------|---------|---------|
| schedule | * | Job scheduling |

## Development Dependencies

### Lint Group

| Dependency | Version |
|------------|---------|
| black | >=23.1,<27.0 |
| ruff | >=0.5.1,<0.15.8 |
| types-toml | ^0.10.8.1 |
| types-pytz | >=2023.3,<2027.0 |
| types-chardet | ^5.0.4.6 |
| mypy-protobuf | >=3,<6 |

### Test Group

| Dependency | Version |
|------------|---------|
| pytest | >=8.1.1,<10.0.0 |

### Dev Group

| Dependency | Version |
|------------|---------|
| black | * |
| ruff | * |
| pytest | * |

## Key Dependency Insights

### LiteLLM (1.76.1)
The pinned litellm version provides a unified interface to multiple LLM providers, abstracting away provider-specific API differences.

### Platform-Specific Async
The project uses uvloop on Linux/Darwin and winloop on Windows for optimized asyncio performance.

### No Major Version Constraints
Most dependencies use wildcards (*), allowing automatic updates while maintaining compatibility via poetry lock file.
