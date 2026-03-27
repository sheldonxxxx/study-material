# Tech Stack

## Language and Runtime

- **Go 1.25+** — The entire project is written in Go from scratch. Not a fork of NanoBot, OpenClaw, or any other project.
- **CGO_ENABLED=0** — Static binary compilation (except macOS which uses CGO_ENABLED=1 for system tray support).
- **Two build tags** control core behavior:
  - `goolm` — JSON lines (JSONL) memory store (default)
  - `stdjson` — Standard JSON encoding fallback
  - `whatsapp_native` — Native WhatsApp (whatsmeow) support; produces larger binaries

## Core Dependencies

### AI / LLM Providers

| Provider | Package | Purpose |
|----------|---------|---------|
| OpenAI | `github.com/openai/openai-go/v3` | GPT models |
| Anthropic | `github.com/anthropics/anthropic-sdk-go` | Claude models |
| Google Gemini | (custom REST) | Gemini models |
| AWS Bedrock | `github.com/aws/aws-sdk-go-v2/*` | Claude/Llama/Mistral on AWS |
| GitHub Copilot | `github.com/github/copilot-sdk/go` | OAuth device code login |
| Antigravity | `github.com/github/copilot-sdk/go` | Google Cloud AI |
| Zhipu (GLM) | `github.com/larksuite/oapi-sdk-go/v3` | Chinese LLM |
| DeepSeek, Qwen, Groq, Mistral, Cerebras, Novita | (custom REST) | Various providers |
| Ollama / vLLM / LiteLLM | (custom REST) | Local or proxy deployments |

### Messaging Channels

| Channel | Package | Protocol |
|---------|---------|----------|
| Telegram | `github.com/mymmrac/telego` | Long polling |
| Discord | `github.com/bwmarrin/discordgo` | WebSocket |
| WhatsApp | `go.mau.fi/whatsmeow` | Native (when `whatsapp_native` tag) |
| Slack | `github.com/slack-go/slack` | Socket Mode |
| Matrix | `maunium.net/go/mautrix` | Sync API |
| IRC | `github.com/ergochat/irc-go` | IRC protocol |
| DingTalk | `github.com/open-dingtalk/dingtalk-stream-sdk-go` | Stream |
| Feishu/Lark | `github.com/larksuite/oapi-sdk-go/v3` | WebSocket/SDK |
| WeCom | `github.com/tencent-connect/botgo` | Webhook |
| LINE | (custom) | Webhook |
| QQ | (custom WebSocket) | WebSocket |
| WeChat | (custom iLink API) | Native QR scan |
| MaixCam | (custom TCP) | Native protocol |
| Pico | (custom) | Native protocol |
| OneBot | (custom WebSocket) | OneBot v11 |

### CLI and UI

| Component | Package |
|-----------|---------|
| TUI framework | `github.com/rivo/tview` + `github.com/gdamore/tcell/v2` |
| System tray | `fyne.io/systray` |
| QR codes | `github.com/mdp/qrterminal/v3` + `rsc.io/qr` |
| Markdown rendering | `github.com/gomarkdown/markdown` |
| Readline | `github.com/ergochat/readline` + `github.com/creack/pty` |
| Web UI | Go backend + pnpm/Vite frontend (in `web/` directory) |

### Data and Storage

| Component | Package |
|-----------|---------|
| SQLite | `modernc.org/sqlite` (pure Go driver) |
| JSON lines memory | Built-in with `goolm` tag |
| Config | `github.com/BurntSushi/toml` + `gopkg.in/yaml.v3` |
| UUID | `github.com/google/uuid` |

### Networking and APIs

| Component | Package |
|-----------|---------|
| WebSocket | `github.com/gorilla/websocket` |
| HTTP client | Standard library + `github.com/valyala/fasthttp` for performance |
| OAuth2 | `golang.org/x/oauth2` |
| MCP (Model Context Protocol) | `github.com/modelcontextprotocol/go-sdk` |
| Cron scheduling | `github.com/adhocore/gronx` |

### Utilities

| Component | Package |
|-----------|---------|
| Logging | `github.com/rs/zerolog` |
| CLI framework | `github.com/spf13/cobra` |
| Config from env | `github.com/caarlos0/env/v11` |
| File type detection | `github.com/h2non/filetype` |
| Protocol buffers | `google.golang.org/protobuf` |

## Project Structure

```
cmd/                    # CLI entry points
  picoclaw/            # Main agent binary
  picoclaw-launcher-tui/  # Terminal UI launcher
pkg/                   # Core packages
  agent/               # Agent loop and reasoning
  auth/                # Authentication providers
  bus/                 # Event bus
  channels/            # Channel implementations
  commands/            # CLI commands
  config/              # Configuration loading
  cron/                # Scheduling
  gateway/             # HTTP/WebSocket gateway
  mcp/                 # MCP protocol
  memory/              # Memory store (JSONL)
  providers/           # LLM provider implementations
  routing/             # Model routing
  session/             # Session management
  skills/              # Skills system
  state/               # State management
  tools/               # Built-in tools
web/
  backend/             # Web UI backend (Go)
  frontend/            # Web UI frontend (pnpm/Vite)
  picoclaw-launcher.desktop  # Linux desktop entry
```

## Build System

- **Makefile** — Primary build orchestration (`make build`, `make build-launcher`, `make build-all`, `make check`, `make test`, `make lint`)
- **GoReleaser** — Multi-platform release automation (`.goreleaser.yaml`)
- **golangci-lint** — Linting with extensive rule configuration (`.golangci.yaml`)

## Notable Architectural Decisions

1. **No TypeScript/Node.js for core** — Pure Go, though the Web UI launcher embeds a pnpm/Vite frontend
2. **Pure Go SQLite** — Uses `modernc.org/sqlite` instead of `mattn/go-sqlite3` (no CGO requirement)
3. **Multiple binaries** — Three separate builds: `picoclaw` (core), `picoclaw-launcher` (Web UI), `picoclaw-launcher-tui` (terminal UI)
4. **Build tags for features** — `goolm`/`stdjson` for memory format, `whatsapp_native` for WhatsApp
5. **MIPS/Loong64/RISC-V support** — Cross-compilation to embedded and non-x86 architectures
6. **AI bootstrapped** — 95% of core code was generated by an AI Agent, validated through human-in-the-loop review
