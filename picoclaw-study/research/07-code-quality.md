# Picoclaw Code Quality Assessment

## Overview

| Aspect | Assessment |
|--------|------------|
| Go Code Quality | Moderate-High (well-structured, needs stricter linting) |
| Test Coverage | Extensive (100+ test files, good patterns) |
| Type Safety | Strong (struct types, interfaces, type aliases) |
| Error Handling | Moderate (sentinels exist, inconsistent adoption) |
| Logging | Good (structured, zerolog, component-aware) |
| Config Management | Excellent (encryption, migration, multi-key) |
| Frontend Quality | Good (ESLint/Prettier, TypeScript, React 19) |

---

## 1. Linting and Formatting

### Go Linting (.golangci.yaml)

**Status**: Partially enabled with acknowledged debt.

The project has a comprehensive `.golangci.yaml` that acknowledges many linters are disabled with explicit TODOs:

```yaml
# TODO: Disabled, because they are failing at the moment, we should fix them and enable (step by step)
- contextcheck
- errcheck
- errorlint
- exhaustive
- forbidigo
- gosec
- revive
- staticcheck
- testifylint
```

**Observations**:
- 29 linters disabled in `linters.disable` section
- 8+ linters acknowledged as "TODO: should fix and enable step by step"
- Test complexity checks excluded (`gocognit`, `gocyclo`, etc. in `_test\.go$` exclusion)
- Multiple formatters enabled: `gci`, `gofmt`, `gofumpt`, `goimports`, `golines`
- Complexity thresholds: funlen (120 lines, 40 statements), gocyclo (20), gocognit (25)

**Risk**: The number of disabled linters indicates technical debt in error handling, type safety, and code security.

### Go Formatting

Formatters properly configured:
- `gci` for import sorting (standard, default, localmodule sections)
- `gofmt` with rewrite rules (`interface{}` to `any`, slice syntax simplification)
- `golines` max 120 characters

### Frontend Linting

ESLint and Prettier configured with sensible defaults:
- ESLint 9.39.4 with React 19 support
- Prettier 3.8.1 with import sorting and Tailwind class ordering

---

## 2. Test Coverage and Patterns

### Test File Distribution

**100+ test files** across the codebase:

| Package Area | Test Count | Notable Coverage |
|--------------|------------|------------------|
| `pkg/agent/` | ~18 | context_budget, hooks, eventbus, thinking, steering |
| `pkg/auth/` | ~5 | token, oauth, pkce, store |
| `pkg/channels/` | ~20 | telegram, discord, irc, matrix, errors |
| `pkg/providers/` | ~15 | anthropic, azure, bedrock, factory |
| `pkg/config/` | ~8 | migration, model_config, security |
| `cmd/` | ~25 | command tests for all CLI subcommands |

### Test Patterns

**Table-driven tests** used extensively:

```go
func TestIsOverContextBudget(t *testing.T) {
    tests := []struct {
        name          string
        contextWindow int
        messages      []providers.Message
        toolDefs      []providers.ToolDefinition
        maxTokens     int
        want          bool
    }{
        // cases...
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := isOverContextBudget(...)
            if got != tt.want { ... }
        })
    }
}
```

**Helper setup functions** reduce duplication:

```go
func newHookTestLoop(t *testing.T, provider providers.LLMProvider) (*AgentLoop, *AgentInstance, func()) {
    t.Helper()
    tmpDir, err := os.MkdirTemp("", "agent-hooks-*")
    // ... setup ...
    return al, agent, func() {
        al.Close()
        _ = os.RemoveAll(tmpDir)
    }
}
```

**Mock providers** for testing:

```go
type llmHookTestProvider struct {
    mu        sync.Mutex
    lastModel string
}

func (p *llmHookTestProvider) Chat(...) (*providers.LLMResponse, error) {
    // mock implementation
}
```

### Test Quality Assessment

| Criterion | Status | Notes |
|-----------|--------|-------|
| Isolation | Good | Uses `t.TempDir()`, deferred cleanup |
| Mocking | Good | Custom mock types, interfaces for dependencies |
| Edge Cases | Good | Tests for empty inputs, boundary conditions |
| Assertions | Moderate | Uses `testify` but inconsistently (see lint config) |
| Coverage Visibility | Missing | No `go test -cover` in Makefile target |

---

## 3. Type Systems

### Go Struct Types

**Strong typing with JSON tags** throughout:

```go
type Config struct {
    Version   int             `json:"version"`
    Agents    AgentsConfig    `json:"agents"`
    Bindings  []AgentBinding  `json:"bindings,omitempty"`
    Channels  ChannelsConfig  `json:"channels"`
    ModelList []*ModelConfig  `json:"model_list"`
    // ...
}
```

**Type aliases** for protocol types (avoids duplication):

```go
type (
    ToolCall               = protocoltypes.ToolCall
    FunctionCall           = protocoltypes.FunctionCall
    LLMResponse            = protocoltypes.LLMResponse
    Message                = protocoltypes.Message
    ToolDefinition         = protocoltypes.ToolDefinition
)
```

