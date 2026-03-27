# NanoClaw Features Deep Dive

**Project:** NanoClaw - Personal Claude assistant daemon
**Synthesized:** 2026-03-26
**Source:** Batches 1-5 from `research/05*-features-batch-*.md`
**Priority Order:** Core features (Tier 1) first, then secondary features (Tier 2)

---

## Project Overview

NanoClaw is a lightweight AI assistant framework that runs Claude agents securely in isolated containers. The architecture is intentionally minimal: a single Node.js process (~15 source files) polls multiple messaging channels, routes messages to containerized agents, and streams responses back. The design philosophy prioritizes security through container isolation, per-group context isolation, and credential never-entering-containers.

**Core Architecture:**
```
Channels --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
     ^                                                                  |
     └───────────────────────── IPC via filesystem ─────────────────────┘
```

---

## Priority Tier 1: Core Features

### Feature 1: Container Isolation

**Purpose:** Agents run in isolated Linux containers with filesystem isolation. Only explicitly mounted directories are accessible inside containers.

**Files:** `src/container-runtime.ts`, `src/container-runner.ts`, `container/`

**Runtime Abstraction (`src/container-runtime.ts`):**

The container runtime abstraction provides a runtime-agnostic interface currently hardcoded to Docker:

```typescript
const CONTAINER_RUNTIME_BIN = 'docker';

function hostGatewayArgs() {
  return ['--add-host=host.docker.internal:host-gateway']; // Linux only
}

function readonlyMountArgs() {
  return '-v hostPath:containerPath:ro';
}
```

**Key security measures:**

1. **Shadow .env with /dev/null**: The main group's project root mount would expose `.env` with real credentials. Shadowing with `/dev/null` nullifies this without changing the mount path:
   ```typescript
   if (isMain) {
     const envFile = path.join(projectRoot, '.env');
     if (fs.existsSync(envFile)) {
       mounts.push({
         hostPath: '/dev/null',
         containerPath: '/workspace/project/.env',
         readonly: true,
       });
     }
   }
   ```

2. **Additional mount allowlist**: Host directories the agent can access are validated against a tamper-proof allowlist at `~/.config/nanoclaw/mount-allowlist.json`:
   ```typescript
   const DEFAULT_BLOCKED_PATTERNS = [
     '.ssh', '.gnupg', '.aws', '.azure', '.gcloud', '.kube',
     '.docker', 'credentials', '.env', '.netrc',
   ];
   ```

3. **Per-group agent-runner source copy**: Each group gets its own copy of the agent-runner source, allowing customization without affecting other groups:
   ```typescript
   const needsCopy = !fs.existsSync(cachedIndex) ||
     (fs.existsSync(srcIndex) && fs.statSync(srcIndex).mtimeMs > fs.statSync(cachedIndex).mtimeMs);
   ```

**Volume Mount Strategy:**

| Mount | Purpose | Readonly |
|-------|---------|----------|
| `/workspace/group` | Per-group working directory | No |
| `/workspace/global` | Shared global memory (non-main only) | Yes |
| `/workspace/ipc` | Inter-process communication | No |
| `/home/node/.claude` | Per-group sessions + skills | No |
| `/app/src` | Per-group agent-runner source | No |

**Output Streaming:**

Sentinel markers enable streaming parsing before container exit:
```typescript
const OUTPUT_START_MARKER = '---NANOCLAW_OUTPUT_START---';
const OUTPUT_END_MARKER = '---NANOCLAW_OUTPUT_END---';

parseBuffer += chunk;
let startIdx: number;
while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
  const endIdx = parseBuffer.indexOf(OUTPUT_END_MARKER, startIdx);
  if (endIdx === -1) break;
  const jsonStr = parseBuffer.slice(startIdx + OUTPUT_START_MARKER.length, endIdx).trim();
  const parsed: ContainerOutput = JSON.parse(jsonStr);
  outputChain = outputChain.then(() => onOutput(parsed));
}
```

