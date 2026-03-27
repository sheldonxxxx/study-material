# Features 13-14: Channel Formatting and Database Persistence

## Feature 13: Channel Formatting

### Overview

**Feature:** Message normalization/transformation
**Key File:** `src/router.ts` (~53 lines)
**Skill:** `.claude/skills/channel-formatting/` (skill branch `skill/channel-formatting`)

### Architecture

Channel formatting operates bidirectionally:

1. **Inbound (messages to agent):** `formatMessages()` in `router.ts` transforms `NewMessage[]` into XML context for the Claude agent
2. **Outbound (agent responses):** `formatOutbound()` strips `<internal>` tags; the channel-formatting skill adds per-channel Markdown transformations

### Core Implementation

#### XML Context Formatting (Inbound)

```typescript
// src/router.ts - formatMessages
export function escapeXml(s: string): string {
  if (!s) return '';
  return s
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

export function formatMessages(
  messages: NewMessage[],
  timezone: string,
): string {
  const lines = messages.map((m) => {
    const displayTime = formatLocalTime(m.timestamp, timezone);
    return `<message sender="${escapeXml(m.sender_name)}" time="${escapeXml(displayTime)}">${escapeXml(m.content)}</message>`;
  });

  const header = `<context timezone="${escapeXml(timezone)}" />\n`;
  return `${header}<messages>\n${lines.join('\n')}\n</messages>`;
}
```

**Key design decisions:**
- Uses XML format for structured message context (not JSON or plain text)
- `displayTime` is converted to local timezone via `formatLocalTime()` before embedding
- All content is XML-escaped to prevent injection (XSS protection)
- Sender names are escaped in attributes, content is escaped in text nodes

#### Internal Tag Stripping (Outbound)

```typescript
export function stripInternalTags(text: string): string {
  return text.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
}

export function formatOutbound(rawText: string): string {
  const text = stripInternalTags(rawText);
  if (!text) return '';
  return text;
}
```

**Purpose:** The agent uses `<internal>...</internal>` blocks for internal reasoning. These are stripped before sending to users. Note that the current `formatOutbound()` doesn't take a `channel` parameter - this is added by the channel-formatting skill.

### Channel-Formatting Skill (Pending Merge)

The `skill/channel-formatting` branch adds `src/text-styles.ts` which provides per-channel Markdown transformation:

| Channel | Transformation |
|---------|---------------|
| WhatsApp | `**bold**` to `*bold*`, `*italic*` to `_italic_`, headings to bold, links flattened |
| Telegram | Same as WhatsApp |
| Slack | Same as WhatsApp, links become `<url\|text>` |
| Discord | Passthrough (already renders Markdown) |
| Signal | Passthrough + native `textStyle` ranges via `parseSignalStyles()` |

Code blocks (fenced and inline) are always protected - their content is never transformed.

**Files added by skill merge:**
- `src/text-styles.ts` - `parseTextStyles(text, channel)` and `parseSignalStyles(text)`
- Modified `src/router.ts` - `formatOutbound` gains optional `channel` parameter
- Modified `src/index.ts` - both outbound `sendMessage` paths pass `channel.name` to `formatOutbound`

### Integration Points

**In `src/index.ts`:**

```typescript
// Line 49 - imports
import { findChannel, formatMessages, formatOutbound } from './router.js';

// Line 260-274 - agent output callback (processGroupMessages)
const output = await runAgent(group, prompt, chatJid, async (result) => {
  if (result.result) {
    const raw = typeof result.result === 'string'
      ? result.result
      : JSON.stringify(result.result);
    const text = raw.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
    if (text) {
      await channel.sendMessage(chatJid, text);  // Direct send, no channel formatting
      outputSentToUser = true;
    }
  }
  // ...
});

// Line 654-661 - scheduler outbound
sendMessage: async (jid, rawText) => {
  const channel = findChannel(channels, jid);
  if (!channel) {
    logger.warn({ jid }, 'No channel owns JID, cannot send message');
    return;
  }
  const text = formatOutbound(rawText);  // No channel parameter in current code
  if (text) await channel.sendMessage(jid, text);
}
```

**Observation:** The current code does NOT pass `channel.name` to `formatOutbound`, meaning the per-channel Markdown transformations from the skill are not yet active.

### Clever Solutions / Technical Debt

