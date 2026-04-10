# Features 10-12 Deep Dive: Local Repo Mode, Sandbox Execution, Compliance & Audit

## Feature 10: Local Repo Mode (GitHub Integration)

### Overview
Local Repo Mode enables cloning a GitHub repository, running an agent on it, auto-committing changes to a session branch, and pushing back to origin. It supports session resumption.

### Core Implementation

**Primary File:** `src/session.ts` (156 lines)

**Example Usage:** `examples/local-repo.ts`

### Architecture

```
initLocalSession() factory function
    |
    +-- authedUrl() / cleanUrl()   -- token injection into Git URL
    +-- git()                       -- thin execSync wrapper
    +-- getDefaultBranch()          -- detects origin/main or origin/master
    |
    +-- Clone or update existing repo
    +-- Create/resume session branch (gitclaw/session-<hex>)
    +-- Scaffold agent.yaml + memory/ if missing
    |
    +-- Return LocalSession object with methods:
          commitChanges(msg?)
          push()
          finalize()
```

### Session Branch Strategy

Branches follow the naming convention `gitclaw/session-<8-char-hex>`. The hex is generated via `randomBytes(4)` when creating a new session, or extracted from the session name when resuming.

**Key flow (lines 57-97):**
1. Clone with `--depth 1 --no-single-branch` (full history for git operations but tracked)
2. On existing dir: reset local to remote default branch (hard reset --hard)
3. For new sessions: create `gitclaw/session-<hex>` branch off current default
4. For resuming: checkout existing branch, pull latest

### Token Security Pattern

Two helper functions manage GitHub PAT injection:

```typescript
function authedUrl(url: string, token: string): string {
  // https://github.com/org/repo → https://<token>@github.com/org/repo
  return url.replace(/^https:\/\//, `https://${token}@`);
}

function cleanUrl(url: string): string {
  // Strip PAT from URL before final push
  return url.replace(/^https:\/\/[^@]+@/, "https://");
}
```

**Security flow:**
- Token injected into clone/fetch URLs for auth
- On `finalize()`, `cleanUrl()` strips the token before storing
- This prevents PAT leakage in git config or remote URL

### Error Handling
- `git()` function uses `stdio: "pipe"` to suppress output but captures it
- Missing remote branches during resume: silently ignored (line 91) since branch may not exist on remote yet
- Commit-on-nothing: catches `git diff --cached --quiet` exit code to skip empty commits

### Clever Solutions
- `--no-single-branch` on clone preserves full git history for later operations while keeping clone fast
- `git reset --hard origin/<branch>` ensures clean state on resume
- Auto-scaffolding of `agent.yaml` with sensible defaults (GPT-4o-mini, 4 tools) when missing

### SDK Integration
`sdk.ts` lines 113-126 wire LocalSession into the query pipeline:

```typescript
if (options.repo) {
  const token = options.repo.token || process.env.GITHUB_TOKEN || process.env.GIT_TOKEN;
  localSession = initLocalSession({
    url: options.repo.url,
    token,
    dir: options.repo.dir || dir,
    session: options.repo.session,
  });
  dir = localSession.dir;
}
```

After agent run, `finalize()` is called in both success and error paths (lines 427-441).

### Technical Debt / Concerns
1. **execSync blocking**: All git operations are synchronous, blocking the event loop during clone
2. **No progress feedback**: Clone of large repos has no streaming output
3. **Sole reliance on origin remote**: No support for multiple remotes or alternative hosting
4. **No branch cleanup**: Old session branches accumulate on remote unless manually deleted
5. **Hard reset on update**: `git reset --hard` discards any uncommitted local changes (by design, but undocumented)

---

## Feature 11: Sandbox Execution

### Overview
Sandbox Execution runs agents in an isolated VM (e2b provider via gitmachine) to prevent destructive operations from affecting the host system.

### Core Implementation

**Primary Files:**
- `src/sandbox.ts` (95 lines) -- SandboxContext factory
- `src/tools/sandbox-cli.ts` (65 lines)
- `src/tools/sandbox-read.ts` (37 lines)
- `src/tools/sandbox-write.ts` (38 lines)
- `src/tools/sandbox-memory.ts` (153 lines)

### Architecture

```
createSandboxContext(config, dir)  -- Factory
    |
    +-- Dynamic import gitmachine (optional peer dep)
    +-- detectRepoUrl()           -- git remote get-url origin
    +-- new GitMachine({...})     -- e2b VM provisioning
    |
    +-- Return SandboxContext:
          gitMachine: GitMachine instance (run, commit, start, stop)
          machine: Machine instance (readFile, writeFile)
          repoPath: /home/user/<repo-name>  (convention)
