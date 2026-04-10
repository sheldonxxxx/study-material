# Design Patterns in agency-agents

## Pattern Index

| # | Pattern | Category | Evidence |
|---|---------|----------|----------|
| 1 | Schema + Content Separation | Structure | YAML frontmatter vs markdown body |
| 2 | Visitor/Transformer | Transformation | `convert.sh` per-tool converters |
| 3 | Accumulator | Transformation | Aider/Windsurf single-file outputs |
| 4 | Factory | Creational | `slugify()` identifier generation |
| 5 | Template Method | Behavioral | Agent body section conventions |
| 6 | Strategy | Behavioral | Color resolution per tool |
| 7 | Facade | Structural | `install.sh` abstracts config complexity |
| 8 | Domain-Driven Organization | Architectural | 13 domain directories |
| 9 | Single Source of Truth | Architectural | One definition, many formats |
| 10 | Canonical Naming | Naming | Kebab-case slugs from human names |

---

## Pattern 1: Schema + Content Separation

**Category:** Structure
**Intent:** Separate machine-readable metadata from human-readable content

**Implementation:**

Every agent file contains YAML frontmatter followed by markdown body:

```
---
name: Agent Name
description: One-line description
color: cyan
emoji: 🎯
vibe: Personality hook
---
## Identity

[Human-readable content]
```

**Evidence:**

From `engineering/engineering-frontend-developer.md`:
```yaml
---
name: Frontend Developer
description: Building pixel-perfect, performant web interfaces
color: "#06B6D4"
emoji: 🎨
vibe: Zero-tolerance for layout shift
---
```

**Benefit:** Parsers extract metadata without reading body. Writers edit human-readable content directly.

---

## Pattern 2: Visitor/Transformer

**Category:** Transformation
**Intent:** Apply tool-specific transformations to a common input structure

**Implementation:**

`convert.sh` defines per-tool converter functions. Each visitor knows how to transform the canonical format for its target:

```bash
convert_claude_code() {
  local agent="$1"
  local output_dir="$2"
  # Direct copy with frontmatter intact
  get_body "$agent" > "$output_dir/$(slugify "$agent").md"
}

convert_cursor() {
  local agent="$1"
  local output_dir="$2"
  # Strip frontmatter, output .mdc rules
  get_body "$agent" | sed '/^---/,/^---/d' > "$output_dir/$(slugify "$agent").mdc"
}

convert_openclaw() {
  local agent="$1"
  local output_dir="$2"
  # Split by section keywords into SOUL.md, AGENTS.md, IDENTITY.md
  split_by_keywords "$agent" "$output_dir"
}
```

**Evidence:** `convert.sh` lines 100-604 contain `convert_*` functions for each supported tool.

**Benefit:** Adding a new tool requires adding a new converter function, not modifying existing ones.

---

## Pattern 3: Accumulator

**Category:** Transformation
**Intent:** Collect multiple inputs into a single output

**Implementation:**

Aider and Windsurf require a single file containing all agent conventions. The accumulator collects all agents into a temp file, then writes once:

```bash
# Accumulation pattern in convert.sh
for tool in aider windsurf; do
  temp_file=$(mktemp)
  for agent in "${AGENTS[@]}"; do
    cat "$agent" >> "$temp_file"
    echo -e "\n---\n" >> "$temp_file"
  done
  mv "$temp_file" "integrations/$tool/CONVENTIONS.md"
done
```

**Evidence:** `convert.sh` uses temp files to accumulate content before final write.

**Benefit:** Single I/O operation instead of N writes. Enables cross-referencing across agents.

---

## Pattern 4: Factory

**Category:** Creational
**Intent:** Create consistent identifiers from human-readable names

**Implementation:**

```bash
slugify() {
  local name="$1"
  echo "$name" | sed 's/ /-/g' | tr '[:upper:]' '[:lower:]'
}
```

**Evidence:** `convert.sh` defines `slugify()` and uses it to generate filenames:
- "Frontend Developer" -> "frontend-developer.md"
- "Security Engineer" -> "security-engineer.md"

**Benefit:** Deterministic, consistent naming across all generated artifacts.

---

## Pattern 5: Template Method

**Category:** Behavioral
**Intent:** Define skeleton of an algorithm, let subclasses redefine certain steps

**Implementation:**

All agent markdown files follow the same section structure:

```
## Identity & Memory
## Communication Style
## Critical Rules
## Core Mission
## Technical Deliverables
## Workflow Process
## Success Metrics
## Advanced Capabilities
```

Each agent fills these sections with domain-specific content, but the structure remains constant.

**Evidence:** `CONTRIBUTING.md` section "Agent File Structure" defines required sections.

**Benefit:** Authors know exactly where to put information. Tools can parse generically.

---

## Pattern 6: Strategy

**Category:** Behavioral
**Intent:** Define family of algorithms, encapsulate each, make interchangeable

**Implementation:**

Color normalization varies by tool capability:

