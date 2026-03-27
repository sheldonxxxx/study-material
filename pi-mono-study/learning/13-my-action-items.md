# My Action Items for Building an AI Agent CLI

Based on pi-mono study. Prioritized by impact and prerequisite order.

---

## Phase 1: Foundation (Week 1-2)

### 1.1 Initialize Monorepo Structure

**Priority:** P0 - Critical

**Actions:**
- [ ] Use npm workspaces for monorepo management (do NOT add Turborepo/Nx initially)
- [ ] Set up package structure:
  ```
  packages/
    ai/           # LLM provider abstraction
    agent-core/   # Agent runtime
    cli/          # Interactive CLI
  ```
- [ ] Create `tsconfig.base.json` with strict mode:
  ```json
  {
    "compilerOptions": {
      "target": "ES2022",
      "module": "Node16",
      "strict": true,
      "declaration": true,
      "declarationMap": true,
      "sourceMap": true
    }
  }
  ```
- [ ] Add Biome for linting/formatting (not ESLint+Prettier)
- [ ] Set up Husky with pre-commit hook running `biome check`

**Why:** npm workspaces is sufficient. Biome is faster than ESLint+Prettier.

---

### 1.2 Implement Provider Abstraction (pi-ai)

**Priority:** P0 - Critical

**Actions:**
- [ ] Define core types in `packages/ai/src/types.ts`:
  ```typescript
  export type AssistantMessageEvent =
    | { type: "start"; partial: AssistantMessage }
    | { type: "text_delta"; delta: string; partial: AssistantMessage }
    | { type: "done"; message: AssistantMessage }
    | { type: "error"; error: AssistantMessage };

  export interface Model<TApi extends string> {
    id: string;
    provider: string;
    api: TApi;
    baseUrl: string;
    contextWindow: number;
  }

  export type StreamFunction<TApi extends string> = (
    model: Model<TApi>,
    messages: Message[],
    options?: StreamOptions
  ) => AsyncIterable<AssistantMessageEvent>;
  ```
- [ ] Implement EventStream class from `packages/ai/src/utils/event-stream.ts` pattern:
  ```typescript
  export class EventStream<T, R = T> implements AsyncIterable<T> {
    private queue: T[] = [];
    private waiting: ((value: IteratorResult<T>) => void)[] = [];

    push(event: T): void { /* deliver to waiting or queue */ }
    end(result?: R): void { /* resolve final result */ }
    async *[Symbol.asyncIterator](): AsyncIterator<T> { /* async iteration */ }
  }
  ```
- [ ] Create API registry (`registerApiProvider`, `getProvider`)
- [ ] Start with 2 providers (Anthropic + OpenAI), add more later

**Verification:** Run `npm run check` and `npm test` with no errors.

---

### 1.3 Implement Agent Core (pi-agent-core)

**Priority:** P0 - Critical

**Actions:**
- [ ] Define AgentEvent types (from `packages/agent/src/types.ts`):
  ```typescript
  export type AgentEvent =
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }
    | { type: "tool_execution_start"; toolCallId: string; toolName: string }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
  ```
- [ ] Implement AgentTool interface:
  ```typescript
  export interface AgentTool<TParameters = any> {
    name: string;
    description: string;
    parameters: TSchema;
    execute: (
      toolCallId: string,
      params: TParameters,
      signal?: AbortSignal,
      onUpdate?: (update: any) => void
    ) => Promise<ToolResult>;
  }
  ```
- [ ] Implement agent loop with tool execution:
  - Parallel tool execution with Promise.all
  - Sequential preflight checks, parallel execution
  - Respect AbortSignal throughout
- [ ] Add beforeToolCall/afterToolCall hooks for permission gates

**Verification:** Write tests for agent loop with mock tools.

---

### 1.4 Implement Basic CLI Tools

**Priority:** P0 - Critical

**Actions:**
- [ ] Implement 4 basic tools: read, bash, edit, write
- [ ] Tool factory pattern from `packages/coding-agent/src/core/tools/*.ts`:
  ```typescript
  export function createReadTool(cwd: string): AgentTool<ReadSchema> {
    return {
      name: "read",
      description: "Read file contents",
      parameters: ReadSchema,
      async execute(toolCallId, params, signal) {
        // Implementation
      }
    };
  }
  ```
