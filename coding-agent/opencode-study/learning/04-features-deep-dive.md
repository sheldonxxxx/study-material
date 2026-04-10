# OpenCode Feature Deep Dive

**Date:** 2026-03-26
**Repository:** `/Users/sheldon/Documents/claw/reference/opencode`
**Source:** 6 batch files covering all 18 features (05a-05f)

This document synthesizes all feature research into a coherent narrative, ordered by the priority established in the feature index: core features first (AI Agent, Multi-Provider, Built-in Agents, LSP, TUI, Desktop, Client/Server), then secondary features.

---

## PART I: CORE FEATURES

---

## Feature 1: AI Coding Agent (Core Engine)

**Location:** `packages/opencode/src/agent/agent.ts` (~414 lines)

### Architecture

The AI Coding Agent is the central orchestrator of OpenCode. It is built on the **Effect framework** for dependency injection and service composition. Every other system -- providers, tools, LSP, storage -- is wired through Effect layers.

```typescript
export namespace Agent {
  export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      // Agent state initialization with permissions, defaults, and custom agents
    })
  )
}
```

The agent uses a **Service Layer pattern** where `ServiceMap` defines the interface and `Layer.effect` provides the implementation. This allows the agent's runtime to be composed at startup without circular dependency issues.

### Permission System

The permission system is the gatekeeper for every tool execution. It merges defaults with user config:

```typescript
const defaults = Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: {
    "*": "ask",
    ...Object.fromEntries(whitelistedDirs.map((dir) => [dir, "allow"])),
  },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",
    "*.env.*": "ask",
    "*.env.example": "allow",
  },
})
```

Of note: `doom_loop` (preventing infinite agent loops) defaults to "ask", and `.env` files always require confirmation even when read is otherwise allowed.

### Dynamic Agent Generation

Agents can be generated dynamically from natural language descriptions using `generateObject` from the AI SDK:

```typescript
export async function generate(input: {
  description: string
  model?: { providerID: ProviderID; modelID: ModelID }
}) {
  const params = {
    temperature: 0.3,
    messages: [
      { role: "system", content: PROMPT_GENERATE },
      { role: "user", content: `Create an agent configuration based on: "${input.description}"...` }
    ],
    model: language,
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  }
  return yield* Effect.promise(() => generateObject(params).then((r) => r.object))
}
```

### Clever Solutions

1. **Plugin Hook for System Prompt**: Agents can transform system prompts via `Plugin.trigger("experimental.chat.system.transform", ...)` -- a backdoor for experimental features without core changes.
2. **Skill Directory Allowlisting**: External directory access is automatically allowed for skill directories (`Truncate.GLOB`), so plugins can safely grant access to their own code.
3. **Instance State Caching**: Uses `InstanceState.make` with `Effect.fn` for memoized state access, avoiding redundant computation.

### Technical Debt

- Direct `process.env` access in several places instead of the `Env` abstraction
- The permission merge logic is complex and hard to reason about for end users

---

## Feature 2: Multi-Provider AI Support

**Location:** `packages/opencode/src/provider/provider.ts` (~1506 lines)

### Provider Architecture

The multi-provider system uses the **Vercel AI SDK** (`ai`) as the core abstraction, with OpenCode's own thin wrapper layer on top. Provider selection is determined by a **branded type** system that prevents mixing up provider IDs and model IDs at the type level:

```typescript
const providerIdSchema = Schema.String.pipe(Schema.brand("ProviderID"))
export type ProviderID = typeof providerIdSchema.Type

export const ProviderID = providerIdSchema.pipe(
  withStatics((schema: typeof providerIdSchema) => ({
    make: (id: string) => schema.makeUnsafe(id),
    zod: z.string().pipe(z.custom<ProviderID>()),
    // Well-known providers
    opencode: schema.makeUnsafe("opencode"),
    anthropic: schema.makeUnsafe("anthropic"),
    openai: schema.makeUnsafe("openai"),
    google: schema.makeUnsafe("google"),
    // ... 20+ more
  })),
)
```

### Bundled Provider Registry

```typescript
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "gitlab-ai-provider": createGitLab,
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
}
```

### Custom Loaders for Provider-Specific Logic

Some providers need special handling beyond what the AI SDK provides:

**Anthropic** -- Sets beta headers for interleaved thinking and fine-grained tool streaming:
```typescript
async anthropic() {
  return {
    autoload: false,
    options: {
      headers: {
        "anthropic-beta": "interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
      },
    },
  }
}
```

**AWS Bedrock** -- Complex region prefixing for cross-region inference. Nova and Claude models get region prefixes like `us.claude-3-5-sonnet` when used in US regions, but only when not using GovCloud.

**GitHub Copilot** -- Dispatches to different Copilot APIs (chat vs. responses) based on model ID:
```typescript
"github-copilot": async () => {
  return {
    autoload: false,
    async getModel(sdk, modelID) {
      if (useLanguageModel(sdk)) return sdk.languageModel(modelID)
      return shouldUseCopilotResponsesApi(modelID) ? sdk.responses(modelID) : sdk.chat(modelID)
    },
  }
}
```

### Cross-Region Inference for AWS Bedrock

The Bedrock region prefixing logic handles three region families (us, eu, ap) with special cases for Australia and Tokyo:

```typescript
switch (regionPrefix) {
  case "us": {
    const modelRequiresPrefix = ["nova-micro", "nova-lite", "nova-pro", "claude", "deepseek"].some(...)
    if (modelRequiresPrefix && !isGovCloud) {
      modelID = `${regionPrefix}.${modelID}`  // e.g., "us.claude-3-5-sonnet"
    }
    break
  }
  // ... eu and ap with their own special cases
}
```

### Model Discovery and Caching

Remote model lists are fetched from `models.dev` with a bundled snapshot fallback:

```typescript
export namespace ModelsDev {
  export const Data = lazy(async () => {
    const result = await Filesystem.readJson(Flag.OPENCODE_MODELS_PATH ?? filepath).catch(() => {})
    if (result) return result
    const snapshot = await import("./models-snapshot.js").then((m) => m.snapshot).catch(() => undefined)
    if (snapshot) return snapshot
    if (Flag.OPENCODE_DISABLE_MODELS_FETCH) return {}
    const json = await fetch(`${url()}/api.json`).then((x) => x.text())
    return JSON.parse(json)
  })
}
```

