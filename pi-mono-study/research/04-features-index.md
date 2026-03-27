# pi-mono Feature Index

Prioritized feature list for the pi-mono monorepo, extracted from README.md and verified against actual package structure.

---

## Core Features (6)

### 1. Unified LLM API
**Package:** `packages/ai` (`@mariozechner/pi-ai`)
**Priority:** P0 - Foundational

Unified multi-provider LLM API supporting 18+ providers with automatic model discovery, token/cost tracking, and cross-provider handoffs.

**Key Capabilities:**
- 18+ provider support: OpenAI, Anthropic, Google, Azure, Mistral, Groq, Cerebras, xAI, OpenRouter, Vercel AI Gateway, MiniMax, Bedrock, Hugging Face, GitHub Copilot, and more
- Tool calling support across all providers with streaming partial JSON parsing
- Thinking/reasoning support with unified interface (`streamSimple`, `completeSimple`)
- Cross-provider handoffs: switch models mid-conversation while preserving context
- Context serialization: JSON-serializable conversation state
- OAuth support for Anthropic, OpenAI Codex, GitHub Copilot, Google Gemini CLI, Antigravity
- Browser-compatible with CORS proxy handling

**Key Files:**
- `packages/ai/src/index.ts` - Main exports
- `packages/ai/src/providers/` - Provider implementations
- `packages/ai/src/types.ts` - Type definitions

---

### 2. Agent Runtime
**Package:** `packages/agent` (`@mariozechner/pi-agent-core`)
**Priority:** P0 - Foundational

Stateful agent with tool execution and event streaming. Built on `pi-ai`. Powers both CLI and Slack bot.

**Key Capabilities:**
- Event-driven architecture: `agent_start`, `turn_start`, `message_update`, `tool_execution_*`, etc.
- Tool execution with `beforeToolCall` and `afterToolCall` hooks for permission gates and auditing
- Steering and follow-up message queues for interrupting/resuming agent work
- Parallel and sequential tool execution modes
- Custom message types via declaration merging
- Session ID support for provider-side prompt caching
- Low-level `agentLoop` API for custom implementations

**Key Files:**
- `packages/agent/src/index.ts` - Agent class and exports
- `packages/agent/src/agent.ts` - Core agent implementation
- `packages/agent/src/tools.ts` - Tool definitions

---

### 3. Interactive Coding Agent CLI
**Package:** `packages/coding-agent` (`@mariozechner/pi-coding-agent`)
**Priority:** P0 - Primary Product

Terminal coding harness with interactive mode, session management, and deep customization.

**Key Capabilities:**
- Interactive CLI with 4 modes: interactive, print, JSON, RPC
- Built-in tools: read, bash, edit, write, grep, find, ls
- Session management with tree-based branching and compaction
- Multi-provider authentication: OAuth, API keys, subscription logins
- Customization: prompt templates, skills (CLI tool packages), extensions, themes
- Pi packages: share extensions/skills/prompts/themes via npm or git
- File context: `@` prefix for including files in prompts
- Image support: paste or drag images into terminal

**Key Files:**
- `packages/coding-agent/src/cli/` - CLI implementation
- `packages/coding-agent/src/core/` - Core agent integration
- `packages/coding-agent/docs/` - Platform guides (Windows, Termux, tmux)

---

### 4. GPU Pod Management
**Package:** `packages/pods` (`@mariozechner/pi-pods`)
**Priority:** P1 - Infrastructure

CLI for deploying and managing vLLM on GPU pods with automatic configuration for agentic workloads.

**Key Capabilities:**
- Pod setup for DataCrunch, RunPod, Vast.ai, AWS, any Ubuntu machine
- Automatic vLLM configuration with tool calling parsers (Qwen, GLM, GPT-OSS)
- Multi-model deployment with smart GPU allocation
- OpenAI-compatible API endpoints for each model
- Interactive agent with file system tools for testing
- Memory and context configuration: `--memory`, `--context`, `--gpus`
- Tensor parallelism and data parallel support for large models
- Predefined configurations for Qwen2.5-Coder, Qwen3-Coder, GPT-OSS, GLM models

**Key Files:**
- `packages/pods/src/commands/` - CLI commands
- `packages/pods/src/agent.ts` - Standalone agent implementation

---

### 5. Terminal UI Framework
**Package:** `packages/tui` (`@mariozechner/pi-tui`)
**Priority:** P1 - Infrastructure

Minimal TUI framework with differential rendering and synchronized output for flicker-free updates.

**Key Capabilities:**
- Three-strategy differential rendering (full, clear+full, incremental)
- CSI 2026 synchronized output for atomic screen updates
- Bracketed paste mode for large pastes
- Component interface: `render()`, `handleInput()`, `invalidate()`
- Built-in components: Text, TruncatedText, Input, Editor, Markdown, Loader, SelectList, SettingsList, Box, Container, Image, Spacer
- Overlay system with anchor-based positioning and percentage sizing
- Focusable interface with IME support for CJK input
- Kitty/iTerm2 inline image support
- Autocomplete: slash commands and file path completion
- Key detection with modifier support (Ctrl, Alt, Shift)

**Key Files:**
- `packages/tui/src/tui.ts` - Core TUI implementation
- `packages/tui/src/components/` - Built-in components

---

### 6. Web UI Components
**Package:** `packages/web-ui` (`@mariozechner/pi-web-ui`)
**Priority:** P1 - User-Facing

