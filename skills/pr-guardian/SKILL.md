---
allowed-tools: Bash(gh pr list:*), Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh pr edit:*), Bash(gh api:*), Bash(gh repo clone:*), Bash(gh label create:*), Bash(gh label list:*), Bash(git log:*), Bash(git diff:*), Bash(git blame:*), Bash(git status:*), Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(git checkout:*), Bash(git ls-files:*), Bash(python3:*), Bash(jq:*), Read, Glob, Grep, Agent, Write, Edit, CronCreate, CronDelete
description: Use this skill when the user asks to "guard PRs", "monitor PRs", "watch PRs", "vigiar PRs", "ficar de olho nos PRs", "validar e corrigir PRs", or wants continuous PR monitoring with auto-review, auto-fix of review comments, and label management across one or more repositories.
version: 2.0.0
---

# PR Guardian

Continuously monitor open PRs across multiple repositories. For each new PR: review code, post inline comments, fix blocking issues, apply labels, and keep watching for new PRs on a recurring interval.

Make a todo list and track progress through all steps.

---

## Step 0 — Determine targets and interval

Parse user input to extract:

1. **Repositories**: GitHub org/repo URLs or names (e.g. `acme-corp/backend`, `https://github.com/org/repo`)
2. **Interval**: How often to check (default: `1m`). Accept formats like `30s`, `1m`, `5m`, `10m`.
3. **Scope**: Which PR states to monitor (default: `open`)

If no repositories provided, ask the user.

If the user provides a GitHub organization URL, list all repos:
```bash
gh repo list <org> --json name,url --limit 50
```

---

## Step 1 — Setup labels

Create standardized labels in all target repositories if they don't exist:

```bash
gh label create "reviewed" --repo <owner/repo> --color "1D76DB" --description "PR reviewed and validated" --force
gh label create "priority: critical" --repo <owner/repo> --color "B60205" --description "Runtime failures, data loss, security" --force
gh label create "priority: high" --repo <owner/repo> --color "D93F0B" --description "Incorrect behavior, performance, API breakage" --force
gh label create "priority: medium" --repo <owner/repo> --color "FBCA04" --description "Missing features, coverage gaps, bad UX" --force
gh label create "priority: low" --repo <owner/repo> --color "0E8A16" --description "Style, docs, minor improvements" --force
```

---

## Step 2 — Scan for unreviewed PRs

For each target repository, list open PRs without the `reviewed` label:

```bash
gh pr list --repo <owner/repo> --state open --json number,title,labels \
  --jq '.[] | select(.labels | map(.name) | index("reviewed") | not) | "#\(.number) \(.title)"'
```

If no unreviewed PRs found, report "No new PRs" and wait for next cycle.

---

## Step 3 — Review each PR (parallel)

Launch one agent per unreviewed PR. Each agent performs:

### 3a. Collect context
```bash
gh pr view <number> --repo <owner/repo> --json title,body,state,isDraft,headRefName,baseRefName
gh pr diff <number> --repo <owner/repo>
```

Check for CLAUDE.md in the repo root and in modified directories.

### 3b. Analyze code changes

Scan the diff for:

**Correctness & Logic**
- Logic errors, wrong conditions, inverted booleans
- Off-by-one, boundary conditions
- Null/undefined/empty handling
- Async issues: race conditions, missing awaits, deadlocks
- Error handling: silent catches, swallowed exceptions, missing rollbacks

**Security (OWASP Top 10)**
- Injection (SQL, command, template)
- Broken authentication, sensitive data exposure
- Broken access control, IDOR, privilege escalation
- XSS, SSRF, path traversal, open redirects
- Hardcoded secrets or credentials

**Performance**
- N+1 queries, missing pagination
- Unnecessary recomputation, blocking I/O in async
- Missing caching, unbounded growth

**Code Quality**
- Unclear naming, functions too long
- DRY/KISS violations, dead code
- Missing error handling at boundaries

### 3c. Score each finding (0-100)

| Score | Meaning |
|-------|---------|
| 0 | False positive or pre-existing |
| 25 | Possible but uncertain |
| 50 | Real but minor |
| 75 | Highly likely, important |
| 100 | Confirmed, will occur frequently |

**Discard findings below 75.**

### 3d. Post inline review comments

Build JSON payload and post via GitHub API:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --method POST --input /tmp/review_<number>.json
```

Build with Python to avoid shell escaping:

```python
import json
review = {
  "body": "...",
  "event": "COMMENT",
  "comments": [
    {"path": "src/file.py", "position": 12, "body": "..."}
  ]
}
with open("/tmp/review_<number>.json", "w") as f:
    json.dump(review, f)
```

Comment format:
```
[Blocking|Suggestion|Question|Comment]

**Problem** — description and why it matters.

**Failure scenario** — concrete example of when it breaks.

**Fix** — code suggestion with explanation.
```

### 3e. Update PR description

If incomplete, rebuild using:
```markdown
## Description
<!-- Modified files with brief explanation -->

## Motivation and Context
<!-- Why + issue reference -->

