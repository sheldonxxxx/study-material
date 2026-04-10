# Features Batch 2 Research (Features 4-6)

## Feature 4: LSP + AST-Grep Tools

### Overview

LSP and AST-Grep are IDE-precision tools that give every agent in oh-my-openagent access to professional-grade code navigation, search, and refactoring capabilities. LSP handles semantic operations (goto definition, find references, rename) while AST-Grep provides pattern-aware code search and rewriting.

### LSP Tools

**Key Files:** `src/tools/lsp/` (38 files)

**Tool Surface:**
- `lsp_goto_definition` - Jump to symbol definition
- `lsp_find_references` - Find all usages across workspace
- `lsp_symbols` - Document outline (file) or workspace symbol search
- `lsp_diagnostics` - Errors/warnings from language server
- `lsp_prepare_rename` / `lsp_rename` - Safe rename with preparation check

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      LSP Manager                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ LSPClient   │  │ LSPClient   │  │ LSPClient   │         │
│  │ (TypeScript)│  │ (Python)    │  │ (Rust)      │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                 │                 │                │
│    ┌────┴─────────────────┴─────────────────┴────┐         │
│    │          Unified LSP Client Wrapper           │         │
│    │  - Document sync (didOpen/didChange/didSave) │         │
│    │  - Request/notification dispatch              │         │
│    │  - Response parsing                          │         │
│    └──────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

**Key Implementation Details:**

1. **LSP Client (`lsp-client.ts`):**
   - Manages document lifecycle: opens files with `textDocument/didOpen`, syncs changes via `textDocument/didChange`, triggers diagnostics with `textDocument/didSave`
   - Converts 1-based lines to 0-based for LSP protocol
   - Maintains `documentVersions` map for incremental sync
   - 1-second sleep after initial open to allow server initialization

