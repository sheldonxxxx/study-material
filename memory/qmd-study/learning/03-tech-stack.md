# QMD Tech Stack

## Language and Runtime

| Aspect | Details |
|--------|---------|
| **Language** | TypeScript (ESNext target) |
| **Module System** | nodenext (with verbatimModuleSyntax) |
| **Strictness** | `strict`, `noUncheckedIndexedAccess`, `noImplicitOverride` |
| **Node.js** | >= 22.0.0 |
| **Bun** | >= 1.0.0 (dual support with npm) |
| **TypeScript** | ^5.9.3 (peer dependency) |

The project maintains strict TypeScript configuration with multiple safety flags enabled. Dual runtime support is achieved via a shell wrapper that auto-detects package manager via lockfile presence.

## Core Dependencies

### Search and AI

| Package | Version | Purpose |
|---------|---------|---------|
| `better-sqlite3` | ^12.4.5 | Synchronous SQLite bindings |
| `sqlite-vec` | ^0.1.7-alpha.2 | Vector similarity search via SQLite extension |
| `node-llama-cpp` | ^3.17.1 | Local LLM inference with GGUF models |
| `@modelcontextprotocol/sdk` | ^1.25.1 | MCP server implementation |

### Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `fast-glob` | ^3.3.0 | High-performance file globbing |
| `picomatch` | ^4.0.0 | Glob matching |
| `yaml` | ^2.8.2 | Configuration file parsing |
| `zod` | ^4.2.1 | Runtime schema validation |

### Optional Platform-Specific

Pre-built `sqlite-vec` binaries for macOS (arm64/x64), Linux (arm64/x64), and Windows.

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `tsx` | ^4.0.0 | TypeScript execution |
| `vitest` | ^3.0.0 | Testing framework |
| `@types/better-sqlite3` | ^7.6.0 | TypeScript types |

## Build System

### TypeScript Configuration

Two `tsconfig` files:
- **tsconfig.json** (dev): `noEmit: true`, strict mode
- **tsconfig.build.json** (prod): emits to `dist/`, generates `.d.ts` declarations, excludes tests

### Build Pipeline

```
npm run build
  -> tsc -p tsconfig.build.json
  -> Prepend shebang to dist/cli/qmd.js
  -> chmod +x dist/cli/qmd.js
```

### CLI Entry Point

- **Shell wrapper**: `bin/qmd` - POSIX shell that resolves symlinks and auto-detects npm vs Bun
- **Source**: `src/cli/qmd.ts`
- **Compiled**: `dist/cli/qmd.js`

### Package Scripts

| Script | Command |
|--------|---------|
| `prepare` | `git hooks install` |
| `build` | `tsc -p tsconfig.build.json` |
| `test` | `vitest run --reporter=verbose test/` |
| `qmd` | `tsx src/cli/qmd.ts` |
| `release` | `./scripts/release.sh` |

## LLM Models (via node-llama-cpp)

Three GGUF models auto-downloaded on first use:

| Model | Size | Purpose |
|-------|------|---------|
| `embeddinggemma-300M-Q8_0` | ~300MB | Vector embeddings (default) |
| `qwen3-reranker-0.6b-q8_0` | ~640MB | Re-ranking |
| `qmd-query-expansion-1.7B-q4_k_m` | ~1.1GB | Query expansion (fine-tuned) |

Models cached in `~/.cache/qmd/models/`.

## Architecture

```
Search Pipeline:
  Query → BM25 (FTS5) ─┐
                       ├─→ Reciprocal Rank Fusion ─→ Results
  Query → Vector (vec) ┘
                       └─→ LLM Reranking (optional)
```

## Development Environments

- **Nix flakes**: `flake.nix` provides Bun, SQLite with extensions, Darwin cctools
- **Docker**: `test/Containerfile` for CI-like testing
- **Git hooks**: `prepare` script installs hooks via `scripts/install-hooks.sh`

## Storage

- **Index database**: `~/.cache/qmd/index.sqlite`
- **Models cache**: `~/.cache/qmd/models/`
- **Smart chunking**: 900 tokens per chunk with 15% overlap, prefers markdown heading boundaries

## Summary

| Category | Choice |
|----------|--------|
| Language | TypeScript (strict mode) |
| Runtimes | Node.js >= 22, Bun |
| Build | TypeScript compiler (tsc) |
| Testing | Vitest v3 |
| Database | SQLite with FTS5 + sqlite-vec |
| AI/ML | node-llama-cpp (local GGUF models) |
| CLI | Shell wrapper auto-detecting npm/Bun |
| Protocol | MCP (Model Context Protocol) |
