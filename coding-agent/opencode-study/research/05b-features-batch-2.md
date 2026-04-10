# Feature Batch 2: LSP, TUI, Desktop (Features 4-6)

## Feature 4: Language Server Protocol (LSP) Support

### Overview
OpenCode provides out-of-the-box LSP support for code intelligence including hover, goto definition, completions, and diagnostics. The LSP implementation uses vscode-jsonrpc for LSP communication.

### Core Implementation Files
- `packages/opencode/src/lsp/index.ts` - Main LSP orchestration (12KB)
- `packages/opencode/src/lsp/server.ts` - LSP server definitions (2093 lines)
- `packages/opencode/src/lsp/client.ts` - LSP client implementation
- `packages/opencode/src/lsp/language.ts` - Language extension mappings
- `packages/opencode/src/lsp/launch.ts` - Process spawning for LSP servers

### Architecture

**LSP Client** (`client.ts`):
```typescript
// Uses vscode-jsonrpc for protocol communication
import { createMessageConnection, StreamMessageReader, StreamMessageWriter } from "vscode-jsonrpc/node"

// Creates bidirectional message connection to LSP server
const connection = createMessageConnection(
  new StreamMessageReader(input.server.process.stdout as any),
  new StreamMessageWriter(input.server.process.stdin as any),
)
```

**LSP Server Discovery** (`server.ts`):
- Dynamically spawns LSP servers based on file extension
- Auto-downloads and installs LSP servers when needed
- Supports multiple concurrent LSP servers per project

**Supported LSP Servers** (17 total):
| Server | Extensions | Auto-Install |
|--------|------------|--------------|
| Deno | .ts, .tsx, .js, .jsx, .mjs | No |
| TypeScript | .ts, .tsx, .js, .jsx, .mjs, .cjs, .mts, .cts | No |
| Vue | .vue | Yes (via `@vue/language-server`) |
| ESLint | .ts, .tsx, .js, .jsx, .vue | Yes (from vscode-eslint) |
| Oxlint | .ts, .tsx, .js, .jsx, .vue, .astro, .svelte | No |
| Biome | .ts, .tsx, .js, .jsx, .json, .css, .graphql, .html | No |
| Go (gopls) | .go | Yes (via `go install`) |
| Ruby (rubocop) | .rb, .rake, .gemspec, .ru | Yes (via `gem install`) |
| Python (ty) | .py, .pyi | No (experimental) |
| Python (pyright) | .py, .pyi | Yes (via `pyright-langserver`) |
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

### Key Implementation Details

**File-based LSP Server Selection** (`index.ts`):
```typescript
async function getClients(file: string) {
  // Only spawns LSP clients for files within the instance directory
  if (!Instance.containsPath(file)) {
    return []
  }

  const extension = path.parse(file).ext || file

  for (const server of Object.values(s.servers)) {
    // Check if server handles this extension
    if (server.extensions.length && !server.extensions.includes(extension)) continue

    // Find project root for this server
    const root = await server.root(file)
    if (!root) continue

    // Reuse existing client or spawn new one
    const match = s.clients.find((x) => x.root === root && x.serverID === server.id)
    if (match) {
      result.push(match)
      continue
    }

    // Spawn new client with lazy initialization
    const client = await LSPClient.create({ serverID: server.id, server: handle, root })
    s.clients.push(client)
  }
}
```

