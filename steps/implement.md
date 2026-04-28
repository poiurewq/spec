# /spec implement — Per-phase implementation orchestration

Walk the user through `## Implementation phases` one phase at a time: kick off implementation in a fresh session, then on re-run audit just that phase's ACs and propose a commit. **The skill does not implement the code itself** — it orchestrates and verifies. Implementation happens in a separate fresh conversation, per the global "separate conversations for generative steps" principle.

## State machine

**Allowed from phases:** `converged` · `verified` (re-run is harmless; useful if user adds/edits a phase post-verify)
**Transitions to:** stays in current phase (does not move the main `phase` field)

State change: appends to `phases_implemented: [<int>]` in `state.yaml` once the user confirms a phase's per-phase verify is acceptable.

**Re-run behavior:** Idempotent. Re-running on a phase already in `phases_implemented` reports "phase N already confirmed; next is phase N+1" and offers to re-audit if requested.

Read `state.yaml` first; validate phase. Write `state.yaml` only when the user confirms a phase as done.

## Protocol

1. **Check preconditions.** Read `state.yaml`; validate `phase` is `converged` or `verified`. If `revised` or earlier, tell the user the spec must be converged first and stop. If `closed`, refuse.

2. **Parse phases.** Read `spec/spec.md`. Extract the `## Implementation phases` section. If absent, tell the user this iteration has no declared phases and they should run `/spec verify` directly when implementation is done; stop. If present but says "None — this iteration introduces no new implementation work.", same outcome.

3. **Determine phase under work.** Let `total = number of phases declared`. Let `done = state.yaml: phases_implemented` (default `[]`). The next phase is `next_phase = min(set(1..total) - set(done))`.
   - If `done == [1..total]`, all phases are confirmed. Tell the user: "All phases implemented and per-phase verified. Run `/spec verify` for the full-spec audit." Stop.
   - User may pass an explicit phase number as argument (`/spec implement 2`); honor it but warn if it skips earlier unconfirmed phases.

4. **Branch on per-phase state.** A phase has two stages: *kickoff* (user has not yet implemented) and *audit* (user has implemented; ready to verify). Distinguish by checking whether the working tree has changes since the most recent commit that modified `spec/state.yaml`'s `phases_implemented` field (or, on the very first phase, since the converge commit).

   Heuristic: run `git log -1 --format=%H -- spec/state.yaml` to find the last state commit, then `git diff <that-SHA>..HEAD` and `git status --short`. If there are any non-`spec/` file changes since that point, treat as **audit** stage. Otherwise **kickoff**.

   When ambiguous, ask the user: "Have you implemented phase `<N>` already, or are we starting it?"

