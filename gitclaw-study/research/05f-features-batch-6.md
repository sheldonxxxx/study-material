# Feature Batch 6 Deep Dive: Schedules, Learning System, Composio Integration

**Features:** 16 - Schedules & Task Runner | 17 - Learning & Skill Capture | 18 - Composio Integration
**Date:** 2026-03-26
**Repo:** /Users/sheldon/Documents/claw/reference/gitclaw

---

## Feature 16: Schedules & Task Runner

### Overview

GitClaw's scheduling system allows agents to run tasks automatically at defined times using cron expressions or one-time `runAt` timestamps. Schedules are git-native YAML files in an agent's `schedules/` directory, and the scheduler is embedded in the voice server.

### Architecture

**Files:**
- `src/schedules.ts` (120 lines) - Schedule discovery, CRUD, YAML persistence
- `src/schedule-runner.ts` (157 lines) - Execution engine with node-cron

**Key Data Flow:**

```
schedules/*.yaml files
       │
       ▼
discoverSchedules() ──────► ScheduleDefinition[]
       │                         │
       ▼                         ▼
startScheduler()          cron.schedule() / setTimeout()
       │                         │
       ▼                         ▼
executeScheduledJob()     runPrompt() → broadcast results
       │
       ▼
updateScheduleMeta() + JSONL log
```

### Core Components

#### ScheduleDefinition Interface
```typescript
interface ScheduleDefinition {
  id: string;           // kebab-case (validated via KEBAB_RE)
  prompt: string;      // The prompt to execute
  cron: string;         // Cron expression (for repeat/once mode)
  mode: "repeat" | "once";
  runAt?: string;      // ISO datetime (alternative for once mode)
  enabled: boolean;
  createdAt: string;
  lastRunAt?: string;
  lastResult?: string;
}
```

#### YAML Persistence Layer (schedules.ts)
- Schedules stored as individual YAML files: `schedules/{id}.yaml`
- `discoverSchedules()` reads all `.yaml`/`.yml` files in `schedules/` directory
- `saveSchedule()` uses `js-yaml.dump()` with `lineWidth: 120`
- ID validation: `KEBAB_RE = /^[a-z0-9]+(-[a-z0-9]+)*$/`
- Partial metadata updates via `updateScheduleMeta()` - re-serializes entire YAML

#### Execution Engine (schedule-runner.ts)

**Three execution paths in `startScheduler()`:**

1. **Once via `runAt`** (setTimeout) - delay computed from `Date.now() - runAt.getTime()`
2. **Once via cron** (node-cron) - fires once, then `disableAfterRun=true`
3. **Repeating via cron** (node-cron) - fires repeatedly

**Concurrency guard:** `runningJobs` Set prevents overlapping executions of the same schedule ID.

**Job execution (`executeScheduledJob()`):**
1. Check if already running (skip if so)
2. Broadcast `schedule_start` message to browsers
3. Call `runPrompt(schedule.prompt)`
4. Write result to `.gitagent/schedule-logs/{id}.jsonl` (result truncated to 5000 chars)
5. Update YAML metadata (lastRunAt, lastResult, enabled=false for once mode)
6. Broadcast `schedule_result` to browsers

**Cleanup for once jobs:**
- Stops and removes from `activeTasks` or `activeTimers`
- Sets `enabled: false` in YAML

### Integration Points

**Voice server integration** (voice/server.ts):
- HTTP endpoints: `/api/schedules/list`, `/api/schedules/save`, `/api/schedules/delete`, `/api/schedules/toggle`, `/api/schedules/run`, `/api/schedules/logs`
- `schedulerOpts` passed to `startScheduler()` with `runPrompt`, `broadcastToBrowsers`, `appendToHistory`
- Scheduler auto-starts on voice server init (line 2698)
- Auto-stops on voice server shutdown (line 2702)

### Notable Patterns & Edge Cases

**Edge case handling:**
- Past `runAt` times are skipped (delay <= 0 check)
- Invalid cron expressions are logged and skipped
- Cron validation via `cron.validate()` before scheduling
- Log write and meta update failures are non-fatal (caught silently)
- Result truncation at 5000 chars for logs, 2000 chars for broadcasts

**Technical debt/quirks:**
- `updateScheduleMeta()` re-parses and re-serializes entire YAML file (no locking)
- No deduplication if same schedule file appears twice (though unlikely)
- `activeTasks` and `activeTimers` are module-level Maps (no restart resilience)
- The `runPrompt` function is passed in as an option rather than being a direct dependency

