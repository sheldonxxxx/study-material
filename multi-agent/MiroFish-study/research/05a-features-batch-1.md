# Feature Deep Dive: Batch 1 (Multi-Agent Simulation, Knowledge Graph, Agent Personas)

## MiroFish Project Deep Analysis

**Date:** 2026-03-27
**Repository:** `/Users/sheldon/Documents/claw/reference/MiroFish`
**Batch Features:**
1. Multi-Agent Simulation Engine (OASIS-powered)
2. Knowledge Graph Construction (GraphRAG)
3. Agent Persona/Profile Generation

---

## Feature 1: Multi-Agent Simulation Engine (OASIS-powered)

### Architecture Overview

The simulation engine is a **dual-platform parallel system** that runs Twitter and Reddit simulations simultaneously. It uses the OASIS library as the core simulation engine and communicates with the Flask backend via file-based IPC.

### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/simulation_runner.py` | 68KB | Main simulation lifecycle management |
| `backend/app/services/simulation_manager.py` | 20KB | Simulation orchestration and preparation |
| `backend/app/services/simulation_ipc.py` | 12KB | Flask-to-simulation IPC communication |
| `backend/scripts/run_parallel_simulation.py` | 62KB | OASIS-powered simulation execution script |

### Key Implementation Details

#### 1. Simulation Lifecycle (`simulation_runner.py`)

The `SimulationRunner` class manages the full simulation lifecycle:

```python
class SimulationRunner:
    # Run states stored in memory + persisted to files
    _run_states: Dict[str, SimulationRunState] = {}
    _processes: Dict[str, subprocess.Popen] = {}
    _monitor_threads: Dict[str, threading.Thread] = {}
```

**Simulation Start Flow:**
1. `start_simulation()` spawns a subprocess running `run_parallel_simulation.py`
2. Sets up UTF-8 environment variables for Windows compatibility
3. Creates a dedicated process group via `start_new_session=True`
4. Launches a monitor thread to track progress

**Key Implementation (lines 437-447):**
```python
process = subprocess.Popen(
    cmd,
    cwd=sim_dir,
    stdout=main_log_file,
    stderr=subprocess.STDOUT,
    text=True,
    encoding='utf-8',
    bufsize=1,
    env=env,
    start_new_session=True,
)
```

#### 2. Dual-Platform Architecture (`simulation_ipc.py`)

The IPC system uses **file-based polling** for cross-process communication:

```
Flask Backend                          Simulation Process
      |                                       |
      |--write command.json------------------>|
      |                                       |--poll commands/
      |<--write response.json----------------|
      |                                       |
```

**Command Types:**
- `INTERVIEW` - Single agent interview
- `BATCH_INTERVIEW` - Multiple agent interviews
- `CLOSE_ENV` - Graceful environment shutdown

**Timeout Mechanism (lines 157-177):**
```python
while time.time() - start_time < timeout:
    if os.path.exists(response_file):
        # Read and parse response
        # Cleanup command/response files
        return response
    time.sleep(poll_interval)
raise TimeoutError(f"等待命令响应超时 ({timeout}秒)")
```

#### 3. Action Log Monitoring (`simulation_runner.py`, lines 579-686)

The runner monitors `actions.jsonl` files for each platform:

```python
twitter_actions_log = os.path.join(sim_dir, "twitter", "actions.jsonl")
reddit_actions_log = os.path.join(sim_dir, "reddit", "actions.jsonl")
```

**Log Entry Format:**
```json
{
  "round": 5,
  "timestamp": "2026-03-27T10:30:00",
  "agent_id": 2,
  "agent_name": "User_123",
  "action_type": "CREATE_POST",
  "action_args": {"content": "Hello world"},
  "success": true
}
```

**Event Types (filtered out from actions):**
- `simulation_end` - Marks platform completion
- `round_end` - Updates round progress

#### 4. Cross-Platform Process Termination (`simulation_runner.py`, lines 716-769)

