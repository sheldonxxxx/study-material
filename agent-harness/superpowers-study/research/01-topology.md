OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/01-topology.md

# Superpowers Project Topology

## Overview

**Project:** Superpowers - A complete software development workflow for AI coding agents
**Repository:** /Users/sheldon/Documents/claw/reference/superpowers
**Type:** Multi-platform plugin (Claude Code, Cursor, Codex, OpenCode, Gemini CLI)
**Version:** 5.0.6

## Directory Tree

```
superpowers/
├── .claude-plugin/              # Claude Code plugin config
│   ├── marketplace.json
│   └── plugin.json
├── .cursor-plugin/              # Cursor plugin config
│   └── plugin.json
├── .codex/                       # Codex installation docs
│   └── INSTALL.md
├── .opencode/                    # OpenCode installation docs + plugin
│   ├── INSTALL.md
│   └── plugins/
│       └── superpowers.js       # Main OpenCode plugin entry
├── agents/                      # Agent prompt templates
│   └── code-reviewer.md
├── commands/                    # Command definitions
│   ├── brainstorm.md
│   ├── execute-plan.md
│   └── write-plan.md
├── docs/                        # Documentation
│   ├── README.codex.md
│   ├── README.opencode.md
│   ├── testing.md
│   ├── windows/
│   │   └── polyglot-hooks.md
│   ├── plans/                   # Historical project plans
│   │   ├── 2025-11-22-opencode-support-design.md
│   │   ├── 2025-11-22-opencode-support-implementation.md
│   │   ├── 2025-11-28-skills-improvements-from-user-feedback.md
│   │   └── 2026-01-17-visual-brainstorming.md
│   └── superpowers/
│       ├── plans/              # Superpowers feature plans
│       │   ├── 2026-01-22-document-review-system.md
│       │   ├── 2026-02-19-visual-brainstorming-refactor.md
│       │   ├── 2026-03-11-zero-dep-brainstorm-server.md
│       │   └── 2026-03-23-codex-app-compatibility.md
│       └── specs/             # Design specifications
│           ├── 2026-01-22-document-review-system-design.md
│           ├── 2026-02-19-visual-brainstorming-refactor-design.md
│           ├── 2026-03-11-zero-dep-brainstorm-server-design.md
│           └── 2026-03-23-codex-app-compatibility-design.md
├── hooks/                       # Session hooks
│   ├── hooks.json              # Claude Code hooks config
│   ├── hooks-cursor.json      # Cursor hooks config
│   ├── run-hook.cmd           # Hook runner
│   └── session-start          # Session initialization script
├── skills/                      # Core skills library (MAIN CONTENT)
│   ├── brainstorming/
│   │   ├── SKILL.md
│   │   ├── spec-document-reviewer-prompt.md
│   │   ├── visual-companion.md
│   │   └── scripts/           # Visual brainstorming server
│   │       ├── frame-template.html
│   │       ├── helper.js
│   │       ├── server.cjs
│   │       ├── start-server.sh
│   │       └── stop-server.sh
│   ├── dispatching-parallel-agents/
│   │   └── SKILL.md
│   ├── executing-plans/
│   │   └── SKILL.md
│   ├── finishing-a-development-branch/
│   │   └── SKILL.md
│   ├── receiving-code-review/
│   │   └── SKILL.md
│   ├── requesting-code-review/
│   │   ├── SKILL.md
│   │   └── code-reviewer.md
│   ├── subagent-driven-development/
│   │   ├── SKILL.md
│   │   ├── code-quality-reviewer-prompt.md
│   │   ├── implementer-prompt.md
│   │   └── spec-reviewer-prompt.md
│   ├── systematic-debugging/
│   │   ├── SKILL.md
│   │   ├── CREATION-LOG.md
│   │   ├── condition-based-waiting-example.ts
│   │   ├── condition-based-waiting.md
│   │   ├── defense-in-depth.md
│   │   ├── find-polluter.sh
│   │   ├── root-cause-tracing.md
│   │   ├── test-academic.md
│   │   ├── test-pressure-1.md
│   │   ├── test-pressure-2.md
│   │   └── test-pressure-3.md
│   ├── test-driven-development/
│   │   ├── SKILL.md
│   │   └── testing-anti-patterns.md
│   ├── using-git-worktrees/
│   │   └── SKILL.md
│   ├── using-superpowers/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── codex-tools.md
│   │       └── gemini-tools.md
│   ├── verification-before-completion/
│   │   └── SKILL.md
│   ├── writing-plans/
│   │   ├── SKILL.md
│   │   └── plan-document-reviewer-prompt.md
│   └── writing-skills/
│       └── (SKILL.md + content)
├── tests/                       # Test suites
│   ├── brainstorm-server/       # WebSocket server tests
│   ├── claude-code/            # Claude Code integration tests
│   ├── explicit-skill-requests/ # Skill triggering tests
│   ├── opencode/               # OpenCode platform tests
│   ├── skill-triggering/        # Skill activation tests
│   └── subagent-driven-dev/     # SDD integration tests
│       ├── go-fractals/
│       └── svelte-todo/
├── .github/                     # GitHub config
│   ├── FUNDING.yml
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
├── gemini-extension.json       # Gemini CLI extension config
├── package.json                 # npm package manifest
├── README.md
├── CHANGELOG.md
├── RELEASE-NOTES.md
├── LICENSE
├── CODE_OF_CONDUCT.md
├── GEMINI.md
├── GEMINI-README.md
└── .gitattributes, .gitignore
```