**Technical Debt:**
- Docker hardcoded with no abstraction for Podman or other runtimes
- 10MB max stdout/stderr with no streaming to disk for large outputs
- No CPU/memory constraints on containers

---

### Feature 2: Multi-Channel Messaging

**Purpose:** Unified interface for messaging via WhatsApp, Telegram, Discord, Slack, and Gmail. Channels self-register via skill branches.

**Files:** `src/channels/registry.ts`, `src/channels/index.ts`, `.claude/skills/add-*`

**Self-Registration Architecture:**

The channel registry uses a factory pattern with side-effect imports:

```typescript
// src/channels/registry.ts
const registry = new Map<string, ChannelFactory>();

export function registerChannel(name: string, factory: ChannelFactory): void {
  registry.set(name, factory);
}

export function getChannelFactory(name: string): ChannelFactory | undefined {
  return registry.get(name);
}

// src/channels/index.ts - barrel file triggers self-registration
import './channels/index.js'; // Side-effect imports trigger registration
```

Each channel module (e.g., `src/channels/telegram.ts`) calls `registerChannel('telegram', factory)` when imported. Adding a new channel only requires merging a skill branch - no code changes to `index.ts` or registry.

**Channel Factory Interface:**
```typescript
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
  syncGroups?(force: boolean): Promise<void>;
}
```

**Skill-Based Channel Installation:**

Channels are added via skill branches that merge code into the repo:
```bash
git remote add telegram https://github.com/qwibitai/nanoclaw-telegram.git
git fetch telegram main
git merge telegram/main
```

The merge brings the channel implementation, tests, updates the barrel file, adds the npm dependency, and adds env vars to `.env.example`.

**JID Naming Convention:**

JIDs identify chats across channels using channel-specific prefixes:
- Telegram: `tg:123456789` or `tg:-1001234567890`
- WhatsApp: `wa:123456789`
- Slack: `slack:CHANNEL_ID`
- Discord: `discord:CHANNEL_ID`

**Clever Solutions:**

1. **Trigger accumulation**: Messages without trigger word are stored but not acted on; they become context when a triggered message arrives:
   ```typescript
   const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
   ```

2. **Cursor rollback on error**: If agent fails before sending output, message cursor rolls back to retry:
   ```typescript
   if (outputSentToUser) return true;  // Don't rollback
   lastAgentTimestamp[chatJid] = previousCursor;  // Rollback for retry
   ```

**Technical Debt:**
- Channels live in separate skill repos, increasing cross-repo coordination
- All channels use polling (SQLite every 2s + channel-specific polling). No webhooks for real-time delivery
- Bare barrel file with commented imports - future skills must uncomment

---

### Feature 3: Group-Based Context Isolation

**Purpose:** Each conversation group has its own `CLAUDE.md` memory, isolated filesystem mount, and container sandbox.

**Files:** `src/ipc.ts`, `src/group-queue.ts`, `groups/*/CLAUDE.md`, `src/group-folder.ts`

**Group Folder Structure:**
```
groups/
  global/              # Global memory (read-only for non-main)
    CLAUDE.md
  main/                # Main control group (no trigger required)
    CLAUDE.md
    logs/
  {group-name}/        # Per-registered-group
    CLAUDE.md          # Generated from template on group creation
    logs/
```

**IPC Directory Structure:**

Each group gets its own IPC directory at `data/ipc/{group-folder}/`:
```
data/ipc/{groupFolder}/
  input/       # Follow-up messages from host to container
  messages/    # Outbound messages written by container
  tasks/       # Scheduled task IPC files
```

**Authorization Model:**

IPC operations verify identity via directory path and `isMain` flag:
```typescript
// Non-main groups can only schedule for themselves
if (!isMain && targetFolder !== sourceGroup) {
  logger.warn({ sourceGroup, targetFolder }, 'Unauthorized schedule_task attempt blocked');
}

// Main can send anywhere; others only to themselves
if (isMain || (targetGroup && targetGroup.folder === sourceGroup)) {
  await deps.sendMessage(data.chatJid, data.text);
}
```

**Path Validation:**