SDK instances are cached by a hash of provider ID, npm package, and options to avoid redundant initialization.

### Provider-Specific Message Transformation

**File:** `packages/opencode/src/provider/transform.ts`

Handles provider-specific quirks:
- **Anthropic**: Filters empty content, sanitizes tool call IDs
- **Mistral**: Requires exactly 9-character alphanumeric tool call IDs
- **Interleaved reasoning**: Normalizes `reasoning_content` / `reasoning_details` fields

### Error Handling

**File:** `packages/opencode/src/provider/error.ts` (~195 lines)

Comprehensive error parsing with overflow detection across 20+ patterns:

```typescript
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,                    // Anthropic
  /input is too long for requested model/i,  // Amazon Bedrock
  /exceeds the context window/i,             // OpenAI
  /input token count.*exceeds the maximum/i,// Google
  // ... 20+ more
]

export function parseAPICallError(input): ParsedAPICallError {
  if (isOverflow(m) || statusCode === 413) {
    return { type: "context_overflow", message: m }
  }
  return { type: "api_error", isRetryable: ..., statusCode: ... }
}
```

### Technical Debt

1. **TODO comment with profanity**: `// @ts-ignore (TODO: kill this code so we dont have to maintain it)` -- `@ai-sdk/github-copilot` is marked for removal
2. **Complex region prefix logic**: 20+ lines of switch/case for 3 regions with different model requirements -- hard to maintain
3. **Direct `process.env` access**: Several places use `process.env` directly instead of the `Env` abstraction

---

## Feature 3: Built-in Agent System

**Location:** `packages/opencode/src/agent/agent.ts` (lines 105-232)

### The Three Primary Agents