1. **XML escaping as XSS protection:** Using XML attributes (quoted) and text nodes (escaped) prevents both attribute injection and text injection in a single pass.

2. **Channel-agnostic core:** The `router.ts` formatting functions don't know about channels. The channel-specific transformations are injected via skill merge, keeping core simple.

3. **Duplicate `<internal>` stripping:** Both `processGroupMessages` (line 269) and `formatOutbound` (via `stripInternalTags`) strip internal tags. This is intentional - `processGroupMessages` strips before displaying to user, while `formatOutbound` is the formal outbound pipeline (used by scheduler/IPC).

4. **Missing `text-styles.ts`:** The `src/text-styles.ts` file doesn't exist in this repo, indicating the channel-formatting skill branch hasn't been merged yet.

---

## Feature 14: Database Persistence (SQLite)

### Overview

**Feature:** Local data storage via SQLite
**Key File:** `src/db.ts` (~720 lines)
**Test Files:** `src/db.test.ts`, `src/db-migration.test.ts`

### Architecture

```
store/messages.db  (SQLite database)
```

Database is initialized once at startup via `initDatabase()`. Uses `better-sqlite3` for synchronous operations (no async overhead).

### Schema

```sql
-- Core tables
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
  created_at TEXT NOT NULL,
  context_mode TEXT DEFAULT 'isolated',  -- Added via migration
  script TEXT                            -- Added via migration
);

CREATE TABLE task_run_logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id TEXT NOT NULL,
  run_at TEXT NOT NULL,
  duration_ms INTEGER NOT NULL,
  status TEXT NOT NULL,
  result TEXT,
  error TEXT,
  FOREIGN KEY (task_id) REFERENCES scheduled_tasks(id)
);

CREATE TABLE router_state (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE sessions (
  group_folder TEXT PRIMARY KEY,
  session_id TEXT NOT NULL
);

CREATE TABLE registered_groups (
  jid TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  folder TEXT NOT NULL UNIQUE,
  trigger_pattern TEXT NOT NULL,
  added_at TEXT NOT NULL,
  container_config TEXT,
  requires_trigger INTEGER DEFAULT 1,
  is_main INTEGER DEFAULT 0  -- Added via migration
);
```

### Schema Migration Pattern

The codebase uses **defensive ALTER TABLE** pattern for migrations:

```typescript
// Add context_mode column if it doesn't exist (migration for existing DBs)
try {
  database.exec(
    `ALTER TABLE scheduled_tasks ADD COLUMN context_mode TEXT DEFAULT 'isolated'`,
  );
} catch {
  /* column already exists */
}
```

**Key insight:** `CREATE TABLE IF NOT EXISTS` handles new installs, `ALTER TABLE ADD COLUMN` with try/catch handles existing databases. This allows seamless upgrades without migration scripts.

### Bot Message Filtering

Bot messages are filtered using a **two-layer backstop**:

```typescript
// Layer 1: is_bot_message flag (explicit)
`is_bot_message = 0`

// Layer 2: Content prefix backstop (for pre-migration messages)
`AND content NOT LIKE ?`  -- WHERE content NOT LIKE 'Andy:%'
```

This handles:
- New messages: `is_bot_message = 0` check
- Old messages written before the migration: Content prefix check (`Andy:`)
- Backfill during migration: `UPDATE messages SET is_bot_message = 1 WHERE content LIKE 'Andy:%'`

### Message Retrieval Pattern

```typescript
// getMessagesSince - uses subquery pattern
const sql = `
  SELECT * FROM (
    SELECT id, chat_jid, sender, sender_name, content, timestamp, is_from_me
    FROM messages
    WHERE chat_jid = ? AND timestamp > ?
      AND is_bot_message = 0 AND content NOT LIKE ?
      AND content != '' AND content IS NOT NULL
    ORDER BY timestamp DESC
    LIMIT ?
  ) ORDER BY timestamp
`;
```

**Clever pattern:** Subquery fetches N most recent messages (DESC), outer query re-sorts chronologically (ASC). This gives you the latest N messages in chronological order.

### JSON State Migration

Legacy JSON files are migrated to SQLite on first run:

```typescript
function migrateJsonState(): void {
  const migrateFile = (filename: string) => {
    const filePath = path.join(DATA_DIR, filename);
    if (!fs.existsSync(filePath)) return null;
    try {
      const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
      fs.renameSync(filePath, `${filePath}.migrated`);  // Backup original
      return data;
    } catch {
      return null;
    }
  };

  // Migrates: router_state.json, sessions.json, registered_groups.json
}
```

