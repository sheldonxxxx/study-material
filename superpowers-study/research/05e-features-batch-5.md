OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/05e-features-batch-5.md

# Feature Batch 5 Analysis: Receiving Code Review, Using Superpowers

**Repository:** `/Users/sheldon/Documents/claw/reference/superpowers`
**Features Analyzed:** Features 13-14
**Date:** 2026-03-27

---

## Feature 13: Receiving Code Review

**Description:** Framework for responding to code review feedback constructively and efficiently.
**Tier:** SECONDARY
**Main File:** `skills/receiving-code-review/SKILL.md`

### Purpose and Context

This skill addresses the psychology and process of receiving code review feedback. Unlike `requesting-code-review` (which prepares code for review), this skill focuses on how an agent should respond when it IS the one being reviewed. The core principle is **technical rigor over social comfort** - verification before implementation, and reasoned pushback when feedback is technically incorrect.

### Core Implementation

**Single file skill:** `skills/receiving-code-review/SKILL.md` (214 lines)

The skill implements a complete framework with these sections:
- Response Pattern (6-step process)
- Forbidden Responses (anti-patterns)
- Handling Unclear Feedback
- Source-Specific Handling
- YAGNI Check
- Implementation Order
- When To Push Back
- Acknowledging Correct Feedback
- Gracefully Correcting Pushback
- Common Mistakes table
- Real Examples
- GitHub Thread Replies

### Key Flow

```
WHEN receiving code review feedback:
1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

### Forbidden Responses (Anti-Patterns)

The skill explicitly lists responses that should NEVER be used:

| Forbidden | Reason |
|-----------|--------|
| "You're absolutely right!" | Explicit violation - performative agreement |
| "Great point!" / "Excellent feedback!" | Performative |
| "Let me implement that now" | Before verification |

**Instead:** Restate the technical requirement, ask clarifying questions, push back with technical reasoning, or just start working (actions > words).

### Handling Unclear Feedback

```
IF any item is unclear:
  STOP - do not implement anything yet
  ASK for clarification on unclear items
```

**Critical insight:** Items may be related. Partial understanding leads to wrong implementation. The example shows this clearly:
- "Fix items 1-6" - understands 1,2,3,6, unclear on 4,5
- WRONG: Implement 1,2,3,6 now, ask about 4,5 later
- RIGHT: "I understand items 1,2,3,6. Need clarification on 4 and 5 before proceeding."

### Source-Specific Handling

The skill distinguishes between feedback sources with different trust levels:

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
- Push back with technical reasoning if suggestion seems wrong
- State if can't verify: "I can't verify this without [X]. Should I [investigate/ask/proceed]?"

### YAGNI Check

When reviewers suggest "implementing properly" with features that seem over-engineered:

```bash
grep codebase for actual usage
IF unused: "This endpoint isn't called. Remove it (YAGNI)?"
IF used: Then implement properly
```

**User's rule embedded:** "You and reviewer both report to me. If we don't need this feature, don't add it."

### Implementation Order

For multi-item feedback:
1. Clarify anything unclear FIRST
2. Then implement in order:
   - Blocking issues (breaks, security)
   - Simple fixes (typos, imports)
   - Complex fixes (refactoring, logic)
3. Test each fix individually
4. Verify no regressions

### When To Push Back

Push back when:
- Suggestion breaks existing functionality
- Reviewer lacks full context
- Violates YAGNI (unused feature)
- Technically incorrect for this stack
- Legacy/compatibility reasons exist
- Conflicts with human partner's architectural decisions

**How to push back:** Technical reasoning, not defensiveness. Ask specific questions. Reference working tests/code. Involve human partner if architectural.

**Easter egg:** Signal phrase if uncomfortable pushing back out loud: "Strange things are afoot at the Circle K"

### Acknowledging Correct Feedback

When feedback IS correct, only these responses are acceptable:

```
"Fixed. [Brief description of what changed]"
"Good catch - [specific issue]. Fixed in [location]."
[Just fix it and show in the code]
```

**NEVER:** Any gratitude expression ("Thanks", "Thank you", etc.)

**Why:** Actions speak. Just fix it. The code itself shows you heard the feedback.

### Gracefully Correcting Pushback

If you pushed back and were wrong:
```
"You were right - I checked [X] and it does [Y]. Implementing now."
"Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."
```

**NEVER:** Long apology, defending why you pushed back, over-explaining.

### GitHub Thread Replies

When replying to inline review comments, reply in the comment thread, not as top-level PR comment.

### Real Examples

**Performative Agreement (Bad):**
```
Reviewer: "Remove legacy code"
WRONG: "You're absolutely right! Let me remove that..."
```

**Technical Verification (Good):**
```
Reviewer: "Remove legacy code"
GOOD: "Checking... build target is 10.15+, this API needs 13+. Need legacy for backward compat. Current impl has wrong bundle ID - fix it or drop pre-13 support?"
```

**YAGNI (Good):**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
GOOD: "Grepped codebase - nothing calls this endpoint. Remove it (YAGNI)? Or is there usage I'm missing?"
```

