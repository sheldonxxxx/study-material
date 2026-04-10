# Vercel AI SDK - Project Overview

## Project Purpose

The Vercel AI SDK is a TypeScript/JavaScript software development kit for building AI-powered applications with Large Language Models (LLMs). It provides a unified interface across multiple AI providers, enabling developers to build applications that can switch between providers or support multiple providers without vendor lock-in.

## Project Scope

### Core Capabilities

- **Multi-Provider Support**: First-class integrations with OpenAI, Anthropic, Google, Azure, Amazon Bedrock, Cohere, DeepSeek, Mistral, Groq, Fireworks, and more
- **Multi-Framework Support**: Official integrations for React, Vue, Svelte, Angular, and React Server Components
- **Generative Features**: Text generation, object generation, image generation, speech synthesis, transcription, video generation, embeddings, and reranking
- **Streaming**: First-class streaming support with transform streams for processing AI responses
- **Telemetry**: OpenTelemetry integration for observability with privacy controls

### Architecture

The SDK follows a layered provider architecture:

```
ai (core functions)
    └── @ai-sdk/provider-utils (shared utilities)
            └── @ai-sdk/provider (interface specifications)

@ai-sdk/<provider> (concrete implementations)
    └── @ai-sdk/provider-utils
            └── @ai-sdk/provider
```

**Key layers:**
1. **Specifications** (`@ai-sdk/provider`): Defines interfaces like `LanguageModelV4`
2. **Utilities** (`@ai-sdk/provider-utils`): Shared code for implementing providers
3. **Providers** (`@ai-sdk/<provider>`): Concrete implementations for each AI service
4. **Core** (`ai`): High-level functions like `generateText`, `streamText`, `generateObject`

### Package Structure

**55 packages** organized into:

| Category | Count | Examples |
|----------|-------|----------|
| Core | 4 | `packages/ai`, `packages/provider`, `packages/provider-utils`, `packages/rsc` |
| Providers | 33 | `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google` |
| Frameworks | 5 | `packages/react`, `packages/vue`, `packages/svelte`, `packages/angular` |
| Utilities | 13 | `packages/gateway`, `packages/devtools`, `packages/codemod` |

## What Makes It Interesting

### Unified Provider Interface

The SDK defines strict interface specifications that all providers must implement. This means you can swap from OpenAI to Anthropic by changing a single line of code:

```typescript
import { createOpenAI } from '@ai-sdk/openai';
import { createAnthropic } from '@ai-sdk/anthropic';

// Swap providers without changing application logic
const model = createOpenAI({ apiKey: process.env.OPENAI_API_KEY });
// const model = createAnthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

### Streaming Architecture

The SDK uses Web Streams API (`TransformStream`) for memory-efficient streaming. It supports:

- **Transform streams**: Chain processing steps for model outputs
- **Smooth streaming**: Configurable output pacing with word/line/regex chunking
- **Tool streaming**: Real-time tool call streaming with support for partial JSON

### Type Safety

The SDK provides comprehensive TypeScript support with:

- Strict mode enabled via shared `tsconfig`
- Type testing via dedicated `.test-d.ts` files using `expectTypeOf()`
- 21+ custom error types with structured data
- Typed configuration objects with explicit validation

### React Framework Integration

First-class React hooks (`useChat`, `useCompletion`, `useObject`) provide:

- Automatic prompt management
- Streaming UI updates with throttling
- Message history management
- Loading and error states

### Telemetry with Privacy Controls

OpenTelemetry integration with user-controlled data recording:

- Telemetry disabled by default
- `recordInputs` and `recordOutputs` flags for privacy
- Lazy attribute evaluation prevents unnecessary computation

### Developer Experience

- **23 example applications** demonstrating various frameworks and providers
- **MDX documentation** with cookbook guides
- **Changesets** for version management
- **Claude Code skills** for development assistance

## Technical Stack

| Component | Technology |
|-----------|------------|
| Package Manager | pnpm (10.11.0) |
| Monorepo | Turborepo (2.4.4) |
| Bundler | tsup |
| Testing | Vitest |
| Linting | Ultracite (oxlint + oxfmt) |
| TypeScript | Strict mode, ES2018 target |
| Node Support | ^18, ^20, ^22, ^24 |

## Repository

- **URL**: https://github.com/vercel/ai
- **License**: Apache-2.0
- **Type**: Monorepo (pnpm workspaces + Turborepo)
