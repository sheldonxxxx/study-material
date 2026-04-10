# Tech Stack

## Runtime

- **Bun** (v1.3.11) — Primary JavaScript runtime and package manager
- **Node.js 24** — Used alongside Bun for certain CI/CD operations
- **Rust** — Desktop application builds (Tauri)
- **TypeScript** (5.8.2) — Primary language throughout

## Core Architecture

### Monorepo Structure

```
opencode/
  packages/
    opencode/          # Core CLI and server (Bun runtime)
    app/               # Shared web UI components (SolidJS)
    desktop/            # Tauri desktop wrapper
    desktop-electron/   # Electron desktop wrapper
    console/           # Console web app
    ui/                # Shared UI primitives
    sdk/               # SDK packages
    plugin/            # Plugin system
  sdks/
    vscode/            # VSCode extension
  github/              # GitHub Action integration
```

### Package Manager

- **Bun** as primary package manager (`packageManager: bun@1.3.11`)
- **npm** workspaces for distribution
- **Turbo** (2.8.13) for monorepo build orchestration

## Key Libraries

### AI / LLM Providers

| Provider | Package | Purpose |
|----------|---------|---------|
| Anthropic | `@ai-sdk/anthropic` | Claude models |
| OpenAI | `@ai-sdk/openai` | GPT models |
| Google | `@ai-sdk/google`, `@ai-sdk/google-vertex` | Gemini models |
| AWS Bedrock | `@ai-sdk/amazon-bedrock` | AWS-hosted models |
| Cohere | `@ai-sdk/cohere` | Command models |
| Groq | `@ai-sdk/groq` | Fast inference |
| Azure | `@ai-sdk/azure` | Azure OpenAI |
| xAI | `@ai-sdk/xai` | Grok models |
| Cerebras | `@ai-sdk/cerebras` | Cerebras models |
| TogetherAI | `@ai-sdk/togetherai` | Open model hosting |
| Vercel AI SDK | `ai` | AI SDK core |

### Web Framework

- **Hono** (4.10.7) — Lightweight web framework for API server
- **hono-openapi** — OpenAPI documentation
- **zod** (4.1.8) — Schema validation

### UI Framework

- **SolidJS** (1.9.10) — Reactive UI framework
- **SolidStart** — Full-stack SolidJS framework
- **@solidjs/router** — Client-side routing
- **@solidjs/meta** — Meta tags management

### TUI Components

- **@opentui/core** (0.1.90) — Terminal UI primitives
- **@opentui/solid** — SolidJS bindings
- **opentui-spinner** — Terminal spinners

### Database

- **Drizzle ORM** (1.0.0-beta.19-d95b7a4) — Type-safe SQL
- **drizzle-kit** — Migration tooling
- **SQLite** via Bun native APIs

### Effect Ecosystem

- **effect** (4.0.0-beta.35) — Functional programming
- **@effect/platform-node** — Node.js platform layer
- **@effect/language-service** — LSP support

### Code Intelligence

- **tree-sitter** / **web-tree-sitter** — Parsing
- **tree-sitter-bash** — Bash parsing
- **shiki** (3.20.0) — Syntax highlighting
- **marked** + **marked-shiki** — Markdown rendering
- **vscode-jsonrpc** / **vscode-languageserver-types** — LSP support

### Developer Tools

- **TypeScript** (5.8.2) — Type checking
- **@typescript/native-preview** — Native TypeScript support
- **Prettier** (3.6.2) — Code formatting
- **Husky** (9.1.7) — Git hooks

### Authentication

- **@openauthjs/openauth** — Authentication framework
- **opencode-gitlab-auth** — GitLab OAuth
- **opencode-poe-auth** — Poe.com auth

### Desktop

- **Tauri** (v2) — Desktop app framework
- **@tauri-apps/api** — Tauri JavaScript API
- **electron** — Alternative desktop runtime
- **electron-builder** — Electron packaging

### Utilities

- **remeda** (2.26.0) — Functional utilities
- **diff** (8.0.2) — Text diffing
- **fuzzysort** (3.1.0) — Fuzzy string matching
- **gray-matter** — Frontmatter parsing
- **turndown** — HTML to Markdown
- **ulid** (3.0.1) — Unique IDs
- **semver** — Semantic versioning
- **yargs** (18.0.0) — CLI argument parsing

### File Watching

- **chokidar** (4.0.3) — File system watcher
- **@parcel/watcher** (2.5.1) — Native file watching
- Platform-specific bindings for macOS, Linux, Windows

### External Integrations

- **@actions/core**, **@actions/github** — GitHub Actions SDK
- **@octokit/rest**, **@octokit/graphql** — GitHub API
- **@modelcontextprotocol/sdk** — MCP protocol
- **@agentclientprotocol/sdk** — Agent protocol
- **@aws-sdk/client-s3** — S3 storage
- **google-auth-library** — Google Cloud auth
- **gitlab-ai-provider** — GitLab AI integration
- **ai-gateway-provider** — AI gateway

### Markdown / Docs

- **marked** (17.0.1) — Markdown parsing
- **marked-shiki** (1.2.1) — Syntax highlighting in markdown
- **DOMPurify** (3.3.1) — HTML sanitization
- **@zip.js/zip.js** (2.7.62) — ZIP file handling

### Testing

- **@playwright/test** (1.51.0) — E2E testing
- **bun test** — Unit testing (Bun built-in)

## Development Environment

### Prerequisites

- **Bun 1.3+** — Required
- **Rust toolchain** — For desktop builds
- Platform-specific build tools (e.g., WebKitGTK for Linux)

### Quick Start

```bash
bun install
bun dev                    # Run OpenCode in dev mode
bun dev <directory>         # Run against specific directory
bun dev serve              # Start headless API server
bun dev web                # Start web interface
```

### Building Standalone CLI

```bash
./packages/opencode/script/build.ts --single
```

Output: `packages/opencode/dist/opencode-<platform>/bin/opencode`
