# Tech Stack

## Language & Runtime

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Primary runtime** | Bun | Compiled binaries (~58MB single-file executable), native SQLite for cookie decryption, built-in HTTP server (Bun.serve), native TypeScript execution |
| **Windows fallback** | Node.js | Bun has a known bug with Playwright's pipe transport on Windows (bun#4253). The browse server falls back to Node.js automatically. |
| **Language** | TypeScript | Full type safety for the CLI and server code |

## Build System

Bun is the sole build tool. No separate transpiler or bundler for development.

**Build pipeline:**
```bash
bun run build
# 1. gen:skill-docs          → generate SKILL.md from templates
# 2. gen:skill-docs --host codex  → generate Codex-compatible SKILL.md
# 3. bun build --compile browse/src/cli.ts --outfile browse/dist/browse
# 4. bun build --compile browse/src/find-browse.ts --outfile browse/dist/find-browse
# 5. bun build --compile bin/gstack-global-discover.ts --outfile bin/gstack-global-discover
# 6. bash browse/scripts/build-node-server.sh
# 7. git rev-parse HEAD > browse/dist/.version
```

**Key binaries produced:**
- `browse/dist/browse` — headless browser CLI (compiled Bun binary)
- `browse/dist/find-browse` — browser discovery utility
- `bin/gstack-global-discover` — skill discovery utility

## Key Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| **playwright** | ^1.58.2 | Headless Chromium browser automation via CDP |
| **diff** | ^7.0.0 | Text diffing for PR review output |
| **@anthropic-ai/sdk** | ^0.78.0 | Dev dependency — used for LLM eval tests |

## Platform Strategy

- **macOS (Apple Silicon)**: Native compiled Bun binary in `browse/dist/`
- **macOS (Intel)**: `./setup` builds from source at install time
- **Linux**: `./setup` builds from source at install time
- **Windows**: Node.js fallback for browse server; Bun still used for build tooling

## SKILL.md Template System

SKILL.md files are **generated** from `.tmpl` templates using `scripts/gen-skill-docs.ts`. This prevents docs drift where instructions reference non-existent commands.

**Template pipeline:**
```
SKILL.md.tmpl (human-written prose + placeholders)
       ↓
gen-skill-docs.ts (reads source code metadata)
       ↓
SKILL.md (committed, auto-generated sections)
```

**Generated sections:**
| Placeholder | Source | What it generates |
|-------------|--------|-------------------|
| `{{COMMAND_REFERENCE}}` | `browse/src/commands.ts` | Categorized command table |
| `{{SNAPSHOT_FLAGS}}` | `browse/src/snapshot.ts` | Flag reference with examples |

The CLI (`browse/src/cli.ts`) and server (`browse/src/server.ts`) are the primary TypeScript source files. Command registry lives in `browse/src/commands.ts`.

## Dependencies Summary

```
dependencies:
  playwright     ^1.58.2   # Browser automation
  diff           ^7.0.0     # Text diffing

devDependencies:
  @anthropic-ai/sdk  ^0.78.0   # LLM eval testing
```

No runtime dependencies beyond Bun and (on Windows) Node.js.
