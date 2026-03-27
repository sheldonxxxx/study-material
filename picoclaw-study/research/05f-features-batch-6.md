# PicoClaw Features Batch 6 - Deep Dive

**Date:** 2026-03-26
**Features:** Steering/Hooks System, Credential Management, Shell/Session Tools
**Repo:** `/Users/sheldon/Documents/claw/reference/picoclaw`

---

## Feature 15: Steering & Hooks System

### Overview

The Steering and Hooks system provides two distinct but complementary mechanisms for influencing a running agent:

1. **Steering** - Message injection into a running agent loop to interrupt/redirect behavior
2. **Hooks** - Event-driven interceptors for observing and modifying LLM/tool behavior

### Steering System

**Core File:** `pkg/agent/steering.go`

#### Architecture

The steering system implements a **thread-safe scoped queue** that holds user messages to be injected into an active agent loop.

```go
type steeringQueue struct {
    mu     sync.Mutex
    queues map[string][]providers.Message  // scoped by session key
    mode   SteeringMode
}
```

**Key Design Decisions:**

1. **Scoped Queues** - Steering is isolated per resolved session scope (not a global queue). This prevents cross-talk between different chat sessions or routed agents.

2. **Two Modes:**
   - `SteeringOneAtATime` (default) - dequeues only the first message per poll
   - `SteeringAll` - drains the entire queue in a single poll

3. **MaxQueueSize = 10** - Hard limit prevents memory abuse

4. **Polling Points in Agent Loop:**
   - At loop start (before first LLM call)
   - After every tool completes
   - After a direct LLM response
   - Right before turn finalization

#### Steering Flow

```
User calls Steer("change direction")
    |
    v
steeringQueue.push(message)  --thread-safe-->
    |
    v
Agent loop polls after current tool finishes
    |
    v
Remaining tools SKIPPED with "Skipped due to queued user message."
    |
    v
Steering message injected into context
    |
    v
LLM called with updated context
```

**Important:** Steering does NOT interrupt a tool currently executing. It waits for the current tool to finish, then checks the queue.

#### Key Methods

```go
// Steer - enqueue a user message
func (al *AgentLoop) Steer(msg providers.Message) error

// Continue - resume idle agent with queued messages
func (al *AgentLoop) Continue(ctx context.Context, sessionKey, channel, chatID string) (string, error)

// InterruptGraceful / InterruptHard - control turn execution
func (al *AgentLoop) InterruptGraceful(hint string) error
func (al *AgentLoop) InterruptHard() error

// InjectFollowUp - enqueue for after current turn completes
func (al *AgentLoop) InjectFollowUp(msg providers.Message) error
```

#### Clever Solution: Bus Drain

When `processMessage` starts, a background goroutine automatically redirects new inbound messages into the steering queue. This means:
- Users on any channel (Telegram, Discord, etc.) automatically get steering behavior
- Audio messages are transcribed before being steered
- Only messages for the same steering scope are redirected

### Hooks System

**Core Files:**
- `pkg/agent/hooks.go` - HookManager and interfaces
- `pkg/agent/hook_process.go` - ProcessHook implementation (JSON-RPC over stdio)

#### Hook Types

| Type | Interface | Stage | Can Modify |
|------|-----------|-------|------------|
| Observer | `EventObserver` | EventBus broadcast | No |
| LLM Interceptor | `LLMInterceptor` | `before_llm` / `after_llm` | Yes |
| Tool Interceptor | `ToolInterceptor` | `before_tool` / `after_tool` | Yes |
| Tool Approver | `ToolApprover` | `approve_tool` | No (allow/deny) |

#### Hook Execution Order

1. In-process hooks first
2. Process hooks (external) second
3. Lower `priority` first within same source
4. Name order as final tie-breaker

#### Hook Actions

```go
const (
    HookActionContinue  HookAction = "continue"
    HookActionModify    HookAction = "modify"
    HookActionDenyTool HookAction = "deny_tool"
    HookActionAbortTurn HookAction = "abort_turn"
    HookActionHardAbort HookAction = "hard_abort"
)
```