Group folders validated against strict pattern to prevent path traversal:
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

**Template Substitution:**

On group creation, assistant name is replaced in CLAUDE.md template:
```typescript
if (ASSISTANT_NAME !== 'Andy') {
  content = content.replace(/^# Andy$/m, `# ${ASSISTANT_NAME}`);
  content = content.replace(/You are Andy/g, `You are ${ASSISTANT_NAME}`);
}
```

**Technical Debt:**
- No group deletion (`unregisterGroup` IPC type missing)
- Race condition in recovery: `recoverPendingMessages()` uses `lastAgentTimestamp` which may not reflect in-progress processing from crashed instance
- Global concurrency limit shared across all groups regardless of priority

---

### Feature 4: Message Orchestration

**Purpose:** Single Node.js process polling loop that routes messages from all channels to Claude Agent SDK and back.

**Files:** `src/index.ts`, `src/router.ts`

**Core Components:**
- `startMessageLoop()` - Main polling loop
- `processGroupMessages()` - Processes pending messages for a specific group
- `runAgent()` - Invokes containerized Claude agent

**Polling Loop Structure:**
```typescript
while (true) {
  const jids = Object.keys(registeredGroups);
  const { messages, newTimestamp } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);

  if (messages.length > 0) {
    lastTimestamp = newTimestamp;  // Atomic cursor advance
    const messagesByGroup = new Map<string, NewMessage[]>();
    // ... grouping logic

    for (const [chatJid, groupMessages] of messagesByGroup) {
      if (queue.sendMessage(chatJid, formatted)) {
        // Piped to active container
      } else {
        queue.enqueueMessageCheck(chatJid);  // Start new container
      }
    }
  }
  await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL));
}
```

**Key Design Decisions:**

1. **Global message cursor**: `lastTimestamp` advances for ALL messages across ALL groups in one atomic write, preventing duplicate processing on crash.

2. **Per-group agent cursor**: `lastAgentTimestamp[chatJid]` tracks the last message each group's agent has consumed, enabling context accumulation between triggers.

3. **Trigger filtering with context preservation**: Non-trigger messages accumulate in SQLite. When a trigger arrives, `getMessagesSince()` pulls everything since `lastAgentTimestamp`.

**Message Formatting (XML Context):**
```typescript
export function formatMessages(messages: NewMessage[], timezone: string): string {
  const lines = messages.map((m) => {
    const displayTime = formatLocalTime(m.timestamp, timezone);
    return `<message sender="${escapeXml(m.sender_name)}" time="${escapeXml(displayTime)}">${escapeXml(m.content)}</message>`;
  });
  const header = `<context timezone="${escapeXml(timezone)}" />\n`;
  return `${header}<messages>\n${lines.join('\n')}\n</messages>`;
}

