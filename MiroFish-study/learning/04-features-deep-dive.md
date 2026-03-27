# MiroFish Features Deep Dive

**Project:** MiroFish - Multi-Agent AI Prediction Engine
**Repository:** `/Users/sheldon/Documents/claw/reference/MiroFish`
**Date:** 2026-03-27
**Synthesized from:** 05a-features-batch-1.md, 05b-features-batch-2.md, 05c-features-batch-3.md, 05d-features-batch-4.md

---

## Executive Summary

MiroFish is a multi-agent simulation engine that constructs high-fidelity parallel digital worlds from real-world seed information (news, policies, financial signals). The system implements a 5-step workflow: Graph Build, Environment Setup, Simulation Run, Report Generation, and Deep Interaction. At its core, OASIS-powered agents with independent personalities, long-term memory (via Zep), and behavioral logic interact freely on dual platforms (Twitter-style and Reddit-style), enabling predictive simulations.

---

## Priority Tier 1: Core Features

### Feature 1: Multi-Agent Simulation Engine (OASIS-powered)

**Largest Backend Component** | 68KB `simulation_runner.py`

#### Architecture

The simulation engine is a **dual-platform parallel system** that runs Twitter and Reddit simulations simultaneously. It uses the OASIS library (`camel-ai`) as the core simulation engine and communicates with the Flask backend via file-based IPC.

```
Flask Backend                          Simulation Process
      |                                       |
      |--write command.json------------------>|
      |                                       |--poll commands/
      |<--write response.json----------------|
```

#### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/simulation_runner.py` | 68KB | Main simulation lifecycle management |
| `backend/app/services/simulation_manager.py` | 20KB | Simulation orchestration and preparation |
| `backend/app/services/simulation_ipc.py` | 12KB | Flask-to-simulation IPC communication |
| `backend/scripts/run_parallel_simulation.py` | 62KB | OASIS-powered simulation execution script |

#### Key Implementation Details

**Simulation Lifecycle Management** (`simulation_runner.py`):
```python
class SimulationRunner:
    _run_states: Dict[str, SimulationRunState] = {}      # In-memory state
    _processes: Dict[str, subprocess.Popen] = {}         # Subprocess handles
    _monitor_threads: Dict[str, threading.Thread] = {}  # Progress tracking
```

**Dual-Platform Completion Detection:**
```python
def _check_all_platforms_completed(cls, state) -> bool:
    twitter_enabled = os.path.exists(twitter_log)
    reddit_enabled = os.path.exists(reddit_log)
    # Only marks complete when ALL enabled platforms finish
```

**Action Log Monitoring:** The runner monitors `actions.jsonl` files for each platform. Event types filtered out: `simulation_end`, `round_end`.

**Interview System:** Real-time agent querying during simulation:
- `interview_agent()` - Single agent
- `interview_agents_batch()` - Multiple agents
- `interview_all_agents()` - Global interview

#### Clever Solutions

1. **Platform Detection for IPC:**
   ```python
   IS_WINDOWS = sys.platform == 'win32'
   ```
   Handles Windows' lack of `SIGHUP` and different process termination.

2. **UTF-8 Encoding Monkey Patch (Windows):**
   ```python
   def _utf8_open(file, mode='r', ...):
       if encoding is None and 'b' not in mode:
           encoding = 'utf-8'
       return _original_open(file, mode, ...)
   builtins.open = _utf8_open
   ```
   Fixes OASIS library's unspecified file encodings.

3. **Cross-Platform Process Termination:**
   ```python
   if IS_WINDOWS:
       subprocess.run(['taskkill', '/PID', str(process.pid), '/T'], ...)
   else:
       pgid = os.getpgid(process.pid)
       os.killpg(pgid, signal.SIGTERM)
   ```

#### Technical Debt

1. **Polling-Based IPC:** File polling every 0.5s is inefficient. Consider websockets or named pipes.
2. **Process Cleanup Complexity:** Multiple signal handlers are fragile.
3. **No Simulation Pause:** Supports `STOPPED` but not `PAUSED` state.
4. **Log File Descriptor Leak:** `_stdout_files` dict stores file handles needing explicit cleanup.

---

### Feature 2: Knowledge Graph Construction (GraphRAG)

**Backend Component** | 17KB `graph_builder.py`, 21KB `zep_graph_memory_updater.py`

#### Architecture

Uses **Zep Cloud** as the knowledge graph backend:
1. Extracts entities and relationships from seed documents
2. Builds a Zep graph with custom ontology
3. Enables agent queries during simulation

#### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/graph_builder.py` | 17KB | Zep graph creation and text ingestion |
| `backend/app/services/zep_graph_memory_updater.py` | 21KB | Real-time simulation activity updates |
| `backend/app/services/zep_entity_reader.py` | 15KB | Entity filtering and retrieval |
| `backend/app/api/graph.py` | 20KB | REST API endpoints |

#### Key Implementation Details

**Graph Building Pipeline** (`graph_builder.py`):
```python
def build_graph_async(self, text, ontology, ...):
    task_id = self.task_manager.create_task(...)
    thread = threading.Thread(
        target=self._build_graph_worker,
        args=(task_id, text, ontology, ...)
    )
    thread.daemon = True
    thread.start()
    return task_id
```

**Worker Thread Steps:**
1. Create Zep graph with unique ID
2. Set custom ontology (entity + edge types)
3. Split text into chunks
4. Batch upload to Zep (3 chunks per batch)
5. Wait for processing via episode polling
6. Retrieve graph stats

**Custom Ontology Setting:** Dynamic Pydantic model creation for entity types with reserved name handling:
```python
RESERVED_NAMES = {'uuid', 'name', 'group_id', 'name_embedding', 'summary', 'created_at'}

def safe_attr_name(attr_name: str) -> str:
    if attr_name.lower() in RESERVED_NAMES:
        return f"entity_{attr_name}"
    return attr_name
```

**Episode Processing Wait:**
```python
def _wait_for_episodes(self, episode_uuids, timeout=600):
    pending_episodes = set(episode_uuids)
    while pending_episodes:
        for ep_uuid in list(pending_episodes):
            episode = self.client.graph.episode.get(uuid_=ep_uuid)
            if getattr(episode, 'processed', False):
                pending_episodes.remove(ep_uuid)
        time.sleep(3)  # Poll every 3 seconds
```

**Graph Memory Updater** (`zep_graph_memory_updater.py`):
```python
class ZepGraphMemoryUpdater:
    BATCH_SIZE = 5      # Batch activities before sending
    SEND_INTERVAL = 0.5  # Inter-batch delay

    def add_activity(self, activity: AgentActivity):
        if activity.action_type == "DO_NOTHING":
            self._skipped_count += 1
            return
        self._activity_queue.put(activity)
```

**Activity Description Generation:** Each activity type generates natural language descriptions for Zep (Chinese):
```python
def _describe_create_post(self) -> str:
    content = self.action_args.get("content", "")
    if content:
        return f"发布了一条帖子：「{content}」"
    return "发布了一条帖子"
```

#### Clever Solutions

1. **Episode Polling with Timeout:** Instead of webhooks, uses polling with exponential backoff.
2. **Graceful Degradation:** If Zep search fails, falls back to entity attributes without failing profile generation.
3. **Natural Language Activity Descriptions:** Converts structured actions into Chinese narrative for better graph extraction.
4. **Batch Size Tuning:** `BATCH_SIZE = 5` balances throughput vs. API rate limits.

#### Technical Debt

1. **No Graph Schema Migration:** If ontology changes, existing graphs cannot be migrated.
2. **Zep API Dependency:** Tight coupling with Zep Cloud. If Zep has outages, entire system fails.
3. **Polling Inefficiency:** `_wait_for_episodes()` polls every 3 seconds for up to 600 seconds.

---

### Feature 3: Agent Persona/Profile Generation

**Backend Component** | 49KB `oasis_profile_generator.py`

#### Architecture

Transforms Zep knowledge graph entities into **OASIS-compatible agent profiles**. Supports both Twitter (CSV) and Reddit (JSON) formats with rich persona attributes.

#### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/oasis_profile_generator.py` | 49KB | Profile generation with LLM |
| `backend/app/utils/llm_client.py` | 3KB | LLM API wrapper |

#### Key Implementation Details

**Profile Data Model:**
```python
@dataclass
class OasisAgentProfile:
    # Universal fields
    user_id: int
    user_name: str
    name: str
    bio: str
    persona: str

    # Reddit-specific
    karma: int = 1000

    # Twitter-specific
    friend_count: int = 100
    follower_count: int = 150
    statuses_count: int = 500

    # Extended attributes
    age: Optional[int] = None
    gender: Optional[str] = None
    mbti: Optional[str] = None
    country: Optional[str] = None
    profession: Optional[str] = None
    interested_topics: List[str] = field(default_factory=list)
```

