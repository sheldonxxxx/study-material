# 12 Lessons Learned from Studying gstack

**Project:** gstack - AI-augmented development workflow system
**Study Date:** 2026-03-27
**Source:** 8 research documents analyzing topology, tech stack, community, features, architecture, code quality, and security/performance

---

## Lesson 1: The Daemon Pattern Wins for Latency-Critical Tools

**What gstack does:** The `browse` tool uses a persistent Chromium daemon with state file coordination. First call starts the server (~3s), subsequent calls take ~100-200ms.

**Why it matters:** For AI agents making multiple tool calls, 3-5 second cold-start latency per command is unusable. The daemon pattern with state file (`~/.gstack/browse.json`) containing `{pid, port, token}` enables:
- Persistent cookies, tabs, localStorage across commands
- Auto-recovery when the server dies (health check + restart)
- Version mismatch detection (binary update auto-picks up on next command)

**The insight:** Long-lived processes with state file coordination are not just an optimization -- they are the only viable architecture for latency-sensitive AI tool calling.

**Specific technique:** HTTP server binds to `localhost` only, auth via random UUID bearer token in every request. This allows the CLI (thin wrapper) to communicate with the server without exposing network attack surface.

---

## Lesson 2: Skills are Vertical Slices, Not Horizontal Layers

**What gstack does:** 28 specialized skills (office-hours, review, qa, ship, etc.), each self-contained with SKILL.md, templates, and supporting docs. Skills compose via shared files in `~/.gstack/projects/{slug}/`.

**Why it matters:** The design doc pipeline -- office-hours writes design doc, plan-ceo-review reads it, plan-eng-review reads it, /qa consumes the test plan -- creates dependency without tight coupling. Skills communicate via file artifacts, not direct imports.

**The insight:** Skills should be vertical slices (a complete feature with model + API + UI equivalent) organized around user jobs-to-be-done, not horizontal layers (all planning skills together, all review skills together).

**Anti-pattern observed:** The test framework bootstrap code is nearly duplicated in /qa and /ship. This suggests extracting shared infrastructure when it appears in 2+ skills, not upfront.

---

## Lesson 3: Template-Driven Generation with Committed Output

**What gstack does:** SKILL.md files are auto-generated from `.tmpl` templates via `bun run gen-skill-docs`. The template (`.tmpl`) is the source of truth; the generated file (SKILL.md) is committed to git.

**Why it matters:**
- Claude reads SKILL.md at skill load time -- no build step needed
- CI validates freshness: `gen:skill-docs --dry-run` + `git diff --exit-code`
- Git blame works on the generated file
- Templates can include conditional content that expands differently per skill

**The insight:** Generated files should be committed when the generator is the source of truth AND the file is read at runtime. This is unusual but correct for Claude Code skill loading.

**Specific technique:** The generator (`scripts/gen-skill-docs.ts`) fills placeholders like `{{COMMAND_REFERENCE}}` (from `commands.ts`), `{{PREAMBLE}}` (shared tiered preamble), `{{BROWSE_SETUP}}` (binary discovery). Placeholders resolve from shared resolver modules.

---

## Lesson 4: Self-Regulation Prevents Runaway AI Behavior

**What gstack does:** Skills implement explicit self-regulation heuristics:

- **WTF-likelihood in /qa:** Starts at 0%, +15% per revert, +5% if fix touches >3 files, hard cap at 50 fixes
- **Risk threshold in /design-review:** After every 5 fixes, compute risk; 20% threshold triggers STOP
- **Fix cap in /qa:** Hard cap of 30 fixes regardless of remaining issues
- **Eval sequential execution:** If first eval fails, stops immediately rather than burning cost on remaining

**Why it matters:** Without explicit self-regulation, AI agents will continue fixing or working past the point of diminishing returns, causing more harm than good.

**The insight:** Self-regulation should be quantitative and explicit, not qualitative. Percentages and thresholds make the behavior predictable and debuggable.

---

## Lesson 5: Error Messages Must Guide the Next Action

**What gstack does:** Every error message tells the AI what to do next, not just what went wrong:

- "Ref @e3 is stale -- run 'snapshot' to get fresh refs"
- "Element not found -- run `$B snapshot -i` to see available elements"
- "Multiple elements matched -- use @refs from snapshot instead"

**Why it matters:** For AI agents, errors are experienced by the agent, not a human reading a stack trace. The error must encode the recovery action.

**Specific technique:** Errors for AI agents follow the format: `[what failed] -- [what to do instead]`. This is a design principle explicitly stated in the architecture docs.

---

## Lesson 6: Two-Tier Testing Balances Cost and Coverage

**What gstack does:**
- **Gate tier:** Runs on every PR, blocks merge. Deterministic, fast, safety-focused.
- **Periodic tier:** Runs weekly via cron. Quality benchmarks, non-deterministic, expensive.

**Why it matters:** LLM-judged evals cost ~$4/run. Running all evals on every PR would be prohibitively expensive. Running only unit tests misses quality issues that require LLM judgment.

**The insight:** Separate "must pass" (gate) from "nice to have" (periodic) with clear criteria for which tier a test belongs in. Diff-based test selection further reduces cost by only running tests affected by the current change.

---

## Lesson 7: Diff-Aware Mode Reduces Friction to Zero

