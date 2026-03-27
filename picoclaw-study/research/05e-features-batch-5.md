# PicoClaw Feature Research: Batch 5

**Features Researched:**
1. Docker Deployment
2. Hardware Device Support (I2C/SPI)
3. Sub-Agent Orchestration

**Research Date:** 2026-03-26
**Source:** `/Users/sheldon/Documents/claw/reference/picoclaw`

---

## Feature 19: Docker Deployment

### Overview

PicoClaw provides containerized deployment via Docker with multiple profiles and variants optimized for different use cases. The Docker infrastructure supports three deployment profiles: `agent` (one-shot queries), `gateway` (long-running bot), and `launcher` (web console + gateway).

### File Locations

| File | Purpose |
|------|---------|
| `docker/Dockerfile` | Minimal runtime image (~40MB) |
| `docker/Dockerfile.full` | Full MCP support with Node.js + uv |
| `docker/Dockerfile.heavy` | Multi-platform builds via goreleaser |
| `docker/Dockerfile.goreleaser` | Launcher variant for goreleaser |
| `docker/docker-compose.yml` | Standard compose with 3 profiles |
| `docker/docker-compose.full.yml` | Full MCP compose with npm cache |
| `docker/entrypoint.sh` | First-run initialization script |

### Architecture

#### Multi-Stage Build Pattern

All Dockerfiles use multi-stage builds to minimize final image size:

```dockerfile
# Stage 1: Build
FROM golang:1.25-alpine AS builder
RUN apk add --no-cache git make
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN make build

# Stage 2: Minimal runtime
FROM alpine:3.23
RUN apk add --no-cache ca-certificates tzdata curl
COPY --from=builder /src/build/picoclaw /usr/local/bin/picoclaw
RUN addgroup -g 1000 picoclaw && adduser -D -u 1000 -G picoclaw picoclaw
USER picoclaw
RUN /usr/local/bin/picoclaw onboard
ENTRYPOINT ["picoclaw"]
CMD ["gateway"]
```

#### Profiles (docker-compose.yml)

```yaml
services:
  picoclaw-agent:
    profiles: [agent]
    # One-shot query: docker compose --profile agent run picoclaw-agent -m "Hello"

  picoclaw-gateway:
    profiles: [gateway]
    restart: on-failure
    # Long-running bot: docker compose --profile gateway up

  picoclaw-launcher:
    profiles: [launcher]
    ports:
      - "127.0.0.1:18800:18800"  # Web console
      - "127.0.0.1:18790:18790"  # Gateway API
```

#### First-Run Initialization (entrypoint.sh)

```sh
#!/bin/sh
set -e

# First-run: neither config nor workspace exists
if [ ! -d "${HOME}/.picoclaw/workspace" ] && [ ! -f "${HOME}/.picoclaw/config.json" ]; then
    picoclaw onboard  # Interactive setup, but non-blocking via TTY detection
    echo "First-run setup complete."
    echo "Edit ${HOME}/.picoclaw/config.json then restart."
    exit 0
fi

exec picoclaw gateway "$@"
```

### Docker Image Variants

| Variant | Base | Size | Use Case |
|---------|------|------|----------|
| `docker.io/sipeed/picoclaw:latest` | alpine:3.23 | ~40MB | Minimal gateway |
| `docker.io/sipeed/picoclaw:launcher` | node:24-alpine | ~200MB+ | Web console + MCP |
| Custom build | golang:1.25-alpine | ~50MB | Development |

### Security Considerations

1. **Non-root user**: Images create and switch to `picoclaw` user (UID 1000)
2. **Health checks**: Alpine-based images include HTTP health checks on port 18790
3. **Volume mounts**: Config is mounted read-only (`config.json:ro`) in production
4. **No sensitive data in image**: API keys stored in mounted volumes, not baked into image

### Technical Observations

1. **Goreleaser integration**: `Dockerfile.heavy` and `Dockerfile.goreleaser` support multi-platform builds via `TARGETPLATFORM` arg, enabling ARM64/AMD64 images from a single build

