OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/05c-features-batch-3.md

# Feature Analysis: Batch 3 (Features 7-9)

**Features Analyzed:**
- Feature 7: Requesting Code Review
- Feature 8: Systematic Debugging
- Feature 9: Verification Before Completion

**Analysis Date:** 2026-03-27
**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`

---

## Feature 7: Requesting Code Review

### Overview

**Description:** Pre-review checklist workflow. Reviews implementation against plan and reports issues by severity. Critical issues block progress.

**Priority:** CORE (Tier 1)

### Core Components

| File | Purpose |
|------|---------|
| `skills/requesting-code-review/SKILL.md` | Main skill definition with workflow |
| `skills/requesting-code-review/code-reviewer.md` | Parameterized prompt template for subagent |
| `skills/code-reviewer.md` | Shared code reviewer prompt (agents directory) |

### Workflow Analysis

The feature implements a clean separation between the **requestor workflow** (in SKILL.md) and the **reviewer prompt** (in code-reviewer.md).

**Requestor Workflow (SKILL.md):**

```
1. Get git SHAs:
   BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
   HEAD_SHA=$(git rev-parse HEAD)

2. Dispatch code-reviewer subagent with filled template

3. Act on feedback:
   - Fix Critical issues immediately
   - Fix Important issues before proceeding
   - Note Minor issues for later
   - Push back if reviewer is wrong (with reasoning)
```

**Reviewer Prompt (code-reviewer.md):**

The reviewer prompt is a parameterized template designed for subagent injection:

```
WHAT_WAS_IMPLEMENTED: {DESCRIPTION}
PLAN_OR_REQUIREMENTS: {PLAN_REFERENCE}
BASE_SHA: {BASE_SHA}
HEAD_SHA: {HEAD_SHA}
```

The reviewer receives:
- Git diff commands to run on the specified range
- A structured checklist covering Code Quality, Architecture, Testing, Requirements, Production Readiness
- An output format requiring specific severity categorization

### Issue Categorization

The reviewer categorizes issues into three severity tiers:

| Severity | Definition | Action |
|----------|------------|--------|
| **Critical** | Bugs, security issues, data loss risks, broken functionality | Must Fix |
| **Important** | Architecture problems, missing features, poor error handling, test gaps | Should Fix |
| **Minor** | Code style, optimization opportunities, documentation improvements | Nice to Have |

### Clever Solutions

1. **Subagent Context Isolation:** The reviewer gets "precisely crafted context for evaluation — never your session's history." This keeps the reviewer focused on the work product, not the thought process.

2. **Git Range Context:** Embedding `git diff --stat` and `git diff` commands allows the reviewer to inspect exactly what changed between commits.

3. **Severity Calibration:** The prompt explicitly warns against marking nitpicks as Critical and provides examples of proper categorization.

4. **Push-Back Guidance:** The skill explicitly encourages pushing back if "reviewer is wrong (with reasoning)" — preventing blind deference.

5. **Integration Points:** The skill documents how it connects with:
   - Subagent-Driven Development (review after EACH task)
   - Executing Plans (review after each batch of 3 tasks)
   - Ad-Hoc Development (review before merge)

### Red Flags (Anti-Patterns)

The skill explicitly defines what NOT to do:

```
Never:
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback
```

### Technical Observations

- The `code-reviewer.md` file in `agents/` is identical to `skills/requesting-code-review/code-reviewer.md` — likely a symlink or copy for discoverability
- The skill is purely prompt-based — no executable code
- Integration requires the orchestrator to support subagent dispatching with the `superpowers:code-reviewer` type
- The template uses placeholder syntax `{PLACEHOLDER}` that an orchestrator must replace before dispatch

---

## Feature 8: Systematic Debugging

### Overview

**Description:** 4-phase root cause analysis process including root-cause-tracing, defense-in-depth, and condition-based-waiting techniques.

**Priority:** SECONDARY (Tier 2)

### Core Components

| File | Purpose |
|------|---------|
| `skills/systematic-debugging/SKILL.md` | Main skill with 4-phase process |
| `skills/systematic-debugging/root-cause-tracing.md` | Backward tracing through call stack |
| `skills/systematic-debugging/defense-in-depth.md` | Multi-layer validation pattern |
| `skills/systematic-debugging/condition-based-waiting.md` | Polling pattern to replace arbitrary timeouts |
| `skills/systematic-debugging/condition-based-waiting-example.ts` | Implementation of waitFor* utilities |
| `skills/systematic-debugging/find-polluter.sh` | Bash bisection script for test pollution |
| `skills/systematic-debugging/CREATION-LOG.md` | Meta-documentation of skill creation |
| `skills/systematic-debugging/test-pressure-*.md` | Test validation documents (3 files) |

### The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

This is repeated 4+ times throughout the skill in different contexts. The CREATION-LOG notes that this redundancy is intentional — it's "bulletproofing" against rationalization under pressure.

### The Four Phases

**Phase 1: Root Cause Investigation**
- Read error messages carefully (don't skip past)
- Reproduce consistently
- Check recent changes
- For multi-component systems: add diagnostic instrumentation at each boundary
- For deep call stacks: use root-cause-tracing technique

**Phase 2: Pattern Analysis**
- Find working examples
- Compare against references (read COMPLETELY, don't skim)
- Identify differences
- Understand dependencies

**Phase 3: Hypothesis and Testing**
- Form single hypothesis ("I think X is root cause because Y")
- Test minimally (one variable at a time)
- Verify before continuing

**Phase 4: Implementation**
- Create failing test case first
- Implement single fix
- Verify fix works
- If 3+ fixes failed: question the architecture

### Clever Solutions

#### 1. Root Cause Tracing (root-cause-tracing.md)

Uses a backward tracing technique:

```
Error: git init failed in /Users/jesse/project/packages/core
  → git init runs with empty cwd parameter
    → WorktreeManager called with empty projectDir
      → Session.create() passed empty string
        → Test accessed context.tempDir before beforeEach
          → setupCoreTest() returns { tempDir: '' } initially
