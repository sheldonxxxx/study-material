OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/05a-features-batch-1.md

# Deep Dive: Feature Batch 1 Analysis

**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`
**Features Analyzed:** Brainstorming, Writing Plans, Test-Driven Development (Features 1-3)
**Date:** 2026-03-27

---

## Feature 1: Brainstorming

### Overview
**Skill File:** `skills/brainstorming/SKILL.md`
**Purpose:** Socratic design refinement skill that activates before writing code. Refines rough ideas through questions, explores alternatives, and presents design in digestible sections for validation.

### Core Workflow

The skill defines a strict 9-step checklist that must be completed in order:

1. Explore project context
2. Offer visual companion (if visual questions anticipated)
3. Ask clarifying questions (one at a time)
4. Propose 2-3 approaches with tradeoffs
5. Present design in sections, get approval after each
6. Write design doc to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
7. Spec self-review (inline fixes)
8. User reviews written spec
9. Transition to implementation via `writing-plans` skill

### Key Design Decisions

**HARD-GATE Anti-Pattern Prevention:**
```
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the
user has approved it.
</HARD-GATE>
```
The skill uses explicit `<HARD-GATE>` markup to prevent bypassing the design phase. This is a strong architectural choice ensuring no implementation happens before design validation.

**One Question at a Time:** Questions are explicitly limited to one per message. This prevents overwhelming the user and allows proper exploration of each topic.

**Visual Companion as Optional Tool:** The visual companion is offered as a consent-based tool, not forced on every session. The decision is per-question:
- Use browser for: UI mockups, architecture diagrams, side-by-side comparisons, visual design
- Use terminal for: requirements, conceptual choices, tradeoff lists, technical decisions

**Terminal State = writing-plans:** The skill explicitly forbids invoking any implementation skill except `writing-plans` after brainstorming completes. The ONLY next step is planning.

### Visual Companion Subsystem

**Server Implementation:** `skills/brainstorming/scripts/server.cjs`

A sophisticated zero-dependency WebSocket server written in pure Node.js (no external libraries):

**Key Implementation Details:**
- **WebSocket Protocol:** Manual RFC 6455 implementation with crypto for handshake
  - `computeAcceptKey()` uses SHA1 hash with magic string
  - `encodeFrame()` / `decodeFrame()` handle variable-length payloads (126/127 extended lengths)
  - Proper masking of client frames

- **Dynamic Port Selection:** Falls back to random port in ephemeral range if env var not set
  ```javascript
  const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
  ```

- **File Watching with Debouncing:** macOS fs.watch reports 'rename' for both new files and overwrites, so the server tracks known files to distinguish new screens from updates
  ```javascript
  const knownFiles = new Set(
    fs.readdirSync(CONTENT_DIR).filter(f => f.endsWith('.html'))
  );
  ```

- **Owner Process Monitoring:** Server can track a parent PID and exit if it dies (common in WSL, Tailscale SSH, cross-user scenarios). Validates at startup and disables if PID already dead.

- **Idle Timeout:** 30-minute timeout auto-shuts down server. Lifecycle check runs every 60 seconds.

**Content Fragments vs Full Documents:**
- Content fragments are automatically wrapped in the frame template
- Full documents (starting with `<!doctype` or `<html>`) are served as-is
- This allows both quick fragments and complete custom pages

**Platform-Specific Launch Notes:**
- Claude Code (Windows): Must use `run_in_background: true` to survive conversation turns
- Codex: Auto-detects `CODEX_CI` env var and uses foreground mode
- Gemini CLI: Must use `--foreground` flag with background shell execution

**CSS Framework:** `frame-template.html` provides OS-aware light/dark theming via `prefers-color-scheme` media query. Includes comprehensive CSS classes:
- `.options` / `.option` for A/B/C choices
- `.cards` / `.card` for visual designs
- `.mockup` container with header
- `.split` for side-by-side comparisons
- `.pros-cons` grid layout

**Client Helper:** `helper.js` implements:
- WebSocket auto-reconnect with 1-second delay
- Click capture on `[data-choice]` elements
- Selection state tracking with visual indicator updates
- Event queue for offline operation

### Spec Document Reviewer Sub-Pattern

**File:** `skills/brainstorming/spec-document-reviewer-prompt.md`

This is a reusable reviewer template for dispatching a spec reviewer subagent. Key calibration guidance:
- Only flag issues that would cause real problems during implementation planning
- Minor wording improvements are NOT issues
- Approve unless serious gaps that would lead to a flawed plan

### Technical Debt / Shortcuts

1. **Commands are Deprecated:** `commands/brainstorm.md` simply redirects to the skill with a deprecation notice. This shows evolution from command-based to skill-based invocation.

2. **Visual Companion is Non-Negotiable:** The skill itself is mandatory ("You MUST use this before any creative work") but the Visual Companion is optional. This is a reasonable tradeoff.

3. **Single Session Directory:** The server writes session info to `$STATE_DIR/server-info` and `server-stopped` marker. No cleanup mechanism for orphaned sessions in `/tmp`.

4. **No Authentication:** The WebSocket server has no authentication. Any local user can connect and send events. Mitigated by only binding to localhost by default.

### Strengths
- Strong anti-pattern enforcement via HARD-GATE
- Clear escalation path from brainstorming to planning
- Well-designed Visual Companion with thoughtful per-question usage guidance
- Platform-aware server with proper lifecycle management

---

## Feature 2: Writing Plans

### Overview
**Skill File:** `skills/writing-plans/SKILL.md`
**Purpose:** Creates detailed implementation plans with bite-sized tasks (2-5 minutes each). Every task includes exact file paths, complete code examples, and verification steps.

### Core Structure

**Plan Document Header Template:**
```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]
**Architecture:** [2-3 sentences about approach]
**Tech Stack:** [Key technologies/libraries]
```

**Task Structure:**
Each task follows a TDD-influenced step structure:
1. Write the failing test (with actual test code)
2. Run test to verify it fails
3. Write minimal implementation
4. Run test to verify it passes
5. Commit

### Key Design Decisions

**No Placeholders Policy:**
The skill explicitly lists plan failures that must never be written:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (must repeat code)
- Steps that describe without showing code
- References to types/functions not defined in any task

**File Structure Before Tasks:**
Before defining tasks, the plan must map out which files will be created/modified. This locks in decomposition decisions. Guidance includes:
- Files that change together should live together
- Prefer smaller, focused files over large ones
- Follow established patterns in existing codebases

**Scope Check:**
If the spec covers multiple independent subsystems, the plan should flag this and suggest breaking into separate plans. Each plan should produce working, testable software on its own.

**Self-Review Checklist:**
After writing the complete plan, the skill mandates a self-review:
1. Spec coverage - can you point to a task for each requirement?
2. Placeholder scan - search for red flags
3. Type consistency - do names match across tasks?

**Execution Handoff:**
After saving the plan, offers two execution paths:
1. **Subagent-Driven (recommended)** - fresh subagent per task + two-stage review
2. **Inline Execution** - batch execution with checkpoints using `executing-plans`

### Task Granularity Philosophy

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

This extremely fine granularity is unusual. Most plan templates use larger tasks (hours). The 2-5 minute constraint suggests optimization for AI agent execution where attention spans are limited and context windows matter.

### Plan Document Reviewer Sub-Pattern

**File:** `skills/writing-plans/plan-document-reviewer-prompt.md`

Similar reviewer template to the spec reviewer:
- Checks completeness, spec alignment, task decomposition, buildability
- Calibration: approve unless serious gaps — missing requirements, contradictory steps, placeholder content, or tasks too vague to act on

### Technical Debt / Observations

1. **Command is Deprecated:** `commands/write-plan.md` simply redirects to the skill. Shows same evolution pattern as brainstorm command.

2. **Assumes TDD:** The task structure heavily implies TDD methodology. Each task has RED/GREEN/REFACTOR steps built in. Plans that skip TDD would need different template.

3. **No Time Estimates:** Plans have no human time estimates, only AI execution time (2-5 minutes per step). This is appropriate for AI-native workflow.

4. **File Path Precision Required:** Plans must use "exact file paths" but the skill doesn't prescribe how to determine those paths. Assumes prior exploration was done during brainstorming.

### Strengths
- Extremely detailed task structure prevents "planner's block"
- No-placeholder policy forces complete thinking before committing plan
- Self-review checklist catches common issues
- Clear handoff to execution layer with two options

### Clever Solution
The "Similar to Task N" prohibition is clever - it forces the planner to repeat code rather than assume sequential reading. Since executors may read tasks out of order or parallelize, this prevents broken references.

---

## Feature 3: Test-Driven Development (TDD)

### Overview
**Skill File:** `skills/test-driven-development/SKILL.md`
**Supporting File:** `skills/test-driven-development/testing-anti-patterns.md`
**Purpose:** Enforces RED-GREEN-REFACTOR cycle during implementation. Writes failing test first, watches it fail, writes minimal code to pass, then refactors.

### Core Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

The skill makes this extremely explicit with enforcement mechanisms:
- "Write code before the test? Delete it. Start over."
- "Don't keep it as 'reference'"
- "Don't 'adapt' it while writing tests"
- "Don't look at it"
- "Delete means delete"
- "Implement fresh from tests. Period."

### RED-GREEN-REFACTOR Cycle

**RED - Write Failing Test:**
- One minimal test showing what should happen
- Requirements: one behavior, clear name, real code (no mocks unless unavoidable)
- Good example tests actual behavior, not mock existence

**Verify RED - Watch It Fail:**
- MANDATORY - never skip
- Confirm: test fails (not errors), failure message is expected, fails because feature missing
- If test passes: you're testing existing behavior, fix the test
- If test errors: fix error until it fails correctly

**GREEN - Minimal Code:**
- Write simplest code to pass the test
- Don't add features, refactor other code, or "improve" beyond the test
- Explicitly shows over-engineered bad example vs good minimal example

**Verify GREEN - Watch It Pass:**
- MANDATORY
- Confirm: test passes, other tests still pass, output pristine

**REFACTOR - Clean Up:**
- After green only
- Remove duplication, improve names, extract helpers
- Keep tests green

### When to Use TDD

**Always:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (must ask human):**
- Throwaway prototypes
- Generated code
- Configuration files

The skill explicitly calls out the rationalization pattern: "Thinking 'skip TDD just this once'? Stop. That's rationalization."

### Anti-Rationalization Arguments

The skill provides detailed rebuttals to common excuses:

| Excuse | Rebuttal |
|--------|----------|
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Already manually tested" | Ad-hoc, no record, can't re-run, not comprehensive |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "TDD will slow me down" | TDD faster than debugging. Pragmatic = test-first. |

### Testing Anti-Patterns Reference

**File:** `testing-anti-patterns.md`

This is a comprehensive reference with 5 main anti-patterns:

**Anti-Pattern 1: Testing Mock Behavior**
- Asserting on mock elements (`*-mock` test IDs) instead of real behavior
- Gate function: "Am I testing real component behavior or just mock existence?"
- Fix: Test real component or unmock it

**Anti-Pattern 2: Test-Only Methods in Production**
- Adding `destroy()` or similar methods only used in tests
- Gate function: "Is this only used by tests?" If yes, put in test utilities
- Fix: Keep production classes clean, use external test utilities

**Anti-Pattern 3: Mocking Without Understanding**
- Mocking "to be safe" without understanding side effects
- Example: Mock breaks config write that test depends on
- Gate function: Understand dependencies before mocking

**Anti-Pattern 4: Incomplete Mocks**
- Partial mocks only including fields you think you need
- Silent failures when downstream code accesses omitted fields
- Iron Rule: Mock the COMPLETE data structure as it exists in reality

**Anti-Pattern 5: Integration Tests as Afterthought**
- "Implementation complete, no tests written"
- Fix: TDD cycle

### Verification Checklist

Before marking work complete:
- [ ] Every new function/method has a test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for expected reason
- [ ] Wrote minimal code to pass each test
- [ ] All tests pass
- [ ] Output pristine
- [ ] Tests use real code (mocks only if unavoidable)
- [ ] Edge cases and errors covered

### When Stuck Table

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write wished-for API, write assertion first |
| Test too complicated | Design too complicated, simplify interface |
| Must mock everything | Code too coupled, use dependency injection |
| Test setup huge | Extract helpers, still complex? Simplify design |

### Debugging Integration

"Bug found? Write failing test reproducing it. Follow TDD cycle. Test proves fix and prevents regression. Never fix bugs without a test."

### Strengths
- Extremely dogmatic enforcement prevents rationalization
- Comprehensive rebuttals to common excuses
- Detailed anti-patterns reference prevents TDD mistakes
- Clear "when stuck" guidance
- Integration with debugging workflow

### Technical Debt / Observations

1. **No Tooling:** The skill doesn't prescribe specific test frameworks or tooling. It's methodology-agnostic. This is appropriate for a multi-platform system.

2. **Dogmatic Tone is Intentional:** The rigid "Iron Law" tone with phrases like "Violating the letter of the rules is violating the spirit of the rules" appears deliberate to overcome human rationalization tendencies.

3. **Integration with writing-plans:** The writing-plans skill embeds TDD structure into tasks automatically. The skills are designed to compose together.

4. **No Automated Verification:** Unlike the other skills, there's no reviewer subagent template for TDD. It's meant to be enforced manually during implementation.

### Clever Solution
The "Delete means delete" enforcement with the "Iron Law" framing addresses a real problem: developers who "write tests after" but claim TDD benefits. By requiring deletion and fresh implementation from tests, it prevents the common failure mode of adapting existing code.

---

## Cross-Feature Analysis

### Skill Composition

These three skills form a pipeline:
```
brainstorming -> writing-plans -> TDD (embedded in execution)
```

**Brainstorming** produces: `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`

**Writing Plans** consumes: spec, produces: `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`

**TDD** is embedded within plan execution, enforced by the writing-plans task structure.

### Shared Patterns

1. **Reviewer Subagents:** Both brainstorming and writing-plans have reusable reviewer prompt templates for dispatching subagents to validate outputs before proceeding.

2. **Deprecated Commands:** `commands/brainstorm.md` and `commands/write-plan.md` simply redirect to skills. Shows evolution toward skill-based invocation model.

3. **No-Placeholder Policies:** Both writing-plans and brainstorming spec reviewer have explicit placeholder/TBD scanning requirements.

4. **Human-in-the-Loop Gates:** Brainstorming has design approval gate, writing-plans has execution choice, TDD has exception-asking.

### Platform Considerations

All three skills are platform-agnostic. The Visual Companion server in brainstorming is the only component with platform-specific launch notes, reflecting the variety of AI coding platforms supported (Claude Code, Cursor, Codex, Gemini CLI, OpenCode).

### Files Created by These Features

| Feature | Files |
|---------|-------|
| Brainstorming | `skills/brainstorming/SKILL.md`, `spec-document-reviewer-prompt.md`, `visual-companion.md`, `scripts/server.cjs`, `scripts/frame-template.html`, `scripts/helper.js`, `scripts/start-server.sh`, `scripts/stop-server.sh` |
| Writing Plans | `skills/writing-plans/SKILL.md`, `plan-document-reviewer-prompt.md` |
| TDD | `skills/test-driven-development/SKILL.md`, `testing-anti-patterns.md` |

---

## Summary

**Brainstorming** is distinguished by its HARD-GATE enforcement preventing implementation before design, and its sophisticated Visual Companion subsystem with zero-dependency WebSocket server and OS-aware theming.

**Writing Plans** is distinguished by its extremely fine task granularity (2-5 minutes per step), no-placeholder policy forcing complete thinking, and TDD-embedded task structure.

**TDD** is distinguished by its uncompromising Iron Law enforcement, comprehensive anti-rationalization arguments, and detailed anti-patterns reference covering 5 distinct testing failure modes.

All three skills show evidence of extensive real-world usage refinement, with patterns like deprecated commands, reviewer subagents, and platform-specific guidance reflecting iterative improvement.
