# Project Topology: memory-lancedb-pro

## Overview

**Repository:** `memory-lancedb-pro`
**Type:** OpenClaw Plugin (Node.js/TypeScript ESM module)
**Version:** 1.1.0-beta.10
**License:** MIT

An enhanced LanceDB-backed long-term memory plugin for OpenClaw with hybrid retrieval (Vector + BM25), cross-encoder reranking, multi-scope isolation, long-context chunking, and management CLI.

---

## Root Directory Structure

```
/memory-lancedb-pro/
├── .git/                     # Git repository metadata
├── .github/                  # GitHub configuration (ISSUE_TEMPLATES, workflows)
├── .gitignore                # Git ignore rules
├── .npmignore                # NPM package ignore rules
├── CHANGELOG-v1.1.0.md       # Version 1.1.0 changelog
├── CHANGELOG.md              # Main changelog
├── README.md                 # English documentation (37KB)
├── README_*.md               # Localized READMEs (CN, DE, ES, FR, IT, JA, KO, PT-BR, RU, TW)
├── cli.ts                    # CLI entry point (47KB standalone file)
├── docs/                     # Documentation directory
├── examples/                 # Example code
├── index.ts                  # Main plugin entry point (155KB)
├── openclaw.plugin.json      # OpenClaw plugin manifest
├── package-lock.json         # NPM lock file
├── package.json              # NPM package definition
├── scripts/                  # Build/maintenance scripts
├── skills/                   # Skills/lessons directory
├── src/                      # Core source code (45 modules)
└── test/                     # Test suite (49 test files)
```

---

## Entry Point Files

### `index.ts` (Main Plugin Entry)
- **Size:** 155KB
- **Role:** Primary OpenClaw plugin entry point
- **Exports:** `MemoryStore`, tool registrations, plugin API hooks
- **Dependencies:** Imports from `src/` for core functionality
- **Pattern:** Implements `OpenClawPluginApi` interface

### `cli.ts` (CLI Entry)
- **Size:** 47KB
- **Role:** Standalone CLI for memory management
- **Uses:** `commander` package for CLI argument parsing
- **Commands:** Memory inspection, OAuth login, governance maintenance

### `openclaw.plugin.json` (Plugin Manifest)
- **Size:** 46KB
- **Role:** OpenClaw plugin configuration and metadata

---

## Source Code (`src/`)

### Core Modules (45 TypeScript files)

| Module | Purpose |
|--------|---------|
| `store.ts` | LanceDB memory store, `MemoryStore` class |
| `embedder.ts` | Embedding generation via OpenAI-compatible APIs |
| `retriever.ts` | Hybrid retrieval (vector + BM25), reranking |
| `scopes.ts` | Multi-scope isolation, session management |
| `migrate.ts` | Schema migration and upgrades |
| `tools.ts` | MCP tools registry (~74KB) |
| `smart-extractor.ts` | Intelligent fact extraction (~45KB) |
| `smart-metadata.ts` | Lifecycle metadata handling |
| `llm-client.ts` | LLM API client abstraction |
| `llm-oauth.ts` | OAuth authentication for LLMs |
| `memory-compactor.ts` | Memory compaction/cleanup |
| `memory-upgrader.ts` | Memory schema upgrades |
| `decay-engine.ts` | Memory decay calculations |
| `tier-manager.ts` | Memory tier management |
| `session-compressor.ts` | Session summarization |
| `session-recovery.ts` | Session recovery mechanisms |
| `reflection-store.ts` | Agent reflection storage |
| `reflection-slices.ts` | Reflection processing |
| `reflection-event-store.ts` | Reflection event tracking |
| `reflection-mapped-metadata.ts` | Reflection metadata mapping |
| `access-tracker.ts` | Memory access tracking |
| `admission-control.ts` | Memory admission policies |
| `admission-stats.ts` | Admission statistics |
| `adaptive-retrieval.ts` | Adaptive retrieval logic |
| `batch-dedup.ts` | Deduplication utilities |
| `chunker.ts` | Text chunking for long contexts |
| `clawteam-scope.ts` | Team-scoped memory isolation |
| `extraction-prompts.ts` | Extraction prompt templates |
| `identity-addressing.ts` | Identity resolution |
| `intent-analyzer.ts` | Intent analysis for memory |
| `memory-categories.ts` | Memory categorization |
| `noise-filter.ts` | Noise filtering |
| `noise-prototypes.ts` | Noise pattern detection |
| `preference-slots.ts` | Preference management |
| `reflection-ranking.ts` | Reflection ranking |
| `reflection-retry.ts` | Retry logic for reflections |
| `retrieval-stats.ts` | Retrieval statistics |
| `retrieval-trace.ts` | Retrieval tracing |
| `self-improvement-files.ts` | Self-improvement tracking |
| `auto-capture-cleanup.ts` | Auto-capture text normalization |
| `workspace-boundary.ts` | Workspace isolation |

---

## Test Suite (`test/`)

### Test Files (49 files)

