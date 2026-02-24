---
name: squash
description: 'Squash all commits on the current branch into a single commit. Use when asked to squash, collapse, flatten, or combine commits.'
---

Squash all commits on the current branch (relative to the base branch) into a single, well-crafted commit.

## Steps

1. **Detect base branch**: Run `git rev-parse --abbrev-ref HEAD` to get the current branch. If it is `main` or `master`, abort with: "You're on the main branch — switch to a feature branch before squashing."

2. **Find the merge base**: Determine the base branch by checking which of `main` or `master` exists (`git rev-parse --verify main 2>/dev/null || git rev-parse --verify master 2>/dev/null`). Then run `git merge-base <base-branch> HEAD` to find the fork point.

3. **List commits to squash**: Run `git log --oneline <merge-base>..HEAD` to show all commits that will be squashed. If there are 0 or 1 commits, print "Nothing to squash — branch has at most one commit ahead of `<base-branch>`." and stop.

4. **Check for uncommitted changes**: Run `git status --porcelain`. If there are uncommitted changes, abort with: "You have uncommitted changes. Commit or stash them before squashing."

5. **Show the user what will happen**: Print the list of commits that will be collapsed, e.g.:

   ```
   Squashing N commits into one:
     abc1234 feat: add user search
     def5678 fix: handle empty query
     ghi9012 refactor: extract search helper
   ```

6. **Compose the squashed commit message**:
   - Run `git diff <merge-base>..HEAD` to understand the full changeset.
   - Read the individual commit messages for context.
   - Write a single conventional commit message that summarizes the overall change, following the same rules as the `commit` skill (imperative mood, value-focused, under 72 chars, body only if needed).
   - If `$ARGUMENTS` is provided, treat it as context or instructions for the commit message.

7. **Perform the squash**: Run the following commands sequentially:
   ```
   git reset --soft <merge-base>
   git commit -m "<message>"
   ```

8. **Verify**: Run `git log --oneline -3` and `git status` to confirm the squash succeeded. Print the new single commit.

9. **Handle remote**: Check if the branch has a remote tracking branch (`git rev-parse --abbrev-ref @{upstream} 2>/dev/null`). If it does, let the user know and offer to force-push with lease:
   - Print: "This branch has been pushed to the remote. Run `git push --force-with-lease` to update it."
   - Use `AskUserQuestion` to ask whether to push now (yes/no).
   - If yes, run `git push --force-with-lease`.
   - If no, remind them they'll need to force-push before opening or updating a PR.

## Important

- This skill uses `git reset --soft` which rewrites history. The user's previous commits on this branch are replaced by a single commit. This is safe for local/unpushed branches.
- Always use `--force-with-lease` (never `--force`) to protect against overwriting others' work on the remote.

## Error handling

- **On main/master**: Refuse to squash and explain why.
- **Uncommitted changes**: Refuse and ask the user to commit or stash first.
- **Single or no commits**: Exit cleanly with an informational message.
- **Detached HEAD**: Refuse and ask the user to check out a branch.