**Workspace Root Detection** (`server.ts`):
```typescript
const NearestRoot = (includePatterns: string[], excludePatterns?: string[]): RootFunction => {
  return async (file) => {
    // Walk up directory tree looking for project markers
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

**LSP Feature Methods** (`index.ts`):
- `hover()` - textDocument/hover
- `definition()` - textDocument/definition
- `references()` - textDocument/references
- `implementation()` - textDocument/implementation
- `documentSymbol()` - textDocument/documentSymbol
- `workspaceSymbol()` - workspace/symbol
- `prepareCallHierarchy()` - textDocument/prepareCallHierarchy
- `incomingCalls()` - callHierarchy/incomingCalls
- `outgoingCalls()` - callHierarchy/outgoingCalls
- `diagnostics()` - aggregated publishDiagnostics

**Experimental Feature Flag**:
```typescript
const filterExperimentalServers = (servers: Record<string, LSPServer.Info>) => {
  if (Flag.OPENCODE_EXPERIMENTAL_LSP_TY) {
    // Disable pyright when ty is enabled
    if (servers["pyright"]) delete servers["pyright"]
  } else {
    // Disable ty when pyright is enabled
    if (servers["ty"]) delete servers["ty"]
  }
}
```

### Clever Solutions

1. **Auto-installation**: LSP servers that aren't found in PATH are automatically downloaded and installed from their respective sources (GitHub releases, npm, gem, go install, etc.)

2. **Lazy client spawning**: Clients are spawned on-demand when a file is first accessed, not upfront

3. **Client reuse**: Multiple files in the same project root share a single LSP client instance

4. **Buffer debouncing**: Diagnostic updates are debounced to prevent overwhelming the UI

5. **Feature filtering**: Workspace symbol results are filtered to show only useful symbol kinds (Class, Function, Method, Interface, Variable, Constant, Struct, Enum)

### Technical Debt / Shortcuts

1. **Blocking on spawn**: The LSP client creation uses `await` which blocks during server initialization (45 second timeout)

2. **No graceful degradation**: If an LSP server fails to spawn, it marks the server as "broken" permanently for that root

3. **Process cleanup**: Broken server processes may not be properly reaped if the client creation fails after spawn

---

## Feature 5: Terminal User Interface (TUI)

### Overview
The TUI provides a terminal-based UI built by neovim users. It uses a PTY (pseudo-terminal) system built on `bun-pty` to provide interactive terminal sessions that can be accessed remotely via WebSocket.

### Core Implementation Files
- `packages/opencode/src/pty/index.ts` - Main PTY service (398 lines)
- `packages/opencode/src/pty/client.ts` - PTY WebSocket client
- `packages/opencode/src/bus/index.ts` - Event bus for UI updates
- `packages/opencode/src/shell/shell.ts` - Shell preference detection

### Architecture

**PTY Service** (`index.ts`):
The PTY system is built using the Effect framework for dependency injection and service management:

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

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      // Service implementation with create, list, get, update, remove, resize, write, connect
    }),
  )
}
```

**Shell Integration** (`shell.ts`):
```typescript
export namespace Shell {
  const BLACKLIST = new Set(["fish", "nu"])

  export const preferred = lazy(() => {
    const s = process.env.SHELL
    if (s) return s
    return fallback()
  })

  export const acceptable = lazy(() => {
    // Returns preferred shell, excluding blacklisted shells (fish, nu)
    const s = process.env.SHELL
    if (s && !BLACKLIST.has(basename(s))) return s
    return fallback()
  })
}
```

**WebSocket Protocol** (`index.ts`):
The PTY uses a custom WebSocket framing protocol:
```typescript
// WebSocket control frame: 0x00 + UTF-8 JSON for cursor position
const meta = (cursor: number) => {
  const json = JSON.stringify({ cursor })
  const bytes = encoder.encode(json)
  const out = new Uint8Array(bytes.length + 1)
  out[0] = 0  // Control frame marker
  out.set(bytes, 1)
  return out
}
```

### Key Implementation Details

**PTY Creation** (`index.ts`):
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

**WebSocket Streaming** (`index.ts`):
```typescript
proc.onData(Instance.bind((chunk) => {
  session.cursor += chunk.length

  // Broadcast to all subscribers
  for (const [key, ws] of session.subscribers.entries()) {
    if (ws.readyState !== 1) {
      session.subscribers.delete(key)
      continue
    }
    ws.send(chunk)
  }

  // Buffer management with circular buffer
  session.buffer += chunk
  if (session.buffer.length <= BUFFER_LIMIT) return
  const excess = session.buffer.length - BUFFER_LIMIT
  session.buffer = session.buffer.slice(excess)
  session.bufferCursor += excess
}))
```

**Buffer Replay on Connect** (`index.ts`):
```typescript
const connect = Effect.fn("Pty.connect")(function* (id: PtyID, ws: Socket, cursor?: number) {
  // Support replaying buffer from a specific cursor position
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

  // Send cursor metadata
  ws.send(meta(end))
})
```

**Event Bus Integration** (`bus/index.ts`):
The PTY publishes events for UI updates:
```typescript
export const Event = {
  Created: BusEvent.define("pty.created", z.object({ info: Info })),
  Updated: BusEvent.define("pty.updated", z.object({ info: Info })),
  Exited: BusEvent.define("pty.exited", z.object({ id: PtyID.zod, exitCode: z.number() })),
  Deleted: BusEvent.define("pty.deleted", z.object({ id: PtyID.zod })),
}
```

