# OASIS Code Quality Practices

## Code Style and Formatting

### Style Guide

OASIS follows the **Google Python Style Guide** with enforcement via **Ruff**.

### Pre-commit Hooks

The project uses pre-commit hooks (`.pre-commit-config.yaml`) for automated quality checks:

| Hook | Version | Purpose |
|------|---------|---------|
| end-of-file-fixer | v4.5.0 | Ensure files end with newline |
| mixed-line-ending | v4.5.0 | Normalize line endings to LF |
| trailing-whitespace | v4.5.0 | Remove trailing whitespace |
| check-toml | v4.5.0 | Validate TOML files |
| check-yaml | v4.5.0 | Validate YAML files |
| yamllint | v1.35.1 | Lint YAML with custom rules |
| isort | 5.13.2 | Sort Python imports |
| flake8 | 7.0.0 | PEP8 compliance checking |
| ruff | v0.3.5 | Fast linting and formatting |
| mdformat | 0.7.17 | Format Markdown files |

### Pre-commit Installation

```bash
# Install pre-commit hooks
pre-commit install

# Run all hooks manually
pre-commit run --all-files
```

## Testing Framework

### Test Structure

```
test/
├── conftest.py           # Pytest fixtures and configuration
├── agent/                # Agent-level tests
│   ├── test_action_docstring.py
│   ├── test_agent_custom_prompt.py
│   ├── test_agent_generator.py
│   ├── test_agent_graph.py
│   ├── test_agent_tools.py
│   ├── test_interview_action.py
│   ├── test_multi_agent_signup_create.py
│   └── test_twitter_user_agent_all_actions.py
├── infra/
│   ├── database/         # Database operation tests
│   │   ├── test_comment.py
│   │   ├── test_post.py
│   │   ├── test_user.py
│   │   ├── test_refresh.py
│   │   ├── test_search.py
│   │   └── ... (20+ test files)
│   └── recsys/          # Recommendation system tests
│       ├── test_recsys.py
│       └── test_update_rec_table.py
```

### Running Tests

```bash
# Run all tests (requires OPENAI_API_KEY)
export OPENAI_API_KEY=<your-key>
pytest .

# Run with coverage
pytest --cov --cov-report=html
# Open htmlcov/index.html for report

# Generate coverage report from scratch
coverage erase
coverage run --source=. -m pytest .
coverage html
```

### Test Dependencies

- **pytest** 8.2.0 - Test framework
- **pytest-asyncio** 0.23.6 - Async test support
- Tests require `OPENAI_API_KEY` environment variable for LLM-based tests

## Documentation Standards

### Docstring Format

OASIS uses **raw triple-quoted docstrings** (`r"""`) with Google-style sections:

```python
r"""Class for managing conversations of OASIS Agents.

Args:
    system_message (BaseMessage): The system message for initializing
        the agent's conversation context.
    model (BaseModelBackend, optional): The model backend to use for
        response generation. Defaults to :obj:`OpenAIModel` with
        `GPT_4O_MINI`. (default: :obj:`OpenAIModel` with `GPT_4O_MINI`)
"""
```

### Documentation Build

Documentation is built with **Mintlify**:

```bash
npm install -g mintlify
cd docs
mintlify dev
```

## Code Review Process

### Review Checklist

**Functionality**
- Correctness: Does the code perform the intended task?
- Edge cases: Are boundary conditions handled?
- Testing: Is there sufficient test coverage?
- Security: Any security vulnerabilities introduced?
- Performance: Any performance regressions?

**Code Quality**
- Readability: Is the code easy to read?
- Maintainability: Is the code structured for future changes?
- Style: Does it follow project style guidelines (Ruff, Google Python Style)?
- Documentation: Are public methods and complex logic documented?

**Design**
- Consistency: Does it follow established patterns?
- Modularity: Are changes self-contained?
- Dependencies: Are dependencies minimized and appropriate?

### PR Labels

| Label | Purpose |
|-------|---------|
| `feat` | New features |
| `fix` | Bug fixes |
| `docs` | Documentation updates |
| `style` | Code style changes |
| `refactor` | Code refactoring |
| `test` | Adding/updating tests |
| `chore` | Maintenance tasks |

## Naming and Logging Conventions

### Naming Principle

Avoid abbreviations in naming. Names should be clear for both human readers and AI agents:

```python
# Bad
msg_win_sz

# Good
message_window_size
```

### Logging Principle

Use the `logging` module instead of `print`:

```python
# Bad
print("Process started")
print(f"User input: {user_input}")

# Good
logger.info("Process started")
logger.debug(f"User input: {user_input}")
```

## Licensing

All source files must include the Apache 2.0 license header:

```python
# =========== Copyright 2023 @ CAMEL-AI.org. All Rights Reserved. ===========
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# =========== Copyright 2023 @ CAMEL-AI.org. All Rights Reserved. ===========
```

License can be added automatically:

```bash
python licenses/update_license.py . licenses/license_template.txt
```

## Type Safety

### Type Annotations

The codebase uses Python type annotations for better IDE support and documentation:

```python
from typing import List, Optional, Union

def __init__(
    self,
    agent_graph: AgentGraph,
    platform: Union[DefaultPlatformType, Platform],
    database_path: str = None,
    semaphore: int = 128,
) -> None:
```

### Runtime Type Checking

The project uses `TYPE_CHECKING` guard for imports only needed for type hints:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from oasis.social_agent import AgentGraph
```

## Contribution Requirements

### For Bug Fixes
- Add a relevant unit test when possible

### For Improvements
- Update affected example scripts in `examples/`
- Update documentation in `docs/`
- Update unit tests when relevant

### For New Features
- Include unit tests in `test/`
- Add a demo script in `examples/`

## Continuous Integration

All PRs must pass:
1. Pre-commit hooks (formatting, linting)
2. pytest test suite
3. Documentation build (for docs changes)

## Versioning

OASIS follows **semantic versioning** (semver):
- Major version 0 (pre-1.0): Minor versions may have breaking changes
- Patch versions: Backwards compatible fixes only
