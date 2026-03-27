# PicoClaw Security and Performance Patterns Analysis

## Overview

This document analyzes the security and performance patterns implemented in PicoClaw, covering secrets management, authentication/authorization, input validation, and performance optimizations.

---

## 1. Secrets Management

### 1.1 Separate Security Configuration File

PicoClaw stores sensitive data in a dedicated `.security.yml` file, separate from the main `config.json`.

**Location:** `pkg/config/security.go`

```go
const (
    SecurityConfigFile = ".security.yml"
)

type SecurityConfig struct {
    ModelList map[string]ModelSecurityEntry `yaml:"model_list"`
    Channels *ChannelsSecurity              `yaml:"channels,omitempty"`
    Web      *WebToolsSecurity              `yaml:"web,omitempty"`
    Skills   *SkillsSecurity               `yaml:"skills,omitempty"`
}
```

**Key security properties:**
- File permissions set to `0o600` on save (read/write for owner only)
- Automatic mapping to config fields by field name
- Environment variable override with highest priority
- Supports multiple API keys per model for load balancing

### 1.2 Encrypted Credentials (`enc://`)

PicoClaw supports AES-256-GCM encryption for stored credentials.

**Location:** `pkg/credential/credential.go`

**Encryption scheme:**
- Format: `enc://<base64-encoded-blob>`
- Blob structure: `salt (16 bytes) + nonce (12 bytes) + ciphertext`
- Key derivation: HKDF-SHA256 using passphrase + SSH key hash
- Passphrase: `PICOCLAW_KEY_PASSPHRASE` environment variable
- SSH key path: `PICOCLAW_SSH_KEY_PATH` or `~/.ssh/picoclaw_ed25519.key`

```go
// Key derivation from passphrase and SSH key
ikm = HMAC-SHA256(key=SHA256(sshKeyBytes), msg=passphrase)
key = HKDF-SHA256(ikm, salt, info="picoclaw-credential-v1", 32 bytes)
```

### 1.3 File Reference Credentials (`file://`)

API keys can reference external files, avoiding storage in config:

```yaml
model_list:
  gpt-5.4:
    api_keys:
      - "file://api-keys/production-key.txt"
```

**Security features:**
- Symlinks resolved before path validation
- Path containment check via `filepath.IsLocal`
- Only files within config directory allowed

### 1.4 Sensitive Data Filtering in Logs

The `SensitiveDataReplacer` uses a `strings.Replacer` compiled once via `sync.Once` to filter sensitive values from logs:

```go
type SensitiveDataCache struct {
    replacer *strings.Replacer
    once     sync.Once
}
```

---

## 2. Authentication and Authorization

### 2.1 OAuth Implementation

**Browser-based OAuth flow with PKCE:**

Location: `pkg/auth/oauth.go`

```go
type OAuthProviderConfig struct {
    Issuer       string
    ClientID     string
    ClientSecret string
    TokenURL     string
    Scopes       string
    Originator   string
    Port         int
}
```

**Security measures:**
- **PKCE (Proof Key for Code Exchange):** `GeneratePKCE()` creates `code_verifier` and `code_challenge`
- **State parameter:** CSRF protection with random 32-byte state
- **Timeout:** 5-minute authentication timeout
- **Redirect URI validation:** Uses localhost callback on specific port
- **Code exchange:** Authorization code exchanged for tokens with PKCE verifier

**OAuth providers supported:**
- OpenAI (`auth.openai.com`)
- Google Antigravity (`accounts.google.com`)
- Device code flow for headless environments

### 2.2 WeCom Channel Authentication

Location: `pkg/channels/wecom/wecom.go`

**WebSocket-based authentication:**
- Bot ID and Secret sent on connection
- Connection timeout: 15 seconds
- Heartbeat interval: 30 seconds

```go
const (
    wecomConnectTimeout    = 15 * time.Second
    wecomHeartbeatInterval = 30 * time.Second
)
```

### 2.3 IP Allowlisting for Web UI

Location: `web/backend/middleware/access_control.go`

```go
func IPAllowlist(allowedCIDRs []string, next http.Handler) (http.Handler, error)
```

- Empty CIDR list allows all requests
- Loopback addresses always allowed
- CIDR ranges: `192.168.1.0/24`, `10.0.0.0/8`, etc.

### 2.4 Token Caching

Location: `pkg/channels/feishu/token_cache.go`

```go
type tokenCache struct {
    mu    sync.RWMutex
    store map[string]*tokenEntry
}

type tokenEntry struct {
    value    string
    expireAt time.Time
}
```

Custom token cache with TTL and invalidation support, works around Lark SDK bugs.

---

## 3. Input Validation and Sanitization

### 3.1 Tool Argument Validation

Location: `pkg/tools/validate.go`

JSON Schema-like validation for tool arguments:

```go
func validateToolArgs(schema map[string]any, args map[string]any) error {
    // Validates:
    // - Required fields presence
    // - Type checking (string, integer, number, boolean, array, object)
    // - Enum constraints
    // - Additional properties allowed/denied
}
```

