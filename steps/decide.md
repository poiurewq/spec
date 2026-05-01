# /spec decide — Decision log

Record a significant decision to `spec/decisions.log`. **Past-tense only** — what was already decided (positively, negatively-permanent, or meta). For future-tense items (feature requests, bugs, ideas not yet committed to any iteration), use `/spec defer` instead, which writes to `spec/deferred.md`.

## State machine

**Allowed from stages:** any
**Transitions to:** unchanged
**Re-run behavior:** always allowed.

## Arguments

- **Explicit invocation forms:**
  - `/spec decide` — fully interactive.
  - `/spec decide "<decision sentence>"` — one-shot decision text; rationale and alternatives still asked interactively.
  - `/spec decide --supersedes DEC-NNN "<decision sentence>"` — flags this entry as revising an earlier decision. Accepts `--supersedes DEC-NNN`, `--supersedes=DEC-NNN`. Validates that `DEC-NNN` exists in `spec/decisions.log`; if not, error and re-prompt.

## When this runs

- **Explicitly** — user runs one of the forms above.
- **Automatically** — during any step, when the user makes a non-obvious choice with rationale (ruling out scope, picking an interpretation, accepting/rejecting a persona critique with reason, choosing one AC structuring over alternatives). Log inline; don't interrupt flow.

## Entry format

**Always append — never insert.** Add the new entry at the end of the file. The file is strictly append-only; placing a new entry anywhere other than the end corrupts the chronological record and violates the cross-iteration audit trail.

Append to `spec/decisions.log`. If first write ever, create with header:
```
# Decision log

# Iteration 1
```

If the `iteration` in state.yaml has changed since the last entry (new iteration started), append a new `# Iteration <n>` header before the entry.

Entry format:

```
## DEC-NNN — YYYY-MM-DD HH:MM — <short title>
**Iteration:** <n>
**Decision:** <one sentence>
**Rationale:** <one to three sentences>
**Alternatives considered:** (omit line if none)
  - <alternative 1> — <why rejected>
  - <alternative 2> — <why rejected>
**Supersedes:** DEC-NNN  (omit line if not superseding a prior decision)
**Related:** <comma-separated AC IDs / invariant references>  (omit line if none)
**Context:** <free-text — e.g., "during interview v2", "addressing ontologist concern #2", "auto-logged during /spec reconcile">

---
```

**Field rules:**

- **ID (`DEC-NNN`)** — zero-padded to 3 digits. Compute next ID by scanning `spec/decisions.log` for lines matching `^## DEC-(\d+)`, taking the max, and incrementing. Default to `DEC-001` if none found. IDs are stable for the entry's lifetime; do not reuse IDs.
- **Iteration** — value of `iteration` in `state.yaml` at write time. If `state.yaml` does not exist (rare — pre-onboarding), use `0`.
- **Supersedes** — points at a single prior `DEC-NNN` that this decision revises. Forward-only pointer (we never modify the prior entry to record the reverse direction — that would violate append-only). Validate the target exists when writing; reject if not.
- **Related** — free-form list of `AC<id>` references and/or `Invariant <n>` references. Used as a backlink for searches like "why is AC2.3 the way it is?" Examples: `AC2.3`, `AC1, Invariant 4`, `Invariant 2`. Omit the line if no relations are obvious.
- **Context** — free-text, *what* triggered the decision (which step, which review concern, etc.). Distinct from Iteration (structured, always present) and from any cross-reference fields.

**Legacy entries.** Pre-existing `decisions.log` files (from before this format was introduced) may contain entries without `DEC-NNN` IDs. The skill does **not** retroactively edit them. ID computation scans only `^## DEC-` lines; legacy entries are ignored for ID purposes and stay as-is. Cross-references can only point at IDed entries.

## Protocol for explicit invocation

1. **Parse arguments.**
   - If `--supersedes DEC-NNN` was provided: validate the target exists in `spec/decisions.log`. If not, error: "DEC-NNN not found in decisions.log" and stop (or re-prompt if interactive).
   - If a decision sentence was supplied as the bare argument, hold it for step 2.

2. **Decision sentence.** If not supplied via argument, ask: "What's the decision, in one sentence?"

3. **Rationale.** Ask: "What's the rationale? (one to three sentences)"

4. **Alternatives.** Ask: "What alternatives did you consider, and why were they rejected?" Apply **condensed diamond**: if fewer than two offered, prompt once for a third; if truly none, accept (not every decision has meaningful alternatives — e.g., a `Spec converged` meta entry).

5. **Supersedes** (skip if `--supersedes` flag was provided). Ask: "Does this revise an earlier decision? Reply with a `DEC-NNN` ID, or `no`." Validate the ID exists if given.

6. **Related.** Ask: "Are there specific ACs or invariants this decision affects? List them (e.g., `AC2.3, Invariant 4`), or `none`."

7. **Compute next ID.** Scan `spec/decisions.log` per the ID rule above.

8. **Compose and append entry.** Use the format above. Read `iteration` from `state.yaml` for the `Iteration:` field. Set `Context:` to a brief description of the user's setting (or empty if the user didn't provide one).

9. **Show the entry back** to the user.

10. **Propose commit:**
    ```
    git add spec/decisions.log
    git commit -m "decide: <DEC-NNN — short title>"
    ```

11. No state.yaml update (decisions don't change stage).

## Protocol for automatic invocation

When another step's dialogue produces a decision worth logging (the calling step decides what to log; this protocol just defines how):

1. **Compute next ID** per the ID rule above.

2. **Compose entry** with these fields populated:
   - `DEC-NNN`, timestamp, short title — derived by the calling step from the decision context.
   - `Iteration:` — from `state.yaml`.
   - `Decision:`, `Rationale:` — supplied by the calling step.
   - `Alternatives considered:` — supplied if applicable, omitted otherwise.
   - `Supersedes:` — typically omitted in auto-invocation; the calling step may pass it explicitly when the cross-reference is unambiguous (rare).
   - `Related:` — supplied by the calling step when known. For example, `/spec reconcile` step 7f promotion entries set `Related:` to the AC IDs introduced/modified by the promotion. `/spec interview` triage entries that promote a deferred item set `Related:` to the deferred item's ID context (e.g., `Promoted from D-XXX`).
   - `Context:` — calling step provides one-line description (e.g., "auto-logged during /spec reconcile (iteration 4)", "addressing ontologist concern #2 during /spec revise").

3. **Append entry.** Show the user a one-line mention: "Logged decision: DEC-NNN — <title>." Don't propose a mid-step commit — the parent step's commit proposal will include `spec/decisions.log`.
