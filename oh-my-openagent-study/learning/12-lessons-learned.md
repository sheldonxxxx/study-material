# 12 Lessons Learned: Building oh-my-openagent

## Executive Summary

oh-my-openagent is a TypeScript monorepo plugin for OpenCode that provides multi-agent orchestration, world-class developer tools, and productivity features. After comprehensive research across topology, tech stack, community, 16 features, architecture, code quality, and security patterns, here are the honest lessons for building a similar project.

---

## Lesson 1: Hook-Based Architecture Wins for Extensibility

**What they did:** The project implements a hook-based plugin architecture where 48+ discrete hooks intercept OpenCode lifecycle events (`session.created`, `tool.execute.before`, `chat.message`, etc.). Hooks are composed via factory functions (`createCoreHooks()`, `createContinuationHooks()`, `createSkillHooks()`) and dispatched through a central event handler.

**Why it works:** Each hook is self-contained with clear boundaries. Adding a new feature means creating a new hook directory, implementing the hook interface, and registering it in the factory. The pattern is visible in `src/hooks/todo-continuation-enforcer/handler.ts` which handles 14 distinct skip conditions before injecting a continuation prompt.

**Emulate this:** Design your core abstraction around events, not direct method calls. Define a hook interface once and build all features as composable hooks.

**Specific pattern:**
```typescript
type Hook = {
  event?: (input: EventInput) => Promise<void>
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  dispose?: () => void
}
```

---

## Lesson 2: Hash-Anchored Edits Solve a Real Problem

**What they built:** The hashline edit tool tags every line with a content hash (`LINE#ID` format). Before applying edits, it validates the hash matches. This eliminated stale-line errors entirely, improving Grok Code Fast success rate from 6.7% to 68.3%.

**The algorithm:** Uses `Bun.hash.xxHash32()` with a clever seed strategy - whitespace-only lines use line number as seed (ensuring unique hashes per line), while content lines use seed=0 (identical content produces identical hash across files).

**Why it matters:** This is a concrete, measurable improvement. The research found 38 files in `src/tools/hashline-edit/` implementing robust validation, overlap detection, error recovery with remapping suggestions, and CRLF handling.

**Emulate this:** Identify the specific failure mode (stale edits) and solve it with a domain-specific algorithm. Generic solutions don't achieve 10x improvements.

---

## Lesson 3: Preemptive Compaction Beats Reactive

**What they built:** The context window management triggers session summarization at 78% capacity (not at 100%). The `preemptive-compaction` hook monitors token usage via `message.updated` events and calls `session.summarize()` proactively.

**Key insight:** Waiting until limits are hit means you're already degraded. The 70% warning threshold and 78% preemptive threshold create a safety margin.

**The implementation:** `src/hooks/preemptive-compaction.ts` tracks token usage in a cache, checks ratios on every `tool.execute.after`, and triggers summarization with a 120-second timeout and 50% context budget reservation.

**Emulate this:** Set thresholds that give you room to act, not thresholds that trigger crisis responses.

---

## Lesson 4: Category-Based Delegation Is Smarter Than Direct Model Selection

**What they built:** Instead of asking users to pick models, the system uses categories (`visual-engineering`, `ultrabrain`, `deep`, `quick`, `writing`). Each category maps to optimized models and includes domain-specific prompt appends.

**Example:** `visual-engineering` mandates design system analysis before coding. `writing` enforces anti-AI-slop rules (no em-dashes, no filler phrases, plain words).

**The resolution chain:** `resolveCategoryExecution()` in `src/tools/delegate-task/category-resolver.ts` handles explicit category model > sisyphus-junior override > category default > system default. It builds fallback chains and detects unstable agents (Gemini, MiniMax, Kimi) for special handling.

**Emulate this:** Abstract user intent into task types, then resolve to appropriate resources. Don't make users understand provider/model internals.

---

## Lesson 5: Background Tasks Need Sophisticated Safety Systems

**What they built:** The `BackgroundManager` in `src/features/background-agent/manager.ts` manages parallel agent sessions with explicit safety systems:

- **Circuit breaker:** Tool call counting with `maxToolCalls` threshold
- **Repetitive tool detection:** Cancels if same tool called repeatedly
- **Stale task detection:** Interrupts tasks with no progress
- **Depth limits:** `maxDepth` prevents infinite subagent recursion
- **Spawn budget:** `maxDescendants` limits total subagents

**Why it matters:** Without these, background agents can consume resources indefinitely or enter infinite loops. The research found 20+ test files covering these scenarios.

**Emulate this:** Plan for failure modes before you need them. Background tasks WILL go wrong; build the circuit breakers first.

---

## Lesson 6: Multi-Agent Orchestration Requires Clear Agent Contracts

**What they built:** 11 agents with explicit roles: Sisyphus (orchestrator), Hephaestus (autonomous deep worker), Prometheus (strategic planner), Oracle (architecture/debugging), Librarian (docs search), Explore (code exploration), Atlas (todo orchestrator), Momus (plan reviewer), Metis (plan consultant), Sisyphus-Junior (category executor), and more.

**Key pattern:** Each agent has `AgentPromptMetadata` with `category`, `cost`, `triggers`, `useWhen`, `avoidWhen`. The dynamic prompt builder (`src/agents/dynamic-agent-prompt-builder.ts`) assembles prompts from modular sections based on context.

**Anti-duplication rule:** Sisyphus prompt explicitly states: "Once explore/librarian is fired, do NOT manually perform the same search." This prevents wasted work.

**Emulate this:** Define agent roles precisely. Write explicit anti-patterns. Build prompts from composable sections, not monolithic strings.

