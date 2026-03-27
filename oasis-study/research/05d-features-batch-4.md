# OASIS Feature Deep Dive: Batch 4 (Features 10-12)

**Project:** OASIS: Open Agent Social Interaction Simulations
**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27
**Features Analyzed:**
- Feature 10: Interview Action (Agent-to-Agent QA)
- Feature 11: Content Reporting / Moderation
- Feature 12: Agent Profile Generation & Visualization

---

## Feature 10: Interview Action (Agent-to-Agent QA)

### Summary
Enables research interactions where agents can be asked specific questions and provide answers, facilitating study of agent reasoning and behavior in social contexts.

### Core Implementation

#### Action Definition (`typing.py`, line 44)
```python
INTERVIEW = "interview"
```
The `INTERVIEW` action is defined in the `ActionType` enum but notably is NOT included in either `get_default_twitter_actions()` or `get_default_reddit_actions()`, making it an external-only action that agents cannot autonomously select.

#### Handler Method (`platform.py`, lines 1348-1392)
```python
async def interview(self, agent_id: int, interview_data):
    """Interview an agent with the given prompt and record the response."""
```

**Key Implementation Details:**

1. **Dual Format Support**: The method handles two input formats:
   - **Old format (string)**: Just the prompt - response is None
   - **New format (dict)**: Contains both `prompt` and `response` fields

2. **Interview ID Generation**: `interview_id = f"{current_time}_{user_id}"` - concatenates timestep and user ID

3. **Trace Recording**: Interview data is stored in the `trace` table with structure:
   - `user_id`: Agent being interviewed
   - `action`: "interview"
   - `info`: JSON containing `{prompt, response?, interview_id}`
   - `created_at`: Simulation timestep

4. **Action Routing** (`platform.py`, line 148): Uses `getattr(self, action.value, None)` pattern - calls `self.interview()` directly

#### Usage Example (`examples/twitter_interview.py`)

The example demonstrates a misinformation study workflow:

1. **Timestep 1**: Agent 0 creates false post ("Earth is flat")
2. **Timestep 2**: Agent 1 posts correction
3. **Timestep 3**: Agent 2 reports original post as misinformation
4. **Timestep 4**: Agent 3 reposts original post
5. **Timestep 5**: Agent 4 reports the repost
6. **Timestep 8**: Multiple agents interviewed about their actions

```python
actions_8[env.agent_graph.get_agent(0)] = ManualAction(
    action_type=ActionType.INTERVIEW,
    action_args={
        "prompt": ("Your post about Earth being flat has been "
                   "reported multiple times. What are your thoughts on this?")
    })
```

**Query for Results:**
```python
cursor.execute("""
    SELECT user_id, info, created_at
    FROM trace
    WHERE action = ?
""", (ActionType.INTERVIEW.value, ))
```

### Technical Debt / Observations

1. **Response Handling**: When `interview_data` is a string (old format), `response` is set to `None` but still recorded in trace. The example code shows researchers manually setting response via dict format, implying the LLM doesn't auto-generate responses.

2. **No Validation**: No validation that `interview_data` contains required fields in dict format - missing keys result in empty strings.

3. **Single-Agent Focus**: Each interview action only targets one agent. No batch interview mechanism.

4. **Trace-Only Storage**: Interviews are only stored in the trace table, not a dedicated interview table with schema.

---

## Feature 11: Content Reporting / Moderation

### Summary
Allows agents to report inappropriate content, supporting moderation studies and enabling simulation of content moderation dynamics.

### Core Implementation

#### Action Definition (`typing.py`, line 27)
```python
REPORT_POST = "report_post"
```

#### Handler Method (`platform.py`, lines 1394-1446)
```python
async def report_post(self, agent_id: int, report_message: tuple):
    post_id, report_reason = report_message
```

**Key Implementation Details:**

1. **Duplicate Prevention**: Before inserting, checks if report record already exists:
   ```python
   check_report_query = "SELECT * FROM report WHERE user_id = ? AND post_id = ?"
   ```
   Returns `{"success": False, "error": "Report record already exists."}` if duplicate.

2. **Post Validation**: Verifies post exists via `_get_post_type()` before processing.

3. **Report Counter Update**: Increments `num_reports` column on the post:
   ```python
   UPDATE post SET num_reports = num_reports + 1 WHERE post_id = ?
   ```

4. **Report Storage** (`schema/report.sql`):
   ```sql
   CREATE TABLE report (
       report_id INTEGER PRIMARY KEY AUTOINCREMENT,
       user_id INTEGER,
       post_id INTEGER,
       report_reason TEXT,
       created_at DATETIME,
       FOREIGN KEY(user_id) REFERENCES user(user_id),
       FOREIGN KEY(post_id) REFERENCES post(post_id)
   );
   ```