**Entity Type Classification:**
```python
INDIVIDUAL_ENTITY_TYPES = [
    "student", "alumni", "professor", "person", "publicfigure",
    "expert", "faculty", "official", "journalist", "activist"
]

GROUP_ENTITY_TYPES = [
    "university", "governmentagency", "organization", "ngo",
    "mediaoutlet", "company", "institution", "group", "community"
]
```

**LLM-Powered Profile Generation with Temperature Annealing:**
```python
for attempt in range(max_attempts):
    try:
        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=[...],
            temperature=0.7 - (attempt * 0.1)  # Lower temp on retry
    except Exception as e:
        time.sleep(1 * (attempt + 1))  # Exponential backoff
```

**Parallel Profile Generation:**
```python
with concurrent.futures.ThreadPoolExecutor(max_workers=parallel_count) as executor:
    future_to_entity = {
        executor.submit(generate_single_profile, idx, entity): (idx, entity)
        for idx, entity in enumerate(entities)
    }
    for future in concurrent.futures.as_completed(future_to_entity):
        result_idx, profile, error = future.result()
        profiles[result_idx] = profile
        save_profiles_realtime()  # Write after each completion
```

**Platform-Specific Output Formats:**
- Twitter CSV: `headers = ['user_id', 'name', 'username', 'user_char', 'description']`
- Reddit JSON: Full profile with normalized gender, age defaults to 30

#### Clever Solutions

1. **Temperature Annealing:** Lower temperature on retry (0.7 -> 0.6 -> 0.5) for more deterministic results.
2. **Fallback Profile Generation:** If LLM fails completely, generates rule-based profiles for continuity.
3. **Gender Normalization:** `gender_map = {"男": "male", "女": "female", "机构": "other"}`
4. **Real-time Progress Saving:** Profiles written after each completion prevents data loss on failure.

#### Technical Debt

1. **No Profile Versioning:** Updates overwrite previous versions.
2. **LLM Cost Scaling:** 2000-character persona descriptions per agent can be expensive.
3. **Hardcoded Entity Types:** Adding new entity types requires code changes.
4. **Username Collision Risk:** Random 3-digit suffixes could theoretically collide.

---

### Feature 4: Ontology & Environment Generation

**Backend Component** | 16KB `ontology_generator.py`

#### Purpose

Extracts entity relationships from seed data and generates environment configurations. Creates the structural foundation for simulations by designing entity types and relationship types tailored for social media opinion simulation.

#### Key Implementation

**System Prompt Design:** The `ONTOLOGY_SYSTEM_PROMPT` is remarkably detailed (150+ lines), specifying:
- Exactly 10 entity types required
- Last 2 must be fallback types: `Person` and `Organization`
- First 8 are concrete types derived from document content
- Clear exclusion rules: no abstract concepts like "opinions", "trends", "sentiment"

**Automatic Fallback Type Injection:**
```python
# If LLM generates > 10 types, removes from end (preserving concrete types)
# If Person/Organization missing, automatically adds them
```

**Validation Constraints:**
```python
MAX_ENTITY_TYPES = 10
MAX_EDGE_TYPES = 10
```

**Code Generation:** `generate_python_code()` transforms ontology into Pydantic models for Zep integration:
```python
ENTITY_TYPES = {
    "Student": Student,
    "Professor": Professor,
    ...
}
EDGE_TYPES = {
    "WORKS_FOR": Works_For,
    ...
}
```

#### Technical Debt

1. **Hardcoded truncation:** `MAX_TEXT_LENGTH_FOR_LLM = 50000` is a magic number
2. **No validation of entity type names:** LLM could output invalid Python identifiers
3. **Limited error recovery:** If LLM call fails, exception propagates up

---

### Feature 5: Report Generation Agent

**Largest Service File** | 99KB `report_agent.py`

#### Purpose

AI agent with rich toolset that deeply interacts with simulation results to generate comprehensive prediction reports. Uses ReACT (Reasoning + Acting) pattern for multi-turn tool calling and reflection.

#### Architecture

**Dual Logging System:**
- **ReportLogger:** Outputs JSONL format to `agent_log.jsonl`
- **ReportConsoleLogger:** Outputs to `console_log.txt` with standard format

**Report Manager Pattern:**
```
reports/{report_id}/
    meta.json          - Report metadata
    outline.json        - Report outline
    progress.json       - Generation progress
    section_01.md      - Chapter 1
    section_02.md      - Chapter 2
    ...
    full_report.md     - Complete report
    agent_log.jsonl    - Agent execution log
    console_log.txt    - Console output
```

#### ReACT Pattern Implementation