**Supported types:**
- `string`, `integer`, `number`, `boolean`, `array`, `object`
- `enum` validation
- Nested object validation

### 3.2 Path Validation and Containment

Location: `pkg/tools/filesystem.go`

```go
const MaxReadFileSize = 64 * 1024 // 64KB limit

func validatePathWithAllowPaths(path, workspace string, restrict bool, patterns []*regexp.Regexp) (string, error) {
    // 1. Resolves relative paths to absolute
    // 2. Checks symlinks for traversal attacks
    // 3. Validates against allow patterns
    // 4. Ensures path within workspace boundary
}
```

**Security features:**
- `filepath.EvalSymlinks()` resolves symlinks before validation
- `isWithinWorkspace()` prevents directory traversal
- `filepath.IsLocal()` for cross-platform path traversal detection
- Restricted mode blocks access outside workspace

### 3.3 Markdown/HTML Sanitization

Location: `pkg/utils/markdown.go`

```go
var skipTags = map[string]bool{
    "script": true, "style": true, "head": true,
    "noscript": true, "template": true,
    "nav": true, "footer": true, "aside": true,
    "header": true, "form": true, "dialog": true,
}

func isSafeHref(href string) bool {
    // Blocks: javascript:, vbscript:, data: schemes
    // Allows: http, https, mailto, or empty
}
```

**Allowed URL schemes:** `http`, `https`, `mailto`, `data:image/*`

### 3.4 Inline Media Sanitization

Location: `pkg/tools/normalization.go`

```go
func sanitizeToolLLMContent(text string) string {
    // Detects and omits:
    // - Inline data URLs in markdown
    // - Large base64 payloads (>1024 chars with >97% base64 chars)
}
```

**Protections:**
- Strips inline `data:` URLs from tool output
- Detects large base64 payloads to prevent context overflow
- Temp files created with `0o700` permissions

### 3.5 Message Type Validation

Location: `pkg/channels/wecom/wecom.go`

```go
switch msg.MsgType {
case "text":
    // validate content
case "voice":
    // extract and validate
case "image", "file", "video":
    // handle media
case "mixed":
    // handle mixed content
default:
    return c.respondImmediate(reqID, "Unsupported WeCom message type: "+msg.MsgType)
}
```

---

## 4. Rate Limiting and Throttling

### 4.1 Per-Channel Rate Limits

Location: `pkg/channels/manager.go`

```go
const (
    defaultChannelQueueSize = 16
    defaultRateLimit        = 10 // default 10 msg/s
    maxRetries              = 3
    rateLimitDelay          = 1 * time.Second
)

var channelRateConfig = map[string]float64{
    "telegram": 20,
    "discord":  1,
    "slack":    1,
    "matrix":   2,
    "line":     10,
    "qq":       5,
    "irc":      2,
}
```

**Implementation:**
- Uses `golang.org/x/time/rate` limiter
- Per-channel configurable limits
- Message queuing with `defaultChannelQueueSize`

### 4.2 Provider-Level Rate Limiting

Location: `pkg/providers/error_classifier.go`

```go
const FailoverRateLimit FailoverReason = "rate_limit"

// Detects rate limit patterns:
// - "rate limit exceeded"
// - "rate_limit reached"
// - HTTP 429 status
// - "overloaded" treated as rate limit
```

### 4.3 Streaming Rate Limiting

Location: `pkg/channels/wecom/wecom.go`

```go
const (
    wecomStreamMaxDuration = 5*time.Minute + 30*time.Second
    wecomStreamMinInterval = 500 * time.Millisecond
)
```

- Minimum 500ms between stream chunks
- 5.5 minute maximum stream duration

---

## 5. Caching Strategies

### 5.1 Search Result Caching

Location: `pkg/skills/search_cache.go`

```go
type SearchCache struct {
    mu         sync.RWMutex
    entries    map[string]*cacheEntry
    order      []string // LRU order
    maxEntries int
    ttl        time.Duration
}

const similarityThreshold = 0.7
```

**Features:**
- Trigram-based similarity matching (Jaccard similarity)
- LRU eviction policy
- Configurable TTL (default 5 minutes)
- Max entries limit (default 50)
- Thread-safe with RWMutex

### 5.2 System Prompt Caching

Location: `pkg/agent/context.go`

```go
type ContextBuilder struct {
    systemPromptMutex  sync.RWMutex
    cachedSystemPrompt string
    cachedAt           time.Time
    existedAtCache     map[string]bool
    skillFilesAtCache  map[string]time.Time
}
```

**Cache invalidation:**
- Monitors mtime of workspace source files
- Detects new file creation and deletion
- Rebuilds cache when files change

### 5.3 Session Storage with Sharding

Location: `pkg/memory/jsonl.go`

```go
const (
    numLockShards = 64
    maxLineSize   = 10 * 1024 * 1024 // 10 MB
)

type JSONLStore struct {
    dir   string
    locks [numLockShards]sync.Mutex
}
```

