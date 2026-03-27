# Lessons Learned: Building PicoClaw-Style AI Agent Projects

**Project Analyzed:** github.com/sipeed/picoclaw
**Analysis Date:** 2026-03-26
**Context:** Study of a production Go-based AI agent with multi-channel, multi-provider support

---

## Executive Summary

PicoClaw is a 26K-star GitHub project (as of March 2026) that demonstrates how to build a production-grade AI agent framework in Go. It connects to 17+ messaging platforms, supports 30+ LLM providers, and runs on hardware as cheap as $10. The project is in active development (v0.2.x), with 1500+ commits since January 2025 and a 6-8 day release cadence.

This document captures what to emulate, what to avoid, and surprises discovered during deep research.

---

## Part 1: What to Emulate

### 1. Interface-First Design with Factory Registry Pattern

**Pattern:** Define interfaces in `pkg/providers/interfaces.go`, `pkg/channels/interfaces.go`, then implement concrete providers/channels that self-register via `init()` functions.

**Example from code:**
```go
// pkg/channels/registry.go
var factories = map[string]ChannelFactory{}

func RegisterFactory(name string, f ChannelFactory) {
    factories[name] = f
}

// pkg/channels/telegram/telegram.go
func init() {
    channels.RegisterFactory("telegram", NewTelegramChannel)
}
```

**Why it works:** Adding a new channel or provider only requires creating the implementation and an `init()` call. No central registration or switch statement needed.

**Emulate this for:** New tool integrations, new platform adapters, new LLM providers.

---

### 2. Event-Driven Architecture with Central MessageBus

**Pattern:** All components communicate via typed channels (inbound, outbound text, outbound media) rather than direct calls.

**Example from code:**
```go
type MessageBus struct {
    inbound       chan InboundMessage
    outbound      chan OutboundMessage
    outboundMedia chan OutboundMediaMessage
    streamDelegate atomic.Value
}
```

**Why it works:** Decouples producers from consumers. Channels, agent loop, and gateway all communicate through the bus without knowing about each other. Head-of-line blocking is prevented by using separate channels for different message types.

**Emulate this for:** Any system with multiple input sources and processing pipelines.

---

### 3. Comprehensive Error Classification with Fallback Chains

**Pattern:** Classify every error into retriable vs. non-retriable categories, then implement provider fallback with cooldown tracking.

**Example from code:**
```go
const (
    FailoverRateLimit        FailoverReason = "rate_limit"
    FailoverOverloaded       FailoverReason = "overloaded_error"
    FailoverTimeout          FailoverReason = "timeout"
    FailoverBilling          FailoverReason = "billing"
    FailoverAuth             FailoverReason = "auth"
    FailoverFormat           FailoverReason = "format"
    FailoverContextOverflow  FailoverReason = "context_overflow"
)

func ClassifyError(err error, provider, model string) *FailoverError {
    // Context cancellation -> nil (no failover)
    // HTTP status code -> reason mapping
    // Pattern matching -> reason mapping
}
```

**Why it works:** A single API call failure doesn't mean the entire agent fails. The system automatically tries the next provider or model with appropriate cooldown periods.

**Emulate this for:** Any system depending on external APIs.

---

### 4. Security-First Design

**Multiple layers of defense:**

1. **Separate security config** (`config.json` vs `.security.yml`)
2. **AES-256-GCM encryption** for credentials at rest
3. **HKDF-SHA256 key derivation** using SSH keys
4. **Path containment validation** for file operations
5. **SSRF protection** with DNS rebinding mitigation
6. **Rate limiting per channel** with token bucket
7. **Deny patterns for shell commands** (40+ regex patterns)

**Example - SSRF protection with DNS rebinding mitigation:**
```go
func newSafeDialContext(dialer *net.Dialer, whitelist *privateHostWhitelist) func(...) {
    return func(ctx context.Context, network, address string) (net.Conn, error) {
        // Resolve DNS at connect time (not pre-flight)
        ipAddrs, err := net.DefaultResolver.LookupIPAddr(ctx, host)
        for _, ipAddr := range ipAddrs {
            if shouldBlockPrivateIP(ipAddr.IP, whitelist) {
                continue
            }
            conn, err := dialer.DialContext(ctx, network, ...)
        }
    }
}
```

