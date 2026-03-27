OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/learning/04-features-deep-dive.md

# Superpowers Features Deep-Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`
**Source:** Analysis of 5 feature batch documents covering all 14 features
**Date:** 2026-03-27

---

## Executive Summary

Superpowers is a composable skills system for AI coding agents that enforces development discipline through explicit process guidance. The system addresses a fundamental problem: AI agents tend to take shortcuts, skip verification, and rationalize away rigorous processes when under pressure.

The architecture follows a **vertical integration pattern** where each skill is self-contained, composable, and designed to prevent specific failure modes. Rather than a linear workflow, Superpowers provides orthogonal skills that layer on top of each other: brainstorming (design), writing-plans (planning), executing-plans/subagent-driven-development (execution), and various quality assurance skills (verification, debugging, review).

**Key architectural insight:** Skills are not procedures to follow - they are enforcement mechanisms against rationalization. The Iron Law pattern (explicit prohibitive statements with no exceptions) appears throughout as the primary anti-rationalization technique.

---

## Core Features (Tier 1)

### Feature 1: Brainstorming

**File:** `skills/brainstorming/SKILL.md`

The brainstorming skill implements a Socratic design refinement process with a critical HARD-GATE that prevents any implementation before design approval. The 9-step checklist structures the entire pre-implementation phase.

**Key Implementation Details:**

The HARD-GATE mechanism uses explicit `<HARD-GATE>` markup to prevent bypassing:
```
<HWARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the
user has approved it.
</HARD-GATE>
```

**Visual Companion Subsystem:** A zero-dependency WebSocket server (`scripts/server.cjs`) provides real-time visual feedback during design discussions. Notable technical choices:
- Manual RFC 6455 WebSocket implementation with crypto-based handshake
- Dynamic port selection in ephemeral range (49152-65535)
- File watching with macOS-specific debouncing (fs.watch reports 'rename' for both new files and overwrites)
- Owner process monitoring for WSL/Tailscale/cross-user scenarios
- 30-minute idle timeout with 60-second lifecycle checks

**Platform-Specific Launch Notes:** Different AI platforms require different foreground/background execution strategies:
- Claude Code (Windows): Must use `run_in_background: true`
- Codex: Auto-detects `CODEX_CI` env var for foreground mode
- Gemini CLI: Requires `--foreground` flag

**Technical Debt:**
- No cleanup mechanism for orphaned sessions in `/tmp`
- No authentication on WebSocket (mitigated by localhost-only binding)
- `commands/brainstorm.md` deprecated in favor of skill-based invocation

---

### Feature 2: Writing Plans

**File:** `skills/writing-plans/SKILL.md`

Writing Plans transforms approved designs into detailed implementation plans with 2-5 minute granularity. The extreme task granularity reflects optimization for AI execution where attention spans and context windows are bounded.

**Key Design Decisions:**

**No Placeholders Policy:** The skill explicitly enumerates prohibited plan failures:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Similar to Task N" (must repeat code instead)
- Steps that describe without showing code

The "Similar to Task N" prohibition is clever - executors may read tasks out of order or parallelize, so assumptions about sequential reading break execution.

**File Structure Before Tasks:** Plans must map out file decomposition before defining tasks, locking in architectural decisions early.

**TDD-Embedded Structure:** Each task follows RED-GREEN-REFACTOR:
1. Write failing test
2. Run test to verify it fails
3. Write minimal implementation
4. Run test to verify it passes
5. Commit

**Self-Review Checklist:**
- Spec coverage - can you point to a task for each requirement?
- Placeholder scan - search for red flags
- Type consistency - do names match across tasks?

**Execution Handoff:** Two paths after saving plan:
1. Subagent-Driven (recommended) - fresh subagent per task + two-stage review
2. Inline Execution - batch execution with checkpoints

**Technical Observations:**
- Assumes TDD methodology; plans that skip TDD need different template
- No human time estimates, only AI execution time (2-5 minutes per step)
- `commands/write-plan.md` deprecated in favor of skill