**Section Generation Loop:** Each section goes through max 5 tool calls, minimum 3 required:
```python
# Iteration structure:
1. LLM responds with thought/action
2. Parse tool calls from response
3. If Final Answer detected and tool calls >= 3, output content
4. If tool call present, execute and inject observation
```

**Conflict Handling:** When LLM outputs BOTH tool call AND Final Answer simultaneously:
```python
# First 2 conflicts: ask LLM to retry with corrected format
if conflict_retries <= 2:
    messages.append({"role": "user", "content": "【格式错误】..."})
# Third conflict: truncate to first tool_call, force execution
else:
    response = response[:first_tool_end + len('</tool_call>')]
```

#### Tool System

Four tools available to the agent:

1. **insight_forge** - Deep insight retrieval
   - Auto-decomposes query into sub-problems
   - Multi-dimensional retrieval: semantic + entity + relationship chains

2. **panorama_search** - Wide-angle view
   - Returns all nodes and edges including expired ones
   - Shows temporal evolution

3. **quick_search** - Fast lookup
   - Lightweight search for specific facts

4. **interview_agents** - Agent interviews (dual-platform)
   - Calls OASIS interview API on Twitter and Reddit simultaneously
   - Selects relevant agents based on topic
   - Returns structured interview data with key quotes

#### Formatting Rules (Strictly Enforced)

- No markdown headings (`#`, `##`, etc.) in section content
- Use **bold** instead of sub-headers
- Quotes must be standalone paragraphs
- Section title auto-added by system, not LLM

#### Clever Solutions

1. **Streaming-friendly design:** Each section saved immediately rather than waiting for full report
2. **JSONL logging with replay capability:** `get_agent_log_stream()` enables real-time log streaming to frontend
3. **Multiple tool call formats supported:** XML style and bare JSON

#### Technical Debt

1. **Chat mode** has 15K char limit for report content context
2. **No validation of tool call arguments**
3. **LLM can occasionally bypass format constraints**

---

### Feature 6: Simulation Configuration Generator

**Backend Component** | 39KB `simulation_config_generator.py`

#### Purpose

Generates simulation parameters including agent counts, interaction rules, time windows, and platform-specific settings. Uses LLM for intelligent configuration with rule-based fallback.

#### Configuration Dataclasses

```python
@dataclass
class AgentActivityConfig:
    agent_id: int
    entity_uuid: str
    entity_name: str
    entity_type: str
    activity_level: float = 0.5      # 0.0-1.0
    posts_per_hour: float = 1.0
    comments_per_hour: float = 2.0
    active_hours: List[int] = field(default_factory=lambda: list(range(8, 23)))
    response_delay_min: int = 5
    response_delay_max: int = 60
    sentiment_bias: float = 0.0      # -1.0 to 1.0
    stance: str = "neutral"
    influence_weight: float = 1.0
```

**China Timezone Configuration (Built-in):**
```python
CHINA_TIMEZONE_CONFIG = {
    "dead_hours": [0, 1, 2, 3, 4, 5],           # 5% activity
    "morning_hours": [6, 7, 8],                  # 40% activity
    "work_hours": [9, 10, 11, 12, 13, 14, 15, 16, 17, 18],  # 70% activity
    "peak_hours": [19, 20, 21, 22],             # 150% activity
    "night_hours": [23],                         # 50% activity
}
```

#### Step-by-Step Generation Strategy

```
Step 1: Time config generation
Step 2: Event config generation
Step 3-N: Batch agent configs (15 agents per batch)
Final: Platform config assignment
```

**Batch Size Optimization:** `AGENTS_PER_BATCH = 15` balances LLM context window vs. API call overhead.

#### Rule-Based Agent Configuration Fallback

When LLM fails to generate agent configs:
```python
# University/Government agency defaults
if entity_type in ["university", "governmentagency", "ngo"]:
    return {
        "activity_level": 0.2,
        "posts_per_hour": 0.1,
        "comments_per_hour": 0.05,
        "active_hours": list(range(9, 18)),  # Work hours only
        "response_delay_min": 60,
        "response_delay_max": 240,  # Slow response
        "influence_weight": 3.0  # High influence
    }

# Student defaults (high activity)
elif entity_type in ["student"]:
    return {
        "activity_level": 0.8,
        "posts_per_hour": 0.6,
        "comments_per_hour": 1.5,
        "active_hours": [8, 9, 10, 11, 12, 13, 18, 19, 20, 21, 22, 23],
        "response_delay_min": 1,
        "response_delay_max": 15,  # Fast response
        "influence_weight": 0.8
    }
```

#### JSON Resilience

**Temperature Annealing:** `temperature=0.7 - (attempt * 0.1)` reduces from 0.7 to 0.5 to 0.3