**Emulate this for:** Any project handling external URLs, file paths, or credentials.

---

### 5. Graceful Degradation Patterns

**Examples:**

- **Streaming fallbacks to placeholders:** If a channel doesn't support streaming, send a "Thinking..." message and edit it later when the response completes.
- **Per-channel capability detection via interface checks:**
```go
if pc, ok := ch.(PlaceholderCapable); ok {
    phID, _ := pc.SendPlaceholder(ctx, chatID)
    m.RecordPlaceholder(channel, chatID, phID)
}
```
- **Feature flags everywhere:** Config-driven feature enablement allows disabling problematic features without code changes.

**Emulate this for:** Projects targeting multiple platforms with varying capabilities.

---

### 6. Structured Logging with Component Awareness

**Pattern:** Every log entry includes component context, making debugging in production tractable.

```go
logger.InfoCF("agent", "Model routing: light model selected", map[string]any{
    "score":    score,
    "model":    agent.Router.LightModel(),
    "threshold": agent.Router.Threshold(),
})
```

**Why it works:** When searching logs for "Model routing" issues, you can filter by `component=agent` and avoid matching unrelated log lines.

**Emulate this for:** Any project that will run in production.

---

### 7. Atomic Operations for Lock-Free State

**Pattern:** Use `atomic.Bool`, `atomic.Value`, `sync.Map` for simple state transitions instead of mutex locks.

```go
type AgentLoop struct {
    running        atomic.Bool
    summarizing    sync.Map
    // ...
}

// Usage
if al.running.Swap(true) {
    return errors.New("already running")
}
```

**Why it works:** Lock-free operations are faster and simpler to reason about for simple state transitions.

**Emulate this for:** Any struct with simple boolean or map state that changes infrequently.

---

### 8. Two-Phase Operations for Lock Contention Reduction

**Pattern:** Modify data structures under lock, then release lock before performing slow operations (file I/O, network).

**Example from MediaStore cleanup:**
```go
// Phase 1: under lock - remove entries from maps
s.mu.Lock()
for ref := range refs {
    if removablePath, shouldDelete := s.releaseRefLocked(ref, fallbackPath); shouldDelete {
        paths = append(paths, removablePath)
    }
}
delete(s.scopeToRefs, scope)
s.mu.Unlock()

// Phase 2: no lock - delete files from disk
for _, p := range paths {
    os.Remove(p)
}
```

**Why it works:** Minimizes lock hold time, preventing goroutine starvation during slow I/O.

**Emulate this for:** Any cleanup operation involving files or network calls.

---

### 9. Configuration Migration System

**Pattern:** Version-based config migration with automatic migration on load.

```go
const CurrentVersion = 1

switch versionInfo.Version {
case 0:
    cfg, err = v.Migrate()
case CurrentVersion:
    cfg = v
}
```

**Why it works:** Users can keep their existing config files across upgrades. The system automatically adapts legacy formats to current schemas.

**Emulate this for:** Any project with persisted configuration.

---

### 10. Comprehensive Test Patterns

**Table-driven tests with helper functions:**
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

**Mock interfaces for testing:**
```go
type llmHookTestProvider struct {
    mu        sync.Mutex
    lastModel string
}

func (p *llmHookTestProvider) Chat(...) (*providers.LLMResponse, error) {
    // mock implementation
}
```

**Emulate this for:** Any non-trivial project.

---

## Part 2: What to Avoid

### 1. Massive Single Files (107KB AgentLoop)

**Problem:** `pkg/agent/loop.go` is 107KB and handles:
- Main agent loop
- Turn management
- MCP integration
- Steering queue
- Media handling
- Context building
- ...and more

**Consequence:** Hard to understand, test, or modify safely.

**Avoid this by:** Splitting into packages by concern. The architecture doc itself notes this as a concern.

---

### 2. Disabled Linters as Technical Debt

**Problem:** `.golangci.yaml` has 30+ linters disabled with TODO comments:

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

**Consequence:** Error handling inconsistencies, potential security issues, and type safety gaps.

**Avoid this by:** Enabling linters from day one or ruthlessly fixing issues before moving on.