---

### Feature 3: Test-Driven Development (TDD)

**File:** `skills/test-driven-development/SKILL.md`
**Supporting File:** `skills/test-driven-development/testing-anti-patterns.md`

TDD enforces RED-GREEN-REFACTOR cycle with uncompromising Iron Law enforcement. The core principle:

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

**Enforcement is Absolute:**
- "Write code before the test? Delete it. Start over."
- "Don't keep it as 'reference'"
- "Don't 'adapt' it while writing tests"
- "Don't look at it"
- "Delete means delete"

The "Delete means delete" enforcement addresses a real failure mode: developers who write tests after but claim TDD benefits by "adapting" existing code.

**The Five Anti-Patterns:**

| Anti-Pattern | Description | Gate Question |
|-------------|-------------|----------------|
| Testing Mock Behavior | Asserting on mock elements instead of real behavior | "Am I testing real component behavior or just mock existence?" |
| Test-Only Methods | Adding `destroy()` methods only used in tests | "Is this only used by tests?" |
| Mocking Without Understanding | Mocking "to be safe" without understanding side effects | "Do I understand what this mock actually does?" |
| Incomplete Mocks | Partial mocks omitting fields downstream code uses | "Did I mock the COMPLETE data structure as it exists?" |
| Integration Tests as Afterthought | "Implementation complete, no tests written" | Fix: TDD cycle |

**Anti-Rationalization Table:**

| Excuse | Rebuttal |
|--------|----------|
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Already manually tested" | Ad-hoc, no record, can't re-run, not comprehensive |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "TDD will slow me down" | TDD faster than debugging. Pragmatic = test-first. |

**Technical Observations:**
- No tooling prescription - methodology-agnostic for multi-platform support
- Dogmatic tone is intentional to overcome human rationalization
- Integrates with writing-plans (TDD structure embedded in tasks)
- No reviewer subagent template - enforced manually during implementation

---

### Feature 4: Subagent-Driven Development

**File:** `skills/subagent-driven-development/SKILL.md`
**Subagent Templates:**
- `implementer-prompt.md`
- `spec-reviewer-prompt.md`
- `code-quality-reviewer-prompt.md`

SDD dispatches fresh subagents for each implementation task with two-stage review (spec compliance BEFORE code quality). The key innovation is **context construction over context inheritance** - the controller provides exactly what each subagent needs rather than relying on session history.

**Architecture Decisions:**

**Spec BEFORE Quality Review:** Architecturally enforced. The skill explicitly states: "Start code quality review before spec compliance is [WRONG ORDER]". Reasoning:
- Spec compliance checks if the RIGHT thing was built
- Code quality checks if the thing was built RIGHT
- Confirm direction before checking execution

**Fresh Context Per Task:** Implementer receives complete context upfront (task text, scene-setting, directory) rather than reading files. Documented red flag: "Don't make subagent read plan file (provide full text instead)."

**Model Tiering for Cost Efficiency:**
```
Mechanical implementation (1-2 files, clear spec) --> Fast/cheap model
Integration/judgment (multi-file, pattern matching) --> Standard model
Architecture/design/review --> Most capable model
```

**Spec Reviewer Anti-Pattern Detection:**
```
"The implementer finished suspiciously quickly. Their report may be incomplete,
 inaccurate, or optimistic. You MUST verify everything independently."
```

This explicitly instructs reviewers to read actual code and compare line-by-line, not trust implementer reports.

**Status Reporting:**
- `DONE` - Proceed to spec review
- `DONE_WITH_CONCERNS` - Completed but flagged doubts
- `NEEDS_CONTEXT` - Missing information needed
- `BLOCKED` - Cannot complete task

**Red Flag System ("Never" statements):**
- Never start on main/master without explicit consent
- Never skip reviews
- Never dispatch multiple implementation subagents in parallel
- Never accept "close enough" on spec compliance

---

### Feature 5: Executing Plans

**File:** `skills/executing-plans/SKILL.md`

