# gstack Architecture Analysis

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Type:** Node.js/Bun monorepo with skill-based CLI tools
**Runtime:** Bun >= 1.0.0, TypeScript
**Primary Domain:** AI-assisted software development toolkit

---

## Architectural Decisions

### AD-01: Daemon-Based Browser Automation

**Decision:** Use a long-running HTTP server (daemon) instead of per-command browser spawning.

**Rationale:**
- Sub-second latency: ~100-200ms vs 3-5s cold start
- Persistent state: cookies, localStorage, login sessions survive across commands
- Tab management: multiple tabs with state

**Trade-offs:**
- Server must manage lifecycle (start, health check, restart on crash)
- State file coordination required (`.gstack/browse.json`)
- Version mismatch handling when binary updates

**Evidence:** `browse/src/server.ts`, `browse/src/cli.ts`

---

### AD-02: Plain HTTP Over WebSocket/MCP

**Decision:** Use simple HTTP request/response instead of WebSocket streaming or MCP protocol.

**Rationale:**
- Debugging: `curl` works directly
- Simplicity: no connection state management
- Token efficiency: no overhead for streaming metadata
- Claude Code tool calling naturally fits request/response

**Trade-offs:**
- No real-time streaming back to agent
- Each command is a separate HTTP request

**Evidence:** `browse/src/server.ts` uses `Bun.serve()` with 10 routes

---

### AD-03: Locator-Based Element Addressing

**Decision:** Address page elements via Playwright Locators (`@e1`, `@e2`) rather than DOM mutation or CSS injection.

**Rationale:**
- CSP-compliant: Content Security Policy blocks DOM modification
- Framework-safe: React/Vue/Svelte hydration strips injected attributes
- Shadow DOM accessible: locators query from outside
- Staleness detection built into locator.count()

**Evidence:** `browse/src/snapshot.ts`, `browse/src/write-commands.ts`

---

### AD-04: Template-Generated Skill Documentation

**Decision:** Generate `SKILL.md` files from templates at build time, commit the output.

**Rationale:**
- No build step when Claude Code loads skill
- CI can validate freshness with `git diff --exit-code`
- Git blame works for tracking changes
- Source of truth is code metadata, not runtime

**Trade-offs:**
- Must run generator before committing
- Template and generated file can diverge

**Evidence:** `scripts/gen-skill-docs.ts`, `SKILL.md.tmpl`

---

### AD-05: Committed over Runtime Generation

**Decision:** All generated files (SKILL.md) are committed to git rather than generated at runtime.

**Rationale:**
- Claude Code loads skills from filesystem - no server required
- Deterministic: same input produces same output
- Reviewable in PRs

---

## Module Responsibilities

### browse/ — Headless Browser CLI

**Core deliverable:** `$B` CLI command for AI agents to control a headless browser.

| File | Responsibility |
|------|-----------------|
| `src/cli.ts` | Thin entry point; reads state file, POSTs to server |
| `src/server.ts` | HTTP daemon; route dispatch, auth, lifecycle |
| `src/browser-manager.ts` | Chromium lifecycle, tab management, cookie storage |
| `src/commands.ts` | Command registry (READ/WRITE/META sets) |
| `src/read-commands.ts` | READ command handlers |
| `src/write-commands.ts` | WRITE command handlers |
| `src/meta-commands.ts` | META command handlers |
| `src/snapshot.ts` | Accessibility tree parsing, ref assignment |
| `src/buffers.ts` | CircularBuffer for bounded logging |
| `src/cookie-*.ts` | Keychain integration, cookie import |

**Public API:** HTTP endpoints on localhost (port from `.gstack/browse.json`)

---

### scripts/ — Build and Developer Experience

| File | Responsibility |
|------|-----------------|
| `gen-skill-docs.ts` | Template processor; fills placeholders from code metadata |
| `skill-check.ts` | Validates skill structure |
| `resolvers/` | Placeholder resolution logic |
| `eval-*.ts` | Evaluation scripts for skill quality |

---

### test/ — Skill Validation and Evals

| File | Responsibility |
|------|-----------------|
| `session-runner.ts` | Runs end-to-end skill tests |
| `eval-store.ts` | Stores evaluation results |
| `llm-judge.ts` | LLM-based evaluation scoring |
| `fixtures/` | Test fixtures |
| `helpers/` | Test utilities |

---

### lib/ — Shared Utilities

| File | Responsibility |
|------|-----------------|
| `gstack-config` | Configuration management |
| `gstack-diff-scope` | Scope diffing for changes |
| `gstack-analytics` | Analytics collection |
| `dev-setup/dev-teardown` | Development environment |

