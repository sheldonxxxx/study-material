# get-shit-done Directory Topology

## Root-Level File Inventory

| File | Purpose |
|------|---------|
| `package.json` | npm package manifest (v1.29.0) - defines `bin` entry point at `bin/install.js` |
| `CHANGELOG.md` | Release history |
| `CONTRIBUTING.md` | Developer contribution guidelines |
| `LICENSE` | MIT license |
| `README.md` | Primary documentation (34KB, multilingual variants exist) |
| `README.*.md` | Localized docs (ja-JP, ko-KR, pt-BR, zh-CN) |
| `SECURITY.md` | Security reporting policy |
| `.release-monitor.sh` | Release monitoring script |
| `.base64scanignore` | Security scan exclusion list |
| `.secretscanignore` | Secret scan exclusion list |

## Directory Tree (2-3 Levels Deep)

```
get-shit-done/
├── bin/                          # Installation entry point
│   └── install.js                # Main installer (174KB executable)
├── get-shit-done/                # Core library package
│   ├── bin/
│   │   ├── gsd-tools.cjs         # CLI tools (36KB)
│   │   └── lib/                  # Core modules (19 .cjs files)
│   │       ├── commands.cjs      # Command registry
│   │       ├── config.cjs        # Configuration handling
│   │       ├── core.cjs         # Core functionality
│   │       ├── frontmatter.cjs  # Frontmatter parsing
│   │       ├── init.cjs         # Initialization logic
│   │       ├── milestone.cjs    # Milestone management
│   │       ├── phase.cjs        # Phase management
│   │       ├── profile-*.cjs    # User profiling pipelines
│   │       ├── roadmap.cjs      # Roadmap handling
│   │       ├── security.cjs     # Security utilities
│   │       ├── state.cjs        # State management
│   │       ├── template.cjs     # Template rendering
│   │       ├── uat.cjs          # UAT utilities
│   │       ├── verify.cjs       # Verification logic
│   │       └── workstream.cjs   # Workstream management
│   ├── commands/                 # Command implementations (57 .md files)
│   │   ├── gsd/                  # Commands invoked as /gsd:*
│   │   └── (57 command files)   # e.g., add-phase.md, execute-phase.md
│   ├── references/               # Reference documentation (16 .md files)
│   │   ├── checkpoints.md
│   │   ├── planning-config.md
│   │   ├── tdd.md
│   │   ├── user-profiling.md
│   │   ├── verification-patterns.md
│   │   └── ...
│   ├── templates/               # Project templates (34 files + subdirs)
│   │   ├── DEBUG.md
│   │   ├── UAT.md
│   │   ├── VALIDATION.md
│   │   ├── roadmap.md
│   │   ├── state.md
│   │   ├── summary*.md
│   │   ├── phase-prompt.md
│   │   ├── context.md
│   │   ├── research.md
│   │   ├── codebase/            # Codebase analysis templates
│   │   └── research-project/     # Research project templates
│   └── workflows/                # Execution workflows (57 .md files)
│       ├── add-phase.md
│       ├── execute-phase.md
│       ├── plan-phase.md
│       ├── verify-phase.md
│       └── ...
├── agents/                      # Agent definitions (17 .md files)
│   ├── gsd-advisor-researcher.md
│   ├── gsd-codebase-mapper.md
│   ├── gsd-debugger.md
│   ├── gsd-executor.md
│   ├── gsd-planner.md
│   ├── gsd-verifier.md
│   └── ...
├── commands/                     # CLI command definitions (57 .md files)
│   ├── add-phase.md
│   ├── complete-milestone.md
│   ├── discuss-phase.md
│   ├── execute-phase.md
│   ├── help.md
│   ├── new-project.md
│   ├── plan-phase.md
│   └── ...
├── hooks/                        # Git hooks
│   ├── build-hooks.js            # Hook builder
│   ├── base64-scan.sh           # Security scanning
│   ├── gsd-check-update.js
│   ├── gsd-context-monitor.js
│   ├── gsd-prompt-guard.js
│   ├── gsd-statusline.js
│   ├── gsd-workflow-guard.js
│   ├── prompt-injection-scan.sh
│   └── secret-scan.sh
├── scripts/                      # Build and utility scripts
│   ├── build-hooks.js            # Compiles hooks to hooks/dist
│   ├── run-tests.cjs             # Test runner
│   ├── gsd-*.js                  # Various GSD tools
│   └── *.sh                      # Shell security scripts
├── tests/                        # Test suite (50 .test.cjs files)
│   ├── agent-*.test.cjs
│   ├── commands.test.cjs
│   ├── config.test.cjs
│   ├── core.test.cjs
│   ├── phase.test.cjs
│   ├── roadmap.test.cjs
│   ├── state.test.cjs
│   ├── verify.test.cjs
│   └── ...
├── assets/                       # Logo and branding assets
│   ├── gsd-logo-*.png
│   ├── gsd-logo-*.svg
│   └── terminal.svg
├── docs/                         # Documentation (localized)
│   ├── ja-JP/
│   ├── ko-KR/
│   ├── pt-BR/
│   ├── superpowers/
│   └── zh-CN/
├── .github/
│   ├── workflows/                # CI/CD workflows
│   │   ├── auto-label-issues.yml
│   │   ├── security-scan.yml
│   │   └── test.yml
│   └── ISSUE_TEMPLATE/
└── .git/                         # Git repository
```

