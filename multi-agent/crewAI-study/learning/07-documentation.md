# Documentation

## Platform

CrewAI uses **Mintlify** for documentation hosting at `docs.crewai.com`.

```json
{
  "theme": "aspen",
  "name": "CrewAI",
  "navigation": {
    "languages": [
      {
        "language": "en",
        "versions": [{ "version": "v1.12.2", "default": true }]
      }
    ]
  }
}
```

## Documentation Structure

```
docs/
├── docs.json              # Mintlify navigation configuration
├── index.mdx              # Landing page
├── common-room-tracking.js
├── reo-tracking.js
├── images/                # Documentation images and assets
├── en/                    # English (default)
├── ar/                    # Arabic
├── ko/                    # Korean
└── pt-BR/                 # Brazilian Portuguese
```

## Multi-Language Support

| Language | Code | Status |
|----------|------|--------|
| English | `en` | Default |
| Arabic | `ar` | Complete |
| Korean | `ko` | Complete |
| Brazilian Portuguese | `pt-BR` | Complete |

Documentation states: "Edit the English version in `docs/en/` first, then update translations."

## Content Sections

### Main Sections (from docs.json navigation)

| Section | Pages | Description |
|---------|-------|-------------|
| **Welcome** | index | Landing page with quick links |
| **Get Started** | introduction, installation, quickstart | Onboarding flow |
| **Guides** | Strategy, Agents, Tools, Flows | How-to documentation |
| **API Reference** | introduction, various endpoints | Auto-generated API docs |
| **Concepts** | agents, crews, flows, tasks, knowledge, memory, etc. | Core concept explanations |
| **Tools** | individual tool READMEs | Tool-specific documentation |
| **Enterprise** | features, guides, deployment | Enterprise features |
| **Examples** | cookbooks, reference implementations | Real-world usage |
| **MCP** | Model Context Protocol docs | Protocol integration |
| **Observability** | telemetry, tracing | Monitoring setup |
| **Learn** | courses, tutorials | Educational content |

### Concepts (lib/crewai/docs/en/concepts)

```
agents.mdx          # Agent concepts
cli.mdx             # CLI reference
collaboration.mdx   # Multi-agent collaboration
crews.mdx           # Crew architecture
event-listener.mdx  # Event handling
files.mdx           # File operations
flows.mdx           # Flow orchestration
knowledge.mdx       # Knowledge management
llms.mdx            # LLM integration
memory.mdx          # Memory system
planning.mdx        # Task planning
processes.mdx       # Process types
production-architecture.mdx
reasoning.mdx       # Agent reasoning
skills.mdx          # Agent skills
tasks.mdx           # Task definitions
testing.mdx         # Testing strategies
tools.mdx           # Tool usage
training.mdx        # Model training
```

### Guides (lib/crewai/docs/en/guides)

```
advanced/
agents/
coding-tools/
concepts/
crews/
flows/
migration/
tools/
agents.mdx
...
```

### Tools Documentation

Individual tool documentation lives alongside code:
- `lib/crewai-tools/src/crewai_tools/tools/{tool}/README.md`

Notable tools with dedicated docs:
- RAG (Retrieval Augmented Generation)
- Code documentation search
- CSV, JSON, XML search tools
- Website scraping tools
- YouTube tools
- Apify, Firecrawl, Scrapegraph integrations

## API Reference

Auto-generated API documentation via Mintlify. Located at `/en/api-reference/introduction`.

## Landing Page Features

The main `docs/index.mdx` provides:

1. **Hero Section** - Marketing copy with CTA buttons
2. **Get Started Cards** - Links to Introduction, Installation, Quickstart
3. **Build the Basics Cards** - Agents, Flows, Tasks & Processes
4. **Enterprise Journey Cards** - Deploy, Triggers, Team management
5. **What's New Cards** - Latest feature highlights
6. **Community Links** - GitHub star prompt, community forum

## Code Examples in Docs

Documentation includes executable code blocks with syntax highlighting:

```python
from crewai import Agent, Crew, Process, Task

# Define agents
researcher = Agent(role="Researcher", goal="Find insights", backstory="...")

# Create tasks
research_task = Task(description="...", agent=researcher)

# Create crew
crew = Crew(agents=[researcher], tasks=[research_task], process=Process.sequential)

# Run
result = crew.kickoff()
```

## External Learning Resources

Linked from README and docs:
- **DeepLearning.ai Courses**: Multi AI Agent Systems with CrewAI, Practical Multi AI Agents
- **learn.crewai.com**: 100,000+ developers certified through community courses

## Changelog

`docs/en/changelog.mdx` - Version history with breaking changes and new features.

## Documentation Quality Assessment

### Strengths

1. **Multi-language** - Full translations for 3 languages beyond English
2. **Comprehensive concepts** - All core abstractions documented
3. **Tool documentation** - Each tool has dedicated README with examples
4. **API reference** - Auto-generated from code
5. **Enterprise docs** - Dedicated section for AMP Suite features
6. **MCP protocol** - Native support documentation
7. **Video tutorials** - YouTube embeds for visual learners
8. **Examples repository** - Separate `crewAI-examples` repo with real implementations

### Gaps and Areas for Improvement

1. **API documentation generation** - No evidence of auto-generated API docs from docstrings (pydocstyle disabled in ruff)
2. **Architecture diagrams** - No explicit architecture section showing system design
3. **Performance tuning** - Limited guidance on optimization
4. **Debugging guide** - No comprehensive debugging/troubleshooting section
5. **Migration guides** - Basic migration guide exists but may be incomplete
6. **Contributing guide depth** - CONTRIBUTING.md exists but could be more detailed
7. **Release process** - No public documentation of how releases work

## Documentation Maintenance

From CONTRIBUTING.md:
- Edit English version first
- Update translations to maintain parity
- Keep MDX/JSX syntax unchanged in translations
- Update `docs.json` navigation when adding new pages

## Deployment

Documentation is automatically deployed via Mintlify integration with GitHub.

## Related Repositories

- **crewAI-examples**: Real-world implementation examples
- **crewAI** (main): Core framework + documentation
