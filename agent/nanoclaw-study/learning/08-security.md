# NanoClaw Security

## Security Philosophy

NanoClaw's security is based on **isolation over trust**. Rather than relying on application-level permission checks, the architecture uses OS-level container isolation so that even if an agent is compromised, the blast radius is limited to what the container can access.

## Container Isolation

NanoClaw runs Claude agents inside **Linux containers** (Docker on macOS/Linux, Apple Container on macOS native).

### Container Configuration

```typescript
// src/container-runner.ts - Container runs as unprivileged node user
const CONTAINER_USER = 'node';  // uid 1000, not root
```

**Isolation properties:**
- **Filesystem isolation**: Only explicitly mounted directories are accessible
- **Safe Bash access**: Commands run inside the container, not on the host
- **Process isolation**: Container processes cannot affect the host
- **Network isolation**: Configurable per-container if needed
- **Non-root execution**: Container runs as unprivileged `node` user (uid 1000)

### Mount Security

Additional directory mounts are validated against an **allowlist** stored outside the project:

```
~/.config/nanoclaw/mount-allowlist.json
```

**Why external?** If the allowlist were inside the project folder, a compromised agent could modify it to gain access to sensitive directories.

### Default Blocked Patterns

```typescript
// src/mount-security.ts - line 29-47
const DEFAULT_BLOCKED_PATTERNS = [
  '.ssh',
  '.gnupg',
  '.gpg',
  '.aws',
  '.azure',
  '.gcloud',
  '.kube',
  '.docker',
  'credentials',
  '.env',
  '.netrc',
  '.npmrc',
  '.pypirc',
  'id_rsa',
  'id_ed25519',
  'private_key',
  '.secret',
];
```

These paths are **always blocked** regardless of allowlist configuration.

### Mount Validation Logic

```typescript
// src/mount-security.ts - line 233-329
export function validateMount(
  mount: AdditionalMount,
  isMain: boolean,
): MountValidationResult {
  const allowlist = loadMountAllowlist();

  // 1. Block if no allowlist exists
  if (allowlist === null) {
    return { allowed: false, reason: `No mount allowlist configured` };
  }

  // 2. Validate container path (no .. escapes)
  if (!isValidContainerPath(containerPath)) {
    return { allowed: false, reason: 'Invalid container path' };
  }

  // 3. Check against blocked patterns
  const blockedMatch = matchesBlockedPattern(realPath, allowlist.blockedPatterns);
  if (blockedMatch !== null) {
    return { allowed: false, reason: `Path matches blocked pattern "${blockedMatch}"` };
  }

  // 4. Check if under an allowed root
  const allowedRoot = findAllowedRoot(realPath, allowlist.allowedRoots);
  if (allowedRoot === null) {
    return { allowed: false, reason: `Path not under any allowed root` };
  }

  // 5. Enforce readonly for non-main groups
  if (!isMain && allowlist.nonMainReadOnly) {
    effectiveReadonly = true;
  }
}
```

### Path Traversal Prevention

```typescript
// src/mount-security.ts - line 202-219
function isValidContainerPath(containerPath: string): boolean {
  // Must not contain .. to prevent path traversal
  if (containerPath.includes('..')) return false;

  // Must not be absolute
  if (containerPath.startsWith('/')) return false;

  // Must not be empty
  if (!containerPath || containerPath.trim() === '') return false;

  return true;
}
```

## Credential Management

### OneCLI Agent Vault

**Key principle:** API keys and secrets **never enter the container**. Instead, outbound requests route through OneCLI Agent Vault which injects credentials at request time.

```
Container --> OneCLI Agent Vault --> External API
              (injects auth)
```

### Env File Handling

```typescript
// src/env.ts - line 11-42
/**
 * Parse the .env file and return values for the requested keys.
 * Does NOT load anything into process.env — callers decide what to
 * do with the values. This keeps secrets out of the process environment
 * so they don't leak to child processes.
 */
export function readEnvFile(keys: string[]): Record<string, string> {
  const envFile = path.join(process.cwd(), '.env');
  // ... selective parsing only requested keys
}
```

**Security note:** The env file is not loaded into `process.env` to prevent child processes from inheriting secrets.

### Selective Env Exposure

Only authentication variables are extracted and mounted:

```typescript
// docs/SPEC.md - line 400-401
// Only CLAUDE_CODE_OAUTH_TOKEN and ANTHROPIC_API_KEY are extracted
// Other .env variables are NOT exposed to the agent
```

The `.env.example` shows minimal required credentials:

```bash
# .env.example
TELEGRAM_BOT_TOKEN=
```

## Group Authorization

### Main vs Non-Main Groups

The **main channel** (self-chat) has elevated privileges. All other groups are restricted.

```typescript
// src/ipc-auth.test.ts - line 389-399
// Authorization check from IPC watcher
function isMessageAuthorized(
  sourceGroup: string,
  isMain: boolean,
  targetChatJid: string,
  registeredGroups: Record<string, RegisteredGroup>,
): boolean {
  const targetGroup = registeredGroups[targetChatJid];
  return isMain || (!!targetGroup && targetGroup.folder === sourceGroup);
}
```