---

## Feature 17: Learning & Skill Capture

### Overview

GitClaw's learning system tracks task outcomes, adjusts skill confidence through reinforcement learning, and can capture successful task sequences as reusable skills. It consists of three tools: `task_tracker`, `skill_learner`, and `capture_photo`.

### Architecture

**Files:**
- `src/learning/reinforcement.ts` (120 lines) - Confidence adjustment math
- `src/tools/task-tracker.ts` (350 lines) - Task tracking tool
- `src/tools/skill-learner.ts` (413 lines) - Skill evaluation/crystallization
- `src/tools/capture-photo.ts` (106 lines) - Photo capture tool
- `src/tools/index.ts` (54 lines) - Tool factory

### Core Data Structures

#### TaskRecord (task-tracker.ts)
```typescript
interface TaskRecord {
  id: string;              // randomUUID
  objective: string;        // What the task aims to accomplish
  steps: TaskStep[];        // Array of {description, timestamp}
  attempts: number;         // How many times this objective was tried
  status: "active" | "succeeded" | "failed";
  outcome?: "success" | "failure" | "partial";
  failure_reason?: string;
  skill_used?: string;      // Which skill was used (if any)
  started_at: string;       // ISO timestamp
  ended_at?: string;
}
```

#### SkillStats (reinforcement.ts)
```typescript
interface SkillStats {
  confidence: number;        // 0.0–1.0
  usage_count: number;
  success_count: number;
  failure_count: number;
  negative_examples: string[]; // capped at 10
}
```

### Task Tracker Tool

**Actions:** `begin`, `update`, `end`, `list`

#### begin Action
- Checks for existing active task with same objective (increments attempts if found)
- Searches for prior failed attempts with same objective
- Parallel searches: local skills directory + SkillsMP marketplace API
- Returns matched skills with relevance scores
- **Critical:** Forces agent to use matched skill with this message:
  > "SKILL MATCH FOUND — YOU MUST USE IT: ... Load skills/{name}/SKILL.md NOW and follow its instructions. Do NOT proceed with a manual approach."

#### update Action
- Adds a timestamped step to the task's steps array
- Validates task exists and is active

#### end Action
- Sets outcome (success/failure/partial)
- **Reinforcement learning trigger:** If skill_used is provided, calls `adjustConfidence()`
- Updates task status to "succeeded" or "failed"
- Suggests calling `skill_learner evaluate` if successful

### Reinforcement Learning (reinforcement.ts)

**Confidence adjustment formula:**
- **Success:** `conf + 0.1 * (1 - conf)` (asymptotic to 1.0)
- **Failure:** `conf - 0.2` (2x penalty)
- **Partial:** `conf - 0.05`

**Asymmetric loss design:** Failures penalize more heavily than successes reward, reflecting the insight that bad outcomes are more informative than good ones.

**Negative examples:** Stored in skill frontmatter, capped at 10, used to track failure patterns.

**Rounding:** `Math.round(confidence * 100) / 100` prevents floating-point drift.

### Skill Learner Tool

**Actions:** `evaluate`, `crystallize`, `status`, `review`, `update`, `delete`

#### evaluate Action
Checks if a completed task deserves to become a reusable skill:
- **multi_step:** At least 3 steps
- **non_trivial:** At least 2 steps
- **novel:** No existing skill with >0.5 Jaccard similarity
- **generalizable:** <30% of steps are project-specific (absolute paths, UUIDs, PascalCase classes)

**Decision:** Pass if >=3 checks pass OR (multi_step AND novel)

#### crystallize Action
Creates a new skill from a successful task:
1. Validates skill_name is kebab-case
2. Builds SKILL.md with YAML frontmatter + steps + "What Worked" + "What Did NOT Work" sections
3. Writes to `skills/{skill_name}/SKILL.md`
4. Git commits

**Skill template:**
```yaml
---
name: {skill_name}
description: {skill_description}
learned_from: task:{task_id}
learned_at: {ISO timestamp}
confidence: 1.0
usage_count: 0
success_count: 0
failure_count: 0
negative_examples: []
---
## Steps
1. {step 1 description}
2. {step 2 description}
...

## What Worked
This approach succeeded on attempt #{n}.

## What Did NOT Work
- {prior failure reason 1}
- {prior failure reason 2}
```

