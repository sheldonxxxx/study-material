# Project Overview: Get Shit Done (GSD)

## What is GSD?

**Get Shit Done (GSD)** is a lightweight meta-prompting, context engineering, and spec-driven development framework designed for AI coding agents (Claude Code, OpenCode, Gemini CLI, Codex, Copilot, Cursor, Windsurf, and Antigravity).

**Core Problem Solved:** Context rot - the quality degradation that happens as an AI fills its context window with accumulated conversation history.

## Project Purpose and Scope

GSD transforms AI coding agents from unreliable vibecoders into consistent, verifiable engineering tools by providing:

1. **Structured Context Engineering** - Pre-formatted artifacts that give the AI everything it needs per task
2. **Multi-Agent Orchestration** - Thin orchestrators that spawn specialized agents with fresh context windows
3. **Spec-Driven Development** - Requirements to research to plans to execution to verification pipeline
4. **Persistent State Management** - File-based project memory across sessions and context resets

## Core Value Proposition

The project's tagline: *"Claude Code is powerful. GSD makes it reliable."*

GSD targets **solo developers** who want to describe what they want and have it built correctly - without enterprise roleplay or managing a team.

## Key Features

### Multi-Runtime Support
GSD installs as custom slash commands across 8 different AI coding runtimes:
- Claude Code (`/gsd:command`)
- OpenCode (`/gsd-command`)
- Gemini CLI (`/gsd:command`)
- Codex (`$gsd-command`)
- Copilot (`/gsd:command`)
- Cursor (`/gsd:command`)
- Windsurf (`/gsd:command`)
- Antigravity (`/gsd:command`)

### Command-Driven Workflow
44 total commands organized into:
- **Core Workflow:** new-project, discuss-phase, plan-phase, execute-phase, verify-work, ship, next
- **Workstreams:** Parallel milestone management
- **UI Design:** ui-phase, ui-review
- **Quality:** review, pr-branch, audit-uat
- **Utilities:** health, stats, debug, fast, thread

### Wave-Based Execution
Plans execute in dependency-ordered waves with parallel execution within each wave, maximizing throughput while respecting dependencies.

### File-Based State
All project state lives in `.planning/` as human-readable Markdown and JSON:
- `PROJECT.md` - Project vision and constraints
- `REQUIREMENTS.md` - Scoped requirements
- `ROADMAP.md` - Phase breakdown
- `STATE.md` - Living memory across sessions
- `phases/` - Per-phase context, plans, summaries, verification

## Architecture

```
User /gsd:command
    │
    v
Commands Layer (commands/gsd/*.md)
    │
    v
Workflow Layer (get-shit-done/workflows/*.md)
    │
    v
CLI Tools Layer (gsd-tools.cjs + lib/*.cjs)
    │
    v
File System (.planning/)
```

### Agent Categories (18 total)
- **Researchers (3):** project-researcher, phase-researcher, ui-researcher
- **Analyzers (2):** assumptions-analyzer, advisor-researcher
- **Synthesizers (1):** research-synthesizer
- **Planners (1):** planner
- **Executors (1):** executor
- **Checkers (3):** plan-checker, integration-checker, ui-checker
- **Verifiers (1):** verifier
- **Auditors (2):** nyquist-auditor, ui-auditor
- **Mappers (1):** codebase-mapper
- **Debuggers (1):** debugger

## Technology Stack

- **Runtime:** Node.js (LTS and Current versions tested)
- **Language:** JavaScript (CommonJS, `.cjs` files)
- **CLI Tool:** Single `gsd-tools.cjs` with 15 domain modules
- **Testing:** Node.js built-in `node:test` runner
- **No external dependencies** in core modules

## Installation

```bash
npx get-shit-done-cc@latest
```

Supports global installation (all projects) or local installation (single project).

## Project Statistics (as of v1.29.0)
- 44 commands
- 46 workflows
- 18 specialized agents
- 15 CLI tool modules
- 13 shared reference documents
- Multi-platform: macOS, Windows, Linux
- Security: CVE response program with security@gsd.build

## Related Documentation
- [Architecture](./02-architecture.md) - System design and component relationships
- [Code Quality](./05-code-quality.md) - Testing, tooling, and standards
- [Security](./08-security.md) - Security hardening and secrets management
