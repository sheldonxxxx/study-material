# Features Batch 6: Land and Deploy, Autoplan, Document Release

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Date:** 2026-03-27
**Batch:** 6 of ~10

---

## Feature 1: Land and Deploy

**Command:** `/land-and-deploy`
**SKILL.md:** `land-and-deploy/SKILL.md` (preamble-tier 4, largest skill at 925 lines)
**Tier:** Priority Tier 3 - Infrastructure & Power Tools

### Core Purpose

Release engineer single command. Merges PR, waits for CI and deploy, verifies production health via canary checks. One command from "approved" to "verified in production."

**Workflow chain:** `/ship` creates the PR -> `/land-and-deploy` merges, deploys, and verifies.

### Feature Flow

```
User invokes /land-and-deploy
        |
        v
Step 1: Pre-flight (auth check, parse args, detect PR)
        |
        v
Step 2: Pre-merge checks (CI status, merge conflicts)
        |
        v
Step 3: Wait for CI (if pending, 15-min timeout)
        |
        v
Step 3.5: Pre-merge readiness gate (reviews staleness, test results, doc check)
        |----BLOCKER----> Stop if failing tests or critical warnings
        |
        v
Step 4: Merge PR (--auto or --squash --delete-branch)
        |
        v
Step 5: Deploy strategy detection (Fly.io, Render, Vercel, Netlify, Heroku, GitHub Actions)
        |
        v
Step 6: Wait for deploy (platform-specific polling, 20-min timeout)
        |
        v
Step 7: Canary verification (conditional depth based on diff-scope)
        |
        v
Step 8: Revert (if needed - git revert on base branch)
        |
        v
Step 9: Deploy report (ASCII summary + JSONL log)
        |
        v
Step 10: Suggest follow-ups (/canary, /benchmark, /document-release)
```

### Key Implementation Details

**Arguments:**
- `/land-and-deploy` - auto-detect PR from current branch
- `/land-and-deploy <url>` - auto-detect PR, verify at this URL
- `/land-and-deploy #123` - specific PR number
- `/land-and-deploy #123 <url>` - specific PR + URL

**Pre-merge readiness gate (Step 3.5)** is the ONE critical safety check before irreversible merge:
- Reviews staleness: compares review commit vs current HEAD, flags if 4+ commits since review
- Free tests: runs `bun test` immediately
- E2E tests: checks recent eval results from `~/.gstack-dev/evals/`
- LLM judge evals: checks for today's results
- PR body accuracy: compares against actual commits
- Document-release check: warns if docs not updated despite new features
- Produces ASCII box report with WARNINGS/BLOCKERS counts

**Deploy strategy detection:**
1. Check persisted config in CLAUDE.md (`## Deploy Configuration`)
2. Auto-detect from config files: `fly.toml`, `render.yaml`, `vercel.json`, `netlify.toml`, `Procfile`, `railway.json`
3. GitHub Actions workflows in `.github/workflows/`
4. `gstack-diff-scope` classifies: FRONTEND, BACKEND, DOCS, CONFIG

**Canary verification depth (conditional):**
| Diff Scope | Depth |
|------------|-------|
| SCOPE_DOCS only | Skip entirely |
| SCOPE_CONFIG only | Smoke: `$B goto` + 200 status |
| SCOPE_BACKEND only | Console errors + perf check |
| SCOPE_FRONTEND (any) | Full: console + perf + screenshot |
| Mixed | Full canary |

**Platform handling:**
- GitHub Actions: poll `gh run list` matching merge commit SHA
- Fly.io: `fly status --app {app}`
- Render: poll production URL until it responds (2-5 min typical)
- Vercel/Netlify: auto-deploy, wait 60s then canary
- Heroku: `heroku releases --app {app} -n 1`

### Clever Solutions

1. **Review staleness check:** Compares stored review commit against HEAD to detect "review done on different code than what's about to merge" - a subtle but important guard against stale reviews.

