# Security and Performance Patterns Analysis: Vercel AI SDK

## Overview

This document analyzes security and performance patterns in the Vercel AI SDK repository, examining how the SDK handles API keys, input validation, rate limiting, caching, streaming, and telemetry.

---

## 1. API Key and Secret Handling

### Pattern: Environment Variable + Parameter Fallback

**Location:** `packages/provider-utils/src/load-api-key.ts`

```typescript
export function loadApiKey({
  apiKey,
  environmentVariableName,
  apiKeyParameterName = 'apiKey',
  description,
}: {
  apiKey: string | undefined;
  environmentVariableName: string;
  apiKeyParameterName?: string;
  description: string;
}): string {
  // 1. Use provided apiKey if it's a string
  if (typeof apiKey === 'string') {
    return apiKey;
  }

  // 2. Check for invalid non-string values
  if (apiKey != null) {
    throw new LoadAPIKeyError({
      message: `${description} API key must be a string.`,
    });
  }

  // 3. Environment check for edge runtime compatibility
  if (typeof process === 'undefined') {
    throw new LoadAPIKeyError({
      message: `${description} API key is missing. Pass it using the '${apiKeyParameterName}' parameter. Environment variables is not supported in this environment.`,
    });
  }

  // 4. Load from environment variable
  apiKey = process.env[environmentVariableName];

  if (apiKey == null) {
    throw new LoadAPIKeyError({
      message: `${description} API key is missing. Pass it using the '${apiKeyParameterName}' parameter or the ${environmentVariableName} environment variable.`,
    });
  }

  return apiKey;
}
```

**Security Observations:**
- API keys are never logged or included in error messages
- Environment variables checked at runtime, not build time
- Edge runtime compatibility checked explicitly
- Type validation ensures only strings are accepted
- Error messages do not expose sensitive values

### Provider Implementation Pattern

**Location:** `packages/anthropic/src/anthropic-provider.ts`

```typescript
const getHeaders = () => {
  const authHeaders: Record<string, string> = options.authToken
    ? { Authorization: `Bearer ${options.authToken}` }
    : {
        'x-api-key': loadApiKey({
          apiKey: options.apiKey,
          environmentVariableName: 'ANTHROPIC_API_KEY',
          description: 'Anthropic',
        }),
      };

  return withUserAgentSuffix(
    {
      'anthropic-version': '2023-06-01',
      ...authHeaders,
      ...options.headers,
    },
    `ai-sdk/anthropic/${VERSION}`,
  );
};
```

---

## 2. Input Validation

### Pattern: Schema-Based Validation with Error Messages

**Location:** `packages/provider-utils/src/parse-json.ts`

The SDK uses a `FlexibleSchema` approach for response validation:

```typescript
export async function parseJSON<T>({
  text,
  schema,
}: {
  text: string;
  schema?: FlexibleSchema<T>;
}): Promise<T> {
  try {
    const value = secureJsonParse(text);

    if (schema == null) {
      return value;
    }

    return validateTypes<T>({ value, schema });
  } catch (error) {
    if (
      JSONParseError.isInstance(error) ||
      TypeValidationError.isInstance(error)
    ) {
      throw error;
    }

    throw new JSONParseError({ text, cause: error });
  }
}
```

### Pattern: Secure JSON Parsing (Prototype Pollution Prevention)

**Location:** `packages/provider-utils/src/secure-json-parse.ts`

The SDK uses `secure-json-parse` (from fastify) to prevent prototype pollution attacks:

```typescript
const suspectProtoRx =
  /"(?:_|\\u005[Ff])(?:_|\\u005[Ff])(?:p|\\u0070)(?:r|\\u0072)(?:o|\\u006[Ff])(?:t|\\u0074)(?:o|\\u006[Ff])(?:_|\\u005[Ff])(?:_|\\u005[Ff])"\s*:/;
const suspectConstructorRx =
  /"(?:c|\\u0063)(?:o|\\u006[Ff])(?:n|\\u006[Ee])(?:s|\\u0073)(?:t|\\u0074)(?:r|\\u0072)(?:u|\\u0075)(?:c|\\u0063)(?:t|\\u0074)(?:o|\\u006[Ff])(?:r|\\u0072)"\s*:/;
```

The parser:
1. First attempts normal JSON.parse
2. Scans for suspicious `__proto__` and `constructor` patterns in the raw text
3. Recursively filters objects to verify no prototype contamination

