# oh-my-openagent Feature Index

## Project Overview

oh-my-openagent (OmO) is a plugin for OpenCode that provides multi-agent orchestration, world-class developer tools, and opinionated productivity features. It orchestrates multiple AI models (Claude, Kimi, GLM, GPT, Gemini, Grok) for different specialized tasks.

**Repository:** `/Users/sheldon/Documents/claw/reference/oh-my-openagent`
**Source Root:** `src/` (TypeScript)
**Config:** `.opencode/` (project-level), `~/.config/opencode/` (user-level)
**Binaries:** `packages/` (platform-specific native builds)

---

## Core Features (Priority Tier 1)

### 1. Multi-Agent Orchestration (Discipline Agents)

**Description:** A team of 11 specialized AI agents, each tuned to specific model strengths. Sisyphus is the main orchestrator that plans, delegates, and drives tasks to completion with aggressive parallel execution. Subagents include Hephaestus (autonomous deep worker), Prometheus (strategic planner), Oracle (architecture/debugging), Librarian (docs/code search), Explore (fast grep), Multimodal-Looker (visual analysis), Atlas (todo-list orchestrator), Momus (plan reviewer), Metis (plan consultant), and Sisyphus-Junior (category-based executor).

**Key Files/Locations:**
- `src/agents/sisyphus.ts` -- Main orchestrator agent
- `src/agents/hephaestus.ts` -- Autonomous deep worker
- `src/agents/prometheus/` -- Strategic planner
- `src/agents/oracle.ts` -- Architecture and debugging consultant
- `src/agents/librarian.ts` -- Documentation and code search
- `src/agents/explore.ts` -- Fast codebase exploration
- `src/agents/metis.ts` -- Plan consultant
- `src/agents/momus.ts` -- Plan reviewer
- `src/agents/atlas/` -- Todo-list orchestrator
- `src/agents/sisyphus-junior/` -- Category-based executor
- `src/agents/dynamic-agent-prompt-builder.ts` -- Agent prompt construction
- `src/agents/types.ts` -- Agent type definitions

**Priority:** CORE

---

### 2. Hash-Anchored Edit Tool (Hashline)

**Description:** An edit tool that tags every line with a content hash (`LINE#ID` format). When editing, the hash validates that the file hasn't changed since the last read. If the hash does not match, the edit is rejected before corruption occurs. Eliminates stale-line errors entirely. Inspired by oh-my-pi. This single change improved Grok Code Fast success rate from 6.7% to 68.3%.

**Key Files/Locations:**
- `src/tools/hashline-edit/` -- Hash-anchored edit implementation (38 files)
- `src/hooks/hashline-read-enhancer/` -- Adds hash markers to read output
- `src/hooks/hashline-edit-diff-enhancer/` -- Enhances edit with diff markers

**Priority:** CORE

---

### 3. Background Agents & Parallel Execution

**Description:** Fire multiple specialized agents in parallel, continuing main work while specialists produce results. Supports tmux integration for visual multi-agent sessions with real-time pane output. Includes `call_omo_agent` and `task` tools for delegation with `run_in_background` flag.

**Key Files/Locations:**
- `src/tools/delegate-task/` -- Category-based task delegation (56 files)
- `src/tools/background-task/` -- Background task management (25 files)
- `src/tools/call-omo-agent.ts` -- Spawn explore/librarian agents
- `src/hooks/background-notification/` -- Notifications on background completion
- `src/hooks/interactive-bash-session/` -- Tmux session management

**Priority:** CORE

---

### 4. LSP + AST-Grep Tools

**Description:** IDE-precision tools for every agent. LSP provides workspace rename, goto definition, find references, and pre-build diagnostics. AST-Grep provides pattern-aware code search and rewriting across 25 languages.

**Key Files/Locations:**
- `src/tools/lsp/` -- LSP integration (38 files): `lsp_rename`, `lsp_goto_definition`, `lsp_find_references`, `lsp_diagnostics`
- `src/tools/ast-grep/` -- AST-aware search and replace (15 files): `ast_grep_search`, `ast_grep_replace`

