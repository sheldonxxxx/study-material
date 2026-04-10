# QMD Project Topology Analysis

**Repo path:** `/Users/sheldon/Documents/claw/reference/qmd/`
**Analysis date:** 2026-03-27
**Project:** @tobilu/qmd v2.0.1

---

## 1. Root-Level File Inventory

| File/Directory | Purpose |
|----------------|---------|
| `package.json` | npm package definition, ESM module, CLI bin entry |
| `tsconfig.json` | TypeScript configuration |
| `tsconfig.build.json` | Build-specific TS config (produces dist/) |
| `vitest.config.ts` | Test runner configuration |
| `CLAUDE.md` | Project-specific guidelines for Claude Code |
| `README.md` | Project documentation |
| `CHANGELOG.md` | Release history |
| `LICENSE` | MIT license |
| `flake.nix`, `flake.lock` | Nix flake for reproducibility |
| `migrate-schema.ts` | Database migration script |
| `example-index.yml` | Example YAML configuration |

**Notable:** No `jest.config`, `eslint.config`, `.prettierrc` — tests use Vitest, linting not prominent.

---

## 2. Directory Tree (Depth 3)

```
qmd/
├── .claude-plugin/          # Claude Code plugin configuration
├── .github/                 # GitHub workflows/actions
├── .pi/                     # Private/placeholder directory
├── assets/                  # Static assets (architecture diagram)
├── bin/
│   └── qmd                  # Shell wrapper for CLI (detects node/bun runtime)
├── docs/
│   └── SYNTAX.md            # QMD file syntax documentation
├── finetune/                # Model fine-tuning experiments (Python, separate concern)
│   ├── configs/
│   ├── data/
│   ├── dataset/
│   ├── evals/
│   ├── experiments/
│   ├── jobs/
│   ├── train.py
│   ├── reward.py
│   ├── eval.py
│   └── ...
├── qmd-study/               # Study/research output directory
├── scripts/                 # Build/release scripts
│   ├── install-hooks.sh
│   ├── extract-changelog.sh
│   ├── pre-push
│   └── release.sh
├── skills/                  # Claude Code skills
│   ├── qmd/
│   └── release/
├── src/                     # Main TypeScript source (see depth 4 below)
├── test/                    # Test suite (Vitest)
│   ├── *.test.ts            # Unit/integration tests
│   ├── eval-docs/           # Test data for evaluations
│   └── eval-harness.ts      # Evaluation framework
├── bun.lock                 # Bun lockfile (primary package manager)
└── package.json
```

### src/ Depth 4

```
src/
├── index.ts                 # SDK entry point — exports createStore, QMDStore interface
├── store.ts                 # Core database operations, search, indexing (~154KB, largest file)
├── llm.ts                   # LlamaCpp wrapper for embeddings/reranking (~50KB)
├── collections.ts           # Collection YAML config management
├── db.ts                    # SQLite database initialization
├── embedded-skills.ts       # Embedded Claude Code skills
├── maintenance.ts          # Housekeeping operations
├── bench-rerank.ts          # Reranking benchmark tool
├── test-preload.ts          # Vitest preload script
├── cli/
│   ├── qmd.ts               # CLI entry point (~115KB, largest CLI file)
│   └── formatter.ts         # Output formatting (JSON, CSV, XML, MD, files)
└── mcp/
    └── server.ts            # MCP server implementation (~30KB)
```

---

## 3. Key Entry Points

### CLI Entry Point
**File:** `bin/qmd` (shell wrapper) -> `dist/cli/qmd.js` (compiled)

The `bin/qmd` shell script:
- Resolves symlinks for global npm/bun installs
- Detects runtime (node vs bun) by checking lockfiles
- Delegates to the compiled JavaScript in `dist/cli/qmd.js`

**Development mode:** `bun src/cli/qmd.ts <command>` or `npm run qmd`

**Production mode:** `qmd <command>` (global install) or `npx @tobilu/qmd <command>`

### SDK Entry Point
**File:** `src/index.ts`

Exports `createStore()` function and `QMDStore` interface for programmatic access:

```typescript
import { createStore } from '@tobilu/qmd'
const store = await createStore({ dbPath: './index.sqlite', config: {...} })
await store.search({ query: "how does auth work?" })
```

### MCP Server Entry Point
**File:** `src/mcp/server.ts`

Starts via: `qmd mcp` (stdio transport) or `qmd mcp --http`

---

## 4. Package Structure

**Type:** Single package (NOT a monorepo)

- **Package name:** `@tobilu/qmd`
- **Module type:** ESM (`"type": "module"`)
- **Exports:** `./dist/index.js` for SDK, `./bin/qmd` for CLI
- **Build output:** `dist/` directory (TypeScript compiled to JS)
- **No workspaces:** No `packages/` or `pnpm-workspace.yaml`

