# Deep Dive: Features Batch 1

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Batch:** 1 of N
**Features:** Office Hours, Plan CEO Review, Plan Engineering Review
**Generated:** 2026-03-27

---

## Executive Summary

Batch 1 features form a **three-stage thinking pipeline**:

```
Idea/Problem ──► /office-hours ──► Design Doc ──► /plan-ceo-review ──► Scope Decision ──► /plan-eng-review ──► Implementation Plan
                 (Problem understanding)          (Strategy & scope)                        (Architecture & tests)
```

All three share:
- **Tier 3 preamble infrastructure** (shared preamble resolver)
- **SPEC_REVIEW_LOOP** placeholder (shared spec review via subagent)
- **REVIEW_DASHBOARD** placeholder (shared review readiness tracking)
- **Same telemetry system** (skill-usage.jsonl, analytics)
- **Same AskUserQuestion format** (reground, simplify, recommend, options)
- **Same BROWSE_SETUP** (headless browser for visual sketches in office-hours)

---

## Feature 1: Office Hours

### Purpose
YC-style startup diagnostic session. Two modes:
- **Startup mode**: Six forcing questions that expose demand reality, status quo, desperate specificity, narrowest wedge, observation, and future-fit
- **Builder mode**: Design thinking brainstorming for side projects, hackathons, learning, open source

### Command
`/office-hours`

### Files
- `office-hours/SKILL.md` (1446 lines — auto-generated)
- `office-hours/SKILL.md.tmpl` (source template)
- Design docs written to `~/.gstack/projects/{slug}/`

### Core Flow

**Phase 1: Context Gathering**
- Reads CLAUDE.md, TODOS.md
- Runs git log and git diff to understand recent context
- Maps relevant codebase areas via Grep/Glob
- Lists existing design docs for cross-pollination
- Asks user goal: startup, intrapreneurship, hackathon, open source, learning, or having fun

**Phase 2A: Startup Mode — Six Forcing Questions**
Smart routing based on product stage:
- Pre-product → Q1, Q2, Q3
- Has users → Q2, Q4, Q5
- Has paying customers → Q4, Q5, Q6
- Pure engineering/infra → Q2, Q4 only

Q1: **Demand Reality** — "What's the strongest evidence someone actually wants this?"
- Push until: specific behavior, payment, building workflow around it
- Red flags: interest without behavior, waitlists, VCs excited

Q2: **Status Quo** — "What are users doing now to solve this?"
- Push until: specific workflow, hours spent, dollars wasted
- Red flags: "nothing exists" usually means not painful enough

Q3: **Desperate Specificity** — "Name the actual human who needs this most"
- Push until: name, role, specific consequence
- Red flags: category-level answers like "healthcare enterprises"

Q4: **Narrowest Wedge** — "Smallest version someone would pay for this week?"
- Push until: one feature, one workflow, ship-in-days
- Red flags: "need full platform first"

Q5: **Observation & Surprise** — "Watched someone use it? What surprised you?"
- Push until: specific surprise that contradicted assumptions
- Red flags: surveys, demos, "as expected"

Q6: **Future-Fit** — "Does this become more or less essential in 3 years?"
- Push until: specific claim about user world change
- Red flags: rising tide arguments every competitor can make

**Phase 2B: Builder Mode**
- Generative questions (not interrogative)
- What's the coolest version? Who would you show this to?
- Fastest path to something shareable?

**Phase 2.5: Related Design Discovery**
- Greps existing design docs for keyword overlap
- Offers to build on prior designs

**Phase 2.75: Landscape Awareness**
- Web search for conventional wisdom in the space
- Three-layer synthesis: Layer 1 (tried-and-true), Layer 2 (new/popular), Layer 3 (first principles)
- Eureka moment detection and logging

**Phase 3: Premise Challenge**
- Is this the right problem?
- What if we do nothing?
- What existing code partially solves this?
- Distribution plan (how will users get the artifact?)

**Phase 3.5: Cross-Model Second Opinion (Optional)**
- Runs Codex cold read if available
- Gets independent perspective on problem statement

**Phase 4: Alternatives Generation (MANDATORY)**
- 2-3 approaches required
- Must include: minimal viable AND ideal architecture
- User approves approach before proceeding

**Design Sketch Phase (UI only)**
- If UI involved: generates wireframe HTML
- Renders via headless browser, takes screenshot
- Iterates on layout

**Phase 5: Design Doc**
- Writes to `~/.gstack/projects/{slug}/{user}-{branch}-design-{datetime}.md`
- Design lineage tracking (Supersedes field for prior docs)
- Startup mode template vs Builder mode template