2. **NPM cache volume**: `docker-compose.full.yml` uses named volume `picoclaw-npm-cache` to speed up repeated MCP server installations

3. **Onboard automation**: The `RUN /usr/local/bin/picoclaw onboard` in Dockerfile creates initial directory structure, but only runs if config doesn't exist (idempotent)

4. **Browser automation support**: `Dockerfile.heavy` includes Chromium + Playwright for `agent-browser` skill support

---

## Feature 13: Hardware Device Support (I2C/SPI)

### Overview

PicoClaw provides hardware peripheral access via I2C and SPI tools, designed for embedded/IoT deployments on Linux. Both tools use direct syscalls (`ioctl`) to communicate with Linux device drivers, bypassing the need for external libraries.

### File Locations

| File | Purpose |
|------|---------|
| `pkg/tools/i2c.go` | I2C tool interface and platform-agnostic logic |
| `pkg/tools/i2c_linux.go` | Linux-specific I2C implementation |
| `pkg/tools/i2c_other.go` | Stub for non-Linux platforms |
| `pkg/tools/spi.go` | SPI tool interface and platform-agnostic logic |
| `pkg/tools/spi_linux.go` | Linux-specific SPI implementation |
| `pkg/tools/spi_other.go` | Stub for non-Linux platforms |

### I2C Tool

#### Interface Definition (`pkg/tools/i2c.go`)

```go
type I2CTool struct{}

func (t *I2CTool) Name() string           { return "i2c" }
func (t *I2CTool) Description() string    { return "Interact with I2C bus devices..." }
func (t *I2CTool) Parameters() map[string]any {
    return map[string]any{
        "properties": map[string]any{
            "action":    map[string]any{"type": "string", "enum": []string{"detect", "scan", "read", "write"}},
            "bus":       map[string]any{"type": "string", "description": "I2C bus number (e.g. \"1\")"},
            "address":   map[string]any{"type": "integer", "description": "7-bit address (0x03-0x77)"},
            "register":  map[string]any{"type": "integer", "description": "Register address"},
            "data":      map[string]any{"type": "array", "items": map[string]any{"type": "integer"}},
            "length":    map[string]any{"type": "integer", "description": "Bytes to read (1-256)"},
            "confirm":   map[string]any{"type": "boolean", "description": "Required for write"},
        },
        "required": []string{"action"},
    }
}
```

#### Actions

1. **detect**: Lists `/dev/i2c-*` devices
2. **scan**: Probes 7-bit addresses 0x08-0x77 for devices
3. **read**: Reads bytes from device at optional register
4. **write**: Writes bytes to device (requires `confirm: true`)

#### Linux Implementation (`pkg/tools/i2c_linux.go`)

Uses raw `syscall.IOCTL` against `/dev/i2c-*` device files:

```go
// Constants from <linux/i2c-dev.h> and <linux/i2c.h>
const (
    i2cSlave       = 0x0703  // Set slave address
    i2cFuncs       = 0x0705  // Query adapter capabilities
    i2cSmbus       = 0x0720  // Perform SMBus transaction
)

// SMBus transaction matching kernel interface
type i2cSmbusData [34]byte
type i2cSmbusArgs struct {
    readWrite uint8
    command   uint8
    size      uint32
    data      *i2cSmbusData
}
```

**Scan Strategy**: Uses `i2cdetect`-compatible hybrid probe:
- SMBus Quick Write for most addresses (safest, no data transferred)
- SMBus Read Byte for EEPROM ranges `0x30-0x37` and `0x50-0x5F` (prevents AT24RF08 corruption)

