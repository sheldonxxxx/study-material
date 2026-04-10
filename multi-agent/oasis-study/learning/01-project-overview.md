# OASIS Project Overview

## Project Purpose

**OASIS (Open Agent Social Interaction Simulations)** is a scalable, open-source social media simulator that leverages Large Language Model (LLM) agents to realistically emulate the behavior of up to one million users on platforms like Twitter and Reddit. The project is developed by [CAMEL-AI.org](https://www.camel-ai.org/) and published on PyPI as `camel-oasis`.

## Project Scope

### Core Capabilities

- **Massive Scale**: Supports simulations of up to 1,000,000 agents, enabling social dynamics research at real-world platform scale
- **Platform Simulation**: Twitter and Reddit platform simulations with realistic user interactions
- **Diverse Actions**: 23 distinct agent actions including posting, commenting, liking, following, reposting, searching, and trending
- **Recommendation Systems**: Built-in recommendation algorithms (Twhin-Bert, Reddit-style hot scoring, random, trace-based personalization)
- **Dynamic Environment**: Real-time social network adaptation and content evolution

### Use Cases

- **Social Science Research**: Study information spread, group polarization, herd behavior, and complex social phenomena
- **Recommendation System Development**: Test and benchmark recommendation algorithms at scale
- **Content Moderation Research**: Simulate and analyze content reporting and moderation patterns
- **AI Safety Research**: Understand LLM agent behavior in social contexts

## Technical Architecture

### Package Structure

```
oasis/
├── clock/           # Simulation time management
├── environment/     # OASIS environment orchestration (OasisEnv)
├── social_agent/    # Agent implementation (SocialAgent, AgentGraph)
└── social_platform/ # Platform simulation (Twitter/Reddit)
    ├── config/      # User profiles, Neo4j configuration
    ├── database.py  # SQLite schema and operations
    ├── platform.py # Platform action handlers
    ├── recsys.py   # Recommendation system algorithms
    └── typing.py   # ActionType enums, RecsysType enums
```

### Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| camel-ai | 0.2.78 | LLM agent framework |
| pandas | 2.2.2 | Data manipulation |
| igraph | 0.11.6 | Social network graph operations |
| neo4j | 5.23.0 | Graph database (optional) |
| pytest | 8.2.0 | Testing framework |
| sentence-transformers | 3.0.0 | Embedding for Twhin-Bert recsys |

### Database Schema

The simulation uses SQLite with 16 tables including:
- `user`, `post`, `follow`, `mute` - Core social entities
- `like`, `dislike`, `comment`, `comment_like`, `comment_dislike` - Engagement tracking
- `trace` - Action history for all agent interactions
- `rec` - Recommendation cache per user
- `report` - Content moderation tracking
- `chat_group`, `group_members`, `group_messages` - Group functionality
- `product` - E-commerce functionality (Reddit mall demo)

## Version and Release

- **Current Version**: 0.2.5
- **License**: Apache 2.0
- **Python Support**: 3.10 - 3.11
- **Published**: PyPI (`pip install camel-oasis`)

## Documentation

- **Documentation Site**: [docs.oasis.camel-ai.org](https://docs.oasis.camel-ai.org/)
- **Research Paper**: [arXiv:2411.11581](https://arxiv.org/abs/2411.11581)
- **Dataset**: [HuggingFace](https://huggingface.co/datasets/echo-yiyiyi/oasis-dataset)

## Quick Start Example

```python
import asyncio
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType
import oasis
from oasis import ActionType, LLMAction, generate_reddit_agent_graph

async def main():
    openai_model = ModelFactory.create(
        model_platform=ModelPlatformType.OPENAI,
        model_type=ModelType.GPT_4O_MINI,
    )
    agent_graph = await generate_reddit_agent_graph(
        profile_path="./data/reddit/user_data_36.json",
        model=openai_model,
        available_actions=[ActionType.LIKE_POST, ActionType.CREATE_POST, ...],
    )
    env = oasis.make(
        agent_graph=agent_graph,
        platform=oasis.DefaultPlatformType.REDDIT,
        database_path="./reddit_simulation.db",
    )
    await env.reset()
    # Run simulation...
```

## Related Research

- [MultiAgent4Collusion](https://github.com/renqibing/MultiAgent4Collusion) - Multi-agent collusion simulation
- [CUBE](https://github.com/echo-yiyiyi/cube) - Unity3D environment simulations
- [MultiAgent4Fraud](https://github.com/zheng977/MutiAgent4Fraud) - Financial fraud detection research
