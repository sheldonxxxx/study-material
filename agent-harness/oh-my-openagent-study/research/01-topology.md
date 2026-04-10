# oh-my-openagent Project Topology

## Overview

**Project:** oh-my-opencode (v3.11.0)
**Type:** Monorepo Node.js/TypeScript project using Bun runtime
**Description:** AI Agent Harness with Multi-Model Orchestration, Parallel Background Agents, and LSP/AST Tools

---

## Directory Structure

```
oh-my-openagent/
|-- bin/                    # CLI wrapper entry point
|   |-- oh-my-opencode.js   # Platform-aware binary launcher
|   |-- platform.js         # Platform detection logic
|   |-- platform.d.ts
|   |-- platform.test.ts
|
|-- src/                    # Main source code
|   |-- index.ts            # Plugin entry point (OhMyOpenCodePlugin)
|   |-- plugin-config.ts     # Configuration loading
|   |-- plugin-interface.ts
|   |-- plugin-dispose.ts
|   |-- plugin-state.ts
|   |-- create-hooks.ts
|   |-- create-managers.ts
|   |-- create-tools.ts
|
|   |-- cli/                # CLI implementation
|   |   |-- index.ts        # CLI entry: runs cli-program.ts
|   |   |-- cli-program.ts  # Main CLI logic
|   |   |-- install.ts      # Installation logic
|   |   |-- config-manager/ # Config management
|   |   |-- run/            # Run command implementation
|   |   |-- doctor/         # Diagnostic checks
|   |   |-- model-fallback/ # Model fallback logic
|   |   |-- mcp-oauth/      # MCP OAuth handling
|   |   |-- tui-install-prompts.ts
|   |   |-- tui-installer.ts
|   |   |-- skill/          # Skill management
|   |   |-- skill-mcp/      # Skill MCP integration
|   |   |-- session-manager/
|   |   |-- get-local-version/
|   |   |-- glob/           # Glob tool
|   |   |-- grep/           # Grep tool
|   |   |-- ast-grep/       # AST-based grep
|   |   |-- look-at/        # Vision tool
|   |   |-- lsp/            # LSP integration
|   |   |-- hashline-edit/  # Edit tool with hashline
|   |   |-- interactive-bash/
|   |   |-- background-task/
|   |   |-- delegate-task/
|   |   |-- call-omo-agent/
|   |   |-- slashcommand/
|   |   |-- task/
|   |
|   |-- agents/             # Agent definitions
|   |   |-- builtin-agents.ts
|   |   |-- dynamic-agent-prompt-builder.ts
|   |   |-- librarian.ts    # Librarian agent
|   |   |-- oracle.ts       # Oracle agent
|   |   |-- momus.ts        # Momus agent
|   |   |-- sisyphus.ts     # Sisyphus agent
|   |   |-- prometheus.ts   # Prometheus agent
|   |   |-- hephaestus/
|   |   |-- sisyphus-junior/
|   |   |-- atlas/
|   |   |-- prometheus/
|   |   |-- builtin-agents/ # Individual agent configs
|   |   |-- types.ts
|   |   |-- types.test.ts
|   |   |-- utils.test.ts
|   |
|   |-- hooks/              # Hook system (76 subdirectories)
|   |   |-- index.ts
|   |   |-- background-agent/
|   |   |-- claude-code-hooks/
|   |   |-- context-window-monitor/
|   |   |-- preemptive-compaction/
|   |   |-- runtime-fallback/
|   |   |-- ralph-loop/
|   |   |-- tmux-subagent/
|   |   |-- model-fallback/
|   |   |-- rules-injector/
|   |   |-- opencode-skill-loader/
|   |   |-- skill-mcp-manager/
|   |   |-- mcp-oauth/
|   |   |-- auto-update-checker/
|   |   |-- auto-slash-command/
|   |   |-- compaction-context-injector/
|   |   |-- compaction-todo-preserver/
|   |   |-- agent-usage-reminder/
|   |   |-- directory-agents-injector/
|   |   |-- directory-readme-injector/
|   |   |-- comment-checker/
|   |   |-- anthropic-context-window-limit-recovery/
|   |   |-- ... (50+ more hooks)
|   |
|   |-- features/           # Feature modules
|   |   |-- index.ts
|   |   |-- openclaw.ts
|   |   |-- context-window-monitor.ts
|   |   |-- preemptive-compaction.ts
|   |   |-- non-interactive-env/
|   |   |-- interactive-bash-session/
|   |   |-- background-notification/
|   |   |-- read-image-resizer/
|   |   |-- ... (20+ more features)
|   |
|   |-- shared/             # Shared utilities (153 files)
|   |   |-- model-capabilities/
|   |   |-- config-errors.ts
|   |   |-- first-message-variant/
|   |   |-- ...
|   |
|   |-- tools/              # Tool implementations
|   |   |-- (various tool modules)
|   |
|   |-- mcp/                # MCP (Model Context Protocol) integration
|   |-- config/             # Configuration types and schema
|   |-- openclaw/           # OpenClaw integration
|   |-- generated/          # Generated code
|
|-- packages/               # Platform-specific native binaries
|   |-- darwin-arm64/
|   |   |-- bin/oh-my-opencode  # Native binary
|   |-- darwin-x64/
|   |-- darwin-x64-baseline/
|   |-- linux-arm64/
|   |-- linux-arm64-musl/
|   |-- linux-x64/
|   |-- linux-x64-baseline/
|   |-- linux-x64-musl/
|   |-- linux-x64-musl-baseline/
|   |-- windows-x64/
|   |-- windows-x64-baseline/
|
|-- tests/                  # Test directory
|   |-- hashline/
|
|-- docs/                   # Documentation
|   |-- examples/
|   |-- guide/
|   |-- reference/
|   |-- superpowers/
|   |-- troubleshooting/
|   |-- manifesto.md
|   |-- model-capabilities-maintenance.md
|
|-- script/                 # Build and release scripts
|   |-- build-binaries.ts
|   |-- build-schema.ts
|   |-- build-model-capabilities.ts
|   |-- publish.ts
|   |-- generate-changelog.ts
|
|-- tests/                  # Integration tests
|   |-- hashline/
|
|-- AGENTS.md              # Agent definitions (root level)
|-- CLA.md                 # Contributor License Agreement
|-- CONTRIBUTING.md
|-- FIX-BLOCKS.md
|-- LICENSE.md
|-- README*.md             # Multiple language READMEs
|-- package.json
|-- tsconfig.json
|-- bun.lock
|-- bunfig.toml
|-- postinstall.mjs
```