**Features:**
- FNV hash-based sharding for O(1) mutex lookup
- Fixed 64 mutexes regardless of session count
- Append-only JSONL (crash-safe)
- Logical truncation via skip offset (metadata)

### 5.4 Token Cache with TTL

Location: `pkg/channels/feishu/token_cache.go`

```go
func (c *tokenCache) Get(_ context.Context, key string) (string, error) {
    e, ok := c.store[key]
    if !ok {
        return "", nil
    }
    if e.expireAt.Before(time.Now()) {
        delete(c.store, key)
        return "", nil
    }
    return e.value, nil
}
```

---

## 6. Connection Management

### 6.1 WebSocket Connection with Reconnect

Location: `pkg/channels/wecom/wecom.go`

```go
func (c *WeComChannel) connectLoop() {
    backoff := time.Second
    for {
        if err := c.runConnection(); err != nil {
            // Exponential backoff: 1s -> 2s -> 4s -> ... -> max 1min
            backoff *= 2
            if backoff > time.Minute {
                backoff = time.Minute
            }
        }
    }
}
```

**Features:**
- Exponential backoff with 1-minute cap
- Context cancellation support
- Connection cleanup on error

### 6.2 HTTP Client Configuration

```go
mediaClient: &http.Client{Timeout: wecomMediaTimeout}

const wecomMediaTimeout = 30 * time.Second
```

- Per-channel HTTP clients with timeouts
- Graceful cleanup on channel stop

### 6.3 Recent Message Deduplication

Location: `pkg/channels/wecom/wecom.go`

```go
type recentMessageSet struct {
    mu   sync.Mutex
    seen map[string]struct{}
    ring []string
    idx  int
}

const wecomRecentMessageMax = 1000
```

- Ring buffer for O(1) dedup check
- 1000 message history

---

## 7. Performance Patterns

### 7.1 Request/Response Matching

Location: `pkg/channels/wecom/wecom.go`

```go
type WeComChannel struct {
    pendingMu sync.Mutex
    pending   map[string]chan wecomEnvelope
}
```

- Request ID to channel mapping
- Timeout-wrapped response waiting
- Automatic cleanup on timeout

### 7.2 Turn Queue Management

```go
type wecomTurn struct {
    ReqID     string
    ChatID    string
    ChatType  uint32
    StreamID  string
    CreatedAt time.Time
}

turnsMu sync.Mutex
turns   map[string][]wecomTurn
```

- Per-chat turn queuing
- FIFO consumption
- TTL-based expiration

### 7.3 Session Manager with In-Memory Cache

Location: `pkg/session/manager.go`

```go
type SessionManager struct {
    sessions map[string]*Session
    mu       sync.RWMutex
    storage  string
}
```

- In-memory sessions with optional disk persistence
- Read-write mutex for concurrent access
- Lazy loading from disk

### 7.4 Parallel Message Processing

Location: `pkg/channels/manager.go`

```go
type channelWorker struct {
    ch         Channel
    queue      chan bus.OutboundMessage
    mediaQueue chan bus.OutboundMediaMessage
    limiter    *rate.Limiter
}
```

- Dedicated goroutine per channel
- Separate queues for regular and media messages
- Rate limiting at worker level

---

## 8. Security Best Practices Summary

| Category | Practice | Location |
|----------|----------|----------|
| Secrets | Separate security config file | `pkg/config/security.go` |
| Secrets | AES-256-GCM encryption | `pkg/credential/credential.go` |
| Secrets | File permission 0o600 | `saveSecurityConfig()` |
| Auth | OAuth PKCE + state parameter | `pkg/auth/oauth.go` |
| Auth | IP allowlisting | `web/backend/middleware/access_control.go` |
| Input | JSON Schema validation | `pkg/tools/validate.go` |
| Input | Path containment checks | `pkg/tools/filesystem.go` |
| Input | HTML sanitization | `pkg/utils/markdown.go` |
| Input | Base64 payload detection | `pkg/tools/normalization.go` |
| Rate Limit | Per-channel limits | `pkg/channels/manager.go` |
| Rate Limit | Provider failover | `pkg/providers/error_classifier.go` |

---

## 9. Performance Best Practices Summary

| Category | Practice | Location |
|----------|----------|----------|
| Cache | Trigram search cache with LRU | `pkg/skills/search_cache.go` |
| Cache | System prompt mtime tracking | `pkg/agent/context.go` |
| Cache | Token cache with TTL | `pkg/channels/feishu/token_cache.go` |
| Sharding | FNV-based mutex sharding | `pkg/memory/jsonl.go` |
| Connection | Exponential backoff | `wecom.connectLoop()` |
| Connection | WebSocket heartbeat | `wecom.heartbeatLoop()` |
| Deduplication | Ring buffer dedup | `recentMessageSet` |
| Queue | Per-channel worker queues | `channelWorker` |