```go
func smbusProbe(fd int, addr int, hasQuick bool) bool {
    // EEPROM ranges: use read byte to avoid corruption
    useReadByte := (addr >= 0x30 && addr <= 0x37) || (addr >= 0x50 && addr <= 0x5F)

    if !useReadByte && hasQuick {
        // SMBus Quick Write: [START] [ADDR|W] [ACK/NACK] [STOP]
        args := i2cSmbusArgs{readWrite: i2cSmbusWrite, size: i2cSmbusQuick, data: nil}
        _, _, errno := syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), i2cSmbus, ...)
        return errno == 0
    }
    // SMBus Read Byte for EEPROM ranges
    var data i2cSmbusData
    args := i2cSmbusArgs{readWrite: i2cSmbusRead, size: i2cSmbusByte, data: &data}
    _, _, errno := syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), i2cSmbus, ...)
    return errno == 0
}
```

#### Safety Features

1. **Bus ID validation**: Prevents path injection via `isValidBusID()` - only accepts numeric IDs
2. **Address range validation**: 7-bit addresses only (0x03-0x77)
3. **Write confirmation**: Requires explicit `confirm: true` for write operations
4. **EBUSY detection**: Reports when kernel driver owns an address
5. **Length limits**: Max 256 bytes per read

### SPI Tool

#### Interface Definition (`pkg/tools/spi.go`)

```go
type SPITool struct{}

func (t *SPITool) Name() string           { return "spi" }
func (t *SPITool) Description() string    { return "Interact with SPI bus devices..." }
func (t *SPITool) Parameters() map[string]any {
    return map[string]any{
        "properties": map[string]any{
            "action":  map[string]any{"type": "string", "enum": []string{"list", "transfer", "read"}},
            "device":  map[string]any{"type": "string", "description": "SPI device (e.g. \"2.0\")"},
            "speed":   map[string]any{"type": "integer", "description": "Clock speed in Hz (default: 1MHz)"},
            "mode":    map[string]any{"type": "integer", "description": "SPI mode 0-3"},
            "bits":    map[string]any{"type": "integer", "description": "Bits per word (default: 8)"},
            "data":    map[string]any{"type": "array", "description": "Bytes to send"},
            "length":  map[string]any{"type": "integer", "description": "Bytes to read (1-4096)"},
            "confirm": map[string]any{"type": "boolean", "required": true for transfer},
        },
        "required": []string{"action"},
    }
}
```

#### Linux Implementation (`pkg/tools/spi_linux.go`)

Uses `ioctl` calls against `/dev/spidev*` device files:

```go
// SPI ioctl constants (calculated from _IOW macro)
const (
    spiIocWrMode        = 0x40016B01
    spiIocWrBitsPerWord = 0x40016B03
    spiIocWrMaxSpeedHz  = 0x40046B04
    spiIocMessage1      = 0x40206B00  // 32-byte spi_ioc_transfer
)

type spiTransfer struct {
    txBuf       uint64
    rxBuf       uint64
    length      uint32
    speedHz     uint32
    delayUsecs  uint16
    bitsPerWord uint8
    csChange    uint8
    txNbits     uint8
    rxNbits     uint8
    wordDelay   uint8
    pad         uint8
}
```

#### Configuration Sequence

```go
func configureSPI(devPath string, mode uint8, bits uint8, speed uint32) (int, *ToolResult) {
    fd, err := syscall.Open(devPath, syscall.O_RDWR, 0)

    // Set mode, bits per word, max speed via ioctl
    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocWrMode, uintptr(unsafe.Pointer(&mode)))
    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocWrBitsPerWord, uintptr(unsafe.Pointer(&bits)))
    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocWrMaxSpeedHz, uintptr(unsafe.Pointer(&speed)))

    return fd, nil
}
```

#### Transfer Pattern

```go
func (t *SPITool) transfer(args map[string]any) *ToolResult {
    // Build tx/rx buffers
    txBuf := make([]byte, len(dataRaw))
    rxBuf := make([]byte, len(txBuf))

    xfer := spiTransfer{
        txBuf:       uint64(uintptr(unsafe.Pointer(&txBuf[0]))),
        rxBuf:       uint64(uintptr(unsafe.Pointer(&rxBuf[0]))),
        length:      uint32(len(txBuf)),
        speedHz:    speed,
        bitsPerWord: bits,
    }

    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocMessage1, uintptr(unsafe.Pointer(&xfer)))
    runtime.KeepAlive(txBuf)  // Prevent GC from collecting txBuf before ioctl completes
    runtime.KeepAlive(rxBuf)
}
```

