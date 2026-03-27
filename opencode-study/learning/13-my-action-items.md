# My Action Items for Building an AI Coding Agent

**Analysis Date:** 2026-03-26
**Source:** Lessons synthesized from OpenCode research
**Purpose:** Prioritized concrete next steps for building a similar AI coding agent project

---

## Priority 1: Foundation (Do First)

### 1.1 Choose Your Type System Strategy

**Rationale:** OpenCode uses both Effect's Schema and Zod. This dual approach provides maximum flexibility but adds complexity.

**Action Items:**
- [ ] Evaluate: Do you need Effect Schema + Zod, or just Zod?
- [ ] If Effect: Use `effect` v2+ (not beta) and enable `@effect/language-service` plugin
- [ ] If Zod-only: Use Zod for all runtime validation (tool params, events, config)
- [ ] Document which validation library is used where

**Reference:** `packages/opencode/src/util/effect-zod.ts` for bridging approach

### 1.2 Implement Tool System First

**Rationale:** Tools are the primary way the agent interacts with the world. OpenCode has 47+ tools. Start with core ones.

**Action Items:**
- [ ] Create `Tool.define()` factory with Zod parameter validation
- [ ] Build core tools: `bash`, `read`, `write`, `edit`, `glob`, `grep`
- [ ] Implement tool registry with `yield* ToolRegistry.Service`
- [ ] Add tool permission system (UX-level, not security boundary)
- [ ] Create tool documentation pattern

**Reference:** `packages/opencode/src/tool/tool.ts`, `packages/opencode/src/tool/*.ts`

### 1.3 Set Up SQLite with Drizzle ORM

**Rationale:** OpenCode migrated from JSON to SQLite. Start with SQLite from day one.

