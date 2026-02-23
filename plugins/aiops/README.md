# aiops

AI hygiene tools for Claude Code — clean up slop and keep AI-generated code human-grade.

## Skills

| Skill | Description |
|-------|-------------|
| `deslop` | Diffs your branch against main and removes AI-generated slop: unnecessary comments, defensive code, type hacks, and style inconsistencies. |

## Install

```
/plugin marketplace add shaharke/ai-doodles
/plugin install aiops
```

## What it does

The **deslop** skill reviews every change on your branch and strips out the telltale signs of AI-generated code:

1. Diffs the current branch against main
2. Reads each changed file for surrounding style context
3. Removes verbose comments, unnecessary try/catch blocks, `as any` casts, and overly defensive checks
4. Reports a short summary of what it cleaned up
