---
name: commit
description: 'Stages all changes and creates a commit with a conventional commit message. Use when asked to commit, save changes, or create a commit.'
---

Stage all current changes and commit them with a well-crafted message.

## Steps

1. Run `git status` (without `-uall`) to see all changed and untracked files.
2. Run `git diff` and `git diff --staged` to understand what changed.
3. Run `git log --oneline -5` to see the repo's recent commit style.
4. Stage all changes with `git add -A`.
5. Write a commit message following **Conventional Commits** (`feat:`, `fix:`, `refactor:`, `chore:`, `docs:`, `test:`, `build:`, `ci:`, `perf:`, `style:`).
6. Commit using a HEREDOC for the message.

## Commit Message Rules

- **Focus on value and functionality**, not implementation details. Describe *what the change enables or fixes for users/developers*, not *which files were touched or how the code changed*.
- Use imperative mood ("add search filtering", not "added search filtering").
- Keep the subject line under 72 characters.
- Add a body (separated by a blank line) only when the "why" isn't obvious from the subject.
- If there's a ticket or issue number available, include it.

### Good examples
- `feat: add batch retry for failed predictions`
- `fix: prevent duplicate charges on concurrent checkout`
- `refactor: simplify claim eligibility checks`

### Bad examples (too implementation-focused)
- `feat: add RetryQueue class and update PredictionService`
- `fix: add null check in PaymentController.process()`
- `refactor: extract method and rename variables in ClaimValidator`

## Additional arguments

If the user provides arguments with `/commit`, treat them as context or instructions for the commit message (e.g., a scope hint or override). Example: `/commit auth` might produce `feat(auth): ...`.
