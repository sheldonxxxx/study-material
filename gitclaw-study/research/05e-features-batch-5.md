# Feature Batch 5: Workflows, Agent Inheritance, Knowledge Base

**Project:** GitClaw
**Date:** 2026-03-26
**Features Covered:** 13 (Workflows), 14 (Agent Inheritance & Composition), 15 (Knowledge Base)
**Researcher:** Claude Code

---

## Feature 13: Workflows

### Core Implementation
**File:** `src/workflows.ts` (167 lines)

#### Description
Multi-step workflow definitions for complex agent tasks. Workflows chain tools and skills together through a declarative YAML format.

#### Key Components

**1. Workflow Discovery (`discoverWorkflows`)**
- Scans `workflows/` directory within agent directory
- Supports two formats:
  - **YAML** (`.yaml`, `.yml`): Full workflow definitions with steps
  - **Markdown** (`.md`): Simple workflow descriptions via frontmatter

```typescript
interface WorkflowMetadata {
  name: string;
  description: string;
  filePath: string;
  format: "yaml" | "markdown";
  type?: "flow" | "basic";
  steps?: SkillFlowStep[];
}
```

**2. Flow Definition (`SkillFlowDefinition`)**
```typescript
interface SkillFlowStep {
  skill: string;
  prompt: string;
  channel?: string;
}

interface SkillFlowDefinition {
  name: string;
  description: string;
  steps: SkillFlowStep[];
}
```

**3. Flow Persistence**
- `loadFlowDefinition(filePath)`: Loads and validates YAML flow
- `saveFlowDefinition(agentDir, flow)`: Persists new flows
- `deleteFlowDefinition(agentDir, name)`: Removes flows
- **Validation:** Flow names must be kebab-case, must have at least one step

#### Flow Analysis

**Discovery Pattern (lines 36-98):**
```
discoverWorkflows(agentDir)
├── check workflows/ is directory (stat)
├── readdir entries
├── for each entry:
│   ├── if .yaml/.yml:
│   │   ├── parse YAML
│   │   ├── if has name + steps → type="flow"
│   │   └── else → type="basic"
│   └── if .md:
│       └── parse frontmatter
│           └── if has description → include
└── sort by name, return
```

**YAML Flow Structure Example:**
```yaml
name: code-review-flow
description: Multi-step code review process
steps:
  - skill: code-review
    prompt: "Review PR for security issues"
  - skill: code-review
    prompt: "Check for performance problems"
```

**Markdown Flow Structure:**
```markdown
---
name: simple-workflow
description: A simple workflow for doing X
---
```

#### Clever Solutions
1. **Dual-format support**: Both YAML (full features) and Markdown (simple descriptions)
2. **Frontmatter parsing**: Custom `parseFrontmatter()` handles both `-` and `\r\n` line endings
3. **Sorted output**: Always returns alphabetically sorted workflows for consistent prompt injection
4. **Graceful degradation**: Skips invalid YAML/MD files silently instead of throwing

#### Edge Cases & Error Handling
- Missing `workflows/` directory → returns empty array
- Invalid YAML → skipped silently
- Missing `name` in YAML → skipped
- Missing `description` in Markdown frontmatter → skipped
- Empty `steps` array → treated as "basic" workflow type

#### Technical Debt / Concerns
1. **No execution engine**: The file defines workflows but there's NO actual executor. Workflows are discovered and formatted for prompts, but the agent itself must interpret and execute them
2. **No flow validation at discovery**: `discoverWorkflows` doesn't validate step structure, only `loadFlowDefinition` does
3. **No built-in workflow commands**: No CLI commands to list/run/delete workflows (unlike skills which have skill-learner)
4. **No flow state persistence**: Running a flow has no state tracking between steps
5. **Silent failures**: Clone failures and invalid files fail silently (`2>/dev/null || true`)

#### Integration Points
- `loader.ts` line 291-293: Discovers workflows during agent loading
- `formatWorkflowsForPrompt()`: Converts discovered workflows to prompt injection
- Skills system is referenced (workflows chain skills together)

---

## Feature 14: Agent Inheritance & Composition

### Core Implementation
**Files:**
- `src/loader.ts` (382 lines) - Inheritance resolution
- `src/agents.ts` (97 lines) - Sub-agent discovery

#### Sub-Agent Discovery (`src/agents.ts`)

**Two Sub-Agent Forms:**

1. **Directory Form:** `agents/<name>/agent.yaml`
   - Full agent manifest with name/description
   - Discovered via reading `agent.yaml` inside directory

2. **File Form:** `agents/<name>.md`
   - Markdown file with frontmatter containing name/description
   - Simpler, instruction-only sub-agent