**Note**: `runtime.KeepAlive()` is critical here - without it, the GC might collect the buffers before the kernel reads them.

### Platform Stubs

Both tools have `*_other.go` files that return errors on non-Linux platforms:

```go
// pkg/tools/i2c_other.go
func (t *I2CTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    return ErrorResult("I2C is only supported on Linux...")
}
```

### Technical Observations

1. **No external dependencies**: Both tools use only `syscall` package - no CGO, no peripheral libraries
2. **Memory safety**: Uses `runtime.KeepAlive()` to prevent GC before ioctl completes
3. **Device detection**: Lists devices via `filepath.Glob("/dev/i2c-*")` and `filepath.Glob("/dev/spidev*")`
4. **Validation rigor**: Validates bus IDs, addresses, data lengths before any syscall
5. **Error messages**: Provide actionable troubleshooting hints (load module, check device tree, configure pinmux)

---

## Feature 14: Sub-Agent Orchestration

### Overview

PicoClaw implements a sophisticated sub-agent system that allows the main agent to spawn parallel async tasks called "SubTurns." The system supports both synchronous (blocking) and asynchronous (fire-and-forget) execution patterns with lifecycle management, concurrency limits, and token budget tracking.

### File Locations

| File | Purpose |
|------|---------|
| `pkg/tools/subagent.go` | SubagentManager, SubagentTask, SubagentTool, SubTurnSpawner interface |
| `pkg/tools/spawn.go` | SpawnTool for async subagent spawning |
| `pkg/tools/spawn_status.go` | SpawnStatusTool for querying subagent status |
| `pkg/agent/subturn.go` | Core SubTurn execution engine (672 lines) |

### Architecture

#### Core Components

1. **SubagentManager** (`pkg/tools/subagent.go`): Central coordinator for all subagents
2. **SpawnTool**: Async subagent launcher (fire-and-forget)
3. **SubagentTool**: Synchronous subagent executor (blocking)
4. **SpawnStatusTool**: Status query tool for checking subagent state
5. **AgentLoopSpawner**: Bridge between tools and agent package

#### SubagentTask Structure

```go
type SubagentTask struct {
    ID            string    // "subagent-1", "subagent-2", etc.
    Task          string    // Task description
    Label         string    // Optional display label
    AgentID       string    // Target agent ID (for delegation)
    OriginChannel string   // Channel for scoping (multi-user safety)
    OriginChatID  string    // Chat ID for scoping
    Status        string    // "running", "completed", "failed", "canceled"
    Result        string    // Execution result
    Created       int64     // Unix timestamp
}
```

### SubTurnConfig (`pkg/agent/subturn.go`)

The core configuration for spawning sub-turns:

```go
type SubTurnConfig struct {
    Model        string
    Tools        []tools.Tool
    SystemPrompt string
    MaxTokens    int

    // Async controls result delivery:
    //   false = synchronous, result returned directly
    //   true  = asynchronous, result delivered to parent's pendingResults channel
    Async bool

    // Critical: if true, continues after parent turn finishes gracefully
    Critical bool

    // Timeout for this SubTurn (default: 5 minutes)
    Timeout time.Duration

    // MaxContextRunes: context size limit (0=auto, -1=no limit, >0=explicit)
    MaxContextRunes int

    // ActualSystemPrompt: injected as true 'system' message
    ActualSystemPrompt string

    // InitialMessages: preload ephemeral session history
    InitialMessages []providers.Message

    // Shared token budget across team members
    InitialTokenBudget *atomic.Int64
}
```

### SubTurn Execution Flow (`pkg/agent/subturn.go`)

