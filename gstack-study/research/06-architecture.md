# gstack Architecture

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Type:** Node.js/Bun monorepo with skill-based CLI tools
**Runtime:** Bun >= 1.0.0, TypeScript
**Primary Purpose:** AI-assisted software development toolkit (browser automation, code review, planning, QA)

---

## Architectural Pattern: Daemon + Command Dispatch

gstack uses a **daemon-based client-server architecture** for browser automation, combined with a **skill-based command system** for AI agent workflows.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code                               │
│                   (or other AI agent)                            │
└─────────────────────────┬───────────────────────────────────────┘
                          │ Tool calls ($B snapshot -i, /review, /qa)
┌─────────────────────────▼───────────────────────────────────────┐
│                     gstack CLI (browse)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐   │
│  │   CLI.ts    │  │  Skill     │  │  Binary discovery        │   │
│  │  (thin)     │  │  runners   │  │  + config management     │   │
│  └──────┬──────┘  └─────┬───────┘  └─────────────────────────┘   │
└─────────┼───────────────┼────────────────────────────────────────┘
          │ state file    │ HTTP POST
          │ (.gstack/     │
          │  browse.json)  │
┌─────────▼───────────────▼────────────────────────────────────────┐
│                  Bun.serve HTTP Server                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐   │
│  │  /command   │  │  /health   │  │  /cookie-picker         │   │
│  │  dispatcher │  │  endpoint  │  │  (localhost-only)        │   │
│  └──────┬──────┘  └─────────────┘  └─────────────────────────┘   │
└─────────┼────────────────────────────────────────────────────────┘
          │ CDP (Chrome DevTools Protocol)
┌─────────▼────────────────────────────────────────────────────────┐
│                    Chromium (headless)                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐   │
│  │  Persistent │  │  Cookie    │  │  Tab management         │   │
│  │  context    │  │  storage   │  │  (multi-tab support)     │   │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Insight

An AI agent interacting with a browser needs **sub-second latency** and **persistent state**. If every command cold-starts a browser, you wait 3-5 seconds per tool call. If the browser dies between commands, you lose cookies, tabs, and login sessions.

---

## Core Subsystems

### 1. Browser Automation (browse/)

**Files:**
- `browse/src/server.ts` - HTTP daemon (Bun.serve)
- `browse/src/cli.ts` - CLI entry point (thin wrapper)
- `browse/src/browser-manager.ts` - Chromium lifecycle management
- `browse/src/commands.ts` - Command registry (single source of truth)
- `browse/src/read-commands.ts` - READ command handlers
- `browse/src/write-commands.ts` - WRITE command handlers
- `browse/src/meta-commands.ts` - META command handlers
- `browse/src/snapshot.ts` - Accessibility tree + ref system
- `browse/src/buffers.ts` - CircularBuffer implementation

**Command Dispatch Architecture:**

```typescript
// commands.ts - Single source of truth
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
```

**Server dispatch (server.ts):**
```typescript
if (READ_COMMANDS.has(command)) {
  result = await handleReadCommand(command, args, browserManager);
} else if (WRITE_COMMANDS.has(command)) {
  result = await handleWriteCommand(command, args, browserManager);
} else if (META_COMMANDS.has(command)) {
  result = await handleMetaCommand(command, args, browserManager, shutdown);
}
```

### 2. The Ref System

Refs (`@e1`, `@e2`, `@c1`) are how the agent addresses page elements without writing CSS selectors or XPath.

**How it works:**
1. Agent runs: `$B snapshot -i`
2. Server calls Playwright's `page.accessibility.snapshot()`
3. Parser walks the ARIA tree, assigns sequential refs: `@e1`, `@e2`, `@e3`...
4. For each ref, builds a Playwright Locator: `getByRole(role, { name }).nth(index)`
5. Stores `Map<string, RefEntry>` on the BrowserManager instance
6. Returns the annotated tree as plain text

**Later:**
7. Agent runs: `$B click @e3`
8. Server resolves `@e3` -> Locator -> `locator.click()`

**Why Locators, not DOM mutation:**
- CSP (Content Security Policy) blocks DOM modification
- React/Vue/Svelte hydration strips injected attributes
- Shadow DOM cannot be reached from outside

Playwright Locators are external to the DOM, using the accessibility tree and `getByRole()` queries.

**Staleness detection:**
```typescript
resolveRef(@e3) -> count = await entry.locator.count()
             -> if count === 0: throw "Ref @e3 is stale — run 'snapshot' to get fresh refs"
```