Manual execution workflow for separate-session execution with human checkpoints. The alternative to SDD when parallel session execution is needed.

**Key Differences from SDD:**

| Aspect | Executing Plans | Subagent-Driven Development |
|--------|-----------------|---------------------------|
| Session | Separate session | Same session |
| Agent | Single agent executing | Multiple subagents dispatched |
| Review | Human checkpoints | Subagent two-stage review |
| Speed | Slower (human wait times) | Faster (continuous progress) |
| Quality | Human-verified | Subagent review loops |

**Critical Instructions:**

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

**Integration:** Both execution skills require `using-git-worktrees` before executing tasks.

---

### Feature 6: Using Git Worktrees

**File:** `skills/using-git-worktrees/SKILL.md`

Git worktrees create isolated workspaces sharing the repository, enabling work on multiple branches simultaneously without switching. The skill systematizes worktree creation with safety verification.

**Directory Selection Priority:**
1. Check for existing directories: `.worktrees/` (preferred) or `worktrees/`
2. Check CLAUDE.md for preference
3. Ask user if neither exists

**Safety Verification (Critical):**
```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```
If NOT ignored: Add to .gitignore, commit, then proceed. Prevents accidentally committing worktree contents to repository ("Jesse's rule: Fix broken things immediately").

**Auto-Detection for Setup:**
```
Node.js: npm install
Rust: cargo build
Python: pip install or poetry install
Go: go mod download
```

**Baseline Verification:** Runs tests before starting work to establish known-good state. If tests fail later, failures are from new work, not pre-existing.

**Final Report Format:**
```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

**Integration Points:**
- Called by: brainstorming (Phase 4), subagent-driven-development, executing-plans
- Pairs with: finishing-a-development-branch (cleanup after work complete)

---

### Feature 7: Requesting Code Review

**Files:**
- `skills/requesting-code-review/SKILL.md`
- `skills/requesting-code-review/code-reviewer.md`
- `skills/code-reviewer.md` (shared, in agents directory)

Pre-review checklist workflow. Reviews implementation against plan and reports issues by severity.

**Workflow:**
```
1. Get git SHAs:
   BASE_SHA=$(git rev-parse HEAD~1)
   HEAD_SHA=$(git rev-parse HEAD)

2. Dispatch code-reviewer subagent with filled template

3. Act on feedback:
   - Fix Critical issues immediately
   - Fix Important issues before proceeding
   - Note Minor issues for later
   - Push back if reviewer is wrong (with reasoning)
```

**Reviewer Prompt Template Parameters:**
```
WHAT_WAS_IMPLEMENTED: {DESCRIPTION}
PLAN_OR_REQUIREMENTS: {PLAN_REFERENCE}
BASE_SHA: {BASE_SHA}
HEAD_SHA: {HEAD_SHA}
```

**Issue Categorization:**

| Severity | Definition | Action |
|----------|------------|--------|
| **Critical** | Bugs, security issues, data loss risks, broken functionality | Must Fix |
| **Important** | Architecture problems, missing features, poor error handling, test gaps | Should Fix |
| **Minor** | Code style, optimization opportunities, documentation improvements | Nice to Have |

**Clever Solutions:**
1. **Subagent Context Isolation:** Reviewer gets "precisely crafted context for evaluation - never your session's history"
2. **Git Range Context:** Embeds `git diff --stat` and `git diff` commands for exact change inspection
3. **Severity Calibration:** Explicit warnings against marking nitpicks as Critical
4. **Push-Back Guidance:** Explicitly encourages pushing back if "reviewer is wrong (with reasoning)"

**Technical Observations:**
- `code-reviewer.md` in agents/ is identical to skills/requesting-code-review/code-reviewer.md
- Purely prompt-based - no executable code
- Integration requires orchestrator support for subagent dispatching

---

## Secondary Features (Tier 2)

### Feature 8: Systematic Debugging

**Files:**
- `skills/systematic-debugging/SKILL.md`
- `skills/systematic-debugging/root-cause-tracing.md`
- `skills/systematic-debugging/defense-in-depth.md`
- `skills/systematic-debugging/condition-based-waiting.md`
- `skills/systematic-debugging/condition-based-waiting-example.ts`
- `skills/systematic-debugging/find-polluter.sh`

4-phase root cause analysis process with Iron Law:

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

**The Four Phases:**

1. **Root Cause Investigation:** Read error messages carefully, reproduce consistently, check recent changes, add diagnostic instrumentation at boundaries

2. **Pattern Analysis:** Find working examples, compare against references (read COMPLETELY, not skim), identify differences, understand dependencies

3. **Hypothesis and Testing:** Form single hypothesis ("I think X is root cause because Y"), test minimally (one variable at a time), verify before continuing

4. **Implementation:** Create failing test case first, implement single fix, verify fix works, if 3+ fixes failed: question the architecture

**Clever Solutions:**

**Root Cause Tracing (root-cause-tracing.md):**
Uses backward tracing through call stack with `new Error().stack` for instrumentation:
```
Error: git init failed in /Users/jesse/project/packages/core
  → git init runs with empty cwd parameter
    → WorktreeManager called with empty projectDir
      → Session.create() passed empty string
        → Test accessed context.tempDir before beforeEach
          → setupCoreTest() returns { tempDir: '' } initially