| Agent | Mode | Purpose | Key Permission Difference |
|-------|------|---------|--------------------------|
| **build** | primary | Default agent, full tool access | `question: "allow"`, `plan_enter: "allow"` |
| **plan** | primary | Read-only analysis | All edit tools denied, only `*.md` edits allowed |
| **general** | subagent | Multi-step research | `todowrite: "deny"` (can't write todo items) |

**Build Agent:**
```typescript
build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_enter: "allow",
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

**Plan Agent:**
```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_exit: "allow",
      edit: { "*": "deny", "*.md": "allow" },  // Only markdown edits allowed
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

### Specialized Subagents

| Agent | Mode | Purpose |
|-------|------|---------|
| **explore** | subagent | Fast codebase search (grep, glob, list, web search only) |
| **compaction** | primary (hidden) | Context compaction |
| **title** | primary (hidden) | Generate session titles |
| **summary** | primary (hidden) | Summarize conversations |

**Explore Agent** -- Highly restricted, only allows read/search tools:
```typescript
explore: {
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      "*": "deny",
      grep: "allow",
      glob: "allow",
      list: "allow",
      bash: "allow",
      webfetch: "allow",
      websearch: "allow",
      codesearch: "allow",
      read: "allow",
    }),
  ),
  mode: "subagent",
  native: true,
}
```

### Agent Prompt Templates

Located at `packages/opencode/src/agent/prompt/`:
- `compaction.txt` -- Context compaction agent
- `explore.txt` -- Codebase exploration
- `summary.txt` -- Conversation summarization
- `title.txt` -- Session titling

### Agent Selection Logic

```typescript
const defaultAgent = Effect.fnUntraced(function* () {
  const c = yield* config()
  if (c.default_agent) {
    const agent = agents[c.default_agent]
    if (!agent) throw new Error(`default agent "${c.default_agent}" not found`)
    if (agent.mode === "subagent") throw new Error(`default agent "${c.default_agent}" is a subagent`)
    if (agent.hidden === true) throw new Error(`default agent "${c.default_agent}" is hidden`)
    return agent.name
  }
  const visible = Object.values(agents).find((a) => a.mode !== "subagent" && a.hidden !== true)
  if (!visible) throw new Error("no primary visible agent found")
  return visible.name
})
```

Users can set a default agent via `opencode.json` config. Subagents and hidden agents cannot be the default.

### Dynamic Agent Customization via Config

Users can override or extend agents via `opencode.json`:

```typescript
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) {
    delete agents[key]
    continue
  }
  item.model = Provider.parseModel(value.model)
  item.variant = value.variant ?? item.variant
  item.prompt = value.prompt ?? item.prompt
  item.temperature = value.temperature ?? item.temperature
  item.mode = value.mode ?? item.mode
  item.permission = Permission.merge(item.permission, Permission.fromConfig(value.permission ?? {}))
}
```

### Clever Solutions

1. **Tab-based Agent Cycling**: Users switch between agents with Tab key -- a natural, discoverable interaction
2. **Mode-based Agent Classification**: `primary` vs `subagent` vs `hidden` modes cleanly separate UI agents from background ones
3. **Permission Inheritance**: User config permissions are merged with defaults rather than replacing them, preserving safety defaults

---

## Feature 4: Language Server Protocol (LSP) Support

**Location:** `packages/opencode/src/lsp/` (server.ts ~2093 lines, index.ts ~12KB)

### Architecture

The LSP system uses **vscode-jsonrpc** for protocol communication. LSP servers are spawned as child processes and communicate over stdin/stdout:

```typescript
import { createMessageConnection, StreamMessageReader, StreamMessageWriter } from "vscode-jsonrpc/node"

const connection = createMessageConnection(
  new StreamMessageReader(input.server.process.stdout as any),
  new StreamMessageWriter(input.server.process.stdin as any),
)
```

### Supported LSP Servers (20 total)

| Server | Extensions | Auto-Install |
|--------|------------|--------------|
| TypeScript | .ts, .tsx, .js, .jsx, .mjs, .cjs, .mts, .cts | No |
| Deno | .ts, .tsx, .js, .jsx, .mjs | No |
| Vue | .vue | Yes (via `@vue/language-server`) |
| ESLint | .ts, .tsx, .js, .jsx, .vue | Yes (from vscode-eslint) |
| Oxlint | .ts, .tsx, .js, .jsx, .vue, .astro, .svelte | No |
| Biome | .ts, .tsx, .js, .jsx, .json, .css, .graphql, .html | No |
| Go (gopls) | .go | Yes (via `go install`) |
| Ruby (rubocop) | .rb, .rake, .gemspec, .ru | Yes (via `gem install`) |
| Python (pyright) | .py, .pyi | Yes (via `pyright-langserver`) |
| Python (ty) | .py, .pyi | No (experimental) |
| Elixir (elixir-ls) | .ex, .exs | Yes (from GitHub) |
| Zig (zls) | .zig, .zon | Yes (from GitHub releases) |
| C# (csharp-ls) | .cs | Yes (via `dotnet tool`) |
| F# (fsautocomplete) | .fs, .fsi, .fsx, .fsscript | Yes (via `dotnet tool`) |
| Swift (sourcekit-lsp) | .swift, .objc | No |
| Rust (rust-analyzer) | .rs | No |
| Clangd | .c, .cpp, .h, .hpp | Yes (from GitHub releases) |
| Svelte | .svelte | Yes (via `svelte-language-server`) |
| Astro | .astro | Yes (via `@astrojs/language-server`) |
| Java (jdtls) | .java | Yes (from Eclipse downloads) |

### Auto-Installation Strategy

Servers that are not found in PATH are automatically downloaded from their native distribution channels:
- **GitHub releases**: Elixir, Zig, Clangd -- downloaded directly via GitHub API
- **npm**: ESLint, Vue, Svelte, Astro -- installed via npm
- **gem**: Ruby -- installed via gem
- **go install**: Go -- installed via go install
- **dotnet tool**: C#, F# -- installed via dotnet tool install

### Workspace Root Detection

```typescript
const NearestRoot = (includePatterns: string[], excludePatterns?: string[]): RootFunction => {
  return async (file) => {
    const files = Filesystem.up({
      targets: includePatterns,
      start: path.dirname(file),
      stop: Instance.directory,
    })
    const first = await files.next()
    if (!first.value) return Instance.directory
    return path.dirname(first.value)
  }
}
```

Walks up the directory tree from the file being accessed until it finds a project marker (e.g., `package.json`, `tsconfig.json`).

### LSP Feature Methods

- `hover()` -- textDocument/hover
- `definition()` -- textDocument/definition
- `references()` -- textDocument/references
- `implementation()` -- textDocument/implementation
- `documentSymbol()` -- textDocument/documentSymbol
- `workspaceSymbol()` -- workspace/symbol
- `diagnostics()` -- aggregated publishDiagnostics
- Call hierarchy: `prepareCallHierarchy()`, `incomingCalls()`, `outgoingCalls()`

### Experimental Feature Flag

```typescript
const filterExperimentalServers = (servers: Record<string, LSPServer.Info>) => {
  if (Flag.OPENCODE_EXPERIMENTAL_LSP_TY) {
    if (servers["pyright"]) delete servers["pyright"]
  } else {
    if (servers["ty"]) delete servers["ty"]
  }
}
```

Python users can choose between ty (experimental) and pyright via feature flag.

### Clever Solutions

1. **Lazy client spawning**: Clients are spawned only when a file is first accessed, not upfront
2. **Client reuse**: Multiple files in the same project root share a single LSP client instance
3. **Buffer debouncing**: Diagnostic updates are debounced to prevent overwhelming the UI
4. **Symbol kind filtering**: Workspace symbol results filtered to Class, Function, Method, Interface, Variable, Constant, Struct, Enum -- removes noise

### Technical Debt

1. **Blocking on spawn**: `await` blocks during server initialization (45 second timeout)
2. **No graceful degradation**: If an LSP server fails to spawn, it's marked "broken" permanently for that root
3. **Process cleanup**: Broken server processes may not be properly reaped if client creation fails after spawn

---

## Feature 5: Terminal User Interface (TUI)

**Location:** `packages/opencode/src/pty/index.ts` (398 lines)

### Architecture

The TUI uses a PTY (pseudo-terminal) system built on `bun-pty` to provide interactive terminal sessions. The PTY service is an Effect-based layer:

```typescript
export namespace Pty {
  const BUFFER_LIMIT = 1024 * 1024 * 2  // 2MB buffer limit
  const BUFFER_CHUNK = 64 * 1024         // 64KB chunks for WebSocket transfer

  type Active = {
    info: Info
    process: IPty
    buffer: string
    bufferCursor: number
    cursor: number
    subscribers: Map<unknown, Socket>
  }
}
```

### PTY Creation

```typescript
const create = Effect.fn("Pty.create")(function* (input: CreateInput) {
  const command = input.command || Shell.preferred()
  const cwd = input.cwd || state.dir

  // Plugin hook for shell environment modification
  const shellEnv = await Plugin.trigger("shell.env", { cwd }, { env: {} })

  const env = {
    ...process.env,
    ...input.env,
    ...shellEnv.env,
    TERM: "xterm-256color",
    OPENCODE_TERMINAL: "1",
  }

  const spawn = await pty()  // Lazy load bun-pty
  const proc = spawn(command, args, {
    name: "xterm-256color",
    cwd,
    env,
  })
})
```

### WebSocket Protocol

Custom WebSocket framing with a control byte prefix:

```typescript
const meta = (cursor: number) => {
  const json = JSON.stringify({ cursor })
  const bytes = encoder.encode(json)
  const out = new Uint8Array(bytes.length + 1)
  out[0] = 0  // Control frame marker (0x00)
  out.set(bytes, 1)
  return out
}
```

Terminal output is sent as raw bytes; cursor position updates use the `0x00` prefix as a control frame marker.

### Buffer Replay on Connect

New WebSocket connections can replay the terminal buffer from any cursor position:

```typescript
const connect = Effect.fn("Pty.connect")(function* (id: PtyID, ws: Socket, cursor?: number) {
  const start = session.bufferCursor
  const end = session.cursor
  const from = cursor === -1 ? end :
               typeof cursor === "number" ? Math.max(0, cursor) : 0

  // Send buffered data in chunks
  if (data) {
    for (let i = 0; i < data.length; i += BUFFER_CHUNK) {
      ws.send(data.slice(i, i + BUFFER_CHUNK))
    }
  }
  ws.send(meta(end))
})
```

### Shell Preference Detection

```typescript
export namespace Shell {
  const BLACKLIST = new Set(["fish", "nu"])

  export const acceptable = lazy(() => {
    const s = process.env.SHELL
    if (s && !BLACKLIST.has(basename(s))) return s
    return fallback()
  })
}
```

Fish and nu shells are explicitly blacklisted. No reason is documented.

### Clever Solutions

1. **Circular buffer**: Terminal output beyond 2MB is pruned from the beginning, preventing memory exhaustion
2. **Chunked WebSocket transfer**: Large sends chunked into 64KB pieces to prevent blocking the event loop
3. **Cursor tracking**: Cursor position tracked separately from buffer, enabling efficient replay of only new content
4. **Subscriber management**: WebSocket subscribers tracked with cleanup on disconnect
5. **Process tree kill**: On Unix, `kill(-pid, SIGTERM)` kills the entire process group

### Technical Debt

1. **Windows locale hack**: On Windows, LC_ALL/LC_CTYPE/LANG forced to C.UTF-8 which may cause encoding issues:
   ```typescript
   if (process.platform === "win32") {
     env.LC_ALL = "C.UTF-8"
     env.LC_CTYPE = "C.UTF-8"
     env.LANG = "C.UTF-8"
   }
   ```
2. **Fish/nu blacklisted**: No documented reason
3. **No explicit reattach**: PTY continues on disconnect; buffer replay is the only recovery mechanism

---

## Feature 6: Desktop Application (Tauri v2)

**Location:** `packages/desktop/src-tauri/src/lib.rs` (602 lines)

### Architecture

The desktop app wraps the OpenCode CLI as a **sidecar process** and provides a native window via Tauri v2. The Rust backend manages the CLI lifecycle while the frontend (SolidJS) connects to it.

**Sidecar Server Pattern** (`server.rs`):
```rust
pub fn spawn_local_server(
    app: AppHandle,
    hostname: String,
    port: u32,
    password: String,
) -> (CommandChild, HealthCheck) {
    let (child, exit) = cli::serve(&app, &hostname, port, &password);

    let health_check = HealthCheck(tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_millis(100)).await;
            if check_health(&url, Some(&password)).await {
                tracing::info!(elapsed = ?timestamp.elapsed(), "Server ready");
                return Ok(());
            }
        }
    }));

    (child, health_check)
}
```

### Single Instance

Uses `tauri_plugin_single_instance` to ensure only one desktop app runs at a time:
```rust
.plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
    if let Some(window) = app.get_webview_window(MainWindow::LABEL) {
        let _ = window.set_focus();
        let _ = window.unminimize();
    }
}))
```

### Window Management

**Loading window strategy**: Shows a dedicated loading window only if SQLite migration takes longer than 1 second, avoiding flicker for fast operations:
```rust
let loading_window = if needs_migration
    && timeout(Duration::from_secs(1), loading_task.clone()).await.is_err()
{
    let loading_window = LoadingWindow::create(&app)?;
    sleep(Duration::from_secs(1)).await;
    Some(loading_window)
} else {
    None
};
MainWindow::create(&app)?;
```

### Linux Display Backend Detection

```rust
#[cfg(target_os = "linux")]
fn configure_display_backend() -> Option<String> {
    let session = SessionEnv::capture();
    let prefer_wayland = opencode_lib::linux_display::read_wayland().unwrap_or(false);
    let decision = select_backend(&session, prefer_wayland)?;

    match decision.backend {
        Backend::X11 => {
            set_env_if_absent("WINIT_UNIX_BACKEND", "x11");
            set_env_if_absent("GDK_BACKEND", "x11");
        }
        Backend::Wayland => {
            set_env_if_absent("WINIT_UNIX_BACKEND", "wayland");
            set_env_if_absent("GDK_BACKEND", "wayland");
        }
        Backend::Auto => {
            set_env_if_absent("GDK_BACKEND", "wayland,x11");
        }
    }
}
```

### Windows Proxy Bypass

Prevents VPN/proxy from intercepting loopback connections:
```rust
const LOOPBACK: [&str; 3] = ["127.0.0.1", "localhost", "::1"];

let upsert = |key: &str| {
    let mut items = std::env::var(key).unwrap_or_default()
        .split(',')
        // ... ensure loopback addresses are in NO_PROXY
};
upsert("NO_PROXY");
upsert("no_proxy");
```

### Clever Solutions

1. **Sidecar CLI pattern**: CLI runs as a child process; desktop provides only the native window and Tauri plugins
2. **Deep link registration**: `opencode://` URL scheme registered for inter-app communication
3. **Window state persistence**: Remembers window size/position across sessions via `tauri_plugin_window_state`
4. **WSL path translation**: Automatic translation between Windows and Linux path formats when WSL is enabled

### Technical Debt

1. **macOSPrivateApi**: Uses `macOSPrivateApi: true` in tauri.conf.json requiring special Apple Developer certificates
2. **Custom Windows titlebar**: Creates a custom titlebar on Windows using a plugin that may conflict with some GPU configurations
3. **Conditional deep link registration**: Deep link registration only on Linux and debug Windows:
   ```rust
   #[cfg(any(target_os = "linux", all(debug_assertions, windows)))]
   app.deep_link().register_all().ok();
   ```

---

## Feature 7: Client/Server Architecture

**Location:** `packages/opencode/src/server/server.ts`

### Architecture

The server is built with **Hono** (lightweight web framework) on Bun. WebSocket support is provided via `hono/bun`.

**Route Structure:**
```
/project   - ProjectRoutes()
/pty       - PtyRoutes()
/config    - ConfigRoutes()
/experimental - ExperimentalRoutes()
/session   - SessionRoutes()
/permission - PermissionRoutes()
/question  - QuestionRoutes()
/provider  - ProviderRoutes()
/          - FileRoutes(), EventRoutes()
/mcp       - McpRoutes()
/tui       - TuiRoutes()
```

### mDNS Discovery

```typescript
const service = bonjour.publish({
  name: `opencode-${port}`,
  type: "http",
  host: domain ?? "opencode.local",
  port,
  txt: { path: "/" },
})
```

Automatic service discovery on local networks via mDNS/Bonjour.

### Event Bus Architecture

Uses Effect's `Stream` for pub/sub messaging:
```typescript
export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(
    def: D,
    properties: z.output<D["properties"]>,
  ) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
  readonly subscribeAll: () => Stream.Stream<Payload>
}
```

### AsyncQueue Bridge for TUI

HTTP requests are bridged to internal TUI processing via async queues:
```typescript
const request = new AsyncQueue<TuiRequest>()
const response = new AsyncQueue<any>()

export async function callTui(ctx: Context) {
  const body = await ctx.req.json()
  request.push({ path: ctx.req.path, body })
  return response.next()
}
```

### Security

- Basic auth with configurable username/password
- CORS with strict origin validation
- Directory path encoding for non-ASCII paths

### Clever Solutions

1. **Port Fallback**: Tries port 4096 first, then random port (0) if specified port is unavailable
2. **Lazy Route Initialization**: Route modules use `lazy()` wrapper to avoid circular dependencies
3. **Event Projectors**: Session events are transformed into sync-friendly formats via projectors, decoupling producers from consumers
4. **Workspace Context Middleware**: Clean per-request workspace isolation via middleware

### Technical Debt

1. **Deprecated profanity comment**: `@deprecated do not use this dumb shit` at line 571
2. **Hardcoded CSP hash extraction**: CSP hash extracted from HTML in a non-obvious way at lines 546-550

---

## PART II: SECONDARY FEATURES

---

## Feature 8: VS Code Extension

**Location:** `sdks/vscode/src/extension.ts` (~137 lines)

The VS Code extension is intentionally minimal -- it uses VS Code's built-in Terminal API to spawn the OpenCode CLI rather than implementing a custom editor integration.

```typescript
async function openTerminal() {
  const port = Math.floor(Math.random() * (65535 - 16384 + 1)) + 16384
  const terminal = vscode.window.createTerminal({
    name: TERMINAL_NAME,
    env: {
      _EXTENSION_OPENCODE_PORT: port.toString(),
      OPENCODE_CALLER: "vscode",
    },
  })
  terminal.show()
  terminal.sendText(`opencode --port ${port}`)
  // Polls for server readiness
}
```

### File Path Insertion

The extension supports inserting the active file path with optional line number selection:
```typescript
function getActiveFile() {
  const relativePath = vscode.workspace.asRelativePath(document.uri)
  let filepathWithAt = `@${relativePath}`

  const selection = activeEditor.selection
  if (!selection.isEmpty) {
    const startLine = selection.start.line + 1
    const endLine = selection.end.line + 1
    if (startLine === endLine) {
      filepathWithAt += `#L${startLine}`
    } else {
      filepathWithAt += `#L${startLine}-${endLine}`
    }
  }
  return filepathWithAt
}
```

### Technical Debt

1. **Hardcoded port range**: Random port 16384-65535, assumes localhost
2. **No real error handling**: Fetch failures silently ignored in polling loop
3. **No WebSocket support**: Relies on HTTP for communication

---

## Feature 9: SDK (`@opencode-ai/sdk`)

**Location:** `packages/sdk/js/`

### Architecture

Auto-generated from OpenAPI spec using `openapi-ts`. Two API generations coexist:
- **v1**: Original client/server factories
- **v2**: Newer, cleaner API with improved type safety

### SDK Generation Pipeline

```typescript
// 1. Generate OpenAPI spec from running server
await $`bun dev generate > ${dir}/openapi.json`.cwd(path.resolve(dir, "../../opencode"))