### 3. CircularBuffer Logging

Three ring buffers (50,000 entries each, O(1) push):

```
Browser events -> CircularBuffer (in-memory) -> Async flush to .gstack/*.log (every 1s)
```

```typescript
export class CircularBuffer<T> {
  private buffer: (T | undefined)[];
  private head: number = 0;
  private _size: number = 0;
  private _totalAdded: number = 0;  // Increments past capacity (flush cursor)

  push(entry: T): void { /* O(1) */ }
  last(n: number): T[] { /* Most recent N entries */ }
  toArray(): T[] { /* All entries in insertion order */ }
}
```

**Key properties:**
- HTTP request handling never blocked by disk I/O
- Logs survive server crashes (max 1 second data loss)
- Memory bounded (50K entries x 3 buffers)
- Disk files append-only, readable by external tools

### 4. State Management

**State file** (`.gstack/browse.json`):
```json
{
  "pid": 12345,
  "port": 34567,
  "token": "uuid-v4",
  "startedAt": "2026-03-27T10:00:00Z",
  "binaryVersion": "abc123"
}
```

**Lifecycle:**
- Atomic write via tmp + rename (mode 0o600)
- CLI reads state file to find server
- Health check (`GET /health`) determines if server is alive
- Version mismatch triggers auto-restart (binary updated -> server restarts)

### 5. Security Model

| Layer | Protection |
|-------|------------|
| Network | HTTP server binds to `localhost` only (not `0.0.0.0`) |
| Auth | Bearer token (random UUID) in every request |
| Cookies | Keychain access requires user approval; decryption in-process |
| Database | Read-only copy of Chromium cookie DB (no lock conflicts) |
| Logs | No cookie values in console/network/dialog logs |

---

## Skill System

### Architecture

Each skill directory contains a `SKILL.md` that defines:
- The slash command (e.g., `/review`, `/qa`, `/ship`)
- Instructions for Claude Code
- When to invoke the skill

```
.agents/skills/           # Skill definitions (SKILL.md files)
├── gstack/               # Main skill
├── gstack-browse/        # Browser automation
├── gstack-review/
├── gstack-qa/
├── gstack-ship/
└── ...

[mode directories]        # Each is a skill implementation
├── review/
├── qa/
├── ship/
└── ...
```

### Template System

```
SKILL.md.tmpl          (human-written prose + placeholders)
       |
       v
gen-skill-docs.ts      (reads source code metadata)
       |
       v
SKILL.md               (committed, auto-generated sections)
```

**Placeholders filled at build time:**

| Placeholder | Source | Generates |
|-------------|--------|-----------|
| `{{COMMAND_REFERENCE}}` | `commands.ts` | Categorized command table |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` | Flag reference with examples |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` | Update check, session tracking, contributor mode |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` | Binary discovery + setup |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` | Dynamic base branch detection |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` | Shared QA methodology |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` | Design audit methodology |

**Why committed, not generated at runtime?**
1. Claude reads SKILL.md at skill load time - no build step
2. CI can validate freshness: `gen:skill-docs --dry-run` + `git diff --exit-code`
3. Git blame works

---

## Key Design Patterns

### 1. Daemon Pattern

**Why daemon over per-command browser:**
- Persistent state (cookies, localStorage, tabs)
- Sub-second commands (~100-200ms vs 3-5s cold start)
- Auto-lifecycle (starts on first use, shuts down after 30min idle)

**Server crash handling:**
- Chromium crash -> server exits immediately
- CLI detects dead server -> auto-restarts on next command
- No self-heal attempt (simpler, more reliable)

### 2. Command Categorization

| Category | Behavior | Idempotent? |
|----------|----------|-------------|
| READ | Returns page state | Yes |
| WRITE | Mutates page state | No |
| META | Server operations | Varies |

### 3. Error Philosophy

Errors are for AI agents, not humans. Every error is actionable:

- "Element not found" -> "Run `snapshot -i` to see available elements"
- "Timeout" -> "Page may be slow or URL wrong"
- "Multiple elements" -> "Use @refs from `snapshot` instead"

### 4. Lockfile Pattern

TOCTOU (time-of-check-time-of-use) race prevention:
```typescript
const fd = fs.openSync(lockPath, fs.constants.O_CREAT | fs.constants.O_EXCL | fs.constants.O_WRONLY);
// Atomic: fails if file already exists
fs.writeSync(fd, `${process.pid}\n`);
```

