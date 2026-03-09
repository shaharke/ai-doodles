---
name: pr-rebase
description: 'Rebase the current PR branch onto its base branch to bring it up to date. Use when asked to rebase a PR, update a branch, or sync with main.'
---

Rebase the current PR's branch onto the latest base branch, then force-push if successful.

## Phase 0: Input resolution

1. If `$ARGUMENTS` contains a GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), parse `owner`, `repo`, and `number` from it.
2. Otherwise, try to infer the PR from the current branch by running `gh pr view --json number,url,headRefName,baseRefName,headRepository 2>/dev/null`. If this succeeds, extract the PR details.
3. If no PR was found, use `AskUserQuestion` to ask: "Which PR would you like to rebase?" with a free-text input.

## Phase 1: Gather PR details

1. Run `gh pr view {number} --repo {owner}/{repo} --json headRefName,baseRefName` to get the head branch and base branch names.
2. Store `headBranch` and `baseBranch`.

## Phase 2: Prepare local branch

1. Run `git rev-parse --abbrev-ref HEAD` to get the current branch.
2. If the current branch is not `headBranch`, run `git checkout {headBranch}`.
3. Check for uncommitted changes with `git status --porcelain`. If there are any, abort with: "You have uncommitted changes. Commit or stash them before rebasing."
4. Run `git fetch origin {baseBranch}` to get the latest base branch.

## Phase 3: Rebase

1. Run `git rebase origin/{baseBranch}`.
2. Check the exit code:

**If rebase fails (conflicts):**
- Run `git diff --name-only --diff-filter=U` to list conflicting files.
- Print the list of conflicting files.
- Use `AskUserQuestion` to ask: "Rebase hit conflicts in the files listed above. How would you like to proceed?" with options:
  - **Abort rebase** — run `git rebase --abort` and stop
  - **Help me resolve** — the user will provide guidance on resolving conflicts
- Follow the user's instructions.

**If rebase succeeds:**
- Print: "Rebase onto `origin/{baseBranch}` succeeded."
- Run `git push --force-with-lease` to update the remote branch.
- Print: "Force-pushed `{headBranch}` to remote."

## Error handling

- **No PR found**: Ask the user for the PR URL.
- **Uncommitted changes**: Refuse and ask to commit or stash first.
- **Checkout fails**: Print the error and stop.
- **Fetch fails**: Print the error and suggest checking network/auth (`gh auth status`).
- **Force-push fails**: Print the error. Common cause: another person pushed to the branch after the rebase started — suggest re-fetching and retrying.