**Action Items:**
- [ ] Set up Drizzle ORM with SQLite
- [ ] Configure WAL mode pragmas (see OpenCode's db.ts)
- [ ] Define schema: `session`, `message`, `part`, `project` tables
- [ ] Add indexes on `session_id`, `project_id`, `workspace_id`
- [ ] Implement transaction helper with Effect context

**Reference:** `packages/opencode/src/storage/db.ts`, `packages/opencode/src/storage/schema.ts`

### 1.4 Build Provider Abstraction

**Rationale:** OpenCode supports 15+ AI providers. Start with 2-3 (Anthropic, OpenAI) and abstract for more.

**Action Items:**
- [ ] Install `@ai-sdk/anthropic`, `@ai-sdk/openai`, `ai`
- [ ] Create `Provider` namespace with branded `ProviderID` and `ModelID` types
- [ ] Build provider registry pattern with custom loaders
- [ ] Add error parsing for overflow detection
- [ ] Implement token counting and context window tracking

**Reference:** `packages/opencode/src/provider/provider.ts`, `packages/opencode/src/provider/schema.ts`

---

## Priority 2: Core Architecture (Do Second)

### 2.1 Implement Agent System with Effect DI

**Rationale:** The agent is the orchestrator. OpenCode uses Effect services for clean dependency injection.

**Action Items:**
- [ ] Create `Agent.Service` extending `ServiceMap.Service`
- [ ] Define built-in agents: `build` (primary), `plan` (read-only), `explore` (fast search)
- [ ] Implement agent permission merging (defaults + user + agent config)
- [ ] Add agent prompt templates stored as `.txt` files
- [ ] Create hidden utility agents for compaction, title, summary

**Reference:** `packages/opencode/src/agent/agent.ts`, `packages/opencode/src/agent/prompt/`

### 2.2 Set Up Server with Hono

**Rationale:** Client/server architecture enables multiple UIs. Hono is lightweight and works with Bun/Cloudflare Workers.

**Action Items:**
- [ ] Create Hono app with routes: `/session`, `/project`, `/config`, `/provider`
- [ ] Implement WebSocket support for streaming
- [ ] Add workspace context middleware
- [ ] Implement Basic Auth for server mode
- [ ] Configure CORS for localhost + production domains

**Reference:** `packages/opencode/src/server/server.ts`, `packages/opencode/src/server/routes/*.ts`

### 2.3 Build Event Bus System

**Rationale:** Decoupled communication between components. OpenCode uses Effect's Stream for pub/sub.

**Action Items:**
- [ ] Create `Bus` namespace with typed event definitions
- [ ] Implement `publish()` and `subscribe()` methods
- [ ] Add event types for: permission.asked, session.message, tool.executed
- [ ] Use `BusEvent.define()` with Zod schemas for type safety
- [ ] Integrate with GlobalBus for cross-instance events

**Reference:** `packages/opencode/src/bus/index.ts`, `packages/opencode/src/bus/bus-event.ts`

### 2.4 Implement Session State Machine

**Rationale:** Sessions track conversation history with messages and parts.

**Action Items:**
- [ ] Define `Message` types: `User`, `Assistant`, `System`
- [ ] Define `Part` discriminated union: `TextPart`, `ToolPart`, `ReasoningPart`, `StepStartPart`, `StepFinishPart`
- [ ] Create `Session` repository with pagination
- [ ] Implement streaming message creation
- [ ] Add session sharing with secret tokens

**Reference:** `packages/opencode/src/session/session.sql.ts`, `packages/opencode/src/session/message.ts`

---

## Priority 3: Polish & Productionize (Do Third)

### 3.1 Add LSP Integration

**Rationale:** Code intelligence differentiates a coding agent from a chatbot.

**Action Items:**
- [ ] Implement LSP client using `vscode-jsonrpc`
- [ ] Create LSP server discovery for TypeScript, Go, Python, Rust
- [ ] Add auto-installation for LSP servers not in PATH
- [ ] Implement features: hover, goto definition, references, diagnostics
- [ ] Add workspace root detection with project markers

**Reference:** `packages/opencode/src/lsp/index.ts`, `packages/opencode/src/lsp/server.ts`

### 3.2 Build Git Integration

**Rationale:** Git worktrees provide safe sandbox isolation for agent operations.

**Action Items:**
- [ ] Create Git service using `effect/unstable/process`
- [ ] Implement worktree lifecycle: create, remove, reset
- [ ] Add branch detection with remote prefix support
- [ ] Configure git with safe defaults (no-optional-locks, core.autocrlf=false)
- [ ] Implement path boundary checking (agent cannot escape worktree)

**Reference:** `packages/opencode/src/git/index.ts`, `packages/opencode/src/worktree/index.ts`

### 3.3 Implement MCP Client

**Rationale:** Model Context Protocol enables extending tools via external servers.

**Action Items:**
- [ ] Integrate `@modelcontextprotocol/sdk`
- [ ] Support transport types: StdioClientTransport (local), StreamableHTTP (remote)
- [ ] Convert MCP tools to AI SDK tool format
- [ ] Add OAuth support for MCP servers requiring auth
- [ ] Sanitize MCP client and tool names to prevent collisions

**Reference:** `packages/opencode/src/mcp/index.ts`, `packages/opencode/src/mcp/oauth-provider.ts`

### 3.4 Add Plugin System

**Rationale:** Extensibility via hooks for auth providers, custom tools, and system modifications.

**Action Items:**
- [ ] Create `Plugin` interface with hooks: `event`, `tool`, `auth`, `chat.message`
- [ ] Implement plugin loading from npm and `file://` paths
- [ ] Add internal plugins: CodexAuth, CopilotAuth
- [ ] Create `@opencode-ai/plugin` SDK package
- [ ] Implement plugin deduplication for named+default exports

**Reference:** `packages/opencode/src/plugin/index.ts`, `packages/plugin/src/index.ts`

---

## Priority 4: Quality & Governance (Do In Parallel)

### 4.1 Enable TypeScript Strictness

**Rationale:** OpenCode inconsistently applies strict mode. Don't repeat this.

**Action Items:**
- [ ] Add `"strict": true` to all package tsconfigs
- [ ] Enable `"noUncheckedIndexedAccess": true`
- [ ] Enable `"noImplicitReturns": true`
- [ ] Run `bun typecheck` in pre-push hook

**Reference:** `packages/opencode/tsconfig.json`, `.husky/pre-push`

### 4.2 Add ESLint

**Rationale:** OpenCode has no ESLint. This is a gap.

**Action Items:**
- [ ] Install `@typescript-eslint/parser`, `@typescript-eslint/eslint-plugin`
- [ ] Configure rules: no-unused-vars, no-floating-promises, consistent-return
- [ ] Add security rules: no-eval, no-new-func
- [ ] Run ESLint in CI, not just pre-push

### 4.3 Build Test Infrastructure

**Rationale:** OpenCode has good unit + E2E coverage. Match this.

**Action Items:**
- [ ] Set up `bun:test` for unit tests
- [ ] Create Effect-aware test utilities (`testEffect`, `provideInstance`)
- [ ] Build fixture system: `tmpdir()`, `provideTmpdirInstance()`
- [ ] Add Playwright for E2E testing (if web UI)
- [ ] Run tests in CI with retries on flakiness

**Reference:** `packages/opencode/test/fixture/fixture.ts`, `packages/app/playwright.config.ts`

### 4.4 Document Conventions

**Rationale:** OpenCode has AGENTS.md for AI developers. Create similar for human developers.

**Action Items:**
- [ ] Create `ARCHITECTURE.md`: System overview, key patterns, module responsibilities
- [ ] Create `CONTRIBUTING.md`: Issue templates, PR requirements, commit conventions
- [ ] Create `STYLE.md`: Naming conventions (single-word tools), formatting rules
- [ ] Create `ONBOARDING.md`: Dev environment setup, first task guide

**Reference:** `packages/opencode/AGENTS.md`, `packages/opencode/CONTRIBUTING.md`

---

## Priority 5: Ship and Iterate (Do Continuously)

### 5.1 Start with Minimal Viable Agent

**Rationale:** OpenCode has 47+ tools. Ship with 5-10 core tools first.

**MVP Tools:**
1. `bash` - Execute shell commands
2. `read` - Read file contents
3. `write` - Write file contents
4. `edit` - Make targeted changes
5. `glob` - Find files by pattern
6. `grep` - Search file contents
7. `lsp` - Code intelligence

### 5.2 Iterate Based on Real Usage

**Rationale:** OpenCode evolved based on user feedback. Build in public, iterate fast.

**Action Items:**
- [ ] Ship early with basic functionality
- [ ] Add telemetry for tool usage patterns
- [ ] Prioritize improvements based on user friction
- [ ] Maintain changelog with each release

### 5.3 Build Community

**Rationale:** OpenCode's vouch system enables trust. Start governance early.

**Action Items:**
- [ ] Set up issue templates from day one
- [ ] Define contribution guidelines (Fixes #123 required)
- [ ] Create AI-agent specific guidelines if allowing AI PRs
- [ ] Establish security reporting process

---

## Non-Goals (What Not to Do)

Based on OpenCode's complexity:

- **Don't build desktop app initially** - Focus on CLI and web
- **Don't support 15+ AI providers** - Start with 2-3, abstract later
- **Don't implement Nix dev environments** - Use standard npm/bun
- **Don't build enterprise features** - Add auth later when needed
- **Don't maintain JSON storage** - SQLite from day one

---

## Key Files to Reference

| File | Purpose |
|------|---------|
| `packages/opencode/src/tool/tool.ts` | Tool factory pattern |
| `packages/opencode/src/agent/agent.ts` | Agent service definition |
| `packages/opencode/src/provider/provider.ts` | Multi-provider abstraction |
| `packages/opencode/src/storage/db.ts` | SQLite configuration |
| `packages/opencode/src/bus/index.ts` | Event pub/sub |
| `packages/opencode/src/server/server.ts` | HTTP/WebSocket server |
| `packages/opencode/src/session/session.sql.ts` | Database schema |
| `packages/opencode/src/lsp/index.ts` | LSP integration |
| `packages/opencode/src/git/index.ts` | Git service |
| `packages/opencode/src/worktree/index.ts` | Worktree sandboxing |
| `packages/opencode/src/mcp/index.ts` | MCP client |
| `packages/opencode/src/plugin/index.ts` | Plugin system |
| `packages/opencode/AGENTS.md` | Agent conventions |

---

## Summary: Build Order

```
Phase 1: Foundation
  - Tool system (bash, read, write, edit, glob, grep)
  - SQLite with Drizzle
  - Provider abstraction (Anthropic + OpenAI)
  - Basic agent (build mode)

Phase 2: Core Architecture
  - Hono server with WebSocket
  - Event bus
  - Session state machine
  - Permission system (UX-level)

Phase 3: Polish
  - LSP integration
  - Git integration + worktrees
  - MCP client
  - Plugin system

Phase 4: Quality
  - TypeScript strict mode
  - ESLint
  - Tests (unit + E2E)
  - Documentation

Phase 5: Ship
  - Minimal viable product
  - Real user feedback
  - Iterate and improve
```
