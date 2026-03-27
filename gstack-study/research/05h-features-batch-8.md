# Deep Dive: Features Batch 8

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Batch:** 8 — Codex (Second Opinion), Careful (Safety Guardrails), Freeze (Edit Lock)
**Date:** 2026-03-27

---

## Feature 22: Codex (Second Opinion)

### Classification
**Type:** External AI Integration / Code Review
**Skill Directory:** `codex/`
**Command:** `/codex`

### What It Does
Wraps OpenAI Codex CLI as a multi-mode second opinion skill. Three distinct modes:

1. **Review Mode** (`/codex review`) — Independent diff review with pass/fail gate
2. **Challenge Mode** (`/codex challenge`) — Adversarial mode that tries to break code
3. **Consult Mode** (`/codex`) — Ask Codex anything with session continuity

### Core Implementation

**Entry Point:** `codex/SKILL.md` (generated from `codex/SKILL.md.tmpl`)

**Binary Detection (Step 0):**
```bash
CODEX_BIN=$(which codex 2>/dev/null || echo "")
[ -z "$CODEX_BIN" ] && echo "NOT_FOUND" || echo "FOUND: $CODEX_BIN"
```
Installs via `npm install -g @openai/codex`. No hardcoded API keys — Codex uses its own auth from `~/.codex/` config.

