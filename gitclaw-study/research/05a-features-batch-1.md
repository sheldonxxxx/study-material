# Feature Deep Dive: Batch 1

**Features Analyzed:** Git-Native Agent Architecture, SDK Programmatic Interface, Multi-Model Support
**Date:** 2026-03-26
**Repo:** /Users/sheldon/Documents/claw/reference/gitclaw

---

## Feature 1: Git-Native Agent Architecture

### Overview

GitClaw's core innovation: **the agent itself is a git repository**. Identity, rules, memory, tools, and skills are all version-controlled files. This enables branching, diffing, and history tracking of agent configuration.

### Core Files

| File | Purpose |
|------|---------|
| `src/loader.ts` | Agent loading, scaffolding, inheritance, dependencies |
| `src/context.ts` | Context snapshot for voice and agent prompts |
| `agent.yaml` | Agent manifest (model, tools, runtime config) |

### Implementation Analysis

#### Agent Loading Pipeline (`loader.ts`)

The `loadAgent()` function orchestrates a multi-phase loading process:

```
loadAgent(dir)
  1. Parse agent.yaml as AgentManifest
  2. Load environment config (config/default.yaml + config/<env>.yaml)
  3. Create .gitagent/ directory and session state
  4. Resolve inheritance (if extends field present)
  5. Resolve dependencies (git clones into .gitagent/deps/)
  6. Discover and load plugins
  7. Validate compliance rules
  8. Read identity files (SOUL.md, RULES.md, DUTIES.md, AGENTS.md)
  9. Build system prompt from all parts
  10. Discover skills, knowledge, workflows, sub-agents, examples
  11. Resolve model (env override > CLI flag > manifest preferred)
  12. Return LoadedAgent
```

#### AgentManifest Structure

```typescript
interface AgentManifest {
  spec_version: string;
  name: string;
  version: string;
  description: string;
  author?: string;
  license?: string;
  tags?: string[];
  metadata?: Record<string, string | number | boolean>;
  model: {
    preferred: string;           // "provider:model" format
    fallback: string[];
    constraints?: {
      temperature?: number;
      max_tokens?: number;
      top_p?: number;
      top_k?: number;
      stop_sequences?: string[];
    };
  };
  tools: string[];               // Tool names
  skills?: string[];              // Skill filter
  runtime: {
    max_turns: number;
    timeout?: number;
  };
  extends?: string;              // Parent agent git URL
  dependencies?: Array<{
    name: string;
    source: string;              // Git URL
    version: string;             // Git branch/tag
    mount: string;
  }>;
  agents?: Record<string, any>;  // Sub-agents
  delegation?: { mode: "auto" | "explicit" | "router"; router?: string };
  compliance?: Record<string, any>;
  plugins?: Record<string, PluginConfig>;
}
```

#### System Prompt Composition

The system prompt is assembled from multiple sources (lines 257-350):

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

Agents can extend parent agents via the `extends` field in agent.yaml:

```yaml
extends: "https://github.com/user/base-agent.git"
```

**Resolution process:**
1. Clone parent into `.gitagent/deps/<parent-name>`
2. Deep merge parent manifest with child manifest (child wins)
3. Union tool lists (child shadows duplicates)
4. Append parent RULES.md to child's RULES.md

**Technical Note:** Uses `execSync` with `2>/dev/null || true` to silently fail if clone doesn't work, then continues without parent. This is a "fail open" approach.

#### Dependency System

Similar to inheritance but for shared resources:

```yaml
dependencies:
  - name: shared-utils
    source: "https://github.com/org/utils.git"
    version: "v1.0"
    mount: "./vendor"
```

Dependencies are cloned into `.gitagent/deps/` and can be referenced in the agent's context.

#### Context Resolution (`context.ts`)

Two context functions for different use cases:

1. **`getContextSnapshot(agentDir, branch)`** - Returns raw content:
   - MEMORY.md (from `.gitagent/memory/` or `memory/`)
   - Chat summary file (`.gitagent/chat-summary-{branch}.md`)
   - Recent chat history (last 30 messages)
   - Recent mood entries