---

### supabase/ — Backend Functions

| Path | Responsibility |
|------|-----------------|
| `functions/` | Edge functions |
| `migrations/` | Database migrations |

---

### Skill Directories (review/, qa/, ship/, etc.)

Each skill is a self-contained unit:
- `SKILL.md` — Generated instructions for Claude Code
- `SKILL.md.tmpl` — Template source
- Supporting docs (checklists, formats)

**Discovery:** Skills in `.agents/skills/` are loaded by Claude Code automatically.

---

## Communication Patterns

### Pattern 1: CLI to Daemon

```
cli.ts -> HTTP POST -> server.ts
```

1. CLI reads `.gstack/browse.json` for port + token
2. CLI POSTs `{command, args}` to `localhost:PORT`
3. CLI prints response to stdout

**Evidence:** `browse/src/cli.ts`

---

### Pattern 2: Daemon to Browser

```
server.ts -> Playwright CDP -> Chromium
```

1. `BrowserManager` holds Playwright `chromium` instance
2. Commands dispatch to `browserManager.page.*` or `browserManager.context.*`
3. CDP (Chrome DevTools Protocol) used internally

**Evidence:** `browse/src/browser-manager.ts`, `browse/src/write-commands.ts`

---

### Pattern 3: Snapshot Ref Resolution

```
snapshot.ts -> accessibility.snapshot() -> @e1, @e2, @e3
write-commands.ts -> resolveRef("@e3") -> locator.click()
```

1. `snapshot -i` calls Playwright's `page.accessibility.snapshot()`
2. Parser walks tree, assigns sequential refs
3. Builds `Map<ref -> Locator>` via `getByRole()`
4. Later commands resolve ref to locator

**Evidence:** `browse/src/snapshot.ts` lines ~100-140

---

### Pattern 4: CircularBuffer Logging

```
Browser events -> CircularBuffer (memory) -> async flush -> .gstack/*.log
```

1. Events push to in-memory ring buffer (50K entries, O(1))
2. Async flush every 1 second to disk
3. Never blocks HTTP request handling

**Evidence:** `browse/src/buffers.ts`

---

### Pattern 5: Template to Generated Doc

```
SKILL.md.tmpl + code metadata -> gen-skill-docs.ts -> SKILL.md
```

Placeholders:
- `{{COMMAND_REFERENCE}}` from `commands.ts`
- `{{SNAPSHOT_FLAGS}}` from `snapshot.ts`
- `{{PREAMBLE}}` from `gen-skill-docs.ts`

**Evidence:** `scripts/gen-skill-docs.ts`

---

## State Management

### Server State (.gstack/browse.json)

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
1. Server starts -> writes state file (mode 0o600)
2. CLI reads state file -> discovers server
3. Health check (`GET /health`) validates server alive
4. Version mismatch -> auto-restart

**Evidence:** `browse/src/server.ts`, `browse/src/cli.ts`

---

## Security Model

| Layer | Mechanism |
|-------|-----------|
| Network | HTTP binds to `localhost` only |
| Auth | Bearer token (UUID) in every request |
| Cookies | Keychain access requires user approval |
| Database | Read-only copy of Chromium cookie DB |
| Logs | No cookie values in logs |

---

## What's Intentionally Excluded

| Exclusion | Reason |
|-----------|--------|
| WebSocket streaming | HTTP request/response simpler, debuggable with curl |
| MCP protocol | Plain HTTP + text is lighter on tokens |
| Multi-user support | One server per workspace |
| iframe support | Ref system doesn't cross frame boundaries |
| Windows/Linux cookie decryption | macOS Keychain only |

---

## Directory Structure Summary

```
gstack/
├── .agents/skills/           # Skill definitions (SKILL.md files)
├── agents/                   # Agent configuration
├── bin/                      # Compiled utilities
├── browse/                   # Browser automation (main deliverable)
│   └── src/
│       ├── cli.ts           # CLI entry point
│       ├── server.ts        # HTTP daemon
│       ├── browser-manager.ts
│       ├── commands.ts      # Command registry
│       ├── read-commands.ts
│       ├── write-commands.ts
│       ├── meta-commands.ts
│       ├── snapshot.ts      # Ref system
│       ├── buffers.ts      # CircularBuffer
│       └── cookie-*.ts
├── lib/                      # Shared utilities
├── scripts/                  # Build + DX tooling
│   ├── gen-skill-docs.ts
│   └── skill-check.ts
├── supabase/                 # Backend
├── test/                     # Skill validation + evals
└── [skill directories]       # review/, qa/, ship/, etc.
```