5. **Trace Recording**: Records action in trace table with `{post_id, report_id}`.

#### Usage Example (`examples/twitter_misinforeport.py`)

```python
actions_3[env.agent_graph.get_agent(2)] = ManualAction(
    action_type=ActionType.REPORT_POST,
    action_args={
        "post_id": 1,
        "report_reason": ("This post spreads dangerous misinformation about science.")
    })
```

### Relationship Between Report and Interview

The `twitter_interview.py` example shows these features working together:
- Reports trigger warning messages on posts (posts with reports get flagged)
- Interview questions reference report actions ("Your post...has been reported multiple times")
- Researchers can study how agents respond to moderation actions

### Technical Debt / Observations

1. **No Report Resolution Mechanism**: Reports are stored but never acted upon - no mechanism to hide, delete, or penalize reported content within the simulation.

2. **Post-Only Reporting**: Can only report posts, not comments (despite LIKE_COMMENT, DISLIKE_COMMENT actions existing).

3. **No Report Categories**: `report_reason` is freeform text - no predefined categories for analysis.

4. **No Threshold Actions**: No automatic actions when `num_reports` exceeds a threshold (e.g., auto-hide at 5 reports).

5. **Missing Index**: The `report` table has no index on `post_id` for efficient queries on all reports for a post.

---

## Feature 12: Agent Profile Generation & Visualization

### Summary
Comprehensive tooling for generating realistic user profiles at scale (using demographic distributions and LLM) and visualizing simulation results (propagation graphs, follow networks, sentiment analysis).

### Part A: Profile Generation

#### Twitter Profile Generation

**Pipeline (`generator/twitter/`):**

1. **`gen.py`** - Main profile generator using RAG
2. **`rag.py`** - RAG-based profile enhancement with Chroma vector DB
3. **`ba.py`** - Barabási-Albert network model for follow relationships
4. **`network.py`** - Network edge generation

**Demographic Distributions (`gen.py`):**
```python
ages = ["13-17", "18-24", "25-34", "35-49", "50+"]
p_ages = [0.066, 0.171, 0.385, 0.207, 0.171]

mbtis = ["ISTJ", "ISFJ", "INFJ", "INTJ", ...]  # 16 types
p_mbti = [0.12625, 0.11625, 0.02125, ...]  # realistic distribution

genders = ["male", "female", "other"]
p_genders = [0.4, 0.4, 0.2]
```

**RAG-Based Enhancement (`rag.py`):**

1. **Vector Store**: Chroma with BGE-m3 embeddings
   ```python
   model_name = "BAAI/bge-m3"
   vectorstore = Chroma(persist_directory='./bge', embedding_function=hf)
   retriever = vectorstore.as_retriever(top_k=3)
   ```

2. **Profile Schema (`rag.py`, lines 78-82)**:
   ```python
   class User(BaseModel):
       realname: str
       username: str
       bio: str
       persona: str  # fictional background story
   ```

3. **Prompt Template** includes age, gender, MBTI, profession, and topic interests to generate contextual profiles.

4. **Topic Selection**: 9 topics (Politics, Urban Legends, Business, Terrorism & War, Science & Technology, Entertainment, Natural Disasters, Health, Education)

**BA Network Generation (`ba.py`, `network.py`):**

1. **Follow Relationships**: Topic-based following using `new_stars.csv` reference data
2. **BA Model**: Creates scale-free network following Barabási-Albert preferential attachment
3. **Output**: CSV with `user_id`, `following_agentid_list`, `activity_level`

#### Reddit Profile Generation

**Simpler Pipeline (`generator/reddit/user_generate.py`):**

1. **Demographics**:
   - Gender: 35.1% female, 63.6% male (based on Reddit survey data)
   - Age: 44% 18-29, 31% 30-49, 11% 50-64, 3% 65+, 11% underage
   - Country: 48.3% US, 7.3% UK, 7.0% Canada, etc.

2. **LLM-Based Generation**: Direct GPT-3.5-turbo calls (no RAG)

3. **Topics**: Economics, IT, Culture & Society, General News, Politics, Business, Fun

4. **Parallel Generation**: ThreadPoolExecutor with 100 workers for scale

### Part B: Visualization Tools

#### 1. Dynamic Follow Network (`visualization/dynamic_follow_network/`)

**Neo4j Visualization** (`vis_neo4j_twitter.py`):

