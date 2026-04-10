# PicoClaw Feature Deep Dive: Batch 2

**Date:** 2026-03-26
**Features:** MCP Integration, Vision Pipeline, Gateway Service
**Repo:** `/Users/sheldon/Documents/claw/reference/picoclaw`

---

## 1. MCP (Model Context Protocol) Integration

### Overview
Native MCP protocol support to extend agent capabilities with external tools and data sources via stdio, SSE, or HTTP transports.

### Core Implementation Files
| File | Purpose |
|------|---------|
| `pkg/mcp/manager.go` | MCP server connection management |
| `pkg/tools/mcp_tool.go` | MCP tool wrapper for tool registry |
| `pkg/agent/loop_mcp.go` | Agent loop integration for MCP initialization |

### Architecture

**Manager (pkg/mcp/manager.go)**
The `Manager` struct manages multiple MCP server connections:

```go
type ServerConnection struct {
    Name    string
    Client  *mcp.Client
    Session *mcp.ClientSession
    Tools   []*mcp.Tool
}

type Manager struct {
    servers map[string]*ServerConnection
    mu      sync.RWMutex
    closed  atomic.Bool
    wg      sync.WaitGroup // tracks in-flight CallTool calls
}
```

**Transport Support:**
1. **stdio** - For command-based servers (auto-detected when `Command` is provided)
2. **SSE/HTTP** - For web-based servers (auto-detected when `URL` is provided)

```go
// Auto-detection logic
if transportType == "" {
    if cfg.URL != "" {
        transportType = "sse"
    } else if cfg.Command != "" {
        transportType = "stdio"
    }
}
```

**Environment Variable Handling (Notable):**
The implementation has a sophisticated env file loader that:
- Loads from `.env` format files
- Parent process environment is the base
- File variables override parent env
- Config variables override file variables

```go
// Build environment variables with proper override semantics
envMap := make(map[string]string)
for _, e := range cmd.Environ() {  // Start with parent
    envMap[e[:idx]] = e[idx+1:]
}
for k, v := range envVars {  // File overrides
    envMap[k] = v
}
for k, v := range cfg.Env {  // Config overrides file
    envMap[k] = v
}
```

**HTTP Header Injection:**
For SSE transport, supports custom headers via a wrapping `RoundTripper`:

```go
type headerTransport struct {
    base    http.RoundTripper
    headers map[string]string
}
```

**Graceful Shutdown:**
Uses atomic.Bool for closed state with double-checked locking to prevent TOCTOU races:

```go
if m.closed.Swap(true) {  // Atomic swap - returns old value
    return nil // already closed
}
m.wg.Wait()  // Wait for in-flight calls before closing sessions
```

### Tool Wrapper (pkg/tools/mcp_tool.go)

**Name Sanitization:**
MCP tool names are sanitized to be compatible with OpenAI-style APIs (max 64 chars, lowercase, alphanumeric with `_` and `-`):

```go
// Format: mcp_{serverName}_{toolName}
// If sanitization is lossy or name too long, append 8-char FNV hash
full := fmt.Sprintf("mcp_%s_%s", sanitizedServer, sanitizedTool)
```

The `sanitizeIdentifierComponent` function:
- Lowercases the string
- Replaces disallowed chars with `_`
- Collapses multiple consecutive `_` into one
- Truncates to 64 chars max
- Falls back to "unnamed" if empty

**Content Type Handling:**
The wrapper handles all MCP content types:
- `TextContent` - Passed through with sanitization
- `ImageContent` - Stored to temp file, registered with MediaStore, returns reference
- `AudioContent` - Same as ImageContent
- `ResourceLink` - Summarized as text
- `EmbeddedResource` - Either stored (if blob) or summarized (if text)

**Media Delivery from MCP Tools:**
When MCP returns binary content (images/audio), it's stored to temp and registered:

```go
scope := fmt.Sprintf(
    "tool:mcp:%s:%s:%s:%d",
    sanitizeIdentifierComponent(t.serverName),
    channel,
    chatID,
    time.Now().UnixNano(),
)
ref, err := t.mediaStore.Store(tmpPath, media.MediaMeta{...}, scope)
```

### Agent Loop Integration (pkg/agent/loop_mcp.go)

**Lazy One-Time Initialization:**
Uses `sync.Once` for initialization that both `Run()` and direct agent mode share:

```go
type mcpRuntime struct {
    initOnce sync.Once
    mu       sync.Mutex
    manager  *mcp.Manager
    initErr  error
}
```

**Per-Server Deferred Mode:**
Servers can be configured as "deferred" (hidden from LLM until discovered via search):

```go
func serverIsDeferred(discoveryEnabled bool, serverCfg config.MCPServerConfig) bool {
    if !discoveryEnabled {
        return false
    }
    if serverCfg.Deferred != nil {
        return *serverCfg.Deferred
    }
    return true
}
```