2. **`getVoiceContext(agentDir, branch)`** - Returns formatted string for voice LLM:
   - Awakening Mode: Fresh agent with no memory
   - Growing Mode: Early memories (<400 chars)
   - Normal Mode: Full memory with reconnection greeting
   - Includes mood patterns, session summary, recent conversation

3. **`getAgentContext(agentDir, branch)`** - Richer context for run_agent:
   - Similar to voice context but with more memory tokens (1200 vs 300)

**Token Estimation:** Uses a rough heuristic of 4 characters per token.

### Clever Solutions

1. **Auto-scaffolding**: If `agent.yaml` doesn't exist, `ensureRepo()` creates it with sensible defaults
2. **.gitignore management**: Automatically adds `.gitagent/` to .gitignore if missing
3. **Deep merge utility**: Reusable `deepMerge()` function handles nested object merging
4. **Silent failures**: Git operations that fail don't crash the agent (graceful degradation)
5. **Memory search path**: Checks both `.gitagent/memory/MEMORY.md` and `memory/MEMORY.md`

### Technical Debt / Issues

1. **`execSync` usage** (lines 161, 207): Git operations are synchronous, blocking the event loop. Should use `exec()` or a git library.

2. **Silent error swallowing**: Clone failures are silently ignored (`2>/dev/null || true`). No logging or user notification.

3. **Model string validation**: `parseModelString()` only checks for colon presence. Invalid provider names will fail at `getModel()` later.

4. **Mutable accumulators** (context.ts lines 91-92): `accText` and `accThinking` are shared mutable state in the closure. Concurrent queries would interfere.

5. **Session state written synchronously**: `writeSessionState()` is async but awaited, but file I/O could fail silently.

6. **No manifest schema validation**: YAML is parsed directly to `AgentManifest` without runtime validation.

### Edge Cases Handled

- No SOUL.md, RULES.md, DUTIES.md: Uses empty strings, skipped in prompt
- Missing .gitignore: Created automatically
- Clone fails: Continues without parent/dependency
- No memory file: Awakening mode triggered in context
- Empty agent.yaml: Minimal defaults applied

---

## Feature 2: SDK Programmatic Interface

### Overview

TypeScript SDK for in-process agent execution with streaming responses, tool callbacks, and lifecycle hooks. Designed to mirror the Claude Agent SDK pattern. Allows programmatic use of GitClaw agents in Node.js applications.

### Core Files

| File | Purpose |
|------|---------|
| `src/sdk.ts` | Main `query()` function implementation |
| `src/sdk-types.ts` | All type definitions for SDK |
| `src/sdk-hooks.ts` | Programmatic hook wrapper |
| `src/exports.ts` | Public API exports |

### Implementation Analysis

#### Query Function Architecture

The `query()` function (lines 81-511) is the main SDK entry point:

```typescript
export function query(options: QueryOptions): Query
```

Returns a `Query` object that implements `AsyncGenerator<GCMessage, void, undefined>`.

#### Message Types (GCMessage)

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

Internal implementation uses a custom Channel<T> (lines 28-65):

```typescript
interface Channel<T> {
  push(v: T): void;           // Send message
  finish(): void;              // Close channel
  pull(): Promise<IteratorResult<T>>;  // Receive message
}
```

**Implementation:**
- Uses a buffer array for backpressure
- Promise-based waiting when buffer is empty
- Done flag for graceful closure

#### Event Mapping

Subscribes to `pi-agent-core` `AgentEvent` and maps to GCMessage:

| AgentEvent | GCM essage |
|------------|------------|
| agent_start | `{type: "system", subtype: "session_start"}` |
| message_update (text_delta) | `{type: "delta", deltaType: "text", content}` |
| message_update (thinking_delta) | `{type: "delta", deltaType: "thinking", content}` |
| message_end | `{type: "assistant", ...}` |
| tool_execution_start | `{type: "tool_use", ...}` |
| tool_execution_end | `{type: "tool_result", ...}` |
| agent_end | `{type: "system", subtype: "session_end"}` |

