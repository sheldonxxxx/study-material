# Dependencies

## Go Module

**Module**: `github.com/sipeed/picoclaw`
**Go version**: `1.25.8` (from `go.mod`)
**go.mod size**: ~5KB, ~119 lines (tight dependency surface)

## Direct Dependencies (Production)

### AI / LLM

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/anthropics/anthropic-sdk-go` | v1.26.0 | Anthropic Claude API |
| `github.com/openai/openai-go/v3` | v3.22.0 | OpenAI GPT API |
| `github.com/aws/aws-sdk-go-v2` | v1.41.4 | AWS SDK core |
| `github.com/aws/aws-sdk-go-v2/config` | v1.32.12 | AWS config |
| `github.com/aws/aws-sdk-go-v2/service/bedrockruntime` | v1.50.2 | AWS Bedrock |
| `github.com/larksuite/oapi-sdk-go/v3` | v3.5.3 | Zhipu/GLM, Feishu/Lark |
| `github.com/github/copilot-sdk/go` | v0.1.32 | GitHub Copilot (OAuth) |
| `github.com/modelcontextprotocol/go-sdk` | v1.4.1 | MCP protocol |

### Messaging Channels

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/mymmrac/telego` | v1.7.0 | Telegram |
| `github.com/bwmarrin/discordgo` | v0.29.0 | Discord |
| `go.mau.fi/whatsmeow` | ( graft) | WhatsApp (native) |
| `github.com/slack-go/slack` | v0.17.3 | Slack |
| `maunium.net/go/mautrix` | v0.26.4 | Matrix |
| `github.com/ergochat/irc-go` | v0.6.0 | IRC |
| `github.com/open-dingtalk/dingtalk-stream-sdk-go` | v0.9.1 | DingTalk |
| `github.com/tencent-connect/botgo` | v0.2.1 | WeCom |
| `github.com/rivo/tview` | v0.42.0 | TUI framework |
| `github.com/gdamore/tcell/v2` | v2.13.8 | Terminal screen handling |
| `fyne.io/systray` | v1.12.0 | System tray (macOS/Windows/Linux) |
| `github.com/mdp/qrterminal/v3` | v3.2.1 | QR code generation |
| `rsc.io/qr` | v0.2.0 | QR code |
| `github.com/ergochat/readline` | v0.1.3 | Readline input |
| `github.com/creack/pty` | v1.1.24 | PTY handling |

### Data and Storage

| Package | Version | Purpose |
|---------|---------|---------|
| `modernc.org/sqlite` | v1.46.1 | Pure Go SQLite (no CGO) |
| `github.com/BurntSushi/toml` | v1.6.0 | TOML config parsing |
| `gopkg.in/yaml.v3` | (std) | YAML config |
| `github.com/google/uuid` | v1.6.0 | UUID generation |

### Networking

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/gorilla/websocket` | v1.5.3 | WebSocket |
| `golang.org/x/oauth2` | v0.36.0 | OAuth2 |
| `golang.org/x/term` | v0.41.0 | Terminal I/O |
| `golang.org/x/time` | v0.14.0 | Time utilities |

### Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `github.com/spf13/cobra` | v1.10.2 | CLI framework |
| `github.com/stretchr/testify` | v1.11.1 | Testing assertions |
| `github.com/rs/zerolog` | v1.34.0 | Structured logging |
| `github.com/caarlos0/env/v11` | v11.4.0 | Env variable config |
| `github.com/adhocore/gronx` | v1.19.6 | Cron scheduling |
| `github.com/h2non/filetype` | v1.1.3 | File type detection |
| `github.com/gomarkdown/markdown` | v0.0.0-20260217112301-37c66b85d6ab | Markdown rendering |
| `google.golang.org/protobuf` | v1.36.11 | Protocol buffers |
| `golang.org/x/crypto` | v0.49.0 | Cryptography |
| `golang.org/x/net` | v0.52.0 | Networking |
| `golang.org/x/sys` | v0.42.0 | System calls |

## Indirect Dependencies (Selected Notable)

| Package | Version | Purpose |
|---------|---------|---------|
| `filippo.io/edwards25519` | v1.2.0 | Ed25519 signatures |
| `github.com/valyala/fasthttp` | v1.69.0 | Fast HTTP client |
| `github.com/klauspost/compress` | v1.18.4 | Compression |
| `github.com/tidwall/gjson` | v1.18.0 | JSON parsing |
| `github.com/bytedance/sonic` | v1.15.0 | JSON serialization |
| `github.com/davecgh/go-spew` | v1.1.1 | Debug printing |
| `github.com/stretchr/testify` | v1.11.1 | Testing |

## Dependency Management

| Command | Purpose |
|---------|---------|
| `make deps` | Download and verify modules |
| `make update-deps` | Update all dependencies (`go get -u ./...` + `go mod tidy`) |
| `go mod tidy` | Clean up go.mod and go.sum |

## Security Scanning

- **PR workflow**: `golang/govulncheck-action` scans `./...` for vulnerabilities
- Go vulmcheck is run as part of the `vuln_check` job in `pr.yml`

## Notable Dependency Choices

1. **Pure Go SQLite** (`modernc.org/sqlite`) instead of `mattn/go-sqlite3` — avoids CGO dependency, enables static builds for all platforms including MIPS and RISC-V
2. **whatsmeow graft** — Uses a grafted version of `go.mau.fi/whatsmeow` for WhatsApp native support (produces larger binaries, gated behind `whatsapp_native` build tag)
3. **Multiple QR packages** — Both `rsc.io/qr` and `github.com/mdp/qrterminal/v3` for different QR code use cases
4. **Two TUI libraries** — `rivo/tview` for UI layout + `gdamore/tcell` for terminal cell manipulation
5. **fasthttp** — Used for performance-sensitive HTTP operations
6. **sonic** — ByteDance's JSON library for fast serialization (used alongside standard `encoding/json`)
