# Community

## Project Governance

### Ownership
**Primary Maintainer**: Jesse Vincent (@obra)
- GitHub: https://github.com/obra
- Email: jesse@fsck.com
- Blog: https://blog.fsck.com

**Organization**: Prime Radiant
- Website: https://primeradiant.com
- Sponsor page: https://github.com/sponsors/obra

### Governance Model
**Single Maintainer + Open Contributions**:
- One primary maintainer makes final decisions
- Community contributions via GitHub PRs
- Maintainer reviews all PRs against quality standards
- No formal governance committee or election process

## Community Size Indicators

### GitHub Statistics (as of analysis)
- **Repository**: obra/superpowers
- **Stars**: Not publicly visible in artifacts
- **Forks**: Not publicly visible in artifacts
- **Open Issues**: ~5-10 based on recent changelog references
- **Closed Issues**: References to issues #774, #780, #783, #770, #723 in recent changelog

### Release Activity
- Version 5.0.5: 2026-03-17
- Version 5.0.6: Current (based on package.json)
- Active development with regular releases

## Contribution Process

### Workflow
1. Fork the repository
2. Create a branch for your skill
3. Follow `skills/writing-skills/SKILL.md` for creating/testing skills
4. Submit a PR with completed template
5. Maintainer reviews against rigor checklist

### PR Requirements (from PR template)
- Problem statement with failure mode
- 1-3 sentence change description
- Appropriateness for core library check
- Alternatives considered
- Related PRs review
- Environment tested (harness, version, model)
- Evaluation methodology
- Rigor checklist:
  - [ ] Adversarial pressure testing completed
  - [ ] Not just happy path
  - [ ] Content shaping reviewed

### Contribution Barriers
**High barrier for skills changes**:
- Must use TDD methodology (RED-GREEN-REFACTOR)
- Must complete adversarial pressure testing
- Must not modify Red Flags/rationalizations without eval evidence
- Must have human review of complete diff

**Appropriateness check**:
- Would this be useful to someone on a different project?
- Is this project-specific, team-specific, or tool-specific?
- Does this integrate third-party services?
- If domain-specific, it belongs in a separate plugin

## Community Support Channels

### Discord
- **Invite**: https://discord.gg/Jd8Vphy9jq
- **Purpose**: Community support, questions, sharing what you're building
- **Owner**: Listed as community channel in README

### GitHub Issues
- **URL**: https://github.com/obra/superpowers/issues
- **Purpose**: Bug reports, feature requests
- **Templates**: Bug report, feature request, platform support

### GitHub Discussions
Not explicitly mentioned in artifacts

## Code of Conduct

### Standard
Contributor Covenant Code of Conduct 2.0

### Enforcement
- **Contact**: jesse@primeradiant.com
- **Response**: All complaints reviewed promptly and fairly
- **Privacy**: Privacy and security of reporters protected

### Enforcement Guidelines (4 levels)
| Level | Trigger | Consequence |
|-------|---------|-------------|
| 1. Correction | Inappropriate language, unprofessional behavior | Private written warning |
| 2. Warning | Single incident or series of actions | Warning + no interaction period |
| 3. Temporary Ban | Serious violation, sustained inappropriate behavior | Temporary ban from community |
| 4. Permanent Ban | Pattern of violation, harassment, aggression | Permanent ban |

## Community Health Indicators

### Positive Indicators
1. **Active maintenance**: Recent changelog entries (2026-03-17)
2. **Comprehensive testing**: Multiple test suites across platforms
3. **Documentation quality**: Standardized skill format, cross-platform docs
4. **Thoughtful PR template**: Enforces rigor and evaluation
5. **Code of conduct**: Contributor Covenant 2.0 adopted
6. **Clear scope**: "Core library" appropriateness check for PRs

### Areas of Concern
1. **Single maintainer**: No backup maintainers or governance transition plan
2. **No public community metrics**: Stars, forks, contributor count not visible
3. **No governance documentation**: No election process, decision-making documented
4. **Potentially overwhelmed maintainer**: One person handling issues, PRs, support

## Diversity & Inclusion

### Observables
- Contributor Covenant 2.0 adopted
- Platform-agnostic design (Windows, macOS, Linux; multiple AI harnesses)
- No discriminatory language or requirements
- Enforcement contact for private reporting

## Licensing

**MIT License**
- Permissive open source license
- Allows commercial use, modification, distribution
- Requires license notice and copyright notice
- No warranty

## Sponsorship

**GitHub Sponsors**
- URL: https://github.com/sponsors/obra
- Mentioned in README: "If Superpowers has helped you do stuff that makes money..."
- Primary income stream for maintainer

## Integration with Broader Ecosystem

### Agent Platforms
Superpowers integrates with multiple AI coding agent platforms:
- Claude Code (Anthropic)
- Cursor (Cursor AI)
- Codex (OpenAI)
- OpenCode (OpenCode.ai)
- Gemini CLI (Google)

**Strategy**: Meet users where they are rather than requiring platform switch.

### Related Projects
- **superpowers-marketplace**: Separate repository for custom Claude Code marketplace
- **Agent skill ecosystem**: Aligns with agentskills.io specification

## Community Documentation

### Getting Help
1. Discord for questions and community support
2. GitHub Issues for bugs and features
3. README.md for installation help
4. Platform-specific docs for platform issues

### How to Contribute
1. Read `skills/writing-skills/SKILL.md`
2. Follow TDD methodology
3. Complete adversarial testing
4. Submit via PR with completed template
