# My Action Items: Building a Git-Native AI Agent

**Based on:** GitClaw research synthesis (12-lessons-learned.md)
**Purpose:** Prioritized concrete next steps for building a similar project

---

## Priority 1: Foundation (Do First)

### 1.1 Adopt Hexagonal Architecture from Day One

**Why:** GitClaw's architecture enables multiple entry points (CLI, SDK, Voice) sharing the same agent core.

**Action:** Separate into three layers:
```
src/
  core/           # Domain: agent loading, tool execution, memory
  adapters/       # Infrastructure: CLI, SDK, voice server
  ports/          # Interfaces: AgentTool, HookDefinition, PluginManifest
```

**Specific patterns to implement:**
- Factory pattern for tool creation (`createBuiltinTools()`)
- Decorator pattern for hooks (`wrapToolWithHooks()`)
- Channel pattern for async streaming (`createChannel<T>()`)
- Repository pattern for file discovery

### 1.2 Implement Git-Native Memory Correctly

**Why:** GitClaw's memory-as-commits is elegant but has rough edges (silent failures, no atomicity).

**Action items:**
- [ ] Use `git add` + `git commit` with proper error handling (not `|| true`)
- [ ] Implement layered memory via `memory.yaml` config
- [ ] Add archive overflow to time-based files (`memory/archive/YYYY-MM.md`)
- [ ] Return git commit status to caller, don't silently swallow failures

**Correct pattern:**
```typescript
try {
  execSync(`git add "${memoryPath}" && git commit -m "${escape(commitMsg)}"`, { stdio: "pipe" });
  return { success: true };
} catch (err) {
  return { success: false, error: err.message };  // Don't silently continue
}
```

### 1.3 Add ESLint + Prettier from Day One

**Why:** GitClaw has no linting. Retrofitting is painful.

**Action:** Add to project before first commit:
```bash
npm init @eslint/config@latest
npx prettier --write .
```

**Configs to include:**
- `@typescript-eslint/recommended`
- `prettier` as last (overrides formatting conflicts)

---

## Priority 2: Type Safety (Do Early)

### 2.1 Avoid `any` in Critical Paths

**Why:** GitClaw uses `any` for `gitMachine` and `modelOptions`, undermining TypeScript's guarantees.

**Action:**
- [ ] If using optional peer dependencies, create proper type wrappers, not `any`
- [ ] Define `ModelOptions` interface with all constraint fields
- [ ] Validate at boundaries (config loading, API responses)

### 2.2 Add Runtime Manifest Validation

**Why:** GitClaw parses `agent.yaml` directly to typed object with no validation.

**Action:**
```bash
npm install zod
```

```typescript
import { z } from "zod";

const AgentManifestSchema = z.object({
  spec_version: z.string(),
  name: z.string(),
  version: z.string(),
  model: z.object({
    preferred: z.string(),
    fallback: z.array(z.string()).optional(),
  }),
  // ... rest of schema
});

function loadAgentManifest(path: string): AgentManifest {
  const raw = yaml.load(readFileSync(path, "utf-8"));
  return AgentManifestSchema.parse(raw);  // Throws on invalid
}
```

### 2.3 Use Typebox for Runtime Validation

**Why:** GitClaw uses Typebox for tool schemas. It's a good pattern.

**Action:** Use Typebox consistently for:
- Tool input/output schemas
- Plugin manifest schemas
- API request/response types

---

## Priority 3: Testing (Do Continuously)

### 3.1 Test the Core, Not Just SDK

**Why:** GitClaw only tests SDK. Built-in tools, CLI, and plugins are untested.

**Action - Minimum viable test suite:**
```
test/
  unit/
    loader.test.ts       # Agent loading, inheritance
    tools.test.ts        # Built-in tools
    memory.test.ts       # Git-backed memory
  integration/
    sdk.test.ts          # Full query lifecycle
    cli.test.ts          # CLI REPL
```

### 3.2 Add Test Step to CI

**Why:** GitClaw's CI only builds. Tests run locally (if at all).

**Action - GitHub Actions workflow:**
```yaml
- name: Test
  run: npm test
- name: Build
  run: npm run build
```

### 3.3 Test Error Paths

**Why:** GitClaw has silent error suppression everywhere.

**Action:** Specifically test:
- Git commit failures
- Missing config files
- Invalid manifest YAML
- Network timeouts

---

## Priority 4: Error Handling (Do Correctly)

### 4.1 Replace Silent Catches with Structured Logging

**Why:** Empty catch blocks make debugging painful.

**Pattern to follow:**
```typescript
// BAD
} catch {
  // silently ignore
}

// GOOD
} catch (err) {
  logger.error({ err, context: "audit_logging" }, "Failed to write audit entry");
  // Decide: rethrow, return error result, or continue with warning
}
```

### 4.2 Use Result Types for Fallible Operations

**Why:** Silent failures return `undefined` or partial state.

**Pattern:**
```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function saveMemory(content: string): Promise<Result<void>> {
  try {
    await writeFile(memoryFile, content);
    execSync(`git add "${memoryFile}" && git commit...`);
    return { ok: true };
  } catch (err) {
    return { ok: false, error: err as Error };
  }
}
```

### 4.3 Add Timeout Configuration

