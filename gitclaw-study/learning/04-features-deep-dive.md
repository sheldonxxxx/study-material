# GitClaw Features Deep Dive

**Synthesized from:** 05a-05f feature batch research files
**Date:** 2026-03-26
**Repo:** /Users/sheldon/Documents/claw/reference/gitclaw

---

## Executive Summary

GitClaw is a universal git-native multimodal AI Agent framework where the agent itself is a git repository. The codebase comprises 18 documented features organized into two priority tiers: 7 core features (Tier 1) that form the foundational architecture, and 11 secondary features (Tier 2) that extend functionality.

The most distinctive architectural choice is the git-native approach: identity, rules, memory, tools, and skills are all version-controlled files within the agent repository. This enables branching, diffing, and history tracking of agent configuration alongside the agent's work product.

---

## Priority Tier 1: Core Features

These seven features form the foundational architecture of GitClaw. Every other feature builds upon them.

---

### Feature 1: Git-Native Agent Architecture

**Files:** `src/loader.ts` (382 lines), `src/context.ts` (82 lines)
**Priority:** Core

#### Concept

The agent itself is a git repository. This is not a metaphor -- the agent directory is a working git repository with `SOUL.md`, `RULES.md`, `DUTIES.md`, `agent.yaml`, and other configuration files tracked by git.

#### AgentManifest Schema

The `agent.yaml` manifest defines the agent's configuration:

```typescript
interface AgentManifest {
  spec_version: string;
  name: string;
  version: string;
  description: string;
  model: {
    preferred: string;           // "provider:model" format
    fallback: string[];
    constraints?: { temperature?, max_tokens?, top_p?, top_k?, stop_sequences? };
  };
  tools: string[];
  skills?: string[];
  runtime: { max_turns: number; timeout?: number };
  extends?: string;              // Parent agent git URL
  dependencies?: Array<{ name, source, version, mount }>;
  plugins?: Record<string, PluginConfig>;
}
```

#### Loading Pipeline

`loadAgent()` executes a 12-phase pipeline:

1. Parse `agent.yaml` as AgentManifest
2. Load environment config (`config/default.yaml` + `config/<env>.yaml`)
3. Create `.gitagent/` directory and session state
4. Resolve inheritance (if `extends` field present)
5. Resolve dependencies (git clones into `.gitagent/deps/`)
6. Discover and load plugins
7. Validate compliance rules
8. Read identity files (SOUL.md, RULES.md, DUTIES.md, AGENTS.md)
9. Build system prompt from all parts
10. Discover skills, knowledge, workflows, sub-agents, examples
11. Resolve model (env override > CLI flag > manifest preferred)
12. Return LoadedAgent

#### System Prompt Composition

The system prompt is assembled from 16 sources in strict order (lines 257-350):
1. Header: `# {name} v{version}\n{description}`
2. SOUL.md - personality/identity
3. RULES.md - behavioral constraints
4. Parent RULES.md - appended (union, not override)
5. DUTIES.md - responsibilities
6. AGENTS.md - sub-agent definitions
7. Memory instructions block
8. Knowledge block (if present)
9. Skills block (filtered by manifest.skills if set)
10. Workflows block
11. Sub-agents block
12. Examples block
13. Plugin prompt additions
14. Compliance context
15. Workspace directory instructions
16. Task learning & skill discovery instructions

#### Inheritance System

Agents can extend parent agents via the `extends` field:

```yaml
extends: "https://github.com/user/base-agent.git"
```

Resolution process:
1. Clone parent into `.gitagent/deps/<parent-name>`
2. Deep merge parent manifest with child manifest (child wins)
3. Union tool lists (child shadows duplicates)
4. Append parent RULES.md to child's RULES.md

**Technical Note:** Uses `execSync` with `2>/dev/null || true` to silently fail if clone doesn't work, then continues without parent. This is a "fail open" approach.

#### Context Resolution

Three context functions for different use cases:

- **`getContextSnapshot(agentDir, branch)`** - Returns raw content: MEMORY.md, chat summary file, recent chat history (last 30 messages), recent mood entries

- **`getVoiceContext(agentDir, branch)`** - Returns formatted string for voice LLM with four modes:
  - Awakening Mode: Fresh agent with no memory
  - Growing Mode: Early memories (<400 chars)
  - Normal Mode: Full memory with reconnection greeting
  - Includes mood patterns, session summary, recent conversation

- **`getAgentContext(agentDir, branch)`** - Richer context for run_agent with more memory tokens (1200 vs 300)

#### Clever Solutions

- **Auto-scaffolding**: If `agent.yaml` doesn't exist, `ensureRepo()` creates it with sensible defaults
- **.gitignore management**: Automatically adds `.gitagent/` to .gitignore if missing
- **Deep merge utility**: Reusable `deepMerge()` function handles nested object merging
- **Silent failures**: Git operations that fail don't crash the agent (graceful degradation)
- **Memory search path**: Checks both `.gitagent/memory/MEMORY.md` and `memory/MEMORY.md`

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| `execSync` usage | lines 161, 207 | Synchronous git operations block the event loop |
| Silent error swallowing | Clone failures | No logging or user notification |
| Model string validation | `parseModelString()` | Only checks for colon; invalid providers fail later |
| Mutable accumulators | context.ts lines 91-92 | `accText` and `accThinking` shared mutable state |
| No manifest schema validation | YAML parsing | No runtime validation of AgentManifest shape |

---

### Feature 2: SDK Programmatic Interface

**Files:** `src/sdk.ts` (511 lines), `src/sdk-types.ts`, `src/sdk-hooks.ts`, `src/exports.ts`
**Priority:** Core

#### Concept

TypeScript SDK for in-process agent execution with streaming responses, tool callbacks, and lifecycle hooks. Mirrors the Claude Agent SDK pattern. Allows programmatic use of GitClaw agents in Node.js applications.

#### Query Function Architecture

