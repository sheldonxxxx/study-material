# Documentation

## Documentation Structure

### Primary Documentation

| File | Purpose | Lines |
|------|---------|-------|
| `README.md` | Main landing page, installation, workflow overview | 188 |
| `CHANGELOG.md` | Version history with categorized changes | ~100+ |
| `RELEASE-NOTES.md` | User-facing release summary | - |
| `CODE_OF_CONDUCT.md` | Contributor Covenant 2.0 | 129 |
| `LICENSE` | MIT License | - |

### Platform-Specific Documentation

| File | Platform | Purpose |
|------|----------|---------|
| `docs/README.codex.md` | Codex | Installation, configuration, troubleshooting |
| `docs/README.opencode.md` | OpenCode | Installation, plugin config, tool mapping |
| `.codex/INSTALL.md` | Codex | Detailed installation steps |
| `.opencode/INSTALL.md` | OpenCode | Detailed installation steps |
| `gemini-extension.json` | Gemini CLI | Extension manifest |

### Skill Documentation

Each skill follows a standardized structure:
```
skills/
  skill-name/
    SKILL.md           # Required: main reference
    supporting-file.*   # Optional: heavy reference, tools
```

**Skill Format**: YAML frontmatter + Markdown body

**Required Frontmatter**:
- `name`: Letters, numbers, hyphens only (max 1024 chars total)
- `description`: Third-person, "Use when..." format, describes triggering conditions NOT workflow

**SKILL.md Template Sections**:
1. Overview (1-2 sentences)
2. When to Use (symptoms, situations, contexts)
3. Core Pattern (before/after, code comparisons)
4. Quick Reference (tables, bullet lists)
5. Implementation (inline or linked files)
6. Common Mistakes

### Reference Documentation

| File | Purpose |
|------|---------|
| `skills/systematic-debugging/root-cause-tracing.md` | Debugging technique |
| `skills/systematic-debugging/defense-in-depth.md` | Security pattern |
| `skills/systematic-debugging/condition-based-waiting.md` | Async technique |
| `skills/writing-skills/anthropic-best-practices.md` | Anthropic guidelines |
| `skills/writing-skills/persuasion-principles.md` | Cialdini-based persuasion |
| `skills/brainstorming/visual-companion.md` | Visual aid companion |

## Documentation Quality

### Strengths

1. **Comprehensive Skill Library**: 12+ skills covering TDD, debugging, collaboration
2. **Standardized Format**: Consistent SKILL.md structure across all skills
3. **Cross-Platform Coverage**: Platform-specific docs for Claude Code, Cursor, Codex, OpenCode, Gemini
4. **Real Examples**: Skills include actual code examples, not templates
5. **Anti-Patterns Documented**: Explicit "don't do this" sections
6. **Flowcharts**: Graphviz diagrams for decision processes
7. **TDD Methodology**: Skill writing itself follows TDD principles

### Documentation Standards

**YAML Frontmatter Requirements**:
- Max 1024 characters total
- `name`: alphanumeric + hyphens only
- `description`: Must start with "Use when..."
- Third-person perspective
- Include error messages, symptoms, tools as keywords

**Claude Search Optimization (CSO)**:
- Rich description field for skill discovery
- Keyword coverage (error messages, symptoms, synonyms)
- Descriptive naming (verb-first, gerunds for processes)
- Token efficiency (<200 words for frequently-loaded)

### Code Examples

**Standards**:
- One excellent example beats many mediocre ones
- Complete and runnable
- Well-commented explaining WHY
- From real scenarios
- Shows pattern clearly

**Anti-patterns explicitly avoided**:
- Narrative examples ("In session 2025-10-03...")
- Multi-language dilution (separate .js, .py, .go files)
- Generic labels (helper1, step3)
- Code in flowcharts

## Documentation Gaps

### Not Observed
- **No API documentation**: This is a documentation/agent skill library, not an API
- **No architecture decision records (ADRs)**: Why decisions were made
- **No inline comments in shell scripts**: Hook scripts lack inline documentation
- **No docs for `agents/` directory**: Barely documented
- **No docs for `commands/` directory**: Barely documented
- **No type definitions**: No TypeScript, no .d.ts files

## Contributing Documentation

### PR Template Requirements
- Problem statement (what broke, failure mode)
- Change description (1-3 sentences)
- Appropriateness for core library check
- Alternatives considered
- Related PRs review
- Environment tested (harness, version, model)
- Evaluation methodology
- Rigor checklist (adversarial testing, human review)

### Skill Creation Process
Documented in `skills/writing-skills/SKILL.md`:
1. RED Phase: Write failing test (pressure scenarios)
2. GREEN Phase: Write minimal skill
3. REFACTOR Phase: Close loopholes

### Code of Conduct
Contributor Covenant 2.0 with:
- 4-level enforcement (Correction, Warning, Temporary Ban, Permanent Ban)
- Community Impact Guidelines
- Enforcement contact: jesse@primeradiant.com