// 2. Generate TypeScript client with openapi-ts
await createClient({
  input: "./openapi.json",
  output: { path: "./src/v2/gen", tsConfigPath: ..., clean: true },
  plugins: [
    { name: "@hey-api/typescript", exportFromIndex: false },
    { name: "@hey-api/sdk", instance: "OpencodeClient", exportFromIndex: false },
    { name: "@hey-api/client-fetch", baseUrl: "http://localhost:4096" },
  ],
})
```

### Key Design Decision: CLI as Server Process

Rather than importing the server module directly, the SDK spawns the `opencode` CLI binary as a child process:

```typescript
export async function createOpencodeServer(options?: ServerOptions) {
  const args = [`serve`, `--hostname=${options.hostname}`, `--port=${options.port}`]
  const proc = spawn(`opencode`, args, {
    signal: options.signal,
    env: { ...process.env, OPENCODE_CONFIG_CONTENT: JSON.stringify(options.config ?? {}) },
  })

  // Parse URL from stdout
  const url = await new Promise<string>((resolve, reject) => {
    proc.stdout?.on("data", (chunk) => {
      const line = chunk.toString()
      if (line.startsWith("opencode server listening")) {
        const match = line.match(/on\s+(https?:\/\/[^\s]+)/)
        resolve(match[1])
      }
    })
  })

  return { url, close() { proc.kill() } }
}
```

This provides process isolation and matches how end users deploy OpenCode.

### Clever Solutions

1. **Config via Environment**: Configuration passed through `OPENCODE_CONFIG_CONTENT` env var containing JSON
2. **Timeout Disabled**: Custom fetch wrapper removes timeout for long-running AI operations
3. **Directory Encoding**: Non-ASCII paths handled with `encodeURIComponent()`
4. **Base URL Resolution**: Waits for "opencode server listening on..." stdout message to determine server URL

---

## Feature 10: Web Dashboard (SolidStart App)

**Location:** `packages/app/src/pages/session.tsx` (61,487 bytes -- the largest single file in the codebase)

### Architecture

- **Framework**: SolidStart (SolidJS meta-framework with file-based routing)
- **Styling**: Tailwind CSS
- **State**: SolidJS stores + TanStack Query
- **Terminal**: ghostty-web (browser-based terminal emulator)

### Entry Point

```typescript
const getCurrentUrl = () => {
  if (location.hostname.includes("opencode.ai")) return "http://localhost:4096"
  if (import.meta.env.DEV)
    return `http://${import.meta.env.VITE_OPENCODE_SERVER_HOST ?? "localhost"}:${import.meta.env.VITE_OPENCODE_SERVER_PORT ?? "4096"}`
  return location.origin
}
```

In production, the web app connects to the local OpenCode server at port 4096. In cloud deployments, it uses its own origin as the server.

### Session History Loading

```typescript
const turnInit = 10
const turnBatch = 8
const turnScrollThreshold = 200
const turnPrefetchBuffer = 16
```

Initial render loads 10 turns. User scrolls trigger batched loads of 8 more with a prefetch buffer of 16 turns.

### Clever Solutions

1. **Ghostty Web Terminal**: Real terminal experience in browser
2. **History Window Optimization**: Bounded initial paint with smart prefetching
3. **Multiple Server Support**: Can connect to multiple OpenCode servers and switch between them

### Technical Debt

1. **Monolithic session component**: 61KB+ session page component that could benefit from splitting
2. **Client-heavy without SSR**: No server-side rendering for initial load
3. **Duplicated history window logic**: Some logic paths appear in multiple places

---

## Feature 11: Documentation Website

**Location:** `packages/web/` -- Astro Starlight site

Built with Astro Starlight at `opencode.ai/docs`. No deep-dive details available in batch files.

---

## Feature 12: Enterprise Features

**Location:** `packages/enterprise/`, `packages/opencode/src/auth/`, `infra/enterprise.ts`

### Architecture

Enterprise features use Cloudflare Workers (R2, KV) for data residency and authentication:
- **Auth**: Cloudflare KV for session storage
- **Storage**: Cloudflare R2 or AWS S3 as storage backends
- **Teams App**: SolidStart on Cloudflare

### Multi-Storage Adapter

Single interface supporting both S3 and R2:
```typescript
function s3(): Adapter {
  const client = new AwsClient({
    region: process.env.OPENCODE_STORAGE_REGION || "us-east-1",
    accessKeyId: process.env.OPENCODE_STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.OPENCODE_STORAGE_SECRET_ACCESS_KEY!,
  })
  return createAdapter(client, `https://s3.${region}.amazonaws.com`, bucket)
}