```

The technique uses `new Error().stack` for stack trace instrumentation:

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**Key insight:** Uses `console.error()` in tests (not logger — logger may be suppressed).

#### 2. find-polluter.sh — Test Pollution Bisection

A bash script that runs tests one-by-one to find which test creates unwanted files/state:

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

Algorithm:
1. Run each test file individually
2. Check if pollution file exists after each
3. Stop at first polluter found
4. Report which test created the pollution

This is a classic bisection pattern implemented in 64 lines of bash.

#### 3. Defense-in-Depth (defense-in-depth.md)

After finding root cause, add validation at EVERY layer data passes through:

| Layer | Purpose | Example |
|-------|---------|---------|
| Layer 1 | Entry point validation | `if (!dir) throw new Error()` |
| Layer 2 | Business logic validation | `if (!projectDir) throw new Error()` |
| Layer 3 | Environment guards | `if (NODE_ENV === 'test' && !inTmpDir()) refuse` |
| Layer 4 | Debug instrumentation | Stack trace logging before operation |

Real example from session: 4 layers added, 1847 tests passed, zero pollution.

#### 4. Condition-Based Waiting (condition-based-waiting.md)

Replaces arbitrary timeouts with polling:

```typescript
// BEFORE (flaky):
await new Promise(r => setTimeout(r, 50));

