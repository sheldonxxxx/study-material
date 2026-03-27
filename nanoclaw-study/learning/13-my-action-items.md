# My Action Items: Building a NanoClaw-Style Project

**Derived from:** 12-lessons-learned.md analysis
**Context:** Building a personal AI assistant with container isolation, multi-channel messaging, and group-based context

---

## Priority 1: Security Architecture (Do First)

### 1.1 Implement Credential Vault Pattern

**Why:** NanoClaw's "never trust the container" model is the most important architectural decision.

**Actions:**
- [ ] Research OneCLI Agent Vault or similar proxy-based credential injection
- [ ] Never pass API keys via environment variables to containers
- [ ] Use local HTTPS proxy that intercepts outbound requests and injects credentials
- [ ] Implement per-agent credential policies (rate limits, allowed hosts)

**Reference:** `src/env.ts`, `src/container-runner.ts` lines 236-249

### 1.2 Shadow .env in Containers

**Why:** Even with a vault, if the project root is mounted read-only, .env would expose secrets.

**Actions:**
- [ ] Mount `/dev/null` over any `.env` path in container mounts
- [ ] Verify containers cannot read secrets via `process.env` inheritance
- [ ] Test that container startup logs don't contain credential values

**Reference:** `src/container-runner.ts:81-90`

### 1.3 External Mount Allowlist

**Why:** NanoClaw stores allowlist at `~/.config/nanoclaw/`, outside the project. This prevents malicious agents from modifying it.

**Actions:**
- [ ] Store mount allowlist at `~/.config/{app-name}/mount-allowlist.json`
- [ ] Block credential-adjacent paths by default: `.ssh`, `.aws`, `.env`, `.gnupg`, `.docker`, `credentials`
- [ ] Validate all mount paths against allowlist before container spawn
- [ ] Resolve symlinks before validation (`fs.realpathSync()`)

**Reference:** `src/mount-security.ts`

---

## Priority 2: Core Architecture

### 2.1 Filesystem-Based IPC

**Why:** Elegant, secure, no network sockets needed. OS permissions provide isolation.

**Actions:**
- [ ] Design per-group IPC directories: `data/ipc/{group}/input/`, `data/ipc/{group}/output/`
- [ ] Host writes messages to `input/` for container consumption
- [ ] Container writes responses to `output/` for host consumption
- [ ] Use atomic rename (`writeFileSync` + `renameSync`) to prevent partial reads
- [ ] Derive authorization from directory path, not user-supplied data

**Reference:** `src/ipc.ts`, `src/container-runner.ts`

### 2.2 SQLite with Defensive Migrations

**Why:** Simple, no separate database process, portable. Defensive migration pattern handles upgrades gracefully.

**Actions:**
- [ ] Use `better-sqlite3` for synchronous SQLite operations
- [ ] Wrap every `ALTER TABLE` in try/catch
- [ ] Use `CREATE TABLE IF NOT EXISTS` for fresh installs
- [ ] Store timestamps as ISO8601 strings (not Unix epochs) for debuggability
- [ ] Implement JSON state migration from file-based storage on first run

**Reference:** `src/db.ts` lines 229-242

### 2.3 Self-Registering Channel Pattern

**Why:** Adding channels requires zero core code changes.

**Actions:**
- [ ] Implement `ChannelFactory` and `Channel` interface
- [ ] Create `channels/registry.ts` with `Map<string, ChannelFactory>`
- [ ] Channels call `registerChannel(name, factory)` on import
- [ ] Auto-discover channels via `fs.readdirSync` or explicit barrel file
- [ ] Channels auto-enable when credentials are present

**Reference:** `src/channels/registry.ts`, `src/channels/index.ts`

### 2.4 Per-Group Context Isolation

**Why:** Each conversation group needs its own memory, filesystem, and session.

**Actions:**
- [ ] Design group folder structure: `groups/{folder}/CLAUDE.md`, `groups/{folder}/logs/`
- [ ] Each group gets isolated session at `data/sessions/{folder}/.claude/`
- [ ] Global memory mounted read-only at `/workspace/global`
- [ ] Implement group registration IPC command
- [ ] Validate group folder names with strict pattern: `/^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$/`
- [ ] Block reserved names: `global`, `main`, `data`, `ipc`

**Reference:** `src/group-folder.ts`, `src/index.ts`

---

## Priority 3: Message Orchestration

### 3.1 Polling Loop with Global Cursor

**Why:** SQLite polling with global cursor prevents duplicate processing on crash.

**Actions:**
- [ ] Implement `while(true)` loop with configurable poll interval (2s default)
- [ ] Track global `lastTimestamp` that advances for ALL messages atomically
- [ ] Track per-group `lastAgentTimestamp` for context accumulation
- [ ] On crash recovery, re-enqueue unprocessed messages

**Reference:** `src/index.ts:404-494`

### 3.2 Trigger Word Processing

**Why:** Prevents assistant from responding to every message in group chats.