---

## Lesson 7: Simple Bun Workspaces Beat Complex Monorepo Tools

**What they use:** Native Bun workspaces (not Turborepo, Nx, or Lerna) for a monorepo with 11 platform-specific packages. Lock file is `bun.lock` (lockfileVersion 1).

**Why it works:** Bun handles the build, test, typecheck, and package management. The simplicity means less configuration overhead and faster toolchain operations.

**The structure:** Root `package.json` declares platform packages as optionalDependencies. Postinstall script (`postinstall.mjs`) handles binary setup.

**Emulate this:** Don't reach for complex tooling until you have a complex problem. Start with what's built-in.

---

## Lesson 8: Testing Isolation Prevents Subtle Bugs

**What they do:** Tests using `mock.module()` run in separate `bun test` processes to prevent module cache pollution. CI explicitly splits mock-heavy tests into isolated runs.

**Example from CI:**
```yaml
# Mock-heavy tests run first in isolation
bun test src/plugin-handlers
bun test src/hooks/atlas
# ... 11 more isolated test directories

# Remaining tests run second
bun test bin script src/config src/mcp src/index.test.ts ...
```

**Why it matters:** Module-level singletons (managers, caches, registries) persist across tests. Without isolation, tests affect each other in non-obvious ways.

**Emulate this:** If you have module-level state, isolate tests that mutate it. The extra CI time is worth not having flaky tests.

---

## Lesson 9: Zod Schema Validation Catches Config Errors Early

**What they built:** 25+ Zod schema files in `src/config/schema/` validating all user-facing configuration. The root schema (`oh-my-opencode-config.ts`) has 25+ fields with nested objects for agents, categories, hooks, tmux, runtime-fallback, etc.

**Runtime validation pattern:**
```typescript
const result = OhMyOpenCodeConfigSchema.safeParse(config)
if (!result.success) {
  // Handle validation error with detailed messages
}
```

**Emulate this:** Validate at boundaries. If config comes from user files, validate before using. Detailed error messages beat silent failures.

---

## Lesson 10: Claude Code Compatibility as a Strategy

**What they built:** A full compatibility layer (`src/plugin-handlers/`, `src/hooks/claude-code-hooks/`) that loads Claude Code hooks, commands, skills, agents, and MCPs from `~/.claude/`, `.claude/`, and Claude Code's `settings.json`.

**The merge order:** Built-in MCPs first, then user config MCPs, then Claude Code MCPs, then plugin MCPs. User-disabled MCPs are preserved even if Claude Code would enable them.

**Why it matters:** Instead of competing with Claude Code ecosystem, they absorbed it. Users can migrate without losing their existing investments.

**Emulate this:** Build compatibility layers for adjacent ecosystems. You get users "for free" when they can migrate their existing work.

---

## Lesson 11: Document Your Technical Debt

**What they acknowledge:** The research consistently found "Technical Debt / Shortcuts" sections with specific edge cases, known issues, and intentional workarounds.

**Examples:**
- "Hardcoded 5-second API timeout may be too short for large contexts"
- "No cancellation support for long-running searches"
- "Token estimation is rough: `chars / 4` is a simplification"
- "File-based state not atomic (potential race conditions)"

**Why it matters:** They know what they don't know. Future maintainers (including future self) can see the pressure points.

**Emulate this:** Maintain an honest inventory of known issues. It guides refactoring and prevents re-introducing solved problems.

---

## Lesson 12: No Automated Linting Is a Real Gap

**What they don't have:** No ESLint, Prettier, Biome, or EditorConfig. The code quality assessment found "Code style inconsistencies possible" due to lack of automated formatting.

**What they use:** Single quotes, semicolons, 2-space indentation, K&R braces - but enforced by convention only, not tooling.

**The impact:** CI runs `tsc --noEmit` and `bun test`, but not `eslint` or `prettier --check`. Style drift is possible.

**Emulate this:** Add linting and formatting tooling early. The friction is minimal; the consistency gains are significant. This is the most actionable lesson - just run `bun add -d eslint prettier` and configure them.

---

## Summary: What to Emulate vs What to Avoid

| Emulate | Avoid |
|---------|-------|
| Hook-based event architecture | Complex conditional prompt string replacement |
| Hashline validation for edits | Hardcoded model string matching |
| Preemptive 78% threshold | No linting/formatting enforcement |
| Category-based delegation | Silent error swallowing |
| Circuit breakers for background tasks | Skip test isolation for speed |
| Zod schema validation at boundaries | Skip coverage reporting |
| Claude Code compatibility layer | Skip documenting technical debt |
| Simple Bun workspaces | Build everything from scratch |

---

## Key Metrics from Research

| Aspect | Value |
|--------|-------|
| Source files | 1,268 TypeScript files, ~160k LOC |
| Hooks | 48+ lifecycle hooks |
| Agents | 11 specialized agents |
| Platform packages | 11 (cross-platform with baseline variants) |
| Test isolation | 11 mock-heavy test directories |
| Config schemas | 25+ Zod schema modules |
| Releases | 60+ in v3.x line, active daily |
| Community | 43,738 stars, 3,241 forks, 64 commits/day |

---

## Final Assessment

oh-my-openagent is a **production-quality, actively maintained** project with excellent architectural decisions in hook-based extensibility, multi-agent orchestration, and developer experience. The code quality is good (strict TypeScript, Zod validation, test isolation) with one significant gap (no automated linting). The community health is excellent with fast response times and active external contributions.

**Primary recommendation:** Start with the architecture (hooks, agents, tools) and steal the category-based delegation and preemptive compaction patterns. Add linting from day one.
