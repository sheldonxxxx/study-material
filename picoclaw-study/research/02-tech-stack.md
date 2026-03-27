# Picoclaw Tech Stack Analysis

## Overview

**Repository:** github.com/sipeed/picoclaw
**Primary Language:** Go
**Secondary Language:** TypeScript/JavaScript (frontend)
**License:** MIT

## Language Versions

| Language | Version | Source |
|----------|---------|--------|
| Go | 1.25.8 | `go.mod: go 1.25.8` |
| Node.js | 24 (Docker full image) | `docker/Dockerfile.full` |
| TypeScript | 5.9.3 | `web/frontend/package.json` |
| React | 19.2.0 | `web/frontend/package.json` |
| Alpine Linux | 3.23 | `docker/Dockerfile` |

## Architecture

Picoclaw follows a **multi-binary architecture** with three distinct entry points:

1. **`picoclaw`** (main agent) - The core CLI and runtime
2. **`picoclaw-launcher`** (web console) - Embedded web UI server with Go backend + React frontend
3. **`picoclaw-launcher-tui`** - Terminal UI version

### Directory Structure

```
picoclaw/
├── cmd/              # CLI entry points
│   ├── picoclaw/     # Main agent
│   └── picoclaw-launcher-tui/
├── pkg/              # Core Go packages (21 subdirs)
├── web/
│   ├── backend/      # Go web server (embedded into launcher)
│   └── frontend/     # React web UI (built and embedded)
├── docker/           # Dockerfiles
└── .github/workflows/# CI/CD pipelines
```

## Go Dependencies

### Build Configuration

- **Build Tags:** `goolm,stdjson` (default), `whatsapp_native` (optional)
- **CGO:** Disabled by default (`CGO_ENABLED=0`), enabled for macOS web launcher
- **GoReleaser:** Used for multi-platform release builds

### Web Framework (Backend)

The web launcher uses **stdlib `net/http`** with `http.ServeMux` - no external web framework (Gin/Echo/etc).

**Middleware Stack:**
- Recovery middleware (`middleware.Recoverer`)
- Logging middleware (`middleware.Logger`)
- JSON content-type enforcement
- IP allowlist/allowlist access control

### Key Go Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `spf13/cobra` | v1.10.2 | CLI framework |
| `rivo/tview` | v0.42.0 | TUI library |
| `gdamore/tcell/v2` | v2.13.8 | Terminal cell manipulation |
| `fyne.io/systray` | v1.12.0 | System tray integration |
| `gorilla/websocket` | v1.5.3 | WebSocket support |
| `rs/zerolog` | v1.34.0 | Structured logging |
| `modernc.org/sqlite` | v1.46.1 | Pure-Go SQLite driver |
| `google/uuid` | v1.6.0 | UUID generation |
| `BurntSushi/toml` | v1.6.0 | TOML config parsing |
| `caarlos0/env/v11` | v11.4.0 | Env var parsing |
| `creack/pty` | v1.1.24 | PTY management |
| `gopkg.in/yaml.v3` | v3.0.1 | YAML config |
| `stretchr/testify` | v1.11.1 | Testing assertions |

### AI/Model Provider SDKs

| Provider | Package | Version |
|----------|---------|---------|
| Anthropic | `anthropics/anthropic-sdk-go` | v1.26.0 |
| OpenAI | `openai/openai-go/v3` | v3.22.0 |
| AWS Bedrock | `aws/aws-sdk-go-v2` + `aws/aws-sdk-go-v2/service/bedrockruntime` | v1.41.4 / v1.50.2 |

### Messaging Platform Libraries

| Platform | Package | Version |
|----------|---------|---------|
| WhatsApp (native) | `go.mau.fi/whatsmeow` | v0.0.0-20260219150138-7ae702b1eed4 |
| Matrix | `maunium.net/go/mautrix` | v0.26.4 |
| IRC | `ergochat/irc-go` | v0.6.0 |
| IRC readline | `ergochat/readline` | v0.1.3 |
| Discord | `bwmarrin/discordgo` | v0.29.0 |
| Telegram | `mymmrac/telego` | v1.7.0 |
| WeChat (Feishu) | `larksuite/oapi-sdk-go/v3` | v3.5.3 |
| WeChat (Work/WeCom) | `tencent-connect/botgo` | v0.2.1 |
| DingTalk | `open-dingtalk/dingtalk-stream-sdk-go` | v0.9.1 |
| Slack | `slack-go/slack` | v0.17.3 |

### Other Notable Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `modelcontextprotocol/go-sdk` | v1.4.1 | MCP protocol support |
| `github/copilot-sdk/go` | v0.1.32 | GitHub Copilot integration |
| `adhocore/gronx` | v1.19.6 | Cron parsing |
| `gomarkdown/markdown` | v0.0.0-20260217112301-37c66b85d6ab | Markdown rendering |
| `mdp/qrterminal/v3` | v3.2.1 | QR code terminal display |
| `rsc.io/qr` | v0.2.0 | QR code generation |
| `h2non/filetype` | v1.1.3 | File type detection |

## Frontend Tech Stack

### Core Framework

| Technology | Version | Purpose |
|------------|---------|---------|
| React | 19.2.0 | UI framework |
| Vite | 7.3.1 | Build tool & dev server |
| TypeScript | 5.9.3 | Type safety |
| Tailwind CSS | 4.2.2 | CSS framework |
| `@tailwindcss/vite` | 4.2.2 | Vite plugin for Tailwind |

