# Security & Performance Analysis: nanoclaw

## Security Model

### Trust Architecture

| Entity | Trust Level | Rationale |
|--------|-------------|-----------|
| Main group | Trusted | Private self-chat, admin control |
| Non-main groups | Untrusted | Other users may be malicious |
| Container agents | Sandboxed | Isolated execution via Linux containers |
| Incoming messages | User input | Potential prompt injection |

### Security Boundaries

```
UNTRUSTED ZONE                     HOST PROCESS (TRUSTED)           CONTAINER (SANDBOXED)
Incoming Messages ──────────────> Container Lifecycle ──────────> Agent Execution
(potentially malicious)           Mount Validation                  No real credentials
                                   IPC Authorization                 Limited filesystem
                                   OneCLI Agent Vault               Network via gateway
```

---

## Secrets Management

### OneCLI Agent Vault (`@onecli/sh/sdk`)

Real API credentials **never enter containers**. The OneCLI gateway proxies all outbound HTTPS requests.

**Flow:**
1. Credentials registered once with `onecli secrets create` (stored by OneCLI, not nanoclaw)
2. On container spawn, `onecli.applyContainerConfig()` routes outbound traffic through the gateway
3. Gateway matches requests by host/path and injects the real credential
4. Agents cannot discover real credentials -- not in environment, stdin, files, or `/proc`

**Implementation** (`src/container-runner.ts:236-249`):
```typescript
const onecliApplied = await onecli.applyContainerConfig(args, {
  addHostMapping: false,
  agent: agentIdentifier,
});
if (!onecliApplied) {
  logger.warn('OneCLI gateway not reachable -- container will have no credentials');
}
```

### `.env` Shadowing

The `.env` file in the project root is shadowed with `/dev/null` in container mounts so agents cannot read secrets even if the project root is mounted:

```typescript
// src/container-runner.ts:81-90
const envFile = path.join(projectRoot, '.env');
if (fs.existsSync(envFile)) {
  mounts.push({
    hostPath: '/dev/null',
    containerPath: '/workspace/project/.env',
    readonly: true,
  });
}
```

### Per-Group Agent Identity

Each NanoClaw group gets its own OneCLI agent identity, enabling per-group credential policies.

---

## Container Isolation

### Docker Configuration

**Image:** `node:22-slim` (not `node:22` -- slim variant reduces attack surface)

**Runtime user:** Non-root `node` user (uid 1000):
```dockerfile
# container/Dockerfile:60-64
RUN chown -R node:node /workspace && chmod 777 /home/node
USER node
```

**Ephemeral containers:** `--rm` flag ensures containers are destroyed after use, preventing state leakage between invocations.

**Filesystem defaults:**
- Main group: project root mounted **read-only** at `/workspace/project`
- Non-main groups: no project root access
- Group folder: **read-write** at `/workspace/group`
- Global memory: **read-only** at `/workspace/global`

### Workspace Layout

```
/workspace/project   # Main group only (ro) - source code, cannot be modified
/workspace/group    # Per-group isolated filesystem
/workspace/global   # Shared global state (ro for non-main)
/workspace/extra    # Additional user mounts
/workspace/ipc/    # Per-group IPC (messages, tasks, input)
```

---

## Input Validation & Sanitization

### Group Folder Validation

Strict allowlist validation prevents path traversal and injection:

**`src/group-folder.ts:5-16`:**
```typescript
const GROUP_FOLDER_PATTERN = /^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$/;
const RESERVED_FOLDERS = new Set(['global']);

export function isValidGroupFolder(folder: string): boolean {
  if (!folder) return false;
  if (folder !== folder.trim()) return false;
  if (!GROUP_FOLDER_PATTERN.test(folder)) return false;
  if (folder.includes('/') || folder.includes('\\')) return false;
  if (folder.includes('..')) return false;
  if (RESERVED_FOLDERS.has(folder.toLowerCase())) return false;
  return true;
}
```

