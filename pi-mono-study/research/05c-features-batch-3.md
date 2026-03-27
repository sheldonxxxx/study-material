# Features Batch 3 Deep Dive

Research Date: 2026-03-26
Features: Extensions System, Skills and Prompt Templates, Multi-Model Authentication, Session Persistence and Branching

---

## Feature 8: Extensions System

### Overview

TypeScript module system for extending the coding agent with custom tools, commands, keyboard shortcuts, and UI. Extensions can hook into lifecycle events, intercept tool calls, register LLM-callable tools, and provide custom UI components.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `packages/coding-agent/src/core/extensions/loader.ts` | Extension discovery and loading using jiti |
| `packages/coding-agent/src/core/extensions/runner.ts` | Event emission and extension lifecycle management |
| `packages/coding-agent/src/core/extensions/types.ts` | Full TypeScript type definitions (~1450 lines) |
| `packages/coding-agent/src/core/event-bus.ts` | Shared event bus for inter-extension communication |

### Key Architecture Decisions

**1. jiti-Based TypeScript Execution**

Extensions are loaded using the `jiti` library (a fork with virtualModules support for compiled Bun binaries):

```typescript
const jiti = createJiti(import.meta.url, {
  moduleCache: false,
  ...(isBunBinary
    ? { virtualModules: VIRTUAL_MODULES, tryNative: false }
    : { alias: getAliases() }),
});
const module = await jiti.import(extensionPath, { default: true });
```

This allows TypeScript extensions to run without compilation. The `VIRTUAL_MODULES` object bundles all allowed packages (`@sinclair/typebox`, `@mariozechner/pi-*`) directly into the binary.

**2. Separation of Extension API and Runtime**

The `ExtensionAPI` passed to extension factories only contains registration methods (`on`, `registerTool`, `registerCommand`, etc.). Action methods (`sendMessage`, `appendEntry`, etc.) delegate to a shared `ExtensionRuntime`:

```typescript
function createExtensionAPI(extension, runtime, cwd, eventBus): ExtensionAPI {
  const api = {
    on(event, handler) { extension.handlers.get(event)?.push(handler); },
    registerTool(tool) { extension.tools.set(tool.name, {...}); runtime.refreshTools(); },
    sendMessage(msg, opts) { runtime.sendMessage(msg, opts); },
    // ...
  };
  return api;
}
```

This design allows runtime stub methods to throw helpful errors if called during extension load before binding.

**3. Lazy Action Binding with Provider Queuing**

Provider registrations during extension load are queued and flushed when the runner binds core actions:

```typescript
// In createExtensionRuntime():
registerProvider: (name, config, extensionPath = "<unknown>") => {
  runtime.pendingProviderRegizations.push({ name, config, extensionPath });
},

// In runner.bindCore():
for (const { name, config, extensionPath } of this.runtime.pendingProviderRegistrations) {
  try {
    this.modelRegistry.registerProvider(name, config);
  } catch (err) { /* emit error */ }
}
runtime.pendingProviderRegistrations = [];
```

This ensures providers are registered in order after all extensions have loaded.

**4. Event System with Typed Handlers**

Events are strongly typed with discriminated unions. The `tool_call` event has specific typed variants for each built-in tool:

```typescript
export interface BashToolCallEvent extends ToolCallEventBase {
  toolName: "bash";
  input: BashToolInput;
}
// ... similar for read, edit, write, grep, find, ls
```

Type narrowing uses `isToolCallEventType()` guards:

```typescript
if (isToolCallEventType("bash", event)) {
  // event.input is { command: string; timeout?: number }
}
```

**5. Rich Session Event Model**

Extensions can hook into comprehensive session lifecycle:

- `session_directory` - CLI startup only, no context
- `session_start`, `session_switch`, `session_shutdown`
- `session_before_fork`, `session_fork` - branching hooks
- `session_before_compact`, `session_compact` - context compaction
- `session_before_tree`, `session_tree` - tree navigation

### Extension Loading Flow

1. `discoverAndLoadExtensions()` finds extensions in:
   - `~/.pi/agent/extensions/*.ts` (global)
   - `.pi/extensions/*.ts` (project-local)
   - Configured paths
   - Package manifests with `pi.extensions` field