### Clever Solutions / Design Patterns

1. **Anti-pattern tables:** Forbidden responses and common mistakes are presented as tables with fixes - easy to scan during review

2. **Source trust hierarchy:** Different verification rigor for human partner vs external reviewers

3. **Signal phrase:** "Strange things are afoot at the Circle K" - culturally aware workaround for agents who might avoid pushback

4. **Embedded user authority:** "your human partner's rule" citations embed the human's decisions directly into the skill, creating traceability

5. **Gratitude prohibition:** The explicit "no thanks" rule is psychologically interesting - it recognizes that performative agreement often comes from politeness conditioning

### Technical Debt / Shortcuts

- Pure markdown skill - no executable code
- No test coverage (being a skill document, this is expected)
- Compact implementation (~214 lines) for the depth of guidance provided

---

## Feature 14: Using Superpowers

**Description:** Introduction skill explaining the overall skills system and how skills trigger automatically.
**Tier:** SECONDARY
**Main Files:**
- `skills/using-superpowers/SKILL.md`
- `skills/using-superpowers/references/codex-tools.md`
- `skills/using-superpowers/references/gemini-tools.md`

### Purpose and Context

This is the **entry point skill** that establishes the fundamental framework of the Superpowers system. It is invoked at the start of every conversation (per `hooks/session-start`) and explains:
1. How to access skills
2. How to use skills (the rule)
3. The priority hierarchy (User > Skills > Default)
4. Red flags that indicate rationalization
5. Platform adaptation

### Core Implementation

**Main skill file:** `skills/using-superpowers/SKILL.md` (116 lines)

The skill has these sections:
- SUBAGENT-STOP condition
- EXTREMELY-IMPORTANT directive
- Instruction Priority hierarchy
- How to Access Skills (per platform)
- Platform Adaptation note
- Using Skills (The Rule + DOT diagram)
- Red Flags table
- Skill Priority
- Skill Types (Rigid vs Flexible)
- User Instructions

### Instruction Priority Hierarchy

```
1. User's explicit instructions (CLAUDE.md, GEMINI.md, AGENTS.md, direct requests) — HIGHEST
2. Superpowers skills — override default system behavior where they conflict
3. Default system prompt — LOWEST
```

**Key principle:** If user instructions say "don't use TDD" and a skill says "always use TDD," follow the user's instructions. The user is in control.

### The Rule

```
Invoke relevant or requested skills BEFORE any response or action.
Even a 1% chance a skill might apply means you should invoke the skill.
If an invoked skill turns out to be wrong for the situation, you don't need to use it.
```

### Skill Flow DOT Diagram

The skill includes a DOT (graphviz) diagram showing the decision flow:

```
User message received
    ↓
About to EnterPlanMode? → (branch to check if already brainstormed)
    ↓
Already brainstormed? → NO → Invoke brainstorming skill
    ↓ YES
Might any skill apply? → YES → Invoke Skill tool (even 1% chance)
    ↓ NO (definitely not)
Respond (including clarifications)
    ↓
Invoke Skill tool → Announce: "Using [skill] to [purpose]"
    ↓
Has checklist? → YES → Create TodoWrite todo per item
    ↓ NO
Follow skill exactly
    ↓
Respond
```

### Red Flags (Rationalization Detection)

These thoughts mean STOP - you're rationalizing:

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "I can check git/files quickly" | Files lack conversation context. Check for skills. |
| "Let me gather information first" | Skills tell you HOW to gather information. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I remember this skill" | Skills evolve. Read current version. |
| "This doesn't count as a task" | Action = task. Check for skills. |
| "The skill is overkill" | Simple things become complex. Use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |
| "This feels productive" | Undisciplined action wastes time. Skills prevent this. |
| "I know what that means" | Knowing the concept ≠ using the skill. Invoke it. |

### Skill Priority

When multiple skills apply:
1. **Process skills first** (brainstorming, debugging) - determine HOW to approach
2. **Implementation skills second** (frontend-design, mcp-builder) - guide execution

Examples:
- "Let's build X" → brainstorming first, then implementation skills
- "Fix this bug" → debugging first, then domain-specific skills

### Skill Types

- **Rigid** (TDD, debugging): Follow exactly. Don't adapt away discipline.
- **Flexible** (patterns): Adapt principles to context.

The skill itself declares which type it is.

### Platform Adaptation

**Claude Code:** `Skill` tool - load and present content, follow directly
**Gemini CLI:** `activate_skill` tool - load metadata at session start, full content on demand
**Other environments:** Check platform documentation

Cross-platform tool mappings:
- Codex: `references/codex-tools.md`
- Gemini CLI: `references/gemini-tools.md`

### Key Insights from Reference Files