export function escapeXml(s: string): string {
  if (!s) return '';
  return s
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}
```

**Idle Timeout:**
```typescript
const resetIdleTimer = () => {
  if (idleTimer) clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    queue.closeStdin(chatJid);  // Signal container to wind down
  }, IDLE_TIMEOUT);  // Default: 30 minutes
};
```

**Clever Solution - Cursor Rollback on Error:**
If the agent fails after sending output to the user, the cursor is NOT rolled back to prevent duplicate responses. Only when no output was sent does it roll back for retry.

---

### Feature 5: Scheduled Tasks

**Purpose:** Cron-style recurring jobs that run Claude agents automatically and can message results back.

**Files:** `src/task-scheduler.ts`, `src/timezone.ts`

**Schedule Types:**
```typescript
interface ScheduledTask {
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;      // cron expr, ms interval, or ignored
  context_mode: 'group' | 'isolated';  // Use group session or fresh
  next_run: string | null;
  last_run: string | null;
  last_result: string | null;
  status: 'active' | 'paused' | 'completed';
}
```

**Drift Prevention via Time Anchoring:**

For interval tasks, next run is calculated from `task.next_run` (the original scheduled time), NOT `Date.now()`:
```typescript
if (task.schedule_type === 'interval') {
  const ms = parseInt(task.schedule_value, 10);
  let next = new Date(task.next_run!).getTime() + ms;
  while (next <= now) { next += ms; }  // Catch up missed intervals
  return new Date(next).toISOString();
}
```

**Task/Message Container Distinction:**

Tasks run through the same `GroupQueue` as messages but with special handling:
- `isTaskContainer` flag prevents `sendMessage()` from piping to task containers
- Tasks close stdin after 10 seconds (`TASK_CLOSE_DELAY_MS`) vs 30 minutes for message containers
- Tasks use prompt delivery via `runContainerAgent()` rather than streaming

**Context Modes:**
- `'group'`: Uses the group's current conversation session
- `'isolated'`: Creates a fresh session for each task run

**Technical Debt:**
- 60-second scheduler poll interval (misses sub-minute cron precision)
- No distributed scheduling - single Node.js process
- No global task deduplication (only per-group `runningTaskId` check)

---

### Feature 6: Per-Group Queue with Concurrency Control

**Purpose:** Global concurrency limiting ensures resources are shared fairly across groups while preventing overload.

**Files:** `src/group-queue.ts`

**State Tracking:**
```typescript
interface GroupState {
  active: boolean;
  idleWaiting: boolean;           // Container finished but still alive
  isTaskContainer: boolean;       // True for scheduled tasks
  runningTaskId: string | null;   // Prevents double-task execution
  pendingMessages: boolean;
  pendingTasks: QueuedTask[];
  process: ChildProcess | null;
  containerName: string | null;
  groupFolder: string | null;
  retryCount: number;
}
```

**Concurrency Limit:** `MAX_CONCURRENT_CONTAINERS` (default: 5, configurable via env)

**Waiting Groups FIFO:**
```typescript
private drainWaiting(): void {
  while (
    this.waitingGroups.length > 0 &&
    this.activeCount < MAX_CONCURRENT_CONTAINERS
  ) {
    const nextJid = this.waitingGroups.shift()!;  // FIFO fairness
    const state = this.getGroup(nextJid);

    // Tasks take priority over messages
    if (state.pendingTasks.length > 0) {
      this.runTask(nextJid, state.pendingTasks.shift()!);
    } else if (state.pendingMessages) {
      this.runForGroup(nextJid, 'drain');
    }
  }
}
```

**Atomic File Writes for IPC:**
```typescript
const tempPath = `${filepath}.tmp`;
fs.writeFileSync(tempPath, JSON.stringify({ type: 'message', text }));
fs.renameSync(tempPath, filepath);  // Atomic rename prevents partial messages
```

**Exponential Backoff Retry:**
```typescript
private scheduleRetry(groupJid: string, state: GroupState): void {
  state.retryCount++;
  if (state.retryCount > MAX_RETRIES) {  // 5 max
    logger.error({ groupJid }, 'Max retries exceeded');
    return;
  }
  const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
  // 5s, 10s, 20s, 40s, 80s
}
```

**Graceful Shutdown:**

Containers are "detached" not killed, allowing them to finish naturally via `IDLE_TIMEOUT`. The `--rm` flag cleans them up on exit.

---

### Feature 7: Credential Security (Agent Vault)

**Purpose:** API keys and tokens never enter containers. Outbound requests route through OneCLI Agent Vault which injects credentials at request time.

**Files:** `src/env.ts`, `src/config.ts`, `src/container-runner.ts`, `.claude/skills/init-onecli/`

**Security Model:**

| Layer | Protection |
|-------|-----------|
| `env.ts` | Secrets stay out of `process.env` |
| `.env` shadowing | Main group's container can't read `.env` via `/dev/null` |
| OneCLI gateway | Intercepts HTTPS, injects at request time |
| Mount blocklist | Prevents mounting credential-adjacent paths |
| Per-group agents | OneCLI supports per-agent policies and rate limits |

**Environment Variable Isolation:**

```typescript
export function readEnvFile(keys: string[]): Record<string, string> {
  const envFile = path.join(process.cwd(), '.env');
  // Does NOT load into process.env — callers decide what to do
  // This keeps secrets out of the process environment so they don't leak
}
```

**Container Credential Injection:**

```typescript
// OneCLI gateway handles credential injection — containers never see secrets
const onecliApplied = await onecli.applyContainerConfig(args, {
  addHostMapping: false,
  agent: agentIdentifier,
});
if (onecliApplied) {
  logger.info({ containerName }, 'OneCLI gateway config applied');
} else {
  logger.warn({ containerName }, 'OneCLI gateway not reachable');
}
```

**Mount Security Blocklist:**
```typescript
const DEFAULT_BLOCKED_PATTERNS = [
  '.ssh', '.gnupg', '.aws', '.azure', '.gcloud', '.kube',
  '.docker', 'credentials', '.env', '.netrc',
];
```

**Two Credential Approaches:**

1. **OneCLI Agent Vault** (default): External gateway service intercepts HTTPS traffic
2. **Native Credential Proxy** (optional skill): Built-in HTTP proxy on port 3001 reading from `.env`

**Technical Debt:**
- OneCLI SDK coupling: If OneCLI changes its API, `applyContainerConfig()` could break
- Native credential proxy (`src/credential-proxy.ts`) not in main branch - merged from skill

---

## Priority Tier 2: Secondary Features

### Feature 8: Web Access (Browser Automation)

**Purpose:** Agents can search and fetch content from the web via the `agent-browser` CLI tool.

**Files:** `container/Dockerfile`, `container/skills/agent-browser/SKILL.md`

**Container Setup:**

The container image includes Chromium and `agent-browser`:
```dockerfile
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium
RUN npm install -g agent-browser @anthropic-ai/claude-code
```

**Agent-Browser CLI Interface:**
```bash
agent-browser open <url>        # Navigate to page
agent-browser snapshot -i       # Get interactive elements with refs
agent-browser click @e1        # Click element by ref
agent-browser fill @e2 "text"   # Fill input by ref
agent-browser close             # Close browser
```

**Authentication Pattern:**
```bash
agent-browser open https://app.example.com/login
agent-browser state save auth.json  # After logging in once
# Later: load saved state
agent-browser state load auth.json
agent-browser open https://app.example.com/dashboard
```

**Technical Note:** The `container/skills/agent-browser/SKILL.md` is purely documentation. The actual `agent-browser` CLI is a separate npm package installed globally in the container.

---

### Feature 9: Trigger Word Processing

**Purpose:** Messages are filtered by trigger word (default: `@Andy`) for targeted assistant interaction.

**Files:** `src/config.ts`, `src/sender-allowlist.ts`

**Trigger Pattern:**
```typescript
export function buildTriggerPattern(trigger: string): RegExp {
  return new RegExp(`^${escapeRegex(trigger.trim())}\\b`, 'i');
}
// Pattern: ^@Andy\b  (must be at start, word boundary after)
// Matches: "@Andy hello", "@Andy's thing"
// Does NOT match: "hello @Andy", "@Andyextra"
```

**Trigger Checking Flow:**
```
Message arrives
       |
