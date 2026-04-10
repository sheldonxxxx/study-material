# Security and Performance Patterns: oh-my-openagent

## Overview

| Property | Value |
|----------|-------|
| **Project** | oh-my-openagent (v3.11.0) |
| **Analysis Focus** | Security and Performance Patterns |
| **Runtime** | Bun 1.x |
| **Language** | TypeScript (strict mode) |

---

## 1. Secrets Management

### Environment Variables

The project uses environment variables for sensitive configuration with careful handling:

**`OPENCODE_SERVER_USERNAME` / `OPENCODE_SERVER_PASSWORD`** (src/shared/opencode-server-auth.ts)
- HTTP Basic Auth for OpenCode SDK client
- Password is optional; defaults to username "opencode" if not set
- Token generated via `Buffer.from()` encoding
- Multiple injection strategies for SDK client compatibility:
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

**`ANTHROPIC_1M_CONTEXT` / `VERTEX_ANTHROPIC_1M_CONTEXT`** (src/shared/context-limit-resolver.ts)
- Enable 1M token context window for specific providers
- Feature flags for controlling expensive operations

**`XDG_DATA_HOME` / `XDG_CACHE_HOME`** (src/shared/data-path.ts)
- Standard XDG base directory specification
- Falls back to `~/.local/share` and `~/.cache` respectively
- Handles non-writable directories with `os.tmpdir()` fallback

### Caching Security

Cache files stored in user-level directories:
- **Cache**: `~/.cache/oh-my-opencode/`
- **Data**: `~/.local/share/opencode-data/`

Cache files include timestamps for TTL validation:
- `model-capabilities.json`
- `connected-providers.json`
- `provider-models.json`

---

## 2. Authentication & Authorization

### HTTP Basic Auth Injection

The OpenCode SDK client has auth injected via multiple strategies (src/shared/opencode-server-auth.ts):

1. **Headers injection** via `setConfig()` - MERGES headers, doesn't replace
2. **Request interceptors** - checks for existing Authorization before setting
3. **Fetch wrapper** - wraps existing fetch with auth header injection
4. **Direct fetch replacement** - last resort for incompatible clients

The auth is injected BEFORE the request reaches the network, ensuring all SDK calls are authenticated.

### Blocked Operations

Interactive bash tool (src/tools/interactive-bash/tools.ts) blocks dangerous tmux subcommands:
- `kill`, `detach` - session manipulation
- `send-keys` - command injection
- Protected via BLOCKED_TMUX_SUBCOMMANDS constant

---

## 3. Input Validation & Sanitization

### Zod Schema Validation

Comprehensive configuration validation using Zod v4 (src/config/schema.ts):

```typescript
// Main config schema (src/config/schema/oh-my-opencode-config.ts)
export const OhMyOpenCodeConfigSchema = z.object({
  $schema: z.string().optional(),
  new_task_system_enabled: z.boolean().optional(),
  default_run_agent: z.string().optional(),
  disabled_mcps: z.array(AnyMcpNameSchema).optional(),
  disabled_agents: z.array(z.string()).optional(),
  disabled_skills: z.array(BuiltinSkillNameSchema).optional(),
  // ... 25+ more fields with type safety
})
```

Each sub-schema is independently validated:
- `agent-overrides.ts`, `categories.ts`, `claude-code.ts`
- `comment-checker.ts`, `experimental.ts`, `fallback-models.ts`
- `model-capabilities.ts`, `skills.ts`, `tmux.ts`, etc.

### Line Reference Validation (Hashline Edit)

The hashline-edit tool (src/tools/hashline-edit/validation.ts) implements robust input validation:

```typescript
export function validateLineRefs(lines: string[], refs: string[]): void {
  // Validates each ref exists and hash matches
  // Throws HashlineMismatchError with remapping suggestions
}

export class HashlineMismatchError extends Error {
  readonly remaps: ReadonlyMap<string, string>
  // Provides correct line references when file changes
}
```

Pattern validation: `/([0-9]+#[ZPMQVRWSNKTXJBYH]{2})/`

### Command Tokenization