**Windows vs Unix handling:**
```python
if IS_WINDOWS:
    # Uses taskkill to terminate process tree
    subprocess.run(['taskkill', '/PID', str(process.pid), '/T'], ...)
else:
    # Uses process group signaling
    pgid = os.getpgid(process.pid)
    os.killpg(pgid, signal.SIGTERM)
    # Falls back to SIGKILL if timeout
```

#### 5. Interview System (lines 1366-1603)

Allows real-time agent querying during simulation:
- `interview_agent()` - Single agent
- `interview_agents_batch()` - Multiple agents
- `interview_all_agents()` - Global interview

### Clever Solutions

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

3. **Dual-Platform Completion Detection:**
   ```python
   def _check_all_platforms_completed(cls, state) -> bool:
       twitter_enabled = os.path.exists(twitter_log)
       reddit_enabled = os.path.exists(reddit_log)
       # Only marks complete when ALL enabled platforms finish
   ```

### Technical Debt / Concerns

1. **Polling-Based IPC:** File polling every 0.5s is inefficient. Consider websockets or named pipes for better latency.

2. **Process Cleanup Complexity:** The `cleanup_handler` with multiple signal handlers is fragile. Could benefit from a unified cleanup manager.

3. **No Simulation Pause:** The system supports `STOPPED` but not `PAUSED` state. Resume capability is missing.

4. **Log File Descriptor Leak:** The `_stdout_files` dict stores file handles that need explicit cleanup.

---

## Feature 2: Knowledge Graph Construction (GraphRAG)

### Architecture Overview

Uses **Zep Cloud** as the knowledge graph backend. The system:
1. Extracts entities and relationships from seed documents
2. Builds a Zep graph with custom ontology
3. Enables agent queries during simulation

### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/graph_builder.py` | 17KB | Zep graph creation and text ingestion |
| `backend/app/services/zep_graph_memory_updater.py` | 21KB | Real-time simulation activity updates |
| `backend/app/services/zep_entity_reader.py` | 15KB | Entity filtering and retrieval |
| `backend/app/api/graph.py` | 20KB | REST API endpoints |

### Key Implementation Details

#### 1. Graph Building Pipeline (`graph_builder.py`)

**Async Build Flow (lines 53-94):**
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

**Worker Thread Steps (lines 96-185):**
1. Create Zep graph with unique ID
2. Set custom ontology (entity + edge types)
3. Split text into chunks
4. Batch upload to Zep (3 chunks per batch)
5. Wait for processing via episode polling
6. Retrieve graph stats

#### 2. Custom Ontology Setting (`graph_builder.py`, lines 199-286)

Dynamic Pydantic model creation for entity types:

```python
def set_ontology(self, graph_id, ontology):
    for entity_def in ontology.get("entity_types", []):
        name = entity_def["name"]
        # Create dynamic class
        attrs = {"__doc__": description}
        annotations = {}
        for attr_def in entity_def.get("attributes", []):
            attr_name = safe_attr_name(attr_def["name"])
            attrs[attr_name] = Field(description=attr_desc, default=None)
            annotations[attr_name] = Optional[EntityText]
        attrs["__annotations__"] = annotations
        entity_class = type(name, (EntityModel,), attrs)
```

**Reserved Name Handling:**
```python
RESERVED_NAMES = {'uuid', 'name', 'group_id', 'name_embedding', 'summary', 'created_at'}

def safe_attr_name(attr_name: str) -> str:
    if attr_name.lower() in RESERVED_NAMES:
        return f"entity_{attr_name}"
    return attr_name
```

#### 3. Episode Processing Wait (`graph_builder.py`, lines 341-395)

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

#### 4. Graph Memory Updater (`zep_graph_memory_updater.py`)

**Real-time Activity Streaming:**

```python
class ZepGraphMemoryUpdater:
    BATCH_SIZE = 5  # Batch activities before sending
    SEND_INTERVAL = 0.5  # Inter-batch delay

    def add_activity(self, activity: AgentActivity):
        if activity.action_type == "DO_NOTHING":
            self._skipped_count += 1
            return
        self._activity_queue.put(activity)
```

**Activity Description Generation (lines 34-198):**

Each activity type generates natural language descriptions for Zep:

