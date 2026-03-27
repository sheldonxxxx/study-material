# Security and Performance Patterns: GitClaw

## Project Overview

**GitClaw** is a universal git-native multimodal AI agent framework. This analysis examines security patterns for secrets management, authentication, input validation, and performance patterns including caching, pagination, and resource management.

---

## Security Patterns

### 1. Secrets Management

#### Environment Variable Handling

GitClaw uses environment variables extensively for API keys and tokens, with multiple layers of precedence:

| Source | Variable(s) | Purpose |
|--------|-------------|---------|
| CLI flag | `--pat` | Personal access token for repo operations |
| Environment | `GITHUB_TOKEN`, `GIT_TOKEN` | Git authentication tokens |
| Environment | `ANTHROPIC_API_KEY` | Claude agent API |
| Environment | `OPENAI_API_KEY` | OpenAI voice/realtime API |
| Environment | `GEMINI_API_KEY` | Gemini voice API |
| Environment | `COMPOSIO_API_KEY` | Composio integration |
| Environment | `TELEGRAM_BOT_TOKEN` | Telegram messaging |
| Environment | `GITCLAW_ENV` | Environment selector for config |
| Environment | `COMPOSIO_USER_ID` | Composio user identifier |

**Token resolution precedence** (from `sdk.ts:115`, `index.ts:318`, `sandbox.ts:61-63`):
```typescript
const token = options.repo.token || process.env.GITHUB_TOKEN || process.env.GIT_TOKEN;
const apiKey = process.env.ANTHROPIC_API_KEY;
```

#### .env File Loading

Custom .env parser in `voice/server.ts:357-379` and `index.ts:363-376`:

```typescript
function loadEnvFile(dir: string) {
  const envPath = join(dir, ".env");
  const content = readFileSync(envPath, "utf-8");
  for (const line of content.split("\n")) {
    const trimmed = line.trim();
    if (!trimmed || trimmed.startsWith("#")) continue; // Skip comments
    const eq = trimmed.indexOf("=");
    if (eq < 1) continue;
    const key = trimmed.slice(0, eq).trim();
    let val = trimmed.slice(eq + 1).trim();
    // Strip surrounding quotes
    if ((val.startsWith('"') && val.endsWith('"')) || (val.startsWith("'") && val.endsWith("'"))) {
      val = val.slice(1, -1);
    }
    // Won't overwrite existing vars (preserves shell env precedence)
    if (!process.env[key]) {
      process.env[key] = val;
    }
  }
}
```

**Security note**: The parser strips quotes but does not escape or sanitize values. Values with special characters are passed as-is to `process.env`.

#### Git URL Token Embedding

In `session.ts:26-33`, tokens are embedded in git URLs for authentication:

```typescript
function authedUrl(url: string, token: string): string {
  // https://github.com/org/repo → https://<token>@github.com/org/repo
  return url.replace(/^https:\/\//, `https://${token}@`);
}

function cleanUrl(url: string): string {
  // Strip PAT from remote URL after operations
  return url.replace(/^https:\/\/[^@]+@/, "https://");
}
```

The `finalize()` method strips the token from the remote URL after operations complete (`session.ts:146-151`), preventing token leakage in `.git/config`.

#### Plugin Config Environment Interpolation

In `plugins.ts:74-77`, config values support `${ENV_VAR}` syntax:

```typescript
if (typeof value === "string") {
  value = value.replace(/\$\{(\w+)\}/g, (_, envName) => process.env[envName] || "");
}
```

Config resolution priority (`plugins.ts:71-83`):
1. User-provided config value
2. Environment variable (via `prop.env`)
3. Default value

### 2. Authentication and Authorization

#### Local Session Authentication

The `LocalSession` system (`session.ts:57-155`) manages git-based authentication:

- Tokens are embedded in git remote URLs during clone/fetch operations
- Sessions use git branches (`gitclaw/session-<hex>`) for isolation
- Tokens are stripped from URLs on `finalize()`

#### Tool Access Control

The SDK supports tool allowlisting/denylisting (`sdk.ts:195-203`):

```typescript
if (options.allowedTools) {
  const allowed = new Set(options.allowedTools);
  tools = tools.filter((t) => allowed.has(t.name));
}
if (options.disallowedTools) {
  const denied = new Set(options.disallowedTools);
  tools = tools.filter((t) => !denied.has(t.name));
}
```

#### Sandbox Isolation

Sandbox execution (`sandbox.ts`) uses the `gitmachine` package with e2b provider for code execution isolation. The sandbox receives a limited set of environment variables and operates on a cloned repository.

### 3. Input Validation and Sanitization

#### TypeBox Schema Validation

All built-in tools use TypeBox for runtime schema validation (`tools/shared.ts:15-59`):

```typescript
export const cliSchema = Type.Object({
  command: Type.String({ description: "Shell command to execute" }),
  timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (default: 120)" })),
});

