# My Action Items: Building a PicoClaw-Style AI Agent

**Project Goal:** Build a similar multi-channel, multi-provider AI agent framework in Go
**Priority:** High-level guidance, not a technical specification
**Based on:** Lessons learned from github.com/sipeed/picoclaw analysis

---

## Phase 1: Foundation (Do First)

### 1.1 Establish Core Architecture

**Priority: CRITICAL**

- [ ] **Define interfaces before implementation**
  - Create `pkg/providers/interfaces.go` with `LLMProvider`, `StreamingProvider`, `ThinkingCapable` interfaces
  - Create `pkg/channels/interfaces.go` with `Channel`, `MediaSender`, `MessageEditor` interfaces
  - Create `pkg/tools/interfaces.go` with `Tool` interface
  - Document that interface-first design is non-negotiable

- [ ] **Implement MessageBus as central nervous system**
  - Three typed channels: `inbound`, `outbound`, `outboundMedia`
  - Buffered channels (64) for backpressure
  - StreamDelegate pattern for capability discovery

- [ ] **Set up factory registry pattern**
  - Self-registering factories via `init()`
  - Clear separation: interface in core, implementation in subpackages

- [ ] **Enable linting from day one**
  - Configure golangci-lint with all linters enabled
  - Do NOT add TODO comments to disable linters
  - Target zero lint errors before any commit

### 1.2 Establish Project Structure

**Priority: HIGH**

- [ ] **Use Go module monorepo** (`go.mod` in root)
- [ ] **Organize by domain, not layer:**
  ```
  pkg/
    agent/        # Core loop, context, turns
    providers/   # LLM provider implementations
    channels/    # Chat platform implementations
    tools/       # Tool registry and implementations
    bus/         # Message bus
    skills/      # Extensibility system
    config/      # Configuration management
  cmd/
    myagent/     # Main CLI entry point
    launcher/    # Web UI launcher
  web/
    frontend/    # React UI
    backend/     # Go web server
  ```
- [ ] **Set up embedded frontend** using `//go:embed` pattern
- [ ] **Configure goreleaser** for multi-platform releases from day one

### 1.3 Configure Build System

**Priority: HIGH**

- [ ] **Set up Makefile with:**
  - `make build` - current platform
  - `make build-all` - cross-platform builds
  - `make test` - with coverage output
  - `make lint` - golangci-lint
  - `make docker-build` - Docker image

- [ ] **Configure CGO appropriately:**
  - Default: `CGO_ENABLED=0` for static binaries
  - Exception: `CGO_ENABLED=1` for macOS if needed

- [ ] **Add GitHub Actions workflows:**
  - PR validation
  - Multi-platform build
  - Release on tag

---

## Phase 2: Core Features (Ship MVP)

### 2.1 Implement Agent Loop

**Priority: CRITICAL**

- [ ] **AgentLoop with clean responsibilities:**
  - Turn state management
  - Context building
  - LLM call orchestration
  - Tool execution loop
  - Hook integration

- [ ] **Keep file size under 500 lines per file**
  - If a file exceeds this, split by concern
  - Turn management -> separate package
  - Media handling -> separate package

- [ ] **Implement turn phases:**
  ```
  setup -> running -> tools -> finalizing -> completed/aborted
  ```

- [ ] **Add graceful shutdown:**
  - Drain in-flight messages
  - Stop all channels
  - Close agent loop

### 2.2 Implement First Provider (OpenAI)

**Priority: CRITICAL**

- [ ] **OpenAI provider as reference implementation:**
  - Chat completions API
  - Streaming support
  - Tool calling
  - Vision (image input)

- [ ] **Provider configuration:**
  - API key management
  - Base URL override for proxies
  - Timeout configuration
  - Retry with exponential backoff

- [ ] **Error classification:**
  - Rate limit detection
  - Auth failure detection
  - Context overflow detection
  - Retriable vs. non-retriable classification

### 2.3 Implement First Channel (Telegram)

**Priority: HIGH**

- [ ] **Telegram bot integration:**
  - Long polling via Bot API
  - Message sending with markdown/HTML
  - Reply support
  - Command registration

- [ ] **Rate limiting:**
  - Token bucket per channel
  - Configurable limits

- [ ] **Message handling:**
  - Inbound -> MessageBus
  - Outbound -> Channel worker

### 2.4 Implement Core Tools

**Priority: HIGH**

- [ ] **Tool registry with plugin architecture:**
  ```go
  type ToolRegistry struct {
      tools map[string]Tool
  }
  ```