**Priority:** CORE

---

### 5. Built-in MCPs (Model Context Protocol)

**Description:** Three built-in MCP servers always available: Exa (real-time web search), Context7 (official documentation lookup for any library/framework), and Grep.app (ultra-fast code search across public GitHub repos). Also supports skill-embedded MCPs where skills carry their own on-demand MCP servers that spin up scoped to task and disappear when done, keeping context window clean.

**Key Files/Locations:**
- `src/mcp/` -- MCP server implementations
- `src/tools/skill-mcp/` -- Skill-embedded MCP invocation tool
- `src/hooks/claude-code-hooks/` -- Claude Code hooks compatibility

**Priority:** CORE

---

### 6. ultrawork / Ralph Loop

**Description:** A self-referential development loop. Type `ultrawork` or `/ralph-loop` and the agent works continuously toward the goal, detecting `<promise>DONE</promise>` for completion. If the agent stops without completing, the system yanks it back. Does not stop until 100% done. `/ulw-loop` runs the same with ultrawork mode (maximum intensity, parallel agents, aggressive exploration).

**Key Files/Locations:**
- `src/hooks/ralph-loop/` -- Self-referential loop management (27 files)
- `src/hooks/keyword-detector/` -- Detects `ultrawork`/`ulw` keywords to activate mode
- `.opencode/command/ralph-loop.md` -- Ralph loop command definition

**Priority:** CORE

---

### 7. Todo Enforcer & Task System

**Description:** A file-based task system with full dependency management. Tasks persist across sessions (stored in `.sisyphus/tasks/`), support `blockedBy`/`blocks` relationships, and run parallel tasks automatically when blockers complete. The `todo-continuation-enforcer` hook yanks idle agents back to work if they go off-track.

**Key Files/Locations:**
- `src/tools/task/` -- Task management tools (15 files): `task_create`, `task_get`, `task_list`, `task_update`
- `src/hooks/todo-continuation-enforcer/` -- Enforces todo completion
- `src/hooks/compaction-todo-preserver/` -- Preserves todo state during compaction
- `src/hooks/tasks-todowrite-disabler/` -- Disables TodoWrite when task system active

**Priority:** CORE

---

## Secondary Features (Priority Tier 2)

### 8. Category-Based Task Delegation

**Description:** A system of agent configuration presets (categories) optimized for specific domains: `visual-engineering` (frontend/UI), `ultrabrain` (deep reasoning), `deep` (autonomous problem-solving), `artistry` (creative tasks), `quick` (single-file changes), `writing`, and two `unspecified` tiers. Instead of delegating to a single model, specialists are selected based on task category. Model selection is automatic.

**Key Files/Locations:**
- `src/agents/dynamic-agent-prompt-builder.ts` -- Category-to-agent mapping
- `src/shared/session-category-registry.ts` -- Category registration
- `src/hooks/category-skill-reminder/` -- Reminds agents about available categories
- `src/agents/sisyphus-junior/` -- Category-spawned executor agent

**Priority:** SECONDARY

---

### 9. Built-in Skills (git-master, playwright, frontend-ui-ux)

**Description:** Pre-packaged skill workflows with embedded MCPs and detailed instructions. `git-master` handles atomic commits, style detection, and rebase surgery. `playwright` provides browser automation for testing and screenshots. `frontend-ui-ux` brings a designer-turned-developer persona for UI implementation. Also includes `agent-browser`, `dev-browser`, and `playwright-cli`.

**Key Files/Locations:**
- `.opencode/skills/` -- Skill definitions
- `src/tools/skill/` -- Skill loading and invocation tool
- `src/hooks/category-skill-reminder/` -- Category-skill pairing reminders

**Priority:** SECONDARY

---

### 10. Deep Initialization (/init-deep)

**Description:** Generates hierarchical `AGENTS.md` files throughout the project. Running `/init-deep` creates project-wide, directory-specific, and component-specific context files. Agents auto-read relevant context automatically without manual management.