```

**find-polluter.sh:** Bash bisection script (64 lines) for finding which test creates unwanted files/state:
```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

**Defense-in-Depth (defense-in-depth.md):** After finding root cause, add validation at EVERY layer data passes through:
| Layer | Purpose | Example |
|-------|---------|---------|
| Layer 1 | Entry point validation | `if (!dir) throw new Error()` |
| Layer 2 | Business logic validation | `if (!projectDir) throw new Error()` |
| Layer 3 | Environment guards | `if (NODE_ENV === 'test' && !inTmpDir()) refuse` |
| Layer 4 | Debug instrumentation | Stack trace logging before operation |

Real example: 4 layers added, 1847 tests passed, zero pollution.

**Condition-Based Waiting (condition-based-waiting.md):** Replaces arbitrary timeouts with polling:
```typescript
// BEFORE (flaky):
await new Promise(r => setTimeout(r, 50));

// AFTER (reliable):
await waitFor(() => getResult() !== undefined);
```

Poll every 10ms (not 1ms - wastes CPU; not 100ms - too slow).

**Pressure Signals:** When human partner says "Is that not happening?", "Will it show us...?", "Stop guessing", "Ultrathink this", or "We're stuck?" (frustrated) - STOP and return to Phase 1.

**Technical Debt:**
- find-polluter.sh is bash in a TypeScript codebase (could be Node)
- Repetition of "ALWAYS find root cause" (intentional but verbose)
- Purely prompt-based - no enforcement mechanism

---

### Feature 9: Verification Before Completion

**File:** `skills/verification-before-completion/SKILL.md`

Ensures bugs are actually fixed before declaring completion. Iron Law:

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