2. **Diff-scope-driven canary depth:** Skipping expensive browser checks for docs-only PRs, minimal smoke for config-only, full suite for frontend changes. Avoids over-verifying trivial changes.

3. **Merge queue handling:** `gh pr merge --auto` is used first (respects repo settings), with polling for state=MERGED. Handles both fast-forward and queue-based merges.

4. **Branch deletion:** `--delete-branch` cleans up feature branch automatically after merge.

5. **Deploy config persistence:** Writes to CLAUDE.md so future runs skip detection entirely.

6. **Deploy report with timing breakdown:** CI wait, queue, deploy, canary durations tracked separately for debugging slow deploys.

### Technical Debt / Shortcuts

1. **GitLab explicitly unsupported:** Step 0 explicitly stops with "GitLab support for /land-and-deploy is not yet implemented." This is a known limitation, not deferred.

2. **No retry on transient failures:** Deploy polling fails if platform API is temporarily unavailable. No exponential backoff or retry logic.

3. **Merge conflict resolution not automated:** Stops at "PR has merge conflicts. Resolve them and push before landing." - human must fix.

4. **Merge queue timeout (30 min) is long:** Could be reduced for non-merge-queue scenarios.

5. **No auto-detected deploy timeout per-platform:** Uses generic 20-minute timeout even though Fly.io and Render typically deploy in 2-5 minutes.

6. **Revert uses `--no-edit`:** `git revert --no-edit` works but doesn't provide a revert PR title/description. Could be more descriptive.

### Error Handling

- **Auth failure:** `gh auth status` checked upfront - stops if not authenticated
- **No PR found:** Detects from current branch or `#NNN` argument
- **Already merged/closed:** Validates PR state before attempting merge
- **CI failures:** Blocks merge entirely with failure list
- **Merge conflicts:** Blocks merge with clear message
- **Permission denied:** Stops with "ask a maintainer" message
- **Merge queue removal:** If PR goes back to OPEN state, stops immediately
- **Deploy failure:** Offers revert as escape hatch
- **Canary alert:** Offers investigate/revert/dismiss based on severity

### Interaction with Other Skills

- **Depends on `/setup-deploy`:** First run triggers config detection and CLAUDE.md persistence
- **Uses `/canary`:** Step 7 runs canary verification using `$B` (browse daemon)
- **Uses review dashboard:** Step 3.5 reads review logs via `gstack-review-read`
- **Suggests `/document-release`:** In Step 10 follow-ups

---

## Feature 2: Autoplan

**Command:** `/autoplan`
**SKILL.md:** `autoplan/SKILL.md` (preamble-tier 3, 977 lines - the largest skill)
**Tier:** Priority Tier 3 - Infrastructure & Power Tools

### Core Purpose

Automated review pipeline. Runs CEO, design, and eng review automatically with encoded decision principles. Surfaces only taste decisions for approval. One command, fully reviewed plan out.

**Key insight:** Reads the full skill files from disk and follows them at full depth - same rigor as running each skill manually, but auto-deciding intermediate questions.

### Feature Flow

```
User invokes /autoplan
        |
        v
Phase 0: Intake + Restore Point
        - Capture restore point (plan file backed up to ~/.gstack/projects/$SLUG/)
        - Read CLAUDE.md, TODOS.md, git log -30, git diff --stat
        - Detect UI scope (grep for view/rendering terms)
        - Load skill files from disk: plan-ceo-review, plan-design-review (if UI), plan-eng-review
        |
        v
Phase 1: CEO Review (Strategy & Scope)
        - Runs ALL sections from plan-ceo-review/SKILL.md at full depth
        - Dual voices: Claude subagent + Codex run simultaneously
        - Auto-decides every AskUserQuestion using 6 principles
        - GATE: Presents premises to user for confirmation (the ONE human decision)
        |
        v
Phase 2: Design Review (conditional - only if UI scope detected)
        - Runs ALL sections from plan-design-review/SKILL.md
        - Dual voices: Claude subagent + Codex simultaneously
        - Auto-decides using 6 principles
        |
        v
Phase 3: Eng Review + Dual Voices
        - Runs ALL sections from plan-eng-review/SKILL.md
        - Dual voices: Claude subagent + Codex simultaneously
        - Produces test plan artifact at ~/.gstack/projects/$SLUG/
        - Writes TODOS.md updates
        |
        v
Phase 4: Final Approval Gate
        - Presents summary: N taste decisions, M auto-decided
        - User chooses: Approve / Approve with overrides / Interrogate / Revise / Reject
        |
        v
Write Review Logs (gstack-review-log)
```