---

## Module Map

### Primary Modules

| Module | Responsibility | Key Files |
|--------|---------------|-----------|
| `browse/` | Headless browser CLI | `cli.ts`, `server.ts`, `browser-manager.ts` |
| `scripts/` | Build + DX tooling | `gen-skill-docs.ts`, `skill-check.ts` |
| `test/` | Skill validation + evals | `session-runner.ts`, `eval-store.ts`, `llm-judge.ts` |
| `lib/` | Shared utilities | `worktree.ts` |
| `supabase/` | Backend functions | Edge functions, migrations |

### Skill Directories

Each skill is a self-contained unit with:
- `SKILL.md` - Generated instructions for Claude Code
- `SKILL.md.tmpl` - Template source
- Supporting docs (checklists, formats)

---

## Technology Choices

### Why Bun?

1. **Compiled binaries** - `bun build --compile` produces single ~58MB executable
2. **Native SQLite** - Cookie decryption reads Chromium's SQLite directly
3. **Native TypeScript** - No compilation step in development
4. **Built-in HTTP** - `Bun.serve()` is fast and simple (10 routes total)

### Why Playwright?

- Cross-browser support (Chromium, Firefox, WebKit)
- Built-in CDP support
- Locator abstraction (ref system relies on this)
- Mature, well-maintained

---

## Data Flow: Command Execution

```
User: $B click @e3
  |
  v
cli.ts:
  1. Read .gstack/browse.json (port + token)
  2. POST {command: "click", args: ["@e3"]} to localhost:PORT
  3. Print response to stdout
  |
  v
server.ts:
  1. validateAuth() - check Bearer token
  2. handleCommand() - dispatch by category
  |
  v
write-commands.ts:
  resolveRef("@e3") -> BrowserManager.refMap.get("e3")
  locator.click()
  |
  v
Response: "Clicked @e3"
```

---

## Observability Infrastructure

```
test/skill-e2e-*.test.ts
        |
        | generates runId, passes testName + runId
        |
  ┌─────┼──────────────────────┐
  │     │                      │
  │  runSkillTest()      evalCollector
  │  (session-runner.ts) (eval-store.ts)
  │     │                      │
  │  per tool call:      per addTest():
  │  ┌──┼──┐             savePartial()
  │  │  │  │                  │
  ▼  ▼  ▼  ▼                  ▼
 [HB] [PL] [NJ]       _partial-e2e.json
  │    │    │          (atomic overwrite)
  ▼    ▼    ▼
 e2e-  prog-  {name}
 live  ress   .ndjson
 .json .log

  ALL files in ~/.gstack-dev/
```

---

## What's Intentionally Not Here

- **No WebSocket streaming** - HTTP request/response is simpler, debuggable with curl
- **No MCP protocol** - Plain HTTP + plain text is lighter on tokens
- **No multi-user support** - One server per workspace
- **No iframe support** - Ref system doesn't cross frame boundaries
- **No Windows/Linux cookie decryption** - macOS Keychain only

---

## Directory Structure

```
gstack/
├── .agents/skills/           # Skill definitions
├── agents/                   # Agent configuration
├── bin/                      # Compiled utilities
├── browse/                   # Browser automation (main deliverable)
│   ├── src/
│   │   ├── cli.ts           # CLI entry point
│   │   ├── server.ts        # HTTP daemon
│   │   ├── browser-manager.ts
│   │   ├── commands.ts      # Command registry
│   │   ├── read-commands.ts
│   │   ├── write-commands.ts
│   │   ├── meta-commands.ts
│   │   ├── snapshot.ts      # Ref system
│   │   ├── buffers.ts      # CircularBuffer
│   │   └── config.ts
│   ├── test/
│   └── dist/                 # Compiled binary
├── lib/                      # Shared utilities
├── scripts/                  # Build + DX tooling
│   ├── gen-skill-docs.ts    # Template generator
│   ├── skill-check.ts
│   └── resolvers/           # Placeholder resolvers
├── supabase/                 # Backend
├── test/                    # Skill validation + evals
│   ├── helpers/             # Test infrastructure
│   └── fixtures/
├── [skill directories]      # review/, qa/, ship/, etc.
├── CLAUDE.md                # Root instructions
├── SKILL.md                 # Main skill (generated)
└── SKILL.md.tmpl            # Main skill template
```
