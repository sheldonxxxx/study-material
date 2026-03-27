OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/04-features-index.md

# Superpowers Feature Index

## Feature Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`
**Purpose:** Complete software development workflow for coding agents via composable skills

---

## Core Features (Priority Tier 1)

### 1. Brainstorming
**Description:** Socratic design refinement skill that activates before writing code. Refines rough ideas through questions, explores alternatives, and presents design in digestible sections for validation.
**Key Files/Locations:**
- `skills/brainstorming/SKILL.md`
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/brainstorming/visual-companion.md`
- `skills/brainstorming/scripts/`
**Priority:** CORE

### 2. Writing Plans
**Description:** Creates detailed implementation plans with bite-sized tasks (2-5 minutes each). Every task includes exact file paths, complete code examples, and verification steps.
**Key Files/Locations:**
- `skills/writing-plans/SKILL.md`
- `docs/write-plan.md`
**Priority:** CORE

### 3. Test-Driven Development (TDD)
**Description:** Enforces RED-GREEN-REFACTOR cycle during implementation. Writes failing test first, watches it fail, writes minimal code to pass, then refactors. Includes testing anti-patterns reference.
**Key Files/Locations:**
- `skills/test-driven-development/SKILL.md`
- `skills/test-driven-development/code-quality-reviewer-prompt.md`
- `skills/test-driven-development/implementer-prompt.md`
- `skills/test-driven-development/spec-reviewer-prompt.md`
- `skills/test-driven-development/testing-anti-patterns.md`
**Priority:** CORE

### 4. Subagent-Driven Development
**Description:** Fast iteration workflow that dispatches fresh subagent per task with two-stage review (spec compliance, then code quality). Enables autonomous working sessions for hours.
**Key Files/Locations:**
- `skills/subagent-driven-development/SKILL.md`
- `skills/subagent-driven-development/plan-document-reviewer-prompt.md`
**Priority:** CORE

### 5. Executing Plans
**Description:** Batch execution of plan tasks with human checkpoints at appropriate intervals. Ensures plans are followed precisely with verification at each step.
**Key Files/Locations:**
- `skills/executing-plans/`
- `docs/execute-plan.md`
**Priority:** CORE

### 6. Using Git Worktrees
**Description:** Creates isolated workspace on new branch for parallel development. Runs project setup and verifies clean test baseline before starting work.
**Key Files/Locations:**
- `skills/using-git-worktrees/SKILL.md`
**Priority:** CORE

### 7. Requesting Code Review
**Description:** Pre-review checklist workflow. Reviews implementation against plan and reports issues by severity. Critical issues block progress.
**Key Files/Locations:**
- `skills/requesting-code-review/SKILL.md`
- `skills/requesting-code-review/code-reviewer.md`
- `skills/code-reviewer.md`
**Priority:** CORE

---

## Secondary Features (Priority Tier 2)

### 8. Systematic Debugging
**Description:** 4-phase root cause analysis process including root-cause-tracing, defense-in-depth, and condition-based-waiting techniques.
**Key Files/Locations:**
- `skills/systematic-debugging/SKILL.md`
- `skills/systematic-debugging/root-cause-tracing.md`
- `skills/systematic-debugging/defense-in-depth.md`
- `skills/systematic-debugging/condition-based-waiting.md`
**Priority:** SECONDARY

### 9. Verification Before Completion
**Description:** Ensures bugs are actually fixed before declaring completion. Validates the fix works in multiple scenarios.
**Key Files/Locations:**
- `skills/verification-before-completion/`
**Priority:** SECONDARY

### 10. Dispatching Parallel Agents
**Description:** Concurrent subagent workflow for parallelizing development tasks across multiple agents.
**Key Files/Locations:**
- `skills/dispatching-parallel-agents/SKILL.md`
**Priority:** SECONDARY

### 11. Finishing a Development Branch
**Description:** Merge/PR decision workflow when tasks complete. Verifies tests and presents options for branch management.
**Key Files/Locations:**
- `skills/finishing-a-development-branch/SKILL.md`
**Priority:** SECONDARY

### 12. Writing Skills
**Description:** Guide for creating new skills following best practices, including testing methodology for skill validation.
**Key Files/Locations:**
- `skills/writing-skills/SKILL.md`
- `skills/writing-skills/CREATION-LOG.md`
- `skills/writing-skills/testing-skills-with-subagents.md`
- `skills/writing-skills/anthropic-best-practices.md`
**Priority:** SECONDARY

### 13. Receiving Code Review
**Description:** Framework for responding to code review feedback constructively and efficiently.
**Key Files/Locations:**
- `skills/receiving-code-review/SKILL.md`
**Priority:** SECONDARY

### 14. Using Superpowers
**Description:** Introduction skill explaining the overall skills system and how skills trigger automatically.
**Key Files/Locations:**
- `skills/using-superpowers/SKILL.md`
**Priority:** SECONDARY

---

## Feature Summary Table

| Feature | Tier | Main File |
|---------|------|-----------|
| Brainstorming | CORE | `skills/brainstorming/SKILL.md` |
| Writing Plans | CORE | `skills/writing-plans/SKILL.md` |
| Test-Driven Development | CORE | `skills/test-driven-development/SKILL.md` |
| Subagent-Driven Development | CORE | `skills/subagent-driven-development/SKILL.md` |
| Executing Plans | CORE | `skills/executing-plans/` |
| Using Git Worktrees | CORE | `skills/using-git-worktrees/SKILL.md` |
| Requesting Code Review | CORE | `skills/requesting-code-review/SKILL.md` |
| Systematic Debugging | SECONDARY | `skills/systematic-debugging/SKILL.md` |
| Verification Before Completion | SECONDARY | `skills/verification-before-completion/` |
| Dispatching Parallel Agents | SECONDARY | `skills/dispatching-parallel-agents/SKILL.md` |
| Finishing a Development Branch | SECONDARY | `skills/finishing-a-development-branch/SKILL.md` |
| Writing Skills | SECONDARY | `skills/writing-skills/SKILL.md` |
| Receiving Code Review | SECONDARY | `skills/receiving-code-review/SKILL.md` |
| Using Superpowers | SECONDARY | `skills/using-superpowers/SKILL.md` |

---

## Directory Structure Reference

```
superpowers/
├── skills/                    # Core skills library
│   ├── brainstorming/         # Pre-code design refinement
│   ├── writing-plans/         # Implementation planning
│   ├── test-driven-development/  # TDD enforcement
│   ├── subagent-driven-development/  # Parallel execution
│   ├── executing-plans/       # Plan batch execution
│   ├── using-git-worktrees/   # Branch isolation
│   ├── requesting-code-review/  # Review workflow
│   ├── systematic-debugging/  # Debug methodology
│   ├── verification-before-completion/  # Fix validation
│   ├── dispatching-parallel-agents/  # Concurrent agents
│   ├── finishing-a-development-branch/  # Merge workflow
│   ├── writing-skills/       # Skill authoring guide
│   ├── receiving-code-review/ # Review response
│   ├── using-superpowers/     # System introduction
│   └── code-reviewer.md       # Shared review prompt
├── docs/                      # Documentation
│   ├── plans/                 # Plan templates
│   ├── superpowers/           # Platform-specific docs
│   └── testing.md            # Testing methodology
├── agents/                    # Agent configurations
├── commands/                 # Command hooks
└── hooks/                    # Session hooks
```