// AFTER (reliable):
await waitFor(() => getResult() !== undefined);
```

Implementation in `condition-based-waiting-example.ts` provides three utilities:

```typescript
waitForEvent(threadManager, threadId, 'TOOL_RESULT', timeoutMs)
waitForEventCount(threadManager, threadId, 'TOOL_CALL', 2, timeoutMs)
waitForEventMatch(threadManager, threadId, predicate, description, timeoutMs)
```

**Key pattern:** Poll every 10ms (not 1ms — wastes CPU; not 100ms — too slow).

#### 5. CREATION-LOG — Meta-Documentation

The `CREATION-LOG.md` documents not just what the skill does, but WHY certain design choices were made:

- Language choices: "ALWAYS" / "NEVER" vs "should" / "try to"
- Structural defenses: Phase 1 required, single hypothesis rule
- Redundancy for bulletproofing: Root cause mandate appears 4+ times
- Testing: 4 validation tests created with different pressure scenarios

This is a meta-level document showing how the skill itself was validated.

### Anti-Rationalization Patterns

The skill explicitly names common rationalizations:

| Excuse | Reality |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from start |
| "I'll write test after confirming fix works" | Untested fixes don't stick |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause |

### Pressure Signals

The skill defines "your human partner's Signals You're Doing It Wrong":

```
- "Is that not happening?" - You assumed without verifying
- "Will it show us...?" - You should have added evidence gathering
- "Stop guessing" - You're proposing fixes without understanding
- "Ultrathink this" - Question fundamentals, not just symptoms
- "We're stuck?" (frustrated) - Your approach isn't working
```

When these appear: STOP, return to Phase 1.

### Technical Debt/Shortcuts

1. **Bash script vs Node:** `find-polluter.sh` is bash, while rest of codebase is likely TypeScript. Could be ported to Node for consistency.

2. **Repetition:** The "ALWAYS find root cause" message appears many times. While intentional for bulletproofing, could be seen as verbose.

3. **No enforcement mechanism:** Like Requesting Code Review, purely prompt-based. Depends on human/agent discipline to follow.

---

## Feature 9: Verification Before Completion

### Overview

**Description:** Ensures bugs are actually fixed before declaring completion. Validates the fix works in multiple scenarios.

**Priority:** SECONDARY (Tier 2)

### Core Components

| File | Purpose |
|------|---------|
| `skills/verification-before-completion/SKILL.md` | Main skill with Iron Law and gate function |

### The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

**Violating the letter of this rule is violating the spirit of this rule.**

### The Gate Function

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

### Key Insight

The skill recognizes that **claiming work is complete without verification is dishonesty, not efficiency.** This is a values statement, not just a process recommendation.

### Claim Verification Table

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check, extrapolation |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red-green cycle verified | Test passes once |
| Agent completed | VCS diff shows changes | Agent reports "success" |
| Requirements met | Line-by-line checklist | Tests passing |

### Clever Solutions

#### 1. Anti-Satisfaction Language

The skill explicitly prevents premature celebration:

```
Red Flags - STOP:
- Expressing satisfaction before verification ("Great!", "Perfect!", "Done!")
```

This blocks the common pattern of expressing relief/satisfaction before actually verifying.

#### 2. Trust Broken Prevention

Documents specific failure memories:

```
From 24 failure memories:
- your human partner said "I don't believe you" - trust broken
- Undefined functions shipped - would crash
- Missing requirements shipped - incomplete features
- Time wasted on false completion → redirect → rework
```

This grounds the rule in real consequences.

#### 3. TDD Integration

Explicitly connects with TDD's red-green-refactor cycle:

```
Regression tests (TDD Red-Green):
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

#### 4. Agent Delegation Verification

When agents report success:

```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

This prevents blind trust in delegated work.

### Anti-Rationalization Patterns

| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | Verify independently |
| "I'm tired" | Exhaustion ≠ excuse |
| "Partial check is enough" | Partial proves nothing |
| "Different words so rule doesn't apply" | Spirit over letter |

### When To Apply

The skill specifies **ALWAYS before:**
- ANY variation of success/completion claims
- ANY expression of satisfaction
- ANY positive statement about work state
- Committing, PR creation, task completion
- Moving to next task
- Delegating to agents

### Technical Observations

- **Purely conceptual:** No executable code, purely a values and process statement
- **Self-referential:** The skill itself demonstrates its own principle — it doesn't just say "verify", it makes you pause before claiming anything
- **Minimal file:** At 140 lines, the shortest skill in the three analyzed here
- **Strong values alignment:** References "Honesty is a core value. If you lie, you'll be replaced."

---

## Cross-Cutting Analysis

### Shared Patterns

1. **Iron Law Pattern:** All three features use an "Iron Law" formulation with strong prohibitive language to prevent shortcuts.

2. **Anti-Rationalization Tables:** Features 8 and 9 both include explicit tables naming common excuses and the reality behind them.

3. **Purely Prompt-Based:** None of these features have executable code — they're all guidance frameworks that depend on human/agent discipline.

4. **Meta-Documentation:** Feature 8 includes a CREATION-LOG showing HOW the skill was built, not just what it does.

### Feature Relationships

```
Systematic Debugging (8)
    ↓
    └── Verification Before Completion (9)
            │
            └── Requesting Code Review (7)
```

The debugging skill feeds into verification (fix must be verified), which feeds into code review (verification evidence supports the review).

### Notable Technical Debt

1. **No enforcement:** All three are guidance-only; no hooks or automation enforce their principles.

2. **Duplication:** `skills/code-reviewer.md` and `skills/requesting-code-review/code-reviewer.md` appear identical.

3. **find-polluter.sh:** Bash script in a TypeScript codebase (could be Node for consistency).

---

## Files Analyzed

```
skills/requesting-code-review/
├── SKILL.md
└── code-reviewer.md

skills/code-reviewer.md (shared)

skills/systematic-debugging/
├── SKILL.md
├── root-cause-tracing.md
├── defense-in-depth.md
├── condition-based-waiting.md
├── condition-based-waiting-example.ts
├── find-polluter.sh
├── CREATION-LOG.md
└── test-pressure-*.md (3 files)

skills/verification-before-completion/
└── SKILL.md
```
