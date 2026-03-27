# Lessons Learned: nanoclaw Project Analysis

**Analysis Date:** 2026-03-26
**Source:** `/Users/sheldon/Documents/claw/reference/nanoclaw`
**Research Files:** 8 deep-dive documents covering architecture, tech stack, community, features, code quality, and security

---

## Executive Summary

NanoClaw is a personal Claude assistant daemon that runs agents in isolated Docker containers, connected to multiple messaging channels (WhatsApp, Telegram, Slack, Discord, Gmail). The codebase demonstrates sophisticated systems engineering: ~15 core source files, 19 test files, 657 commits, active development with daily releases.

**Key impression:** This is a well-architected, security-conscious project with clever solutions to hard distributed systems problems. However, it has accumulated technical debt in testing infrastructure, CI gaps, and operational concerns that would matter for production deployment.

---

## WHAT TO EMULATE (Positive Patterns)

### 1. Security-First Architecture

**The "never trust the container" model is exceptional.**

- Secrets **never** enter `process.env` -- `readEnvFile()` returns values without setting them
- `.env` shadowed with `/dev/null` in containers -- even if project root is mounted, credentials are nullified
- OneCLI Agent Vault intercepts HTTPS and injects credentials at request time
- Per-group OneCLI agent identities enable per-group credential policies and rate limits

**Implementation** (`src/env.ts`):
```typescript
export function readEnvFile(keys: string[]): Record<string, string> {
  const envFile = path.join(process.cwd(), '.env');
  // Does NOT load into process.env — callers decide what to do
  // This keeps secrets out of the process environment so they don't leak to child processes
}
```

**Quantified:** The mount blocklist explicitly blocks 15+ credential-adjacent patterns (`.ssh`, `.aws`, `.env`, `credentials`, etc.)

### 2. Per-Group IPC Namespace via Filesystem

**The filesystem-as-IPC pattern is elegant and secure.**

Each group gets isolated IPC directories:
- `data/ipc/{groupFolder}/input/` -- host-to-container messages
- `data/ipc/{groupFolder}/messages/` -- container-to-host messages
- `data/ipc/{groupFolder}/tasks/` -- task IPC commands

Container polls `input/` directory, host polls `messages/` and `tasks/`. Authorization derived from directory path (sourceGroup), not user-supplied data.

**Why this is clever:** No network sockets, no IPC primitives, no shared memory. Just filesystem operations that already have OS-level permission enforcement.

### 3. Context Accumulation Between Triggers

**Non-trigger messages accumulate in SQLite and are pulled as context when a trigger arrives.**

```typescript
// Pull all messages since lastAgentTimestamp so non-trigger
// context that accumulated between triggers is included.
const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
```

This means: User can have a conversation in the group, and when they finally tag `@Andy`, the agent sees the full context, not just the trigger message.

### 4. Cursor Rollback on Partial Failure

**If agent fails before sending output, message cursor rolls back for retry.**

```typescript
if (outputSentToUser) return true;  // Don't rollback - user got response
lastAgentTimestamp[chatJid] = previousCursor;  // Rollback for retry
```

If agent crashes mid-processing, the system retries on next poll cycle. If agent succeeded and sent output, no rollback (prevents duplicate responses).

### 5. Self-Registering Channel Pattern

**Adding a new channel requires zero changes to core code.**

```typescript
// src/channels/index.ts (barrel file)
import './whatsapp/index.js';  // registers itself
import './telegram/index.js';  // registers itself
```

Each channel module calls `registerChannel(name, factory)` when imported. Channels auto-enable when credentials are present.

### 6. Exponential Backoff with Per-Group State

**Failed groups retry independently without affecting others.**

```typescript
const BASE_RETRY_MS = 5000;
const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
// 5s, 10s, 20s, 40s, 80s then give up
```

Each group tracks its own `retryCount`, so a failing Telegram integration doesn't block WhatsApp.

### 7. SQLite Defensive Migration Pattern

**Every schema change wrapped in try/catch for both fresh installs and existing databases.**

```typescript
try {
  database.exec(
    `ALTER TABLE scheduled_tasks ADD COLUMN context_mode TEXT DEFAULT 'isolated'`,
  );
} catch {
  /* column already exists */
}
```

`CREATE TABLE IF NOT EXISTS` handles new installs, `ALTER TABLE ADD COLUMN` with try/catch handles existing. No migration scripts needed.

### 8. Task Scheduling Drift Prevention

**Interval tasks anchor to original scheduled time, not `Date.now()`.**

```typescript
// Anchor to the scheduled time, not now, to prevent drift.
let next = new Date(task.next_run!).getTime() + ms;
while (next <= now) { next += ms; }
```

A 5-minute task running every 5 minutes stays aligned to the original schedule, not drifting over time.

### 9. Atomic IPC File Writes

**Messages written to `.tmp` then renamed, preventing partial reads.**

