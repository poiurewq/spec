# Clarity gate — self-rated checklist

Rate each item yes or no. **Do not let the agent rate for you.** If any item is "no," one more focused interview question is warranted on that axis.

## Core items (both modes)

1. **Goal.** Can I state what this project is, in one sentence, without hedging?
2. **Constraints.** Have I listed the hard limits (tech, time, cost, compatibility, regulatory) that will shape the solution?
3. **Success criteria.** Do I know, measurably, how I will tell whether this worked?
4. **Scope boundary.** Can I name at least two things that are explicitly *not* in scope?
5. **Root-vs-symptom.** Am I solving a root need, not a symptom — and can I articulate why in one sentence?

## Iteration-mode items (additional, only when `mode: iteration`)

6. **Delta clarity.** Can I state, in one sentence, what is changing in this iteration?
7. **Invariants.** Do I know what must *not* break? Can I name at least two existing behaviors that must be preserved?

---

- **All yes** → proceed to `/spec seed` in a new conversation.
- **Any no** → one more targeted interview round on that axis, then re-present this gate.
