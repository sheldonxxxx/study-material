OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/05b-features-batch-2.md

# Feature Batch 2 Analysis: Subagent-Driven Development, Executing Plans, Using Git Worktrees

## Feature 4: Subagent-Driven Development

### Core Concept

Subagent-Driven Development (SDD) is a same-session parallel execution workflow that dispatches fresh subagents for each implementation task, followed by a two-stage review process (spec compliance first, then code quality). The key innovation is that the controller agent constructs exactly the context each subagent needs rather than inheriting session history.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `skills/subagent-driven-development/SKILL.md` | Main skill definition with process flow |
| `skills/subagent-driven-development/implementer-prompt.md` | Template for dispatching implementer subagents |
| `skills/subagent-driven-development/spec-reviewer-prompt.md` | Template for spec compliance verification |
| `skills/subagent-driven-development/code-quality-reviewer-prompt.md` | Template for code quality review |

### Process Flow

```
Controller reads plan once, extracts all tasks with full text
    |
    v
For each task:
    Dispatch implementer subagent (fresh context)
        |
        v
    Implementer asks questions? --> YES --> Controller answers --> Re-dispatch
        |
        NO
        v
    Implementer implements, tests, commits, self-reviews
        |
        v
    Dispatch spec reviewer subagent
        |
        v
    Spec compliant? --> NO --> Implementer fixes --> Re-review
        |
        YES
        v
    Dispatch code quality reviewer
        |
        v
    Code quality approved? --> NO --> Implementer fixes --> Re-review
        |
        YES
        v
    Mark task complete
    |
    v
After all tasks: Final code review --> finishing-a-development-branch
```

### Key Design Decisions

**1. Fresh Context Per Task**
The implementer subagent receives complete context upfront (task text, scene-setting, directory) rather than reading files. This is explicitly documented as a red flag to avoid: "Don't make subagent read plan file (provide full text instead)."

**2. Two-Stage Review Order (Spec BEFORE Quality)**
This is architecturally enforced. The skill explicitly states: "Start code quality review before spec compliance is [WRONG ORDER]". The reasoning is:
- Spec compliance checks if the RIGHT thing was built
- Code quality checks if the thing was built RIGHT
- You must confirm direction before checking execution

**3. Model Tiering for Cost Efficiency**
```
Mechanical implementation (1-2 files, clear spec) --> Fast/cheap model
Integration/judgment (multi-file, pattern matching) --> Standard model
Architecture/design/review --> Most capable model
```

**4. Explicit Status Reporting**
Implementers report one of four statuses:
- `DONE` - Proceed to spec review
- `DONE_WITH_CONCERNS` - Completed but flagged doubts
- `NEEDS_CONTEXT` - Missing information needed
- `BLOCKED` - Cannot complete task

The controller must handle each appropriately - never ignoring escalations.

### Spec Reviewer Anti-Pattern Detection

The spec-reviewer-prompt.md contains a critical instruction under "CRITICAL: Do Not Trust the Report":

> "The implementer finished suspiciously quickly. Their report may be incomplete, inaccurate, or optimistic. You MUST verify everything independently."

This explicitly instructs reviewers to:
- NOT trust implementer claims about completeness
- Read actual code and compare line-by-line
- Look for missing pieces AND extra unrequested work
- Verify by reading code, not trusting reports

### Quality Gates

**Self-Review Checklist (before implementer reports back):**
- Completeness: Did I implement everything? Edge cases handled?
- Quality: Is this my best work? Names clear?
- Discipline: Did I avoid overbuilding (YAGNI)?
- Testing: Do tests verify actual behavior?

**Spec Reviewer Checks:**
- Missing requirements
- Extra/unneeded work
- Misunderstandings of requirements

**Code Quality Reviewer Additional Checks:**
- Does each file have one clear responsibility?
- Are units decomposed for independent testing?
- Did implementation follow the plan's file structure?
- Did this change create new large files or significantly grow existing ones?

### Clever Solutions

**1. Question-Before-Implementation Protocol**
Implementers are explicitly told: "If you have questions about the requirements or approach, ask them now. Raise any concerns before starting work." This surfaces ambiguity early rather than after implementation.

**2. Escalation Safety Valve**
"If you feel uncertain about whether your approach is correct, stop and escalate." The implementer prompt explicitly states: "Bad work is worse than no work."

**3. File Size Concerns**
The code quality reviewer specifically checks if the implementation "created new files that are already large, or significantly grow existing files" - this prevents scope creep through incremental file bloat.

**4. Red Flag System**
The skill extensively documents "Never" statements that represent known failure modes:
- Never start on main/master without explicit consent
- Never skip reviews
- Never dispatch multiple implementation subagents in parallel
- Never accept "close enough" on spec compliance