## Entry Points

### Plugin Entry Points (per platform)

| Platform | Entry Point |
|----------|-------------|
| **Claude Code** | `.claude-plugin/plugin.json` + `hooks/session-start` |
| **Cursor** | `.cursor-plugin/plugin.json` + `hooks/session-start` |
| **OpenCode** | `.opencode/plugins/superpowers.js` |
| **Codex** | Instructions via `.codex/INSTALL.md` |
| **Gemini CLI** | `gemini-extension.json` |

### Primary Skills Entry

- **Main skill catalog:** `skills/` directory
- **Core introduction skill:** `skills/using-superpowers/SKILL.md`
- **Session initialization:** `hooks/session-start` (reads using-superpowers and injects context)

### Command Entry Points

- `commands/brainstorm.md`
- `commands/execute-plan.md`
- `commands/write-plan.md`

## Architectural Layers

### 1. Platform Integration Layer
```
.claude-plugin/, .cursor-plugin/, .opencode/, .codex/, gemini-extension.json
```
Provides platform-specific plugin loading and hook configuration.

### 2. Hook System Layer
```
hooks/
```
Session hooks for initialization and skill triggering across platforms.

### 3. Skills Library Layer (Core)
```
skills/
```
14 distinct skills organized by category:

**Testing:**
- `test-driven-development/` - RED-GREEN-REFACTOR methodology

**Debugging:**
- `systematic-debugging/` - Root cause analysis techniques
- `verification-before-completion/` - Fix validation

**Collaboration:**
- `brainstorming/` - Socratic design refinement
- `writing-plans/` - Implementation planning
- `executing-plans/` - Plan execution with checkpoints
- `dispatching-parallel-agents/` - Concurrent subagent workflows
- `requesting-code-review/` - Pre-review checklists
- `receiving-code-review/` - Feedback response
- `using-git-worktrees/` - Branch management
- `finishing-a-development-branch/` - Merge workflow
- `subagent-driven-development/` - Two-stage review SDD

**Meta:**
- `writing-skills/` - Skill creation methodology
- `using-superpowers/` - System introduction

### 4. Agent Prompts Layer
```
agents/, commands/
```
Standalone prompts consumed by AI agents.

### 5. Documentation Layer
```
docs/
```
Design documents, specs, and platform-specific README files.

### 6. Testing Layer
```
tests/
```
Integration tests for each supported platform and feature.

## Key Patterns

### Skill Structure
Each skill follows the pattern:
- `SKILL.md` - Main skill definition
- Supporting files (prompts, scripts, references) as needed

### Visual Brainstorming Subsystem
Unique component with dedicated server:
- `skills/brainstorming/scripts/server.cjs` - WebSocket server
- `skills/brainstorming/scripts/frame-template.html` - UI frame
- `skills/brainstorming/scripts/helper.js` - Client helper

### Cross-Platform Hook System
`hooks/session-start` detects platform via environment variables:
- `CURSOR_PLUGIN_ROOT` - Cursor
- `CLAUDE_PLUGIN_ROOT` - Claude Code
- Fallback - other platforms
