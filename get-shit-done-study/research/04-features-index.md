# GSD Features Index

Prioritized feature list derived from README.md documentation and directory structure analysis.

---

## Core Features

### 1. Project Initialization
- **Description:** Single-command project setup that transforms ideas into structured plans via guided questions, research, and roadmap generation.
- **Key files:** `get-shit-done/workflows/new-project.md`, `get-shit-done/agents/gsd-project-researcher.md`
- **Priority:** core

### 2. Phase Discussion
- **Description:** Captures implementation decisions and preferences before planning, producing CONTEXT.md that locks user decisions and feeds researcher/planner.
- **Key files:** `get-shit-done/workflows/discuss-phase.md`, `get-shit-done/agents/gsd-assumptions-analyzer.md`
- **Priority:** core

### 3. Phase Planning
- **Description:** Multi-agent planning with research synthesis, XML-structured plan creation, and automated verification against requirements. Creates 2-3 atomic task plans per phase.
- **Key files:** `get-shit-done/workflows/plan-phase.md`, `get-shit-done/agents/gsd-planner.md`, `get-shit-done/agents/gsd-plan-checker.md`, `get-shit-done/agents/gsd-phase-researcher.md`
- **Priority:** core

### 4. Wave-Based Phase Execution
- **Description:** Parallel plan execution with fresh 200k-token contexts per plan, atomic git commits per task, and automatic dependency management across waves.
- **Key files:** `get-shit-done/workflows/execute-phase.md`, `get-shit-done/agents/gsd-executor.md`
- **Priority:** core

### 5. Work Verification (UAT)
- **Description:** User acceptance testing workflow with guided walkthroughs, automated failure diagnosis, and fix plan generation.
- **Key files:** `get-shit-done/workflows/verify-work.md`, `get-shit-done/agents/gsd-verifier.md`
- **Priority:** core

### 6. Quick Task Mode
- **Description:** Fast-path for ad-hoc tasks with GSD guarantees (atomic commits, state tracking) but without full planning overhead. Supports `--discuss`, `--research`, and `--full` flags.
- **Key files:** `get-shit-done/workflows/quick.md`
- **Priority:** core

### 7. Codebase Mapping
- **Description:** Analyzes existing codebase before new-project to understand stack, architecture, conventions, and concerns. Spawns parallel agents for each analysis area.
- **Key files:** `get-shit-done/workflows/map-codebase.md`, `get-shit-done/agents/gsd-codebase-mapper.md`
- **Priority:** core

---

## Secondary Features

### 8. Multi-Agent Orchestration
- **Description:** Thin orchestrator pattern that spawns specialized agents (researchers, planners, executors, verifiers) and routes results between stages. Keeps main context at 30-40%.
- **Key files:** `get-shit-done/agents/` (all), `get-shit-done/references/model-profiles.md`
- **Priority:** secondary

### 9. Context Engineering System
- **Description:** File-based context management using PROJECT.md, research/, REQUIREMENTS.md, ROADMAP.md, STATE.md, PLAN.md, SUMMARY.md to prevent context rot across sessions.
- **Key files:** `get-shit-done/references/` (planning-config.md, continuation-format.md), `.planning/` directory structure
- **Priority:** secondary

### 10. UI Design Workflow
- **Description:** UI phase contract generation (UI-SPEC.md) and retroactive 6-pillar visual audit of implemented frontend code.
- **Key files:** `get-shit-done/workflows/ui-phase.md`, `get-shit-done/workflows/ui-review.md`, `get-shit-done/agents/gsd-ui-auditor.md`, `get-shit-done/agents/gsd-ui-researcher.md`
- **Priority:** secondary

### 11. Git Integration & Atomic Commits
- **Description:** Per-task atomic commits with surgical traceability, branch strategies (none/phase/milestone), and PR creation from verified work.
- **Key files:** `get-shit-done/references/git-integration.md`, `get-shit-done/references/git-planning-commit.md`, `get-shit-done/workflows/ship.md`, `get-shit-done/workflows/pr-branch.md`
- **Priority:** secondary

### 12. Security Hardening
- **Description:** Defense-in-depth security including path traversal prevention, prompt injection detection, safe JSON parsing, and shell argument validation.
- **Key files:** `hooks/gsd-prompt-guard.js`, `get-shit-done/agents/gsd-integration-checker.md`
- **Priority:** secondary

### 13. Session Management
- **Description:** Pause/resume work with HANDOFF.json persistence, session reports, and health checks for `.planning/` directory integrity.
- **Key files:** `get-shit-done/workflows/pause-work.md`, `get-shit-done/workflows/resume-work.md`, `get-shit-done/workflows/session-report.md`, `get-shit-done/workflows/health.md`
- **Priority:** secondary

### 14. Debugging & Forensics
- **Description:** Systematic debugging with persistent state (`/gsd:debug`) and post-mortem investigation of failed workflow runs (`/gsd:forensics`).
- **Key files:** `get-shit-done/workflows/debug.md`, `get-shit-done/workflows/forensics.md`, `get-shit-done/agents/gsd-debugger.md`
- **Priority:** secondary

### 15. Milestone & Phase Management
- **Description:** Complete milestone lifecycle (new, audit, complete) and phase management (add, insert, remove, validate) with gap closure planning.
- **Key files:** `get-shit-done/workflows/complete-milestone.md`, `get-shit-done/workflows/audit-milestone.md`, `get-shit-done/workflows/add-phase.md`, `get-shit-done/workflows/plan-milestone-gaps.md`
- **Priority:** secondary

### 16. Multi-Project Workspaces
- **Description:** Isolated workspaces with repo copies (worktrees or clones) for parallel milestone work, plus workstream management for namespaced parallel tracks.
- **Key files:** `get-shit-done/workflows/new-workspace.md`, `get-shit-done/workflows/workstreams.md`
- **Priority:** secondary

### 17. Backlog, Seeds & Threads
- **Description:** Long-term idea management: backlog parking lot, forward-looking seeds with trigger conditions, and persistent context threads for cross-session work.
- **Key files:** `get-shit-done/workflows/plant-seed.md`, `get-shit-done/workflows/add-backlog.md`, `get-shit-done/workflows/thread.md`
- **Priority:** secondary

---

## Feature Count Summary

| Tier | Count |
|------|-------|
| Core | 7 |
| Secondary | 10 |
| **Total** | **17** |

## Key Architectural Patterns

- **Orchestrator Pattern:** Thin coordinators spawn specialized agents, never do heavy lifting
- **Vertical Slices:** Plans organized by feature end-to-end (not horizontal layers) for parallel execution
- **Context Budget:** 50% context target per plan, fresh 200k tokens per executor run
- **Wave Execution:** Independent plans run in parallel; dependent plans wait for dependencies
- **Atomic Commits:** Each task gets its own commit for bisect-friendly history
