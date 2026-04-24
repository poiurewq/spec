# /spec help — User-facing help

Print the following to the user. If `spec/state.yaml` exists and parses, prepend a one-line status: `You are currently in: iteration <n>, mode <mode>, phase <phase>. Suggested next: /spec <command>.`

---

## `/spec` — guided spec development

A skill that walks you through building a high-quality software specification before writing code. Works for new projects (greenfield) and for iterating on existing ones.

### Why

Most AI-assisted development fails at the input stage. `/spec` gives you a disciplined loop to convert a vague idea into an MECE acceptance-criteria tree you can trust, surface assumptions before they cost you, and stop iterating at the right moment.

### Core loop

1. **Interview** — Socratic dialogue to clarify intent. Pass the self-rated clarity gate.
2. **Seed** — an Opus sub-agent drafts `spec/spec.md` from the interview transcript.
3. **Review** — three skeptical personas (Ontologist, Contrarian, Simplifier) critique in parallel.
4. **Revise** — you address each critique in conversation; the agent proposes the revision.
5. **Check** — declare convergence when two consecutive revisions changed only wording.
6. **Verify** — after implementation: audit code against spec; get a per-AC evidence report.
7. **Close** — resolve gaps, generate the takeaway, archive the iteration. Ready for iteration N+1.

### Subcommands

| Command | What it does |
|---|---|
| `/spec` | Report phase; suggest next command |
| `/spec interview` | Conduct the Socratic interview (optional `--iteration N` at init) |
| `/spec adopt` | Bootstrap `spec/` for an existing brownfield project with code + optional rough spec doc (optional `--iteration N`) |
| `/spec seed` | Draft or revise `spec.md` from interview |
| `/spec review` | Three-persona review |
| `/spec revise` | Incorporate critiques (three-turn) |
| `/spec check` | Test convergence |
| `/spec verify` | Audit code against spec |
| `/spec close` | Finalize iteration, generate takeaway |
| `/spec decide "<text>"` | Log a decision |
| `/spec help` | This help |

### Example workflow — greenfield

Each line is a **fresh conversation**.

```
/spec interview "Build a CLI tool that converts Markdown tables to CSV"
/spec seed
/spec review
/spec revise
/spec check                    # may loop back to /spec review if not converged
# -- implement the spec in a separate coding session --
/spec verify                   # re-run freely as implementation progresses
/spec close                    # resolves any remaining gaps, writes takeaway.md
```

### Example workflow — adopting an existing brownfield project

One-shot bootstrap, then the normal loop. The adoption iteration (iteration 1 by default, or set via `--iteration N`) is mostly about converting claims to verified facts via `/spec verify`.

```
/spec adopt                    # asks for rough spec doc path + optional focus paths; Explore sub-agent ingests
/spec interview                # resumes the pre-populated session file; ratify, correct, pass clarity gate
/spec seed                     # drafts spec.md from iteration template; ACs tagged [adopted] (claim) / [delta] / [regression]
/spec review
/spec revise
/spec check
/spec verify                   # the big moment — converts every [adopted] claim to PASS / GAP / UNCLEAR
/spec close                    # writes first takeaway.md; next iteration runs with mode: iteration
```

### Example workflow — iterating

Starts right after `/spec close` of a prior iteration:

```
/spec interview                # auto-detects iteration mode from takeaway.md
/spec seed                     # drafts delta-oriented spec.md
/spec review                   # personas apply their in-iteration addenda
/spec revise
/spec check
# -- implement the delta --
/spec verify
/spec close                    # writes new takeaway.md; old one archived
```

### Conventions

- **One conversation per step** — keeps context bounded.
- **You commit; the skill proposes messages.** You stay in control of git history.
- **`state.yaml` is skill-owned.** Inspect freely; don't hand-edit.
- **Clarity gate is self-rated.** The agent presents; you rate.
- **`/spec verify` is evidence, not verdict.** You judge pass/fail.

### Files under `./spec/`

- `spec.md` — current spec (the "Seed")
- `takeaway.md` — last-closed iteration's shipped reality (appears after first `/spec close`)
- `decisions.log` — cross-iteration decision history
- `state.yaml` — current phase (skill-managed)
- `archive/` — timestamped working files + per-iteration snapshots, all in one flat directory with `<date>-v<n>-<kind>.md` names