### The 6 Decision Principles

1. **Choose completeness** - Ship the whole thing. Pick the approach covering more edge cases.
2. **Boil lakes** - Fix everything in blast radius (< 1 day CC effort)
3. **Pragmatic** - Pick the cleaner solution when two fix the same thing
4. **DRY** - Reject duplicates, reuse existing
5. **Explicit over clever** - 10-line obvious fix > 200-line abstraction
6. **Bias toward action** - Merge > review cycles > stale deliberation

**Conflict resolution by phase:**
- CEO: P1 (completeness) + P2 (boil lakes) dominate
- Eng: P5 (explicit) + P3 (pragmatic) dominate
- Design: P5 (explicit) + P1 (completeness) dominate

### Decision Classification

**Mechanical decisions:** One clearly right answer. Auto-decide silently.
- Example: run codex (always yes), run evals (always yes)

**Taste decisions:** Reasonable people could disagree. Auto-decide with recommendation, surface at gate:
1. Close approaches - top two both viable with different tradeoffs
2. Borderline scope - in blast radius but 3-5 files
3. Codex disagreements - codex recommends differently

### Dual Voices Architecture

For each phase, runs TWO independent evaluators simultaneously:

**Claude subagent (via Agent tool):**
- Independent Claude instance with NO prior-phase context
- Evaluates same dimensions as Codex
- Produces independent findings

**Codex (via Bash):**
```
codex exec "<phase-specific prompt>" -C "$(git rev-parse --show-toplevel)" -s read-only --enable web_search_cached
```

**Degradation matrix:**
- Both fail -> "single-reviewer mode" warning
- Codex only -> tag `[codex-only]`
- Subagent only -> tag `[subagent-only]`
- Both succeed -> consensus table with CONFIRMED/DISAGREE columns

### Pre-Gate Verification Checklist

Before presenting the Final Approval Gate, verifies ALL required outputs were produced:
- CEO: premise challenge, error & rescue registry, failure modes registry, "NOT in scope", "What already exists", dream state delta, completion summary, dual voices, consensus table
- Design (if run): all 7 dimensions rated, issues auto-decided, dual voices, litmus scorecard
- Eng: scope challenge with code analysis, ASCII diagram, test diagram, test plan artifact on disk, "NOT in scope", failure modes registry, completion summary, dual voices, consensus table
- Cross-phase: themes section
- Audit trail: at least one row per auto-decision

### Clever Solutions

1. **Restore point capture:** Before doing anything, backs up plan file with re-run instructions. If autoplan crashes or user interrupts, they can restore and continue.

2. **Skill file loading from disk:** Instead of embedding review logic, reads actual skill files. This means any updates to plan-ceo-review etc. are automatically incorporated into autoplan.

3. **Sequential phases that build:** CEO -> Design -> Eng, each phase's outputs inform the next. CEO establishes mode (selective expansion, hold, etc.) which guides Design. Design informs Eng's focus areas.

4. **Decision audit trail:** Every auto-decision logged to plan file incrementally via Edit. Keeps audit on disk, not accumulated in context.

5. **Pre-gate verification checklist:** Forces completion of ALL required outputs before presenting gate. Max 2 retry attempts to prevent infinite loops.

6. **Cross-phase theme detection:** "For any concern that appeared in 2+ phases' dual voices independently" - high-confidence signal that gets its own section.

7. **Test plan artifact on disk:** Written to `~/.gstack/projects/$SLUG/{user}-{branch}-test-plan-{datetime}.md` - separate from plan file to avoid bloat.