## Types of changes
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation

## Checklist
- [ ] Self-review done
- [ ] Tests added
- [ ] Documentation updated
```

### 3f. Apply labels

1. Apply `reviewed` label
2. Apply contextual labels based on changes: `bug`, `enhancement`, `documentation`, `security`
3. Only use labels that exist in the repo

---

## Step 4 — Check for pending review comments

After reviewing new PRs, check ALL reviewed PRs for unresolved comments:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments \
  --jq '.[] | "\(.path):\(.original_line) \(.body | split("\n")[0])"'
```

Compare last comment date vs last commit date:
- If **last commit < last comment**: comments are unresolved
- If **last commit > last comment**: developer may have addressed them

Report unresolved comments with their severity.

---

## Step 5 — Fix blocking comments (MANDATORY)

**This step is NOT optional.** After reviewing PRs, ALWAYS fix all `[Blocking]` comments automatically. Do NOT wait for the user to ask.

For each PR with unresolved `[Blocking]` comments:

1. Clone the repo on the PR branch:
   ```bash
   gh repo clone <owner/repo> /tmp/fix-pr<number> -- -b <branch>
   ```

2. Read the review comment to understand the required fix

3. Read the affected file(s)

4. Apply the fix

5. Commit with the appropriate convention:
   ```bash
   git commit -m "$(cat <<'EOF'
   <emoji> <type> Fix review comment on <file> (#<issue>)
   EOF
   )"
   ```

6. Push to the PR branch

7. **Resolve the review thread** using the GitHub GraphQL API:
   ```bash
   # Get thread ID
   gh api graphql -f query='query { repository(owner:"<owner>",name:"<repo>") { pullRequest(number:<N>) { reviewThreads(first:20) { nodes { id isResolved comments(first:1) { nodes { body } } } } } } }' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false) | .id'

   # Resolve each thread
   gh api graphql -f query='mutation { resolveReviewThread(input:{threadId:"<THREAD_ID>"}) { thread { isResolved } } }'
   ```

Rules for fixing:
- **ALWAYS fix `[Blocking]` comments** — this is mandatory, not optional
- Also fix `[Suggestion]` comments when the fix is straightforward (< 5 lines)
- Never change code beyond what the comment requests
- Preserve existing code style and conventions
- If the fix requires architectural changes, report instead of fixing
- **ALWAYS resolve the GitHub review thread after fixing** — never leave threads unresolved

---

## Step 6 — Verify all threads resolved

After fixing, verify NO unresolved threads remain:

```bash
gh api graphql -f query='query { repository(owner:"<owner>",name:"<repo>") { pullRequest(number:<N>) { reviewThreads(first:20) { nodes { isResolved } } } } }' --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)] | length'
```

If any remain unresolved, either fix them or explain why they cannot be fixed.

---

## Step 7 — Report results

After each cycle, output a summary:

```
## PR Monitor — Cycle Report

**Repos**: repo1, repo2, repo3
**New PRs reviewed**: N
**Blockers fixed**: N
**Threads resolved**: N
**Pending comments**: N

| Repo | PR | Title | Verdict | Issues | Fixed |
|------|----|-------|---------|--------|-------|
| repo | #N | title | LGTM / N blockers | details | Y/N |

**Next check**: in Xm
```

---

## Step 8 — Start monitoring loop

Set up recurring check using CronCreate:

```
CronCreate:
  cron: "*/N * * * *"  (based on requested interval)
  prompt: "Check for unreviewed PRs in <repos> and review them"
  recurring: true
```

Report the job ID so the user can cancel with `CronDelete <id>`.

---

## Guidelines

### What to flag
- Real bugs introduced by the PR (not pre-existing)
- Security vulnerabilities
- Performance regressions
- Breaking API changes
- Missing error handling at boundaries
- CLAUDE.md violations

### What NOT to flag
- Pre-existing issues not introduced by this PR
- Issues a linter/compiler would catch (assume CI runs those)
- Nitpicks a senior engineer wouldn't raise
- Intentional changes clearly part of the PR's goal

### Review comment rules
- Use `[Blocking]`, `[Suggestion]`, `[Question]`, `[Comment]` prefixes
- Be didactic — explain WHY something is wrong
- Use Nonviolent Communication
- Include concrete code fix suggestions
- Never mention AI, automation, or Claude in comments
- Link to code using full commit SHA

### Parallel execution
- Review multiple PRs in parallel using Agent tool
- Fix multiple PRs in parallel when requested
- Never duplicate work across agents

### Mandatory post-review actions
- **ALWAYS fix [Blocking] comments** — clone branch, apply fix, commit, push
- **ALWAYS resolve GitHub review threads** after fixing via GraphQL API
- **ALWAYS verify** zero unresolved threads remain before reporting
- Never leave a cycle with unresolved blocking comments
- Suggestions with < 5 line fixes should also be fixed automatically

### Safety
- Never force-push or amend commits
- Never merge PRs (only review and fix)
- Never skip git hooks
- Stage specific files, never `git add .`
- Never commit secrets or credentials