Also uses `ensureWithinBase()` to verify resolved paths do not escape the base directory after symlink resolution.

### Trigger Pattern Regex Escaping

Trigger patterns are escaped to prevent regex injection:

**`src/config.ts:63-68`:**
```typescript
function escapeRegex(str: string): string {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

export function buildTriggerPattern(trigger: string): RegExp {
  return new RegExp(`^${escapeRegex(trigger.trim())}\\b`, 'i');
}
```

### XML Output Escaping

All message content is XML-escaped before being sent to the container agent:

**`src/router.ts:4-11`:**
```typescript
export function escapeXml(s: string): string {
  if (!s) return '';
  return s
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

### IPC Authorization

IPC operations are verified against the group identity derived from the directory path:

**`src/ipc.ts:76-94`:**
```typescript
// Authorization: verify this group can send to this chatJid
const targetGroup = registeredGroups[data.chatJid];
if (isMain || (targetGroup && targetGroup.folder === sourceGroup)) {
  await deps.sendMessage(data.chatJid, data.text);
} else {
  logger.warn({ chatJid: data.chatJid, sourceGroup }, 'Unauthorized IPC message attempt blocked');
}
```

### Sender Allowlist Validation

**`src/sender-allowlist.ts:23-31`:**
```typescript
function isValidEntry(entry: unknown): entry is ChatAllowlistEntry {
  if (!entry || typeof entry !== 'object') return false;
  const e = entry as Record<string, unknown>;
  const validAllow = e.allow === '*' ||
    (Array.isArray(e.allow) && e.allow.every((v) => typeof v === 'string'));
  const validMode = e.mode === 'trigger' || e.mode === 'drop';
  return validAllow && validMode;
}
```

### Cron Expression Validation

**`src/ipc.ts:217-228`:**
```typescript
if (scheduleType === 'cron') {
  try {
    const interval = CronExpressionParser.parse(data.schedule_value, { tz: TIMEZONE });
    nextRun = interval.next().toISOString();
  } catch {
    logger.warn({ scheduleValue: data.schedule_value }, 'Invalid cron expression');
    break;
  }
}
```

### SQL Injection Prevention

All database queries use parameterized statements with `better-sqlite3`:

**`src/db.ts:341-343`:**
```typescript
const rows = db
  .prepare(sql)
  .all(lastTimestamp, ...jids, `${botPrefix}:%`, limit) as NewMessage[];
