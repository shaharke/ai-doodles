# gitops

Git workflow automation plugin for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| `commit` | Stages all changes and creates a conventional commit. Auto-invoked when Claude detects commit intent, or use `/commit` explicitly. |
| `pr-resolve` | Resolves unresolved PR review comments by suggesting and applying code fixes. Use `/pr-resolve` or `/pr-resolve <PR-URL>`. |
| `squash` | Squashes all commits on the current branch into a single conventional commit. Use `/squash` or `/squash <context>`. |

## Install

```
/plugin marketplace add shaharke/ai-doodles
/plugin install gitops
```

## What it does

The **commit** skill automates the entire commit workflow:

1. Inspects staged and unstaged changes
2. Reviews recent commit history for style consistency
3. Stages all changes
4. Writes a conventional commit message focused on value, not implementation details
5. Commits using a HEREDOC for proper formatting

Pass arguments to hint at scope: `/commit auth` produces `feat(auth): ...`.

### pr-resolve

The **pr-resolve** skill automates addressing PR review feedback in two phases:

1. **Gather** (Haiku) — Fetches all unresolved review threads via the GitHub GraphQL API, collects comment conversations, and reads surrounding file context.
2. **Analyze** (Opus) — For each thread, summarizes the reviewer's concern, categorizes it (required change / suggestion / question), and produces an exact code fix or draft reply.

After analysis, it presents a summary table and lets you choose which fixes to apply. Use `/pr-resolve <PR-URL>` or just `/pr-resolve` on a branch that already tracks a PR.

### squash

The **squash** skill collapses all commits on the current branch into a single commit:

1. Finds the fork point from the base branch (`main`/`master`)
2. Shows the commits that will be squashed
3. Composes a single conventional commit message summarizing the overall change
4. Uses `git reset --soft` + `git commit` to squash

Pass arguments to hint at the commit message: `/squash auth feature`. If the branch has been pushed, it warns that a force-push will be needed.