function r2() {
  const accountId = process.env.OPENCODE_STORAGE_ACCOUNT_ID!
  const client = new AwsClient({
    accessKeyId: process.env.OPENCODE_STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.OPENCODE_STORAGE_SECRET_ACCESS_KEY!,
  })
  return createAdapter(client, `https://${accountId}.r2.cloudflarestorage.com`, bucket)
}
```

### Auth Subsystem

Three auth types modeled with Effect-compatible schemas:
```typescript
export class Oauth extends Schema.Class<Oauth>("OAuth")({
  type: Schema.Literal("oauth"),
  refresh: Schema.String,
  access: Schema.String,
  expires: Schema.Number,
  accountId: Schema.optional(Schema.String),
  enterpriseUrl: Schema.optional(Schema.String),
}) {}

export class Api extends Schema.Class<Api>("ApiAuth")({
  type: Schema.Literal("api"),
  key: Schema.String,
}) {}

export class WellKnown extends Schema.Class<WellKnown>("WellKnownAuth")({
  type: Schema.Literal("wellknown"),
  key: Schema.String,
  token: Schema.String,
}) {}
```

### Auth Plugins

**CodexAuthPlugin** (`packages/opencode/src/plugin/codex.ts`):
- OAuth with PKCE for ChatGPT Pro/Plus
- Built-in OAuth callback server on port 1455
- Two auth methods: browser-based and headless device flow
- Automatic token refresh

**CopilotAuthPlugin** (`packages/opencode/src/plugin/copilot.ts`):
- GitHub Copilot authentication via device code OAuth
- Enterprise GitHub (GHE) support via domain normalization

### Technical Debt

1. **Minimal enterprise app routes**: Routes are placeholder ("Hello World")
2. **Storage path encoding**: Uses `.json` extension on all keys, limiting flexibility

---

## Feature 13: Slack Integration

**Location:** `packages/slack/src/index.ts`

### Architecture Pattern: Per-Thread Session

Each Slack thread maps to a separate OpenCode session:

```typescript
const sessions = new Map<string, { client: any; server: any; sessionId: string; channel: string; thread: string }>()

