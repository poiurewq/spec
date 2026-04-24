# /spec — Specification Development Skill

A human-driven, agent-assisted spec-development workflow: Socratic interview, MECE acceptance criteria, three parallel skeptical review personas, convergence-by-judgment, evidence-based post-implementation audit, and explicit iteration closure with takeaway generation. Supports three modes: **greenfield** (new project), **iteration** (continuing a prior closed iteration), and **adopted** (existing brownfield project with code and optionally a rough spec doc being brought under the skill's care for the first time).

## Purpose

Force discipline at the input stage of software projects: surface assumptions, enumerate alternatives at every decision, catch drift via rotating review personas, and stop iterating when the spec has stabilized. Produce a trustworthy specification before writing code.

## Architecture

- **Single entry point:** `/spec <subcommand>` — router in `SKILL.md`
- **Artifacts under `./spec/`** in the user's project directory
- **State tracked in `./spec/state.yaml`** — single source of truth for phase, iteration, mode
- **Sub-agents pinned to Opus** for seed drafting and persona reviews
- **`Explore` subagent** for `/spec verify` codebase reading
- **Plain text + git** — no databases, no hidden state

## File layout (user's cwd)

```
spec/
├── spec.md                              # current iteration target
├── takeaway.md                          # appears after first /spec close
├── decisions.log                        # cross-iteration, append-only
├── state.yaml                           # phase machine state (skill-writes only)
└── archive/                             # flat, v-prefixed; working + snapshots
    ├── v<NNN>-YYYY-MM-DD-HHMM-interview.md
    ├── v<NNN>-YYYY-MM-DD-HHMM-<persona>.md
    ├── v<NNN>-YYYY-MM-DD-HHMM-verify.md
    ├── v<NNN>-YYYY-MM-DD-HHMM-spec.md   # snapshot at close
    └── v<NNN>-YYYY-MM-DD-HHMM-takeaway.md  # snapshot at close
```

## Subcommands

| Command | Purpose |
|---|---|
| `/spec` | Report current phase; suggest next command |
| `/spec interview` | Socratic interview; clarity-gate before seed. Optional `--iteration N` on init |
| `/spec adopt` | Bootstrap `spec/` for an existing brownfield project (with optional rough spec doc). Lands in `interviewing` with pre-populated context. Optional `--iteration N` |
| `/spec seed` | Draft or revise `spec.md` from interview |
| `/spec review` | Three parallel Opus personas critique the spec |
| `/spec revise` | Three-turn: summarize → user addresses → propose revision |
| `/spec check` | Convergence test (two consecutive wording-only revisions) |
| `/spec verify` | Explore subagent audits code against spec, renders evidence |
| `/spec close` | Finalize iteration: resolve gaps, generate takeaway, archive |
| `/spec decide` | Log a decision (explicit or auto-triggered mid-step) |
| `/spec help` | Print user-facing help |

## State machine

Six phases per iteration, linear except for the review↔revise↔check loop:

```
  [no state.yaml]
        │
        ├── /spec interview ──────────────┐  (greenfield)
        └── /spec adopt ──────────────────┤  (brownfield — pre-populates session file via Explore + rough-spec ingestion)
                                          ▼
   interviewing ─── (gate passes; /spec seed) ───► seeded
                                                     │
                                                     │ /spec review
                                                     ▼
                                                 in-review ◄─── /spec review (re-run warn)
                                                     │
                                                     │ /spec revise
                                                     ▼
                                                  revised ─── /spec review ──► in-review
                                                     │
                                                     │ /spec check
                                              (not converged: stays revised)
                                              (converged)
                                                     ▼
                                                converged
                                                     │
                                                     │ /spec verify
                                                     ▼
                                                 verified ◄─── /spec verify (re-run)
                                                     │
                                                     │ /spec close (gaps resolved)
                                                     ▼
                                                  closed
                                                     │
                                                     │ /spec interview
                                                     ▼
                                           interviewing (iter N+1)
```

### Command preconditions + transitions

| Command | Allowed from phase(s) | Transitions to |
|---|---|---|
| `/spec interview` | (no state) · `closed` · `interviewing` (resume) | `interviewing` |
| `/spec adopt` | (no state) **only** | `interviewing` (mode `adopted`) |
| `/spec seed` | `interviewing` (post-gate) · `seeded` (re-draft) | `seeded` |
| `/spec review` | `seeded` · `revised` · `in-review` (re-run warn) | `in-review` |
| `/spec revise` | `in-review` | `revised` |
| `/spec check` | `revised` | `converged` or stays `revised` |
| `/spec verify` | `converged` · `verified` (re-run) | `verified` |
| `/spec close` | `verified` | `closed` (**refused** if already `closed`) |
| `/spec decide` | any | unchanged |
| `/spec help` | any | unchanged |

Every step file includes a `## State machine` section at the top declaring these constraints and what it writes back to `state.yaml`.

## Implementation is not a skill command

Notice the gap in the state diagram between `revised` / `converged` and `verified`: that's where you actually build the thing. There is deliberately no `/spec implement` command. The skill's value is concentrated in the ambiguity-rich steps — interviewing, reviewing, revising, verifying, closing — where a structured process beats ad-hoc conversation. Implementation is the opposite: the lowest-ambiguity step in the loop, because the spec's `## Change delta` + `[delta]`-tagged ACs already enumerate every deliverable. Wrapping it in skill ceremony would add overhead without adding judgment, and the skill's principle of "separate conversations for generative steps" argues for implementation being its own fresh session anyway.

**Recipe for the implementation session:**

1. Commit the current `spec/` state. Open a fresh conversation in the same repo.
2. Prompt the agent along these lines:
   > Read `spec/spec.md`. Implement every `[delta]` AC. Treat `[adopted]` ACs, `## Invariants`, and `## Provisional invariants` as constraints to preserve — do not regress them. Items under `## Test-pending` are deferred to a future iteration; do not pick them up now.
3. The agent works through the `[delta]` ACs; each one is independently testable by construction.
4. When all deltas are done, open a new conversation and run `/spec verify` to collect evidence that both the `[delta]` deliverables landed and the `[adopted]`/invariant constraints still hold.

Why this works: the spec is already the implementation TODO list. AC numbers give commits a natural reference scheme, and `/spec verify` is the structured audit that catches any regression the implementation session introduced — so the rigor lives on the *verification* side rather than the *implementation* side. If you find yourself wanting more structure during implementation (e.g., invariant-preservation enforcement per-AC), that's a signal the spec is under-specified, not that the skill needs a new command.

## state.yaml schema

```yaml
schema_version: 1
iteration: <int>                # 1 by default at init; user may override via --iteration N on /spec interview or /spec adopt; otherwise increments only on post-close interview
mode: greenfield | iteration | adopted
phase: <phase-name>
started_at: <ISO timestamp>     # when current iteration began
last_command: <string>
last_command_at: <ISO timestamp>
spec_sha: <git SHA | null>      # last commit touching spec/spec.md
latest_interview: <filename | null>
latest_review_stamp: <prefix | null>   # e.g., "v002-2026-04-22-1100"
latest_verify: <filename | null>
```

Notes:
- **Skill is the sole writer.** Users may read for inspection; do not hand-edit.
- **If missing or malformed:** `/spec` bare offers to auto-reconstruct from archive/ filenames + `spec.md`/`takeaway.md` presence. Requires user confirmation before writing fresh state.yaml.
- **No gap tracking in state.** Accepted GAP rationales live permanently in `takeaway.md`; state.yaml only drives phase transitions.

## Design philosophy

- **Clarity gate is self-rated**, never LLM-scored.
- **Convergence is user-judged.** No embedding thresholds.
- **`/spec verify` renders evidence, not verdict.** The user decides pass/fail from the per-AC report.
- **MECE enforced at two gates:** seed drafting and revise Turn 3.
- **Condensed diamond** (≥3 alternatives at non-trivial decisions) is a cross-cutting norm, applied in interview, seed, and revise.
- **One conversation per step.** Context stays bounded; each session serves one purpose.

## Distilled from Ouroboros

The core loop is adapted from the Ouroboros AI workflow engine, stripped of: event-sourced SQLite, PAL Router cost tiers, Double Diamond four-phase execution, autonomous Ralph loops, and LLM-based ambiguity/convergence scoring. What remains is the discipline: Socratic input gating, MECE decomposition, rotating skeptical reviewers, human-judged convergence, and evidence-based implementation verification.
