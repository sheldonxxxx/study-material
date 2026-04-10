# Code Quality Practices: agency-agents

## Quality Philosophy

The agency-agents project embeds quality practices directly into agent personalities rather than enforcing them through external tooling. Quality is a first-class concern in the Engineering and Testing divisions, with agents designed to catch issues, enforce standards, and drive continuous improvement.

## Agent-Based Quality Assurance

### Testing Division Agents

The Testing division (8 agents) implements a layered QA approach:

#### Evidence Collector Agent
The Evidence Collector operates on a "screenshots don't lie" philosophy:
- Requires visual proof for every claim
- Defaults to finding 3-5+ issues on first implementation
- Uses Playwright-based screenshot capture for cross-device testing
- Flags "fantasy reporting" (claims without evidence)

**Key Quality Behaviors:**
```bash
# Automated screenshot capture via Playwright
./qa-playwright-capture.sh http://localhost:8000 public/qa-screenshots

# Cross-device evidence: desktop, tablet, mobile
# Interactive element testing: accordions, forms, navigation
# Dark mode validation across all device sizes
```

**Evidence Requirements:**
- responsive-desktop.png (1920x1080)
- responsive-tablet.png (768x1024)
- responsive-mobile.png (375x667)
- Dark mode variants for each viewport
- Before/after interaction screenshots

#### Reality Checker Agent
The Reality Checker is the final integration gate:
- Stops "fantasy approvals" — unrealistic quality assessments
- Default status: "NEEDS WORK" unless overwhelming proof exists
- Cross-validates QA findings against actual implementation
- Requires 2-3 revision cycles for typical first implementations

**Realistic Quality Ratings:**
- C+ / B- / B / B+ (no A+ fantasies on first attempts)
- Production Readiness: FAILED / NEEDS WORK / READY
- Explicit revision cycle expectations

#### Test Results Analyzer
Analyzes test output and provides quality insights:
- Coverage reporting
- Metric analysis against success criteria
- Regression detection across test cycles

#### Performance Benchmarker
Performance testing and optimization agent:
- Speed testing across device categories
- Load testing under various conditions
- Core Web Vitals validation
- Performance tuning recommendations

### Code Review Agent

The Code Reviewer agent implements a structured review methodology:

**Review Priorities:**
1. **Correctness** — Does it do what it's supposed to?
2. **Security** — Vulnerabilities, input validation, auth checks?
3. **Maintainability** — Will someone understand this in 6 months?
4. **Performance** — Bottlenecks, N+1 queries?
5. **Testing** — Are important paths covered?

**Issue Classification:**
| Priority | Symbol | Examples |
|----------|--------|----------|
| Blocker | 🔴 | Security vulnerabilities, data loss risks, race conditions, breaking API contracts |
| Suggestion | 🟡 | Missing input validation, unclear naming, missing tests, N+1 queries |
| Nit | 💭 | Style inconsistencies, minor naming, documentation gaps |

**Review Comment Format:**
```
🔴 **Security: SQL Injection Risk**
Line 42: User input is interpolated directly into the query.

**Why:** An attacker could inject `'; DROP TABLE users; --` as the name parameter.

**Suggestion:**
- Use parameterized queries: `db.query('SELECT * FROM users WHERE name = $1', [name])`
```

### Frontend Developer Quality Standards

The Frontend Developer agent embeds quality practices:

**Performance Standards:**
- Core Web Vitals optimization: LCP < 2.5s, FID < 100ms, CLS < 0.1
- Bundle optimization with code splitting and tree shaking
- Image optimization with WebP/AVIF and responsive loading
- Lighthouse scores exceeding 90 for Performance and Accessibility

**Accessibility Standards:**
- WCAG 2.1 AA compliance
- Semantic HTML and proper ARIA labels
- Keyboard navigation throughout
- Screen reader compatibility testing

**Code Quality:**
- TypeScript with proper typing
- Memoization and useCallback for performance
- Virtualized lists for large data sets
- Comprehensive unit and integration tests

## Technical Quality Infrastructure

### Linting Script

The project includes `lint-agents.sh` for validating agent markdown files:

**Required Frontmatter Fields (ERROR if missing):**
- `name`
- `description`
- `color`

**Recommended Sections (WARN if missing):**
- Identity & Memory
- Core Mission
- Critical Rules

**Content Validation:**
- Minimum 50 words in body
- Valid YAML frontmatter delimiters
- File must have meaningful content

**Usage:**
```bash
# Lint all agents
./scripts/lint-agents.sh