#### status Action
Lists all skills with confidence, usage count, and success ratio.

#### review Action
Shows skills with confidence < 0.4 (flagged as unreliable).

### Photo Capture Tool

**Purpose:** Capture memorable moments from webcam during agent sessions.

**Behavior:**
1. Reads latest frame from `memory/.latest-frame.jpg` (written by voice server)
2. Validates frame age < 5 seconds (stale check)
3. Saves photo as `memory/photos/{date}_{time}_{slug}.jpg`
4. Updates `memory/photos/INDEX.md` with markdown entry
5. Git adds and commits both files

**Slug generation:** Lowercase, alphanumeric/hyphens only, max 40 chars.

### Skill Search

**Local search** (`searchLocalSkills()`):
- Reads `skills/*/SKILL.md` frontmatter
- Extracts name + description keywords
- Uses keyword overlap: `matches / max(a.length, b.length)`
- Returns matches with relevance > 0.1

**Marketplace search** (`searchSkillsMP()`):
- Calls `https://api.skillsmp.com/v1/search` with Bearer token
- 5-second timeout
- Returns marketplace skills with relevance scores

### Integration

**In tools/index.ts (`createBuiltinTools()`):**
```typescript
if (config.gitagentDir) {
  tools.push(createTaskTrackerTool(config.dir, config.gitagentDir));
  tools.push(createSkillLearnerTool(config.dir, config.gitagentDir));
}
// capture_photo is always added
tools.push(createCapturePhotoTool(config.dir));
```

**Persistence location:** `.gitagent/learning/tasks.json`

### Notable Patterns & Edge Cases

**Project-specific detection (isProjectSpecific):**
- Absolute paths: `/\/[a-zA-Z][\w/.-]{5,}/`
- UUIDs: `/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/i`
- PascalCase classes (3+ parts): `/[A-Z][a-z]+(?:[A-Z][a-z]+){2,}/`

**Task retry behavior:** Same objective resumes existing task, increments attempts. Prior failure reasons are surfaced in the response to the agent.

**Non-fatal git commits:** If git commit fails for skill crystallize or photo capture, the file is still written - just not committed.

---

## Feature 18: Composio Integration

### Overview

GitClaw integrates with Composio (a tool action platform) to give agents access to 100+ external toolkits (Gmail, Slack, GitHub, Notion, etc.). The integration consists of a REST API client and an adapter that converts Composio tools into GitClaw's tool format.

### Architecture

**Files:**
- `src/composio/client.ts` (243 lines) - REST API v3 client
- `src/composio/adapter.ts` (120 lines) - GitClaw tool adapter
- `src/composio/index.ts` (2 lines) - Re-exports

### ComposioClient (client.ts)

**Zero-dependency design:** Uses native `fetch()` - no axios, no node-fetch.

**Base URL:** `https://backend.composio.dev/api/v3`

**Key interfaces:**
```typescript
interface ComposioToolkit {
  slug: string;
  name: string;
  description: string;
  logo: string;
  authSchemes: string[];
  noAuth: boolean;
  connected: boolean;
}

interface ComposioConnection {
  id: string;
  toolkitSlug: string;
  status: string;
  createdAt: string;
}

interface ComposioTool {
  name: string;
  slug: string;
  description: string;
  toolkitSlug: string;
  parameters: Record<string, any>;
}
```

**API Methods:**
- `listToolkits(userId?)` - Available toolkits with connection status
- `searchTools(query, toolkitSlugs?, limit?)` - Semantic tool search
- `listTools(toolkitSlug)` - All tools in a toolkit
- `getOrCreateAuthConfig(toolkitSlug)` - Get/create OAuth2 auth config
- `initiateConnection(toolkitSlug, userId, redirectUrl?)` - Start OAuth flow
- `listConnections(userId)` - User's active connections
- `deleteConnection(id)` - Remove a connection
- `executeTool(toolSlug, userId, params, connectedAccountId?)` - Execute action

**Response handling:** Defensive array extraction from varied response shapes:
```typescript
const items: any[] = Array.isArray(resp) ? resp : (resp.items ?? resp.toolkits ?? resp.tools ?? resp.connections ?? []);
```

**Auth config caching:** `authConfigCache` Map avoids recreating auth configs on every connection.