After migration, files are renamed to `.migrated` (not deleted) for safety.

### Chat Metadata Storage

Chat metadata is stored separately from messages:

```typescript
export function storeChatMetadata(
  chatJid: string,
  timestamp: string,
  name?: string,
  channel?: string,
  isGroup?: boolean,
): void {
  // Upsert with MAX(timestamp) to preserve newer time
  db.prepare(`
    INSERT INTO chats (jid, name, last_message_time, channel, is_group) VALUES (?, ?, ?, ?, ?)
    ON CONFLICT(jid) DO UPDATE SET
      last_message_time = MAX(last_message_time, excluded.last_message_time),
      channel = COALESCE(excluded.channel, channel),
      is_group = COALESCE(excluded.is_group, is_group)
  `).run(chatJid, name || chatJid, timestamp, channel, isGroup ? 1 : 0);
}
```

**Design choice:** `name` is optional but `last_message_time` is required. If name is missing on insert, JID is used as default name.

### Key Access Patterns

| Operation | Function |
|-----------|----------|
| Store message | `storeMessage(NewMessage)`, `storeMessageDirect(...)` |
| Retrieve messages | `getMessagesSince(chatJid, since, botPrefix, limit)`, `getNewMessages(jids[], since, botPrefix, limit)` |
| Chat metadata | `storeChatMetadata(...)`, `getAllChats()`, `updateChatName(...)` |
| Tasks | `createTask(...)`, `getTaskById()`, `getAllTasks()`, `getTasksForGroup()`, `updateTask()`, `deleteTask()`, `getDueTasks()` |
| Task logs | `logTaskRun(TaskRunLog)` |
| Router state | `getRouterState(key)`, `setRouterState(key, value)` |
| Sessions | `getSession(groupFolder)`, `setSession(groupFolder, sessionId)`, `getAllSessions()` |
| Groups | `getRegisteredGroup(jid)`, `setRegisteredGroup(jid, RegisteredGroup)`, `getAllRegisteredGroups()` |

### RegisteredGroup Validation

```typescript
export function getRegisteredGroup(jid: string): (RegisteredGroup & { jid: string }) | undefined {
  const row = db.prepare('SELECT * FROM registered_groups WHERE jid = ?').get(jid) as {...} | undefined;
  if (!row) return undefined;
  if (!isValidGroupFolder(row.folder)) {
    logger.warn({ jid: row.jid, folder: row.folder }, 'Skipping registered group with invalid folder');
    return undefined;
  }
  // ...
}
```

**Security feature:** Invalid folder paths in registered_groups are skipped rather than causing errors. This prevents broken references from crashing the system.

### Test Patterns

```typescript
// In-memory database for testing
export function _initTestDatabase(): void {
  db = new Database(':memory:');
  createSchema(db);
}

// Usage in tests
beforeEach(() => {
  _initTestDatabase();
});
```

### Clever Solutions / Technical Debt

1. **ON CONFLICT DO UPDATE:** Used for upserts (chats, router_state, sessions, registered_groups) - eliminates "check then insert/update" race conditions.

2. **MAX(last_message_time):** When storing chat metadata, uses `MAX()` to preserve the newest timestamp on conflict.

3. **GROUP BY with MAX:** Message queries use subquery pattern to get recent N messages in chronological order without application-level sorting.

4. **Defensive migrations:** Every ALTER TABLE wrapped in try/catch. Gracefully handles both fresh installs (column never exists) and existing DBs (column may exist).

5. **JSON columns:** `container_config` stored as JSON string in SQLite. Retrieved and parsed with fallback.

6. **No foreign key enforcement on messages.chats:** Despite `FOREIGN KEY (chat_jid) REFERENCES chats(jid)`, the code doesn't enable foreign keys in SQLite (requires `PRAGMA foreign_keys = ON`). This is a minor technical debt.

7. **is_bot_message backfill:** The migration that adds `is_bot_message` column backfills existing bot messages by checking content prefix. This is a one-time migration that could cause issues if:
   - Bot uses different prefix format
   - User messages coincidentally contain the prefix

8. **Timestamp as string:** All timestamps stored as ISO8601 strings, not SQLite INTEGER (Unix epoch). Simpler debugging but slightly less efficient for range queries.