**Actions:**
- [ ] Implement trigger pattern with word boundary: `^@Name\b`
- [ ] Escape regex special characters in trigger input
- [ ] Non-trigger messages accumulate in DB but don't advance agent cursor
- [ ] When trigger arrives, pull ALL messages since `lastAgentTimestamp` for context
- [ ] `is_from_me` messages bypass trigger check

**Reference:** `src/config.ts:257-269`, `src/index.ts:444-457`

### 3.3 XML Message Formatting

**Why:** XML provides structured context with attribute syntax for metadata.

**Actions:**
- [ ] Format messages as XML with sender, timestamp, content
- [ ] XML-escape all content (`&`, `<`, `>`, `"`)
- [ ] Sender names in attributes, content in text nodes
- [ ] Strip `<internal>` tags from outbound responses

**Reference:** `src/router.ts:13-44`

---

## Priority 4: Container Management

### 4.1 Container Lifecycle

**Why:** Containers are spawned per message-batch, kept alive for follow-ups.

**Actions:**
- [ ] Implement `CONTAINER_TIMEOUT` (default 30 min) hard limit
- [ ] Implement `IDLE_TIMEOUT` (default 30 min) keep-alive for containers
- [ ] On idle timeout, send `_close` sentinel to signal graceful exit
- [ ] Use `--rm` flag for ephemeral containers
- [ ] Use `docker stop -t 1` for graceful shutdown, SIGKILL after 15s

**Reference:** `src/container-runner.ts`, `src/group-queue.ts`

### 4.2 Concurrency Control

**Why:** Prevent resource exhaustion with global concurrency limit.

