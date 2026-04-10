# Vercel AI SDK - Security Patterns

## API Key and Secret Handling

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

### Security Observations

- API keys are never logged or included in error messages
- Environment variables checked at runtime, not build time
- Edge runtime compatibility checked explicitly
- Type validation ensures only strings are accepted
- Error messages do not expose sensitive values

### Provider Implementation Pattern

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

## Input Validation

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
  // ...
}
```

## Rate Limiting and Throttling

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

  // retry-after header (standard HTTP)
  const retryAfter = headers['retry-after'];
  if (retryAfter && ms === undefined) {
    const timeoutSeconds = parseFloat(retryAfter);
    if (!Number.isNaN(timeoutSeconds)) {
      ms = timeoutSeconds * 1000;
    } else {
      ms = Date.parse(retryAfter) - Date.now();
    }
  }

  // Validate delay is reasonable (0-60 seconds max)
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
  this.isRetryable = isRetryable;
}
```

## Telemetry and Sensitive Data Handling

### Pattern: OpenTelemetry Integration with Privacy Controls

**Location:** `packages/ai/src/telemetry/select-telemetry-attributes.ts`

```typescript
interface TelemetrySettings {
  isEnabled: boolean;
  recordInputs?: boolean;    // Default: true
  recordOutputs?: boolean;   // Default: true
  functionId?: string;
  metadata?: Record<string, unknown>;
}
```

### Sensitive Data Handling

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

4. **Telemetry Disabled by Default**: Returns empty object immediately when disabled to avoid serialization overhead

## Error Handling Patterns

### Pattern: Structured Error Types

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

## Security Middleware and Guards

### Abort Signal Integration

Proper `AbortSignal` integration throughout the codebase ensures requests can be properly cancelled:

```typescript
const mergedAbortSignal = mergeAbortSignals(
  abortSignal,
  totalTimeoutMs != null ? AbortSignal.timeout(totalTimeoutMs) : undefined,
  stepAbortController?.signal,
);
```

### Custom Fetch Middleware

Providers support custom fetch implementations for interception or testing:

```typescript
interface AnthropicProviderSettings {
  /**
   * Custom fetch implementation. You can use it as a middleware to intercept requests,
   * or to provide a custom fetch implementation for e.g. testing.
   */
  fetch?: FetchFunction;
}
```

## Streaming Implementation

### Transform Stream Architecture

The SDK uses Web Streams API (`TransformStream`) for memory-efficient processing:

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

### Conditional Response Body Inclusion

Users can opt out of including large payloads to reduce memory usage:

```typescript
const stepResponse = {
  ...currentModelResponse.response,
  body:
    (include?.responseBody ?? true)
      ? currentModelResponse.response?.body
      : undefined,
};
```

## Security Patterns Summary

| Pattern | Location | Purpose |
|---------|----------|---------|
| Environment variable API key loading | `provider-utils/src/load-api-key.ts` | Secure key retrieval with edge runtime support |
| Secure JSON parsing | `provider-utils/src/secure-json-parse.ts` | Prototype pollution prevention |
| Input validation | Throughout provider-utils | Type-safe API interactions |
| Telemetry privacy controls | `telemetry/select-telemetry-attributes.ts` | User-controlled data recording |
| Retry header respect | `retry-with-exponential-backoff.ts` | RFC-compliant rate limit handling |
| Abort signal integration | Throughout core AI package | Proper cancellation support |
| Error wrapping | `prompt/wrap-gateway-error.ts` | Consistent error handling |

## Key Security Principles Observed

1. **Never expose secrets in errors**: API keys never logged or included in messages
2. **Validate early, fail fast**: Input validation at entry points with descriptive errors
3. **Prototype pollution prevention**: Secure JSON parsing with regex scanning
4. **Privacy by default**: Telemetry disabled unless explicitly enabled
5. **Type safety**: Strict TypeScript preventing accidental data leakage
6. **Proper cancellation**: AbortSignal support for request termination
7. **Rate limit respect**: Exponential backoff with RFC-compliant header parsing