**Error handling:** Non-2xx responses throw with method, path, status, and response text.

### ComposioAdapter (adapter.ts)

**Purpose:** Wraps `ComposioClient` and converts tools to GitClaw's `GCToolDefinition` format.

**Caching:** 30-second TTL cache for `getTools()` results (`cachedTools`, `cacheExpiry`).

**Tool name generation:**
```typescript
const safeName = `composio_${t.toolkitSlug}_${t.slug}`.replace(/[^a-zA-Z0-9_]/g, "_");
// Example: composio_gmail_SEND_EMAIL
```

**Email tool hints:** Special descriptions for email tools:
- `SEND_EMAIL`: "— USE THIS to send emails directly."
- `CREATE_EMAIL_DRAFT`: "— Only use when the user explicitly asks for a draft."

**Key methods:**
- `getTools()` - All tools from connected toolkits (cached)
- `getToolsForQuery(query, limit?)` - Semantic search, sorted with drafts last
- `getConnectedToolkitSlugs()` - Deduplicated list
- `getToolkits()` - All available toolkits with connection status
- `connect(toolkit, redirectUrl?)` - Initiate OAuth
- `getConnections()` - Current connections
- `disconnect(connectionId)` - Remove connection, invalidate cache

**Tool execution flow:**
```typescript
handler: async (args: any) => {
  const result = await this.client.executeTool(t.slug, this.userId, args);
  return typeof result === "string" ? result : JSON.stringify(result);
}
```

### Tool Search Behavior

**Parallel per-toolkit search:** When toolkit slugs are specified, `searchTools()` makes parallel requests to each toolkit (since API doesn't support comma-separated slugs with query).

**Draft sorting:** Tools with "DRAFT" in their slug are sorted last, prioritizing direct-action tools.

### Integration Pattern

The adapter is used in the voice server or SDK to inject tools into the agent. It acts as a bridge between Composio's external tool ecosystem and GitClaw's internal tool interface.

### Notable Patterns & Edge Cases

**Connection failure tolerance:** In `listToolkits()`, if `listConnections()` fails (e.g., user has no connections), it silently falls back to showing all toolkits as disconnected rather than throwing.

**Auth config race condition:** Multiple concurrent `getOrCreateAuthConfig()` calls for the same toolkit could create duplicates (no distributed locking). The cache mitigates this for repeated calls.

**API key exposure:** The Composio API key is passed to the client constructor and sent as `x-api-key` header with every request.

---

## Cross-Feature Observations

### Shared Patterns

1. **Git-native persistence:** Schedules and skills both use git-committed files (YAML, JSON, markdown)
2. **YAML frontmatter:** Both schedules (skill-like metadata) and skills use js-yaml for parsing/serialization
3. **Graceful degradation:** Most operations have non-fatal failures (log writes, git commits, API calls)
4. **UUID-based IDs:** Task IDs use `randomUUID()`, schedule IDs use kebab-case slugs

### Tool Factory Pattern

All three learning tools use the factory pattern: `create{ToolName}Tool(config)` returns an `AgentTool<Schema>` object. This allows lazy instantiation and dependency injection.

### Concurrency Handling

- Schedules: `runningJobs` Set prevents concurrent execution
- Task tracker: No concurrency protection (assumed single-threaded execution)
- Composio: No concurrency protection at the adapter level

### Configuration Dependencies

- Schedules need `agentDir` (where `schedules/` directory lives)
- Task tracker needs both `agentDir` (for skills/) and `gitagentDir` (for `.gitagent/learning/`)
- Composio adapter needs `apiKey` and optionally `userId`
- Capture photo needs `cwd` for file paths

---

## Maturity Assessment

| Feature | Files | Lines | Status | Notes |
|---------|-------|-------|--------|-------|
| Schedules | 2 | ~277 | Secondary | Functional, lacks restart resilience |
| Learning System | 4 | ~989 | Secondary | Complex, well-thought-out RL |
| Composio | 3 | ~365 | Secondary | Clean client, OAuth race condition |

### Recommendations for Future Investigation

1. **Schedules:** Consider adding restart resilience for activeTasks/activeTimers Maps
2. **Learning:** The SkillsMP marketplace API integration is optional (env var gated) - worth exploring if there's a broader marketplace strategy
3. **Composio:** Auth config race condition could be addressed with a mutex or server-side deduplication
