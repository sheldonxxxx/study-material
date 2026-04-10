# pi-mono Project Topology

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/pi-mono`
**Type:** Monorepo (npm workspaces)
**Purpose:** Tools for building AI agents and managing LLM deployments
**Node.js:** >=20.0.0

## Monorepo Structure

```
pi-mono/
├── packages/           # 7 packages (npm workspaces)
├── scripts/            # Build and release scripts
├── .pi/               # Project-specific prompts and extensions for Claude
├── .github/           # GitHub workflows and issue templates
├── .husky/            # Git hooks
└── Root config files  # tsconfig.json, biome.json, package.json, etc.
```

## Root Configuration

| File | Purpose |
|------|---------|
| `package.json` | Monorepo workspace root, defines `workspaces: ["packages/*"]` |
| `tsconfig.json` | Base TypeScript configuration |
| `tsconfig.base.json` | Shared TypeScript settings |
| `biome.json` | Code linting/formatting (Biome) |
| `pi-mono.code-workspace` | VSCode workspace configuration |

## Package Inventory

### 1. `@mariozechner/pi-ai` (packages/ai)
**Purpose:** Unified multi-provider LLM API client
**Entry Point:** `src/index.ts`
**CLI Entry:** `src/cli.ts` (OAuth login tool)

| Path | Description |
|------|-------------|
| `src/index.ts` | Main exports (types, models, providers, stream utilities) |
| `src/cli.ts` | OAuth login CLI |
| `src/providers/` | LLM provider implementations (Anthropic, OpenAI, Google, Azure, Mistral, Amazon Bedrock, etc.) |
| `src/models.generated.ts` | Auto-generated model definitions |
| `src/types.ts` | Core type definitions |
| `src/stream.ts` | Streaming utilities |

### 2. `@mariozechner/pi-agent-core` (packages/agent)
**Purpose:** Agent runtime with tool calling and state management
**Entry Point:** `src/index.ts`
**Exports:** Agent, agent-loop, proxy, types

| Path | Description |
|------|-------------|
| `src/index.ts` | Core exports |
| `src/agent.ts` | Main agent implementation |
| `src/agent-loop.ts` | Agent execution loop |
| `src/proxy.ts` | Proxy utilities |
| `src/types.ts` | Type definitions |

### 3. `@mariozechner/pi-coding-agent` (packages/coding-agent)
**Purpose:** Interactive coding agent CLI (the main `pi` command)
**Entry Point:** `src/index.ts`
**CLI Entry:** `src/cli.ts`
**Main:** `src/main.ts`

| Path | Description |
|------|-------------|
| `src/index.ts` | Main exports (350+ lines - extensive API) |
| `src/cli.ts` | CLI entry point (`#!/usr/bin/env node`) |
| `src/main.ts` | Main function (`export { main }`) |
| `src/core/` | Core functionality (agent-session, auth-storage, compaction, event-bus, extensions, etc.) |
| `src/modes/` | Interactive and print modes |
| `src/modes/interactive/components/` | TUI components (FooterComponent, ModelSelectorComponent, etc.) |
| `src/modes/interactive/theme/` | Theme utilities |
| `src/core/tools/` | Tool implementations (bash, edit, find, grep, ls, read, write) |
| `src/core/sdk.ts` | Programmatic SDK |
| `test/` | 77 test files |
| `examples/` | Example projects including extensions |
| `docs/` | Documentation |

### 4. `@mariozechner/pi-tui` (packages/tui)
**Purpose:** Terminal UI library with differential rendering
**Entry Point:** `src/index.ts`

| Path | Description |
|------|-------------|
| `src/index.ts` | Main exports (TUI, components, keybindings, terminal) |
| `src/tui.ts` | Core TUI class and Container |
| `src/components/` | UI components (Box, Editor, Input, Loader, Markdown, SelectList, SettingsList, Text, etc.) |
| `src/autocomplete.ts` | Autocomplete support |
| `src/keybindings.ts` | Keybinding management |
| `src/keys.ts` | Key parsing and handling |
| `src/terminal.ts` | Terminal interface |
| `src/terminal-image.ts` | Terminal image rendering |
| `src/stdin-buffer.ts` | Input buffering |

### 5. `@mariozechner/pi-web-ui` (packages/web-ui)
**Purpose:** Web components for AI chat interfaces
**Entry Point:** `src/index.ts`

