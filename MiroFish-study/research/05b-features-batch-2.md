# MiroFish Feature Deep Dive - Batch 2

## Features Analyzed
- Ontology & Environment Generation
- Report Generation Agent
- Simulation Configuration Generator

---

## 1. Ontology & Environment Generation

**File:** `backend/app/services/ontology_generator.py` (16KB)

### Purpose
Extracts entity relationships from seed data and generates environment configurations. Creates the structural foundation for simulations by designing entity types and relationship types tailored for social media opinion simulation.

### Core Implementation

#### System Prompt Design
The `ONTOLOGY_SYSTEM_PROMPT` is remarkably detailed (150+ lines), specifying:
- 10 exact entity types required
- Last 2 must be fallback types: `Person` and `Organization`
- First 8 are concrete types derived from document content
- Clear exclusion rules: no abstract concepts like "opinions", "trends", "sentiment"

```python
# Lines 77-80: Fallback type enforcement
A. **兜底类型（必须包含，放在列表最后2个）**:
   - `Person`: 任何自然人个体的兜底类型
   - `Organization`: 任何组织机构的兜底类型
```

#### Key Method: `generate()`
```python
def generate(
    self,
    document_texts: List[str],
    simulation_requirement: str,
    additional_context: Optional[str] = None
) -> Dict[str, Any]:
```

Uses `chat_json` with temperature=0.3 and max_tokens=4096. Text truncated at 50,000 characters.

#### Validation & Post-processing (`_validate_and_process`)
Defensive programming with strict constraints:
```python
# Lines 288-289: Zep API limits
MAX_ENTITY_TYPES = 10
MAX_EDGE_TYPES = 10

# Lines 292-336: Fallback injection logic
# If LLM generates > 10 types, removes from end (preserving concrete types)
# If Person/Organization missing, automatically adds them
```

### Clever Solutions

1. **Automatic fallback type injection** - If LLM omits `Person` or `Organization`, they are automatically added (lines 318-336)

2. **Reserve slot calculation** - Before adding fallbacks, calculates if space is available; if not, removes concrete types from the end (lines 329-333)

3. **Code generation** - `generate_python_code()` method transforms ontology into Pydantic models for Zep integration (lines 347-452)

```python
# Lines 424-436: Generates ENTITY_TYPES and EDGE_TYPES dictionaries
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

### Error Handling
- Text truncation at 50K chars with notification to LLM
- Description length enforced at 100 chars max (lines 274-276, 284-285)
- Graceful degradation: missing fields get empty defaults

### Technical Debt / Shortcuts
1. **Hardcoded truncation** - `MAX_TEXT_LENGTH_FOR_LLM = 50000` is a magic number
2. **No validation of entity type names** - LLM could output invalid Python identifiers
3. **Limited error recovery** - If LLM call fails, exception propagates up

---

## 2. Report Generation Agent

**File:** `backend/app/services/report_agent.py` (99KB - largest service file)

### Purpose
AI agent with rich toolset that deeply interacts with simulation results to generate comprehensive prediction reports. Uses ReACT (Reasoning + Acting) pattern for multi-turn tool calling and reflection.

### Architecture

#### Dual Logging System
Two parallel logging mechanisms capture the report generation process:

**ReportLogger** (`ReportLogger` class, lines 35-303)
- Outputs JSONL format to `agent_log.jsonl`
- Records every action: `report_start`, `planning_start`, `react_thought`, `tool_call`, `tool_result`, `llm_response`, `section_content`, `section_complete`, `report_complete`, `error`
- Includes elapsed time tracking

```python
# Line 84-93: Log entry structure
log_entry = {
    "timestamp": datetime.now().isoformat(),
    "elapsed_seconds": round(self._get_elapsed_time(), 2),
    "report_id": self.report_id,
    "action": action,
    "stage": stage,
    "section_title": section_title,
    "section_index": section_index,
    "details": details
}
```

**ReportConsoleLogger** (`ReportConsoleLogger` class, lines 306-385)
- Outputs to `console_log.txt` with standard `[timestamp] LEVEL: message` format
- Attaches to multiple loggers: `mirofish.report_agent`, `mirofish.zep_tools`

#### Report Manager Pattern
`ReportManager` (lines 1883-2571) handles file-based persistence:
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

### ReACT Pattern Implementation

#### Section Generation Loop (`_generate_section_react`, lines 1220-1530)

**Iteration structure:**
1. LLM responds with thought/action
2. Parse tool calls from response
3. If Final Answer detected and tool calls >= 3, output content
4. If tool call present, execute and inject observation
5. Max 5 tool calls per section, min 3 required

**Conflict Handling** (lines 1327-1361):
When LLM outputs BOTH tool call AND Final Answer simultaneously:
```python
# First 2 conflicts: ask LLM to retry with corrected format
if conflict_retries <= 2:
    messages.append({"role": "user", "content": "【格式错误】..."})
    continue
