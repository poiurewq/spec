# /spec seed — Draft or revise the Seed

Summarize the latest interview transcript into `spec/spec.md`. Bounded transform — run as an Opus sub-agent.

## State machine

**Allowed from phases:** `interviewing` (after gate passes) · `seeded` (re-draft)
**Transitions to:** `seeded`
**Re-run behavior:** Allowed. Warn user that current `spec.md` will be replaced.

Read state.yaml first; validate phase. Write state.yaml on completion.

## Protocol

1. **Check prerequisites.** Read state.yaml. Confirm `latest_interview` points to a file that exists. If not, tell the user to run `/spec interview` first and stop.

2. **Pick template based on mode** (from state.yaml). Paths are relative to the skill base directory (announced at skill invocation):
   - `mode: greenfield` → `templates/spec.md`
   - `mode: iteration` → `templates/spec-iteration.md`
   - `mode: adopted` → `templates/spec-iteration.md` (same template; tagging rules differ — see sub-agent prompt below)

   Resolve to an absolute path by prefixing the skill base directory when passing to the sub-agent (sub-agents do not inherit the skill's working context).

3. **Spawn drafting sub-agent.** Use the Agent tool with `subagent_type: "general-purpose"` and `model: "sonnet"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7** (dump full file content on Write denial). Prompt:

   > Read the interview file at `spec/archive/<latest_interview-from-state.yaml>`. If `spec/spec.md` already exists, also read it. If mode is iteration and `spec/takeaway.md` exists, also read it (as "prior iteration shipped state").
   >
   > Produce a new `spec/spec.md` using the template at `<template-path-from-step-2>`.
   >
   > Requirements:
   > - Goal in one sentence.
   > - Constraints bulleted.
   > - Success criteria measurable.
   > - Explicit out-of-scope.
   > - **MECE** AC tree: mutually exclusive, collectively exhaustive relative to Goal + Success criteria, each leaf independently testable.
   > - **In iteration mode:** fill Motivation, Current state `[from-code]` (with file refs), Change delta (Added / Modified / Removed), Invariants, Migration (if applicable). Tag each AC as `[delta]` or `[regression]`.
   > - **In adopted mode:** fill Motivation (often "formalize the spec for an existing system"), Current state `[from-code]` (pulling from the interview's pre-populated `### Shipped reality` section and the user's corrections during interview — with file refs), Change delta (may be empty — say "None — adoption iteration ratifies current behavior."), Invariants (= everything the user confirmed the system currently does), Migration (usually "None.").
   >   - **AC tagging for adopted mode:** every AC that describes a claim about existing behavior gets `[adopted]` — this means "we think the system does this; `/spec verify` is the first rigorous confirmation." If the rough spec + interview also introduce genuinely new work, tag those ACs `[delta]`. Use `[regression]` only for behaviors the user *explicitly verified by hand* during the interview (rare in adoption). Do **not** silently downgrade `[adopted]` to `[regression]` — the epistemic distinction matters for the verify step.
   >   - **`[build-change-todo]` items from Phase A of the adoption interview** represent drift the user decided to resolve by changing *code* (not spec). Translate each into an AC tagged `[delta]` describing the spec's retained claim as the target behavior, and note under `## Change delta → Modified` that the code currently diverges and must be updated. Do not drop these — they are the reason "Change delta may be empty" does not apply to this adoption iteration.
   >   - **`[iteration-scope]` items from Phase A** represent resolutions the user affirmed only for this iteration (expected to change in a future iteration — e.g., "no `SettingsViewModel` yet; will extract next iteration"). In the spec body, describe the *current* shape as shipped reality — do **not** enshrine it as a permanent invariant. Also ensure each `[iteration-scope]` item has a corresponding entry in `spec/deferred.md` (the interview step should have appended these via `/spec defer`'s automatic-invocation pattern with source `via /spec interview Phase A (iteration <n>)`; if any are missing, add them now — if `spec/deferred.md` does not exist, create it first with the header: `# Deferred items\n\n_Backlog of feature requests, bug reports, and ideas not yet committed to any iteration. Triaged at each iteration's interview._`). Do **not** route these to `decisions.log` — they are future-tense items.
   >   - **`[invariant-provisional]` items from Phase B** represent invariants the user committed to as invariants but flagged as revisable under evidence-driven pressure (e.g., "theme-switch checklist is invariant, but real-world bug reports may motivate refinement"). List these in a **separate `## Provisional invariants` section** (placed immediately after `## Invariants` — do **not** mix them into `## Invariants`, which must remain a scannable list of hard guarantees). Each item carries an inline `Trigger: <brief description>` clause naming what evidence would motivate revision. Do **not** silently downgrade to regular invariants (loses the user's explicit epistemic caveat); do **not** promote to `[iteration-scope]` deferrals (there is no scheduled change). If there are no such items, omit the section entirely.
   > - **Implementation phases.** After the AC tree, fill the `## Implementation phases` section. Group leaf ACs into ordered phases such that **phase N is fully implementable without any work from phase N+1 or later** — dependencies flow only backward. The implementer must be able to work strictly in numerical order without ever needing to reach into a later phase to make an earlier one work.
   >   - **Greenfield mode:** every leaf AC is phased; each leaf appears in exactly one phase.
   >   - **Iteration / adopted modes:** phase only `[delta]` ACs (including `[delta]` ACs derived from `[build-change-todo]` items). Do **not** phase `[regression]` or `[adopted]` ACs — those are verification targets, not implementation work. If the iteration has no `[delta]` ACs, write "None — this iteration introduces no new implementation work." under the section.
   >   - For each phase, write a one-line `**Delivers:**` (what works at end of phase) and either `**Unblocks:**` (for the foundation phase) or `**Depends on:**` (citing the earlier phase and which specific piece). **Always name the referenced phase by purpose, not just number** — e.g., `**Unblocks:** Phase 12 (submission-readiness assembly)` or `**Depends on:** Phase 1 (research-gate decisions)`. Bare-number references (`**Unblocks:** Phase 12`) silently rot when phase numbering shifts during revision; the purpose phrase makes stale references self-evident on read.
   >   - Prefer fewer, larger phases over many small ones. If the work is one coherent unit, use one phase. Splintering artificially is a defect.
   >   - If multiple valid phase orderings exist (e.g., two independent strands could go in either order), pick one and append the alternative to `spec/decisions.log` with one-line rationale.
   > - When multiple reasonable AC structurings exist, pick one; append losers to `spec/decisions.log` with one-line rationale each.
   > - Do not invent requirements the interview did not surface. Missing info → flag at top under `## Open questions`.
   >
   > Write `spec/spec.md` (use the absolute path). Attempt the Write tool; if denied, retry once; if still denied, include the full intended file content verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can write it. Never report success without either having written the file or having dumped its full content.
   >
   > Report back: path, 3-line summary of changes (or "initial draft"), open questions, and (if Write was denied) the full content dump.

4. **Verify the file exists.** After the sub-agent returns, check whether `spec/spec.md` was actually written. If not, extract the dumped content from the sub-agent's reply and write it directly using the Write tool in the parent session.

5. **User review.** Show the sub-agent's summary. Ask the user to open `spec/spec.md` and confirm, edit directly, or request revisions. Do not auto-accept.

6. **Update state.yaml:** `phase: seeded`, `last_command: /spec seed`, `last_command_at: <timestamp>`. `spec_sha` updates after user commits.

7. **Propose commit:**
   ```
   git add spec/spec.md spec/decisions.log spec/state.yaml
   git commit -m "spec: <initial draft | re-draft from interview v<n>>"
   ```

8. **Next step.** Tell user to run `/spec review` in a new conversation.
