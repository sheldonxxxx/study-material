# My Action Items for Building an AI Agent Framework

**Derived from:** crewAI codebase study
**Priority Framework:**
- P0: Security-critical (do first)
- P1: Architecture-quality (do second)
- P2: Code quality (do third)

---

## P0: Security (Do First)

### P0-1: Implement Parameterized Queries from Day One

**Why:** crewAI has a SQL injection vulnerability in their NL2SQL tool using f-string interpolation.

**Action:** Establish a database helper that enforces parameterized queries:

```python
# core/database.py
def execute_query(query: str, params: dict | None = None) -> list[dict]:
    """All database queries must use this helper or parameterized alternatives."""
    if not params:
        raise ValueError("All queries must use parameters, not string interpolation")
    # Implementation using SQLAlchemy text()
```

**Verification:** Add bandit rule S608 to CI; fail if any SQL string contains f-string or % formatting.

---

### P0-2: Use Docker for Any Code Execution, Not Python Sandboxing

**Why:** crewAI shipped an "INSECURE" sandbox that doesn't actually isolate.

**Action:** For any code execution feature:
- Use Docker containers with minimal images
- Set resource limits (CPU, memory, time)
- Never trust Python's `exec()`, `eval()`, or restricted `__import__`
- Document the isolation mechanism clearly

**Verification:** Code execution features require security review and Docker verification tests.

---

### P0-3: Never Ship Stub Security Features

**Why:** crewAI's HallucinationGuardrail is a no-op that logs a warning. Users may enable it believing it works.

**Action:**
- If a feature requires a paid API key, raise `ImportError` or `ConfigurationError` at initialization
- Never return `True` silently for disabled security features
- Add a `PREMIUM_FEATURE` annotation and CI check that prevents stubs from shipping

**Verification:** Audit all security-adjacent code for silent no-op behavior.

---

### P0-4: Audit Token Storage for CLI Authentication

**Why:** crewAI uses JWT tokens via TokenManager but storage mechanism wasn't reviewed.

**Action:**
- Verify tokens are stored in system keyring (keyring library), not plaintext files
- Document the storage mechanism
- Add automated test that storage is not world-readable

**Verification:** Security review of TokenManager and token storage paths.

---

## P1: Architecture (Do Second)

### P1-1: Enforce File Size Limits in CI

**Why:** crewAI has 129KB files that are unmaintainable. `flow.py` (3257 lines) should have been 5+ files.

**Action:** Add to CI:

```yaml
# .github/workflows/lint.yml
- name: Check file sizes
  run: |
    find lib -name "*.py" -exec wc -l {} \; | awk '$1 > 500 {print $2 " has " $1 " lines"}'
    if [ $? -ne 0 ]; then exit 1; fi
```

**Verification:** PR fails if any Python file exceeds 500 lines.

---

### P1-2: Use Protocol Instead of ABC for Plugin Systems

**Why:** ABC with `@abstractmethod` is rigid; duck typing is more flexible for testing.

**Action:** For LLM providers, tools, storage backends:

```python
# core/interfaces.py
@runtime_checkable
class LLMProvider(Protocol):
    def call(self, messages: list[dict], **kwargs) -> str: ...
    async def acall(self, messages: list[dict], **kwargs) -> str: ...

# Remove ABC inheritance from BaseLLM
# Allow any object with the right methods
```

**Verification:** Write tests that mock LLMProvider without inheritance.

---

### P1-3: Centralize Error Handling with Specific Exceptions

**Why:** crewAI has 30+ files with bare `except Exception:` that catch too much.

**Action:** Create `core/exceptions.py`:

```python
# Specific exceptions
class ConfigurationError(Exception): pass
class ToolExecutionError(Exception): pass
class LLMCallError(Exception): pass
class MemoryError(Exception): pass

# Decorator for common patterns
def graceful_degradation(default=None, log_level="error"):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            try:
                return fn(*args, **kwargs)
            except (MemoryError, KeyboardInterrupt):
                raise  # Don't catch critical exceptions
            except Exception as e:
                logger.log(log_level, f"{fn.__name__} failed: {e}")
                return default
        return wrapper
    return decorator
```

**Verification:** bandit S112 warnings fail CI; all files use specific exception types.

---

### P1-4: Implement Feature Flags for Enterprise vs Open Source

**Why:** crewAI mixes core and enterprise features (HallucinationGuardrail no-op).

