# /spec check — Convergence stop rule

Test: two consecutive revisions changed only wording, not structure.

## State machine

**Allowed from stages:** `revised`
**Transitions to:** `converged` (if user declares converged) OR stays `revised` (loop continues)
**Re-run behavior:** Free — read/judge only, no artifact writes until convergence declared.

Read state.yaml first; validate stage. Write state.yaml if convergence is declared.

## Protocol

1. Read state.yaml; validate stage.

2. Find commits touching `spec/spec.md`:
   ```
   git log --oneline -- spec/spec.md
   ```
   If fewer than two exist, tell the user it's too early to check and stop.

3. Show diff between the two latest revision-bearing commits. Use the actual SHAs — don't assume `HEAD~1 HEAD` (other files may intervene).

4. Ask exactly two questions:
   - "Did this revision change any *structural* element — goals, constraints, success criteria, AC tree topology (in iteration mode: also Invariants or Change delta)?"
   - "Or did it change *only* wording, examples, or phrasing?"

5. **Structural →** not converged. Recommend `/spec review` in a new conversation. Stage stays `revised`. No state.yaml update needed.

6. **Wording-only →** check the revision before that (same two questions).
   - Also wording-only → **convergence declared**. Go to step 7.
   - Structural → one more review/revise cycle, then re-check. Stage stays `revised`.

7. **On convergence:**
   - Append decisions.log entry via `steps/decide.md`: title `Spec converged`, decision `Converged at <short-SHA> on <YYYY-MM-DD>`, rationale `Two consecutive wording-only revisions`.
   - Update state.yaml: `stage: converged`, `last_command: /spec check`, `last_command_at`.
   - Propose:
     ```
     git add spec/decisions.log spec/state.yaml
     git commit -m "spec: converged at <short-SHA>"
     ```
   - Tell the user: "Spec ready for implementation. After building, run `/spec verify` here to audit code against spec."