Is main group? --> Yes --> Process immediately
       |
requiresTrigger === false? --> Yes --> Process immediately
       |
Has trigger word AND sender allowed?
       |
       +-- Yes --> Pull all messages since lastAgentTimestamp
       |         (including accumulated context)
       +-- No --> Skip (messages stay in DB for later)
```

**Per-Group Trigger Configuration:**

Each group can have a custom trigger stored in SQLite `registered_groups` table:
```typescript
interface RegisteredGroup {
  trigger: string;
  isMain?: boolean;
}
```

**`is_from_me` Bypass:**

Messages from the assistant itself bypass trigger check, allowing follow-ups and tool use continuations.

**Technical Note:** The `\b` word boundary with apostrophe (`@Andy's`) is clever edge case handling - JavaScript's `\b` treats apostrophe as a word character, so `@Andy` matches before `'s`.

---

### Feature 10: Main Channel (Self-Chat)

**Purpose:** Private admin channel for controlling groups, listing tasks, and managing the assistant without group context.

**Files:** `groups/main/CLAUDE.md`, `src/index.ts`, `src/ipc.ts`

**Privileges of Main Group:**

| Operation | Main | Non-Main |
|-----------|------|----------|
| Trigger required | No | Yes |
| Register groups | Yes | No |
| Schedule tasks for other groups | Yes | No |
| See all tasks | Yes | No |
| See all available groups | Yes | No |
| Access project root | Read-only | No |