2. Each extension is loaded via jiti, factory function called with `ExtensionAPI`

3. Runner emits `session_start` after core binding

4. Extensions register handlers, tools, commands, shortcuts

### Custom Tool Implementation

Tools use TypeBox for schema definition and have a rich execution interface:

```typescript
pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Does something",
  parameters: Type.Object({ action: StringEnum(["list", "add"]), text: Type.Optional(Type.String()) }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    onUpdate?.({ content: [{ type: "text", text: "Working..." }] });
    return { content: [{ type: "text", text: "Done" }], details: { result: "..." } };
  },
  renderCall(args, theme, context) { /* optional custom rendering */ },
  renderResult(result, options, theme, context) { /* optional custom rendering */ },
});
```

### Session Persistence for Extensions

Extensions can persist state via `appendEntry()` which creates `CustomEntry` records:

```typescript
pi.appendEntry("my-state", { count: 42 });

// On reload, scan entries:
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      // Reconstruct from entry.data
    }
  }
});
```

### Clever Solutions

1. **File Mutation Queue**: Built-in tools and extensions share `withFileMutationQueue()` to prevent parallel edits to the same file from racing.

2. **Output Truncation Utilities**: Extensions get `truncateHead()`, `truncateTail()`, `truncateLine()` helpers to avoid overwhelming context.

3. **Event Bus for Extension Communication**: Shared pub/sub between extensions without coupling.

4. **Source Info Tracking**: Every registered item carries provenance (`source`, `scope`, `baseDir`) for diagnostics.

### Technical Debt / Rough Edges

1. **No Isolation**: Extensions share the same process. No sandboxing. Security is trust-based.

2. **Hot Reload Complexity**: `ctx.reload()` behavior is subtle - old code frame continues, new runtime starts. Documentation warns about in-memory state assumptions.

3. **Circular Dependency Risk**: Note in loader.ts comments that `index.ts` doesn't re-export from loader to avoid circular deps.

---

## Feature 9: Skills and Prompt Templates

### Overview

Skills are self-contained capability packages invoked via `/skill:name`, following the Agent Skills standard. Prompt templates are Markdown snippets expanding via `/name`.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `packages/coding-agent/src/core/skills.ts` | Skill loading, validation, formatting |
| `packages/coding-agent/src/core/prompt-templates.ts` | Template loading, argument substitution |

### Skills Implementation

**Discovery and Loading**

Skills are loaded from:
- `~/.pi/agent/skills/` (user global)
- `~/.agents/skills/` (Agent Skills compat)
- `.pi/skills/` (project)
- `.agents/skills/` (walk up to repo root)
- `skills/` directories in npm packages
- Explicit `--skill` paths

Two discovery modes:
1. Direct `.md` files in skills directory root
2. Recursive `SKILL.md` files under subdirectories

**Validation (Agent Skills Standard)**

```typescript
function validateName(name: string, parentDirName: string): string[] {
  const errors = [];
  if (name !== parentDirName) errors.push("name does not match parent directory");
  if (name.length > 64) errors.push("name exceeds 64 characters");
  if (!/^[a-z0-9-]+$/.test(name)) errors.push("invalid characters");
  if (name.startsWith("-") || name.endsWith("-")) errors.push("no leading/trailing hyphens");
  if (name.includes("--")) errors.push("no consecutive hyphens");
  return errors;
}
```

Skills without description are NOT loaded. Other violations produce warnings.

**Progressive Disclosure Pattern**

```typescript
export function formatSkillsForPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter(s => !s.disableModelInvocation);
  // Returns XML format for system prompt
  // Only names/descriptions included, full content loaded on-demand via read tool
}
```

The system prompt includes skill names/descriptions. The agent must use `read` to load actual SKILL.md content.

**Symlink and Collision Handling**

```typescript
const realPath = realpathSync(skill.filePath); // Resolve symlinks
if (realPathSet.has(realPath)) continue; // Skip duplicate files
```

Name collisions warn but keep the first-loaded skill.

