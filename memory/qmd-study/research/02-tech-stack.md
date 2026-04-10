# QMD Tech Stack Analysis

## Language and Version Constraints

| Aspect | Version/Details |
|--------|-----------------|
| **Primary Language** | TypeScript |
| **TypeScript Target** | ESNext |
| **TypeScript Module** | nodenext (with verbatimModuleSyntax) |
| **Node.js Requirement** | >= 22.0.0 |
| **Package Managers** | npm (package-lock.json) and Bun (bun.lock) - dual support |
| **Runtime Detection** | Shell wrapper auto-detects npm vs Bun via lockfile presence |

The project uses strict TypeScript with `noUncheckedIndexedAccess`, `noImplicitOverride`, and `strict` mode enabled.

## Key Dependencies and Their Purposes

### Core Search/AI Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `better-sqlite3` | ^12.4.5 | Synchronous SQLite bindings for local database operations |
| `sqlite-vec` | ^0.1.7-alpha.2 | SQLite extension for vector similarity search |
| `node-llama-cpp` | ^3.17.1 | Local LLM inference using GGUF models (embeddings, reranking, query expansion) |
| `@modelcontextprotocol/sdk` | ^1.25.1 | MCP (Model Context Protocol) server implementation |

### Utility Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `fast-glob` | ^3.3.0 | High-performance file globbing |
| `picomatch` | ^4.0.0 | Glob matching library |
| `yaml` | ^2.8.2 | YAML parsing for configuration |
| `zod` | ^4.2.1 | Runtime schema validation |

### Optional Platform-Specific Dependencies

The project ships pre-built `sqlite-vec` binaries for:
- `sqlite-vec-darwin-arm64` (Apple Silicon)
- `sqlite-vec-darwin-x64` (Intel Mac)
- `sqlite-vec-linux-arm64`
- `sqlite-vec-linux-x64`
- `sqlite-vec-windows-x64`

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `tsx` | ^4.0.0 | TypeScript execution (tsx runner) |
| `vitest` | ^3.0.0 | Testing framework |
| `@types/better-sqlite3` | ^7.6.0 | TypeScript types for better-sqlite3 |

## Build System and Tooling

### TypeScript Configuration

**tsconfig.json** (development/base):
- Target: ESNext
- Module: nodenext
- Strict mode enabled
- `noEmit: true` (no compilation in dev mode)

**tsconfig.build.json** (production):
- Extends tsconfig.json
- `noEmit: false`, outputs to `dist/`
- Generates declarations (`dist/index.d.ts`)
- Excludes test files, benchmarks, and preload scripts

### Build Process

```bash
npm run build
# or: bun run build
```

The build command:
1. Compiles TypeScript via `tsc -p tsconfig.build.json`
2. Prepends shebang `#!/usr/bin/env node` to dist/cli/qmd.js
3. Makes the CLI executable

### CLI Entry Point

- **Shell wrapper**: `bin/qmd` - POSIX shell script that:
  - Resolves symlinks for global installs
  - Auto-detects runtime (npm vs Bun) via lockfile detection
  - Routes to appropriate runtime with `dist/cli/qmd.js`

- **Source entry**: `src/cli/qmd.ts` - Main CLI implementation
- **Compiled output**: `dist/cli/qmd.js`

### Package Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `prepare` | `git hooks install` | Sets up git hooks on `npm install` |
| `build` | `tsc -p tsconfig.build.json` | Compiles TypeScript |
| `test` | `vitest run --reporter=verbose test/` | Run tests with vitest |
| `qmd` | `tsx src/cli/qmd.ts` | Run CLI directly from source |
| `index` | `tsx src/cli/qmd.ts index` | Index command |
| `vector` | `tsx src/cli/qmd.ts vector` | Vector operations |
| `search` | `tsx src/cli/qmd.ts search` | BM25 search |
| `vsearch` | `tsx src/cli/qmd.ts vsearch` | Vector search |
| `rerank` | `tsx src/cli/qmd.ts rerank` | Reranking |
| `inspector` | MCP inspector | Debug MCP server |
| `release` | `./scripts/release.sh` | Release workflow |

## Test Framework

### Vitest Configuration

Located in `vitest.config.ts`:
- Test timeout: 30 seconds
- Test pattern: `test/**/*.test.ts`
- Uses Vitest's defineConfig

