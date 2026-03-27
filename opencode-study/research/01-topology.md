# Opencode Project Topology

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Package Manager:** Bun (v1.3.11)
**Architecture:** Monorepo with Turborepo
**Main Language:** TypeScript

## Root Structure

```
opencode/
├── AGENTS.md                    # Agent instructions/prompts
├── CONTRIBUTING.md              # Contribution guidelines
├── README.md                    # Primary documentation
├── package.json                 # Root workspace config
├── bun.lock                     # Bun lockfile
├── bunfig.toml                  # Bun configuration
├── turbo.json                   # Turborepo config
├── sst.config.ts                # SST (Serverless Stack) config
├── tsconfig.json                # TypeScript base config
├── flake.nix                    # Nix flake for dev environment
├── install                      # Installation script
│
├── .github/                     # GitHub workflows and configs
├── .opencode/                   # Opencode-specific settings
├── .vscode/                     # VSCode settings
├── .zed/                        # Zed editor settings
├── .husky/                      # Git hooks
│
├── packages/                    # Main packages (19 packages)
├── github/                      # GitHub integration infrastructure
├── infra/                       # Infrastructure as code
├── script/                      # Build/release scripts
├── sdks/                        # SDK implementations
├── patches/                     # Dependency patches
├── specs/                       # Specifications
└── nix/                         # Nix packages
```

## Packages Directory (`packages/`)

The monorepo contains 19 packages organized by function:

### Core Packages

| Package | Purpose |
|---------|---------|
| `opencode` | **Main CLI application** - AI-powered development tool core |
| `app` | Web application (SolidStart-based) |
| `console` | Backend services (SST-based) |
| `web` | Marketing website (Astro-based) |
| `desktop` | Desktop application (Tauri + Solid.js) |
| `desktop-electron` | Electron-specific code for desktop |
| `docs` | Documentation site |
| `storybook` | Component storybook |

### Integration Packages

| Package | Purpose |
|---------|---------|
| `github` | GitHub App/integration |
| `slack` | Slack integration |
| `identity` | Authentication/identity service |
| `plugin` | Plugin system |
| `function` | Serverless functions |
| `containers` | Container-related tooling |

### Utility Packages

| Package | Purpose |
|---------|---------|
| `ui` | Shared UI components |
| `sdk` | JavaScript SDK for opencode API |
| `script` | Shared scripts library |
| `util` | Shared utilities |
| `enterprise` | Enterprise features |

## Entry Points

### Primary CLI Entry Point

**File:** `packages/opencode/src/index.ts`

The main CLI is built with Yargs and registers these commands:

```
opencode run          # Run agent in current directory
opencode agent        # Agent management
opencode chat         # Chat interface
opencode console      # Console/account management
opencode providers    # AI provider management
opencode models       # Model management
opencode mcp          # MCP (Model Context Protocol) tools
opencode serve        # Start server mode
opencode workspace-serve  # Local workspace server (dev only)
opencode debug        # Debug utilities
opencode stats        # Usage statistics
opencode export       # Export data
opencode import       # Import data
opencode github       # GitHub integration
opencode pr           # Pull request tools
opencode session       # Session management
opencode db           # Database tools
opencode acp          # ACP (Agent Control Plane) commands
opencode upgrade      # Self-upgrade
opencode uninstall    # Uninstall
opencode generate     # Code generation
opencode web          # Web interface
```

### Development Entry Points

```json
{
  "dev": "bun run --cwd packages/opencode --conditions=browser src/index.ts",
  "dev:desktop": "bun --cwd packages/desktop tauri dev",
  "dev:web": "bun --cwd packages/app dev",
  "dev:console": "ulimit -n 10240; bun run --cwd packages/console/app dev",
  "dev:storybook": "bun --cwd packages/storybook storybook"
}
```

### Desktop Entry Point

**File:** `packages/desktop/src-tauri/`

- Tauri-based desktop application
- Uses Solid.js for UI
- Rust backend (src-tauri)

### Web Entry Point

**File:** `packages/web/src/`

- Astro-based static site
- Marketing and landing pages

### App Entry Point

**File:** `packages/app/src/`

- SolidStart application
- Main web application

## Package Internal Structure

### `packages/opencode/src/` Directory Map