**What gstack does:** When invoked on a feature branch, skills automatically:
1. Analyze the git diff to identify affected files/pages/routes
2. Map file changes to their runtime URLs (controllers -> routes, components -> pages)
3. Scope work to only the affected areas

**Why it matters:** The user doesn't have to specify what to test or review -- the skill figures it out from the change. This reduces the command from `/qa --url http://...` to simply `/qa`.

**Specific technique:** File change -> URL mapping heuristics:
- Next.js: `pages/` or `app/` directory structure -> URL routes
- Rails: controllers -> routes
- Components: identify which pages render them

---

## Lesson 8: Security Must Be Architectural, Not Incidental

**What gstack does:**
- **SSRF protection:** Cloud metadata endpoint blocklist (169.254.169.254, metadata.google.internal) + async DNS resolution to catch DNS rebinding
- **Shell injection hardening:** Output sanitization (alphanumeric only) in `gstack-slug`
- **SQL injection prevention:** Parameterized queries in cookie import
- **Secrets redaction:** JWT pattern detection (`eyJ...`), GitHub PAT prefix (`ghp_`), redacted in all output
- **Path traversal prevention:** `/tmp` whitelist, `..` blocking, path validation for eval/file commands

**Why it matters:** Browser automation tools have significant attack surface (arbitrary URL navigation, file access, cookie handling). Security must be designed in, not added as an afterthought.

**The insight:** Security patterns should be implemented at the architecture level (URL validation before navigation, query parameterization at the data layer) not scattered through the codebase as inline checks.

---

## Lesson 9: Documentation Lives in the Repo, Analytics Lives in Home

**What gstack does:**
- Design docs, CEO plans, test plans: stored in `~/.gstack/projects/{slug}/` (home directory)
- Review logs, security reports, QA reports: stored in `.gstack/` subdirectories (project-local, typically gitignored)
- CHANGELOG.md: committed to repo, for users not contributors

**Why it matters:** Personal working documents (plans, reviews) shouldn't be forced into the repo. Project artifacts (CHANGELOG, docs) should be committed. The separation is intentional and makes the repo cleaner.

**The insight:** Distinguish between "artifacts meant to be shared" (committed) and "artifacts meant for personal continuity" (home dir or gitignored). Don't force everything into the repo.

---

## Lesson 10: The Preamble Tier System Enables Code Reuse Without Coupling

**What gstack does:** All skills share a tiered preamble infrastructure:
- **T1:** core + upgrade + lake + telemetry + contributor + completion
- **T2:** T1 + ask format + completeness
- **T3:** T2 + repo mode + search before building
- **T4:** T3 + test failure triage

Skills declare their tier; the preamble generator includes the appropriate sections.

**Why it matters:** Common infrastructure (update checking, telemetry prompts, contributor mode) is shared without copy-paste. Each skill gets exactly what it needs.

**The insight:** Shared code doesn't always mean shared modules. For prompt-based skills, shared prose fragments via template placeholders achieve reuse without creating import dependencies.

---

## Lesson 11: Quantitative Health Scores Enable Progress Tracking

**What gstack does:** QA and design-review compute numerical health scores:
- **QA health score:** Weighted categories (Console 15%, Functional 20%, UX 15%, etc.), deductions per severity level
- **Design score:** Letter grades (A-F) across 10 categories
- **Security trend tracking:** IMPROVING / DEGRADING / STABLE based on SHA-256 fingerprinting of findings

**Why it matters:** A single number (72/100, grade B) is easier to track over time than a list of findings. Regression mode compares current run against baseline to show what got worse.

**The insight:** Health metrics should be computed, not subjective. The scoring rubrics are explicit and published (in SKILL.md templates), making the methodology transparent and reproducible.

---

## Lesson 12: Non-Interactive Automation Is a Design Choice

**What gstack does:** `/ship` is explicitly non-interactive. User says `/ship`, workflow runs straight through. Only stops for:
- Merge conflicts
- In-branch test failures
- MINOR/MAJOR version bumps
- Coverage below minimum threshold

**Why it matters:** Asking for input during a release workflow breaks the flow. The design principle is: shipping should be a pipeline, not a conversation.

**Contrast:** `/review` and `/qa` are interactive -- they use `AskUserQuestion` frequently. This is intentional -- review and QA are judgment-heavy tasks that benefit from human input.

**The insight:** Each skill should have a clear "interaction budget." Some skills (ship) should be fully automated. Others (review, qa) should pause for judgment calls. The interaction style is a feature, not a limitation.

---

## Summary: Key Principles

| Principle | Implementation |
|-----------|---------------|
| Latency matters | Daemon + state file pattern |
| Skills are vertical slices | Self-contained, file-based coupling |
| Generated files committed | SKILL.md from .tmpl, CI validates freshness |
| Self-regulation is quantitative | WTF-likelihood, risk thresholds, fix caps |
| Errors guide next action | "[what failed] -- [what to do]" format |
| Two-tier testing | Gate (deterministic, cheap) vs periodic (LLM-judged, expensive) |
| Diff-aware reduces friction | Auto-scope to changed files/pages |
| Security is architectural | URL validation, query parameterization, secrets redaction at boundaries |
| Docs vs analytics separation | Committed docs, home-dir analytics |
| Shared infrastructure via templates | Preamble tiers, not shared modules |
| Health scores are computed | Explicit rubrics, not subjective |
| Non-interactive is a feature | Ship is pipeline; review is conversation |