**IPC Privileges (enforced in `src/ipc.ts`):**

1. Task management across groups: Main can pause/resume/cancel any task
2. Scheduling across groups: Main can `schedule_task` for any group
3. Group registration: Only main can call `register_group` IPC
4. Group refresh: Only main can trigger `refresh_groups`

**Defense in Depth:**

The agent cannot set `isMain` via IPC even if it tries:
```typescript
// Defense in depth: agent cannot set isMain via IPC
deps.registerGroup(data.jid, {
  // ... other fields
  // isMain is deliberately NOT set from IPC data
});
```

---

### Feature 11: Agent Swarms

**Purpose:** Spin up teams of specialized agents that collaborate on complex multi-step tasks.

**Files:** `.claude/skills/add-telegram-swarm/SKILL.md`, `container/agent-runner/src/ipc-mcp-stdio.ts`

**Core Concept:**

1. Main agent spawns sub-agents (teammates) for parallel work
2. Each sub-agent uses `mcp__nanoclaw__send_message` with a `sender` parameter
3. IPC watcher routes these through a Telegram bot pool
4. Each bot in the pool gets renamed to match its role

**MCP Tool Interface:**
```typescript
server.tool(
  'send_message',
  "Send a message to the user or group...",
  {
    text: z.string(),
    sender: z.string().optional().describe('Your role/identity name (e.g. "Researcher")'),
  },
  async (args) => {
    const data: Record<string, string | undefined> = {
      type: 'message',
      chatJid,
      text: args.text,
      sender: args.sender || undefined,
      groupFolder,
      timestamp: new Date().toISOString(),
    };
    writeIpcFile(MESSAGES_DIR, data);
    return { content: [{ type: 'text', text: 'Message sent.' }] };
  },
);
```

**Bot Pool Architecture:**

When a sub-agent sends `sender: "Marine Biologist"`:
1. Looks up sender name in pool's `senderBotMap`
2. If not mapped, assigns unused bot and renames via `setMyName`
3. Sends message from that bot
4. 2-second delay after `setMyName` for Telegram propagation

**Technical Debt:**
- Bot name global state: `setMyName` changes bot name globally, not per-chat
- Sender-to-bot mapping resets on service restart
- Telegram caches names client-side

---

### Feature 12: Sender Allowlist

**Purpose:** Security mechanism to restrict which senders can trigger the assistant.

**Files:** `src/sender-allowlist.ts`, `src/index.ts`

**Configuration Schema:**
```typescript
interface ChatAllowlistEntry {
  allow: '*' | string[];  // '*' = everyone, or specific sender IDs
  mode: 'trigger' | 'drop';  // 'trigger' = require trigger, 'drop' = silent drop
}

interface SenderAllowlistConfig {
  default: ChatAllowlistEntry;
  chats: Record<string, ChatAllowlistEntry>;  // per-chat overrides
  logDenied: boolean;
}
```

**Two-Mode Design:**

1. **trigger mode** (default): All messages stored, but only allowed senders can trigger assistant
2. **drop mode**: Messages from denied senders discarded before storage

**Config File Location:** `~/.config/nanoclaw/sender-allowlist.json` (outside project, never mounted)

**Fail-Open Design:** If config file missing or invalid, all senders are allowed.

---

### Feature 13: Channel Formatting

**Purpose:** Normalizes message formatting across different channel formats.

**Files:** `src/router.ts`, `.claude/skills/channel-formatting/`

**Inbound Formatting (XML Context):**

