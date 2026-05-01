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
| `/spec implement` | Orchestrates per-phase implementation: kickoff prompt for a fresh session, then per-phase Explore audit on re-run. Does **not** implement code itself |
| `/spec verify` | Explore subagent audits code against spec, renders evidence |
| `/spec reconcile` | Pull post-converge code drift back into the spec — bidirectional ingestion. Four-bucket triage (decision-only / minor / structural / major) routes phase regression accordingly |
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
                                                converged ◄──┐
                                                     │       │ /spec implement (per-phase loop;
                                                     │       │  appends to phases_implemented,
                                                     │       │  does not change phase)
                                                     │ ──────┘
                                                     │
                                                     │ /spec verify
                                                     ▼
                                                 verified ◄─── /spec verify (re-run)
                                                     │
                                                     │ (drift discovered? /spec reconcile —
                                                     │  buckets 1–2 stay in place; bucket 3
                                                     │  drops to revised; bucket 4 to in-review)
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
| `/spec implement` | `converged` · `verified` (re-audit) | unchanged (appends to `phases_implemented` on user confirmation) |
| `/spec verify` | `converged` · `verified` (re-run) | `verified` |
| `/spec reconcile` | `converged` · `verified` | unchanged · `revised` · `in-review` (depends on highest-severity bucket used) |
| `/spec close` | `verified` | `closed` (**refused** if already `closed`) |
| `/spec decide` | any | unchanged |
| `/spec help` | any | unchanged |

Every step file includes a `## State machine` section at the top declaring these constraints and what it writes back to `state.yaml`.

## Implementation: orchestrated, not performed

The skill does **not** implement code itself. Implementation happens in fresh conversations the user opens — the principle "separate conversations for generative steps" applies, and implementation is the lowest-ambiguity step in the loop (the spec's `[delta]`-tagged ACs already enumerate every deliverable). What the skill *does* provide, via `/spec implement`, is **orchestration**: walking through the spec's `## Implementation phases` one phase at a time, with a per-phase Explore audit between phases as an early-warning evidence layer before the final full-spec `/spec verify`.

**Recipe (with phases declared):**

1. Run `/spec implement` after `converged`. It identifies the next un-confirmed phase, shows you its ACs, and gives you a copy-pasteable kickoff prompt.
2. Open a fresh conversation, paste the kickoff prompt, implement just that phase. The prompt restricts scope to the current phase's ACs and reminds the agent to preserve `[adopted]` claims, `## Invariants`, and `## Provisional invariants`.
3. Return to the `/spec implement` session and re-run the command. It runs an Explore audit over only that phase's ACs (plus any plausibly-touched invariants), presents per-AC PASS/GAP/UNCLEAR evidence, and asks you to confirm.
4. On confirm, the phase is appended to `phases_implemented` and the skill proposes a per-phase commit. Repeat for the next phase.
5. When all phases are confirmed, run `/spec verify` for the full-spec audit, then `/spec close`.

**Recipe (no phases declared, or single-phase iteration):**

1. After `converged`, open a fresh conversation. Prompt:
   > Read `spec/spec.md`. Implement every `[delta]` AC. Treat `[adopted]` ACs, `## Invariants`, and `## Provisional invariants` as constraints to preserve — do not regress them. Items under `## Test-pending` are deferred to a future iteration; do not pick them up now.
2. When deltas are done, open a new conversation and run `/spec verify`.

Why orchestration earns its keep even though implementation itself is low-ambiguity: phases are the natural commit boundary, "which phase is next" is genuinely state worth tracking, and per-phase audits catch regressions earlier and at smaller scope than waiting for a single end-of-iteration `/spec verify`. The rigor still lives on the verification side — the implementation step is a thin wrapper around "give the user the right prompt, then audit what they did."

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
phases_implemented: [<int>]            # phase numbers user confirmed via /spec implement; default []. Reset to [] on /spec close.
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