```bash
# Strategy per tool
resolve_color_claude_code() {
  echo "$1"  # Accept any format
}

resolve_color_aider() {
  # Map named colors to hex
  case "$1" in
    cyan) echo "#06B6D4" ;;
    blue) echo "#3B82F6" ;;
    *) echo "#6B7280" ;;
  esac
}

resolve_color_qwen() {
  # Qwen uses different palette
  case "$1" in
    cyan) echo "CYN" ;;
    blue) echo "BLU" ;;
    *) echo "GRY" ;;
  esac
}
```

**Evidence:** `convert.sh` contains tool-specific color handling with different output formats per tool.

**Benefit:** Each tool gets colors in its native format without conditional logic in the conversion core.

---

## Pattern 7: Facade

**Category:** Structural
**Intent:** Provide simple interface to a complex subsystem

**Implementation:**

`install.sh` presents a simple interactive interface that hides complexity of:
- Detecting installed AI coding tools
- Finding platform-specific config directories
- Handling permissions and file placement

```bash
# Facade: simple user-facing command
install_agents() {
  echo "Detecting installed tools..."
  detect_tools  # Complex subsystem
  echo "Select tools to install agents for:"
  select_tools  # User interaction
  copy_to_config  # Filesystem operations
}
```

**Evidence:** `install.sh` is ~500 lines of complex path detection and file copying, abstracted behind simple prompts.

**Benefit:** Users interact with simple prompts, not filesystem paths and tool-specific configs.

---

## Pattern 8: Domain-Driven Organization

**Category:** Architectural
**Intent:** Organize by business domain, not technical function

**Implementation:**

```
agency-agents/
├── engineering/      # Software development roles
├── marketing/        # Growth, content, social media
├── design/           # UI/UX, visual design
├── specialized/      # Niche roles
├── game-development/  # Subdivided by engine
├── academic/         # Research, education
├── product/         # Product management
├── sales/           # Sales roles
├── strategy/        # Strategic planning
├── support/         # Customer support
├── testing/         # QA roles
├── paid-media/      # Advertising
├── project-management/
└── spatial-computing/
```

**Evidence:** 13 top-level directories in repository, each containing domain-specific agent markdown files.

**Rationale:** Domain organization enables intuitive discovery by non-technical stakeholders. Technical function would require understanding software architecture.

---

## Pattern 9: Single Source of Truth

**Category:** Architectural
**Intent:** Maintain one canonical definition, generate multiple representations

**Implementation:**

```
domain/engineering-frontend-developer.md (canonical)
     |
     +---> convert.sh ---> integrations/claude-code/frontend-developer.md
     +---> convert.sh ---> integrations/cursor/frontend-developer.mdc
     +---> convert.sh ---> integrations/aider/CONVENTIONS.md (accumulated)
     +---> convert.sh ---> integrations/openclaw/SOUL.md (split)
     +---> convert.sh ---> ... (other tools)
```

**Evidence:** `convert.sh` is the single transformation point. All 10 tool formats derive from the same source.

**Benefit:** One edit updates all formats. No synchronization drift between representations.

---

## Pattern 10: Canonical Naming

**Category:** Naming
**Intent:** Convert human names to consistent identifiers

**Implementation:**

```bash
slugify() {
  local name="$1"
  # Lowercase, spaces to hyphens
  echo "$name" | sed 's/ /-/g' | tr '[:upper:]' '[:lower:]'
}
```

**Evidence:** Used throughout `convert.sh` to generate:
- Filenames (frontend-developer.md)
- Section anchors
- Tool-specific identifiers

**Examples:**
| Human Name | Slug |
|------------|------|
| Frontend Developer | frontend-developer |
| AI Engineer | ai-engineer |
| DevOps Automator | devops-automator |

**Benefit:** Predictable, URL-safe identifiers without manual naming decisions.

---

## Pattern Relationships

```
Single Source of Truth
         |
         v
Schema + Content Separation
         |
         +---> Visitor/Transformer (convert.sh)
         |             |
         |             +---> Factory (slugify)
         |             +---> Strategy (color resolution)
         |             +---> Accumulator (aider/windsurf)
         |
         +---> Template Method (agent structure)
         |
         v
Domain-Driven Organization
         |
         v
Facade (install.sh)
```

---

## Anti-Patterns Present

| Anti-Pattern | Manifestation | Impact |
|--------------|---------------|--------|
| God Script | `convert.sh` handles 10 tools | Single responsibility violation |
| Hardcoded Arrays | `AGENT_DIRS` in multiple scripts | Data duplication |
| Gitignored Output | `integrations/` not versioned | No generated file history |
| No Test Coverage | Scripts not tested | Refactoring risk |

---

## Pattern Summary by File

| File | Patterns |
|------|----------|
| `domain/*.md` | Schema+Content Separation, Template Method, Domain-Driven Organization |
| `convert.sh` | Visitor/Transformer, Factory, Strategy, Accumulator |
| `install.sh` | Facade |
| `lint-agents.sh` | (Validation only, minimal patterns) |
| `examples/*.md` | (Documentation, not code) |
