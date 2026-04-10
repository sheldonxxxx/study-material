# Tech Stack Analysis: Vercel AI SDK Repository

## Overview

**Repository:** https://github.com/vercel/ai
**License:** Apache-2.0
**Type:** Monorepo (pnpm workspaces + Turborepo)

## Languages and Runtimes

| Language | Version | Notes |
|----------|---------|-------|
| TypeScript | 5.8.3 | Strict mode enabled |
| JavaScript | ESNext | Via TypeScript compilation |

**Node.js Support:** `>=18` (supports v18, v20, v22, v24)

## Package Manager

- **pnpm** v10.11.0 (specified via `packageManager` field)
- Workspace structure defined in `pnpm-workspace.yaml`

```yaml
packages:
  - 'packages/*'
  - 'tools/*'
  - 'examples/*'
  - 'packages/rsc/tests/e2e/next-server'
```

## Monorepo Tooling

### Turborepo

**Version:** 2.4.4

Root-level `turbo.json` defines task pipeline:

| Task | Dependencies | Notes |
|------|--------------|-------|
| `build` | `^build` (ancestors) | 50+ env vars for API keys |
| `type-check` | `^build` + `build` | Full type validation |
| `test` | `^build` + `build` | Runs in parallel (concurrency 16) |
| `dev` | none (cache disabled) | Persistent, local+remote cache |
| `clean` | `^clean` | Cascading clean |
| `publint` | `^build` + `build` | Package linting |

### Changesets

**Version:** 2.27.10

Used for versioning and changelog management. CI commands:
- `ci:version`: Runs changeset version, cleans example changesets
- `ci:release`: Clean, build, and publish

### No Nx or Lerna

This is a pure pnpm + Turbo monorepo (no Nx, no Lerna).

## Build System

### Package Build Tool

**tsup** v7.2.0 / v8

Most packages use tsup for TypeScript compilation and bundling:
- Outputs ESM (`.mjs`), CJS (`.js`), and type definitions (`.d.ts`)
- Example from `packages/ai`:
  ```json
  "build": "pnpm clean && tsup --tsconfig tsconfig.build.json"
  ```

### Build Output Patterns

```json
{
  "main": "./dist/index.js",        // CommonJS
  "module": "./dist/index.mjs",      // ESM
  "types": "./dist/index.d.ts"       // TypeScript
}
```

### esbuild

**Version:** 0.24.2

Used as a dependency for tsup. Marked as `onlyBuiltDependencies` in root pnpm config.

## Framework and UI Integration Packages

| Package | Purpose | Key Dependencies |
|---------|---------|------------------|
| `@ai-sdk/react` | React hooks (useChat, useCompletion, etc.) | react, swr, throttleit |
| `@ai-sdk/vue` | Vue composition functions | vue |
| `@ai-sdk/svelte` | Svelte stores and functions | svelte |
| `@ai-sdk/angular` | Angular services | @angular/core |
| `@ai-sdk/rsc` | React Server Components support | react |
| `@ai-sdk/vercel` | Vercel AI integration | - |

### Framework Package: React 19

The React package supports React 19 with peer dependencies:
```json
"react": "^18 || ~19.0.1 || ~19.1.2 || ^19.2.1"
```

## Core Packages Architecture

```
ai (main package)
  ├── @ai-sdk/provider (specifications/interfaces)
  ├── @ai-sdk/provider-utils (shared utilities)
  └── @ai-sdk/gateway (workspace:*)
```

### Provider Packages (30+ providers)

Examples: `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google`, `@ai-sdk/azure`, `@ai-sdk/amazon-bedrock`, etc.

Each provider follows the same pattern:
- Depends on `@ai-sdk/provider` (interface)
- Depends on `@ai-sdk/provider-utils` (shared implementation)
- Uses tsup for build

## Testing

### Test Framework

**Vitest** 4.1.0

Used across all packages. Each package has:
- `test` - runs both node and edge tests
- `test:node` - Node.js runtime only
- `test:edge` - Edge runtime only (via `@edge-runtime/vm`)

### Additional Test Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Playwright | 1.44.1 | E2E testing |
| `@testing-library/react` | 16.0.1 | React component testing |
| MSW | 2.6.4 | API mocking |
| jsdom | 24.0.0 | DOM emulation for React tests |

## Linting and Formatting

### Ultracite

**Version:** 7.3.2

Unified linting/formatting tool:
- `pnpm check` - Run linting (oxlint) and formatting (oxfmt) checks
- `pnpm fix` - Fix issues automatically

### Individual Tools

| Tool | Version | Purpose |
|------|---------|---------|
| oxlint | 1.56.0 | Linting |
| oxfmt | 0.41.0 | Formatting |

### Pre-commit Hooks

**husky** 9.1.7 + **lint-staged** 15.5.1

Runs `ultracite fix` on staged JS/TS/TSX files before commit.

## TypeScript Configuration

Root `tsconfig.json` uses project references for all packages:

```json
{
  "references": [
    { "path": "packages/ai" },
    { "path": "packages/provider" },
    // ... 50+ packages
  ],
  "compilerOptions": {
    "strictNullChecks": true
  }
}
```

**Note:** `include: []` is empty - all actual typing is in package-level tsconfigs.

Shared TypeScript config via `@vercel/ai-tsconfig` (workspace protocol).

## CI/CD

### GitHub Actions

Multiple workflows in `.github/workflows/`:

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Main CI pipeline |
| `release.yml` | NPM release process |
| `release-snapshot.yml` | Snapshot builds |
| `verify-changesets.yml` | Validates changeset entries |
| `ai-provider-models.yml` | Provider model tests |
| `ai-provider-api-changes.yml` | API change detection |

### Environment Variables (Build-time)

50+ API keys and configuration env vars declared in `turbo.json` for build tasks:
- Provider API keys: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, etc.
- AWS credentials for Bedrock
- Azure resources
- Vercel deployment vars

## Version Management

### Package Versioning

- Root: `"private": true` (not published)
- Core packages: `ai`, `@ai-sdk/react`, `@ai-sdk/*` use independent versioning
- Beta versions: `4.0.0-beta.X`, `7.0.0-beta.44`
- Changesets handle version bumps

## Dependency Overrides

```json
"pnpm": {
  "overrides": {
    "tinyexec": "1.0.2"
  },
  "onlyBuiltDependencies": [
    "esbuild"
  ]
}
```

## Key Observations

1. **Mature monorepo:** Well-structured pnpm + Turbo setup with proper task caching
2. **Multi-provider:** 30+ AI provider implementations with consistent patterns
3. **Multi-framework:** React, Vue, Svelte, Angular, RSC support
4. **TypeScript-first:** Strict null checks, project references, type testing
5. **Dual runtime:** Tests run on both Node.js and Edge runtimes
6. **No Docker:** No Dockerfile or docker-compose - deployment via Vercel
7. **No Nx/Lerna:** Pure Turbo + pnpm (simpler than Nx, more capable than Lerna)
8. **Comprehensive testing:** Vitest + Playwright + MSW for full coverage
9. **Automated releases:** Changesets + CI for semantic versioning
10. **ESM + CJS:** Both module formats via tsup bundling
