# Tech Stack & Choices Analysis

## Runtime/Engine Requirements

- **Node.js 22** (specified in CI pipeline via `actions/setup-node@v4` with `node-version: 22`)
- **ESM Modules** (`"type": "module"` in package.json)
- No explicit `engines` field in package.json, but CI enforces Node 22

## Package Manager

- **npm** (uses `npm ci` for CI installs)
- No pnpm, yarn, or monorepo tooling detected

## Build Tooling

| Tool | Version | Purpose |
|------|---------|---------|
| TypeScript | ^5.9.3 | Type checking (devDependency) |
| jiti | ^2.6.0 | TypeScript/ESM execution without compilation |
| commander | ^14.0.0 | CLI argument parsing (devDependency) |

**Build Approach:** No bundling/compilation step. The project uses **jiti** to execute TypeScript directly at runtime. Tests run via `node --test` (Node's built-in test runner) and direct `node` execution of `.mjs`/`.js` files.

**No tsc compilation step** - TypeScript is used for type checking only during development, not as a build step.

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
This is an **OpenClaw plugin** (a memory system for the OpenClaw AI coding agent). The plugin interface is defined in `openclaw.plugin.json` with:
- **Plugin ID:** `memory-lancedb-pro`
- **Plugin Kind:** `memory`
- **Extension Entry:** `./index.ts`

### Project Structure
- **Single-package repository** (no workspaces/monorepo)
- **Entry point:** `index.ts` (main plugin implementation)
- **CLI:** `cli.ts` (47KB - substantial CLI implementation)
- **Source modules:** `src/` directory with 44 TypeScript modules
- **Tests:** `test/` directory with 27+ test files
- **Scripts:** `scripts/` directory for version syncing

### Architecture Patterns
- **TypeScript-first** - All source is `.ts` files
- **ESM native** - No CommonJS, fully ES modules
- **Plugin architecture** - Exports extensions via OpenClaw plugin manifest
- **Memory tiering** - Core/Working/Peripheral memory classification
- **Hybrid retrieval** - Vector + BM25 search with cross-encoder reranking
- **Admission control** - A-MAC style filtering on write path

## TypeScript Configuration

**No tsconfig.json found** - TypeScript is installed but no explicit configuration. This implies:
- Default TypeScript settings are used
- Or configuration is in package.json (unlikely given no tsc script)

## CI/CD Configuration

GitHub Actions workflows in `.github/workflows/`:
- `ci.yml` - Main CI: version sync check + CLI smoke test
- `auto-assign.yml` - Issue auto-assignment
- `claude-code-review.yml` - Claude Code review automation
- `claude.yml` - Claude Code CI integration

CI runs on **Node 22** with `npm ci` install followed by `npm test`.

## Test Setup

Tests use Node's built-in test runner (`node --test` for .test.mjs files) and direct execution for other test files. Test command chains 27+ test files including:
- Unit tests with `--test` flag
- Integration tests as `.mjs` files
- E2E tests (`functional-e2e.mjs`, `context-support-e2e.mjs`)

## External Service Integrations

The plugin integrates with:
- **OpenAI API** - Embeddings and LLM
- **Jina AI** - Reranking service (cross-encoder)
- **Azure OpenAI** - Alternative embedding provider
- **OAuth** - For LLM authentication (OpenAI Codex)

## Summary Table

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