```python
def _describe_create_post(self) -> str:
    content = self.action_args.get("content", "")
    if content:
        return f"发布了一条帖子：「{content}」"
    return "发布了一条帖子"

def _describe_like_post(self) -> str:
    post_content = self.action_args.get("post_content", "")
    post_author = self.action_args.get("post_author_name", "")
    if post_content and post_author:
        return f"点赞了{post_author}的帖子：「{post_content}」"
    # ...
```

**Platform-Specific Buffers:**
```python
self._platform_buffers: Dict[str, List[AgentActivity]] = {
    'twitter': [],
    'reddit': [],
}
```

#### 5. Entity Filtering (`zep_entity_reader.py`)

**Smart Filtering Logic (lines 252-270):**
```python
custom_labels = [l for l in labels if l not in ["Entity", "Node"]]
if not custom_labels:
    continue  # Skip nodes with only default labels

if defined_entity_types:
    matching_labels = [l for l in custom_labels if l in defined_entity_types]
    if not matching_labels:
        continue
```

**Enrichment with Edges and Related Nodes:**
```python
for edge in all_edges:
    if edge["source_node_uuid"] == node["uuid"]:
        related_edges.append({
            "direction": "outgoing",
            "edge_name": edge["name"],
            "fact": edge["fact"],
            "target_node_uuid": edge["target_node_uuid"],
        })
```

### Clever Solutions

1. **Episode Polling with Timeout:** Instead of webhooks, uses polling with exponential backoff in `_call_with_retry()`.

2. **Graceful Degradation:** If Zep search fails, falls back to entity attributes without failing the whole profile generation.

3. **Natural Language Activity Descriptions:** Converts structured actions into Chinese narrative descriptions for better graph extraction.

4. **Batch Size Tuning:** `BATCH_SIZE = 5` balances throughput vs. API rate limits.

### Technical Debt / Concerns

1. **No Graph Schema Migration:** If ontology changes, existing graphs cannot be migrated.

2. **Zep API Dependency:** Tight coupling with Zep Cloud. If Zep has outages, entire system fails.

3. **Polling Inefficiency:** `_wait_for_episodes()` polls every 3 seconds for potentially 600 seconds.

4. **No Graph Query Language:** The system only stores and retrieves; no complex queries like path finding.

---

## Feature 3: Agent Persona/Profile Generation

### Architecture Overview

Transforms Zep knowledge graph entities into **OASIS-compatible agent profiles**. Supports both Twitter (CSV) and Reddit (JSON) formats with rich persona attributes.

### Core Files

| File | Size | Purpose |
|------|------|---------|
| `backend/app/services/oasis_profile_generator.py` | 49KB | Profile generation with LLM |
| `backend/app/utils/llm_client.py` | 3KB | LLM API wrapper |

### Key Implementation Details

#### 1. Profile Data Model (`oasis_profile_generator.py`, lines 28-139)

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

#### 2. LLM-Powered Profile Generation (`oasis_profile_generator.py`, lines 496-580)

**Retry Logic with Temperature Annealing:**
```python
for attempt in range(max_attempts):
    try:
        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=[...],
            temperature=0.7 - (attempt * 0.1)  # Lower temp on retry
        )
    except Exception as e:
        time.sleep(1 * (attempt + 1))  # Exponential backoff
```

**Truncated JSON Recovery:**
```python
def _fix_truncated_json(self, content: str) -> str:
    open_braces = content.count('{') - content.count('}')
    open_brackets = content.count('[') - content.count(']')
    content += ']' * open_brackets
    content += '}' * open_braces
    return content
```

#### 3. Entity Type Classification (`oasis_profile_generator.py`, lines 168-178)

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

**Different prompts for different types:**
- Individual: 2000-character detailed biography with personal history
- Group/Institution: Official account persona with institutional voice

#### 4. Parallel Profile Generation (`oasis_profile_generator.py`, lines 850-1009)

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

**Real-time File Output:**
```python
def save_profiles_realtime():
    if output_platform == "reddit":
        with open(realtime_output_path, 'w') as f:
            json.dump([p.to_reddit_format() for p in existing_profiles], f)
    else:
        # Twitter CSV format
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(profiles_data)
```

