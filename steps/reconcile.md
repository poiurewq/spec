# /spec reconcile — Pull reality back into the spec

Capture drift between `spec/spec.md` and the codebase as it actually is — design decisions made during implementation, unstated assumptions surfaced while building, AC text that turned out to be wrong. **Bidirectional ingestion**: most other steps project the spec onto the code; reconcile pulls the code's reality back into the spec.

This is the right tool when, mid-implementation or post-verify, you discover something the spec didn't constrain (and didn't need to) but should now record. It is **not** a substitute for `/spec revise` — that's where revisions in response to *review critiques* live.

## State machine

**Allowed from phases:** `converged` · `verified`
**Transitions to:** depends on resolution bucket (see Buckets below) — may stay, may drop to `revised`, may drop to `in-review`.
**Re-run behavior:** Free; each invocation processes a fresh batch of drift items.

**Refused from other phases** with redirects:
- `interviewing` / `seeded` / `in-review` / `revised` → "The spec is still being formed. Use `/spec interview` (if pre-seed) or `/spec revise` (if post-seed). Reconcile is for drift discovered after convergence."
- `closed` → "Iteration is closed. Drift discovered post-close belongs in the next iteration's interview. Run `/spec interview` to start iteration N+1."

Read state.yaml first; validate phase. Write state.yaml on completion (only if phase changes).

## Entry modes

Three ways to invoke. They differ only in **how drift items are sourced**; the per-item dialogue and resolution are identical.

1. **`/spec reconcile`** (bare) — spawn a drift-detection sub-agent over the whole codebase. **Confirm with the user before spawning** (see step 2 below).
2. **`/spec reconcile "<description>"`** — user describes the specific drift in their own words (e.g., "I added retries with exponential backoff to the webhook publisher; spec doesn't mention it"). No sub-agent.
3. **`/spec reconcile --from-verify`** — load drift candidates from the latest verify report's GAPs and UNCLEARs. Refused if `latest_verify` in state.yaml is null.

## Protocol

1. **Check preconditions.** Read state.yaml; validate phase per the State machine section. Refuse with the appropriate redirect if outside `converged` / `verified`.

2. **Confirm intent (bare invocation only).** If invoked as bare `/spec reconcile` with no description and no `--from-verify`, ask the user once before doing any codebase exploration:

   > Bare `/spec reconcile` will spawn a sub-agent to scan the codebase for drift against `spec/spec.md`. This is the broadest and most expensive entry mode. Alternatives:
   > - `/spec reconcile "<description>"` — you describe the specific drift; no scan.
   > - `/spec reconcile --from-verify` — load drift candidates from the latest verify report (`<latest_verify-filename>`).
   >
   > Proceed with full codebase scan, or switch to one of the alternatives?

   Wait for confirmation. If the user picks an alternative, restart at step 1 with the new entry mode.

