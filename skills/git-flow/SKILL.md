---
name: git-flow
description: "Enforce branch naming and commit message conventions. Use when the user asks to implement something from an issue, create a branch, or commit changes. Branches follow the pattern feature/ISSUE-NUMBER from develop. Commits follow Conventional Commits 1.0.0 (type(scope): description, issue in the footer)."
version: 2.0.1
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

Every commit message follows [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

When the commit is derived from an issue, reference it in a footer:

```
fix(parser): handle empty task list

Refs: #42
```

When there is **no issue associated**, omit the footer.

### Types

| Type       | When to use                                                  |
|------------|--------------------------------------------------------------|
| `feat`     | New feature (bumps MINOR)                                    |
| `fix`      | Bug fix (bumps PATCH)                                        |
| `docs`     | Documentation only                                           |
| `style`    | Formatting, whitespace, lint fixes — no behavior change      |
| `refactor` | Code change that neither fixes a bug nor adds a feature      |
| `perf`     | Performance improvement                                      |
| `test`     | Adding or fixing tests                                       |
| `build`    | Build system or dependencies (`pyproject.toml`, `package.json`, lock files, `Cargo.toml`, `go.mod`) |
| `ci`       | CI configuration and scripts                                 |
| `chore`    | Maintenance that touches neither source nor tests (releases, tooling) |
| `revert`   | Reverts a previous commit                                    |

The same types apply to every language — there are no language-specific types.

### Scope

Optional, in parentheses, a noun naming the part of the codebase: `feat(api):`, `fix(cli):`, `build(deps):`. Use the scopes the project already uses (check `git log`); do not invent a new scope per commit.

### Breaking changes

Mark with `!` before the colon, a `BREAKING CHANGE:` footer, or both:

```
feat(api)!: drop the v1 endpoints

BREAKING CHANGE: clients must call /v2.
Refs: #42
```

### Formatting rules

- `type` and `scope` in lowercase
- Description in English, imperative mood, lowercase first letter, no trailing period
- Subject line (`type(scope): description`) under 72 characters
- Blank line between subject, body and footers
- Footers use `Token: value` (`Refs: #42`, `BREAKING CHANGE: ...`)

### Examples

```
feat(storage): add S3 storage provider
fix(executor): stop busy-wait loop in parallel mode
style: apply ruff format to abc/server.py
build(deps): bump dotflow to 0.15.0.dev5
test(serializer): cover list and tuple Context items
docs: add install instructions to README
ci: run tests on Python 3.13
```

---

## Message length

Commit messages must be **short**. Default to **subject line** plus the
`Refs: #N` footer when there is an issue. Only add a body when there is
a non-obvious *why* that fits in **one short sentence**. No
multi-paragraph bodies, no bullet lists, no context dumps — the diff
and the PR description already carry detail.

### Good

```
feat(storage): add S3 storage provider

Refs: #42
```

### Bad

```
feat: Added S3 storage provider.

This commit introduces a new S3 storage provider that allows
users to persist workflow state in S3 buckets. The provider
implements the Storage ABC and supports...
(long body rambles on)
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

1. Build / dependency changes (`pyproject.toml`, `package.json`, `Cargo.toml`, lock files)
2. Implementation (source code)
3. Tests
4. Documentation (`README.md`, `DEPLOY.md`, release notes)
5. Formatting / lint fixes

### Example of structured commits for a single issue

**Python:**
```bash
git add dotflow/core/serializers/task.py
git commit -m "fix(serializer): handle non-Context list items" -m "Refs: #247"

git add tests/core/test_serializer_task.py
git commit -m "test(serializer): cover list, tuple and mixed Context items" -m "Refs: #247"

git add pyproject.toml
git commit -m "build: limit pytest testpaths to tests/" -m "Refs: #247"

git add poetry.lock
git commit -m "build(deps): regenerate poetry.lock" -m "Refs: #247"
```

**JavaScript / React:**
```bash
git add src/components/Dashboard.tsx
git commit -m "feat(dashboard): add chart component" -m "Refs: #42"

git add src/__tests__/Dashboard.test.tsx
git commit -m "test(dashboard): cover chart component" -m "Refs: #42"

git add package.json
git commit -m "build(deps): add recharts" -m "Refs: #42"

git add package-lock.json
git commit -m "build(deps): regenerate package-lock.json" -m "Refs: #42"
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
- Types outside the Conventional Commits list above, uppercase types, or emoji prefixes

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

4. **Commit using the matching type** based on the issue label:
   - `bug` label → `fix`
   - `enhancement` label → `feat`
   - `documentation` label → `docs`
   - No label or unclear → pick the type that describes the change (`refactor`, `chore`, …)

5. **Push the branch:**
   ```bash
   git push origin feature/N
   ```

6. **Open a PR targeting `develop`** (never `main`/`master` directly), with the title in the same format as a commit subject:
   ```bash
   gh pr create --base develop --title "feat(scope): description" --body "Closes #N"
   ```

---

## Rules summary

- Never commit to `main`, `master`, or `develop` directly
- Never branch from `main` or `master` for feature work
- Always include the issue number in the branch name and a `Refs: #N` footer in the commit
- Commit messages follow Conventional Commits 1.0.0, in English
- **Always create one commit per file or per logical concern — never bundle unrelated changes**
- **Keep messages short — subject line plus footer by default; body only for a one-sentence *why* when non-obvious**
- **Stage specific files explicitly — never use `git add .` or `git add -A`**
- Always confirm with the user before pushing or opening a PR