**Key Files/Locations:**
- `.opencode/command/` -- Command definitions (including `init-deep`)
- `src/hooks/directory-agents-injector/` -- Auto-injects AGENTS.md context

**Priority:** SECONDARY

---

### 11. Prometheus Strategic Planner

**Description:** An interview-mode strategic planner. Before touching code, Prometheus questions the user, identifies scope and ambiguities, and builds a verified plan. The `/start-work` command invokes Prometheus, and Atlas executes the resulting plan systematically. Metis acts as a pre-planning consultant identifying hidden intentions and AI failure points.

**Key Files/Locations:**
- `src/agents/prometheus/` -- Prometheus planning agent
- `src/agents/metis.ts` -- Plan consultant (pre-planning analysis)
- `src/hooks/start-work/` -- `/start-work` command handling
- `src/hooks/prometheus-md-only/` -- Enforces markdown-only output

**Priority:** SECONDARY

---

### 12. Context Window Management & Session Compaction

**Description:** Proactive session compaction that preserves critical context before hitting token limits. Includes context window monitoring, dynamic truncation of tool outputs (Grep, Glob, LSP, AST-grep), and a `compaction-context-injector` hook. The `preemptive-compaction` hook compacts sessions before limits are reached.

**Key Files/Locations:**
- `src/hooks/preemptive-compaction.ts` -- Proactive compaction (30+ files, tests)
- `src/hooks/context-window-monitor.ts` -- Token consumption tracking
- `src/hooks/tool-output-truncator.ts` -- Dynamic output truncation
- `src/hooks/compaction-context-injector/` -- Context preservation during compaction
- `src/shared/dynamic-truncator.ts` -- Truncation logic

**Priority:** SECONDARY

---

### 13. Session Recovery & Model Fallback

**Description:** Multi-layer recovery system. `session-recovery` handles missing tool results and thinking block issues. `runtime-fallback` automatically switches to backup models on retryable API errors (429, 503, 529), provider key misconfigurations, and auto-retry signals. `model-fallback` manages fallback chains when primary models are unavailable.

**Key Files/Locations:**
- `src/hooks/session-recovery/` -- Session error recovery (20 files)
- `src/hooks/runtime-fallback/` -- Model fallback on API errors (30 files)
- `src/hooks/model-fallback/` -- Model availability fallback
- `src/shared/model-error-classifier.ts` -- Error classification
- `src/shared/fallback-chain-from-models.ts` -- Fallback chain logic

**Priority:** SECONDARY

---

### 14. Claude Code Compatibility Layer

**Description:** Full compatibility for Claude Code hooks, commands, skills, agents, MCPs, and plugins. All existing Claude Code configurations work unchanged. Includes support for loading from `~/.claude/`, `.claude/`, and Claude Code's settings.json hooks system.

**Key Files/Locations:**
- `src/plugin-handlers/` -- Plugin compatibility handlers (27 files)
- `src/hooks/claude-code-hooks/` -- Claude Code hooks executor
- `src/shared/opencode-config-dir.ts` -- Config directory resolution
- `src/shared/opencode-command-dirs.ts` -- Command path resolution
- `src/shared/skill-path-resolver.ts` -- Skill path resolution
- `src/plugin/` -- Plugin system implementation

**Priority:** SECONDARY

---

### 15. Session Notifications & Auto-Update Checking

**Description:** OS-native notifications when background agents complete or agents go idle. Works on macOS, Linux, and Windows. `auto-update-checker` notifies of new versions on session creation.

**Key Files/Locations:**
- `src/hooks/session-notification.ts` -- OS notification system (27 files, extensive tests)
- `src/hooks/background-notification/` -- Background task completion alerts
- `src/hooks/auto-update-checker/` -- Version update checker

**Priority:** SECONDARY

---

### 16. Comment Checker & Code Quality Hooks

**Description:** The `comment-checker` hook reminds agents to reduce excessive comments after code writes, while smartly ignoring BDD tests, directives, and docstrings. The `thinking-block-validator` validates thinking blocks to prevent API errors. `edit-error-recovery` recovers from edit tool failures.