**The `finetune/` directory is a Python subproject** for model fine-tuning but is NOT integrated as a workspace or monorepo package.

---

## 5. Notable Patterns in Organization

### Pattern 1: Dual-Mode Package
QMD operates as both:
1. **CLI tool** (`bin/qmd` entry point)
2. **SDK library** (`src/index.ts` exports)

This is configured via `package.json`:
```json
{
  "bin": { "qmd": "bin/qmd" },
  "exports": { ".": { "import": "./dist/index.js" } }
}
```

### Pattern 2: Shell Wrapper for Runtime Detection
The `bin/qmd` wrapper detects whether `node` or `bun` should run the compiled code, checking for `package-lock.json` vs `bun.lock`. This prevents ABI mismatches with native modules like `better-sqlite3`.

### Pattern 3: Source vs Compiled Separation
- **Source:** `src/` (TypeScript, run via `tsx`)
- **Compiled:** `dist/` (JavaScript, run in production)
- Build command: `npm run build` -> `tsc -p tsconfig.build.json`

### Pattern 4: Single Massive Core File
The core logic (store operations, search, indexing) resides in a single `src/store.ts` file (~154KB). This is unusual for a project of this complexity and may indicate:
- Organic growth without refactoring
- Strong cohesion preference over file-per-class patterns
- Performance considerations (fewer module imports)

### Pattern 5: MCP as First-Class Citizen
The MCP server (`src/mcp/server.ts`) is a primary integration point, not an afterthought. QMD is designed to work with AI agents via the Model Context Protocol.

### Pattern 6: Test Colocation
Tests are in `test/` directory (not `src/*.test.ts`), but they are integration-focused tests that import compiled `dist/` files or run via `tsx` directly against source.

### Pattern 7: Embedded Skills
`src/embedded-skills.ts` contains Claude Code skill content that can be embedded/injected into the system, suggesting Claude Code plugin integration.

---

## 6. Technology Stack Summary

| Layer | Technology |
|-------|------------|
| Runtime | Node.js 22+ or Bun |
| Language | TypeScript |
| Package Manager | Bun (primary), npm compatible |
| Test Runner | Vitest |
| Database | SQLite + sqlite-vec (vector extension) + better-sqlite3 |
| Full-Text Search | SQLite FTS5 (BM25) |
| Vector Search | sqlite-vec |
| LLM/Embeddings | node-llama-cpp (GGUF models) |
| MCP | @modelcontextprotocol/sdk |
| Config | YAML (collections.ts) |
| Glob | fast-glob, picomatch |

---

## 7. Dependency Graph (Entry Points to Core)

```
bin/qmd (shell)
  └─> dist/cli/qmd.js
        └─> src/cli/qmd.ts
              ├─> src/store.ts (core database operations)
              ├─> src/llm.ts (LlamaCpp wrapper)
              ├─> src/collections.ts (YAML config)
              ├─> src/cli/formatter.ts (output)
              └─> src/db.ts (SQLite setup)

src/index.ts (SDK entry)
  └─> src/store.ts
        ├─> src/db.ts
        └─> src/llm.ts

src/mcp/server.ts (MCP entry)
  └─> src/index.ts (re-exports from store.ts)
```

---

## 8. Commands Exposed by CLI

The CLI (`src/cli/qmd.ts`) exposes these command groups:

| Command Group | Description |
|---------------|-------------|
| `collection` | Add, list, remove, rename collections |
| `context` | Add, list, check, remove context for paths |
| `get` | Retrieve single document by path or docid |
| `multi-get` | Batch retrieve by glob or docids |
| `ls` | List files in collection |
| `update` | Re-index all collections |
| `embed` | Generate vector embeddings |
| `query` | Hybrid search with expansion + reranking |
| `search` | BM25 keyword search |
| `vsearch` | Vector similarity search |
| `rerank` | Rerank pre-fetched results |
| `mcp` | Start MCP server (stdio or HTTP) |
| `status` | Show index status |

---

## 9. Build and Development Flow

```bash
# Development (run from source)
bun src/cli/qmd.ts <command>

# Production (after build)
npm run build          # Compile TypeScript to dist/
qmd <command>          # Uses bin/qmd wrapper

# Tests
bun test --preload ./src/test-preload.ts test/

# Global install
bun link               # Link globally as 'qmd'
```

---

## 10. Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/store.ts` | ~4,200 | Core: DB ops, search, indexing, chunking |
| `src/cli/qmd.ts` | ~3,200 | CLI commands and argument parsing |
| `src/llm.ts` | ~1,400 | LlamaCpp embeddings, reranking, query expansion |
| `src/index.ts` | ~530 | SDK public API and QMDStore interface |
| `src/mcp/server.ts` | ~850 | MCP server tools and resources |
| `src/collections.ts` | ~400 | YAML config read/write for collections |
| `bin/qmd` | 33 | Shell wrapper for runtime detection |
