# Contributing to Skills Marketplace

Thank you for contributing! This guide explains how to create, test, and submit a skill.

---

## File Structure

Every skill lives in its own directory under `skills/`:

```
skills/
└── your-skill-name/
    └── SKILL.md
```

Two manifest files must also be updated:

| File | Purpose |
|------|---------|
| `registry.json` | Used by the `/marketplace` skill command to list available skills |
| `.claude-plugin/marketplace.json` | Used by the Claude Code plugin system (`claude plugins install`) |

---

## SKILL.md Format

```markdown
---
name: skill-name
description: One-line description — Claude uses this to decide when to invoke the skill. Be specific about triggers.
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

Full prompt and instructions for the skill...
```

### Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Lowercase, hyphenated. Must match the directory name. |
| `description` | yes | Used by Claude to auto-trigger the skill. Include trigger phrases. |
| `version` | yes | Follows semver: `MAJOR.MINOR.PATCH` |
| `allowed-tools` | yes | Comma-separated list. Only include tools the skill actually needs. |

---

## Writing a Good Description

The `description` field is critical — Claude reads it to decide when to invoke the skill automatically.

**Good:**
```
description: Use when the user asks to "review PR", "validate PR", "checar PR", or mentions a PR number to review before merging.
```

**Bad:**
```
description: Reviews pull requests.
```

Rules:
- Include trigger phrases the user would naturally say (in multiple languages if relevant)
- Be specific about what the skill does vs. what it does not do
- Add `DO NOT TRIGGER when:` if there are common false-positive cases

---

## allowed-tools Best Practices

Only request tools the skill genuinely needs. Fewer tools = better security and trust.

| Tool | Use when |
|------|----------|
| `Bash` | Running shell commands (`gh`, `git`, `curl`, etc.) |
| `Read` | Reading local files |
| `Edit` | Modifying existing files |
| `Write` | Creating new files |
| `Glob` | Finding files by pattern |
| `Grep` | Searching file contents |
| `WebFetch` | Fetching URLs |
| `WebSearch` | Searching the web |
| `Agent` | Spawning subagents for parallel work |

Avoid requesting `Write` or `Edit` if the skill is read-only. Avoid `Bash` if only file tools are needed.

---

## Versioning

Follow [semver](https://semver.org/):

| Change | Version bump |
|--------|-------------|
| Bug fix or wording improvement | `PATCH` (1.0.0 → 1.0.1) |
| New feature, backward compatible | `MINOR` (1.0.0 → 1.1.0) |
| Breaking change in behavior | `MAJOR` (1.0.0 → 2.0.0) |

Update the version in **all three places**: `SKILL.md`, `registry.json`, and `.claude-plugin/marketplace.json`.

---

## Testing a Skill Before Submitting

1. Install your skill locally:
   ```bash
   claude plugins install --local ./skills/your-skill-name
   ```

2. Start a new Claude Code session and verify the skill appears in the system prompt.

3. Trigger it with a natural language phrase matching your `description`.

4. Confirm it uses only the tools listed in `allowed-tools`.

---

## Submitting a Pull Request

1. Fork this repository
2. Create a branch: `feature/your-skill-name`
3. Add your skill to `skills/your-skill-name/SKILL.md`
4. Add an entry to `registry.json`
5. Add an entry to `.claude-plugin/marketplace.json`
6. Open a PR targeting `master`

PR title format: `⚙️ FEATURE: Add <skill-name> skill`