### Routing & Data Fetching

| Technology | Version | Purpose |
|------------|---------|---------|
| TanStack React Router | 1.167.0 | SPA routing |
| TanStack React Query | 5.90.21 | Server state management |
| TanStack Router Plugin | 1.164.0 | Code generation for routes |

### UI Components

| Technology | Version | Purpose |
|------------|---------|---------|
| Radix UI | 1.4.3 | Headless UI primitives |
| shadcn | 4.1.0 | UI component library |
| @tabler/icons-react | 3.40.0 | Icon library |
| class-variance-authority | 0.7.1 | Component variants |
| sonner | 2.0.7 | Toast notifications |
| tw-animate-css | 1.4.0 | Animation utilities |

### Internationalization

| Technology | Version | Purpose |
|------------|---------|---------|
| i18next | 25.8.14 | Translation framework |
| react-i18next | 16.5.8 | React bindings |
| i18next-browser-languagedetector | 8.2.1 | Auto language detection |

### Content Rendering

| Technology | Version | Purpose |
|------------|---------|---------|
| react-markdown | 10.1.0 | Markdown rendering |
| remark-gfm | 4.0.1 | GitHub Flavored Markdown |
| rehype-raw | 7.0.0 | Raw HTML in Markdown |
| rehype-sanitize | 6.0.0 | HTML sanitization |

### Build & Linting

| Tool | Version | Purpose |
|------|---------|---------|
| ESLint | 9.39.4 | Linting |
| Prettier | 3.8.1 | Code formatting |
| @trivago/prettier-plugin-sort-imports | 6.0.2 | Import sorting |
| prettier-plugin-tailwindcss | 0.7.2 | Tailwind class ordering |

## Build System

### Makefile Targets

| Target | Purpose |
|--------|---------|
| `make build` | Build main binary for current platform |
| `make build-launcher` | Build web launcher (with embedded frontend) |
| `make build-launcher-tui` | Build TUI launcher |
| `make build-all` | Cross-platform builds (8 platforms) |
| `make build-whatsapp-native` | Build with WhatsApp native support |
| `make docker-build` | Build Alpine-based Docker image |
| `make docker-build-full` | Build full-featured Docker (Node.js 24) |
| `make test` | Run Go and frontend tests |
| `make lint` | Run golangci-lint |

### Multi-Platform Build Support

| OS | Architectures |
|----|---------------|
| Linux | amd64, arm64, armv7, loong64, riscv64, mipsle |
| macOS | amd64, arm64 |
| Windows | amd64 |
| FreeBSD | amd64 |
| NetBSD | amd64, arm64 |

### Build Special Features

1. **MIPS e_flags patching** - ELF metadata fix for NaN2008 kernels (Ingenic X2600)
2. **Loong64 pty support** - Patch for missing `ztypes_loong64.go`
3. **CGO differences** - Enabled for macOS launcher, disabled elsewhere

## Containerization

### Docker Images

| Image | Base | Purpose |
|-------|------|---------|
| `picoclaw` (goreleaser) | Alpine 3.23 | Minimal runtime |
| `picoclaw` (full) | Node.js 24-alpine3.23 | Full MCP support with uv |
| `picoclaw-agent` | Alpine 3.23 | Agent mode |
| `picoclaw-gateway` | Alpine 3.23 | Gateway mode |

### Docker Compose

- `docker-compose.yml` - Minimal Alpine-based setup
- `docker-compose.full.yml` - Full-featured with Node.js 24

## CI/CD

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `build.yml` | Push to main | Multi-platform Go build |
| `release.yml` | Git tag | GoReleaser release |
| `nightly.yml` | Schedule | Nightly builds |
| `docker-build.yml` | Push to main | Docker image builds |
| `pr.yml` | Pull request | PR validation |
| `upload-tos.yml` | Manual | Terms of service upload |

### GoReleaser Configuration

- Three binaries: `picoclaw`, `picoclaw-launcher`, `picoclaw-launcher-tui`
- Docker V2 manifest for multi-arch images
- macOS code signing and notarization support
- RPM/DEB package generation

## Environment & Configuration

### Configuration Format

- **Primary:** TOML (via `BurntSushi/toml`)
- **Fallback:** YAML (via `gopkg.in/yaml.v3`)
- **Environment:** Env vars via `caarlos0/env`

### Key Environment Variables

Configured via `github.com/sipeed/picoclaw/pkg/config`:
- `PICOCLAW_HOME` - Config/workspace directory (default: `~/.picoclaw`)
- `WORKSPACE_DIR` - Workspace root
- `INSTALL_PREFIX` - Installation prefix

## Testing

### Go Testing

- `stretchr/testify` for assertions
- Table-driven tests common
- API endpoint tests in `*_test.go` files

### Frontend Testing

- ESLint for static analysis
- Prettier for formatting checks
- `pnpm check` for combined linting

## Summary

Picoclaw is a **Go-centric multi-platform CLI tool** with:
- **Core runtime** in Go 1.25.8
- **Optional web UI** with React 19 frontend embedded in Go binary
- **Optional TUI** with rivo/tview
- **Rich integrations** for AI providers and messaging platforms
- **Container-first** deployment with Docker
- **Professional DevOps** with GoReleaser, multi-arch builds, and CI/CD

The project prioritizes **portability** (8+ platforms) and **embeddability** (single-binary deployment) over using external web frameworks, opting for stdlib `net/http` where possible.
