# /spec setup — First-run onboarding

The canonical first command a new user runs. Configures `.claude/` permissions, orients the user (if fresh), and routes them to the right starting command (`/spec interview` for greenfield, `/spec adopt` for brownfield). **Router, not creator** — never writes `spec/state.yaml`.

## State machine

**Allowed from stages:** any, including `[no state.yaml]`.
**Transitions to:** unchanged. Setup never moves the stage machine — it delegates state creation to `/spec interview` or `/spec adopt`.
**Re-run behavior:** Idempotent. Permissions config is safe to re-apply. Orientation is suppressed when `spec/state.yaml` already exists.

Setup is **not required** — `/spec interview` and `/spec adopt` work without it. The permissions config is a convenience.

## Procedure

1. **Configure permissions.** Always runs.
   - Resolve project root from current working directory.
   - Read `.claude/settings.json` if it exists; treat as `{}` if missing.
   - Ensure `permissions.allow` contains both `"Edit(spec/)"` and `"Write(spec/)"`. Do not duplicate. Preserve all other keys and array entries.
   - Write the result back. Create `.claude/` if it doesn't exist.
   - Record whether either entry was added (vs already present) — used in the report later.

2. **Detect onboarding state.**
   - **`spec/state.yaml` exists** → already onboarded. Read it; capture `iteration`, `mode`, `stage`. Skip to step 6 (re-run report).
   - **`spec/` exists with artifacts but no `state.yaml`** → corrupted/manual. Skip to step 6 (corruption report).
   - **Otherwise** → fresh onboarding. Continue to step 3.

3. **Orient the user.** Print this blurb verbatim (8–12 lines, no more):

   ```
   /spec is a disciplined workflow for building a high-quality specification before writing code.

   The loop, one conversation per step:
     interview → seed → review → revise → check → implement → verify → close
       (Socratic intake → MECE AC tree → 3 skeptical personas → revisions →
        convergence test → per-phase orchestration → evidence-based audit → takeaway)

   Two starting modes:
     • greenfield — new project, no prior code or spec
     • adopted   — existing brownfield project (with code, optionally a rough spec doc)

   Conventions: you commit, the skill proposes messages. The state machine lives in spec/state.yaml.
   See /spec help for the full subcommand reference; README.md for design rationale.
   ```

4. **Detect greenfield vs brownfield signals.** Run cheap commands:
   - `git log --oneline 2>/dev/null | wc -l` → commit count.
   - `git ls-files 2>/dev/null | wc -l` → tracked file count.
   - `git ls-files 2>/dev/null | grep -E '\.(py|ts|tsx|js|jsx|go|rs|java|kt|rb|php|swift|c|cc|cpp|h|hpp|cs|scala|clj|ex|exs|ml|hs)$' | wc -l` → source file count by common extension.
   - Note presence of any of: `README.md`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `Gemfile`, `composer.json`.

5. **Suggest a starting path; ask the user.** Classification rule (suggest, do not auto-pick):
   - **Greenfield-leaning** if commit count ≤ 2 AND source file count ≤ 5 AND no substantive manifest file.
   - **Brownfield-leaning** if commit count ≥ 5 OR source file count ≥ 20 OR a substantive manifest file is present.
   - **Ambiguous** otherwise — present both options as roughly equal.

   Present a numbered summary of the signals you found, then:

   > **Recommended:** `/spec <interview-or-adopt>` — <one-line rationale tied to the signals>.
   >
   > **Alternatives:**
   > - `/spec <the-other-one>` — <one-line gloss>.
   > - Skip — stop here; you can pick later.
   >
   > Which?

   For ambiguous cases, present both equally without ranking. Wait for the user's answer.

6. **Report and hand off.**
   - **Fresh onboarding:** print which permission entries were added (or confirm they were already present), then echo back the user's chosen next command:
     - "Run `/spec interview` in a fresh conversation to start." (greenfield)
     - "Run `/spec adopt` in a fresh conversation to start." (brownfield)
     - "OK, stopping here. Run `/spec setup` again when you're ready, or `/spec help` for the full reference." (skipped)
   - **Already onboarded (`state.yaml` exists):** print exactly one line: `Already set up: iteration <n>, mode <mode>, stage <stage>. Run /spec (bare) for the next-step suggestion.` Plus the permissions delta line if anything changed. **Do not print the orientation blurb.**
   - **Corruption (artifacts but no `state.yaml`):** print: "Found `spec/` artifacts but no `state.yaml`. Run `/spec` (bare) — it has an auto-reconstruction path." Plus permissions delta line.

7. **Propose commit (only if `.claude/settings.json` actually changed).** If permission entries were added (not just confirmed), propose:

   ```
   git add .claude/settings.json
   git commit -m "spec: configure permissions for /spec workflow"
   ```

   If the file was already correct (no entries added), skip the commit proposal.

   **Do not** invoke `/spec interview` or `/spec adopt` automatically, even with consent. They own their own session/turn structure (clarity gate, path validation, focus-paths Q&A) and chaining them here would muddle that. Setup hands off; the user invokes the next command in a fresh conversation.

## Notes

- **Never writes `spec/state.yaml`.** That responsibility belongs to `/spec interview` and `/spec adopt`, which own their preconditions.
- **Never modifies anything outside `.claude/settings.json`.** No `CLAUDE.md` edits, no `.gitignore` changes, no hooks.
- **Setup is not gating.** Direct invocation of `/spec interview` or `/spec adopt` continues to work for users who know what they want.
