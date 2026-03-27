# Security and Performance Analysis

## Overview

**Project:** get-shit-done-cc (GSD)
**Type:** Local CLI workflow orchestration tool
**Analysis Date:** 2026-03-26
**Files Examined:** `get-shit-done/bin/lib/security.cjs`, `get-shit-done/bin/lib/core.cjs`, `get-shit-done/bin/lib/config.cjs`, `get-shit-done/bin/lib/commands.cjs`, `hooks/gsd-prompt-guard.js`, `hooks/gsd-workflow-guard.js`, `hooks/gsd-context-monitor.js`, `SECURITY.md`

---

## 1. Authentication and Authorization

**Not applicable.** GSD is a local CLI tool that operates within a single developer's workspace. There is:

- No user authentication (single-user local tool)
- No authorization model (operates on behalf of the invoking user)
- No multi-tenant or network-accessible components
- No secrets stored in the GSD codebase itself

### Secrets Management

GSD uses environment variables for external API keys and supports a file-based key discovery pattern:

| API Service | Environment Variable | Key File Location |
|-------------|---------------------|-------------------|
| Brave Search | `BRAVE_API_KEY` | `~/.gsd/brave_api_key` |
| Firecrawl | `FIRECRAWL_API_KEY` | `~/.gsd/firecrawl_api_key` |
| Exa Search | `EXA_API_KEY` | `~/.gsd/exa_api_key` |

**Key file discovery** in `config.cjs`:
```javascript
const hasBraveSearch = !!(process.env.BRAVE_API_KEY || fs.existsSync(braveKeyFile));
```

- API keys are read from environment or `~/.gsd/<service>_api_key` files
- Keys are never committed to git (`.planning/` is gitignored)
- Keys are not stored in config.json

---

## 2. Input Validation and Sanitization

### 2.1 Path Traversal Prevention

**Module:** `security.cjs` - `validatePath()`, `requireSafePath()`

The `validatePath()` function is the primary defense against path traversal attacks:

```javascript
function validatePath(filePath, baseDir, opts = {}) {
  // 1. Null byte rejection
  if (filePath.includes('\0')) {
    return { safe: false, resolved: '', error: 'Path contains null bytes' };
  }

  // 2. Realpath resolution to resolve symlinks (macOS /var -> /private/var)
  resolvedBase = fs.realpathSync(path.resolve(baseDir));

  // 3. Check containment: resolved path must start with base + sep
  if (resolvedPath !== resolvedBase && !normalizedPath.startsWith(normalizedBase)) {
    return { safe: false, ... };
  }
}
```

**Defenses:**
- Null byte rejection prevents null-byte injection
- `fs.realpathSync()` resolves symlinks before comparison
- Normalized path comparison ensures directory containment
- Supports `allowAbsolute` option for controlled absolute path usage

**Usage in codebase:**
- `requireSafePath()` wraps `validatePath()` for CLI commands (throws on invalid)
- `cmdVerifyPathExists()` rejects null bytes before path resolution
- `cmdFrontmatterGet/Set()` reject null bytes in file paths
- `setActiveWorkstream()` validates workstream names via regex: `/^[a-zA-Z0-9_-]+$/`

### 2.2 Prompt Injection Detection

**Module:** `security.cjs` - `scanForInjection()`, `sanitizeForPrompt()`, `sanitizeForDisplay()`

GSD's core threat model: user-controlled text (PRDs, commit messages, todo content) flows into markdown files that become LLM agent prompts. These files could be vehicle for indirect prompt injection.

**Detection patterns** (`INJECTION_PATTERNS` array, 14 patterns):

| Category | Patterns |
|----------|----------|
| Instruction override | `ignore previous instructions`, `disregard previous`, `forget your instructions`, `override system prompt` |
| Role manipulation | `you are now a`, `act as a`, `pretend you are`, `from now on you are` |
| Prompt extraction | `print your system prompt`, `reveal your instructions`, `output your prompt` |
| Hidden markers | `<system>`, `<assistant>`, `<human>`, `[SYSTEM]`, `[INST]`, `<<SYS>>` |
| Exfiltration | `curl to https://`, `base64 and send`, `fetch to external URL` |
| Tool manipulation | `run bash tool`, `execute shell command` |