#### 5. Zep Context Enrichment (`oasis_profile_generator.py`, lines 285-411)

**Hybrid Search Implementation:**
```python
def _search_zep_for_entity(self, entity: EntityNode) -> Dict[str, Any]:
    # Parallel edges and nodes search
    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        edge_future = executor.submit(search_edges)
        node_future = executor.submit(search_nodes)
        edge_result = edge_future.result(timeout=30)
        node_result = node_future.result(timeout=30)

    # Merge results
    all_facts = {edge.fact for edge in edge_result.edges if edge.fact}
    all_summaries = {node.summary for node in node_result.nodes}
```

#### 6. Platform-Specific Output Formats

**Twitter CSV Format (lines 1065-1114):**
```python
headers = ['user_id', 'name', 'username', 'user_char', 'description']
# user_char: Full persona for LLM system prompt
# description: Public-facing bio
```

**Reddit JSON Format (lines 1141-1188):**
```python
item = {
    "user_id": profile.user_id,
    "username": profile.user_name,
    "name": profile.name,
    "bio": profile.bio[:150],
    "persona": profile.persona,
    "age": profile.age or 30,
    "gender": self._normalize_gender(profile.gender),
    # ...
}
```

### Clever Solutions

1. **Temperature Annealing:** Lower temperature on retry (0.7 -> 0.6 -> 0.5) for more deterministic results.

2. **Fallback Profile Generation:** If LLM fails completely, generates rule-based profiles for continuity.

3. **Gender Normalization:**
   ```python
   gender_map = {"男": "male", "女": "female", "机构": "other", ...}
   ```

4. **Real-time Progress Saving:** Profiles written after each completion prevents data loss on failure.

### Technical Debt / Concerns

1. **No Profile Versioning:** Updates to profiles overwrite previous versions.

2. **LLM Cost Scaling:** 2000-character persona descriptions per agent can be expensive with large agent counts.

3. **Hardcoded Entity Types:** Adding new entity types requires code changes.

4. **Username Collision Risk:** Random 3-digit suffixes (`username_123`) could theoretically collide.

---

## Cross-Feature Integration

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

### Key Integration Points

1. **Graph ID Propagation:** `graph_id` flows from `GraphBuilder` through `ZepEntityReader` to `OasisProfileGenerator` for context enrichment.

2. **Entity-to-Profile Mapping:** `source_entity_uuid` and `source_entity_type` fields maintain lineage from Zep to OASIS.

3. **Simulation IPC:** `SimulationIPCClient` in Flask communicates with running simulation for interviews.

---

## Error Handling Patterns

### Consistent Retry with Exponential Backoff
```python
for attempt in range(max_retries):
    try:
        return func()
    except Exception as e:
        delay *= 2  # Exponential backoff
        time.sleep(delay)
```

### Graceful Degradation
- Profile generation falls back to rule-based if LLM fails
- Zep search failures don't abort profile generation
- IPC timeout returns error response, not exception

### Structured Error Propagation
```python
try:
    # operation
except Exception as e:
    logger.error(f"Operation failed: {e}")
    raise ValueError(f"context: {original_error}")
```

---

## Security Considerations

1. **File-based IPC is not authenticated** - Any process with filesystem access can send commands
2. **No rate limiting** on Zep API calls or LLM generations
3. **Subprocess execution** of user-provided config files could be risky if not sandboxed

---

## Performance Characteristics

| Operation | Typical Duration | Bottleneck |
|-----------|-----------------|-------------|
| Graph build (100 chunks) | 5-10 minutes | Zep API processing |
| Profile generation (50 agents) | 2-5 minutes | LLM API latency |
| Simulation (72h simulated) | Variable | OASIS + LLM |
| Zep search | 1-3 seconds | Network + Zep |

---

## Recommendations for Future Development

1. **Simulation Persistence:** Add pause/resume capability for long simulations
2. **Graph Versioning:** Support ontology updates to existing graphs
3. **Distributed IPC:** Replace file polling with message queue for better latency
4. **Cost Control:** Add LLM call caching and profile template reuse
5. **Monitoring:** Add metrics for Zep API latency, LLM token usage, simulation throughput
