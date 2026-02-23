# gitops

Git workflow automation plugin for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| `commit` | Stages all changes and creates a conventional commit. Auto-invoked when Claude detects commit intent, or use `/commit` explicitly. |

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