#### ProcessHook Implementation (JSON-RPC over stdio)

```go
type ProcessHook struct {
    name string
    opts ProcessHookOptions
    cmd  *exec.Cmd
    stdin io.WriteCloser
    pending map[uint64]chan processHookRPCMessage
    nextID atomic.Uint64
    // ...
}
```

The `ProcessHook` spawns an external process and communicates via JSON-RPC over stdio:

1. Handshake: `hook.hello` with capabilities
2. Notifications: `hook.event` (one-way, no response needed)
3. Requests: `hook.before_llm`, `hook.after_llm`, `hook.before_tool`, `hook.after_tool`, `hook.approve_tool`

#### Hook Timeouts

Default timeouts (configurable via `hooks.defaults`):
- Observer: 500ms
- Interceptor: 5 seconds
- Approval: 60 seconds

#### Example: Python Process Hook

The docs provide a complete Python example that implements the JSON-RPC protocol:

```python
def handle_request(method: str, params: dict) -> dict:
    if method == "hook.hello":
        return {"ok": True, "name": "python-review-gate"}
    if method == "hook.before_tool":
        return handle_before_tool(params)  # returns {"action": "continue"}
    if method == "hook.approve_tool":
        return {"approved": True}  # or {"approved": False, "reason": "..."}
```

### Technical Debt / Concerns

1. **No per-process-hook timeouts** - Timeouts are global defaults, not per-hook configurable
2. **Limited external hook capability** - Process hooks cannot initiate RPCs back to PicoClaw (only respond)
3. **Sequential tool execution required** - Steering requires tools to run sequentially (was parallel), introducing latency
4. **Hook scope limitations** - Not suitable for suspending a turn and waiting for human approval replies

---

## Feature 16: Credential Management

### Overview

The credential system (`pkg/credential/`) provides secure resolution and storage of API credentials with encryption support. It centralizes how raw credential strings (plaintext, file references, or encrypted) are resolved into actual values.

### Credential Formats Supported

| Format | Example | Resolution |
|--------|---------|------------|
| Plaintext | `sk-abc123` | Returned as-is |
| File ref | `file://filename.key` | Content read from `configDir/filename.key` |
| Encrypted | `enc://<base64>` | AES-256-GCM decrypt via passphrase + SSH key |
| Empty | `""` | Returned as-is (for OAuth etc.) |

### Encryption Architecture

**File:** `pkg/credential/credential.go`

Encryption uses **AES-256-GCM** with **HKDF-SHA256 key derivation**:

```
Key Derivation:
HKDF-SHA256(
    ikm=HMAC-SHA256(SHA256(sshKeyBytes), passphrase),
    salt,
    info="picoclaw-credential-v1"
) → 32 bytes
```

**Key Components:**
- **Salt:** 16 random bytes (stored with ciphertext)
- **Nonce:** 12 bytes for GCM (stored with ciphertext)
- **Passphrase:** From `PICOCLAW_KEY_PASSPHRASE` env var (or custom provider)
- **SSH Key:** Ed25519 private key used for key derivation

#### Resolver

```go
type Resolver struct {
    configDir         string
    resolvedConfigDir string  // symlink-resolved form
}

func (r *Resolver) Resolve(raw string) (string, error)
```

Key security features:
- **Symlink traversal prevention** - Resolves symlinks before enforcing directory containment
- **Directory escape detection** - `isWithinDir()` uses `filepath.IsLocal()` for cross-platform safety
- **Path validation** - File references cannot escape the config directory

#### Passphrase Provider Pattern

```go
// Default: reads from environment
var PassphraseProvider func() string = func() string {
    return os.Getenv(PassphraseEnvVar)
}

// Can be replaced for custom sources (e.g., SecureStore)
credential.PassphraseProvider = apiHandler.passphraseStore.Get
```

