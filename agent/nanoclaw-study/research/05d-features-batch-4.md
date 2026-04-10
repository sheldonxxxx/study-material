# Features 10-12 Deep Dive: Main Channel, Agent Swarms, Sender Allowlist

## Feature 10: Main Channel (Self-Chat)

### Overview
The Main Channel (also called "self-chat") is a privileged group with `isMain: true` that serves as the private admin control channel. It bypasses trigger requirements and has elevated privileges for managing other groups, scheduling tasks, and controlling the assistant without any group context.

### Key Files
- `groups/main/CLAUDE.md` - Agent instructions for the main channel
- `src/index.ts` - Orchestrator that handles `isMain` group logic throughout
- `src/ipc.ts` - IPC watcher that enforces `isMain` authorization for privileged operations
- `src/container-runner.ts` - Builds different container mounts based on `isMain`
- `src/types.ts` - `RegisteredGroup` interface with `isMain?: boolean`

### Implementation Details

**Registration:**
During setup, the main group is registered with `isMain: true`:
```typescript
// setup/register.ts:60
result.isMain = true;
```

**Trigger Bypass:**
In `src/index.ts:206-227`, the main group skips trigger checking entirely:
```typescript
const isMainGroup = group.isMain === true;

if (!isMainGroup && group.requiresTrigger !== false) {
  const triggerPattern = getTriggerPattern(group.trigger);
  const allowlistCfg = loadSenderAllowlist();
  const hasTrigger = missedMessages.some(
    (m) =>
      triggerPattern.test(m.content.trim()) &&
      (m.is_from_me || isTriggerAllowed(chatJid, m.sender, allowlistCfg)),
  );
  if (!hasTrigger) return true;
}
```
For `isMainGroup`, the trigger check is skipped and all messages are processed.

**CLAUDE.md Template Selection:**
When a new group is registered, the system copies a CLAUDE.md template. Main groups get the `main` template while others get `global`:
```typescript
// src/index.ts:144
const templateFile = path.join(
  GROUPS_DIR,
  group.isMain ? 'main' : 'global',
  'CLAUDE.md',
);
```

**Container Mount Differences:**
Main group gets read-only access to the project and read-write to its group folder:
```typescript
// src/container-runner.ts:69
if (isMain) {
  // Main has more permissive mounts
}
```

**IPC Privileges:**
The `isMain` flag gates several privileged operations in `src/ipc.ts`:

1. **Task management across groups**: Main can pause/resume/cancel any task; non-main can only touch their own (lines 283, 302, 321)
2. **Scheduling across groups**: Main can `schedule_task` for any group; non-main can only schedule for themselves (line 206)
3. **Group registration**: Only main can call `register_group` IPC (line 429)
4. **Group refresh**: Only main can trigger `refresh_groups` (line 405)

**Defense in Depth:**
In `src/ipc.ts:444`, the agent cannot set `isMain` via IPC even if it tries:
```typescript
// Defense in depth: agent cannot set isMain via IPC
deps.registerGroup(data.jid, {
  // ... other fields
  // isMain is deliberately NOT set from IPC data
});
```

**OneCLI Agent Bypass:**
Main groups skip OneCLI agent creation:
```typescript
// src/index.ts:80
if (group.isMain) return;
```

### Clever Solution: Self-Message Context Accumulation
Non-trigger messages in regular groups accumulate in the database for context when a trigger arrives. However, main group processes every message immediately without trigger filtering.

### Edge Cases
- If the main group's CLAUDE.md doesn't exist at registration time, it's created from template
- The main group is identified by `isMain: true` in the registered_groups table, not by folder name
- Recovery on startup (`recoverPendingMessages`) includes main group

---

## Feature 11: Agent Swarms

### Overview
Agent Swarms enable a team of specialized agents to collaborate on complex multi-step tasks. The main agent can spawn sub-agents that each have their own identity and can send messages back to the group chat. Currently implemented specifically for Telegram via a bot pool.

### Key Files
- `.claude/skills/add-telegram-swarm/SKILL.md` - Full implementation guide
- `container/agent-runner/src/ipc-mcp-stdio.ts` - MCP tool that supports `sender` parameter
- `src/ipc.ts` - IPC message routing with sender support