Interactive bash implements safe command parsing (src/tools/interactive-bash/tools.ts):

```typescript
export function tokenizeCommand(cmd: string): string[] {
  // Quote-aware tokenizer with escape handling
  // Prevents command injection through proper splitting
}
```

### Safe Path Handling

Working directory validation (src/tools/lsp/lsp-process.ts):

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

---

## 4. Performance Patterns

### Caching Architecture

**Multi-level cache system:**

1. **In-memory cache** (membrane-cached snapshots)
   - `memSnapshot` in model-capabilities-cache.ts
   - `memConnected` in connected-providers-cache.ts

2. **Disk cache** (JSON files in XDG cache dir)
   - `~/.cache/oh-my-opencode/model-capabilities.json`
   - `~/.cache/oh-my-opencode/connected-providers.json`
   - `~/.cache/oh-my-opencode/provider-models.json`

**Cache read pattern:**

```typescript
function readModelCapabilitiesCache(): ModelCapabilitiesSnapshot | null {
  if (memSnapshot !== undefined) {
    return memSnapshot  // In-memory hit
  }
  // ... disk read
  memSnapshot = snapshot  // Populate memory
  return snapshot
}
```

### Token & Context Management

**Token estimation** (src/tools/delegate-task/token-limiter.ts):

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

**Dynamic truncation** (src/shared/dynamic-truncator.ts):

```typescript
export async function dynamicTruncate(
  ctx: PluginInput,
  sessionID: string,
  output: string,
  options: TruncationOptions = {}
): Promise<TruncationResult> {
  const usage = await getContextWindowUsage(ctx, sessionID)
  const maxOutputTokens = Math.min(
    usage.remainingTokens * 0.5,  // Reserve 50% for response
    targetMaxTokens
  )
  return truncateToTokenLimit(output, maxOutputTokens)
}
```

### Tool Output Truncation

Hook-based truncation for specific tools (src/hooks/tool-output-truncator.ts):

```typescript
const TRUNCATABLE_TOOLS = [
  "grep", "Grep", "safe_grep",
  "glob", "Glob", "safe_glob",
  "lsp_diagnostics", "ast_grep_search",
  "interactive_bash", "webfetch"
]

const TOOL_SPECIFIC_MAX_TOKENS = {
  webfetch: 10_000,  // Aggressive for web content
  // Default: 50_000
}
```

### Concurrency Control

**Semaphore for ripgrep** (src/tools/shared/semaphore.ts):

```typescript
export class Semaphore {
  private queue: (() => void)[] = []
  private running = 0

  constructor(private readonly max: number) {}

  async acquire(): Promise<void> {
    if (this.running < this.max) {
      this.running++
      return
    }
    return new Promise<void>((resolve) => {
      this.queue.push(() => { this.running++; resolve() })
    })
  }

  release(): void {
    this.running--
    const next = this.queue.shift()
    if (next) next()
  }
}

export const rgSemaphore = new Semaphore(2)  // Limit concurrent ripgrep
```

### Process Management

**Lazy LSP spawning** (src/tools/lsp/lsp-process.ts):

```typescript
function shouldUseNodeSpawn(): boolean {
  return process.platform === "win32"  // Bun spawn segfault workaround
}

export function spawnProcess(
  command: string[],
  options: { cwd: string; env: Record<string, string | undefined> }
): UnifiedProcess {
  const cwdValidation = validateCwd(options.cwd)
  if (!cwdValidation.valid) {
    throw new Error(`[LSP] ${cwdValidation.error}`)
  }
  // Platform-appropriate spawning
}
```

### Error Recovery & Retry

**Model error classification** (src/shared/model-error-classifier.ts):

```typescript
const RETRYABLE_ERROR_NAMES = new Set([
  "providermodelnotfounderror",
  "ratelimiterror",
  "quotaexceedederror",
  "insufficientcreditserror",
  "modelunavailableerror",
  "providerconnectionerror",
  "authenticationerror"
])

const NON_RETRYABLE_ERROR_NAMES = new Set([
  "messageabortederror",
  "permissiondeniederror",
  "contextlengtherror",
  "timeouterror",
  "validationerror"
])
```

