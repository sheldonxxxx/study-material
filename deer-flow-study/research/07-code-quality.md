# Code Quality Assessment: deer-flow

## Summary

| Dimension | Assessment |
|-----------|------------|
| **Type Systems** | TypeScript (frontend, strict), Python with type hints (backend, no mypy) |
| **Linting** | Ruff (Python), ESLint + TypeScript-ESLint (frontend) |
| **Formatting** | Ruff format (Python), Prettier (frontend implicit via ESLint) |
| **Testing** | Extensive pytest suite (backend), minimal Node test runner (frontend) |
| **CI/CD** | GitHub Actions with lint + unit test + build checks |
| **Error Handling** | Structured exception patterns, middleware chain architecture |
| **Logging** | Moderate logging usage (~72 files), logging module used |
| **Secrets** | Environment variable pattern, no hardcoded credentials |
| **Code Review** | Conventional commit format enforced, PR regression tests |

---

## Type Systems

### Frontend (TypeScript)
- **Configuration**: `frontend/tsconfig.json`
- **Strictness**: `strict: true`, `noUncheckedIndexedAccess: true`
- **Notable**: `verbatimModuleSyntax: true` enforces explicit type imports
- **Weaknesses**: `noImplicitAny: false` — allows implicit `any` (intentional relaxation)
- **Coverage**: Full TypeScript across Next.js app, components, and core business logic

### Backend (Python)
- **Type Hints**: Used throughout `packages/harness/deerflow/` and `app/`
- **No mypy/pyright**: Project relies solely on Ruff for linting, not static type checking
- **Import Style**: Uses `from __future__ import annotations` pattern implicitly via esModuleInterop
- **Coverage**: Type hints in function signatures, class definitions, and config schemas

---

## Linting & Formatting

### Python (Backend)
- **Tool**: Ruff (`uvx ruff check .`)
- **Config**: `pyproject.toml` defines dev dependency `ruff>=0.14.11`
- **Commands**:
  - `make lint` — `uvx ruff check .`
  - `make format` — `uvx ruff check . --fix && uvx ruff format .`
- **Rules**: 240-character line length, double quotes, space indentation
- **Enforcement**: CI runs lint check on every push to main and all PRs

### TypeScript/Frontend
- **Tool**: ESLint with `typescript-eslint`
- **Config**: `frontend/eslint.config.js`
- **Extended configs**: `next/core-web-vitals`, `typescript-eslint/recommended`, `typescript-eslint/recommendedTypeChecked`, `typescript-eslint/stylisticTypeChecked`
- **Notable rules**:
  - `import/order` — strict alphabetized import ordering with groups
  - `@typescript-eslint/consistent-type-imports` — warns for inline type imports
  - `@typescript-eslint/no-misused-promises` — enforced with `error` level
- **Disabling**: UI components (`src/components/ui/**`) and AI elements (`src/components/ai-elements/**`) are ESLint-ignored (generated code)
- **Commands**:
  - `pnpm lint` — ESLint only
  - `pnpm typecheck` — `tsc --noEmit`
  - `pnpm check` — lint + typecheck (pre-commit gate)

---

## Testing

### Backend Test Suite
- **Framework**: pytest (`pytest>=8.0.0`)
- **Coverage**: 67 test files covering:
  - Configuration (`test_acp_config.py`, `test_app_config_reload.py`, `test_model_config.py`, `test_config_version.py`)
  - Middleware (`test_dangling_tool_call_middleware.py`, `test_guardrail_middleware.py`, `test_loop_detection_middleware.py`, `test_subagent_limit_middleware.py`, `test_todo_middleware.py`, `test_title_middleware_core_logic.py`, `test_tool_error_handling_middleware.py`)
  - Client (`test_client.py` — 77 unit tests including Gateway conformance, `test_client_e2e.py`, `test_client_live.py`)
  - Memory (`test_memory_updater.py`, `test_memory_prompt_injection.py`, `test_memory_upload_filtering.py`)
  - Skills (`test_skills_parser.py`, `test_skills_loader.py`, `test_skills_installer.py`, `test_skills_archive_root.py`)
  - Channels (`test_channels.py`, `test_feishu_parser.py`)
  - Sandbox (`test_aio_sandbox_provider.py`, `test_docker_sandbox_mode_detection.py`, `test_local_sandbox_encoding.py`, `test_sandbox_tools_security.py`)
  - MCP (`test_mcp_client_config.py`, `test_mcp_oauth.py`, `test_mcp_sync_wrapper.py`)
  - Boundary checks (`test_harness_boundary.py` — enforces harness/app import firewall)
- **Conftest**: `tests/conftest.py` handles circular import mocks for executor
- **Command**: `make test` runs `PYTHONPATH=. uv run pytest tests/ -v`

### Frontend Test Suite
- **Minimal**: One test file `frontend/src/core/api/stream-mode.test.ts`
- **Framework**: Node.js built-in `node:test` with `node:assert/strict`
- **Coverage**: Only tests `sanitizeRunStreamOptions` function for stream mode filtering
- **Assessment**: Frontend testing is significantly under-developed compared to backend

---

## CI/CD

### Workflows
1. **`backend-unit-tests.yml`**: Runs pytest on Python 3.12, 15-minute timeout, skips draft PRs
2. **`lint-check.yml`**: Three parallel jobs:
   - Python lint (`make lint`)
   - Frontend lint (`pnpm lint`)
   - Frontend typecheck + build (`pnpm typecheck && pnpm build`)

