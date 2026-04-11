---
name: git-flow
description: Enforce branch naming and commit message conventions. Use when the user asks to implement something from an issue, create a branch, or commit changes. Branches follow the pattern feature/ISSUE-NUMBER from develop. Commits follow the icon+type+issue pattern.
version: 1.1.0
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
ICON TYPE-#NUMBER: Comment in English
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
⚙️ FEATURE-#42: Add S3 storage provider
🪲 BUG-#68: Fix busy-wait loop in parallel execution mode
📌 ISSUE-#73: Add LLM pipeline example to documentation
📝 PEP8-#70: Fix return type annotation on TaskBuilder.add
❤️ TEST-#42: Add unit tests for StorageS3
📘 DOCS-#73: Add AI/LLM pipeline usage guide
⬆️ CI/CD-#48: Add GitHub Actions workflow for automated tests
⚠️ SECURITY-#0: Sanitize user input in CLI arguments
```

---

## Structured and separated commits

When implementing multiple fixes or changes in the same branch, **always create one commit per file or per logical concern**. Never bundle unrelated changes into a single commit.

### Rules for structuring commits

1. **One commit per file or concern** — each commit must be independently understandable and revertable
2. **Stage only related files** — use `git add <specific files>` instead of `git add .` or `git add -A`
3. **Commit in logical order** — infrastructure/config changes first, then implementation, then tests, then docs
4. **Separate by type** — never mix bug fixes with formatting, tests with docs, or config with implementation in the same commit

### Commit ordering priority

1. Config / dependency changes (`pyproject.toml`, `requirements.txt`, `poetry.lock`)
2. Implementation (source code)
3. Tests
4. Documentation (`README.md`, `DEPLOY.md`, release notes)
5. Formatting / PEP8 fixes

### Example of structured commits for a single issue

```bash
# 1. Implementation fix
git add dotflow/core/serializers/task.py
git commit -m "🪲 BUG-#247: Fix _serialize_context crash with non-Context list items"

# 2. Tests for the fix
git add tests/core/test_serializer_task.py
git commit -m "❤️ TEST-#247: Add tests for list, tuple, and mixed Context serialization"

# 3. Config change
git add pyproject.toml
git commit -m "⚙️ FEATURE-#247: Configure pytest testpaths to exclude examples"

# 4. Lock file
git add poetry.lock
git commit -m "📦 PyPI-#247: Regenerate poetry.lock"

# 5. Documentation
git add docs/nav/development/release-notes.md
git commit -m "📘 DOCS-#247: Add PR #249 to release notes"

# 6. Formatting
git add dotflow/core/serializers/task.py
git commit -m "📝 PEP8-#247: Apply ruff format with project config"
```

### When NOT to split commits

- A test and the exact implementation it covers when they are tightly coupled
- Import changes required by and only meaningful alongside a specific fix
- Minor co-located changes (e.g. fixing a typo in the same line as a bug fix)

### Strictly forbidden

- `git add .` or `git add -A` — always stage specific files by name
- Bundling unrelated files in one commit (e.g. `task.py` + `README.md`)
- Committing lock files together with source code changes
- Committing formatting fixes together with logic changes

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
- **Always create one commit per file or per logical concern — never bundle unrelated changes**
- **Stage specific files explicitly — never use `git add .` or `git add -A`**
- Always confirm with the user before pushing or opening a PR
