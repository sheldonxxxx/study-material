# Feature Deep Dive: Batch 5

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Batch:** 05e — Investigate, Canary, Benchmark
**Date:** 2026-03-27

---

## Feature 1: Investigate (Debugger)

### Command
`/investigate`

### Core Implementation

This is a **pure prompt-based skill** — there is no executable code. The debugging methodology is entirely encoded in the SKILL.md template (`investigate/SKILL.md.tmpl`). The skill's "implementation" is a structured prompt that guides Claude through a systematic root-cause investigation process.

### Key Files
- `investigate/SKILL.md` — auto-generated from template
- `investigate/SKILL.md.tmpl` — source template (edits go here)
- `freeze/bin/check-freeze.sh` — shell script invoked as PreToolUse hook

### Architecture

```
/investigate SKILL.md
    |
    +-- PreToolUse hooks (Edit/Write tools)
    |       |
    |       +-- freeze/bin/check-freeze.sh
    |               |
    |               +-- Reads ~/.gstack/freeze-dir.txt
    |               +-- Blocks edits outside freeze boundary
    |
    +-- Prompt phases (run by Claude)
            |
            +-- Phase 1: Root Cause Investigation
            +-- Phase 2: Pattern Analysis
            +-- Phase 3: Hypothesis Testing
            +-- Phase 4: Implementation
            +-- Phase 5: Verification & Report
```

### Methodology (The "Iron Law")

**Core principle:** NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.

The skill enforces a strict 5-phase debugging workflow:

1. **Phase 1: Root Cause Investigation** — Collect symptoms, read code, check `git log`, attempt to reproduce
2. **Scope Lock** — After forming hypothesis, lock edits to the affected directory using `/freeze`
3. **Phase 2: Pattern Analysis** — Check against known bug patterns (race condition, nil/null propagation, state corruption, etc.)
4. **Phase 3: Hypothesis Testing** — Verify hypothesis before fixing. 3-strike rule: after 3 failed hypotheses, escalate
5. **Phase 4: Implementation** — Fix root cause, minimal diff, write regression test
6. **Phase 5: Verification** — Reproduce original bug, confirm fix, run test suite

### Interesting Integration: Freeze Hook

The skill uses a **PreToolUse hook** to invoke `freeze/bin/check-freeze.sh` before any Edit or Write operation:

```bash
# From investigate/SKILL.md.tmpl
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
```

The `check-freeze.sh` script:
- Reads `file_path` from tool input JSON
- Compares against `~/.gstack/freeze-dir.txt`
- Returns `{"permissionDecision":"deny", ...}` to block, or `{}` to allow
- Uses grep with Python fallback for JSON parsing of the tool input

### Clever Solutions

1. **Scope Lock Pattern** — Rather than hardcoding a freeze, the skill dynamically detects the affected directory from the hypothesis and sets the freeze boundary. This prevents accidental edits to unrelated code during debugging.

2. **3-Strike Escalation** — Forces the user to make a decision after 3 failed hypotheses, preventing endless debugging loops.

3. **Sanitized Web Search** — When hypothesis fails, the skill sanitizes error messages (strips IPs, paths, SQL, customer data) before searching. This prevents leaking sensitive data while still enabling external knowledge.

4. **Pattern Table** — Pre-defined bug signatures (race condition, nil propagation, etc.) provide a structured starting point before hypothesis formation.

### Technical Debt / Shortcuts

1. **Shell-based Freeze Check** — Uses `grep | sed` with Python fallback to parse JSON. Fragile if Claude's tool input format changes.

2. **No Persistent Investigation State** — The investigation state (hypothesis, tested paths) is not persisted between sessions. A user returning to the bug must re-establish context.

3. **Freeze Script Race Condition** — The freeze check runs as a separate process reading from a file. There's no locking, so concurrent edits could theoretically bypass the boundary.

4. **Python Fallback is Silent** — If Python fails in the fallback path, it silently allows the operation. This is safe but means edge cases silently succeed.

### Error Handling

- **Parse failures silently allow** — If `check-freeze.sh` cannot extract the file path, it allows the operation
- **No freeze file = allow all** — If `~/.gstack/freeze-dir.txt` doesn't exist, no blocking occurs
- **Escalation protocol** — Formal "STOP" conditions: 3 failed attempts, uncertain about security change, scope exceeds verification

### Edge Cases

