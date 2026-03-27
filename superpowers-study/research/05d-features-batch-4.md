OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/05d-features-batch-4.md

# Feature Batch 4 Analysis: Dispatching Parallel Agents, Finishing a Development Branch, Writing Skills

## Feature 10: Dispatching Parallel Agents

**File:** `skills/dispatching-parallel-agents/SKILL.md`

### Purpose

A coordination skill for dispatching multiple independent subagents to work in parallel on separate problem domains. Preserves the orchestrator's context for coordination while delegating focused investigation work.

### Core Pattern

**When to use:**
- 3+ test files failing with different root causes
- Multiple subsystems broken independently
- Each problem can be understood without context from others
- No shared state between investigations

**When NOT to use:**
- Failures are related (fix one might fix others)
- Need to understand full system state
- Agents would interfere with each other
- Exploratory debugging (don't know what's broken yet)

### The Four-Step Pattern

```
1. Identify Independent Domains
   Group failures by what's broken (e.g., tool approval vs batch completion)

2. Create Focused Agent Tasks
   Each agent gets: specific scope, clear goal, constraints, expected output

3. Dispatch in Parallel
   Use Task() to dispatch multiple agents concurrently

4. Review and Integrate
   Read summaries, check for conflicts, run full test suite
```

### Agent Prompt Structure

Good prompts are:
- **Focused** - One clear problem domain
- **Self-contained** - All context needed to understand the problem
- **Specific about output** - What should the agent return?

```markdown
Fix the 3 failing tests in src/agents/agent-tool-abort.test.ts:

1. "should abort tool with partial output capture" - expects 'interrupted at' in message
2. "should handle mixed completed and aborted tools" - fast tool aborted instead of completed
3. "should properly track pendingToolCount" - expects 3 results but gets 0

These are timing/race condition issues. Your task:

1. Read the test file and understand what each test verifies
2. Identify root cause - timing issues or actual bugs?
3. Fix by:
   - Replacing arbitrary timeouts with event-based waiting
   - Fixing bugs in abort implementation if found
   - Adjusting test expectations if testing changed behavior

Do NOT just increase timeouts - find the real issue.

Return: Summary of what you found and what you fixed.
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| "Fix all the tests" | Agent gets lost | "Fix agent-tool-abort.test.ts" - focused scope |
| "Fix the race condition" | Agent doesn't know where | Paste error messages and test names |
| No constraints | Agent might refactor everything | "Do NOT change production code" or "Fix tests only" |
| "Fix it" | Unclear what changed | "Return summary of root cause and changes" |

### Real Example from Session

**Scenario:** 6 test failures across 3 files after major refactoring

```
Failures:
- agent-tool-abort.test.ts: 3 failures (timing issues)
- batch-completion-behavior.test.ts: 2 failures (tools not executing)
- tool-approval-race-conditions.test.ts: 1 failure (execution count = 0)
```

**Decision:** Independent domains - abort logic separate from batch completion separate from race conditions

**Dispatch:**
```
Agent 1 → Fix agent-tool-abort.test.ts
Agent 2 → Fix batch-completion-behavior.test.ts
Agent 3 → Fix tool-approval-race-conditions.test.ts
```

**Results:**
- Agent 1: Replaced timeouts with event-based waiting
- Agent 2: Fixed event structure bug (threadId in wrong place)
- Agent 3: Added wait for async tool execution to complete

**Integration:** All fixes independent, no conflicts, full suite green

### Key Design Decisions

1. **Token efficiency through isolation** - Agents should never inherit session context; orchestrator constructs exactly what each needs
2. **Verification after return** - Must run full test suite to catch conflicts
3. **Spot checking** - Agents can make systematic errors; orchestrator must verify

### Technical Notes

- The skill uses `Task()` function syntax from Claude Code for parallel dispatch
- Includes inline Graphviz flowchart for decision logic
- The skill itself is self-contained - no additional supporting files needed
- No external dependencies or scripts

---

## Feature 11: Finishing a Development Branch

**File:** `skills/finishing-a-development-branch/SKILL.md`

### Purpose

Guides the final integration step after development work is complete. Presents structured options for branch cleanup and ensures tests pass before any integration action.

### The Five-Step Process

```
Step 1: Verify Tests (BLOCKING)
    ↓
