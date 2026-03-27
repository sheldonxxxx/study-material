# Lessons Learned from pi-mono Study

## Executive Summary

pi-mono is a production-grade monorepo (~28k GitHub stars) maintaining 7 packages with ~23 commits/day. Built by a single maintainer (badlogic) with help from 137 approved contributors. The project demonstrates how thoughtful architecture can scale a complex AI agent toolkit while maintaining developer velocity.

---

## What to Emulate

### 1. Layered Architecture with Clean Boundaries

**Pattern:** Provider Layer (pi-ai) -> Runtime Layer (pi-agent-core) -> Application Layer (pi-coding-agent) -> Presentation Layer (pi-tui, pi-web-ui)

**Why it works:** Each layer has a single responsibility and dependency only flows downward. pi-ai knows nothing about agents, pi-agent-core knows nothing about CLI concerns.

**Specific implementation:**
```
packages/ai/src/types.ts
  - StreamFunction<TApi, TOptions> interface
  - Providers implement stream() returning AssistantMessageEventStream
  - Failures encoded in stream, not thrown

packages/agent/src/agent-loop.ts
  - Consumes pi-ai stream functions
  - Orchestrates tool execution
  - Emits typed AgentEvent (agent_start, turn_start, message_update, tool_execution_*)

packages/coding-agent/src/core/agent-session.ts
  - Subscribes to agent events
  - Handles session persistence, compaction, branching
```

**Action:** Start with a clear dependency diagram. Enforce no upward dependencies through tooling or code review.

---

### 2. Event-Driven Streaming Architecture

**Pattern:** Fine-grained events enable responsive UI without polling.

**Specific implementation from `packages/ai/src/types.ts`:**
```typescript
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
  | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

**Why better than polling:** UI components render partial results in real-time. TUI (pi-tui) uses these events to update the display without refreshing the entire screen.

**Action:** Design streaming events to be granular enough for meaningful partial renders.

---

### 3. Registry Pattern for Extensibility

**Pattern:** Central registration for providers, tools, extensions, models.

**Specific implementation from `packages/ai/src/api-registry.ts`:**
```typescript
export interface ApiProvider<TApi extends Api = Api, TOptions extends StreamOptions = StreamOptions> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}

registerApiProvider({
  api: "anthropic-messages",
  stream: streamAnthropic,
  streamSimple: streamSimpleAnthropic,
});
```

**For models:** `packages/ai/src/models.ts` uses a Map of Maps:
```typescript
const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();
export function getModel<TProvider extends KnownProvider, TModelId extends ...>(...): Model<...>
```

**Action:** Use Map-based registries with typed getter functions rather than if/else chains or switch statements.

---

### 4. Tree-Based Session Persistence

**Pattern:** JSONL files with explicit parentId forming a tree. Current position tracked via `leafId`.

**Specific implementation from `packages/coding-agent/src/core/session-manager.ts`:**
```
Session entries (JSONL):
{"type":"session","id":"abc123",...}
{"type":"message","id":"def456","parentId":"abc123",...}
{"type":"message","id":"ghi789","parentId":"def456",...}

Leaf pointer: ghi789 (current position)
```

**Benefits:**
- Branching: Create new leaf from any historical node
- History: Parent chain gives you full conversation tree
- Compaction: Summarize subtrees, keep summarized entry

**Action:** Model conversation history as a tree, not a list. Enable branching from day one.

---

### 5. Contract-Based Error Handling

**Pattern:** Providers must not throw for request/model/runtime failures. Errors are encoded in the stream.

**Specific implementation:**
```typescript
// packages/ai/src/types.ts
| { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };

// packages/agent/src/agent-loop.ts
// Tool errors returned as ToolResultMessage with isError: true
// Abort signal handling for all async operations
```

**Why better:** UI can display errors inline, streaming continues gracefully, no try/catch waterfalls.

**Action:** Define error states in return types. Do not use thrown exceptions for expected failure modes.

---

### 6. OAuth Token Refresh with Distributed Locking

**Pattern:** Multiple pi instances starting simultaneously with expired tokens must not all refresh. Use proper-lockfile.

**Specific implementation from `packages/coding-agent/src/core/auth-storage.ts`:**
```typescript
async refreshOAuthTokenWithLock(providerId: string) {
  const result = await this.storage.withLockAsync(async (current) => {
    // Double-checked locking: another instance may have refreshed
    if (Date.now() < currentData[providerId].expires) {
      return { result: currentData[providerId].apiKey };
    }
    // Refresh token...
  });
}
```

**Lock config:**
```typescript
const release = await lockfile.lock(this.authPath, {
  retries: { retries: 10, factor: 2, minTimeout: 100, maxTimeout: 10000 },
  stale: 30000,
});
```

**Action:** Use file locking for shared state across processes. Implement double-checked locking pattern.

---

### 7. Differential TUI Rendering

**Pattern:** Only re-render what changed. Three strategies based on what changed.

**Specific implementation from `packages/tui/src/tui.ts`:**
```typescript
// 1. First render: full output without clearing
// 2. Width/height changes: full clear and re-render
// 3. Content changes: incremental render from first changed line