The `query()` function is the main SDK entry point, returning a `Query` object that implements `AsyncGenerator<GCMessage, void, undefined>`.

#### Message Types

Unified message types for all events:

```typescript
type GCMessage =
  | GCAssistantMessage    // Final assistant response
  | GCUserMessage          // User input
  | GCToolUseMessage       // Tool invocation started
  | GCToolResultMessage    // Tool execution completed
  | GCSystemMessage        // System events (session_start, session_end, error)
  | GCStreamDelta;         // Streaming text/thinking deltas
```

#### Channel Pattern for Async Streaming

Internal implementation uses a custom `Channel<T>` (lines 28-65):

```typescript
interface Channel<T> {
  push(v: T): void;           // Send message
  finish(): void;              // Close channel
  pull(): Promise<IteratorResult<T>>;  // Receive message
}
```

Uses a buffer array for backpressure, Promise-based waiting when buffer is empty, and a Done flag for graceful closure.

#### Event Mapping

Subscribes to `pi-agent-core` `AgentEvent` and maps to GCMessage:

| AgentEvent | GCMessage |
|------------|-----------|
| agent_start | `{type: "system", subtype: "session_start"}` |
| message_update (text_delta) | `{type: "delta", deltaType: "text", content}` |
| message_update (thinking_delta) | `{type: "delta", deltaType: "thinking", content}` |
| message_end | `{type: "assistant", ...}` |
| tool_execution_start | `{type: "tool_use", ...}` |
| tool_execution_end | `{type: "tool_result", ...}` |
| agent_end | `{type: "system", subtype: "session_end"}` |

#### Hook Systems

Two hook systems supported:

1. **Script-based hooks** (from `hooks.ts`): Defined in YAML, execute as shell scripts, wrapped via `wrapToolWithHooks()`

2. **Programmatic hooks** (from `sdk-hooks.ts`): Defined as TypeScript functions, execute in-process, wrapped via `wrapToolWithProgrammaticHooks()`

**Hook types:**
```typescript
interface GCHooks {
  onSessionStart?: (ctx: GCHookContext) => Promise<GCHookResult>;
  preToolUse?: (ctx: GCPreToolUseContext) => Promise<GCHookResult>;
  postResponse?: (ctx: GCHookContext) => Promise<void>;
  onError?: (ctx: GCHookContext & { error: string }) => Promise<void>;
}

interface GCHookResult {
  action: "allow" | "block" | "modify";
  reason?: string;
  args?: Record<string, any>;  // For modify action
}
```

**Pre-tool hook flow:**
1. Hook called with tool name and args
2. If `action === "block"`: throw error, tool not executed
3. If `action === "modify"`: use `result.args` instead of original args
4. If `action === "allow"`: proceed normally

#### Query Interface

```typescript
interface Query extends AsyncGenerator<GCMessage, void, undefined> {
  abort(): void;                    // Abort the running query
  steer(_message: string): void;    // Placeholder for steering
  sessionId(): string;              // Get current session ID
  manifest(): AgentManifest;         // Get loaded manifest (throws if not loaded)
  messages(): GCMessage[];         // Get all collected messages
}
```

**Note:** `steer()` is explicitly a placeholder (lines 472-475) -- full steering requires agent reference not currently exposed.

#### Usage Example

```typescript
import { query, tool } from "gitclaw";

const greet = tool("greet", "Greet someone", {
  properties: { name: { type: "string" } },
  required: ["name"],
}, async (args) => `Hello, ${args.name}!`);

for await (const msg of query({
  prompt: "Use greet to say hello to Zeus",
  dir: process.cwd(),
  model: "openai:gpt-4o-mini",
  tools: [greet],
  hooks: {
    preToolUse: async (ctx) => {
      console.log(`[hook] ${ctx.toolName}`, ctx.args);
      return { action: "allow" };
    },
  },
})) {
  // Handle messages...
}
```

#### Clever Solutions

- **AsyncGenerator protocol**: Query implements standard async iteration
- **Channel buffering**: Decouples producer (agent events) from consumer
- **Tool collision detection**: Warns if plugin tools conflict with existing tools
- **Message accumulation**: Collects all messages for `messages()` method
- **Manifest lazy loading**: `manifest()` throws if called before load completes

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| Placeholder `steer()` | lines 472-475 | Full steering not implemented |
| Mutable accumulators | lines 91-92 | `accText` and `accThinking` shared across async operations |
| No backpressure | Channel buffer | Could grow unbounded |
| Silent hook failures | `.catch(() => {})` | Swallows all hook errors silently |
| No abort propagation | AbortController | Signal not always passed through |
| Manifest as any | line 320 | Casts `raw as any` without validation |

---

### Feature 3: Multi-Model Provider Support

**Files:** `src/index.ts`, `src/config.ts`, `src/loader.ts`
**Priority:** Core

#### Concept

Unified interface for multiple LLM providers (Anthropic, OpenAI, Google, xAI, Groq, Mistral) via `@mariozechner/pi-ai` integration. Supports fallback chains and per-model constraints.

#### Model Resolution Priority

Model selection follows this priority (loader.ts line 355):

```
1. envConfig.model_override  (from config/<env>.yaml)
2. modelFlag                  (from --model CLI argument)
3. manifest.model.preferred   (from agent.yaml)
```

#### Model String Format

Models are specified as `"provider:model"` strings:

```
"anthropic:claude-sonnet-4-5-20250929"
"openai:gpt-4o"
"google:gemini-2.0-flash"
"xai:grok-3"
"groq:llama-3.3-70b"
"mistral:mistral-large"
```

#### Supported Providers

| Provider | Env Variable | Example Model |
|----------|--------------|---------------|
| anthropic | ANTHROPIC_API_KEY | claude-sonnet-4-5-20250929 |
| openai | OPENAI_API_KEY | gpt-4o, gpt-4o-mini |
| google | GOOGLE_API_KEY | gemini-2.0-flash |
| xai | XAI_API_KEY | grok-3 |
| groq | GROQ_API_KEY | llama-3.3-70b |
| mistral | MISTRAL_API_KEY | mistral-large |