5. **Kickoff stage.** The kickoff prompt is **generated**, not templated — the orchestrator extracts the exact context this phase needs and embeds it inline so the fresh worker session does minimal exploration before writing code.

   **Selection over transcription.** Your job here is to *select*, not to dump. The worker has the full spec at `spec/spec.md` and can read it. Verbatim text in the prompt is for items the worker must internalize before writing code — anything else should be a line-number pointer the worker can follow on demand. A kickoff prompt that pastes 18 invariants when only 3 apply to this phase is worse than one that names the 3 and points to the rest by line range: it dilutes attention away from what actually matters. Bias toward terse and pointed; the worker will read the source if it needs more.

   **5a. Extract context from `spec/spec.md`** (record line ranges as you go — the prompt will cite them):

   - **Phase block:** the `### Phase <N>` heading, its `**Delivers:**` line, and its `**Depends on:**` / `**Unblocks:**` clause. Capture the line range.
   - **Phase ACs (verbatim):** for each leaf AC ID listed under this phase, find its full text in the `## Acceptance criteria` tree. Capture both the ID-and-text and the source line for each. ACs *are* the work — these stay verbatim.
   - **Adjacent phases:** the headings + Delivers lines of phase `<N-1>` (already done) and phase `<N+1>` (not yet started), if they exist. These bound the scope. Capture line ranges only — do not embed verbatim.
   - **Constraints to preserve (selectively):** read all items under `## Invariants`, `## Provisional invariants`, and any `[adopted]`-tagged ACs. Pick out only the items plausibly load-bearing for this phase's scope. Embed the selected items verbatim. Always cite the section's full line range so the worker can scan the rest if needed. If you cannot tell whether an item applies, include it — the bar is "plausibly relevant," not "definitely relevant."
   - **Deferred items:** the line range of `## Test-pending`, if present. Do not embed — just point.
   - **Mode-specific context** from `state.yaml`:
     - `mode: adopted` → note that `[adopted]` ACs are unverified claims about pre-existing behavior; if the worker finds them already satisfied, minimal-touch is correct.
     - `mode: iteration` → if `spec/takeaway.md` exists, capture the path; the worker may consult it for prior-iteration context.
     - `mode: greenfield` → no extra context.

   **5b. Determine phase-position narrative:**

   - **Foundation** (no `**Depends on:**` clause, or `**Unblocks:** foundation`): "This is the bedrock; no prior phase output to integrate with."
   - **Dependent** (`**Depends on:** Phase X (...)`): "Phase `<X>` is already implemented and committed (SHA `<short-SHA>` from `git log -1 --format=%h <state-commit>`). Build on what it delivers; do not re-implement it."
   - **Final phase** (no later phase exists): "This phase completes the iteration's implementation work; `/spec verify` runs next."

   **5c. Show the user** the extracted summary (phase heading, Delivers, deps, AC list with text, constraints) — same as before, so the user can sanity-check before pasting.

   **5d. Compose the kickoff prompt** with this structure (fill in from extractions; omit sections that are empty):

   ```
   Implement phase <N> of spec/spec.md ("<phase short name>"). Do not implement other phases.

   ## What this phase delivers
   <verbatim Delivers: line>
   <phase-position narrative from 5b — one or two sentences>

   ## ACs to satisfy (this phase only)
   <For each leaf AC, on its own line:>
   - <AC-ID>: <verbatim AC text>   (spec.md:<line>)

   ## Constraints — do not regress these
   <Embed only the invariants / provisional invariants / [adopted] ACs you judged plausibly load-bearing for this phase in 5a. For each, include verbatim text + a one-line note on why it's relevant here. Always cite the source section's full line range so the worker can read the unselected items on demand.>
   <If Invariants selected:>
   Invariants (full list at spec.md:<start>-<end>; selected here as relevant to this phase):
   - <verbatim invariant>  — relevant because <one line>
   <If Provisional invariants selected, same shape.>
   <If [adopted] ACs selected:>
   [adopted] ACs — claims about existing behavior to preserve (full list at spec.md:<start>-<end>; selected here):
   - <AC-ID>: <verbatim text>  — relevant because <one line>
   <If you selected nothing from a section, omit the section entirely — do not write "none selected.">

   ## Out of scope
   - Phase <N+1>+ ACs (spec.md:<line-range-of-later-phases>) — do not pre-build.
   - Items under ## Test-pending (spec.md:<line>) — deferred; ignore.
   <If mode == adopted:>
   - [adopted] ACs not in this phase's scope are verification targets for /spec verify, not implementation work for you.

   ## Reference
   - Full phase block: spec.md:<phase-line-range>
   - Full AC tree: spec.md:<AC-section-line-range>
   <If mode == iteration and takeaway.md exists:>
   - Prior iteration takeaway: spec/takeaway.md  (consult only if you need historical context)

   ## Design decisions — surface, don't decide
   The ACs and constraints above pin down *what* must be true, not *how* to get there. Where the spec is silent or ambiguous on an implementation choice — data shape, API surface, library selection, error semantics, naming, file/module layout, migration vs. rewrite, sync vs. async, etc. — do not pick unilaterally. Stop, name the fork and the options you see (with the tradeoffs you'd weigh), and ask the user before writing code down that path. The working principle that "the user judges" should apply to design decisions during implementation. A small clarifying question now is cheaper than rework after audit.

   When done, do not commit. Tell the user the implementation is ready for audit; they will re-run `/spec implement` from the orchestrating session, which uses uncommitted git state to scope the audit.
   ```

   Render this prompt inside a single fenced code block so the user can copy it cleanly.

   **5e. Tell the user:** "Open a fresh conversation in this repo, paste the prompt above, implement phase `<N>`, then return here and re-run `/spec implement`. Do not commit between sessions — the per-phase audit uses uncommitted working-tree state to scope evidence." Stop.