**Tool Discovery (Optional):**
When `Discovery.Enabled` is true, registers BM25 and/or regex search tools:

```go
if useRegex {
    agent.Tools.Register(tools.NewRegexSearchTool(agent.Tools, ttl, maxSearchResults))
}
if useBM25 {
    agent.Tools.Register(tools.NewBM25SearchTool(agent.Tools, ttl, maxSearchResults))
}
```

### Technical Debt / Notes
1. MCP SDK dependency: `github.com/modelcontextprotocol/go-sdk/mcp` (external)
2. Tool discovery requires explicit config - no automatic fallback
3. No connection pooling or reconnection logic visible
4. Tool names capped at 64 chars which may cause collisions with FNV hash (mitigated by hash uniqueness)

---

## 2. Vision Pipeline

### Overview
Send images and files directly to the agent with automatic base64 encoding for multimodal LLMs, or local path injection for file access tools.

### Core Implementation Files
| File | Purpose |
|------|---------|
| `pkg/agent/loop_media.go` | Media ref resolution in agent loop |
| `pkg/media/store.go` | Media storage and lifecycle management |
| `pkg/tools/send_file.go` | File sending tool for LLM |

### Architecture

**Media Ref Resolution (pkg/agent/loop_media.go)**

The `resolveMediaRefs` function transforms `media://` references into actual content:

```go
func resolveMediaRefs(messages []providers.Message, store media.MediaStore, maxSize int) []providers.Message
```

**Dual-Path Strategy:**

1. **Images** - Base64-encoded as data URLs for multimodal LLMs:
```go
if strings.HasPrefix(mime, "image/") {
    dataURL := encodeImageToDataURL(localPath, mime, info, maxSize)
    resolved = append(resolved, dataURL)  // "data:image/png;base64,iVBORw0KGgo..."
    continue
}
```

2. **Non-Images** - Local path injected as structured tags:
```go
// Creates [audio:/path], [video:/path], or [file:/path] tags
pathTags = append(pathTags, buildPathTag(mime, localPath))
// Injected into message content for file tool access
result[i].Content = injectPathTags(result[i].Content, pathTags)
```

**MIME Detection:**
Uses `h2non/filetype` for magic-byte detection, falls back to extension lookup:

```go
func detectMIME(localPath string, meta media.MediaMeta) string {
    if meta.ContentType != "" {
        return meta.ContentType
    }
    kind, err := filetype.MatchFile(localPath)
    if err != nil || kind == filetype.Unknown {
        return ""
    }
    return kind.MIME.Value
}
```

**Path Tag Injection:**
Generic tags like `[audio]`, `[video]`, `[file]` in message content are replaced with path-bearing versions:

```go
func injectPathTags(content string, tags []string) string {
    for _, tag := range tags {
        // [audio:/path] replaces [audio] if present, else appends
        if strings.Contains(content, generic) {
            content = strings.Replace(content, generic, tag, 1)
        }
    }
}
```

### Media Store (pkg/media/store.go)

**Interface:**
```go
type MediaStore interface {
    Store(localPath string, meta MediaMeta, scope string) (ref string, err error)
    Resolve(ref string) (localPath string, err error)
    ResolveWithMeta(ref string) (localPath string, meta MediaMeta, err error)
    ReleaseAll(scope string) error
}
```

**Scope-Based Organization:**
Media refs are organized by "scope" (e.g., `tool:send_file:telegram:12345`) enabling bulk cleanup:

```go
type FileMediaStore struct {
    refs        map[string]mediaEntry           // ref -> entry
    scopeToRefs map[string]map[string]struct{}  // scope -> refs
    refToScope  map[string]string               // ref -> scope
    refToPath   map[string]string               // ref -> path
    pathStates  map[string]pathRefState         // path -> refCount/deleteEligible
}
```

**Cleanup Policies:**
```go
const (
    CleanupPolicyDeleteOnCleanup = "delete_on_cleanup"  // Store-managed, can delete
    CleanupPolicyForgetOnly      = "forget_only"        // External, never delete
)
```

The `send_file` tool uses `CleanupPolicyForgetOnly` since the user already has the file.

**Two-Phase Cleanup (Lock-Free File Deletion):**
Files are removed in two phases to minimize lock contention:

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

**Background TTL Cleanup:**
Configurable interval-based cleanup of expired entries:

```go
go func() {
    ticker := time.NewTicker(s.cleanerCfg.Interval)
    for {
        select {
        case <-ticker.C:
            if n := s.CleanExpired(); n > 0 {
                logger.InfoCF("media", "cleanup: removed expired entries", map[string]any{"count": n})
            }
        case <-s.stop:
            return
        }
    }
}()
```