# Third conflict: truncate to first tool_call, force execution
else:
    response = response[:first_tool_end + len('</tool_call>')]
```

**Graceful Degradation:**
- If LLM returns None (API failure), adds placeholder and retries (lines 1310-1318)
- If max iterations reached, forces final answer with timeout prompt (lines 1502-1520)

### Tool System

Four tools available to the agent (lines 918-953):

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

### Report Generation Flow

```python
# Lines 1532-1764: generate_report() method
1. Create report folder and initial state
2. Initialize loggers (ReportLogger + ReportConsoleLogger)
3. PLANNING phase: plan_outline() with context from zep_tools
4. GENERATING phase:
   - For each section: _generate_section_react()
   - Save section immediately after completion
5. ASSEMBLE phase: assemble_full_report() with title cleanup
```

### Formatting Rules (Strictly Enforced)

The system prompt specifies (lines 670-702):
- No markdown headings (`#`, `##`, etc.) in section content
- Use **bold** instead of sub-headers
- Quotes must be standalone paragraphs
- Section title auto-added by system, not LLM

```python
# Lines 2131-2196: _clean_section_content() post-processing
# Removes duplicate headings, converts ### to bold, cleans separators
```

### Clever Solutions

1. **Streaming-friendly design** - Each section saved immediately (line 1673) rather than waiting for full report

2. **JSONL logging with replay capability** - `get_agent_log_stream()` enables real-time log streaming to frontend

3. **Multiple tool call formats supported** (lines 1066-1111):
   - XML style: `<tool_call>{"name": "tool", ...}</tool_call>`
   - Bare JSON if response is single object with valid tool name

4. **LLM temperature annealing** - `temperature=0.7 - (attempt * 0.1)` reduces creativity on retries (simulation_config_generator.py line 449)

### Edge Cases & Error Handling

1. **Missing Final Answer prefix** (line 1488-1500):
   - If LLM outputs content but no "Final Answer:" prefix, adopts the output anyway

2. **Insufficient tool calls** (lines 1377-1389):
   - If Final Answer given before 3 tool calls, prompts to continue

3. **Report already exists** (report.py lines 72-84):
   - Checks for existing completed report before regenerating
   - Supports `force_regenerate=true` to override

4. **Chat mode** (lines 1766-1880):
   - Separate ReACT loop with max 2 tool calls
   - Report content included in context (15K char limit)

---

## 3. Simulation Configuration Generator

**File:** `backend/app/services/simulation_config_generator.py` (39KB)

### Purpose
Generates simulation parameters including agent counts, interaction rules, time windows, and platform-specific settings. Uses LLM for intelligent configuration with rule-based fallback.

### Configuration Dataclasses

