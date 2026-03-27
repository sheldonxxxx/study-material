# PicoClaw Feature Deep Dive - Batch 3

**Features covered:** Smart Model Routing, Web UI Launcher, TUI Launcher
**Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/picoclaw`

---

## Feature 1: Smart Model Routing (Cost Optimization)

### Overview

Smart Model Routing is a rule-based cost optimization system that automatically routes simple queries to a lightweight/cheap model and complex tasks to a premium model. It operates entirely offline with no API calls - scoring is based purely on structural message features.

### Architecture

```
pkg/routing/
  router.go         - Router struct, SelectModel(), config validation
  classifier.go     - RuleClassifier: weighted scoring algorithm
  features.go       - ExtractFeatures(): structural signal extraction
  route.go         - RouteResolver: agent/message routing (separate system)
  session_key.go    - Session key construction for DM scoping
  agent_id.go      - Agent ID normalization
```

### Configuration

Defined in `pkg/config/config.go` (lines 277-287):

```go
type RoutingConfig struct {
    Enabled    bool    `json:"enabled"`
    LightModel string  `json:"light_model"` // model_name from model_list
    Threshold  float64 `json:"threshold"`   // [0,1]; score >= threshold → primary
}
```

Applied at agent creation time in `pkg/agent/instance.go` (lines 172-186):

```go
if rc := defaults.Routing; rc != nil && rc.Enabled && rc.LightModel != "" {
    resolved := resolveModelCandidates(cfg, defaults.Provider, rc.LightModel, nil)
    if len(resolved) > 0 {
        router = routing.New(routing.RouterConfig{
            LightModel: rc.LightModel,
            Threshold:  rc.Threshold,
        })
        lightCandidates = resolved
    }
}
```

Key design: Light model candidates are **pre-resolved at agent instantiation** to avoid per-message model_list lookups.

### Feature Extraction (`features.go`)

All features are **language-agnostic** - no keyword or pattern matching against natural language content.

```go
type Features struct {
    TokenEstimate     int     // CJK=1 token each, non-CJK=0.25 tokens each
    CodeBlockCount    int     // fenced ``` pairs / 2
    RecentToolCalls   int     // tool_call messages in last 6 history entries
    ConversationDepth int     // total history length
    HasAttachments    bool    // data URIs or media extensions
}
```

**Token estimation** (lines 53-70): Handles CJK character sets correctly to avoid 3x underestimation:

```go
func estimateTokens(msg string) int {
    total := utf8.RuneCountInString(msg)
    cjk := 0
    for _, r := range msg {
        if r >= 0x2E80 && r <= 0x9FFF || r >= 0xF900 && r <= 0xFAFF || r >= 0xAC00 && r <= 0xD7AF {
            cjk++
        }
    }
    return cjk + (total-cjk)/4
}
```

**Code block counting** (lines 72-79): Complete fenced blocks only. Odd fence count = 0 blocks (may be inline code or typo).

**Attachment detection** (lines 99-127): Checks for `data:image/`, `data:audio/`, `data:video/` URIs and common media extensions (`.jpg`, `.mp3`, `.mp4`, etc.). Conservative - false negatives fall back to primary model anyway.

### Scoring Algorithm (`classifier.go`)

The `RuleClassifier` uses a weighted sum with a hard cap at 1.0:

```go
// Individual weights (multiple signals can fire simultaneously):
// token > 200:              0.35  — very long prompts
// token 50-200:             0.15  — medium length
// code block present:       0.40  — coding/technical task
// tool calls > 3 (recent):  0.25  — active agentic workflow
// tool calls 1-3 (recent): 0.10  — some tool activity
// conversation depth > 10:  0.10  — accumulated complexity
// attachments present:      1.00  — hard gate; always heavy
```

**Decision boundary:** Default threshold 0.35. Some examples:
- Pure greetings/trivial Q&A: 0.00 → light
- Medium prose (50-200 tokens): 0.15 → light
- Message with code block: 0.40 → heavy
- Long message (>200 tokens): 0.35 → heavy
- Any message with image/audio: 1.00 → heavy

### Integration Point

Called in `pkg/agent/loop.go` (line 2756) during message processing:

```go
_, usedLight, score := agent.Router.SelectModel(userMsg, history, agent.Model)
if !usedLight {
    logger.DebugCF("agent", "Model routing: primary model selected", ...)
    return agent.Candidates, resolvedCandidateModel(agent.Candidates, agent.Model)
}
logger.InfoCF("agent", "Model routing: light model selected", ...)
return agent.LightCandidates, resolvedCandidateModel(agent.LightCandidates, agent.Router.LightModel())
```

### Routing System (Agent/Message Routing)

Separately, `pkg/routing/route.go` handles **agent and session routing** (not model selection). It implements a 7-level priority cascade:

1. Peer binding (most specific)
2. Parent peer binding
3. Guild binding
4. Team binding
5. Account binding
6. Channel wildcard binding
7. Default agent

This determines which agent instance handles a message, not which model is used.

### Technical Debt / Shortcuts

1. **No interface for alternative classifiers**: While `Classifier` is an interface allowing future ML-based implementations, only `RuleClassifier` exists.

2. **Token estimation assumes uniform distribution**: The CJK/non-CJK split is a rough heuristic - doesn't account for Korean Hangul syllable boundaries or mixed scripts.

3. **Hard-coded lookback window of 6**: Comment notes this "covers roughly one full tool-use round-trip" but this is an approximation.

### Tests

`pkg/routing/router_test.go` covers:
- Token estimation for ASCII, CJK, and mixed content
- Code block counting (including odd fence edge case)
- Recent tool call counting with history truncation
- Conversation depth tracking
- Attachment detection

---

## Feature 2: Web UI Launcher

### Overview

The Web UI Launcher is a browser-based configuration interface running on `localhost:18800` by default. It provides one-click setup for providers, models, channels, and gateway management. The frontend is a React application embedded directly into the Go binary.

### Entry Point

`web/backend/main.go` - Standard library `net/http` server:

```go
func main() {
    port := flag.String("port", "18800", "Port to listen on")
    public := flag.Bool("public", false, "Listen on all interfaces (0.0.0.0)")
    noBrowser := flag.Bool("no-browser", false, "Don't auto-open browser")
    // ...
}
```

Key behaviors:
- Binds to `127.0.0.1:{port}` by default (not 0.0.0.0) for security
- Auto-starts the gateway subprocess after backend begins listening
- Runs system tray when not in console mode
- Logs to file via `logger.EnableFileLogging()` when not in console mode

### Frontend Embedding

`web/backend/embed.go` uses Go 1.16+ `//go:embed`:

