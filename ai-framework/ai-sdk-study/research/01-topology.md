# Repository Topology: Vercel AI SDK

## Project Identity

**Name:** Vercel AI SDK
**Repository:** https://github.com/vercel/ai
**Type:** Monorepo (pnpm workspaces + Turborepo)
**License:** Apache-2.0

The AI SDK is a TypeScript/JavaScript SDK for building AI-powered applications with Large Language Models (LLMs). It provides a unified interface for multiple AI providers and framework integrations.

---

## Directory Structure

```
/reference/ai/
├── .changeset/           # Changesets for version management
├── .claude/              # Claude Code settings
├── .github/              # GitHub workflows and scripts
├── .husky/               # Git hooks
├── architecture/         # Architecture decision records (ADRs)
├── assets/               # Static assets
├── content/              # Documentation source (MDX)
│   ├── cookbook/
│   ├── docs/
│   ├── providers/
│   └── tools-registry/
├── contributing/         # Contributor guides
├── examples/             # Example applications (23 examples)
│   ├── ai-e2e-next/
│   ├── ai-functions/
│   ├── angular/
│   ├── express/
│   ├── fastify/
│   ├── hono/
│   ├── mcp/
│   ├── nest/
│   ├── next/             # Next.js example
│   ├── next-agent/
│   ├── next-fastapi/
│   ├── next-google-vertex/
│   ├── next-langchain/
│   ├── next-openai-*/
│   ├── nuxt-openai/
│   ├── sveltekit-openai/
│   └── ...
├── packages/             # 55 packages (core + providers + frameworks)
├── skills/               # Claude Code skills for development
├── tools/                # Internal tooling
│   ├── analyze-downloads/
│   ├── generate-llms-txt/
│   └── tsconfig/
├── pnpm-lock.yaml       # Lock file
├── pnpm-workspace.yaml   # Workspace config
├── turbo.json            # Turborepo config
├── tsconfig.json         # Root TypeScript config
├── package.json          # Root package.json
└── CLAUDE.md             # Symlink to AGENTS.md (developer guide)
```

---

## Packages Overview (55 packages)

### Core Packages

| Package | Purpose | Entry Point |
|---------|---------|-------------|
| `packages/ai` | Main SDK (`ai` on npm) | `src/index.ts` |
| `packages/provider` | Provider interface specs (`@ai-sdk/provider`) | `src/index.ts` |
| `packages/provider-utils` | Shared utilities (`@ai-sdk/provider-utils`) | - |
| `packages/rsc` | React Server Components support | `src/index.ts` |

### Provider Packages (33)

Each implements AI provider integration:

`@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google`, `@ai-sdk/google-vertex`, `@ai-sdk/azure`, `@ai-sdk/amazon-bedrock`, `@ai-sdk/cohere`, `@ai-sdk/deepseek`, `@ai-sdk/mistral`, `@ai-sdk/groq`, `@ai-sdk/fireworks`, `@ai-sdk/bedrock`, `@ai-sdk/openai-compatible`, and others.

### Framework Packages (5)

| Package | Framework |
|---------|-----------|
| `packages/react` | React hooks (`useChat`, `useCompletion`, `useObject`) |
| `packages/vue` | Vue composables |
| `packages/svelte` | Svelte stores |
| `packages/angular` | Angular services |
| `packages/rsc` | React Server Components |

### Utility Packages

- `packages/gateway` - Gateway service
- `packages/devtools` - Developer tools
- `packages/codemod` - Automated migrations

---

## Key Entry Points

### 1. Main SDK Entry: `packages/ai/src/index.ts`

```typescript
// Core functions
export * from './agent';
export * from './embed';
export * from './generate-image';
export * from './generate-object';
export * from './generate-speech';
export * from './generate-text';
export * from './generate-video';
export * from './transcribe';
// etc.
```

### 2. Provider Interface: `packages/provider/src/index.ts`

```typescript
// Exports model interfaces (LanguageModelV4, EmbeddingModelV3, etc.)
// Defines provider contract
export * from './language-model';
export * from './embedding-model';
export * from './image-model';
// etc.
```

### 3. React Hooks: `packages/react/src/`

- `use-chat.ts` - Chat UI hook
- `use-completion.ts` - Completion hook
- `use-object.ts` - Object generation hook
- `index.ts` - Main exports

### 4. Example Entry Points

**Next.js example:** `examples/next/app/`
- `app/page.tsx` - Home page
- `app/chat/` - Chat example route
- `app/api/` - API routes

**AI Functions examples:** `examples/ai-functions/src/`
- Structured by function: `stream-text/`, `generate-text/`, etc.
- Structured by provider: `openai/`, `anthropic/`, etc.

---

## Dependency Architecture

```
ai ─────────────────┬──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
                    │
@ai-sdk/<provider> ─┴──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
```

The SDK follows a layered provider architecture:

1. **Specifications** (`@ai-sdk/provider`): Defines interfaces like `LanguageModelV4`
2. **Utilities** (`@ai-sdk/provider-utils`): Shared code for implementing providers
3. **Providers** (`@ai-sdk/<provider>`): Concrete implementations for each AI service
4. **Core** (`ai`): High-level functions like `generateText`, `streamText`, `generateObject`

---

## Development Commands

| Command | Purpose |
|---------|---------|
| `pnpm install` | Install dependencies |
| `pnpm build` | Build all packages |
| `pnpm test` | Run all tests (excludes examples) |
| `pnpm check` | Run linting and formatting |
| `pnpm fix` | Fix linting/formatting issues |
| `pnpm changeset` | Add changeset for PR |
| `pnpm type-check:full` | Full TypeScript check |

---

## File Naming Conventions

- Source files: `kebab-case.ts`
- Test files: `kebab-case.test.ts`
- Type test files: `kebab-case.test-d.ts`
- React/UI components: `kebab-case.tsx`
- Fixtures: `__fixtures__/` subfolders
- Snapshots: `__snapshots__/` subfolders

---

## Build System

- **Bundler:** tsup (configured via `tsup.config.ts` in each package)
- **Monorepo:** Turborepo (2.4.4)
- **Package Manager:** pnpm (10.11.0)
- **Node Version:** ^18.0.0 || ^20.0.0 || ^22.0.0 || ^24.0.0
