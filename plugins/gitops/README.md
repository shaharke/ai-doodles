# gitops

Git workflow automation plugin for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| `squash` | Squashes all commits on the current branch into a single conventional commit. Use `/squash` or `/squash <context>`. |
| `pr-rebase` | Rebases the current PR branch onto its base branch and force-pushes. Use `/pr-rebase` or `/pr-rebase <PR-URL>`. |
| `pr-fix-yolo` | Continuously fixes PR review comments and failing CI checks in a loop until everything is green. Use `/pr-fix-yolo`. |

## Install

```
/plugin marketplace add shaharke/ai-doodles
/plugin install gitops
```

### squash

The **squash** skill collapses all commits on the current branch into a single commit:

1. Finds the fork point from the base branch (`main`/`master`)
2. Shows the commits that will be squashed
3. Composes a single conventional commit message summarizing the overall change
4. Uses `git reset --soft` + `git commit` to squash

Pass arguments to hint at the commit message: `/squash auth feature`. If the branch has been pushed, it warns that a force-push will be needed.

### pr-rebase

The **pr-rebase** skill rebases a PR branch onto the latest base branch:

1. Resolves the PR from a URL argument or the current branch
2. Fetches the latest base branch from the remote
3. Runs `git rebase origin/<base-branch>`
4. If conflicts arise, shows the conflicting files and asks how to proceed
5. If successful, force-pushes with `--force-with-lease`

Use `/pr-rebase <PR-URL>` or just `/pr-rebase` on a branch that already tracks a PR.

### pr-fix-yolo

The **pr-fix-yolo** skill autonomously fixes a PR in a loop:

1. Resolves review comments and applies code fixes
2. Fixes failing CI checks
3. Commits, pushes, and repeats until the PR is fully green and all feedback is addressed

Use `/pr-fix-yolo` or `/pr-fix-yolo <PR-URL>`.
