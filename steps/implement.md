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

4. **Branch on per-phase state.** A phase has two stages: *kickoff* (user has not yet implemented) and *audit* (user has implemented; ready to verify). Distinguish by looking for non-`spec/` changes since the **state boundary** — the most recent commit that modified `spec/state.yaml`'s `phases_implemented` field (or, on the very first phase, the converge commit).

   Detection procedure:
   - Run `git log -1 --format=%H -- spec/state.yaml` to find the state-boundary SHA. (Sanity-check it actually touches `phases_implemented`; if not, walk back with `git log --format=%H -- spec/state.yaml` until you find one that does, or fall back to the converge commit.)
   - Collect the **phase diff candidates**: `git log --format='%h %s' <boundary>..HEAD` (commits since boundary) plus `git status --short` (uncommitted). Filter to entries that touch non-`spec/` files.
   - If both lists are empty → **kickoff**.
   - Otherwise → **audit**. The union of files touched across these commits + uncommitted changes is the **phase diff scope** — pass it to step 6.

   **Disambiguation.** If the candidate commits look noisy or unrelated to phase `<N>` — e.g., more than ~5 commits since the boundary, commit messages that don't read like phase-`<N>` work, or commits that appear to span multiple phases — show the user the candidate list (`<short-SHA> <subject>` per line, plus any uncommitted file count) and ask:

   > "Which commits implement phase `<N>`? Reply with a range (`abc123..def456`), a comma-separated list (`abc123,def456`), `all` to use everything since the state boundary, or `none` if you haven't implemented this phase yet."

   Honor the user's answer as the authoritative scope. If the user answers `none`, treat as **kickoff**. Also ask when the candidate list is empty but the user previously indicated they had implemented (e.g., re-running after a manual stash).

