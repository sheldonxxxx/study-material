# CoPaw Code Quality Analysis

## Type System Usage

### Python Type Hints

The Python backend uses **comprehensive type hints** with PEP 563 postponed evaluation (`from __future__ import annotations`).

**Evidence:**
- `src/copaw/config/timezone.py`: `Optional[str]`, `-> str`, `Tuple[str, ...]`
- `src/copaw/providers/openai_provider.py`: `List[ModelInfo]`, `tuple[bool, str]`, `TYPE_CHECKING` imports
- `src/copaw/config/utils.py`: `Tuple[str, str, Optional[str]]`, `Tuple[Optional[str], Optional[str]]`

**Patterns observed:**
```python
from typing import TYPE_CHECKING, Any, List, Optional, Tuple

def _probe_python() -> Optional[str]: ...
def get_available_channels() -> Tuple[str, ...]: ...
async def check_connection(self, timeout: float = 5) -> tuple[bool, str]:
```

### TypeScript

The console (React frontend) uses **full TypeScript with strict mode**.

**Configuration** (`console/tsconfig.app.json`):
```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

**Evidence from `useAgentConfig.tsx`:**
```typescript
import type { AgentsRunningConfig } from "../../../api/types";
const [loading, setLoading] = useState<boolean>(true);
const [error, setError] = useState<string | null>(null);
const fetchConfig = useCallback(async () => { ... }
```

**TypeScript ESLint config** (`console/eslint.config.js`):
- Uses `typescript-eslint` with recommended rules
- React Hooks rules enforced (`eslint-plugin-react-hooks`)
- React Refresh rules for HMR safety

---

## Testing Approach

### Framework and Tools

| Tool | Version | Purpose |
|------|---------|---------|
| pytest | >=8.3.5 | Test runner |
| pytest-asyncio | >=0.23.0 | Async test support |
| pytest-cov | >=6.2.1 | Coverage reporting |
| hypothesis | >=6.0.0 | Property-based testing |

### Test Structure

```
tests/
  unit/
    agents/
    channels/
      test_qq_channel.py
    cli/
      test_cli_shutdown.py
      test_cli_update.py
      test_cli_version.py
    memory/
      test_copaw_token_counter.py
    providers/
      test_anthropic_provider.py
      test_default_provider.py
      test_gemini_provider.py
      test_kimi_provider.py
      test_ollama_provider.py
      test_ollama_manager_timeout.py
      test_openai_provider.py
      test_openai_stream_toolcall_compat.py
      test_provider_manager.py
    workspace/
      test_agent_creation.py
      test_agent_id.py
      test_agent_model.py
      test_cli_agent_id.py
      test_prompt.py
      test_workspace.py
  integrated/
    test_app_startup.py
    test_version.py
```

### pytest Configuration (`pyproject.toml`)

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
]
```

### Testing Patterns

**Async testing** (from `test_openai_provider.py`):
```python
async def test_check_connection_success(monkeypatch) -> None:
    provider = _make_provider()
    # ... async test with monkeypatch mocking
    ok, msg = await provider.check_connection(timeout=2.5)
    assert ok is True
```

**Subprocess/integration testing** (from `test_app_startup.py`):
```python
def test_app_startup_and_console() -> None:
    process = subprocess.Popen([sys.executable, "-m", "copaw", "app", ...])
    # Thread reads stdout, polling for backend readiness
    with httpx.Client(timeout=5.0) as client:
        response = client.get(f"http://{host}:{port}/api/version")
```

**Mocking with monkeypatch**:
```python
monkeypatch.setattr(provider, "_client", lambda timeout=5: fake_client)
```

**CI Coverage reporting**: Uses `orgoro/coverage@v3` to post coverage comments on PRs.

---

## Linting and Formatting Tooling

### Python

| Tool | Config | Key Settings |
|------|--------|--------------|
| **Black** | `.flake8` + pre-commit | `line-length = 79` |
| **Flake8** | `.flake8` | Ignores F401, F403, W503, E731 |
| **Pylint** | pre-commit | Extensive disables for rapid dev |
| **MyPy** | pre-commit | `--ignore-missing-imports`, `--follow-imports=skip` |
| **Pre-commit** | `.pre-commit-config.yaml` | 10+ hooks including mypy, black, flake8, pylint |

### TypeScript/JavaScript

| Tool | Config | Purpose |
|------|--------|---------|
| **ESLint** | `console/eslint.config.js` | typescript-eslint, react-hooks |
| **Prettier** | pre-commit | Formats `.tsx?` files |

### Pre-commit Hooks (`.pre-commit-config.yaml`)

1. `check-ast` - Python syntax validation
2. `sort-simple-yaml` - YAML sorting
3. `check-yaml`, `check-toml`, `check-json`, `check-xml` - Config validation
4. `check-docstring-first` - Docstring placement
5. `detect-private-key` - Secret detection
6. `add-trailing-comma` - Trailing comma enforcement
7. `mypy` - Type checking (skips skills, pb2.py, grpc.py)
8. `black` - Code formatting (line-length: 79)
9. `flake8` - Linting (ignores E203)
10. `pylint` - Advanced linting (30+ rules disabled)
11. `prettier` - JS/TS formatting

---

## Error Handling Patterns

### Python Patterns

**1. Tuple-based error returns for connection checks:**
```python
async def check_connection(self, timeout: float = 5) -> tuple[bool, str]:
    try:
        await client.models.list(timeout=timeout)
        return True, ""
    except APIError:
        return False, f"API error when connecting to `{self.base_url}`"
    except Exception:
        return False, f"Unknown exception when connecting to `{self.base_url}`"
```

**2. Specific exception types caught:**
```python
try:
    await client.models.list(timeout=timeout)
except APIError:
    return False, f"API error..."
except (json.JSONDecodeError, ValueError):
    return None
except OSError as exc:
    logger.warning("Failed to read %s: %s", path, exc)
```

**3. Best-effort error handling with logging:**
```python
def _chmod_best_effort(path: Path, mode: int) -> None:
    try:
        os.chmod(path, mode)
    except OSError:
        # Some systems/filesystems may not support chmod semantics.
        pass
```

**4. Graceful degradation:**
```python
try:
    result = subprocess.run(...)
except (FileNotFoundError, subprocess.TimeoutExpired):
    logger.debug("Telemetry upload failed: %s", e)
```

---

## Logging Approach

### Python Standard Library Logging

**Pattern**: `logging.getLogger(__name__)` per module.

**Evidence from grep** (20+ modules use this pattern):
```python
logger = logging.getLogger(__name__)

logger.error("envs.json path exists but is not a regular file: %s", path)
logger.warning("Failed to read %s: %s", path, exc)
logger.info("Downloading binary...")
logger.debug("Telemetry upload failed: %s", e)
```

**Log level control**: App startup accepts `--log-level info` argument.

### No Third-party Logging Libraries

Uses only Python standard library `logging` module throughout.

---

## Config and Secrets Management

### Environment Variable Storage

**Two-layer strategy** (from `src/copaw/envs/store.py`):

1. **envs.json** - Canonical store, survives process restarts
2. **os.environ** - Injected into current Python process

```python
_BOOTSTRAP_WORKING_DIR = Path(os.environ.get("COPAW_WORKING_DIR", "~/.copaw"))
_BOOTSTRAP_SECRET_DIR = Path(os.environ.get("COPAW_SECRET_DIR", f"{_BOOTSTRAP_WORKING_DIR}.secret"))
_ENVS_JSON = _BOOTSTRAP_SECRET_DIR / "envs.json"
```

### Secrets Handling

**Secrets stored in `COPAW_SECRET_DIR`** (defaults to `~/.copaw.secret`):
- Permissions set to `0o700` (user-only read/write/execute)
- `chmod_best_effort` for filesystems that may not support it

**Secret fields masking**:
```python
# From cli/channels_cmd.py
_SECRET_FIELDS = {
    # Fields like api_key, auth_token are masked in output
}

# Provider info masks secrets
assert info.api_key == "sk-******"  # Secret masked in get_info()
```

**Env var configuration**:
```python
from dotenv import load_dotenv
load_dotenv()  # Standard python-dotenv usage
```

### No Hardcoded Secrets

- API keys loaded from environment or config
- Credentials prompted via CLI when needed (`click.prompt`)
- GitHub tokens from environment variables

---

## Overall Quality Assessment

### Strengths

| Area | Assessment |
|------|------------|
| **Type Safety** | Strong - Python type hints throughout, TypeScript strict mode |
| **Test Coverage** | Good - Unit + integrated tests, async support, coverage reporting |
| **Code Style** | Excellent - Black formatting, ESLint, pre-commit enforcement |
| **Type Checking** | Good - MyPy in CI, type annotations standard practice |
| **Error Handling** | Good - Specific exception types, graceful degradation |
| **Secrets** | Good - Separate secret directory, permissions, masking |
| **CI/CD** | Excellent - Multi-platform testing, coverage reports, maintainer approval gate |
| **Documentation** | Good - CONTRIBUTING guide, PR template, Conventional Commits |

### Areas for Improvement

| Area | Observation |
|------|-------------|
| **Pylint Disabled Rules** | 30+ rules disabled in pre-commit may hide issues |
| **MyPy Strictness** | `--ignore-missing-imports` and `--follow-imports=skip` may miss type errors |
| **Async Test Patterns** | Some complexity in monkeypatching async methods |
| **Skills Directory** | Excluded from many linting/type checks - may have lower quality |
| **Front-end Testing** | No visible frontend tests in repository |

### Code Review Process

1. **PR Template** requires:
   - Description and related issues
   - Security considerations
   - Type of change and components affected
   - Checklist: pre-commit passes, pytest passes, docs updated

2. **Required local gate**:
   ```bash
   pre-commit run --all-files
   pytest
   ```

3. **CI enforcement**:
   - Maintainer approval gate
   - Unit tests on Ubuntu, macOS, Windows (Python 3.10, 3.13)
   - Integrated tests on same platforms
   - Coverage report posted on PR

### Security Signals

- Secrets in separate directory with restricted permissions
- No secrets in logs
- Secret fields masked in provider info
- Pre-commit `detect-private-key` hook
- SECURITY.md policy document exists