```typescript
fs.writeFileSync(tempPath, JSON.stringify({ type: 'message', text }));
fs.renameSync(tempPath, filepath);  // Atomic write
```

Container never sees a partial message.

### 10. Graceful Container Shutdown

**Containers detached on shutdown, not killed. They finish naturally via idle timeout.**

```typescript
// Don't kill active containers — they'll finish via idle timeout
// The --rm flag cleans them up on exit
```

This prevents WhatsApp reconnection restarts from killing working agents mid-response.

---

## WHAT TO AVOID (Negative Patterns)

### 1. Bare Barrel File Pattern is Fragile

**The `src/channels/index.ts` has commented imports. Future skills must uncomment or the pattern breaks.**

```typescript
// discord
// gmail
// slack
// telegram
// whatsapp
```

This is a maintenance hazard. A skill that forgets to uncomment its import line silently fails to register.

**Better pattern:** Use `fs.readdirSync()` to auto-discover channels, or have each channel create a manifest file.

### 2. Coverage Installed but Not Configured

**@vitest/coverage-v8 is installed but `vitest.config.ts` has no coverage block.**

```typescript
// vitest.config.ts - current (missing coverage)
export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'setup/**/*.test.ts'],
  },
});
```

This means: Tests run but coverage is never measured. The team installed the tooling but never enabled it.

**Impact:** Unknown test coverage. 19 test files exist but we don't know if critical paths are tested.

### 3. No CI Run for Skills Tests

**`vitest.skills.config.ts` is not executed in CI pipeline.**

CI runs:
1. `npm ci`
2. Format check
3. Typecheck
4. `npx vitest run` (default config only)

Skills tests (`vitest.skills.config.ts`) are skipped. If a skill branch breaks tests, CI won't catch it.

### 4. No Resource Limits on Containers

**No CPU/memory constraints on Docker containers (--memory, --cpus flags not used).**

A misbehaving agent could consume unlimited resources. NanoClaw relies on the global `MAX_CONCURRENT_CONTAINERS` limit but not per-container resource caps.

### 5. No Group Deletion Mechanism

**No `unregisterGroup` IPC type or cleanup of group folders/sessions.**

If a group is registered and later no longer needed, its folders, sessions, and IPC directories persist indefinitely.

### 6. 60-Second Scheduler Poll Interval

**60 seconds between scheduler iterations. Sub-minute cron tasks cannot be precise.**

For a personal assistant, this is likely fine. But for any time-sensitive automation, this is a hard limitation.

### 7. Global Concurrency Limit Ignores Group Priority

**All groups share `MAX_CONCURRENT_CONTAINERS` regardless of group importance.**

The main admin group competes equally with random Telegram groups for container slots. No priority queue or weight system.

### 8. Allowlist File Not Watched

**Sender allowlist loaded fresh on each message. Changes require restart.**

```typescript
// src/sender-allowlist.ts - loaded per-message
const cfg = loadSenderAllowlist();
```

For a daemon meant to run 24/7, this means changing access control requires restarting the service.

### 9. Drop Mode Silent Failure

**When in drop mode, denied messages are simply not stored with no notification.**

No log entry (unless `logDenied: true`), no user notification, no retry. If you misconfigure the allowlist in drop mode, you won't know messages are being discarded.

### 10. No GitHub Releases

**657 commits but no formal GitHub Releases with release notes.**

Tags exist (`1.2.35`) but no GitHub Releases. The CHANGELOG.md exists but isn't associated with a Release artifact.

---

## SURPRISES (Unexpected Findings)

### 1. XML for Message Context (Not JSON)

**Messages are formatted as XML for the Claude agent, not JSON.**

```xml
<context timezone="America/New_York" />
<messages>
<message sender="Alice" time="10:30 AM">Hello</message>
</messages>
```

This is interesting because JSON is more common in TypeScript projects. XML was likely chosen for its attribute syntax (`sender="..."`) and text node distinction.

### 2. Better-SQLite3 (Synchronous API)

**SQLite operations are synchronous, not async.**

`better-sqlite3` provides a synchronous API that blocks during queries. This is unusual in Node.js but makes sense here: the polling loops already run in `async` functions, and blocking I/O is actually fine when the main loop isn't waiting on other things.

### 3. Two Separate TypeScript Projects

**Root project and `container/agent-runner/` have independent `tsconfig.json` and `package.json`.**

Not a monorepo, not workspaces. Two independent npm packages that happen to share a repo. The container agent-runner has its own dependencies (Claude Agent SDK, MCP SDK).

### 4. Agent Runner Source Copied Per-Group

**On container spawn, agent-runner source is copied to per-group directory.**

```typescript
const needsCopy =
  !fs.existsSync(groupAgentRunnerDir) ||
  !fs.existsSync(cachedIndex) ||
  (fs.existsSync(srcIndex) &&
    fs.statSync(srcIndex).mtimeMs > fs.statSync(cachedIndex).mtimeMs);
```

