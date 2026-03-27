# pi-mono Security and Performance Patterns Analysis

## Executive Summary

The pi-mono project implements a multi-provider LLM agent runtime with comprehensive security controls and sophisticated performance optimizations. The architecture separates concerns cleanly: `@mariozechner/pi-ai` handles LLM provider abstraction, `@mariozechner/pi-agent-core` provides the agent runtime, and `@mariozechner/pi-coding-agent` delivers the CLI product.

---

## Security Patterns

### 1. Secrets Management

#### Environment Variable Handling

API keys are resolved through a layered hierarchy defined in `packages/ai/src/env-api-keys.ts`:

```typescript
// Priority order for API keys:
// 1. Runtime override (CLI --api-key)
// 2. OAuth token from auth.json (auto-refreshed)
// 3. API key from auth.json
// 4. Environment variable
// 5. Fallback resolver (models.json custom providers)
```

Provider-specific environment variable mapping:
```typescript
const envMap: Record<string, string> = {
  openai: "OPENAI_API_KEY",
  "azure-openai-responses": "AZURE_OPENAI_API_KEY",
  google: "GEMINI_API_KEY",
  // ... 15+ additional providers
};
```

#### Credential Storage (`packages/coding-agent/src/core/auth-storage.ts`)

Credentials are persisted in `auth.json` with file locking to prevent race conditions:

```typescript
// File permissions: 0o600 (user read/write only)
chmodSync(this.authPath, 0o600);

// File locking via proper-lockfile
const release = await lockfile.lock(this.authPath, {
  retries: {
    retries: 10,
    factor: 2,
    minTimeout: 100,
    maxTimeout: 10000,
    randomize: true,
  },
  stale: 30000,  // Lock considered stale after 30s
  onCompromised: (err) => { /* handle corrupted lock */ },
});
```

#### `.gitignore` Configuration

The root `.gitignore` properly excludes:
```
.env                    # Environment files
.pi_config/             # Agent configuration
*.tsbuildinfo           # Build artifacts
```

#### AWS Credentials Support

Bedrock provider supports multiple AWS credential sources:
- `AWS_PROFILE` - Named profile from `~/.aws/credentials`
- `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` - Standard IAM keys
- `AWS_BEARER_TOKEN_BEDROCK` - Bedrock API keys
- `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` - ECS task roles
- `AWS_WEB_IDENTITY_TOKEN_FILE` - IRSA (IAM Roles for Service Accounts)
- Google ADC via `~/.config/gcloud/application_default_credentials.json`

### 2. Authentication Implementation

#### OAuth Token Management

OAuth tokens are automatically refreshed with distributed locking:

```typescript
// Token refresh with file lock prevents race conditions
private async refreshOAuthTokenWithLock(providerId: OAuthProviderId) {
  return await this.storage.withLockAsync(async (current) => {
    // Double-checked locking pattern
    const cred = currentData[providerId];
    if (Date.now() < cred.expires) {
      return { apiKey: provider.getApiKey(cred) };
    }
    // Refresh token...
  });
}
```

#### Dynamic API Key Resolution

The agent supports dynamic API key resolution for expiring tokens:

```typescript
// From packages/agent/src/agent.ts
export interface AgentOptions {
  /** Resolves an API key dynamically for each LLM call.
   *  Useful for expiring tokens (e.g., GitHub Copilot OAuth). */
  getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;
}
```

#### Anthropic OAuth Detection

OAuth tokens are detected by prefix:
```typescript
function isOAuthToken(apiKey: string): boolean {
  return apiKey.includes("sk-ant-oat");
}
```

### 3. Input Validation and Sanitization

#### TypeBox Schema Validation

Tool arguments are validated against TypeBox schemas using AJV:

```typescript
// From packages/ai/src/utils/validation.ts
import AjvModule from "ajv";
import addFormatsModule from "ajv-formats";

let ajv: any = null;
if (canUseRuntimeCodegen()) {
  ajv = new Ajv({
    allErrors: true,
    strict: false,
    coerceTypes: true,
  });
  addFormats(ajv);
}

export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
  const validate = ajv.compile(tool.parameters);
  const args = structuredClone(toolCall.arguments);  // Clone before AJV mutation
  if (validate(args)) {
    return args;
  }
  // Format errors with instance path
  const errors = validate.errors?.map((err) =>
    `  - ${err.instancePath}: ${err.message}`
  ).join("\n");
  throw new Error(`Validation failed for tool "${toolCall.name}":\n${errors}`);
}
```

#### Browser Extension Compatibility

