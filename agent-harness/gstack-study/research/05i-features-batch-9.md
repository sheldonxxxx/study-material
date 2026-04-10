# Features Batch 9 Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Batch:** 9 — Guard (Full Safety), Unfreeze (Unlock), Gstack Upgrade
**Date:** 2026-03-27

---

## Feature 1: Guard (Full Safety)

### What It Does

Guard combines `/careful` (destructive command warnings) and `/freeze` (directory-scoped edit blocking) into a single command. It activates maximum safety during production work or live debugging.

### Architecture

**Composition pattern, not a standalone implementation.** Guard's SKILL.md defines PreToolUse hooks that delegate to sibling skill hook scripts:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../careful/bin/check-careful.sh"
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
```

**Dependency note in SKILL.md:** "This skill references hook scripts from the sibling `/careful` and `/freeze` skill directories. Both must be installed." This is a key architectural constraint - Guard will silently fail if the sibling skills aren't present.

### Setup Flow

1. User runs `/guard`
2. Claude uses `AskUserQuestion` to prompt: "Which directory should edits be restricted to?"
3. The path is resolved to absolute, trailing slash added
4. Written to `~/.gstack/freeze-dir.txt`
5. Two protections reported to user: destructive command warnings + edit boundary

### Key Files

| File | Purpose |
|------|---------|
| `guard/SKILL.md` | Skill definition with hook references |
| `careful/bin/check-careful.sh` | Bash command destructiveness checker |
| `freeze/bin/check-freeze.sh` | Edit/Write path boundary checker |

### check-careful.sh Implementation

The destructive command checker:
- Reads `command` field from tool_input JSON using grep/sed with Python fallback
- Applies safe exception list first: `rm -rf node_modules/.next/dist/__pycache__/.cache/build/.turbo/coverage`
- Pattern checks (returns `permissionDecision: "ask"`):
  - `rm -r` / `rm --recursive`
  - `DROP TABLE` / `DROP DATABASE`
  - `TRUNCATE`
  - `git push --force` / `-f`
  - `git reset --hard`
  - `git checkout .` / `git restore .`
  - `kubectl delete`
  - `docker rm -f` / `docker system prune`
- Logs hook fire events to `~/.gstack/analytics/skill-usage.jsonl` (pattern name only, never command content)

### check-freeze.sh Implementation

The edit boundary checker:
- Reads `file_path` from tool_input JSON (grep/sed + Python fallback)
- Reads freeze boundary from `~/.gstack/freeze-dir.txt`
- If no freeze file exists or is empty, allows everything
- Resolves relative paths to absolute
- Normalizes: removes double slashes, strips trailing slash
- Returns `permissionDecision: "deny"` if path doesn't start with freeze directory

**Subtle issue:** The freeze script normalizes paths but the freeze-dir.txt is written with a trailing slash. The check uses `${FREEZE_DIR}` prefix matching, which means `/src` will match `/src/` (the freeze boundary) but `/src-old` won't match `/src/`. This is the intended behavior per the freeze SKILL.md note: "The trailing `/` on the freeze directory prevents `/src` from matching `/src-old`."

### Clever Solutions

1. **Composition over duplication** — Guard doesn't reimplement careful or freeze logic, it composes existing hooks. This avoids code duplication but creates a hard dependency on sibling skill installation.

2. **Graceful degradation** — Both hook scripts return `{}` (allow) when they can't parse input, preventing accidental blocks on parse failures.

3. **Analytics without PII** — The careful hook logs only the pattern name, never the actual command content.

### Edge Cases & Technical Debt

1. **Silent failure on missing siblings** — If `careful` or `freeze` directories don't exist, the hook commands will fail silently (bash will print an error but the tool will proceed). The skill documentation notes the dependency but there's no runtime check.

2. **Bash commands bypass freeze** — The freeze SKILL.md explicitly states: "This prevents accidental edits, not a security boundary — Bash commands like `sed` can still modify files outside the boundary." This is a known limitation.

3. **No state file cleanup on session end** — The freeze boundary persists in `~/.gstack/freeze-dir.txt` until `/unfreeze` is run. If the session ends without unfreezing, the state file remains.

---

## Feature 2: Unfreeze (Unlock)

### What It Does

Removes the freeze boundary set by `/freeze`, restoring full editing access.

### Architecture

**Minimal implementation** — Unfreeze is the simplest skill in the batch. It:
1. Checks if `~/.gstack/freeze-dir.txt` exists
2. If yes, reads and reports the previous boundary
3. Deletes the state file
4. Reports "Edits are now allowed everywhere"
5. If no state file, reports "No freeze boundary was set"

### Key Insight

The `/freeze` hooks remain registered for the session after `/unfreeze` runs. They will continue to fire on Edit/Write operations, but since the state file no longer exists, `check-freeze.sh` returns `{}` (allow) for every path.

This is explicitly noted in the SKILL.md: "Note that `/freeze` hooks are still registered for the session — they will just allow everything since no state file exists."

### Files

| File | Purpose |
|------|---------|
| `unfreeze/SKILL.md` | Skill definition (setup flow only) |

No hook scripts, no bin directory. Pure bash in the SKILL.md itself.

### Edge Cases

1. **Race condition** — If multiple Claude sessions share the same `~/.gstack` state directory (possible in certain configurations), one session's `/unfreeze` would affect another's freeze state.

---

## Feature 3: Gstack Upgrade

### What It Does

Self-updater that upgrades gstack to the latest version. Handles multiple install types (global git, local git, vendored), syncs local copies, shows changelog, and supports auto-upgrade with escalating snooze backoff.

### Architecture

**Complex multi-step flow** with seven steps:

#### Step 1: User Prompt (or Auto-Upgrade)

Checks `GSTACK_AUTO_UPGRADE` env var and `~/.gstack/config.yaml auto_upgrade` setting. If auto-upgrade is enabled, skips `AskUserQuestion`. Otherwise presents four options:
- "Yes, upgrade now"
- "Always keep me up to date" (sets auto_upgrade true)
- "Not now" (writes escalating snooze)
- "Never ask again" (sets update_check false)

**Snooze backoff implementation:**
- Level 1: 24 hours
- Level 2: 48 hours
- Level 3+: 7 days
- New version resets level to 0

#### Step 2: Detect Install Type

Scans for gstack in priority order:
1. `$HOME/.claude/skills/gstack/.git` → global-git
2. `$HOME/.gstack/repos/gstack/.git` → global-git
3. `.claude/skills/gstack/.git` → local-git
4. `.agents/skills/gstack/.git` → local-git
5. `.claude/skills/gstack` (no .git) → vendored
6. `$HOME/.claude/skills/gstack` (no .git) → vendored-global

This ordering matters — local-git takes precedence over vendored when both exist.

#### Step 3: Save Old Version

Reads `VERSION` file from install directory before upgrading.

#### Step 4: Upgrade by Install Type

**For git installs:** `git stash → git fetch → git reset --hard origin/main → ./setup`

**For vendored installs:** Fresh `git clone --depth 1` to temp dir, then swap:
```bash
mv "$INSTALL_DIR" "$INSTALL_DIR.bak"
mv "$TMP_DIR/gstack" "$INSTALL_DIR"
cd "$INSTALL_DIR" && ./setup
rm -rf "$INSTALL_DIR.bak" "$TMP_DIR"
```

**Forced depth=1 clone is clever** — minimal download for vendored users. But it discards all git history, making it impossible to `git pull` future updates to a vendored install. Each vendored upgrade is effectively a fresh download.

#### Step 4.5: Sync Local Vendored Copy

After upgrading the primary install, checks if there's also a local vendored copy in the current repo:
```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
if [ -n "$_ROOT" ] && [ -d "$_ROOT/.claude/skills/gstack" ]; then
  _RESOLVED_LOCAL=$(cd "$_ROOT/.claude/skills/gstack" && pwd -P)
  _RESOLVED_PRIMARY=$(cd "$INSTALL_DIR" && pwd -P)
  if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
    LOCAL_GSTACK="$_ROOT/.claude/skills/gstack"
  fi