### Implementation Details

**Core Concept:**
The swarm works by:
1. Main agent spawns sub-agents (teammates) for parallel work
2. Each sub-agent uses `mcp__nanoclaw__send_message` with a `sender` parameter
3. IPC watcher routes these messages through a Telegram bot pool
4. Each bot in the pool gets renamed to match its role

**MCP Tool Interface:**
In `container/agent-runner/src/ipc-mcp-stdio.ts:42-63`:
```typescript
server.tool(
  'send_message',
  "Send a message to the user or group immediately while you're still running...",
  {
    text: z.string(),
    sender: z.string().optional().describe('Your role/identity name (e.g. "Researcher"). When set, messages appear from a dedicated bot in Telegram.'),
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
    return { content: [{ type: 'text' as const, text: 'Message sent.' }] };
  },
);
```

**Bot Pool Architecture:**
The swarm uses a pool of Telegram bots. When a sub-agent sends a message with `sender: "Marine Biologist"`, the system:
1. Looks up the sender name in the pool's `senderBotMap`
2. If not mapped, assigns an unused bot and renames it via `setMyName`
3. Sends the message from that bot

From the skill file:
```
- Pool bots use Grammy's `Api` class — lightweight, no polling, just send
- Bot names are set via `setMyName` — changes are global to the bot, not per-chat
- A 2-second delay after `setMyName` allows Telegram to propagate the name change before the first message
- Sender→bot mapping is stable within a group (keyed as `{groupFolder}:{senderName}`)
```

**IPC Routing Update:**
In `src/ipc.ts`, the message handler routes Telegram messages with sender through the bot pool:
```typescript
if (data.sender && data.chatJid.startsWith('tg:')) {
  await sendPoolMessage(
    data.chatJid,
    data.text,
    data.sender,
    sourceGroup,
  );
} else {
  await deps.sendMessage(data.chatJid, data.text);
}
```

**Team Instructions in CLAUDE.md:**
The main group's CLAUDE.md instructs the lead agent on swarm behavior:
```markdown
### Lead agent behavior
As the lead agent who created the team:
- You do NOT need to react to or relay every teammate message. The user sees those directly from the teammate bots.
- Send your own messages only to comment, share thoughts, synthesize, or direct the team.
- When processing an internal update from a teammate that doesn't need a user-facing response, wrap your *entire* output in `<internal>` tags.
```

### Edge Cases
- Pool runs out of bots: round-robin reuse
- Bot name caching: Telegram caches names client-side, users may need to restart to see updates
- Pool bots must be members of the Telegram group with privacy disabled
- Service restart resets sender-to-bot mapping

---

## Feature 12: Sender Allowlist

### Overview
Security mechanism to control which senders can trigger the assistant in each group. Two modes: "trigger" mode (store all, trigger only allowed) and "drop" mode (discard denied messages before storage).

### Key Files
- `src/sender-allowlist.ts` - Core allowlist logic
- `src/index.ts` - Integrates allowlist into message processing
- `src/config.ts` - `SENDER_ALLOWLIST_PATH` configuration

### Implementation Details

**Configuration Schema:**
```typescript
export interface ChatAllowlistEntry {
  allow: '*' | string[];
  mode: 'trigger' | 'drop';
}

export interface SenderAllowlistConfig {
  default: ChatAllowlistEntry;
  chats: Record<string, ChatAllowlistEntry>;
  logDenied: boolean;
}
```

**Config File Location:**
Stored at `~/.config/nanoclaw/sender-allowlist.json` (outside project root, never mounted into containers).

**Two-Mode Design:**

1. **trigger mode** (default): All messages are stored, but only allowed senders can trigger the assistant
   ```typescript
   // In index.ts:218-227, during message processing
   if (!isMainGroup && group.requiresTrigger !== false) {
     const hasTrigger = missedMessages.some(
       (m) =>
         triggerPattern.test(m.content.trim()) &&
         (m.is_from_me || isTriggerAllowed(chatJid, m.sender, allowlistCfg)),
     );
     if (!hasTrigger) return true;
   }
   ```

