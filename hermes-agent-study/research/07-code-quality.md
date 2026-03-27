# Code Quality Assessment: Hermes Agent

## Type System Usage and Coverage

### Language Level
- **Python >= 3.11** enforced in `pyproject.toml` (`requires-python = ">=3.11"`)
- `from __future__ import annotations` used in key modules for forward references

### Primary Type System: Type Hints + Dataclasses
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

**Evidence from `pyproject.toml`:**
```toml
dependencies = [
  "pydantic>=2.12.5,<3",
]
```

### Typing Patterns Observed
- `typing.Dict`, `typing.List`, `typing.Optional`, `typing.Any` (not `dict[]`, `list[]`)
- `typing.Callable`, `typing.Awaitable` for callbacks
- `enum.Enum` for discriminated unions (e.g., `Platform`, `MessageType`)
- `pathlib.Path` for file paths
- Custom type aliases: `MessageHandler = Callable[[MessageEvent], Awaitable[Optional[str]]]`

**File references:**
- `gateway/platforms/base.py` lines 259-335: Enum and dataclass definitions
- `agent/smart_model_routing.py` lines 1-7: Type imports
- `gateway/config.py` lines 1-20: Core config dataclasses

---

## Testing Approach and Coverage

### Test Framework
- **pytest** with plugins: `pytest-asyncio`, `pytest-xdist`
- Test configuration in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: marks tests requiring external services (API keys, Modal, etc.)",
]
addopts = "-m 'not integration' -n auto"
```

### Test Structure
- **~100+ test files** across organized directories:
  - `tests/acp/` - ACP protocol tests (6 files)
  - `tests/agent/` - Agent logic tests (9 files)
  - `tests/gateway/` - Gateway/platform tests (80+ files)
  - `tests/tools/` - Tool-specific tests (60+ files)
  - `tests/hermes_cli/` - CLI tests
  - `tests/cron/` - Cron job tests

### Test Infrastructure (conftest.py)

**Key fixtures:**
```python
# tests/conftest.py
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

**Example test structure from `tests/tools/test_approval.py`:**
```python
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

**Test categories observed:**
1. Unit tests with mocked dependencies
2. Integration tests marked with `@pytest.mark.integration`
3. Platform-specific tests (Discord, Telegram, etc.)

### Coverage Areas
- Security patterns (`test_approval.py` - dangerous command detection)
- Model routing logic (`test_smart_model_routing.py`)
- Platform adapters (Discord, Telegram, WhatsApp)
- Session management
- Tool registries
- PII redaction (`test_pii_redaction.py`)
- Secret redaction (`agent/redact.py`)

**Evidence of thorough testing:**
- `tests/tools/test_approval.py` - 515 lines with comprehensive regression tests
- `tests/gateway/` - 80+ files covering all platform adapters
- `tests/conftest.py` - 120 lines of shared infrastructure

---

## Linting and Formatting Tooling

### Python Tooling
**NO explicit linting configuration found:**
- No `ruff.toml`
- No `.pre-commit-config.yaml`
- No `pyproject.toml` `[tool.ruff]` section
- No `.flake8` configuration

**Only build/lint tooling present:**
```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

### JavaScript/TypeScript Tooling (Website)
- `website/tsconfig.json` present
- `website/package.json` for Docusaurus

### CI Testing Only
```yaml
# .github/workflows/tests.yml
- name: Run tests
  run: uv pip install -e ".[dev]" && pytest tests/ -m 'not integration'
```

### Supply Chain Security (Not Linting)
```yaml
# .github/workflows/supply-chain-audit.yml
# Scans for: .pth files, base64+exec, marshal/pickle/compile
```

### Missing Tooling
- **No Ruff** - Modern Python linter/formatter not configured
- **No Black** - Code formatter not configured
- **No MyPy** - Static type checker not configured
- **No pre-commit hooks** - No automated formatting

**Impact:** Code style is not enforced automatically; relies on developer discipline.

---

## Error Handling Patterns

### Standard Pattern
```python
# gateway/platforms/base.py lines 394-408
try:
    from gateway.status import write_runtime_status
    write_runtime_status(platform=self.platform.value, ...)
except Exception:
    pass  # Silent failure - don't let status write crash the app
```

### Try/Except with Specific Exceptions
```python
# agent/smart_model_routing.py lines 149-175
try:
    runtime = resolve_runtime_provider(...)
except Exception:
    return {"model": primary.get("model"), ...}  # Fallback on error
```

### Error Propagation Patterns
- **Graceful degradation:** Platform adapters catch exceptions and set fatal error state
- **Fallback behavior:** Return default values on failure
- **Silent failures:** Some error cases use `pass` instead of logging

### Logging Pattern
```python
# Standard pattern throughout codebase
logger = logging.getLogger(__name__)

# Usage
logger.info("[%s] Sending response (%d chars)", self.name, len(text_content))
logger.warning("[%s] Auto-TTS failed: %s", self.name, tts_err)
logger.error("[%s] Failed to send image: %s", self.name, img_result.error)
```

**Evidence files:**
- `agent/redact.py` - Uses `logging.getLogger(__name__)`
- `gateway/platforms/base.py` - Uses `logger` throughout
- `gateway/config.py` - Uses `logger`

### Exception Handling Anti-patterns Observed
```python
# gateway/platforms/base.py line 1119-1122
except Exception as e:
    print(f"[{self.name}] Error handling message: {e}")
    import traceback
    traceback.print_exc()
```
- Uses `print()` instead of `logger.error()` for some errors
- Uses `traceback.print_exc()` instead of `logger.exception()`

---

## Code Organization Patterns

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
├── honcho_integration/ # Honcho session management
└── tests/              # Comprehensive test suite
```

### Platform Adapter Pattern
All messaging platforms follow a base class pattern:

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

**Implementations:**
- `gateway/platforms/telegram.py`
- `gateway/platforms/discord.py`
- `gateway/platforms/whatsapp.py`
- `gateway/platforms/slack.py`
- `gateway/platforms/matrix.py`

### Tool Registration Pattern
Tools are registered in a registry system (`tools/__init__.py`, `tools/registry.py`).

### Session Management Pattern
```python
# gateway/session.py
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    user_id: Optional[str] = None
    thread_id: Optional[str] = None
```

### Configuration Pattern
- YAML-based configuration (`~/.hermes/config.yaml`)
- Python `dataclasses` for config objects
- Environment variable overrides via `python-dotenv`

---

## Summary Table

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| **Type System** | Type hints + dataclasses (not Pydantic) | `gateway/platforms/base.py`, `gateway/config.py` |
| **Type Coverage** | Moderate - core modules typed, some `Any` usage | `agent/smart_model_routing.py` |
| **Test Framework** | pytest with pytest-asyncio, pytest-xdist | `pyproject.toml`, `tests/conftest.py` |
| **Test Count** | ~100+ test files, comprehensive | `tests/gateway/`, `tests/tools/` |
| **Test Isolation** | Excellent - HERMES_HOME isolation, timeouts | `tests/conftest.py` lines 19-40, 108-119 |
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