### Quality Gates
- Draft PRs skip unit tests but still run lint
- Concurrency group cancels in-progress runs on new commits
- Regression tests for Docker/provisioner mode detection run in CI

---

## Error Handling

### Patterns Observed
- **Middleware Chain**: 10+ middleware components in `deerflow/agents/middlewares/` handle errors at each stage
- **Tool Error Handling**: `ToolErrorHandlingMiddleware` wraps tool execution
- **Guardrails**: `GuardrailMiddleware` for pre-tool authorization
- **Explicit Exceptions**: `raise ValueError`, `raise RuntimeError` used in configuration and validation (~72 files)
- **Error Messages**: Actionable error messages with install hints (e.g., `uv add langchain-google-genai`)

### Examples
- `PathTraversalError` in uploads manager for file path security
- Actionable ACP errors instead of raw `FileNotFoundError`
- Config version mismatch warnings with upgrade instructions

---

## Logging

### Usage
- **Scope**: ~72 Python files contain logging usage
- **Pattern**: `logging.getLogger(__name__)` pattern used throughout
- **Modules with logging**:
  - `deerflow/client.py` — 5 occurrences
  - `deerflow/skills/installer.py` — 9 occurrences
  - `deerflow/uploads/manager.py` — 5 occurrences
  - `deerflow/tools/builtins/present_file_tool.py` — 4 occurrences
  - Various middleware components

### Assessment
- Logging present but not comprehensive — not all modules use logging
- No structured logging library (no `structlog`)
- Log level configurable via `feat: add configurable log level` (#1301)

---

## Config Management & Secrets

### Config System
- **Backend**: YAML (`config.yaml`) with environment variable resolution (`$VAR_NAME` pattern)
- **Extensions**: JSON (`extensions_config.json`) for MCP servers and skills
- **Frontend**: Environment variables validated via `@t3-oss/env-nextjs` with Zod schemas

### Secrets Handling
- **Pattern**: `os.getenv()` for environment variables (~31 files)
- **No hardcoded credentials**: All API keys, tokens via env vars or config
- **Credential Loader**: `deerflow/models/credential_loader.py` handles multiple sources (env, file paths, OAuth)
- **Examples**:
  - `INFOQUEST_API_KEY`, `JINA_API_KEY` checked via `os.getenv()`
  - `DEER_FLOW_CONFIG_PATH`, `DEER_FLOW_EXTENSIONS_CONFIG_PATH` for config location
  - `CLAUDE_CODE_CREDENTIALS_PATH`, `CLAUDE_CODE_OAUTH_TOKEN` for Claude auth

---

## Code Review Patterns

### Commit Messages
- **Format**: Conventional commits (`feat:`, `fix:`, `test:`, `docs:`, `refactor:`, `build:`)
- **Scope**: Format includes scope in parentheses when applicable: `fix(gateway):`, `fix(LLM):`, `fix(config):`
- **Examples** from recent history:
  - `fix(gateway): enforce safe download for active artifact MIME types to mitigate stored XSS`
  - `test: add unit tests for TodoMiddleware`
  - `feat(harness): integration ACP agent tool`
  - `fix: add null checks for runtime.context and tighten langgraph constraint`

### Pull Request Practices
- **Regression tests**: CI runs backend unit tests on every PR
- **Docker mode detection tests**: `test_docker_sandbox_mode_detection.py`, `test_provisioner_kubeconfig.py` specifically for PR regression
- **Boundary enforcement**: `test_harness_boundary.py` ensures harness never imports app layer
- **Gateway conformance tests**: `TestGatewayConformance` in `test_client.py` catches API drift

### Documentation Policy
- **Mandatory updates**: CLAUDE.md and README.md must be updated after every code change
- **Test coverage**: CONTRIBUTING.md mandates TDD — "Every new feature or bug fix MUST be accompanied by unit tests. No exceptions."

---

## Strengths

1. **Comprehensive backend test suite** — 67 test files with good coverage of middleware, config, memory, skills, and client
2. **Strict TypeScript config** — strict mode, verbatimModuleSyntax, noUncheckedIndexedAccess
3. **Enforced import ordering** — ESLint import/order with alphabetization catches style issues
4. **Harness/App boundary** — architectural rule enforced by automated test
5. **Gateway conformance tests** — client must match Gateway API schemas or CI fails
6. **Config versioning** — `config_version` field enables auto-upgrade
7. **Actionable error messages** — missing dependencies suggest install commands
8. **Security-conscious** — XSS mitigation for artifacts, path traversal protection

## Weaknesses

1. **No Python static type checking** — mypy/pyright not configured; type hints are documentation only
2. **Minimal frontend testing** — only 1 test file covering stream mode sanitization
3. **Implicit `any` allowed** — TypeScript `noImplicitAny: false` weakens type safety
4. **Inconsistent logging** — ~72 files use logging but not all modules
5. **Generated code excluded** — UI and AI elements ESLint-ignored (acceptable for shadcn/ui, but worth noting)
6. **No frontend unit tests for core logic** — thread hooks, API client, artifact loading untested

---

## Recommendations

1. **Add mypy to CI** — Enable static type checking for Python: `uvx mypy deerflow/ app/`
2. **Expand frontend tests** — Add tests for thread hooks, API client, message processing
3. **Enable `noImplicitAny`** — Gradually fix implicit any issues in TypeScript
4. **Standardize logging** — Consider structured logging across all modules
5. **Add pre-commit hooks** — `ruff check` + `pnpm check` as pre-commit gate