```typescript
interface SubAgentMetadata {
  name: string;
  description: string;
  type: "directory" | "file";
  path: string;
}
```

**Discovery Pattern (lines 21-76):**
```
discoverSubAgents(agentDir)
├── check agents/ is directory
├── readdir withFileTypes
├── for each entry:
│   ├── if directory:
│   │   ├── read agents/<name>/agent.yaml
│   │   └── if has name + description → include
│   └── if .md file:
│       ├── parse frontmatter
│       └── if has description → include
└── sort by name, return
```

**Delegation Format in Prompt:**
```xml
<agent>
<name>code-reviewer</name>
<description>Reviews code for bugs and security</description>
<type>directory</type>
<path>agents/code-reviewer</path>
</agent>

To delegate: gitclaw --dir agents/code-reviewer -p "task description"
```

#### Inheritance Resolution (`src/loader.ts` lines 144-192)

**AgentManifest Extension Support:**
```yaml
extends: "https://github.com/org/base-agent.git"
```

**Resolution Process:**
```
resolveInheritance(manifest, agentDir, gitagentDir)
├── if no extends → return original manifest
├── mkdir .gitagent/deps/
├── git clone --depth 1 parent.git → .gitagent/deps/parent-name
├── read parent agent.yaml
├── deepMerge(parent, child) → child wins
├── union tools arrays (child shadows duplicates)
├── read parent RULES.md
└── return { merged manifest, parentRules }
```

**Deep Merge Strategy (lines 126-142):**
```typescript
function deepMerge(base, override):
  for each key in override:
    if both are non-array objects → recursive merge
    else → override wins
```

**Tools Union Logic (lines 182-186):**
```typescript
if (parentManifest.tools && manifest.tools) {
  const toolSet = new Set([...parentManifest.tools, ...manifest.tools]);
  merged.tools = [...toolSet];  // union, child shadows
}
```

#### Dependencies Resolution (`src/loader.ts` lines 194-215)

```yaml
dependencies:
  - name: shared-tools
    source: "https://github.com/org/shared-tools.git"
    version: main
    mount: tools
```

**Process:**
```
resolveDependencies(manifest, agentDir, gitagentDir)
├── if no dependencies → return
├── mkdir .gitagent/deps/
└── for each dep:
    └── git clone --depth 1 --branch <version> <source> <depDir>
```

#### Clever Solutions
1. **Git URL inheritance**: Agents can extend any git repository, not just local paths
2. **Deep merge with precedence**: Child manifest completely overrides parent except for explicit unions
3. **Tool union**: Both parent and child tools are available, no override
4. **Parent rules appending**: RULES.md is concatenated, not merged (union behavior)
5. **Graceful clone failures**: `|| true` prevents single failed clone from breaking entire agent load

#### Edge Cases & Error Handling
- No `extends` field → inheritance skipped
- Clone fails → continues with empty parentRules
- Parent has no agent.yaml → uses empty manifest defaults
- Dependency clone fails → skipped silently
- Invalid parent YAML → treated as empty parent
- Circular inheritance → possible infinite clone loop (no detection)

#### Technical Debt / Concerns
1. **No sub-agent execution**: `discoverSubAgents` only provides metadata; actual delegation requires spawning new `gitclaw` process
2. **No dependency mounting logic**: Clones to `deps/` but no actual mount mechanism (the `mount` field in dependencies is unused)
3. **No circular dependency detection**: Could infinite loop on circular `extends`
4. **Shallow clones only**: `--depth 1` means no git history, but saves bandwidth
5. **No inheritance validation**: Invalid parent URLs fail silently
6. **Session state per agent instance**: Each cloned parent gets separate `.gitagent/` dir

#### Integration Points
- `loader.ts` line 52: `extends?: string` in manifest
- `loader.ts` line 54: `dependencies?: Array<{name, source, version, mount}>`
- `loader.ts` line 233-239: Inheritance resolution during load
- `loader.ts` line 241-242: Dependencies resolution during load
- `loader.ts` line 263: Parent rules appended to system prompt

---

## Feature 15: Knowledge Base

### Core Implementation
**File:** `src/knowledge.ts` (82 lines)

#### Description
Knowledge entries for agent context. Files in `knowledge/` directory indexed and available to agent via `knowledge/index.yaml`.

#### Key Components

**1. Knowledge Entry Schema (`knowledge/index.yaml`):**
```yaml
entries:
  - path: "api-docs/authentication.md"
    tags: ["api", "auth"]
    priority: "high"
    always_load: true

  - path: "guides/getting-started.md"
    tags: ["guide"]
    priority: "medium"
```