```go
//go:embed all:dist
var frontendFS embed.FS

func registerEmbedRoutes(mux *http.ServeMux) {
    subFS, err := fs.Sub(frontendFS, "dist")
    // Serves static assets with SPA fallback to index.html
    mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // API paths return 404, not SPA fallback
        if strings.HasPrefix(r.URL.Path, "/api/") {
            http.NotFound(w, r)
            return
        }
        // Unknown asset paths return 404, not SPA
        // Everything else falls back to index.html for React Router
    }))
}
```

Build requirement: `pnpm build:backend` must be run before building the Go backend to populate the `dist/` directory.

### API Routes

All in `web/backend/api/`:

| File | Routes | Purpose |
|------|--------|---------|
| `models.go` | `GET/POST /api/models`, `PUT/DELETE /api/models/{index}`, `POST /api/models/default` | Model list CRUD, default selection |
| `channels.go` | `GET /api/channels/catalog` | List supported channels |
| `gateway.go` | `GET /api/gateway/status`, `GET /api/gateway/logs`, `POST /api/gateway/{start,stop,restart,logs/clear}` | Gateway lifecycle, log streaming |
| `config.go` | Various | Config file operations |
| `oauth.go` | OAuth flow | Provider OAuth authentication |

### Model API (`models.go`)

The model management API handles the `model_list` array in config.json:

```go
type modelResponse struct {
    Index      int    `json:"index"`
    ModelName  string `json:"model_name"`
    Model      string `json:"model"`
    APIBase    string `json:"api_base,omitempty"`
    APIKey     string `json:"api_key"`  // Masked in responses
    // ... advanced fields
    Configured bool   `json:"configured"`  // Has credentials?
    IsDefault  bool   `json:"is_default"`
    IsVirtual  bool   `json:"is_virtual"`
}
```

Key design decisions:
- API keys are **masked** before returning (`maskAPIKey()` shows first 3 + last 4 chars)
- On update, if `api_key` is omitted (empty string), the **existing key is preserved**
- On update, if `extra_body` is omitted (`nil`), the **existing value is preserved**; if sent as empty object `{}`, it is **cleared**
- Virtual models cannot be set as default (preventing routing misconfiguration)

### Gateway Management (`gateway.go`)

The backend **spawns and manages the picoclaw gateway as a subprocess**:

