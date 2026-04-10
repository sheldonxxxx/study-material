# Code Quality: Swarms

## Quality Practices

### Code Formatting

The project uses **Black** for code formatting with the following configuration:
- Target Python version: 3.8+
- Line length: 70 characters
- Excludes standard directories (.git, .venv, build, dist, docs)

```toml
[tool.black]
target-version = ["py38"]
line-length = 70
```

### Linting

**Ruff** is used for linting with line-length set to 70 characters:
```toml
[tool.ruff]
line-length = 70
```

### Type Safety

The project uses:
- **Pydantic** for runtime data validation and settings management
- **mypy-protobuf** for type checking protobuf definitions
- Type hints throughout the codebase for static analysis

### Logging

**Loguru** is the standard logging solution, providing:
- Structured logging with context
- Easy configuration via environment variables
- Default global logger available via `loguru.logger`

## Testing

### Framework

**pytest** is used as the testing framework:
```
[tool.poetry.group.test.dependencies]
pytest = ">=8.1.1,<10.0.0"
```

### Test Organization

Tests are organized in `/tests/` directory:
- `test___init__.py`: Package initialization tests
- `test_main_features.py`: Core functionality tests
- `benchmarks/`: Performance benchmarks
- `structs/`: Workflow struct tests
- `tools/`: Tool integration tests
- `utils/`: Utility function tests

### Running Tests

```bash
pytest tests/
```

## Error Handling

### Retry Logic

**Tenacity** is used for retry logic with exponential backoff:
```python
from tenacity import retry, stop_after_attempt, wait_exponential
```

### Pydantic Models

Data validation uses Pydantic models throughout:
- Settings/configuration via `pydantic.BaseSettings`
- Request/response schemas in `swarms/schemas/`
- Agent configuration validation

## Best Practices Observed

1. **Environment-based Configuration**: API keys and settings via environment variables
2. **Async-First Design**: Async/await patterns with aiohttp, aiofiles
3. **Modular Structure**: Clear separation between agents, structs, tools, utils
4. **Protocol Support**: Extensible architecture for MCP, X402, AOP
5. **No Telemetry**: Privacy-first with `SWARMS_TELEMETRY_ON=false` default

## Code Style Guidelines

- 70 character line limit (enforced by Black and Ruff)
- Type hints required for function signatures
- Docstrings for public APIs
- Pydantic models for all configuration and data transfer objects
- Use of `loguru.logger` instead of standard logging
