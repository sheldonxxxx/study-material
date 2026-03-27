# OpenViking Code Quality Assessment

## Overview

OpenViking is a **polyglot project** (Python/Rust/C++/TypeScript) with formalized quality tooling for Python and Rust, moderate tooling for C++, and **significant gaps** for TypeScript. The project uses modern linting (ruff), type checking (mypy), comprehensive testing (pytest), and automated PR review via PR-Agent.

---

## 1. Type Systems

### Python

| Setting | Value | Notes |
|---------|-------|-------|
| `python_version` | 3.10 | Minimum enforced |
| `disallow_untyped_defs` | `false` | Lenient - untyped functions allowed |
| `disallow_incomplete_defs` | `false` | Lenient |
| `check_untyped_defs` | `true` | Executes type checks on untyped code |
| `warn_return_any` | `false` | Suppressed |
| `no_implicit_optional` | `false` | Lenient |
| `warn_redundant_casts` | `true` | Enabled |
| `warn_unused_ignores` | `true` | Enabled |
| `ignore_missing_imports` | `true` | Suppressed globally |

**Pattern:** Files universally use `from __future__ import annotations` (PEP 563) for forward references without string quoting. Explicit `typing` imports (Optional, Dict, List, Union, Any) are common.

```python
from __future__ import annotations
from typing import Any, Dict, List, Optional, Union
```

**Assessment:** Type annotations are present in most files but not enforced strictly. The mypy configuration prioritizes speed over strictness.

### Rust

Strong typing with `thiserror` for ergonomic enum-based errors:

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("Configuration error: {0}")]
    Config(String),
    #[error("Network error: {0}")]
    Network(String),
}

pub type Result<T> = std::result::Result<T, Error>;
```

**Assessment:** Strong typing. Rust edition 2024 in Cargo.toml. No clippy in CI (see Section 3).

### C++

No explicit type system enforcement (no clang-tidy). `.clang-format` present with Google-based style.

---

## 2. Linting and Formatting

### Python (Primary)

**Tool:** ruff (v0.1.0+) - replaces flake8, isort, parts of black

**Configuration** (pyproject.toml):
```toml
[tool.ruff]
line-length = 100
target-version = "py310"
exclude = ["third_party"]

[tool.ruff.lint]
select = ["E", "W", "F", "I", "C", "B"]
ignore = [
    "E501",  # line too long (handled by ruff-format)
    "B008",  # do not perform function calls in argument defaults
    "C901",  # too complex
    "B006",  # mutable data structures for argument defaults
    "B904",  # raise ... from err in except
    "E741",  # ambiguous variable name
    "E722",  # bare except
    "B027",  # empty method in abstract base class
]
```

**Pre-commit hooks:**
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.14
    hooks:
      - id: ruff
        args: [ --fix ]
      - id: ruff-format
```

**Format style:** double quotes, space indent, auto line endings.

**Notable:** `E722` (bare except) and `B006` (mutable defaults) are ignored - these are common bug sources.

### C++

**Tool:** `.clang-format` (Google-based, C++11 standard)

Key settings:
- `IndentWidth: 2`, `TabWidth: 2`
- `ColumnLimit: 80`
- `PointerAlignment: Left`
- `BreakBeforeTernaryOperators: true`
- `AllowShortFunctionsOnASingleLine: false`

**Assessment:** Strict formatting rules, short functions banned, 80-char limit enforced.

### Rust

**No clippy in CI** - `rust-cli.yml` only runs `cargo build --release`.

**Assessment:** Rust quality relies on developer discipline; no automated lint gate.

### TypeScript/Node.js

**No ESLint, Prettier, or TypeScript configuration** found in `bot/package.json`. The bot package has no scripts, no devDependencies, no linting tools.

**Assessment:** TypeScript/JavaScript code in `bot/` has no automated quality enforcement.

---

## 3. Testing

### Test Infrastructure

**Framework:** pytest with asyncio support

**Configuration:**
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
asyncio_mode = "auto"
addopts = "-v --cov=openviking --cov-report=term-missing"
```

### Test Organization

```
tests/
├── unit/           # 32 test files
├── integration/    # 21 directories
├── api_test/       # 17 directories
├── cli/
├── client/
├── engine/
├── eval/
├── parse/
├── server/
├── session/        # 10 subdirs
├── storage/
├── vectordb/
└── ...
```

### Test Quality Example

**test_circuit_breaker.py (336 lines)** follows AAA pattern:

```python
def test_trip_on_threshold(self):
    """Test breaker trips after reaching failure threshold."""
    cb = CircuitBreaker(failure_threshold=3)
    for _ in range(3):
        cb.record_failure(Exception("500 Error"))
    with pytest.raises(CircuitBreakerOpen):
        cb.check()
```

Coverage includes:
- Happy path (initial state, success resets)
- Threshold behavior (trip at threshold, not before)
- Half-open state transitions
- Thread safety (10 threads, 100 iterations each)
- Edge cases (zero threshold, negative timeout, mixed error types)

**Assessment:** High test quality with comprehensive coverage, edge cases, and thread safety testing.

### CI Test Matrix

**Full suite:** Ubuntu 24.04, macOS 14, Windows Latest x Python 3.10, 3.11, 3.12, 3.13 = 12 configurations.

**Current status:** Full pytest suite is **disabled** in CI due to unresolved issues. Only "lite" integration test runs.

```yaml
# TODO: Once unit tests are fixed, switch this back to running the full test suite
- name: Run Lite Integration Test (Temporary Replacement)
  run: uv run python tests/integration/test_quick_start_lite.py