**Custom types for domain concepts**:

```go
type FailoverReason string

const (
    FailoverAuth            FailoverReason = "auth"
    FailoverRateLimit       FailoverReason = "rate_limit"
    FailoverBilling         FailoverReason = "billing"
    FailoverTimeout         FailoverReason = "timeout"
    FailoverFormat          FailoverReason = "format"
    FailoverContextOverflow FailoverReason = "context_overflow"
)
```

### Interfaces

**Provider interface** (core abstraction):

```go
type LLMProvider interface {
    Chat(
        ctx context.Context,
        messages []Message,
        tools []ToolDefinition,
        model string,
        options map[string]any,
    ) (*LLMResponse, error)
    GetDefaultModel() string
}

type StatefulProvider interface {
    LLMProvider
    Close()
}

type StreamingProvider interface {
    ChatStream(...) (*LLMResponse, error)
}

type ThinkingCapable interface {
    SupportsThinking() bool
}

type NativeSearchCapable interface {
    SupportsNativeSearch() bool
}
```

**Hook interfaces** for extensibility:

```go
type LLMObserverHook interface {
    OnEvent(ctx context.Context, evt Event) error
    BeforeLLM(ctx context.Context, req *LLMHookRequest) (*LLMHookRequest, HookDecision, error)
    AfterLLM(ctx context.Context, resp *LLMHookResponse) (*LLMHookResponse, HookDecision, error)
}
```

---

## 4. Error Handling

### Sentinel Errors

**Well-defined error hierarchy** in `pkg/channels/errors.go`:

```go
var (
    ErrNotRunning = errors.New("channel not running")
    ErrRateLimit = errors.New("rate limited")
    ErrTemporary = errors.New("temporary failure")
    ErrSendFailed = errors.New("send failed")
)
```

### Error Classification

```go
func ClassifySendError(statusCode int, rawErr error) error {
    switch {
    case statusCode == http.StatusTooManyRequests:
        return fmt.Errorf("%w: %v", ErrRateLimit, rawErr)
    case statusCode >= 500:
        return fmt.Errorf("%w: %v", ErrTemporary, rawErr)
    case statusCode >= 400:
        return fmt.Errorf("%w: %v", ErrSendFailed, rawErr)
    default:
        return rawErr
    }
}
```

### Failover Error

Rich error metadata for fallback decisions:

```go
type FailoverError struct {
    Reason   FailoverReason
    Provider string
    Model    string
    Status   int
    Wrapped  error
}

func (e *FailoverError) IsRetriable() bool {
    return e.Reason != FailoverFormat && e.Reason != FailoverContextOverflow
}
```

### Error Handling Gaps

Despite good patterns in `channels/errors.go`, the lint config reveals:
- `errcheck` disabled (unchecked error returns)
- `errorlint` disabled (inconsistent error wrapping)
- `revive` disabled (various error issues)
- `testifylint` disabled (assertion style issues)

---

## 5. Logging

### Structured Logging with zerolog

**Centralized logger package** with component support:

```go
package logger

func InfoCF(component string, message string, fields map[string]any) {
    logMessage(INFO, component, message, fields)
}

func ErrorF(message string, fields map[string]any) {
    logMessage(ERROR, "", message, fields)
}
```

**Caller skip logic** to show actual source location:

```go
func getCallerSkip() int {
    for i := 2; i < 15; i++ {
        pc, file, _, ok := runtime.Caller(i)
        if !ok { continue }
        fn := runtime.FuncForPC(pc)
        if strings.HasSuffix(file, "/logger.go") ||
           strings.HasSuffix(file, "/log.go") {
            continue
        }
        // ...
    }
}
```

**Dual output** (console + file):

```go
func logMessage(level LogLevel, component string, message string, fields map[string]any) {
    // Console output
    event.CallerSkipFrame(skip).Msg(message)
    // File output
    if fileLogger.GetLevel() != zerolog.NoLevel {
        fileEvent.CallerSkipFrame(skip).Msg(message)
    }
}
```

### Logging Patterns in Use

```go
// Component-scoped logging
logger.WarnCF("agent", "Failed to parse AGENT.md frontmatter", map[string]any{
    "path":  path,
    "error": err.Error(),
})

// Field-based structured logging
logger.Infof("config migrate success", map[string]any{
    "from": versionInfo.Version,
    "to": CurrentVersion,
})
```

### Configuration

Environment-driven:
```go
func ConfigureFromEnv() {
    if logFile := os.Getenv("PICOCLAW_LOG_FILE"); logFile != "" {
        // enable file logging
    }
}
```

---

## 6. Configuration Management

### Multi-Format Support

TOML primary with YAML fallback:
```go
// Primary: BurntSushi/toml
// Fallback: gopkg.in/yaml.v3
// Environment: caarlos0/env
```

### Security Separation

