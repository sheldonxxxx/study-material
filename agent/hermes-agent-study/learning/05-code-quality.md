# Code Quality: Hermes Agent

## Type System

### Language Level
- **Python >= 3.11** enforced in `pyproject.toml`
- `from __future__ import annotations` used for forward references

### Primary Type System: Dataclasses

The codebase uses **Python dataclasses** as the primary type system, NOT Pydantic models:

```python
# gateway/platforms/base.py
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
    raw_message: Any = None
    media_urls: List[str] = field(default_factory=list)
```

**Evidence:**
- `gateway/session.py`: Uses `@dataclass` for `SessionSource`
- `gateway/config.py`: Uses `@dataclass` for `HomeChannel`, `SessionResetPolicy`
- `gateway/platforms/base.py`: Uses `@dataclass` for `MessageEvent`, `SendResult`
- `agent/smart_model_routing.py`: Uses `Dict`, `Optional`, `Any` type hints

### Pydantic Usage (Limited)
Pydantic is declared as a dependency but used primarily in:
- **Environment modules** (`environments/hermes_base_env.py`, `environments/agentic_opd_env.py`)
- **Documentation examples** (skills references)
- **NOT in core gateway, agent, or tools code**

### Typing Patterns
- `typing.Dict`, `typing.List`, `typing.Optional`, `typing.Any` (not shorthand)
- `typing.Callable`, `typing.Awaitable` for callbacks
- `enum.Enum` for discriminated unions (e.g., `Platform`, `MessageType`)
- `pathlib.Path` for file paths
- Custom type aliases: `MessageHandler = Callable[[MessageEvent], Awaitable[Optional[str]]]`

---

## Testing

### Test Framework
- **pytest** with plugins: `pytest-asyncio`, `pytest-xdist`
- Default run excludes integration tests: `pytest -m 'not integration' -n auto`

### Test Structure
**~100+ test files** across organized directories:
- `tests/acp/` - ACP protocol tests (6 files)
- `tests/agent/` - Agent logic tests (9 files)
- `tests/gateway/` - Gateway/platform tests (80+ files)
- `tests/tools/` - Tool-specific tests (60+ files)
- `tests/hermes_cli/` - CLI tests
- `tests/cron/` - Cron job tests

### Test Infrastructure
Key fixtures in `tests/conftest.py`:

```python
@pytest.fixture(autouse=True)
def _isolate_hermes_home(tmp_path, monkeypatch):
    """Redirect HERMES_HOME to a temp dir so tests never write to ~/.hermes/."""

@pytest.fixture(autouse=True)
def _enforce_test_timeout():
    """Kill any individual test that takes longer than 30 seconds."""
    signal.alarm(30)
```

**Test isolation practices:**
- Automatic `HERMES_HOME` redirection to temp directory
- Plugin singleton reset between tests
- Session environment variable cleanup
- Global 30-second timeout per test (Unix SIGALRM)
- Event loop management for async tests

### Test Patterns
```python
# Example from tests/tools/test_approval.py
class TestDetectDangerousRm:
    def test_rm_rf_detected(self):
        is_dangerous, key, desc = detect_dangerous_command("rm -rf /home/user")
        assert is_dangerous is True
        assert key is not None
        assert "delete" in desc.lower()
```

**Mock usage:**
- `unittest.mock.patch` for config and module mocking
- `monkeypatch` from pytest for environment manipulation

### Coverage Areas
- Security patterns (`test_approval.py` - dangerous command detection)
- Model routing logic (`test_smart_model_routing.py`)
- Platform adapters (Discord, Telegram, WhatsApp)
- Session management, Tool registries
- PII redaction (`test_pii_redaction.py`)
- Secret redaction (`agent/redact.py`)

---

## Linting and Formatting

### Current State: None Configured

**Missing tooling:**
- **No Ruff** - Modern Python linter/formatter not configured
- **No Black** - Code formatter not configured
- **No MyPy** - Static type checker not configured
- **No pre-commit hooks** - No automated formatting