**The Gate Function:**
```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

**Key Insight:** Claiming work is complete without verification is dishonesty, not efficiency.

**Anti-Satisfaction Language:** Explicitly prevents premature celebration:
```
Red Flags - STOP:
- Expressing satisfaction before verification ("Great!", "Perfect!", "Done!")
```

**Agent Delegation Verification:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

**Anti-Rationalization Patterns:**

| Excuse | Reality |
|--------|----------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | Verify independently |
| "I'm tired" | Exhaustion ≠ excuse |

**Technical Observations:**
- Purely conceptual - no executable code
- Self-referential: the skill demonstrates its own principle
- Shortest skill analyzed (~140 lines)
- Strong values alignment: "Honesty is a core value. If you lie, you'll be replaced."

---

### Feature 10: Dispatching Parallel Agents

**File:** `skills/dispatching-parallel-agents/SKILL.md`

Coordination skill for dispatching multiple independent subagents to work in parallel on separate problem domains.

**When to Use:**
- 3+ test files failing with different root causes
- Multiple subsystems broken independently
- Each problem can be understood without context from others
- No shared state between investigations

**When NOT to Use:**
- Failures are related (fix one might fix others)
- Need to understand full system state
- Agents would interfere with each other
- Exploratory debugging (don't know what's broken yet)

**The Four-Step Pattern:**
1. Identify Independent Domains
2. Create Focused Agent Tasks (specific scope, clear goal, constraints, expected output)
3. Dispatch in Parallel (use Task() to dispatch multiple agents concurrently)
4. Review and Integrate (read summaries, check for conflicts, run full test suite)

**Agent Prompt Structure:**
Good prompts are:
- **Focused** - One clear problem domain
- **Self-contained** - All context needed to understand the problem
- **Specific about output** - What should the agent return?

**Real Example from Session:**
```
Failures:
- agent-tool-abort.test.ts: 3 failures (timing issues)
- batch-completion-behavior.test.ts: 2 failures (tools not executing)
- tool-approval-race-conditions.test.ts: 1 failure (execution count = 0)

Decision: Independent domains - abort logic separate from batch completion separate from race conditions

Dispatch:
Agent 1 → Fix agent-tool-abort.test.ts
Agent 2 → Fix batch-completion-behavior.test.ts
Agent 3 → Fix tool-approval-race-conditions.test.ts
```

**Common Mistakes:**

| Mistake | Problem | Fix |
|---------|---------|-----|
| "Fix all the tests" | Agent gets lost | "Fix agent-tool-abort.test.ts" - focused scope |
| "Fix the race condition" | Agent doesn't know where | Paste error messages and test names |
| No constraints | Agent might refactor everything | "Do NOT change production code" or "Fix tests only" |

**Key Design Decisions:**
1. **Token efficiency through isolation** - Agents should never inherit session context
2. **Verification after return** - Must run full test suite to catch conflicts
3. **Spot checking** - Agents can make systematic errors; orchestrator must verify

---

### Feature 11: Finishing a Development Branch

**File:** `skills/finishing-a-development-branch/SKILL.md`

Guides final integration after development work complete. Five-step process:

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

**Step 1 is Blocking:** Tests MUST pass before presenting options. If tests fail:
```
Tests failing (<N> failures). Must fix before completing:
[Show failures]
Cannot proceed with merge/PR until tests pass.
```

**Four Options (exactly 4, no more, no less):**
```
1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work
```

**Option Execution Matrix:**

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | Yes | No | No | Yes (local) |
| 2. Create PR | No | Yes | Yes | No |
| 3. Keep as-is | No | No | Yes | No |
| 4. Discard | No | No | No | Yes (force) |

**Discard Requires Typed Confirmation:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

**Integration Points:**
- Called by: subagent-driven-development (Step 7), executing-plans (Step 5)
- Pairs with: using-git-worktrees (cleanup worktree created by that skill)

---

### Feature 12: Writing Skills

**Files:**
- `skills/writing-skills/SKILL.md`
- `skills/writing-skills/anthropic-best-practices.md`
- `skills/writing-skills/testing-skills-with-subagents.md`
- `skills/writing-skills/persuasion-principles.md`
- `skills/writing-skills/graphviz-conventions.dot`
- `skills/writing-skills/render-graphs.js`

Guide for creating new skills following TDD methodology. Core principle:

**"Writing skills IS Test-Driven Development applied to process documentation."**

**TDD Mapping:**

| TDD Concept | Skill Creation |
|-------------|----------------|
| **Test case** | Pressure scenario with subagent |
| **Production code** | Skill document (SKILL.md) |
| **Test fails (RED)** | Agent violates rule without skill (baseline) |
| **Test passes (GREEN)** | Agent complies with skill present |
| **Refactor** | Close loopholes while maintaining compliance |

**Iron Law:**
```
NO SKILL WITHOUT A FAILING TEST FIRST
```

This applies to NEW skills AND EDITS to existing skills. No exceptions.

**Skill Types:**

| Type | What it is | Examples |
|------|------------|----------|
| **Technique** | Concrete method with steps | condition-based-waiting, root-cause-tracing |
| **Pattern** | Way of thinking about problems | flatten-with-flags, test-invariants |
| **Reference** | API docs, syntax guides | office docs |

**Claude Search Optimization (CSO) Trap:**

The description should ONLY describe triggering conditions, NOT summarize the skill's workflow. Testing revealed that when a description summarizes the workflow, Claude may follow the description instead of reading the full skill content.

```yaml
# BAD: Summarizes workflow - Claude may follow this instead of reading skill
description: Use when executing plans - dispatches subagent per task with code review between tasks

