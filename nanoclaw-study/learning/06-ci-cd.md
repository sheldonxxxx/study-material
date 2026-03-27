# CI/CD

## GitHub Actions Workflows

### CI Pipeline (`.github/workflows/ci.yml`)

Triggered on: **Pull requests to `main` branch**

```yaml
on:
  pull_request:
    branches: [main]
```

**Runner**: `ubuntu-latest`

**Steps**:

1. **Checkout** - `actions/checkout@v4`
2. **Setup Node** - `actions/setup-node@v4`
   - Node version: `20`
   - Cache: `npm`
3. **Install dependencies** - `npm ci`
4. **Format check** - `npm run format:check`
5. **Typecheck** - `npx tsc --noEmit`
6. **Tests** - `npx vitest run`

### PR Labeling (`.github/workflows/label-pr.yml`)

Triggered on: **PR opened or edited**

```yaml
on:
  pull_request:
    types: [opened, edited]
```

**Purpose**: Auto-labels PRs based on checkbox selections in PR template.

**Labels applied**:

| PR Type (checked in template) | Labels Applied |
|-------------------------------|----------------|
| Feature skill | `PR: Skill`, `PR: Feature` |
| Utility skill | `PR: Skill` |
| Operational/container skill | `PR: Skill` |
| Fix | `PR: Fix` |
| Simplification | `PR: Refactor` |
| Documentation | `PR: Docs` |
| Follows contributing guide | `follows-guidelines` |

**Implementation**: Uses `actions/github-script@v7` to parse PR body and apply labels via GitHub API.

## Code Quality Tools

### ESLint Configuration (`.eslint.config.js`)

```javascript
{
  ignores: ['node_modules/', 'dist/', 'container/', 'groups/'],
  plugins: { 'no-catch-all': noCatchAll },
  rules: {
    'preserve-caught-error': ['error', { requireCatchParameter: true }],
    '@typescript-eslint/no-unused-vars': ['error', { /* strict config */ }],
    'no-catch-all/no-catch-all': 'warn',
    '@typescript-eslint/no-explicit-any': 'warn',
  }
}
```

**Notable rules**:
- `no-catch-all/no-catch-all`: Warns against catching generic `catch (e)` without typed error
- `preserve-caught-error`: Requires catch parameter to be used
- Stricter unused variable detection with `_` prefix allowance

### Prettier Configuration (`.prettierrc`)

```json
{
  "singleQuote": true
}
```

### Git Hooks

**Husky pre-commit hook** (`.husky/pre-commit`):
- Runs `husky` during `npm prepare`
- Hook script not visible in repo (managed by husky)

## Testing

### Vitest Configuration (vitest.config.ts)

```typescript
export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'setup/**/*.test.ts'],
  },
});
```

**Test framework**: Vitest with V8 coverage

**Test files located**:
- `src/*.test.ts` - Core unit tests
- `setup/*.test.ts` - Setup script tests

### Run Commands

```bash
npm test              # Run tests once
npm run test:watch    # Watch mode
npx vitest run        # Direct vitest invocation
```

## Build Pipeline

### Host Application

```bash
npm run build         # Compile TypeScript
npm run dev          # Run with tsx hot reload
npx tsc --noEmit     # Typecheck only
```

### Agent Container

Located in `container/`:
- `agent-runner/` - Node.js app with TypeScript
- `build.sh` - Rebuilds the Docker container image
- Dockerfile based on `node:22-slim` with Chromium

```bash
./container/build.sh  # Rebuild agent container
```

## Version Management

- **Version in package.json**: `1.2.34` (semver)
- **CHANGELOG.md**: Tracks breaking changes and release history
- **Documentation site**: Full release history at `docs.nanoclaw.dev/changelog`

## CODEOWNERS

Core code ownership restricted to maintainers:
```
/src/                 @gavrielc @gabi-simons
/container/           @gavrielc @gabi-simons
/groups/              @gavrielc @gabi-simons
/launchd/             @gavrielc @gabi-simons
/package.json         @gavrielc @gabi-simons
/package-lock.json    @gavrielc @gabi-simons
```

Skills are open to community contributions:
```
/.claude/skills/      (no owner - open)
```
