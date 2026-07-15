# /spec seed — Draft or revise the Seed

Summarize the latest interview transcript into `spec/spec.md`. A generative step with real judgment calls (AC structuring, phase ordering) — run **in-conversation in a fresh session**, not as a sub-agent, so the user can steer at forks. A fresh session still reads only the interview file and template from disk, so resume-ability and the transcript-completeness forcing function are preserved.

## State machine

**Allowed from stages:** `interviewing` (after gate passes) · `seeded` (re-draft)
**Transitions to:** `seeded`
**Re-run behavior:** Allowed. Warn user that current `spec.md` will be replaced.

Read state.yaml first; validate stage. Write state.yaml on completion.

## Protocol

1. **Check prerequisites.** Read state.yaml. Confirm `latest_interview` points to a file that exists. If not, tell the user to run `/spec interview` first and stop.

2. **Pick template based on mode** (from state.yaml). Paths are relative to the skill base directory (announced at skill invocation):
   - `mode: greenfield` → `templates/spec.md`
   - `mode: iteration` → `templates/spec-iteration.md`
   - `mode: adopted` → `templates/spec-iteration.md` (same template; tagging rules differ — see step 3)

   Resolve to an absolute path by prefixing the skill base directory.

3. **Draft the spec (you, in this conversation).** Working directly in this fresh session, produce the spec per the requirements below. Because this is in-conversation, at a genuine fork — multiple reasonable AC structurings, or multiple valid phase orderings — surface the alternatives to the user and let them pick rather than silently choosing; fall back to pick-and-log only if the user defers the choice back to you. The requirements:

   > Read the interview file at `spec/archive/<latest_interview-from-state.yaml>`. It ends with a `## Seed handoff` section — the interviewer's distilled intent (axis summary, emphasis, ruled-out interpretations, open tensions). Weight it heavily; route its "Open tensions" into `## Open questions`, and do not re-introduce any "Ruled-out interpretations". If `spec/spec.md` already exists, also read it. If mode is iteration and `spec/takeaway.md` exists, also read it (as "prior iteration shipped state").
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
   >   - **`[build-change-todo]` items from Part A of the adoption interview** represent drift the user decided to resolve by changing *code* (not spec). Translate each into an AC tagged `[delta]` describing the spec's retained claim as the target behavior, and note under `## Change delta → Modified` that the code currently diverges and must be updated. Do not drop these — they are the reason "Change delta may be empty" does not apply to this adoption iteration.
   >   - **`[iteration-scope]` items from Part A** represent resolutions the user affirmed only for this iteration (expected to change in a future iteration — e.g., "no `SettingsViewModel` yet; will extract in a future iteration"). In the spec body, describe the *current* shape as shipped reality — do **not** enshrine it as a permanent invariant. Also ensure each `[iteration-scope]` item has a corresponding entry in `spec/deferred.md` (the interview step should have appended these via `/spec defer`'s automatic-invocation pattern with source `via /spec interview Part A (iteration <n>)`; if any are missing, add them now — if `spec/deferred.md` does not exist, create it first with the header: `# Deferred items\n\n_Backlog of feature requests, bug reports, and ideas not yet committed to any iteration. Triaged at each iteration's interview._`). Do **not** route these to `decisions.md` — they are future-tense items.
   >   - **`[invariant-provisional]` items from Part B** represent invariants the user committed to as invariants but flagged as revisable under evidence-driven pressure (e.g., "theme-switch checklist is invariant, but real-world bug reports may motivate refinement"). List these in a **separate `## Provisional invariants` section** (placed immediately after `## Invariants` — do **not** mix them into `## Invariants`, which must remain a scannable list of hard guarantees). Each item carries an inline `Trigger: <brief description>` clause naming what evidence would motivate revision. Do **not** silently downgrade to regular invariants (loses the user's explicit epistemic caveat); do **not** promote to `[iteration-scope]` deferrals (there is no scheduled change). If there are no such items, omit the section entirely.
   > - **Implementation phases.** After the AC tree, fill the `## Implementation phases` section. Group leaf ACs into ordered phases such that **phase N is fully implementable without any work from phase N+1 or later** — dependencies flow only backward. The implementer must be able to work strictly in numerical order without ever needing to reach into a later phase to make an earlier one work.
   >   - **Greenfield mode:** every leaf AC is phased; each leaf appears in exactly one phase.
   >   - **Iteration / adopted modes:** phase only `[delta]` ACs (including `[delta]` ACs derived from `[build-change-todo]` items). Do **not** phase `[regression]` or `[adopted]` ACs — those are verification targets, not implementation work. If the iteration has no `[delta]` ACs, write "None — this iteration introduces no new implementation work." under the section.
   >   - For each phase, write a one-line `**Delivers:**` (what works at end of phase) and either `**Unblocks:**` (for the foundation phase) or `**Depends on:**` (citing the earlier phase and which specific piece). **Always name the referenced phase by purpose, not just number** — e.g., `**Unblocks:** Phase 12 (submission-readiness assembly)` or `**Depends on:** Phase 1 (research-gate decisions)`. Bare-number references (`**Unblocks:** Phase 12`) silently rot when phase numbering shifts during revision; the purpose phrase makes stale references self-evident on read.
   >   - **Identify parallelizable phases explicitly.** The numbering is a topological order, not a mandate to work serially. Two phases are parallelizable when neither (transitively) depends on the other — i.e., they share no dependency edge. For every phase that has at least one such sibling, add a `**Parallelizable with:**` line naming those phases by number and purpose (e.g., `**Parallelizable with:** Phase 4 (export pipeline)`). Omit the line for phases that must run alone (everything before them is a transitive dependency). The `**Depends on:**` edges remain the source of truth; `**Parallelizable with:**` is the read-off that makes independence visible to the implementer instead of leaving it to be re-derived.
   >   - Prefer fewer, larger phases over many small ones. If the work is one coherent unit, use one phase. Splintering artificially is a defect — do **not** manufacture parallelism by splitting a coherent unit.
   >   - If two independent strands exist, do **not** arbitrarily serialize them and bury the alternative in `decisions.md`. Give each its own phase with no `**Depends on:**` edge between them and cross-link them via `**Parallelizable with:**`. Only fall back to picking-and-logging when the orderings are genuine alternatives over the *same* work (not independent strands).
   > - When multiple reasonable AC structurings exist, pick one. If you logged any losers, propose entries via `steps/decide.md`'s auto-invocation protocol (mandatory Consent gate) — typically batched at the end of drafting, one proposed entry per discarded structuring with the user's stated reason for picking the winner. Do not invent reasons; if the user picked the winner without articulating why, ask them ("why this structuring over <other>?") before proposing the entry. Most seed-time forks are minor and the user will skip the gate; that's fine — there is no requirement that every discarded alternative gets logged.
   > - **Motivation, Goal, Constraints, and Success criteria are user-sourced.** Use the user's own framing from the interview's `## Seed handoff` and transcript. Do not synthesize a Motivation from inferred convention or from the shape of the codebase — if the Seed handoff is thin on motivation and the mode requires one (iteration / adopted), surface this as an `## Open questions` item rather than inventing a plausible-sounding reason. Same for Constraints rationale and for any "why" the spec asserts on the user's behalf.
   > - **No forward iteration-number projection.** Do not assign deferred work, runway, or "reserved for later" notes to a concrete future `iter-N`. Write **"a future iteration"** and, where useful, a temporal-distance qualifier (**near-term / medium-term / long-term**). References to the **current** iteration and to **past / baseline** iterations are factual and remain as-is. Future iteration numbers are guesses that ossify into false commitments; the deferral is real, the iteration assignment is not.
   > - Do not invent requirements the interview did not surface. Missing info → flag at top under `## Open questions`.
   >
   > Write `spec/spec.md` (use the absolute path) with the Write tool. If Write is denied, ask the user to grant it — do not silently drop the draft.

4. **User review.** State the path, a 3-line summary of changes (or "initial draft"), and any open questions. Ask the user to open `spec/spec.md` and confirm, edit directly, or request revisions. Do not auto-accept.

5. **Update state.yaml:** `stage: seeded`, `last_command: /spec seed`, `last_command_at: <timestamp>`. `spec_sha` updates after user commits.

6. **Propose commit:**
   ```
   git add spec/spec.md spec/decisions.md spec/state.yaml
   git commit -m "spec: <initial draft | re-draft from interview v<n>>"
   ```

7. **Next step.** Tell user to run `/spec review` in a new conversation.
