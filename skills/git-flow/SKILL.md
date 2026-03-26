---
name: git-flow
description: Enforce branch naming and commit message conventions. Use when the user asks to implement something from an issue, create a branch, or commit changes. Branches follow the pattern feature/ISSUE-NUMBER from develop. Commits follow the icon+type+issue pattern.
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

You are enforcing the project's Git flow conventions. Follow these rules strictly for every branch and commit in this session.

---

## Branch Convention

When implementing something derived from a GitHub issue:

1. **Always branch from `develop`** (never from `main` or `master` directly)
2. **Branch name pattern:** `feature/ISSUE-NUMBER`
   - Example: issue #42 → `feature/42`
3. Create the branch with:
   ```bash
   git checkout develop && git pull origin develop
   git checkout -b feature/ISSUE-NUMBER
   ```

---

## Commit Convention

Every commit message must follow this exact format:

```
ICON TYPE-#NUMBER: (type) Comment in English
```

### Icons and types

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| ⚙️   | FEATURE  | New feature                                |
| 📝   | PEP8     | Formatting fixes following PEP8            |
| 📌   | ISSUE    | Reference to issue / work in progress      |
| 🪲   | BUG      | Bug fix                                    |
| 📘   | DOCS     | Documentation changes                      |
| 📦   | PyPI     | PyPI releases                              |
| ❤️   | TEST     | Automated tests                            |
| ⬆️   | CI/CD    | Changes in continuous integration/delivery |
| ⚠️   | SECURITY | Security improvements                      |

### Examples

```
⚙️ FEATURE-#42: (feature) Add S3 storage provider
🪲 BUG-#68: (bug) Fix busy-wait loop in parallel execution mode
📌 ISSUE-#73: (feature) Add LLM pipeline example to documentation
📝 PEP8-#70: (pep8) Fix return type annotation on TaskBuilder.add
❤️ TEST-#42: (test) Add unit tests for StorageS3
📘 DOCS-#73: (docs) Add AI/LLM pipeline usage guide
⬆️ CI/CD-#48: (ci) Add GitHub Actions workflow for automated tests
⚠️ SECURITY-#0: (security) Sanitize user input in CLI arguments
```

---

## Workflow for implementing an issue

When the user says "implement issue #N" or "work on issue #N":

1. **Fetch the issue** to understand scope:
   ```bash
   gh issue view N --repo OWNER/REPO
   ```

2. **Create the branch from develop:**
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/N
   ```

3. **Implement the changes.**

4. **Commit using the correct icon and type** based on the issue label:
   - `bug` label → 🪲 BUG
   - `enhancement` label → ⚙️ FEATURE
   - `documentation` label → 📘 DOCS
   - `discovery` label → 📌 ISSUE
   - No label or unclear → use 📌 ISSUE

5. **Push the branch:**
   ```bash
   git push origin feature/N
   ```

6. **Open a PR targeting `develop`** (never `main`/`master` directly):
   ```bash
   gh pr create --base develop --title "⚙️ FEATURE-#N: description" --body "Closes #N"
   ```

---

## Rules summary

- Never commit to `main`, `master`, or `develop` directly
- Never branch from `main` or `master` for feature work
- Always include the issue number in both branch name and commit message
- Commit messages must be in English
- Always confirm with the user before pushing or opening a PR
