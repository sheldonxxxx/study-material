# Documentation

## Overview

The AI SDK maintains comprehensive documentation across multiple channels: in-repo MDX content, API reference documentation generated from source, CONTRIBUTING guides for contributors, and external documentation at ai-sdk.dev.

## Documentation Structure

### In-Repo Content (`content/`)

```
content/
  docs/                      # Main documentation
    00-introduction/         # Getting started intro
    02-getting-started/     # Quick start guides
    02-foundations/         # Core concepts
    03-ai-sdk-core/         # Core API reference
    03-agents/              # Agent patterns
    04-ai-sdk-ui/           # UI integrations
    05-ai-sdk-rsc/          # React Server Components
    06-advanced/            # Advanced topics
    07-reference/          # General reference
    08-migration-guides/    # Version migration
    09-troubleshooting/     # Debug guides
  providers/                 # Provider documentation
    openai/
    anthropic/
    google/
    google-vertex/
    amazon-bedrock/
    azure/
    # ... 30+ provider docs
  cookbook/                  # Usage recipes
  tools-registry/            # Tools documentation
```

### Contributing Guides (`contributing/`)

| File | Purpose |
|------|---------|
| `add-new-provider.md` | Guide for adding new AI providers |
| `add-new-model.md` | Guide for adding new models |
| `add-new-tool-to-registry.md` | Tool registry contributions |
| `testing.md` | Testing standards and patterns |
| `documentation.md` | Doc contribution guidelines |
| `provider-architecture.md` | Provider implementation patterns |
| `packages.md` | Package structure overview |
| `building-new-features.md` | Feature development guide |
| `codemods.md` | Codemod contribution guide |
| `naming-conventions.md` | File and function naming |
| `project-philosophies.md` | Core design philosophies |
| `pre-release-cycle.md` | Release process details |
| `releases.md` | Release management |
| `zod.md` | Zod 3/4 usage guidelines |
| `decisions/` | Architecture Decision Records |

### Architecture Decision Records

Located in `contributing/decisions/`, these capture key architectural choices with:
- Status (proposed/accepted/deprecated)
- Context and problem statement
- Decision and rationale
- Consequences

## Documentation Generation

### Package Documentation

The `ai` package embeds documentation at build time:

```json
"scripts": {
  "prepack": "cp -r ../../content/docs ./docs",
  "postpack": "del-cli docs"
}
```

When the package is packed for publishing, MDX docs are copied from `content/docs` into the package for consumption by the documentation site.

### API Documentation

API docs are generated from:
- TypeScript source with JSDoc comments
- Inline documentation in MDX files
- Provider specification interfaces

## Documentation Standards

### MDX Content Guidelines

From `contributing/documentation.md`:

1. **Clarity**: Use clear, concise language
2. **Examples**: Include runnable code examples
3. **Code Style**: Follow project conventions (see naming-conventions.md)
4. **Links**: Cross-reference related docs
5. **Versioning**: Note breaking changes in migration guides

### File Organization

- Documentation files: kebab-case.mdx
- Provider docs: Organized by provider in `content/providers/`
- Examples: Place in `examples/` directory with descriptive paths

### Doc Precommit Hook

- lint-staged processes MDX files for formatting
- ultracite fix runs on staged documentation changes

## External Documentation

### Documentation Site

**URL**: https://ai-sdk.dev/docs

The site consumes:
- MDX content from `content/docs`
- Provider documentation from `content/providers`
- Package-level docs embedded during prepack

### API Reference

Auto-generated from TypeScript source:
- Type definitions in `packages/*/src/**/*.ts`
- JSDoc comments preserved in build output
- Export signatures from `index.ts` files

## Documentation Contribution Workflow

1. **Small fixes**: Edit directly on GitHub or via github.dev (press `.`)
2. **Larger changes**: Follow code contribution workflow
3. **Prettier issues**: Run `ultracite fix` to resolve formatting
4. **Preview locally**: Use Next.js dev server for content preview

## README Files

Each package includes a README.md with:
- Package purpose and overview
- Installation instructions
- Basic usage example
- Link to full documentation

Package READMEs are published to npm along with the package.

## Documentation for Providers

Provider-specific documentation exists for each AI provider:
- Authentication requirements
- Model availability
- Provider-specific options
- Error handling differences

Structure in `content/providers/<provider-name>/`:
- Setup guides
- Usage examples
- API differences from core SDK