```python
# Lines 50-80: AgentActivityConfig
@dataclass
class AgentActivityConfig:
    agent_id: int
    entity_uuid: str
    entity_name: str
    entity_type: str
    activity_level: float = 0.5  # 0.0-1.0
    posts_per_hour: float = 1.0
    comments_per_hour: float = 2.0
    active_hours: List[int] = field(default_factory=lambda: list(range(8, 23)))
    response_delay_min: int = 5
    response_delay_max: int = 60
    sentiment_bias: float = 0.0  # -1.0 to 1.0
    stance: str = "neutral"  # supportive, opposing, neutral, observer
    influence_weight: float = 1.0

# Lines 82-110: TimeSimulationConfig
@dataclass
class TimeSimulationConfig:
    total_simulation_hours: int = 72  # 3 days default
    minutes_per_round: int = 60
    peak_hours: List[int] = field(default_factory=lambda: [19, 20, 21, 22])
    off_peak_hours: List[int] = field(default_factory=lambda: [0, 1, 2, 3, 4, 5])
    # ... with multipliers

# Lines 112-126: EventConfig
@dataclass
class EventConfig:
    initial_posts: List[Dict[str, Any]] = field(default_factory=list)
    scheduled_events: List[Dict[str, Any]] = field(default_factory=list)
    hot_topics: List[str] = field(default_factory=list)
    narrative_direction: str = ""
```

### Step-by-Step Generation Strategy

The `generate_config()` method (lines 242-378) uses a batched approach:

```
Step 1: Time config generation
Step 2: Event config generation
Step 3-N: Batch agent configs (15 agents per batch)
Final: Platform config assignment
```

#### Progress Callback Pattern
```python
# Lines 278-283: Progress reporting
def report_progress(step: int, message: str):
    nonlocal current_step
    current_step = step
    if progress_callback:
        progress_callback(step, total_steps, message)
```

Total steps calculated dynamically:
```python
num_batches = math.ceil(len(entities) / self.AGENTS_PER_BATCH)  # 15 per batch
total_steps = 3 + num_batches  # time + event + N agent batches + platform
```

### China Timezone Configuration (Built-in)

```python
# Lines 27-47: CHINA_TIMEZONE_CONFIG
CHINA_TIMEZONE_CONFIG = {
    "dead_hours": [0, 1, 2, 3, 4, 5],           # 5% activity
    "morning_hours": [6, 7, 8],                  # 40% activity
    "work_hours": [9, 10, 11, 12, 13, 14, 15, 16, 17, 18],  # 70% activity
    "peak_hours": [19, 20, 21, 22],             # 150% activity
    "night_hours": [23],                         # 50% activity
    "activity_multipliers": {...}
}
```

### Rule-Based Agent Configuration Fallback

When LLM fails to generate agent configs, `_generate_agent_config_by_rule()` (lines 904-985) provides defaults:

```python
# Lines 908-920: University/Government agency defaults
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

# Lines 947-958: Student defaults (high activity)
elif entity_type in ["student"]:
    return {
        "activity_level": 0.8,
        "posts_per_hour": 0.6,
        "comments_per_hour": 1.5,
        "active_hours": [8, 9, 10, 11, 12, 13, 18, 19, 20, 21, 22, 23],
        "response_delay_min": 1,
        "response_delay_max": 15,  # Fast response
        "influence_weight": 0.8  # Lower influence
    }
```

### Intelligent Post-Assignment

`_assign_initial_post_agents()` (lines 725-808) matches initial posts to agents:

```python
# Lines 746-756: Type alias mapping
type_aliases = {
    "official": ["official", "university", "governmentagency", "government"],
    "university": ["university", "official"],
    "mediaoutlet": ["mediaoutlet", "media"],
    "student": ["student", "person"],
    ...
}
```

Fallback chain when no exact match:
1. Direct type lookup
2. Alias mapping
3. Use highest-influence agent (lines 789-797)

### JSON Resilience

`_call_llm_with_retry()` (lines 433-480) handles API failures:

```python
# Line 449: Temperature annealing
temperature=0.7 - (attempt * 0.1)  # Reduces from 0.7 to 0.5 to 0.3

# Lines 456-459: Truncation detection
if finish_reason == 'length':
    content = self._fix_truncated_json(content)

# Lines 468-470: JSON fix attempts
fixed = self._try_fix_config_json(content)
if fixed:
    return fixed
```

