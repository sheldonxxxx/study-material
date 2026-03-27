# Features 4-6: Deep Dive Research

**Features:** Built-in Tools, Declarative Tool System, Plugin System
**Project:** GitClaw
**Research Date:** 2026-03-26

---

## Feature 4: Built-in Tools

### Overview
The built-in tools provide the foundational capabilities for agents to interact with the filesystem, execute shell commands, and persist memory. They are implemented as TypeScript modules that conform to the `AgentTool` interface from `pi-agent-core`.

### Core Files
- `src/tools/index.ts` - Factory that creates all built-in tools
- `src/tools/cli.ts` - Shell command execution
- `src/tools/read.ts` - File reading with pagination
- `src/tools/write.ts` - File writing
- `src/tools/memory.ts` - Git-committed memory operations
- `src/tools/shared.ts` - Common schemas and utilities

### Tool: CLI (`cli.ts`)

**Purpose:** Execute shell commands and return output.

**Key Implementation Details:**
```typescript
spawn("sh", ["-c", command], { cwd, stdio: ["ignore", "pipe", "pipe"] })
```
- Uses `sh -c` for command execution
- stdin is ignored (no piped input)
- Combined stdout/stderr collected as output
- Supports `onUpdate` callback for streaming responses

**Error Handling & Edge Cases:**
- `signal?.aborted` checked before starting and during `close`
- Timeout via `setTimeout` + `child.kill("SIGTERM")` - NOT `SIGKILL`
- Truncates output to ~100KB (last portion if exceeds `MAX_OUTPUT`)
- Exit code appended to error message when `code !== 0`

**Technical Debt/Shortcuts:**
- Output truncation slices from the **end** (`text.slice(-MAX_OUTPUT)`), keeping the most recent output
- No way to distinguish stdout vs stderr in output
- Shell exits with null code if signal kills before completion (distinction lost)

---

### Tool: Read (`read.ts`)

**Purpose:** Read file contents with pagination.

**Key Implementation Details:**
- Path resolution: `~` expansion, relative to `cwd`
- Binary detection: Checks first 8KB for null bytes
- Pagination via `paginateLines()` from shared utilities

**Pagination Logic (`shared.ts`):**
```typescript
const startLine = offset ? Math.max(0, offset - 1) : 0;  // 1-indexed offset
const maxLines = limit ?? MAX_LINES;  // Default 2000 lines
```
- Offset is 1-indexed (user-friendly, not 0-indexed)
- If `offset > totalLines`, throws error
- Also truncates by bytes (~100KB) if line limit insufficient

**Error Handling:**
- Returns `[Binary file: ${path} (${buffer.length} bytes)]` for binary
- Returns friendly message for missing files (via `readFile` throwing)
- Pagination error if offset beyond file end

---

### Tool: Write (`write.ts`)

**Purpose:** Create or overwrite files.

**Key Implementation Details:**
- Path resolution same as read (`~` expansion, relative to `cwd`)
- `createDirs` defaults to `true` - parent directories created automatically
- Uses `mkdir(dirname, { recursive: true })`

**Error Handling:**
- Abort check before starting
- No validation of path validity (can write anywhere accessible)
- Returns byte count: `Buffer.byteLength(content, "utf-8")`

**Technical Debt:**
- No git operations (contrast with memory tool)
- No file existence check before writing
- No atomic write (no temp file + rename)

---

### Tool: Memory (`memory.ts`)

**Purpose:** Git-backed persistent memory with full history.

**Key Implementation Details:**

1. **Multi-layer Architecture:**
   - Reads config from `memory/memory.yaml`
   - Supports multiple named layers
   - Falls back to `memory/MEMORY.md` if no config
   - Plugin-provided memory layers can be merged in

2. **Git Commit Integration:**
   ```typescript
   execSync(`git add "${memoryPath}" && git commit -m "${commitMsg.replace(/"/g, '\\"')}"`, { cwd, stdio: "pipe" })
   ```
   - Commits atomically with the memory file
   - Commit message escaped for quotes
   - Gracefully handles non-git directories (warning instead of error)

3. **Archive Overflow:**
   - When `max_lines` configured in memory layer
   - Archives oldest entries to `memory/archive/YYYY-MM.md`
   - Tries to `git add` the archive file

**Error Handling:**
- Git commit failure is a **soft failure** - file is still written, warning returned
- No memories yet returns friendly message
- Missing memory file returns same friendly message

**Clever Solution:**
The archive policy creates time-based archive files, making memory history queryable via git log.

---

### Shared Utilities (`shared.ts`)

**Constants:**
- `MAX_OUTPUT = 100_000` (~100KB)
- `MAX_LINES = 2000`
- `MAX_BYTES = 100_000`
- `DEFAULT_TIMEOUT = 120` seconds
- `DEFAULT_MEMORY_PATH = "memory/MEMORY.md"`

**Schemas:** All tools use Typebox (`@sinclair/typebox`) for schema definition.

---

## Feature 5: Declarative Tool System

### Overview
The declarative tool system allows defining tools via YAML files with script implementations. Tools are loaded from the `tools/` directory in the agent root, and scripts receive arguments as JSON on stdin.

### Core Files
- `src/tool-loader.ts` - YAML tool loading and execution
- `src/tool-utils.ts` - Tool utilities (includes `buildTypeboxSchema` used by plugin-sdk)

### Tool Definition Schema (`tool-loader.ts`)

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  input_schema: Record<string, any>;
  output_schema?: Record<string, any>;
  implementation: {
    script: string;      // Relative path to script
    runtime?: string;   // Shell to use (default: "sh")
  };
}
```

