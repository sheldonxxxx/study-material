# Security: oh-my-openagent

## Secrets Management

### Environment Variables

**OPENCODE_SERVER_USERNAME / OPENCODE_SERVER_PASSWORD**
- HTTP Basic Auth for OpenCode SDK client
- Password optional; defaults to username "opencode" if unset
- Multiple injection strategies for SDK compatibility:
  1. `setConfig()` headers (primary)
  2. Request interceptors
  3. Fetch wrapper
  4. Mutable internal config

```typescript
export function getServerBasicAuthHeader(): string | undefined {
  const password = process.env.OPENCODE_SERVER_PASSWORD
  if (!password) return undefined
  const username = process.env.OPENCODE_SERVER_USERNAME ?? "opencode"
  return `Basic ${Buffer.from(`${username}:${password}`, "utf8").toString("base64")}`
}
```

**ANTHROPIC_1M_CONTEXT / VERTEX_ANTHROPIC_1M_CONTEXT**
- Enable 1M token context window for specific providers
- Feature flags for controlling expensive operations

**XDG_DATA_HOME / XDG_CACHE_HOME**
- Standard XDG base directory specification
- Falls back to `~/.local/share` and `~/.cache`
- Handles non-writable directories with `os.tmpdir()` fallback

### Caching Security

Cache files stored in user-level directories:
- **Cache**: `~/.cache/oh-my-opencode/`
- **Data**: `~/.local/share/opencode-data/`

Cache files include timestamps for TTL validation:
- `model-capabilities.json`
- `connected-providers.json`
- `provider-models.json`

## Authentication & Authorization

### HTTP Basic Auth Injection

OpenCode SDK client auth injected via multiple strategies (fallback chain):

1. **Headers injection** via `setConfig()` - merges headers, doesn't replace
2. **Request interceptors** - checks for existing Authorization before setting
3. **Fetch wrapper** - wraps existing fetch with auth header injection
4. **Direct fetch replacement** - last resort for incompatible clients

Auth is injected before requests reach the network.

### Blocked Operations

Interactive bash tool blocks dangerous tmux subcommands:
- `kill`, `detach` - session manipulation
- `send-keys` - command injection

Protected via `BLOCKED_TMUX_SUBCOMMANDS` constant.

## Input Validation & Sanitization

### Zod Schema Validation

Comprehensive configuration validation using Zod v4:
- 25+ schema modules in `src/config/schema/`
- Main schema: `OhMyOpenCodeConfigSchema` with 25+ fields
- Sub-schemas independently validated: `agent-overrides.ts`, `categories.ts`, `experimental.ts`, etc.

### Hashline Edit Validation

The hashline-edit tool implements robust line reference validation:
```typescript
export function validateLineRefs(lines: string[], refs: string[]): void {
  // Validates each ref exists and hash matches
  // Throws HashlineMismatchError with remapping suggestions
}
```

### Command Tokenization

Safe command parsing prevents injection:
```typescript
export function tokenizeCommand(cmd: string): string[] {
  // Quote-aware tokenizer with escape handling
  // Prevents command injection through proper splitting
}
```

### Safe Path Handling

Working directory validation:
```typescript
export function validateCwd(cwd: string): { valid: boolean; error?: string } {
  try {
    if (!existsSync(cwd)) {
      return { valid: false, error: `Working directory does not exist: ${cwd}` }
    }
    if (!stats.isDirectory()) {
      return { valid: false, error: `Path is not a directory: ${cwd}` }
    }
    return { valid: true }
  } catch {
    return { valid: false, error: `Cannot access working directory` }
  }
}
```

## Performance Patterns

### Caching Architecture

**Multi-level cache system:**
1. **In-memory cache** - `memSnapshot` in model-capabilities-cache.ts
2. **Disk cache** - JSON files in XDG cache dir (`~/.cache/oh-my-opencode/`)

**Cache read pattern:**
```typescript
function readModelCapabilitiesCache(): ModelCapabilitiesSnapshot | null {
  if (memSnapshot !== undefined) return memSnapshot  // In-memory hit
  // ... disk read
  memSnapshot = snapshot  // Populate memory
  return snapshot
}
```

### Token & Context Management

Token estimation and dynamic truncation:
```typescript
const CHARACTERS_PER_TOKEN = 4

export function estimateTokenCount(text: string): number {
  return Math.ceil(text.length / CHARACTERS_PER_TOKEN)
}

export function truncateToTokenBudget(content: string, maxTokens: number): string {
  const maxCharacters = maxTokens * CHARACTERS_PER_TOKEN
  if (content.length <= maxCharacters) return content
  // Truncate at word boundary with [TRUNCATED] marker
}
```

### Concurrency Control

Semaphore limits concurrent ripgrep operations:
```typescript
export const rgSemaphore = new Semaphore(2)  // Limit concurrent ripgrep
```

### Error Recovery & Retry

Model error classification:
```typescript
const RETRYABLE_ERROR_NAMES = new Set([
  "providermodelnotfounderror", "ratelimiterror", "quotaexceedederror",
  "insufficientcreditserror", "modelunavailableerror",
  "providerconnectionerror", "authenticationerror"
])

const NON_RETRYABLE_ERROR_NAMES = new Set([
  "messageabortederror", "permissiondeniederror", "contextlengtherror",
  "timeouterror", "validationerror"
])
```

## Cost Optimization

- **Conservative output limiting** - Uses 50% of remaining context for output
- **Skill content truncation** - Priority-based reduction of skill prompts
- **Model fallback chain** - Automatic model switching on errors
- **Provider selection** - Connected provider caching avoids unnecessary API calls
- **1M context support** - Environment-gated expensive feature

## Observability

### Structured Logging

```typescript
log("[model-capabilities-cache] Read cache", {
  modelCount: Object.keys(snapshot.models).length,
  generatedAt: snapshot.generatedAt,
})
```

### Error Context

Rich error information preserved through retry chains:
```typescript
log("[model-suggestion-retry] Model not found, retrying with suggestion", {
  original: `${suggestion.providerID}/${suggestion.modelID}`,
  suggested: suggestion.suggestion,
})
```

## Security Considerations

### Current Mitigations

| Pattern | Implementation |
|---------|----------------|
| Basic Auth injection | `opencode-server-auth.ts` |
| Env var handling | `data-path.ts`, `context-limit-resolver.ts` |
| Zod validation | `config/schema/*.ts` |
| Hashline validation | `hashline-edit/validation.ts` |
| Command tokenization | `interactive-bash/tools.ts` |
| Path validation | `lsp-process.ts`, `data-path.ts` |

### Potential Improvements

1. **No secret rotation mechanism** - `OPENCODE_SERVER_PASSWORD` is static
2. **Cache files not encrypted** - Sensitive data in provider responses stored in plaintext
3. **No request signing** - Relies on HTTP Basic Auth only
4. **Shell injection surface** - Despite tokenization, complex commands may have edge cases
