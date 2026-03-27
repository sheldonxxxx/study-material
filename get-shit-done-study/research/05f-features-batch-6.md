# Batch 6 Feature Analysis: Features 16-17

## Feature 16: Multi-Project Workspaces

### Feature Name and Description

**Multi-Project Workspaces** (secondary) — Isolated workspaces with repo copies (worktrees or clones) for parallel milestone work, plus workstream management for namespaced parallel tracks.

Two related but distinct sub-features:
1. **Physical Workspaces** (`/gsd:new-workspace`, `/gsd:list-workspaces`, `/gsd:remove-workspace`) — Creates isolated directory structures with actual git copies
2. **Workstreams** (`/gsd:workstreams`) — Virtual milestone namespaces within a single repo

### Core Implementation Files

#### 1. `commands/gsd/new-workspace.md` — Workspace Creation Command
Thin command dispatcher that delegates to the actual workflow at `~/.claude/get-shit-done/workflows/new-workspace.md`:

```markdown
<execution_context>
@~/.claude/get-shit-done/workflows/new-workspace.md
</execution_context>

<process>
Execute the new-workspace workflow from @~/.claude/get-shit-done/workflows/new-workspace.md end-to-end.
```

**Arguments:**
- `--name` (required) — Workspace name
- `--repos` — Comma-separated repo paths/names
- `--path` — Target directory (defaults to `~/gsd-workspaces/<name>`)
- `--strategy` — `worktree` (default) or `clone`
- `--branch` — Branch name (defaults to `workspace/<name>`)
- `--auto` — Skip interactive questions

#### 2. `commands/gsd/workstreams.md` — Workstream Management Command
Thin CLI wrapper around `gsd-tools workstream` subcommands:

```bash
# list — list all workstreams with status
Run: node "$GSD_TOOLS" workstream list --raw --cwd "$CWD"

# create — create new workstream
Run: node "$GSD_TOOLS" workstream create <name> --raw --cwd "$CWD"

# switch — set active workstream
Run: node "$GSD_TOOLS" workstream set <name> --raw --cwd "$CWD"

# status — detailed status for one workstream
Run: node "$GSD_TOOLS" workstream status <name> --raw --cwd "$CWD"

# progress — progress summary across all workstreams
Run: node "$GSD_TOOLS" workstream progress --raw --cwd "$CWD"

# complete — archive completed workstream
Run: node "$GSD_TOOLS" workstream complete <name> --raw --cwd "$CWD"

# resume — resume work in a workstream
```

#### 3. `get-shit-done/bin/lib/workstream.cjs` — Workstream Core Library (491 lines)
The actual workstream CRUD implementation.

**Key exports:**

```javascript
module.exports = {
  migrateToWorkstreams,      // Migrate flat .planning/ to workstream mode
  cmdWorkstreamCreate,        // Create new workstream
  cmdWorkstreamList,          // List all workstreams
  cmdWorkstreamStatus,        // Detailed status
  cmdWorkstreamComplete,      // Archive to milestones/
  cmdWorkstreamSet,           // Set active workstream
  cmdWorkstreamGet,           // Get active workstream
  cmdWorkstreamProgress,       // Progress summary
  getOtherActiveWorkstreams,  // Collision detection
};
```

**Directory Structure** (from `workstream-flag.md`):

```
.planning/
├── PROJECT.md          # Shared
├── config.json         # Shared
├── milestones/         # Shared
├── codebase/           # Shared
├── active-workstream   # Points to current ws
└── workstreams/
    ├── feature-a/      # Workstream A
    │   ├── STATE.md
    │   ├── ROADMAP.md
    │   ├── REQUIREMENTS.md
    │   └── phases/
    └── feature-b/      # Workstream B
        ├── STATE.md
        ├── ROADMAP.md
        ├── REQUIREMENTS.md
        └── phases/
```

**Workstream Active Detection** (`cmdWorkstreamGet`):
```javascript
function cmdWorkstreamGet(cwd, raw) {
  const active = getActiveWorkstream(cwd);
  const wsRoot = path.join(planningRoot(cwd), 'workstreams');
  output({ active, mode: fs.existsSync(wsRoot) ? 'workstream' : 'flat' }, raw, active || 'none');
}
```

**Migration Logic** (`migrateToWorkstreams`):
```javascript
function migrateToWorkstreams(cwd, workstreamName) {
  // Validates name has no path separators
  if (!workstreamName || /[/\\]/.test(workstreamName) || workstreamName === '.' || workstreamName === '..') {
    throw new Error('Invalid workstream name for migration');
  }

  // Moves these from flat .planning/ to .planning/workstreams/{name}/
  const toMove = [
    { name: 'ROADMAP.md', type: 'file' },
    { name: 'STATE.md', type: 'file' },
    { name: 'REQUIREMENTS.md', type: 'file' },
    { name: 'phases', type: 'dir' },
  ];

  // Shared files stay in place: PROJECT.md, config.json, milestones/, research/, codebase/
}
```

