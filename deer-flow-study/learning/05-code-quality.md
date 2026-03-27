# Code Quality: DeerFlow 2.0

## Quality Philosophy

DeerFlow maintains high code quality through mandatory test-driven development (backend), strict type systems, automated linting/formatting, and architectural enforcement of module boundaries.

## Backend Quality Practices

### Python Standards

| Aspect | Standard |
|--------|----------|
| **Python Version** | 3.12+ |
| **Type Checking** | Type hints throughout |
| **Linter** | ruff (E, F, I, UP rules) |
| **Formatter** | ruff with line-length 240 |
| **Quote Style** | Double quotes |
| **Test Framework** | pytest |

### Ruff Configuration

```toml
# backend/ruff.toml
line-length = 240
target-version = "py312"

[lint]
select = ["E", "F", "I", "UP"]

[lint.isort]
known-first-party = ["deerflow", "app"]

[format]
quote-style = "double"
indent-style = "space"
```

### Test-Driven Development (Mandatory)

**Every new feature or bug fix MUST be accompanied by unit tests.**

```
backend/tests/
├── test_acp_config.py
├── test_aio_sandbox_provider.py
├── test_app_config_reload.py
├── test_artifacts_router.py
├── test_channel_file_attachments.py
├── test_channels.py
├── test_checkpointer.py
├── test_client.py           # 77 unit tests + conformance
├── test_client_e2e.py
├── test_client_live.py      # Live integration tests
├── test_config_version.py
├── test_credential_loader.py
├── test_custom_agent.py
├── test_dangling_tool_call_middleware.py
├── test_docker_sandbox_mode_detection.py
├── test_feishu_parser.py
├── test_guardrail_middleware.py  # 25 tests
├── test_harness_boundary.py       # Architectural enforcement
├── test_infoquest_client.py
├── test_invoke_acp_agent_tool.py
├── test_lead_agent_model_resolution.py
├── test_memory_updater.py
├── test_memory_upload_filtering.py
├── test_model_factory.py
├── test_mcp_client_config.py
├── test_mcp_oauth.py
└── ... (60+ test files)
```

**Test execution:**
```bash
cd backend
make test        # Full test suite
make lint        # Ruff linting
make format      # Ruff formatting
```

### Gateway Conformance Testing

`TestGatewayConformance` in `test_client.py` validates that every dict-returning `DeerFlowClient` method conforms to the corresponding Gateway Pydantic response model. If Gateway adds a required field that the client does not provide, Pydantic raises `ValidationError` and CI catches the drift.

**Covered responses:**
- `ModelsListResponse`
- `ModelResponse`
- `SkillsListResponse`
- `SkillResponse`
- `SkillInstallResponse`
- `McpConfigResponse`
- `UploadResponse`
- `MemoryConfigResponse`
- `MemoryStatusResponse`

### Architectural Boundary Enforcement

`tests/test_harness_boundary.py` ensures the harness/app split is respected:

```
Harness (deerflow.*) ──► App (app.*)    ✓ ALLOWED
App (app.*) ──► Harness (deerflow.*)   ✓ ALLOWED
Harness (deerflow.*) ──► App (app.*)   ✗ FORBIDDEN (enforced by CI)
```

This boundary is critical because the harness (`packages/harness/deerflow/`) is published as a separate package (`deerflow-harness`), while `app/` contains unpublished application code.

### Regression Coverage

Regression tests exist for:
- Docker sandbox mode detection (`test_docker_sandbox_mode_detection.py`)
- Provisioner kubeconfig handling (`test_provisioner_kubeconfig.py`)
- Guardrail middleware behavior (`test_guardrail_middleware.py`)
- Memory updater deduplication (`test_memory_updater.py`)

## Frontend Quality Practices

### TypeScript Standards

| Aspect | Standard |
|--------|----------|
| **TypeScript Version** | 5.8 |
| **Type Checking** | Strict via `tsc --noEmit` |
| **Linter** | ESLint + TypeScript-ESLint |
| **Formatter** | Prettier |
| **UI Components** | Shadcn UI, Radix UI primitives |

