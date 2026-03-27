# gstack Feature Deep Dive — Batch 3

**Features:** Browse (Headless Browser), Plan Design Review, Design Consultation
**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Analysis date:** 2026-03-27

---

## Feature 1: Browse (Headless Browser)

### Overview

A persistent Chromium daemon with ~100ms per command latency. State (cookies, tabs, localStorage) persists across commands. The daemon model means Chromium starts once (~3s), then every subsequent command reuses the same browser instance.

### Architecture

**Daemon model: CLI + HTTP server**

```
CLI (browse/src/cli.ts)
  └── reads .gstack/browse.json for {pid, port, token}
  └── ensures server is running (health check)
  └── HTTP POST to http://127.0.0.1:{port}/command
  └── prints response to stdout

Server (browse/src/server.ts)
  └── Bun.serve HTTP on localhost (random port 10000-60000)
  └── routes commands to READ, WRITE, or META handlers
  └── manages browser lifecycle via BrowserManager

BrowserManager (browse/src/browser-manager.ts)
  └── Playwright chromium.launch() — single browser instance
  └── manages multiple tabs (Map<tabId, Page>)
  └── dialog auto-handling (prevents browser lockup)
  └── state save/restore for handoff and useragent changes
```

**Command dispatch architecture:**

```
server.ts: handleCommand()
  └── READ_COMMANDS  → handleReadCommand()    (browse/src/read-commands.ts)
  └── WRITE_COMMANDS → handleWriteCommand()    (browse/src/write-commands.ts)
  └── META_COMMANDS  → handleMetaCommand()    (browse/src/meta-commands.ts)
  └── handleSnapshot() in meta-commands.ts      (browse/src/snapshot.ts)
```

**State file:** `.gstack/browse.json` written by server at startup, read by CLI on every invocation. Contains `{pid, port, token, startedAt, serverPath, binaryVersion}`.

**Command registry** (`browse/src/commands.ts`) is the single source of truth for all commands, with load-time validation that every command in `READ_COMMANDS | WRITE_COMMANDS | META_COMMANDS` has a corresponding entry in `COMMAND_DESCRIPTIONS` and vice versa. This prevents desync between the command list and the help text.

### Key Implementation Details

#### Snapshot with Ref System (`snapshot.ts`)

The most sophisticated piece. Uses Playwright's `page.ariaSnapshot()` to get an accessibility tree, then:

1. Parses the YAML-like output line by line
2. Assigns `@e1`, `@e2`, ... refs to interactive elements
3. Stores `Map<ref, {locator: Playwright.Locator, role, name}>` on BrowserManager
4. Subsequent commands (`click @e3`, `fill @e4`) look up the locator and call `locator.click()` / `locator.fill()`

This is elegant because Locators are live references — they re-resolve on every call, so stale DOM elements don't cause false failures. The `nth()` disambiguation handles multiple elements with the same role+name.

**Diff mode (`-D`):** Stores the raw text snapshot (not DOM), uses the `diff` npm package to produce unified diffs. Smart — only text changes are captured, not structural DOM changes, but this means structural changes are invisible in diffs.

**Cursor-interactive scan (`-C`):** Falls back to JavaScript evaluation (`document.querySelectorAll('*')`) to find divs with `cursor:pointer`, `onclick`, or `tabindex >= 0` that aren't in the ARIA tree. Marks these as `@c1`, `@c2`, etc. Can fail under CSP restrictions — this is handled gracefully with a "(cursor scan failed — CSP restriction)" message.

**Annotated screenshot (`-a`):** Injects `__browse_annotation__` divs at each ref's bounding box, takes a full-page screenshot, removes overlays. The overlays are injected via `page.evaluate()` and removed in a `finally`-equivalent pattern — even if the screenshot fails.

#### Circular Buffers (`buffers.ts`)

O(1) insert ring buffers with 50,000 entry capacity:

```
push() writes at (head+size) % capacity, overwrites oldest when full
toArray() returns entries in insertion order, O(n)
last(n) returns the n most recent entries
totalAdded keeps incrementing past capacity (used as flush cursor)
get(i) / set(i) for network response matching (backward scan to update status)
```