1. **Bug spans entire repo** — Skip scope lock and note why
2. **No reproduction possible** — Gather more evidence, don't guess
3. **Fix touches >5 files** — AskUserQuestion about blast radius before proceeding
4. **"Quick fix for now" red flag** — Reject it; fix it right or escalate

---

## Feature 2: Canary (Post-Deploy Monitoring)

### Command
`/canary <url> [--duration 5m] [--baseline] [--pages /,/dashboard] [--quick]`

### Core Implementation

This is a **browse-daemon-dependent skill**. It uses the headless Chromium (`$B` command) to navigate pages, capture screenshots, check console errors, and measure performance. The skill logic is a prompt, but it drives real browser automation.

### Key Files
- `canary/SKILL.md` — auto-generated from template
- `canary/SKILL.md.tmpl` — source template
- `browse/src/commands.ts` — command registry
- `browse/src/snapshot.ts` — snapshot implementation
- `browse/src/browser-manager.ts` — browser lifecycle

### Architecture

```
/canary SKILL.md
    |
    +-- BROWSE_SETUP (browse daemon startup check)
    |
    +-- BASE_BRANCH_DETECT (git platform detection)
    |
    +-- Prompt phases (drive $B commands)
            |
            +-- Phase 1: Setup (mkdir .gstack/canary-reports/)
            +-- Phase 2: Baseline Capture (--baseline mode)
            +-- Phase 3: Page Discovery ($B goto, links, snapshot)
            +-- Phase 4: Pre-Deploy Snapshot
            +-- Phase 5: Continuous Monitoring Loop (60s intervals)
            +-- Phase 6: Health Report
            +-- Phase 7: Baseline Update
            |
            +-- $B commands used:
                    goto, snapshot, console --errors, perf, text, links
```

### Monitoring Workflow

**Startup:**
```bash
$B goto <page-url>
$B snapshot -i -a -o ".gstack/canary-reports/screenshots/pre-<page>.png"
$B console --errors
$B perf
```

**Continuous Loop (every 60s):**
```bash
$B goto <page-url>
$B snapshot -i -a -o ".gstack/canary-reports/screenshots/<page>-<check#>.png"
$B console --errors
$B perf
```

### Alert Levels

| Level | Condition | Threshold |
|-------|-----------|-----------|
| CRITICAL | Page load failure | `goto` error or timeout |
| HIGH | New console errors | Errors not in baseline |
| MEDIUM | Performance regression | Load time > 2x baseline |
| LOW | Broken links | New 404s not in baseline |

**Smart tolerance:** Only fires alerts on patterns persisting across **2+ consecutive checks**. Single transient blips are ignored.

### Clever Solutions

1. **Baseline Comparison, Not Absolutes** — A page with 3 existing console errors is fine if it still has 3. Only NEW errors trigger alerts. This prevents alert fatigue.

2. **Pre-Deploy Snapshot Fallback** — If user didn't run `--baseline` before deploying, takes a quick snapshot as reference point instead of failing.

3. **Auto Page Discovery** — If `--pages` not specified, auto-discovers top 5 navigation links from the homepage and asks the user which to monitor.

4. **Screenshot as Evidence** — Every alert includes a screenshot path. Screenshots are stored in `.gstack/canary-reports/screenshots/` with timestamps.

5. **JSONL Logging** — Results logged to `~/.gstack/projects/$SLUG/` for the review dashboard integration:
   ```json
   {"skill":"canary","timestamp":"<ISO>","status":"<STATUS>","url":"<url>","duration_min":<N>,"alerts":<N>}
   ```

### Technical Debt / Shortcuts

1. **Hardcoded 60s Intervals** — Monitoring checks every 60 seconds. Not configurable. For 10-minute monitoring = 10 checks. If user wants 30-minute monitoring at 30s intervals, not possible.

2. **2x Threshold is Arbitrary** — "Load time exceeds 2x baseline" for MEDIUM alert. Some apps may legitimately vary more. No adaptive threshold.

3. **No Alert Deduplication** — If the same alert fires 10 times, it fires 10 times. The tolerance is per-check, not across the entire monitoring session.

4. **No Background Execution** — The monitoring loop runs synchronously in Claude's context. If Claude disconnects, monitoring stops. No daemon mode.

5. **Screenshot Storage** — Screenshots stored in project directory (`.gstack/canary-reports/`). Could accumulate large files. No auto-cleanup.

