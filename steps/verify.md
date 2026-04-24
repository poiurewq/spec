# /spec verify — Audit code against spec

Spawn an `Explore` sub-agent to render a per-AC evidence report comparing `spec/spec.md` to the current codebase. **The agent gathers evidence. You render the verdict.**

## State machine

**Allowed from phases:** `converged` · `verified` (re-run)
**Transitions to:** `verified`
**Re-run behavior:** Allowed freely. Latest report supersedes; old reports retained in archive.

Read state.yaml first; validate phase. Write state.yaml on completion.

## Protocol

1. **Check preconditions.** Read state.yaml; validate phase. If not `converged` or `verified`, suggest the correct step (commonly `/spec check` if not yet converged).

2. **Determine filename.** Iteration `v<NNN>` (zero-padded to 3 digits) + timestamp from state.yaml: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-verify.md`.

3. **Capture current spec.md SHA.** Use `git log -1 --format=%H -- spec/spec.md` to record it; include in the report header for traceability.

4. **Spawn `Explore` sub-agent.** Agent tool with `subagent_type: "Explore"` and `model: "sonnet"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7**. Prompt:

   > Read `spec/spec.md`. For **each acceptance criterion** in the AC tree (at every level — AC1, AC1.1, AC1.2, etc.):
   >
   > - Search the codebase for evidence the AC is implemented.
   > - Render one verdict: **PASS**, **GAP**, or **UNCLEAR**.
   > - Include specific `file:line` references for every claim.
   > - Include a one-line reason.
   >
   > If the spec has an Invariants section (iteration mode), give each invariant its own entry with the same PASS/GAP/UNCLEAR verdict.
   >
   > **Do not render an overall pass/fail judgment.** Produce the raw per-item evidence so the user can judge. No hedging language in verdicts — pick one of the three values.
   >
   > Write the report to `spec/archive/<FILENAME>` using this structure:
   >
   > ```
   > # Verify report — iteration <n> — <timestamp>
   > > spec.md SHA at verify: <full-SHA>
   > 
   > ## AC1 — <title>
   > **Verdict:** PASS|GAP|UNCLEAR
   > **Evidence:** <file:line refs>
   > **Reason:** <one line>
   > 
   > ## AC1.1 — <title>
   > ...
   > 
   > ## Invariants (if applicable)
   > ### Invariant 1 — <title>
   > ...
   > 
   > ## Summary
   > Total: <n>, PASS: <n>, GAP: <n>, UNCLEAR: <n>
   > ```
   >
   > Attempt the Write tool for the absolute report path. If denied, retry once; if still denied, include the full intended report verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can write it. Never report success without either having written the file or having dumped its full content.
   >
   > Report back: the file path, the summary line, and (if Write was denied) the full content dump.

5. **Verify the report exists.** After the sub-agent returns, check whether the report was actually written. If not, extract the dumped content and write it directly using the Write tool in the parent session.

6. **Present the report to the user.** Read the file back. Show:
   - The summary line (counts).
   - A terse list of every GAP and UNCLEAR with one-line reasons.
   - Reference to full report path for deeper review.

7. **Update state.yaml:** `phase: verified`, `latest_verify: <filename>`, `last_command: /spec verify`, `last_command_at`.

8. **Next step.** Tell the user:
   - **If any GAPs/UNCLEARs:** "Either address them in code and re-run `/spec verify`, or when ready, run `/spec close` — you'll be asked to provide a rationale for each outstanding gap before closing."
   - **If all PASS:** "Run `/spec close` to finalize the iteration and generate the takeaway."

9. **Propose commit** (optional — verify reports are working artifacts but small):
   ```
   git add spec/archive/<verify-file> spec/state.yaml
   git commit -m "spec: verify v<n> at <short-SHA>"
   ```
