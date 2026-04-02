---
name: pr-align
description: 'Align a PR title and description to match the commits on the branch. Use when asked to update a PR description, sync PR with commits, fix PR title, rewrite PR summary, or when the PR description is stale or out of date after changes were pushed.'
---

Update an existing pull request's title and description so they accurately reflect the commits on the branch. The goal is to give reviewers a clear, honest picture of what changed and — more importantly — why.

## Phase 0: Input resolution

1. If `$ARGUMENTS` contains a GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), parse `owner`, `repo`, and `number` from it.
2. Otherwise, infer the PR from the current branch by running `gh pr view --json number,url,headRefName,baseRefName,title,body 2>/dev/null`. If this succeeds, extract the PR details.
3. If no PR was found, use `AskUserQuestion` to ask: "Which PR would you like to align?" with a free-text input.

## Phase 1: Gather context

Run these commands to understand what the branch actually contains:

1. `gh pr view {number} --repo {owner}/{repo} --json title,body,baseRefName,headRefName` — get the current PR title, description, and branch names.
2. `git log --format='%h %s%n%b---' origin/{baseRefName}..HEAD` — get all commit messages (subject + body) on the branch since it diverged from the base.
3. `git diff --stat origin/{baseRefName}..HEAD` — get a high-level summary of files changed, insertions, and deletions.
4. `git diff origin/{baseRefName}..HEAD` — read the actual diff to understand the substance of the changes. For large diffs (>500 lines), focus on the most significant files based on the diffstat.

## Phase 2: Compose the new title and description

### Title

Write a conventional commit title (`feat:`, `fix:`, `refactor:`, `chore:`, etc.) that captures the overall intent of the branch. If the commits span multiple types, pick the dominant one. Keep it under 72 characters.

The title should describe what the change **enables or fixes**, not how it was implemented.

**Good:** `feat: add retry mechanism for failed webhook deliveries`
**Bad:** `feat: add RetryQueue class and update WebhookService`

### Description

The description serves two audiences: the reviewer who needs to understand the PR right now, and the future developer who will read this in `git log` six months from now. Write for both.

Structure:

```
## Summary

One to three sentences explaining **why** this change exists — what problem it solves or what it enables. This is the most important part. A reviewer who reads only this section should understand the motivation.

## Approach

The key technical decisions and trade-offs. Not a list of every file touched, but the important choices:
- Why this approach over alternatives
- Any non-obvious design decisions
- Things the reviewer should pay attention to

## Changes

A concise list of the concrete changes, grouped logically. Each item should be a meaningful unit of work, not a file name.
```

### Writing principles

- **Lead with why, not what.** The diff already shows what changed — the description should explain the reasoning behind it.
- **Be specific about decisions.** "Used a queue instead of immediate retry because the downstream service rate-limits at 100 req/s" is useful. "Improved the retry logic" is not.
- **Skip the obvious.** If a commit message says `fix typo in README`, don't inflate it in the PR description. Focus on substance.
- **Don't fabricate motivation.** If the commits don't reveal a clear "why", describe what the changes do factually. Don't invent a narrative.
- **Match the scope.** A single-commit bugfix gets a one-liner description. A multi-week feature branch gets proper sections.

## Phase 3: Preview and confirm

Present the proposed title and description to the user in a clear format:

```
Proposed PR title:
  feat: add retry mechanism for failed webhook deliveries

Proposed PR description:
  ## Summary
  ...

  ## Approach
  ...

  ## Changes
  ...
```

Then show the current title for comparison:
```
Current PR title:
  WIP: webhook stuff
```

Use `AskUserQuestion` to ask: "Apply these changes to the PR?" with options:
- **Apply** — update the PR title and description
- **Edit title only** — apply the description but let me revise the title
- **Revise** — let me give you feedback to adjust the description

## Phase 4: Apply

1. Write the description to a temporary file to avoid shell escaping issues.
2. Run `gh pr edit {number} --repo {owner}/{repo} --title "{title}" --body-file /tmp/pr-description.md`.
3. Clean up the temporary file.
4. Print the PR URL as confirmation.

## Error handling

- **No PR found**: Ask the user for the PR URL.
- **No commits ahead of base**: Print "This branch has no commits ahead of `{baseRefName}` — nothing to align." and stop.
- **PR is already merged or closed**: Print a warning and stop.
- **gh CLI not authenticated**: Suggest running `gh auth status` to diagnose.