### SSH Key Management

**File:** `pkg/credential/keygen.go`

```go
// DefaultSSHKeyPath returns ~/.ssh/picoclaw_ed25519.key
func DefaultSSHKeyPath() (string, error)

// GenerateSSHKey creates Ed25519 key pair with proper permissions
func GenerateSSHKey(path string) error
```

**SSH Key Path Resolution Priority:**
1. Explicit argument to `Encrypt()` / `resolveEncrypted()`
2. `PICOCLAW_SSH_KEY_PATH` env var
3. `~/.ssh/picoclaw_ed25519.key`

**Allowed SSH Key Locations:**
- Exact match with `PICOCLAW_SSH_KEY_PATH`
- Within `PICOCLAW_HOME` directory
- Within `~/.ssh/`

### SecureStore

**File:** `pkg/credential/store.go`

```go
type SecureStore struct {
    val atomic.Pointer[string]  // lock-free reads/writes
}

func (s *SecureStore) SetString(passphrase string)
func (s *SecureStore) Get() string
func (s *SecureStore) IsSet() bool
func (s *SecureStore) Clear()
```

Uses `atomic.Pointer[string]` for lock-free concurrent access. The passphrase is never written to disk.

### Encryption Flow

```go
// Encrypt plaintext → enc:// string
func Encrypt(passphrase, sshKeyPath, plaintext string) (string, error) {
    // 1. Generate random salt (16 bytes)
    // 2. Derive key: HKDF-SHA256(HMAC-SHA256(sshHash, passphrase), salt)
    // 3. Encrypt: AES-256-GCM(key, nonce, plaintext)
    // 4. Return: "enc://" + base64(salt + nonce + ciphertext)
}

// Decrypt enc:// string → plaintext
func resolveEncrypted(raw string) (string, error) {
    // 1. Extract passphrase from PassphraseProvider
    // 2. Extract salt, nonce, ciphertext from base64
    // 3. Derive key using same HKDF process
    // 4. Decrypt and verify GCM tag
}
```

### Credential Errors

```go
var ErrPassphraseRequired = errors.New("credential: enc:// passphrase required")
var ErrDecryptionFailed = errors.New("credential: enc:// decryption failed (wrong passphrase or SSH key?)")
```

### Security Considerations

1. **No key stretching** - HKDF-SHA256 is fast, suitable for embedded but less resistant to brute force
2. **SSH key in key derivation** - Makes offline attacks harder but requires secure key storage
3. **Atomic writes for key files** - Uses `os.WriteFile` with 0o600 permissions
4. **No memory zeroing** - Passphrase remains in memory after use (standard Go behavior)

---

## Feature 17: Shell/Session Tools

### Overview

Shell and session tools (`pkg/tools/shell.go`, `pkg/tools/session.go`, `pkg/tools/filesystem.go`) provide secure command execution, interactive sessions, and file operations with sandboxing.

### ExecTool

**File:** `pkg/tools/shell.go` (1137 lines)

The `ExecTool` provides shell command execution with extensive safety features.

#### Actions

| Action | Description |
|--------|-------------|
| `run` | Execute a command synchronously |
| `list` | List active sessions |
| `poll` | Check session status |
| `read` | Read session output |
| `write` | Write to session stdin |
| `kill` | Terminate a session |
| `send-keys` | Send key sequences to PTY session |

#### Security: Deny Patterns

The tool blocks dangerous commands using regex patterns:

```go
defaultDenyPatterns = []*regexp.Regexp{
    regexp.MustCompile(`\brm\s+-[rf]{1,2}\b`),           // rm -rf
    regexp.MustCompile(`\bdel\s+/[fq]\b`),               // Windows del /f/q
    regexp.MustCompile(`\b(format|mkfs|diskpart)\b\s`),  // Disk wipe
    regexp.MustCompile(`\bdd\s+if=`),                     // dd with input
    regexp.MustCompile(`>\s*/dev/(sd[a-z]|...)`),         // Block device writes
    regexp.MustCompile(`\b(shutdown|reboot|poweroff)\b`), // System control
    regexp.MustCompile(`\bsudo\b`),                        // Privilege escalation
    regexp.MustCompile(`\bcurl\b.*\|\s*(sh|bash)`),       // Pipe to shell
    regexp.MustCompile(`\bgit\s+push\b`),                  // Git push (could force)
    // ... 40+ patterns total
}
```