#### Model Constraints

Constraints are specified per-model in agent.yaml with both camelCase and snake_case support:

```yaml
model:
  preferred: "openai:gpt-4o"
  constraints:
    temperature: 0.3
    max_tokens: 4096
    top_p: 0.9
    top_k: 40
    stop_sequences:
      - "###"
```

Both variants are processed (SDK options take precedence if both set).

#### Environment-Based Override

Multi-environment config system:

```typescript
export async function loadEnvConfig(agentDir: string, env?: string): Promise<EnvConfig> {
  const configDir = join(agentDir, "config");
  const envName = env || process.env.GITCLAW_ENV;
  const base = await loadYamlFile(join(configDir, "default.yaml"));
  if (envName) {
    const envOverride = await loadYamlFile(join(configDir, `${envName}.yaml`));
    return deepMerge(base, envOverride) as EnvConfig;
  }
  return base as EnvConfig;
}
```

#### Token Estimation

Used in context.ts for truncating context (rough heuristic: 4 chars/token):

```typescript
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

function truncateToTokens(text: string, maxTokens: number): string {
  const maxChars = maxTokens * 4;
  if (text.length <= maxChars) return text;
  return "[...earlier messages truncated]\n" + text.slice(-maxChars);
}
```

#### Clever Solutions

- **Snake_case/camelCase compatibility**: Both `max_tokens` and `maxTokens` work
- **Silent fallback**: If preferred model fails, pi-ai handles fallback
- **Environment config**: Easy per-environment model override
- **API key validation**: Fails fast with clear error message
- **Provider-agnostic**: Same interface regardless of provider

#### Technical Debt

| Issue | Impact |
|-------|--------|
| Minimal model string validation | Only checks for colon; invalid providers fail at runtime |
| `as any` casts | Type safety bypassed with `as any` on provider |
| Rough token estimation | 4 chars/token is approximate |
| No explicit fallback execution | Fallback chain is implicit in pi-ai |
| Provider enum not enforced | String-based provider names, no compile-time checking |

---

### Feature 4: Built-in Tools

**Files:** `src/tools/index.ts`, `src/tools/cli.ts`, `src/tools/read.ts`, `src/tools/write.ts`, `src/tools/memory.ts`, `src/tools/shared.ts`
**Priority:** Core

#### Concept

The built-in tools provide the foundational capabilities for agents to interact with the filesystem, execute shell commands, and persist memory. They are implemented as TypeScript modules conforming to the `AgentTool` interface from `pi-agent-core`.

#### Tool: CLI (`cli.ts`)

Executes shell commands and returns output:

```typescript
spawn("sh", ["-c", command], { cwd, stdio: ["ignore", "pipe", "pipe"] })
```

Key characteristics:
- Uses `sh -c` for command execution
- Supports `onUpdate` callback for streaming responses
- Timeout via `setTimeout` + `child.kill("SIGTERM")` - NOT `SIGKILL`
- Truncates output to ~100KB (last portion if exceeds `MAX_OUTPUT`)
- Exit code appended to error message when `code !== 0`

**Technical Debt:** Output truncation slices from the **end** (`text.slice(-MAX_OUTPUT)`), keeping the most recent output. No way to distinguish stdout vs stderr.

#### Tool: Read (`read.ts`)

Reads file contents with pagination:

- Path resolution: `~` expansion, relative to `cwd`
- Binary detection: Checks first 8KB for null bytes
- Pagination via `paginateLines()` (1-indexed offset)

#### Tool: Write (`write.ts`)

Creates or overwrites files:

- Path resolution same as read
- `createDirs` defaults to `true` - parent directories created automatically
- Returns byte count

**Technical Debt:** No git operations (contrast with memory tool), no file existence check, no atomic write.

#### Tool: Memory (`memory.ts`)

Git-backed persistent memory with full history:

1. **Multi-layer Architecture:**
   - Reads config from `memory/memory.yaml`
   - Supports multiple named layers
   - Falls back to `memory/MEMORY.md` if no config

2. **Git Commit Integration:**
   ```typescript
   execSync(`git add "${memoryPath}" && git commit -m "${commitMsg.replace(/"/g, '\\"')}"`, { cwd, stdio: "pipe" })
   ```

3. **Archive Overflow:** When `max_lines` configured, archives oldest entries to `memory/archive/YYYY-MM.md`

**Clever Solution:** Archive policy creates time-based archive files, making memory history queryable via `git log`.

#### Shared Constants (`shared.ts`)

- `MAX_OUTPUT = 100_000` (~100KB)
- `MAX_LINES = 2000`
- `MAX_BYTES = 100_000`
- `DEFAULT_TIMEOUT = 120` seconds
- `DEFAULT_MEMORY_PATH = "memory/MEMORY.md"`

---

### Feature 5: Declarative Tool System

**Files:** `src/tool-loader.ts`, `src/tool-utils.ts`
**Priority:** Core

#### Concept

YAML-defined tools with script implementations. Tools are defined declaratively and scripts receive args as JSON on stdin.

#### Tool Definition Schema

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  input_schema: Record<string, any>;
  output_schema?: Record<string, any>;
  implementation: {
    script: string;      // Relative path to script
    runtime?: string;    // Shell to use (default: "sh")
  };
}
```

#### Execution Flow

1. **Loading:** Scans `tools/` directory for `.yaml`/`.yml` files
2. **Validation:** Requires `name`, `description`, `input_schema`, `implementation.script`
3. **Schema Conversion:** `buildTypeboxSchema()` converts JSON-schema-like to Typebox
4. **Execution:** Spawns runtime with script, passes args as JSON on stdin

#### Script Interface

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

#### Key Implementation Details

