# Tech Stack

## Overview

The Vercel AI SDK is a TypeScript/JavaScript monorepo for building AI-powered applications with Large Language Models (LLMs). It provides a unified interface across multiple AI providers with first-class framework integrations.

## Core Technologies

### Runtime & Language
- **Node.js**: v18, v20, v22, v24 (development uses v22)
- **TypeScript**: 5.8.3 (strict mode)
- **Package Manager**: pnpm v10.11.0 (workspace configuration)

### Build & Development
- **Turborepo**: 2.4.4 (build orchestration, task caching)
- **tsup**: ESM-first TypeScript bundler for packages
- **tsx**: TypeScript execute for examples/benchmarks
- **esbuild**: 0.24.2 (fast JavaScript/TypeScript compilation)

### Testing
- **Vitest**: 4.1.0 (unit testing framework)
- **Playwright**: 1.44.1 (E2E testing)
- **@playwright/test**: 1.44.1

### Code Quality
- **oxlint**: 1.56.0 (linting)
- **oxfmt**: 0.41.0 (formatting)
- **ultracite**: 7.3.2 (unified lint/format runner)
- **@changesets/cli**: 2.27.10 (versioning and changelog)

### Framework Integrations
The SDK supports multiple UI frameworks:
- **React**: 19.0.0-rc
- **Next.js**: 15.0.7
- **Vue**: Framework integration packages
- **Svelte**: Framework integration packages
- **Angular**: 17+ (standalone components)
- **SvelteKit**: Via framework packages
- **Nuxt**: Via framework packages

## Architecture

### Monorepo Structure

```
ai-repo/
  packages/
    ai/                    # Main SDK package
    provider/              # Provider interface specifications
    provider-utils/        # Shared utilities
    <provider>/           # AI provider implementations
    <framework>/          # UI framework integrations
    codemod/              # Automated migrations
  examples/               # Example applications
  content/                # Documentation (MDX)
  contributing/           # Contributor guides
```

### Package Dependency Graph

```
ai ─────────────────┬──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
                    │
@ai-sdk/<provider> ─┴──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
```

### Provider Implementations (30+)

| Provider | Package | Category |
|----------|---------|----------|
| OpenAI | @ai-sdk/openai | LLM |
| Anthropic | @ai-sdk/anthropic | LLM |
| Google | @ai-sdk/google, @ai-sdk/google-vertex | LLM |
| AWS Bedrock | @ai-sdk/amazon-bedrock | LLM |
| Azure | @ai-sdk/azure | LLM |
| Mistral | @ai-sdk/mistral | LLM |
| Cohere | @ai-sdk/cohere | LLM |
| DeepSeek | @ai-sdk/deepseek | LLM |
| xAI | @ai-sdk/xai | LLM |
| Hugging Face | @ai-sdk/huggingface | LLM |
| Fireworks | @ai-sdk/fireworks | LLM |
| Together AI | @ai-sdk/togetherai | LLM |
| Perplexity | @ai-sdk/perplexity | LLM |
| Groq | @ai-sdk/groq | LLM |
| Replicate | @ai-sdk/replicate | LLM |
| OpenAI Compatible | @ai-sdk/openai-compatible | LLM |

### Framework Packages

| Framework | Package | Path |
|-----------|---------|------|
| React | @ai-sdk/react | packages/react |
| Next.js RSC | @ai-sdk/rsc | packages/rsc |
| Vue | @ai-sdk/vue | packages/vue |
| Svelte | @ai-sdk/svelte | packages/svelte |
| Angular | @ai-sdk/angular | packages/angular |

## Key Dependencies

### Production Dependencies (Core Packages)

**@ai-sdk/provider** (minimal):
- json-schema: ^0.4.0

**ai** (main SDK):
- @ai-sdk/provider: workspace:*
- @ai-sdk/provider-utils: workspace:*
- @ai-sdk/gateway: workspace:*
- @opentelemetry/api: 1.9.0

**Provider Utils**:
- zod: 3.25.76 | 4.1.8 (peer dependency)

### Development Dependencies

```json
{
  "turbo": "2.4.4",
  "typescript": "5.8.3",
  "vitest": "4.1.0",
  "oxlint": "1.56.0",
  "oxfmt": "0.41.0",
  "ultracite": "7.3.2",
  "@changesets/cli": "2.27.10",
  "@playwright/test": "1.44.1",
  "tsup": "^7.2.0 | ^8",
  "tsx": "^4.19.2"
}
```

## Build Pipeline

### Turborepo Task Configuration

| Task | Depends On | Outputs |
|------|-----------|---------|
| build | ^build (dependencies) | dist/**, .next/**, .nuxt/** |
| type-check | ^build, build | .tsbuildinfo |
| test | ^build, build | test results |
| publint | ^build, build | lint output |
| clean | ^clean | - |
| dev | - | cache=false, persistent=true |

### Build Outputs

- **packages**: ESM (.mjs) + CJS (.js) + TypeScript declarations (.d.ts)
- **examples**: Next.js (.next/), Nuxt (.nuxt/), SvelteKit (.svelte-kit/)

## Environment Configuration

### Node.js Version Policy

```json
"engines": {
  "node": "^18.0.0 || ^20.0.0 || ^22.0.0 || ^24.0.0"
}
```

### pnpm Workspace Configuration

```json
"packageManager": "pnpm@10.11.0",
"pnpm": {
  "onlyBuiltDependencies": ["esbuild"],
  "overrides": {
    "tinyexec": "1.0.2"
  }
}
```

## Schema Validation

### Zod Support

The SDK supports both Zod 3 and Zod 4 with proper import separation:

```typescript
// Zod 3 (compatibility)
import * as z3 from 'zod/v3';

// Zod 4
import * as z4 from 'zod/v4';
import { z4 as z } from 'zod/v4'; // Alias for convenience
```

Peer dependency constraint:
```json
"zod": "^3.25.76 || ^4.1.8"
```
