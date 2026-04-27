# /spec decide — Decision log

Record a significant decision to `spec/decisions.log`.

## State machine

**Allowed from phases:** any
**Transitions to:** unchanged
**Re-run behavior:** always allowed.

## When this runs

- **Explicitly** — user runs `/spec decide` or `/spec decide "<text>"`.
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
## YYYY-MM-DD HH:MM — <short title>
**Decision:** <one sentence>
**Rationale:** <one to three sentences>
**Alternatives considered:** (omit line if none)
  - <alternative 1> — <why rejected>
  - <alternative 2> — <why rejected>
**Context:** <e.g., "during interview v2", "addressing ontologist concern #2">

---
```

## Protocol for explicit invocation

1. If the user provided text after `/spec decide`, parse as the decision sentence. Otherwise ask: "What's the decision, in one sentence?"
2. Ask for rationale. Ask for alternatives — apply **condensed diamond**: if fewer than two offered, prompt once for a third; if truly none, accept (not every decision has meaningful alternatives).
3. Append the entry; show it back.
4. Propose:
   ```
   git add spec/decisions.log
   git commit -m "decide: <short title>"
   ```
5. No state.yaml update (decisions don't change phase).

## Protocol for automatic invocation

Emit entry silently. One-line user-facing mention: "Logged decision: <title>." Don't propose a mid-step commit — the parent step's commit proposal will include `spec/decisions.log`.
