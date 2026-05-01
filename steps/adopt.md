# /spec adopt — Bootstrap spec/ for an existing project

Initialize `spec/` for a project that already has code and (optionally) a rough specification document. Lands in stage `interviewing` with a session file pre-populated by an Explore sub-agent, so `/spec interview` resumes from synthesized context rather than a blank page.

## State machine

**Allowed from stages:** `[no state.yaml]` **only**. Pure bootstrap command.
**Transitions to:** `interviewing` (mode `adopted`).
**Re-run behavior:** **Refused** if `spec/state.yaml` exists. Tell the user: "Adopt is a bootstrap command. Found existing `spec/state.yaml` (stage: <stage>). Use `/spec` to see the current state, or `/spec interview` if you're ready to start the next iteration."

**Not** surfaced by the `/spec` bare auto-reconstruction path — adoption is a pre-first-use step, distinct from recovering a partially-built `spec/`.

## Arguments

- `--iteration N` *(optional, default `1`)* — starting iteration number. Use when adoption represents work that picks up from a known prior version (e.g., "this is really v3 of the system"). Must be a positive integer.
  - Accepted forms: `/spec adopt --iteration 3`, `/spec adopt --iteration=3`.
  - If the user passes a bare positional integer (`/spec adopt 3`), treat it as `--iteration 3`.

## Protocol

1. **Precondition guard.** Check for `spec/state.yaml`. If present, refuse per above and stop.

2. **Parse `--iteration`** if provided; otherwise default to `1`. Reject non-positive integers with a one-line error and stop.

3. **Ask two questions, in this order, one at a time:**

   **Q1 — Rough spec source:**
   > "Path to an existing rough spec / design doc (Markdown, PDF, etc.), or type `none` if the project's specification only lives in scattered code comments and READMEs."

   Accept either a file path (validate it exists — if not, re-ask once) or the literal `none`.

   **Q2 — Explore focus paths:**
   > "By default the Explore sub-agent will scan the entire project directory. Optionally name specific files or directories to focus on (space-separated), or press enter to accept the default."

   Accept either a whitespace-separated list of paths (validate each exists) or empty (meaning: whole project).

4. **Create `spec/` and `spec/archive/`** if they don't exist.

5. **Determine interview session filename.** Iteration `v<NNN>` (zero-padded to 3 digits) + current timestamp: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-interview.md`.

6. **Write initial session file header:**

   ```markdown
   # <Project title inferred from cwd basename — confirm with user if ambiguous> — Interview v<n> (adopted)

   Date: <YYYY-MM-DD>
   Mode: adopted (iteration <n>)
   Rough spec source: <path | "none">
   Explore focus: <comma-separated paths | "entire project directory">

   ---

   ## Adopted context (pre-interview)

   _Synthesized by Explore sub-agent at adoption time. The standard Socratic interview
   will ratify, correct, or extend this content before `/spec seed` runs._
   ```

7. **Triage and ingest.** Before spawning, assess project scope:
   - Run `find . -not -path '*/\.*' -not -path '*/node_modules/*' -type f | wc -l` (file count).
   - Run `find . -mindepth 1 -maxdepth 1 -not -name '.*' -type d | wc -l` (top-level directory count).

   - **Direct path — handle in-session** if **all** hold: rough spec source is `none` (no external doc to parse), AND file count ≤ 20, AND the user named specific focus paths (not "entire project directory"). Read those paths directly with Read and Bash. Write the ingestion sections yourself directly into the session file using the structure shown in the sub-agent prompt below. Proceed to step 8.
   - **Sub-agent path** — if any condition above fails (rough spec doc provided, project is large, or scope is entire codebase): use the Agent tool with `subagent_type: "Explore"`, `model: "sonnet"`, thoroughness `"very thorough"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7** (since the sub-agent appends to a file). Prompt:

   > You are ingesting an existing project for adoption into a spec-development workflow. Produce synthesized context that a Socratic interview will later ratify.
   >
   > **Sources:**
   > - Rough spec document: `<path from Q1, or "none">`. If provided, read it in full.
   > - Code scope: `<focus paths from Q2, or "entire project directory">`. Scan for architecture, entry points, public surface, data model, external integrations, obvious invariants, and anything the rough spec mentions.
   >
   > **Append to the file at `spec/archive/<session-filename>`:**
   >
   > ```markdown
   > ### Shipped reality `[from-code]`
   >
   > - <factual bullet with file:line ref>
   > - ...
   >
   > ### Rough-spec claims `[from-rough-spec]`    (omit this section if rough spec source was "none")
   >
   > - <claim paraphrased, tagged as a claim not a verified fact>
   > - ...
   >
   > ### Ambiguities and tensions
   >
   > - <thing the rough spec says that the code seems to contradict, or code behavior the rough spec doesn't mention>
   > - ...
   >
   > ### Suggested axes for interview to confirm
   >
   > - Motivation for this iteration (the rough spec may or may not state one explicitly)
   > - Invariants (what must not break)
   > - Change delta (is this iteration meant to extend, replace, or ratify existing behavior?)
   > - Scope boundary
   > - Success criteria
   > ```
   >
   > Keep `[from-code]` factual and file-referenced — no interpretation. Keep `[from-rough-spec]` as *claims*, explicitly not verified. Surface every tension you notice in the Ambiguities section; these are the highest-value questions for the human interview.
   >
   > Do **not** draft acceptance criteria. Do **not** write `spec/spec.md`. The seed stage happens later.
   >
   > Attempt to append to the session file (use Edit or Write on the absolute path). If the tool is denied, retry once; if still denied, include the full intended appended content verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can append it. Never report success without either having appended the content or having dumped it.
   >
   > Report back: path to the session file, a 3-line summary of what you found (# of shipped-reality bullets, # of rough-spec claims, # of ambiguities), and (if Write was denied) the full content dump.

   After the sub-agent returns, verify the session file contains the appended sections. If not, extract the dumped content and append it directly in the parent session.

8. **Write `spec/state.yaml`:**

   ```yaml
   schema_version: 1
   iteration: <n>
   mode: adopted
   stage: interviewing
   started_at: <ISO timestamp>
   last_command: /spec adopt
   last_command_at: <ISO timestamp>
   spec_sha: null
   latest_interview: <session filename>
   latest_review_stamp: null
   latest_verify: null
   ```

9. **Present the sub-agent's summary** to the user and tell them:

   > Adoption complete. Stage is `interviewing` (mode: adopted, iteration <n>). The session file at `spec/archive/<session-filename>` has been pre-populated with synthesized context.
   >
   > **Next:** start a new conversation and run `/spec interview` to ratify this context, answer the outstanding ambiguities, and pass the clarity gate before `/spec seed`.

10. **Propose commit:**

    ```
    git add spec/
    git commit -m "spec: adopt existing project at iteration <n>"
    ```

## Notes for downstream steps

- `/spec interview` (see `steps/interview.md`) detects `mode: adopted` and resumes the pre-populated session file rather than starting from "what are you trying to build?"
- `/spec seed` (see `steps/seed.md`) uses `templates/spec-iteration.md` for `mode: adopted` and instructs the drafting agent to tag adopted-claim ACs `[adopted]` (distinct from `[delta]` / `[regression]`).
- After `/spec close` on the adoption iteration, the next `/spec interview` increments iteration and switches `mode` to `iteration` — the adoption baseline is now a normal prior takeaway.