```go
var gateway = struct {
    mu               sync.Mutex
    cmd              *exec.Cmd
    owned            bool    // true if we started the process
    bootDefaultModel string  // model when gateway was started
    runtimeStatus    string  // "stopped", "starting", "running", "restarting", "error"
    logs             *LogBuffer
}{}
```

Key behaviors:
- **Attaches to existing gateway** if one is already running (via health endpoint PID check)
- Forks `picoclaw gateway -E` subprocess with same config path via environment variable
- Captures stdout/stderr via pipe scanning into a 200-line circular log buffer
- Health probing: polls `/health` endpoint for 15 seconds after start to confirm ready
- Graceful shutdown via SIGTERM (SIGKILL on Windows)
- Restart handles "existing gateway didn't stop in time" with force kill

```go
func (h *Handler) startGatewayLocked(initialStatus string, existingPid int) (int, error) {
    execPath := utils.FindPicoclawBinary()
    cmd = exec.Command(execPath, "gateway", "-E")
    cmd.Env = os.Environ()
    if h.configPath != "" {
        cmd.Env = append(cmd.Env, config.EnvConfig+"="+h.configPath)
    }
    // ...
}
```

### System Tray

`web/backend/systray.go` provides system tray integration:
- Shows/hides the web UI window
- Displays gateway status indicator
- Quit option
- Platform-specific implementations: Windows (full), macOS (stub without icon on no-CGO builds)

### Security

IP allowlist middleware (`web/backend/middleware/ip_allowlist.go`):
- Validates `AllowedCIDRs` from launcher config
- Applied to all routes via middleware stack
- Errors fatall if CIDR configuration is invalid

### Configuration

Two config files:
1. `~/.picoclaw/config.json` - Main PicoClaw config (model_list, agents, channels)
2. `~/.picoclaw/launcher.json` or passed as argument - Launcher-specific settings (port, public, allowed CIDRs)

---

## Feature 3: TUI Launcher

### Overview

The TUI Launcher is a full-featured terminal UI for headless environments, servers, and SSH access. It provides the same core functionality as the Web UI but in a terminal-native interface.

### Entry Point

`cmd/picoclaw-launcher-tui/main.go`:

```go
func main() {
    configPath := tuicfg.DefaultConfigPath()  // ~/.picoclaw/tui.toml
    if len(os.Args) > 1 {
        configPath = os.Args[1]
    }

    // If config doesn't exist, run onboard
    configDir := filepath.Dir(configPath)
    if _, err := os.Stat(configDir); os.IsNotExist(err) {
        cmd := exec.Command("picoclaw", "onboard")
        cmd.Stdin = os.Stdin
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stderr
        _ = cmd.Run()
    }

    cfg, err := tuicfg.Load(configPath)
    app := ui.New(cfg, configPath)
    app.OnModelSelected = func(scheme tuicfg.Scheme, user tuicfg.User, modelID string) {
        _ = tuicfg.SyncSelectedModelToMainConfig(scheme, user, modelID)
    }
    app.Run()
}
```

### TUI Configuration

`cmd/picoclaw-launcher-tui/config/config.go` - Own config file at `~/.picoclaw/tui.toml`:

```go
type TUIConfig struct {
    Version  string   `toml:"version"`
    Model    Model    `toml:"model"`
    Provider Provider `toml:"provider"`
}

type Model struct {
    Type string `toml:"type"` // "provider" (default) | "manual"
}

type Provider struct {
    Schemes []Scheme        `toml:"schemes"`  // API endpoint definitions
    Users   []User          `toml:"users"`    // Credentials per scheme
    Current ProviderCurrent `toml:"current"`  // Selected scheme/user/model
}

type Scheme struct {
    Name    string `toml:"name"`
    BaseURL string `toml:"baseURL"`
    Type    string `toml:"type"` // "openai-compatible" (default) | "anthropic"
}

type User struct {
    Name   string `toml:"name"`
    Scheme string `toml:"scheme"`
    Type   string `toml:"type"` // "key" (default) | "OAuth"
    Key    string `toml:"key"`
}
```

**Atomic file writes** via `fileutil.WriteFileAtomic()` - safe for flash/SD storage.

### UI Framework

Uses `github.com/rivo/tview` and `github.com/gdamore/tcell/v2`:

```go
// Cyberpunk Theme
tview.Styles.PrimitiveBackgroundColor = tcell.NewHexColor(0x050510)  // Deep Void
tview.Styles.BorderColor = tcell.NewHexColor(0x00f0ff)                // Neon Cyan
tview.Styles.GraphicsColor = tcell.NewHexColor(0xff00ff)              // Neon Magenta
tview.Styles.PrimaryTextColor = tcell.NewHexColor(0xe0e0e0)           // Off-white
tview.Styles.SecondaryTextColor = tcell.NewHexColor(0x00f0ff)         // Neon Cyan
tview.Styles.TertiaryTextColor = tcell.NewHexColor(0x39ff14)         // Neon Lime
```