// CSI 2026 for atomic screen updates
// \x1b[?2026h ... \x1b[?2026l
```

**Action:** Track what changed (lines, viewport position) and only redraw affected areas.

---

### 8. Context Overflow Detection

**Pattern:** 40+ provider-specific error patterns for context overflow detection.

**Specific implementation from `packages/ai/src/utils/overflow.ts`:**
```typescript
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,                    // Anthropic
  /input is too long for requested model/i, // Amazon Bedrock
  /exceeds the context window/i,            // OpenAI
  /input token count.*exceeds the maximum/i, // Google
  /maximum prompt length is \d+/i,           // xAI
  /reduce the length of the messages/i,      // Groq
  // ... 20+ more patterns
];
```

**Action:** Build a pattern library of provider error messages. Include "silent overflow" detection (stopReason=stop but inputTokens > contextWindow).

---

### 9. Partial JSON Parsing for Streaming

**Pattern:** Streaming tool call arguments may arrive mid-JSON-object.

**Specific implementation from `packages/ai/src/utils/json-parse.ts`:**
```typescript
export function parseStreamingJson<T = any>(partialJson: string | undefined): T {
  // Try standard parsing first (fastest for complete JSON)
  try {
    return JSON.parse(partialJson) as T;
  } catch {
    // Try partial-json for incomplete streaming JSON
    try {
      const result = partialParse(partialJson);
      return (result ?? {}) as T;
    } catch {
      return {} as T;  // Fail gracefully
    }
  }
}
```

**Action:** Use `partial-json` library for streaming JSON. Always fail gracefully with empty object.

---

### 10. Progressive Disclosure for Skills

**Pattern:** System prompt includes skill names/descriptions. Actual content loaded on-demand via `read` tool.

**Specific implementation from `packages/coding-agent/src/core/skills.ts`:**
```typescript
export function formatSkillsForPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter(s => !s.disableModelInvocation);
  // Returns XML format for system prompt
  // Only names/descriptions included, full content loaded on-demand
}
```

**Why:** Avoids overwhelming context with full skill content. Model learns about available skills without consuming tokens.

**Action:** Implement progressive disclosure for any optional capabilities (skills, tools, docs).

---

## What to Avoid

### 1. Large Generated Files in Source Tree

**Problem:** `packages/ai/src/models.generated.ts` is 351KB of auto-generated model definitions. Affects IDE load times.

**Mitigation observed:** File is excluded from Biome linting:
```json
"excludes": [
  "!**/models.generated.ts",
]
```

**Recommendation:** Keep generated files separate from source. Use build scripts to generate, not commit generated content.

---

### 2. Provider-Specific Hacks

**Problem:** Some providers require special handling that accumulates:
```typescript
// From providers/anthropic.ts
requiresThinkingAsText?: boolean;
requiresAssistantAfterToolResult?: boolean;