**Action:**

```python
# core/features.py
from enum import Enum

class Feature(Enum):
    HALLUCINATION_DETECTION = "hallucination_detection"
    TRACE_COLLECTION = "trace_collection"

ENABLED_FEATURES: set[Feature] = set()

def require_feature(feature: Feature):
    """Decorator that raises if feature is not enabled."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            if feature not in ENABLED_FEATURES:
                raise EnterpriseFeatureError(f"{feature.value} requires enterprise license")
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

**Verification:** Enterprise features raise `EnterpriseFeatureError` in open-source builds.

---

### P1-5: Automate Model Metadata Discovery

**Why:** Hardcoded context window sizes require manual maintenance.

**Action:** Choose one:
1. Fetch from provider APIs on first use (cache results)
2. Use a maintained third-party model metadata library
3. Store in versioned JSON, update via CI script

**Verification:** Context window for any model not in dictionary falls back to API fetch, not a hardcoded guess.

---

## P2: Code Quality (Do Third)

### P2-1: Add Pre-Commit Configuration Immediately

**Why:** crewAI has no `.pre-commit-config.yaml`, letting issues reach CI.

**Action:** Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.1
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mypy
    rev: v1.19.1
    hooks:
      - id: mypy
        additional_dependencies: [pydantic>=2.0]
  - repo: https://github.com/pycqa/bandit
    rev: v1.9.2
    hooks:
      - id: bandit
        args: [-r]
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v4.13.9
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

**Verification:** `pre-commit run --all-files` passes before any PR.

---

### P2-2: Enforce Test Coverage Floors for All Packages

**Why:** crewAI's `crewai-files` has 0.08 coverage ratio.

**Action:** Add to CI:

```yaml
# pytest.ini or pyproject.toml
[tool.coverage.report]
fail_under = 60  # Minimum 60% coverage
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]

# Per-package enforcement
[tool.coverage.pyramid]
"crewai/*" = {fail_under = 80}
"crewai-tools/*" = {fail_under = 70}
"crewai-files/*" = {fail_under = 50}
```

**Verification:** Coverage report fails CI if below threshold.

---

### P2-3: Replace Global stdout Modification with Structured Logging

**Why:** crewAI's `FilteredStream` modifies `sys.stdout` globally to suppress LiteLLM banners.

**Action:**
- Use a logger for library output, not stdout redirection
- Configure LiteLLM's logging directly if available
- Never modify global streams

**Verification:** Search for `sys.stdout` modifications; none should exist.

---

### P2-4: Add Async-first Testing Patterns

**Why:** crewAI has separate sync and async code paths with duplicate logic.

**Action:**
- Default to async implementations
- Use `pytest-asyncio` with `async def` test functions
- Extract common logic into shared helpers used by both paths

**Verification:** grep for `_invoke.*_` patterns; if sync and async versions exist, they should share implementation.

---

### P2-5: Document Telemetry Privacy Clearly

**Why:** crewAI documents telemetry well but the complexity (3 disable env vars, OTEL vs Plus API) could confuse users.

**Action:**
- Single environment variable: `CREWAI_TELEMETRY_ENABLED=false`
- Document in a single `TELEMETRY.md`
- Clearly state what is sent, what is hashed, what requires opt-in

**Verification:** `CREWAI_TELEMETRY_ENABLED=false` disables all telemetry without requiring 3 different vars.

---

## Summary: Prioritized Execution Order

| Priority | Item | Time Estimate |
|----------|------|---------------|
| P0-1 | Parameterized queries | 1 day |
| P0-2 | Docker sandboxing | 1 week |
| P0-3 | No stub security features | 2 days |
| P0-4 | Token storage audit | 1 day |
| P1-1 | File size limits | 1 day |
| P1-2 | Protocol over ABC | 3 days |
| P1-3 | Centralized errors | 2 days |
| P1-4 | Feature flags | 2 days |
| P1-5 | Auto model metadata | 3 days |
| P2-1 | Pre-commit | 1 day |
| P2-2 | Coverage floors | 1 day |
| P2-3 | No stdout modification | 1 day |
| P2-4 | Async-first testing | 3 days |
| P2-5 | Telemetry docs | 1 day |

**Total:** ~3 weeks for a solo developer

The first week should focus entirely on P0 (Security) and P1-1 (File size limits), as these prevent the most damage and are foundational to code quality.