# GOOD: Just triggering conditions, no workflow summary
description: Use when executing implementation plans with independent tasks in the current session
```

**SKILL.md Frontmatter Format:**
```yaml
---
name: Skill-Name-With-Hyphens
description: Use when [specific triggering conditions and symptoms]
---
```

Rules:
- Two required fields: `name` and `description`
- Max 1024 characters total
- `name`: Letters, numbers, hyphens only (no parentheses, special chars)
- `description`: Third-person, describes ONLY when to use (NOT what it does)

**RED-GREEN-REFACTOR Cycle for Skills:**

**RED Phase:**
1. Create pressure scenarios (3+ combined pressures for discipline skills)
2. Run scenarios WITHOUT skill - document baseline behavior verbatim
3. Identify patterns in rationalizations/failures

**GREEN Phase:**
1. Name uses only letters, numbers, hyphens
2. YAML frontmatter with required fields
3. Description starts with "Use when..." and includes specific triggers
4. Clear overview with core principle
5. Address specific baseline failures identified in RED
6. Run scenarios WITH skill - verify agents now comply

**REFACTOR Phase:**
1. Identify NEW rationalizations from testing
2. Add explicit counters (if discipline skill)
3. Build rationalization table from all test iterations
4. Create red flags list
5. Re-test until bulletproof

**Key Design Decisions:**
1. **TDD for documentation** - Same discipline applied to code applies to documentation
2. **Skill types determine testing approach** - Different skills need different verification strategies
3. **Rationalization resistance** - Discipline-enforcing skills must be bulletproofed
4. **Token efficiency** - Skills should be concise; move details to supporting files
5. **Flat namespace** - All skills in one directory for easy search

---

### Feature 13: Receiving Code Review

**File:** `skills/receiving-code-review/SKILL.md`

Framework for responding to code review feedback constructively. The core principle is **technical rigor over social comfort** - verification before implementation, and reasoned pushback when feedback is technically incorrect.

**Response Pattern (6-step process):**
```
1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

**Forbidden Responses:**

| Forbidden | Reason |
|-----------|--------|
| "You're absolutely right!" | Explicit violation - performative agreement |
| "Great point!" / "Excellent feedback!" | Performative |
| "Let me implement that now" | Before verification |

**Instead:** Restate the technical requirement, ask clarifying questions, push back with technical reasoning, or just start working (actions > words).

**Source-Specific Handling:**

**From human partner:**
- Trusted - implement after understanding
- Still ask if scope unclear
- No performative agreement
- Skip to action or technical acknowledgment

**From external reviewers:**
- 5 checks before implementing:
  1. Technically correct for THIS codebase?
  2. Breaks existing functionality?
  3. Reason for current implementation?
  4. Works on all platforms/versions?
  5. Does reviewer understand full context?

**Signal Phrase:** "Strange things are afoot at the Circle K" - culturally aware workaround for agents who might avoid pushback.

**Anti-Pattern Tables:** Forbidden responses and common mistakes presented as tables for easy scanning during review.

**Technical Observations:**
- Pure markdown skill - no executable code
- Compact implementation (~214 lines) for depth of guidance
- No test coverage (being a skill document, this is expected)

---

### Feature 14: Using Superpowers

**Files:**
- `skills/using-superpowers/SKILL.md`
- `skills/using-superpowers/references/codex-tools.md`
- `skills/using-superpowers/references/gemini-tools.md`

