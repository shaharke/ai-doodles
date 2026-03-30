# ai-doodles → ai-marketplace Sync Analysis

## Overview

`ai-doodles` has 2 plugins (`gitops`, `aiops`) with 6 total skills. `ai-marketplace` has 7 plugins with 24+ skills. Three skills overlap between repos, and three skills in `ai-doodles` are absent from `ai-marketplace`.

---

## Duplicate Skills

### 1. `commit`

| Feature | ai-doodles (`gitops`) | ai-marketplace (`lmnd-devx`) |
|---|---|---|
| Auto-invocation | yes | `disable-model-invocation: true` |
| Staging strategy | `git add -A` (all changes) | Session files only (`git add <specific-files>`) |
| Execution | Background via `Task` tool | Foreground |
| Message rules | Detailed — focus on value, imperative mood, 72-char limit, good/bad examples | Minimal |
| Argument support | `/commit auth` → `feat(auth): ...` | None |
| Commit command | HEREDOC | `echo -e \| git commit -F -` |

**Recommendation**: ai-marketplace has better staging discipline (session-only, avoids staging unrelated work). ai-doodles has better commit message rules, background execution, and argument support. **Merge**: keep ai-marketplace's selective staging + `disable-model-invocation`, add ai-doodles' message rules, background Task execution, and argument support.

---

### 2. `deslop`

| Feature | ai-doodles (`aiops`) | ai-marketplace (`lmnd-devx`) |
|---|---|---|
| Auto-invocation | yes | `disable-model-invocation: true` |
| Steps | Explicit 5-step process | Single paragraph |
| File context | Reads full file for surrounding style | Not mentioned |
| Slop categories | Listed in detail (comments, try/catch, `as any`, style) | Bullet list only |

**Recommendation**: ai-marketplace's version is a stripped-down copy. The ai-doodles version is substantially better. **Replace** ai-marketplace's body with the ai-doodles version, keeping `disable-model-invocation: true`.

---

### 3. `pr-resolve`

The two versions are nearly identical (ai-marketplace's was derived from ai-doodles). Two divergences:

| Feature | ai-doodles (`gitops`) | ai-marketplace (`lmnd-devx`) |
|---|---|---|
| Phase 3 user prompt | Uses `AskUserQuestion` explicitly | Generic "ask the user" |
| After applying fixes | Auto-commits and pushes | Suggests user run `/commit` |

**Recommendation**: Merge both improvements into ai-marketplace: use `AskUserQuestion` in Phase 3, and auto-commit+push after applying fixes.

---

## Skills Missing from ai-marketplace

All three are in `ai-doodles/plugins/gitops/` and have no equivalent in `ai-marketplace`.

### 4. `pr-fix-checks`
**Description**: Inspect failing CI checks on a GitHub PR, fetch logs, and suggest fixes.

**Workflow summary**:
- Phase 0: Resolve PR from URL or current branch
- Phase 1 (Haiku subagent): Fetch check statuses, filter to FAILURE/ERROR; exit early if all passing
- Phase 1.5: Auto-select single failure or prompt user to pick among multiple
- Phase 2a (Haiku subagent): Fetch failure logs via `gh run view --log-failed`; truncate to 2000 lines if huge
- Phase 2b (Opus subagent): Analyze root cause; produce structured report (Problem Summary, Error Details, Root Cause, Suggested Fix with Edit blocks, Confidence, Alternatives)
- Phase 3: Present report; ask Apply / Show full logs / Skip; apply edits

**Recommendation**: Add to `lmnd-devx` with `disable-model-invocation: true`.

---

### 5. `pr-fix-yolo`
**Description**: Continuously fix PR review comments and failing CI checks in a loop until everything is green.

**Workflow summary**:
- Phase 0: Resolve PR; set loop interval (default 5m); create `CronCreate` job; run first iteration immediately
- Each iteration:
  1. Wait if BugBot review is still pending
  2. Check CI status + unresolved review threads in parallel
  3. If all green and resolved → cancel cron, done
  4. Fix failing CI checks autonomously
  5. For each unresolved thread: classify as minor (auto-apply) or major (pause, notify user, cancel cron)
  6. Commit and push all file changes

**Design principle**: Handles 80% of feedback autonomously; pauses for major/architectural changes.

**Recommendation**: Add to `lmnd-devx` with `disable-model-invocation: true`.

---

### 6. `pr-rebase`
**Description**: Rebase the current PR branch onto its base branch to bring it up to date.

**Workflow summary**:
- Phase 0: Resolve PR from URL or current branch
- Phase 1: Get `headRefName` and `baseRefName` via `gh pr view`
- Phase 2: Checkout head branch if not already on it; abort if uncommitted changes; fetch latest base
- Phase 3: Run `git rebase origin/{baseBranch}`; on conflict: list files, ask user to abort or get help; on success: force-push with `--force-with-lease`

**Recommendation**: Add to `lmnd-devx` with `disable-model-invocation: true`.

---

## Summary of Recommended Changes

| Action | Target | Notes |
|---|---|---|
| Update | `lmnd-devx/commit` | Merge message rules + background execution + arg support from ai-doodles |
| Update | `lmnd-devx/deslop` | Replace body with ai-doodles' 5-step version |
| Update | `lmnd-devx/pr-resolve` | Add `AskUserQuestion` in Phase 3; auto-commit+push after applying |
| Add | `lmnd-devx/pr-fix-checks` | New skill from ai-doodles |
| Add | `lmnd-devx/pr-fix-yolo` | New skill from ai-doodles |
| Add | `lmnd-devx/pr-rebase` | New skill from ai-doodles |
| Bump version | `lmnd-devx/plugin.json` | `1.2.1` → `1.3.0` (3 new skills) |
