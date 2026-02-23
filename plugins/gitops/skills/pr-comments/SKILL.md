---
name: pr-comments
description: 'Fetch unresolved PR review comments and suggest code fixes. Use when asked to address PR feedback, handle review comments, or fix PR discussions.'
---

Fetch all unresolved review comments on a pull request, analyze each one, and suggest concrete code fixes.

## Phase 0: Input resolution

1. If `$ARGUMENTS` contains a GitHub PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), parse `owner`, `repo`, and `number` from it.
2. If no arguments were provided, run `gh pr view --json url,number` to resolve the PR from the current branch.
3. If neither works, use `AskUserQuestion` to ask the user for the PR URL.

## Phase 1: Gather unresolved comments (Haiku subagent)

Dispatch a **single** `Task` subagent with `model: "haiku"` and `subagent_type: "general-purpose"`. Give it the following instructions:

> Use `gh api graphql` to fetch all review threads for the PR. Use the query below (substitute owner, repo, number):
>
> ```graphql
> query {
>   repository(owner: "{owner}", name: "{repo}") {
>     pullRequest(number: {number}) {
>       reviewThreads(first: 100) {
>         nodes {
>           isResolved
>           isOutdated
>           path
>           line
>           diffSide
>           startLine
>           comments(first: 50) {
>             nodes {
>               body
>               author { login }
>               createdAt
>             }
>           }
>         }
>       }
>     }
>   }
> }
> ```
>
> Filter to threads where `isResolved == false`.
>
> For each unresolved thread, collect:
> - `path` (file path)
> - `line` (line number in the diff)
> - `startLine` (if the comment spans a range)
> - All comment bodies and authors (the full conversation)
> - Whether the thread is outdated
>
> For each **unique file path** that has unresolved comments, use the `Read` tool to read approximately 30 lines of context around each commented line (15 lines above, 15 below). If the file does not exist locally, note that in the output.
>
> Return a JSON array of thread objects, each containing:
> ```json
> {
>   "path": "src/foo.ts",
>   "line": 42,
>   "startLine": 40,
>   "isOutdated": false,
>   "comments": [
>     { "author": "reviewer", "body": "This should handle the null case" }
>   ],
>   "fileContext": "...surrounding lines of code..."
> }
> ```

If the subagent returns **zero** unresolved threads, print "No unresolved review comments found — nothing to do." and stop.

## Phase 2: Analyze and suggest fixes (Opus subagents)

For each unresolved thread (or in small batches), dispatch `Task` subagents with `model: "opus"` and `subagent_type: "general-purpose"`. These can run **in parallel** since threads are independent.

Each subagent receives one thread object and these instructions:

> You are analyzing a PR review comment. You receive:
> - The full comment conversation
> - The file path and line number
> - Surrounding code context
>
> Produce the following:
>
> 1. **Summary**: One sentence describing the reviewer's concern.
> 2. **Category**: One of:
>    - `required change` — the reviewer is requesting a specific code change
>    - `suggestion` — the reviewer is proposing an improvement but not blocking
>    - `question` — the reviewer is asking for clarification
> 3. **Suggested fix**:
>    - If a code change is warranted, provide exact `old_string` / `new_string` blocks that can be used with the `Edit` tool. Make the fix minimal — only change what the reviewer asked for.
>    - If the comment is a question, draft a short reply comment that addresses it.
> 4. **Confidence**: `high` / `medium` / `low` — how confident you are that your fix correctly addresses the feedback.
>
> Return your analysis as structured text with clear section headers.

## Phase 3: Present results and apply fixes

1. Print a numbered summary table of all comments:

   ```
   #  | File              | Line | Category        | Summary                          | Confidence
   ---|-------------------|------|-----------------|----------------------------------|----------
   1  | src/foo.ts        | 42   | required change | Handle null case in parser       | high
   2  | src/bar.ts        | 18   | suggestion      | Extract helper for readability   | medium
   3  | src/baz.ts        | 7    | question        | Why not use the shared utility?  | high
   ```

2. For each thread, show the suggested fix (code diff or draft reply).

3. Use `AskUserQuestion` to ask which fixes to apply. Options:
   - **Apply all** — apply every code fix
   - **Select specific** — let the user pick by number
   - **Skip** — don't apply any fixes

4. Apply the approved fixes using the `Edit` tool.

5. After applying, suggest the user run `/commit` to commit the changes.

## Error handling

- **Invalid or inaccessible PR URL**: Print a clear error and ask the user to verify the URL and their `gh` auth status.
- **No unresolved threads**: Print a success message and exit cleanly (Phase 1 early exit).
- **File not found locally**: If a file referenced in a comment doesn't exist in the working tree, warn the user. Suggest checking out the PR branch with `gh pr checkout`.
- **Branch mismatch**: If the current branch doesn't match the PR head branch, print a warning so the user knows fixes may not align with the diff.
- **GraphQL API errors**: Print the raw error and suggest checking `gh auth status`.