8. **Context-dependent principle dominance:** Different principles win in different phases, reflecting that CEO cares about scope, Eng cares about clarity, Design cares about aesthetics.

### Technical Debt / Shortcuts

1. **Skip list for loaded skills:** "SKIP these sections (they are already handled by /autoplan)" - but this list could drift out of sync if the parent skills change.

2. **No auto-rerun on phase failure:** If Phase 1 CEO fails catastrophically, no built-in retry. Max 2 attempts only applies to missing outputs.

3. **Agent tool + Codex simultaneously:** This uses TWO expensive LLM calls per phase. Could be expensive for large codebases.

4. **No cancellation support:** If user approves taste decisions and then changes mind during Phase 2, no clean exit. Must wait for completion or interrupt.

5. **Test plan file path complexity:** `~/.gstack/projects/$SLUG/{user}-{branch}-test-plan-{datetime}.md` - user and branch can contain special characters that might cause path issues.

6. **Review logs written at approval only:** If user rejects at gate, no logs written. But the review work is done and wasted.

### Error Handling

- **Codex auth/timeout/empty:** Non-blocking, proceeds with Claude subagent only, tagged `[single-model]`
- **Claude subagent fails:** Tagged `[single-model]`, proceeds with Codex only
- **Both fail:** "Outside voices unavailable - continuing with primary review"
- **Missing outputs:** Max 2 retry attempts to produce them, then proceeds with warning
- **Taste decision overflow:** If 8+ taste decisions, groups by phase and warns "unusually high ambiguity"

---

## Feature 3: Document Release

**Command:** `/document-release`
**SKILL.md:** `document-release/SKILL.md` (preamble-tier 2, 653 lines)
**Tier:** Priority Tier 3 - Infrastructure & Power Tools

### Core Purpose

Post-ship documentation update. Reads all project docs, cross-references the diff, updates README/ARCHITECTURE/CONTRIBUTING/CLAUDE.md to match what shipped, polishes CHANGELOG voice, cleans up TODOS, and optionally bumps VERSION.

**Trigger:** Runs after `/ship` (code committed, PR exists or about to) but before PR merges. Proactively suggested after a PR is merged or code is shipped.

### Feature Flow

```
User invokes /document-release (or auto-suggested after /ship)
        |
        v
Step 0: Detect platform and base branch
        - GitHub vs GitLab detection via remote URL + CLI availability
        |
        v
Step 1: Pre-flight & Diff Analysis
        - Check on feature branch (aborts if on base branch)
        - git diff --stat, git log --oneline, git diff --name-only
        - find all .md files (maxdepth 2, excluding .git, node_modules, .gstack)
        - Classify changes: new features, changed behavior, removed functionality, infrastructure
        |
        v
Step 2: Per-File Documentation Audit
        - README.md: features, install instructions, examples, troubleshooting
        - ARCHITECTURE.md: ASCII diagrams, design decisions (conservative updates)
        - CONTRIBUTING.md: smoke test for new contributors
        - CLAUDE.md: project structure, commands, build/test instructions
        - Other .md files: cross-reference against diff
        - Classification per file: Auto-update vs Ask user
        |
        v
Step 3: Apply Auto-Updates
        - Factual corrections using Edit tool (never Write)
        - One-line summary per file: "README.md: added /new-skill to skills table"
        |
        v
Step 4: Ask About Risky/Questionable Changes
        - Uses AskUserQuestion for narrative changes, removals, rewrites >10 lines
        - Apply approved changes immediately
        |
        v
Step 5: CHANGELOG Voice Polish
        - Read existing entries - NEVER clobber them
        - Polish wording: "You can now..." not "Refactored..."
        - Sell test: would user think "oh nice, I want to try that"?
        - Auto-fix minor voice, AskUserQuestion for rewrites
        |
        v
Step 6: Cross-Doc Consistency & Discoverability Check
        - README features match CLAUDE.md?
        - ARCHITECTURE component list matches CONTRIBUTING?
        - CHANGELOG version matches VERSION file?
        - Discoverability: every doc reachable from README or CLAUDE.md?
        - Auto-fix factual inconsistencies, AskUserQuestion for narrative contradictions
        |
        v
Step 7: TODOS.md Cleanup
        - Mark completed items (if evidence in diff)
        - Update stale descriptions (AskUserQuestion)
        - New deferred work: scan for TODO, FIXME, HACK, XXX comments
        |
        v
Step 8: VERSION Bump Question
        - Check if VERSION was modified on branch
        - If not: AskUserQuestion (patch/minor/skip)
        - If already bumped: verify it covers all changes
        |
        v
Step 9: Commit & Output
        - Stage by specific filename (never git add -A)
        - Single commit with Co-Authored-By
        - Push to branch
        - Update PR/MR body with doc diff preview
        - Documentation health summary
```