**Why:** GitClaw hardcodes 10s for hooks, 120s for tools.

**Action:** Make timeouts configurable via agent.yaml:
```yaml
runtime:
  timeout:
    hooks: 30      # seconds
    tools: 300     # seconds
    declarative: 120
```

---

## Priority 5: Security (Do Proactively)

### 5.1 Audit Environment Variable Exposure

**Why:** GitClaw's CLI tool passes full `process.env` to subprocesses.

**Action:**
- [ ] Implement environment scrubbing for CLI tool
- [ ] Whitelist which env vars are passed to tools
- [ ] Document security model clearly

### 5.2 Add Path Traversal Validation

**Why:** GitClaw has inconsistent path handling (sandbox has guards, knowledge doesn't).

**Action:** Centralize path resolution:
```typescript
function resolveAgentPath(relative: string, agentDir: string): string {
  const resolved = resolve(agentDir, relative);
  if (!resolved.startsWith(agentDir)) {
    throw new Error("Path traversal detected");
  }
  return resolved;
}
```

### 5.3 Don't Embed Tokens in URLs

**Why:** GitClaw's session.ts embeds PAT in git URLs, strips later.

**Better approach:** Use GitHub's token-based auth header instead of URL embedding.

---

## Priority 6: Extensibility (Do Right)

### 6.1 Design Plugin System Carefully

**GitClaw's approach was good but imperfect:**
- [ ] Support both declarative (YAML) and programmatic (TypeScript)
- [ ] Three discovery locations (local, global, installed)
- [ ] Tool collision detection (skip entire plugin, not just colliding tools)
- [ ] **Fix:** Allow tool-level collision handling, not plugin-level
- [ ] **Fix:** Add namespace isolation for plugins

### 6.2 Implement Skills Properly

**GitClaw's skills are well-designed:**
- [ ] Kebab-case naming enforced
- [ ] Name must match directory
- [ ] Frontmatter with metadata
- [ ] Slash command expansion

**Missing in GitClaw (add):**
- [ ] Skill versioning
- [ ] Skill dependencies
- [ ] Built-in skill discovery from URLs (like agent inheritance)

### 6.3 Consider Learning System

**GitClaw's reinforcement learning approach is sophisticated:**
- [ ] Confidence tracking per skill
- [ ] Asymmetric loss (2x penalty for failure)
- [ ] Negative example storage
- [ ] Skill crystallization from successful tasks

**Start simpler:** Just track usage counts first. Add confidence later.

---

## Priority 7: Observability (Do Early)

### 7.1 Structured Logging Over console.*

**Why:** GitClaw uses `console.error` for diagnostics.

**Action:**
```bash
npm install pino
```

```typescript
import pino from "pino";
const logger = pino({ level: process.env.LOG_LEVEL || "info" });

logger.info({ tool: "cli", args }, "Executing CLI tool");
logger.error({ err }, "Tool execution failed");
```

### 7.2 Add Audit Logging

**GitClaw's JSONL approach is good:**
- Append-only
- Line-oriented for easy parsing
- Session-correlated

**Action:** Implement early, not as an afterthought.

### 7.3 Metrics and Health Checks

**For voice/server mode:**
- CPU/memory usage tracking
- Request latency histograms
- Connection counts

---

## Priority 8: Documentation (Do Alongside Code)

### 8.1 Document Architecture Decisions

**Why:** GitClaw's decisions are scattered across code comments.

**Action:** Create `ARCHITECTURE.md` with:
- Component map
- Data flow diagrams
- Key interface definitions

### 8.2 Document Security Model

**Why:** Agent frameworks have complex security implications.

**Document:**
- What environment variables are exposed to tools
- Path traversal protections
- Sandbox isolation guarantees
- Token handling

### 8.3 Document Extension Points

**GitClaw's extension model:**
- Adding a new tool
- Adding a new voice adapter
- Creating a plugin

**Create guides for each.**

---

## Summary: Priority Order

| Priority | Category | Key Action |
|----------|----------|------------|
| 1 | Foundation | Hexagonal architecture + git-native memory with proper error handling |
| 2 | Type Safety | Avoid `any`, add runtime validation with Zod |
| 3 | Testing | Test core paths, add test to CI |
| 4 | Error Handling | Structured logging, no silent failures |
| 5 | Security | Audit env exposure, add path validation |
| 6 | Extensibility | Well-designed plugin + skills systems |
| 7 | Observability | Structured logging, audit logging |
| 8 | Documentation | Architecture, security, extension guides |

---

## What NOT to Copy from GitClaw

1. **Silent error suppression** - Always handle/return errors
2. **execSync for git operations** - Use async or a git library
3. **Hardcoded timeouts** - Make configurable
4. **No linting** - Add ESLint/Prettier from day one
5. **No CI tests** - Tests run in CI before merge
6. **`any` types in core** - Maintain type safety throughout
7. **Mutable shared state** - Thread-safe or scoped state
8. **Empty catch blocks** - Log or propagate errors

---

## Inspiration to Keep

1. **Git-native theme is coherent** - Apply it consistently
2. **Multi-model abstraction** - Clean `provider:model` format
3. **Hook system** - Allow/block/modify is powerful
4. **Learning system** - Confidence-based skill improvement
5. **Clear entry point separation** - CLI/SDK/Voice share core