**Only build tooling present:**
```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

**CI Testing:**
```yaml
# .github/workflows/tests.yml
- name: Run tests
  run: uv pip install -e ".[dev]" && pytest tests/ -m 'not integration'
```

**Impact:** Code style is not enforced automatically; relies on developer discipline.

---

## Error Handling

### Standard Pattern
```python
try:
    from gateway.status import write_runtime_status
    write_runtime_status(platform=self.platform.value, ...)
except Exception:
    pass  # Silent failure - don't let status write crash the app
```

### Try/Except with Specific Fallbacks
```python
try:
    runtime = resolve_runtime_provider(...)
except Exception:
    return {"model": primary.get("model"), ...}  # Fallback on error
```

### Patterns
- **Graceful degradation:** Platform adapters catch exceptions and set fatal error state
- **Fallback behavior:** Return default values on failure
- **Silent failures:** Some error cases use `pass` instead of logging

### Logging Pattern
```python
logger = logging.getLogger(__name__)

# Usage
logger.info("[%s] Sending response (%d chars)", self.name, len(text_content))
logger.warning("[%s] Auto-TTS failed: %s", self.name, tts_err)
logger.error("[%s] Failed to send image: %s", self.name, img_result.error)
```

### Anti-patterns Observed
```python
# Uses print() instead of logger.error() for some errors
# Uses traceback.print_exc() instead of logger.exception()
except Exception as e:
    print(f"[{self.name}] Error handling message: {e}")
    import traceback
    traceback.print_exc()
```

---

## Code Organization

### Module Structure
```
hermes-agent/
├── agent/              # Core agent logic (prompt building, model routing)
├── gateway/            # Messaging gateway (multi-platform support)
│   └── platforms/      # Platform adapters (Discord, Telegram, WhatsApp, etc.)
├── tools/              # Tool implementations (~50 modules)
│   └── browser_providers/
├── hermes_cli/        # CLI implementation (~35 modules)
├── skills/             # Bundled skills (manifests + scripts)
├── environments/       # Benchmark/evaluation environments
├── acp_adapter/        # Agent Client Protocol
├── cron/               # Cron job integration
└── tests/              # Comprehensive test suite
```

### Platform Adapter Pattern
```python
# gateway/platforms/base.py
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool: ...

    @abstractmethod
    async def disconnect(self) -> None: ...

    @abstractmethod
    async def send(self, chat_id: str, content: str, ...) -> SendResult: ...
```

### Configuration Pattern
- YAML-based configuration (`~/.hermes/config.yaml`)
- Python `dataclasses` for config objects
- Environment variable overrides via `python-dotenv`

---

## Quality Summary

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| **Type System** | Type hints + dataclasses (not Pydantic) | `gateway/platforms/base.py`, `gateway/config.py` |
| **Type Coverage** | Moderate - core modules typed, some `Any` usage | `agent/smart_model_routing.py` |
| **Test Framework** | pytest with pytest-asyncio, pytest-xdist | `pyproject.toml`, `tests/conftest.py` |
| **Test Count** | ~100+ test files, comprehensive | `tests/gateway/`, `tests/tools/` |
| **Test Isolation** | Excellent - HERMES_HOME isolation, timeouts | `tests/conftest.py` |
| **Linting** | None configured | No ruff.toml, no pre-commit |
| **Formatting** | None configured | No Black, no Ruff |
| **Error Handling** | Try/except with graceful degradation | `gateway/platforms/base.py` |
| **Logging** | Structured logging throughout | `logger = logging.getLogger(__name__)` |
| **Code Organization** | Excellent separation of concerns | Modular structure |
| **Platform Pattern** | Base adapter pattern | `BasePlatformAdapter` ABC |

---

## Recommendations

1. **Add Ruff** for linting and formatting (replaces isort, flake8, black)
2. **Add MyPy** for static type checking
3. **Add pre-commit hooks** for automated formatting
4. **Standardize exception handling** - use `logger.exception()` instead of `traceback.print_exc()`
5. **Replace `print()` calls** with `logger` calls in production code
6. **Reduce `Any` type usage** with more specific types where possible