AJV validation is disabled in browser extension environments (Manifest V3 CSP restrictions):

```typescript
const isBrowserExtension = typeof globalThis !== "undefined" &&
  (globalThis as any).chrome?.runtime?.id !== undefined;

function canUseRuntimeCodegen(): boolean {
  if (isBrowserExtension) return false;
  try {
    new Function("return true;");
    return true;
  } catch {
    return false;
  }
}
```

#### Unicode Sanitization

Surrogate characters are sanitized to prevent encoding issues:

```typescript
// From packages/ai/src/utils/sanitize-unicode.ts
export function sanitizeSurrogates(text: string): string {
  // Replace lone surrogates with replacement character
  return text.replace(/[\u{D800}-\u{DFFF}]/gu, "\u{FFFD}");
}
```

Used in:
- `anthropic.ts`: `sanitizeSurrogates(block.text)`
- `amazon-bedrock.ts`: `sanitizeSurrogates(systemPrompt)`
- `google-gemini-cli.ts`: Applied via shared conversion

#### Streaming JSON Parsing

Partial JSON during streaming is handled safely:

```typescript
// From packages/ai/src/utils/json-parse.ts
import { parse as partialParse } from "partial-json";

export function parseStreamingJson<T = any>(partialJson: string | undefined): T {
  if (!partialJson || partialJson.trim() === "") {
    return {} as T;
  }
  // Try standard parsing first (fastest for complete JSON)
  try {
    return JSON.parse(partialJson) as T;
  } catch {
    // Try partial-json for incomplete streaming JSON
    try {
      const result = partialParse(partialJson);
      return (result ?? {}) as T;
    } catch {
      return {} as T;  // Fail gracefully
    }
  }
}
```

### 4. Tool Call Security

#### Tool Name Canonicalization

Claude Code tool names are normalized to prevent case-sensitivity issues:

```typescript
// From packages/ai/src/providers/anthropic.ts
const claudeCodeTools = [
  "Read", "Write", "Edit", "Bash", "Grep", "Glob",
  "AskUserQuestion", "EnterPlanMode", "ExitPlanMode",
  // ... more tools
];
const ccToolLookup = new Map(claudeCodeTools.map((t) => [t.toLowerCase(), t]));

const toClaudeCodeName = (name: string) =>
  ccToolLookup.get(name.toLowerCase()) ?? name;
```

#### Tool Call ID Normalization

Tool call IDs are sanitized to match provider requirements:

```typescript
// Anthropic: max 64 chars, alphanumeric + underscore/dash
function normalizeToolCallId(id: string): string {
  return id.replace(/[^a-zA-Z0-9_-]/g, "_").slice(0, 64);
}

// Bedrock: same pattern
function normalizeToolCallId(id: string): string {
  const sanitized = id.replace(/[^a-zA-Z0-9_-]/g, "_");
  return sanitized.length > 64 ? sanitized.slice(0, 64) : sanitized;
}
```

### 5. Transport Security

#### Browser SDK Compatibility

Anthropic SDK configured for browser use:
```typescript
const client = new Anthropic({
  apiKey,
  dangerouslyAllowBrowser: true,  // Required for browser extensions
});
```

---

## Performance Patterns

### 1. Streaming Architecture

#### Event Stream Implementation

Custom async iterator-based event stream from `packages/ai/src/utils/event-stream.ts`:

```typescript
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  push(event: T): void {
    if (this.done) return;
    if (this.isComplete(event)) {
      this.done = true;
      this.resolveFinalResult(this.extractResult(event));
    }
    // Deliver to waiting consumer or queue it
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      this.queue.push(event);
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        return;
      } else {
        const result = await new Promise<IteratorResult<T>>(
          (resolve) => this.waiting.push(resolve)
        );
        if (result.done) return;
        yield result.value;
      }
    }
  }
}
```

#### Partial Message Updates

Streaming updates are pushed incrementally during LLM generation:

```typescript
for await (const event of anthropicStream) {
  if (event.type === "text_delta") {
    block.text += event.delta.text;
    stream.push({
      type: "text_delta",
      contentIndex: index,
      delta: event.delta.text,
      partial: output,
    });
  }
  // Similar for thinking, tool calls, etc.
}
```

### 2. Caching Strategies

#### Prompt Caching

Multiple provider-specific caching mechanisms:

