# Features Batch 7 Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Batch:** 7 — Retro, Setup Browser Cookies, Setup Deploy
**Date:** 2026-03-27

---

## Feature 1: Retro (Retrospective)

**SKILL.md:** `retro/SKILL.md`
**Command:** `/retro`
**Type:** Skill-only (bash + git analysis, no separate implementation files)

### Core Implementation

The `/retro` skill is a pure-bash analytics tool that analyzes git history to produce engineering retrospectives. It has **no separate implementation files** — the SKILL.md IS the implementation. This is a deliberate architectural choice: the skill uses git commands directly rather than wrapping them in a separate module.

### Key Files Traced

| File | Purpose |
|------|---------|
| `retro/SKILL.md` | Complete implementation — 1100+ lines of bash/git analysis |
| `~/.gstack/greptile-history.md` | Optional Greptile triage signal (if available) |
| `~/.gstack/analytics/skill-usage.jsonl` | Skill usage telemetry for usage patterns |
| `~/.gstack/retros/global-*.json` | Cross-project retro snapshots |
| `.context/retros/*.json` | Per-repo retro snapshots |

### Data Flow

```
git log → commit parser → metrics computer → narrative generator
                ↓
         JSON snapshot saved to .context/retros/
```

### Argument Parsing (Lines 308-335)

```
/retro           → 7d window (default)
/retro 24h       → 24 hour window
/retro 14d       → 14 day window
/retro 30d       → 30 day window
/retro compare    → current vs prior window
/retro global     → cross-project (reads from ~/.gstack/retros/)
/retro global 14d → cross-project with explicit window
```

**Argument validation:** Rejects invalid arguments with a usage message — no silent failures.

### Git Analysis Pipeline (Step 1)

12 parallel git commands gather raw data:
1. `git log --format="%H|%aN|%ae|%ai|%s" --shortstat` — commits with stats
2. `git log --format="COMMIT:%H|%aN" --numstat` — per-commit LOC breakdown (test vs prod)
3. `git log --format="%at|%aN|%ai|%s" | sort -n` — timestamps for session detection
4. `git log --format="" --name-only` — files changed for hotspot analysis
5. `git log --format="%s" | grep -oE '[#!][0-9]+'` — PR/MR numbers
6. `git log --format="AUTHOR:%aN" --name-only` — per-author file hotspots
7. `git shortlog -sn --no-merges` — per-author commit counts
8. `~/.gstack/greptile-history.md` — Greptile triage data
9. `TODOS.md` — backlog health (if exists)
10. Test file count via `find`
11. Regression test commits via `git log --grep`
12. Skill usage telemetry from JSONL

### Session Detection Algorithm (Step 4)

Uses a **45-minute gap threshold** between consecutive commits to detect work sessions. Sessions are classified:
- **Deep sessions:** 50+ minutes
- **Medium sessions:** 20-50 minutes
- **Micro sessions:** <20 minutes (single commits)

### Midnight Alignment (Lines 321-322)

A subtle and important detail: day/week windows are midnight-aligned to avoid time-zone drift issues:

```bash
# Good — midnight start
git log --since="2026-03-11T00:00:00"

# Bad — wall-clock time at 11pm means 11pm, not midnight
git log --since="2026-03-11"
```

### Metrics Computed (Step 2)

| Metric | Source |
|--------|--------|
| Commits, contributors, PRs | git shortlog / log |
| Total insertions/deletions | --shortstat |
| Test LOC ratio | --numstat filtering test/\|spec/ |
| Active days | unique commit dates |
| Sessions, deep sessions | 45-min gap algorithm |
| LOC/session-hour | sessions × average session length |
| Greptile signal | greptile-history.md parsing |
| Backlog health | TODOS.md if exists |

### Global Retrospective Mode (Lines 806-1091)

The `/retro global` mode is architecturally distinct:

1. Uses `gstack-global-discover` binary to find all AI coding sessions across repos
2. Runs git log on each discovered repo independently
3. Aggregates personal stats across all repos
4. Saves to `~/.gstack/retros/` (not `.context/retros/`)
5. Produces a shareable personal card (designed for X/Twitter)

### Clever Solutions