---

### 3. Dual Gateway Management Implementations

**Problem:** Web UI (`web/backend/api/gateway.go`) and TUI (`cmd/picoclaw-launcher-tui/ui/gateway.go`) have separate code paths for managing the gateway subprocess.

**Differences:**
| Aspect | Web UI | TUI |
|--------|--------|-----|
| Detection | Health endpoint polling | PID file + process check |
| Logs | 200-line buffer | Not captured |
| Config | Environment variable | Environment variable |

**Consequence:** Divergence risk, duplicated maintenance effort, inconsistent behavior.

**Avoid this by:** Extracting shared gateway management into a common package.

---

### 4. Magic Numbers Not Centralized

**Problem:** Token budgets, timeouts, thresholds are scattered throughout code.

**Examples observed:**
- `maxOutputBufferSize = 1MB`
- `typingStopTTL = 5 minutes`
- `defaultChannelQueueSize = 16`
- `defaultConcurrencyTimeout = 30 * time.Second`
- `defaultSubTurnTimeout = 5 * time.Minute`

**Avoid this by:** Creating a `pkg/constants/constants.go` with all tunable parameters documented.

---

### 5. Large Config Struct (2000+ Lines)

**Problem:** `pkg/config/config.go` is 2175 lines with deeply nested structures.

**Consequence:** Hard to navigate, understand all options, or safely modify.

**Avoid this by:** Splitting into separate files by domain (`agents.go`, `channels.go`, `providers.go`, `security.go`).

---

### 6. Concurrent Model Cache Refresh Could Cause Thundering Herd

**Problem:** When TUI starts, it fetches models for ALL users simultaneously:

```go
func (a *App) refreshModelCache(onDone func()) {
    go func() {
        a.refreshMu.Lock()
        defer a.refreshMu.Unlock()
        // Fetches models for ALL users concurrently
    }()
}
```

**Consequence:** With many users configured, this could overwhelm provider endpoints.

**Avoid this by:** Implementing request coalescing or rate limiting on startup.

---

### 7. No Per-Process-Hook Timeouts

**Problem:** Hook timeouts are global defaults, not configurable per hook:

```go
const (
    defaultObserverTimeout = 500ms
    defaultInterceptorTimeout = 5 seconds
    defaultApprovalTimeout = 60 seconds
)
```

**Avoid this by:** Allowing per-hook timeout configuration in the hooks config section.

---

## Part 3: Surprises

### 1. Self-Registering Factories via init() Are Clean

**Discovery:** The factory registry uses Go's `init()` function for self-registration. This is elegant and eliminates central registration code.

**Surprise level:** Medium (this is a known Go pattern, but seeing it used systematically was clarifying)

---

### 2. Trigram-Based Similarity Caching

**Discovery:** The skills search cache uses trigram Jaccard similarity to recognize similar queries:

```go
func buildTrigrams(s string) []uint32 {
    // "hello" -> {"hel", "ell", "llo"}
    trigrams := make([]uint32, 0, len(s)-2)
    for i := 0; i <= len(s)-3; i++ {
        trigrams[i] = uint32(s[i])<<16 | uint32(s[i+1])<<8 | uint32(s[i+2])
    }
    slices.Sort(trigrams)
    // Deduplicate
}
```

**Surprise level:** High. This is sophisticated caching for a skills marketplace.

---

### 3. Wake Channel Pattern for Cron Scheduling

**Discovery:** The cron service uses a non-blocking channel send to wake up the scheduling loop immediately when a new job is added:

```go
func (cs *CronService) notify() {
    select {
    case cs.wakeChan <- struct{}{}:
    default:  // Non-blocking; if full, loop will wake soon anyway
    }
}
```

**Surprise level:** Medium. Clever use of Go's select with default.

---

### 4. runtime.KeepAlive for Syscall Safety

**Discovery:** When using `ioctl` for I2C/SPI, `runtime.KeepAlive()` is required to prevent GC from collecting buffer before the syscall completes:

```go
syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), spiIocMessage1, uintptr(uintptr(unsafe.Pointer(&xfer))))
runtime.KeepAlive(txBuf)  // Prevent GC from collecting txBuf before ioctl completes
runtime.KeepAlive(rxBuf)
```

