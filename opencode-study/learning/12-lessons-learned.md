# OpenCode Project: Lessons Learned

**Analysis Date:** 2026-03-26
**Source:** 8 research documents covering topology, tech stack, community, features (6 batches), architecture, code quality, and security/performance
**Purpose:** Honest assessment of patterns to emulate, mistakes to avoid, and unexpected findings for building a similar AI coding agent

---

## What to Emulate (Successful Patterns)

### 1. Effect Framework for Dependency Injection
**Location:** `packages/opencode/src/effect/`, throughout codebase

The Effect framework provides a clean, type-safe dependency injection system that scales well:

```typescript
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const spawner = yield* ChildProcessSpawner.ChildProcessSpawner
    return Service.of({ /* implementation */ })
  })
)
```

**Why it works:** Services are composable, testable, and the Effect type system catches missing dependencies at compile time. The `yield*` pattern makes dependencies explicit.

### 2. Provider Abstraction for Multi-LLM Support
**Location:** `packages/opencode/src/provider/provider.ts` (1500+ lines)

Supporting 15+ AI providers (Anthropic, OpenAI, Google, AWS Bedrock, etc.) through a unified adapter pattern:

```typescript
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  // ... 15+ providers
}
```

Custom loaders handle provider-specific quirks (e.g., AWS region prefixing, GitHub Copilot's dual API).

### 3. Tool System with Zod Validation
**Location:** `packages/opencode/src/tool/tool.ts`, `packages/opencode/src/tool/*.ts`

Tools use a factory pattern with Zod schemas for input validation:

```typescript
export const BashTool = Tool.define("bash", async () => ({
  description: "Execute shell commands",
  parameters: z.object({
    command: z.string(),
    timeout: z.number().optional(),
    workdir: z.string().optional(),
  }),
  async execute(params, ctx) {
    // Implementation
    return { title: "...", output: "...", metadata: {} }
  }
}))
```

### 4. Event Bus for Decoupled Communication
**Location:** `packages/opencode/src/bus/index.ts`

Typed pub/sub system for in-process events:

```typescript
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define("permission.replied", Reply),
}

export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(def: D, properties: ...) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
}
```

### 5. Client/Server Architecture
**Location:** `packages/opencode/src/server/server.ts`, `packages/opencode/src/client/`

Enables multiple UI entry points (CLI, Desktop, Web) to connect to the same backend:

```
CLI/Desktop/Web → HTTP/WebSocket → Server (Hono + Bun) → SQLite, Tools, MCP
```

Server can run as sidecar (desktop), embedded (CLI run), or standalone (network serve).

### 6. Git Worktree for Sandbox Isolation
**Location:** `packages/opencode/src/worktree/index.ts` (600 lines)

Uses native git worktrees to create isolated project sandboxes:

```typescript
const removed = yield* git(["worktree", "remove", "--force", entry.path], { cwd: Instance.worktree })
if (removed.code !== 0) {
  // Handle stale entries gracefully
}
```

### 7. SQLite with WAL Mode and Optimized Pragmas
**Location:** `packages/opencode/src/storage/db.ts`

```typescript
db.run("PRAGMA journal_mode = WAL")
db.run("PRAGMA synchronous = NORMAL")
db.run("PRAGMA busy_timeout = 5000")
db.run("PRAGMA cache_size = -64000")  // 64MB
db.run("PRAGMA foreign_keys = ON")
```

### 8. Strict Community Governance
**Location:** `.github/VOUCHED.td`, `.github/TEAM_MEMBERS`, `CONTRIBUTING.md`

The vouch system and strict issue templates maintain quality:

- Issue templates enforced (blank issues auto-close after 2 hours)
- `Fixes #123` required for all PRs
- No AI-generated walls of text in PRs
- Vouched contributors vs denounced spammers explicitly tracked

### 9. Schema-First Design with Effect-Zod Bridge
**Location:** `packages/opencode/src/util/effect-zod.ts`

Seamless interoperability between Effect's Schema and Zod:

```typescript
export function zod<S extends Schema.Top>(schema: S): z.ZodType<Schema.Schema.Type<S>> {
  return walk(schema.ast) as z.ZodType<Schema.Schema.Type<S>>
}
```

### 10. LSP Auto-Installation
**Location:** `packages/opencode/src/lsp/server.ts`

LSP servers are automatically downloaded and installed from their sources (GitHub releases, npm, gem, go install) when not found in PATH. Supports 20+ languages.

---

## What to Avoid (Mistakes and Weaknesses)

### 1. Monolithic Components
**Location:** `packages/app/src/pages/session.tsx` (61,487 bytes)

The session page is a single 61KB component handling message timeline, history window, file diffs, terminal, comments, and commands. Should be split into smaller, focused components.

**Fix:** Decompose into `<MessageTimeline>`, `<HistoryWindow>`, `<DiffViewer>`, `<TerminalPanel>`, `<CommentThread>` components.

### 2. Inconsistent TypeScript Strictness
**Location:** Most packages lack `strict: true` in tsconfig

| Package | strict | noUncheckedIndexedAccess |
|---------|--------|--------------------------|
| `packages/opencode` | implicit (false) | false |
| `packages/app` | implicit (false) | false |
| `github` | implicit (false) | **true** (only one) |

**Fix:** Enable `strict: true` and `noUncheckedIndexedAccess: true` across all packages.

### 3. No ESLint Configuration
**Location:** None found

The project relies solely on Prettier and TypeScript for code quality. No ESLint rules for:
- No `no-unused-vars` enforcement
- No complexit bounds
- No security rules

**Fix:** Add ESLint with `@typescript-eslint` and security-focused rules.

### 4. Direct process.env Access
**Location:** `packages/opencode/src/provider/provider.ts:276`

Several places bypass the `Flag` abstraction:

```typescript
const envToken = process.env.AWS_BEARER_TOKEN_BEDROCK
if (envToken) return envToken
```

**Fix:** Use the `Flag` system consistently for all environment variables.

### 5. Technical Debt Left in Code
**Location:** Multiple files

- `// @ts-ignore (TODO: kill this code so we dont have to maintain it)` in `provider.ts:472`
- `@deprecated do not use this dumb shit` comment in `server.ts:571`

**Fix:** Address TODOs before merging, remove deprecated code rather than commenting.

### 6. No Plugin Unloading
**Location:** `packages/opencode/src/plugin/index.ts`

Plugins are loaded but never unloaded:

```typescript
// Note: No unload/cleanup function exists
// This could cause memory leaks with dynamically updated plugins
```

**Fix:** Implement plugin lifecycle with proper cleanup on unload/dispose.

### 7. Hardcoded Values
Multiple locations have magic numbers without explanation:

| Location | Issue |
|----------|-------|
| `server.ts:4096` | Default server port |
| `extension.ts:16384-65535` | VS Code port range |
| `mcp/oauth-provider.ts:19876` | Fixed OAuth callback port |
| `shell.ts: fish, nu` | Blacklisted shells (no documented reason) |

**Fix:** Extract to configuration with documented rationales.

### 8. Windows Encoding Hack
**Location:** `packages/opencode/src/pty/index.ts:328-332`

```typescript
if (process.platform === "win32") {
  env.LC_ALL = "C.UTF-8"
  env.LC_CTYPE = "C.UTF-8"
  env.LANG = "C.UTF-8"
}
```

Forces UTF-8 locale on Windows which may cause encoding issues. Better to detect system encoding properly.

### 9. JSON Storage Still Present
**Location:** `packages/opencode/src/storage/storage.ts`

Despite ongoing migration to SQLite, JSON file storage still exists with no cleanup after migration. `OPENCODE_SKIP_MIGRATIONS` flag doesn't skip migration table writes.

**Fix:** Clean up JSON files after successful SQLite migration.

### 10. LSP Graceful Degradation Missing
**Location:** `packages/opencode/src/lsp/index.ts`

If an LSP server fails to spawn, it's marked "broken" permanently with no recovery mechanism:

```typescript
// If spawn fails after process creation, broken state persists
// No retry or reset mechanism
```

**Fix:** Implement exponential backoff retry or manual reset for failed LSP servers.

---

## Surprises (Unexpected Findings)

### 1. Permissions Are UX, Not Security
**Location:** `SECURITY.md`, `packages/opencode/src/permission/index.ts`

OpenCode explicitly states:
> "OpenCode does not sandbox the agent; permissions are a UX feature, not security"

This is an honest acknowledgment that an LLM agent with file system access cannot be truly sandboxed through configuration alone.

**Implication:** If security isolation is needed, use Docker/VM containment.

### 2. Default Branch Is `dev`
**Location:** Repository default branch

Most projects use `main` as default. OpenCode uses `dev`, likely for:
- Clearer separation between stable (tags) and development
- GitHub Actions triggered on `dev` for alpha/beta releases

### 3. Tauri v2 for Desktop with Electron Variant
**Location:** `packages/desktop/`, `packages/desktop-electron/`

Desktop app uses Tauri v2 (Rust backend) as primary, with Electron as secondary. This is unusual - typically projects pick one. Electron likely maintained for:
- Broader Node.js ecosystem compatibility
- Specific Electron-only features
- Migration path from Electron to Tauri

### 4. Development in Public with Nix
**Location:** `flake.nix`, `nix/` directory

Nix flake for reproducible development environments, but main development uses Bun directly (v1.3.11). Nix may be for CI or specific contributors.

### 5. SST Deploys to Cloudflare, Not AWS
**Location:** `sst.config.ts`

```typescript
home: "cloudflare",  // Primary hosting
providers: {
  stripe: { apiKey: process.env.STRIPE_SECRET_KEY! },
  planetscale: "0.4.1",  // MySQL-compatible
}
```

Despite using AWS Bedrock for AI, the web infrastructure runs on Cloudflare Workers/KV/R2.

### 6. SolidJS for UI, Not React
**Location:** `packages/app/`, `packages/ui/`

The entire UI ecosystem (app, desktop, UI components) uses SolidJS, not React. Reasons likely include:
- Fine-grained reactivity (better for real-time updates)
- Smaller bundle size
- Better performance characteristics for terminal-style UIs

### 7. Bun as Primary Runtime
**Location:** `package.json: packageManager`

```json
"packageManager": "bun@1.3.11"
```

Bun is the package manager and runtime, not just a build tool. This is early adoption - Bun 1.x is production-stable but Bun adoption in large projects is still relatively rare.

### 8. Single-Word Naming Convention
**Location:** `packages/opencode/AGENTS.md`

Agents, tools, and commands use single-word naming enforced via documentation:
- `build`, `plan`, `general` agents
- `bash`, `read`, `write`, `glob` tools
- `run`, `chat`, `serve`, `debug` commands

No hyphenated or camelCase names.

### 9. Effect Framework is Beta
**Location:** `packages/opencode/package.json`

```json
"effect": "4.0.0-beta.35"
```

Using beta-stage Effect (4.0.0-beta) for a production tool is a calculated risk. They accept potential breaking changes in exchange for latest features.

### 10. Shared Type Between Frontend and Backend
**Location:** `packages/opencode/src/sql.d.ts`, `packages/app/src/`

TypeScript project references and shared types between packages. SQL type declarations (`sql.d.ts`) are shared across packages for type-safe database access.

---

## Summary Assessment

| Dimension | Rating | Notes |
|-----------|--------|-------|
| Architecture | Strong | Clean layered design, good separation of concerns |
| Code Quality | Moderate | Good patterns but inconsistent strictness, no ESLint |
| Type Safety | Moderate | Zod everywhere but TypeScript strict not enforced |
| Testing | Good | Unit + E2E infrastructure present |
| Security | Honest | Explicit about UX vs security tradeoffs |
| Performance | Good | SQLite WAL, lazy init, caching well-implemented |
| Developer Experience | Strong | Effect DI, good tooling, clear conventions |
| Technical Debt | Moderate | Some TODOs, monolithic components, hardcoded values |
| Documentation | Strong | AGENTS.md, CONTRIBUTING.md, multi-language README |
| Community | Strong | Governance, vouch system, strict issue triage |

**Overall:** A well-architected project that made pragmatic tradeoffs. The technical debt is manageable and the team is actively developing. Key areas for improvement are TypeScript strictness, component decomposition, and ESLint adoption.
