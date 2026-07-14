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
- **Automatically** — during any step, when the user makes a non-obvious choice with rationale (ruling out scope, picking an interpretation, accepting/rejecting a persona critique with reason, choosing one AC structuring over alternatives). The calling step *proposes* the entry; nothing is written until the user explicitly confirms via the **Consent gate** below. There is no "log inline, don't interrupt flow" path — `decisions.log` fills up fast, and silent appends are how it becomes unreadable.

## Consent gate (all auto-invocations)

Every auto-invocation, including operational/structural markers like "Spec converged" or "Iteration closed", must traverse this gate. The gate is not optional and has no per-step exception list.

Before writing any entry from auto-invocation:

1. **Compose the proposed entry in working memory.** Fill in `Title`, `Decision`, `Rationale`, `Alternatives`, `Related`, `Context` per the calling step's context. Do **not** invent any field — see *Rationale and anti-rationalization* below; rationale and alternatives must come from what the user actually said.

2. **Present the proposed entry to the user.** Show the title, decision sentence, rationale, and any alternatives — one block per entry. Then ask one of the two forms below.

   - **Single entry** (one proposed log):

     > Propose to log this decision:
     > **<title>**
     > Decision: <one sentence>
     > Rationale: <one to three sentences>
     > (Alternatives: <list if any>)
     >
     > Log it? Reply `y` (log as-is), `n` (skip — don't log), or `edit` (revise before logging).

   - **Batched** (multiple proposed logs at a natural turn boundary, e.g., end of `/spec revise` Turn 2 or `/spec reconcile` Turn 3):

     > Propose to log the following decisions:
     >
     > 1. **<title-1>** — <decision>. Rationale: <rationale>.
     > 2. **<title-2>** — <decision>. Rationale: <rationale>.
     > 3. ...
     >
     > For each, reply `y` (log), `n` (skip), or `edit N` (revise entry N). You may reply `y all` or `n all` to apply one verdict to every entry.

3. **Honor the user's reply.**
   - `n` (or `n all`) — **do not write the entry**. Don't argue or re-propose later in the same step. The user judged it not worth logging.
   - `edit` — accept the user's revised title / decision / rationale / alternatives verbatim, then re-show and re-ask `y / n`. Do not re-edit the user's edit.
   - `y` — proceed to step 4.
   - Anything ambiguous → re-ask once with the same prompt. If still ambiguous, default to `n` (skip). Erring toward skipping a borderline entry is much cheaper than erring toward logging one — that's the whole reason this gate exists.

4. **Write the confirmed entries** per the entry format below. One-line user-facing mention per written entry: "Logged decision: DEC-NNN — <title>."

5. **Do not propose a mid-step commit.** The parent step's commit proposal already includes `spec/decisions.log`. If every proposed entry was skipped (`n all`), drop `spec/decisions.log` from the parent step's `git add` line — there is nothing to commit.

### Rationale and anti-rationalization

The skill must not invent reasons. This is a hard rule:

- **Allowed:** paraphrasing, tightening, re-ordering, or copying verbatim a reason the user actually surfaced in conversation. "Splitting along lifecycle felt cleaner" → "Lifecycle split chosen for testability" is fine if the user said the part about testability.
- **Not allowed:** retro-justifying a user choice with a reason the user never gave ("because it follows the project's existing convention" when convention was never mentioned), extrapolating from the shape of the artifact, or producing rationale text in a register the user didn't use. Plausible ≠ true. Confabulated rationale is worse than no rationale because it looks load-bearing later.
- **When user-supplied rationale is thin or missing:** before proposing the entry, ask the user. Examples: *"You picked option (b) — what made it the right call?"* or *"What's the underlying reason — convention, evidence, gut, or something else?"* Surface the gap; don't paper over it. If the choice is small and the user has nothing to say, that's a signal the entry probably shouldn't be logged at all — propose `n` by default.
- **Founder-said-so is an acceptable terminal rationale.** The product owner's bare prerogative is a complete answer requiring no further justification — e.g. `Rationale: product-owner decision (no further justification given).` Stop-and-ask remains correct when a load-bearing rationale is genuinely missing; *founder-said-so* is a valid **answer** to that ask, not a bypass of it. When the owner declines to justify a call, record owner-prerogative **truthfully** rather than pressing for a manufactured "why." Do not invent a plausible-sounding reason to avoid writing the prerogative form.
- **Same rule applies to `Alternatives considered`.** Only list alternatives the user actually weighed. Do not invent a third "for the diamond" if the user only considered two.

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
**Context:** <free-text — e.g., "during interview v2", "addressing ontologist concern #2", "during /spec reconcile">

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

When another step's dialogue produces a decision worth logging (the calling step decides what to propose; this protocol just defines how, and the **Consent gate** above is mandatory):

1. **Compose the proposed entry** with these fields populated:
   - **Title** — short title derived by the calling step from the decision context. The calling step may propose; the gate lets the user edit.
   - `Iteration:` — from `state.yaml`.
   - `Decision:` — supplied by the calling step. State the user's choice in one sentence — do not editorialize.
   - `Rationale:` — only the reason the user surfaced. If the user has not surfaced a reason and the entry is being seriously considered, the calling step must **ask the user for one before reaching the gate** (see *Rationale and anti-rationalization* above). If after asking the user declines further justification, use the founder-said-so form (`product-owner decision (no further justification given)`) and still propose the entry when the decision itself is worth logging. If the choice is small and neither a substantive reason nor an explicit prerogative call is present, do not propose the entry.
   - `Alternatives considered:` — only the alternatives the user actually weighed. Omit the line otherwise. Do not invent third options.
   - `Supersedes:` — typically omitted; the calling step may pass it explicitly when the cross-reference is unambiguous (rare).
   - `Related:` — supplied by the calling step when known. For example, `/spec reconcile` step 7f promotion entries set `Related:` to the AC IDs introduced/modified by the promotion. `/spec interview` triage entries that promote a deferred item set `Related:` to the deferred item's ID context (e.g., `Promoted from D-XXX`).
   - `Context:` — calling step provides one-line description (e.g., "during /spec reconcile (iteration 4)", "addressing ontologist concern #2 during /spec revise"). Note: do **not** prefix with "auto-logged" — every entry is gated and user-approved, so all entries are equally first-class.

2. **Traverse the Consent gate** above. Compute `DEC-NNN` (per the ID rule) only at the moment of writing — i.e., after the user has confirmed.

3. **Write confirmed entries; skip declined ones.** Emit one-line mention per written entry: "Logged decision: DEC-NNN — <title>." Don't propose a mid-step commit — the parent step's commit proposal will include `spec/decisions.log` (or drop it if every entry was skipped).
