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
| [email-profile](skills/email-profile/SKILL.md) | 1.0.0 | development | Generate Python code using the email-profile library for email automation (IMAP/SMTP) |
| [git-flow](skills/git-flow/SKILL.md) | 1.3.0 | development | Enforce branch naming, structured commits per file, language-specific commit types (Python/JS/Rust/Go), and git-flow conventions |
| [repo-audit](skills/repo-audit/SKILL.md) | 1.0.0 | development | Deep technical audit of a repository: find bugs, gaps, missing tests and security issues, then create GitHub issues for findings |
| [review-issues](skills/review-issues/SKILL.md) | 1.0.0 | development | Analyze open issues across one or more repositories |
| [smart-review-pr](skills/smart-review-pr/SKILL.md) | 2.0.0 | development | Comprehensive PR review covering code quality, security, architecture and design patterns |
| [telegram-bridge](skills/telegram-bridge/SKILL.md) | 1.0.0 | automation | Telegram bot that bridges messages to Claude Code CLI — control Claude from Telegram |

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