#### Codex Tool Mapping (`codex-tools.md`)

Notable limitations and workarounds:

1. **No named agent registry:** Claude Code skills reference named agent types like `superpowers:code-reviewer`. Codex `spawn_agent` creates generic agents. Workaround: Find the agent's prompt file, read content, fill placeholders, spawn worker with that content.

2. **Multi-agent support requires opt-in:** Codex config needs:
   ```toml
   [features]
   multi_agent = true
   ```

3. **Message framing:** Use task-delegation framing ("Your task is...") not persona framing ("You are..."). Wrap in XML tags. End with execution directive.

4. **Environment detection:** Codex sandbox may block branch/push operations. Fallback: commit all work, inform user to use App's native controls.

#### Gemini CLI Tool Mapping (`gemini-tools.md`)

Notable limitations:

1. **No subagent support:** Skills relying on `Task` tool (subagent-driven-development, dispatching-parallel-agents) fall back to single-session execution via `executing-plans`.

2. **Additional Gemini tools:** These have no Claude Code equivalent:
   - `list_directory` - list files
   - `save_memory` - persist facts to GEMINI.md
   - `ask_user` - request structured input
   - `tracker_create_task` - rich task management
   - `enter_plan_mode` / `exit_plan_mode` - read-only research mode

### Clever Solutions / Design Patterns

1. **1% rule:** "Even a 1% chance a skill might apply means you should invoke the skill." This is a clever threshold that prevents rationalization - if you're wondering whether to check, you should check.

2. **Red flag table:** The rationalization detection table is psychologically sophisticated - it recognizes that agents (like humans) develop rationalizations to avoid discipline.

3. **Process-first priority:** The skill priority hierarchy (process skills before implementation skills) ensures agents don't rush to solutions before understanding the problem.

4. **Rigid vs Flexible distinction:** Not all skills require the same adherence level. This prevents over-engineering when flexibility is appropriate.

5. **SUBAGENT-STOP:** The skill explicitly stops itself when dispatched as a subagent - prevents meta-invocation loops.

6. **Platform-specific documentation:** References directory keeps platform adaptation info separate, avoiding clutter in the main skill.

### Technical Debt / Shortcuts

- Pure declarative markdown - no executable code
- DOT diagram is documentation, not rendered (would need graphviz to view)
- Codex workaround for named agents is acknowledged as technical debt (can be removed when `RawPluginManifest` gains `agents` field)
- Gemini CLI subagent fallback is a known limitation

### Integration Points

- **Hook system:** `hooks/session-start` invokes this skill at session initialization
- **Platform plugins:** Each platform's plugin configuration loads this skill
- **Other skills:** All other skills build on the framework established here

### Relationship to Other Features

- All other skills assume this framework is loaded
- `requesting-code-review` pairs with this (reviewer vs reviewee perspective)
- The `code-reviewer.md` agent prompt in `agents/` is referenced by skills that dispatch code review

---

## Cross-Feature Analysis

### Complementary Design

Features 13 and 14 are thematically linked:
- **Using Superpowers:** Establishes HOW to use the skill system
- **Receiving Code Review:** Demonstrates the application of those skills with a specific, complex workflow

Both share:
- Emphasis on verification before action
- Anti-pattern awareness (forbidden responses, red flags)
- Technical rigor over performative behavior
- Platform adaptation considerations

### Shared Patterns

1. **Tables over prose:** Both use tables for quick reference during execution
2. **Explicit anti-patterns:** Forbidden responses vs Red flags - both prevent rationalization
3. **User authority embedded:** Both reference "your human partner's" decisions explicitly
4. **Compact implementation:** Both are lightweight markdown files with high information density

### Interesting Observations

1. **Self-referential skill:** Using-superpowers is invoked to explain how to invoke skills - there's a recursive quality that requires careful framing (hence the SUBAGENT-STOP)

2. **Cultural awareness:** "Strange things are afoot at the Circle K" in receiving-code-review and the general tone suggests these skills were designed for a specific cultural context (nerd/geek culture)

3. **Anti-social skills:** Both skills explicitly push back against "natural" social behaviors (performative agreement, starting with exploration). This suggests the human partner values technical rigor over social harmony.

4. **Explicit hierarchy:** The instruction priority hierarchy (User > Skills > Default) is stated explicitly, which is unusual - most systems leave this implicit or buried in documentation.

---

## Summary

| Aspect | Receiving Code Review | Using Superpowers |
|--------|----------------------|-------------------|
| **Type** | Workflow skill | Entry point/meta skill |
| **Lines** | ~214 | ~116 |
| **Complexity** | Medium | Low |
| **Dependencies** | None (standalone) | Depends on session-start hook |
| **Test coverage** | N/A (skill doc) | N/A (skill doc) |
| **Platform-specific** | No | Yes (reference files) |
| **Key insight** | Verify before implementing | 1% rule for skill invocation |