2. **Process Management (`lsp-process.ts`):**
   - **Critical Windows Fix:** Uses Node.js `child_process` spawn instead of Bun spawn on Windows due to segfault bug (oven-sh/bun#25798)
   - Validates working directory exists before spawning
   - Creates unified process abstraction bridging Bun Subprocess and Node.js ChildProcess with identical API

3. **Server Resolution (`server-resolution.ts` + `server-definitions.ts`):**
   - 30+ builtin language servers: typescript, deno, vue, eslint, biome, gopls, ruby-lsp, basedpyright, pyright, ruff, rust (rust-analyzer), clangd, svelte, astro, bash, jdtls, yaml-ls, lua-ls, php, dart, terraform, dockerfile, gleam, clojure-lsp, nixd, tinymist, haskell-language-server, kotlin-ls, ocaml-lsp, prisma, texlab, fsharp, csharp, elixir-ls, zls, sourcekit-lsp
   - **Synced with OpenCode:** Comment in `server-definitions.ts` notes sync with OpenCode's server.ts
   - Priority: project config > user config > opencode config > builtin
   - Install hints provided for each server (e.g., `npm install -g typescript-language-server`)

4. **Server Installation Check (`server-installation.ts`):**
   - Checks PATH for command existence
   - Supports Windows PATHEXT (.exe, .cmd, .bat, .ps1)
   - Supports absolute paths
   - Checks additional path bases via `getLspServerAdditionalPathBases()`
   - Hardcoded exception: `bun` and `node` always available

5. **Config Loading (`server-config-loader.ts`):**
   - Loads from three config locations with priority:
     1. Project: `{cwd}/.opencode/oh-my-opencode.json`
     2. User: `~/.config/opencode/oh-my-opencode.json`
     3. OpenCode: `~/.config/opencode/opencode/oh-my-opencode.json`
   - Uses JSONC parser (allows comments)
   - Custom servers can override builtins

6. **Workspace Root Finding (`lsp-client-wrapper.ts`):**
   - Searches up from file path for markers: `.git`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`

7. **Directory Diagnostics (`directory-diagnostics.ts`):**
   - Scans directory for files matching extension
   - Runs LSP diagnostics on each file
   - Aggregates results
   - Respects severity filters

**Clever Solutions:**
- Dual-result handling: LSP returns can be `Location | Location[] | LocationLink[]` - the code handles all three
- Graceful degradation: diagnostics endpoint falls back to cached diagnostics if `textDocument/diagnostic` fails
- Preparation step for rename prevents invalid renames from touching files

**Technical Debt / Shortcuts:**
- 1-second hardcoded sleep after file open (arbitrary but functional)
- No streaming/chunking for large symbol results
- Directory diagnostics runs sequentially per file

---

### AST-Grep Tools

**Key Files:** `src/tools/ast-grep/` (15 files)

**Tool Surface:**
- `ast_grep_search` - Pattern-aware code search
- `ast_grep_replace` - AST-aware code rewriting

**Language Support:** 25 languages via CLI: bash, c, cpp, csharp, css, elixir, go, haskell, html, java, javascript, json, kotlin, lua, nix, php, python, ruby, rust, scala, solidity, swift, typescript, tsx, yaml

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    ast_grep_search                          │
│                    ast_grep_replace                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    │   runSg()   │
                    │  (cli.ts)   │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────┴────┐      ┌─────┴─────┐    ┌─────┴─────┐
    │  PATH   │      │ @ast-grep │    │  Download │
    │   env   │      │   /cli    │    │   binary │
    │   var   │      │  (fallback)│    │  (cache) │
    └─────────┘      └───────────┘    └───────────┘
```

**Key Implementation Details:**

1. **CLI Invocation (`cli.ts` - `runSg()`):**
   - Spawns `sg` binary with args: `-p {pattern} --lang {lang} --json=compact`
   - Context lines via `-C {n}`, globs via `--globs {glob}`
   - For replace: `-r {rewrite}`

2. **Binary Management (`downloader.ts`):**
   - Auto-downloads from GitHub releases (`ast-grep/ast-grep`)
   - Caches in platform-specific location:
     - Linux/macOS: `~/.cache/oh-my-opencode/bin/` (or `$XDG_CACHE_HOME`)
     - Windows: `%LOCALAPPDATA%/oh-my-opencode/bin/`
   - Version from `@ast-grep/cli/package.json` or fallback `0.41.1`
   - Platform mapping: darwin-arm64, darwin-x64, linux-arm64, linux-x64, win32-x64, etc.

3. **Two-Pass Rewrite Pattern (`cli.ts`):**
   ```typescript
   // ast-grep CLI silently ignores --update-all when --json is present
   // So for rewrite + updateAll, we run TWO invocations:
   // Pass 1: --json=compact to get match results
   // Pass 2: --update-all to actually write files
   ```
   This is a known ast-grep CLI behavior quirk that the code works around.

4. **Helpful Error Messages (`tools.ts`):**
   - Detects empty results and provides hints:
     - Python: Remove trailing colon from `def $FUNC:` patterns
     - JavaScript/TypeScript: Function patterns need params/body

5. **Output Parsing (`sg-compact-json-output.ts`):**
   - Parses ast-grep's compact JSON format
   - Handles truncated results (max_matches, max_output_bytes, timeout)

**Clever Solutions:**
- Automatic fallback chain: PATH > @ast-grep/cli > download > error message with install options
- Recursive retry on ENOENT (auto-downloads if binary missing)
- URL-encoded API keys in Exa URL (handles special characters)

**Technical Debt / Shortcuts:**
- No cancellation support for long-running searches
- No incremental results (waits for full completion)
- Default timeout 300 seconds (5 min) is quite long

---

## Feature 5: Built-in MCPs

### Overview

Three built-in MCP servers are always available without manual configuration: Exa (web search), Context7 (library documentation), and Grep.app (GitHub code search). The system also supports skill-embedded MCPs that spin up on-demand and clean up after task completion.

### Architecture

**Key Files:** `src/mcp/` (10 files)

```
┌─────────────────────────────────────────────────────────────┐
│                 createBuiltinMcps()                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  websearch  │  context7  │  grep_app  │ [disabled]  │   │
│  └─────────────┴────────────┴────────────┴─────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
    │    Exa     │  │  Context7   │  │  Grep.app   │
    │ (mcp.exa.ai)│  │(mcp.context7│  │(mcp.grep.app)
    │            │  │   .com)     │  │            │
    └─────────────┘  └─────────────┘  └─────────────┘
```

### Three Built-in MCPs

**1. Websearch (Exa/Tavily)**

**File:** `src/mcp/websearch.ts`

```typescript
interface WebsearchConfig {
  type: "remote"
  url: string
  enabled: boolean
  headers?: Record<string, string>
  oauth?: false
}
```

- **Default Provider: Exa**
  - URL: `https://mcp.exa.ai/mcp?tools=web_search_exa`
  - If `EXA_API_KEY` set: adds `?exaApiKey=...` and `x-api-key` header
- **Alternative Provider: Tavily**
  - URL: `https://mcp.tavily.com/mcp/`
  - Requires `TAVILY_API_KEY`
  - Uses `Authorization: Bearer {key}` header

**Clever Detail:** API keys are URL-encoded in the Exa URL to handle special characters like `+`, `&`, `=`.

**2. Context7**

**File:** `src/mcp/context7.ts`

```typescript
const context7 = {
  type: "remote" as const,
  url: "https://mcp.context7.com/mcp",
  enabled: true,
  headers: process.env.CONTEXT7_API_KEY
    ? { Authorization: `Bearer ${process.env.CONTEXT7_API_KEY}` }
    : undefined,
  oauth: false as const,  // Disable OAuth auto-detection
}
```

- **Note:** Has conditional `CONTEXT7_API_KEY` support (though header says Bearer)
- **Important:** `oauth: false` explicitly disables OAuth auto-detection since Context7 uses API key header, not OAuth

**3. Grep.app**

**File:** `src/mcp/grep-app.ts`

```typescript
const grep_app = {
  type: "remote" as const,
  url: "https://mcp.grep.app",
  enabled: true,
  oauth: false as const,
}
```

- No authentication - public API
- Ultra-fast code search across public GitHub repos

### MCP Creation Factory

**File:** `src/mcp/index.ts`

```typescript
export function createBuiltinMcps(disabledMcps: string[] = [], config?: OhMyOpenCodeConfig) {
  const mcps: Record<string, RemoteMcpConfig> = {}

  if (!disabledMcps.includes("websearch")) {
    mcps.websearch = createWebsearchConfig(config?.websearch)
  }

  if (!disabledMcps.includes("context7")) {
    mcps.context7 = context7
  }

  if (!disabledMcps.includes("grep_app")) {
    mcps.grep_app = grep_app
  }

  return mcps
}
```

- Each MCP can be disabled individually via `disabledMcps` array
- Configuration can override websearch provider (exa vs tavily)

### Skill-Embedded MCPs

**Key Files:**
- `src/tools/skill-mcp/` - Skill MCP invocation tool
- `src/hooks/skill-mcp-manager/` - Manages skill MCP lifecycle

The skill-embedded MCP system allows skills to carry their own on-demand MCP servers:
- Spin up scoped to current task
- Disappear when task completes
- Keep context window clean by not leaving persistent MCP connections

### MCP Type System

**File:** `src/mcp/types.ts`

```typescript
export const McpNameSchema = z.enum(["websearch", "context7", "grep_app"])
export type McpName = z.infer<typeof McpNameSchema>
```

### Clever Solutions

1. **Provider Flexibility:** Websearch supports two providers (Exa, Tavily) with runtime switching
2. **API Key Handling:** Different providers use different auth mechanisms (header vs Bearer URL param)
3. **Graceful Degradation:** If no API key, MCPs still work (possibly rate-limited or with reduced quota)
4. **OAuth Explicit Disable:** Context7 explicitly disables OAuth auto-detection since it uses API key auth

### Technical Debt / Shortcuts

- Context7's `CONTEXT7_API_KEY` vs `Bearer` mismatch in code comment
- No retry logic for MCP connection failures
- No connection pooling for multiple MCP requests

---

## Feature 6: ultrawork / Ralph Loop

### Overview

The Ralph Loop is a self-referential development loop where the agent works continuously toward a goal, detecting `<promise>DONE</promise>` for completion. If the agent stops without completing, the system "yanks it back." Ultrawork is an enhanced version with Oracle verification that requires skeptical review before considering work complete.

**Key Files:** `src/hooks/ralph-loop/` (27 files)

### Activation Triggers

**File:** `src/hooks/keyword-detector/`

The `keyword-detector` hook monitors for special keywords:
- `ultrawork` or `ulw` (case insensitive, word boundary) - triggers ultrawork mode
- Patterns also detected in code blocks are ignored (prevents false triggers from documentation)

```typescript
const KEYWORD_DETECTORS = [
  {
    pattern: /\b(ultrawork|ulw)\b/i,
    message: getUltraworkMessage,
  },
  // ... search and analyze modes
]
```

### State Management

**File:** `src/hooks/ralph-loop/storage.ts`

State persists in `.sisyphus/ralph-loop.local.md` as YAML frontmatter:

```yaml
---
active: true
iteration: 1
max_iterations: 100
completion_promise: "DONE"
started_at: "2026-03-26T..."
session_id: "session-uuid"
ultrawork: false
strategy: "continue"
---
Original task prompt goes here
```

**State Fields:**
- `active` - Loop is running
- `iteration` - Current iteration number
- `max_iterations` - Cap (undefined for ultrawork)
- `completion_promise` - Pattern to detect (default: "DONE")
- `initial_completion_promise` - Original promise before verification
- `ultrawork` - Whether ultrawork mode
- `verification_pending` - Oracle verification in progress
- `verification_session_id` - Oracle's session ID
- `strategy` - "reset" (new session) or "continue" (same session)

### Completion Detection

**File:** `src/hooks/ralph-loop/completion-promise-detector.ts`

Two detection mechanisms:

1. **Transcript Scan (Primary):**
   - Reads `.opencode/transcripts/{session_id}.jsonl`
   - Filters for user messages
   - Uses timestamp filtering (only messages after `started_at`)
   - Regex match: `<promise>{PROMISE}</promise>`

2. **API Fallback:**
   - Calls `ctx.client.session.messages()` API
   - Extracts text from assistant messages only
   - Scoped to messages since `message_count_at_start`

### Loop Flow

**File:** `src/hooks/ralph-loop/ralph-loop-event-handler.ts`

On `session.idle` event:

```
┌─────────────────────────────────────────────────────────────┐
│                    session.idle Event                       │
└──────────────────────────┬────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────┴────┐      ┌─────┴─────┐    ┌─────┴─────┐
    │Completion│      │Verification│    │ Max Iter  │
    │Detected? │      │  Pending?  │    │  Reached? │
    └────┬────┘      └─────┬─────┘    └─────┬─────┘
         │                 │                 │
    ┌────┴────┐      ┌─────┴─────┐    ┌─────┴─────┐
    │Handle   │      │  Handle   │    │  Clear &  │
    │Complete │      │ Pending   │    │  Stop     │
    └─────────┘      │Verification│    └───────────┘
                     └─────┬─────┘
                           │
                    ┌──────┴──────┐
                    │  Increment  │
                    │  Iteration  │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Continue   │
                    │   Loop      │
                    │ (inject     │
                    │  prompt)    │
                    └─────────────┘
```

### Iteration Continuation

**File:** `src/hooks/ralph-loop/iteration-continuation.ts`

Two strategies:

1. **`continue` (default):** Injects continuation prompt into same session
2. **`reset`:** Creates new child session, injects there, switches TUI to new session

**Continuation Prompt Template:**
```
[SYSTEM DIRECTIVE] - RALPH LOOP {ITERATION}/{MAX}]

Your previous attempt did not output the completion promise. Continue working on the task.

IMPORTANT:
- Review your progress so far
- Continue from where you left off
- When FULLY complete, output: <promise>{PROMISE}</promise>
- Do not stop until the task is truly done

Original task:
{PROMPT}
```

### Ultrawork Verification

**File:** `src/hooks/ralph-loop/completion-handler.ts`

When `<promise>DONE</promise>` is detected in ultrawork mode:

1. **Phase 1 - Verification Pending:**
   - Changes `completion_promise` to "VERIFIED"
   - Injects Oracle verification prompt:
   ```
   REQUIRED NOW:
   - Call Oracle using task(subagent_type="oracle", ...)
   - Ask Oracle to verify whether the original task is actually complete
   - The system will inspect the Oracle session for verification result
   ```

2. **Phase 2 - Oracle Reviews:**
   - System monitors for Oracle's `<promise>VERIFIED</promise>`
   - Looks for Oracle session ID in task metadata

3. **Phase 3 - Success or Retry:**
   - If verified: loop ends successfully
   - If not verified: restart with `VERIFICATION FAILED` prompt, Oracle's concerns included

### Error Handling & Recovery

**File:** `src/hooks/ralph-loop/loop-session-recovery.ts`

- **Orphaned State Detection:** If loop state exists but session was deleted, clears state
- **Verification Recovery:** If verification session ID lost, scans parent session for Oracle evidence
- **API Timeout Protection:** All API calls wrapped in 5-second timeout
- **Session Existence Check:** Optional callback to verify session still exists

### State Controller

**File:** `src/hooks/ralph-loop/loop-state-controller.ts`

```typescript
const loopState = {
  startLoop(sessionID, prompt, options) → boolean
  cancelLoop(sessionID) → boolean
  getState() → RalphLoopState | null
  clear() → boolean
  incrementIteration() → RalphLoopState | null
  setSessionID(sessionID) → RalphLoopState | null
  markVerificationPending(sessionID) → RalphLoopState | null
  setVerificationSessionID(sessionID, verificationSessionID) → RalphLoopState | null
  restartAfterFailedVerification(sessionID) → RalphLoopState | null
}
```

### Session Reset Strategy

**File:** `src/hooks/ralph-loop/session-reset-strategy.ts`

```typescript
async function createIterationSession(ctx, parentSessionID, directory) {
  // Creates child session with parentID reference
  const result = await ctx.client.session.create({
    body: {
      parentID: parentSessionID,
      title: "Ralph Loop Iteration",
    },
    query: { directory },
  })
  return result.data.id
}
```

- Uses OpenCode's session create API
- Sets parent-child relationship for traceability
- Updates TUI to show new session

### Continuation Prompt Injection

**File:** `src/hooks/ralph-loop/continuation-prompt-injector.ts`

```typescript
async function injectContinuationPrompt(ctx, options) {
  // 1. Get agent/model/tools from source session
  // 2. Inherit tools from parent session
  // 3. Call promptAsync with inherited context
  await ctx.client.session.promptAsync({
    path: { id: options.sessionID },
    body: {
      agent, model, tools,
      parts: [createInternalAgentTextPart(options.prompt)],
    },
    query: { directory: options.directory },
  })
}
```

- Preserves agent, model, and tool permissions from source session
- Uses `promptAsync` for injection

### Clever Solutions

1. **Completion Tag Regex:** `<promise>.*?</promise>` with `s` flag (dotall) matches across lines
2. **Timestamp Filtering:** Only considers messages after loop started, ignoring historical context
3. **Message Scoping:** Uses `message_count_at_start` to scope search to loop-relevant messages
4. **Session Hierarchy:** Child sessions inherit context but track separately
5. **Two-Phase Ultrawork:** Separates "done" from "verified" with Oracle as final arbiter
6. **Tool Inheritance:** Continuation preserves tool permissions from parent session
7. **Transcript Fallback:** If API fails, falls back to direct transcript file reading

### Technical Debt / Shortcuts

- Hardcoded 5-second API timeout may be too short for large contexts
- State file polling (no event-driven updates)
- No graceful degradation if Oracle agent unavailable
- Max iterations default (100) may be too high for some use cases
- File-based state not atomic (potential race conditions in concurrent access)

### Key Constants

**File:** `src/hooks/ralph-loop/constants.ts`

```typescript
export const HOOK_NAME = "ralph-loop"
export const DEFAULT_STATE_FILE = ".sisyphus/ralph-loop.local.md"
export const COMPLETION_TAG_PATTERN = /<promise>(.*?)<\/promise>/is
export const DEFAULT_MAX_ITERATIONS = 100
export const DEFAULT_COMPLETION_PROMISE = "DONE"
export const ULTRAWORK_VERIFICATION_PROMISE = "VERIFIED"
```