```

---

## 4. Error Handling

### Python Pattern

**Logging:** loguru (not standard logging)

```python
from openviking_cli.utils.logger import get_logger
logger = get_logger(__name__)
```

**Custom exceptions:** Semantic, domain-specific exceptions:

```python
class CircuitBreakerOpen(Exception):
    """Raised when the circuit breaker is open and blocking requests."""

class DataDirectoryLocked(RuntimeError):
    """Raised when another OpenViking process holds the data directory lock."""
```

**Error classification:** Pattern-based error categorization:
```python
_PERMANENT_PATTERNS = ("403", "401", "Forbidden", "Unauthorized", "AccountOverdue")
_TRANSIENT_PATTERNS = ("429", "500", "502", "503", "504", "RateLimit", "timeout", ...)
```

### Rust Pattern

**thiserror-based enum** with From implementations:

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("Configuration error: {0}")]
    Config(String),
    #[error("Network error: {0}")]
    Network(String),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

**CliError struct** with exit codes for user-facing errors:

```rust
pub struct CliError {
    pub message: String,
    pub code: String,
    pub exit_code: i32,
}
```

---

## 5. CI/CD Quality Gates

### Lint Gate (_lint.yml)

1. **Changed file detection:** `git diff --name-only` against base branch
2. **ruff format check:** `ruff format --check`
3. **ruff lint check:** `ruff check`
4. **mypy type check:** `mypy` on changed files (continue-on-error: true)

**Limitation:** mypy runs on individual files only, missing cross-file type errors.

### Test Gate

- **Lite tests:** Always run (single integration test)
- **Full tests:** Matrix across OS and Python version (currently disabled)

### Security Gate

CodeQL analysis for security vulnerabilities.

### Rust Gate

- Builds for 5 platforms (Linux x86_64/arm64, macOS x86_64/arm64, Windows)
- SHA256 checksums generated and verified locally
- **No clippy or cargo test in CI**

### PR-Agent Integration

**.pr_agent.toml** configures automated PR review with:
- Custom labels (memory-pipeline, async-change, embedding-vectorization, etc.)
- 15 domain-specific review rules
- Severity classification (Critical, Bug, Perf, Suggestion)
- Test requirements enforcement

---

## 6. Code Review Patterns

### Commit Style

**Conventional commits** with scope:
```
fix(parse): treat .cc files as processable in directory scan (#1008)
build(deps): bump actions/cache from 4 to 5 (#1001)
fix(vlm): scope LiteLLM thinking param to DashScope providers only (#958)
feat: 添加完整的 API 测试套件 (#950)
```

Format: `<type>(<scope>): <description> (#pr)`

### License Headers

**Required on all source files:**
```python
# Copyright (c) 2026 Beijing Volcano Engine Technology Co., Ltd.
# SPDX-License-Identifier: Apache-2.0
```

Enforced via PR-Agent rule R6.

---

## 7. Summary Table

| Dimension | Python | Rust | C++ | TypeScript |
|-----------|--------|------|-----|------------|
| **Linter** | ruff (strong) | None in CI | clang-format only | None |
| **Type checker** | mypy (lenient) | Rustc (strong) | None | None |
| **Formatter** | ruff-format | rustfmt (implicit) | clang-format | None |
| **Tests** | pytest (comprehensive) | None in CI | None | None |
| **Error handling** | Custom exceptions + loguru | thiserror enum | Not reviewed | Not reviewed |
| **CI gates** | lint + lite test | build only | None | None |
| **Security scan** | CodeQL | None | None | None |
| **License headers** | Enforced | Enforced | Not reviewed | Not reviewed |

---

## 8. Gaps and Recommendations

### Critical Gaps

1. **TypeScript/Node.js has no linting or type checking**
   - `bot/package.json` has no devDependencies, no scripts, no ESLint/Prettier
   - JavaScript code is unvalidated

2. **Rust has no clippy in CI**
   - Quality relies on developer discipline
   - `cargo build --release` only, no lint/test gate

3. **Full test suite disabled**
   - CI runs only a "lite" integration test
   - 12-config matrix underutilized

4. **mypy runs per-file, not project-wide**
   - Cannot catch cross-file type errors
   - `continue-on-error: true` means failures are not blocking

### Recommendations

1. **Enable clippy in Rust CI** - Add `cargo clippy --all-targets` to rust-cli.yml
2. **Fix and re-enable full pytest suite** - The 12-config matrix is underutilized
3. **Add ESLint/Prettier for bot/** - TypeScript code needs automated quality gates
4. **Strengthen mypy configuration** - Consider `disallow_untyped_defs = true` for new code
5. **Run mypy project-wide** - Not just changed files, to catch cross-module type errors
6. **Remove E722/B006 from ignore list** - These catch real bugs; fix the violations
