# /spec check — Convergence stop rule

Two paths to convergence:

1. **Mechanical** — two consecutive revisions changed only wording, not structure.
2. **Product-owner prerogative** — the user explicitly declares the spec converged regardless of the mechanical test.

Both paths produce the same stage transition (`revised` → `converged`). The mechanical path is the default discipline; prerogative is always available and must be an explicit user act (not inferred from silence or impatience).

## State machine

**Allowed from stages:** `revised`
**Transitions to:** `converged` (mechanical pass, or user explicitly declares via product-owner prerogative) OR stays `revised` (loop continues)
**Re-run behavior:** Free — read/judge only, no artifact writes until convergence declared.

Read state.yaml first; validate stage. Write state.yaml if convergence is declared.

## Protocol

1. Read state.yaml; validate stage.

2. Find commits touching `spec/spec.md`:
   ```
   git log --oneline -- spec/spec.md
   ```

3. **Thin history (fewer than two commits, or no `spec.md` commits yet).**
   - Report that the mechanical two-revision test cannot run yet.
   - Do **not** stop as a hard refusal. State that product-owner prerogative can still declare convergence, and that this requires an **explicit** declaration (see step 8).
   - If the user has not already declared, offer the prerogative option once (wording below). Wait for their response. Do not auto-converge.
   - On explicit prerogative → step 9 with path `prerogative`. On decline or silence about converging → stay `revised`; suggest `/spec revise` (or seed/review as appropriate) until there is something to check.

4. **With ≥2 commits:** Show diff between the two latest revision-bearing commits. Use the actual SHAs — don't assume `HEAD~1 HEAD` (other files may intervene).

5. Ask exactly two questions about the **latest** revision:
   - "Did this revision change any *structural* element — goals, constraints, success criteria, AC tree topology (in iteration mode: also Invariants or Change delta)?"
   - "Or did it change *only* wording, examples, or phrasing?"

6. **Structural (latest) →** mechanical test fails for this pair.
   - Recommend `/spec review` in a new conversation for another loop.
   - **Warn** (do not block): converging now would skip further review of structural change.
   - Offer product-owner prerogative once if the user has not already declared (step 8). Stage stays `revised` until they either loop or explicitly declare.

7. **Wording-only (latest) →** check the revision before that (same two questions).
   - Also wording-only → **mechanical convergence**. Go to step 9 with path `mechanical`.
   - Structural (previous) → mechanical test fails. Recommend one more review/revise cycle, then re-check. **Warn** (do not block) that only one wording-only revision is on the stack. Offer product-owner prerogative once if not already declared (step 8). Stage stays `revised` until loop or explicit declare.

8. **Product-owner prerogative (always available).**
   - **When:** At any point in this step — thin history, structural latest, only-one-wording-only, or even mid-discussion after the mechanical path would have passed and the user still wants to frame it as an owner call. Prerogative is never blocked by commit count or structural status.
   - **Explicitness:** The user must clearly declare intent to converge *by prerogative / as owner / despite the mechanical result* (or equivalent plain language: "converge anyway," "I'm done — ship it," "owner call: converged," etc.). Do **not** treat vague "looks fine," "LGTM," or answering the wording questions alone as prerogative. Do **not** invent a prerogative declaration the user did not make.
   - **Offering:** If the mechanical path has failed or cannot run, and the user has not mentioned prerogative, the agent **may offer** once, e.g.:
     > The mechanical stop rule (two consecutive wording-only revisions) is not met. You can run another review/revise loop, or declare convergence now by **product-owner prerogative** if you are ready to implement. Which do you want?
     Do not nag; one clear offer is enough. If they already know and decline, leave stage `revised`.
   - **Structural override:** When the latest (or previous) revision was structural, include a one-line warning in the offer or in the confirmation of their declaration — review surface may still be hot — then proceed if they still declare.
   - On valid explicit declaration → step 9 with path `prerogative`.

9. **On convergence** (either path):
   - Propose a `decisions.log` entry via `steps/decide.md`'s auto-invocation protocol (mandatory Consent gate):
     - title: `Spec converged`
     - decision: `Converged at <short-SHA> on <YYYY-MM-DD>.` (use the tip short-SHA of `spec/spec.md` if available; if no commit yet, say `Converged (uncommitted / no spec.md SHA yet) on <YYYY-MM-DD>.`)
     - rationale — pick exactly one path form (true operational reason; no embellishment about why convergence matters):
       - **mechanical:** `Two consecutive wording-only revisions per /spec check.`
       - **prerogative:** `Product-owner prerogative — declared converged via /spec check.` When useful, append a short factual clause about what was overridden, e.g. ` (latest revision structural)`, ` (only one wording-only revision)`, or ` (fewer than two spec.md revisions)`. Do not invent extra motivation.
     - Alternatives: omit.
   - The user may decline the gate; if so, the stage still transitions to `converged` — only the log entry is skipped, and drop `spec/decisions.log` from the proposed `git add` line below.
   - Update state.yaml: `stage: converged`, `last_command: /spec check`, `last_command_at`.
   - Propose (drop `spec/decisions.log` from the `git add` if the user skipped the gate):
     ```
     git add spec/decisions.log spec/state.yaml
     git commit -m "spec: converged at <short-SHA>"
     ```
   - Tell the user: "Spec ready for implementation. After building, run `/spec verify` here to audit code against spec."
