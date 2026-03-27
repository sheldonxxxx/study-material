# Documentation

## Overview

OASIS maintains comprehensive documentation hosted at https://docs.oasis.camel-ai.org/ using the Mintlify platform.

## Documentation Platform

| Aspect | Technology |
|--------|------------|
| Generator | Mintlify |
| Hosting | docs.oasis.camel-ai.org |
| Deployment | Automatic on push to main |
| Local Preview | `mintlify dev` command |

## Documentation Structure

```
docs/
├── docs.json              # Navigation and configuration
├── introduction.mdx      # Welcome page
├── quickstart.mdx         # Getting started guide
├── overview.mdx          # Project overview
├── development.mdx       # Development guidelines
├── api-reference/        # API documentation
├── key_modules/          # Core module docs
│   ├── environments
│   ├── agent_graph
│   ├── social_agent
│   ├── models
│   ├── toolkits
│   ├── platform
│   └── actions
├── user_generation/      # User profile generation docs
├── visualization/        # Visualization documentation
├── cookbooks/            # Example-driven tutorials
│   ├── twitter_simulation
│   ├── reddit_simulation
│   ├── sympy_tools_simulation
│   ├── search_tools_simulation
│   ├── custom_prompt_simulation
│   └── twitter_interview
└── essentials/          # Essential guides
```

## Navigation Structure (docs.json)

```json
{
  "tabs": [
    {
      "tab": "Guides",
      "groups": [
        "Get Started" -> introduction, quickstart
        "Overview" -> overview
        "Key Modules" -> environments, agent_graph, user_generation, social_agent, models, toolkits, platform, actions
        "Cookbooks" -> twitter_simulation, reddit_simulation, sympy_tools_simulation, search_tools_simulation, custom_prompt_simulation, twitter_interview
        "Visualization" -> visualization
      ]
    }
  ],
  "global": {
    "anchors": [
      "Documentation" -> docs.oasis.camel-ai.org
      "Community" -> discord
      "Blog" -> camel-ai.org/blogs/oasis
    ]
  }
}
```

## Documentation Guidelines

### Contribution Requirements
- Comprehensive docstrings for all classes and methods
- Google Python Style Guide for docstrings
- Raw docstrings with `r"""` prefix
- 79-character line limit
- Include Args, Returns sections where applicable

### Local Development
```bash
# Install Mintlify CLI
npm i -g mintlify

# Preview changes
cd docs
mintlify dev

# If issues arise
mintlify install
```

## README Documentation

The main README.md includes:
- Project banner and badges
- Feature overview
- Demo video embed
- Use case illustrations
- Quick start guide with code example
- News/changelog section
- Citation information
- Star history chart
- Community links and contact information

## Additional Documentation

| Document | Purpose |
|----------|---------|
| CONTRIBUTING.md | Contribution guidelines, code review process, sprint workflow |
| LICENSE | Apache 2.0 license terms |
| examples/experiment/README.md | Experiment examples |
| .container/README.md | Container deployment guide |

## Documentation Quality

- **Strengths**: Well-structured navigation, multiple tutorial formats (cookbooks, quickstart)
- **Gaps**: No API reference files found in `docs/api-reference/`, limited searchability mentioned
- **Maintenance**: Active development indicated by regular updates in README changelog