### Key Implementation Details

**What NEVER happens:**
- CHANGELOG entries overwritten, replaced, or regenerated
- Write tool on CHANGELOG.md (always Edit with exact old_string)
- Bump VERSION without asking
- Auto-update narrative sections (README intro, ARCHITECTURE philosophy, security model)
- Remove entire sections from any document

**CHANGELOG voice rules:**
- Lead with what user can DO, not implementation details
- "You can now..." not "Refactored the..."
- Flag entries that read like commit messages
- Internal/contributor changes -> separate "For contributors" subsection

**Discoverability rule:** Every doc file should be reachable from README.md or CLAUDE.md. If ARCHITECTURE.md exists but neither README nor CLAUDE.md links to it, flag it.

**Auto-update vs Ask user classification:**

| Auto-update | Ask user |
|-------------|----------|
| Factual corrections from diff | Narrative changes |
| Adding to tables/lists | Section removals |
| Updating paths/counts/versions | Security model changes |
| Fixing stale cross-references | Large rewrites (>10 lines) |
| CHANGELOG voice polish | Adding entirely new sections |
| Marking TODOS complete | |

### Clever Solutions

1. **"Never clobber CHANGELOG" rule with backstory:** The SKILL explicitly notes "A real incident occurred where an agent replaced existing CHANGELOG entries when it should have preserved them." This became a hard rule encoded in the skill.

2. **Generic heuristics:** Not gstack-specific - the audit checks "work on any repo." Uses README.md, ARCHITECTURE.md, CONTRIBUTING.md conventions that are widespread, not gstack-specific.

3. **VERSION bump scope verification:** If VERSION was already bumped, checks whether the CHANGELOG entry actually covers ALL changes on the branch. A VERSION bump set for "feature A" shouldn't silently absorb "feature B."

4. **PR/MR body update is idempotent and race-safe:** Checks for existing "## Documentation" section, replaces if found, appends if not. Uses PID-unique tempfile.

5. **Co-Authored-By in commit:** `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` - proper attribution for AI-generated content.

6. **Cross-doc consistency checks:** Version mismatches, contradictory statements across docs caught automatically.

7. **TODO comment scanning:** Finds `TODO`, `FIXME`, `HACK`, `XXX` in diff and offers to capture meaningful ones in TODOS.md.

### Technical Debt / Shortcuts

1. **Maxdepth 2 for doc discovery:** `find . -maxdepth 2` misses deeply nested documentation. Could use broader patterns or ask user for custom paths.

2. **No automated link validation:** Discoverability check only verifies links exist in README/CLAUDE.md, doesn't verify those links actually work.

3. **CONTRIBUTING.md smoke test is walk-through only:** "Walk through the setup instructions as if you are a brand new contributor" - doesn't actually run commands, just verifies they look correct.

4. **No handling of internationalization:** All docs assumed English. No multi-language doc support.

5. **VERSION bump decision is purely human:** No heuristics for when to bump (e.g., always bump for new features, never for docs-only).

6. **Single-pass diff analysis:** Doesn't track cumulative changes across multiple document-release runs on the same branch.