fi
```

If different from primary, copies from primary (stripping `.git`) and runs `./setup`. This handles the common case where a developer has `~/.claude/skills/gstack` as the global install and `.claude/skills/gstack` as a vendored repo-local copy.

**Backup/restore on failure:** If `./setup` fails, restores from `.bak` and warns user.

#### Step 5: Write Marker + Clear Cache

```bash
mkdir -p ~/.gstack
echo "$OLD_VERSION" > ~/.gstack/just-upgraded-from
rm -f ~/.gstack/last-update-check
rm -f ~/.gstack/update-snoozed
```

The `just-upgraded-from` marker causes `gstack-update-check` to output `JUST_UPGRADED` on next run, triggering the "What's New" display.

#### Step 6: Show What's New

Reads `CHANGELOG.md` from the install directory, extracts entries between old and new version, formats as 5-7 bullets grouped by theme.

#### Step 7: Continue

After showing changelog, continues with whatever skill was originally invoked. The upgrade preamble is designed to be prepended to any skill's SKILL.md.

### Key Binaries

| Binary | Purpose |
|--------|---------|
| `bin/gstack-update-check` | Version checker with caching, snooze, telemetry |
| `bin/gstack-config` | YAML config reader/writer for `~/.gstack/config.yaml` |

### gstack-update-check Implementation

**Three output types:**
- `JUST_UPGRADED <old> <new>` — from marker file after upgrade
- `UPGRADE_AVAILABLE <old> <new>` — remote VERSION differs from local
- `UP_TO_DATE <version>` — cached, or `UP_TO_DATE` written to cache file

**Cache TTLs:**
- `UP_TO_DATE`: 60 minutes (fast detection of new releases)
- `UP_UPGRADE_AVAILABLE`: 720 minutes (less aggressive refresh when update available)

**Snooze check:** Before outputting `UPGRADE_AVAILABLE`, checks snooze file. If not expired, silent.

**One-time migrations:** The script runs a Codex description size fix on first execution (heals oversized descriptions >1024 chars by deleting `.agents/skills/*/SKILL.md` files).

**Supabase telemetry ping:** Fires a background POST to `/functions/v1/update-check` (respects telemetry opt-out).

**Version validation:** Rejects remote VERSION content that doesn't match `^[0-9]+\.[0-9.]+$` to avoid HTML error pages being treated as version numbers.

### gstack-config Implementation

Minimal YAML config utility:
- `get <key>` — reads value using `grep | tail-1 | awk`
- `set <key> <value>` — uses `sed -i ''` on macOS (not portable to Linux without platform detection)
- `list` — cats the config file

**Platform issue:** Uses `sed -i ''` (macOS syntax). On Linux, this creates a backup file named `''` instead of editing in place. This is a portability bug.

### Edge Cases & Technical Debt

1. **Vendored = no git history** — Depth-1 clone means vendored installs lose all history. Future vendored upgrades must be full clones again.

2. **Snooze persists across upgrades** — Step 5 clears `update-snoozed`, but the snooze file is checked during the upgrade flow itself (Step 1). If a user selects "Not now" during upgrade, the snooze is recorded but then immediately cleared by Step 5. This seems unintentional — snoozing during upgrade should probably be preserved.

3. **sed -i '' portability** — `gstack-config set` uses macOS-specific sed syntax.

4. **No rollback mechanism** — For git installs, `./setup` failure triggers a restore from stash. For vendored installs, backup is used. But there's no mechanism to roll back if `./setup` modifies system state (like global symlinks) that can't be undone.

5. **LOCAL_GSTACK sync uses cp -Rf** — If the local vendored copy has uncommitted changes, they're overwritten without warning.

---

## Cross-Feature Observations

### Shared State Pattern

All three features use `~/.gstack/` as the state directory:
- `freeze-dir.txt` — freeze boundary path
- `config.yaml` — user preferences (auto_upgrade, update_check, telemetry)
- `last-update-check` — upgrade cache
- `update-snoozed` — snooze state
- `just-upgraded-from` — post-upgrade marker
- `skill-usage.jsonl` — analytics

### Analytics Opt-Out

All three skills write to `~/.gstack/analytics/skill-usage.jsonl` with event data. This happens even when telemetry is nominally "off" — the analytics file is just a local log, not sent anywhere unless Supabase telemetry is configured.

### Composition Pattern

Guard demonstrates a composition pattern where a skill aggregates sibling skills. This is elegant but fragile — any hook script path change breaks Guard without warning.

---

## Summary Table

| Aspect | Guard | Unfreeze | Gstack Upgrade |
|--------|-------|----------|----------------|
| **Lines of code** | ~50 (SKILL.md only) | ~15 (SKILL.md only) | ~250 (SKILL.md + 2 binaries) |
| **Hook-based** | Yes (composes siblings) | No | No |
| **State persistence** | `~/.gstack/freeze-dir.txt` | Deletes state file | `~/.gstack/config.yaml`, markers |
| **Complexity** | Low (composition) | Minimal | High (multi-step flow) |
| **Failure modes** | Silent (missing siblings) | None significant | Backup/restore, sed portability |
| **Install types handled** | N/A | N/A | 6 (git + vendored variants) |