**Anthropic Cache Control:**
```typescript
// From packages/ai/src/providers/anthropic.ts
function getCacheControl(baseUrl: string, cacheRetention?: CacheRetention) {
  const retention = resolveCacheRetention(cacheRetention);
  if (retention === "none") return { retention };
  // 1h TTL only on anthropic.com, not proxies
  const ttl = retention === "long" && baseUrl.includes("api.anthropic.com") ? "1h" : undefined;
  return {
    retention,
    cacheControl: { type: "ephemeral", ...(ttl && { ttl }) },
  };
}
```

**AWS Bedrock Cache Points:**
```typescript
// From packages/ai/src/providers/amazon-bedrock.ts
function supportsPromptCaching(model: Model): boolean {
  const id = model.id.toLowerCase();
  // Claude 4.x, 3.7 Sonnet, 3.5 Haiku
  if (id.includes("-4-") || id.includes("-4.") ||
      id.includes("claude-3-7-sonnet") || id.includes("claude-3-5-haiku")) {
    return true;
  }
  return false;
}
```

#### Cache Retention Levels

```typescript
// From packages/ai/src/types.ts
export type CacheRetention = "none" | "short" | "long";
```

### 3. Context Management

#### Context Overflow Detection

Comprehensive detection across 40+ provider error patterns:

```typescript
// From packages/ai/src/utils/overflow.ts
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,                    // Anthropic
  /input is too long for requested model/i, // Amazon Bedrock
  /exceeds the context window/i,            // OpenAI
  /input token count.*exceeds the maximum/i, // Google
  /maximum prompt length is \d+/i,           // xAI
  /reduce the length of the messages/i,      // Groq
  // ... 20+ more patterns
];

export function isContextOverflow(message: AssistantMessage, contextWindow?: number): boolean {
  // Pattern 1: Error message matching
  if (message.stopReason === "error" && message.errorMessage) {
    if (OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!))) {
      return true;
    }
  }
  // Pattern 2: Silent overflow (z.ai style)
  if (contextWindow && message.stopReason === "stop") {
    const inputTokens = message.usage.input + message.usage.cacheRead;
    if (inputTokens > contextWindow) {
      return true;
    }
  }
  return false;
}
```

### 4. Concurrency Handling

#### Parallel vs Sequential Tool Execution

```typescript
// From packages/agent/src/agent-loop.ts
async function executeToolCalls(
  currentContext: AgentContext,
  assistantMessage: AssistantMessage,
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
): Promise<ToolResultMessage[]> {
  const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
  if (config.toolExecution === "sequential") {
    return executeToolCallsSequential(...);
  }
  return executeToolCallsParallel(...);
}

// Parallel execution with Promise.all
async function executeToolCallsParallel(...) {
  const runningCalls = runnableCalls.map((prepared) => ({
    prepared,
    execution: executePreparedToolCall(prepared, signal, emit),
  }));

  for (const running of runningCalls) {
    const executed = await running.execution;
    results.push(await finalizeExecutedToolCall(...));
  }
}
```

#### Steering and Follow-up Queues

```typescript
// From packages/agent/src/agent.ts
export interface AgentOptions {
  /** Steering mode: "all" = send all steering messages at once */
  steeringMode?: "all" | "one-at-a-time";
  /** Follow-up mode: "all" = send all follow-up messages at once */
  followUpMode?: "all" | "one-at-a-time";
}

// Queue management
steer(m: AgentMessage) {
  this.steeringQueue.push(m);  // Delivered after current turn
}

followUp(m: AgentMessage) {
  this.followUpQueue.push(m);  // Delivered when agent would stop
}
```

### 5. Retry and Backoff Logic

#### Rate Limit Header Parsing

```typescript
// From packages/ai/src/providers/google-gemini-cli.ts
export function extractRetryDelay(errorText: string, response?: Response | Headers): number | undefined {
  // Header-based extraction
  const headers = response instanceof Headers ? response : response?.headers;
  if (headers) {
    const retryAfter = headers.get("retry-after");
    // Parse Retry-After header
    const rateLimitReset = headers.get("x-ratelimit-reset");
    const rateLimitResetAfter = headers.get("x-ratelimit-reset-after");
  }

  // Body pattern extraction
  // "Your quota will reset after 39s"
  // "Please retry in Xs" or "Please retry in Xms"
  // "retryDelay": "34.074824224s"
}
```

#### Retry Configuration

```typescript
// From packages/ai/src/providers/google-gemini-cli.ts
const MAX_RETRIES = 3;
const BASE_DELAY_MS = 1000;
const MAX_EMPTY_STREAM_RETRIES = 2;
const EMPTY_STREAM_BASE_DELAY_MS = 500;
```

### 6. Connection Management