**Phase 6: Handoff**
Three-beat closing:
1. **Signal reflection** — quotes user's own words back, connects to "Golden Age" framing
2. **"One more thing."** — genre shift separator
3. **Garry's Personal Plea** — YC application push (tiered by founder signals)

### Output Artifacts
- Design doc saved to `~/.gstack/projects/{slug}/`
- Analytics logged to `~/.gstack/analytics/skill-usage.jsonl`
- Eureka moments logged to `~/.gstack/analytics/eureka.jsonl`
- Spec review metrics to `~/.gstack/analytics/spec-review.jsonl`

### Clever Solutions

1. **Smart-skip routing** — doesn't ask all 6 questions, routes based on product stage
2. **Anti-sycophancy rules** — explicit list of phrases to NEVER say ("That's interesting", "There are many ways to think about this")
3. **Pushback patterns** — detailed examples showing good vs bad pushback
4. **Founder signal synthesis** — tracks 8 founder qualities, uses count to tier the closing
5. **Design lineage** — Supersedes field creates revision chain across sessions
6. **Spec review loop** — subagent adversarial review with convergence guard (max 3 iterations)

### Technical Debt / Shortcuts

1. **No hard state between questions** — if user leaves mid-session, context is lost
2. **Design doc stored in home dir** — not in project repo (by design, but makes sharing harder)
3. **WebSearch privacy gate** — asks permission before searching, but the privacy implications of sending problem space terms are not fully surfaced
4. **Phase 2.75 always runs** — even for very specific technical problems where landscape search is low-value

### Hard Gate
"**HARD GATE:** Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action. Your only output is a design document."

---

## Feature 2: Plan CEO Review

### Purpose
CEO/founder-mode plan review. Rethinks the problem to find the 10-star product. Four modes:
- **SCOPE EXPANSION**: Dream big, push scope UP
- **SELECTIVE EXPANSION**: Hold baseline, cherry-pick expansions
- **HOLD SCOPE**: Make it bulletproof
- **SCOPE REDUCTION**: Find minimum viable version

### Command
`/plan-ceo-review`

### Dependencies
- `benefits-from: [office-hours]` — offers to run office-hours inline if no design doc found

### Files
- `plan-ceo-review/SKILL.md` (1446 lines — auto-generated)
- `plan-ceo-review/SKILL.md.tmpl`
- CEO plans written to `~/.gstack/projects/{slug}/ceo-plans/`
- Handoff notes written to `~/.gstack/projects/{slug}/`

### Core Flow

**Pre-Review System Audit**
- Runs git log, git diff, git stash list
- Greps for TODO/FIXME/HACK/XXX
- Reads CLAUDE.md, TODOS.md
- Checks for design doc from /office-hours
- Checks for CEO handoff notes from prior sessions

**Step 0: Nuclear Scope Challenge + Mode Selection**

0A: **Premise Challenge** — Is this the right problem? Could different framing yield simpler solution? What's actual user outcome?

0B: **Existing Code Leverage** — What already partially/fully solves each sub-problem? Can we capture outputs from existing flows?

0C: **Dream State Mapping** — Current → This Plan → 12-month ideal

0C-bis: **Implementation Alternatives (MANDATORY)** — 2-3 approaches with effort/risk/pros/cons
- One must be minimal viable
- One must be ideal architecture
- One can be creative/lateral

0D: **Mode-Specific Analysis**
- EXPANSION: 10x check, platonic ideal, delight opportunities, expansion opt-in ceremony
- SELECTIVE EXPANSION: Hold scope first, then surface cherry-picks
- HOLD SCOPE: Complexity check, minimum set
- REDUCTION: Ruthless cut to absolute minimum

0D-POST: **Persist CEO Plan** (EXPANSION/SELECTIVE only)
- Writes to `~/.gstack/projects/$SLUG/ceo-plans/{date}-{feature-slug}.md`
- Includes vision, scope decisions, accepted/deferred proposals
- Spec review loop on the CEO plan itself

0E: **Temporal Interrogation** — What decisions need to be made NOW?

0F: **Mode Selection** — User picks one of four modes

**Review Sections (10 sections)**