#### Concurrency Control

```go
// Semaphore acquisition with timeout
select {
case parentTS.concurrencySem <- struct{}{}:
    semAcquired = true
    defer func() {
        if semAcquired {
            <-parentTS.concurrencySem  // Release immediately after runTurn
        }
    }()
case <-timeoutCtx.Done():
    return nil, fmt.Errorf("%w: all %d slots occupied for %v",
        ErrConcurrencyTimeout, rtCfg.maxConcurrent, rtCfg.concurrencyTimeout)
}
```

**Critical design**: Semaphore is released **immediately after `runTurn` completes**, not in a defer. This prevents deadlock where sub-turns block on cleanup while parent waits for semaphore slots.

#### Depth Limit

```go
if parentTS.depth >= rtCfg.maxDepth {
    return nil, ErrDepthLimitExceeded
}
```

Default max depth: 3 levels

#### Child Context Isolation

```go
// Create INDEPENDENT child context (not derived from parent ctx)
// This allows child to continue after parent finishes gracefully
childCtx, cancel := context.WithTimeout(context.Background(), timeout)
defer cancel()
```

#### Ephemeral Session Store

SubTurns use in-memory-only sessions that don't persist or pollute parent history:

```go
ephemeralStore := newEphemeralSession(nil)
agent := *baseAgent  // shallow copy
agent.Sessions = ephemeralStore
agent.Tools = baseAgent.Tools.Clone()  // Don't pollute parent's registry
```

### Result Delivery

For async SubTurns, results are delivered via a channel with orphan handling:

```go
func deliverSubTurnResult(al *AgentLoop, parentTS *turnState, childID string, result *tools.ToolResult) {
    parentTS.mu.Lock()
    isFinished := parentTS.isFinished.Load()
    resultChan := parentTS.pendingResults
    parentTS.mu.Unlock()

    if isFinished || resultChan == nil {
        // Parent finished - emit orphan event
        al.emitEvent(EventKindSubTurnOrphan, ...)
        return
    }

    select {
    case resultChan <- result:
        al.emitEvent(EventKindSubTurnResultDelivered, ...)
    case <-parentTS.Finished():
        // Parent finished while waiting to deliver
        al.emitEvent(EventKindSubTurnOrphan, ...)
    }
}
```

### SpawnTool (`pkg/tools/spawn.go`)

Async subagent launcher used by the LLM:

```go
func (t *SpawnTool) execute(ctx context.Context, args map[string]any, cb AsyncCallback) *ToolResult {
    task := args["task"].(string)

    if t.spawner != nil {
        go func() {
            result, err := t.spawner.SpawnSubTurn(ctx, SubTurnConfig{
                Model:        t.defaultModel,
                Tools:        nil,
                SystemPrompt: systemPrompt,
                Async:        true,  // Fire-and-forget
            })
            if cb != nil {
                cb(ctx, result)
            }
        }()
        return AsyncResult(fmt.Sprintf("Spawned subagent for task: %s", task))
    }
    return ErrorResult("Subagent manager not configured")
}
```

### SubagentTool (`pkg/tools/subagent.go`)

Synchronous subagent executor (blocking until complete):

```go
func (t *SubagentTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    if t.spawner != nil {
        result, err := t.spawner.SpawnSubTurn(ctx, SubTurnConfig{
            Model:        t.defaultModel,
            SystemPrompt: systemPrompt,
            Async:        false,  // Synchronous - blocks until complete
        })
        // Format and return result
    }
    return ErrorResult("Subagent manager not configured")
}
```

### SpawnStatusTool (`pkg/tools/spawn_status.go`)

Status query tool with conversation scoping for multi-user safety:

```go
func (t *SpawnStatusTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    callerChannel := ToolChannel(ctx)
    callerChatID := ToolChatID(ctx)

    // Filter tasks to current conversation only
    for _, task := range tasks {
        if cpy.OriginChannel != "" && cpy.OriginChannel != callerChannel {
            continue
        }
        if cpy.OriginChatID != "" && cpy.OriginChatID != callerChatID {
            continue
        }
        tasks = append(tasks, cpy)
    }
    // Return status report with counts by status
}
```

