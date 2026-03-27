# Community

## Project Links

| Resource | URL |
|----------|-----|
| Website | [nanoclaw.dev](https://nanoclaw.dev) |
| Documentation | [docs.nanoclaw.dev](https://docs.nanoclaw.dev) |
| Discord | [discord.gg/VDdww8qS42](https://discord.gg/VDdww8qS42) |
| GitHub | [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) |

## Communication Channels

### Discord (Primary)

- **Invite**: [discord.gg/VDdww8qS42](https://discord.gg/VDdww8qS42)
- **Badge**: Shown on README.md
- **Purpose**: Questions, ideas, support, announcements

### GitHub

- **Issues**: Bug reports, feature requests
- **PRs**: Code contributions via forks
- **Discussions**: Architecture decisions, RFCs

## Maintainers

| Name | Role |
|------|------|
| `@gavrielc` | Core maintainer |
| `@gabi-simons` | Core maintainer |

### CODEOWNERS

Core code ownership:
```
/src/                 @gavrielc @gabi-simons
/container/          @gavrielc @gabi-simons
/groups/              @gavrielc @gabi-simons
/launchd/            @gavrielc @gabi-simons
/package.json        @gavrielc @gabi-simons
/package-lock.json   @gavrielc @gabi-simons
```

Skills are community-open:
```
/.claude/skills/      (no owner - open contribution)
```

## Contributing Model

### Skills-First Approach

The project explicitly states: **"Don't add features. Add skills."**

Rather than accepting PRs that modify core code for new capabilities (Telegram support, etc.), contributors:
1. Fork the repository
2. Create a branch with their feature code
3. Add a SKILL.md with setup instructions
4. Open a PR

The maintainers then create a `skill/<name>` branch from the PR that users can merge.

### Accepted Source Changes

Only these types of source code changes are accepted to main:
- Bug fixes
- Security fixes
- Simplifications
- Reductions in code

### Not Accepted

- New features
- New capabilities
- Compatibility additions
- Enhancements

These should be skills.

### Skills Taxonomy

| Type | Contribution Method | Example |
|------|---------------------|---------|
| Feature skill | Branch merge + SKILL.md | `/add-telegram` |
| Utility skill | SKILL.md + code files | `/claw` |
| Operational skill | SKILL.md only | `/setup`, `/debug` |
| Container skill | `container/skills/` | `agent-browser` |

## PR Labels

Auto-applied via `label-pr.yml` workflow:

| Label | Meaning |
|-------|---------|
| `PR: Skill` | Skill contribution |
| `PR: Feature` | Feature skill (adds channel/integration) |
| `PR: Fix` | Bug/security fix |
| `PR: Refactor` | Simplification |
| `PR: Docs` | Documentation only |
| `follows-guidelines` | PR follows CONTRIBUTING.md template |

## Request for Skills (RFS)

The README explicitly lists desired skills that the community could build:

### Communication Channels
- `/add-signal` - Add Signal as a channel

## Internationalization

The project welcomes translations:
- English (primary)
- Chinese Simplified (`README_zh.md`)
- Japanese (`README_ja.md`)

## Contributor Recognition

`CONTRIBUTORS.md` acknowledges contributors.

## Discord Community Features

- Support and troubleshooting
- Feature discussions
- Announcements
- User showcase

## PR Template

Located at `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Type of Change
- [ ] Feature skill
- [ ] Utility skill
- [ ] Operational/container skill
- [ ] Fix
- [ ] Simplification
- [ ] Documentation

## Description
(write description)

## For Skills
- [ ] SKILL.md contains instructions, not inline code
- [ ] SKILL.md is under 500 lines
- [ ] I tested this skill on a fresh clone
```

## Philosophy Alignment

The community philosophy emphasizes:

1. **Small enough to understand** - Single process, minimal files
2. **Secure by isolation** - Container-based, no shared memory
3. **Built for the individual** - Bespoke forks, not monolithic
4. **Customization = code changes** - No config sprawl
5. **AI-native** - No wizards, Claude Code handles everything
6. **Skills over features** - Extensibility without bloat

## Before Creating PR Checklist

Per CONTRIBUTING.md:

1. Search for existing PRs/issues before starting work
2. Read the Philosophy section
3. Ensure one thing per PR
4. Test on fresh clone before submitting
5. Link related issues with `Closes #123`
6. Select appropriate change type in PR template

## Testing Requirement

Contributors must test their contribution on a fresh clone before submitting. For skills, this means running the skill end-to-end and verifying it works.
