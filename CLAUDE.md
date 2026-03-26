# Skills Marketplace — Claude Guidelines

This is a marketplace of Claude Code skills and plugins. When working in this repository, follow these rules.

---

## Repository Structure

```
skills/
└── <skill-name>/
    └── SKILL.md              ← skill prompt and frontmatter

.claude-plugin/
└── marketplace.json          ← plugin system manifest (used by `claude plugins install`)

registry.json                 ← skill list (used by the /marketplace slash command)
CONTRIBUTING.md               ← contribution guide for new skills
```

---

## When Adding or Updating a Skill

Always update **all three** locations together:

1. `skills/<name>/SKILL.md` — the skill file itself
2. `registry.json` — add or update the entry
3. `.claude-plugin/marketplace.json` — add or update the entry

Forgetting any one of these will cause the skill to be missing from either the plugin system or the `/marketplace` command.

---

## SKILL.md Requirements

Every `SKILL.md` must have valid frontmatter:

```markdown
---
name: skill-name
description: <specific trigger description with example phrases>
version: <semver>
allowed-tools: <comma-separated list of only tools needed>
---
```

- `name` must be lowercase and hyphenated, matching the directory name
- `description` must be specific enough for Claude to auto-trigger correctly
- `allowed-tools` must be minimal — only tools the skill genuinely uses

---

## Versioning Rules

| Change | Bump |
|--------|------|
| Wording fix, bug fix | PATCH |
| New capability, backward compatible | MINOR |
| Behavior change that breaks existing use | MAJOR |

Always bump the version in `SKILL.md`, `registry.json`, and `.claude-plugin/marketplace.json` simultaneously.

---

## Branch and Commit Convention

Follow the project git-flow conventions:

**Branch:** `feature/ISSUE-NUMBER` (when derived from an issue) or `feature/<description>`

**Commit format:**

| Icon | Type | When |
|------|------|------|
| ⚙️ | FEATURE | New skill or capability |
| 🪲 | BUG | Fix broken skill behavior |
| 📘 | DOCS | README, CONTRIBUTING, CLAUDE.md |
| 📝 | PEP8 | Formatting, wording cleanup |
| ❤️ | TEST | Testing or validation |
| ⬆️ | CI/CD | GitHub Actions, automation |

Examples:
```
⚙️ FEATURE: Add git-flow skill
📘 DOCS: Update README with install instructions
🪲 BUG: Fix allowed-tools missing WebFetch in review-issues
```

---

## What NOT to Do

- Do not add a skill to only `registry.json` without updating `.claude-plugin/marketplace.json` — the plugin system won't find it
- Do not use broad `allowed-tools` (e.g., listing all tools) — request only what is needed
- Do not write vague `description` fields — Claude relies on them for auto-invocation
- Do not bump MAJOR version for minor wording changes
- Do not commit directly to `master` — always use a branch and PR