### Pattern: Retry Configuration Validation

**Location:** `packages/ai/src/util/prepare-retries.ts`

```typescript
export function prepareRetries({
  maxRetries,
  abortSignal,
}: {
  maxRetries: number | undefined;
  abortSignal: AbortSignal | undefined;
}): {
  maxRetries: number;
  retry: RetryFunction;
} {
  if (maxRetries != null) {
    if (!Number.isInteger(maxRetries)) {
      throw new InvalidArgumentError({
        parameter: 'maxRetries',
        value: maxRetries,
        message: 'maxRetries must be an integer',
      });
    }

    if (maxRetries < 0) {
      throw new InvalidArgumentError({
        parameter: 'maxRetries',
        value: maxRetries,
        message: 'maxRetries must be >= 0',
      });
    }
  }

  const maxRetriesResult = maxRetries ?? 2;
  // ...
}
```

---

## 3. Rate Limiting and Throttling

### Pattern: Exponential Backoff with Header Respect

**Location:** `packages/ai/src/util/retry-with-exponential-backoff.ts`

```typescript
function getRetryDelayInMs({
  error,
  exponentialBackoffDelay,
}: {
  error: APICallError;
  exponentialBackoffDelay: number;
}): number {
  const headers = error.responseHeaders;

  if (!headers) return exponentialBackoffDelay;

  let ms: number | undefined;

  // retry-ms is more precise than retry-after and used by e.g. OpenAI
  const retryAfterMs = headers['retry-after-ms'];
  if (retryAfterMs) {
    const timeoutMs = parseFloat(retryAfterMs);
    if (!Number.isNaN(timeoutMs)) {
      ms = timeoutMs;
    }
  }

  // About the Retry-After header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
  const retryAfter = headers['retry-after'];
  if (retryAfter && ms === undefined) {
    const timeoutSeconds = parseFloat(retryAfter);
    if (!Number.isNaN(timeoutSeconds)) {
      ms = timeoutSeconds * 1000;
    } else {
      ms = Date.parse(retryAfter) - Date.now();
    }
  }

  // check that the delay is reasonable:
  if (
    ms != null &&
    !Number.isNaN(ms) &&
    0 <= ms &&
    (ms < 60 * 1000 || ms < exponentialBackoffDelay)
  ) {
    return ms;
  }

  return exponentialBackoffDelay;
}
```

**Key Features:**
- Respects `retry-after-ms` header (OpenAI pattern)
- Respects `retry-after` header (standard HTTP)
- Validates delay is reasonable (0-60 seconds max)
- Falls back to exponential backoff if headers are unreasonable

### Pattern: Retryable Error Classification

**Location:** `packages/provider/src/errors/api-call-error.ts`

```typescript
readonly isRetryable: boolean;

constructor({
  // ...
  isRetryable = statusCode != null &&
    (statusCode === 408 || // request timeout
      statusCode === 409 || // conflict
      statusCode === 429 || // too many requests
      statusCode >= 500), // server error
  // ...
}) {
  // ...
  this.isRetryable = isRetryable;
}
```

### Pattern: React UI Throttling

**Location:** `packages/react/src/throttle.ts`

```typescript
import throttleFunction from 'throttleit';

export function throttle<T extends (...args: any[]) => any>(
  fn: T,
  waitMs: number | undefined,
): T {
  return waitMs != null ? throttleFunction(fn, waitMs) : fn;
}
```

Used in React hooks (`use-chat.ts`, `use-completion.ts`) to limit update frequency.

---

## 4. Caching Patterns

### Pattern: Telemetry Lazy Evaluation

**Location:** `packages/ai/src/telemetry/select-telemetry-attributes.ts`

```typescript
export async function selectTelemetryAttributes({
  telemetry,
  attributes,
}: {
  telemetry?: TelemetrySettings;
  attributes: {
    [attributeKey: string]:
      | AttributeValue
      | { input: ResolvableAttributeValue }
      | { output: ResolvableAttributeValue }
      | undefined;
  };
}): Promise<Attributes> {
  // when telemetry is disabled, return an empty object to avoid serialization overhead:
  if (telemetry?.isEnabled !== true) {
    return {};
  }

  // ...
  for (const [key, value] of Object.entries(attributes)) {
    if (value == null) {
      continue;
    }

    // input value, check if it should be recorded:
    if (
      typeof value === 'object' &&
      'input' in value &&
      typeof value.input === 'function'
    ) {
      // default to true:
      if (telemetry?.recordInputs === false) {
        continue;
      }

      const result = await value.input();
      // ... only if not disabled
    }
  }
}
```