1. **Architecture Review** — System design, data flow (all 4 paths), state machines, coupling, scaling, SPOFs, security, rollback
2. **Error & Rescue Map** — Every method that can fail, exception class, rescued status, rescue action, user impact
3. **Security & Threat Model** — Attack surface, input validation, authorization, secrets, dependencies
4. **Data Flow & Interaction Edge Cases** — All shadow paths (nil, empty, error), interaction edge cases
5. **Code Quality Review** — DRY violations, naming, error handling, complexity
6. **Test Review** — 100% coverage goal, ASCII coverage diagram, E2E decision matrix
7. **Performance Review** — N+1 queries, memory, indexes, caching, slow paths
8. **Observability & Debuggability** — Logging, metrics, tracing, alerts, dashboards
9. **Deployment & Rollout** — Migrations, feature flags, rollback plan
10. **Long-Term Trajectory** — Technical debt, path dependency, reversibility
11. **Design & UX Review** (if UI scope detected) — Information architecture, interaction states, AI slop risk

**Outside Voice (Optional)**
- Runs Codex plan review or falls back to Claude subagent
- Cross-model tension detection and logging

### Output Artifacts
- CEO plan doc to `~/.gstack/projects/{slug}/ceo-plans/`
- Handoff notes to `~/.gstack/projects/{slug}/` (if session paused for /office-hours)
- Review log to `~/.gstack/analytics/`
- Completion summary table

### Required Outputs
- "NOT in scope" section (explicitly deferred items)
- "What already exists" section
- "Dream state delta" section
- Error & Rescue Registry
- Failure Modes Registry
- TODOS.md updates (one per AskUserQuestion)
- Scope Expansion Decisions (EXPANSION/SELECTIVE)
- 6 ASCII diagrams mandatory
- Stale Diagram Audit

### Clever Solutions

1. **CEO Plan persistence** — Vision and scope decisions survive beyond conversation
2. **Spec review loop on CEO plan** — Subagent adversarial review before presenting
3. **Four-mode framework** — Clear decision tree for different situations
4. **"Prime Directives"** — Zero silent failures, every error has a name, data flows have shadow paths
5. **Mode quick reference table** — 8-column comparison for fast lookup
6. **Handoff note system** — If CEO review pauses to run /office-hours, picks up where left off
7. **Cross-model tension tracking** — Logs where Claude and Codex disagree

### Technical Debt / Shortcuts

1. **Complexity check threshold** — "8 files or 2 new classes" threshold is arbitrary
2. **No automatic mode selection** — User must pick, but defaults are provided per context type
3. **10 review sections is aggressive** — Can overwhelm users, but "never skip" directive for highest-leverage outputs
4. **Manual diagram maintenance** — "Stale diagrams are worse than none" but no automatic detection

### Cognitive Patterns Referenced
- Classification instinct (Bezos one-way/two-way doors)
- Paranoid scanning (Horowitz)
- Inversion reflex (Munger)
- Focus as subtraction (Jobs)
- Speed calibration (Bezos 70% info)
- Founder-mode bias (Chesky/Graham)
- Willfulness as strategy (Altman)
- Edge case paranoia (design)

---

## Feature 3: Plan Engineering Review

### Purpose
Engineering manager-mode plan review. Locks architecture, data flow, edge cases, and tests. Forces hidden assumptions into the open with ASCII diagrams.

### Command
`/plan-eng-review`

### Dependencies
- `benefits-from: [office-hours]` — offers to run office-hours inline if no design doc found

### Files
- `plan-eng-review/SKILL.md` (1002 lines — auto-generated)
- `plan-eng-review/SKILL.md.tmpl`
- Test plan artifact to `~/.gstack/projects/{slug}/`

### Core Flow

**Design Doc Check**
- Looks for `~/.gstack/projects/$SLUG/*-$BRANCH-design-*.md`
- If found, uses as source of truth
- If not, offers to run /office-hours inline

**Step 0: Scope Challenge**
1. What existing code already solves sub-problems?
2. Minimum set of changes to achieve goal?
3. Complexity check (8+ files or 2+ new classes = smell)
4. Search check (Layer 1/2/3 for each pattern)
5. TODOS cross-reference
6. Completeness check (lake vs ocean)
7. Distribution check (CI/CD for new artifacts)

**Review Sections (4 sections)**

1. **Architecture Review** — System design, dependency graph, data flow, scaling, security, ASCII diagrams, distribution architecture

2. **Code Quality Review** — DRY violations, error handling, technical debt, over/under-engineering, existing ASCII diagram accuracy

