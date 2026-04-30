# /spec setup

Configure the current project so that all Claude agents can write to `spec/` without permission prompts.

## What this does

Adds `Edit(spec/)` and `Write(spec/)` to the `permissions.allow` array in `.claude/settings.json` at the project root. This file is committed to the repo, so the permissions apply to every agent running in the project.

## Procedure

1. **Resolve the project root.** Use the current working directory.

2. **Read `.claude/settings.json` if it exists.** If the file is missing, treat the existing content as `{}`.

3. **Merge permissions.** Ensure `permissions.allow` contains both `"Edit(spec/)"` and `"Write(spec/)"`. Do not add duplicates if either entry is already present. Preserve all existing keys and array entries.

4. **Write the result** back to `.claude/settings.json`. Create `.claude/` if it does not exist.

5. **Report** which entries were added (or confirm they were already present). Do not propose a git commit — this is a one-time config change the user can commit as part of their normal workflow.