### Error Handling

- **On base branch:** Aborts with "You're on the base branch. Run from a feature branch."
- **Empty diff (no docs modified):** "All documentation is up to date." - exits without committing
- **gh pr edit fails:** Warns but continues - "documentation changes are in the commit"
- **CHANGELOG entry looks wrong:** AskUserQuestion before fixing, never silently modify
- **VERSION doesn't exist:** Skips silently

### Interaction with Other Skills

- **Runs after `/ship`:** /ship creates the PR, document-release updates docs before merge
- **Referenced by `/land-and-deploy`:** Step 3.5 checks if document-release was run, warns if docs not updated despite new features
- **Uses review/TODOS-format.md:** Step 7 reads this for canonical TODO item format if available

---

## Cross-Feature Analysis

### Shared Infrastructure

All three skills share:
- **Same preamble structure:** Session management, telemetry, upgrade check, repo-mode detection
- **Same AskUserQuestion format:** Re-ground, simplify, recommend, options
- **Same completion status protocol:** DONE, DONE_WITH_CONCERNS, BLOCKED, NEEDS_CONTEXT
- **Same telemetry pattern:** Log to `~/.gstack/analytics/skill-usage.jsonl`, `gstack-telemetry-log` on completion
- **Same Plan Status Footer:** Writes `## GSTACK REVIEW REPORT` to plan files

### Dependency Graph

```
/ship (creates PR)
/ship
   |
   +-> /document-release (updates docs before merge)
   |
   +-> /land-and-deploy (merges, deploys, verifies)
          |
          +-> /setup-deploy (one-time config, persists to CLAUDE.md)
          |
          +-> /canary (verifies production health)
          |      |
          |      +-> $B (browse daemon)
          |
          +-> gstack-review-read (checks review staleness)

/autoplan (orchestrates reviews)
   |
   +-> /plan-ceo-review (loaded from disk, followed at full depth)
   |
   +-> /plan-design-review (loaded from disk, conditional on UI scope)
   |
   +-> /plan-eng-review (loaded from disk, followed at full depth)
          |
          +-> Codex CLI (via Bash, read-only web search)
          |
          +-> gstack-review-log (writes review entries)

/document-release
   |
   +-> review/TODOS-format.md (for TODO cleanup)
```

### Design Philosophy Patterns

1. **Human-in-the-loop for irreversible actions:** Merge (irreversible) has pre-merge gate, VERSION bump (irreversible) always asks, CHANGELOG (historical record) never clobbered.

2. **Baked-in lessons from incidents:** CHANGELOG clobbering rule came from "a real incident." Review staleness check came from understanding that reviews become outdated.

3. **Auto-detect + persist:** Deploy platform, base branch, review staleness - all auto-detected and cached for future runs or future steps.

4. **Escalation paths always available:** "This is too hard for me" / "I'm not confident in this result" always acceptable. 3-attempt limit before escalation.

5. **Evidence-based everything:** Screenshots for canary, diffs for document-release, consensus tables for dual voices, audit trails for decisions.

---

## Observations for Future Study

1. **Autoplan's dual-voice degradation** is sophisticated but complex. The degradation matrix (both fail, one fail, both succeed) adds resilience but increases code complexity significantly.

2. **Land-and-deploy's Step 3.5 readiness gate** is the most carefully designed safety mechanism in gstack. It aggregates multiple evidence sources (reviews, tests, docs, PR body) into a single decision point.

3. **Document-release's CHANGELOG rule** demonstrates how one bad incident (agent clobbered entries) gets encoded as a permanent safeguard.

4. **All three skills are "glue" skills** - they orchestrate other tools (git, gh, browse daemon, Codex CLI) rather than implementing new functionality. This is a common pattern in gstack's infrastructure tier.

5. **Preamble-tier correlates with reversibility risk:** Tier 4 (land-and-deploy) is highest risk (irreversible merge), Tier 2 (document-release) is lower (docs can be re-edited). The tier number seems to encode caution level.