**Key Features:**
- Telemetry disabled by default (returns empty object immediately)
- `recordInputs` and `recordOutputs` flags for privacy control
- Lazy function evaluation (`{ input: () => ... }`) prevents unnecessary computation

### Pattern: Conditional Response Body Inclusion

**Location:** `packages/ai/src/generate-text/generate-text.ts`

```typescript
const stepRequest: LanguageModelRequestMetadata =
  (include?.requestBody ?? true)
    ? (currentModelResponse.request ?? {})
    : { ...currentModelResponse.request, body: undefined };

const stepResponse = {
  ...currentModelResponse.response,
  // Conditionally include response body:
  body:
    (include?.responseBody ?? true)
      ? currentModelResponse.response?.body
      : undefined,
};
```

Users can opt out of including large payloads (e.g., base64-encoded images) to reduce memory usage.

---

## 5. Streaming Implementation

### Pattern: Transform Stream Architecture

**Location:** `packages/ai/src/generate-text/stream-model-call.ts`

The SDK uses Web Streams API (`TransformStream`) for processing:

```typescript
const standardizedStream = languageModelStream.pipeThrough(
  createLanguageModelStreamPartToModelCallStreamPartTransform({
    tools,
    system: standardizedPrompt.system,
    messages: standardizedPrompt.messages,
    repairToolCall,
  }),
);

return {
  stream: createAsyncIterableStream(standardizedStream),
  response,
  request,
};
```

### Pattern: Smooth Streaming (Performance Optimization)

**Location:** `packages/ai/src/generate-text/smooth-stream.ts`

```typescript
export function smoothStream<TOOLS extends ToolSet>({
  delayInMs = 10,
  chunking = 'word',
}: {
  delayInMs?: number | null;
  chunking?: 'word' | 'line' | RegExp | ChunkDetector | Intl.Segmenter;
} = {}): (options: {
  tools: TOOLS;
}) => TransformStream<TextStreamPart<TOOLS>, TextStreamPart<TOOLS>> {
  return () => {
    let buffer = '';

    return new TransformStream<TextStreamPart<TOOLS>, TextStreamPart<TOOLS>>({
      async transform(chunk, controller) {
        // Handle non-smoothable chunks: flush buffer and pass through
        if (chunk.type !== 'text-delta' && chunk.type !== 'reasoning-delta') {
          flushBuffer(controller);
          controller.enqueue(chunk);
          return;
        }

        // Buffer text and release in chunks
        buffer += chunk.text;
        let match;

        while ((match = detectChunk(buffer)) != null) {
          controller.enqueue({ type, text: match, id });
          buffer = buffer.slice(match.length);
          await delay(delayInMs);
        }
      },
    });
  };
}
```

**Features:**
- Configurable delay between chunks (default 10ms)
- Multiple chunking strategies: word, line, regex, Intl.Segmenter
- Non-text chunks pass through immediately
- Provider metadata preservation

### Pattern: Streaming Timeouts

**Location:** `packages/ai/src/prompt/call-settings.ts`

```typescript
export type TimeoutConfiguration<TOOLS extends ToolSet> =
  | number
  | {
      totalMs?: number;
      stepMs?: number;
      chunkMs?: number;  // Streaming-specific: timeout between chunks
      toolMs?: number;
      tools?: Partial<Record<`${keyof TOOLS & string}Ms`, number>>;
    };
```

---

## 6. Security Middleware and Guards

### Pattern: Error Wrapping

**Location:** `packages/ai/src/prompt/wrap-gateway-error.ts`

```typescript
export function wrapGatewayError(error: unknown) {
  // Wraps errors in APICallError for consistent error handling
  // while preserving original error information
}
```

### Pattern: Abort Signal Integration

Throughout the codebase, `AbortSignal` is properly integrated:

```typescript
const mergedAbortSignal = mergeAbortSignals(
  abortSignal,
  totalTimeoutMs != null ? AbortSignal.timeout(totalTimeoutMs) : undefined,
  stepAbortController?.signal,
);

const { retry } = prepareRetries({
  maxRetries: maxRetriesArg,
  abortSignal: mergedAbortSignal,
});
```