1. **Midnight-aligned windows** — prevents time-zone drift over week-long retrospectives
2. **45-minute gap session detection** — simple but effective heuristic
3. **Co-Authored-By parsing** — credits AI collaborators separately from human authors
4. **Compare mode** — prior window metrics computed with explicit `--since` AND `--until` to avoid overlap
5. **Personal card format** — left-border-only ASCII box (LLMs can't reliably align right borders)

### Technical Debt / Shortcuts

1. **No caching of git queries** — every retro re-fetches and re-parses all history
2. **TODOS.md parsing is fragile** — relies on specific section naming (`## Completed`)
3. **Greptile history path is hardcoded** — `~/.gstack/greptile-history.md` with no fallback
4. **No handling of bare git repos without origin** — would fail on repos without remotes
5. **GitLab support is incomplete** — MR detection exists but most analysis is git-native

### Error Handling

- Zero commits in window: outputs "zero commits" message, suggests longer window
- Missing telemetry files: skips the row silently
- Failed git commands: caught and reported as "N repos could not be reached"
- Invalid argument format: shows usage message and stops

---

## Feature 2: Setup Browser Cookies

**SKILL.md:** `setup-browser-cookies/SKILL.md`
**Command:** `/setup-browser-cookies`
**Key Files:**
- `browse/src/cookie-import-browser.ts` — decryption engine
- `browse/src/cookie-picker-routes.ts` — HTTP API routes
- `browse/src/cookie-picker-ui.ts` — self-contained HTML UI
- `browse/test/cookie-import-browser.test.ts` — unit tests

### Core Implementation

This feature imports encrypted browser cookies into Playwright sessions for authenticated testing. The architecture has three layers:

```
cookie-picker-ui.ts     → HTML/CSS/JS (runs in user's real browser)
cookie-picker-routes.ts → HTTP API (runs in browse server)
cookie-import-browser.ts → Decryption engine (pure logic, no HTTP)
```

### Cookie Import Pipeline

```
Browser Cookie DB (SQLite) → Decrypt (AES-128-CBC) → Playwright Cookie → page.context().addCookies()
```

### Supported Browsers

| Browser | macOS Data Dir | Linux Data Dir | Keychain Service |
|---------|---------------|----------------|------------------|
| Comet | `Comet/` | — | `Comet Safe Storage` |
| Chrome | `Google/Chrome/` | `google-chrome/` | `Chrome Safe Storage` |
| Chromium | `chromium/` | `chromium/` | `Chromium Safe Storage` |
| Arc | `Arc/User Data/` | — | `Arc Safe Storage` |
| Brave | `BraveSoftware/Brave-Browser/` | `BraveSoftware/Brave-Browser/` | `Brave Safe Storage` |
| Edge | `Microsoft Edge/` | `microsoft-edge/` | `Microsoft Edge Safe Storage` |

### Decryption Pipeline (Lines 7-36 in cookie-import-browser.ts)

The comment block is a **transport diagram** documenting the decryption algorithm:

```
1. Resolve cookie DB path (platform-specific)
2. Derive AES key:
   - macOS v10: Keychain → PBKDF2(..., iter=1003)
   - Linux v10: "peanuts" → PBKDF2(..., iter=1)
   - Linux v11: libsecret → PBKDF2(..., iter=1)
3. For v10/v11 encrypted cookies:
   - Ciphertext = encrypted_value[3:]
   - IV = 16 bytes of 0x20 (space character)
   - AES-128-CBC decrypt
   - Remove PKCS7 padding
   - Skip first 32 bytes (Chromium metadata)
   - Remaining = cookie value (UTF-8)
4. Empty encrypted_value + non-empty value field → use value directly (unencrypted)
5. Chromium epoch → Unix: (microseconds - 11644473600000000) / 1000000
6. sameSite mapping: 0=None, 1=Lax, 2=Strict
```

### Key Security Considerations

1. **Key cache** (Lines 118, 425-431): Derived AES keys are cached per browser to avoid repeated Keychain lookups. Cache is in-memory Map.

2. **Profile validation** (Lines 303-309): Rejects path traversal (`../`) and control characters:
   ```typescript
   if (/[/\\]|\.\./.test(profile) || /[\x00-\x1f]/.test(profile)) {
     throw new CookieImportError(`Invalid profile name: '${profile}'`, 'bad_request');
   }
   ```

3. **SQL injection prevention** (Lines 254-256): Parameterized queries throughout:
   ```typescript
   const placeholders = domains.map(() => '?').join(',');
   db.query(`SELECT ... WHERE host_key IN (${placeholders})`, ...domains);
   ```

4. **No shell interpolation**: Browser names go through a registry lookup, never directly into shell commands.

### Clever Solutions

1. **SQLite locked handling** (Lines 388-416): If DB is locked, copy to `/tmp` with WAL/SHM files, read from copy, schedule cleanup on close. Handles browser-in-use scenario.

2. **Profile display names** (Lines 186-203): Reads Chrome's `Preferences` JSON to get account email — much more useful than "Person 2".

3. **Browser alias system** (Lines 105-112): Supports both full names and common aliases (`chrome`, `google-chrome`, `google-chrome-stable` all resolve to Chrome).

4. **Platform search order** (Lines 317-325): Searches current platform first, then cross-platform as fallback. Avoids finding stale browser installations on dual-boot systems.

5. **Import tracking** (Lines 24-25 in cookie-picker-routes.ts): Maintains `importedDomains` Set to track what's been imported — prevents double-import and enables the "remove" functionality.

### Technical Debt / Shortcuts

1. **No Windows support**: Browser registry only has macOS and Linux entries. Windows would need separate paths and likely DPAPI-based key derivation.

2. **No parallel decryption**: Cookies are decrypted sequentially, not parallelized. For 100+ cookies this could be slow.

3. **Keychain timeout is 10s hardcoded** (Line 470): If user doesn't respond to macOS Keychain dialog in 10 seconds, the operation fails. No retry with longer timeout.

4. **importedDomains is in-memory**: If the browse server restarts, imported cookie state is lost. No persistence.

5. **No domain validation**: Any domain string is accepted — could import cookies for malicious sites if user clicks blindly.

### Error Handling

| Error | Code | Action |
|-------|------|--------|
| Browser not installed | `not_installed` | Shows attempted paths |
| Keychain access denied | `keychain_denied` | retry — user needs to click Allow |
| Keychain timeout | `keychain_timeout` | retry — with longer wait |
| DB locked | `db_locked` | retry — close the browser |
| DB corrupt | `db_corrupt` | fatal — browser may need repair |

### HTTP API Routes

| Route | Method | Purpose |
|-------|--------|---------|
| `/cookie-picker` | GET | Serve HTML picker UI |
| `/cookie-picker/browsers` | GET | List installed browsers |
| `/cookie-picker/profiles` | GET | List profiles for browser |
| `/cookie-picker/domains` | GET | List domains + counts |
| `/cookie-picker/import` | POST | Decrypt + add to Playwright |
| `/cookie-picker/remove` | POST | Clear cookies for domains |
| `/cookie-picker/imported` | GET | Currently imported domains |

### Test Coverage

The test file (`cookie-import-browser.test.ts`) is comprehensive:
- Encryption/decryption round-trip
- Fixture DB with known encrypted values
- Linux v10 and v11 key derivation
- Profile validation (path traversal rejection)
- Unknown browser errors
- Chromium epoch conversion
- sameSite mapping

---

## Feature 3: Setup Deploy

**SKILL.md:** `setup-deploy/SKILL.md`
**Command:** `/setup-deploy`
**Type:** Configuration skill (no separate implementation — pure skill)

### Core Implementation

This is a **configuration-only skill** that detects deployment setup and writes it to `CLAUDE.md`. It does not perform deployments — it configures what `/land-and-deploy` needs to know.

### Data Flow

```
Platform detection → User confirmation → Write to CLAUDE.md
       ↓
/land-and-deploy reads CLAUDE.md for deploy config
```

### Platform Detection (Step 2)

Detects platforms by looking for config files:

```bash
[ -f fly.toml ]                    → Fly.io
[ -f render.yaml ]                 → Render
[ -f vercel.json ] || [ -d .vercel ] → Vercel
[ -f netlify.toml ]                → Netlify
[ -f Procfile ]                    → Heroku
[ -f railway.json ] || [ -f railway.toml ] → Railway
# Plus GitHub Actions workflow detection
```

### Configuration Written to CLAUDE.md (Step 4)

```markdown
## Deploy Configuration (configured by /setup-deploy)
- Platform: {platform}
- Production URL: {url}
- Deploy workflow: {workflow file or "auto-deploy on push"}
- Deploy status command: {command or "HTTP health check"}
- Merge method: {squash/merge/rebase}
- Project type: {web app / API / CLI / library}
- Post-deploy health check: {health check URL or command}

### Custom deploy hooks
- Pre-merge: {command or "none"}
- Deploy trigger: {command or "automatic on push to main"}
- Deploy status: {command or "poll production URL"}
- Health check: {URL or command}
```

### Idempotency Design

The skill is designed to be run multiple times safely:
1. Checks for existing config in CLAUDE.md
2. Offers to overwrite, edit specific fields, or keep existing
3. Clean find-and-replace of the `## Deploy Configuration` section

### Integration with /land-and-deploy

The `land-and-deploy` SKILL.md reads this config in Step 5:

```bash
DEPLOY_CONFIG=$(grep -A 20 "## Deploy Configuration" CLAUDE.md 2>/dev/null)
if [ "$DEPLOY_CONFIG" != "NO_CONFIG" ]; then
  PROD_URL=$(echo "$DEPLOY_CONFIG" | grep -i "production.*url" | ...)
  PLATFORM=$(echo "$DEPLOY_CONFIG" | grep -i "platform" | ...)
fi
```

### Platform-Specific Behavior

| Platform | Health Check | Deploy Status | Notes |
|----------|-------------|---------------|-------|
| Fly.io | `fly status --app {app}` | Poll until `started` | CLI optional |
| Render | Production URL | Poll until HTTP 200 | Auto-deploy on push |
| Vercel | Production URL | Auto | No deploy trigger needed |
| Netlify | Production URL | Auto | No deploy trigger needed |
| Heroku | `heroku releases` | CLI check | CLI optional |
| GitHub Actions | `gh run list` | Poll workflow | Most flexible |
| Custom | User-provided | User-provided | Most flexible |

### Security Considerations

1. **No secret exposure** (Line 458): API keys shown only as `head -c 4` — partial preview only
2. **Confirmation required** (Line 459): Always shows detected config before writing
3. **CLAUDE.md is source of truth** (Line 460): Config lives with the project, not in gstack's config dir

### User Flow

```
1. Check existing config → offer reconfigure/edit/keep
2. Detect platform from files
3. Platform-specific setup (ask for confirmation)
4. Custom deploy hooks via AskUserQuestion
5. Write to CLAUDE.md
6. Verify (curl health check, run deploy status command)
7. Summary output
```

### Error Handling

- Health check fails: Notes it but doesn't block — config is still useful
- Deploy status command fails: Notes it but continues
- No platform detected: Falls back to AskUserQuestion with 5 options

### Limitations

1. **Single deploy target only**: Writes one platform config, assumes single deployment
2. **No rollback config**: Doesn't capture how to roll back
3. **No staging environment**: Only production URL
4. **No secrets management**: Doesn't integrate with env vars or secret stores

---

## Cross-Feature Observations

### Shared Patterns

1. **Idempotent design**: Both `setup-deploy` and `setup-browser-cookies` are designed to be run multiple times safely
2. **Platform abstraction**: All three features abstract platform differences (browsers, deploy platforms) through registries
3. **User confirmation gates**: `setup-deploy` confirms before writing; `setup-browser-cookies` waits for user to click "done" in picker UI
4. **Security-first**: Cookie decryption doesn't expose values in UI; deploy config doesn't print full API keys

### Dependencies Between Features

```
setup-deploy → writes → CLAUDE.md
                      ↓
              land-and-deploy → reads → CLAUDE.md

setup-browser-cookies → uses → browse binary
                              ↓
                         Playwright context
```

`setup-deploy` and `/land-and-deploy` are coupled through CLAUDE.md. If the deploy config format changes, both need to be updated.

### Quality Assessment

| Feature | Completeness | Error Handling | Test Coverage |
|---------|-------------|----------------|---------------|
| Retro | High — comprehensive git analysis | Good — graceful degradation | N/A (skill-only) |
| Setup Browser Cookies | High — 6 browsers, 3 platforms | Excellent — specific error codes | Good — unit tests with fixtures |
| Setup Deploy | Medium — no rollback/staging | Good — warnings not blockers | N/A (skill-only) |

---

## Recommendations

### For Setup Browser Cookies

1. **Consider parallel decryption** for large cookie sets (>50 cookies)
2. **Windows support** would expand the user base significantly
3. **Persistent import state** would survive server restarts

### For Setup Deploy

1. **Add staging environment** support for blue-green deployments
2. **Capture rollback method** — the inverse of the deploy command
3. **Integrate with secret stores** — many teams use Vault/Doppler/Secrets Manager

### For Retro

1. **Cache git query results** — subsequent retros re-use data
2. **Support bare repos** without origin remotes
3. **GitLab MR detection** is implemented but not fully tested