- [ ] **Essential tools to implement:**
  - `shell` - Command execution (with deny patterns!)
  - `read_file` - With path containment validation
  - `write_file` - Atomic writes
  - `search` - Web search

- [ ] **Tool security:**
  - Workspace restriction
  - Path traversal prevention
  - Deny patterns for dangerous commands
  - Max output buffer (1MB)

---

## Phase 3: Reliability (Production Ready)

### 3.1 Error Handling & Fallback

**Priority: HIGH**

- [ ] **Implement FallbackChain:**
  - Multi-candidate failover
  - Cooldown tracking per provider/model
  - Aggregate errors on complete failure

- [ ] **Error sentinel hierarchy:**
  ```go
  var (
      ErrNotRunning = errors.New("channel not running")
      ErrRateLimit = errors.New("rate limited")
      ErrTemporary = errors.New("temporary failure")
      ErrSendFailed = errors.New("send failed")
  )
  ```

- [ ] **Graceful degradation:**
  - Streaming fallbacks to placeholders
  - Per-channel capability detection
  - Feature flags for optional capabilities

### 3.2 Configuration Management

**Priority: HIGH**

- [ ] **Separate security config:**
  - `config.json` - main config
  - `.security.yml` - API keys, tokens (gitignored)

- [ ] **Encryption support:**
  - AES-256-GCM for credentials at rest
  - HKDF key derivation
  - File reference support (`file://`)
  - Encrypted reference support (`enc://`)

- [ ] **Config migration system:**
  - Version-based migration
  - Automatic migration on load
  - Legacy format support

- [ ] **Centralize magic numbers:**
  ```go
  // pkg/constants/constants.go
  const (
      DefaultChannelQueueSize = 16
      DefaultRateLimit = 10
      MaxOutputBufferSize = 1 * 1024 * 1024 // 1MB
      TypingStopTTL = 5 * time.Minute
      // ...
  )
  ```

### 3.3 Testing Infrastructure

**Priority: HIGH**

- [ ] **Test patterns to establish:**
  - Table-driven tests
  - Helper setup functions (`t.Helper()`)
  - Mock interfaces
  - Deferred cleanup

- [ ] **Coverage requirements:**
  - Add `go test -cover` to Makefile
  - Target 70%+ coverage for core packages
  - Integration tests for provider/channel pairs

- [ ] **Test file distribution:**
  ```
  pkg/
    agent/
      loop.go
      loop_test.go       # Core tests
      context_test.go    # Context tests
  pkg/providers/
    factory_test.go
    fallback_test.go
  ```

---

## Phase 4: Extensibility (Scale)

### 4.1 Hook System

**Priority: MEDIUM**

- [ ] **Hook types:**
  - Observer (event notifications)
  - LLM Interceptor (before/after LLM calls)
  - Tool Interceptor (before/after tool calls)
  - Tool Approver (approval gates)

- [ ] **Hook implementation:**
  - In-process hooks
  - External process hooks via JSON-RPC over stdio
  - Priority ordering
  - Timeout handling

- [ ] **Hook actions:**
  - Continue
  - Modify
  - DenyTool
  - AbortTurn
  - HardAbort

### 4.2 Skills System

**Priority: MEDIUM**

- [ ] **SkillsLoader:**
  - Load from filesystem
  - Priority: workspace > global > builtin
  - YAML/JSON frontmatter parsing

- [ ] **Registry integration:**
  - Search with caching
  - Trigram similarity for query deduplication
  - Concurrent multi-registry search

- [ ] **Skill format:**
  - `SKILL.md` files
  - Markdown body for prompt content
  - Structured metadata

### 4.3 Additional Providers

**Priority: MEDIUM**

- [ ] **Provider priority:**
  1. OpenAI (reference implementation)
  2. Anthropic (native SDK)
  3. Local (Ollama, vLLM)
  4. OpenAI-compatible (Groq, DeepSeek, etc.)

- [ ] **Multi-key support:**
  - Round-robin key selection
  - Per-key cooldown tracking

### 4.4 Additional Channels

**Priority: MEDIUM**

- [ ] **Channel priority:**
  1. Telegram (reference implementation)
  2. Discord
  3. Slack
  4. Matrix

- [ ] **Channel capabilities:**
  - Typing indicators
  - Message editing
  - Streaming
  - Media sending

---

## Phase 5: Polish (Feature Complete)

### 5.1 Smart Routing

**Priority: LOW (implement after core works)**

