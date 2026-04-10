# MiroFish Feature Index

## Project Overview
MiroFish is a multi-agent based AI prediction engine that constructs high-fidelity parallel digital worlds from real-world seed information (news, policies, financial signals). Agents with independent personalities, long-term memory, and behavioral logic interact freely, enabling predictive simulations.

---

## Core Features (Priority Tier 1)

### 1. Multi-Agent Simulation Engine
**Description:** The core OASIS-powered simulation engine that runs parallel agent interactions. Supports dual-platform execution with real-time state management.

**Key Files:**
- `backend/app/services/simulation_runner.py` (68KB - main simulation logic)
- `backend/app/services/simulation_manager.py` (20KB - orchestration)
- `backend/app/services/simulation_ipc.py` (12KB - inter-process communication)
- `backend/scripts/run_parallel_simulation.py` (62KB)

**Priority:** CORE

---

### 2. Knowledge Graph Construction (GraphRAG)
**Description:** Extracts entity relationships from seed data and builds knowledge graphs with GraphRAG. Enables agents to query structured world knowledge.

**Key Files:**
- `backend/app/services/graph_builder.py` (17KB)
- `backend/app/services/zep_graph_memory_updater.py` (21KB)
- `backend/app/api/graph.py` (20KB)

**Priority:** CORE

---

### 3. Agent Persona/Profile Generation
**Description:** Generates distinct personalities, backgrounds, and behavioral patterns for simulation agents using LLM. Creates believable autonomous agents.

**Key Files:**
- `backend/app/services/oasis_profile_generator.py` (49KB)
- `backend/app/utils/llm_client.py` (3KB)

**Priority:** CORE

---

### 4. Ontology & Environment Generation
**Description:** Extracts entity relationships and generates environment configurations. Creates the structural foundation for simulations.

**Key Files:**
- `backend/app/services/ontology_generator.py` (16KB)

**Priority:** CORE

---

### 5. Report Generation Agent
**Description:** AI agent with rich toolset that deeply interacts with simulation results to generate comprehensive prediction reports.

**Key Files:**
- `backend/app/services/report_agent.py` (99KB - largest service file)
- `backend/app/api/report.py` (30KB)

**Priority:** CORE

---

### 6. Simulation Configuration Generator
**Description:** Generates simulation parameters including agent counts, interaction rules, time windows, and platform-specific settings.

**Key Files:**
- `backend/app/services/simulation_config_generator.py` (39KB)
- `backend/app/api/simulation.py` (95KB)

**Priority:** CORE

---

### 7. Interactive 5-Step Workflow UI
**Description:** Frontend wizard guiding users through simulation creation: Graph Build, Environment Setup, Simulation Run, Report Generation, and Deep Interaction.

**Key Files:**
- `frontend/src/views/Step1GraphBuild.vue` (17KB)
- `frontend/src/views/Step2EnvSetup.vue` (68KB)
- `frontend/src/views/Step3Simulation.vue` (39KB)
- `frontend/src/views/Step4Report.vue` (145KB)
- `frontend/src/views/Step5Interaction.vue` (64KB)

**Priority:** CORE

---

## Secondary Features (Priority Tier 2)

### 8. Long-Term Memory System (Zep Integration)
**Description:** Provides persistent memory for agents using Zep Cloud. Enables temporal memory updates and entity tracking across simulation runs.

**Key Files:**
- `backend/app/services/zep_entity_reader.py` (15KB)
- `backend/app/services/zep_tools.py` (66KB)
- `backend/app/utils/zep_paging.py` (4KB)

**Priority:** SECONDARY

---

### 9. Graph Visualization Panel
**Description:** Interactive knowledge graph visualization component showing entities, relationships, and simulation state.

**Key Files:**
- `frontend/src/components/GraphPanel.vue` (40KB)

**Priority:** SECONDARY

---

### 10. Simulation History & Project Management
**Description:** Database storage for simulation history, projects, and tasks. Enables resuming and comparing past simulations.

**Key Files:**
- `frontend/src/views/HistoryDatabase.vue` (34KB)
- `backend/app/models/project.py` (9KB)
- `backend/app/models/task.py` (6KB)

**Priority:** SECONDARY

---

### 11. Platform-Specific Simulations
**Description:** Pre-built simulation templates for different domains (Reddit, Twitter-style interactions) with specialized agent behaviors.

**Key Files:**
- `backend/scripts/run_reddit_simulation.py` (27KB)
- `backend/scripts/run_twitter_simulation.py` (27KB)

**Priority:** SECONDARY

---

### 12. Data Input Processing
**Description:** Text and document parsing for seed material. Supports various input formats for simulation initialization.

**Key Files:**
- `backend/app/services/text_processor.py` (2KB)
- `backend/app/utils/file_parser.py` (5KB)

**Priority:** SECONDARY

---

## Feature Map: README Claims vs Implementation

| README Claim | Implementation Location | Verified |
|-------------|------------------------|----------|
| GraphRAG Knowledge Graph | `graph_builder.py`, `zep_graph_memory_updater.py` | Yes |
| Entity/Relation Extraction | `ontology_generator.py` | Yes |
| Agent Persona Generation | `oasis_profile_generator.py` | Yes |
| Environment Config Agent | `simulation_config_generator.py` | Yes |
| Dual-Platform Parallel Simulation | `simulation_ipc.py`, `run_parallel_simulation.py` | Yes |
| Report Agent | `report_agent.py` | Yes |
| Deep Interaction | `Step5Interaction.vue`, `InteractionView.vue` | Yes |
| Memory System (Zep) | `zep_*.py` services | Yes |

---

## Architecture Summary

```
frontend/
├── views/           # 5-step workflow views
├── components/      # GraphPanel, etc.
├── api/             # API clients
└── router/          # Vue Router config

backend/
├── api/             # REST endpoints (graph, report, simulation)
├── services/        # Business logic (simulation, profile, report agents)
├── models/          # Data models (project, task)
└── utils/           # Utilities (LLM client, file parser, Zep paging)
```

---

## Workflow Steps (Frontend)

1. **Step1GraphBuild** - Upload seed data, build knowledge graph
2. **Step2EnvSetup** - Configure entities, personas, environment
3. **Step3Simulation** - Run multi-agent simulation
4. **Step4Report** - Generate and view prediction report
5. **Step5Interaction** - Deep dive interaction with agents and ReportAgent