**Truncation Detection and Fix:**
```python
def _fix_truncated_json(content: str) -> str:
    open_braces = content.count('{') - content.count('}')
    open_brackets = content.count('[') - content.count(']')
    content += ']' * open_brackets
    content += '}' * open_braces
    return content
```

#### Technical Debt

1. **Magic numbers throughout:** `AGENTS_PER_BATCH = 15`, `MAX_CONTEXT_LENGTH = 50000`
2. **Platform configs hardcoded:** Could be LLM-generated
3. **Limited retry logic:** 3 attempts with fixed delays, no exponential backoff
4. **No validation of generated entity types:** If LLM invents new types, they pass through unchecked

---

### Feature 7: Interactive 5-Step Workflow UI

**Frontend Components** | 17KB - 145KB Vue components

#### Overview

The frontend implements a wizard-style workflow guiding users through simulation creation across 5 distinct steps. Each step is a separate Vue component with its own concerns.

#### Step Components

| File | Size | Purpose |
|------|------|---------|
| `frontend/src/views/Step1GraphBuild.vue` | 17KB | Ontology generation + GraphRAG build |
| `frontend/src/views/Step2EnvSetup.vue` | 68KB | Simulation instance + Agent profiles + Config |
| `frontend/src/views/Step3Simulation.vue` | 39KB | Dual-platform simulation execution |
| `frontend/src/views/Step4Report.vue` | 145KB | Report generation workflow |
| `frontend/src/views/Step5Interaction.vue` | 64KB | Deep interaction with agents |

#### Step 1: Graph Build

**Core Flow:**
1. **Phase 0:** Ontology Generation via `POST /api/graph/ontology/generate`
2. **Phase 1:** GraphRAG Construction via `POST /api/graph/build`
3. **Phase 2:** Build Complete - Creates simulation via `POST /api/simulation/create`

#### Step 2: Environment Setup (Largest Component)

**Five sub-phases:**
1. Simulation Instance - Calls `POST /api/simulation/create`
2. Agent Profile Generation - `POST /api/simulation/prepare` with LLM-generated personas
3. Dual-Platform Config - Time config, agent configs, platform configs
4. Initial Orchestration - Narrative direction, hot topics, initial posts
5. Ready to Simulate - Round configuration with auto/custom slider

**Polling Pattern:**
```javascript
// 2-3 second polling intervals
const pollPrepareStatus = async () => {
  const res = await getPrepareStatus({ task_id, simulation_id })
  // Checks for 'completed', 'ready', or 'already_prepared' status
}
```

#### Step 3: Simulation

**Dual-Platform Parallel Execution:**

**Platform Actions:**
| Platform | Actions |
|----------|---------|
| Twitter (Info Plaza) | POST, LIKE, REPOST, QUOTE, FOLLOW, IDLE |
| Reddit (Topic Community) | POST, COMMENT, LIKE, DISLIKE, SEARCH, TREND, FOLLOW, MUTE, REFRESH, IDLE |

**Simulation Start Logic:**
```javascript
const params = {
  simulation_id,
  platform: 'parallel',  // Dual-platform mode
  force: true,          // Force restart
  enable_graph_memory_update: true  // Dynamic graph updates
}
```

**Status Polling:**
```javascript
const checkPlatformsCompleted = (data) => {
  const twitterCompleted = data.twitter_completed === true
  const redditCompleted = data.reddit_completed === true
  // Must check both platforms AND ensure they were enabled
}
```

#### Step 4: Report

**Split Layout:**
- Left panel: Report content with collapsible sections
- Right panel: Workflow timeline + agent logs

**Workflow Timeline Events:**
- `report_start`, `planning_start/planning_complete`, `section_start/section_content/section_complete`
- `tool_call/tool_result`, `llm_response`, `report_complete`

#### Step 5: Interaction

**Three Interaction Modes:**

1. **Chat with Report Agent** - Access to 4 tools:
   - `InsightForge` (deep attribution, purple badge)
   - `PanoramaSearch` (breadth traversal, blue badge)
   - `QuickSearch` (immediate query, orange badge)
   - `InterviewSubAgent` (multi-round dialogue, green badge)

2. **Chat with World Agents** - Direct conversation with simulated agents
   - Agent dropdown selector with profile card

3. **Survey Mode** - Batch questioning with individual responses

---

## Priority Tier 2: Secondary Features

### Feature 8: Long-Term Memory System (Zep Integration)

**Backend Components** | 15KB - 66KB service files

#### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/zep_entity_reader.py` | 15KB | Entity filtering and reading |
| `backend/app/services/zep_tools.py` | 66KB | Search tools for Report Agent |
| `backend/app/utils/zep_paging.py` | 4KB | Cursor-based pagination |

#### Zep Paging Utility

**Problem Solved:** Zep's list APIs use UUID cursor pagination. Transparent complete list retrieval:
```python
def fetch_all_nodes(client: Zep, graph_id: str, page_size: int = 100, max_items: int = 2000):
    all_nodes = []
    cursor = None
    while True:
        kwargs = {"limit": page_size}
        if cursor:
            kwargs["uuid_cursor"] = cursor
        batch = _fetch_page_with_retry(client.graph.node.get_by_graph_id, graph_id, **kwargs)
        # ... handles `uuid_` vs `uuid` attribute naming
```

#### Search Result Types

**SearchResult:** `facts`, `edges`, `nodes`, `query`, `total_count`

**InsightForgeResult:** Deep retrieval with auto-generated sub-queries, semantic facts, entity insights, relationship chains

**PanoramaResult:** Broad retrieval including expired/historical content

**InterviewResult:** Agent interviews via OASIS interview API (dual-platform)

#### Graceful Degradation

```python
except Exception as e:
    logger.warning(f"Zep Search API失败，降级为本地搜索: {str(e)}")
    return self._local_search(graph_id, query, limit, scope)
```

#### Technical Debt

1. **Zep API Attribute Naming Inconsistency:** Files use both `uuid_` and `uuid` interchangeably
2. **Simulation Environment Dependency:** `interview_agents` requires OASIS to be running
3. **Chat History Memory:** No persistence across page refreshes

---

### Feature 9: Graph Visualization Panel

**Frontend Component** | 40KB `GraphPanel.vue`

#### D3 Force Simulation Configuration

```javascript
const simulation = d3.forceSimulation(nodes)
  .force('link', d3.forceLink(edges).id(d => d.id).distance(d => {
    const baseDistance = 150
    const edgeCount = d.pairTotal || 1
    return baseDistance + (edgeCount - 1) * 50
  }))
  .force('charge', d3.forceManyBody().strength(-400))
  .force('center', d3.forceCenter(width / 2, height / 2))
  .force('collide', d3.forceCollide(50))
  .force('x', d3.forceX(width / 2).strength(0.04))
  .force('y', d3.forceY(height / 2).strength(0.04))
```

#### Edge Handling

**Multi-Edge Curvature:**
```javascript
if (totalCount > 1) {
  const curvatureRange = Math.min(1.2, 0.6 + totalCount * 0.15)
  curvature = ((currentIndex / (totalCount - 1)) - 0.5) * curvatureRange * 2
  if (isReversed) {
    curvature = -curvature
  }
}
```

**Self-Loop Handling:**
```javascript
if (e.source_node_uuid === e.target_node_uuid) {
  // Group by source node, only one rendered with count
}
```

#### Interactive Features

- **Node Drag:** With movement threshold detection
- **Detail Panel:** Shows node/edge properties, summary, labels
- **Simulation Building Hint:** Animated brain icon during graph updates

#### Technical Debt

1. **Force simulation on large graphs** (2000+ nodes) may lag
2. **No virtualization or clustering** for large datasets
3. **UUID Exposure** in UI detail panels

---

### Feature 10: Simulation History & Project Management

**Frontend + Backend** | 34KB Vue, 9KB model, 6KB model

#### Architecture

File-based JSON persistence rather than traditional database.

**Storage Structure:**
```
backend/uploads/projects/
  proj_{id}/
    project.json        # Metadata
    extracted_text.txt  # Cached extracted text
    files/              # Original uploaded files
```

**ProjectManager Key Methods:**
- `create_project(name)` - Creates project with directory structure
- `save_project(project)` - Persists to `project.json`
- `get_project(project_id)` - Loads from `project.json`
- `list_projects(limit)` - Lists all projects, sorted by creation time

**TaskManager:** Singleton pattern with thread-safe locking, in-memory task storage with optional cleanup of old tasks (24-hour default).

#### HistoryDatabase Vue Component

**Fan-folded collapsed state:**
```javascript
transform: translate(${x}px, ${y}px) rotate(${r}deg) scale(${s})
```

**IntersectionObserver for auto-expand:**
```javascript
threshold: [0.4, 0.6, 0.8],
rootMargin: '0px 0px -150px 0px'
```

#### Technical Debt