### Prompt Templates Implementation

**Argument Substitution (Bash-style)**

```typescript
export function substituteArgs(content: string, args: string[]): string {
  let result = content;

  // Positional args ($1, $2) FIRST - before wildcards
  result = result.replace(/\$(\d+)/g, (_, num) => args[parseInt(num) - 1] ?? "");

  // Slice syntax ${@:start} or ${@:start:length}
  result = result.replace(/\$\{@:(\d+)(?::(\d+))?\}/g, (_, startStr, lengthStr) => {
    let start = parseInt(startStr) - 1;
    if (start < 0) start = 0;
    if (lengthStr) return args.slice(start, start + parseInt(lengthStr)).join(" ");
    return args.slice(start).join(" ");
  });

  // $ARGUMENTS and $@ for all args
  result = result.replace(/\$ARGUMENTS/g, args.join(" "));
  result = result.replace(/\$@/g, args.join(" "));

  return result;
}
```

Order matters: positional must be replaced before wildcards to avoid `$1` being caught by `$@`.

**Template Loading**

Templates from:
- `~/.pi/agent/prompts/*.md` (user)
- `.pi/prompts/*.md` (project)
- `prompts/` in packages
- Explicit `--prompt-template` paths

**Expansion Flow**

```typescript
export function expandPromptTemplate(text: string, templates: PromptTemplate[]): string {
  if (!text.startsWith("/")) return text;
  const templateName = text.slice(1, text.indexOf(" "));
  const argsString = text.slice(text.indexOf(" ") + 1);
  const template = templates.find(t => t.name === templateName);
  if (template) {
    return substituteArgs(template.content, parseCommandArgs(argsString));
  }
  return text;
}
```

### Skills vs Templates Comparison

| Aspect | Skills | Templates |
|--------|--------|-----------|
| Invocation | `/skill:name [args]` | `/name [args]` |
| Content | Full workflow instructions | Prompt snippet |
| Standard | Agent Skills spec | Custom |
| Model Visibility | Progressive disclosure | Always in context |
| Filesystem | Directory with SKILL.md | Single .md file |

### Key Insight: Skills Follow Agent Skills Standard