Reusable web components for building AI chat interfaces powered by `pi-ai` and `pi-agent-core`.

**Key Capabilities:**
- ChatPanel: complete chat interface with streaming and tool execution
- AgentInterface: lower-level chat interface for custom layouts
- Artifacts panel: interactive HTML, SVG, Markdown with sandboxed execution
- Attachments: PDF, DOCX, XLSX, PPTX, images with preview and text extraction
- IndexedDB-backed storage: sessions, API keys, settings, custom providers
- JavaScript REPL tool (sandboxed browser execution)
- Document extraction tool
- Custom tool renderers
- CORS proxy handling for browser environments
- Internationalization support

**Key Files:**
- `packages/web-ui/src/components/` - Web components
- `packages/web-ui/src/stores/` - IndexedDB storage

---

## Secondary Features (5)

### 7. Slack Bot Integration
**Package:** `packages/mom` (`@mariozechner/pi-mom`)
**Priority:** P2

Slack bot (Master Of Mischief) that delegates to the coding agent with workspace memory and self-managing skills.

**Key Capabilities:**
- Slack integration via Socket Mode
- Per-channel conversation history with full context management
- Global and channel-specific memory (MEMORY.md files)
- Self-managing environment: installs tools, configures credentials autonomously
- Docker sandbox isolation (recommended)
- Skills system: create CLI tools via SKILL.md files
- Events system: scheduled reminders and periodic tasks via cron
- Thread-based details: clean main messages with verbose tool results in threads

**Key Files:**
- `packages/mom/src/main.ts` - Entry point
- `packages/mom/src/agent.ts` - Agent runner
- `packages/mom/src/slack.ts` - Slack integration

---

### 8. Extensions System
**Package:** `packages/coding-agent`
**Priority:** P2

TypeScript module system for extending the coding agent with custom tools, commands, keyboard shortcuts, and UI.

**Key Capabilities:**
- Custom tools (or replace built-in tools)
- Custom commands
- Custom keyboard shortcuts
- Event handlers (`tool_call`, etc.)
- Custom UI: editors, widgets, status lines, footers, overlays
- Example extensions: Doom game, MCP server integration, Git checkpointing

**Key Files:**
- `packages/coding-agent/docs/extensions.md`
- `packages/coding-agent/examples/extensions/`

---

### 9. Skills and Prompt Templates
**Package:** `packages/coding-agent`
**Priority:** P2

Reusable capability packages and prompt templates following the Agent Skills standard.

**Key Capabilities:**
- Skills: invoke via `/skill:name`, auto-loadable
- Prompt templates: expand via `/templatename`
- Skills stored in conventional directories: `skills/`, `.pi/skills/`, `~/.pi/agent/skills/`
- Pi packages: bundle skills, prompts, extensions, themes for npm/git distribution

**Key Files:**
- `packages/coding-agent/docs/skills.md`
- `packages/coding-agent/docs/prompt-templates.md`

---

### 10. Multi-Model Authentication
**Package:** `packages/ai`, `packages/coding-agent`
**Priority:** P2

Multiple authentication methods across providers: OAuth, API keys, subscription-based logins.

**Key Capabilities:**
- API keys via environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.)
- OAuth flows for Anthropic Pro/Max, OpenAI Codex, GitHub Copilot, Google Gemini CLI
- CLI login command (`/login`) with provider selection
- Token refresh handling for OAuth credentials
- Auth storage for programmatic usage

**Key Files:**
- `packages/ai/src/oauth/` - OAuth implementations
- `packages/coding-agent/src/cli/args.ts` - Environment variable documentation

---

### 11. Session Persistence and Branching
**Package:** `packages/coding-agent`, `packages/pods`
**Priority:** P2

Tree-based session storage with in-place branching, compaction, and continuation.

**Key Capabilities:**
- JSONL-based session storage organized by working directory
- Tree navigation: jump to any point, continue from there, switch branches
- Session forking: create new session from current branch
- Automatic and manual context compaction for long sessions
- Session forking from CLI: `--fork <path|id>`
- Session resume: `-c`, `-r` flags

**Key Files:**
- `packages/coding-agent/docs/session.md` - Session file format
- `packages/pods/src/session.ts` - Pod agent session management

---

## Summary

| Tier | Feature | Primary Package |
|------|---------|----------------|
| Core | Unified LLM API | `pi-ai` |
| Core | Agent Runtime | `pi-agent-core` |
| Core | Interactive Coding Agent CLI | `pi-coding-agent` |
| Core | GPU Pod Management | `pi-pods` |
| Core | Terminal UI Framework | `pi-tui` |
| Core | Web UI Components | `pi-web-ui` |
| Secondary | Slack Bot Integration | `pi-mom` |
| Secondary | Extensions System | `pi-coding-agent` |
| Secondary | Skills and Prompt Templates | `pi-coding-agent` |
| Secondary | Multi-Model Authentication | `pi-ai` |
| Secondary | Session Persistence and Branching | `pi-coding-agent` |

## Package-to-Feature Mapping

| Package | Features |
|---------|----------|
| `packages/ai` | Unified LLM API, Multi-Model Authentication |
| `packages/agent` | Agent Runtime |
| `packages/coding-agent` | Interactive Coding Agent, Extensions, Skills/Prompts, Session Management |
| `packages/mom` | Slack Bot Integration |
| `packages/pods` | GPU Pod Management, Session Management |
| `packages/tui` | Terminal UI Framework |
| `packages/web-ui` | Web UI Components |