### YAML Example (conceptual):
```yaml
name: my-tool
description: Does something useful
input_schema:
  properties:
    input:
      type: string
      description: The input text
  required: [input]
implementation:
  script: scripts/my-tool.sh
  runtime: sh
```

### Execution Flow

1. **Loading:** Scans `tools/` directory for `.yaml`/`.yml` files
2. **Validation:** Requires `name`, `description`, `input_schema`, `implementation.script`
3. **Schema Conversion:** `buildTypeboxSchema()` converts JSON-schema-like to Typebox
4. **Execution:** Spawns runtime with script, passes args as JSON on stdin

### Script Interface

Scripts receive JSON on stdin:
```bash
#!/bin/bash
read -r ARGS_JSON
# Parse $ARGS_JSON and use jq or similar
```

Scripts should output JSON or text to stdout:
```json
{ "text": "result" }
# or just plain text
```

### Key Implementation Details

**Timeout:** 120 seconds hardcoded
```typescript
const timeout = setTimeout(() => {
  child.kill("SIGTERM");
  reject(new Error(`Tool "${def.name}" timed out after 120s`));
}, 120_000);
```

**Abort Signal Handling:**
- Checks `signal?.aborted` before starting
- Registers abort handler to kill child process
- Removes handler after completion

**Output Parsing:**
```typescript
try {
  const parsed = JSON.parse(text);
  if (parsed.text) text = parsed.text;
  else if (parsed.result) text = typeof parsed.result === "string" ? parsed.result : JSON.stringify(parsed.result);
} catch {
  // Raw text output is fine
}
```
- Supports `{ "text": "..." }` format
- Supports `{ "result": ... }` format
- Falls back to raw text

**Error Handling:**
- Non-zero exit code rejected with stderr
- Script not found → child error event
- Missing required fields → skips invalid definitions

### Integration with Plugin System

Declarative tools are loaded by the plugin system via `loadDeclarativeTools()`:
```typescript
// From plugins.ts
if (manifest.provides?.tools) {
  tools = await loadDeclarativeTools(pluginDir);
}
```

### Technical Debt/Shortcuts
- No validation of script file existence until execution
- No way to pass environment variables to scripts (inherits `process.env`)
- No working directory control beyond agent root
- 120s timeout not configurable per-tool

---

## Feature 6: Plugin System

### Overview
The plugin system provides reusable extensions that can add tools, hooks, skills, prompts, and memory layers. Plugins support both declarative (YAML) and programmatic (TypeScript) approaches.

### Core Files
- `src/plugins.ts` - Discovery and loading engine
- `src/plugin-cli.ts` - CLI commands for plugin management
- `src/plugin-sdk.ts` - Programmatic plugin API
- `src/plugin-types.ts` - Type definitions

### Plugin Manifest (`plugin-types.ts`)

```typescript
interface PluginManifest {
  id: string;              // Kebab-case (validated)
  name: string;
  version: string;
  description: string;
  author?: string;
  license?: string;
  engine?: string;         // Min gitclaw version (e.g., ">=0.3.0")
  provides?: {
    tools?: boolean;
    hooks?: { ... };
    skills?: boolean;
    prompt?: string;       // Path to prompt file
  };
  config?: {
    properties?: Record<string, PluginConfigProperty>;
    required?: string[];
  };
  entry?: string;          // Programmatic entry point
}
```

### Discovery Locations (in priority order)

1. **Local:** `<agent-dir>/plugins/<name>/`
2. **Global:** `~/.gitclaw/plugins/<name>/`
3. **Installed:** `<agent-dir>/.gitagent/plugins/<name>/`

### Installation Flow

**From git URL:**
```typescript
// plugins.ts
execFileSync("git", ["clone", "--depth", "1", "--branch", version, source, pluginDir], { stdio: "pipe" });
```
- Clones with `--depth 1` (shallow clone)
- Optionally checks out specific version/branch
- Installs to `.gitagent/plugins/`

**From local path:**
- Copies directory to `plugins/` in agent directory

### Config Resolution (`resolvePluginConfig`)

Priority order:
1. User config in `agent.yaml` → `plugins.<name>.config`
2. Environment variable (via `env` field)
3. Default value

```typescript
// Env var substitution
if (typeof value === "string") {
  value = value.replace(/\$\{(\w+)\}/g, (_, envName) => process.env[envName] || "");
}
```

Supports `${ENV_VAR}` syntax in config values.

### Loading Process