---

## Entry Points

### 1. CLI Entry Point
**File:** `bin/oh-my-opencode.js`
- Wrapper script that detects platform (OS + arch + libc variant)
- Resolves platform-specific binary from `packages/{platform}/bin/oh-my-opencode`
- Spawns the binary with process.argv
- Falls back to baseline variants if AVX2 is unsupported

### 2. CLI Program Entry
**File:** `src/cli/index.ts`
```typescript
#!/usr/bin/env bun
import { runCli } from "./cli-program"
runCli()
```

### 3. Plugin Entry Point (OpenCode Plugin)
**File:** `src/index.ts`
```typescript
const OhMyOpenCodePlugin: Plugin = async (ctx) => { ... }
export default OhMyOpenCodePlugin
```
This is the main plugin that integrates with OpenCode's plugin system.

### 4. Native Binary Entry
**File:** `packages/{platform}/bin/oh-my-opencode`
Platform-specific binaries for each supported OS/arch combination.

---

## Key Organizational Patterns

### 1. Monorepo with Platform-Specific Packages
- Root `package.json` declares all platform packages as `optionalDependencies`
- Each `packages/{platform}/` contains a native binary that gets installed based on detected platform
- Baseline variants exist for CPUs without AVX2 support

### 2. Hook-Based Architecture
- `src/hooks/` contains 76+ hook implementations
- Hooks are created via `createHooks()` factory
- Each hook is a discrete feature module (e.g., `context-window-monitor`, `preemptive-compaction`)

### 3. Agent System
- `src/agents/` defines multiple named agents: `librarian`, `oracle`, `momus`, `sisyphus`, `prometheus`, `hephaestus`
- Agents built via `agent-builder.ts`
- Dynamic prompt building via `dynamic-agent-prompt-builder.ts`

### 4. Feature Modules
- `src/features/` contains 22+ feature implementations
- Features include: `openclaw`, `context-window-monitor`, `preemptive-compaction`, `non-interactive-env`, etc.

### 5. Tool Implementations
- `src/cli/` contains tool implementations as subdirectories
- Tools: `glob`, `grep`, `ast-grep`, `look-at`, `lsp`, `hashline-edit`, `interactive-bash`, `background-task`, `delegate-task`, `call-omo-agent`

### 6. Shared Utilities
- `src/shared/` contains 153 files of shared code
- Includes: `model-capabilities/`, `config-errors.ts`, `first-message-variant/`

---

## Configuration Files

| File | Purpose |
|------|---------|
| `package.json` | Main package definition, bun-based build |
| `tsconfig.json` | TypeScript configuration |
| `bunfig.toml` | Bun configuration |
| `postinstall.mjs` | Post-install script for binary setup |

---

## Build System

- **Runtime:** Bun
- **Build command:** `bun run build` (compiles `src/index.ts` to `dist/`)
- **Binary build:** `bun run build:binaries` (via `script/build-binaries.ts`)
- **Schema build:** `bun run build:schema` (generates JSON schema)
- **Model capabilities:** `bun run build:model-capabilities`

---

## Key Dependencies

- `@opencode-ai/plugin` - OpenCode plugin system
- `@opencode-ai/sdk` - OpenCode SDK
- `@modelcontextprotocol/sdk` - MCP protocol
- `@ast-grep/napi` - AST grep library
- `commander` - CLI framework
- `zod` - Schema validation
- `picocolors` - Terminal colors
- `picomatch` - Glob matching