6. **Audit stage.** Determine filename: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-implement-phase<N>.md`. Spawn an `Explore` sub-agent with `model: "sonnet"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7**:

   > Read `spec/spec.md`. From the `## Implementation phases` section, identify phase `<N>` and the leaf AC IDs it covers. For **each AC in this phase only** (do not audit other phases or the full spec):
   >
   > - Search the codebase for evidence the AC is implemented.
   > - Render one verdict: **PASS**, **GAP**, or **UNCLEAR**.
   > - Include specific `file:line` references for every claim.
   > - Include a one-line reason.
   >
   > Additionally, for any `## Invariants` or `## Provisional invariants` listed in `spec.md`, run a quick regression check (PASS / GAP / UNCLEAR) — if the phase's implementation could plausibly have touched the same code paths. Skip invariants that are clearly orthogonal; say "skipped — orthogonal to phase scope" for those.
   >
   > **Do not render an overall verdict.** Per-AC evidence only. The user judges.
   >
   > Write the report to `spec/archive/<FILENAME>` (absolute path) using this structure:
   >
   > ```
   > # Implement audit — iteration <n> — phase <N> — <timestamp>
   > > spec.md SHA: <full-SHA>
   > > Phase scope: <comma-separated AC IDs>
   >
   > ## AC1.1 — <title>
   > **Verdict:** PASS|GAP|UNCLEAR
   > **Evidence:** <file:line refs>
   > **Reason:** <one line>
   >
   > ## Invariant regression check
   > ### Invariant 1 — <title>
   > **Verdict:** PASS|GAP|UNCLEAR|skipped — orthogonal
   > ...
   >
   > ## Summary
   > Phase ACs: <n>, PASS: <n>, GAP: <n>, UNCLEAR: <n>
   > Invariants checked: <n>, regressions detected: <n>
   > ```
   >
   > Attempt the Write tool. If denied, retry once; if still denied, dump the full report verbatim in your final reply inside a fenced code block labeled with the absolute target path. Never report success without writing or dumping.
   >
   > Report back: file path, summary line, and (if Write was denied) the full content dump.

7. **Verify the report exists.** If not written by the sub-agent, extract dumped content and write it directly using the Write tool.

8. **Present evidence.** Read the report. Show the summary line, every GAP / UNCLEAR with one-line reasons, and any invariant regressions detected. Reference the full report path.

9. **Ask the user to judge.** Three options:
   - **Confirm** — phase is acceptable as-is (user may explicitly accept GAPs/UNCLEARs with rationale; if so, log them via `/spec decide` automatically with a one-line decision).
   - **Iterate** — user wants to fix issues; tell them to address the code, then re-run `/spec implement` (loops back to audit stage on this same phase).
   - **Abort** — back out; no state change.

10. **On confirm:**
    - Append `<N>` to `phases_implemented` in `state.yaml`.
    - Update `last_command: /spec implement`, `last_command_at`.
    - Propose:
      ```
      git add <implementation files> spec/archive/<implement-audit-file> spec/state.yaml
      git commit -m "spec: implement phase <N> — <phase short name> (iteration <n>)"
      ```
      List the implementation files explicitly (use `git status --short` output) so the user can edit the `git add` line if needed. Do not run.
    - If `len(phases_implemented) == total`, append: "All phases confirmed. Next: `/spec verify` for the full-spec audit." Otherwise: "Next phase: `<N+1>`. Run `/spec implement` again after implementing it in a fresh session."

## Notes

- This step deliberately does **not** transition `phase` in `state.yaml`. The main phase remains `converged` until `/spec verify` runs the full-spec audit and the user accepts. Per-phase audits are early-warning evidence, not substitutes for the final verification.
- If the user adds or removes phases in `spec.md` mid-implementation (e.g., a revise pass), the `phases_implemented` list may go stale. Detect by comparing `total` to `max(phases_implemented)`; if mismatched, warn and ask the user whether to clear the list and start over.
- Skipping phases (`/spec implement 3` while `phases_implemented = []`) is allowed but warned — the no-forward-deps invariant is a guideline, not a lock.