#### Streaming Accumulators

Lines 91-92 declare mutable accumulators:
```typescript
let accText = "";
let accThinking = "";
```

These accumulate streaming deltas within a single message, then reset when `message_end` arrives. **Issue**: These are shared mutable state in the closure scope.

#### Tool Injection

SDK allows providing external tools via `options.tools`:

```typescript
if (options.tools) {
  const converted = options.tools.map(toAgentTool);
  tools = [...tools, ...converted];
}
```

Tools are converted via `toAgentTool()` and merged with built-in + declarative + plugin tools.

#### Hook System

Two hook systems supported:

1. **Script-based hooks** (from `hooks.ts`):
   - Defined in YAML
   - Execute as shell scripts
   - Wrapped via `wrapToolWithHooks()`

2. **Programmatic hooks** (from `sdk-hooks.ts`):
   - Defined as TypeScript functions
   - Execute in-process
   - Wrapped via `wrapToolWithProgrammaticHooks()`

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

#### Multi-Turn Support

The `prompt` option accepts either:
- `string`: Single prompt
- `AsyncIterable<GCUserMessage>`: Multi-turn conversation

```typescript
if (typeof options.prompt === "string") {
  await agent.prompt(options.prompt);
} else {
  for await (const userMsg of options.prompt) {
    pushMsg({ type: "user", content: userMsg.content });
    await agent.prompt(userMsg.content);
  }
}
```

#### Sandbox Support

Options `repo` and `sandbox` are mutually exclusive (line 107-109):
```typescript
if (options.repo && options.sandbox) {
  throw new Error("repo and sandbox options are mutually exclusive");
}
```

#### Query Interface

```typescript
interface Query extends AsyncGenerator<GCMessage, void, undefined> {
  abort(): void;                    // Abort the running query
  steer(_message: string): void;     // Placeholder for steering
  sessionId(): string;              // Get current session ID
  manifest(): AgentManifest;        // Get loaded manifest (throws if not loaded)
  messages(): GCMessage[];          // Get all collected messages
}
```

**Note:** `steer()` is explicitly a placeholder (lines 472-475):
```typescript
steer(_message: string) {
  // Steering requires agent reference — for now this is a placeholder.
  // Full steering support would require exposing the Agent instance.
}
```

#### Error Handling

1. **LLM errors** (lines 326-338): Even if stopReason is "error", the assistant message is still emitted so callers can inspect `errorMessage`.

2. **Catch block** (lines 438-464): On any error:
   - Finalizes local session
   - Stops sandbox
   - Fires `onError` hook (non-blocking)
   - Pushes error system message
   - Finishes channel

3. **Non-blocking hooks**: `post_response` and `onSessionStart` hooks are fired with `.catch(() => {})` to not block execution.

#### Local Repo Mode Integration

When `options.repo` is provided:
1. Uses `initLocalSession()` to clone/fetch repo
2. Sets `dir` to session directory
3. Session branch auto-commits on completion

### SDK Usage Example

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

### Clever Solutions

1. **AsyncGenerator protocol**: Query implements standard async iteration, works with `for await...of`
2. **Channel buffering**: Decouples producer (agent events) from consumer (async iteration)
3. **Tool collision detection**: Warns if plugin tools conflict with existing tools
4. **Message accumulation**: Collects all messages for `messages()` method
5. **Manifest lazy loading**: `manifest()` throws if called before load completes

### Technical Debt / Issues

1. **Placeholder `steer()`**: Full steering requires agent reference not currently exposed
2. **Mutable accumulators**: `accText` and `accThinking` shared across async operations
3. **No backpressure**: Channel buffer could grow unbounded
4. **Silent hook failures**: `.catch(() => {})` swallows all hook errors silently
5. **No abort propagation**: `AbortController` created but signal not always passed through
6. **Manifest as any**: Line 320 casts `raw as any` then `raw as AssistantMessage` without validation

### Edge Cases Handled

