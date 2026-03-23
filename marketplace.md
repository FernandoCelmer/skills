---
name: marketplace
description: Install, list or search skills from FernandoCelmer's skills marketplace (https://github.com/FernandoCelmer/skills). Use when the user says "install skill", "list skills", "list marketplace", "search marketplace", "show skills", or similar.
---

You are the skills marketplace for https://github.com/FernandoCelmer/skills.

The registry is at:
https://raw.githubusercontent.com/FernandoCelmer/skills/master/registry.json

## Instructions

### List or search
Fetch registry.json and display available skills in a table with name, version, category and description.

### Install a skill
1. Fetch registry.json
2. Find the entry where `name` matches
3. Download the file from `entry.url` and save to `~/.claude/commands/<name>.md`
4. Tell the user to start a new Claude Code session and use `/<name>`

Use this bash pattern:
```bash
curl -fsSL <entry.url> -o ~/.claude/commands/<name>.md
```

### Update all skills
Download and overwrite all skills from registry.json into `~/.claude/commands/`.

### Self-update
To update the marketplace itself:
```bash
curl -fsSL https://raw.githubusercontent.com/FernandoCelmer/skills/master/marketplace.md -o ~/.claude/commands/marketplace.md
```

If the user provides no specific action, list available skills and ask what they want to install.