```python
# Creates nodes and relationships in Neo4j from SQLite
session.execute_write(create_user_node, user_id, info_dict, created_at)
session.execute_write(create_follow_relationship, follower_id, followee_id, created_at)
```

**Usage Flow:**
1. Run simulation -> SQLite database
2. Export to Neo4j via visualization script
3. Explore via Neo4j browser - filter by `follow-timestamp` for temporal views

#### 2. Twitter Propagation Analysis (`visualization/twitter_simulation/align_with_real_world/`)

**Graph Metrics** (`graph.py`):

```python
class prop_graph:
    def build_graph(self):
        # Builds repost propagation graph from "repost from X" content

    def plot_depth_time(self):
        # Tracks how deep misinformation spreads over time

    def plot_scale_time(self):
        # Tracks total users reached over time

    def plot_max_breadth_time(self):
        # Tracks widest layer of propagation

    def plot_structural_virality_time(self):
        # Average shortest path length (how viral)
```

**Metrics Computed:**
- `total_depth`: Maximum propagation depth
- `total_scale`: Total nodes reached
- `total_max_breadth`: Widest propagation layer
- `total_structural_virality`: `nx.average_shortest_path_length(undirected_graph)`

#### 3. Group Polarization (`visualization/twitter_simulation/group_polarization/`)

**`group_polarization_eval.py`**: Compares answer extremism across simulation rounds using GPT-4o-mini to rank which answer is more radical.

#### 4. Reddit Analysis (`visualization/reddit_simulation_*`)

**Score Analysis**: Compares upvoted/downvoted/control group behaviors
**Counterfactual Analysis**: Analyzes "what-if" scenarios by manipulating content visibility

### Technical Debt / Observations

1. **Hardcoded Paths**: Many visualization scripts have hardcoded paths like `path/to/the/first/round/eval.csv`

2. **Missing Dependencies**: Twitter generator requires `complete_user_char.csv` and `new_stars.csv` that don't exist in repo

3. **API Key in Code**: Reddit generator has `client = OpenAI(api_key='sk-xxx')` - placeholder must be manually replaced

4. **No Unified Visualization Pipeline**: Each visualization tool is standalone with different DB formats and outputs

5. **No Real-Time Visualization**: All visualization is post-hoc; no live dashboard during simulation

6. **NetworkX Dependency**: Heavy use of NetworkX for graph algorithms - no native graph database for large-scale analysis

7. **RAG Model Hardcoded**: BGE-m3 model hardcoded to `cuda:0` - no CPU fallback or config option

---

## Cross-Feature Observations

1. **Research Focus**: All three features are designed for research用例 (interviews, moderation studies, behavior analysis) rather than production simulation.

2. **Manual Action Dependency**: Both Interview and Report require `ManualAction` - not available to LLM-driven agents as autonomous choices.

3. **Trace Table Centrality**: Both Interview and Report store metadata in the generic `trace` table rather than specialized tables.

4. **No Feedback Loops**: Report counts are tracked but don't influence agent behavior or content visibility in current implementation.

5. **Study-Specific Design**: Features appear designed for specific research studies (misinformation, content moderation) rather than general-purpose simulation.

---

## Files Referenced

| Feature | File | Purpose |
|---------|------|---------|
| 10 | `oasis/social_platform/typing.py` | `INTERVIEW` action enum |
| 10 | `oasis/social_platform/platform.py:1348-1392` | `interview()` handler |
| 10 | `oasis/social_platform/schema/trace.sql` | Trace table schema |
| 10 | `examples/twitter_interview.py` | Usage example |
| 11 | `oasis/social_platform/typing.py` | `REPORT_POST` action enum |
| 11 | `oasis/social_platform/platform.py:1394-1446` | `report_post()` handler |
| 11 | `oasis/social_platform/schema/report.sql` | Report table schema |
| 11 | `examples/twitter_misinforeport.py` | Usage example |
| 12 | `generator/twitter/gen.py` | Twitter profile generator |
| 12 | `generator/twitter/rag.py` | RAG-based profile enhancement |
| 12 | `generator/twitter/ba.py` | BA network generation |
| 12 | `generator/twitter/network.py` | Edge generation |
| 12 | `generator/reddit/user_generate.py` | Reddit profile generator |
| 12 | `visualization/dynamic_follow_network/code/vis_neo4j_*.py` | Neo4j export |
| 12 | `visualization/twitter_simulation/align_with_real_world/code/graph.py` | Propagation analysis |
| 12 | `visualization/twitter_simulation/group_polarization/group_polarization_eval.py` | Polarization analysis |
| 12 | `examples/experiment/user_generation_visualization.md` | User documentation |