### Page Structure

All in `cmd/picoclaw-launcher-tui/ui/`:

| File | Page | Purpose |
|------|------|---------|
| `app.go` | Root App | Navigation stack, page management, modal system |
| `home.go` | Home | Main menu with model/channel/gateway/chat/quit |
| `schemes.go` | Schemes | Manage API endpoint schemes |
| `users.go` | Users | Manage credentials per scheme |
| `models.go` | Models | Fetch and select from provider's model list |
| `channels.go` | Channels | View/edit channel configs from main config.json |
| `gateway.go` | Gateway | Start/stop/status of gateway daemon |

### Navigation Model

```go
type App struct {
    tapp           *tview.Application
    pages          *tview.Pages      // Stack-based page management
    pageStack      []string
    pageRefreshFns map[string]func() // Refresh functions per page
    modalOpen      map[string]bool
}

// ESC key pops the page stack
a.tapp.SetInputCapture(func(event *tcell.EventKey) *tcell.EventKey {
    if event.Key() == tcell.KeyEscape {
        if len(a.modalOpen) > 0 {
            return event  // Close modal first
        }
        return a.goBack()  // Pop page stack
    }
    return event
})
```

### Model Fetching

`ui/models.go` - Fetches model list from provider's `/models` endpoint:

```go
func fetchModels(baseURL, apiKey string) ([]modelEntry, error) {
    url := strings.TrimRight(baseURL, "/") + "/models"
    req, _ := http.NewRequest(http.MethodGet, url, nil)
    req.Header.Set("Authorization", "Bearer "+apiKey)
    resp, err := client.Do(req)
    // Returns parsed JSON with .data array OR flat array
}
```

Handles two response formats:
1. OpenAI format: `{"data": [...]}`
2. Flat array: `[...]`

Concurrent model cache refresh across all users:

```go
func (a *App) refreshModelCache(onDone func()) {
    go func() {
        a.refreshMu.Lock()
        defer a.refreshMu.Unlock()
        // Fetches models for ALL users concurrently
        // Updates modelCache map
        // Calls onDone via QueueUpdateDraw when complete
    }()
}
```

### Model Selection Sync

When a model is selected in the TUI, it syncs to the main config:

```go
func SyncSelectedModelToMainConfig(scheme Scheme, user User, modelID string) error {
    // Adds/replaces "tui-prefer" model entry in ~/.picoclaw/config.json
    // Sets it as the default model
    tuiModel := map[string]any{
        "model_name": "tui-prefer",
        "model":      modelID,
        "api_key":    user.Key,
        "api_base":   scheme.BaseURL,
    }
    // ... merges into model_list, updates agents.defaults.model
}
```

### Gateway Management (TUI)

`ui/gateway.go` - Separate from Web UI's gateway management:

```go
func getGatewayStatus() gatewayStatus {
    pidPath := getPidPath()  // ~/.picoclaw/gateway.pid
    data, _ := os.ReadFile(pidPath)
    pid, _ := strconv.Atoi(strings.TrimSpace(string(data)))
    if !isProcessRunning(pid) {
        os.Remove(pidPath)  // Clean up stale PID file
        return gatewayStatus{running: false}
    }
    return gatewayStatus{running: true, pid: pid}
}

func isProcessRunning(pid int) bool {
    if runtime.GOOS == "windows" {
        return strings.Contains(output, fmt.Sprintf(" %d ", pid))
    }
    if runtime.GOOS == "darwin" {
        return strings.Contains(string(output), fmt.Sprintf(" %d ", pid))
    }
    // Linux: check /proc/{pid}
}
```

**Note:** The TUI gateway management is simpler than the Web UI version - it uses PID files and `ps`/`tasklist` rather than health endpoint probing.

### Home Page Actions

```go
func (a *App) newHomePage() tview.Primitive {
    list.AddItem("MODEL: "+a.cfg.CurrentModelLabel(), ..., 'm', func() {
        a.navigateTo("schemes", a.newSchemesPage())
    })
    list.AddItem("CHANNELS: Configure...", ..., 'n', func() {
        a.navigateTo("channels", a.newChannelsPage())
    })
    list.AddItem("GATEWAY MANAGEMENT", ..., 'g', func() {
        a.navigateTo("gateway", a.newGatewayPage())
    })
    list.AddItem("CHAT: Start AI agent chat", ..., 'c', func() {
        a.tapp.Suspend(func() {
            cmd := exec.Command("picoclaw", "agent")
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
            _ = cmd.Run()
        })
    })
    list.AddItem("QUIT SYSTEM", ..., 'q', func() { a.tapp.Stop() })
}
```