### SubTurn Lifecycle Events

The system emits events for observability:

| Event | Payload |
|-------|---------|
| `SubTurnSpawn` | AgentID, Label, ParentTurnID |
| `SubTurnEnd` | AgentID, Status |
| `SubTurnResultDelivered` | ContentLen |
| `SubTurnOrphan` | ParentTurnID, ChildTurnID, Reason |

### Default Configuration

```go
const (
    defaultMaxSubTurnDepth        = 3
    defaultMaxConcurrentSubTurns = 5
    defaultConcurrencyTimeout     = 30 * time.Second
    defaultSubTurnTimeout         = 5 * time.Minute
    maxEphemeralHistorySize       = 50  // Prevents memory accumulation
)
```

### Technical Observations

1. **No circular dependency**: `SubTurnSpawner` interface in `pkg/tools` allows tools to spawn turns without importing `pkg/agent`

2. **Thread-safe snapshots**: `GetTaskCopy()` and `ListTaskCopies()` return consistent data under read lock

3. **Conversation scoping**: Prevents cross-conversation task leakage in multi-user deployments

4. **Token budget inheritance**: SubTurns can share a budget via `InitialTokenBudget *atomic.Int64`

5. **Ephemeral history limit**: `maxEphemeralHistorySize = 50` prevents memory accumulation in long-running sub-turns

6. **Panic recovery**: `spawnSubTurn` has defer with panic recovery and emits error events

7. **Parent-child relationship**: Established thread-safely under mutex before goroutine starts

8. **Active turn tracking**: Uses `sync.Map` (`activeTurnStates`) to track all active SubTurns for debugging/inspection

---

## Cross-Feature Observations

### 1. Modular Design Pattern

All three features follow a consistent pattern:
- **Platform-specific implementations**: I2C/SPI have `*_linux.go` and `*_other.go` files
- **Docker variants**: Multiple Dockerfiles for different capabilities (minimal, full MCP, heavy)
- **Tool abstraction**: Sub-agent system uses interface (`SubTurnSpawner`) to decouple from agent core

### 2. Error Handling Rigor

| Feature | Error Cases Handled |
|---------|---------------------|
| I2C | Non-Linux OS, invalid bus ID, address out of range, EBUSY (driver ownership), permissions |
| SPI | Non-Linux OS, invalid device format, speed out of range, transfer failures |
| SubAgent | Depth limit exceeded, concurrency timeout, context cancellation, orphan results |

### 3. Safety Mechanisms

- **I2C write**: Requires explicit `confirm: true`
- **SPI transfer**: Requires explicit `confirm: true`
- **SubAgent allowlist**: Optional `allowlistCheck` function for multi-tenant deployments
- **Non-root Docker**: Images run as `picoclaw` user, not root

### 4. Observability

- **Docker**: HTTP health checks on `:18790/health`
- **SubAgent**: Event emission for spawn, end, result delivery, orphan detection
- **I2C/SPI**: Detailed error messages with troubleshooting hints

### 5. Performance Considerations

- **Docker**: Multi-stage builds minimize image size (~40MB base)
- **I2C/SPI**: Direct syscall, no marshalling overhead
- **SubAgent**: Ephemeral sessions with 50-message limit prevent memory bloat
- **SubAgent**: Semaphore released immediately after turn completes (not deferred)

---

## Potential Technical Debt

1. **I2C/SPI**: No test coverage visible - hard to verify correctness without hardware
2. **SubTurn orphan handling**: Late results are logged but not stored/retrievable
3. **Docker entrypoint**: Interactive `onboard` in Dockerfile could fail in automated CI/CD
4. **SubagentManager**: TODO comment indicates SubTurns will eventually be modeled as "child turns inside pkg/agent" - current implementation is legacy