2. **drop mode**: Messages from denied senders are discarded before storage
   ```typescript
   // In index.ts:599-613, before storeMessage
   if (!msg.is_from_me && !msg.is_bot_message && registeredGroups[chatJid]) {
     const cfg = loadSenderAllowlist();
     if (
       shouldDropMessage(chatJid, cfg) &&
       !isSenderAllowed(chatJid, msg.sender, cfg)
     ) {
       if (cfg.logDenied) {
         logger.debug(
           { chatJid, sender: msg.sender },
           'sender-allowlist: dropping message (drop mode)',
         );
       }
       return;
     }
   }
   storeMessage(msg);
   ```

**Bypass for Own Messages:**
`is_from_me` messages bypass the allowlist check explicitly. This allows the assistant's own messages to trigger responses regardless of allowlist.

**Fail-Open Design:**
If the config file is missing or invalid, all senders are allowed:
```typescript
// src/sender-allowlist.ts:39-48
try {
  raw = fs.readFileSync(filePath, 'utf-8');
} catch (err: unknown) {
  if ((err as NodeJS.ErrnoException).code === 'ENOENT') return DEFAULT_CONFIG;
  // ...
}
```

**Validation:**
Entries are validated before use:
```typescript
function isValidEntry(entry: unknown): entry is ChatAllowlistEntry {
  if (!entry || typeof entry !== 'object') return false;
  const e = entry as Record<string, unknown>;
  const validAllow =
    e.allow === '*' ||
    (Array.isArray(e.allow) && e.allow.every((v) => typeof v === 'string'));
  const validMode = e.mode === 'trigger' || e.mode === 'drop';
  return validAllow && validMode;
}
```

### Configuration Example
```json
{
  "default": { "allow": "*", "mode": "trigger" },
  "chats": {
    "120363336345536173@g.us": {
      "allow": ["sender-id-1", "sender-id-2"],
      "mode": "trigger"
    }
  },
  "logDenied": true
}
```

### Integration Points

1. **Message arrival** (`src/index.ts:599-613`): Drop mode check before storing
2. **Trigger evaluation** (`src/index.ts:221-225`): Trigger mode check when evaluating if messages should be processed
3. **IPC authorization** (`src/ipc.ts`): Uses `isMain` flag, not allowlist

### Edge Cases
- Bot messages (`is_bot_message`) are filtered out by the database query before allowlist evaluation
- Invalid chat entries in config are skipped with a warning
- `logDenied: false` suppresses debug logs for dropped messages
- Per-chat config overrides default, but missing chats fall back to default

---

## Cross-Feature Interactions

### Main Channel + Sender Allowlist
The main channel (`isMain: true`) does NOT bypass the sender allowlist in the same way it bypasses triggers. However, since main group processes all messages without trigger checking, the allowlist trigger check is effectively moot for main.

### Main Channel + Agent Swarms
The main group serves as the "lead agent" that spawns sub-agents. The swarm skill specifically instructs the main agent on team coordination.

### Agent Swarms + Sender Allowlist
Sub-agents using `send_message` with a `sender` parameter route through the Telegram bot pool. The sender identity is preserved in IPC for routing but the allowlist is evaluated against the actual Telegram user who triggered the lead agent.

---

## Technical Debt and Observations

1. **isMain as boolean vs undefined**: The `RegisteredGroup.isMain` is `boolean | undefined`, where undefined means "not main". This creates subtle `=== true` checks throughout the codebase.

2. **No swarm persistence**: The sender-to-bot mapping for agent swarms resets on service restart. From the skill: "Mapping resets on service restart — pool bots get reassigned fresh."

3. **Bot name global state**: Using `setMyName` changes the bot's display name globally across all chats, not per-chat. This is a Telegram API limitation.

4. **Allowlist file watching**: The allowlist is loaded fresh on each message, not watched. Changes require restart.

5. **IPC authorization simplicity**: The `isMain` flag is passed as a verified boolean derived from directory path, not user-supplied data. This is architecturally clean.

6. **Drop mode silent failure**: When in drop mode, denied messages are simply not stored with no notification to the user.
