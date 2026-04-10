# Feature Research: Batch 4 (Features 10-12)

## Feature 10: Deep Initialization (/init-deep)

### Overview
Generates hierarchical `AGENTS.md` files throughout the project. Running `/init-deep` creates project-wide, directory-specific, and component-specific context files. Agents auto-read relevant context automatically without manual management.

### Core Implementation Files
- `src/features/builtin-commands/templates/init-deep.ts` - Command template (306 lines)
- `src/hooks/directory-agents-injector/` - Auto-injects AGENTS.md context (9 files)

### Key Architecture

#### init-deep.ts Template
The `/init-deep` command is a sophisticated template-driven generator:

```
Workflow (High-Level):
1. Discovery + Analysis (concurrent) - Fire background explore agents immediately
2. Score & Decide - Determine AGENTS.md locations from merged findings
3. Generate - Root first, then subdirs in parallel
4. Review - Deduplicate, trim, validate
```

**Scoring Matrix:**
| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3x | >20 | bash |
| Subdir count | 2x | >5 | bash |
| Code ratio | 2x | >70% | bash |
| Module boundary | 2x | Has index.ts/__init__.py | bash |
| Symbol density | 2x | >30 symbols | LSP |
| Reference centrality | 3x | >20 refs | LSP |

**Decision Rules:**
- Root (.) - ALWAYS create
- Score >15 - Create AGENTS.md
- Score 8-15 - Create if distinct domain
- Score <8 - Skip (parent covers)

**Dynamic Agent Spawning:**
Scales explore agents based on project scale:
- >100 files: +1 agent per 100 files
- >10k lines: +1 agent per 10k lines
- Depth >=4: +2 for deep exploration
- >10 large files (>500 lines): +1 for complexity hotspots
- Monorepo detected: +1 per package/workspace
- Multiple languages: +1 per language

#### directory-agents-injector Hook
Automatically injects AGENTS.md content when files are read:

```
tool.execute.after (for "read" tool) -> processFilePathForAgentsInjection()
```

