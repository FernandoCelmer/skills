---
name: git-flow
description: Enforce branch naming and commit message conventions. Use when the user asks to implement something from an issue, create a branch, or commit changes. Branches follow the pattern feature/ISSUE-NUMBER from develop. Commits follow the icon+type+issue pattern.
version: 1.2.0
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

### Icons and types by language/ecosystem

**Detect the project language** from `pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, etc. and use only the icons that apply.

#### Universal (all languages)

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| ⚙️   | FEATURE  | New feature                                |
| 📌   | ISSUE    | Reference to issue / work in progress      |
| 🪲   | BUG      | Bug fix                                    |
| 📘   | DOCS     | Documentation changes                      |
| ❤️   | TEST     | Automated tests                            |
| ⬆️   | CI/CD    | Changes in continuous integration/delivery |
| ⚠️   | SECURITY | Security improvements                      |

#### Python only

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| 📝   | PEP8     | Formatting fixes following PEP8            |
| 📦   | PyPI     | PyPI releases, dependency updates          |

#### JavaScript / TypeScript (React, Node, etc.)

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| 📝   | LINT     | ESLint, Prettier, formatting fixes         |
| 📦   | NPM      | npm/yarn releases, dependency updates      |

#### Rust

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| 📝   | CLIPPY   | Clippy lints and formatting fixes          |
| 📦   | CRATE    | Crate releases, dependency updates         |

#### Go

| Icon | Type     | When to use                                |
|------|----------|--------------------------------------------|
| 📝   | LINT     | golangci-lint, gofmt fixes                 |
| 📦   | MOD      | go.mod dependency updates                  |

### Examples

**Python:**
```
⚙️ FEATURE-#42: Add S3 storage provider
🪲 BUG-#68: Fix busy-wait loop in parallel execution mode
📝 PEP8-#70: Fix return type annotation on TaskBuilder.add
📦 PyPI-#42: Update dotflow to 0.15.0.dev5
```

**JavaScript / React:**
```
⚙️ FEATURE-#42: Add dashboard chart component
🪲 BUG-#68: Fix state leak in useEffect cleanup
📝 LINT-#70: Fix ESLint warnings in api module
📦 NPM-#42: Update react to 19.1.0
```

**Rust:**
```
⚙️ FEATURE-#42: Add S3 backend for storage
🪲 BUG-#68: Fix deadlock in async task pool
📝 CLIPPY-#70: Fix needless borrow warnings
📦 CRATE-#42: Bump tokio to 1.40
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

1. Config / dependency changes (`pyproject.toml`, `package.json`, `Cargo.toml`, lock files)
2. Implementation (source code)
3. Tests
4. Documentation (`README.md`, `DEPLOY.md`, release notes)
5. Formatting / lint fixes

### Example of structured commits for a single issue

**Python:**
```bash
git add dotflow/core/serializers/task.py
git commit -m "🪲 BUG-#247: Fix _serialize_context crash with non-Context list items"

git add tests/core/test_serializer_task.py
git commit -m "❤️ TEST-#247: Add tests for list, tuple, and mixed Context serialization"

git add pyproject.toml
git commit -m "⚙️ FEATURE-#247: Configure pytest testpaths to exclude examples"

git add poetry.lock
git commit -m "📦 PyPI-#247: Regenerate poetry.lock"
```

**JavaScript / React:**
```bash
git add src/components/Dashboard.tsx
git commit -m "⚙️ FEATURE-#42: Add dashboard chart component"

git add src/__tests__/Dashboard.test.tsx
git commit -m "❤️ TEST-#42: Add unit tests for Dashboard component"

git add package.json
git commit -m "📦 NPM-#42: Add recharts dependency"

git add package-lock.json
git commit -m "📦 NPM-#42: Regenerate package-lock.json"
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
- Using Python-specific types (PEP8, PyPI) in non-Python projects
- Using JS-specific types (LINT, NPM) in non-JS projects

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
- **Detect the project language and use only applicable commit types**
- **Always create one commit per file or per logical concern — never bundle unrelated changes**
- **Stage specific files explicitly — never use `git add .` or `git add -A`**
- Always confirm with the user before pushing or opening a PR