Messages formatted as XML for Claude agent:
```xml
<context timezone="America/New_York" />
<messages>
<message sender="John" time="2026-03-26 10:30:00">Hello there</message>
<message sender="Andy" time="2026-03-26 10:31:00">Hi John!</message>
</messages>
```

**Outbound Formatting (Internal Tag Stripping):**

```typescript
export function stripInternalTags(text: string): string {
  return text.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
}
```

**Channel-Formatting Skill (per-channel Markdown):**

| Channel | Transformation |
|---------|---------------|
| WhatsApp | `**bold**` to `*bold*`, `*italic*` to `_italic_` |
| Telegram | Same as WhatsApp |
| Slack | Same, links become `<url\|text>` |
| Discord | Passthrough (renders Markdown natively) |
| Signal | Passthrough + native `textStyle` ranges |

**Technical Note:** The `src/text-styles.ts` file doesn't exist in this repo - the channel-formatting skill branch hasn't been merged yet. The current `formatOutbound()` doesn't take a `channel` parameter.

---

### Feature 14: Database Persistence (SQLite)

**Purpose:** Local storage for messages, groups, sessions, and application state.

**Files:** `src/db.ts`, `src/db.test.ts`, `src/db-migration.test.ts`

**Schema:**

```sql
CREATE TABLE chats (
  jid TEXT PRIMARY KEY,
  name TEXT,
  last_message_time TEXT,
  channel TEXT,
  is_group INTEGER DEFAULT 0
);

CREATE TABLE messages (
  id TEXT,
  chat_jid TEXT,
  sender TEXT,
  sender_name TEXT,
  content TEXT,
  timestamp TEXT,
  is_from_me INTEGER,
  is_bot_message INTEGER DEFAULT 0,
  PRIMARY KEY (id, chat_jid),
  FOREIGN KEY (chat_jid) REFERENCES chats(jid)
);

CREATE TABLE scheduled_tasks (
  id TEXT PRIMARY KEY,
  group_folder TEXT NOT NULL,
  chat_jid TEXT NOT NULL,
  prompt TEXT NOT NULL,
  schedule_type TEXT NOT NULL,
  schedule_value TEXT NOT NULL,
  next_run TEXT,
  last_run TEXT,
  last_result TEXT,
  status TEXT DEFAULT 'active',
  context_mode TEXT DEFAULT 'isolated',
  script TEXT
);
```

**Defensive Migration Pattern:**
```typescript
try {
  database.exec(
    `ALTER TABLE scheduled_tasks ADD COLUMN context_mode TEXT DEFAULT 'isolated'`,
  );
} catch {
  /* column already exists */
}
```

**Clever Pattern - Subquery for Recent Messages:**
```typescript
const sql = `
  SELECT * FROM (
    SELECT ... FROM messages
    WHERE chat_jid = ? AND timestamp > ?
    ORDER BY timestamp DESC
    LIMIT ?
  ) ORDER BY timestamp
`;
```
Subquery fetches N most recent (DESC), outer query re-sorts chronologically (ASC).

**ON CONFLICT DO UPDATE Pattern:**
```typescript
db.prepare(`
  INSERT INTO chats (jid, name, last_message_time, channel, is_group)
  VALUES (?, ?, ?, ?, ?)
  ON CONFLICT(jid) DO UPDATE SET
    last_message_time = MAX(last_message_time, excluded.last_message_time),
    channel = COALESCE(excluded.channel, channel),
    is_group = COALESCE(excluded.is_group, is_group)
`).run(chatJid, name || chatJid, timestamp, channel, isGroup ? 1 : 0);
```

**Technical Debt:**
- No foreign key enforcement on `messages.chat_jid` (SQLite requires `PRAGMA foreign_keys = ON`)
- `is_bot_message` backfill by content prefix (`Andy:`) could miss bot messages with different format
- All timestamps stored as ISO8601 strings, not Unix epoch integers

---

## Cross-Feature Interactions

### Container Isolation + Group Context