3. **Source the drift items.**

   **3a. From description (`/spec reconcile "<text>"`).** Parse the user's text into one or more discrete drift items. If ambiguous, ask the user to enumerate.

   **3b. From verify report (`--from-verify`).** Read `spec/archive/<latest_verify>`. Each GAP and UNCLEAR entry is a candidate drift item. PASS entries are not candidates.

   **3c. From codebase scan (bare, after confirmation).** Triage scope before spawning, the same way `/spec verify` does:
   - Count total leaf ACs + invariants in `spec/spec.md`.
   - Count source breadth: `find . -mindepth 1 -maxdepth 2 -not -path '*/\.*' -not -path '*/node_modules/*' -type d | wc -l`.

   Direct path — handle in-session — if **all** hold: total leaf ACs + invariants ≤ 5, AND source code is concentrated in ≤ 3 directories. Use Bash `grep`/`find` and Read to compile the drift report yourself directly into the structure shown in the sub-agent prompt below.

   Sub-agent path — if any condition above fails: use the Agent tool with `subagent_type: "Explore"`, `model: "sonnet"`, thoroughness `"very thorough"`, `background: false`. The prompt must include the **write-fallback instruction from SKILL.md principle 7**. Determine drift report filename: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-reconcile-drift.md`. Prompt:

   > Read `spec/spec.md`. Compare it to the current state of the codebase. Your task is to surface **drift** — places where the spec and the code don't agree, or where the code reveals constraints/decisions the spec didn't capture.
   >
   > Produce a drift report with three sections. Each item must include `file:line` references and a one-line description.
   >
   > 1. **Code-not-in-spec** — behavior, constraints, error semantics, or design choices present in code but not asserted (or constrained) by any AC, invariant, or `[adopted]` claim in the spec.
   > 2. **Spec-not-in-code** — ACs or invariants in the spec that the code does not implement, contradicts, or implements differently than asserted.
   > 3. **Implicit-constraints** — invariants the code clearly relies on (orderings, idempotency assumptions, schema shapes, retry behavior, etc.) that the spec does not state.
   >
   > **Do not render verdicts** ("this is a bug", "this should be added"). Just describe what you observed. The user will classify each item in dialogue with the orchestrator.
   >
   > Skip items that are obviously orthogonal noise (lint config, build scripts, dev-only test fixtures) — drift means semantically meaningful divergence.
   >
   > Write the report to `spec/archive/<FILENAME>` (absolute path) using this structure:
   >
   > ```
   > # Reconcile drift report — iteration <n> — <timestamp>
   > > spec.md SHA: <full-SHA>
   >
   > ## Code-not-in-spec
   > 1. <one-line description>  (<file:line>)
   > 2. ...
   >
   > ## Spec-not-in-code
   > 1. <one-line description>  (<file:line>; relates to AC <ID> at spec.md:<line>)
   > 2. ...
   >
   > ## Implicit-constraints
   > 1. <one-line description>  (<file:line>)
   > 2. ...
   >
   > ## Summary
   > Code-not-in-spec: <n>, Spec-not-in-code: <n>, Implicit-constraints: <n>
   > ```
   >
   > Attempt the Write tool. If denied, retry once; if still denied, dump the full report verbatim in your final reply inside a fenced code block labeled with the absolute target path. Never report success without writing or dumping.
   >
   > Report back: file path, summary line, and (if Write was denied) the full content dump.

   After the sub-agent returns, verify the report exists. If not, extract the dumped content and write it directly. If zero drift items were found across all three sections, tell the user "No drift detected. Spec and code are aligned." and stop.

4. **Turn 1 — Present drift items with suggested buckets.** Show the user a numbered list of drift items (regardless of source). For each item, suggest a bucket with one-line rationale. **Do not classify silently — the user makes the final call.** The four buckets:

   1. **Decision-only** — append to `decisions.log` via `/spec decide`'s entry format; `spec.md` untouched. Phase unchanged.
      *Suggest when:* the drift is a non-load-bearing implementation choice (library/algorithm pick, naming, internal refactor) that no AC or invariant constrains.
   2. **Minor spec edit** — direct edit to `spec.md`. MECE re-check. Phase unchanged. May include adding `[adopted]`-tagged ACs to record de-facto behavior, clarifying wording without changing assertion, or adding an obviously-true unstated invariant.
      *Suggest when:* the spec needs to absorb the reality but no existing AC's *assertion* changes and no AC is removed.
   3. **Structural spec edit** — edit `spec.md`; phase drops to `revised`, forcing `/spec check` before re-converging. May include changing an AC's assertion, splitting/merging ACs, modifying an existing invariant, **deleting an AC**, or adding/removing an implementation phase.
      *Suggest when:* an existing AC's truth value changes, ACs are added or removed, or invariants change.
   4. **Major / contested** — edit `spec.md`; phase drops to `in-review`, forcing a fresh persona review pass.
      *Suggest when:* the drift implies a goal/scope rethink, the change cascades across multiple ACs, or the user themselves is unsure of the right resolution.

   End the turn with:

   > For each drift item, tell me:
   > - The bucket you want (1/2/3/4), or `reject` if it isn't real drift.
   > - Your override rationale if you're overriding my suggestion.

5. **Turn 2 — User addresses items.** User responds per item. During dialogue:
   - For any "unsure" item, apply **condensed diamond** — enumerate ≥3 ways to resolve the drift with tradeoffs before helping the user converge on a bucket.
   - Log significant choices to `decisions.log` per `steps/decide.md` (under the current iteration's header). In particular, every override of the suggested bucket gets a one-line decision entry capturing *why*.
   - Do **not** apply changes yet. Wait until the user signals all items are addressed (or explicitly deferred to a later reconcile).

6. **Turn 3 — Apply.** Compute the **final phase** from the highest-severity bucket used:
   - Any bucket-4 item → final phase `in-review`.
   - Else any bucket-3 item → final phase `revised`.
   - Else (only buckets 1 and 2) → phase unchanged.

   **6a. `phases_implemented` impact check.** If any bucket-3 or bucket-4 item changes or removes an AC that lives in a phase already in `phases_implemented`, list each affected phase and ask:

   > Phase `<N>` is in `phases_implemented` but its AC `<ID>` is being modified/removed. Drop phase `<N>` from `phases_implemented` (it will need re-implementation) or keep it (the change describes what's already built)?

   Honor the user's per-phase answer. Do not auto-clear.

   **6b. Bucket 1 items.** For each, append an entry to `decisions.log` per `steps/decide.md`'s entry format. Use `**Context:**` line `during /spec reconcile (iteration <n>)`.

   **6c. Bucket 2 items.** Apply spec.md edits inline as a unified diff (small) or via direct `Write` (large) — same shape as `/spec revise` Turn 3. **MECE re-check** before applying. In iteration/adopted modes, verify `[delta]` / `[regression]` / `[adopted]` tags are still correct.

   **6d. Bucket 3 / 4 items.** Apply spec.md edits the same way. **MECE re-check.** Tag any new ACs that record pre-existing behavior with `[adopted]`.

   **6e. Update state.yaml.**
   - `last_command: /spec reconcile`, `last_command_at: <ISO timestamp>`.
   - `phase`: set per the final-phase computation above.
   - `phases_implemented`: drop any phase numbers the user marked for re-implementation in 6a.
   - `latest_review_stamp`: leave unchanged. (When phase drops to `in-review`, the user will run `/spec review` next, which writes a fresh stamp.)

   **6f. Propose commit.**
   ```
   git add spec/spec.md spec/decisions.log spec/state.yaml spec/archive/<drift-report-if-any>
   git commit -m "spec: reconcile drift (iteration <n>)"
   ```

7. **Tell the user what's next** based on the final phase:
   - **Unchanged phase (`converged` / `verified`):** "Reconcile complete. Phase remains `<phase>`. <If verified and any spec.md edits were applied:> Note: `spec.md` changed; consider re-running `/spec verify` for a fresh full-spec audit."
   - **`revised`:** "Reconcile complete. Phase dropped to `revised` because of structural changes. Run `/spec check` to re-confirm convergence (likely cheap if the changes were truly contained), then `/spec implement` or `/spec verify` as appropriate."
   - **`in-review`:** "Reconcile complete. Phase dropped to `in-review` because of major changes. Run `/spec review` for a fresh persona pass, then `/spec revise` and `/spec check`."

## Notes for downstream steps

- **Interaction with `/spec verify`.** When phase stays at `verified` after a bucket-2 reconcile, the existing verify report is *partially stale* — it was rendered against the prior `spec.md`. `state.yaml` does not capture this; the message in step 7 nudges the user. Do not invalidate `latest_verify` automatically (the report is still useful for the unchanged ACs).
- **Interaction with `/spec implement`.** A reconcile during `converged` that touches phase ACs may invalidate per-phase audit reports for the affected phases. Step 6a covers the explicit case (dropping phases from `phases_implemented`); the per-phase audit files in `spec/archive/` are not deleted — they remain as historical record and will be superseded when the phase is re-implemented.
- **Decision-only reconciles do not require a drift report file.** If all items are bucket 1 (or come from a description with no scan), no `*-reconcile-drift.md` archive file is produced. Only the codebase-scan entry mode writes that file.