### Clever Solutions

1. **Circular buffer**: Terminal output beyond the buffer limit is automatically pruned from the beginning, preventing memory exhaustion

2. **Chunked transfer**: Large WebSocket sends are chunked into 64KB pieces to prevent blocking

3. **Cursor tracking**: The cursor position is tracked separately from the buffer, allowing efficient replay of only new content

4. **Subscriber management**: WebSocket subscribers are tracked with weak references and cleaned up on disconnect

5. **Process tree kill**: On Unix, the shell uses `kill(-pid, SIGTERM)` to kill the entire process group, ensuring child processes are cleaned up

### Technical Debt / Shortcuts

1. **Windows locale hack**: On Windows, LC_ALL/LC_CTYPE/LANG are forced to C.UTF-8 which may cause encoding issues
```typescript
if (process.platform === "win32") {
  env.LC_ALL = "C.UTF-8"
  env.LC_CTYPE = "C.UTF-8"
  env.LANG = "C.UTF-8"
}
```

2. **Fish/nu blacklisted**: These shells are excluded from the acceptable shell list, though no reason is documented

3. **No pty on WebSocket reconnect**: If a WebSocket disconnects and reconnects, the PTY session continues but there's no explicit reattach mechanism beyond buffer replay

---

## Feature 6: Desktop Application (Tauri v2)

### Overview
OpenCode provides a native cross-platform desktop application built with Tauri v2, available for macOS, Windows, and Linux (DMG, EXE, deb, rpm, AppImage).

### Core Implementation Files
- `packages/desktop/src-tauri/src/main.rs` - Rust entry point
- `packages/desktop/src-tauri/src/lib.rs` - Tauri app initialization (602 lines)
- `packages/desktop/src-tauri/src/server.rs` - Sidecar server management
- `packages/desktop/src-tauri/src/cli.rs` - CLI integration
- `packages/desktop/src-tauri/src/windows.rs` - Window management
- `packages/desktop/src-tauri/tauri.conf.json` - Tauri configuration
- `packages/desktop/src/` - Frontend Solid.js application

### Architecture

**Desktop App Initialization** (`lib.rs`):
```rust
pub fn run() {
    let builder = tauri::Builder::default()
        .plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
            // Focus existing window when another instance is launched
            if let Some(window) = app.get_webview_window(MainWindow::LABEL) {
                let _ = window.set_focus();
                let _ = window.unminimize();
            }
        }))
        .plugin(tauri_plugin_deep_link::init())
        .plugin(tauri_plugin_os::init())
        // ... more plugins
        .setup(move |app| {
            // Spawn sidecar server
            let (child, health_check) = server::spawn_local_server(app.clone(), ...);
            // Show loading window if needed
            MainWindow::create(&app).expect("Failed to create main window");
        })
}
```

**Sidecar Server Pattern** (`server.rs`):
The desktop app spawns the CLI as a "sidecar" process:
```rust
pub fn spawn_local_server(
    app: AppHandle,
    hostname: String,
    port: u32,
    password: String,
) -> (CommandChild, HealthCheck) {
    let (child, exit) = cli::serve(&app, &hostname, port, &password);

    let health_check = HealthCheck(tokio::spawn(async move {
        // Wait for server to become healthy
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

**Window Management** (`windows.rs`):
```rust
impl MainWindow {
    pub fn create(app: &AppHandle) -> Result<Self, tauri::Error> {
        let window_builder = base_window_config(
            WebviewWindowBuilder::new(app, Self::LABEL, WebviewUrl::App("/".into())),
            app,
            decorations,
        )
        .title("OpenCode")
        .disable_drag_drop_handler()
        .zoom_hotkeys_enabled(false)
        .visible(true)
        .maximized(true)

        let window = window_builder.build()?;
    }
}
```

**Frontend Integration** (`src/index.tsx`):
```typescript
import { commands, type InitStep } from "./bindings"
import { createMenu } from "./menu"

const deepLinkEvent = "opencode:deep-link"

const emitDeepLinks = (urls: string[]) => {
  window.__OPENCODE__ ??= {}
  window.__OPENCODE__.deepLinks = [...pending, ...urls]
  window.dispatchEvent(new CustomEvent(deepLinkEvent, { detail: { urls } }))
}