// Provider-specific overrides accumulate
```

**Recommendation:** Encapsulate hacks in provider adapter, not in core. Consider a `ProviderModifier` interface.

---

### 3. Ad-hoc Console Logging

**Problem:** Most packages use `console.log` (~700 occurrences). No structured logging library.

| Package | Logging Pattern |
|---------|---------------|
| mom | Dedicated `log.ts` with chalk, structured |
| coding-agent | `PI_TUI_WRITE_LOG` env var for debug only |
| Others | `console.log` scattered throughout |

**Recommendation:** Use pino or winston from the start. mom package shows the right pattern.

---

### 4. No Pre-commit Hooks for Lint

**Problem:** Husky only runs `prepare` script. No lint checks on commit.

**From `package.json`:**
```json
"prepare": "husky install"
```

**Recommendation:** Add `.husky/pre-commit` that runs `npm run check` or at minimum `biome check`.

---

### 5. Direct Commits to Main

**Problem:** No visible PR review process in commit history. Commits go directly to main.

**Observation:** High velocity (23+ commits/day) suggests trust-based model with APPROVED_CONTRIBUTORS list.

**Recommendation:** For a solo developer or small team, direct commits are fine. For open source with many contributors, require PR reviews.

---

### 6. Large Monolithic Files

**Problem:** `agent-session.ts` and `main.ts` are ~1000+ lines each.

**Impact:** Hard to navigate, review, test. Single concern wrapped in many concerns.

**Recommendation:** Break into smaller files by concern. Use index.ts to re-export.

---

### 7. Missing Tests for Some Packages

**Observation:** pi-mom (Slack bot) and pi-pods (GPU CLI) have no visible unit tests.

**Recommendation:** Add at minimum integration tests. These packages have external integrations where bugs are costly.

---

### 8. Limited Type Safety in AJV Validation

**Problem:** AJV validation returns `any`, defeating TypeScript's type narrowing.

**From `packages/ai/src/utils/validation.ts`:**
```typescript
export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
  const validate = ajv.compile(tool.parameters);
  // validate.errors is typed broadly
}
```

**Recommendation:** Consider TypeBox's native validation with type inference, or wrap AJV with typed error handlers.

---

## Surprises

### 1. npm Workspaces Without Turborepo/Nx

**Observation:** pi-mono uses pure npm workspaces for monorepo management. No Turborepo, Nx, or Lerna.

**Why surprising:** Conventional wisdom suggests needing task orchestration for multi-package builds.

**Why it works:**
- Build order is fixed and fast (sequential but total ~30 seconds)
- No complex task graphs needed
- Simpler toolchain, less to maintain

**Insight:** Don't add complexity you don't need. Re-evaluate assumptions about "required" tooling.

---

### 2. Biome Instead of ESLint/Prettier

**Observation:** Uses Biome (Rust-based linter/formatter) instead of ESLint + Prettier.

**Why surprising:** At time of project start, Biome was less mature. ESLint + Prettier is the conventional choice.

**Why it works:** Single tool for lint + format, faster execution, consistent config.

**Insight:** Evaluate tools on their merits, not conventions. Biome's speed advantage is real.

---

### 3. Single Maintainer with 28k Stars

**Observation:** Despite 28,137 stars and active development, the project is essentially maintained by one person (@badlogic).

**Why surprising:** We assume large open source projects need multiple maintainers.

**How it works:**
- APPROVED_CONTRIBUTORS list (137 people) for PR quality control
- Clear contribution guidelines
- Issue-first workflow (proposal before PR)
- Small team means faster decisions

**Insight:** Curation (who can contribute) matters more than quantity (how many contributors).

---

### 4. tsgo Wrapper Around tsx

**Observation:** Project uses `tsgo` (wrapper around tsx) instead of `tsc` for TypeScript compilation.

**Why surprising:** `tsc` is the standard TypeScript compiler.

**Why it works:** tsx has faster startup time, works well for development builds. The project isn't publishing type declarations directly (declarations generated separately).

**Insight:** Understand what your build tools actually do. `tsc` generates declarations; `tsx` just runs TypeScript.

---

### 5. File Locking for Auth State

**Observation:** Uses `proper-lockfile` npm package rather than OS keychain or secrets manager.

**Why surprising:** Conventional wisdom says use OS-native secret storage.

**Why it works:**
- Auth tokens are refresh tokens, not long-lived secrets
- Lockfile approach handles multi-instance refresh coordination
- Simpler dependency

**Insight:** Consider the actual threat model. For OAuth refresh tokens with short expiry, coordinated refresh matters more than hardware-level encryption.

---

### 6. Extensions Without Sandbox

**Observation:** pi-mono extensions run in the same process with full access.

**Why surprising:** Extensions execute arbitrary TypeScript. No WASM sandbox, no process isolation.

**Why it works:** Trust-based model. Extensions are user-installed, not from untrusted sources. Clear in documentation.

**Insight:** Security posture should match actual risk. Don't sandbox what users explicitly trust.

---

### 7. Custom jiti Fork for Virtual Modules

**Observation:** Project maintains a fork of jiti with `virtualModules` support for Bun binary compilation.

**Why it exists:** Extensions use jiti to load TypeScript at runtime. In compiled Bun binary, jiti can't find modules normally. Virtual modules bundle allowed packages into the binary.

**Insight:** Deep integration with build tooling often requires custom patches. Factor in maintenance cost of custom tooling.

---

## Summary

| Category | Emulate | Avoid | Surprise |
|----------|---------|-------|----------|
| Architecture | Layered, event-driven | Monolithic files | Clean boundaries scale |
| Extensibility | Registry pattern | Hardcoded cases | Registration over configuration |
| Error Handling | Contract-based | Exception throwing | Errors as return types |
| Streaming | Fine-grained events | Polling | Partial results are first-class |
| Session Model | Tree with leaf pointer | Linear history | Enables branching cheaply |
| Auth | OAuth with file locking | ad-hoc storage | Coordinated refresh > encryption |
| Tooling | Biome, npm workspaces | ESLint+Prettier, Turborepo | Simpler tooling works |
| Community | Curated contributors | Open merge policy | Small team, large impact |

---

## Key Files Reference

| Pattern | File |
|---------|------|
| Provider interface | `packages/ai/src/types.ts` |
| Event types | `packages/ai/src/types.ts` |
| API registry | `packages/ai/src/api-registry.ts` |
| Agent loop | `packages/agent/src/agent-loop.ts` |
| Session manager | `packages/coding-agent/src/core/session-manager.ts` |
| Auth storage | `packages/coding-agent/src/core/auth-storage.ts` |
| TUI core | `packages/tui/src/tui.ts` |
| Overflow detection | `packages/ai/src/utils/overflow.ts` |
| Event stream | `packages/ai/src/utils/event-stream.ts` |