### Pattern: Custom Fetch Middleware

**Location:** `packages/anthropic/src/anthropic-provider.ts`

```typescript
interface AnthropicProviderSettings {
  /**
   * Custom fetch implementation. You can use it as a middleware to intercept requests,
   * or to provide a custom fetch implementation for e.g. testing.
   */
  fetch?: FetchFunction;
}
```

---

## 7. Telemetry and Sensitive Data Handling

### Pattern: OpenTelemetry Integration with Privacy Controls

**Location:** `packages/ai/src/telemetry/open-telemetry-integration.ts`

```typescript
interface TelemetrySettings {
  isEnabled: boolean;
  recordInputs?: boolean;    // Default: true
  recordOutputs?: boolean;    // Default: true
  functionId?: string;
  metadata?: Record<string, unknown>;
}
```

**Sensitive Data Handling:**

1. **Lazy Attribute Evaluation**: Telemetry attributes are wrapped in functions that are only called when telemetry is enabled and recording is allowed

2. **Span Recording Guards**:
```typescript
if (telemetry?.recordInputs === false) {
  continue;  // Skip input recording
}

if (telemetry?.recordOutputs === false) {
  continue;  // Skip output recording
}
```

3. **Error Serialization Safety**:
```typescript
try {
  span.setAttributes(
    selectAttributes(telemetry, {
      'ai.toolCall.result': {
        output: () => JSON.stringify(event.output),
      },
    }),
  );
} catch (_ignored) {
  // JSON.stringify might fail for non-serializable results
}
```

4. **Header Recording**: Response headers are recorded in telemetry but API keys should not be in headers (they use `x-api-key` or `Authorization` which may be filtered by providers)

---

## 8. Error Handling Patterns

### Pattern: Structured Error Types

**Location:** `packages/provider/src/errors/api-call-error.ts`

```typescript
export class APICallError extends AISDKError {
  readonly url: string;
  readonly requestBodyValues: unknown;
  readonly statusCode?: number;
  readonly responseHeaders?: Record<string, string>;
  readonly responseBody?: string;
  readonly isRetryable: boolean;
  readonly data?: unknown;

  static isInstance(error: unknown): error is APICallError {
    return AISDKError.hasMarker(error, marker);
  }
}
```

### Pattern: Error Recovery with Retry

```typescript
async function _retryWithExponentialBackoff<OUTPUT>(
  f: () => PromiseLike<OUTPUT>,
  // ...
  errors: unknown[] = [],
): Promise<OUTPUT> {
  try {
    return await f();
  } catch (error) {
    if (isAbortError(error)) {
      throw error; // don't retry when the request was aborted
    }

    if (maxRetries === 0) {
      throw error; // don't wrap the error when retries are disabled
    }

    if (
      error instanceof Error &&
      APICallError.isInstance(error) &&
      error.isRetryable === true &&
      tryNumber <= maxRetries
    ) {
      await delay(getRetryDelayInMs({...}));
      return _retryWithExponentialBackoff(f, {...});
    }

    throw new RetryError({...});
  }
}
```

---

## Summary of Security Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| Environment variable API key loading | `provider-utils/src/load-api-key.ts` | Secure key retrieval with edge runtime support |
| Secure JSON parsing | `provider-utils/src/secure-json-parse.ts` | Prototype pollution prevention |
| Input validation | Throughout provider-utils | Type-safe API interactions |
| Telemetry privacy controls | `telemetry/select-telemetry-attributes.ts` | User-controlled data recording |
| Retry header respect | `retry-with-exponential-backoff.ts` | RFC-compliant rate limit handling |
| Abort signal integration | Throughout core AI package | Proper cancellation support |
| Error wrapping | `prompt/wrap-gateway-error.ts` | Consistent error handling |

## Summary of Performance Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| Transform streams | `stream-model-call.ts` | Memory-efficient streaming |
| Smooth streaming | `smooth-stream.ts` | Configurable output pacing |
| Lazy telemetry evaluation | `telemetry/select-telemetry-attributes.ts` | Avoid unnecessary computation |
| Conditional body inclusion | `generate-text.ts` | Memory optimization for large payloads |
| Streaming timeouts | `call-settings.ts` | Prevent hung streams |
| Throttling (React) | `react/src/throttle.ts` | UI update rate limiting |
| AsyncIterable streams | `util/async-iterable-stream.ts` | Universal async iteration |