**Custom Allow Patterns** can exempt commands from deny checks.

#### Workspace Restriction

```go
type ExecTool struct {
    restrictToWorkspace bool  // If true, commands cannot escape workspace
    allowedPathPatterns []*regexp.Regexp  // Additional allowed paths
}
```

Path validation:
1. Detects `..` traversal attempts
2. Validates absolute paths against workspace
3. Allows safe paths (`/dev/null`, `/dev/zero`, etc.)
4. Handles URL path components (exempts `https://` paths)

#### PTY Support

For interactive commands, PTY mode provides:
- Terminal emulation with proper ANSI handling
- **Key mode detection** - Tracks CSI vs SS3 escape sequences for arrow keys
- Signal forwarding (Ctrl-C, etc.)

```go
type PtyKeyMode uint8
const (
    PtyKeyModeCSI PtyKeyMode = iota  // Standard: \x1b[A for up
    PtyKeyModeSS3                     // Alternate: \x1bOA for up
)
```

Key sequences supported:
- Arrow keys, function keys (F1-F12)
- Ctrl modifier (`ctrl-c`, `c-c`)
- Alt modifier (`alt-x`, `m-x`)
- Shift modifier (`s-up` for Shift+Up)

#### Output Handling

- **Max buffer:** 1MB (`maxOutputBufferSize`)
- **Truncation marker:** `"\n... [output truncated, exceeded 1MB]\n"`
- **Automatic truncation** when buffer full

#### Remote Channel Restriction

```go
// GHSA-pv8c-p6jf-3fpp mitigation
if !t.allowRemote {
    channel := ToolChannel(ctx)
    if channel == "" || !constants.IsInternalChannel(channel) {
        return ErrorResult("exec is restricted to internal channels")
    }
}
```

### Session Management

**File:** `pkg/tools/session.go`

```go
type ProcessSession struct {
    ID        string
    PID       int
    Command   string
    PTY       bool
    Status    string  // "running", "done", "exited"
    ExitCode  int
    // ...
}

type SessionManager struct {
    sessions map[string]*ProcessSession
    // Background cleaner goroutine: 5 min interval, 30 min expiry
}
```

**Auto-cleanup:** Sessions done for >30 minutes are automatically removed.

### Session State Flow

```
runBackground() creates session
    |
    v
Session added to SessionManager
    |
    v
Output reader goroutine populates outputBuffer
    |
    v
Agent polls via read() to get output
    |
    v
Agent sends keys via send-keys()
    |
    v
Agent calls kill() to terminate OR process exits
    |
    v
After 30 minutes, SessionManager removes session
```

### Filesystem Tool

**File:** `pkg/tools/filesystem.go`

Provides sandboxed file operations:

| Tool | Purpose |
|------|---------|
| `read_file` | Read file contents with pagination |
| `write_file` | Write content (atomic with sync) |
| `list_dir` | List directory contents |

#### File System Abstraction

Three implementations based on restriction level:

```go
type fileSystem interface {
    ReadFile(path string) ([]byte, error)
    WriteFile(path string, data []byte) error
    ReadDir(path string) ([]os.DirEntry, error)
    Open(path string) (fs.File, error)
}
```

1. **hostFs** - Unrestricted (no sandbox)
2. **sandboxFs** - Restricted to workspace via `os.Root`
3. **whitelistFs** - Sandbox + path exceptions

#### Sandbox Implementation