app.message(async ({ message, say }) => {
  const channel = message.channel
  const thread = (message as any).thread_ts || message.ts
  const sessionKey = `${channel}-${thread}`

  let session = sessions.get(sessionKey)
  if (!session) {
    const createResult = await client.session.create({
      body: { title: `Slack thread ${thread}` },
    })
    session = { client, server, sessionId: createResult.data.id, channel, thread }
    sessions.set(sessionKey, session)
  }

  const result = await session.client.session.prompt({
    path: { id: session.sessionId },
    body: { parts: [{ type: "text", text: message.text }] },
  })
})
```

### Event Streaming

Subscribes to OpenCode server events and forwards tool completion notifications to Slack:
```typescript
const events = await opencode.client.event.subscribe()
for await (const event of events.stream) {
  if (event.type === "message.part.updated") {
    const part = event.properties.part
    if (part.type === "tool" && part.state.status === "completed") {
      const toolMessage = `*${part.tool}* - ${part.state.title}`
      await app.client.chat.postMessage({ channel, thread_ts: thread, text: toolMessage })
    }
  }
}
```

### Clever Solutions

1. **Socket Mode**: Uses Slack Socket Mode for webhook-free event handling
2. **Thread Affinity**: Conversation context preserved per thread
3. **Share URL Generation**: New sessions generate shareable URLs for viewing in the web UI

### Technical Debt

1. **No session cleanup**: Sessions accumulate in the Map indefinitely
2. **No error recovery**: Session creation failures not explicitly handled

---

## Feature 14: Plugin System

**Location:** `packages/plugin/`, `packages/opencode/src/plugin/`, `packages/opencode/src/mcp/`

### Hooks Interface

```typescript
export interface Hooks {
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>
  tool?: { [key: string]: ToolDefinition }
  auth?: AuthHook
  "chat.message"?: (input, output) => Promise<void>
  "chat.params"?: (input, output) => Promise<void>
  "chat.headers"?: (input, output) => Promise<void>
  "permission.ask"?: (input, output) => Promise<void>
  "command.execute.before"?: (input, output) => Promise<void>
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  "shell.env"?: (input, output) => Promise<void>
  "experimental.session.compacting"?: (input, output) => Promise<void>
  "experimental.text.complete"?: (input, output) => Promise<void>
}
```

### Plugin Loading

1. Internal plugins (Codex, Copilot, GitLab, Poe) loaded first
2. User plugins from config installed via `BunProc.install()`
3. Plugins deduplicated via `seen` Set to prevent double-initialization
4. Deprecated npm package names silently skipped

### MCP Implementation

The MCP client supports multiple transport types:
- **Remote**: StreamableHTTP (primary), SSE with OAuth support
- **Local**: StdioClientTransport for spawning local processes

Tool name collision prevention:
```typescript
result[sanitizedClientName + "_" + sanitizedToolName] = convertMcpTool(...)
```

### Clever Solutions

1. **Transport Fallback**: Multiple transports tried in sequence with graceful degradation
2. **OAuth State Machine**: Full OAuth 2.0 with PKCE and CSRF protection
3. **Process Tree Cleanup**: Proper subprocess termination on disconnect via recursive PID descent

### Technical Debt

1. **Fixed OAuth callback port**: Uses hardcoded port 19876
2. **No plugin unload**: Plugins loaded but never unloaded (memory leak potential)
3. **Credential invalidation**: `invalidateCredentials()` uses `await` on potentially async operation without proper error handling

---

## Feature 15: Database & Storage

**Location:** `packages/opencode/src/storage/` (db.ts, schema.ts, storage.ts, json-migration.ts)

### Dual Storage Architecture

OpenCode maintains two parallel storage systems:
1. **SQLite** via Drizzle ORM -- for persistent structured data
2. **JSON file storage** -- legacy storage with an active migration path to SQLite

### SQLite Performance Pragmas

```typescript
db.run("PRAGMA journal_mode = WAL")
db.run("PRAGMA synchronous = NORMAL")
db.run("PRAGMA busy_timeout = 5000")
db.run("PRAGMA cache_size = -64000")  // 64MB
db.run("PRAGMA foreign_keys = ON")
db.run("PRAGMA wal_checkpoint(PASSIVE)")
```

### JSON Storage

File-based storage with proper locking:
```typescript
export async function read<T>(key: string[]) {
  const target = path.join(dir, ...key) + ".json"
  return withErrorHandling(async () => {
    using _ = await Lock.read(target)  // File locking
    const result = await Filesystem.readJson<T>(target)
    return result as T
  })
}
```

### JSON to SQLite Migration

Batch processing with 1000-item batches and FK-aware ordering (projects first, then sessions, then messages, etc.):
```typescript
const batchSize = 1000
sqlite.exec("BEGIN TRANSACTION")
// Process projects, sessions, messages, parts, todos, permissions, shares
// FK-aware: skip orphans whose parents don't exist
sqlite.exec("COMMIT")
```

### Channel-Based Database Isolation

Different release channels get separate DB files:
```typescript
if (["latest", "beta"].includes(channel) || Flag.OPENCODE_DISABLE_CHANNEL_DB)
  return path.join(Global.Path.data, "opencode.db")
