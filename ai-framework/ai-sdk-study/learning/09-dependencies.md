# Dependencies

## Overview

The AI SDK manages dependencies through a pnpm workspace monorepo. Dependencies are split between production runtime dependencies (minimal) and development dependencies (extensive tooling).

## Workspace Structure

### Root Dependencies

Root `package.json` manages workspace-wide tooling:

```json
{
  "devDependencies": {
    "lint-staged": "^15.5.1",
    "@changesets/cli": "2.27.10",
    "@playwright/test": "^1.44.1",
    "del-cli": "^5.1.0",
    "husky": "^9.1.7",
    "next": "15.0.7",
    "oxfmt": "^0.41.0",
    "oxlint": "^1.56.0",
    "playwright": "^1.44.1",
    "publint": "0.2.12",
    "react": "19.0.0-rc-cc1ec60d0d-20240607",
    "react-dom": "19.0.0-rc-cc1ec60d0d-20240607",
    "turbo": "2.4.4",
    "typescript": "5.8.3",
    "ultracite": "7.3.2",
    "update-ts-references": "^3.6.0",
    "vitest": "4.1.0"
  }
}
```

### pnpm Configuration

```json
{
  "packageManager": "pnpm@10.11.0",
  "pnpm": {
    "onlyBuiltDependencies": ["esbuild"],
    "overrides": {
      "tinyexec": "1.0.2"
    }
  }
}
```

## Package-Level Dependencies

### @ai-sdk/provider (Core Interface)

Minimal production dependencies:

```json
{
  "dependencies": {
    "json-schema": "^0.4.0"
  }
}
```

### ai (Main SDK)

```json
{
  "dependencies": {
    "@ai-sdk/gateway": "workspace:*",
    "@ai-sdk/provider": "workspace:*",
    "@ai-sdk/provider-utils": "workspace:*",
    "@opentelemetry/api": "1.9.0"
  },
  "peerDependencies": {
    "zod": "^3.25.76 || ^4.1.8"
  }
}
```

### @ai-sdk/provider-utils

```json
{
  "dependencies": {
    "zod": "^3.25.76 || ^4.1.8"
  }
}
```

### Provider Packages

Provider packages typically have:
- Workspace dependency on `@ai-sdk/provider-utils`
- No additional production dependencies
- API key configuration via environment variables

### Framework Packages

Framework packages (React, Vue, Svelte, Angular):
- Workspace dependency on `ai` and `@ai-sdk/provider-utils`
- Framework-specific peer dependencies

## Dependency Types

### Production Dependencies

| Package | Purpose | Type |
|---------|---------|------|
| json-schema | JSON Schema validation | Runtime |
| zod | Schema validation | Runtime (peer) |
| @opentelemetry/api | Observability/tracing | Runtime |

### Development Dependencies

| Package | Purpose |
|---------|---------|
| turbo | Build orchestration |
| typescript | Type checking |
| vitest | Unit testing |
| @playwright/test | E2E testing |
| oxlint | Linting |
| oxfmt | Formatting |
| ultracite | Unified lint/format |
| @changesets/cli | Version management |
| tsup | TypeScript bundler |
| tsx | TypeScript executor |
| del-cli | Directory cleanup |
| husky | Git hooks |
| publint | Package linting |

## Version Constraints

### Node.js Engine Support

```json
"engines": {
  "node": "^18.0.0 || ^20.0.0 || ^22.0.0 || ^24.0.0"
}
```

Supports four major Node.js versions:
- Node 18 (LTS)
- Node 20 (LTS)
- Node 22 (LTS, recommended for development)
- Node 24 (Current)

### TypeScript Version

```json
"typescript": "5.8.3"
```

### Package Manager

```json
"packageManager": "pnpm@10.11.0"
```

## Workspace Dependencies

### Internal Package References

The SDK uses workspace protocol for internal packages:

```json
{
  "@ai-sdk/provider": "workspace:*",
  "@ai-sdk/provider-utils": "workspace:*",
  "@ai-sdk/gateway": "workspace:*",
  "@ai-sdk/test-server": "workspace:*",
  "@vercel/ai-tsconfig": "workspace:*"
}
```

Benefits:
- Always uses latest local version during development
- Ensures packages stay in sync
- No version drift between co-dependent packages

## External API Dependencies

### No Hard-Coded AI API Dependencies

Provider packages do NOT include AI SDKs as dependencies:
- OpenAI SDK - not included
- Anthropic SDK - not included
- Google AI SDK - not included

Instead, providers implement the `@ai-sdk/provider` interface by:
- Making HTTP requests directly to provider APIs
- Using lightweight fetch-based utilities
- Implementing provider-specific authentication

This approach:
- Reduces bundle size
- Avoids API version conflicts
- Allows fine-grained control over API calls

## Build-Time Dependencies

### esbuild (onlyBuiltDependencies)

```json
"pnpm": {
  "onlyBuiltDependencies": ["esbuild"]
}
```

esbuild is the only package allowed to run post-install scripts.

## Dependency Management Commands

```bash
pnpm install              # Install all dependencies
pnpm update-references    # Update tsconfig.json references after adding deps
pnpm clean                # Clean all packages
```

## Dependency Conflicts

### Zod Version Handling

The SDK supports both Zod 3 and Zod 4:

```typescript
// Zod 3 imports (for compatibility code)
import * as z3 from 'zod/v3';

// Zod 4 imports
import * as z4 from 'zod/v4';
```

This is necessary because:
- Some internal code uses Zod 3 patterns
- External packages may depend on different Zod versions
- Migration to Zod 4 is gradual

### tinyexec Override

```json
"overrides": {
  "tinyexec": "1.0.2"
}
```

Forces a specific version of tinyexec to avoid conflicts.

## Lockfile Management

- Uses `pnpm-lock.yaml` (v10.11.0 format)
- CI uses `--frozen-lockfile` for reproducible builds
- Development allows lockfile updates

## Adding New Dependencies

When adding dependencies between packages:

```bash
pnpm update-references
```

This updates the `references` section in `tsconfig.json` files to maintain proper build order.