```go
type sandboxFs struct {
    workspace string
}

func (r *sandboxFs) execute(path string, fn func(*os.Root, string) error) error {
    root, err := os.OpenRoot(r.workspace)
    relPath, err := getSafeRelPath(r.workspace, path)
    return fn(root, relPath)
}
```

**os.Root** provides:
- Path traversal prevention
- Symbolic link resolution
- Subdirectory access control

#### Atomic Writes

```go
func (h *hostFs) WriteFile(path string, data []byte) error {
    return fileutil.WriteFileAtomic(path, data, 0o600)
}
```

Sandbox writes use explicit sync:
1. Write to `.tmp-<pid>-<nanotime>` file
2. `Sync()` to storage medium
3. `Rename()` to target path
4. `Sync()` directory for durability

#### Read File Pagination

```go
// Parameters
path    string   // Required
offset  int64    // Byte offset (default 0)
length  int64    // Max bytes (default 64KB, capped at MaxReadFileSize)

// Response header
[file: <filename> | total: <size> bytes | read: bytes <start>-<end>]
[TRUNCATED - file has more content. Call read_file again with offset=<next>]

[END OF FILE - no further content.]
```

Binary detection: First 512 bytes sniffed to detect binary content before loading into LLM context.

### Clever Solutions

1. **Key mode auto-detection** - PTY session tracks ANSI SMKX/RMKX sequences to properly interpret arrow keys
2. **Process group termination** - `prepareCommandForTermination()` uses `syscall.SysProcAttr{Setpgid: true}` for proper process tree cleanup
3. **Signal forwarding** - PTY mode forwards signals (Ctrl-C = SIGINT) to the process
4. **Output truncation marker** - Distinguishes genuine end-of-output from buffer overflow
5. **Channel context in tools** - `ToolChannel(ctx)` extracts channel from context for remote restriction

### Technical Debt / Concerns

1. **Windows PTY limitations** - `pty.Open()` not supported on Windows; fallback to non-PTY mode
2. **No per-command timeout** - Global timeout applies to all sync commands (configurable at tool creation)
3. **Session ID entropy** - Uses `uuid.New().String()[:8]` (8 hex chars = 32 bits entropy)
4. **Blocking read on PTY** - On Linux, closing ptyMaster doesn't interrupt blocking Read(); requires `cmd.Wait()` in separate goroutine

---

## Cross-Feature Integration

### Steering uses Session/Agent Loop

```go
// From steering.go
func (al *AgentLoop) Continue(ctx context.Context, sessionKey, channel, chatID string) (string, error) {
    // Dequeues steering messages and runs agent loop
    return al.continueWithSteeringMessages(ctx, agent, sessionKey, channel, chatID, steeringMsgs)
}
```

### Hooks can intercept Tool Execution

```go
// From hooks.go
func (hm *HookManager) BeforeTool(ctx context.Context, call *ToolCallHookRequest) (*ToolCallHookRequest, HookDecision) {
    // Each interceptor can modify call.Arguments or return deny action
}
```

### Credential resolution happens at config load time

API keys are resolved from config before being passed to providers:
- `file://` references resolved to actual key content
- `enc://` references decrypted using `PassphraseProvider`

---

## Summary

| Feature | Strengths | Weaknesses |
|---------|-----------|------------|
| **Steering** | Scoped queues prevent cross-talk, automatic bus drain, graceful tool skipping | Sequential-only tool execution, no human-in-loop support |
| **Hooks** | Two mounting modes (in-process + external), JSON-RPC stdio protocol, comprehensive stages | No per-hook timeouts, external hooks can't call back |
| **Credential** | AES-256-GCM encryption, SSH key key derivation, symlink protection, lock-free SecureStore | No key stretching, passphrase not zeroed from memory |
| **Shell/Session** | 40+ deny patterns, PTY with key mode detection, atomic writes, channel restrictions | Windows PTY unsupported, no per-command timeout |
| **Filesystem** | os.Root sandboxing, whitelistFs for exceptions, pagination, binary detection | No symlink target limits in whitelist |