- **Timeout:** 120 seconds hardcoded
- **Abort Signal Handling:** Checks `signal?.aborted` before starting, registers abort handler to kill child process
- **Output Parsing:** Supports `{ "text": "..." }` and `{ "result": ... }` formats, falls back to raw text

#### Technical Debt

- No validation of script file existence until execution
- No way to pass environment variables to scripts
- 120s timeout not configurable per-tool

---

### Feature 6: Plugin System

**Files:** `src/plugins.ts`, `src/plugin-cli.ts`, `src/plugin-sdk.ts`, `src/plugin-types.ts`
**Priority:** Core

#### Concept

Reusable extensions providing tools, hooks, skills, prompts, and memory layers. Supports both declarative (YAML) and programmatic (TypeScript entry point) plugins.

#### Plugin Manifest

```typescript
interface PluginManifest {
  id: string;              // Kebab-case (validated)
  name: string;
  version: string;
  description: string;
  engine?: string;         // Min gitclaw version (e.g., ">=0.3.0")
  provides?: {
    tools?: boolean;
    hooks?: { ... };
    skills?: boolean;
    prompt?: string;
  };
  config?: {
    properties?: Record<string, PluginConfigProperty>;
    required?: string[];
  };
  entry?: string;          // Programmatic entry point
}
```

#### Discovery Locations (priority order)

1. **Local:** `<agent-dir>/plugins/<name>/`
2. **Global:** `~/.gitclaw/plugins/<name>/`
3. **Installed:** `<agent-dir>/.gitagent/plugins/<name>/`

#### Installation Flow

**From git URL:**
```typescript
execFileSync("git", ["clone", "--depth", "1", "--branch", version, source, pluginDir], { stdio: "pipe" });
```
- Clones with `--depth 1` (shallow clone)
- Installs to `.gitagent/plugins/`

**From local path:** Copies directory to `plugins/` in agent directory.

#### Config Resolution

Priority order:
1. User config in `agent.yaml` → `plugins.<name>.config`
2. Environment variable (via `env` field)
3. Default value

Supports `${ENV_VAR}` syntax in config values.

#### Loading Process

1. Validate manifest (kebab-case id, required fields)
2. Check engine version compatibility
3. Resolve config values
4. Load declarative tools
5. Load declarative hooks
6. Discover skills
7. Load prompt file
8. Execute programmatic entry point

#### Programmatic Plugin API

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

#### Tool Name Collision Detection

```typescript
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

#### Clever Solutions

- **Three-tier discovery** allows flexible plugin development
- **Shallow clone** keeps plugin install fast
- **Config env var substitution** with `${VAR}` syntax enables secure deployments
- **Memory layer composition** - plugins can extend memory without modifying core memory logic

#### Technical Debt

- Engine version check only parses `>=X.Y.Z` format
- Plugin load failure silently skips with warning
- Tool collision skips entire plugin (not just colliding tools)
- No namespace isolation

---

### Feature 7: Lifecycle Hooks System

**Files:** `src/hooks.ts` (186 lines), `src/sdk-hooks.ts` (52 lines)
**Priority:** Core

#### Concept

Programmatic and shell-script-based hooks for gating, logging, and controlling agent behavior at key lifecycle points. Hooks can block tool execution, modify arguments, or run side effects.

#### Hook Types

Four hook points defined in `HooksConfig`:

```
on_session_start  — fires once when agent session begins
pre_tool_use      — fires before each tool execution (can block/modify)
post_response     — fires after each agent response
on_error          — fires when an error occurs
```

#### HookDefinition Interface

```typescript
interface HookDefinition {
  script: string;           // Shell script path (relative to hooks/ dir)
  description?: string;
  baseDir?: string;         // Override base dir (for plugin hooks)
  _handler?: Function;      // Programmatic hook (in-process callback)
}

