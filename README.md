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
| [git-flow](skills/git-flow/SKILL.md) | 1.0.0 | development | Enforce branch naming (feature/ISSUE-NUMBER from develop) and commit message conventions (icon + type + issue reference) for every implementation |
| [repo-audit](skills/repo-audit/SKILL.md) | 1.0.0 | development | Deep technical audit of a repository: find bugs, gaps, missing tests and security issues, then create GitHub issues for findings |
| [review-issues](skills/review-issues/SKILL.md) | 1.0.0 | development | Analyze open issues across one or more repositories |
| [smart-review-pr](skills/smart-review-pr/SKILL.md) | 2.0.0 | development | Comprehensive PR review covering code quality, security, architecture and design patterns |

---

## Installing a skill

```bash
claude plugins install <skill-name>@FernandoCelmer-skills
```

Example:

```bash
claude plugins install git-flow@FernandoCelmer-skills
```

---

## How to contribute

1. Fork this repository
2. Create your skill at `skills/<name>/SKILL.md`
3. Add the entry to [registry.json](registry.json) and [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json)
4. Open a Pull Request

### Skill file format

```markdown
---
name: skill-name
description: What this skill does (used by Claude to decide when to invoke it)
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

Full prompt/instructions for the skill...
```
