# gitops

Git workflow automation plugin for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| `commit` | Stages all changes and creates a conventional commit. Auto-invoked when Claude detects commit intent, or use `/commit` explicitly. |
| `pr-comments` | Fetches unresolved PR review comments and suggests code fixes. Use `/pr-comments <PR-URL>` or auto-invoked when asked to address review feedback. |

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

### pr-comments

The **pr-comments** skill automates addressing PR review feedback in two phases:

1. **Gather** (Haiku) — Fetches all unresolved review threads via the GitHub GraphQL API, collects comment conversations, and reads surrounding file context.
2. **Analyze** (Opus) — For each thread, summarizes the reviewer's concern, categorizes it (required change / suggestion / question), and produces an exact code fix or draft reply.

After analysis, it presents a summary table and lets you choose which fixes to apply. Use `/pr-comments <PR-URL>` or just `/pr-comments` on a branch that already tracks a PR.