Step 2: Determine Base Branch
    ↓
Step 3: Present Options (exactly 4)
    ↓
Step 4: Execute Choice
    ↓
Step 5: Cleanup Worktree
```

### Step 1: Verify Tests

```bash
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

**Critical:** This step BLOCKS progression. The skill will not present options if tests fail.

### Step 2: Determine Base Branch

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or asks: "This branch split from main - is that correct?"

### Step 3: Present Options

Present exactly 4 options - no more, no less:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Design decision:** No explanation added to options - keeps them concise and unambiguous.

### Step 4: Execute Choice Matrix

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | Yes | No | No | Yes (local) |
| 2. Create PR | No | Yes | Yes | No |
| 3. Keep as-is | No | No | Yes | No |
| 4. Discard | No | No | No | Yes (force) |

#### Option 1: Merge Locally

```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
<test command>              # Verify tests on merged result
git branch -d <feature-branch>  # If tests pass
```

#### Option 2: Push and Create PR

```bash
git push -u origin <feature-branch>
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

#### Option 3: Keep As-Is

Reports: "Keeping branch <name>. Worktree preserved at <path>."

**Important:** Does NOT cleanup worktree.

#### Option 4: Discard

Requires typed confirmation:

```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

### Step 5: Worktree Cleanup Logic

**For Options 1, 2, 4:**

```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree (the work might be needed later).

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Skipping test verification | Merge broken code | Always verify tests first |
| Open-ended questions | "What should I do next?" ambiguous | Present exactly 4 structured options |
| Auto worktree cleanup | Remove when might need it (Option 2, 3) | Only cleanup for Options 1 and 4 |
| No confirmation for discard | Accidentally delete work | Require typed "discard" |

### Red Flags (NEVER)

- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request

### Integration Points

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill

### Technical Notes

- Uses `gh pr create` for GitHub PR creation (assumes GitHub CLI available)
- Uses `git worktree` commands for branch isolation cleanup
- The skill is self-contained with no supporting files
- All git operations use `2>/dev/null` to silently fall back when commands fail

---

## Feature 12: Writing Skills

**Files:**
- `skills/writing-skills/SKILL.md` (22,624 bytes)
- `skills/writing-skills/anthropic-best-practices.md` (45,820 bytes)
- `skills/writing-skills/testing-skills-with-subagents.md` (12,558 bytes)
- `skills/writing-skills/persuasion-principles.md` (5,908 bytes)
- `skills/writing-skills/graphviz-conventions.dot` (5,857 bytes)
- `skills/writing-skills/render-graphs.js` (4,857 bytes)
- `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` (5,423 bytes)

### Purpose

Guide for creating new skills following TDD methodology. Ensures skills are tested before deployment and resistant to rationalization.

### Core Principle

**"Writing skills IS Test-Driven Development applied to process documentation."**

### The TDD Mapping

| TDD Concept | Skill Creation |
|-------------|----------------|
| **Test case** | Pressure scenario with subagent |
| **Production code** | Skill document (SKILL.md) |
| **Test fails (RED)** | Agent violates rule without skill (baseline) |
| **Test passes (GREEN)** | Agent complies with skill present |
| **Refactor** | Close loopholes while maintaining compliance |

### The Iron Law

```
NO SKILL WITHOUT A FAILING TEST FIRST
```

This applies to NEW skills AND EDITS to existing skills. No exceptions.

### When to Create a Skill

**Create when:**
- Technique wasn't intuitively obvious to you
- You'd reference this again across projects
- Pattern applies broadly (not project-specific)
- Others would benefit

**Don't create for:**
- One-off solutions
- Standard practices well-documented elsewhere
- Project-specific conventions (put in CLAUDE.md)
- Mechanical constraints (if enforceable with regex/validation, automate it)

### Skill Types

| Type | What it is | Examples |
|------|------------|----------|
| **Technique** | Concrete method with steps | condition-based-waiting, root-cause-tracing |
| **Pattern** | Way of thinking about problems | flatten-with-flags, test-invariants |
| **Reference** | API docs, syntax guides | office docs |

### Directory Structure

```
skills/
  skill-name/
    SKILL.md              # Main reference (required)
    supporting-file.*     # Only if needed