**Unit Tests:**
- `access-tracker.test.mjs`
- `batch-dedup.test.mjs`
- `cjk-recursion-regression.test.mjs`
- `clawteam-scope.test.mjs`
- `config-session-strategy-migration.test.mjs`
- `embedder-error-hints.test.mjs`
- `governance-metadata.test.mjs`
- `intent-analyzer.test.mjs`
- `jsonl-distill-slash-filter.test.mjs`
- `llm-api-key-client.test.mjs`
- `llm-oauth-client.test.mjs`
- `memory-governance-tools.test.mjs`
- `memory-upgrader-diagnostics.test.mjs`
- `migrate-legacy-schema.test.mjs`
- `preference-slots.test.mjs`
- `recall-text-cleanup.test.mjs`
- `reflection-bypass-hook.test.mjs`
- `resolve-env-vars-array.test.mjs`
- `retrieval-trace.test.mjs`
- `retriever-tag-query.test.mjs`
- `scope-access-undefined.test.mjs`
- `self-improvement.test.mjs`
- `session-compressor.test.mjs`
- `session-recovery-paths.test.mjs`
- `smart-extractor-scope-filter.test.mjs`
- `store-empty-scope-filter.test.mjs`
- `strip-envelope-metadata.test.mjs`
- `sync-plugin-version.test.mjs`
- `temporal-facts.test.mjs`
- `update-consistency-lancedb.test.mjs`
- `vector-search-cosine.test.mjs`
- `workflow-fork-guards.test.mjs`

**Integration/E2E Tests:**
- `cli-smoke.mjs`
- `functional-e2e.mjs`
- `context-support-e2e.mjs`
- `memory-reflection.test.mjs`
- `memory-update-supersede.test.mjs`
- `openclaw-host-functional.mjs`
- `plugin-manifest-regression.mjs`
- `retriever-rerank-regression.mjs`
- `smart-extractor-branches.mjs`
- `smart-memory-lifecycle.mjs`
- `smart-metadata-v2.mjs`
- `cli-oauth-login.test.mjs`
- `cross-process-lock.test.mjs`
- `memory-compactor.test.mjs`

**Helpers:**
- `test/helpers/` - Test utility modules

---

## Documentation (`docs/`)

| File | Content |
|------|---------|
| `CHANGELOG-v1.1.0.md` | Version 1.1.0 specific changelog |
| `long-context-chunking.md` | Long context chunking strategy |
| `memory_architecture_analysis.md` | Architecture deep-dive (25KB) |
| `openclaw-integration-playbook.md` | Integration guide (EN) |
| `openclaw-integration-playbook.zh-CN.md` | Integration guide (Chinese) |

---

## Scripts (`scripts/`)

| File | Purpose |
|------|---------|
| `governance-maintenance.mjs` | Memory governance maintenance |
| `jsonl_distill.py` | JSONL data distillation |
| `migrate-governance-metadata.mjs` | Governance metadata migration |
| `smoke-openclaw.sh` | OpenClaw smoke test |
| `sync-plugin-version.mjs` | Plugin version synchronization |

---

## Package Structure

### `package.json` Highlights

```json
{
  "name": "memory-lancedb-pro",
  "version": "1.1.0-beta.10",
  "type": "module",
  "main": "index.ts",
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@lancedb/lancedb` | ^0.26.2 | Vector database |
| `@sinclair/typebox` | 0.34.48 | Type validation |
| `apache-arrow` | 18.1.0 | Arrow data format |
| `json5` | ^2.2.3 | JSON parsing |
| `openai` | ^6.21.0 | OpenAI API client |
| `proper-lockfile` | ^4.1.2 | File locking |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `commander` | ^14.0.0 | CLI framework |
| `jiti` | ^2.6.0 | TypeScript/ESM runtime |
| `typescript` | ^5.9.3 | TypeScript compiler |

---

## Examples & Skills

### `examples/`
- `new-session-distill/` - Session distillation example

### `skills/`
- `lesson/` - Learning materials

---

## GitHub Configuration

### Workflows (`.github/workflows/`)

| Workflow | Purpose |
|----------|---------|
| `auto-assign.yml` | Auto-assign issues |
| `ci.yml` | Continuous integration |
| `claude-code-review.yml` | Claude code review automation |
| `claude.yml` | Claude AI workflow |

### Issue Templates
- `bug_report.yml` - Bug report template
- `feature_request.yml` - Feature request template
- `config.yml` - Issue configuration

---

## Key Architectural Patterns

### Module Organization
- **Flat structure:** All `src/` modules at top level (no subdirectories)
- **Explicit exports:** Named exports from each module
- **ESM-first:** `"type": "module"` in package.json

### Plugin Integration
- **Entry:** `index.ts` exports `MemoryStore` and registers tools
- **Config:** `openclaw.plugin.json` defines plugin extension point
- **CLI:** `cli.ts` provides standalone management commands

### Data Flow
1. User interaction captured via OpenClaw hooks
2. `embedder.ts` generates vector embeddings
3. `store.ts` persists to LanceDB
4. `retriever.ts` handles hybrid search
5. `tools.ts` exposes MCP tools

### Testing Strategy
- Unit tests alongside source files (`.test.mjs`)
- E2E tests in `test/` directory
- Integration tests via `openclaw-host-functional.mjs`