1. Validate manifest (kebab-case id, required fields)
2. Check engine version compatibility
3. Resolve config values
4. Load declarative tools
5. Load declarative hooks
6. Discover skills
7. Load prompt file
8. Execute programmatic entry point

### Programmatic Plugin API (`plugin-sdk.ts`)

```typescript
interface GitclawPluginApi {
  pluginId: string;
  pluginDir: string;
  config: Record<string, any>;
  registerTool(def: GCToolDefinition): void;
  registerHook(event: HookEvent, handler: HookHandler): void;
  addPrompt(text: string): void;
  registerMemoryLayer(layer: { name, path, description }): void;
  logger: { info, warn, error };
}
```

### Hook Events
- `on_session_start`
- `pre_tool_use`
- `post_response`
- `on_error`

### Tool Name Collision Detection

```typescript
// From plugins.ts
const allPluginToolNames = [
  ...plugin.tools.map((t) => t.name),
  ...plugin.programmaticTools.map((t) => t.name),
];
const collisions = allPluginToolNames.filter((name) => toolNames.has(name));
if (collisions.length > 0) {
  console.error(`Plugin "${pluginName}": tool name collision(s): ${collisions.join(", ")}. Skipping plugin.`);
  continue;
}
```

**First-loaded wins** - plugins loaded in order, later plugins with collisions are skipped.

### Hook Merging

```typescript
// From plugins.ts
export function mergeHooksConfigs(base, plugins): HooksConfig {
  // Merges all plugin hooks into base hooks
  // Plugins hooks run AFTER base hooks
}
```

Hooks are appended, not merged - base hooks run first, then plugin hooks.

### CLI Commands (`plugin-cli.ts`)

| Command | Description |
|---------|-------------|
| `install <source>` | Install from git URL or local path |
| `list` | List all discovered plugins |
| `remove <name>` | Remove plugin |
| `enable <name>` | Enable plugin in agent.yaml |
| `disable <name>` | Disable plugin in agent.yaml |
| `init <name>` | Scaffold new plugin |

**Init Scaffolding:**
```
plugins/
  <name>/
    plugin.yaml
    tools/
    hooks/
    skills/
    README.md
```

### Memory Layers from Plugins

Plugins can register memory layers via `registerMemoryLayer()`:
```typescript
// These are merged into the memory tool's layer config
memoryLayers = api.getMemoryLayers();
```

The memory tool merges these at load time:
```typescript
if (pluginLayers && pluginLayers.length > 0) {
  config.layers.push({ name: layer.name, path: layer.path, format: "markdown" });
}
```

### Technical Debt/Shortcuts

1. **Engine version check:** Only parses `>=X.Y.Z` format, others are allowed by default
2. **Plugin load failure:** Silently skips with warning, doesn't fail agent
3. **Tool collision:** Entire plugin skipped, not just colliding tools
4. **No namespace isolation:** Tools/hooks share namespace across plugins
5. **Programmatic hooks:** Uses synthetic script paths with attached handlers - requires special execution path

### Clever Solutions

1. **Three-tier discovery** allows flexible plugin development (local dev, user-wide install, distributed)
2. **Shallow clone** (`--depth 1`) keeps plugin install fast
3. **Config env var substitution** with `${VAR}` syntax enables secure deployments
4. **agent.yaml preserved** when reinstalling (using yaml library that preserves formatting)
5. **Memory layer composition** - plugins can extend memory without modifying core memory logic

---

## Cross-Feature Integration

### Tool Loading Pipeline

```
loader.ts: loadAgent()
    │
    ├── discoverAndLoadPlugins()
    │       │
    │       ├── loadDeclarativeTools() → declarative tools
    │       └── loadPlugin() → programmatic tools via plugin-sdk
    │
    └── Built-in tools created separately
            │
            └── createBuiltinTools()
                    ├── createCliTool()
                    ├── createReadTool()
                    ├── createWriteTool()
                    └── createMemoryTool() ← plugin memory layers merged in
```

### Plugin Prompt Composition

```typescript
// loader.ts
for (const plugin of plugins) {
  if (plugin.promptAddition) {
    parts.push(`# Plugin: ${plugin.manifest.name}\n\n${plugin.promptAddition}`);
  }
}
```

### Skills Discovery with Plugin Skills

```typescript
// loader.ts
for (const plugin of plugins) {
  skills = [...skills, ...plugin.skills];
}
// Note: Plugin skills NOT filtered by manifest.skills (considered trusted)
```

---

## Summary

| Aspect | Built-in Tools | Declarative Tools | Plugin System |
|--------|---------------|-------------------|---------------|
| Definition | TypeScript | YAML | YAML + TypeScript |
| Execution | Direct | Script spawn | Mixed |
| Args Passing | Function params | JSON stdin | Function calls |
| Tool Collision | N/A | N/A | Entire plugin skipped |
| Timeout | 120s default | 120s hardcoded | N/A |
| Git Integration | Memory only | None | Optional |
| Composition | Single layer | Single layer | Multi-layer |
