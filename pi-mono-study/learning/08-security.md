# Security Patterns

## Overview

pi-mono implements comprehensive security controls across secrets management, authentication, input validation, and transport security. The architecture demonstrates a layered approach to security with defense in depth.

## Secrets Management

### API Key Resolution Hierarchy

API keys are resolved through a layered priority order (from `packages/ai/src/env-api-keys.ts`):

1. Runtime override (CLI `--api-key`)
2. OAuth token from `auth.json` (auto-refreshed)
3. API key from `auth.json`
4. Environment variable
5. Fallback resolver (models.json custom providers)

### Environment Variable Mapping

Provider-specific environment variables are mapped:
```typescript
const envMap: Record<string, string> = {
  openai: "OPENAI_API_KEY",
  "azure-openai-responses": "AZURE_OPENAI_API_KEY",
  google: "GEMINI_API_KEY",
  // ... 15+ additional providers
};
```

### Credential Storage

Credentials are persisted in `auth.json` with security measures:

**File permissions:** `0o600` (user read/write only)
```typescript
chmodSync(this.authPath, 0o600);
```

**File locking:** Uses `proper-lockfile` to prevent race conditions:
```typescript
const release = await lockfile.lock(this.authPath, {
  retries: {
    retries: 10,
    factor: 2,
    minTimeout: 100,
    maxTimeout: 10000,
    randomize: true,
  },
  stale: 30000,
});
```

### AWS Credentials Support

Bedrock provider supports multiple credential sources:
- `AWS_PROFILE` - Named profile from `~/.aws/credentials`
- `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` - Standard IAM keys
- `AWS_BEARER_TOKEN_BEDROCK` - Bedrock API keys
- `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` - ECS task roles
- `AWS_WEB_IDENTITY_TOKEN_FILE` - IRSA
- Google ADC via `~/.config/gcloud/application_default_credentials.json`

### Gitignore Configuration

Root `.gitignore` properly excludes:
```
.env                    # Environment files
.pi_config/             # Agent configuration
*.tsbuildinfo           # Build artifacts
```

## Authentication Implementation

### OAuth Token Management

OAuth tokens are automatically refreshed with distributed locking (double-checked locking pattern):
```typescript
private async refreshOAuthTokenWithLock(providerId: OAuthProviderId) {
  return await this.storage.withLockAsync(async (current) => {
    const cred = currentData[providerId];
    if (Date.now() < cred.expires) {
      return { apiKey: provider.getApiKey(cred) };
    }
    // Refresh token...
  });
}
```

### Dynamic API Key Resolution

The agent supports dynamic API key resolution for expiring tokens:
```typescript
export interface AgentOptions {
  /** Resolves an API key dynamically for each LLM call.
   *  Useful for expiring tokens (e.g., GitHub Copilot OAuth). */
  getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;
}
```

### Anthropic OAuth Detection

OAuth tokens are detected by prefix:
```typescript
function isOAuthToken(apiKey: string): boolean {
  return apiKey.includes("sk-ant-oat");
}
```

## Input Validation and Sanitization

### TypeBox Schema Validation

Tool arguments are validated against TypeBox schemas using AJV:
```typescript
import AjvModule from "ajv";
import addFormatsModule from "ajv-formats";

const ajv = new Ajv({
  allErrors: true,
  strict: false,
  coerceTypes: true,
});
addFormats(ajv);

export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
  const validate = ajv.compile(tool.parameters);
  const args = structuredClone(toolCall.arguments);  // Clone before mutation
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

### Browser Extension Compatibility

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

### Unicode Sanitization

Surrogate characters are sanitized to prevent encoding issues:
```typescript
export function sanitizeSurrogates(text: string): string {
  return text.replace(/[\u{D800}-\u{DFFF}]/gu, "\u{FFFD}");
}
```

Used in provider implementations for anthropic, amazon-bedrock, and google-gemini-cli.

### Streaming JSON Parsing

Partial JSON during streaming is handled safely:
```typescript
import { parse as partialParse } from "partial-json";