```

### Provider Abstraction

The sandbox currently hardcodes `provider: "e2b"` but the config interface accepts any string. The architecture suggests future provider support.

### Tool Wrapping Pattern

`src/tools/index.ts` `createBuiltinTools()` (lines 29-37) decides at runtime whether to return sandbox-backed or local tools:

```typescript
if (config.sandbox) {
  return [
    createSandboxCliTool(config.sandbox, config.timeout),
    createSandboxReadTool(config.sandbox),
    createSandboxWriteTool(config.sandbox),
    createSandboxMemoryTool(config.sandbox),
  ];
}
```

### Sandbox CLI Tool (`sandbox-cli.ts`)

Key characteristics:
- Uses `ctx.gitMachine.run()` instead of child_process exec
- Supports `onStdout` and `onStderr` callbacks for streaming output
- `onUpdate` callback enables real-time streaming to LLM
- Exit code != 0 throws an Error (lines 53-56)
- Output truncated to ~100KB via `truncateOutput()`

### Path Resolution

Sandbox tools use `resolveSandboxPath()` from `shared.ts`:

```typescript
export function resolveSandboxPath(path: string, repoRoot: string): string {
  if (path.startsWith("~/") || path === "~") {
    path = homedir() + path.slice(1);
  }
  if (path.startsWith("/")) return path;
  return repoRoot.endsWith("/") ? repoRoot + path : repoRoot + "/" + path;
}
```

**Convention:** Sandbox repo is always at `/home/user/<repo-name>` (line 87 of sandbox.ts).

### Sandbox Memory Tool (`sandbox-memory.ts`)

Notable features:
- Supports `memory/memory.yaml` config with named layers
- **Auto-archiving**: When `max_lines` is exceeded, older entries are moved to `memory/archive/<year>-<month>.md`
- Each save creates a git commit via `ctx.gitMachine.commit()`
- Graceful degradation: if commit fails, file is still written but user is warned

### Lifecycle Management

In `sdk.ts` lines 143-149, sandbox is started before tools are built:

```typescript
if (options.sandbox) {
  const sandboxConfig: SandboxOptions = options.sandbox === true
    ? { provider: "e2b" }
    : options.sandbox;
  sandboxCtx = await createSandboxContext(sandboxConfig, dir);
  await sandboxCtx.gitMachine.start();
}
```

Cleanup happens in both success and error paths (lines 432-446).

### Optional Dependency Pattern

Gitmachine is loaded via dynamic import with a clear error message:

```typescript
try {
  gitmachine = await import("gitmachine");
} catch {
  throw new Error(
    "Sandbox mode requires the 'gitmachine' package.\n" +
    "Install it with: npm install gitmachine",
  );
}
```

This allows the package to install without gitmachine -- sandbox features just fail with a helpful message.

### Technical Debt / Concerns
1. **Hardcoded e2b provider**: Only e2b is supported despite provider field in config
2. **Fixed repo path convention**: `/home/user/<repo-name>` is assumed but not enforced by gitmachine
3. **No network isolation verification**: The degree of isolation is determined by gitmachine/e2b
4. **Timeout at gitmachine level**: The `timeout` config is passed to GitMachine but error handling for timeouts in tools is unclear
5. **Sandbox stop on error**: The catch block calls `stop()` but it's not clear if VM is guaranteed to terminate

---

## Feature 12: Compliance & Audit Logging

### Overview
Built-in compliance validation against agent manifest rules, audit logging to `.gitagent/audit.jsonl`, and regulatory framework support (SOC2, GDPR).

### Core Implementation

**Primary Files:**
- `src/compliance.ts` (161 lines)
- `src/audit.ts` (77 lines)

### Compliance Validation

`validateCompliance()` in `compliance.ts` runs rule-based checks on the agent manifest:

| Rule ID | Severity | Condition |
|---------|----------|-----------|
| `high_risk_hitl` | warning | High/critical risk without human_in_the_loop |
| `critical_audit` | error | Critical risk without audit_logging |
| `regulatory_recordkeeping` | warning | Regulatory frameworks without recordkeeping |
| `high_risk_review` | warning | High/critical risk without review config |
| `audit_retention` | warning | Audit logging without retention_days |
| `data_classification` | warning | Regulatory frameworks without data_classification |

**Design pattern:** Rules return warnings (severity: warning) or errors (severity: error). Errors block agent startup.

### Compliance Manifest Schema

```typescript
export interface ComplianceConfig {
  risk_level?: "low" | "medium" | "high" | "critical";
  human_in_the_loop?: boolean;
  data_classification?: string;
  regulatory_frameworks?: string[];
  recordkeeping?: {
    audit_logging?: boolean;
    retention_days?: number;
  };
  review?: {
    required_approvers?: number;
    auto_review?: boolean;
  };
}
```

### Compliance Context Loading

`loadComplianceContext()` reads optional YAML files from `compliance/` directory:
- `compliance/regulatory-map.yaml` -- maps framework names to their requirements
- `compliance/validation-schedule.yaml` -- defines periodic compliance checks

These are formatted and appended to the system prompt (lines 121-152).

### Audit Logger

`AuditLogger` class in `audit.ts` appends JSONL entries to `.gitagent/audit.jsonl`:

```typescript
export interface AuditEntry {
  timestamp: string;
  session_id: string;
  event: string;
  tool?: string;
  args?: Record<string, any>;
  result?: string;
  error?: string;
  [key: string]: any;
}
```

**Non-blocking design:** Logging failures are silently swallowed (lines 40-42):

```typescript
try {
  await mkdir(dirname(this.logPath), { recursive: true });
  await appendFile(this.logPath, JSON.stringify(entry) + "\n", "utf-8");
} catch {
  // Audit logging failures are non-fatal
}
```

### Event Types

| Event | When Logged |
|-------|-------------|
| `tool_use` | Tool execution starts |
| `tool_result` | Tool completes (result truncated to 1000 chars) |
| `response` | LLM response generated |
| `error` | Any error occurs |
| `session_start` | Session begins |
| `session_end` | Session ends |

### Integration Points

**CLI integration (`src/index.ts` line 436):**
```typescript
console.log(yellow(formatComplianceWarnings(complianceWarnings)));
```

**Agent loading (`src/loader.ts`):**
- `validateCompliance()` called at line 248
- `loadComplianceContext()` called at line 313, appended to system prompt parts
- Warnings displayed but errors don't halt loader (error severity is informational at load time)

**Audit enablement check (`audit.ts` line 73):**
```typescript
export function isAuditEnabled(compliance?: Record<string, any>): boolean {
  if (!compliance) return false;
  return compliance.recordkeeping?.audit_logging === true;
}
```

### Clever Solutions
- **JSONL format**: Append-only, line-oriented JSON enables easy streaming reads and simple parsing without a database
- **Result truncation**: Tool results capped at 1000 chars to prevent log bloat while preserving context
- **Graceful degradation**: Audit logging failures never crash the agent session
- **Compliance context in system prompt**: Regulatory requirements and validation schedules become part of the agent's context

### Technical Debt / Concerns
1. **No compliance enforcement**: `validateCompliance()` returns warnings/errors but doesn't actually block agent startup for errors -- it's informational
2. **Audit logs not rotated**: No retention enforcement despite `retention_days` config
3. **No audit log integrity**: No signing or tamper-detection for audit entries
4. **Compliance context files optional**: No enforcement that `compliance/` directory exists
5. **No audit log query API**: Logs are written but there's no utility to search/filter them
6. **Warning display only**: Errors from `critical_audit` rule are severity "error" but still just printed as warnings

---

## Cross-Feature Interactions

### Local Repo + Sandbox
Both can be used together (validated in `sdk.ts` line 107-108 as mutually exclusive options). The `repo` option clones locally first; `sandbox` option provisions an e2b VM.

### Compliance + Audit
Compliance config determines if audit is enabled via `compliance.recordkeeping.audit_logging`. Audit entries include session_id for correlation.

### Sandbox + Memory
Sandbox memory tool has archiving built-in, independent of the non-sandbox memory tool's layer system.

---

## Summary Table

| Aspect | Local Repo Mode | Sandbox Execution | Compliance & Audit |
|--------|-----------------|-------------------|-------------------|
| **Lines of Code** | ~156 | ~398 | ~238 |
| **Key Dependency** | child_process/execSync | gitmachine (optional) | None (filesystem) |
| **Blocking I/O** | Yes (sync git ops) | No (async gitmachine) | No (async file append) |
| **Error Recovery** | Silent failures on pull | Best-effort stop | Silent swallowing |
| **Security** | PAT injection/cleaning | VM isolation | Non-fatal logging |
| **Extension Point** | Provider-agnostic git | Provider abstraction exists | Config-driven rules |