1. **No database transactions** - File-based storage vulnerable to concurrent access
2. **In-memory TaskManager** - Tasks lost on server restart
3. **No pagination** - `list_projects()` loads all projects into memory
4. **Project ID collision possible** - UUID hex slice (12 chars) has collision risk

---

### Feature 11: Platform-Specific Simulations

**Backend Scripts** | 27KB each

#### Architecture

Both scripts are standalone execution scripts using OASIS framework:
- `backend/scripts/run_reddit_simulation.py`
- `backend/scripts/run_twitter_simulation.py`

#### Common Architecture

Both share identical structure:
1. **IPCHandler** - File-based inter-process communication
2. **SimulationRunner** - Main simulation orchestration
3. **Signal handlers** - Graceful shutdown on SIGTERM/SIGINT

**IPC Mechanism:**
```
simulation_dir/
  ipc_commands/     # JSON files with commands
  ipc_responses/    # JSON files with responses
  env_status.json   # Current environment status
```

#### Platform-Specific Actions

**Reddit:** LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT, LIKE_COMMENT, DISLIKE_COMMENT, SEARCH_POSTS, SEARCH_USER, TREND, REFRESH, DO_NOTHING, FOLLOW, MUTE

**Twitter:** CREATE_POST, LIKE_POST, REPOST, FOLLOW, DO_NOTHING, QUOTE_POST

#### Agent Activation Logic

```python
def _get_active_agents_for_round(self, env, current_hour, round_num):
    base_min = time_config.get("agents_per_hour_min", 5)
    base_max = time_config.get("agents_per_hour_max", 20)

    # Time-based multipliers
    if current_hour in peak_hours:
        multiplier = time_config.get("peak_activity_multiplier", 1.5)
    elif current_hour in off_peak_hours:
        multiplier = time_config.get("off_peak_activity_multiplier", 0.3)

    target_count = int(random.uniform(base_min, base_max) * multiplier)
    # Filter candidates by active_hours and activity_level
```

#### Technical Debt (Significant)

1. **Code duplication** - Reddit and Twitter scripts are nearly identical
2. **Polling-based IPC** - Inefficient compared to async message queues
3. **No authentication on IPC** - Any process can send commands
4. **Profile format mismatch** - Reddit uses JSON, Twitter uses CSV
5. **No simulation state persistence** - Lost on crash
6. **Hardcoded semaphore=30** - Max concurrent LLM requests not configurable
7. **No retry logic on LLM failures** - Simulation halts on API error

---

### Feature 12: Data Input Processing

**Backend Components** | 2KB - 5KB

#### FileParser

**Supported Formats:** `.pdf`, `.md`, `.markdown`, `.txt`

**Encoding Detection Fallback Chain:**
```python
def _read_text_with_fallback(file_path: str) -> str:
    # 1. Try UTF-8
    # 2. charset_normalizer.from_bytes(data).best()
    # 3. chardet.detect(data)
    # 4. Fallback to UTF-8 with errors='replace'
```

**Text Extraction Methods:**
- PDF: PyMuPDF (fitz)
- Markdown/TXT: With encoding detection

#### Text Chunking

**Smart Sentence Boundary Detection:**
```python
def split_text_into_chunks(text, chunk_size=500, overlap=50):
    separators = ['。', '！', '？', '.\n', '!\n', '?\n', '\n\n', '. ', '! ', '? ']
    for sep in separators:
        last_sep = text[start:end].rfind(sep)
        if last_sep != -1 and last_sep > chunk_size * 0.3:
            end = start + last_sep + len(sep)
            break
```

#### TextProcessor

**High-level Interface:**
```python
class TextProcessor:
    @staticmethod
    def extract_from_files(file_paths: List[str]) -> str:
        return FileParser.extract_from_multiple(file_paths)

    @staticmethod
    def split_text(text, chunk_size=500, overlap=50) -> List[str]:
        return split_text_into_chunks(text, chunk_size, overlap)

    @staticmethod
    def preprocess_text(text: str) -> str:
        # Normalize newlines, remove 3+ consecutive newlines, trim whitespace
```

#### Technical Debt

1. **No file size limits** - Large PDFs could exhaust memory
2. **No streaming for large files** - Loads entire file into memory
3. **Limited format support** - No DOCX, XLSX, HTML support
4. **No image/table extraction from PDF** - Only raw text
5. **Chunk overlap could create duplicate context**
6. **No language detection** - Assumes Chinese/English punctuation

---

## Cross-Cutting Analysis

### Data Flow