### Send File Tool (pkg/tools/send_file.go)

**Path Validation:**
Validates path against workspace restrictions and allow-list patterns:

```go
func (t *SendFileTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    resolved, err := validatePathWithAllowPaths(path, t.workspace, t.restrict, t.allowPaths)
    // ...
}
```

**MIME Detection Priority:**
1. Magic-byte detection (filetype)
2. Extension lookup (mime.TypeByExtension)
3. Fallback to `application/octet-stream`

### Technical Debt / Notes
1. No image resizing/compression - large images sent as-is (capped only by maxSize)
2. Temp files created in `os.TempDir()` - could conflict with other processes
3. No content-addressable storage - same file stored multiple times
4. Scope TTL is fixed at store level, not per-entry

---

## 3. Gateway Service

### Overview
HTTP/WebSocket gateway that bridges chat channels to the AI agent, handling message routing, authentication, and protocol translation.

### Core Implementation Files
| File | Purpose |
|------|---------|
| `pkg/gateway/gateway.go` | Main gateway orchestration, service lifecycle |
| `pkg/channels/manager.go` | Channel orchestration, message routing, worker queues |

### Architecture (pkg/gateway/gateway.go)

**Service Startup Order:**
```go
func setupAndStartServices(cfg *config.Config, agentLoop *agent.AgentLoop, msgBus *bus.MessageBus) (*services, error) {
    // 1. Cron service (scheduled tasks)
    // 2. Heartbeat service (periodic health checks)
    // 3. Media store (with background cleanup)
    // 4. Channel manager (connects to all chat platforms)
    // 5. Device service (hardware events)
    // 6. Health server (HTTP endpoints)
}
```

**Hot Reload Support:**
Config reloads via:
1. File watcher (polls every 2 seconds for mtime/size changes)
2. Manual trigger via `POST /reload` endpoint

```go
func setupConfigWatcherPolling(configPath string, debug bool) (chan *config.Config, func()) {
    ticker := time.NewTicker(2 * time.Second)
    for {
        select {
        case <-ticker.C:
            currentModTime := getFileModTime(configPath)
            currentSize := getFileSize(configPath)
            if currentModTime.After(lastModTime) || currentSize != lastSize {
                newCfg, err := config.LoadConfig(configPath)
                // Validate before sending
                if err := newCfg.ValidateModelList(); err != nil {
                    continue  // Use previous valid config
                }
                configChan <- newCfg
            }
        }
    }
}
```

**Reload Process:**
1. Stops all services (except channel manager)
2. Recreates provider with new config
3. Reloads agent loop with new provider and config
4. Restarts all services

### Channel Manager (pkg/channels/manager.go)

**Architecture Pattern:**
- Worker-based async message processing
- Per-channel rate limiting (token bucket)
- Dedicated queues for text and media messages
- TTL-based cleanup for ephemeral state

**Worker Structure:**
```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage      // text messages
    mediaQueue chan bus.OutboundMediaMessage // media messages
    done       chan struct{}
    mediaDone  chan struct{}
    limiter    *rate.Limiter                  // token bucket per channel
}
```

**Rate Limits by Channel:**
```go
var channelRateConfig = map[string]float64{
    "telegram": 20,
    "discord":  1,
    "slack":    1,
    "matrix":   2,
    "line":     10,
    "qq":       5,
    "irc":      2,
}
// Burst = ceil(rate / 2)
burst := int(math.Max(1, math.Ceil(rateVal/2)))
```

**Per-Channel Capabilities via Interface Checks:**
```go
// Typing indicators
if pc, ok := ch.(PlaceholderCapable); ok {
    phID, _ := pc.SendPlaceholder(ctx, chatID)
    m.RecordPlaceholder(channel, chatID, phID)
}

// Message editing (for placeholder updates)
if editor, ok := ch.(MessageEditor); ok {
    editor.EditMessage(ctx, msg.ChatID, entry.id, msg.Content)
}

// Streaming support
if sc, ok := ch.(StreamingCapable); ok {
    streamer, _ := sc.BeginStream(ctx, chatID)
}
```

**Message Pre-Send Hooks:**
Before every outbound message, the manager:
1. Stops typing indicator
2. Undoes any reaction
3. Deletes or edits placeholder message