- [ ] Add bash output truncation (rolling buffer, temp file for large output)
- [ ] Add proper error encoding (return error in result, don't throw)

**Verification:** Write unit tests for each tool with edge cases.

---

## Phase 2: User Experience (Week 3-4)

### 2.1 Session Persistence

**Priority:** P1 - High

**Actions:**
- [ ] Implement JSONL session storage from day one:
  ```
  ~/.myagent/sessions/<timestamp>_<id>.jsonl
  ```
- [ ] Entry format:
  ```json
  {"type":"session","version":1,"id":"abc123","timestamp":"..."}
  {"type":"message","id":"def456","parentId":"abc123","message":{...}}
  ```
- [ ] Track `leafId` for current position
- [ ] Implement `branch(branchFromId)` for conversation branching
- [ ] Auto-save only after first assistant response (avoid empty session files)

**From pi-mono insight:** Tree model enables branching cheaply. Start with it.

---

### 2.2 Interactive TUI

**Priority:** P1 - High

**Actions:**
- [ ] Implement basic terminal UI (can start simple, improve later)
- [ ] Add streaming display for agent responses
- [ ] Implement differential rendering (only redraw changed lines):
  ```typescript
  // Track first/last changed line
  // Only re-render affected area
  ```
- [ ] Add keyboard input handling (basic line editing)
- [ ] Show tool execution progress

**Reference:** `packages/tui/src/tui.ts` for rendering strategies

---

### 2.3 Multi-Provider Auth

**Priority:** P1 - High

**Actions:**
- [ ] Implement env var lookup for API keys:
  ```typescript
  const envMap = {
    anthropic: "ANTHROPIC_API_KEY",
    openai: "OPENAI_API_KEY",
    // ...
  };
  ```
- [ ] Add AuthStorage with file locking for OAuth tokens:
  - Store in `~/.myagent/auth.json` with 0o600 permissions
  - Use proper-lockfile for refresh coordination
  - Double-checked locking pattern for token refresh
- [ ] Add OAuth flow for at least one provider (Anthropic)

**Reference:** `packages/coding-agent/src/core/auth-storage.ts`

---

### 2.4 Context Overflow Handling

**Priority:** P1 - High

**Actions:**
- [ ] Build overflow pattern library:
  ```typescript
  const OVERFLOW_PATTERNS = [
    /prompt is too long/i,
    /exceeds the context window/i,
    /input token count.*exceeds/i,
    // ...
  ];
  ```
- [ ] Implement compaction system:
  - Token counting (estimate via character count * 4)
  - Find "cut points" for summarization
  - Summarize old messages, replace with summary entry
- [ ] Add manual `/compact` command

**Reference:** `packages/ai/src/utils/overflow.ts`, `packages/coding-agent/src/core/compaction/`

---

## Phase 3: Extensibility (Week 5-6)

### 3.1 Extension System

**Priority:** P2 - Medium

**Actions:**
- [ ] Implement extension API:
  ```typescript
  interface ExtensionAPI {
    registerTool(tool: AgentTool): void;
    registerCommand(name: string, handler: CommandHandler): void;
    on(event: string, handler: EventHandler): void;
  }
  ```
- [ ] Use jiti for TypeScript extension loading (or eval with tsx)
- [ ] Extension discovery paths:
  - `~/.myagent/extensions/`
  - `./.myagent/extensions/`
- [ ] Event hooks: session_start, tool_call, before_llm, after_llm
- [ ] Add file mutation queue to prevent race conditions

**Reference:** `packages/coding-agent/src/core/extensions/`

**Note:** Do NOT add sandboxing initially. Trust user-installed extensions.

---

### 3.2 Skills System

**Priority:** P2 - Medium

**Actions:**
- [ ] Follow Agent Skills standard (agentsskills.io)
- [ ] SKILL.md format with frontmatter:
  ```yaml
  ---
  name: my-skill
  description: What this skill does
  ---
  # My Skill
  Instructions...
  ```
- [ ] Discovery from: `~/.myagent/skills/`, `./skills/`
- [ ] Progressive disclosure: include name/description in prompt, load content on demand
- [ ] Validate skill names per spec (lowercase, no leading/trailing hyphens)

**Reference:** `packages/coding-agent/src/core/skills.ts`

---

### 3.3 Prompt Templates

**Priority:** P2 - Medium

**Actions:**
- [ ] Templates in `~/.myagent/prompts/*.md`
- [ ] Argument substitution (Bash-style):
  ```typescript
  // Order matters: positional FIRST before wildcards
  result.replace(/\$(\d+)/g, (_, n) => args[parseInt(n) - 1]);
  result.replace(/\$@/g, args.join(" "));
  result.replace(/\$ARGUMENTS/g, args.join(" "));
  ```
- [ ] Expansion via `/templatename arg1 arg2`

---

## Phase 4: Polish (Week 7+)

### 4.1 Structured Logging

**Priority:** P2 - Medium

**Actions:**
- [ ] Replace console.log with pino (or similar):
  ```typescript
  import pino from 'pino';
  const logger = pino({ level: process.env.LOG_LEVEL || 'info' });
  ```
- [ ] Add log levels: debug, info, warn, error
- [ ] Filter sensitive data (API keys, tokens)
- [ ] Add request ID tracking for debugging

**Reference:** mom package (`packages/mom/src/log.ts`) has the right pattern.

---

### 4.2 Tool Rendering Customization

**Priority:** P3 - Lower

**Actions:**
- [ ] Add custom renderCall/renderResult to tools
- [ ] Theme support for terminal colors
- [ ] Syntax highlighting for code output (use highlight.js or cli-highlight)

---

### 4.3 Binary Distribution

**Priority:** P3 - Lower

**Actions:**
- [ ] Bun for cross-platform binary compilation:
  ```bash
  bun build --compile ./dist/cli.js --outfile dist/myagent
  ```
- [ ] Platform targets: darwin-arm64, darwin-x64, linux-x64, windows-x64

---

## Non-Negotiable Practices

### Must Have from Day One

1. **Strict TypeScript** - No `any` without explicit reason
2. **Biome check in CI** - Lint + format in one tool
3. **Vitest for unit tests** - Per package test directories
4. **Event-driven architecture** - Not polling, not synchronous
5. **Contract error handling** - Errors in return types, not thrown
6. **AbortSignal everywhere** - For cancellation support
7. **Session persistence** - Tree-based, not linear

### Must Avoid Initially

1. **No Turborepo/Nx** - npm workspaces sufficient
2. **No ORM** - Start with simple file storage
3. **No microservices** - Single process, clear modules
4. **No GraphQL** - Simple REST or direct function calls
5. **No Docker for dev** - Native execution simpler
6. **No Kubernetes** - Manual deployment or simple VPS initially

---

## Tech Stack Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Runtime | Node.js >=20 | Required by project spec |
| Language | TypeScript 5.x | Strict mode, declaration maps |
| Monorepo | npm workspaces | Sufficient, no extra tooling |
| Bundler | tsx/tsgo | Fast dev execution |
| Linter | Biome | Single tool, faster than ESLint |
| Testing | Vitest | Good DX, TypeScript support |
| Auth | File locking | Appropriate for OAuth refresh tokens |
| Logging | pino | Structured, fast, async |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Provider API changes | Registry pattern isolates provider code |
| Context overflow | Build overflow detection early |
| Tool execution failures | Contract-based error encoding |
| Extension security | Trust-based model (document clearly) |
| Multi-instance conflicts | File locking for shared state |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Build time | < 30 seconds for full rebuild |
| CLI cold start | < 2 seconds |
| Session load | < 100ms for 1000-entry session |
| Type errors | 0 (strict mode) |
| Test coverage | > 80% for core packages |
| Provider switching | < 5 lines of code to change provider |

---

## Key Files to Reference

| Pattern | File |
|---------|------|
| Provider types | `packages/ai/src/types.ts` |
| Event stream | `packages/ai/src/utils/event-stream.ts` |
| Agent loop | `packages/agent/src/agent-loop.ts` |
| Session manager | `packages/coding-agent/src/core/session-manager.ts` |
| Auth storage | `packages/coding-agent/src/core/auth-storage.ts` |
| TUI rendering | `packages/tui/src/tui.ts` |
| Overflow detection | `packages/ai/src/utils/overflow.ts` |
| Tools (bash) | `packages/coding-agent/src/core/tools/bash.ts` |
| Extensions | `packages/coding-agent/src/core/extensions/` |
| Skills | `packages/coding-agent/src/core/skills.ts` |