**Key Files/Locations:**
- `src/hooks/comment-checker/` -- Comment quality enforcement (12 files)
- `src/hooks/thinking-block-validator/` -- Thinking block validation
- `src/hooks/edit-error-recovery/` -- Edit failure recovery
- `src/hooks/write-existing-file-guard/` -- Prevents overwrites without reading

**Priority:** SECONDARY

---

## Summary Table

| # | Feature | Priority | Key Directory |
|---|---------|----------|---------------|
| 1 | Multi-Agent Orchestration (11 agents) | CORE | `src/agents/` |
| 2 | Hash-Anchored Edit Tool | CORE | `src/tools/hashline-edit/` |
| 3 | Background Agents & Parallel Execution | CORE | `src/tools/background-task/`, `src/tools/delegate-task/` |
| 4 | LSP + AST-Grep Tools | CORE | `src/tools/lsp/`, `src/tools/ast-grep/` |
| 5 | Built-in MCPs (Exa, Context7, Grep.app) | CORE | `src/mcp/` |
| 6 | ultrawork / Ralph Loop | CORE | `src/hooks/ralph-loop/` |
| 7 | Todo Enforcer & Task System | CORE | `src/tools/task/`, `src/hooks/todo-continuation-enforcer/` |
| 8 | Category-Based Delegation | SECONDARY | `src/agents/dynamic-agent-prompt-builder.ts` |
| 9 | Built-in Skills (git-master, playwright) | SECONDARY | `.opencode/skills/` |
| 10 | Deep Initialization (/init-deep) | SECONDARY | `.opencode/command/` |
| 11 | Prometheus Strategic Planner | SECONDARY | `src/agents/prometheus/` |
| 12 | Context Window Management | SECONDARY | `src/hooks/preemptive-compaction.ts` |
| 13 | Session Recovery & Model Fallback | SECONDARY | `src/hooks/runtime-fallback/`, `src/hooks/session-recovery/` |
| 14 | Claude Code Compatibility | SECONDARY | `src/plugin-handlers/`, `src/hooks/claude-code-hooks/` |
| 15 | Session Notifications | SECONDARY | `src/hooks/session-notification.ts` |
| 16 | Comment Checker & Quality Hooks | SECONDARY | `src/hooks/comment-checker/` |

**Total: 7 Core Features, 9 Secondary Features (16 features)**

---

## Cross-Reference: README Claims vs. Implementation

| README Claim | Implemented | Notes |
|---|---|---|
| Discipline Agents (Sisyphus, Hephaestus, Prometheus, Oracle, Librarian, Explore) | YES | All present in `src/agents/` |
| ultrawork / ulw | YES | Via `src/hooks/keyword-detector/` and ralph-loop |
| IntentGate | YES | `buildGeminiIntentGateEnforcement()` in `src/agents/sisyphus/gemini.ts:213` — Gemini-specific prompt enforcing intent verbalization before action |
| Hash-Anchored Edit Tool | YES | `src/tools/hashline-edit/` |
| LSP + AST-Grep | YES | `src/tools/lsp/`, `src/tools/ast-grep/` |
| Background Agents | YES | `src/tools/background-task/`, `src/tools/delegate-task/` |
| Built-in MCPs (Exa, Context7, Grep.app) | YES | `src/mcp/` |
| Ralph Loop / /ulw-loop | YES | `src/hooks/ralph-loop/` |
| Todo Enforcer | YES | `src/hooks/todo-continuation-enforcer/` |
| Comment Checker | YES | `src/hooks/comment-checker/` |
| Tmux Integration | YES | `src/tools/interactive-bash/`, `src/hooks/interactive-bash-session/` |
| Claude Code Compatible | YES | `src/plugin-handlers/`, `src/hooks/claude-code-hooks/` |
| Skill-Embedded MCPs | YES | `src/tools/skill-mcp/` |
| Prometheus Planner | YES | `src/agents/prometheus/` |
| /init-deep | YES | Command definition in `.opencode/command/` |
