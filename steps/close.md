# /spec close — Finalize iteration

Resolve any outstanding GAPs, generate `spec/takeaway.md`, and archive the current iteration's `spec.md` + `takeaway.md` snapshots.

## State machine

**Allowed from phases:** `verified`
**Transitions to:** `closed`
**Re-run behavior:** **Refused** if already `closed` — tell user "Iteration N already closed. Run `/spec interview` to start iteration N+1."

Read state.yaml first; validate phase. Write state.yaml at the end of Turn 2.

## Turn 1 — Review gaps and collect rationale

1. **Validate phase.** Read state.yaml. If `closed`: refuse per above. If not `verified`: suggest the correct prior step.

2. **Load latest verify report** from `spec/archive/<latest_verify>`.

3. **Extract GAPs and UNCLEARs.** Present them to the user as a numbered list. For each:

   > GAP #<n> — AC<ref>: <one-line reason from report>
   >
   > To close this iteration, please pick one:
   > (a) Fix it in code and re-run `/spec verify`, then resume `/spec close`.
   > (b) Accept the gap with a rationale (one or two sentences) — recorded in `takeaway.md`.
   > (c) Defer it to a future iteration — recorded as a new item in `spec/deferred.md`. Especially natural for `[adopted]` ACs that turned out to be feature requests rather than verified existing behavior.

4. **Stop.** Wait for the user to address each GAP/UNCLEAR. Do not proceed until every one is (a) resolved by re-verification, (b) accepted with a rationale, or (c) deferred.

5. **Apply per-item resolutions.**
   - **(a)** items: handled outside this command (user re-runs `/spec verify`).
   - **(b)** items: collect rationales in working memory (the conversation). These will be passed into the takeaway-generation prompt in Turn 2 — do **not** write them to `state.yaml` (they belong in `takeaway.md`, not state).
   - **(c)** items: append each to `spec/deferred.md` per `steps/defer.md`'s automatic-invocation pattern, with source `via /spec close gap resolution (iteration <n>)`. Use the AC ID and one-line reason from the verify report as the deferred item's title and description. Emit one-line confirmation per item: "Deferred: <title> (D-XXX)." Do **not** propose a mid-step commit; the Turn 2 commit picks up `spec/deferred.md`.

## Turn 2 — Generate takeaway and archive

Once all GAPs/UNCLEARs are resolved or accepted:

1. **Determine snapshot filenames.** Iteration `v<NNN>` (zero-padded to 3 digits) + close timestamp `YYYY-MM-DD-HHMM` from state.yaml:
   - `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-spec.md`
   - `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-takeaway.md`

2. **Spawn takeaway generator.** Agent tool, `subagent_type: "general-purpose"`, `model: "sonnet"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7**. Prompt:

   > Generate `spec/takeaway.md` using the template at `<skill-base-dir>/templates/takeaway.md` (resolve `<skill-base-dir>` to the absolute path announced at skill invocation before passing to the sub-agent).
   >
   > Synthesize inputs:
   > - `spec/spec.md` — the converged intent.
   > - `spec/archive/<latest_verify-filename>` — the verify evidence report.
   > - `spec/decisions.log` — entries under this iteration's header.
   > - Accepted gaps with rationales (injected inline here): <list from Turn 1, each as "AC<ref> — <rationale>">.
   > - Read a few relevant source files referenced in the verify report to flesh out the `## Shipped state [from-code]` section with concrete, file-referenced bullets.
   >
   > Follow the template structure. Keep `Shipped state` factual and file-referenced. Keep `Discoveries during implementation` honest — include things learned that the spec didn't anticipate, even if they're seeds for the next iteration.
   >
   > Write `spec/takeaway.md` (use the absolute path). Attempt the Write tool; if denied, retry once; if still denied, include the full intended file content verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can write it. Never report success without either having written the file or having dumped its full content.
   >
   > Report back: path + 3-line summary + (if Write was denied) the full content dump.

3. **Verify the file exists.** After the sub-agent returns, check whether `spec/takeaway.md` was actually written. If not, extract the dumped content and write it directly using the Write tool in the parent session.

4. **User review.** Show the generator's summary. Ask user to read `spec/takeaway.md` and confirm or request edits. Do not proceed until confirmed.

5. **Archive snapshots.** Copy:
   - `spec/spec.md` → `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-spec.md`
   - `spec/takeaway.md` → `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-takeaway.md`

6. **Append iteration-boundary marker** to `spec/decisions.log`:
   ```
   ## <date> — Iteration <n> closed
   **Spec SHA at close:** <sha>
   **Accepted gaps:** <n> (details in archive/v<NNN>-<YYYY-MM-DD-HHMM>-takeaway.md)
   
   ---
   ```

7. **Update state.yaml:** `phase: closed`, `last_command: /spec close`, `last_command_at`. Reset `phases_implemented: []` so the next iteration starts with an empty per-phase audit trail.

8. **Propose commit:**
   ```
   git add spec/
   git commit -m "spec: close iteration <n>"
   ```

9. **Next step.** Tell the user: "Iteration <n> closed. Start a new conversation and run `/spec interview` to begin iteration <n+1>, or stop here if this spec is complete."
