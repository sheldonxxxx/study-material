# QMD Documentation

## Existing Documentation

### README.md

Comprehensive user-facing documentation including:
- Quick start guide with installation commands
- Collection and context management
- Search commands (`search`, `vsearch`, `query`)
- MCP server setup (Claude Desktop, Claude Code, HTTP transport)
- SDK/library usage with TypeScript examples
- Architecture diagram and search pipeline explanation
- Requirements and model information
- Data storage schema

Quality: High. Includes practical examples, output format explanations, and architecture diagrams.

### docs/SYNTAX.md

Detailed query document grammar specification covering:
- All search modes and their differences
- Output format options (JSON, CSV, MD, XML, files)
- Document retrieval commands
- Query flow and fusion strategy
- Score normalization explanation
- Environment variables

Quality: High. Well-structured with tables and code examples.

### CHANGELOG.md

Full version history using Keep a Changelog format:
- Structured entries under `[Unreleased]` and version headers
- Changes and fixes clearly separated
- External contributors credited with PR numbers
- Performance metrics included ("2.7x faster", "17x less memory")

Quality: High. Detailed, versioned history with good contributor recognition.

### CLAUDE.md

Developer conventions including:
- Runtime preference (Bun over Node)
- CLI command reference
- Collection and context management
- Development workflow
- Architecture notes
- Important restrictions (no auto-indexing, no direct DB modification, no compile)
- Release process reference

Quality: High. Practical developer guide with clear do/don't patterns.

### skills/release/SKILL.md

Release workflow documentation referenced by CLAUDE.md.

## Documentation Gaps

### Missing

| Document | Status |
|----------|--------|
| CONTRIBUTING.md | Does not exist |
| CODEOWNERS | Does not exist |
| MAINTAINERS | Does not exist |
| Issue/PR templates | Not visible in repository |
| Project roadmap | No public roadmap |

### Observed Gaps

1. **No CONTRIBUTING.md**: No explicit contribution guidelines in repository. Community follows PR-based workflow inferred from changelog patterns.

2. **No governance docs**: Single-maintainer project with no formal governance structure documented.

3. **No issue templates**: No structured bug/feature report templates.

4. **No roadmap**: No public visibility into planned features or future direction.

## Summary

| Aspect | Assessment |
|--------|------------|
| User documentation | Excellent (README, docs/SYNTAX) |
| Developer documentation | Good (CLAUDE.md) |
| Version history | Excellent (CHANGELOG) |
| Release process | Well-documented (skills/release/SKILL.md) |
| Contributing guidelines | Minimal/absent |
| Governance documentation | None |
| Issue templates | None |
| Roadmap | None |

The project has strong user and developer documentation. The main gaps are around community governance (contributing guidelines, issue templates, roadmap) which is understandable for a single-maintainer project with informal governance.