### Error Handling

- **CRITICAL/HIGH alert** — Immediately asks user via AskUserQuestion with 4 options: Investigate, Continue, Rollback, Dismiss
- **Goto timeout** — Caught as CRITICAL alert
- **Missing baseline.json** — Falls back to pre-deploy snapshot approach

### Edge Cases

1. **URL with no navigation** — Falls back to homepage-only monitoring
2. **Site requires auth** — Canary doesn't handle authentication. User must set up cookies first via `/setup-browser-cookies`
3. **Transient vs real issue** — The 2-check tolerance helps, but no mechanism to distinguish network blip from real regression
4. **Duration limits** — 1m minimum, 30m maximum. Hardcoded.

---

## Feature 3: Benchmark (Performance Engineering)

### Command
`/benchmark <url> [--baseline] [--quick] [--pages /,/dashboard] [--diff] [--trend]`

### Core Implementation

Similar to Canary, this is a **browse-daemon-dependent skill**. It uses Playwright's `performance.getEntriesByType()` JavaScript APIs to collect detailed timing and resource metrics. The skill logic is a prompt that drives browser instrumentation.

### Key Files
- `benchmark/SKILL.md` — auto-generated from template
- `benchmark/SKILL.md.tmpl` — source template
- `browse/src/commands.ts` — `perf` and `eval` commands
- `browse/src/read-commands.ts` — `handlePerf()` implementation

### Architecture

```
/benchmark SKILL.md
    |
    +-- BROWSE_SETUP
    |
    +-- Prompt phases
            |
            +-- Phase 1: Setup
            +-- Phase 2: Page Discovery
            +-- Phase 3: Performance Data Collection
            +-- Phase 4: Baseline Capture
            +-- Phase 5: Comparison
            +-- Phase 6: Slowest Resources
            +-- Phase 7: Performance Budget
            +-- Phase 8: Trend Analysis
            +-- Phase 9: Save Report
            |
            +-- $B commands used:
                    goto, perf, eval (JavaScript instrumentation)
```

### Performance Metrics Collected

**Navigation Timing:**
```javascript
// Via $B eval
JSON.stringify(performance.getEntriesByType('navigation')[0])

// Extracted:
// - TTFB: responseStart - requestStart
// - FCP: from paint entries
// - LCP: from PerformanceObserver (Largest Contentful Paint)
// - DOM Interactive: domInteractive - navigationStart
// - DOM Complete: domComplete - navigationStart
// - Full Load: loadEventEnd - navigationStart
```

**Resource Analysis:**
```javascript
// Top 15 slowest resources
performance.getEntriesByType('resource')
  .map(r => ({name, type, size: transferSize, duration}))
  .sort((a,b) => b.duration - a.duration)
  .slice(0,15)

// Bundle sizes (JS and CSS separately)
performance.getEntriesByType('resource')
  .filter(r => r.initiatorType === 'script')
  .map(r => ({name, size: transferSize}))
```

**Network Summary:**
```javascript
// Total requests, total transfer size, breakdown by type
performance.getEntriesByType('resource')
```

### Regression Thresholds

| Metric | Regression | Warning |
|--------|------------|---------|
| Timing (>50% increase OR >500ms absolute) | >50% OR >500ms | >20% |
| Bundle size | >25% increase | >10% increase |
| Request count | — | >30% increase |

### Clever Solutions

1. **JavaScript Instrumentation** — Uses `performance.getEntriesByType()` directly rather than relying on Playwright's built-in metrics. This gives granular control over what is measured.

2. **Resource Attribution** — Resources are named by parsing `r.name.split('/').pop().split('?')[0]` to get clean filenames (removes path and query string). This makes bundle reports readable.

3. **Third-Party Separation** — The slowest resources report flags third-party scripts (e.g., `analytics.js`) separately from first-party. User is told "you can't fix Google Analytics" — focusing recommendations on actionable items.

4. **Diff Mode** — `--diff` uses `gh pr view` to detect base branch, then `git diff` to find affected files. Only benchmarks pages potentially impacted by the PR.

5. **Trend Analysis** — `--trend` loads historical baseline files to show performance over time. Detects creeping regressions (e.g., "JS bundle growing 50KB/week").

6. **Performance Budget** — Industry-standard budgets (FCP < 1.8s, LCP < 2.5s, Total JS < 500KB). Gives letter grades (A-F).