This allows per-group customization of the agent runner without rebuilding the container image. Clever but potentially messy if many groups accumulate different versions.

### 5. Bot Pool for Agent Swarms

**Agent swarms use a pool of Telegram bots, each renamed to match a sub-agent identity.**

If "Researcher" agent sends a message, the system:
1. Looks up sender-to-bot mapping
2. If not mapped, assigns unused bot and calls `setMyName` to rename it
3. 2-second delay for Telegram propagation
4. Sends from that bot

This is a creative solution to Telegram's per-bot identity limitation.

### 6. OneCLI is a Hard Dependency

**The project migrated from direct `.env` injection to OneCLI Agent Vault.**

This is a significant architectural change. The `@onecli/sh/sdk` dependency is non-optional for credential security. The "native credential proxy" skill exists as an alternative but requires merging a skill branch.

---

## CODE QUALITY ASSESSMENT

| Dimension | Rating | Notes |
|-----------|--------|-------|
| Type Safety | Strong | TypeScript strict mode, modern NodeNext config |
| Testing | Good (but unmeasured) | 19 test files, good isolation, but no coverage |
| Error Handling | Excellent | Structured logging, global handlers, rollback patterns |
| Input Validation | Thorough | Regex escaping, XML escaping, path validation |
| Security | Excellent | Secrets never in containers, mount allowlist, IPC auth |
| Code Organization | Good | Small focused files, clear responsibilities |
| CI/CD | Moderate | Missing coverage gate, skills tests not run |

---

## ARCHITECTURAL HIGHLIGHTS

### The Core Loop

```
Channels (polling) --> SQLite --> Message Loop --> GroupQueue --> Container --> Response
                         ^                                                   |
                         └─────────────────── IPC via filesystem ────────────┘
```

Single `while(true)` loop polls SQLite every 2 seconds for new messages. Global cursor prevents duplicates.

### Group Isolation

Each group gets:
- Isolated filesystem mount (`groups/{folder}/`)
- Isolated CLAUDE.md memory
- Isolated session (`.claude/`)
- Isolated IPC namespace (`data/ipc/{folder}/`)
- Independent trigger pattern
- Per-group OneCLI agent identity

### Credential Flow

```
.env file --> readEnvFile() --> OneCLI vault
                                  |
Container spawn --> applyContainerConfig() --> Gateway intercepts HTTPS --> Injects credentials
```

Credentials never enter container environment.

---

## NEGATIVE FINDINGS THAT ARE CRITICAL

### 1. Missing Coverage Reporting

The most concerning finding. The team has:
- 19 test files
- `@vitest/coverage-v8` installed
- But coverage is **never computed**

This suggests tests exist but nobody knows what's actually covered. For a security-critical project running untrusted code in containers, this is dangerous.

### 2. No Security Scanning in CI

No `eslint-plugin-security` or similar. The security architecture is good, but there's no automated scanning for:
- Hardcoded credentials
- SQL injection (though they use parameterized queries -- good)
- Path traversal
- XSS

### 3. Skills Tests Excluded from CI

If you add a channel via skill branch and it has tests, those tests won't run in CI. The branch could pass CI but have broken skill tests.

---

## POSITIVE SURPRISES

### 1. Comprehensive Input Sanitization

Every input vector is sanitized:
- Group folder names: strict regex pattern + reserved word check
- Trigger patterns: regex escaped
- Message content: XML-escaped before agent context
- SQL: parameterized queries throughout
- Mount paths: symlink-resolved and allowlist-validated

### 2. Graceful Degradation

Multiple layers of fallback:
- OneCLI unreachable: containers start without credentials, logged as warning
- Allowlist file missing: defaults to permissive
- Invalid group folder in DB: skipped rather than crashed
- IPC file read error: moved to `errors/` directory, processing continues

### 3. Active Maintenance

657 commits with daily releases. The project is actively maintained and the community health report shows positive trajectory.

---

## KEY METRICS

| Metric | Value |
|--------|-------|
| Total Commits | 657 |
| Latest Version | 1.2.35 |
| Source Files | ~15 core files |
| Test Files | 19 |
| Dependencies (runtime) | 7 |
| Directory Structure | Clean, predictable |
| CI Pipeline Steps | 4 (format, typecheck, test) |

---

## CONCLUSION

NanoClaw is a **well-engineered, security-conscious personal assistant platform**. Its core strengths are:
- Exceptional security architecture (secrets never in containers)
- Clean separation of concerns (layered + event-driven)
- Thoughtful input validation throughout
- Good error handling and recovery patterns

The main areas for improvement are:
- **Enable coverage reporting** (tooling installed but not configured)
- **Add skills tests to CI** (currently excluded)
- **Add container resource limits** (CPU/memory caps)
- **Consider group deletion mechanism** (no cleanup path)

For someone building a similar project, the security model and per-group isolation patterns are the standout learnings. The codebase demonstrates that small, focused teams can build sophisticated systems when they prioritize security and clean architecture over feature proliferation.