**2. Data Structures:**
```typescript
interface KnowledgeEntry {
  path: string;           // relative to knowledge/
  tags: string[];          // for filtering/organization
  priority: "high" | "medium" | "low";
  always_load?: boolean;   // preloaded into system prompt
}

interface LoadedKnowledge {
  preloaded: Array<{ path: string; content: string }>;  // always_load=true
  available: KnowledgeEntry[];                          // on-demand
}
```

#### Loading Flow (lines 23-56)

```
loadKnowledge(agentDir)
├── read knowledge/index.yaml
├── if missing → return { preloaded: [], available: [] }
├── parse YAML entries
├── for each entry:
│   ├── if always_load:
│   │   ├── read knowledge/<path>
│   │   └── trim content → preloaded[]
│   └── else:
│       └── available[]
└── return { preloaded, available }
```

**Error Handling:**
- Missing `index.yaml` → returns empty (not an error)
- Missing entry file (always_load) → skipped silently
- Missing `entries` array → returns empty
- Non-array entries → returns empty

#### Prompt Formatting (lines 58-81)

**Preloaded (always_load=true):**
```xml
<knowledge path="api-docs/authentication.md">
Full file content here...
</knowledge>
```

**Available (on-demand):**
```xml
<available_knowledge>
<doc path="knowledge/guides/getting-started.md" priority="medium" tags="guide" />
</available_knowledge>

Use the `read` tool to load any available knowledge document when needed.
```

#### Clever Solutions
1. **Two-tier loading**: Always-load files injected directly; others available via read tool
2. **Content trimming**: `.trim()` on loaded content removes trailing whitespace
3. **Tag formatting**: Tags joined with comma in attribute format
4. **Empty check**: Returns `""` if no knowledge, preventing empty section in prompt

#### Edge Cases & Error Handling
- Empty `knowledge/` directory → works fine
- Missing `index.yaml` → returns empty gracefully
- File referenced but missing on disk → skipped (for always_load)
- Invalid YAML in index → yaml.load throws, propagates up
- Non-array entries → returns empty (guards against malformed index)

#### Technical Debt / Concerns
1. **No indexing**: No search/index functionality; agent must use `read` tool
2. **No versioning**: Knowledge files aren't versioned like memory
3. **No priority enforcement**: Priority field is informational only
4. **No knowledge update detection**: Agent must manually re-read to get updates
5. **Path traversal risk**: Entry paths are joined with knowledge dir; no sanitization

#### Integration Points
- `loader.ts` line 271-274: Knowledge loading during agent initialization
- `formatKnowledgeForPrompt()`: Converts to prompt section

---

## Cross-Feature Analysis

### Shared Patterns

**1. Directory-based Discovery:**
All three features follow the same pattern:
```
<feature>Dir = join(agentDir, "<feature>")
├── check exists/isDirectory (stat)
├── readdir entries
├── filter files
├── parse content (YAML/frontmatter)
├── return sorted metadata[]
```

**2. Prompt Injection:**
All three use similar XML-based formatting for system prompt injection:
- Workflows: `<workflow>`, `<available_workflows>`
- Sub-agents: `<agent>`, `<available_agents>`
- Knowledge: `<knowledge>`, `<available_knowledge>`

**3. Graceful Degradation:**
All silently skip errors (missing files, invalid YAML, clone failures)

### Relationships

```
Workflows ──────► Skills
   │                  │
   └──────┬───────────┘
          │
          ▼
      Agent Load
          │
          ├──► Knowledge
          ├──► Sub-Agents
          └──► Inheritance
```

- **Workflows** chain **Skills** together
- **Skills**, **Knowledge**, **Sub-Agents**, and **Inheritance** are all discovered during **Agent Load**

### Common Technical Debt

1. **No validation CLI**: No `gitclaw workflow validate` or `gitclaw knowledge index`
2. **Silent failures**: Debugging requires adding console.logs
3. **No caching**: Re-discovers on every agent load
4. **No incremental updates**: Full re-scan required for changes

---

## File Summary

| File | Lines | Purpose |
|------|-------|---------|
| `src/workflows.ts` | 167 | Workflow discovery, parsing, formatting |
| `src/agents.ts` | 97 | Sub-agent discovery |
| `src/loader.ts` | 382 | Agent loading including inheritance resolution |
| `src/knowledge.ts` | 82 | Knowledge base loading |

---

## Recommendations for Further Research

1. **Workflow Execution**: Trace how workflows are actually executed (if at all)
2. **Delegation Mechanism**: How does `gitclaw --dir <agent>` actually work?
3. **Plugin Knowledge**: Do plugins contribute to knowledge base?
4. **Skill vs Workflow**: What distinguishes a workflow from a chain of skill invocations?