Entry point skill that establishes the fundamental framework. Invoked at session start (`hooks/session-start`).

**Instruction Priority Hierarchy:**
```
1. User's explicit instructions (CLAUDE.md, GEMINI.md, AGENTS.md, direct requests) — HIGHEST
2. Superpowers skills — override default system behavior where they conflict
3. Default system prompt — LOWEST
```

If user instructions say "don't use TDD" and a skill says "always use TDD," follow the user's instructions.

**The Rule:**
```
Invoke relevant or requested skills BEFORE any response or action.
Even a 1% chance a skill might apply means you should invoke the skill.
If an invoked skill turns out to be wrong for the situation, you don't need to use it.
```

**Red Flags (Rationalization Detection):**

| Thought | Reality |
|---------|----------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "I can check git/files quickly" | Files lack conversation context. Check for skills. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |
| "I know what that means" | Knowing the concept ≠ using the skill. Invoke it. |

**Skill Priority:**
1. **Process skills first** (brainstorming, debugging) - determine HOW to approach
2. **Implementation skills second** (frontend-design, mcp-builder) - guide execution

**Skill Types:**
- **Rigid** (TDD, debugging): Follow exactly. Don't adapt away discipline.
- **Flexible** (patterns): Adapt principles to context.

**Platform Adaptation:**

Claude Code: `Skill` tool - load and present content, follow directly
Gemini CLI: `activate_skill` tool - load metadata at session start, full content on demand

**Notable Platform Limitations:**

**Codex:**
- No named agent registry - must find agent prompt file, read content, fill placeholders
- Multi-agent requires opt-in config: `multi_agent = true` in features
- Message framing: Use task-delegation framing ("Your task is..."), wrap in XML tags

**Gemini CLI:**
- No subagent support - falls back to single-session execution via `executing-plans`
- Additional tools: `list_directory`, `save_memory`, `ask_user`, `tracker_create_task`, `enter_plan_mode`/`exit_plan_mode`

**Key Design Patterns:**
1. **1% rule** - Even 1% chance means check; prevents rationalization
2. **Red flag table** - Psychologically sophisticated rationalization detection
3. **Process-first priority** - Ensures agents don't rush to solutions
4. **Rigid vs Flexible distinction** - Prevents over-engineering when flexibility appropriate
5. **SUBAGENT-STOP** - Prevents meta-invocation loops when dispatched as subagent

---

## Cross-Feature Analysis

### Shared Architectural Patterns

**1. Iron Law Pattern**

All discipline-enforcing skills use explicit prohibitive language:

| Feature | Iron Law |
|---------|----------|
| TDD | "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST" |
| Systematic Debugging | "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST" |
| Verification Before Completion | "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE" |
| Writing Skills | "NO SKILL WITHOUT A FAILING TEST FIRST" |

**2. Anti-Rationalization Tables**

Features 3, 8, 9, 12, and 13 all include explicit tables naming common excuses and the reality behind them.

**3. Verification Before Progression**

Both execution skills (SDD and Executing Plans) require verification before proceeding:
- Worktrees: Baseline tests before starting
- Finishing a Branch: Tests must pass before merge options
- Systematic Debugging: Hypothesis must be verified

**4. Fresh Context Over Inheritance**

- SDD: Fresh subagent per task with constructed context
- Worktrees: Fresh workspace on new branch
- Dispatching Parallel Agents: Orchestrator constructs exactly what each agent needs

**5. Structured Options Over Open-Ended**

- Finishing a Branch: Exactly 4 options, no explanation added
- Dispatching Parallel Agents: Explicit prompt structure requirements

### Workflow Integration

```
brainstorming → writing-plans → using-git-worktrees → [subagent-driven-development | executing-plans]
                                                                                    ↓
                                        [finishing-a-development-branch | dispatching-parallel-agents]
                                                                                    ↓
                                              [requesting-code-review | receiving-code-review]
                                                                                    ↓
                                                    systematic-debugging
                                                                                            ↓
                                                    verification-before-completion
```