The implementation explicitly follows the [Agent Skills standard](https://agentskills.io/specification):

- XML format for skill listings in system prompt
- Frontmatter fields: `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`, `disable-model-invocation`
- Name validation per spec (lowercase, no leading/trailing/consecutive hyphens)
- Lenient loading (warnings instead of errors) for most violations

---

## Feature 10: Multi-Model Authentication

### Overview

Supports API keys via environment variables, OAuth flows for Anthropic/GitHub Copilot/Google Gemini CLI/Antigravity/OpenAI Codex, and token refresh with file locking.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `packages/ai/src/utils/oauth/index.ts` | OAuth provider registry |
| `packages/ai/src/utils/oauth/types.ts` | OAuth type definitions |
| `packages/ai/src/utils/oauth/anthropic.ts` | Anthropic OAuth implementation |
| `packages/ai/src/utils/oauth/github-copilot.ts` | GitHub Copilot OAuth |
| `packages/ai/src/utils/oauth/google-gemini-cli.ts` | Google Gemini CLI OAuth |
| `packages/ai/src/utils/oauth/google-antigravity.ts` | Antigravity OAuth |
| `packages/ai/src/utils/oauth/openai-codex.ts` | OpenAI Codex OAuth |
| `packages/ai/src/utils/oauth/pkce.ts` | PKCE code generation |
| `packages/ai/src/utils/oauth/oauth-page.ts` | OAuth callback HTML |
| `packages/coding-agent/src/core/auth-storage.ts` | Credential storage with locking |

### OAuth Provider Interface

```typescript
export interface OAuthProviderInterface {
  readonly id: OAuthProviderId;
  readonly name: string;
  usesCallbackServer?: boolean;
  login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
  refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;
  getApiKey(credentials: OAuthCredentials): string;
  modifyModels?(models, credentials): Model[]; // Optional hook
}
```

### Anthropic OAuth Flow

Uses authorization code + PKCE with local callback server:

```typescript
async function loginAnthropic(options) {
  const { verifier, challenge } = await generatePKCE();
  const server = await startCallbackServer(verifier);

  // Open browser for auth
  options.onAuth({ url: `${AUTHORIZE_URL}?${authParams}`, instructions: "..." });

  // Wait for callback OR manual code input
  const result = await server.waitForCode();

  // Exchange code for tokens
  return exchangeAuthorizationCode(code, state, verifier, redirectUri);
}
```

Key features:
- Starts HTTP server on `127.0.0.1:53692`
- Supports manual code paste if browser on different machine
- 30-second timeout on token exchange
- State verification to prevent CSRF

### Provider Registry

```typescript
const BUILT_IN_OAUTH_PROVIDERS = [
  anthropicOAuthProvider,
  githubCopilotOAuthProvider,
  geminiCliOAuthProvider,
  antigravityOAuthProvider,
  openaiCodexOAuthProvider,
];

export function registerOAuthProvider(provider: OAuthProviderInterface): void {
  oauthProviderRegistry.set(provider.id, provider);
}
```

Custom providers can be registered at runtime. Built-in providers can be overridden.

### Auth Storage with File Locking

```typescript
export class AuthStorage {
  private storage: AuthStorageBackend; // File or InMemory

  async getApiKey(providerId: string): Promise<string | undefined> {
    // Priority: runtime override > api_key > oauth (auto-refresh) > env var > fallback

    const runtimeKey = this.runtimeOverrides.get(providerId);
    if (runtimeKey) return runtimeKey;

    const cred = this.data[providerId];
    if (cred?.type === "api_key") return resolveConfigValue(cred.key);

    if (cred?.type === "oauth") {
      if (Date.now() >= cred.expires) {
        // Locked refresh prevents race condition with multiple pi instances
        const result = await this.refreshOAuthTokenWithLock(providerId);
        return result?.apiKey;
      }
      return provider.getApiKey(cred);
    }
  }
}
```

### Clever Solution: Token Refresh Race Condition Prevention

When multiple pi instances start with expired OAuth tokens, they could all try to refresh simultaneously. The `proper-lockfile` library prevents this:

```typescript
async refreshOAuthTokenWithLock(providerId: string) {
  const result = await this.storage.withLockAsync(async (current) => {
    // Re-read current data while holding lock
    const currentData = JSON.parse(current);

    // If another instance refreshed, use those creds
    if (Date.now() < currentData[providerId].expires) {
      return { result: currentData[providerId].apiKey };
    }

    // Refresh token, return new creds
    const refreshed = await getOAuthApiKey(providerId, currentData);
    return { result: refreshed, next: JSON.stringify(merged) };
  });
  return result;
}
```

### Credential Types

```typescript
export type ApiKeyCredential = { type: "api_key"; key: string };
export type OAuthCredential = { type: "oauth" } & OAuthCredentials;
export type AuthCredential = ApiKeyCredential | OAuthCredential;

export type OAuthCredentials = {
  refresh: string;
  access: string;
  expires: number; // Date.now() when token expires (with buffer subtracted)
  [key: string]: unknown; // Allow extension-specific fields
};
```

### Storage Location

Auth storage is at `~/.pi/agent/auth.json` with 0o600 permissions:

```typescript
constructor(authPath: string = join(getAgentDir(), "auth.json")) {
  // Creates parent dir with 0o700, file with 0o600
}
```

### Runtime Overrides

CLI `--api-key` flags set non-persisted overrides that take highest priority:

```typescript
setRuntimeApiKey(provider: string, apiKey: string): void {
  this.runtimeOverrides.set(provider, apiKey);
}
```

---

## Feature 11: Session Persistence and Branching

### Overview

Tree-based session storage with JSONL files. Supports branching, compaction, labels, and session forking across projects.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `packages/coding-agent/src/core/session-manager.ts` | SessionManager class (~1400 lines) |
| `packages/coding-agent/src/core/messages.ts` | Extended message types (BashExecution, Custom, etc.) |
| `packages/coding-agent/src/core/compaction/` | Compaction logic |

### Session File Format

Sessions are JSONL (JSON Lines) with a tree structure:

```
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"...","message":{...}}
{"type":"compaction","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"...","summary":"..."}
```

### Session Version Migration

```typescript
export const CURRENT_SESSION_VERSION = 3;

// v1: Linear entries (no id/parentId)
// v2: Tree structure with id/parentId
// v3: Renamed hookMessage role to custom

function migrateV1ToV2(entries: FileEntry[]): void {
  let prevId: string | null = null;
  for (const entry of entries) {
    entry.id = generateId(ids);
    entry.parentId = prevId;
    prevId = entry.id;
  }
}

function migrateV2ToV3(entries: FileEntry[]): void {
  // Rename hookMessage role to custom
  if (msgEntry.message.role === "hookMessage") {
    msgEntry.message.role = "custom";
  }
}
```

### Tree Data Structure

Each entry has `id` (8-char hex) and `parentId` forming a tree. The "leaf" pointer tracks current position:

```typescript
class SessionManager {
  private leafId: string | null = null;  // Current position
  private fileEntries: FileEntry[] = [];  // All entries
  private byId: Map<string, SessionEntry> = new Map(); // Quick lookup

  appendMessage(message): string {
    const entry: SessionMessageEntry = {
      id: generateId(this.byId),
      parentId: this.leafId, // Child of current leaf
      // ...
    };
    this.fileEntries.push(entry);
    this.byId.set(entry.id, entry);
    this.leafId = entry.id; // Advance leaf
    return entry.id;
  }

  branch(branchFromId: string): void {
    this.leafId = branchFromId; // Move leaf back
    // Next append creates child of this earlier entry
  }
}
```

### Entry Types

| Type | Purpose | Participates in Context |
|------|---------|------------------------|
| `message` | User, assistant, tool result messages | Yes |
| `thinking_level_change` | Model thinking level | No (metadata) |
| `model_change` | Provider/model switch | No (metadata) |
| `compaction` | Summarized old context | Yes (emits summary) |
| `branch_summary` | Abandoned branch summary | Yes (emits summary) |
| `custom` | Extension state (not in context) | No |
| `custom_message` | Extension-injected message | Yes |
| `label` | Entry bookmark | No |
| `session_info` | Display name | No |

### Context Building

```typescript
export function buildSessionContext(entries, leafId?, byId?): SessionContext {
  // Walk from leaf to root
  const path: SessionEntry[] = [];
  let current = leaf;
  while (current) {
    path.unshift(current);
    current = current.parentId ? byId.get(current.parentId) : undefined;
  }

  // Extract model/thinking from path
  // If compaction on path: emit summary + kept messages + messages after
  // Otherwise: emit all messages

  return { messages, thinkingLevel, model };
}
```

### Branching with Summaries

When branching, the abandoned path can be summarized:

```typescript
branchWithSummary(branchFromId, summary, details?, fromHook?): string {
  this.leafId = branchFromId; // Move leaf back
  const entry: BranchSummaryEntry = {
    type: "branch_summary",
    id: generateId(this.byId),
    parentId: branchFromId,
    fromId: branchFromId,
    summary,
    // ...
  };
  this._appendEntry(entry);
  return entry.id;
}
```

### Session Persistence Strategy

```typescript
_persist(entry: SessionEntry): void {
  if (!this.persist || !this.sessionFile) return;

  // Don't write until assistant arrives
  const hasAssistant = this.fileEntries.some(
    e => e.type === "message" && e.message.role === "assistant"
  );
  if (!hasAssistant) {
    this.flushed = false; // Defer write
    return;
  }

  if (!this.flushed) {
    // First assistant: write all accumulated entries
    for (const e of this.fileEntries) {
      appendFileSync(this.sessionFile, `${JSON.stringify(e)}\n`);
    }
    this.flushed = true;
  } else {
    appendFileSync(this.sessionFile, `${JSON.stringify(entry)}\n`);
  }
}
```

This avoids creating session files for sessions that never get an assistant response.

### Session Foraking Across Projects

```typescript
static forkFrom(sourcePath, targetCwd, sessionDir?): SessionManager {
  // Copy all entries from source to new session file in target
  // New header has parentSession pointing to source

  appendFileSync(newSessionFile, `${JSON.stringify(newHeader)}\n`);
  for (const entry of sourceEntries) {
    if (entry.type !== "session") {
      appendFileSync(newSessionFile, `${JSON.stringify(entry)}\n`);
    }
  }
  return new SessionManager(targetCwd, dir, newSessionFile, true);
}
```

### Extended Message Types (messages.ts)

```typescript
export interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean; // !! prefix
}

export interface CustomMessage<T = unknown> {
  role: "custom";
  customType: string;
  content: string | (TextContent | ImageContent)[];
  display: boolean;
  details?: T;
}
```

Declaration merging extends AgentMessage union:

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    bashExecution: BashExecutionMessage;
    custom: CustomMessage;
    branchSummary: BranchSummaryMessage;
    compactionSummary: CompactionSummaryMessage;
  }
}
```

### Clever Solutions

1. **ID Generation with Collision Check**: 8-char hex IDs with retry on collision:
   ```typescript
   function generateId(byId: { has(id: string): boolean }): string {
     for (let i = 0; i < 100; i++) {
       const id = randomUUID().slice(0, 8);
       if (!byId.has(id)) return id;
     }
     return randomUUID(); // Fallback
   }
   ```

2. **Defensive Tree Building**: Orphaned entries (broken parent chain) returned as roots in `getTree()`.

3. **Session Info Reverse Lookup**: `getSessionName()` walks entries in reverse to find latest `session_info`.

4. **Symlink-Aware Collision Detection**: Uses `realpathSync()` to detect if same file accessed via different paths.

### File Location Pattern

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

Where path is `/` replaced with `-` and leading `/` removed.

### Labels for Navigation

```typescript
appendLabelChange(targetId, label): string {
  const entry: LabelEntry = {
    type: "label",
    targetId,
    label,
    // ...
  };
  // Stored separately in labelsById map for quick lookup
  this.labelsById.set(targetId, label);
}
```

Labels persist but don't appear in context. Useful for bookmarks in `/tree` navigation.

---

## Cross-Feature Interactions

### Extensions + Sessions

Extensions can persist state across reloads via `CustomEntry`:

```typescript
pi.on("session_start", async (_event, ctx) => {
  const entries = ctx.sessionManager.getEntries();
  // Scan for extension's custom entries, reconstruct state
});
```

### Extensions + Skills

Skills can register tools that extensions then enhance or intercept:

```typescript
// Skill provides: ./scripts/process.sh
// Extension intercepts tool_call for "bash" to wrap execution
```

### Extensions + Auth

Extensions can register custom OAuth providers:

```typescript
pi.registerProvider("my-corp", {
  baseUrl: "https://ai.corp.com",
  oauth: {
    name: "Corporate SSO",
    login(callbacks) { /* ... */ },
    refreshToken(creds) { /* ... */ },
    getApiKey(creds) { return creds.access; }
  }
});
```

### Sessions + Compaction

Compaction is orthogonal to sessions. The session just stores `compaction` entries. Compaction logic (in `compaction/`) handles:
- Deciding what to summarize
- Generating summary text
- Recording which entries are kept

### Skills + Prompt Templates

Both use the `/` prefix syntax but:
- Skills: `/skill:name` expands to skill content
- Templates: `/name` expands to template content with argument substitution

---

## Summary of Technical Patterns

1. **Extensibility via Registration**: Tools, commands, shortcuts, providers all registered at runtime rather than configured statically.

2. **Event-Driven Architecture**: Extensions and session management use typed event systems with handler chains.

3. **File-Based Persistence with Atomic Updates**: JSONL append-only, session files created only after first assistant message.

4. **Tree over Linear**: Session model explicitly represents branching conversation history.

5. **Standard Compliance**: Agent Skills spec followed (leniently), Agent Skills standard compatible.

6. **Locking for Race Conditions**: proper-lockfile prevents multiple processes from corrupting auth state during refresh.

7. **Type-Based Code Generation**: Declaration merging extends base types rather than wrapper classes.