**Collision Detection** (`getOtherActiveWorkstreams`):
Used when a workstream finishes to check if other workstreams are still active:
```javascript
function getOtherActiveWorkstreams(cwd, excludeWs) {
  // ...
  // Excludes workstreams with status containing "milestone complete" or "archived"
  if (status.toLowerCase().includes('milestone complete') ||
      status.toLowerCase().includes('archived')) {
    continue;
  }
}
```

#### 4. `get-shit-done/references/workstream-flag.md` — Workstream Flag Documentation
Documents the `--ws` flag resolution priority:
1. `--ws <name>` flag (explicit, highest)
2. `GSD_WORKSTREAM` env var
3. `.planning/active-workstream` file
4. `null` — flat mode

All workflow routing commands include `${GSD_WS}` which expands to `--ws <name>` or empty.

#### 5. `tests/workstream.test.cjs` — Comprehensive Test Suite (459 lines)

**Notable Test Coverage:**
- Path traversal rejection for `--ws` flag and `GSD_WORKSTREAM` env var
- Poisoned `active-workstream` file detection
- Migration from flat mode to workstream mode
- Collision detection (completed workstreams excluded from active count)
- `--ws` flag overrides `GSD_WORKSTREAM` env var

```javascript
const maliciousNames = [
  '../../etc',
  '../foo',
  'ws/../../../passwd',
  'a/b',
  'ws name with spaces',
  '..',
  '.',
  'ws..traversal',
];
```

### Flow Trace

**Creating a Physical Workspace:**

```
/gsd:new-workspace --name feature-x --repos repo1,repo2 --strategy worktree
  → reads init (new-workspace) to get workspace_base, child_repos
  → validates target path is empty, source repos exist
  → for each repo: git worktree add <path>/<repo> -b workspace/<name>
  → writes WORKSPACE.md manifest
  → initializes .planning/ directory
  → offers to run /gsd:new-project
```

**Workstream Lifecycle:**

```
/gsd:workstreams create feature-x
  → cmdWorkstreamCreate in workstream.cjs
  → if flat mode with existing work: migrateToWorkstreams
  → creates .planning/workstreams/feature-x/{STATE,ROADMAP,REQUIREMENTS,phases}
  → sets active-workstream to feature-x

/gsd:workstreams switch feature-y
  → cmdWorkstreamSet
  → setActiveWorkstream(cwd, 'feature-y')
  → GSD_WS env var set for current session
  → all subsequent GSD ops scoped to feature-y workstream

/gsd:workstreams complete feature-x
  → cmdWorkstreamComplete
  → moves to .planning/milestones/ws-feature-x-{date}
  → clears active-workstream if it was active
  → removes workstream directory
  → if no remaining workstreams: reverts to flat mode
```

### Notable Implementation Details

**Good:**
- **Migration safety**: `migrateToWorkstreams` has rollback on failure (lines 56-61)
- **Slug sanitization**: Workstream names sanitized to lowercase hyphenated (line 74: `name.toLowerCase().replace(/[^a-z0-9]+/g, '-')`)
- **Path traversal hardening**: `active-workstream` file content validated against malicious names (test at line 429-438)
- **Backward compatibility**: "flat mode" when no `workstreams/` directory exists
- **Progress calculation**: `cmdWorkstreamProgress` counts both ROADMAP.md phase matches and actual phase directories (lines 403-406)

**Potential Issues:**
- **No actual workspace workflow file in reference repo**: `commands/gsd/new-workspace.md` delegates to `~/.claude/get-shit-done/workflows/new-workspace.md` which could not be verified in the reference repo (only in home install)
- **Worktree failure handling**: If `git worktree add` fails due to existing branch, tries timestamped branch fallback — may succeed silently with unexpected branch name
- **Workstream vs Workspace confusion**: Two related concepts (physical workspace with git copies vs virtual milestone namespace) may confuse users

---

## Feature 17: Backlog, Seeds & Threads

### Feature Name and Description

**Backlog, Seeds & Threads** (secondary) — Long-term idea management:
- **Backlog**: Parking lot using 999.x phase numbering for unsequenced ideas
- **Seeds**: Forward-looking ideas with trigger conditions that auto-surface during milestone planning
- **Threads**: Persistent cross-session context for work spanning multiple sessions

### Core Implementation Files

#### 1. `commands/gsd/add-backlog.md` — Backlog Item Command (77 lines)