- Mutually exclusive repo/sandbox options: Throws error
- Missing GITHUB_TOKEN: Throws with helpful message
- Tool collisions: Skips duplicate tools with warning
- Empty prompt: Handled (goes to REPL mode in CLI)
- Multi-turn with async iterable: Properly sequences messages
- LLM error with partial response: Emits error message but still returns response

---

## Feature 3: Multi-Model Support

### Overview

Unified interface for multiple LLM providers (Anthropic, OpenAI, Google, xAI, Groq, Mistral) via `@mariozechner/pi-ai` integration. Supports fallback chains and per-model constraints.

### Core Files

| File | Purpose |
|------|---------|
| `src/loader.ts` | Model resolution and loading |
| `src/index.ts` | CLI model selection and API key validation |
| `src/config.ts` | Environment-based model override |

### Implementation Analysis

#### Model Resolution Priority

Model selection follows this priority (loader.ts line 355):

```
1. envConfig.model_override  (from config/<env>.yaml)
2. modelFlag                  (from --model CLI argument)
3. manifest.model.preferred   (from agent.yaml)
```

If no model is configured after all three, throws error with helpful message.

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

**Parsing** (loader.ts lines 68-79):
```typescript
function parseModelString(modelStr: string): { provider: string; modelId: string } {
  const colonIndex = modelStr.indexOf(":");
  if (colonIndex === -1) {
    throw new Error(`Invalid model format: "${modelStr}". Expected "provider:model"`);
  }
  return {
    provider: modelStr.slice(0, colonIndex),
    modelId: modelStr.slice(colonIndex + 1),
  };
}
```

#### Model Instantiation

Uses `getModel()` from `@mariozechner/pi-ai` (line 363):
```typescript
const { provider, modelId } = parseModelString(modelStr);
const model = getModel(provider as any, modelId as any);
```

The `as any` casts bypass TypeScript's provider validation. Invalid providers will fail at runtime when the agent tries to use them.

#### Fallback Chain

The `manifest.model.fallback` array provides fallback models:

```yaml
model:
  preferred: "openai:gpt-4o"
  fallback:
    - "anthropic:claude-sonnet-4-5-20250929"
    - "openai:gpt-4o-mini"
```

**Note:** The fallback array is defined in the manifest but the actual fallback logic is handled by `@mariozechner/pi-ai` (the `getModel()` function returns a model that may have fallback support built-in).

#### Model Constraints

Constraints are specified per-model in agent.yaml:

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

**Constraint mapping** (loader.ts lines 259-270 and index.ts lines 525-532):

```typescript
// SDK path
const constraints = options.constraints ?? loaded.manifest.model.constraints;
if (constraints) {
  const c = constraints as any;
  if (c.temperature !== undefined) modelOptions.temperature = c.temperature;
  if (c.maxTokens !== undefined) modelOptions.maxTokens = c.maxTokens;
  if (c.max_tokens !== undefined) modelOptions.maxTokens = c.max_tokens;
  if (c.topP !== undefined) modelOptions.topP = c.topP;
  if (c.top_p !== undefined) modelOptions.topP = c.top_p;
  if (c.topK !== undefined) modelOptions.topK = c.topK;
  if (c.top_k !== undefined) modelOptions.topK = c.top_k;
}
```

**Technical Note:** Both camelCase and snake_case variants are supported, with the SDK options (`maxTokens`, `topP`) taking precedence if both are set.

#### Environment-Based Override

`config.ts` provides multi-environment config:

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

This allows:
- `config/default.yaml` - Base config
- `config/production.yaml` - Production overrides
- `config/dev.yaml` - Development overrides

Set via `--env production` or `GITCLAW_ENV=production`.

#### API Key Validation

The CLI validates required API keys before running (index.ts lines 468-484):

```typescript
const apiKeyEnvVars: Record<string, string> = {
  anthropic: "ANTHROPIC_API_KEY",
  openai: "OPENAI_API_KEY",
  google: "GOOGLE_API_KEY",
  xai: "XAI_API_KEY",
  groq: "GROQ_API_KEY",
  mistral: "MISTRAL_API_KEY",
};

const provider = loaded.model.provider;
const envVar = apiKeyEnvVars[provider];
if (envVar && !process.env[envVar]) {
  console.error(red(`Error: ${envVar} environment variable is not set.`));
  process.exit(1);
}
```