**Surprise level:** High. This is an edge case specific to syscall interactions with GC.

---

### 5. Pre-Reset Pattern Prevents Double Execution

**Discovery:** The cron service clears `NextRunAtMS` before releasing the lock to prevent the same job from being picked up twice if a new check starts during execution:

```go
// Reset NextRunAtMS before unlocking to prevent duplicate execution
for i := range cs.store.Jobs {
    if dueMap[cs.store.Jobs[i].ID] {
        cs.store.Jobs[i].State.NextRunAtMS = nil
    }
}
cs.saveStoreUnsafe()
cs.mu.Unlock()
```

**Surprise level:** Medium. This is a subtle but important concurrency pattern.

---

### 6. Cyberpunk Theme Consistency

**Discovery:** Both TUI and Web UI use a cyberpunk/neon aesthetic with consistent hex color values:

- TUI: `#050510` (Deep Void), `#00f0ff` (Neon Cyan), `#ff00ff` (Neon Magenta)
- Web UI: Same palette via Tailwind CSS

**Surprise level:** Low (expected for a branded product), but nice attention to detail.

---

### 7. Embedded Frontend in Go Binary

**Discovery:** The React frontend is built and embedded directly into the Go binary using `//go:embed`:

```go
//go:embed all:dist
var frontendFS embed.FS

func registerEmbedRoutes(mux *http.ServeMux) {
    subFS, err := fs.Sub(frontendFS, "dist")
    mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // SPA fallback to index.html
    }))
}
```

**Surprise level:** Medium. This is elegant for single-binary distribution but requires build step before Go build.

---

### 8. Platform-Specific Build Fixes

**Discovery:** The Makefile contains workarounds for upstream library gaps:

1. **MIPS ELF flags patching** for NaN2008 kernel compatibility:
```makefile
define PATCH_MIPS_FLAGS
    printf '\004\024\000\160' | dd of=$(1) bs=1 seek=36 count=4 conv=notrunc
endef
```

2. **Loong64 PTY types generation:**
```makefile
printf '//go:build linux && loong64\npackage pty\ntype (_C_int int32; _C_uint uint32)\n' > "$$pty_dir/ztypes_loong64.go"
```

**Surprise level:** High. This level of platform-specific build engineering is impressive.

---

### 9. CGO_ENABLED=0 Except for macOS Web Launcher

**Discovery:** The project uses `CGO_ENABLED=0` for fully static binaries everywhere except macOS, where the web launcher requires CGO for some dependencies.

**Surprise level:** Low (known Go limitation), but the exception handling is noteworthy.

---

### 10. Memory Footprint vs. Feature Velocity Trade-off

**Discovery:** The project's goal is <10MB RAM, but current builds run 10-20MB due to rapid feature additions. The team plans to optimize at v1.0 after feature stabilization.

**Surprise level:** Medium. This is honest technical debt management - ship features first, optimize later.

---

## Summary Table

| Category | Count | Examples |
|----------|-------|----------|
| **Emulate** | 10 | Factory registry, MessageBus, error classification, security-first, graceful degradation |
| **Avoid** | 7 | 107KB files, disabled linters, duplicated implementations, magic numbers |
| **Surprises** | 10 | Trigram caching, wake channels, runtime.KeepAlive, platform build fixes |

---

## Files Referenced

- `pkg/agent/loop.go` - Main agent loop (107KB)
- `pkg/bus/bus.go` - MessageBus implementation
- `pkg/providers/types.go` - Provider interfaces
- `pkg/channels/registry.go` - Channel factory registry
- `pkg/providers/error_classifier.go` - Error classification
- `pkg/providers/fallback.go` - Fallback chain
- `.golangci.yaml` - Linting configuration
- `pkg/credential/credential.go` - Credential encryption
- `pkg/tools/web.go` - Web fetch with SSRF protection
- `pkg/cron/service.go` - Cron scheduling
- `pkg/skills/search_cache.go` - Trigram caching
- `pkg/tools/i2c_linux.go` - I2C with runtime.KeepAlive
- `Makefile` - Platform-specific build fixes
- `docker/Dockerfile` - Multi-stage builds