#### HTTP Handler Configuration

```typescript
// From packages/ai/src/providers/amazon-bedrock.ts
if (process.env.HTTP_PROXY || process.env.HTTPS_PROXY) {
  const nodeHttpHandler = await import("@smithy/node-http-handler");
  const proxyAgent = await import("proxy-agent");

  const agent = new proxyAgent.ProxyAgent();

  // Bedrock uses HTTP/2 by default since v3.798.0
  // Use NodeHttpHandler to support http agent
  config.requestHandler = new nodeHttpHandler.NodeHttpHandler({
    httpAgent: agent,
    httpsAgent: agent,
  });
} else if (process.env.AWS_BEDROCK_FORCE_HTTP1 === "1") {
  // Some custom endpoints require HTTP/1.1
  config.requestHandler = new nodeHttpHandler.NodeHttpHandler();
}
```

#### Abort Signal Handling

All async operations respect `AbortSignal`:

```typescript
// Agent prompt cancellation
abort() {
  this.abortController?.abort();
}

// Stream cancellation
if (options.signal?.aborted) {
  throw new Error("Request was aborted");
}

// Proxy reader cancellation
const abortHandler = () => {
  if (reader) {
    reader.cancel("Request aborted by user").catch(() => {});
  }
};
```

### 7. Token Budget Management

#### Thinking Budgets

```typescript
// From packages/ai/src/types.ts
export interface ThinkingBudgets {
  minimal?: number;
  low?: number;
  medium?: number;
  high?: number;
}

// Default budgets for Bedrock
const defaultBudgets: Record<ThinkingLevel, number> = {
  minimal: 1024,
  low: 2048,
  medium: 8192,
  high: 16384,
  xhigh: 16384,
};
```

### 8. Output Truncation

#### Bash Output Management

```typescript
// From packages/coding-agent/src/core/bash-executor.ts
const maxOutputBytes = DEFAULT_MAX_BYTES * 2;  // Rolling buffer

// Temp file for large output
if (totalBytes > DEFAULT_MAX_BYTES && !tempFilePath) {
  const id = randomBytes(8).toString("hex");
  tempFilePath = join(tmpdir(), `pi-bash-${id}.log`);
  tempFileStream = createWriteStream(tempFilePath);
}

// Rolling buffer maintains last N bytes
while (outputBytes > maxOutputBytes && outputChunks.length > 1) {
  const removed = outputChunks.shift()!;
  outputBytes -= removed.length;
}
```

---

## Architecture Observations

### Strengths

1. **Layered Security**: Environment variables, file-based credentials with locking, OAuth refresh all work together
2. **Provider Abstraction**: Unified interface across 15+ LLM providers with provider-specific optimizations
3. **Streaming-First Design**: Custom event stream implementation enables real-time updates
4. **Graceful Degradation**: Partial JSON parsing, silent overflow detection, CSP-aware validation
5. **Concurrent Tool Execution**: Parallel/sequential modes with proper await handling

### Areas of Note

1. **File Lock Retry Logic**: Uses exponential backoff with jitter (10 retries, 100ms-10s range)
2. **OAuth Token Storage**: Refresh tokens persisted to `auth.json` - encryption at rest depends on filesystem permissions
3. **Browser Extension Mode**: Validation falls back to pass-through when AJV unavailable
4. **Context Window Management**: Relies on provider error messages; some providers (Ollama) silently truncate

---

## Key Files Reference

| Category | File |
|----------|------|
| API Key Resolution | `packages/ai/src/env-api-keys.ts` |
| Credential Storage | `packages/coding-agent/src/core/auth-storage.ts` |
| Input Validation | `packages/ai/src/utils/validation.ts` |
| Unicode Sanitization | `packages/ai/src/utils/sanitize-unicode.ts` |
| JSON Streaming | `packages/ai/src/utils/json-parse.ts` |
| Event Stream | `packages/ai/src/utils/event-stream.ts` |
| Context Overflow | `packages/ai/src/utils/overflow.ts` |
| Agent Loop | `packages/agent/src/agent-loop.ts` |
| Tool Execution | `packages/agent/src/agent.ts` |
| Anthropic Provider | `packages/ai/src/providers/anthropic.ts` |
| Bedrock Provider | `packages/ai/src/providers/amazon-bedrock.ts` |
| Gemini CLI Provider | `packages/ai/src/providers/google-gemini-cli.ts` |
| Bash Execution | `packages/coding-agent/src/core/bash-executor.ts` |
| Proxy Streaming | `packages/agent/src/proxy.ts` |