# Lint specific files
./scripts/lint-agents.sh engineering/frontend-developer.md

# Results: error count, warning count, PASSED/FAILED status
```

### Integration Conversion Scripts

The `convert.sh` and `install.sh` scripts handle multi-tool integration:
- Parallel conversion support for faster processing
- Tool-specific format generation
- Installation validation
- Config file management per tool

## Quality Patterns in Agent Code Examples

### Input Validation Pattern (from Security Engineer)

```python
from pydantic import BaseModel, Field, field_validator
import re

class UserInput(BaseModel):
    """Input validation with strict constraints."""
    username: str = Field(..., min_length=3, max_length=30)
    email: str = Field(..., max_length=254)

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_-]+$", v):
            raise ValueError("Username contains invalid characters")
        return v
```

### React Component Quality Pattern (from Frontend Developer)

```tsx
interface DataTableProps {
  data: Array<Record<string, any>>;
  columns: Column[];
  onRowClick?: (row: any) => void;
}

export const DataTable = memo<DataTableProps>(({ data, columns, onRowClick }) => {
  // useMemo for expensive computations
  const rowVirtualizer = useVirtualizer({
    count: data.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  // useCallback to prevent unnecessary re-renders
  const handleRowClick = useCallback((row: any) => {
    onRowClick?.(row);
  }, [onRowClick]);

  return (
    <div role="table" aria-label="Data table">
      {/* Virtualized rendering for performance */}
    </div>
  );
});
```

### Security Headers Pattern (from Security Engineer)

```nginx
server {
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; frame-ancestors 'none';" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
    server_tokens off;
}
```

## Quality Metrics

Agents define measurable success metrics:

**Frontend Developer:**
- Page load < 3 seconds on 3G networks
- Lighthouse scores > 90 for Performance and Accessibility
- Component reusability rate > 80%
- Zero console errors in production

**Code Reviewer:**
- One complete review, no drip-feeding of comments
- Every blocker paired with remediation guidance
- Issue classification consistency

**Evidence Collector:**
- Minimum 3-5 issues found on first implementation
- Visual evidence for every claim
- Specification compliance verification

**Reality Checker:**
- Systems approved actually work in production
- Quality assessments match user experience reality
- Zero broken functionality reaches end users

## CI/CD Quality Integration

The Security Engineer agent defines a security scanning pipeline:

```yaml
jobs:
  sast:
    name: Static Analysis
    steps:
      - uses: semgrep/semgrep-action@v1
        with:
          config: p/owasp-top-ten, p/cwe-top-25

  dependency-scan:
    name: Dependency Audit
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  secrets-scan:
    name: Secrets Detection
    steps:
      - uses: gitleaks/gitleaks-action@v2
```

## Quality Anti-Patterns

The Testing division explicitly identifies anti-patterns:

**Fantasy Reporting:**
- "Zero issues found" without evidence
- Perfect scores (A+, 98/100) on first implementation
- "Production ready" without demonstrated excellence
- "Luxury/premium" claims for basic implementations

**Evidence Failures:**
- Claims not supported by screenshots
- Specification requirements not implemented
- Cross-device inconsistencies
- Performance problems (>3 second load times)

## Contributing Quality Standards

The CONTRIBUTING.md defines agent design guidelines:

1. **Agent File Structure** — Required frontmatter, recommended sections
2. **Pull Request Process** — lint-agents.sh must pass
3. **Content Requirements** — Minimum 50 words, meaningful deliverables
4. **Test Before Submitting** — Agent should work in real scenarios

## Summary

Quality in agency-agents is implemented through:
- Embedded quality consciousness in agent personalities
- Dedicated Testing division with evidence-based methodology
- Linting infrastructure for agent markdown validation
- Security scanning patterns in CI/CD
- Measurable success metrics per agent
- Explicit anti-pattern identification
