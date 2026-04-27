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

5. **Kickoff stage.** Show the user:
   - The phase heading, `Delivers:` line, and dependency clause from `spec.md`.
   - The full list of leaf ACs in this phase (with their text from the AC tree, not just IDs).
   - Any `[adopted]`-tagged ACs and `## Invariants` / `## Provisional invariants` items as constraints to preserve.
   - A copy-pasteable kickoff prompt for a fresh conversation:

     ```
     Read spec/spec.md. Implement phase <N> ("<phase name>") only — the ACs listed under that phase
     in the ## Implementation phases section. Treat ACs from later phases as out-of-scope for now.
     Treat [adopted] ACs, ## Invariants, and ## Provisional invariants as constraints to preserve —
     do not regress them. Items under ## Test-pending are deferred; do not pick them up.
     When done, ask me to run `/spec implement` again to audit the work.
     ```

   - Tell the user: "Open a fresh conversation in this repo, paste the prompt above, implement phase `<N>`, then return here and re-run `/spec implement`. Do not commit between sessions if you'd like the audit to detect your changes — the per-phase verify uses git state to scope evidence." Stop.

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