export const readSchema = Type.Object({
  path: Type.String({ description: "Path to the file to read" }),
  offset: Type.Optional(Type.Number({ description: "Line number to start from (1-indexed)" })),
  limit: Type.Optional(Type.Number({ description: "Maximum number of lines to read" })),
});
```

#### Plugin Manifest Validation

In `plugins.ts:40-58`, plugin manifests are validated:

```typescript
function validatePluginManifest(manifest: any, pluginDir: string): manifest is PluginManifest {
  if (!manifest || typeof manifest !== "object") return false;
  if (!manifest.id || typeof manifest.id !== "string") return false;
  if (!KEBAB_RE.test(manifest.id)) {
    console.warn(`Plugin "${manifest.id}": id must be kebab-case`);
    return false;
  }
  if (!manifest.name || !manifest.version || !manifest.description) return false;
  return true;
}
```

#### Branch Name Sanitization

In `chat-history.ts:9-11`:

```typescript
function sanitizeBranch(branch: string): string {
  return branch.replace(/\//g, "__");
}
```

#### Path Traversal Prevention

The `resolveSandboxPath` function (`tools/shared.ts:109-115`) resolves paths relative to sandbox root:

```typescript
export function resolveSandboxPath(path: string, repoRoot: string): string {
  if (path.startsWith("~/") || path === "~") {
    path = homedir() + path.slice(1);
  }
  if (path.startsWith("/")) return path;
  return repoRoot.endsWith("/") ? repoRoot + path : repoRoot + "/" + path;
}
```

### 4. Security Concerns

#### YAML Deserialization

GitClaw uses `js-yaml` for parsing configuration files (`config.ts:31-32`):

```typescript
const raw = await readFile(path, "utf-8");
return (yaml.load(raw) as Record<string, any>) || {};
```

**Note**: `js-yaml` uses full schema by default which can trigger arbitrary code execution with specially crafted YAML. However, in this usage, the YAML comes from local files controlled by the agent owner, not external sources.

#### CLI Tool Environment Exposure

The CLI tool (`tools/cli.ts:27-31`) passes the full `process.env` to spawned processes:

```typescript
const child = spawn("sh", ["-c", command], {
  cwd,
  stdio: ["ignore", "pipe", "pipe"],
  env: { ...process.env }, // Full environment leaked to subprocess
});
```

This exposes all environment variables, including API keys, to shell commands executed by the agent.

#### No User Authentication

GitClaw is designed as a single-user local development tool. There is no built-in user authentication, session management for multiple users, or access control beyond tool allowlisting.

---

## Performance Patterns

### 1. Caching

#### Composio Tools Cache

In `composio/adapter.ts:14-16, 24-28`, tools are cached with a 30-second TTL:

```typescript
private cachedTools: GCToolDefinition[] | null = null;
private cacheExpiry = 0;
private static CACHE_TTL = 30_000; // 30s

async getTools(): Promise<GCToolDefinition[]> {
  const now = Date.now();
  if (this.cachedTools && now < this.cacheExpiry) return this.cachedTools;
  // ... fetch tools ...
  this.cachedTools = tools;
  this.cacheExpiry = now + ComposioAdapter.CACHE_TTL;
  return tools;
}
```

The cache is invalidated when connections change (`disconnect()` at line 96).

#### Composio Auth Config Cache

In `composio/client.ts:36-37`:

```typescript
private authConfigCache = new Map<string, string>();
```

Prevents repeated auth config fetches for the same toolkit.

#### Schedule Runner Task Tracking

In `schedule-runner.ts:18-20`:

```typescript
const activeTasks = new Map<string, ScheduledTask>();
const activeTimers = new Map<string, ReturnType<typeof setTimeout>>();
const runningJobs = new Set<string>(); // Prevents concurrent execution
```

### 2. Pagination and Output Limits

#### CLI Output Truncation

In `tools/cli.ts:83-88` and `tools/shared.ts:63-69`:

```typescript
// ~100KB max output
const MAX_OUTPUT = 100_000;

if (text.length > MAX_OUTPUT) {
  text = text.slice(-MAX_OUTPUT);
  text = `[output truncated, showing last ~100KB]\n${text}`;
}
```

#### Paginated File Reads

In `tools/shared.ts:75-106`:

```typescript
const MAX_LINES = 2000;
const MAX_BYTES = 100_000;

export function paginateLines(
  text: string,
  offset?: number,
  limit?: number,
): { text: string; hasMore: boolean; shownRange: [number, number]; totalLines: number } {
  const allLines = text.split("\n");
  const startLine = offset ? Math.max(0, offset - 1) : 0;
  const maxLines = limit ?? MAX_LINES;
  const endLine = Math.min(startLine + maxLines, totalLines);
  let selected = allLines.slice(startLine, endLine).join("\n");

  // Also truncate by bytes if needed
  if (Buffer.byteLength(selected, "utf-8") > MAX_BYTES) {
    selected = selected.slice(0, MAX_BYTES);
  }
  // ...
}
```

### 3. Chat History Management

#### Message Skipping

In `chat-history.ts:6-7, 21-24`:

```typescript
const SKIP_TYPES = new Set(["audio_delta", "agent_thinking"]);

export function appendMessage(agentDir: string, branch: string, msg: ServerMessage): void {
  if (SKIP_TYPES.has(msg.type)) return;
  if (msg.type === "transcript" && msg.partial) return; // Skip partial transcripts
  // ...
}
```

#### History Summarization

In `chat-history.ts:70-130`, when message count exceeds 10, history is summarized:

```typescript
export async function summarizeHistory(agentDir: string, branch: string): Promise<string> {
  const count = getMessageCount(agentDir, branch);
  if (count < 10) return ""; // Only summarize when 10+ messages

  // Truncate to last ~4000 chars for summarization prompt
  if (transcript.length > 4000) {
    transcript = transcript.slice(-4000);
  }
  // ...
}
```

### 4. Lazy Loading Patterns

#### Dynamic Plugin Loading

In `plugins.ts:241-272`, plugin entry points are dynamically imported:

```typescript
if (manifest.entry) {
  const { createPluginApi } = await import("./plugin-sdk.js");
  const entryPath = join(pluginDir, manifest.entry);
  const mod = await import(entryPath);
  if (typeof mod.register === "function") {
    await mod.register(api);
  }
}
```

#### Declarative Tool Discovery

In `tool-loader.ts`, tools are discovered from YAML files in the `tools/` directory and loaded on demand.

#### Async Iterator for Streaming

The SDK uses async generators (`sdk.ts:34-65`) for streaming responses:

```typescript
function createChannel<T>(): Channel<T> {
  const buffer: T[] = [];
  return {
    push(v: T) { /* ... */ },
    finish() { /* ... */ },
    pull(): Promise<IteratorResult<T>> { /* ... */ },
  };
}
```

### 5. File Operation Patterns

#### Append-Only JSONL for Chat History

In `chat-history.ts:29-30`:

```typescript
const line = JSON.stringify({ ts: Date.now(), msg }) + "\n";
appendFileSync(historyPath(agentDir, branch), line, "utf-8");
```

#### JSONL Schedule Logs

In `schedule-runner.ts:113-118`:

```typescript
const logEntry = JSON.stringify({ ts, success, result: result.slice(0, 5000) }) + "\n";
appendFileSync(logFile, logEntry, "utf-8");
```

### 6. Scheduler Concurrency Control

In `schedule-runner.ts:88-93`:

```typescript
export async function executeScheduledJob(schedule: ScheduleDefinition, opts: SchedulerOptions, disableAfterRun = false): Promise<void> {
  if (runningJobs.has(schedule.id)) {
    console.log(dim(`[scheduler] Skipping "${schedule.id}" — already running`));
    return;
  }
  runningJobs.add(schedule.id);
  // ...
  runningJobs.delete(schedule.id); // Clean up on completion
}
```

---

## Summary

### Security Posture

| Area | Status | Notes |
|------|--------|-------|
| Secrets management | Moderate | Environment variables with .env support; tokens stripped from git URLs after use |
| Input validation | Good | TypeBox schemas for all tool inputs; plugin manifest validation |
| Authentication | Limited | Git-based session isolation; no multi-user auth |
| Authorization | Basic | Tool allowlisting/denylist; sandbox isolation via e2b |
| Secrets exposure | Risk | CLI tool receives full process.env; YAML from local files only |

### Performance Characteristics

| Area | Pattern | Benefit |
|------|---------|---------|
| API caching | 30s TTL Composio tools | Reduces external API calls |
| Output limits | 100KB max truncation | Prevents LLM context overflow |
| File pagination | 2000 line / 100KB limits | Memory-efficient file operations |
| History management | Truncation + summarization | Bounded conversation size |
| Concurrency | Running jobs Set | Prevents duplicate scheduled job execution |
| Streaming | Async channel/iterator | Non-blocking response delivery |
