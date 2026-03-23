# Skills

A collection of Claude Code skills ready to install.

## Available Skills

| Name | Version | Category | Description |
|---|---|---|---|
| [repo-audit](repo-audit/commands/repo-audit.md) | 1.0.0 | development | Deep technical audit of a repository: find bugs, gaps, missing tests and security issues, then create GitHub issues for findings |
| [review-issues](review-issues/commands/review-issues.md) | 1.0.0 | development | Analyze open issues across one or more repositories |
| [smart-review-pr](smart-review-pr/commands/smart-review-pr.md) | 2.0.0 | development | Comprehensive PR review covering code quality, security, architecture and design patterns |

## How to install

Open Claude Code and run:

```
install skill repo-audit from https://github.com/FernandoCelmer/skills
```

```
install skill review-issues from https://github.com/FernandoCelmer/skills
```

```
install skill smart-review-pr from https://github.com/FernandoCelmer/skills
```

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