export function parseStreamingJson<T = any>(partialJson: string | undefined): T {
  if (!partialJson || partialJson.trim() === "") {
    return {} as T;
  }
  try {
    return JSON.parse(partialJson) as T;
  } catch {
    try {
      const result = partialParse(partialJson);
      return (result ?? {}) as T;
    } catch {
      return {} as T;  // Fail gracefully
    }
  }
}
```

## Tool Call Security

### Tool Name Canonicalization

Claude Code tool names are normalized to prevent case-sensitivity issues:
```typescript
const ccToolLookup = new Map(claudeCodeTools.map((t) => [t.toLowerCase(), t]));

const toClaudeCodeName = (name: string) =>
  ccToolLookup.get(name.toLowerCase()) ?? name;
```

### Tool Call ID Normalization

Tool call IDs are sanitized to match provider requirements:
```typescript
// Anthropic: max 64 chars, alphanumeric + underscore/dash
function normalizeToolCallId(id: string): string {
  return id.replace(/[^a-zA-Z0-9_-]/g, "_").slice(0, 64);
}
```

## Context Overflow Detection

Comprehensive detection across 40+ provider error patterns:
```typescript
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
  // Pattern 2: Silent overflow detection
  if (contextWindow && message.stopReason === "stop") {
    const inputTokens = message.usage.input + message.usage.cacheRead;
    if (inputTokens > contextWindow) {
      return true;
    }
  }
  return false;
}
```

## Transport Security

### Browser SDK Compatibility

Anthropic SDK configured for browser use with `dangerouslyAllowBrowser: true` (required for browser extensions).

### Proxy Support

HTTP proxy support with configurable agent:
```typescript
if (process.env.HTTP_PROXY || process.env.HTTPS_PROXY) {
  const proxyAgent = await import("proxy-agent");
  const agent = new proxyAgent.ProxyAgent();
  config.requestHandler = new nodeHttpHandler.NodeHttpHandler({
    httpAgent: agent,
    httpsAgent: agent,
  });
}
```

## Abort Signal Handling

All async operations respect `AbortSignal` for cancellation:
```typescript
abort() {
  this.abortController?.abort();
}

if (options.signal?.aborted) {
  throw new Error("Request was aborted");
}
```

## Key Security Files Reference

| Category | File |
|----------|------|
| API Key Resolution | `packages/ai/src/env-api-keys.ts` |
| Credential Storage | `packages/coding-agent/src/core/auth-storage.ts` |
| Input Validation | `packages/ai/src/utils/validation.ts` |
| Unicode Sanitization | `packages/ai/src/utils/sanitize-unicode.ts` |
| JSON Streaming | `packages/ai/src/utils/json-parse.ts` |
| Context Overflow | `packages/ai/src/utils/overflow.ts` |
| Tool Name Normalization | `packages/ai/src/providers/anthropic.ts` |
| Tool Call ID Normalization | `packages/ai/src/providers/anthropic.ts` |

## Security Observations

### Strengths

1. **Layered Security**: Environment variables, file-based credentials with locking, OAuth refresh all work together
2. **Provider Abstraction**: Unified interface with provider-specific security handling
3. **Graceful Degradation**: Partial JSON parsing, silent overflow detection, CSP-aware validation
4. **Secrets Never Logged**: Placeholders used (e.g., `<authenticated>`)

### Areas of Note

1. **File Lock Retry Logic**: Uses exponential backoff with jitter (10 retries, 100ms-10s range)
2. **OAuth Token Storage**: Refresh tokens persisted to `auth.json` - encryption at rest depends on filesystem permissions
3. **Browser Extension Mode**: Validation falls back to pass-through when AJV unavailable
4. **Context Window Management**: Relies on provider error messages; some providers (Ollama) silently truncate