Secrets stored separately from main config:
- `config.json` - main configuration
- `.security.yml` - API keys, tokens, secrets (gitignored)

### API Key Encryption

```go
func encryptPlaintextAPIKeys(
    models map[string]ModelSecurityEntry,
    passphrase string,
) (map[string]ModelSecurityEntry, error) {
    // AES-GCM encryption
    sealed := make(map[string]ModelSecurityEntry, len(models))
    for k, m := range models {
        for i, key := range m.APIKeys {
            encrypted, err := credential.Encrypt(passphrase, "", key)
            // ...
        }
    }
}
```

### Multi-Key Model Expansion

Single config entry expands to multiple for key-level failover:

```go
// Input: {"model_name": "gpt-4", "api_keys": ["k1", "k2", "k3"]}
// Output: 3 entries with round-robin selection
```

### Flexible Types

```go
type FlexibleStringSlice []string

func (f *FlexibleStringSlice) UnmarshalJSON(data []byte) error {
    // Handles []string, []interface{}, mixed types
}

func (f *FlexibleStringSlice) UnmarshalText(text []byte) error {
    // Handles comma-separated (English and Chinese) from env vars
}
```

### Config Migration

Version-based migration system:
```go
const CurrentVersion = 1

switch versionInfo.Version {
case 0:
    // Migrate from legacy format
    cfg, err = v.Migrate()
case CurrentVersion:
    // Current format
}
```

---

## 7. Patterns and Conventions

### Strengths

1. **Interface-first design**: `LLMProvider`, `StreamingProvider`, `ThinkingCapable` allow pluggable implementations
2. **Hook system**: Extensible before/after hooks for LLM calls, tool execution, approvals
3. **Component-scoped logging**: `InfoCF()`, `ErrorCF()` with component context
4. **Deferred cleanup**: Tests and resources use `defer cleanup()` patterns
5. **Context propagation**: `context.Context` passed through call chains
6. **Graceful degradation**: Fallback providers, multi-key expansion, migration paths

### Concerns

1. **Disabled linters**: 30+ linters disabled suggests technical debt
2. **Inconsistent error handling**: `fmt.Errorf` wrapping varies (some use `%w`, some `%v`)
3. **Magic numbers**: Some hardcoded values (e.g., token budgets, timeouts) not centralized
4. **Large config struct**: `Config` struct is 2000+ lines with deep nesting
5. **Test coverage metrics**: No `go test -cover` in CI/Makefile
6. **No `io.Reader`/`io.Writer` abstractions**: File operations use `os.ReadFile`/`os.WriteFile` directly

---

## 8. Frontend Code Quality

### TypeScript/React Setup

- TypeScript 5.9.3 with strict mode likely (tsconfig patterns)
- React 19.2.0
- Vite 7.3.1 build tool
- TanStack React Router + Query for state management

### Linting/Formatting

ESLint configured:
```javascript
// eslint.config.js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
```

Prettier with Tailwind plugin:
```javascript
// prettier.config.js
import tailwindPlugin from 'prettier-plugin-tailwindcss';
```

### Component Patterns

Uses shadcn/radix primitives:
```json
{
  "dependencies": {
    "@radix-ui/react-dialog": "1.4.3",
    "class-variance-authority": "0.7.1",
    "sonner": "2.0.7"
  }
}
```

---

## Recommendations

### High Priority

1. **Enable `errcheck` and `errorlint`**: Address the TODO comments in `.golangci.yaml`
2. **Add test coverage reporting**: `make test` should output coverage
3. **Enable `revive` and `staticcheck`**: Address recurring code quality issues

### Medium Priority

4. **Centralize magic numbers**: Token budgets, timeouts, thresholds in constants
5. **Reduce `Config` struct size**: Consider splitting into `AgentsConfig`, `ChannelsConfig`, etc. as separate files
6. **Add integration tests**: Current tests are mostly unit tests

### Low Priority

7. **Consider `io.Reader`/`io.Writer` abstractions**: For better testability
8. **Document error handling policy**: When to use sentinel errors vs custom types
9. **Frontend test coverage**: Add Vitest/Cypress for React components

---

## Files Examined

### Go Source
- `.golangci.yaml` - Linting configuration
- `pkg/logger/logger.go` - Logging implementation
- `pkg/channels/errors.go` - Sentinel errors
- `pkg/channels/errutil.go` - Error utilities
- `pkg/config/config.go` - Main configuration (2175 lines)
- `pkg/providers/types.go` - Provider interfaces
- `pkg/agent/definition.go` - Agent definitions
- `pkg/agent/context_budget_test.go` - Test patterns
- `pkg/agent/hooks_test.go` - Hook system tests
- `pkg/auth/store_test.go` - Store tests
- `pkg/commands/executor_test.go` - Executor tests

### Frontend
- `web/frontend/package.json` - Dependencies
- `web/frontend/eslint.config.js` - ESLint config
- `web/frontend/prettier.config.js` - Prettier config

### Build
- `Makefile` - Build targets including `test`, `lint`
