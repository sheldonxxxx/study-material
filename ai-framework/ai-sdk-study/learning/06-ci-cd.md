# CI/CD Pipeline

## Overview

The AI SDK uses GitHub Actions for continuous integration and release automation. The pipeline enforces code quality, runs comprehensive tests across multiple Node.js versions, and automates releases via Changesets.

## Workflows

### CI Pipeline (`.github/workflows/ci.yml`)

**Trigger Conditions**:
- Push to `main` or `release-v*` branches
- Pull requests targeting `main` or `release-v*` branches

**Jobs**:

| Job | Name | Key Steps |
|-----|------|-----------|
| `check` | Lint & Format | pnpm install, ultracite check |
| `build-examples` | Build Examples | pnpm install, pnpm run build:examples |
| `types` | TypeScript | pnpm install, pnpm run type-check:full |
| `bundle-size` | Bundle Size Check | Build packages, check-bundle-size |
| `test_matrix` | Test (Node 20, 22, 24) | Install Playwright, pnpm test |
| `test` | Aggregate Test Status | Consolidates matrix results |
| `load-time_matrix` | Load Time Check | Measures module load times |
| `load-time` | Aggregate Load Time | Consolidates load time results |

#### Test Matrix Strategy

```yaml
strategy:
  matrix:
    node-version: [20, 22, 24]
```

Tests run on three Node.js versions to ensure broad compatibility.

#### Bundle Size Checks

- Runs `pnpm run check-bundle-size` in packages/ai
- Uploads results as artifacts (bundle-size-metafiles)

#### Load Time Benchmarks

Pre-configured thresholds per module:

| Module | Max Load Time |
|--------|---------------|
| ai | 105ms |
| @ai-sdk/openai | 65ms |
| @ai-sdk/openai-compatible | 65ms |
| @ai-sdk/anthropic | 65ms |
| @ai-sdk/google | 65ms |

### Release Pipeline (`.github/workflows/release.yml`)

**Trigger Conditions**:
- Push to `main` or `release-v*` with changes to `.changeset/**` or the workflow file itself
- Manual `workflow_dispatch`

**Concurrency**: Single release group, no cancel-in-progress

**Steps**:
1. Create GitHub App token for automation
2. Configure git user for GitHub App
3. Checkout with full history
4. Setup pnpm v10.11.0 + Node.js 22
5. Install dependencies
6. Run Changesets action (version + publish)
7. Notify released packages (GitHub script)

**Environment Variables**:
- `GITHUB_TOKEN`: GitHub App token
- `NPM_TOKEN`: Elevated npm access for publishing

### Supporting Workflows

| Workflow | Purpose |
|----------|---------|
| `backport.yml` | Automated backport PRs |
| `triage.yml` | Issue/PR triage automation |
| `release-snapshot.yml` | Snapshot release process |
| `ai-provider-models.yml` | Provider model updates |
| `ai-provider-api-changes.yml` | API change detection |
| `assign-team-pull-request.yml` | Auto-assign team to PRs |
| `auto-merge-release-prs.yml` | Auto-merge release PRs |
| `ultracite-on-automerge.yml` | Run ultracite checks on merge |
| `validate-npm-token.yml` | Validate npm tokens |
| `verify-changesets.yml` | Verify changeset configuration |
| `slack-team-review-notification.yml` | Slack notifications |
| `slack-workflow-failure-notification.yml` | Failure alerts |
| `discussions-auto-close-new.yml` | Auto-close new discussions |
| `discussions-auto-close-stale.yml` | Auto-close stale discussions |

## Build Pipeline (Local)

### Turborepo Commands

```bash
pnpm build              # Build all packages (concurrency 16)
pnpm build:examples     # Build only examples
pnpm build:packages     # Build only @ai-sdk/* and ai packages
pnpm dev                # Development with local/remote caching
pnpm clean              # Clean build artifacts
```

### Task Dependencies

```
build task:
  dependsOn: ["^build"]  # Build dependencies first

type-check task:
  dependsOn: ["^build", "build"]

test task:
  dependsOn: ["^build", "build"]
```

### Environment Variables for Build

**Global Env**: `CI`, `PORT`

**Build Env** (60+ provider API keys):

```bash
AI_GATEWAY_API_KEY, ANTHROPIC_API_KEY, OPENAI_API_KEY,
GOOGLE_API_KEY, GOOGLE_VERTEX_API_KEY, AZURE_API_KEY,
AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION,
FIREWORKS_API_KEY, MISTRAL_API_KEY, DEEPSEEK_API_KEY,
XAI_API_KEY, GROQ_API_KEY, COHERE_API_KEY, TOGETHER_API_KEY,
HUGGINGFACE_API_KEY, REPLICATE_API_TOKEN, ELEVENLABS_API_KEY,
FAL_KEY, SENTRY_AUTH_TOKEN, VERCEL_*... # 60+ total
```

## Release Process

### Changesets Workflow

1. **Author adds changeset**: `pnpm changeset` in workspace root
2. **Changeset format**: Describes patch/minor/major bump per package
3. **CI creates version PR**: When main receives changeset changes
4. **Release publishes to npm**: When version PR is merged

### Release Commands

```bash
pnpm ci:version    # Run changeset version + cleanup examples + pnpm install
pnpm ci:release     # Clean + build + changeset publish
```

### Changeset Configuration

```json
{
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch"
}
```

## Pre-commit Hooks

### Husky Integration

```bash
pnpm prepare    # Sets up Husky hooks via package.json script
```

### lint-staged Configuration

```json
"lint-staged": {
  "*.{js,jsx,ts,tsx,mjs,cjs}": ["ultracite fix"]
}
```

When committing:
- Staged JS/TS/JSX/TSX/MJS/CJS files are automatically formatted
- If package.json changes, `pnpm install` runs automatically
- Skip hooks with `ARTISANAL_MODE=1`

## Quality Gates

| Gate | Command | Fail Condition |
|------|---------|----------------|
| Linting | `pnpm check` (ultracite) | Lint errors |
| Formatting | `pnpm check` (ultracite) | Format violations |
| Type Checking | `pnpm type-check:full` | Type errors |
| Tests | `pnpm test` | Test failures |
| Bundle Size | `check-bundle-size` | Exceeds baseline |
| Load Time | Benchmark script | Exceeds threshold |
| Publint | `pnpm publint` | Package structure issues |