Console, network, and dialog entries are captured via Playwright page event listeners (`page.on('console')`, `page.on('request')`, `page.on('response')`, `page.on('dialog')`), pushed to buffers, and flushed to disk every 1 second via `setInterval(flushBuffers, 1000)`. The async disk writes mean command latency is unaffected.

#### Browser Lifecycle (`browser-manager.ts`)

**Launch:** `chromium.launch({headless: true})` with platform-specific handling:
- Container/CI: adds `--no-sandbox` automatically
- Windows: disables Chromium sandbox (Bun process chain incompatibility, GitHub #276)
- Extensions: if `BROWSE_EXTENSIONS_DIR` is set, launches headed with off-screen window (`--window-position=-9999,-9999 --window-size=1,1`)

**Chromium crash:** `browser.on('disconnected')` handler calls `process.exit(1)`. The CLI detects the dead server on the next command via the health check and auto-restarts it.

**Dialog auto-handling:** `page.on('dialog')` auto-accepts by default (configurable via `dialog-accept`/`dialog-dismiss` commands). Dialog entries are buffered so the AI can inspect what dialogs fired even if it auto-accepted them.

**State save/restore:** `saveState()` captures cookies + per-page URLs + localStorage/sessionStorage. `restoreState()` recreates pages, navigates to saved URLs, and restores storage. Individual page failures are swallowed — partial restore is better than complete failure. Falls back to clean slate if the full restore fails.

**Context recreation (useragent change):** `recreateContext()` saves state, closes old context, creates new context with updated userAgent, restores state. Falls back to clean slate on any failure.

#### Handoff: Headless to Headed (`browser-manager.ts`)

The `handoff()` command launches a new headed browser while keeping the headless one running. The flow is launch-first-close-second:

1. Save state from headless browser
2. Launch new headed browser (if this fails, headless stays running — rollback)
3. Restore state into headed browser
4. Close old headless browser (fire-and-forget — `.close()` can hang when another Playwright instance is active)
5. Swap `this.browser` / `this.context` references

This is a careful pattern — the headed browser can fail to launch (e.g., no display on Linux), and in that case the headless browser remains untouched.

#### CLI Server Lifecycle (`cli.ts`)

- **Lockfile:** `O_CREAT | O_EXCL` atomic check-and-create to prevent concurrent startup races. If lock is held by a dead process (stale lock), removes it and retries.
- **Health check:** HTTP `GET /health` — more reliable than PID checking (PID check fails on Windows because Bun's `process.kill()` throws ESRCH for Windows PIDs in compiled binaries).
- **Version mismatch:** On every command, CLI reads the current binary's version hash and compares against the server's stored `binaryVersion`. If different, auto-restarts the server. This means updating the binary automatically picks up the new version on the next command.
- **Connection loss recovery:** If a command gets ECONNREFUSED or ECONNRESET, retries once by restarting the server. If it crashes twice in a row, aborts.

#### Security

- **Path validation:** `eval <file>` and `cookie-import <json-file>` validate that absolute paths are within `TEMP_DIR` or `cwd`, and reject `..` traversal sequences.
- **Sensitive value redaction:** `storage` command redacts values matching patterns for tokens, API keys, JWTs, and other credentials (both by key name and by value prefix like `eyJ`, `sk-`, `ghp_`, etc.).
- **Header redaction:** `header` command redacts sensitive headers (authorization, cookie, x-api-key) in the success message.

#### Cookie Import (`cookie-import-browser.ts`)

Supports direct import via CLI args (`--domain`, `--profile`) or an interactive picker UI served by the browse server at `/cookie-picker`. The picker lists detected Chromium-based browsers (Brave, Arc, Chrome, Edge, Comet) and their profile directories, lets users select domains, and imports cookies via the `chrome-cookies` approach (reading the browser's on-disk SQLite cookies and decrypting with the system keychain on macOS).

#### Platform Handling

- **Windows:** Uses Node.js `child_process.spawn` with `{detached: true}` to launch the server as a truly independent process. Bun's `spawn + unref` doesn't achieve true detachment on Windows.
- **macOS/Linux:** Uses Bun's `spawn` with `unref()` — works correctly.
- **Error log on Windows:** Since Windows server launch uses `stdio: ['ignore', 'ignore', 'ignore']` for detachment, stderr is unavailable. The server writes errors to `browse-startup-error.log` in the state directory, which the CLI reads on startup failure.
- **Cleanup:** `cleanupLegacyState()` removes old `/tmp/browse-server*.json` files from before project-local state was introduced, verifying PID ownership before sending SIGTERM.

### Edge Cases and Error Handling

- **Stale refs:** If a ref points to an element that no longer exists (count == 0), throws a clear error: "Ref @e3 (button "Submit") is stale — element no longer exists. Run 'snapshot' for fresh refs."
- **Multi-element selector:** If a CSS selector matches multiple elements and the user passes it to `click`/`fill` directly (not via @ref), Playwright throws. The error message is enhanced to suggest using `@refs` from snapshot.
- **`<option>` click:** Auto-detects when clicking an `<option>` element and throws a helpful error suggesting `select` instead. This is a known Playwright footgun.
- **Failed element wait:** `click`/`fill`/`hover` use 5s timeout. Timeout errors from Playwright are wrapped into actionable messages.
- **URL validation:** `validateNavigationUrl()` rejects `file://` URLs and other schemes that could access local filesystem.
- **Blob URLs:** Handled correctly via `waitForLoadState` after navigation.
- **about:blank pages:** `getCurrentUrl()` returns `'about:blank'` rather than crashing when no page is available.

### Testing

Tests cover: config resolution, command registry validation, URL validation, path traversal prevention, snapshot parsing, cookie import, cookie picker routes, handoff state, platform detection, and the bun polyfill.

---

## Feature 2: Plan Design Review

### Overview

A senior product designer who reviews a plan (not a live site) and improves it by finding missing design decisions. The output is a better plan, not a document about the plan. Uses a 7-pass structured review with 0-10 ratings per dimension.

### Architecture

**Workflow:**
1. Pre-review audit: git log, diff, CLAUDE.md, DESIGN.md, TODOS.md
2. Step 0: Initial design rating + focus area negotiation with user
3. Design Outside Voices (optional): Codex + Claude subagent run in parallel, synthesized into a litmus scorecard
4. 7 review passes, each rated 0-10, with fixes applied iteratively until 8+ or user says "move on"
5. Completion summary with ratings before/after each pass
6. TODOS.md proposals (each as individual AskUserQuestion)
7. Review log → `~/.gstack/` → Review Readiness Dashboard

**Rating method:** For each section, rate 0-10, explain the gap (what a 10 looks like), apply fixes to the plan, re-rate, ask user if there's a genuine design choice to resolve, repeat.

### Key Design Principles

The skill articulates 9 explicit design principles:
1. Empty states are features
2. Every screen has a hierarchy
3. Specificity over vibes
4. Edge cases are user experiences
5. AI slop is the enemy
6. Responsive is not "stacked on mobile"
7. Accessibility is not optional
8. Subtraction default
9. Trust is earned at the pixel level

### Cognitive Patterns

12 perceptual instincts that run automatically during review:
- Seeing systems, not screens
- Empathy as simulation (not "I feel for the user" but running mental simulations)
- Hierarchy as service
- Constraint worship
- The question reflex
- Edge case paranoia
- "Would I notice?" test
- Principled taste (taste is debuggable, not subjective)
- Subtraction default
- Time-horizon design (5-sec visceral, 5-min behavioral, 5-year reflective)
- Design for trust (strangers sharing a home)
- Storyboard the journey

### The 7 Passes

**Pass 1: Information Architecture** — What does the user see first, second, third? ASCII diagram of screen/page structure and navigation flow.

**Pass 2: Interaction State Coverage** — Loading, empty, error, success, partial states per feature. Empty states are features — specify warmth, primary action, context.

**Pass 3: User Journey & Emotional Arc** — Storyboard table (STEP | USER DOES | USER FEELS | PLAN SPECIFIES?). Applies time-horizon design.

**Pass 4: AI Slop Risk** — Detects generic patterns. Classifier determines which rule set applies:
- **MARKETING/LANDING PAGE:** Brand-first hierarchy, full-bleed hero, typography (no defaults), no cards in hero, 2-3 intentional motions
- **APP UI:** Calm surface hierarchy, dense but readable, utility language, minimal chrome
- **HYBRID:** Both rule sets apply to their respective sections

Hard rejection criteria (instant-fail):
1. Generic SaaS card grid as first impression
2. Beautiful image with weak brand
3. Strong headline with no clear action
4. Busy imagery behind text
5. Sections repeating same mood statement
6. Carousel with no narrative purpose
7. App UI made of stacked cards instead of layout

AI slop blacklist (10 patterns):
- Purple/violet/indigo gradients
- 3-column feature grid with icon-in-colored-circle
- Icons in colored circles as decoration
- Centered everything
- Uniform bubbly border-radius
- Decorative blobs/floating circles/wavy SVG dividers
- Emoji as design elements
- Colored left-border on cards
- Generic hero copy
- Cookie-cutter section rhythm

**Pass 5: Design System Alignment** — Against DESIGN.md if it exists. New components checked for fit with existing vocabulary.

**Pass 6: Responsive & Accessibility** — Mobile/tablet specs (not "stacked on mobile"), keyboard nav, ARIA landmarks, 44px minimum touch targets, color contrast.

**Pass 7: Unresolved Design Decisions** — Table of ambiguities that will haunt implementation, each with one AskUserQuestion.

### Outside Voices

Launches Codex and Claude subagent in parallel. Codex evaluates against OpenAI's design hard rules and litmus checks. Claude subagent does independent completeness review. Results are synthesized into a litmus scorecard:

```
  Check                                    Claude  Codex  Consensus
  ───────────────────────────────────────  ─────── ─────── ─────────
  1. Brand unmistakable in first screen?   —       —      —
  2. One strong visual anchor?             —       —      —
  ...
  Hard rejections triggered:               —       —      —
```

Hard rejections from outside voices are pre-loaded as the FIRST items in Pass 1, tagged `[HARD REJECTION]`.

### Review Readiness Dashboard

At completion, reads review logs and displays:
```
| Review     | Runs | Last Run      | Status | Required |
| Eng Review |  1   | 2026-03-16    | CLEAR  | YES      |
```

With verdict logic:
- CLEARED if Eng Review has >= 1 entry within 7 days from `review` or `plan-eng-review` with status "clean" (or `skip_eng_review` is true)
- NOT CLEARED if missing, stale, or has open issues

Also writes a `## GSTACK REVIEW REPORT` section to the plan file itself (plan mode exception).

### Completion Status Protocol

Uses DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT with escalation:
- 3 failed attempts → STOP and escalate
- Security-sensitive uncertainty → STOP and escalate
- Scope exceeds verifyable bounds → STOP and escalate

### Shortcuts / Technical Debt

- The 7 passes are sequential with user pause after each. This is intentional (one issue at a time, per the skill design), but could be slow for large plans.
- AskUserQuestion is used heavily — this is by design for genuine design choices, but the skill has a lot of decision points that require user input.
- No automated verification that design decisions actually fix the issues raised — relies on re-rating.
- "Review chaining" at the end recommends `/plan-eng-review` and optionally `/plan-ceo-review` based on what was found.

---

## Feature 3: Design Consultation

### Overview

A senior product designer who acts as a design consultant — listening, researching, proposing, and refining a complete design system. The output is `DESIGN.md` and updates to `CLAUDE.md`. Proposes SAFE/RISK breakdowns and generates a live HTML preview page.

### Architecture

**Workflow:**
1. Phase 0: Pre-checks — existing DESIGN.md? codebase context from README/package.json/office-hours
2. Phase 1: One comprehensive question covering product, users, type, and research preference
3. Phase 2 (optional): Competitive research via WebSearch + browse with visual evidence, 3-layer synthesis
4. Design Outside Voices: Codex + Claude subagent in parallel
5. Phase 3: Complete proposal with SAFE/RISK breakdown, user chooses which risks to take
6. Phase 4: Drill-downs on specific sections (only if user requests adjustments)
7. Phase 5: Font & color preview HTML page (self-contained, dogfoods the design system)
8. Phase 6: Write DESIGN.md + update CLAUDE.md

### Design Knowledge (Built In)

**Aesthetic directions (10):**
Brutally Minimal, Maximalist Chaos, Retro-Futuristic, Luxury/Refined, Playful/Toy-like, Editorial/Magazine, Brutalist/Raw, Art Deco, Organic/Natural, Industrial/Utilitarian

**Decoration levels:** minimal / intentional / expressive

**Layout approaches:** grid-disciplined / creative-editorial / hybrid

**Color approaches:** restrained / balanced / expressive

**Motion approaches:** minimal-functional / intentional / expressive

**Font recommendations by purpose:**
- Display/Hero: Satoshi, General Sans, Instrument Serif, Fraunces, Clash Grotesk, Cabinet Grotesk
- Body: Instrument Sans, DM Sans, Source Sans 3, Geist, Plus Jakarta Sans, Outfit
- Data/Tables: Geist (tabular-nums), DM Sans (tabular-nums), JetBrains Mono, IBM Plex Mono
- Code: JetBrains Mono, Fira Code, Berkeley Mono, Geist Mono

**Font blacklist:** Papyrus, Comic Sans, Lobster, Impact, Jokerman, Bleeding Cowboys, Permanent Marker, Bradley Hand, Brush Script, Hobo, Trajan, Raleway, Clash Display, Courier New (for body)

**Overused fonts:** Inter, Roboto, Arial, Helvetica, Open Sans, Lato, Montserrat, Poppins (never recommend as primary)

### Three-Layer Synthesis (Competitive Research)

Layer 1 (tried and true): What every product in the category shares — table stakes
Layer 2 (new and popular): What design discourse is saying, what's trending
Layer 3 (first principles): Given this product's users and positioning, should we break from convention?

Layer 3 can produce "Eureka" moments — genuine design insights logged to `~/.gstack/analytics/eureka.jsonl`.

### SAFE/RISK Breakdown

The core proposal structure:

```
SAFE CHOICES (category baseline — users expect these):
  - [2-3 decisions matching category conventions, with rationale]

RISKS (where the product gets its own face):
  - [2-3 deliberate departures from convention]
  - For each risk: what it is, why it works, what it gains, what it costs
```

This is explicitly designed so that every proposal has creative risks — design coherence is table stakes (every product in a category can be coherent and still look identical). The real question is where the product takes creative risks.

### Preview Page

Generated as a self-contained HTML file:
- Loads proposed fonts from Google Fonts
- Dogfoods the color palette throughout
- Shows real product name (not lorem ipsum)
- Font specimen section: each font in its proposed role
- Color swatches with hex values and names
- Sample UI components: buttons (primary, secondary, ghost), cards, form inputs, alerts
- Realistic product mockups based on project type (dashboard → data table, marketing → hero section, etc.)
- Light/dark mode toggle via CSS custom properties
- Responsive layout

The preview page is opened in the user's browser with `open <path>`. If that fails (headless environment), tells the user the path to open manually.

### Coherence Validation

When the user overrides one section, checks if the rest still coheres:
- Brutalist/Minimal aesthetic + expressive motion → gentle nudge
- Bold palette + restrained decoration → gentle nudge
- Editorial layout + data-heavy product → gentle nudge

Never blocks — always accepts the user's final choice.

### DESIGN.md Structure

Written to repo root with sections:
- Product Context (what, who, space/industry, project type)
- Aesthetic Direction (name, decoration level, mood, reference sites)
- Typography (display, body, UI/labels, data/tables, code, loading, scale)
- Color (approach, primary, secondary, neutrals, semantic, dark mode strategy)
- Spacing (base unit, density, scale)
- Layout (approach, columns per breakpoint, max content width, border radius)
- Motion (approach, easing, duration)
- Decisions Log (date, decision, rationale)

### CLAUDE.md Update

Appends a section:
```markdown
## Design System
Always read DESIGN.md before making any visual or UI decisions.
All font choices, colors, spacing, and aesthetic direction are defined there.
Do not deviate without explicit user approval.
In QA mode, flag any code that doesn't match DESIGN.md.
```

### Shortcuts / Technical Debt

- The browse binary check (`NEEDS_SETUP`) is a user-facing prompt that blocks — the user must say "OK" before setup proceeds. This is intentional (binary builds can be heavy) but creates friction.
- Research phase (Phase 2) is optional and relies on browse being available. If browse isn't set up, falls back to WebSearch only, and if WebSearch also fails, uses built-in design knowledge. The degradation path is graceful but the quality of research varies significantly.
- Font loading from Google Fonts in the preview page requires internet access — no offline fallback.
- No automated verification that the DESIGN.md is internally consistent or matches what was proposed in the preview.
- The "conversational tone" and "propose, don't present menus" philosophy means the skill has a lot of branching paths and depends heavily on user engagement to drive toward closure.