#### Voice Mode Model Override

Voice mode allows model override via `--model` flag (index.ts lines 399-405):

```typescript
const cleanup = await startVoiceServer({
  adapter: adapterBackend,
  adapterConfig: { apiKey },
  agentDir: dir,
  model,    // CLI --model flag
  env,
});
```

### Supported Providers

| Provider | Env Variable | Example Model |
|----------|--------------|---------------|
| anthropic | ANTHROPIC_API_KEY | claude-sonnet-4-5-20250929 |
| openai | OPENAI_API_KEY | gpt-4o, gpt-4o-mini |
| google | GOOGLE_API_KEY | gemini-2.0-flash |
| xai | XAI_API_KEY | grok-3 |
| groq | GROQ_API_KEY | llama-3.3-70b |
| mistral | MISTRAL_API_KEY | mistral-large |

### Token Estimation

Used in context.ts for truncating context to fit model limits:

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

**Note:** This is a rough heuristic. Actual tokenizers vary (4-5 chars per token for English).

### Clever Solutions

1. **Snake_case/camelCase compatibility**: Both `max_tokens` and `maxTokens` work
2. **Silent fallback**: If preferred model fails, pi-ai handles fallback
3. **Environment config**: Easy per-environment model override
4. **API key validation**: Fails fast with clear error message
5. **Provider-agnostic**: Same interface regardless of provider

### Technical Debt / Issues

1. **Minimal model string validation**: Only checks for colon; invalid providers fail at runtime
2. **`as any` casts** (line 363): Type safety bypassed with `as any` on provider
3. **Rough token estimation**: 4 chars/token is approximate; could under/over estimate
4. **No explicit fallback execution**: Fallback chain is implicit in pi-ai; not visible in code
5. **Provider enum not enforced**: String-based provider names, no compile-time checking
6. **pi-ai as internal dependency**: Model support tied to specific version of pi-ai

### Edge Cases Handled

- No model configured: Throws with helpful setup instructions
- Invalid model format (no colon): Throws with expected format
- Missing API key: Exits with clear error message
- Empty fallback array: Works fine (no fallbacks)
- Both snake_case and camelCase constraints: Both processed, SDK takes precedence

---

## Cross-Feature Interactions

### loader.ts → sdk.ts

The SDK uses `loadAgent()` from loader.ts to get the loaded agent including:
- `systemPrompt`: Pre-built from all components
- `model`: Resolved model instance
- `skills`, `knowledge`, `workflows`: Discovered metadata
- `plugins`: Loaded plugins with tools

### context.ts → sdk.ts / index.ts

Both SDK and CLI use context functions for:
- Voice mode: `getVoiceContext()` for voice LLM system prompt
- Agent mode: `getAgentContext()` for run_agent context
- Session resumption: Chat history and summary

### Multi-Model → loader.ts

Model resolution happens during agent loading:
1. `loadEnvConfig()` loads environment-specific config
2. `loadAgent()` resolves model string
3. Model is passed to `Agent` constructor

---

## Summary

### Git-Native Agent Architecture

**Strengths:**
- Elegant concept: agent as git repo enables natural versioning
- Rich manifest with inheritance and dependencies
- Auto-scaffolding reduces friction
- Modular system prompt composition

**Weaknesses:**
- Synchronous git operations block event loop
- Silent failures in inheritance/dependencies
- No manifest validation

### SDK

**Strengths:**
- Clean AsyncGenerator-based streaming API
- Good message type coverage
- Dual hook systems (script + programmatic)
- Multi-turn support

**Weaknesses:**
- Mutable accumulators in closure
- `steer()` is placeholder
- Silent hook failure swallowing

### Multi-Model

**Strengths:**
- Simple provider:model string format
- Good API key validation
- Environment-based config override
- Constraint support (temperature, tokens, etc.)

**Weaknesses:**
- Validation deferred to runtime
- Relies on internal pi-ai for fallback
- Token estimation is rough