**Model suggestion retry** (src/shared/model-suggestion-retry.ts):

```typescript
// Parses "did you mean: model/provider" from error messages
export function parseModelSuggestion(error: unknown): ModelSuggestionInfo | null {
  // Regex: /model not found:\s*([^\/\s]+)\s*\/\s*([^\.\s]+)/i
  // Extracts providerID, modelID, and suggestion
}
```

### Context Window Management

**1M context support** (src/shared/context-limit-resolver.ts):

```typescript
const DEFAULT_ANTHROPIC_ACTUAL_LIMIT = 200_000

function getAnthropicActualLimit(modelCacheState?: ContextLimitModelCacheState): number {
  return (modelCacheState?.anthropicContext1MEnabled ?? false) ||
    process.env.ANTHROPIC_1M_CONTEXT === "true" ||
    process.env.VERTEX_ANTHROPIC_1M_CONTEXT === "true"
    ? 1_000_000
    : DEFAULT_ANTHROPIC_ACTUAL_LIMIT
}
```

---

## 5. Cost Optimization Strategies

### Token Budget Management

1. **Conservative output limiting** - Uses 50% of remaining context for output
2. **Skill content truncation** - Priority-based reduction of skill prompts
3. **Header preservation** - First 3 lines kept for grep/diagnostic context

### Model Fallback Chain

The `model_fallback` config enables automatic model switching:

```typescript
// src/config/schema/oh-my-opencode-config.ts
model_fallback: z.boolean().optional(),

// src/shared/model-error-classifier.ts
export function getNextFallback(
  fallbackChain: FallbackEntry[],
  attemptCount: number
): FallbackEntry | undefined
```

### Provider Selection

Connected provider caching avoids unnecessary API calls:

```typescript
export function selectFallbackProvider(
  providers: string[],
  preferredProviderID?: string
): string {
  const connectedProviders = readConnectedProvidersCache()
  // Priority: connected provider > preferred > first available
}
```

---

## 6. Observability

### Structured Logging

All caches and critical paths log activity:

```typescript
log("[model-capabilities-cache] Read cache", {
  modelCount: Object.keys(snapshot.models).length,
  generatedAt: snapshot.generatedAt,
})

log("[connected-providers-cache] Error updating cache", {
  error: String(err)
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

---

## 7. Security Considerations

### Potential Improvements

1. **No secret rotation mechanism** - `OPENCODE_SERVER_PASSWORD` is static
2. **Cache files are not encrypted** - Sensitive data in provider responses stored in plaintext
3. **No request signing** - Relies on HTTP Basic Auth only
4. **Shell injection surface** - Despite tokenization, complex commands may have edge cases

### Current Mitigations

1. **Zod validation** on all config inputs
2. **Blocked tmux subcommands** prevent dangerous operations
3. **Working directory validation** prevents path traversal
4. **Error handling** with graceful degradation (tool-output-truncator)

---

## 8. Summary Table

| Pattern | Location | Type |
|---------|----------|------|
| Basic Auth injection | `opencode-server-auth.ts` | Security |
| Env var handling | `data-path.ts`, `context-limit-resolver.ts` | Security |
| Zod validation | `config/schema/*.ts` | Security |
| Hashline validation | `hashline-edit/validation.ts` | Security |
| Command tokenization | `interactive-bash/tools.ts` | Security |
| Path validation | `lsp-process.ts`, `data-path.ts` | Security |
| In-memory cache | `model-capabilities-cache.ts` | Performance |
| Disk cache | `connected-providers-cache.ts` | Performance |
| Token estimation | `token-limiter.ts` | Performance |
| Dynamic truncation | `dynamic-truncator.ts` | Performance |
| Semaphore | `semaphore.ts` | Performance |
| Process spawning | `lsp-process.ts` | Performance |
| Error retry | `model-error-classifier.ts` | Resilience |
| Model fallback | `model-suggestion-retry.ts` | Resilience |
| 1M context | `context-limit-resolver.ts` | Cost |