```
[Seed Documents]
       |
       v
[OntologyGenerator] --> [Ontology Definition]
       |                        |
       v                        v
[GraphBuilder] ---------> [Zep Knowledge Graph]
       |                        |
       |                        v
       |               [ZepEntityReader] --> [Filtered Entities]
       |                        |
       |                        v
       |               [OasisProfileGenerator] --> [Agent Profiles]
       |                        |
       |                        v
       |               [SimulationManager] --> [SimulationRunner]
       |                        |
       v                        v
[OASIS Simulation] <------------+
       |
       v
[ZepGraphMemoryUpdater] (real-time updates)
```

### Shared Patterns

1. **Progress callbacks** - All async features use callback-based progress reporting
2. **File-based output** - Report Agent and Simulation write to `Config.UPLOAD_FOLDER`
3. **LLM client reuse** - All use `LLMClient` from `../utils/llm_client.py`
4. **Zep integration** - Both Report Agent and Simulation Config use Zep services
5. **China-specific defaults** - Incorporates Chinese user patterns

### Error Handling Patterns

1. **Retry with Exponential Backoff** - Consistent pattern in both frontend and backend
2. **Graceful Degradation** - Zep search falls back to local keyword search
3. **JSON Repair** - Custom truncation detection and bracket closing
4. **Temperature Annealing** - Reduces creativity on LLM retries

### Security Considerations

1. **File-based IPC is not authenticated** - Any process with filesystem access can send commands
2. **No rate limiting** on Zep API calls or LLM generations
3. **UUID Exposure** - Node/edge UUIDs exposed in UI detail panels
4. **No input sanitization** - Project names, file paths could contain special characters

### Performance Characteristics

| Operation | Typical Duration | Bottleneck |
|-----------|-----------------|------------|
| Graph build (100 chunks) | 5-10 minutes | Zep API processing |
| Profile generation (50 agents) | 2-5 minutes | LLM API latency |
| Simulation (72h simulated) | Variable | OASIS + LLM |
| Zep search | 1-3 seconds | Network + Zep |

---

## Key Technical Debt Summary

### High Priority

1. **No Simulation Pause/Resume** - Must complete in one run
2. **File-based IPC** - Polling inefficiency, no authentication
3. **Code Duplication** - Reddit/Twitter scripts nearly identical
4. **In-memory TaskManager** - Tasks lost on server restart

### Medium Priority

1. **Zep API Dependency** - Tight coupling, no fallback
2. **No Graph Schema Migration** - Ontology changes break existing graphs
3. **Hardcoded Platform Configs** - Could be LLM-generated
4. **Profile Format Mismatch** - JSON vs CSV inconsistency

### Low Priority

1. **No Profile Versioning** - Overwrites previous versions
2. **No Pagination** - Project listing loads all into memory
3. **Limited Format Support** - No DOCX, XLSX, HTML
4. **Username Collision Risk** - Random suffix could theoretically collide

---

## File Size Summary

| File | Size | Feature |
|------|------|---------|
| `report_agent.py` | 99KB | Report Generation Agent |
| `Step4Report.vue` | 145KB | Report Workflow UI |
| `simulation.py` (API) | 95KB | Simulation Endpoints |
| `simulation_runner.py` | 68KB | Simulation Engine |
| `zep_tools.py` | 66KB | Zep Search Tools |
| `run_parallel_simulation.py` | 62KB | Parallel Simulation Script |
| `oasis_profile_generator.py` | 49KB | Profile Generation |
| `simulation_config_generator.py` | 39KB | Config Generation |
| `Step5Interaction.vue` | 64KB | Interaction UI |
| `GraphPanel.vue` | 40KB | Graph Visualization |
| `Step2EnvSetup.vue` | 68KB | Environment Setup |
| `Step3Simulation.vue` | 39KB | Simulation UI |
| `zep_graph_memory_updater.py` | 21KB | Graph Memory Updates |
| `simulation_manager.py` | 20KB | Simulation Orchestration |
| `graph.py` (API) | 20KB | Graph API |
| `graph_builder.py` | 17KB | Graph Construction |
| `Step1GraphBuild.vue` | 17KB | Graph Build UI |
| `zep_entity_reader.py` | 15KB | Entity Reading |
| `ontology_generator.py` | 16KB | Ontology Generation |
| `HistoryDatabase.vue` | 34KB | History UI |
| `project.py` | 9KB | Project Model |
| `simulation_ipc.py` | 12KB | IPC Communication |
| `task.py` | 6KB | Task Model |
| `file_parser.py` | 5KB | File Parsing |
| `zep_paging.py` | 4KB | Zep Pagination |
| `llm_client.py` | 3KB | LLM Client |
| `text_processor.py` | 2KB | Text Processing |