**Key flow:**
1. `findAgentsMdUp()` walks up from the read file's directory toward root
2. For each AGENTS.md found (excluding root - already loaded by OpenCode's system.ts)
3. Content is truncated via `dynamicTruncator` to save context
4. Appends to tool output: `[Directory Context: {path}]\n{content}`
5. Caches injected paths per session to avoid duplicate injection

**Finder logic** (`finder.ts`):
- Walks up from startDir to rootDir
- Skips root AGENTS.md (loaded via `custom()` per issue #379)
- Returns paths in reverse order (closest to file first, root last)
- `resolveFilePath()` handles absolute/relative path resolution

**Clever solution:** Root AGENTS.md is intentionally skipped because OpenCode's system.ts already loads it via `custom()` - the hook only handles subdirectory AGENTS.md files.

### Technical Debt/Issues
1. **Issue #379**: Root AGENTS.md was being injected twice before the fix - now explicitly skipped
2. **Silent failures**: `processFilePathForAgentsInjection` catches errors silently with empty catch block
3. **Session cache persistence**: Uses file-based storage (`storage.ts`) which could fail mid-session

### Edge Cases
- No AGENTS.md files in project: Hook silently does nothing
- Very large AGENTS.md files: Truncated via dynamicTruncator to preserve context window
- Session deleted/compacted: Cache is cleared via event handlers

---

## Feature 11: Prometheus Strategic Planner

### Overview
An interview-mode strategic planner. Before touching code, Prometheus questions the user, identifies scope and ambiguities, and builds a verified plan. The `/start-work` command invokes Prometheus, and Atlas executes the resulting plan systematically. Metis acts as a pre-planning consultant identifying hidden intentions and AI failure points.

### Core Implementation Files
- `src/agents/prometheus/` - Prometheus planning agent (12 files)
  - `index.ts` - Exports
  - `system-prompt.ts` - Main prompt assembly (combines all sections)
  - `identity-constraints.ts` - Core identity (336 lines)
  - `interview-mode.ts` - Phase 1 interview strategies (336 lines)
  - `plan-generation.ts` - Phase 2 plan generation (214 lines)
  - `high-accuracy-mode.ts` - Momus review workflow
  - `plan-template.ts` - Work plan template structure
  - `behavioral-summary.ts` - Agent behavioral guidelines
  - `gpt.ts` - GPT-specific prompt optimizations
  - `gemini.ts` - Gemini-specific prompt optimizations
- `src/agents/metis.ts` - Pre-planning consultant agent (337 lines)
- `src/hooks/prometheus-md-only/` - Enforces markdown-only output (5 files)
- `src/hooks/start-work/` - `/start-work` command handling (8 files)
- `src/features/builtin-commands/templates/start-work.ts` - Start-work template

### Prometheus Architecture

#### Two-Phase Workflow

**Phase 1: Interview Mode (DEFAULT)**
- Intent classification first (Trivial, Refactoring, Build from Scratch, Mid-sized, Collaborative, Architecture, Research)
- Different interview strategies per intent type
- Research agents fire in parallel (explore, librarian)
- Draft file continuously updated at `.sisyphus/drafts/{name}.md`
- Clearance check after every turn to auto-transition to Phase 2

**Phase 2: Plan Generation (AUTO-TRANSITION)**
Triggered when ALL clearance checklist items pass:
- Core objective clearly defined
- Scope boundaries established (IN/OUT)
- No critical ambiguities remaining
- Technical approach decided
- Test strategy confirmed
- No blocking questions outstanding

#### Model-Specific Prompts
The system selects optimized prompts based on model:
- **Claude (default)**: Modular sections (identity + interview + plan-gen + high-accuracy + template + behavioral)
- **GPT**: XML-tagged, principle-driven format (from `gpt.ts`)
- **Gemini**: Aggressive tool-call enforcement, thinking checkpoints (from `gemini.ts`)

#### Critical Constraints (from identity-constraints.ts)

1. **INTERVIEW MODE BY DEFAULT** - Consultant first, planner second
2. **AUTOMATIC PLAN GENERATION** - Self-clearance check after every turn
3. **MARKDOWN-ONLY FILE ACCESS** - Enforced by `prometheus-md-only` hook
4. **STRICT PATH ENFORCEMENT** - Only `.sisyphus/plans/*.md` and `.sisyphus/drafts/*.md`
5. **MAXIMUM PARALLELISM PRINCIPLE** - 5-8 tasks per wave, one task = one module
6. **SINGLE PLAN MANDATE** - Everything in ONE plan, never split
7. **INCREMENTAL WRITE PROTOCOL** - Write skeleton + Edit batches to avoid output limits
8. **DRAFT AS WORKING MEMORY** - Update after every meaningful exchange

#### prometheus-md-only Hook
Enforces markdown-only output and restricted file paths:

**Blocked tools:** `edit`, `write`, `apply_patch` for non-.md files

**Allowed paths ONLY:**
- `.sisyphus/plans/{name}.md`
- `.sisyphus/drafts/{name}.md`

**Blocked paths:**
- `docs/` - Wrong directory
- `plan/` or `plans/` - Wrong directory
- Any path outside `.sisyphus/`

When blocked tool used on allowed path (`.sisyphus/plans/`), injects workflow reminder message.

#### Metis Pre-Planning Consultant
Named after the Greek goddess of wisdom. Called by Prometheus BEFORE generating plans.

**Key responsibilities:**
- Identify hidden intentions and unstated requirements
- Detect ambiguities that could derail implementation
- Flag AI-slop patterns (over-engineering, scope creep)
- Generate clarifying questions for the user
- Prepare directives for the planner agent

**Intent-specific analysis** (same 7 types as Prometheus):
- Refactoring: Safety focus, regression prevention
- Build from Scratch: Discovery focus, pattern exploration first
- Mid-sized Task: Guardrails, exact boundaries
- Collaborative: Dialogue focus, incremental clarity
- Architecture: Strategic, Oracle recommendation
- Research: Investigation focus, exit criteria

**Output format** includes:
- Intent classification with confidence
- Pre-analysis findings
- Questions for user (priority ordered)
- Identified risks with mitigations
- Directives for Prometheus (MUST/MUST NOT/TOOL)
- QA/acceptance criteria directives (ZERO USER INTERVENTION PRINCIPLE)

### Start-Work Flow
1. Find plans at `.sisyphus/plans/`
2. Check for active boulder state at `.sisyphus/boulder.json`
3. If active plan exists and incomplete: APPEND session, continue work
4. If no active plan or complete: List available plans, auto-select if single
5. Worktree support via `--worktree <path>` flag
6. Create/update boulder.json with session tracking

### Clever Solutions

1. **Incremental Write Protocol**: Large plans would hit output limits - solution is Write skeleton + Edit batches of 2-4 tasks
2. **Model-specific prompts**: Different models get optimized prompt structures (GPT=XML, Gemini=checkpoints, Claude=modular)
3. **Single Plan Mandate**: Prevents context loss between planning sessions
4. **Draft as Working Memory**: External memory beyond context window limits
5. **Automatic Clearance Check**: Prevents premature/ill-formed plan generation

### Technical Debt/Issues
1. **Complex prompt composition**: 6 different files combined into one system prompt - any change requires understanding all 6
2. **Silent failure in Metis consultation**: If Metis task fails, plan generation continues anyway
3. **Output limit vulnerability**: Despite incremental write protocol, very large plans could still stall

---

## Feature 12: Context Window Management & Session Compaction

### Overview
Proactive session compaction that preserves critical context before hitting token limits. Includes context window monitoring, dynamic truncation of tool outputs, and a `compaction-context-injector` hook.

### Core Implementation Files
- `src/hooks/preemptive-compaction.ts` - Proactive compaction (194 lines)
- `src/hooks/context-window-monitor.ts` - Token consumption tracking (113 lines)
- `src/hooks/compaction-context-injector/` - Context preservation during compaction (16 files)
- `src/shared/dynamic-truncator.ts` - Truncation logic (223 lines)

### Context Window Monitor
Tracks token usage and warns when approaching limits.

**Key threshold:** `CONTEXT_WARNING_THRESHOLD = 0.70` (70%)

```typescript
// Tracks via message.updated events
interface CachedTokenState {
  providerID: string
  modelID: string
  tokens: {
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
}

// On tool.execute.after (if usage > 70%):
output.output += `
[Context Status: ${usedPct}% used (${usedTokens}/${limitTokens} tokens), ${remainingPct}% remaining]`
```

### Preemptive Compaction
Triggers session summarization BEFORE hitting limits.

**Key threshold:** `PREEMPTIVE_COMPACTION_THRESHOLD = 0.78` (78%)

```typescript
// Flow:
1. Track token usage via message.updated events (cached in tokenCache)
2. On tool.execute.after:
   - Check usage ratio
   - If > 78%, trigger summarize() via session API
3. Uses resolveCompactionModel() to select appropriate model for summarization
4. 120 second timeout on compaction operation
```

**Compaction model resolver:** Selects model based on plugin config and session requirements.

### Dynamic Truncator (`src/shared/dynamic-truncator.ts`)

Core truncation logic with two key functions:

#### `truncateToTokenLimit()`
- Estimates tokens via `chars / 4` (rough approximation)
- Preserves header lines (default 3)
- Truncates content lines until target tokens reached
- Adds truncation notice: `[X more lines truncated due to context window limit]`

#### `dynamicTruncate()`
- Gets actual context window usage via `getContextWindowUsage()`
- Calculates max output tokens as `min(remainingTokens * 0.5, targetMaxTokens)`
- If remaining < 0: returns `[Output suppressed - context window exhausted]`

**Truncation options:**
```typescript
interface TruncationOptions {
  targetMaxTokens?: number        // Default: 50,000
  preserveHeaderLines?: number    // Default: 3
  contextWindowLimit?: number
}
```

### Compaction Context Injector
Preserves critical context during compaction events.

**Core functions:**
1. **capture()**: Before compaction, saves agent config checkpoint
2. **inject()**: After compaction, injects recovery context

**Key hook events handled:**
- `session.compacted`: Trigger recovery and re-inject context
- `session.deleted`: Clear checkpoints
- `session.idle`: Check for no-text-tail issues
- `message.updated`: Track message changes
- `message.part.delta`: Track streaming output
- `message.part.updated`: Track completed parts

**Agent config checkpoint** (via `compaction-agent-config-checkpoint`):
```typescript
{
  agent?: string
  model?: { providerID: string; modelID: string }
  tools?: Record<string, boolean | "allow" | "deny" | "ask">
}
```

### Session Prompt Config Resolver
Resolves agent/model/tools from session message history:

```typescript
// Walks messages from newest to oldest
// First non-compaction agent wins
// First model/provider wins
// First tools definition wins
// Stops early if all three found
```

**Fallback chain:**
1. Try to resolve from session messages
2. Fall back to stored session model state
3. Return partial config if some fields missing

### Clever Solutions

1. **Preemptive not reactive**: Compacts BEFORE hitting limit, not when already at limit
2. **Degradation monitoring**: `preemptive-compaction-degradation-monitor` tracks if compaction causes quality degradation
3. **No-text-tail detection**: Catches sessions where assistant produces empty responses
4. **Context preservation**: Agent config checkpoint ensures compaction doesn't lose agent identity
5. **Streaming-aware**: Tracks `message.part.delta` events for real-time output monitoring

### Technical Debt/Issues

1. **Token estimation is rough**: `chars / 4` is a simplification - actual tokenizers vary
2. **Silent failures**: Many operations catch and log errors but continue silently
3. **Cache persistence**: Session caches stored in memory - lost on crash
4. **Model-specific context limits**: `resolveActualContextLimit()` requires model-specific cache - could fail for unknown models
5. **Timeout on compaction**: 120 second timeout could leave session in inconsistent state if summarize hangs

### Edge Cases
- Unknown model: Context limit resolver returns null, compaction skipped
- Empty session: No messages to extract config from
- Already compacted session: Tracked via `compactedSessions` Set to avoid re-compaction
- Streaming messages: Tracked via delta events, finalized on part.updated