`_fix_truncated_json()` (lines 482-498):
```python
# Calculates unclosed brackets/braces
open_braces = content.count('{') - content.count('}')
open_brackets = content.count('[') - content.count(']')
# Closes them
content += ']' * open_brackets
content += '}' * open_braces
```

### Validation & Clamping

`_parse_time_config()` (lines 609-642) validates generated values:

```python
# Lines 616-622: Bounds checking
if agents_per_hour_min > num_entities:
    agents_per_hour_min = max(1, num_entities // 10)
if agents_per_hour_max > num_entities:
    agents_per_hour_max = max(agents_per_hour_min + 1, num_entities // 2)
```

### Clever Solutions

1. **Batch size optimization** - `AGENTS_PER_BATCH = 15` balances LLM context window vs. API call overhead

2. **Context reuse** - Same context passed to all batches (line 320) to maintain consistency

3. **Entity summarization for context** - `_summarize_entities()` (lines 408-431) groups by type and limits display:
   - Max 20 entities per type displayed
   - Each summary truncated to 300 chars

4. **Reasoning chain preservation** - `generation_reasoning` field accumulates LLM reasoning from each step (line 373)

5. **Automatic status correction** - simulation.py lines 323-331 auto-updates `preparing` -> `ready` when files detected

### Technical Debt / Shortcuts

1. **Magic numbers** throughout:
   - `AGENTS_PER_BATCH = 15`
   - `MAX_CONTEXT_LENGTH = 50000`
   - `TIME_CONFIG_CONTEXT_LENGTH = 10000`

2. **Platform configs hardcoded** (lines 341-358):
   ```python
   if enable_twitter:
       twitter_config = PlatformConfig(
           platform="twitter",
           recency_weight=0.4,
           popularity_weight=0.3,
           relevance_weight=0.3,
           viral_threshold=10,
           echo_chamber_strength=0.5
       )
   ```
   These values could be LLM-generated.

3. **Limited retry logic** - 3 attempts with fixed delays, no exponential backoff

4. **No validation of generated entity types** - If LLM invents new types, they pass through unchecked

---

## Cross-Feature Integration

### Data Flow
```
Ontology Generator
    ↓ (entity/edge types)
Graph Builder
    ↓ (entities with types)
Zep Entity Reader → Simulation Config Generator
    ↓ (agent configs)
Simulation Runner
    ↓ (simulation results)
Report Agent ← Zep Tools (search, interview)
```

### Shared Patterns

1. **Progress callbacks** - All three features use callback-based progress reporting for async operations

2. **File-based output** - Report Agent and Simulation both write to `Config.UPLOAD_FOLDER`

3. **LLM client reuse** - All use `LLMClient` from `../utils/llm_client.py`

4. **Zep integration** - Both Report Agent and Simulation Config use Zep services

5. **China-specific defaults** - Both Report Agent (timezone context) and Simulation Config (activity multipliers) incorporate Chinese user patterns

---

## Key Files Summary

| File | Size | Purpose |
|------|------|---------|
| `ontology_generator.py` | 16KB | Entity/relationship type generation |
| `report_agent.py` | 99KB | ReACT-based report generation with rich logging |
| `simulation_config_generator.py` | 39KB | Step-by-step simulation parameter generation |
| `zep_tools.py` | 66KB | Tool wrappers for Zep graph operations |
| `simulation.py` (API) | 95KB | Simulation orchestration endpoints |
| `report.py` (API) | 30KB | Report generation endpoints |

---

## Notable Implementation Patterns

1. **Temperature annealing** on retries (reducing creativity on each attempt)

2. **Graceful degradation** - When LLM fails, rule-based fallbacks ensure operation completes

3. **JSON repair** - Custom truncation detection and bracket closing for truncated LLM output

4. **Dual logging** - Both structured JSONL and human-readable console logs

5. **Immediate file persistence** - Sections/configs saved as generated, not at end

6. **Type alias mapping** - Flexible matching of entity types with fallbacks

7. **Batch processing** - Large workloads split into manageable chunks with progress tracking
