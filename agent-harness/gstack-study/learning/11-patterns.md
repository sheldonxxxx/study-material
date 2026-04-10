# gstack Design Patterns

## Pattern Index

1. [Daemon Pattern](#1-daemon-pattern)
2. [Command Dispatch Pattern](#2-command-dispatch-pattern)
3. [Ref System Pattern](#3-ref-system-pattern)
4. [CircularBuffer Pattern](#4-circularbuffer-pattern)
5. [State File Pattern](#5-state-file-pattern)
6. [Template Generation Pattern](#6-template-generation-pattern)
7. [Lockfile Pattern](#7-lockfile-pattern)
8. [Error Messaging Pattern](#8-error-messaging-pattern)
9. [Skill Discovery Pattern](#9-skill-discovery-pattern)
10. [Vertical Slice Monorepo](#10-vertical-slice-monorepo)

---

## 1. Daemon Pattern

**Problem:** Per-command browser spawning has 3-5s cold start latency, loses state between commands.

**Solution:** Long-running HTTP server that maintains Chromium instance, cookies, tabs.

**Implementation:**

```typescript
// browse/src/server.ts
const server = Bun.serve({
  port: PORT,
  async fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/command") {
      return handleCommand(req, browserManager);
    }
    // ...
  }
});
```

**Evidence:** `browse/src/server.ts`, `browse/src/browser-manager.ts`

**Trade-off:**
- Pro: Sub-second commands (~100-200ms)
- Pro: Persistent cookies, localStorage, tabs
- Con: Lifecycle management complexity
- Con: Server crash requires restart handling

---

## 2. Command Dispatch Pattern

**Problem:** Need to route commands to appropriate handlers based on category (READ/WRITE/META).

**Solution:** Command categorization with dispatch based on Set membership.

**Implementation:**

```typescript
// browse/src/commands.ts
export const READ_COMMANDS = new Set([
  'text', 'html', 'links', 'forms', 'accessibility',
  'js', 'eval', 'css', 'attrs',
  'console', 'network', 'cookies', 'storage', 'perf',
  'dialog', 'is',
]);

export const WRITE_COMMANDS = new Set([
  'goto', 'back', 'forward', 'reload',
  'click', 'fill', 'select', 'hover', 'type', 'press', 'scroll', 'wait',
  'viewport', 'cookie', 'cookie-import', 'cookie-import-browser', 'header', 'useragent',
  'upload', 'dialog-accept', 'dialog-dismiss',
]);

export const META_COMMANDS = new Set([
  'tabs', 'tab', 'newtab', 'closetab',
  'status', 'stop', 'restart',
  'screenshot', 'pdf', 'responsive',
  'chain', 'diff',
  'url', 'snapshot',
  'handoff', 'resume',
]);

// browse/src/server.ts dispatch:
if (READ_COMMANDS.has(command)) {
  result = await handleReadCommand(command, args, browserManager);
} else if (WRITE_COMMANDS.has(command)) {
  result = await handleWriteCommand(command, args, browserManager);
} else if (META_COMMANDS.has(command)) {
  result = await handleMetaCommand(command, args, browserManager, shutdown);
}
```

**Evidence:** `browse/src/commands.ts`, `browse/src/server.ts` lines ~60-80

---

## 3. Ref System Pattern

**Problem:** AI agent needs to reference page elements without brittle CSS selectors or XPath.

**Solution:** Assign sequential refs (`@e1`, `@e2`) from accessibility tree, resolve to Playwright Locators.

**Implementation:**

```typescript
// browse/src/snapshot.ts - assignment
const refMap = new Map<string, RefEntry>();
let refIndex = 0;

function walkTree(node: AccessibilityNode): void {
  if (node.role && node.name) {
    const ref = `@e${++refIndex}`;
    const locator = page.getByRole(node.role, { name: node.name }).nth(count);
    refMap.set(ref, { locator, role: node.role, name: node.name });
  }
  node.children?.forEach(walkTree);
}

// browse/src/write-commands.ts - resolution
async function resolveRef(ref: string, browserManager: BrowserManager) {
  const entry = browserManager.refMap.get(ref);
  if (!entry) throw new Error(`Unknown ref: ${ref}`);
  const count = await entry.locator.count();
  if (count === 0) throw new Error(`Ref ${ref} is stale — run 'snapshot' to get fresh refs`);
  return entry.locator;
}
```

**Evidence:** `browse/src/snapshot.ts`, `browse/src/write-commands.ts`

**Why Locators:**
- CSP blocks DOM mutation
- React/Vue/Svelte hydration strips injected attributes
- Shadow DOM accessible from outside
- Staleness detectable via `locator.count()`

---

## 4. CircularBuffer Pattern

**Problem:** Need bounded logging that doesn't block request handling, survives crashes.

**Solution:** In-memory ring buffer with async flush to disk.

**Implementation:**

```typescript
// browse/src/buffers.ts
export class CircularBuffer<T> {
  private buffer: (T | undefined)[];
  private head: number = 0;
  private _size: number = 0;
  private _totalAdded: number = 0;

  push(entry: T): void {
    this.buffer[this.head] = entry;
    this.head = (this.head + 1) % this.buffer.length;
    if (this._size < this.buffer.length) this._size++;
    this._totalAdded++;
  }

  last(n: number): T[] { /* most recent N entries */ }
  toArray(): T[] { /* all entries in insertion order */ }
}

// Usage: 3 buffers (50K entries each), flush every 1s
```

**Evidence:** `browse/src/buffers.ts`

**Properties:**
- O(1) push
- Memory bounded (capacity x item size)
- Async flush never blocks HTTP
- Append-only disk files survive crashes (max 1s data loss)

---

## 5. State File Pattern

**Problem:** CLI and server run in same process tree but separate processes; need coordination.

**Solution:** JSON state file with atomic write (tmp + rename).

**Implementation:**

```typescript
// browse/src/server.ts - server writes state
const state = {
  pid: process.pid,
  port: PORT,
  token: crypto.randomUUID(),
  startedAt: new Date().toISOZString(),
  binaryVersion: VERSION,
};

// Atomic write
const tmp = `${STATE_DIR}/browse.json.tmp`;
fs.writeFileSync(tmp, JSON.stringify(state), { mode: 0o600 });
fs.renameSync(tmp, STATE_PATH);

// browse/src/cli.ts - CLI reads state
const state = JSON.parse(fs.readFileSync(STATE_PATH, 'utf-8'));
const alive = await fetch(`http://localhost:${state.port}/health`);
```

**Evidence:** `browse/src/server.ts`, `browse/src/cli.ts`

**Key properties:**
- Atomic write (O_EXCL + rename prevents TOCTOU)
- Mode 0o600 (owner read/write only)
- Version mismatch triggers restart

---

## 6. Template Generation Pattern

**Problem:** Skill documentation needs to reflect code reality but should be loadable without a build step.

**Solution:** Template with placeholders, generated at build time, committed to git.

**Implementation:**

```typescript
// SKILL.md.tmpl
# Browse Skill
...
{{COMMAND_REFERENCE}}
{{SNAPSHOT_FLAGS}}

// scripts/gen-skill-docs.ts
const template = fs.readFileSync('SKILL.md.tmpl', 'utf-8');
const output = template
  .replace('{{COMMAND_REFERENCE}}', generateCommandTable())
  .replace('{{SNAPSHOT_FLAGS}}', generateSnapshotFlags());
fs.writeFileSync('SKILL.md', output);
```

**Evidence:** `scripts/gen-skill-docs.ts`, `SKILL.md.tmpl`

**Placeholders:**
| Placeholder | Source |
|-------------|--------|
| `{{COMMAND_REFERENCE}}` | `commands.ts` |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` |

**Why committed:**
1. Claude Code loads SKILL.md at skill load time - no build step
2. CI can validate freshness: `gen:skill-docs --dry-run` + `git diff --exit-code`
3. Git blame works

---

## 7. Lockfile Pattern

**Problem:** Prevent TOCTOU race when multiple processes try to claim the same resource.

**Solution:** `O_CREAT | O_EXCL` open syscall - atomic failure if file exists.

**Implementation:**

```typescript
// browse/src/server.ts
const fd = fs.openSync(lockPath,
  fs.constants.O_CREAT | fs.constants.O_EXCL | fs.constants.O_WRONLY,
  0o600);
// Atomic: fails if file already exists
fs.writeSync(fd, `${process.pid}\n`);
```

**Evidence:** `browse/src/server.ts` (lifecycle startup)

---

## 8. Error Messaging Pattern

**Problem:** Errors for AI agents must be actionable, not just technical.

**Solution:** Every error includes suggested recovery action.

**Implementation:**

```typescript
// Error messages follow pattern:
// <what failed> -> <how to fix>

"Element not found" -> "Run `snapshot -i` to see available elements"
"Timeout" -> "Page may be slow or URL wrong"
"Multiple elements" -> "Use @refs from `snapshot` instead"
"Ref @e3 is stale" -> "Run `snapshot` to get fresh refs"
```

**Evidence:** `browse/src/write-commands.ts` (resolveRef), error messages throughout

---

## 9. Skill Discovery Pattern

**Problem:** Claude Code needs to discover and load skill instructions dynamically.

**Solution:** File-based discovery from `.agents/skills/` directory.

**Implementation:**

```
.agents/skills/
├── gstack/
├── gstack-review/
├── gstack-qa/
└── gstack-ship/

Each contains: SKILL.md (generated from SKILL.md.tmpl)
```

Claude Code automatically discovers skills from this directory and loads their `SKILL.md` when the corresponding slash command is invoked.

**Evidence:** `.agents/skills/`, `SKILL.md`, `SKILL.md.tmpl`

---

## 10. Vertical Slice Monorepo

**Problem:** Related tools (browser automation, code review, QA) need shared utilities but should be independently deployable.

**Solution:** Each skill is a vertical slice (self-contained unit with its own docs + implementation), with shared `lib/` utilities.

**Structure:**

```
gstack/
├── browse/          # Vertical slice: browser automation
│   ├── SKILL.md
│   └── src/
├── review/          # Vertical slice: code review
│   ├── SKILL.md
│   └── ...
├── qa/              # Vertical slice: QA
│   ├── SKILL.md
│   └── ...
├── lib/             # Shared utilities across slices
│   ├── gstack-config
│   └── gstack-diff-scope
└── scripts/         # Shared build tooling
```

**Evidence:** `browse/`, `review/`, `qa/`, `ship/` directories

---

## Pattern Relationships

```
Daemon Pattern
    └── Command Dispatch Pattern
            ├── Ref System Pattern
            └── CircularBuffer Pattern

State File Pattern
    └── Lockfile Pattern (used during server startup)

Template Generation Pattern
    └── Skill Discovery Pattern

Vertical Slice Monorepo
    └── Skill Discovery Pattern
```

---

## Summary Table

| Pattern | Problem Solved | Key Files |
|---------|-----------------|-----------|
| Daemon | Cold start latency, state loss | `server.ts`, `browser-manager.ts` |
| Command Dispatch | Command routing | `commands.ts`, `server.ts` |
| Ref System | Element addressing without selectors | `snapshot.ts`, `write-commands.ts` |
| CircularBuffer | Bounded logging, crash survival | `buffers.ts` |
| State File | CLI-server coordination | `server.ts`, `cli.ts` |
| Template Generation | Documentation reflects code | `gen-skill-docs.ts`, `SKILL.md.tmpl` |
| Lockfile | TOCTOU race prevention | `server.ts` |
| Error Messaging | Actionable AI errors | Throughout commands |
| Skill Discovery | Claude Code skill loading | `.agents/skills/` |
| Vertical Slice | Independent deployability | `browse/`, `review/`, `qa/` |