**Process:**
```bash
# 1. Read ROADMAP.md to find existing backlog entries
# 2. Find next backlog number using 999.x decimal
NEXT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" phase next-decimal 999 --raw)
# If no 999.x phases exist, start at 999.1

# 3. Create phase directory
SLUG=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" generate-slug "$ARGUMENTS")
mkdir -p ".planning/phases/${NEXT}-${SLUG}"
touch ".planning/phases/${NEXT}-${SLUG}/.gitkeep"

# 4. Add to ROADMAP.md under ## Backlog section
# 5. Commit
# 6. Report
```

**ROADMAP.md entry format:**
```markdown
## Backlog

### Phase {NEXT}: {description} (BACKLOG)

**Goal:** [Captured for future planning]
**Requirements:** TBD
**Plans:** 0 plans

Plans:
- [ ] TBD (promote with /gsd:review-backlog when ready)
```

**Key Design Decisions:**
- 999.x numbering keeps backlog out of active phase sequence
- No `Depends on:` field — unsequenced by definition
- Sparse numbering is fine (999.1, 999.3) — always uses next-decimal
- Phase directories created immediately so `/gsd:discuss-phase` and `/gsd:plan-phase` work on them

#### 2. `commands/gsd/review-backlog.md` — Backlog Review Command (62 lines)

**Operations per backlog item:**
- **Promote**: Move to active milestone sequence (renumber from 999.x to sequential)
- **Keep**: Leave in backlog
- **Remove**: Delete phase directory and ROADMAP entry

**Promote process:**
```bash
# Find next sequential phase number
NEW_NUM=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" phase add "${DESCRIPTION}" --raw)

# Rename directory from 999.x-slug to {new_num}-slug
# Move accumulated artifacts to new phase directory
# Update ROADMAP.md: move entry from ## Backlog to active list
# Remove (BACKLOG) marker
# Add Depends on field
```

#### 3. `commands/gsd/plant-seed.md` — Seed Planting Command (28 lines)
Thin wrapper delegating to `~/.claude/get-shit-done/workflows/plant-seed.md`:

```markdown
<execution_context>
@~/.claude/get-shit-done/workflows/plant-seed.md
</execution_context>

<process>
Execute the plant-seed workflow from @~/.claude/get-shit-done/workflows/plant-seed.md end-to-end.
</process>
```

#### 4. `get-shit-done/workflows/plant-seed.md` — Seed Workflow (170 lines)

**Seed File Location**: `.planning/seeds/SEED-NNN-slug.md`

**Frontmatter:**
```yaml
---
id: SEED-{PADDED}
status: dormant
planted: {ISO date}
planted_during: {current milestone/phase from STATE.md}
trigger_when: {$TRIGGER}
scope: {$SCOPE}
---
```

**Workflow Steps:**
1. Parse idea from `$ARGUMENTS` (or ask user)
2. Ask for trigger condition ("When should this surface?")
3. Ask for why ("Why does this matter?")
4. Ask for scope (Small/Medium/Large)
5. Search codebase for breadcrumbs (grep for relevant keywords)
6. Check STATE.md, ROADMAP.md, todos/ for related items
7. Generate seed ID (SEED-001, SEED-002, etc.)
8. Write seed file with structured format

**Seed File Format:**
```markdown
# SEED-{PADDED}: {idea}

## Why This Matters
{WHY}

## When to Surface
**Trigger:** {TRIGGER}

This seed should be presented during `/gsd:new-milestone` when the milestone
scope matches any of these conditions:
- {trigger condition 1}
- {trigger condition 2}

## Scope Estimate
**{SCOPE}** — {elaboration}

## Breadcrumbs
{list of related files with paths}

## Notes
{additional context}
```

**Success Criteria:**
- Seed file created in `.planning/seeds/`
- Frontmatter includes status, trigger, scope
- Breadcrumbs collected from codebase
- Committed to git
- User shown confirmation with trigger info

#### 5. `commands/gsd/thread.md` — Thread Management Command (128 lines)

**Three modes of operation:**

**List mode** (no arguments):
```bash
ls .planning/threads/*.md 2>/dev/null
```
Displays:
```
## Active Threads

| Thread | Status | Last Updated |
|--------|--------|-------------|
| fix-deploy-key-auth | OPEN | 2026-03-15 |
| pasta-tcp-timeout | RESOLVED | 2026-03-12 |
| perf-investigation | IN PROGRESS | 2026-03-17 |
```

**Resume mode** (`$ARGUMENTS` matches existing thread file):
- Loads thread context into session
- Updates status to `IN PROGRESS` if was `OPEN`

**Create mode** (`$ARGUMENTS` is new description):
```bash
SLUG=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" generate-slug "$ARGUMENTS")
mkdir -p .planning/threads
```