### ESLint Configuration

```javascript
// frontend/eslint.config.js
export default tseslint.config(
  { ignores: [".next", "src/components/ui/**", "src/components/ai-elements/**"] },
  ...compat.extends("next/core-web-vitals"),
  {
    files: ["**/*.ts", "**/*.tsx"],
    extends: [...tseslint.configs.recommended, ...tseslint.configs.recommendedTypeChecked],
    rules: {
      "@typescript-eslint/consistent-type-imports": ["warn", { prefer: "type-imports" }],
      "@typescript-eslint/no-unused-vars": ["warn", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-misused-promises": ["error", { checksVoidReturn: { attributes: false } }],
      // ... additional rules
    }
  }
);
```

**Import ordering enforced:**
```typescript
import { builtin } from "fs";           // builtin
import { external } from "package";    // external
import { internal } from "@/lib/utils"; // internal (@/*)
import { parent } from "../utils";      // parent
import { sibling } from "./hooks";      // sibling
```

### Prettier Configuration

```javascript
// frontend/prettier.config.js
export default {
  plugins: ["prettier-plugin-tailwindcss"],
};
```

### Quality Commands

```bash
pnpm check      # ESLint + TypeScript type check
pnpm lint       # ESLint only
pnpm lint:fix   # ESLint with auto-fix
pnpm typecheck  # TypeScript only (tsc --noEmit)
```

**Note**: Frontend has no configured test framework.

## Shared Practices

### Environment Validation

Frontend uses `@t3-oss/env-nextjs` with Zod schemas for environment validation:

```typescript
// frontend/src/env.js
// Validates NEXT_PUBLIC_* variables at build time
// Skip with SKIP_ENV_VALIDATION=1
```

Backend config system validates `config.yaml` against Pydantic models with version checking.

### Documentation Update Policy

Per `CLAUDE.md`:
> **CRITICAL: Always update README.md and CLAUDE.md after every code change**

- Update `README.md` for user-facing changes
- Update `CLAUDE.md` for development changes

### Code Style Enforcement

| Language | Tool | Line Length | Quote Style |
|----------|------|-------------|-------------|
| Python | ruff | 240 | double |
| TypeScript | ESLint + Prettier | 100 (default) | double |

### Unused Variable Handling

- **Python**: Ruff handles via F841 (unused local variable)
- **TypeScript**: Prefix unused variables with `_`

```typescript
// TypeScript
const _unusedVariable = calculateSomething(); // ESLint warns but allows
```

## CI/CD Quality Gates

### Backend CI

GitHub Actions workflow (`backend-unit-tests.yml`) runs on every PR:
1. Ruff linting
2. pytest full suite
3. Harness boundary verification
4. Gateway conformance tests

### Frontend CI

No explicit CI configuration visible — quality enforced via `pnpm check` before commit.

## Dependency Management

### Backend
- **Package Manager**: uv
- **Workspace**: `packages/harness` as internal package
- **Dev Dependencies**: pytest, ruff

### Frontend
- **Package Manager**: pnpm 10.26.2
- **Lock File**: pnpm-lock.yaml
- **Dependencies**: Pinned major versions (e.g., `"next": "^16.1.7"`)

## Static Analysis

### Python
- ruff: Linting + import sorting
- Type hints: Runtime validation via Pydantic

### TypeScript
- TypeScript compiler (`tsc --noEmit`): No compilation to JavaScript
- ESLint: Code quality, anti-patterns
- TypeScript-ESLint: Type-aware linting

## Code Review Standards

From CONTRIBUTING.md:
- All submissions require review
- Regression coverage mandatory for behavioral changes
- Documentation must sync with code changes

## Quality Metrics

| Metric | Backend | Frontend |
|--------|---------|----------|
| **Test Files** | 60+ | None configured |
| **Type System** | Python type hints | TypeScript (strict) |
| **Linter** | ruff | ESLint |
| **Formatter** | ruff | Prettier |
| **TDD Policy** | Mandatory | Not enforced |
| **Boundary Tests** | Yes (harness/app) | N/A |