### Test Files (in `test/` directory)

| File | Purpose |
|------|---------|
| `cli.test.ts` | CLI integration tests |
| `collections-config.test.ts` | Collection configuration tests |
| `eval-bm25.test.ts` | BM25 ranking evaluation |
| `eval.test.ts` | General evaluation tests |
| `eval-harness.ts` | Evaluation test harness |
| `formatter.test.ts` | Output formatting tests |
| `intent.test.ts` | Intent detection tests |
| `llm.test.ts` | LLM integration tests |
| `mcp.test.ts` | MCP server tests |
| `multi-collection-filter.test.ts` | Multi-collection filtering |
| `rrf-trace.test.ts` | Reciprocal Rank Fusion tracing |
| `sdk.test.ts` | MCP SDK tests |
| `store-paths.test.ts` | Storage path handling |
| `store.helpers.unit.test.ts` | Store helper unit tests |
| `store.test.ts` | Core storage tests |
| `structured-search.test.ts` | Structured search tests |

### CI Testing

**Node.js tests** (`test-node` job):
- Runs on Ubuntu and macOS
- Tests Node versions 22 and 23
- Uses `npx vitest run --reporter=verbose --testTimeout 60000`
- Requires `libsqlite3-dev` (Linux) or `sqlite` (macOS)

**Bun tests** (`test-bun` job):
- Runs on Ubuntu and macOS
- Uses `bun test --timeout 60000 --preload ./src/test-preload.ts`
- Requires setting `DYLD_LIBRARY_PATH` (macOS) and `LD_LIBRARY_PATH` (Linux)

## Special Runtime and Deployment Requirements

### SQLite Extensions

The project requires SQLite with loadable extension support:
- **FTS5**: Built-in, used for BM25 full-text search
- **sqlite-vec**: Loadable extension for vector similarity search

### LLM Models (via node-llama-cpp)

The project uses local LLMs for:
- **Embeddings**: embeddinggemma model
- **Reranking**: qwen3-reranker model
- **Query Expansion**: Qwen3 model

### Architecture Overview

```
Search Pipeline:
  Query → BM25 (FTS5) ─┐
                       ├─→ Reciprocal Rank Fusion ─→ Results
  Query → Vector (vec) ┘
                       └─→ LLM Reranking (optional)
```

### Data Storage

- **Index database**: `~/.cache/qmd/index.sqlite`
- **Smart chunking**: 900 tokens per chunk with 15% overlap
- Prefers markdown heading boundaries for chunking

## Development Environment Setup

### Local Development

1. **Clone and install**:
   ```bash
   npm install  # or: bun install
   ```

2. **Run from source**:
   ```bash
   bun src/cli/qmd.ts <command>
   ```

3. **Run tests**:
   ```bash
   npm test  # or: bun test
   ```

### Nix Development Environment

The project includes `flake.nix` for reproducible development:

```bash
nix develop
```

The flake provides:
- Bun runtime
- SQLite with loadable extension support (custom override)
- Darwin cctools on macOS (for node-gyp)
- Python3 (for node-gyp compilation)

### Docker/Container

A `Containerfile` exists in `test/` for testing:
- Base: Debian Bookworm slim
- Installs mise for runtime management
- Pre-installs both Node.js and Bun
- Installs qmd via both npm and Bun
- Test project copied to `/opt/qmd/`

### Git Hooks

The `prepare` script installs git hooks via `scripts/install-hooks.sh`:
- Uses `pre-push` hook for pre-push checks

## Summary

| Category | Choice |
|----------|--------|
| **Language** | TypeScript (ESNext, strict mode) |
| **Runtimes** | Node.js >= 22, Bun |
| **Build** | TypeScript compiler (tsc) |
| **Testing** | Vitest v3 |
| **Database** | SQLite with FTS5 + sqlite-vec |
| **AI/ML** | node-llama-cpp (local GGUF models) |
| **CLI** | Shell wrapper auto-detecting npm/Bun |
| **Protocol** | MCP (Model Context Protocol) |
| **DevEnv** | Nix flakes, Docker, Git hooks |
| **CI** | GitHub Actions (Ubuntu + macOS, Node + Bun) |