```

---

## Mount Security

### External Allowlist

The mount allowlist is stored **outside** the project root at `~/.config/nanoclaw/mount-allowlist.json`, making it tamper-proof from container agents.

### Default Blocked Patterns

```typescript
const DEFAULT_BLOCKED_PATTERNS = [
  '.ssh', '.gnupg', '.gpg', '.aws', '.azure', '.gcloud', '.kube',
  '.docker', 'credentials', '.env', '.netrc', '.npmrc', '.pypirc',
  'id_rsa', 'id_ed25519', 'private_key', '.secret',
];
```

### Symlink Resolution

Paths are resolved via `fs.realpathSync()` before validation to prevent symlink traversal attacks:

**`src/mount-security.ts:139-145`:**
```typescript
function getRealPath(p: string): string | null {
  try {
    return fs.realpathSync(p);
  } catch {
    return null;
  }
}
```

### Container Path Validation

**`src/mount-security.ts:202-219`:**
```typescript
function isValidContainerPath(containerPath: string): boolean {
  if (containerPath.includes('..')) return false;    // No traversal
  if (containerPath.startsWith('/')) return false;   // No absolute paths
  if (!containerPath || containerPath.trim() === '') return false;
  return true;
}
```

---

## Performance Patterns

### Concurrency Control

**`src/config.ts:58-61`:**
```typescript
export const MAX_CONCURRENT_CONTAINERS = Math.max(
  1,
  parseInt(process.env.MAX_CONCURRENT_CONTAINERS || '5', 10) || 5,
);
```

The `GroupQueue` (`src/group-queue.ts`) implements:
- Waiting list for groups when at concurrency limit
- Task priority over messages (tasks won't be re-discovered like messages)
- Exponential backoff retry (BASE_RETRY_MS * 2^retryCount) for failed groups

### Caching

| Cache | Location | Invalidation |
|-------|----------|--------------|
| Mount allowlist | `cachedAllowlist` in `mount-security.ts` | Process restart only |
| Session state | In-memory `sessions` map | On write to SQLite |
| Group state | In-memory `registeredGroups` map | On write to SQLite |

### Timeouts

| Timeout | Default | Purpose |
|---------|---------|---------|
| `CONTAINER_TIMEOUT` | 30 min | Hard limit per container invocation |
| `IDLE_TIMEOUT` | 30 min | Keep-alive waiting for follow-up messages |
| `TASK_CLOSE_DELAY_MS` | 10 sec | Close task containers promptly after result |

**Graceful shutdown:** Container stop uses `-t 1` (1 second grace), with hard `SIGKILL` after 15 seconds if graceful fails.

### Pagination & Streaming

**Message queries** (`src/db.ts:329-350`):
- Subquery pattern: inner query gets N most recent, outer query re-sorts chronologically
- Explicit `LIMIT` parameter (default 200)
- `is_bot_message` flag + content prefix backstop for pre-migration messages

**Container output** (`src/container-runner.ts`):
- Streaming parse via `OUTPUT_START_MARKER` / `OUTPUT_END_MARKER` sentinel pairs
- Output chain: `Promise` chain ensures ordered processing
- 10MB max output size (`CONTAINER_MAX_OUTPUT_SIZE`) with truncation flag

### Container Reuse

Containers are **kept alive** via idle timeout waiting for follow-up messages, avoiding the cost of container startup for conversational follow-ups. Only task containers close promptly after producing a result.

**`src/container-runner.ts:447-451`:**
```typescript
const resetTimeout = () => {
  clearTimeout(timeout);
  timeout = setTimeout(killOnTimeout, timeoutMs);
};
```

### Task Scheduling

**Drift prevention** via anchored computation (`src/task-scheduler.ts:53-59`):
```typescript
// Anchor to the scheduled time, not now, to prevent drift.
let next = new Date(task.next_run!).getTime() + ms;
while (next <= now) {
  next += ms;
}
```

### Agent Runner Build Cache

The agent-runner source is copied to a per-group directory (`data/sessions/{group}/agent-runner-src/`) and only re-copied if the source is newer than the cached copy:

**`src/container-runner.ts:197-206`:**
```typescript
const needsCopy =
  !fs.existsSync(groupAgentRunnerDir) ||
  !fs.existsSync(cachedIndex) ||
  (fs.existsSync(srcIndex) &&
    fs.statSync(srcIndex).mtimeMs > fs.statSync(cachedIndex).mtimeMs);
```

---

## Security Summary

| Category | Implementation |
|----------|---------------|
| Secrets | OneCLI Agent Vault gateway injection |
| Container isolation | Docker, non-root user, ephemeral, read-only project root |
| Filesystem access | External allowlist, blocked patterns, symlink resolution |
| Input validation | Regex escaping, XML escaping, type checking, parameterization |
| IPC authorization | Group identity from directory path |
| SQL injection | Parameterized queries only |

## Performance Summary

| Category | Implementation |
|----------|---------------|
| Concurrency | MAX_CONCURRENT_CONTAINERS (default 5), GroupQueue with waiting |
| Caching | In-memory allowlist, sessions, group state |
| Timeouts | Configurable container/idle/task timeouts |
| Pagination | LIMIT on message queries (default 200) |
| Streaming | Sentinel markers, Promise chain ordering |
| Container reuse | Idle timeout keeps containers warm for follow-ups |
| Task scheduling | Drift-prevention via anchored computation |