Containers are spawned per-message-batch, not per-group. Each container's filesystem view is isolated via volume mounts:

```
Container for group-A:
  /workspace/group    -> groups/main/
  /workspace/global   -> groups/global/
  /workspace/ipc      -> data/ipc/{group-folder}/
  /home/node/.claude  -> data/sessions/{group-folder}/.claude/

Container for group-B:
  /workspace/group    -> groups/other_group/
  /workspace/global   -> groups/global/
  /workspace/ipc      -> data/ipc/{other_group}/
  /home/node/.claude  -> data/sessions/{other_group}/.claude/
```

### IPC + Container Isolation

IPC files flow from container to host via mounted directories:
- Container writes response to `/workspace/ipc/messages/{uuid}.json`
- Host IPC watcher reads, sends via channel, deletes file
- Host writes follow-up to `/workspace/ipc/input/{uuid}.json`
- Container reads input file (polled by agent-runner)
- `_close` sentinel in `input/` signals graceful termination

### Trigger + Allowlist + Credential Security

When a trigger message is processed:
1. Sender allowlist checked first
2. If allowed, message formatted and sent to container
3. Container has no credentials - OneCLI gateway injects at HTTP time
4. Response routes back through `formatOutbound()` which strips `<internal>` tags

### Message Orchestration + Group Queue

```
startMessageLoop()
    |
    v
getNewMessages() --> SQLite
    |
    v
queue.sendMessage() --(if active)--> IPC file write
    |
    v (if no active container)
queue.enqueueMessageCheck()
    |
    v
GroupQueue.runForGroup()
    |
    v
processGroupMessages() --> runAgent() --> Container
    |
    v
Streaming callback --> channel.sendMessage()
    |
    v
queue.notifyIdle() --> closeStdin() if pending tasks
```

---

## Key Design Patterns Summary

| Pattern | Implementation | Purpose |
|---------|---------------|---------|
| Self-registering channels | Side-effect imports in barrel file | Zero-config channel addition |
| XML context | XML-escaped messages with sender/time | Structured prompt injection |
| Atomic file renames | `writeFileSync` + `renameSync` | Prevent partial IPC messages |
| Time anchoring | `task.next_run + interval` not `now + interval` | Prevent schedule drift |
| Global cursor | `lastTimestamp` atomic advance | Exactly-once message processing |
| Per-group cursor | `lastAgentTimestamp[chatJid]` | Context accumulation between triggers |
| FIFO waiting queue | `waitingGroups.shift()` | Fairness across groups |
| Exponential backoff | `BASE_RETRY_MS * Math.pow(2, n)` | Graceful degradation |
| Task preemption | `notifyIdle()` checks `pendingTasks` | Priority for scheduled work |
| Graceful shutdown | Containers detached, not killed | Prevent work loss on restart |
| Drift prevention | Time anchoring for interval tasks | Maintain schedule accuracy |

---

## File Reference

| File | Purpose |
|------|---------|
| `src/container-runtime.ts` | Runtime abstraction (Docker) |
| `src/container-runner.ts` | Container spawning, mount building, output parsing |
| `src/channels/registry.ts` | Channel factory registry |
| `src/channels/index.ts` | Channel self-registration barrel |
| `src/index.ts` | Main orchestrator, channel setup, message loop |
| `src/router.ts` | Message formatting, channel routing |
| `src/group-queue.ts` | Per-group concurrency control |
| `src/group-folder.ts` | Path validation and resolution |
| `src/ipc.ts` | IPC watcher, task processing, authorization |
| `src/task-scheduler.ts` | Scheduled task execution |
| `src/env.ts` | Environment variable isolation |
| `src/config.ts` | Configuration constants |
| `src/sender-allowlist.ts` | Sender allowlist logic |
| `src/db.ts` | SQLite database operations |
| `src/types.ts` | Type definitions |
| `groups/*/CLAUDE.md` | Per-group memory templates |
| `container/skills/agent-browser/SKILL.md` | Browser automation documentation |
| `.claude/skills/add-*/SKILL.md` | Channel installation skills |