### Edge Cases and Error Handling

**When subagent asks questions:**
- Controller answers clearly and completely
- Additional context provided if needed
- Subagent doesn't rush into implementation

**When reviewer finds issues:**
- Same implementer subagent fixes (not controller)
- Reviewer re-reviews after fixes
- Loop continues until approved

**When subagent fails task:**
- Dispatch fix subagent with specific instructions
- Controller doesn't try to fix manually (would cause context pollution)

### Integration Points

**Required workflow skills:**
- `superpowers:using-git-worktrees` - Must set up isolated workspace before starting
- `superpowers:writing-plans` - Creates the plan SDD executes
- `superpowers:requesting-code-review` - Code review template
- `superpowers:finishing-a-development-branch` - Completion workflow

**Subagents use:**
- `superpowers:test-driven-development` - Subagents follow TDD for each task

### Test Validation

Two test projects exist:
- `tests/subagent-driven-dev/svelte-todo/` - 12-task Svelte todo app
- `tests/subagent-driven-dev/go-fractals/` - 10-task Go CLI fractal generator

Both are real projects designed to validate the SDD workflow end-to-end.

---

## Feature 5: Executing Plans

### Core Concept

Executing Plans is a manual (human-in-the-loop) execution workflow for when you need to execute an implementation plan in a separate session with review checkpoints. It is the alternative to Subagent-Driven Development when parallel session execution is needed.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `skills/executing-plans/SKILL.md` | Main skill definition |
| `commands/execute-plan.md` | Deprecated command entry point |

Note: The `commands/execute-plan.md` is deprecated - it simply tells the user to use the skill instead.

### Key Differences from SDD

| Aspect | Executing Plans | Subagent-Driven Development |
|--------|-----------------|---------------------------|
| Session | Separate session | Same session |
| Agent | Single agent executing | Multiple subagents dispatched |
| Review | Human checkpoints | Subagent two-stage review |
| Speed | Slower (human wait times) | Faster (continuous progress) |
| Quality | Human-verified | Subagent review loops |

The SKILL.md explicitly notes: "Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support."

### Process Flow

```
Step 1: Load and Review Plan
    |
    v
    Read plan file
    Review critically (identify questions/concerns)
        |
        v
    Concerns found? --> YES --> Raise with human before starting
        |
        NO
        v
    Create TodoWrite, proceed
    |
    v
Step 2: Execute Tasks (for each task)
    |
    v
    Mark task in_progress
    Follow each step exactly (plan has bite-sized steps)
    Run verifications as specified
    Mark task completed
    |
    v
Step 3: Complete Development
    |
    v
    Announce: "finishing-a-development-branch skill"
    Use superpowers:finishing-a-development-branch (REQUIRED)
    Verify tests, present options, execute choice
```

### Critical Instructions

**Stop and Ask for Help When:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- Don't understand an instruction
- Verification fails repeatedly

**Return to Review When:**
- Partner updates plan based on your feedback
- Fundamental approach needs rethinking

**Remember:**
- Never start implementation on main/master without explicit user consent
- Don't force through blockers - stop and ask
- Follow plan steps exactly
- Don't skip verifications

### Integration Requirements

The skill mandates:
- `superpowers:using-git-worktrees` - REQUIRED: Set up isolated workspace before starting
- `superpowers:writing-plans` - Creates the plan this skill executes
- `superpowers:finishing-a-development-branch` - Complete development after all tasks

### Design Observations

**1. Simpler Than SDD**
This is a single-agent workflow without subagent dispatch complexity. The agent executes tasks sequentially with human verification at appropriate intervals.

**2. Plan-Centric Execution**
The plan is the source of truth. The agent must "follow each step exactly" and "run verifications as specified."

**3. Built-in Critical Review**
Before execution begins, the agent is expected to "review critically - identify any questions or concerns" and raise them with the human partner before starting.

**4. Human-in-the-Loop**
Unlike SDD's automated review loops, this skill uses human checkpoints between tasks or task batches.

---

## Feature 6: Using Git Worktrees

### Core Concept

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching. This skill systematizes worktree creation with smart directory selection and safety verification.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `skills/using-git-worktrees/SKILL.md` | Main skill definition (comprehensive) |

### Directory Selection Priority

The skill defines a clear priority order for worktree location:

```
1. Check for existing directories (in priority order):
   - .worktrees/ (preferred, hidden)
   - worktrees/ (alternative)

2. Check CLAUDE.md for preference:
   grep -i "worktree.*director" CLAUDE.md

3. Ask user if neither exists:
   - .worktrees/ (project-local, hidden)
   - ~/.config/superpowers/worktrees/<project-name>/ (global location)
```

