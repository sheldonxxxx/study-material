# Vercel AI SDK — Learning Reference

## What This Is
Analysis of the [Vercel AI SDK](https://github.com/vercel/ai) codebase for informing development of a similar AI SDK / provider abstraction layer.

## Key Takeaways
- **Layered architecture works**: `ai` (core API) → `provider` (interface specs) → `provider-utils` (shared logic) → `<provider-*>` (implementations) provides clean separation and extensibility
- **Interface versioning is essential**: The v2/v3/v4 coexistence with adapter functions (`asLanguageModelV4`) enables smooth migrations without breaking existing providers
- **Step loop pattern unifies text, tools, and agents**: `generateText` implements the agent loop internally — `ToolLoopAgent` is just a thin configuration wrapper
- **Framework abstraction via shared core**: `AbstractChat` + `ChatState` allows React/Vue/Svelte/Angular to share all business logic while using native reactivity primitives
- **Feature maturity is uneven**: Core text generation is extremely polished; speech/video are clearly `experimental_` with no telemetry — expect the same if building a similar SDK

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, provider interface design, error hierarchy, streaming implementation, type testing approach
- **Do Not Trust**: README simplicity claims — the actual codebase is complex; `experimental_` prefix features are not production-ready; linter configuration has many disabled rules

## At a Glance
- **Language:** TypeScript 5.8.3 (strict)
- **Architecture:** Layered provider abstraction with adapter pattern
- **Key Libraries:** Zod (schema validation), Web Streams API, OpenTelemetry, Vitest, Turborepo, pnpm
- **Notable Patterns:** Step loop for agents, OutputStrategy for structured output, middleware wrapping, DelayedPromise for lazy streams
- **Stars / Activity:** Active Vercel-sponsored project with regular beta releases, 40+ provider packages
- **License:** Likely Apache 2.0 or similar — verify before copying
