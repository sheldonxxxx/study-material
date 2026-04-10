# Community

## Project Ownership

**Maintainer:** Garry Tan — President & CEO of Y Combinator

**Background:**
- Built Bookface (YC's internal social network) in 2013 — 772 GitHub contributions that year
- In 2026, achieved 1,237 contributions using gstack — same person, dramatically different output
- Published productivity claims: 600K+ lines of production code in 60 days, 10K-20K lines/day part-time while running YC

**Motivation (from README):**
> "I don't think I've typed like a line of code probably since December, basically, which is an extremely large change." — Andrej Karpathy, March 2026

gstack is described as "how I do it" — the tools that enable one person to ship like a team.

## License

**MIT License** — "Free forever. Go build something."

No commercial license, no premium tier, no enterprise features locked behind payment.

## Contribution Model

### Community PR Triage (Wave Process)

When community PRs accumulate, they are batched into themed waves:

1. **Categorize** — group by theme (security, features, infra, docs)
2. **Deduplicate** — if two PRs fix the same thing, pick the one with fewer lines changed
3. **Collector branch** — create `pr-wave-N`, merge clean PRs, resolve conflicts for dirty ones
4. **Close with context** — every closed PR gets a comment explaining why and what supersedes it
5. **Ship as one PR** — single PR to main with all attributions preserved in merge commits

Example wave: PR #205 (v0.8.3).

### Contributor Mode

gstack has a self-improving feedback loop:

1. Enable contributor mode: `gstack-config set gstack_contributor true`
2. Claude Code periodically reflects on gstack usage — rating 0-10 at end of major workflow steps
3. When something rates below 10, Claude files a report to `~/.gstack/contributor-logs/`
4. Logs include: what happened, repro steps, what would make it better

**The contributor workflow:**
1. Use gstack normally — contributor mode reflects and logs issues automatically
2. Check logs: `ls ~/.gstack/contributor-logs/`
3. Fork and clone gstack
4. Symlink fork into the project where you hit the bug
5. Fix the issue — changes are live immediately in this project
6. Open a PR from your fork

### Dev Mode for Contributors

For working on gstack itself:

```bash
git clone <repo> && cd gstack
bun install
bin/dev-setup          # symlinks .claude/skills/ to your working tree
# edit any SKILL.md, changes are live immediately
bin/dev-teardown       # restore global install
```

## Skill Development

### SKILL.md Editing

SKILL.md files are **generated from templates** — don't edit `.md` directly:

1. Edit the `.tmpl` file (e.g., `review/SKILL.md.tmpl`)
2. Run `bun run gen:skill-docs` (regenerates Claude output) and `bun run gen:skill-docs --host codex` (Codex output)
3. Commit both the `.tmpl` and generated `.md` files

A GitHub Action fails if generated files are stale.

### Adding a New Browse Command

1. Add command to `browse/src/commands.ts`
2. Add snapshot flags to `SNAPSHOT_FLAGS` in `browse/src/snapshot.ts`
3. Run `bun run build`
4. Run `bun run gen:skill-docs` to update documentation

### Dual-Host Development

gstack generates SKILL.md for two hosts:

| Aspect | Claude | Codex |
|--------|--------|-------|
| Output directory | `{skill}/SKILL.md` | `.agents/skills/gstack-{skill}/SKILL.md` |
| Frontmatter | Full | Minimal (name + description only) |
| `/codex` skill | Included | Excluded |

## Testing as Contribution

Test tiers provide multiple ways to validate changes:

| Tier | Cost | What it tests |
|------|------|---------------|
| 1 — Static | Free | Command validation, snapshot flags, SKILL.md correctness |
| 2 — E2E | ~$3.85/run | Full skill execution via `claude -p` |
| 3 — LLM eval | ~$0.15/run | LLM-as-judge scoring of generated docs |

Contributor workflow: fix the issue → run `bun test` → run `bun run test:evals` → open PR.

## Activity Metrics

From README (2026):
- 1,237 contributions in 2026
- 600,000+ lines of production code (35% tests) in 60 days
- 10,000-20,000 lines per day
- One week `/retro` across 3 projects: 140,751 lines added, 362 commits, ~115k net LOC

## Community Links

- **GitHub:** github.com/garrytan/gstack
- **Conductor:** Multiple Claude Code sessions in parallel — gstack's sprint structure enables parallelism
- **Greptile integration:** Skills documentation includes Greptile integration for codebase-aware search

## YC Hiring

README includes hiring notice:
> "We're hiring. Want to ship 10K+ LOC/day and help harden gstack? Come work at YC — ycombinator.com/software"

## Governance

gstack is a **personal open source project** with MIT license:
- No formal governance structure
- Single maintainer (Garry Tan)
- Wave PR process for community contributions
- Contributor mode for self-directed improvement
- No corporate backing (though YC affiliation provides resources)

## Transparency

Notable transparency practices:
- **Contributor mode logs** are for the contributor, not hidden
- **CHANGELOG is comprehensive** (113KB) — every version documented
- **Telemetry is opt-in with clear disclosure** — exactly what's sent, what's not
- **Generated docs are committed** — not hidden behind build steps
- **Architecture decisions explained in ARCHITECTURE.md** — the "why" behind choices
