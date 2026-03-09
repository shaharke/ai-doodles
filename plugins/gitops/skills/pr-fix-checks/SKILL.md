---
name: pr-fix-checks
description: 'Inspect failing CI checks on a GitHub PR, fetch logs, and suggest fixes. Use when asked to fix checks, investigate CI failures, debug pipeline errors, or resolve failing builds.'
---

Fetch failing CI checks on a pull request, retrieve failure logs, analyze root causes, and suggest concrete code fixes.

## Phase 0: Input resolution

1. If `$ARGUMENTS` contains a GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), parse `owner`, `repo`, and `number` from it directly.
2. Otherwise, try to infer the PR from the current branch by running `gh pr view --json number,url,headRepositoryOwner,baseRepository 2>/dev/null`. If this succeeds, parse `owner`, `repo`, and `number` from the returned JSON.
3. If no PR was found, use `AskUserQuestion` to ask the user for the PR URL. Prompt: "Which PR would you like to fix checks for?" with a free-text input.
4. Parse `owner`, `repo`, and `number` from the provided URL.

## Phase 1: Fetch failing checks (Haiku)

Dispatch a single `Task` subagent with `model: "haiku"` and `subagent_type: "general-purpose"`. Give it the following instructions:

> Run `gh pr checks {number} --repo {owner}/{repo} --json name,workflow,state,detailsUrl` to fetch all check/build statuses for the PR.
>
> Filter to only checks where `state` is `FAILURE` or `ERROR`.
>
> For each failing check, collect:
> - `name` — the check name
> - `workflow` — the workflow name
> - `state` — the check state
> - `runUrl` — the `detailsUrl` value
> - `runId` — extracted from the URL (the numeric ID in the path, e.g. from `https://github.com/{owner}/{repo}/actions/runs/12345` extract `12345`)
>
> Return the results as a JSON array. If there are zero failing checks, check whether any checks have `state` equal to `PENDING` or `IN_PROGRESS` or `QUEUED`. Report that distinction clearly.

**Early exits:**
- If zero failing checks and none pending → print "All checks are passing." and stop.
- If zero failing checks but some are pending/in-progress → print "Checks are still running — wait for them to complete and try again." and stop.

## Phase 1.5: User selection (if multiple failures)

- **One failure** → auto-select it.
- **Multiple failures** → print a numbered list of failing checks (name + workflow) and use `AskUserQuestion` to let the user pick one by number, or type "All" to investigate all of them.

## Phase 2a: Fetch logs (Haiku)

For the selected check, dispatch a `Task` subagent with `model: "haiku"` and `subagent_type: "general-purpose"`. Give it the following instructions:

> Fetch the failure logs for the check run. Use the following approach:
>
> 1. Run `gh run view {runId} --repo {owner}/{repo} --json jobs` to get the list of jobs and identify which jobs failed.
> 2. For each failed job, run `gh run view {runId} --repo {owner}/{repo} --log-failed` to fetch the failure logs.
> 3. If logs exceed 5000 lines, truncate to the **last 2000 lines** (failures are typically at the end).
>
> Return:
> - `checkName` — the name of the check
> - `failedSteps` — a list of objects with `jobName` and `stepName` for each failed step
> - `logs` — the log content (truncated if necessary, noting truncation occurred)

**Error handling:** If log fetching fails, return the `runUrl` so the user can check manually.

## Phase 2b: Analyze and suggest fix (Opus)

Dispatch a `Task` subagent with `model: "opus"` and `subagent_type: "general-purpose"`. Provide it with: the failure logs, failed step metadata, PR number, owner/repo. Give it the following instructions:

> You are analyzing CI failure logs for PR #{number} in {owner}/{repo}. You have access to the full codebase.
>
> 1. **Read the logs carefully** to identify exact error(s) — look for file paths, line numbers, stack traces, and error messages.
> 2. **Pinpoint relevant source files** using the error details directly (e.g., file paths in stack traces, module names in import errors, test file references). Read those files for context.
> 3. If the error doesn't point to specific files (e.g., vague timeout, generic failure), fall back to checking which files were changed in the PR by running `gh pr diff {number} --repo {owner}/{repo} --name-only` and inspect those files.
> 4. **Determine the root cause.**
> 5. **Suggest a fix** with exact `old_string`/`new_string` edit blocks that can be used with the `Edit` tool. Make fixes minimal — only change what's needed to resolve the failure.
> 6. **Rate your confidence**: `high` / `medium` / `low`.
> 7. **Categorize the failure**: `code fix needed` | `configuration issue` | `infrastructure/flaky` | `unknown`.
>
> Structure your output as:
>
> ### Problem Summary
> One-sentence summary of the failure.
>
> ### Error Details
> Key error messages, file paths, and line numbers from the logs.
>
> ### Root Cause
> Explanation of why the failure occurred.
>
> ### Suggested Fix
> Exact `Edit` tool blocks (`file_path`, `old_string`, `new_string`) for each change needed. If no code fix is applicable (e.g., infrastructure/flaky test), say so and suggest re-running the check instead.
>
> ### Confidence
> `high` / `medium` / `low` with brief justification.
>
> ### Alternative Approaches
> Only include this section if confidence is not `high`. List other possible fixes or investigation steps.

## Phase 3: Present results and apply

1. Print the analysis report from Phase 2b.

2. Use `AskUserQuestion` to ask the user what to do next. Options:
   - **Apply fix** — apply the suggested code edits
   - **Show full logs** — print the raw failure logs from Phase 2a
   - **Skip** — don't apply any fixes

3. If the user chose "Apply fix", apply the edits using the `Edit` tool.

4. After applying, suggest the user run `/commit` to commit the changes.

5. If the user chose "All" in Phase 1.5, loop to the next failing check and repeat from Phase 2a.

## Error handling

- **Invalid or inaccessible PR URL**: Print a clear error and ask the user to verify the URL and their `gh auth status`.
- **No failing checks**: Print "All checks are passing." and exit cleanly (Phase 1 early exit).
- **Pending checks**: Print "Checks are still running — wait for them to complete and try again." and exit cleanly.
- **Log fetch failure**: Print the run URL and suggest the user check it manually in the browser.
- **Logs too large**: Truncate to last 2000 lines and note that full logs are available at the run URL.
- **Branch mismatch**: If the current branch doesn't match the PR head branch, print a warning so the user knows fixes may not align with the working tree.
- **Non-code failure (infrastructure/flaky)**: Opus reports "no code fix applicable" and suggests re-running the check via `gh run rerun {runId} --repo {owner}/{repo}`.