**Meta-skill:** Writing Skills teaches how to create other skills following the same TDD principles it advocates.

### Technical Debt Summary

| Debt | Location | Impact |
|------|----------|--------|
| Deprecated commands | commands/brainstorm.md, write-plan.md, execute-plan.md | Redirect to skills; evolution pattern |
| File duplication | skills/code-reviewer.md = agents/code-reviewer.md | Potential sync issues |
| No enforcement mechanism | All prompt-based skills | Relies on discipline |
| find-polluter.sh in bash | TypeScript codebase | Inconsistency |
| No gh fallback | finishing-a-development-branch | Fails if GitHub CLI unavailable |
| DOT diagrams not rendered | using-superpowers | Documentation only |

### Clever Solutions Summary

1. **HARD-GATE markup** - Explicit enforcement preventing bypass
2. **"Similar to Task N" prohibition** - Forces complete task independence
3. **"Delete means delete"** - Prevents "adaptation" rationalization
4. **1% rule** - Threshold that prevents rationalization
5. **Spec BEFORE quality review** - Direction confirmation before execution check
6. **"Suspiciously quickly" detection** - Reviewer mistrust prompt
7. **Defense-in-depth** - Multiple validation layers
8. **Condition-based waiting** - Replaces arbitrary timeouts with polling
9. **Signal phrase** - Cultural workaround for pushback avoidance
10. **CSO trap** - Prevents description-summary bypass

### Notable Technical Innovations

**Zero-dependency WebSocket server** (brainstorming Visual Companion):
- Manual RFC 6455 implementation
- Dynamic port selection
- macOS fs.watch debouncing
- Owner process monitoring

**Bash bisection script** (find-polluter.sh):
- 64 lines for test pollution diagnosis
- Runs tests one-by-one to isolate polluter

**Root cause tracing via Error().stack**:
- Uses console.error in tests (not logger - may be suppressed)
- Backward tracing through call stack

---

## Feature Statistics

| Feature | Lines (SKILL.md) | Supporting Files | Tier |
|---------|-----------------|------------------|------|
| Brainstorming | ~500 | 6 files | CORE |
| Writing Plans | ~400 | 2 files | CORE |
| TDD | ~400 | 1 file | CORE |
| Subagent-Driven Dev | ~600 | 3 files | CORE |
| Executing Plans | ~300 | 1 file | CORE |
| Using Git Worktrees | ~350 | 0 files | CORE |
| Requesting Code Review | ~250 | 2 files | CORE |
| Systematic Debugging | ~450 | 7 files | SECONDARY |
| Verification Before Completion | ~140 | 0 files | SECONDARY |
| Dispatching Parallel Agents | ~300 | 0 files | SECONDARY |
| Finishing a Development Branch | ~300 | 0 files | SECONDARY |
| Writing Skills | ~600 | 6 files | SECONDARY |
| Receiving Code Review | ~214 | 0 files | SECONDARY |
| Using Superpowers | ~116 | 2 files | SECONDARY |

---

## Key Takeaways

**Architecture Philosophy:** Superpowers is designed to prevent rationalization under pressure. The Iron Law pattern appears consistently because it's the most effective way to prevent "just this once" exceptions.

**Skill Composability:** Skills are orthogonal and composable. Brainstorming feeds Writing Plans, which feeds execution, which feeds finishing. But quality assurance skills (debugging, verification, review) layer on at any point.

**Context vs Enforcement:** The system distinguishes between context-providing skills (brainstorming, writing plans) and enforcement skills (TDD, verification, debugging). Enforcement skills are non-negotiable; context skills are frameworks to follow.

**Human Authority:** The priority hierarchy (User > Skills > Default) ensures the human remains in control. Skills recommend, humans decide.

**Platform Adaptation:** The system acknowledges different AI platforms have different capabilities. Skills include platform-specific guidance and fallback procedures for missing features (e.g., Gemini CLI's lack of subagent support).