### Technical Debt / Shortcuts

1. **No Real Core Web Vitals** — FCP and LCP are approximated from paint entries, not actual PerformanceObserver-based CWV measurement. The "LCP" in the report is not true LCP.

2. **Single Page Load Sample** — Metrics are from one page load. No warm-up runs, no median/percentile. A single slow network hiccup skews results.

3. **No Network Conditioning** — Doesn't simulate slow 3G, throttled connections, or CPU slowdown. Measures only the current network.

4. **Baseline File Accumulation** — Stores baselines in `.gstack/benchmark-reports/baselines/baseline.json`. No cleanup of old baselines.

5. **`--diff` Uses git, Not Code Analysis** — `git diff --name-only` tells which files changed, but doesn't know which pages those files affect. Could benchmark pages unrelated to the change.

6. **Timing from JavaScript, Not Navigation Timing API Properly** — The code reads `responseStart - requestStart` for TTFB but this is unreliable in some browsers. The Navigation Timing API is the authoritative source but requires proper handling of cross-origin timing.

### Error Handling

- **Missing baseline** — Reports absolute numbers but can't detect regressions. Prompts user to capture baseline.
- **Page load failure** — Handled as CRITICAL in the monitoring context, but benchmark doesn't have explicit handling for this.
- **JavaScript errors in eval** — Not explicitly handled. If `performance.getEntriesByType()` fails, the eval returns an error string.

### Edge Cases

1. **SPA (Single Page App)** — After `goto`, the page may hydrate and change. The benchmark measures initial load only. Subsequent navigation via JavaScript wouldn't be measured.

2. **Authenticated Pages** — Similar to Canary, no auth handling. User must set up cookies.

3. **CDN-Cached Responses** — If a resource is cached, transfer size is 0 or near-zero. This skews bundle size comparisons.

4. **HTTP/2 Multiplexing** — Resource timing shows individual resource durations but doesn't account for HTTP/2 concurrent connection savings. A page with 50 resources may actually load faster than a page with 10.

---

## Cross-Feature Observations

### Shared Infrastructure

1. **Browse Daemon** — Both Canary and Benchmark depend on `$B` commands. The browse daemon is a long-lived Playwright/Chromium process. First call starts it (~3s), subsequent calls ~100-200ms.

2. **Report Storage** — Both use `.gstack/canary-reports/` and `.gstack/benchmark-reports/` directories within the project.

3. **JSONL Logging** — Both log to `~/.gstack/projects/$SLUG/` for the review dashboard. Uses `eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"` to get the project slug.

4. **Snapshot System** — Both use browse's snapshot command for evidence. Snapshot creates an accessibility tree with `@e` refs for element selection.

5. **Platform Detection** — Both include `BASE_BRANCH_DETECT` template variable that detects GitHub vs GitLab via remote URL and CLI availability.

### Template System

All three skills follow the pattern:
- `SKILL.md.tmpl` is the source of truth
- `SKILL.md` is auto-generated via `bun run gen:skill-docs`
- Template variables like `{{PREAMBLE}}`, `{{BROWSE_SETUP}}`, `{{BASE_BRANCH_DETECT}}` are resolved during generation

This means the skills are entirely prompt-driven. There is no executable code beyond the browse daemon and the freeze shell script.

### Preamble Commonality

All three skills include the standard gstack preamble:
- Session tracking (`~/.gstack/sessions/$PPID`)
- Telemetry opt-out prompts (LAKE_INTRO, TEL_PROMPTED, PROACTIVE_PROMPTED)
- Contributor mode prompts
- Update checking
- Telemetry logging on completion

---

## Summary Table

| Aspect | Investigate | Canary | Benchmark |
|--------|-------------|--------|-----------|
| **Type** | Prompt-only | Prompt + browse | Prompt + browse |
| **External deps** | None | Browse daemon | Browse daemon |
| **Key hook** | freeze/check-freeze.sh | $B goto/snapshot | $B eval (JS) |
| **Output** | Debug report | Canary report + screenshots | Performance report |
| **Baseline concept** | No | Yes (screenshots + errors) | Yes (JSON metrics) |
| **Alert tolerance** | 3-strike rule | 2 consecutive checks | >50% thresholds |
| **Main limitation** | No persistent state | No background daemon | Single sample, no CWV |
