# Code Quality: Get Shit Done

## Testing Approach

### Node.js Built-in Test Runner
GSD uses exclusively Node.js built-in testing utilities:
- `node:test` - Test runner
- `node:assert/strict` - Assertion library
- **No Jest, Mocha, Chai, or external test frameworks**

### Required Test Imports
```javascript
const { describe, it, test, beforeEach, afterEach, before, after } = require('node:test');
const assert = require('node:assert/strict');
```

### Test Structure Patterns

**Use Hooks for Setup/Cleanup (Not try/finally)**
```javascript
// GOOD - hooks handle setup/cleanup
describe('my feature', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('does the thing', () => {
    assert.strictEqual(result, expected);
  });
});

// BAD - try/finally masks failures
test('does the thing', () => {
  const tmpDir = createTempProject();
  try {
    assert.strictEqual(result, expected);
  } finally {
    cleanup(tmpDir);
  }
});
```

### Centralized Test Helpers

Import helpers from `tests/helpers.cjs`:
```javascript
const { createTempProject, createTempGitProject, createTempDir, cleanup, runGsdTools } = require('./helpers.cjs');
```

| Helper | Creates | Use When |
|--------|---------|----------|
| `createTempProject(prefix?)` | tmpDir with `.planning/phases/` | Testing GSD tools needing planning structure |
| `createTempGitProject(prefix?)` | Same + git init + initial commit | Testing git-dependent features |
| `createTempDir(prefix?)` | Bare temp directory | Testing features without `.planning/` |
| `cleanup(tmpDir)` | Removes directory recursively | Always use in `afterEach` |
| `runGsdTools(args, cwd, env?)` | Executes gsd-tools.cjs | Testing CLI commands |

### Node.js Version Compatibility

Tests must pass on:
- **Node 22** (LTS)
- **Node 24** (Current)

Forward-compatible with Node 26. Do not use deprecated APIs.

### Assertions

Use `node:assert/strict` for strict equality:
```javascript
assert.strictEqual(actual, expected);      // ===
assert.deepStrictEqual(actual, expected);  // deep ===
assert.ok(value);                          // truthy
assert.throws(() => { ... }, /pattern/);   // throws
assert.rejects(async () => { ... });       // async throws
```

### Test Coverage

GSD's own test suite (`npm test`) covers:
- Core utilities
- State management
- Phase operations
- Roadmap parsing
- Config handling
- Template operations
- Frontmatter operations
- Model resolution
- Security functions
- Prompt injection scanning

## Code Standards

### CommonJS Only
- **All code uses `require()`, not ESM `import`**
- File extension: `.cjs`
- No build step required

### No External Dependencies in Core
The `gsd-tools.cjs` CLI and all lib modules use **only Node.js built-ins**:
- `fs`, `path`, `child_process`, `crypto`, `readline`, etc.
- No npm packages in core functionality

### Conventional Commits
Format: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `ci:`

Examples:
```
feat(08-02): add email confirmation flow
fix(auth): handle expired session tokens
docs(readme): update installation instructions
refactor(core): extract path validation utility
test(verify): add coverage for artifact validation
ci(test): add Node 24 to test matrix
```

### CI Testing Matrix
Tests run on multiple platforms:
- Ubuntu, macOS, Windows
- Node 22, Node 24

## Type Safety

### Dynamic Typing with Runtime Validation
GSD uses JavaScript's dynamic typing with defensive runtime checks:
- JSON parsing with error handling
- Path validation before file operations
- Config schema validation on load

### No TypeScript
The project is pure JavaScript (CommonJS). No type annotations or type checking build step.

### Frontmatter Validation
YAML frontmatter schemas validate required fields:
- Plans require: `phase`, `plan`, `type`, `wave`, `depends_on`, `files_modified`, `autonomous`, `requirements`, `must_haves`
- Summaries require: `phase`, `plan`, `type`, `summary`, `tasks`, `artifacts`

## Tooling

### gsd-tools.cjs CLI
17 domain modules for all GSD operations:
- `core.cjs` - Error handling, output formatting, shared utilities
- `state.cjs` - STATE.md parsing and updating
- `phase.cjs` - Phase directory operations
- `roadmap.cjs` - ROADMAP.md parsing
- `config.cjs` - config.json read/write
- `verify.cjs` - Plan structure and phase completeness validation
- `template.cjs` - Template selection and filling
- `frontmatter.cjs` - YAML frontmatter CRUD
- `init.cjs` - Compound context loading
- `milestone.cjs` - Milestone archival
- `commands.cjs` - Utilities (slug, timestamp, todos, stats)
- `model-profiles.cjs` - Model resolution
- `security.cjs` - Path traversal, prompt injection, safe parsing
- `uat.cjs` - UAT file parsing and verification audit

### Verification Agents

| Agent | Purpose |
|-------|---------|
| `gsd-plan-checker` | 8-dimension verification before execution |
| `gsd-integration-checker` | Cross-phase integration verification |
| `gsd-ui-checker` | UI-SPEC.md design contract validation |
| `gsd-nyquist-auditor` | Test coverage gap filling |
| `gsd-verifier` | Post-execution goal verification |

### Quality Gates

1. **Plan-phase verification** - Plans checked against 8 dimensions before execution
2. **Pre-commit hooks** - Style and format enforcement
3. **Wave-based execution** - Parallel plans with dependency ordering
4. **Post-execution verification** - Goal-backward analysis against phase goals
5. **UAT (User Acceptance Testing)** - Human verification as final gate
6. **Cross-phase regression** - Prior phases' test suites run after execution

## File Structure

```
get-shit-done/
  bin/
    install.js          # Multi-runtime installer
    gsd-tools.cjs       # CLI utility
    lib/                # 15 domain modules (.cjs)
  workflows/            # 46 workflow definitions (.md)
  agents/               # 18 agent definitions (.md)
  commands/gsd/         # 44 slash commands (.md)
  references/           # 13 shared reference docs (.md)
  templates/            # Planning artifact templates
  hooks/                # Runtime integration hooks (.js)
  tests/                # Test suite (.test.cjs)
    helpers.cjs         # Shared test utilities
```

## Continuous Integration

### GitHub Actions Workflows
- **Test workflow** - Node 22/24 on Ubuntu, macOS, Windows
- **Security scanning** - Prompt injection, base64, and secret scanning
- **Multi-runtime validation**

### Security Scanning (v1.29+)
- CI-ready injection scanner (`prompt-injection-scan.test.cjs`)
- Scans all agent/workflow/command files for embedded injection vectors

## Quality Metrics

### Commit Safety
- **Atomic commits** - Each task gets its own commit
- **Parallel commit safety** - `--no-verify` flag for parallel executors
- **STATE.md file locking** - Lockfile-based mutual exclusion prevents race conditions

### Verification Debt Tracking
- `audit-uat` command scans all phases for unresolved items
- `status: partial` distinguishes incomplete testing
- `result: blocked` with `blocked_by` for external dependencies
- `human_needed` items persist as HUMAN-UAT.md files

## Best Practices Established

1. **Hooks over try/finally** - Cleaner tests, better failure visibility
2. **Centralized helpers** - DRY test infrastructure
3. **Fresh context per agent** - No context degradation
4. **Verifiable artifacts** - Every task has explicit verification criteria
5. **Human verification as gate** - Automated checks + human confirmation