const safe = channel.replace(/[^a-zA-Z0-9._-]/g, "-")
return path.join(Global.Path.data, `opencode-${safe}.db`)
```

### Context-Based Transaction Propagation

Uses Effect's Context system for dependency injection into SQLite transactions:
```typescript
const ctx = Context.create<{
  tx: TxOrDb
  effects: (() => void | Promise<void>)[]
}>("database")

export function use<T>(callback: (trx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)
  } catch (err) {
    if (err instanceof Context.NotFound) {
      const effects: (() => void | Promise<void>)[] = []
      const result = ctx.provide({ effects, tx: Client() }, () => callback(Client()))
      for (const effect of effects) effect()
      return result
    }
    throw err
  }
}
```

### Technical Debt

1. **No schema migrations for SQLite**: Schema defined via `.sql` files, not managed migrations
2. **JSON storage not cleaned up**: JSON files remain after migration to SQLite
3. **No vacuum scheduling**: No compaction or vacuum for SQLite

---

## Feature 16: Git Integration

**Location:** `packages/opencode/src/git/index.ts` (309 lines), `packages/opencode/src/worktree/index.ts` (600 lines)

### Git Service Layer

Built as an Effect service layer using `effect/unstable/process` for spawning git:

```typescript
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Git") {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const spawner = yield* ChildProcessSpawner.ChildProcessSpawner
    // ...
  }),
)
```

### Global Git Configuration

Pre-configured safe defaults applied to every command:
```typescript
const cfg = [
  "--no-optional-locks",
  "-c", "core.autocrlf=false",
  "-c", "core.fsmonitor=false",
  "-c", "core.longpaths=true",
  "-c", "core.symlinks=true",
  "-c", "core.quotepath=false",
] as const
```

### Worktree Management

The worktree system extends native git worktrees with project sandbox isolation. Key methods:
- `makeWorktreeInfo()` -- Generate unique name/branch/directory
- `create()` -- Setup + boot in one operation
- `remove()` -- Clean removal with branch cleanup
- `reset()` -- Hard reset + clean + submodule update + restart scripts

### Platform-Aware Path Handling

```typescript
const canonical = Effect.fnUntraced(function* (input: string) {
  const abs = pathSvc.resolve(input)
  const real = yield* fsys.realPath(abs).pipe(Effect.catch(() => Effect.succeed(abs)))
  const normalized = pathSvc.normalize(real)
  return process.platform === "win32" ? normalized.toLowerCase() : normalized
})
```

Windows paths are lowercased for consistent comparison.

### VCS Service Layer

**File:** `packages/opencode/src/project/vcs.ts` (237 lines)

Subscribes to HEAD file changes to detect branch switches and publish `Event.BranchUpdated`.

### GitHub Integration (CLI)

**File:** `packages/opencode/src/cli/cmd/github.ts` (1647 lines)

Comprehensive GitHub App integration that installs via browser flow, creates workflow files, and handles PR comments, issue comments, and reviews.

---

## Feature 17: Cloud Infrastructure (SST)

**Location:** `infra/` (sst.config.ts, app.ts, console.ts, enterprise.ts, stage.ts)

### Configuration

Uses SST (Serverless Stack) with Cloudflare as the provider:
```typescript
export default $config({
  app(input) {
    return {
      name: "opencode",
      removal: input?.stage === "production" ? "retain" : "remove",
      protect: ["production"].includes(input?.stage),
      home: "cloudflare",
      providers: {
        stripe: { apiKey: process.env.STRIPE_SECRET_KEY! },
        planetscale: "0.4.1",
      },
    }
  },
})
```

### Multi-Stage Domain Strategy

| Stage | Domain | Database Branch |
|-------|--------|----------------|
| production | opencode.ai | production |
| dev | dev.opencode.ai | dev |
| others | {stage}.dev.opencode.ai | {stage} (branched from production) |

### Database Branching Pattern

Development stages branch from production for realistic testing:
```typescript
const branch =
  $app.stage === "production"
    ? planetscale.getBranchOutput({ name: "production", ... })
    : new planetscale.Branch("DatabaseBranch", {
        parentBranch: "production",  // Dev branches from production
      })