**Review Mode Flow:**
- Detects base branch via GitHub CLI (`gh pr view`) or GitLab CLI (`glab mr view`) or git-native fallback
- Runs: `codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached`
- Gate verdict: FAIL if `[P1]` markers present, PASS otherwise
- **Cross-model comparison:** If `/review` (Claude's own review) ran earlier, Codex compares findings with Claude's

**Challenge Mode Flow:**
- Uses `codex exec` with JSONL streaming output
- Parses JSONL events to extract `[codex thinking]` reasoning traces
- Constructs adversarial prompt: "Your job is to find ways this code will fail in production. Think like an attacker..."

**Consult Mode Flow:**
- Supports session continuity via `thread.started` event's `thread_id`
- Saves session ID to `.context/codex-session-id`
- Prompts with persona: "You are a brutally honest technical reviewer..."

### Key Technical Decisions

**No Model Hardcoding:** Uses Codex's current default (frontier agentic coding model). Passes `-m` flag through if user specifies one. This means /codex automatically benefits from OpenAI model improvements without code changes.

**Maximum Reasoning Effort:** All modes use `model_reasoning_effort="xhigh"` — maximum reasoning power for quality analysis.

**Web Search Cached:** All commands use `--enable web_search_cached` for looking up docs/APIs during review without extra cost or latency.

**Read-Only Sandbox:** All Codex runs use `-s read-only` flag. The skill itself is read-only and never modifies files.

**5-Minute Timeout:** All bash calls to codex use `timeout: 300000` to prevent hanging on large diffs.

**Token Cost Tracking:** Parses `tokens used\nN` from stderr. Displays as `Tokens: N | Est. cost: ~$X.XX`.

### Cross-Model Analysis

When both `/review` (Claude Code) and `/codex review` have run, Codex produces a comparison:

```
CROSS-MODEL ANALYSIS:
  Both found: [overlapping findings]
  Only Codex found: [unique to Codex]
  Only Claude found: [unique to Claude's /review]
  Agreement rate: X% (N/M total unique findings overlap)
```

This is valuable because different AI models catch different classes of issues.

### Error Handling

| Error | Detection | Response |
|-------|-----------|----------|
| Binary not found | Step 0 `which codex` | Stop with install instructions |
| Auth error | stderr output | "Run `codex login`" |
| Timeout (5 min) | Bash timeout exit 124 | "Try smaller scope" |
| Empty response | `$TMPRESP` empty | "Check stderr for errors" |
| Session resume failure | resume command fails | Delete session, start fresh |

### Review Log Persistence

Uses `gstack-review-log` to persist results:
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-review","timestamp":"TIMESTAMP","status":"STATUS","gate":"GATE","findings":N,"findings_fixed":N,"commit":"..."}'
```

### Plan File Integration

When run in plan mode, Codex updates `## GSTACK REVIEW REPORT` section in the plan file. The report table tracks all review types: CEO Review, Codex Review, Eng Review, Design Review.

### E2E Test Coverage

`test/codex-e2e.test.ts` tests Codex with diff-based selection:
- `codex-discover-skill` — Verifies Codex can list available skills
- `codex-review-findings` — Verifies Codex can run a diff review and produce findings

Tests run with `EVALS=1` and use Codex's own auth. Accepts timeout (exit 124/137) as non-failure since that's a Codex CLI performance issue.

### Observations

**Strengths:**
- Clean separation between Claude's review (`/review`) and Codex's review — different models, different catches
- Session continuity in consult mode enables follow-up conversations
- JSONL parsing for challenge mode exposes internal reasoning traces (`[codex thinking]`), not just final output
- Diff-based test selection aligns with the project's evaluation philosophy

**Shortcuts/Technical Debt:**
- Cost estimation is token-based but actual cost varies by model pricing
- The `codex-e2e.test.ts` mentions tokens tracking but doesn't actually report cost_usd (set to 0 with comment "Codex doesn't report cost in the same way")
- Plan file review report uses grep-based section replacement which could race with concurrent edits

**Edge Cases:**
- When no diff exists, falls back to plan file review (with warning about cross-project contamination)
- Session resume failure gracefully degrades to fresh session
- Large diffs gracefully timeout with user-friendly message

---

## Feature 23: Careful (Safety Guardrails)

### Classification
**Type:** Safety / PreToolUse Hook
**Skill Directory:** `careful/`
**Command:** `/careful`

### What It Does
Warns before destructive bash commands: `rm -rf`, `DROP TABLE`, force-push, `git reset --hard`, `kubectl delete`, `docker rm -f`. User can override each warning. Active for the session duration.

### Core Implementation

**Hook Configuration (from SKILL.md frontmatter):**
```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-careful.sh"
          statusMessage: "Checking for destructive commands..."
```

**Hook Script:** `careful/bin/check-careful.sh`

The script:
1. Reads JSON from stdin containing `tool_input.command`
2. Extracts command via grep/sed (99% case) or Python fallback (escaped quotes)
3. Checks against destructive patterns
4. Returns `{"permissionDecision":"ask","message":"..."}` to warn, or `{}` to allow

### Destructive Patterns Protected

| Pattern | Detection | Warning |
|---------|-----------|---------|
| `rm -rf`, `rm -r`, `rm --recursive` | `grep -qE 'rm\s+(-[a-zA-Z]*r\|--recursive)'` | "Destructive: recursive delete" |
| `DROP TABLE`, `DROP DATABASE` | Case-insensitive SQL check | "Destructive: SQL DROP detected" |
| `TRUNCATE` | Word boundary regex | "Destructive: SQL TRUNCATE detected" |
| `git push --force` / `-f` | `grep -qE 'git\s+push\s+.*(-f\b\|--force)'` | "Destructive: git force-push rewrites history" |
| `git reset --hard` | `grep -qE 'git\s+reset\s+--hard'` | "Destructive: discards uncommitted changes" |
| `git checkout .` / `git restore .` | `grep -qE 'git\s+(checkout\|restore)\s+\.'` | "Destructive: discards working tree changes" |
| `kubectl delete` | `grep -qE 'kubectl\s+delete'` | "Destructive: removes K8s resources" |
| `docker rm -f` / `docker system prune` | `grep -qE 'docker\s+(rm\s+-f\|system\s+prune)'` | "Destructive: Docker force-remove or prune" |

### Safe Exceptions

These patterns are allowed WITHOUT warning:
- `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

The safe exception logic parses rm arguments and checks each target against the allowlist. Flags like `-f` are skipped. If ALL targets are safe, the command is allowed.

### Return Value Design

- **Warn (destructive):** `{"permissionDecision":"ask","message":"[careful] ..."}`
  - User sees warning and can choose to proceed or cancel
- **Allow (safe):** `{}`
  - Empty object = no intervention = command runs normally

This is `permissionDecision: "ask"` not `"deny"` — user retains control, can override.

### Telemetry

Logs hook fire events to `~/.gstack/analytics/skill-usage.jsonl`:
```bash
echo '{"event":"hook_fire","skill":"careful","pattern":"<pattern>","ts":"...","repo":"..."}' >> ~/.gstack/analytics/skill-usage.jsonl
```

Note: Only pattern name is logged, NEVER the actual command content (privacy).

### Observations

**Strengths:**
- Graceful degradation: if JSON parsing fails (no command found), allows the command
- Python fallback handles edge cases with escaped quotes in JSON
- Safe exceptions prevent alert fatigue for common build-cleanup commands
- Pattern logging without command content protects user privacy
- Case-insensitive SQL matching catches `drop table`, `DROP TABLE`, `Drop Table`

**Shortcuts/Technical Debt:**
- Single-pattern detection: only sets one warning even if multiple destructive patterns match (due to `if [ -z "$WARN" ]` chain)
- No protection against: `rsync --delete`, `dd`, `shred`, `terraform destroy`, `az vm delete` — narrower than full destructive category
- The `--recursive` flag detection is imprecise: `rm -rf` detected but `rm -fr` would not match the regex (though this is a very unusual ordering)

**Edge Cases:**
- Commands inside strings or comments are still flagged (e.g., `echo "rm -rf /"` would trigger warning) — conservative by design
- Aliased commands (e.g., `alias rm='rm -rf'`) bypass detection since hooks see the expanded command
- Interactive commands (e.g., `rm -i`) not protected but are inherently safer

---

## Feature 24: Freeze (Edit Lock)

### Classification
**Type:** Safety / PreToolUse Hook
**Skill Directory:** `freeze/`
**Command:** `/freeze`

### What It Does
Restricts Edit and Write operations to a specific directory for the session. Any Edit/Write to a file outside the allowed path is BLOCKED (not warned). Use `/unfreeze` to remove the boundary or end session.

### Core Implementation

**Hook Configuration (from SKILL.md frontmatter):**
```yaml
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-freeze.sh"
          statusMessage: "Checking freeze boundary..."
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-freeze.sh"
          statusMessage: "Checking freeze boundary..."
```

**Hook Script:** `freeze/bin/check-freeze.sh`

The script:
1. Reads freeze directory from `~/.gstack/freeze-dir.txt` (created by `/freeze` or `/guard`)
2. Extracts `file_path` from tool_input JSON (grep/sed or Python fallback)
3. Resolves to absolute path, normalizes double slashes
4. Returns `{"permissionDecision":"deny","message":"..."}` if outside boundary

### Setup Flow

1. User runs `/freeze`
2. Skill asks: "Which directory should I restrict edits to?"
3. User provides path
4. Skill resolves to absolute, adds trailing slash, saves to `~/.gstack/freeze-dir.txt`

```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
FREEZE_DIR="${FREEZE_DIR%/}/"
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
```

### Trailing Slash Security

The trailing `/` is critical for security:
- Without trailing slash: `/src` would match `/src-old`
- With trailing slash: `/src/` only matches `/src/`, `/src/foo`, not `/src-old`

This prevents prefix-matching bypass.

### Return Value Design

- **Block (outside boundary):** `{"permissionDecision":"deny","message":"[freeze] Blocked: %s is outside the freeze boundary (%s). Only edits within the frozen directory are allowed."}`
- **Allow (inside boundary):** `{}`

Unlike `/careful` which uses `ask`, `/freeze` uses `deny` — the operation is blocked, not just warned.

### What Is Protected vs Unprotected

**Protected:** Edit and Write tools
**Unprotected:** Read, Bash, Glob, Grep, and all other tools

The SKILL.md explicitly notes: "This prevents accidental edits, not a security boundary — Bash commands like `sed` can still modify files outside the boundary."

This is a deliberate tradeoff: maximum protection against the most common accidental edit mistakes without breaking legitimate bash-based workflows.

### State Persistence

- **Session-scoped:** Freeze state persists via `~/.gstack/freeze-dir.txt` across tool calls within the session
- **Lifetime:** Until `/unfreeze` clears it, or session ends
- **Path:** `~/.gstack/` (configurable via `CLAUDE_PLUGIN_DATA` env var)

### Observations

**Strengths:**
- Simple, effective, low-noise — blocks the most common mistake (accidental edits outside scope) without constant warnings
- Trailing slash prevents prefix-matching bypass attacks
- Graceful degradation: if no freeze file, no state, or can't parse path, allows the operation (fail open for parse errors)
- Clear error message tells user both the blocked path AND the allowed boundary

**Shortcuts/Technical Debt:**
- No protection against `Bash` with `sed`, `echo >`, `>>`, `tee`, etc. — user must be disciplined
- `sed -i` would bypass the freeze entirely since it's a bash command
- No directory-level protection for Read — could still read sensitive files outside frozen scope
- The `CLAUDE_PLUGIN_DATA` fallback to `$HOME/.gstack` means freeze state could leak between projects using different plugin data dirs

**Edge Cases:**
- Relative paths are resolved to absolute before comparison
- Double slashes normalized away
- Empty freeze file or no freeze file = allow everything (fail-open)
- Symlinks: the resolved path is checked, not the original path — symlink traversal could bypass

---

## Feature 25: Guard (Full Safety Mode)

### Classification
**Type:** Safety / Combined Skill
**Skill Directory:** `guard/`
**Command:** `/guard`

### What It Does
Combines `/careful` and `/freeze` into a single command for maximum safety when touching prod or debugging live systems. Asks for a directory to freeze, enables both destructive command warnings and edit boundary enforcement.

### Implementation

The `/guard` skill simply references the sibling skill hook scripts:
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

This is a clean composition pattern: `guard` doesn't implement its own hooks, it composes the hooks from `careful` and `freeze`. The sibling reference `${CLAUDE_SKILL_DIR}/../careful/bin/` assumes all three skills are installed together (which they are by the gstack setup script).

### Feature Interactions

| Feature | Careful | Freeze | Guard |
|---------|---------|--------|-------|
| Destructive command warning | Yes | No | Yes |
| Edit boundary | No | Yes | Yes |
| `permissionDecision` for violations | `ask` | `deny` | Both |
| Requires directory argument | No | Yes | Yes |

---

## Feature 26: Unfreeze (Edit Unlock)

### Classification
**Type:** Safety / State Clearer
**Skill Directory:** `unfreeze/`
**Command:** `/unfreeze`

### What It Does
Clears the freeze boundary set by `/freeze`, allowing edits to all directories again. Does NOT disable `/careful` — only removes the freeze boundary.

### Implementation

```bash
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
if [ -f "$STATE_DIR/freeze-dir.txt" ]; then
  PREV=$(cat "$STATE_DIR/freeze-dir.txt")
  rm -f "$STATE_DIR/freeze-dir.txt"
  echo "Freeze boundary cleared (was: $PREV). Edits are now allowed everywhere."
else
  echo "No freeze boundary was set."
fi
```

Note: Reports what the previous boundary was (helpful confirmation) before deleting.

### Observations

- Clear separation: `/unfreeze` only clears freeze, doesn't touch careful
- Reports previous state even after clearing (user confirmation)
- Idempotent: if no freeze set, just reports "No freeze boundary was set"
- The hooks themselves remain registered for the session — they just allow everything since no state file exists

---

## Cross-Feature Analysis

### Design Pattern: PreToolUse Hook Composition

All three safety features use Claude Code's PreToolUse hook system. The hook scripts are:
- **Standalone bash scripts** — not TypeScript, not dependent on the skill system
- **Read JSON from stdin** — tool_input structure provided by Claude Code
- **Return permission decision** — `{}` (allow), `{"permissionDecision":"ask",...}` (warn), or `{"permissionDecision":"deny",...}` (block)
- **Stateless** — each invocation reads the state file fresh

This is a clean separation: hook scripts are pure functions (input JSON + state file -> output JSON), making them easy to test and reason about.

### State Management Pattern

| State | Storage | Format |
|-------|---------|--------|
| Freeze boundary | `~/.gstack/freeze-dir.txt` | Single path with trailing slash |
| Freeze not set | No file | Hook allows all |
| Telemetry | `~/.gstack/analytics/skill-usage.jsonl` | JSONL, one event per line |

The state file approach means:
- State survives across tool calls in a session
- State does NOT persist across sessions (intentional — freeze is session-scoped)
- No complex state management needed

### Security Properties

| Property | Achieved? | Mechanism |
|----------|-----------|-----------|
| Blocks Edit outside boundary | Yes | PreToolUse hook with `deny` |
| Blocks Write outside boundary | Yes | PreToolUse hook with `deny` |
| Warns before destructive Bash | Yes | PreToolUse hook with `ask` |
| Prevents all file modifications outside | No | Bash commands like `sed` bypass |
| Security boundary against malicious actor | No | Designed for accidents, not attacks |

The SKILL.md explicitly acknowledges this: "This prevents accidental edits, not a security boundary."

### Telemetry Without Privacy Leakage

Both `/careful` and `/freeze` log hook fires to analytics, but:
- `/careful`: Only logs pattern name (`rm_recursive`, `drop_table`), NEVER the actual command
- `/freeze`: Only logs that a deny occurred, NOT the path that was denied

This is a thoughtful privacy decision — the team can see that safety features are being used without capturing potentially sensitive command/filepath data.

### Error Handling Philosophy

Both hook scripts follow "fail open for uncertainty":
- Can't parse JSON input? Allow the operation
- No state file? Allow the operation
- Empty state? Allow the operation

This prevents the safety system from blocking legitimate work due to parse errors or misconfiguration. The cost is a small window where a genuinely destructive command could slip through if the hook fails, but the alternative (fail closed causing false positives) would be worse for usability.

### Testing Gaps

- No dedicated E2E tests for `/careful` hook — the destructive command patterns are implicitly tested via usage
- No dedicated E2E tests for `/freeze` hook — edit boundary enforcement is implicitly tested
- The safety-critical nature of these hooks suggests they might benefit from unit tests with various JSON input shapes