**Thread File Format:**
```markdown
# Thread: {description}

## Status: OPEN

## Goal
{description}

## Context
*Created from conversation on {today's date}.*

## References
- *(add links, file paths, or issue numbers)*

## Next Steps
- *(what the next session should do first)*
```

**Key Design Decisions:**
- Threads are NOT phase-scoped — exist independently of roadmap
- Lighter weight than `/gsd:pause-work` — no phase state, no plan context
- Can be promoted to phases or backlog: `/gsd:add-phase` or `/gsd:add-backlog`
- Thread files live in `.planning/threads/` — no collision with phases

### Flow Trace

**Adding a Backlog Item:**
```
/gsd:add-backlog "Consider adding dark mode support"
  → reads ROADMAP.md
  → calculates next 999.x number
  → creates .planning/phases/999.1-consider-adding-dark-mode/
  → updates ROADMAP.md with backlog entry
  → commits
  → user can later: /gsd:discuss-phase 999.1 or /gsd:review-backlog
```

**Reviewing and Promoting Backlog:**
```
/gsd:review-backlog
  → lists all 999.x items with accumulated context
  → user selects Promote/Keep/Remove per item
  → Promote: renumbers to sequential, moves to active roadmap
  → Remove: deletes directory and entry
  → commits changes
```

**Planting a Seed:**
```
/gsd:plant-seed "Offline-first sync"
  → asks: When should this surface? → "when we add data persistence"
  → asks: Why does this matter? → "users expect data to survive refresh"
  → asks: How big? → Small/Medium/Large
  → searches codebase for related files
  → creates .planning/seeds/SEED-001-offline-first-sync.md
  → commits
  → during /gsd:new-milestone: seeds with matching triggers are presented
```

**Managing a Thread:**
```
/gsd:thread perf-investigation
  → if exists: loads thread context, sets IN PROGRESS
  → if new: creates .planning/threads/perf-investigation.md
  → user can add context, references, next steps
  → resume later: /gsd:thread perf-investigation

/gsd:thread
  → lists all threads with status and last updated date
```

### Notable Implementation Details

**Good:**
- **Seed trigger system**: Seeds don't just get lost in backlog — they have explicit trigger conditions that `/gsd:new-milestone` scans for
- **Breadcrumb collection**: Seeds proactively search the codebase for related files, preserving context that would otherwise be lost
- **Thread lightweightness**: Deliberately avoids phase state, making it truly session-independent
- **Backlog numbering**: Using 999.x is clever — stays out of any reasonable phase numbering scheme
- **Status tracking**: Threads have OPEN → IN PROGRESS → RESOLVED lifecycle

**Potential Issues / Technical Debt:**
- **Seed consumption not verifiable**: The `/gsd:new-milestone` workflow (in home directory) was not fully verified — seed scanning logic could not be confirmed in the reference repo
- **No seed aging**: Seeds remain `dormant` forever unless manually removed — no automatic stale seed detection
- **Thread promotion is manual**: No `/gsd:promote-thread` command — user must manually extract context and use `/gsd:add-phase` or `/gsd:add-backlog`
- **Breadcrumb grep limitation**: Uses simple `grep -rl "$KEYWORD"` which may return noisy results for common terms
- **Backlog review is all-or-nothing**: `/gsd:review-backlog` presents all items — no partial review option (review 999.1, skip 999.3-999.7)

### Shared Patterns Between Features

Both features use:
- **`gsd-tools.cjs` for slug generation**: `generate-slug` for consistent naming
- **Phase directory conventions**: Consistent with other GSD phases
- **Git commit per operation**: Each action produces an atomic commit
- **Structured frontmatter**: Both seeds and backlog items use YAML frontmatter for metadata
- **`.planning/` subdirectories**: Backlog (phases/999.x), Seeds (seeds/), Threads (threads/) — all namespace-separated

---

## Summary

| Aspect | Feature 16: Multi-Project Workspaces | Feature 17: Backlog, Seeds & Threads |
|--------|---------------------------------------|--------------------------------------|
| **Type** | Secondary | Secondary |
| **Complexity** | Medium (physical + virtual isolation) | Low-Medium (file-based management) |
| **Entry Points** | 3 commands + library | 4 commands |
| **Core Library** | `workstream.cjs` (491 lines) | No dedicated library — command-only |
| **Test Coverage** | `workstream.test.cjs` (459 lines) | Not verified |
| **Key Innovation** | Git worktree integration + flat-to-workstream migration | Trigger-based seed surfacing |
| **Security Hardening** | Path traversal rejection in workstream names | Slug-based file naming limits traversal risk |
| **Technical Debt** | Worktree failure fallback creates unexpected branches | Seed consumption logic not in reference repo |
