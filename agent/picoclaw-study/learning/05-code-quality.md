# PicoClaw Code Quality

## Language and Tooling

### Go 1.25+
PicoClaw is written in Go with minimum version 1.25.8 specified in `go.mod`. The project leverages Go's strong standard library and modern concurrency primitives.

### Linting Configuration (`.golangci.yaml`)

PicoClaw uses golangci-lint with an extensive configuration:

**Enabled Linters (default: all)**
- Standard linters including `errcheck`, `govet`, `staticcheck`, `revive`

**Explicitly Disabled**
- `cyclop`, `cyclomatic` complexity checks
- `depguard` dependency checking
- `err113` error checking
- `exhaustive` exhaustiveness checking
- `funlen`, `gocognit`, `gocyclo` complexity metrics
- `gosec` security scanning (identified as failing, needs fixing)
- `nestif` nesting depth
- `testpackage` test organization
- `varnamelen` variable naming
- `wrapcheck` error wrapping
- `wsl` whitespace formatting

**Complexity Thresholds**
```yaml
funlen:
  lines: 120
  statements: 40
gocognit:
  min-complexity: 25
gocyclo:
  min-complexity: 20
```

**Formatters Enabled**
- `gci` - Import organization
- `gofmt` - Code formatting with simplifications
- `gofumpt` - Stricter formatting
- `goimports` - Import handling
- `golines` - Line length (max 120)

### Testing Framework

The project uses **`testify`** (`github.com/stretchr/testify`) as the testing framework, providing:
- `assert` for assertions
- `require` for fatal assertions
- `suite` for test suites

Test files follow Go convention: `*_test.go` suffix, located alongside implementation files.

## Code Quality Practices

### 1. Atomic File Operations

All config and credential writes use atomic operations via `pkg/fileutil.WriteFileAtomic()`:

```go
// WriteFileAtomic guarantees:
// 1. Original file untouched until successful rename
// 2. Temp file cleanup on error
// 3. Data synced to disk before rename
// 4. Directory metadata synced for durability
err := fileutil.WriteFileAtomic(path, data, 0o600)
```

This is critical for:
- Edge device reliability (SD cards, flash storage)
- Preventing corruption on crash during write
- Secure permission handling (0o600 for sensitive files)

### 2. Structured Error Handling

Errors wrap context using `fmt.Errorf("action: %w", err)` pattern:

```go
if err != nil {
    return fmt.Errorf("login failed: %w", err)
}
```

The codebase uses error wrapping consistently for debugging and error tracing.

### 3. Configuration Migration

The `pkg/config` package implements versioned configuration with migration support:

```go
// Config versions track schema evolution
const ConfigVersion = 1

// Migration functions upgrade configs between versions
func migrateV0ToV1(cfg *ConfigV0) *ConfigV1
```

### 4. Module Organization

Clear separation of concerns:
- `cmd/` - CLI entry points and command implementations
- `pkg/` - Shared libraries organized by domain
- Internal packages use `internal/` to prevent external imports

## Testing Coverage

### Test File Distribution

Tests are co-located with implementation files:
```
pkg/auth/store.go         -> pkg/auth/store_test.go
pkg/agent/loop_test.go    -> pkg/agent/loop_mcp_test.go
cmd/picoclaw/internal/cron/... -> cmd/picoclaw/internal/cron/*_test.go
```

### Test Patterns

**Mock Provider Testing** (`pkg/agent/mock_provider_test.go`):
```go
type MockProvider struct{}
func (m *MockProvider) Complete(ctx context.Context, req Request) Response
```

**Table-Driven Tests**:
```go
func TestSomething(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        expected string
    }{
        {"case1", "input1", "expected1"},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := MyFunc(tt.input)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

### Build Tags and Platform Testing

AWS Bedrock support uses build tags:
```go
//go:build bedrock
import _ "github.com/aws/aws-sdk-go-v2/config"
```

## Code Organization Patterns

### 1. Command Pattern (Cobra)

CLI commands use spf13/cobra:
```go
cmd := &cobra.Command{
    Use:   "login",
    Short: "Login via OAuth or paste token",
    Args:  cobra.NoArgs,
    RunE: func(cmd *cobra.Command, _ []string) error {
        return authLoginCmd(provider, useDeviceCode, useOauth)
    },
}
```

### 2. Registry Pattern (Channels, Skills)

Channels use a registry for dynamic discovery:
```go
// pkg/channels/registry.go
type Channel interface {
    Init(cfg *ChannelConfig) error
    Process(msg *Message) (*Response, error)
}
// Global registry for channel plugins
```

### 3. Hook System (Agent)

Extensible hooks for agent behavior modification:
```go
// pkg/agent/hook_process.go
type HookType string
const BeforeModelCall HookType = "before_model_call"

type HookManager struct {
    hooks map[HookType][]Hook
}
```

### 4. Event Bus (Pub/Sub)

Components communicate via event bus:
```go
// pkg/bus/bus.go
type EventBus struct {
    subscribers map[string][]Handler
    mu          sync.RWMutex
}
bus.Subscribe("channel:message", handler)
bus.Publish("channel:message", event)
```

## Known Quality Debt

The golangci-lint configuration explicitly disables several linters that are **known to fail**:

```yaml
# TODO: Disabled, because they are failing at the moment
- contextcheck      # Context passing
- errcheck          # Error checking
- errchkjson        # JSON error checking
- errorlint         # Error style checking
- exhaustive        # Exhaustive match
- forbidigo         # Forbidden identifiers
- gosec             # Security scanning
- staticcheck       # Static analysis
- testifylint       # Testify best practices
- unparam           # Unused parameters
```

**Implication**: The codebase has identified but not yet resolved issues in these areas. Security-related linters (`gosec`) are disabled, indicating security hardening is ongoing.

## Development Commands

```bash
make deps          # Install dependencies
make build         # Build core binary
make build-launcher # Build WebUI launcher
make build-all     # Cross-platform builds
make test          # Run tests (if configured)
```
