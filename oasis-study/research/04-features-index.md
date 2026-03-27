# OASIS Feature Index

**Project:** OASIS: Open Agent Social Interaction Simulations with One Million Agents
**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27

## Feature Prioritization Summary

| Priority | Count | Description |
|----------|-------|-------------|
| Core Features | 7 | Essential simulation capabilities |
| Secondary Features | 5 | Extended functionality |
| **Total** | **12** | |

---

## Core Features

### 1. Massive-Scale Agent Simulation
**Priority:** Tier 1 (Critical)
**Description:** Supports simulations of up to one million LLM-powered agents, enabling studies of social media dynamics at real-world platform scale.
**Key Files/Locations:**
- `oasis/social_agent/agents_generator.py` - Agent creation and graph management
- `oasis/social_agent/agent_graph.py` - AgentGraph class for managing large agent populations
- `examples/experiment/twitter_simulation_1M_agents/` - Million-agent demonstration
- `generator/twitter/` and `generator/reddit/` - Scale-out generation utilities

---

### 2. Multi-Platform Simulation (Twitter + Reddit)
**Priority:** Tier 1 (Critical)
**Description:** Simulates both Twitter and Reddit platforms with platform-specific action sets, interfaces, and social dynamics.
**Key Files/Locations:**
- `oasis/social_platform/platform.py` - Platform base class with Twitter/Reddit implementations
- `oasis/social_platform/typing.py` - `DefaultPlatformType` enum (TWITTER, REDDIT)
- `oasis/social_platform/schema/` - Database schemas for both platforms
- `examples/twitter_simulation_openai.py`, `examples/reddit_simulation_openai.py` - Platform examples

---

### 3. LLM-Powered Agent Decision Making
**Priority:** Tier 1 (Critical)
**Description:** Agents use large language models (GPT-4, GPT-4O-MINI, vLLM) to autonomously decide social media actions based on their profiles and context.
**Key Files/Locations:**
- `oasis/social_agent/agent.py` - SocialAgent class with LLM integration
- `oasis/environment/env_action.py` - `LLMAction` class for autonomous decisions
- `examples/different_model_simulation.py` - Multi-model support demonstration
- `examples/search_tools_simulation.py`, `examples/sympy_tools_simulation.py` - Tool-augmented agents

---

### 4. Diverse Social Action Space (23 Actions)
**Priority:** Tier 1 (Critical)
**Description:** Rich action space including posting, commenting, liking/disliking, following, searching, reposting, quoting, muting, group management, and more.
**Key Files/Locations:**
- `oasis/social_platform/typing.py` - `ActionType` enum (lines 17-49)
- `oasis/social_platform/platform.py` - Action implementations (~1600 lines)
- **Default Twitter Actions:** CREATE_POST, LIKE_POST, REPOST, FOLLOW, DO_NOTHING, QUOTE_POST
- **Default Reddit Actions:** LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT, SEARCH_POSTS, SEARCH_USER, TREND, FOLLOW, MUTE

---

### 5. Recommendation Systems
**Priority:** Tier 1 (Critical)
**Description:** Multiple recommendation algorithms including interest-based (Twhin-Bert embeddings, TF-IDF) and hot-score-based algorithms for content discovery.
**Key Files/Locations:**
- `oasis/social_platform/recsys.py` - Main recommendation system (~750 lines)
- `oasis/social_platform/process_recsys_posts.py` - Post vector generation
- `oasis/social_platform/typing.py` - `RecsysType` enum (TWITTER, TWHIN, REDDIT, RANDOM)
- `generator/rag.py` - RAG-based profile generation

---

### 6. Dynamic Environment State Management
**Priority:** Tier 1 (Critical)
**Description:** Real-time environment that adapts to agent actions, updates social graphs, and maintains simulation state across timesteps.
**Key Files/Locations:**
- `oasis/environment/env.py` - Main environment class
- `oasis/social_platform/platform.py` - Platform state management
- `oasis/social_platform/platform_utils.py` - Utility functions
- `oasis/environment/make.py` - Environment factory (`make()` function)

---

### 7. Database-Backed Persistence (SQLite)
**Priority:** Tier 1 (Critical)
**Description:** SQLite-based persistent storage for simulation data including users, posts, comments, likes, follows, traces, and recommendation tables.
**Key Files/Locations:**
- `oasis/social_platform/database.py` - Database interface
- `oasis/social_platform/schema/` - SQL schema definitions (15+ schema files)
- `oasis/social_platform/schema/user.sql`, `post.sql`, `comment.sql`, `follow.sql`, `like.sql`, etc.