## Key Entry Points

### 1. CLI Entry Point (npm bin)
```
bin/install.js (174KB)
```
- Registered as `get-shit-done-cc` in package.json
- Invoked when user runs `get-shit-done-cc` command

### 2. Core Tools Entry Point
```
get-shit-done/bin/gsd-tools.cjs (36KB)
```
- Provides subcommands: `commit`, `frontmatter`, `history-digest`, `verify`
- Called by workflows and commands

### 3. Command Dispatcher
```
get-shit-done/bin/lib/commands.cjs
```
- Registry mapping command names to implementation files
- Loads command prompts from `commands/` directory

### 4. User-Invoked Commands
```
commands/*.md           # 57 command definition files
get-shit-done/commands/*.md  # Same commands, different context
```
- User invokes via `/gsd:command-name` in Claude Code
- Examples: `/gsd:plan-phase`, `/gsd:execute-phase`, `/gsd:new-project`

### 5. Agent Definitions
```
agents/gsd-*.md        # 17 agent definition files
```
- Define agent behaviors: planner, executor, verifier, debugger, etc.
- Used by workflows to spawn specialized agents

### 6. Workflow Templates
```
get-shit-done/workflows/*.md   # 57 workflow files
```
- Orchestrate how agents and commands combine
- Examples: `plan-phase.md`, `execute-phase.md`, `verify-phase.md`

## Directory Purposes

| Directory | Purpose |
|-----------|---------|
| `bin/` | Installation and CLI entry point |
| `get-shit-done/` | Core package containing all business logic |
| `get-shit-done/bin/` | Core CLI tools and library modules |
| `get-shit-done/commands/` | Command implementations (prompts) |
| `get-shit-done/workflows/` | Workflow orchestrations |
| `get-shit-done/references/` | Technical reference documentation |
| `get-shit-done/templates/` | Project scaffolding templates |
| `agents/` | Agent behavior definitions |
| `commands/` | CLI command definitions |
| `hooks/` | Git hooks for security/scanning |
| `scripts/` | Build and utility scripts |
| `tests/` | Comprehensive test suite |
| `assets/` | Logo and branding images |
| `docs/` | Localized documentation |
| `.github/` | GitHub configuration (CI, issue templates) |

## Architecture Summary

```
User invokes /gsd:command
        ↓
commands/*.md (dispatcher looks up command)
        ↓
get-shit-done/commands/*.md (command implementation)
        ↓
workflows/*.md (orchestrates agents + tools)
        ↓
agents/*.md (spawns specialized agents)
        ↓
get-shit-done/bin/gsd-tools.cjs (provides utilities: commit, verify, etc.)
        ↓
get-shit-done/bin/lib/*.cjs (core modules)
```