const listenForDeepLinks = async () => {
  const startUrls = await getCurrent().catch(() => null)
  if (startUrls?.length) emitDeepLinks(startUrls)
  await onOpenUrl((urls) => emitDeepLinks(urls)).catch(() => undefined)
}
```

### Key Implementation Details

**Linux Display Backend Detection** (`linux_display.rs`, `linux_windowing.rs`):
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

**Windows Proxy Bypass** (`lib.rs`):
```rust
// Prevents VPN/proxy from intercepting loopback connections
const LOOPBACK: [&str; 3] = ["127.0.0.1", "localhost", "::1"];

let upsert = |key: &str| {
    let mut items = std::env::var(key).unwrap_or_default()
        .split(',')
        // ... ensure loopback addresses are in NO_PROXY
};
upsert("NO_PROXY");
upsert("no_proxy");
```

**SQLite Migration Handling** (`lib.rs`):
```rust
let needs_migration = !sqlite_file_exists();
let sqlite_done = needs_migration.then(|| {
    let (done_tx, done_rx) = oneshot::channel::<()>();

    let id = SqliteMigrationProgress::listen(&app, move |e| {
        if matches!(e.payload, SqliteMigrationProgress::Done) {
            let _ = done_tx.send(());
        }
    });

    tokio::spawn(done_rx.map(async move |_| {
        app.unlisten(id);
    }))
});
```

**Loading Window Strategy** (`lib.rs`):
```rust
// Show loading window for SQLite migrations if they take >1s
let loading_window = if needs_migration
    && timeout(Duration::from_secs(1), loading_task.clone()).await.is_err()
{
    let loading_window = LoadingWindow::create(&app)?;
    sleep(Duration::from_secs(1)).await;
    Some(loading_window)
} else {
    None
};

// Main window is shown immediately regardless
MainWindow::create(&app)?;
```

**Tauri Configuration** (`tauri.conf.json`):
```json
{
  "productName": "OpenCode Dev",
  "identifier": "ai.opencode.desktop.dev",
  "mainBinaryName": "OpenCode",
  "bundle": {
    "externalBin": ["sidecars/opencode-cli"],
    "targets": ["deb", "rpm", "dmg", "nsis", "app"]
  },
  "plugins": {
    "deep-link": {
      "desktop": { "schemes": ["opencode"] }
    }
  }
}
```

### Clever Solutions

1. **Sidecar server**: The CLI is run as a sidecar process, with the desktop app acting as a wrapper that provides the native window and native features

2. **Single instance**: Uses `tauri_plugin_single_instance` to ensure only one desktop app runs at a time, focusing the existing window if a second is launched

3. **Deep link support**: Registers `opencode://` URL scheme for inter-app communication

4. **Loading window**: Shows a dedicated loading window only if SQLite migration takes longer than 1 second, avoiding flicker for fast operations

5. **Window state persistence**: Uses `tauri_plugin_window_state` to remember window size/position across sessions

6. **WSL path translation**: On Windows with WSL enabled, paths are automatically translated between Windows and Linux formats

7. **Platform-specific decorations**: Linux uses custom windowing logic based on desktop environment detection (X11 vs Wayland)

### Technical Debt / Shortcuts

1. **macOS Entitlements**: Uses `macOSPrivateApi: true` in tauri.conf.json which may require special Apple Developer certificates

2. **Windows overlay titlebar**: Creates a custom titlebar on Windows using a plugin that may have compatibility issues with some GPU configurations

3. **No auto-updater in dev**: The updater plugin is conditionally enabled only when `UPDATER_ENABLED` is set

4. **CLI installation only on Unix**: The `install_cli` command returns an error on Windows, though the sidecar binary approach should work

5. **Deep link only on Linux/debug Windows**: The deep link registration is conditionally compiled:
```rust
#[cfg(any(target_os = "linux", all(debug_assertions, windows)))]
app.deep_link().register_all().ok();
```

---

## Summary

### LSP Integration
- **Pattern**: On-demand LSP server spawning with auto-installation
- **Strength**: Supports 20+ languages with automatic setup
- **Weakness**: No graceful degradation when servers fail

### TUI
- **Pattern**: Effect-based service with WebSocket streaming
- **Strength**: Efficient buffer management with cursor tracking
- **Weakness**: Windows encoding handling is a hack

### Desktop (Tauri v2)
- **Pattern**: Sidecar CLI process with native window wrapper
- **Strength**: Rich native integration (deep links, window state, single instance)
- **Weakness**: Complex platform-specific code paths (Linux alone has 3+ files for display detection)