**Sanitization functions:**
```javascript
// Strips zero-width characters: \u200B-\u200F, \u2028-\u202F, \uFEFF, \u00AD
sanitized = sanitized.replace(/[\u200B-\u200F\u2028-\u202F\uFEFF\u00AD]/g, '');

// Neutralizes XML tags: <system> -> ＜system-text＞
// (preserves <instructions> which GSD uses legitimately)
sanitized = sanitized.replace(/<(\/?)(?:system|assistant|human)>/gi, '＜${slash}system-text＞');

// Neutralizes bracket markers: [SYSTEM] -> [SYSTEM-TEXT]
sanitized = sanitized.replace(/\[(SYSTEM|INST)\]/gi, '[$1-TEXT]');
```

**Context-aware allowlisting:** Generic types like `Promise<User | null>` do not trigger false positives (the regex requires `>` to close the tag, not just whitespace).

### 2.3 Shell Argument Validation

**Module:** `security.cjs` - `validateShellArg()`

```javascript
function validateShellArg(value, label) {
  // Rejects null bytes
  if (value.includes('\0')) { throw new Error(...); }

  // Rejects command substitution: both $() and backticks
  if (/[$`]/.test(value) && /\$\(|`/.test(value)) {
    throw new Error('contains potential command substitution');
  }
  return value;
}
```

Note: The pattern `[/[$`]/.test(value) && /\$\(|`/.test(value)]` checks for dollar sign or backtick combined with `$()` or backtick - but this logic is slightly flawed (any `$` combined with `$()` would trigger). However, this is defense-in-depth; the codebase uses `spawnSync` with array-based `execFileSync` for git operations, which avoids shell interpretation entirely.

**Git operations use array-based execFileSync** in `core.cjs`:
```javascript
execFileSync('git', ['check-ignore', '-q', '--no-index', '--', targetPath], {
  cwd, stdio: 'pipe',
});
```

### 2.4 JSON Safety

**Module:** `security.cjs` - `safeJsonParse()`

```javascript
function safeJsonParse(text, opts = {}) {
  const maxLength = opts.maxLength || 1048576; // 1MB default

  if (text.length > maxLength) {
    return { ok: false, error: `${label}: input exceeds ${maxLength} byte limit` };
  }

  try {
    return { ok: true, value: JSON.parse(text) };
  } catch (err) {
    return { ok: false, error: `${label}: parse error — ${err.message}` };
  }
}
```

- Default 1MB size limit prevents DoS via large payloads
- All JSON.parse calls in the codebase use this wrapper
- Used for config file parsing, template fields, frontmatter JSON values

### 2.5 Phase Number and Field Name Validation

**Phase number validation** (`validatePhaseNumber()`):
- Accepts: `1`, `12A`, `2.1`, `12.3.1`, `PROJ-42`, `AUTH-101`
- Rejects: shell injection (`1; rm -rf /`), path traversal (`../../etc/passwd`), script injection (`<script>`)

**Field name validation** (`validateFieldName()`):
- Must start with letter, max 60 chars
- Allows: alphanumeric, spaces, hyphens, underscores, dots, slashes
- Rejects: regex metacharacters (`.*`, `(group)`, `{1,5}`)

---

## 3. Security Hardening Measures

### 3.1 Claude Code Hooks (Defense-in-Depth)

GSD installs Claude Code hooks that run before/after tool use:

**gsd-prompt-guard.js** (PreToolUse hook):
- Scans Write/Edit operations targeting `.planning/` files
- Checks for prompt injection patterns in content being written
- Advisory warning (does NOT block) — only injects context for agent awareness
- Rationale: blocking would cause false-positive deadlocks; the hook surfaces suspicious content

```javascript
// Triggered on Write/Edit to .planning/ files
// Advisory output (not blocking):
hookSpecificOutput: {
  hookEventName: 'PreToolUse',
  additionalContext: `\u26a0\ufe0f PROMPT INJECTION WARNING: ...`
}
```

**gsd-workflow-guard.js** (PreToolUse hook):
- Detects when Claude edits files outside `.planning/` without an active GSD workflow
- Advisory warning to nudge toward `/gsd:quick` or `/gsd:fast`
- Configurable via `hooks.workflow_guard: true` in config.json
- Allows edits to `.planning/` files, `.gitignore`, `.env`, `CLAUDE.md`, etc.

**gsd-context-monitor.js** (PostToolUse/AfterTool hook):
- Monitors context usage percentage from statusline bridge file
- Injects warnings when context drops below 35% (warning) or 25% (critical)
- Debounced to avoid warning spam (5 tool uses between warnings)
- Severity escalation bypasses debounce

### 3.2 File Locking for Concurrent Access

**Module:** `core.cjs` - `withPlanningLock()`

Prevents concurrent worktrees from corrupting shared planning files:

```javascript
function withPlanningLock(cwd, fn) {
  const lockPath = path.join(planningDir(cwd), '.lock');

  // Atomic create — fails if file exists
  fs.writeFileSync(lockPath, data, { flag: 'wx' });

  // Lock acquired — run function
  return fn();
  // Finally: unlink lock file
}
```

- 10-second lock timeout with 100ms retry intervals
- Stale lock recovery: locks older than 30 seconds are force-released
- Used for all `.planning/` write operations

### 3.3 Temp File Cleanup

**Module:** `core.cjs` - `reapStaleTempFiles()`

```javascript
function reapStaleTempFiles(prefix = 'gsd-', { maxAgeMs = 5 * 60 * 1000, dirsOnly = false } = {}) {
  // Removes stale temp files older than 5 minutes
  // Runs opportunistically before each new temp file write
}
```

- Prevents unbounded temp file accumulation in `/tmp`
- Default 5-minute age threshold
- Cleanup is non-blocking (errors ignored)

### 3.4 Project Root Resolution

**Module:** `core.cjs` - `findProjectRoot()`

Prevents `.planning/` creation inside sub-repos in monorepo workspaces:

```javascript
function findProjectRoot(startDir) {
  // 1. If startDir has .planning/, it IS the project root
  // 2. Walk up looking for parent .planning/
  //    - Check config.json sub_repos listing
  //    - Check legacy multiRepo flag
  //    - Heuristic: parent has .planning/ + we're inside git repo
  // 3. Never walks above home directory
}
```

### 3.5 Commit Message Sanitization

**Module:** `commands.cjs` - `cmdCommit()`

```javascript
// Commit messages are sanitized before git operations
if (message) {
  const { sanitizeForPrompt } = require('./security.cjs');
  message = sanitizeForPrompt(message);
}
```

- Commit messages flow back into agent context on future reads
- Sanitized to prevent stored prompt injection via git history

### 3.6 Config Key Allowlisting

**Module:** `config.cjs`

```javascript
const VALID_CONFIG_KEYS = new Set([
  'mode', 'granularity', 'parallelization', 'commit_docs', 'model_profile',
  'search_gitignored', 'brave_search', 'firecrawl', 'exa_search',
  'workflow.research', 'workflow.plan_check', 'workflow.verifier',
  // ... 28 total keys
  'hooks.context_warnings',
]);

// Dynamic patterns allowed: agent_skills.<agent-type>
if (/^agent_skills\.[a-zA-Z0-9_-]+$/.test(keyPath)) return true;
```

- Only documented config keys are accepted
- Prevents injection of arbitrary config values
- Suggests corrections for typos (e.g., `nyquist.validation_enabled` -> `workflow.nyquist_validation`)

---

## 4. Performance Patterns

### 4.1 Large Output Handling

**Module:** `core.cjs` - `output()`

```javascript
function output(result, raw, rawValue) {
  const json = JSON.stringify(result, null, 2);

  // Large payloads (>50KB) exceed Claude Code's Bash tool buffer
  // Write to tmpfile and output path prefixed with @file:
  if (json.length > 50000) {
    reapStaleTempFiles();
    const tmpPath = path.join(require('os').tmpdir(), `gsd-${Date.now()}.json`);
    fs.writeFileSync(tmpPath, json, 'utf-8');
    data = '@file:' + tmpPath;
  }

  fs.writeSync(1, data);  // Blocking write ensures buffer drains
}
```

### 4.2 JSON Parsing with Size Limits

All JSON parsing uses `safeJsonParse()` with configurable size limits (default 1MB). This prevents:
- Memory exhaustion from malformed JSON
- Catastrophic backtracking in deeply nested structures

### 4.3 Frontmatter Parsing

**Module:** `frontmatter.cjs` - `extractFrontmatter()`

- Handles CRLF normalization
- Detects multiple frontmatter blocks (CRLF corruption indicator) and uses the last one
- No external YAML parser dependency (custom lightweight parser)
- Simple regex-based parsing avoids parser vulnerabilities

### 4.4 Stale Metrics Detection

**Module:** `hooks/gsd-context-monitor.js`

```javascript
const STALE_SECONDS = 60;  // Ignore metrics older than 60s
if (metrics.timestamp && (now - metrics.timestamp) > STALE_SECONDS) {
  process.exit(0);  // Silently ignore stale context metrics
}
```

### 4.5 Async/Never-Blocking Hooks

All hooks include timeout guards:
```javascript
const stdinTimeout = setTimeout(() => process.exit(0), 3000);  // 3 second timeout
```

Background operations use `child.unref()` for detached processes:
```javascript
const child = spawn(...);
child.unref();  // Parent doesn't wait for child
```

---

## 5. Security Gaps and Concerns

### 5.1 Advisory-Only Prompt Guard

The `gsd-prompt-guard.js` hook does NOT block suspicious content — it only warns. A sufficiently sophisticated prompt injection attack could:
1. Embed instructions that pass through the guard
2. Rely on the agent not noticing the warning
3. Exploit timing windows between guard updates

**Mitigation:** The hook documentation acknowledges this is defense-in-depth. Primary defense is input/output boundary design in agent prompts.

### 5.2 Regex DoS Potential

While `validateFieldName()` prevents regex injection into field name patterns, several regex operations in the codebase could be vulnerable to catastrophic backtracking:
- `extractFrontmatter()` uses regex-based line parsing
- `escapeRegex()` is used to build patterns dynamically

**Current mitigation:** Input size limits on JSON parsing, short max lengths for field names (60 chars).

### 5.3 No Encryption at Rest

The `.planning/` directory contains:
- Project plans and state
- Potentially sensitive milestone information
- User decisions and discussions

This directory is gitignored but not encrypted. If an attacker gains filesystem access, all planning data is readable.

**Status:** Working-as-intended for a local CLI tool. Full disk encryption is the responsibility of the OS.

### 5.4 Workstream Name Validation

Workstream names are validated with `/^[a-zA-Z0-9_-]+$/` before being used in paths. This is sufficient to prevent path traversal, but does not prevent:
- Unicode homoglyph attacks (e.g., Cyrillic 'a' vs Latin 'a')
- Invisible character injection despite the regex

**Current status:** Should be considered for hardening.

### 5.5 Config Migration Side Effects

The `loadConfig()` function in `core.cjs` auto-migrates config keys and syncs `sub_repos` with filesystem state, writing changes back to disk:

```javascript
if (configDirty) {
  fs.writeFileSync(configPath, JSON.stringify(parsed, null, 2), 'utf-8');
}
```

This automatic mutation could be surprising in some contexts (e.g., read-only mounts, simultaneous edits from multiple sources).

---

## 6. Test Coverage

Security-relevant tests in `tests/security.test.cjs`:
- `validatePath`: 11 test cases including null bytes, traversal, symlinks
- `scanForInjection`: 15+ test cases covering all pattern categories
- `sanitizeForPrompt`: 9 test cases including edge cases
- `validateShellArg`: 7 test cases
- `safeJsonParse`: 6 test cases including size limits
- `validatePhaseNumber`: 9 test cases including injection attempts
- `validateFieldName`: 6 test cases

CI runs `tests/prompt-injection-scan.test.cjs` which scans all agents, workflows, commands, hooks, and lib files for prompt injection patterns.

---

## 7. Vulnerability Reporting

Per `SECURITY.md`:

- **Report method:** Email to `security@gsd.build` (or DM @glittercowboy on Discord/Twitter if email bounces)
- **Do NOT report** via public GitHub issues
- **Response timeline:**
  - Acknowledgment: 48 hours
  - Initial assessment: 1 week
  - Critical fixes: 24-48 hours
  - High: 1 week
  - Medium/Low: Next release

**Scope:** Code execution on user machines, credential exposure, plan/code integrity.

---

## 8. Summary

| Category | Assessment |
|----------|------------|
| Authentication | N/A (local CLI tool) |
| Secrets Management | Environment variables + key files, no hardcoding |
| Path Traversal | Strong: realpath resolution, null byte rejection, containment checks |
| Prompt Injection | Comprehensive detection + sanitization, CI scanning |
| Shell Injection | Defense-in-depth via array-based exec, validation wrapper |
| JSON Safety | Size-limited parsing with graceful error handling |
| Concurrent Access | File locking with stale recovery |
| Hook Security | Advisory-only (intentional trade-off) |
| Config Validation | Allowlist-based, auto-migration with disk writes |
| Performance | Large output streaming, temp file cleanup, non-blocking hooks |

The codebase demonstrates mature security awareness for a local CLI tool, with particular strength in input validation for its primary attack vectors (path traversal, prompt injection, shell argument safety). The main trade-off is the advisory-only nature of the prompt guard hook, which prioritizes workflow continuity over strict blocking.