interface HookResult {
  action: "allow" | "block" | "modify";
  reason?: string;
  args?: Record<string, any>; // Modified tool arguments
}
```

#### Execution Model

**Two execution modes:**

1. **Programmatic hooks** (`_handler` field): Called directly in-process

2. **Shell hooks** (`script` field): Spawns `sh` subprocess with script path. Input is JSON-serialized to stdin, output is JSON from stdout.

**Path Traversal Protection:**
```typescript
const resolvedScript = resolve(scriptPath);
const allowedBase = resolve(baseDir);
if (!resolvedScript.startsWith(allowedBase + "/") && resolvedScript !== allowedBase) {
  reject(new Error(`Hook "${hook.script}" escapes its base directory`));
}
```

**Timeout:** 10 seconds hard limit, after which the process is killed with SIGTERM.

#### Key Design Decisions

1. **Hook chaining with early termination**: `runHooks()` iterates through hook array, stopping on first `block` or `modify` result.

2. **Hook errors do NOT block execution**: Errors are logged but `allow` is the default fallback.

3. **Tool wrapping pattern**: `wrapToolWithHooks()` creates a proxy around `tool.execute()`.

4. **Non-JSON fallback**: If stdout is not valid JSON, the hook is treated as `allow`.

#### Technical Debt

| Issue | Impact |
|-------|--------|
| 10s timeout hardcoded | No configuration for long-running hooks |
| No hook result caching | Each tool execution triggers hook evaluation |
| hooks.yaml discovery is optional | Silent - no warning if missing |
| No hook concurrency control | Parallel tool execution could cause race conditions |

---

## Priority Tier 2: Secondary Features

These features extend the core functionality but build upon or integrate with the core features.

---

### Feature 8: Voice UI

**Files:** `src/voice/server.ts` (~1100+ lines), `src/voice/adapter.ts`, `src/voice/openai-realtime.ts`, `src/voice/gemini-live.ts`, `src/voice/chat-history.ts`, `src/voice/ui.html` (115KB)
**Priority:** Secondary

#### Concept

Browser-based voice interface running at `localhost:3333`. Supports two backends: OpenAI Realtime API and Google Gemini Multimodal Live. Includes audio processing, mood tracking, memory automation, session journaling, photo capture, Telegram/WhatsApp integration, and workflow execution.

#### Adapter Pattern

The `MultimodalAdapter` interface abstracts over OpenAI and Gemini:

```typescript
interface MultimodalAdapter {
  connect(opts: {
    toolHandler: (query: string) => Promise<string>;
    onMessage: (msg: ServerMessage) => void;
  }): Promise<void>;
  send(msg: ClientMessage): void;
  disconnect(): Promise<void>;
}
```

**ClientMessage types:** `audio`, `video_frame`, `text`, `file`
**ServerMessage types:** `audio_delta`, `transcript`, `agent_working`, `agent_done`, `tool_call`, `tool_result`, `error`, `interrupt`

#### Audio Processing

**24kHz ↔ 16kHz conversion** required because:
- Browser audio API: 24kHz
- Gemini: 16kHz
- OpenAI: handles internally

```typescript
// Gemini input: downsample 24k→16k
function downsample24kTo16k(base64_24k: string): string {
  const samples24 = new Int16Array(/* 24kHz data */);
  const outLength = Math.floor(samples24.length * 2 / 3);
  // Linear interpolation
  for (let i = 0; i < outLength; i++) {
    const srcIdx = i * 1.5;
    const lo = Math.floor(srcIdx);
    const frac = srcIdx - lo;
    const hi = Math.min(lo + 1, samples24.length - 1);
    samples16[i] = Math.round(samples24[lo] * (1 - frac) + samples24[hi] * frac);
  }
  return Buffer.from(samples16.buffer).toString("base64");
}
```

#### OpenAI Realtime Adapter

- **Dual auth strategy**: Tries direct WebSocket first, falls back to ephemeral token via REST API
- **Server-side VAD**: Uses `server_vad` with threshold 0.6, prefix padding 400ms, silence duration 800ms
- **Video frame handling**: OpenAI doesn't support continuous video. Latest frame stored and injected on next user turn
- **Speech interruption**: On `speech_started` event: sets `interrupted = true`, injects latest video frame, sends `response.cancel`
- **Whisper transcription**: `input_audio_transcription: { model: "whisper-1" }`

#### Gemini Live Adapter

- **Continuous video**: Gemini supports native continuous video streaming
- **Tool calling**: Single `run_agent` function
- **Context window compression**: 25000 trigger tokens with 12500 token sliding window

#### Voice Server Key Subsystems

1. **Memory automation**: Server-side regex patterns detect personal information. If matched, triggers background memory save.

2. **Moment detection**: Patterns like "haha", "lol", "love it", "that's amazing" trigger photo capture.

3. **Mood tracking**: Tracks mood distribution across 5 categories (happy, frustrated, curious, excited, calm) using regex patterns.

4. **Session journaling**: Uses the agent itself to generate journal entries reflecting on the conversation.

5. **Vitals snapshot**: Centralized CPU/memory/heap/uptime tracking with 1-second cache.

6. **Approval gates**: For workflow execution, can request approval via Telegram or WhatsApp before proceeding (5-minute timeout).

7. **File watching**: Snapshots file tree before/after agent runs, detects new/modified files via `diffSnapshots()`.

#### Clever Solutions

- **Background memory save**: Pattern matching happens server-side, not relying on LLM to remember
- **Video frame injection on speech start**: Captures what user was looking at when they started talking
- **Broadcast to all browser clients**: Allows multiple tabs to see the same voice session
- **Telegram long polling with offset tracking**: Custom polling implementation
- **Vitals delta-based CPU calculation**

#### Technical Debt

| Issue | Impact |
|-------|--------|
| UI.html is 115KB | Large embedded UI, cumbersome to edit |
| Hardcoded port 3333 | No configuration option |
| Telegram polling | Custom long-polling needs proper reconnection/backoff |
| WhatsApp uses `any` type | No typed WhatsApp client |
| Approval gate timeout hardcoded | 5 minutes, no configuration |
| Git commits for mood/journal | Can fail silently |

---

### Feature 9: Skills System

**Files:** `src/skills.ts` (194 lines)
**Priority:** Secondary

#### Concept

Composable instruction modules for agents. Skills are directories containing a `SKILL.md` file with YAML frontmatter defining name/description, plus optional `scripts/` and `references/` subdirectories.

#### Skill Discovery

```typescript
interface SkillMetadata {
  name: string;
  description: string;
  directory: string;
  filePath: string;
  confidence?: number;
  usage_count?: number;
  success_count?: number;
  failure_count?: number;
}
```

Discovery process:
1. Scans `agentDir/skills/` for directories or symlinks
2. Reads `SKILL.md` from within
3. Parses YAML frontmatter for `name` and `description`
4. Validates: name must match directory name, must be kebab-case
5. Sorts alphabetically by name

#### Frontmatter Format

```markdown
---
name: example-skill
description: Example skill demonstrating the gitclaw skills system.
confidence: 0.9
usage_count: 42
---

# Skill Content Below
```

#### Skill Invocation

**Slash command expansion:**
```typescript
// Input: "/skill:example-skill arg1 arg2"
// Output: {
//   expanded: "<skill name=\"example-skill\" baseDir=\"...\">\n...",
//   skillName: "example-skill"
// }
```

Pattern: `/skill:([a-z0-9-]+)\s*([\s\S]*)$`

#### Prompt Formatting

Skills are formatted as mandatory-first instructions:

```
# Skills — FIRST PRIORITY (MANDATORY)

CRITICAL: You have installed skills that provide specialized capabilities.
Before attempting ANY task — simple or complex — you MUST check if an installed skill handles it.

## Rules (MUST follow in order)
1. ALWAYS scan the skill list below BEFORE taking ANY action
2. If a skill's description matches, you MUST load its full instructions
3. Follow the loaded skill instructions EXACTLY
...
```

#### Skill Structure

```
example-skill/
  SKILL.md          # Required: name, description frontmatter + instructions
  scripts/          # Optional: executable scripts
    hello.sh
  references/       # Optional: reference documents
    README.md