```
src/
├── index.ts              # CLI entry point (yargs)
├── node.ts               # Node.js specific entry
├── sql.d.ts              # SQL type declarations
│
├── account/              # Account management
├── acp/                  # Agent Control Plane
├── agent/                # Agent core (llm.ts, message.ts, processor.ts)
├── auth/                 # Authentication
├── bus/                  # Event bus
├── cli/                  # CLI commands
│   └── cmd/              # Individual commands (run, chat, agent, etc.)
├── command/              # Command infrastructure
├── config/               # Configuration handling
├── control-plane/        # Control plane client
├── effect/               # Effect framework integration
├── env/                  # Environment variables
├── file/                 # File operations
├── filesystem/          # Filesystem utilities
├── flag/                 # Feature flags
├── format/               # Code formatting
├── git/                  # Git integration
├── global/               # Global state
├── id/                   # ID generation
├── ide/                  # IDE integration
├── installation/         # Installation management
├── lsp/                  # Language Server Protocol
├── mcp/                  # MCP server implementation
├── permission/           # Permission handling
├── plugin/               # Plugin system
├── project/              # Project management
├── provider/             # AI provider abstraction
├── pty/                  # PTY (pseudo-terminal)
├── question/             # Question/answer system
├── server/               # Server implementation
│   └── routes/           # HTTP routes
├── session/             # Session management
├── share/                # Sharing functionality
├── shell/                # Shell execution
├── skill/                # Skill system
├── snapshot/             # Snapshot/restore
├── storage/              # Storage implementations
├── sync/                 # Sync functionality
├── tool/                 # Tool implementations (47 tools)
│   ├── bash.ts           # Bash execution
│   ├── edit.ts           # File editing
│   ├── read.ts           # File reading
│   ├── write.ts          # File writing
│   ├── glob.ts           # Glob pattern matching
│   ├── grep.ts           # Text search
│   ├── lsp.ts            # LSP integration
│   ├── webfetch.ts       # Web fetching
│   ├── websearch.ts      # Web search
│   └── ...
└── util/                 # Utilities (43 modules)
    ├── log.ts            # Logging
    ├── filesystem.ts     # Filesystem helpers
    └── ...
```

### Key Tool Modules in `src/tool/`

- `bash.ts` - Shell command execution
- `edit.ts` - Code editing
- `read.ts` / `write.ts` - File I/O
- `glob.ts` / `grep.ts` - Code search
- `lsp.ts` - Language Server Protocol
- `webfetch.ts` / `websearch.ts` - Web access
- `task.ts` / `todowrite.ts` - Task management
- `apply_patch.ts` - Patch application
- `codesearch.ts` - Code search
- `multiedit.ts` - Multi-file editing

### Key Agent Modules in `src/agent/`

- `index.ts` - Main agent orchestrator
- `llm.ts` - LLM interaction
- `message.ts` / `message-v2.ts` - Message handling
- `processor.ts` - Message processing
- `prompt.ts` - Prompt generation (70KB)
- `compaction.ts` - Context compaction
- `instruction.ts` - Instruction handling
- `projectors.ts` - Output projection
- `retry.ts` - Retry logic
- `revert.ts` - Revert handling
- `schema.ts` - Schema definitions
- `summary.ts` - Summarization
- `system.ts` - System prompt

## Infrastructure

### `infra/` - Infrastructure as Code

```
infra/
├── README.md
├── action.yml           # GitHub Action definition
├── index.ts             # Main infrastructure code
├── script/              # Infra scripts
└── ...
```

### `github/` - GitHub App Infrastructure

```
github/
├── app/                 # GitHub App implementation
├── console/             # Console integration
├── containers/          # Container configs
├── desktop/             # Desktop integration
├── docs/                # Docs integration
├── enterprise/          # Enterprise GitHub
├── extensions/          # GitHub extensions
├── function/            # Serverless functions
├── identity/            # Identity integration
├── opencode/            # Main GitHub integration
├── plugin/              # Plugin integration
├── script/              # Scripts
├── sdk/                 # SDK integration
├── slack/               # Slack integration
├── storybook/           # Storybook integration
├── ui/                  # UI integration
├── util/                # Utilities
└── web/                 # Web integration
```

## Build System

### Workspaces Configuration

```json
"workspaces": {
  "packages": [
    "packages/*",
    "packages/console/*",
    "packages/sdk/js",
    "packages/slack"
  ]
}
```

### Key Dependencies

- **UI Framework:** Solid.js, SolidStart
- **Desktop:** Tauri, Electron
- **Web:** Astro
- **AI:** AI SDK (Vercel), various LLM providers
- **Database:** Drizzle ORM, SQLite
- **Auth:** OpenAuth.js
- **API:** Hono
- **Syntax Highlighting:** Shiki
- **Styling:** Tailwind CSS

## Summary

The opencode project is a **large TypeScript monorepo** with:

1. **Main deliverable:** A CLI tool (`packages/opencode`) for AI-assisted development
2. **Multiple UIs:** Desktop (Tauri), Web (Astro), App (SolidStart)
3. **Backend services:** Console app with serverless functions
4. **Integrations:** GitHub App, Slack, MCP server
5. **Plugin system:** Extensible architecture
6. **Rich toolset:** 47+ tools for file operations, code editing, search, web access, etc.