5. **Kickoff stage.** The kickoff prompt is **generated**, not templated — the orchestrator extracts the exact context this phase needs and embeds it inline so the fresh worker session does minimal exploration before writing code.

   **Selection over transcription.** Your job here is to *select*, not to dump. The worker has the full spec at `spec/spec.md` and can read it. Verbatim text in the prompt is for items the worker must internalize before writing code — anything else should be a line-number pointer the worker can follow on demand. A kickoff prompt that pastes 18 invariants when only 3 apply to this phase is worse than one that names the 3 and points to the rest by line range: it dilutes attention away from what actually matters. Bias toward terse and pointed; the worker will read the source if it needs more.

   **5a. Extract context from `spec/spec.md`** (record line ranges as you go — the prompt will cite them):

   - **Phase block:** the `### Phase <N>` heading, its `**Delivers:**` line, and its `**Depends on:**` / `**Unblocks:**` clause. Capture the line range.
   - **Phase ACs (verbatim):** for each leaf AC ID listed under this phase, find its full text in the `## Acceptance criteria` tree. Capture both the ID-and-text and the source line for each. ACs *are* the work — these stay verbatim.
   - **Adjacent phases:** the headings + Delivers lines of phase `<N-1>` (already done) and phase `<N+1>` (not yet started), if they exist. These bound the scope. Capture line ranges only — do not embed verbatim.
   - **Constraints to preserve (selectively):** read all items under `## Invariants`, `## Provisional invariants`, and any `[adopted]`-tagged ACs. Pick out only the items plausibly load-bearing for this phase's scope. Embed the selected items verbatim. Always cite the section's full line range so the worker can scan the rest if needed. If you cannot tell whether an item applies, include it — the bar is "plausibly relevant," not "definitely relevant."
   - **Deferred items live outside `spec.md`** in `spec/deferred.md`. Do not include or reference — they are explicitly out of scope for any implement worker.
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
   <If mode == adopted:>
   - [adopted] ACs not in this phase's scope are verification targets for /spec verify, not implementation work for you.

   ## Reference
   - Full phase block: spec.md:<phase-line-range>
   - Full AC tree: spec.md:<AC-section-line-range>
   <If mode == iteration and takeaway.md exists:>
   - Prior iteration takeaway: spec/takeaway.md  (consult only if you need historical context)

   ## Design decisions — surface, don't decide
   The ACs and constraints above pin down *what* must be true, not *how* to get there. Where the spec is silent or ambiguous on an implementation choice — data shape, API surface, library selection, error semantics, naming, file/module layout, migration vs. rewrite, sync vs. async, etc. — do not pick unilaterally. Stop, name the fork and the options you see (with the tradeoffs you'd weigh), and ask the user before writing code down that path. The working principle that "the user judges" should apply to design decisions during implementation. A small clarifying question now is cheaper than rework after audit.

   When done, tell the user the implementation is ready for audit; they will re-run `/spec implement` from the orchestrating session. You may commit your work normally (one or more commits is fine) or leave it uncommitted — the orchestrator scopes the audit from commits since the last `spec/state.yaml` commit plus any uncommitted changes, and will ask the user to disambiguate if the boundary is unclear.
   ```

   Render this prompt inside a single fenced code block so the user can copy it cleanly.

   **5e. Tell the user:** "Open a fresh conversation in this repo, paste the prompt above, implement phase `<N>`, then return here and re-run `/spec implement`. Committing between sessions is fine — the audit will scan recent git log entries (since the last `state.yaml` commit) plus any uncommitted changes to find the phase diff, and will ask you to confirm the commit range if it's ambiguous." Stop.

6. **Audit stage.** Determine filename: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-implement-phase<N>.md`. Before spawning, triage scope using the leaf AC count for phase `<N>` (already extracted in step 5a).

   The **phase diff scope** from step 4 (commits + any uncommitted files) tells the auditor *where to look first*. Pass it through as a hint — the auditor still reads code at HEAD (committed state takes precedence; uncommitted is layered on top via `git diff`).

   - **Direct path — handle in-session** if the phase has ≤ 3 leaf ACs: use Bash `grep`/`find` and Read to gather evidence for each AC and run the invariant regression check. For changed files, read the current state at HEAD; for any uncommitted modifications, layer in `git diff` output. Write the audit report yourself to the filename above using the structure shown in the sub-agent prompt below. Proceed to step 7.
   - **Sub-agent path** — if the phase has > 3 leaf ACs, or if the invariant scope is wide enough that a single-pass grep would be unreliable: spawn an `Explore` sub-agent with `model: "sonnet"`. Include the phase diff scope (commit SHAs + file list) in the prompt. The prompt must include the **write-fallback instruction from SKILL.md principle 7**:

   > Read `spec/spec.md`. From the `## Implementation phases` section, identify phase `<N>` and the leaf AC IDs it covers. The orchestrator has identified the phase diff as: commits `<SHA-list-or-"none">`; uncommitted files `<file-list-or-"none">`. Use this as a starting point but do not limit yourself to it — search the whole codebase if AC evidence might live elsewhere. For **each AC in this phase only** (do not audit other phases or the full spec):
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
    - Propose a commit. The shape depends on whether the implementation is already committed:
      - **All implementation already committed** (no uncommitted non-`spec/` files):
        ```
        git add spec/archive/<implement-audit-file> spec/state.yaml
        git commit -m "spec: confirm phase <N> audit — <phase short name> (iteration <n>)"
        ```
      - **Some implementation still uncommitted**: list those files explicitly (from `git status --short`) so the user can decide whether to roll them into one commit or split:
        ```
        git add <uncommitted implementation files> spec/archive/<implement-audit-file> spec/state.yaml
        git commit -m "spec: implement phase <N> — <phase short name> (iteration <n>)"
        ```

      Do not run; let the user edit the `git add` line.
    - If `len(phases_implemented) == total`, append: "All phases confirmed. Next: `/spec verify` for the full-spec audit." Otherwise: "Next phase: `<N+1>`. Run `/spec implement` again after implementing it in a fresh session."

## Notes

- This step deliberately does **not** transition `phase` in `state.yaml`. The main phase remains `converged` until `/spec verify` runs the full-spec audit and the user accepts. Per-phase audits are early-warning evidence, not substitutes for the final verification.
- If the user adds or removes phases in `spec.md` mid-implementation (e.g., a revise pass), the `phases_implemented` list may go stale. Detect by comparing `total` to `max(phases_implemented)`; if mismatched, warn and ask the user whether to clear the list and start over.
- Skipping phases (`/spec implement 3` while `phases_implemented = []`) is allowed but warned — the no-forward-deps invariant is a guideline, not a lock.