```go
func (m *Manager) preSend(ctx context.Context, name string, msg bus.OutboundMessage, ch Channel) bool {
    // 1. Stop typing
    if v, loaded := m.typingStops.LoadAndDelete(key); loaded {
        entry.stop()  // idempotent
    }
    // 2. Undo reaction
    if v, loaded := m.reactionUndos.LoadAndDelete(key); loaded {
        entry.undo()  // idempotent
    }
    // 3. Handle streaming finalize marker
    if _, loaded := m.streamActive.LoadAndDelete(key); loaded {
        // Stream finalized - delete placeholder
        if deleter, ok := ch.(MessageDeleter); ok {
            deleter.DeleteMessage(ctx, msg.ChatID, entry.id)
        }
        return true  // skip Send
    }
    // 4. Try editing placeholder
    if editor, ok := ch.(MessageEditor); ok {
        if err := editor.EditMessage(ctx, msg.ChatID, entry.id, msg.Content); err == nil {
            return true  // edited successfully
        }
    }
    return false  // proceed with Send
}
```

**Retry with Exponential Backoff:**
```go
for attempt := 0; attempt <= maxRetries; attempt++ {
    lastErr = w.ch.Send(ctx, msg)
    if lastErr == nil {
        return
    }
    // Permanent failures - don't retry
    if errors.Is(lastErr, ErrNotRunning) || errors.Is(lastErr, ErrSendFailed) {
        break
    }
    // Rate limit - fixed delay
    if errors.Is(lastErr, ErrRateLimit) {
        time.Sleep(rateLimitDelay)
        continue
    }
    // Temporary - exponential backoff
    backoff := min(baseBackoff * 2^attempt, maxBackoff)
    time.Sleep(backoff)
}
```

**Message Splitting:**
Two-stage splitting:
1. Marker-based (if enabled in config) - splits on semantic markers like `---`
2. Length-based fallback - respects channel's `MaxMessageLength`

```go
// Stage 1: Marker-based
if markerChunks := SplitByMarker(msg.Content); len(markerChunks) > 1 {
    for _, chunk := range markerChunks {
        chunks = append(chunks, splitByLength(chunk, maxLen)...)
    }
}
// Stage 2: Length-based
if len(chunks) == 0 {
    chunks = splitByLength(msg.Content, maxLen)
}
```

**TTL Janitor:**
Background goroutine cleans up stale state:
```go
func (m *Manager) runTTLJanitor(ctx context.Context) {
    ticker := time.NewTicker(janitorInterval) // 10 seconds
    for {
        m.typingStops.Range(func(key, value any) bool {
            if now.Sub(entry.createdAt) > typingStopTTL { // 5 minutes
                m.typingStops.LoadAndDelete(key)
                entry.stop()
            }
            return true
        })
        // Similar for placeholders (10 min TTL) and reactionUndos (5 min TTL)
    }
}
```

**Shared HTTP Server:**
All channels share one HTTP server for webhooks and health:
```go
func (m *Manager) SetupHTTPServer(addr string, healthServer *health.Server) {
    m.mux = http.NewServeMux()
    // Register health endpoints
    healthServer.RegisterOnMux(m.mux)
    // Discover webhook handlers from channels
    for name, ch := range m.channels {
        if wh, ok := ch.(WebhookHandler); ok {
            m.mux.Handle(wh.WebhookPath(), wh)
        }
        if hc, ok := ch.(HealthChecker); ok {
            m.mux.HandleFunc(hc.HealthPath(), hc.HealthHandler)
        }
    }
}
```

### Graceful Shutdown Order
1. Shutdown HTTP server (5s timeout)
2. Cancel dispatcher (stops new message processing)
3. Drain worker queues (wait for in-flight messages)
4. Stop all channels
5. Close agent loop

### Technical Debt / Notes
1. No dead letter queue for failed messages
2. Worker queue depth is fixed at 16 - could overflow during spikes
3. Config file polling every 2s is coarse-grained
4. No back-pressure when channel send is slow
5. HTTP server timeout is fixed (30s read/write)

---

## Cross-Feature Interactions

**MCP -> Media:**
When MCP tools return binary content (images, audio), they use the MediaStore to temporarily store files, creating scopes like `tool:mcp:server-name:telegram:chatid:timestamp`.

**Gateway -> Channels:**
The gateway orchestrates channels through the ChannelManager, which handles all protocol translation and rate limiting.

**Vision -> Gateway:**
Media files sent via the `send_file` tool flow through MediaStore and are delivered via the ChannelManager's media queue.

---

## Key Design Patterns

1. **Two-Phase Operations:** Media cleanup and state cleanup use two-phase commit (modify maps under lock, then delete files without lock) to minimize lock contention.

2. **Interface-Based Capability Detection:** Channels advertise capabilities via interface checks, allowing flexible feature detection without inheritance.

3. **Scope-Based Lifecycle:** Media and other resources are grouped by scope (typically `channel:chatID`) enabling bulk cleanup.

4. **Wrapper Streams:** `finalizeHookStreamer` wraps streaming implementations to run hooks on finalize without modifying the underlying streamer.

5. **Atomic State with Double-Checked Locking:** Closed state uses `atomic.Bool` with `Swap()` to prevent TOCTOU races in MCP manager.