**Actions:**
- [ ] Implement `MAX_CONCURRENT_CONTAINERS` (configurable, default 5)
- [ ] Per-group queues for messages and tasks
- [ ] FIFO waiting list for fairness across groups
- [ ] Tasks prioritized over messages (tasks won't be rediscovered)
- [ ] Implement exponential backoff for failed groups (5s, 10s, 20s, 40s, 80s)

**Reference:** `src/group-queue.ts`

### 4.3 Non-Root Container User

**Why:** Principle of least privilege for container execution.

**Actions:**
- [ ] Use `node:22-slim` base image
- [ ] Create non-root user (uid 1000) in Dockerfile
- [ ] Run containers as non-root user
- [ ] Only bind-mount explicitly allowed directories

**Reference:** `container/Dockerfile:60-64`

---

## Priority 5: Input Validation

### 5.1 Comprehensive Sanitization

**Why:** Running untrusted code in containers requires defense in depth.

**Actions:**
- [ ] Regex-escape trigger patterns before building RegExp
- [ ] XML-escape all message content
- [ ] Parameterized SQL queries only (no string interpolation)
- [ ] Path traversal prevention: validate resolved paths stay within base
- [ ] Validate cron expressions with `CronExpressionParser`
- [ ] Validate sender allowlist entries with type guards

**Reference:** `src/router.ts`, `src/group-folder.ts`, `src/sender-allowlist.ts`

### 5.2 IPC Authorization

**Why:** Prevent malicious agents from escalating privileges.

**Actions:**
- [ ] Derive group identity from IPC file's parent directory name
- [ ] Non-main groups can only schedule tasks for themselves
- [ ] Non-main groups cannot register new groups
- [ ] Main group has `isMain: true` flag, enforced throughout
- [ ] Defense in depth: agent cannot set `isMain` via IPC

**Reference:** `src/ipc.ts:76-94`, `src/ipc.ts:444`

---

## Priority 6: Testing & CI

### 6.1 Enable Coverage Reporting

**Why:** NanoClaw has tooling installed but not configured. Don't make this mistake.

**Actions:**
- [ ] Configure Vitest coverage block in `vitest.config.ts`
- [ ] Use v8 provider with text and HTML reporters
- [ ] Set coverage threshold gate in CI (aim for >70%)
- [ ] Track coverage trend over time

**Reference:** `vitest.config.ts` (current missing coverage block)

### 6.2 Comprehensive Test Suite

**Why:** 19 test files good, but unknown coverage is bad.

**Actions:**
- [ ] Test database operations with in-memory SQLite
- [ ] Test path validation edge cases
- [ ] Test trigger pattern matching
- [ ] Test IPC authorization scenarios
- [ ] Test container spawn and cleanup
- [ ] Test message formatting and escaping

**Reference:** `src/*.test.ts`

### 6.3 Add Skills Tests to CI

**Why:** NanoClaw excludes skills tests from CI. Fix this.

**Actions:**
- [ ] Run both `vitest.config.ts` and `vitest.skills.config.ts` in CI
- [ ] Or consolidate into single test run with multiple config files

---

## Priority 7: Operational Excellence

### 7.1 Structured Logging

**Why:** Pinpoint production issues faster.

**Actions:**
- [ ] Use Pino for structured JSON logging
- [ ] Log levels: debug, info, warn, error, fatal
- [ ] Include context object in every log (err, chatJid, groupFolder, etc.)
- [ ] Global handlers for uncaughtException and unhandledRejection

**Reference:** `src/logger.ts`

### 7.2 Graceful Shutdown

**Why:** Prevent work loss on restart.

**Actions:**
- [ ] Don't kill active containers on shutdown
- [ ] Let them finish via idle timeout
- [ ] `--rm` flag cleans up on exit
- [ ] Log active containers for visibility

**Reference:** `src/group-queue.ts:347-364`

### 7.3 Error File Isolation

**Why:** Enable post-mortem on failed IPC processing.

**Actions:**
- [ ] Move failed IPC files to `errors/` subdirectory
- [ ] Don't delete -- preserve for debugging
- [ ] Log when files are moved to errors/

**Reference:** `src/ipc.ts`

---

## Priority 8: Features to Consider

### 8.1 Scheduled Tasks

**Why:** Valuable for automation, already well-designed in NanoClaw.

**Actions:**
- [ ] Implement cron and interval schedule types
- [ ] Anchor interval next-run to original scheduled time (not `Date.now()`)
- [ ] Implement `context_mode`: 'group' (use session) vs 'isolated' (fresh session)
- [ ] Log task runs with duration and status
- [ ] Task preemption: close idle containers for pending tasks

**Reference:** `src/task-scheduler.ts`

### 8.2 Sender Allowlist

**Why:** Access control per group.

**Actions:**
- [ ] Two modes: 'trigger' (store all, filter on trigger) and 'drop' (discard before storage)
- [ ] Per-chat configuration overrides default
- [ ] Fail-open if config missing (permissive default)
- [ ] Optional deny logging

**Reference:** `src/sender-allowlist.ts`

### 8.3 Main Channel (Self-Chat)

**Why:** Privileged admin group without trigger requirement.

**Actions:**
- [ ] `isMain: true` group bypasses trigger checking
- [ ] Main can register groups, refresh groups, schedule for any group
- [ ] Main sees all tasks and all available groups
- [ ] Different CLAUDE.md template for main group

**Reference:** `src/index.ts:206-227`, `groups/main/CLAUDE.md`

---

## Priority 9: What to Skip

### 9.1 Don't: Agent Swarms (Yet)

**Why:** Complex Telegram-specific bot pool architecture. Requires deep Telegram API knowledge.

**Defer until:** Core messaging is stable and you have Telegram-specific needs.

### 9.2 Don't: Multiple Container Runtimes

**Why:** Docker hardcoded is fine for single-platform. Podman/Apple Container adds complexity.

**Defer until:** Multi-platform support is actually needed.

### 9.3 Don't: Webhooks for Real-Time

**Why:** Polling is simpler and sufficient for personal assistant scale.

**Defer until:** Latency requirements demand webhooks.

---

## Implementation Order

```
Phase 1: Security Foundation
  - Credential vault pattern
  - Mount allowlist
  - Container isolation (non-root, /dev/null shadow)

Phase 2: Core Messaging
  - SQLite with migrations
  - Channel registry
  - Polling loop with cursors
  - Trigger processing
  - IPC via filesystem

Phase 3: Container Management
  - Container lifecycle
  - Concurrency control
  - Idle timeout reuse

Phase 4: Polish
  - Scheduled tasks
  - Sender allowlist
  - Main channel
  - Structured logging

Phase 5: Testing & CI
  - Enable coverage
  - Comprehensive tests
  - Skills tests in CI
```

---

## Key Files to Reference

| NanoClaw File | Purpose | Action Item |
|--------------|---------|-------------|
| `src/env.ts` | Credential reading | Study readEnvFile pattern |
| `src/container-runner.ts` | Container spawn, mounts | Study mount building |
| `src/container-runtime.ts` | Docker abstraction | Minimal (hardcoded docker) |
| `src/mount-security.ts` | Allowlist validation | Implement similar |
| `src/db.ts` | SQLite operations | Study migration pattern |
| `src/ipc.ts` | IPC watcher | Study authorization |
| `src/group-folder.ts` | Path validation | Copy strict pattern |
| `src/group-queue.ts` | Concurrency | Study FIFO + backoff |
| `src/index.ts` | Main loop | Study orchestration |
| `src/router.ts` | Message formatting | Study XML escaping |
| `src/config.ts` | Trigger patterns | Study regex escaping |
| `src/sender-allowlist.ts` | Access control | Study validation |
| `src/task-scheduler.ts` | Cron/interval | Study drift prevention |

---

## Summary

Building a NanoClaw-style project requires prioritizing:

1. **Security first**: Never trust containers, use credential vault, validate everything
2. **Simple IPC**: Filesystem-based IPC is elegant and secure
3. **Defensive databases**: SQLite with try/catch migrations, no migration scripts
4. **Clean errors**: Structured logging, rollback on failure, graceful shutdown
5. **Measure quality**: Enable coverage, run all tests in CI

The most valuable lesson: NanoClaw's security architecture is not an afterthought. Every decision (no process.env secrets, /dev/null shadow, external allowlist) reinforces the "never trust the container" principle.