| Path | Description |
|------|-------------|
| `src/index.ts` | Main exports |
| `src/ChatPanel.ts` | Main chat panel component |
| `src/components/` | Web components |
| `src/dialogs/` | Dialog components |
| `src/storage/` | Storage utilities |
| `src/tools/` | Tool components |
| `src/utils/` | Utilities |
| `example/` | Example Vite app |

### 6. `@mariozechner/pi-mom` (packages/mom)
**Purpose:** Slack bot that delegates messages to the pi coding agent
**CLI Entry:** `src/main.ts` (via `bin.mom`)
**Entry Point:** `src/index.ts` (minimal)

| Path | Description |
|------|-------------|
| `src/main.ts` | CLI entry point and bot implementation |
| `src/agent.ts` | Agent runner integration |
| `src/slack.ts` | Slack bot integration |
| `src/store.ts` | Channel store |
| `src/events.ts` | Event handling |
| `src/context.ts` | Context management |
| `src/sandbox.ts` | Sandbox configuration |
| `src/log.ts` | Logging |
| `src/download.ts` | Channel download utility |
| `src/tools/` | Tool implementations |

### 7. `@mariozechner/pi-pods` (packages/pods)
**Purpose:** CLI for managing vLLM deployments on GPU pods
**CLI Entry:** Likely `src/main.ts` (needs further investigation)

| Path | Description |
|------|-------------|
| `src/main.ts` | Main CLI entry |
| `src/cli.ts` | CLI utilities |
| `src/config.ts` | Configuration |
| `src/core/` | Core functionality |

## Key Entry Points

### CLI Entry Points

| Package | Bin | Entry File |
|---------|-----|------------|
| `pi-coding-agent` | `pi` | `packages/coding-agent/src/cli.ts` |
| `pi-ai` | `@mariozechner/pi-ai` | `packages/ai/src/cli.ts` |
| `pi-mom` | `mom` | `packages/mom/src/main.ts` |
| `pi-pods` | `pi` (global install) | `packages/pods/src/main.ts` |

### Library Entry Points

| Package | Main Export | Description |
|---------|-------------|-------------|
| `pi-ai` | `src/index.ts` | AI provider API (types, models, stream) |
| `pi-agent-core` | `src/index.ts` | Agent runtime (agent, agent-loop, proxy) |
| `pi-coding-agent` | `src/index.ts` | Full coding agent SDK (main, modes, tools, UI) |
| `pi-tui` | `src/index.ts` | TUI components library |
| `pi-web-ui` | `src/index.ts` | Web UI components |
| `pi-mom` | `src/index.ts` | Minimal - mom-specific types only |
| `pi-pods` | `src/index.ts` | Minimal exports |

## Development Commands

```bash
npm install          # Install all dependencies
npm run build        # Build all packages (sequential per package.json)
npm run dev          # Run all packages in dev mode (concurrently)
npm run check        # Lint, format, type check
npm run test         # Run tests (--workspaces)
./test.sh            # Run tests (skips LLM-dependent tests)
./pi-test.sh         # Run pi from sources
```

## Build Order (per package.json)

```
cd packages/tui && npm run build &&
cd packages/ai && npm run build &&
cd packages/agent && npm run build &&
cd packages/coding-agent && npm run build &&
cd packages/mom && npm run build &&
cd packages/web-ui && npm run build &&
cd packages/pods && npm run build
```

## Project-Specific Configuration

### `.pi/` Directory
Contains Claude agent configuration for this project:

| Path | Purpose |
|------|---------|
| `.pi/prompts/` | Prompt templates (cl.md, pr.md, wr.md, is.md) |
| `.pi/extensions/` | VSCode extensions (diff.ts, files.ts, prompt-url-widget.ts, redraws.ts, tps.ts) |
| `.pi/git/` | Git hooks |
| `.pi/npm/` | NPM configuration |

## Key Patterns

1. **Package Structure:** Each package follows a consistent pattern with `src/`, `test/`, `package.json`, `tsconfig.build.json`
2. **TypeScript:** Uses `tsgo` for TypeScript compilation
3. **Testing:** Vitest for unit tests
4. **Linting:** Biome for code formatting
5. **Entry Points:** CLI tools use `#!/usr/bin/env node` shebang, library packages export from `index.ts`
6. **Dependencies:** Internal packages depend on each other (e.g., `pi-coding-agent` depends on `pi-ai`, `pi-agent-core`, `pi-tui`)