```

**Flat namespace** - all skills in one searchable namespace

**Separate files for:**
1. **Heavy reference** (100+ lines) - API docs, comprehensive syntax
2. **Reusable tools** - Scripts, utilities, templates

**Keep inline:**
- Principles and concepts
- Code patterns (< 50 lines)
- Everything else

### SKILL.md Format

**Frontmatter (YAML):**
```yaml
---
name: Skill-Name-With-Hyphens
description: Use when [specific triggering conditions and symptoms]
---
```

**Frontmatter rules:**
- Two required fields: `name` and `description`
- Max 1024 characters total
- `name`: Use letters, numbers, and hyphens only (no parentheses, special chars)
- `description`: Third-person, describes ONLY when to use (NOT what it does)

### The Claude Search Optimization (CSO) Trap

**Critical insight:** The description should ONLY describe triggering conditions, NOT summarize the skill's workflow.

**Why this matters:** Testing revealed that when a description summarizes the skill's workflow, Claude may follow the description instead of reading the full skill content.

```yaml
# BAD: Summarizes workflow - Claude may follow this instead of reading skill
description: Use when executing plans - dispatches subagent per task with code review between tasks

# GOOD: Just triggering conditions, no workflow summary
description: Use when executing implementation plans with independent tasks in the current session
```

### SKILL.md Structure Template

```markdown
## Overview
What is this? Core principle in 1-2 sentences.

## When to Use
[Small inline flowchart IF decision non-obvious]
Bullet list with SYMPTOMS and use cases
When NOT to use

## Core Pattern (for techniques/patterns)
Before/after code comparison

## Quick Reference
Table or bullets for scanning common operations

## Implementation
Inline code for simple patterns
Link to file for heavy reference or reusable tools

## Common Mistakes
What goes wrong + fixes

## Real-World Impact (optional)
Concrete results
```

### RED-GREEN-REFACTOR Cycle for Skills

**RED Phase - Write Failing Test:**
1. Create pressure scenarios (3+ combined pressures for discipline skills)
2. Run scenarios WITHOUT skill - document baseline behavior verbatim
3. Identify patterns in rationalizations/failures

**GREEN Phase - Write Minimal Skill:**
1. Name uses only letters, numbers, hyphens
2. YAML frontmatter with required fields
3. Description starts with "Use when..." and includes specific triggers
4. Clear overview with core principle
5. Address specific baseline failures identified in RED
6. Run scenarios WITH skill - verify agents now comply

**REFACTOR Phase - Close Loopholes:**
1. Identify NEW rationalizations from testing
2. Add explicit counters (if discipline skill)
3. Build rationalization table from all test iterations
4. Create red flags list
5. Re-test until bulletproof

### Testing All Skill Types

**Discipline-Enforcing Skills (rules/requirements)**
- Examples: TDD, verification-before-completion
- Test with: Academic questions, pressure scenarios, rationalization identification
- Success: Agent follows rule under maximum pressure

**Technique Skills (how-to guides)**
- Examples: condition-based-waiting, root-cause-tracing
- Test with: Application scenarios, variation scenarios, missing information tests
- Success: Agent successfully applies technique to new scenario

**Pattern Skills (mental models)**
- Examples: reducing-complexity, information-hiding
- Test with: Recognition scenarios, application scenarios, counter-examples
- Success: Agent correctly identifies when/how to apply pattern

**Reference Skills (documentation/APIs)**
- Examples: API documentation, command references
- Test with: Retrieval scenarios, application scenarios, gap testing
- Success: Agent finds and correctly applies reference information

### Bulletproofing Against Rationalization

**Close Every Loophole Explicitly:**
```markdown
Write code before test? Delete it. Start over.

