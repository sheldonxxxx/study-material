# Dependencies

## Root Dependencies

```json
{
  "@mariozechner/jiti": "^2.6.5",
  "@mariozechner/pi-coding-agent": "^0.30.2",
  "get-east-asian-width": "^1.4.0"
}
```

## Dev Dependencies (Root)

```json
{
  "@biomejs/biome": "2.3.5",
  "@types/node": "^22.10.5",
  "@typescript/native-preview": "7.0.0-dev.20260120.1",
  "concurrently": "^9.2.1",
  "husky": "^9.1.7",
  "tsx": "^4.20.3",
  "typescript": "^5.9.2",
  "shx": "^0.4.0"
}
```

## Dependency Overrides

```json
"overrides": {
  "rimraf": "6.1.2",
  "fast-xml-parser": "5.3.8",
  "gaxios": {
    "rimraf": "6.1.2"
  }
}
```

## Package-to-Package Dependencies

### @mariozechner/pi-ai (0.62.0)

Unified multi-provider LLM API.

**Key runtime dependencies:**
- `@anthropic-ai/sdk`: ^0.73.0
- `@google/genai`: ^1.40.0
- `@mistralai/mistralai`: 1.14.1
- `openai`: 6.26.0
- `@aws-sdk/client-bedrock-runtime`: ^3.983.0
- `@sinclair/typebox`: ^0.34.41 (validation)
- `partial-json`: ^0.1.7 (streaming JSON parsing)

### @mariozechner/pi-agent-core (0.62.0)

Agent runtime with tool calling and state management.

**Dependencies:**
- `@mariozechner/pi-ai`: ^0.62.0
- Testing: `vitest`: ^3.2.4

### @mariozechner/pi-coding-agent (0.62.0)

Interactive coding agent CLI (`pi`).

**Dependencies:**
- `@mariozechner/pi-agent-core`: ^0.62.0
- `@mariozechner/pi-ai`: ^0.62.0
- `@mariozechner/pi-tui`: ^0.62.0
- `@silvia-odwyer/photon-node`: ^0.3.4 (image processing)
- `cli-highlight`: ^2.1.11 (syntax highlighting)
- `diff`: ^8.0.2
- `marked`: ^15.0.12 (markdown rendering)
- `yaml`: ^2.8.2
- `undici`: ^7.19.1 (HTTP client)
- `glob`: ^13.0.1

### @mariozechner/pi-mom (0.62.0)

Slack bot.

**Dependencies:**
- `@slack/socket-mode`: ^2.0.0
- `@slack/web-api`: ^7.0.0
- `@anthropic-ai/sandbox-runtime`: ^0.0.16
- `croner`: ^9.1.0 (cron parsing)

### @mariozechner/pi-tui (0.62.0)

Terminal UI library.

**Dependencies:**
- `chalk`: ^5.5.0 (terminal colors)
- `marked`: ^15.0.12 (markdown rendering)
- `mime-types`: ^3.0.1
- `get-east-asian-width`: ^1.3.0
- `koffi`: ^2.9.0 (optional native binding for VT input)

**Dev dependencies:**
- `@xterm/xterm`: ^5.5.0
- `@xterm/headless`: ^5.5.0

### @mariozechner/pi-web-ui (0.62.0)

Reusable web UI components (Lit web components).

**Dependencies:**
- `@mariozechner/pi-ai`: ^0.62.0
- `@mariozechner/pi-tui`: ^0.62.0
- `lit`: ^3.3.1 (peer dependency)
- `lucide`: ^0.544.0 (icons)
- `pdfjs-dist`: 5.4.394 (PDF rendering)
- `docx-preview`: ^0.3.7 (Word document preview)
- `xlsx`: 0.20.3 (Excel/spreadsheet support)
- `@lmstudio/sdk`: ^1.5.0
- `ollama`: ^0.6.0

### @mariozechner/pi-pods (0.62.0)

CLI for managing vLLM deployments.

**Dependencies:**
- `@mariozechner/pi-agent-core`: ^0.62.0
- `chalk`: ^5.5.0

## Workspace Structure

```json
"workspaces": [
  "packages/*",
  "packages/web-ui/example",
  "packages/coding-agent/examples/extensions/with-deps",
  "packages/coding-agent/examples/extensions/custom-provider-anthropic",
  "packages/coding-agent/examples/extensions/custom-provider-gitlab-duo",
  "packages/coding-agent/examples/extensions/custom-provider-qwen-cli"
]
```

## Engine Requirements

```json
"engines": {
  "node": ">=20.0.0"
}
```

CI uses Node.js 22.
