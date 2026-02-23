---
name: deslop
description: 'Remove AI-generated code slop from the current branch. Checks diff against main and cleans up unnecessary comments, defensive code, type hacks, and style inconsistencies.'
---

# Remove AI code slop

Check the diff against main, and remove all AI generated slop introduced in this branch.

This includes:
- Extra comments that a human wouldn't add or is inconsistent with the rest of the file
- Extra defensive checks or try/catch blocks that are abnormal for that area of the codebase (especially if called by trusted / validated codepaths)
- Casts to any to get around type issues
- Any other style that is inconsistent with the file

## Steps

1. Run `git diff main...HEAD` to get the full diff of changes introduced on this branch.
2. For each changed file, read the full file to understand the surrounding style and conventions.
3. Identify and remove slop:
   - Comments that explain obvious code, are overly verbose, or don't match the comment style in the rest of the file
   - Unnecessary try/catch blocks or null/undefined checks that the surrounding code doesn't use
   - `as any` casts or other type escapes that paper over real type issues (fix the types instead)
   - Overly defensive validation on internal/trusted inputs
   - Any formatting, naming, or structural patterns that are inconsistent with the rest of the file
4. Make edits to remove or fix each instance of slop.
5. Report at the end with only a 1-3 sentence summary of what you changed.