3. **Test Review** (highest detail)
   - **Test Framework Detection** — Auto-detects runtime and test framework
   - **Codepath Tracing** — Read plan, trace data through every branch
   - **User Flow Mapping** — Real user interactions and edge cases
   - **Coverage Diagram** — ASCII diagram showing tested/untested paths
   - **E2E Decision Matrix** — When to recommend E2E vs unit vs eval
   - **Regression Rule** — CRITICAL: regressions must have regression tests
   - **Test Plan Artifact** — Written to `~/.gstack/projects/{slug}/` for /qa consumption

4. **Performance Review** — N+1, memory, caching, slow paths, connection pool pressure

**Outside Voice (Optional)**
Same as Plan CEO Review.

### Output Artifacts
- Test plan artifact to `~/.gstack/projects/{slug}/{user}-{branch}-eng-review-test-plan-{datetime}.md`
- Consumed by `/qa` and `/qa-only` as primary test input
- Review log to `~/.gstack/analytics/`

### Required Outputs
- "NOT in scope" section
- "What already exists" section
- TODOS.md updates (one per AskUserQuestion)
- ASCII diagrams (in plan and in code comments)
- Failure modes registry
- Completion summary

### Clever Solutions

1. **Test coverage diagram format** — Shows code path coverage AND user flow coverage in same diagram
2. **Quality scoring rubric** — ★★★ (edge cases + error paths) vs ★★ (happy path) vs ★ (smoke test)
3. **E2E decision matrix** — Explicit criteria for when E2E is right tool vs unit
4. **Regression rule with IRON RULE** — Explicitly highest priority test type
5. **Test plan artifact** — Consumed by downstream /qa skill, not just documentation
6. **"Tests are cheapest lake to boil"** — Explicit anti-deferral argument

### Technical Debt / Shortcuts

1. **Auto-detect test framework** — Could be wrong for monorepos with multiple test frameworks
2. **No mock/fixture management** — Test review assumes mocks are used appropriately but doesn't audit
3. **"Two-week smell test"** — Only mentioned in cognitive patterns, not enforced
4. **Coverage diagram is aspirational** — No automated verification that diagram matches actual code

### Cognitive Patterns Referenced
- State diagnosis (Larson: falling behind, treading water, repaying debt, innovating)
- Blast radius instinct
- Boring by default (Choose Boring Technology)
- Incremental over revolutionary (strangler fig)
- Systems over heroes
- Reversibility preference
- Failure is information
- DX is product quality
- Essential vs accidental complexity (Brooks)
- Own your code in production

---

## Shared Infrastructure Analysis

### Template System

All three SKILL.md files are **auto-generated** from `.tmpl` templates via `bun run gen:skill-docs`. The template system uses placeholders:

```
{{PREAMBLE}}           → Tier 3 preamble (shared across all three)
{{BROWSE_SETUP}}      → Headless browser setup
{{SLUG_EVAL}}         → Project slug detection
{{SLUG_SETUP}}        → Project slug setup for writing
{{CODEX_SECOND_OPINION}} → Optional Codex cold read
{{DESIGN_SKETCH}}     → Wireframe generation (office-hours only)
{{SPEC_REVIEW_LOOP}}  → Subagent adversarial review loop
```

Resolvers are in `scripts/resolvers/`:
- `preamble.ts` — Tiered preamble (T1-T4), upgrade check, lake intro, telemetry, contributor mode
- `review.ts` — Review dashboard, plan file report, spec review loop, benefits from
- `browse.ts` — Command reference, snapshot flags, browse setup
- `design.ts` — Design methodology, hard rules, outside voices, sketch
- `testing.ts` — Test bootstrap, coverage audit
- `utility.ts` — Slug eval/setup, base branch detect, deploy bootstrap, QA methodology
- `constants.ts` — Codex error handling

### Preamble Tier System

Skills are classified by tier (1-4) determining which preamble sections they get:

| Tier | Skills | Sections |
|------|--------|----------|
| T1 | browse, setup-cookies, benchmark | core + upgrade + lake + telemetry + contributor + completion |
| T2 | investigate, cso, retro, doc-release, setup-deploy, canary | T1 + ask format + completeness |
| T3 | autoplan, codex, design-consult, office-hours, ceo/design/eng-review | T2 + repo mode + search before building |
| T4 | ship, review, qa, qa-only, design-review, land-deploy | T3 + test failure triage |

Batch 1 features are all **Tier 3**.

### AskUserQuestion Format (Shared)

All three features use the same 4-part format:
1. **Re-ground** — Project, branch, current task
2. **Simplify** — Plain English, concrete examples
3. **Recommend** — Recommendation with `Completeness: X/10`
4. **Options** — Lettered options with effort scales `(human: ~X / CC: ~Y)`