```

#### Technical Debt

| Issue | Impact |
|-------|--------|
| No skill versioning | Running sessions don't see updates until restarted |
| No skill dependencies | Composition is implicit via SkillFlows |
| Skill loading reads from disk every time | No caching |
| No built-in skill discovery from URLs | Unlike agents, skills must be local |
| Confidence/usage stats not automatically updated | Presumably handled by learning system |

---

### Feature 10: Local Repo Mode (GitHub Integration)

**Files:** `src/session.ts` (156 lines)
**Priority:** Secondary

#### Concept

Clone a GitHub repository, run an agent on it, auto-commit changes to a session branch, and push back to origin. Supports session resumption.

#### Session Branch Strategy

Branches follow the naming convention `gitclaw/session-<8-char-hex>`. The hex is generated via `randomBytes(4)` when creating a new session.

**Key flow:**
1. Clone with `--depth 1 --no-single-branch` (full history but tracked)
2. On existing dir: reset local to remote default branch (hard reset)
3. For new sessions: create `gitclaw/session-<hex>` branch off current default
4. For resuming: checkout existing branch, pull latest

#### Token Security Pattern

```typescript
function authedUrl(url: string, token: string): string {
  // https://github.com/org/repo → https://<token>@github.com/org/repo
  return url.replace(/^https:\/\//, `https://${token}@`);
}

function cleanUrl(url: string): string {
  // Strip PAT from URL before final push
  return url.replace(/^https:\/\/[^@]+@/, "https://");
}
```

Token injected into clone/fetch URLs, stripped via `cleanUrl()` before storing. Prevents PAT leakage.

#### Clever Solutions

- `--no-single-branch` on clone preserves full git history while keeping clone fast
- `git reset --hard origin/<branch>` ensures clean state on resume
- Auto-scaffolding of `agent.yaml` with sensible defaults when missing

#### Technical Debt

| Issue | Impact |
|-------|--------|
| execSync blocking | All git operations synchronous |
| No progress feedback | Clone of large repos has no streaming |
| No branch cleanup | Old session branches accumulate |
| Hard reset discards uncommitted local changes | By design but undocumented |

---

### Feature 11: Sandbox Execution

**Files:** `src/sandbox.ts` (95 lines), `src/tools/sandbox-cli.ts`, `src/tools/sandbox-read.ts`, `src/tools/sandbox-write.ts`, `src/tools/sandbox-memory.ts`
**Priority:** Secondary

#### Concept

Run agents in isolated VM (e2b provider via gitmachine) to prevent destructive operations from affecting the host system.

#### Architecture

```
createSandboxContext(config, dir)
    ├── Dynamic import gitmachine (optional peer dep)
    ├── detectRepoUrl()           -- git remote get-url origin
    ├── new GitMachine({...})      -- e2b VM provisioning
    └── Return SandboxContext:
          gitMachine: GitMachine instance
          machine: Machine instance (readFile, writeFile)
          repoPath: /home/user/<repo-name>
```

#### Tool Wrapping Pattern

```typescript
if (config.sandbox) {
  return [
    createSandboxCliTool(config.sandbox, config.timeout),
    createSandboxReadTool(config.sandbox),
    createSandboxWriteTool(config.sandbox),
    createSandboxMemoryTool(config.sandbox),
  ];
}
```

#### Sandbox Memory Tool

Notable features:
- Supports `memory/memory.yaml` config with named layers
- **Auto-archiving**: When `max_lines` exceeded, older entries moved to `memory/archive/<year>-<month>.md`
- Each save creates git commit via `ctx.gitMachine.commit()`
- Graceful degradation: if commit fails, file is still written but user is warned

#### Optional Dependency Pattern

Gitmachine is loaded via dynamic import with clear error message:
```typescript
try {
  gitmachine = await import("gitmachine");
} catch {
  throw new Error(
    "Sandbox mode requires the 'gitmachine' package.\n" +
    "Install it with: npm install gitmachine",
  );
}
```

#### Technical Debt

| Issue | Impact |
|-------|--------|
| Hardcoded e2b provider | Only e2b supported despite provider field in config |
| Fixed repo path convention | Assumed but not enforced |
| No network isolation verification | Degree of isolation determined by gitmachine/e2b |
| Timeout handling unclear | Config passed to GitMachine but error handling unclear |

---

### Feature 12: Compliance & Audit Logging

**Files:** `src/compliance.ts` (161 lines), `src/audit.ts` (77 lines)
**Priority:** Secondary

#### Concept

Built-in compliance validation against agent manifest rules, audit logging to `.gitagent/audit.jsonl`, and regulatory framework support (SOC2, GDPR).

#### Compliance Validation

`validateCompliance()` runs rule-based checks:

| Rule ID | Severity | Condition |
|---------|----------|-----------|
| `high_risk_hitl` | warning | High/critical risk without human_in_the_loop |
| `critical_audit` | error | Critical risk without audit_logging |
| `regulatory_recordkeeping` | warning | Regulatory frameworks without recordkeeping |
| `high_risk_review` | warning | High/critical risk without review config |
| `audit_retention` | warning | Audit logging without retention_days |
| `data_classification` | warning | Regulatory frameworks without data_classification |

**Design pattern:** Rules return warnings or errors. Errors should block startup but are currently informational.

#### Audit Logger

`AuditLogger` class appends JSONL entries to `.gitagent/audit.jsonl`:

```typescript
interface AuditEntry {
  timestamp: string;
  session_id: string;
  event: string;
  tool?: string;
  args?: Record<string, any>;
  result?: string;
  error?: string;
  [key: string]: any;
}
```

**Non-blocking design:** Logging failures are silently swallowed.

#### Event Types

| Event | When Logged |
|-------|-------------|
| `tool_use` | Tool execution starts |
| `tool_result` | Tool completes (result truncated to 1000 chars) |
| `response` | LLM response generated |
| `error` | Any error occurs |
| `session_start` | Session begins |
| `session_end` | Session ends |

#### Clever Solutions

- **JSONL format**: Append-only, line-oriented JSON enables easy streaming reads
- **Result truncation**: Tool results capped at 1000 chars to prevent log bloat
- **Graceful degradation**: Audit logging failures never crash session

#### Technical Debt

| Issue | Impact |
|-------|--------|
| No compliance enforcement | Errors are informational, not blocking |
| Audit logs not rotated | No retention enforcement |
| No audit log integrity | No signing or tamper-detection |
| Warning display only | Errors from `critical_audit` printed as warnings |

---

### Feature 13: Workflows

**Files:** `src/workflows.ts` (167 lines)
**Priority:** Secondary

#### Concept

Multi-step workflow definitions for complex agent tasks. Workflows chain tools and skills together through declarative YAML format.

#### Key Components

**WorkflowMetadata:**
```typescript
interface WorkflowMetadata {
  name: string;
  description: string;
  filePath: string;
  format: "yaml" | "markdown";
  type?: "flow" | "basic";
  steps?: SkillFlowStep[];
}
```

**SkillFlowDefinition:**
```typescript
interface SkillFlowStep {
  skill: string;
  prompt: string;
  channel?: string;
}

interface SkillFlowDefinition {
  name: string;
  description: string;
  steps: SkillFlowStep[];
}
```

#### Discovery Pattern

```
discoverWorkflows(agentDir)
├── check workflows/ is directory
├── readdir entries
├── for each entry:
│   ├── if .yaml/.yml:
│   │   ├── if has name + steps → type="flow"
│   │   └── else → type="basic"
│   └── if .md:
│       └── parse frontmatter
└── sort by name, return
```

#### Critical Technical Debt

**There is NO actual executor.** The file defines workflows but the agent itself must interpret and execute them. This is a major gap:

- No execution engine in `workflows.ts`
- Workflows are discovered and formatted for prompts, but not executed by code
- No flow state persistence
- No CLI commands to list/run/delete workflows

---

### Feature 14: Agent Inheritance & Composition

**Files:** `src/agents.ts` (97 lines), `src/loader.ts`
**Priority:** Secondary

#### Sub-Agent Discovery

**Two forms:**

1. **Directory Form:** `agents/<name>/agent.yaml`
   - Full agent manifest with name/description

2. **File Form:** `agents/<name>.md`
   - Markdown file with frontmatter containing name/description

```typescript
interface SubAgentMetadata {
  name: string;
  description: string;
  type: "directory" | "file";
  path: string;
}
```

#### Inheritance Resolution

```yaml
extends: "https://github.com/org/base-agent.git"
```

Resolution:
1. Clone parent into `.gitagent/deps/<parent-name>`
2. Deep merge parent manifest with child (child wins)
3. Union tool lists (child shadows duplicates)
4. Append parent RULES.md to child's RULES.md

#### Technical Debt

| Issue | Impact |
|-------|--------|
| No sub-agent execution | Discovery only; actual delegation requires spawning new gitclaw process |
| No dependency mounting logic | `mount` field in dependencies is unused |
| No circular dependency detection | Possible infinite clone loop |
| Shallow clones only | `--depth 1` means no git history |

---

### Feature 15: Knowledge Base

**Files:** `src/knowledge.ts` (82 lines)
**Priority:** Secondary

#### Concept

Files in `knowledge/` directory indexed and available to agent via `knowledge/index.yaml`.

#### Knowledge Entry Schema

```yaml
entries:
  - path: "api-docs/authentication.md"
    tags: ["api", "auth"]
    priority: "high"
    always_load: true

  - path: "guides/getting-started.md"
    tags: ["guide"]
    priority: "medium"
```

#### Two-Tier Loading

1. **Always-load files:** Injected directly into system prompt
2. **Available files:** Listed as on-demand references via read tool

#### Technical Debt

| Issue | Impact |
|-------|--------|
| No indexing | Agent must use `read` tool directly |
| No versioning | Knowledge files aren't versioned |
| Priority is informational only | No enforcement |
| Path traversal risk | Entry paths joined with knowledge dir; no sanitization |

---

### Feature 16: Schedules & Task Runner

**Files:** `src/schedules.ts` (120 lines), `src/schedule-runner.ts` (157 lines)
**Priority:** Secondary

#### Concept

Scheduled task execution for agents using cron expressions or one-time `runAt` timestamps. Schedules are git-native YAML files in `schedules/` directory.

#### ScheduleDefinition

```typescript
interface ScheduleDefinition {
  id: string;           // kebab-case
  prompt: string;       // The prompt to execute
  cron: string;         // Cron expression
  mode: "repeat" | "once";
  runAt?: string;       // ISO datetime
  enabled: boolean;
  createdAt: string;
  lastRunAt?: string;
  lastResult?: string;
}
```

#### Execution Engine

**Three paths in `startScheduler()`:**

1. **Once via `runAt`** (setTimeout) - delay computed from `Date.now() - runAt.getTime()`
2. **Once via cron** (node-cron) - fires once, then disables
3. **Repeating via cron** (node-cron) - fires repeatedly

**Concurrency guard:** `runningJobs` Set prevents overlapping executions.

#### Technical Debt

| Issue | Impact |
|-------|--------|
| `updateScheduleMeta()` re-serializes entire YAML | No locking |
| Module-level Maps | No restart resilience |
| Log write failures | Non-fatal, caught silently |

---

### Feature 17: Learning & Skill Capture

**Files:** `src/learning/reinforcement.ts` (120 lines), `src/tools/task-tracker.ts` (350 lines), `src/tools/skill-learner.ts` (413 lines), `src/tools/capture-photo.ts` (106 lines)
**Priority:** Secondary

#### Concept

Tracks task outcomes, adjusts skill confidence through reinforcement learning, and can capture successful task sequences as reusable skills.

#### Task Record

```typescript
interface TaskRecord {
  id: string;              // randomUUID
  objective: string;        // What the task aims to accomplish
  steps: TaskStep[];        // Array of {description, timestamp}
  attempts: number;         // How many times this objective was tried
  status: "active" | "succeeded" | "failed";
  outcome?: "success" | "failure" | "partial";
  failure_reason?: string;
  skill_used?: string;
  started_at: string;
  ended_at?: string;
}
```

#### Reinforcement Learning

**Confidence adjustment formula:**
- **Success:** `conf + 0.1 * (1 - conf)` (asymptotic to 1.0)
- **Failure:** `conf - 0.2` (2x penalty)
- **Partial:** `conf - 0.05`

**Asymmetric loss design:** Failures penalize more heavily than successes reward.

#### Skill Learner Actions

- **evaluate**: Checks if a completed task deserves to become a reusable skill
  - multi_step: At least 3 steps
  - non_trivial: At least 2 steps
  - novel: No existing skill with >0.5 Jaccard similarity
  - generalizable: <30% of steps are project-specific

- **crystallize**: Creates a new skill from a successful task
  - Builds SKILL.md with YAML frontmatter + steps + "What Worked" + "What Did NOT Work"

#### Technical Debt

| Issue | Impact |
|-------|--------|
| Task tracker no concurrency protection | Assumed single-threaded |
| SkillsMP marketplace integration | Optional (env var gated) |

---

### Feature 18: Composio Integration

**Files:** `src/composio/client.ts` (243 lines), `src/composio/adapter.ts` (120 lines)
**Priority:** Secondary

#### Concept

Integration with Composio for extended tool actions (100+ external toolkits: Gmail, Slack, GitHub, Notion, etc.).

#### ComposioClient

**Zero-dependency design:** Uses native `fetch()`.

```typescript
interface ComposioTool {
  name: string;
  slug: string;
  description: string;
  toolkitSlug: string;
  parameters: Record<string, any>;
}
```

**Key methods:**
- `listToolkits(userId?)` - Available toolkits with connection status
- `searchTools(query, toolkitSlugs?, limit?)` - Semantic tool search
- `executeTool(toolSlug, userId, params, connectedAccountId?)` - Execute action

#### ComposioAdapter

Converts Composio tools to GitClaw's `GCToolDefinition` format:

```typescript
const safeName = `composio_${t.toolkitSlug}_${t.slug}`.replace(/[^a-zA-Z0-9_]/g, "_");
// Example: composio_gmail_SEND_EMAIL
```

**Caching:** 30-second TTL cache for `getTools()` results.

#### Technical Debt

| Issue | Impact |
|-------|--------|
| Auth config race condition | Multiple concurrent `getOrCreateAuthConfig()` calls could create duplicates |
| API key exposure | Composio API key sent with every request |

---

## Cross-Feature Patterns

### Git-Native Persistence

Many features use git-committed files as their persistence layer:

- **Memory**: `.gitagent/memory/`, `memory/MEMORY.md`, `memory/archive/`
- **Schedules**: `schedules/*.yaml`
- **Skills**: `skills/*/SKILL.md`
- **Audit**: `.gitagent/audit.jsonl`
- **Learning**: `.gitagent/learning/tasks.json`

### Directory-Based Discovery Pattern

Skills, Knowledge, Workflows, Sub-Agents, and Schedules all follow the same discovery pattern:

```
<feature>Dir = join(agentDir, "<feature>")
├── check exists/isDirectory
├── readdir entries
├── parse content (YAML/frontmatter)
├── return sorted metadata[]
```

### Graceful Degradation

Most features silently skip errors rather than failing:
- Missing files: Returns empty/default
- Invalid YAML: Skipped silently
- Clone failures: Continues without parent
- Git commits: File written even if commit fails

### Tool Wrapping Pattern

Several systems wrap the CLI tool:
- Hooks: `wrapToolWithHooks()`
- Sandbox: `createSandboxCliTool()`
- This allows adding cross-cutting behavior without modifying the core tool.

---

## Maturity Assessment

| Feature | Files | Lines | Status | Key Concern |
|---------|-------|-------|--------|-------------|
| Git-Native Agent | 2 | ~464 | Core | execSync blocking |
| SDK | 4 | ~600+ | Core | Placeholder steer() |
| Multi-Model | 3 | ~300 | Core | Runtime validation |
| Built-in Tools | 6 | ~400 | Core | Output truncation |
| Declarative Tools | 2 | ~200 | Core | 120s hardcoded timeout |
| Plugin System | 4 | ~500 | Core | Tool collision skips plugin |
| Lifecycle Hooks | 2 | ~238 | Core | 10s hardcoded timeout |
| Voice UI | 6 | ~1400 | Secondary | 115KB ui.html |
| Skills | 1 | ~194 | Secondary | No caching |
| Local Repo Mode | 1 | ~156 | Secondary | No branch cleanup |
| Sandbox | 5 | ~398 | Secondary | e2b only |
| Compliance | 2 | ~238 | Secondary | Informational only |
| Workflows | 1 | ~167 | Secondary | **No execution engine** |
| Inheritance | 2 | ~479 | Secondary | No sub-agent execution |
| Knowledge | 1 | ~82 | Secondary | No indexing |
| Schedules | 2 | ~277 | Secondary | No restart resilience |
| Learning | 4 | ~989 | Secondary | Marketplace API optional |
| Composio | 2 | ~365 | Secondary | Auth race condition |

---

## Key Technical Debt Themes

1. **Blocking I/O**: `execSync` used throughout for git operations, blocking the event loop
2. **Hardcoded Timeouts**: 10s, 120s timeouts not configurable
3. **Silent Failures**: Errors swallowed rather than surfaced
4. **No Execution**: Workflows defined but not executed
5. **Runtime Validation**: Type safety bypassed with `as any`
6. **Missing Implementations**: `steer()`, sub-agent execution, skill discovery from URLs