---

## Secondary Features

### 8. Group Chat / Community Features
**Priority:** Tier 2 (Extended)
**Description:** Support for creating groups, joining/leaving groups, and sending messages within groups.
**Key Files/Locations:**
- `oasis/social_platform/platform.py` - `CREATE_GROUP`, `JOIN_GROUP`, `LEAVE_GROUP`, `SEND_TO_GROUP` actions
- `examples/group_chat_simulation.py` - Group chat demonstration
- `oasis/social_platform/schema/chat_group.sql`, `group_member.sql`, `group_message.sql`

---

### 9. Per-Agent Customization (Models, Tools, Prompts)
**Priority:** Tier 2 (Extended)
**Description:** Support for customizing each agent's model, tools, and system prompts independently.
**Key Files/Locations:**
- `examples/custom_model_simulation.py` - Per-agent model configuration
- `examples/custom_prompt_simulation.py` - Prompt customization
- `examples/search_tools_simulation.py`, `examples/sympy_tools_simulation.py` - Tool customization
- `oasis/social_agent/agent.py` - Agent configuration options

---

### 10. Interview Action (Agent-to-Agent QA)
**Priority:** Tier 2 (Extended)
**Description:** Agents can ask other agents specific questions and receive answers, enabling research interactions.
**Key Files/Locations:**
- `oasis/social_platform/typing.py` - `ActionType.INTERVIEW`
- `oasis/social_platform/platform.py` - `handle_interview()` method
- `examples/twitter_interview.py` - Interview demonstration

---

### 11. Content Reporting / Moderation
**Priority:** Tier 2 (Extended)
**Description:** Agents can report inappropriate content, supporting moderation studies.
**Key Files/Locations:**
- `oasis/social_platform/typing.py` - `ActionType.REPORT_POST`
- `oasis/social_platform/platform.py` - `handle_report_post()` method
- `examples/twitter_misinforeport.py` - Misinformation reporting demo
- `oasis/social_platform/schema/report.sql`

---

### 12. Agent Profile Generation & Visualization
**Priority:** Tier 2 (Extended)
**Description:** Tools for generating user profiles at scale and visualizing simulation results.
**Key Files/Locations:**
- `generator/` - Profile generation utilities (BA network, RAG, user generation)
- `visualization/` - Visualization tools (dynamic_follow_network, reddit/twitter simulations)
- `examples/experiment/user_generation_visualization.md` - User guide
- `data/` - Sample datasets (reddit, twitter_dataset, emall)

---

## Feature-to-File Mapping Quick Reference

| Feature | Primary Module | Key Files |
|---------|---------------|-----------|
| Massive Scale | `oasis/social_agent/` | agents_generator.py, agent_graph.py |
| Multi-Platform | `oasis/social_platform/` | platform.py, typing.py |
| LLM Agents | `oasis/social_agent/` | agent.py, env_action.py |
| Actions | `oasis/social_platform/` | typing.py, platform.py |
| Recommendations | `oasis/social_platform/` | recsys.py, process_recsys_posts.py |
| Environment | `oasis/environment/` | env.py, make.py |
| Persistence | `oasis/social_platform/` | database.py, schema/*.sql |
| Group Chat | `oasis/social_platform/` | platform.py (group actions) |
| Customization | `oasis/social_agent/` | agent.py + example files |
| Interview | `oasis/social_platform/` | typing.py, platform.py |
| Reporting | `oasis/social_platform/` | typing.py, platform.py |
| Generation/Visualization | `generator/`, `visualization/` | Various |

---

## README Key Features vs. Actual Implementation

| README Claim | Status | Implementation Evidence |
|--------------|--------|------------------------|
| 1 million agents | Verified | `examples/experiment/twitter_simulation_1M_agents/` exists |
| 23 actions | Verified | `ActionType` enum has 33 values (exceeds claim) |
| Dynamic environments | Verified | `platform.py`, `env.py` implement real-time state |
| Interest-based recsys | Verified | `recsys.py` with Twhin-Bert, TF-IDF implementations |
| Hot-score recsys | Verified | `recsys.py` includes hot-score algorithms |
| Twitter + Reddit | Verified | `DefaultPlatformType` enum + platform.py |

---

*Generated for OASIS project analysis*