```

### SyncServer Durable Object

The API Worker includes a `SyncServer` Durable Object namespace for real-time sync:
```typescript
transform: {
  worker: (args) => {
    args.bindings = $resolve(args.bindings).apply((bindings) => [
      ...bindings,
      { name: "SYNC_SERVER", type: "durable_object_namespace", className: "SyncServer" },
    ])
  },
}
```

### Clever Solutions

1. **Dual KV Storage**: Separate buckets for migration (`bucket`, `bucketNew`) enable zero-downtime data migrations
2. **Database Branching**: Dev stages get production data for realistic testing without affecting production
3. **Durable Objects for Sync**: SyncServer as DO namespace instead of an external service

---

## Feature 18: Container Images

**Location:** `packages/containers/`

### Image Stack

```
base (Ubuntu 24.04 + common tools)
  |
  +-- bun-node (base + Node.js 24 + Bun 1.3.11)
        |
        +-- rust (bun-node + Rust stable)
              |
              +-- tauri-linux (rust + Tauri Linux build deps)
  +-- publish (bun-node + Docker CLI + AUR tooling)
```

### Multi-Architecture Builds

Docker Buildx builds for both `linux/amd64` and `linux/arm64`:
```bash
docker buildx build --platform linux/amd64,linux/arm64 -f ${file} -t ${image} --push .
```

### Registry

All images published to: `ghcr.io/anomalyco/build/{image}:{tag}`

### Version Pinning

Bun-Node image pins exact versions:
```dockerfile
ARG NODE_VERSION=24.4.0
ARG BUN_VERSION=1.3.11
```

### Technical Debt

1. **No caching strategy documented**: Build scripts don't explicitly use Docker layer caching
2. **Publish image mystery**: `publish` directory exists but Dockerfile not shown
3. **Date-based versioning**: Tag `24.04` suggests monthly release cadence

---

## Cross-Cutting Themes

### Effect Framework as Backbone

OpenCode uses the **Effect** framework (`effect`) as its dependency injection and service composition backbone across nearly every subsystem: agent, provider, git, storage, MCP, plugin, bus. The `ServiceMap`, `Layer`, `Context`, and `Effect.gen` patterns appear everywhere. This is a deliberate architectural choice that enables modular, testable code with clean dependency graphs.

### AI SDK as AI Abstraction

The **Vercel AI SDK** (`ai`) is the core AI abstraction. OpenCode wraps it with custom loaders for provider-specific behaviors (Anthropic beta headers, Bedrock region prefixing, GitHub Copilot API dispatching). The bundled provider registry in `provider.ts` documents 20+ provider packages.

### WebSocket Everywhere

WebSocket is the dominant transport for real-time features: PTY terminal output, server-sent events for session updates, MCP streaming. The PTY uses a custom WebSocket framing protocol with a control byte prefix for cursor metadata.

### Build System: Turborepo + Bun

The monorepo uses **Turborepo** for build orchestration and **Bun** as the JavaScript runtime (not Node.js). The `bun-pty` library provides PTY functionality that would typically require native Node.js modules.

### Technical Debt Patterns

| Pattern | Examples |
|---------|----------|
| Direct `process.env` access | Provider code, shell env |
| Hardcoded ports/paths | VS Code extension (port range), MCP OAuth (19876), OAuth callback (1455) |
| Blocking operations | LSP server spawn (45s timeout), LSP client creation |
| Missing cleanup | Slack session Map accumulation, plugin unload |
| Monolithic files | Session page (61KB), LSP server (2093 lines) |
| Profanity in comments | `@deprecated do not use this dumb shit` |
| Profanity TODO | `TODO: kill this code so we dont have to maintain it` |

### Clever Solution Patterns

1. **Circular buffer for unbounded output**: PTY buffer automatically prunes oldest data
2. **Cursor-based replay**: WebSocket clients reconnect and replay only from their cursor position
3. **Effect Context for transaction scope**: Database transactions injected via Effect Context
4. **Channel-based DB isolation**: Different release channels use different DB files
5. **CLI as sidecar**: Desktop app spawns CLI rather than importing server modules
6. **Dual KV for migrations**: Old + new storage buckets enable zero-downtime data migrations

---

## File Count Summary

| Feature | Primary Location | Files |
|---------|-----------------|-------|
| AI Agent | `packages/opencode/src/agent/agent.ts` | 1 main + prompts |
| Multi-Provider | `packages/opencode/src/provider/` | provider.ts, schema.ts, models.ts, error.ts, transform.ts |
| Built-in Agents | `packages/opencode/src/agent/agent.ts` | (shared with AI Agent) |
| LSP | `packages/opencode/src/lsp/` | index.ts, server.ts, client.ts, language.ts, launch.ts |
| TUI | `packages/opencode/src/pty/` | index.ts, client.ts, bus/index.ts, shell/shell.ts |
| Desktop | `packages/desktop/src-tauri/src/` | lib.rs, server.rs, cli.rs, windows.rs, linux_display.rs |
| Client/Server | `packages/opencode/src/server/` | server.ts, projectors.ts, mdns.ts, routes/tui.ts |
| VS Code | `sdks/vscode/src/` | extension.ts, package.json |
| SDK | `packages/sdk/js/` | v1 + v2 client/server factories, gen/ auto-generated |
| Web Dashboard | `packages/app/src/` | session.tsx (61KB), context/, pages/ |
| Enterprise | `packages/enterprise/`, `infra/` | storage.ts, auth/, enterprise.ts |
| Slack | `packages/slack/src/` | index.ts |
| Plugin | `packages/plugin/`, `packages/opencode/src/plugin/`, `packages/opencode/src/mcp/` | index.ts, tool.ts, oauth-provider.ts |
| Database | `packages/opencode/src/storage/` | db.ts, schema.ts, storage.ts, json-migration.ts |
| Git | `packages/opencode/src/git/`, `packages/opencode/src/worktree/` | index.ts, vcs.ts, github.ts, pr.ts |
| Cloud | `infra/` | sst.config.ts, app.ts, console.ts, enterprise.ts, stage.ts |
| Containers | `packages/containers/` | base/, bun-node/, rust/, tauri-linux/, publish/ |
