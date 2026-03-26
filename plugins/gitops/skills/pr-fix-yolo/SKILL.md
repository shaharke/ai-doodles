---
name: pr-fix-yolo
description: 'Continuously fix PR review comments and failing CI checks on a loop until everything is green and resolved. Use when asked to "yolo fix the PR", "keep fixing until green", "fix everything on the PR", "auto-fix PR in a loop", or any request to autonomously and repeatedly fix a PR until it is fully passing and all review feedback is addressed.'
---

Autonomously fix a pull request in a loop — resolve review comments, fix failing CI checks, commit, push, and repeat until the PR is fully green and all feedback is addressed. No manual intervention needed (that's the yolo part).

## Phase 0: Input resolution and loop setup

### Resolve the PR

1. If `$ARGUMENTS` contains a GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), parse `owner`, `repo`, and `number` from it. Strip the URL from `$ARGUMENTS` — the remainder is treated as the interval token.
2. Otherwise, infer the PR from the current branch: `gh pr view --json number,url,headRepositoryOwner,baseRefName 2>/dev/null`. Parse `owner`, `repo`, and `number`.
3. If neither works, use `AskUserQuestion` to ask the user for the PR URL.

### Parse the interval

After stripping any PR URL, look at the remaining `$ARGUMENTS` for an interval token matching `^\d+[smhd]$` (e.g. `5m`, `10m`, `2h`). If none is found, default to `5m`.

### Set up the recurring loop

Use `CronCreate` to schedule a recurring job:

- **cron**: Convert the interval to a cron expression (e.g. `5m` → `*/5 * * * *`).
- **prompt**: The prompt below (substitute the resolved `owner`, `repo`, `number`, and the cron job ID once you have it — see step below):

```
You are an autonomous PR fixer running on a schedule. Do NOT use AskUserQuestion at any point — this is fully autonomous.

PR: {owner}/{repo}#{number}
Cron job ID to cancel when done: {cron_job_id}

Follow these steps exactly:

## Step 1: Check if BugBot is still running

Run: gh pr checks {number} --repo {owner}/{repo} --json name,state

Look for any check whose name contains "BugBot" (case-insensitive). If BugBot's state is PENDING, IN_PROGRESS, or QUEUED, report "BugBot is still running — skipping this iteration to wait for it to finish." and STOP. Do not proceed — BugBot may submit new review comments when it completes, and acting before that would miss them.

## Step 2: Check current PR status

Run two checks in parallel:

**Check A — CI status:**
Run: gh pr checks {number} --repo {owner}/{repo} --json name,state
Collect all checks. Categorize them:
- failing: state is FAILURE or ERROR
- pending: state is PENDING, IN_PROGRESS, or QUEUED
- passing: everything else

**Check B — Unresolved comments:**
Use gh api graphql to fetch review threads:
```graphql
query {
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          diffSide
          comments(first: 50) {
            nodes {
              databaseId
              body
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}
```
Filter to threads where isResolved is false and isOutdated is false.

## Step 3: Evaluate completion

If ALL of these are true:
- Zero failing checks
- Zero pending checks (including BugBot)
- Zero unresolved, non-outdated comment threads

Then the PR is fully green and resolved. Print a celebratory message, cancel the cron job using CronDelete with ID {cron_job_id}, and STOP.

If there are only pending checks (no failures and no unresolved comments), report "Checks are still running — waiting for next iteration." and STOP this iteration.

## Step 4: Fix failing CI checks

If there are failing checks, fix ALL of them (not just one):

For each failing check:
1. Get the run ID from: gh pr checks {number} --repo {owner}/{repo} --json name,state,detailsUrl — extract the numeric run ID from the detailsUrl.
2. Fetch logs: gh run view {runId} --repo {owner}/{repo} --log-failed (truncate to last 2000 lines if huge).
3. Read the relevant source files based on error messages, stack traces, and file paths in the logs. If errors don't point to specific files, check which files the PR changed: gh pr diff {number} --repo {owner}/{repo} --name-only.
4. Determine the root cause and apply a minimal fix using the Edit tool.

Do NOT ask the user — just apply the fix directly. This is yolo mode.

## Step 5: Fix unresolved review comments

For each unresolved, non-outdated thread:
1. Read the comment conversation to understand what the reviewer wants.
2. Read ~30 lines of context around the commented line in the source file.
3. **Classify the fix as minor or major** before doing anything:

### Minor fixes — apply automatically
A fix is **minor** if it meets ALL of these:
- Touches 1-2 files
- Changes fewer than ~20 lines total
- Is a localized, mechanical change that doesn't affect control flow or public interfaces
- Examples: renaming a variable, fixing a typo, adding a null check, adjusting formatting, updating a log message, adding a missing import, tweaking a string literal, improving a comment, adding input validation to a single function

### Major fixes — pause and ask
A fix is **major** if ANY of these are true:
- Touches 3+ files
- Changes more than ~20 lines
- Alters public API signatures, function contracts, or type definitions
- Changes architectural patterns (e.g., switching from sync to async, restructuring error handling)
- Modifies database schemas, migrations, or data models
- Changes business logic or behavioral semantics (not just how something is written, but what it does)
- You're not confident the fix is correct (low confidence)

If a fix is **major**: do NOT apply it. Instead, reply to the review thread with your proposed fix as a code suggestion and a note that it needs human review. Then cancel the cron job using CronDelete with ID {cron_job_id}, and report to the user: which comment triggered the pause, why you classified it as major, and what you'd propose. STOP the iteration — the user will review and decide how to proceed.

4. For minor fixes and questions, proceed automatically:
   - If it's a code change request → apply the fix with Edit tool.
   - If it's a question → draft a reply.
   - If it's a suggestion you can apply → apply it.
5. Reply to the thread: POST to /repos/{owner}/{repo}/pulls/{number}/comments/{last_comment_id}/replies with a brief description of what you did.
6. If you applied a fix or answered a question, resolve the thread using the resolveReviewThread GraphQL mutation.

## Step 6: Commit and push

If any files were modified in steps 4 or 5:
1. Run git add -A
2. Create a conventional commit message describing the fixes (use a HEREDOC).
3. Run git push.

If no files were modified (e.g., only replied to questions), that's fine — just skip the commit.

Done with this iteration. The cron job will trigger the next one.
```

- **recurring**: `true`

After creating the cron job, note its ID — you'll need to substitute `{cron_job_id}` in the prompt. Since CronCreate returns the ID, create the job first with a placeholder, then update if needed. Alternatively, include instruction in the prompt to list cron jobs and find its own ID if the substitution isn't possible.

### Run the first iteration immediately

After setting up the cron, execute the loop body prompt immediately — don't wait for the first cron fire. Invoke it directly (not via Skill tool — just follow the prompt instructions yourself).

### Confirm to the user

After the first iteration completes, briefly tell the user:
- What PR you're watching
- The loop interval
- The cron job ID (so they can cancel with `CronDelete` if needed)
- What you found and fixed in the first iteration (if anything)

## Design principles

This skill is "yolo" for minor fixes — it applies them directly, resolves threads, commits, and pushes without asking. But it's not reckless: when a review comment requires a major change (3+ files, architectural shifts, API changes, or anything you're unsure about), the loop pauses and surfaces the decision to the user. The goal is to automatically handle the 80% of feedback that's straightforward, while protecting against autonomously making changes that could make things worse.

The BugBot gate in Step 1 is critical — BugBot is an automated reviewer that takes time to analyze a PR. If we fix comments and declare victory before BugBot finishes, we'll miss its feedback entirely. Always wait for BugBot to complete before evaluating whether all comments are resolved.