### Safety Verification (Critical)

**For project-local directories:**

```bash
# MUST verify directory is ignored BEFORE creating worktree
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

If NOT ignored: Add to .gitignore, commit, then proceed.

**Why this matters:** Prevents accidentally committing worktree contents to the repository. The skill explicitly calls this out under "Jesse's rule: Fix broken things immediately."

**For global directories (~/.config/superpowers/worktrees):**
No .gitignore verification needed - outside project entirely.

### Creation Process

```bash
# 1. Detect project name
project=$(basename "$(git rev-parse --show-toplevel)")

# 2. Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"

# 3. Auto-detect and run project setup
# Node.js: npm install
# Rust: cargo build
# Python: pip install or poetry install
# Go: go mod download
```

### Verification Steps

After setup, MUST verify clean baseline:

```bash
# Run project-appropriate tests
npm test  # or
cargo test  # or
pytest  # or
go test ./...
```

If tests fail: Report failures, ask whether to proceed or investigate.
If tests pass: Report ready.

### Final Report Format

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

### Common Mistakes (Documented)

| Mistake | Problem | Fix |
|---------|---------|-----|
| Skipping ignore verification | Worktree contents get tracked | Always use `git check-ignore` |
| Assuming directory location | Inconsistency, convention violation | Follow priority: existing > CLAUDE.md > ask |
| Proceeding with failing tests | Can't distinguish new bugs from pre-existing | Report + get permission |
| Hardcoding setup commands | Breaks on different project types | Auto-detect from package.json, Cargo.toml, etc. |

### Red Flags (Never)

The skill explicitly prohibits:
- Creating worktree without verifying it's ignored (project-local)
- Skipping baseline test verification
- Proceeding with failing tests without asking
- Assuming directory location when ambiguous
- Skipping CLAUDE.md check

### Integration Points

**Called by:**
- `brainstorming` (Phase 4) - REQUIRED when design is approved
- `subagent-driven-development` - REQUIRED before executing tasks
- `executing-plans` - REQUIRED before executing tasks
- Any skill needing isolated workspace

**Pairs with:**
- `finishing-a-development-branch` - REQUIRED for cleanup after work complete

### Key Design Decisions

**1. Convention Over Configuration**
Priority order (existing > CLAUDE.md > ask) means the skill adapts to project conventions rather than imposing them.

**2. Safety First**
The mandatory .gitignore verification before creating project-local worktrees prevents a class of mistakes that would pollute the repository.

**3. Auto-Detection for Setup**
Rather than requiring the user to specify build commands, the skill detects project type from file presence and runs appropriate setup.

**4. Baseline Verification**
Running tests before starting work establishes a known-good state. If tests fail later, it's clear those failures are from the new work, not pre-existing issues.

**5. Clean Reporting**
The final report format gives the human a clear picture: where the worktree is, test status, and what's ready to implement.

### Example Workflow

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

---

## Cross-Feature Integration

These three features form the execution layer of the Superpowers workflow:

```
writing-plans (creates plan)
        |
        v
using-git-worktrees (isolates workspace)
        |
        v
subagent-driven-development OR executing-plans (executes plan)
        |
        v
finishing-a-development-branch (completes work)
```

### Worktree + Execution Integration

Both execution skills (SDD and Executing Plans) require the using-git-worktrees skill before executing any tasks. This ensures:
1. Work happens on a feature branch, not main
2. Current workspace remains clean for other work
3. Tests run on fresh setup to establish baseline

### Decision Tree for Execution Choice

```
Have implementation plan?
    |
    v
Tasks mostly independent? --> NO --> Manual execution or brainstorm first
    |
    YES
    |
    v
Stay in this session? --> NO --> executing-plans (parallel session)
    |
    YES
    |
    v
subagent-driven-development (same session, subagents)
```

---

## Technical Observations

### Pattern: Fresh Context Over Inheritance

Both SDD and worktrees share a philosophy of fresh/isolated context:
- SDD: Fresh subagent per task, constructed context not inherited
- Worktrees: Fresh workspace on new branch, shared repository

### Pattern: Safety Before Speed

The worktree skill and SDD both have explicit safety checks:
- Worktrees: .gitignore verification before creation
- SDD: Spec compliance review before code quality review
- Both: Stop-and-ask protocols before forcing through

### Pattern: Explicit Over Implicit

- Worktrees: Explicit priority order for directory selection
- SDD: Explicit status codes (DONE, DONE_WITH_CONCERNS, BLOCKED, NEEDS_CONTEXT)
- Executing Plans: Explicit "never start on main/master" warnings

### Code Organization

These skills are pure documentation/guidance - no executable code. They define workflows that the AI agent follows. The "implementation" is the agent's execution of the workflow.