The "CHAT" option **suspends the TUI** and runs `picoclaw agent` directly, then returns when the agent exits.

---

## Cross-Feature Observations

### Config Separation

| Config File | Purpose | Format |
|-------------|---------|--------|
| `~/.picoclaw/config.json` | Main PicoClaw config | JSON |
| `~/.picoclaw/tui.toml` | TUI-specific state | TOML |
| `~/.picoclaw/launcher.json` | Web launcher settings | JSON |

The TUI and Web launcher maintain separate state files, but both sync selections to the main `config.json`.

### Model Selection Flow

1. **TUI**: User selects scheme → user → model → `SyncSelectedModelToMainConfig()` writes `tui-prefer` to `config.json`
2. **Web UI**: User selects model via API → writes to `config.json` directly
3. **Agent startup**: Reads `config.json`, resolves model candidates
4. **Message routing**: `Router.SelectModel()` decides light vs primary based on message features

### Cyberpunk Theme Consistency

Both TUI and Web UI use a cyberpunk/neon aesthetic:
- TUI: Direct tcell color assignments
- Web UI: Tailwind CSS with similar hex values (though not examined in detail)

### Gateway Subprocess Management

Both Web UI and TUI manage the gateway, but differently:

| Aspect | Web UI | TUI |
|--------|--------|-----|
| Detection | Health endpoint polling | PID file + process check |
| Startup | Via `picoclaw gateway -E` | Via `picoclaw gateway -E` |
| Config passthrough | Environment variable | Environment variable |
| Logs | 200-line buffer from stdout/stderr | Not captured in TUI |

---

## Technical Debt / Concerns

1. **Two independent gateway management implementations**: Web UI (`web/backend/api/gateway.go`) and TUI (`cmd/picoclaw-launcher-tui/ui/gateway.go`) have separate code paths. Divergence risk.

2. **TUI config uses different format (TOML) than main config (JSON)**: Forces two different config parsers and two different credential storage locations.

3. **No validation that light model exists at router creation time**: If `light_model` in config resolves to no candidates, routing is silently disabled (warning logged).

4. **Concurrent model cache refresh could cause thundering herd**: When TUI starts, it fetches models for all users simultaneously. With many users, this could overwhelm provider endpoints.

5. **Web UI backend lacks graceful shutdown handling**: The `http.Server` is created but `Shutdown()` is never called on signal - just `return` from main loop.

6. **Stale PID file cleanup in TUI**: If gateway crashes, the PID file may remain. TUI cleans it up on status check, but Web UI relies on health endpoint which doesn't have this issue.

---

## File Index

**Smart Model Routing:**
- `pkg/routing/router.go` - Router struct
- `pkg/routing/classifier.go` - RuleClassifier scoring
- `pkg/routing/features.go` - Feature extraction
- `pkg/routing/route.go` - RouteResolver (agent routing)
- `pkg/routing/session_key.go` - Session key building
- `pkg/routing/agent_id.go` - Agent ID normalization
- `pkg/agent/instance.go` - Router initialization
- `pkg/agent/loop.go` - SelectModel call site
- `pkg/agent/model_resolution.go` - Model candidate resolution
- `pkg/config/config.go` - RoutingConfig struct

**Web UI Launcher:**
- `web/backend/main.go` - HTTP server entry point
- `web/backend/embed.go` - Frontend embedding
- `web/backend/api/models.go` - Model CRUD API
- `web/backend/api/channels.go` - Channel catalog
- `web/backend/api/gateway.go` - Gateway subprocess management
- `web/backend/systray.go` - System tray
- `web/backend/i18n.go` - Internationalization

**TUI Launcher:**
- `cmd/picoclaw-launcher-tui/main.go` - Entry point
- `cmd/picoclaw-launcher-tui/config/config.go` - TUI config (tui.toml)
- `cmd/picoclaw-launcher-tui/ui/app.go` - App root/navigation
- `cmd/picoclaw-launcher-tui/ui/home.go` - Home page
- `cmd/picoclaw-launcher-tui/ui/models.go` - Model selection
- `cmd/picoclaw-launcher-tui/ui/channels.go` - Channel config viewer
- `cmd/picoclaw-launcher-tui/ui/gateway.go` - Gateway management
