# Superpowers Architecture

## Overview

**Project:** Superpowers - AI coding agent workflow system
**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`
**Version:** 5.0.6
**Pattern:** Skills-based plugin system with layered platform abstraction

---

## Architectural Decisions

### AD-01: Skills as Primary Reusable Unit

Skills are markdown documents with YAML frontmatter, not commands or agents. The `description` field serves as the triggering condition for AI agent discovery.

**Evidence:** `skills/using-superpowers/SKILL.md` instructs agents to use the `Skill` tool. Commands in `commands/` are deprecated.

---

### AD-02: Markdown as Skill Format

Skills use markdown with YAML frontmatter and embedded Graphviz flowcharts. No runtime dependency on skill processing library.

**Evidence:** All 14 skills follow `SKILL.md` naming convention with standard YAML frontmatter parsing.

---

### AD-03: Flat Namespace for Skills

All skills live in `skills/` directory without nested categories. Prevents hierarchy traversal complexity and name collisions.

**Evidence:** `skills/` contains 14 skill directories at the same level, each with unique hyphenated names.

---

### AD-04: Zero-Dependency Visual Server

Visual brainstorming server uses only Node.js built-in modules (`http`, `crypto`, `fs`).

**Evidence:** `skills/brainstorming/scripts/server.cjs` imports only `crypto`, `http`, `fs`, `path` with no npm dependencies.

---

### AD-05: File-Based IPC for Visual Companion

Screen updates via filesystem, events via JSONL. Survives server restarts and works across process boundaries.

**Evidence:** `skills/brainstorming/scripts/server.cjs` writes HTML to `screen_dir/`, events to `state_dir/events` as JSONL.

---

## Module Responsibilities

### Platform Integration Layer

**Directory:** `.claude-plugin/`, `.cursor-plugin/`, `.opencode/plugins/`, `.codex/`, `gemini-extension.json`

Provides platform-specific plugin loading and hook configuration for each supported AI coding environment.

| Platform | Entry Point | Hook Config | Skills Location |
|----------|-------------|-------------|----------------|
| Claude Code | `plugin.json` + `hooks/session-start` | `hooks/hooks.json` | `skills/` |
| Cursor | `plugin.json` + `hooks/session-start` | `hooks/hooks-cursor.json` | `skills/` via plugin.json |
| OpenCode | `.opencode/plugins/superpowers.js` | Plugin config hook | Dynamically injected |
| Codex | `.codex/INSTALL.md` | Instructions only | Reference docs |
| Gemini CLI | `gemini-extension.json` | Native skill activation | Skills auto-loaded |

---

### Hook System Layer

**Directory:** `hooks/`

Initializes skill context at session start by injecting `using-superpowers` skill content into the agent's context window.

**Files:**
- `session-start` - Session initialization script (bash)
- `hooks.json` - Claude Code hook configuration
- `hooks-cursor.json` - Cursor hook configuration
- `run-hook.cmd` - Cross-platform polyglot wrapper (cmd/bash)

---

### Skills Library Layer

**Directory:** `skills/`

14 skills organized by category:

| Category | Skills |
|----------|--------|
| Testing | `test-driven-development/` |
| Debugging | `systematic-debugging/`, `verification-before-completion/` |
| Collaboration | `brainstorming/`, `writing-plans/`, `executing-plans/`, `dispatching-parallel-agents/`, `requesting-code-review/`, `receiving-code-review/`, `using-git-worktrees/`, `finishing-a-development-branch/`, `subagent-driven-development/` |
| Meta | `writing-skills/`, `using-superpowers/` |

---

### Subagent System Layer

**Directory:** `skills/subagent-driven-development/`

Controller pattern where main agent coordinates specialized subagents with two-stage review after each task.

**Files:**
- `SKILL.md` - Main skill definition
- `implementer-prompt.md` - Template for implementation subagents
- `spec-reviewer-prompt.md` - Template for spec compliance review
- `code-quality-reviewer-prompt.md` - Template for code quality review

---

### Visual Brainstorming Subsystem

**Directory:** `skills/brainstorming/scripts/`

**Files:**
- `server.cjs` - WebSocket server (Node.js, zero dependencies)
- `frame-template.html` - HTML frame with CSS styling
- `helper.js` - Client-side interaction helper
- `start-server.sh` / `stop-server.sh` - Platform-specific launch scripts

---

## Communication Patterns

### Hook Initialization Flow

```
Platform Hook Trigger → run-hook.cmd → session-start script
  → Read skills/using-superpowers/SKILL.md
  → Escape for JSON embedding
  → Output platform-specific format
  → Agent receives bootstrap context
```

### Visual Brainstorming Flow

```
Agent writes HTML → screen_dir/ → server.cjs (file watcher)
  → HTTP endpoint (GET /) → Serves newest HTML wrapped in frame
  → WebSocket endpoint → Broadcasts reload events
User browser → http://localhost:PORT → User interactions
  → events file (JSONL) → Agent reads on next turn
```

### Subagent Status Reporting

Status reporting via structured format:
- `DONE` - Task completed successfully
- `DONE_WITH_CONCERNS` - Completed but with doubts
- `NEEDS_CONTEXT` - Missing information to proceed
- `BLOCKED` - Cannot complete task

---

## Dependency Graph

```
Platform Plugins
  └──> Hook System Layer
            └──> Skills Library (skills/)
                      ├──> using-superpowers/ (entry point)
                      ├──> brainstorming/ → visual-companion.md → server.cjs
                      ├──> subagent-driven-development/
                      │         ├──> implementer-prompt.md
                      │         ├──> spec-reviewer-prompt.md
                      │         └──> code-quality-reviewer-prompt.md
                      ├──> test-driven-development/
                      ├──> systematic-debugging/
                      └──> writing-skills/ → testing-skills-with-subagents.md
```

---

## Cross-Cutting Concerns

### Skill Interdependencies

- `subagent-driven-development` requires `using-git-worktrees`, `writing-plans`, `requesting-code-review`, `finishing-a-development-branch`
- `systematic-debugging` references `test-driven-development` for bug fix verification
- `writing-skills` requires understanding `test-driven-development` first

### Platform Adaptation Layer

Each skill is written for Claude Code tool names. Platform adapters provide tool mappings:
- **OpenCode:** `TodoWrite` → `todowrite`, `Skill` tool → native `skill` tool
- **Gemini CLI:** Tool mapping loaded via `GEMINI.md`
- **Codex:** Reference docs in `references/codex-tools.md`
