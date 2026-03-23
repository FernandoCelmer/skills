# Skills Marketplace

A marketplace of Claude Code skills. Install the marketplace once, then install any skill directly from Claude Code.

## Getting Started

```bash
claude plugins marketplace add FernandoCelmer/skills
```

---

## Available Skills

| Name | Version | Category | Description |
|---|---|---|---|
| [repo-audit](repo-audit/commands/repo-audit.md) | 1.0.0 | development | Deep technical audit of a repository: find bugs, gaps, missing tests and security issues, then create GitHub issues for findings |
| [review-issues](review-issues/commands/review-issues.md) | 1.0.0 | development | Analyze open issues across one or more repositories |
| [smart-review-pr](smart-review-pr/commands/smart-review-pr.md) | 2.0.0 | development | Comprehensive PR review covering code quality, security, architecture and design patterns |

---

## How to contribute

1. Fork this repository
2. Create your skill at `<name>/commands/<name>.md`
3. Add the entry to [registry.json](registry.json)
4. Open a Pull Request

### Skill file format

```markdown
---
name: skill-name
description: What this skill does (used by Claude to decide when to invoke it)
version: 1.0.0
---

Full prompt/instructions for the skill...
```
