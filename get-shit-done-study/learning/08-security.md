# Security: Get Shit Done

## Security Philosophy

GSD's security approach follows **defense in depth** - multiple layers prevent common failure modes. Since GSD generates markdown files that become LLM system prompts, any user-controlled text flowing into planning artifacts is treated as a potential indirect prompt injection vector.

## Built-in Security Hardening

### Path Traversal Prevention
All user-supplied file paths are validated to resolve within the project directory.

**Implementation:** `security.cjs` module's `validatePath()` function
- Used for `--text-file`, `--prd`, and any user-provided paths
- Resolves symbolic links
- Checks final path is within project root
- Prevents escape via `../` or absolute paths

```javascript
// From security.cjs
validatePath(userPath, projectRoot) {
  const resolved = path.resolve(projectRoot, userPath);
  if (!resolved.startsWith(projectRoot)) {
    throw new Error('Path traversal detected');
  }
  return resolved;
}
```

### Prompt Injection Detection

**Centralized scanning** in `security.cjs` detects:
- Role override patterns (`"You are now..."`, `"Ignore previous instructions"`)
- Instruction bypass (`"Ignore all rules"`, `"Disregard system prompt"`)
- System tag injection (`<system>`, `</system>`)
- Delimiter confusion (`<xml>`, `[/xml]`, `<<<`)

**CI Scanner:** `prompt-injection-scan.test.cjs` scans all agent/workflow/command files for embedded injection vectors.

### Shell Argument Validation

**Safe execution patterns:**
- Use `execFileSync` (array args) over `execSync` (string interpolation)
- User text sanitized before shell interpolation
- No direct `eval()` or `new Function()`

```javascript
// BAD - shell injection risk
execSync(`git commit -m "${message}"`);

// GOOD - safe execution
execFileSync('git', ['commit', '-m', message]);
```

### Safe JSON Parsing

Malformed `--fields` arguments caught before they corrupt state:
```javascript
try {
  const data = JSON.parse(userInput);
} catch (e) {
  // Handle gracefully, don't crash
}
```

## Security Hooks (v1.27+)

### Prompt Guard (`gsd-prompt-guard.js`)
- **Trigger:** Write/Edit to `.planning/` files
- **Behavior:** Scans content for prompt injection patterns
- **Enforcement:** Advisory only - logs detection, does not block
- **Patterns:** Inlined subset of `security.cjs` for hook independence
- **Note:** Does not prevent injection, only detects and warns

### Workflow Guard (`gsd-workflow-guard.js`)
- **Trigger:** Write/Edit to non-`.planning/` files
- **Behavior:** Detects edits outside GSD workflow context
- **Advisory:** Suggests using `/gsd:quick` or `/gsd:fast` for state-tracked changes
- **Opt-in:** Enabled via `hooks.workflow_guard: true` (default: false)

## Secrets Management

### Protecting Sensitive Files

GSD's codebase mapping and analysis commands read files to understand projects. **Protect secrets by adding to Claude Code's deny list:**

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/secrets/*)",
      "Read(**/*credential*)",
      "Read(**/*.pem)",
      "Read(**/*.key)"
    ]
  }
}
```

### Defense-in-Depth Recommendation

> GSD includes built-in protections against committing secrets, but deny read access to sensitive files as a first line of defense.

### Built-in Secret Protection

GSD's workflow generates code, not commits. The framework:
- Does not write credentials
- Does not hardcode API keys
- Expects users to manage `.env` files separately

## GitHub Actions Security

### No `${{ }}` in run: Blocks

GSD's CONTRIBUTING guidelines prohibit shell interpolation in GitHub Actions:

```yaml
# BAD - injection risk
run: |
  echo "Deploying ${{ github.event.inputs.version }}"

# GOOD - bind to env first
env:
  VERSION: ${{ github.event.inputs.version }}
run: |
  echo "Deploying $VERSION"
```

### CI Security Scanning

v1.29+ includes security scanning workflows:
- Prompt injection scanning
- Base64 encoded content detection
- Secret scanning patterns

## State Locking

### File-Based State Race Conditions

When multiple executors run in parallel within a wave, GSD prevents read-modify-write race conditions on `STATE.md`:

**Mechanism:** Lockfile-based mutual exclusion
- Creates `STATE.md.lock` with `O_EXCL` atomic creation
- Includes stale lock detection (10s timeout)
- Spin-wait with jitter for lock acquisition
- Removes lockfile after write

```javascript
// Simplified pattern
const lockPath = `${statePath}.lock`;
try {
  // Atomic lock creation
  fs.writeFileSync(lockPath, pid, { flag: 'wx' });
  // Critical section - read, modify, write
  const state = readState();
  state.updated = true;
  writeState(state);
} finally {
  fs.unlinkSync(lockPath);
}
```

## Installation Security

### Multi-Runtime Installer Safety

The `bin/install.js` (~3,000 lines) handles:
- **Path normalization** - Replaces `~/.claude/` paths with runtime-specific paths
- **No arbitrary code execution** - Only file copy and config update
- **Manifest tracking** - `gsd-file-manifest.json` for clean uninstall

### Runtime Adaptation

File content transformation at install time:
- Claude Code: Uses as-is
- OpenCode: Converts agent frontmatter
- Codex: Generates TOML config
- Copilot: Maps tool names
- Gemini: Adjusts hook event names
- Antigravity: Skills-first with model equivalents

## Security Vulnerability Response

### Reporting
- **Email:** security@gsd.build (or DM @glittercowboy on Discord/Twitter)
- **Private disclosure required** - No public GitHub issues

### Response Timeline
- **Acknowledgment:** 48 hours
- **Initial assessment:** 1 week
- **Fix timeline:**
  - Critical: 24-48 hours
  - High: 1 week
  - Medium/Low: Next release

### In-Scope Vulnerabilities
- Arbitrary code execution on user machines
- Sensitive data exposure (API keys, credentials)
- Integrity compromise of generated plans/code

## Security Gaps and Considerations

### Advisory-Only Protections

The prompt guard hook (`gsd-prompt-guard.js`) is **advisory only**:
- Logs detections
- Does not block writes
- Cannot prevent sophisticated injection attempts

### User-Controlled Content

Since GSD accepts user input that flows into LLM prompts:
- Discussion responses
- Requirement descriptions
- Context preferences
- Verification results

These should be treated as potentially untrusted when viewed in aggregate.

### No Code Execution Sandboxing

GSD generates code but does not execute it (the host AI runtime does). Security of executed code depends on:
- The host AI runtime's safety measures
- User's own permissions configuration
- Whether user runs with `--dangerously-skip-permissions`

### Secrets in Planning Artifacts

GSD itself doesn't handle secrets, but user projects may include:
- API keys in generated code
- Database credentials in configurations
- Third-party service tokens

Users must apply their own secrets management practices.

## Security Best Practices

1. **Use `.env` files with proper `.gitignore`**
2. **Add sensitive file patterns to deny list**
3. **Review generated code before committing**
4. **Run with `--dangerously-skip-permissions` only when understood**
5. **Keep GSD updated** - Security fixes in each release
6. **Report vulnerabilities privately** - Don't use public issues