### IPC Command Authorization

| Command | Main Group | Non-Main Group |
|---------|-----------|----------------|
| schedule_task (any group) | Yes | No |
| schedule_task (own group) | Yes | Yes |
| pause_task (any) | Yes | No |
| pause_task (own) | Yes | Yes |
| cancel_task (any) | Yes | No |
| cancel_task (own) | Yes | Yes |
| register_group | Yes | No |
| refresh_groups | Yes | No |

### Authorization Test Evidence

```typescript
// src/ipc-auth.test.ts - line 111-127
it('non-main group cannot schedule for another group', async () => {
  await processTaskIpc(
    {
      type: 'schedule_task',
      prompt: 'unauthorized',
      targetJid: 'main@g.us',
      ...
    },
    'other-group',
    false, // isMain = false
    deps,
  );

  const allTasks = getAllTasks();
  expect(allTasks.length).toBe(0); // Task was rejected
});
```

## Sender Allowlist

Optional per-chat sender restrictions:

```typescript
// src/sender-allowlist.ts - line 33-89
export function loadSenderAllowlist(
  pathOverride?: string,
): SenderAllowlistConfig {
  const filePath = pathOverride ?? SENDER_ALLOWLIST_PATH;

  const DEFAULT_CONFIG: SenderAllowlistConfig = {
    default: { allow: '*', mode: 'trigger' },
    chats: {},
    logDenied: true,
  };
  // ... validation and loading
}

export function isTriggerAllowed(
  chatJid: string,
  sender: string,
  cfg: SenderAllowlistConfig,
): boolean {
  const entry = getEntry(chatJid, cfg);
  if (entry.allow === '*') return true;
  return entry.allow.includes(sender);
}
```

**Configuration example:**

```json
{
  "default": { "allow": "*", "mode": "trigger" },
  "chats": {
    "family@g.us": { "allow": ["+1234567890", "+0987654321"], "mode": "trigger" }
  },
  "logDenied": true
}
```

## Message Processing Security

### Trigger Word Required

Messages must start with the trigger pattern (`@Andy` by default):

```
@Andy what's the weather? --> Processed
Hey @Andy help me         --> Ignored (trigger not at start)
What's up?                --> Ignored (no trigger)
```

### Registered Group Check

Only messages from **registered groups** are processed:

```typescript
// Router checks:
// 1. Is chat_jid in registered groups (SQLite)? --> No: ignore
// 2. Does message match trigger pattern? --> No: store but don't process
```

### Bot Message Filtering

Messages sent by the bot are filtered from conversation history:

```typescript
// src/db.ts - line 326-339
// Filter bot messages using is_bot_message flag AND content prefix
const sql = `
  SELECT * FROM (
    SELECT ... FROM messages
    WHERE timestamp > ? AND chat_jid IN (${placeholders})
      AND is_bot_message = 0 AND content NOT LIKE ?
      AND content != '' AND content IS NOT NULL
  ) ORDER BY timestamp
`;
```

## Group Folder Validation

Folder names are validated to prevent path traversal:

```typescript
// src/group-folder.ts
export function isValidGroupFolder(folder: string): boolean {
  // Must be relative, no .., no absolute paths
  // Folder names follow convention: {channel}_{group-name}
}
```

**Evidence from `src/ipc-auth.test.ts` line 352-367:**

```typescript
it('main group cannot register with unsafe folder path', async () => {
  await processTaskIpc(
    {
      type: 'register_group',
      folder: '../../outside',  // Path traversal attempt
      ...
    },
    'whatsapp_main',
    true,
    deps,
  );

  expect(groups['new@g.us']).toBeUndefined(); // Rejected
});
```

## Prompt Injection Mitigations

### Container Isolation Limits Blast Radius

Even if a message contains malicious instructions, the agent:
- Can only access mounted directories
- Cannot escape to host filesystem
- Cannot access credentials (handled by OneCLI)

### Recommendations (from SPEC.md)

The documentation advises users to:
- Only register trusted groups
- Review additional directory mounts carefully
- Review scheduled tasks periodically
- Monitor logs for unusual activity

## File Permissions

The `groups/` folder contains personal memory and should be protected:

```bash
chmod 700 groups/
```

## Summary of Security Layers

| Layer | Protection |
|-------|-----------|
| Container isolation | Filesystem, process, network isolation |
| Mount allowlist | External config, blocked patterns, path validation |
| Credential proxy | OneCLI Agent Vault, no secrets in containers |
| Group authorization | Main vs non-main IPC permissions |
| Sender allowlist | Optional per-chat restrictions |
| Trigger word | Messages must match trigger pattern |
| Registered groups | Only registered groups processed |
| Bot filtering | Bot messages excluded from conversation |
| Folder validation | Path traversal prevention |
