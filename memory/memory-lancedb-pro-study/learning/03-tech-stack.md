# Tech Stack

## Runtime

- **Node.js 22** -- enforced by CI via `actions/setup-node@v4` with `node-version: 22`
- **No `engines` field** in package.json (CI is the only enforcement)
- ESM-only project (`"type": "module"`)

## Package Manager

- **npm** -- `npm ci` for CI installs, `npm install` for development
- No pnpm, yarn, or monorepo tooling

## Build Approach

**No compilation step.** The project uses **jiti** (`^2.6.0`) to execute TypeScript directly at runtime. TypeScript (`^5.9.3`) is a devDependency used only for type checking during development, not as a build step. There is no `tsconfig.json` and no `tsc` script.

| Tool | Version | Role |
|------|---------|------|
| jiti | ^2.6.0 | TypeScript/ESM execution without compilation |
| TypeScript | ^5.9.3 | Type checking only (devDependency) |
| commander | ^14.0.0 | CLI argument parsing (devDependency) |

## Language

- **TypeScript** -- all source is `.ts` files; no JavaScript source files
- **ESM native** -- no CommonJS, fully ES modules
- **No tsconfig.json** found -- default TypeScript settings assumed

## Key Dependencies

### Vector Database & Storage

| Dependency | Version | Purpose |
|------------|---------|---------|
| @lancedb/lancedb | ^0.26.2 | LanceDB vector database for memory storage |
| apache-arrow | 18.1.0 | Columnar data format for LanceDB interop |

### LLM/AI Integration

| Dependency | Version | Purpose |
|------------|---------|---------|
| openai | ^6.21.0 | OpenAI API client for embeddings and LLM calls |

### Schema & Validation

| Dependency | Version | Purpose |
|------------|---------|---------|
| @sinclair/typebox | 0.34.48 | JSON schema TypeScript types for runtime validation |

### Utilities

| Dependency | Version | Purpose |
|------------|---------|---------|
| json5 | ^2.2.3 | JSON5 parser (for config files) |
| proper-lockfile | ^4.1.2 | File locking for cross-process safety |

## Framework & Architecture

### OpenClaw Plugin

This is an **OpenClaw plugin** -- a memory system for the OpenClaw AI coding agent. The plugin interface is defined in `openclaw.plugin.json`:

- **Plugin ID:** `memory-lancedb-pro`
- **Plugin Kind:** `memory`
- **Extension Entry:** `./index.ts`

### Project Structure

- **Single-package repository** -- no workspaces/monorepo
- **Entry point:** `index.ts` (main plugin implementation)
- **CLI:** `cli.ts` (~47KB -- substantial CLI implementation)
- **Source modules:** `src/` directory with 44 TypeScript modules
- **Tests:** `test/` directory with 27+ test files
- **Scripts:** `scripts/` directory for version syncing

### Architecture Patterns

- **TypeScript-first** -- all source is `.ts` files
- **ESM native** -- no CommonJS, fully ES modules
- **Plugin architecture** -- exports extensions via OpenClaw plugin manifest
- **Memory tiering** -- Core/Working/Peripheral memory classification
- **Hybrid retrieval** -- Vector + BM25 search with cross-encoder reranking
- **Admission control** -- A-MAC style filtering on write path

## Testing Approach

Tests use Node's built-in test runner (`node --test` for `.test.mjs` files) and direct execution for other test files. The `npm test` script chains 27+ test files including unit tests, integration tests, and E2E tests (`functional-e2e.mjs`, `context-support-e2e.mjs`).

## External Service Integrations

The plugin integrates with:

- **OpenAI API** -- embeddings and LLM
- **Jina AI** -- reranking service (cross-encoder)
- **Azure OpenAI** -- alternative embedding provider
- **OAuth** -- for LLM authentication (OpenAI Codex)

## Summary

| Aspect | Choice |
|--------|--------|
| Runtime | Node.js 22 |
| Module System | ESM (type: module) |
| Language | TypeScript |
| Build Approach | jiti (direct TS execution) |
| Package Manager | npm |
| Database | LanceDB |
| LLM | OpenAI SDK |
| Schema Validation | TypeBox |
| CLI | Commander |
| Testing | node --test + direct execution |
| CI/CD | GitHub Actions |
| Plugin Framework | OpenClaw |
| Monorepo | No |