### Review Dashboard (Shared)

All review skills use the same dashboard format showing:
- Runs count
- Last run timestamp
- Status (CLEAR / issues_open)
- Whether required

Eng Review is the **only required gate** for shipping. CEO Review and Design Review are optional.

### Telemetry System (Shared)

All skills write to `~/.gstack/analytics/`:
- `skill-usage.jsonl` — Skill name, timestamp, repo
- `.pending-*` — Pending finalization
- `eureka.jsonl` — First-principles insights
- `spec-review.jsonl` — Spec review metrics
- `gstack-telemetry-log` — Binary for logging events

### Design Doc Pipeline

```
office-hours → writes design doc → plan-ceo-review reads design doc
                              ↓
              plan-eng-review reads design doc
                              ↓
                   /qa reads eng-review test plan
```

The design doc at `~/.gstack/projects/{slug}/` is the connective tissue.

---

## Cross-Feature Observations

### What's Clever

1. **Skill chaining via design docs** — Office hours output becomes CEO review input, CEO review output becomes eng review input. No hard coupling, just shared files.

2. **Prerequisite skill offer** — Both CEO and Eng review offer to run /office-hours inline if no design doc found. Don't block on missing context, generate it.

3. **Spec review loop** — All three features use the same subagent adversarial review pattern with convergence guard.

4. **"Boil the Lake" philosophy** — Explicitly says AI makes completeness near-free, so always recommend complete over shortcut.

5. **Founder signal tracking** — Office hours tracks 8 signals, uses count to tier the YC pitch. Quantitative.

6. **Mode quick reference** — 8-column table makes it easy to remember what each mode does.

7. **Cross-model consensus** — Codex + Claude disagreement is tracked as "cross-model tension" and auto-proposed as TODOs.

8. **Stale diagram audit** — Explicit instruction to check if ASCII diagrams in code comments are still accurate after changes.

### Technical Debt

1. **No validation of design doc quality** — Skills trust design docs from office-hours but don't validate they're complete.

2. **Handoff notes are fragile** — Stored as files in home dir, but no guarantee they'll be found on resume.

3. **No automatic mode selection** — Despite having clear defaults per context type, user must manually select.

4. **Test coverage diagrams are manual** — No automated verification that diagrams match code.

5. **Analytics are fire-and-forget** — Written to jsonl files, but no consumption/processing in the skills themselves.

6. **Smart-skip routing is implicit** — The routing logic for which questions to ask is in prose, not code.

### Edge Cases to Watch

1. **Mid-session resume** — CEO review can pause to run /office-hours, but what if user has multiple pauses?

2. **Design doc conflicts** — If multiple office-hours sessions on same branch produce different design docs, which wins?

3. **Scope creep in SELECTIVE EXPANSION** — Cherry-pick ceremony could still result in too much scope.

4. **Completeness bias** — "Boil the Lake" could lead to over-engineering trivial features.

5. **YC pitch fatigue** — If user has run office-hours before, the Garry plea might feel repetitive.

---

## Dependencies Between Features

```
office-hours (v2.0.0)
    │
    ├── Produces: ~/.gstack/projects/{slug}/*-design-*.md
    │
    └── Used by:
            ├── plan-ceo-review (offers to run inline)
            └── plan-eng-review (offers to run inline)

plan-ceo-review (v1.0.0)
    │
    ├── Benefits from: office-hours
    ├── Reads: design doc from office-hours
    ├── Produces: ~/.gstack/projects/{slug}/ceo-plans/*.md
    └── Used by: plan-eng-review (for scope decisions)

plan-eng-review (v1.0.0)
    │
    ├── Benefits from: office-hours
    ├── Reads: design doc from office-hours
    ├── Reads: CEO plan from plan-ceo-review (if exists)
    └── Produces: ~/.gstack/projects/{slug}/*-eng-review-test-plan-*.md
```

---

## Version History

| Skill | Version | Notable Changes |
|-------|---------|-----------------|
| office-hours | 2.0.0 | Major rewrite (likely added Builder mode, Phase 2.75, spec review loop) |
| plan-ceo-review | 1.0.0 | Initial release |
| plan-eng-review | 1.0.0 | Initial release |

---

## Verification Notes

Since these are prompt-based skills (no executable code), verification means:
- SKILL.md parses correctly (validated by `bun run gen:skill-docs`)
- Template placeholders resolve correctly
- No broken bash syntax in preamble blocks
- All required outputs are specified

The actual "verification" happens when Claude Code executes the skill prompts against real projects.