**No exceptions:**
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete
```

**Address "Spirit vs Letter" Arguments:**
```markdown
**Violating the letter of the rules is violating the spirit of the rules.**
```

**Build Rationalization Table:**
```markdown
| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
```

### Stop Checklist (Mandatory)

**After writing ANY skill, you MUST STOP and complete the deployment process.**

Do NOT:
- Create multiple skills in batch without testing each
- Move to next skill before current one is verified
- Skip testing because "batching is more efficient"

**Deployment checklist:**
- [ ] Commit skill to git and push to your fork (if configured)
- [ ] Consider contributing back via PR (if broadly useful)

### Key Design Decisions

1. **TDD for documentation** - The same discipline applied to code applies to documentation
2. **Skill types determine testing approach** - Different skills need different verification strategies
3. **Rationalization resistance** - Discipline-enforcing skills must be bulletproofed against smart agents finding loopholes
4. **Token efficiency** - Skills should be concise; move details to supporting files or cross-references
5. **Flat namespace** - All skills in one directory for easy search

### Supporting Files

| File | Purpose |
|------|---------|
| `anthropic-best-practices.md` | Official Anthropic guidance on skill authoring |
| `testing-skills-with-subagents.md` | Complete testing methodology with pressure scenarios |
| `persuasion-principles.md` | Psychology foundation for rationalization resistance (Cialdini, Meincke) |
| `graphviz-conventions.dot` | Flowchart style rules |
| `render-graphs.js` | Tool to render skill flowcharts to SVG |
| `examples/CLAUDE_MD_TESTING.md` | Example of testing methodology |

---

## Cross-Feature Observations

### Design Patterns Shared Across Features

1. **Blocking verification steps** - Both Dispatching Parallel Agents (verify fixes work together) and Finishing a Development Branch (verify tests pass) enforce verification before proceeding

2. **Structured options over open-ended** - Finishing a Development Branch presents exactly 4 options; Dispatching Parallel Agents specifies exact prompt structure

3. **Integration awareness** - Skills explicitly note where they're called from:
   - Finishing a Development Branch: Called by subagent-driven-development, executing-plans
   - Writing Skills: Requires TDD background

4. **Anti-patterns documented** - All three skills explicitly document common mistakes

### Skill Hierarchy

```
writing-skills (meta-skill)
    ↓ documents how to create
dispatching-parallel-agents (technique)
finishing-a-development-branch (workflow)
```

Writing Skills is a "meta-skill" that teaches how to create other skills, following the same TDD principles it advocates.

### Technical Debt and Shortcuts

1. **No platform-specific code** - All three skills are pure markdown documentation
2. **Assumes standard tools** - git, gh (GitHub CLI), npm test - no custom scripts needed
3. **Token efficiency focus** - Heavy reference material pushed to separate files
4. **Cross-references without force-loading** - Uses skill names instead of @ file paths

### Edge Cases Handled

1. **Finishing a Development Branch:**
   - Tests might not be npm/cargo/pytest/go - flexible command syntax
   - Base branch might be main or master - tries both
   - gh might not be installed - no fallback documented (potential gap)

2. **Dispatching Parallel Agents:**
   - Agents might conflict - verification step catches this
   - Too broad scope - explicit "don't do this" examples

3. **Writing Skills:**
   - Rationalization under pressure - explicit counters for common excuses
   - Skill too long - word count targets and token efficiency techniques