- [ ] **Rule-based model routing:**
  - Token estimation (with CJK support)
  - Code block detection
  - Attachment detection
  - Conversation depth tracking

- [ ] **Scoring algorithm:**
  - Weighted sum of features
  - Configurable threshold
  - Light model vs. premium model selection

### 5.2 Sub-Agent System

**Priority: LOW**

- [ ] **Async task spawning:**
  - Fire-and-forget with status tracking
  - Synchronous blocking execution
  - Concurrency limits
  - Depth limits

- [ ] **Session isolation:**
  - Ephemeral session stores
  - Token budget sharing
  - Result delivery channels

### 5.3 Cron Scheduling

**Priority: LOW**

- [ ] **Schedule types:**
  - One-time ("remind me in 10 minutes")
  - Interval ("every 2 hours")
  - Cron expressions ("0 9 * * *")

- [ ] **Wake channel pattern:**
  - Non-blocking notify on job add
  - Pre-reset pattern to prevent double execution
  - Atomic writes with fsync

---

## Non-Negotiable Principles

### Code Quality

1. **No disabled linters** - Fix issues or don't commit
2. **File size limits** - Split files over 500 lines
3. **Test coverage** - 70%+ for core, measure it
4. **Structured logging** - Always with component context

### Security

1. **Credentials encrypted at rest** - Never plaintext in config
2. **Path containment validation** - No directory traversal
3. **SSRF protection** - DNS rebinding mitigation
4. **Deny patterns for shell** - Block dangerous commands
5. **Rate limiting** - Per-channel configurable

### Architecture

1. **Interface-first** - Define contracts before implementations
2. **Factory registry** - Self-registering via init()
3. **MessageBus** - Central pub/sub for component communication
4. **Graceful degradation** - Feature flags, capability detection
5. **Config migration** - Version-based, automatic

### Build & Release

1. **Static binaries** - `CGO_ENABLED=0` by default
2. **Multi-platform** - Support ARM, MIPS, RISC-V from day one
3. **Docker** - Multi-stage builds, minimal images
4. **Embedded frontend** - Single binary distribution

---

## Quick Reference: Key PicoClaw Patterns

### Factory Registration
```go
// Core package defines factory
var factories = map[string]FactoryFunc{}
func Register(name string, f FactoryFunc) { factories[name] = f }

// Implementation package self-registers
func init() { Register("telegram", NewTelegramChannel) }
```

### Error Classification
```go
func ClassifyError(err error) FailoverReason {
    switch {
    case errors.Is(err, context.DeadlineExceeded):
        return FailoverTimeout
    case strings.Contains(err.Error(), "rate limit"):
        return FailoverRateLimit
    // ...
    }
}
```

### Capability Detection
```go
if cap, ok := ch.(TypingCapable); ok {
    cap.SendTyping(ctx, chatID)
}
```

### Two-Phase Cleanup
```go
// Phase 1: Under lock
mu.Lock()
delete(map, key)
mu.Unlock()

// Phase 2: Without lock
file, err := os.Open(path)
defer file.Close()
```

### Atomic State
```go
type Manager struct {
    closed atomic.Bool
    running sync.Map
}

if m.closed.Swap(true) {
    return errors.New("already closed")
}
```

---

## Files to Create First

When starting implementation, create these files first:

1. `pkg/providers/interfaces.go` - LLMProvider interface
2. `pkg/channels/interfaces.go` - Channel interface
3. `pkg/bus/bus.go` - MessageBus implementation
4. `pkg/tools/interfaces.go` - Tool interface
5. `pkg/config/config.go` - Configuration with migration
6. `.golangci.yaml` - Linting configuration
7. `Makefile` - Build automation
8. `cmd/myagent/main.go` - Entry point

---

## Inspiration Files (Copy Patterns From)

| PicoClaw Pattern | Source File | Lines |
|-----------------|--------------|-------|
| Factory registry | `pkg/channels/registry.go` | 33 |
| Provider interface | `pkg/providers/types.go` | ~100 |
| Channel interface | `pkg/channels/interfaces.go` | ~150 |
| MessageBus | `pkg/bus/bus.go` | ~200 |
| Error classification | `pkg/providers/error_classifier.go` | 266 |
| Fallback chain | `pkg/providers/fallback.go` | 305 |
| Credential encryption | `pkg/credential/credential.go` | ~300 |
| SSRF protection | `pkg/tools/web.go` | 1100+ |
| Cron service | `pkg/cron/service.go` | ~400 |
| Hook system | `pkg/agent/hooks.go` | ~400 |
