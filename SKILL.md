---
name: spec
description: Guide the user through a deliberate, iterative specification-development workflow for software projects (greenfield, iteration, or adoption of an existing brownfield project). Activates on /spec and its subcommands (/spec interview, /spec adopt, /spec seed, /spec review, /spec revise, /spec check, /spec implement, /spec verify, /spec close, /spec decide, /spec help). Maintains artifacts under ./spec/ with an explicit state machine in ./spec/state.yaml.
---

# /spec — Specification development skill

Human-driven, agent-assisted spec-development: Socratic interview gated by a self-rated clarity checklist, MECE acceptance criteria, condensed diverge/converge discipline, three rotating skeptical review personas, human-judged convergence, evidence-based post-implementation audit, and explicit iteration closure with takeaway generation. Greenfield, iteration, and adoption (existing brownfield project with a rough pre-existing spec) modes all supported.

**See `README.md` in the skill directory for the full state-machine reference and design rationale.**

## Global principles

1. **Self-rated clarity gate.** Never auto-score ambiguity. Present `templates/clarity-gate.md`; user rates.
2. **MECE acceptance criteria.** Enforce at seed drafting and after every revision.
3. **Condensed diamond.** At every non-trivial decision, enumerate ≥3 alternatives before committing. Losers → `decisions.log`.
4. **Plain text + git.** All artifacts under `./spec/`. User commits; skill proposes messages.
5. **Separate conversations for generative steps.** Interview, revise, and close deserve fresh sessions. Seed drafting and persona reviews are bounded sub-agents.
6. **Sub-agent model is Sonnet.** Pass `model: "sonnet"` in every Agent invocation unless a specific step explicitly calls for a different model. `/spec verify` and `/spec adopt` additionally use `subagent_type: "Explore"`.
7. **Sub-agents that write files must have a fallback.** Every sub-agent prompt that instructs the agent to write to a path under `spec/` must include the instruction: *"Attempt the Write tool for the target path. If Write is denied, retry once; if still denied, include the full intended file content verbatim in your final reply inside a fenced code block labeled with the target absolute path, so the parent session can write it. Never report success without either having written the file or having dumped its full content."* The parent session must then check: if the file does not exist after the sub-agent returns, extract the dumped content and write it directly.
8. **Evidence, not verdict.** `/spec verify` gathers evidence; the user judges pass/fail.
9. **State is explicit.** `spec/state.yaml` is the single source of truth for phase, iteration number, and mode. Read at the start of every subcommand; write at the end.

## Routing

| Input | Action |
|---|---|
| `/spec` (bare) | Detect state and report (procedure below) |
| `/spec interview` (optional `--iteration N`) | Read `steps/interview.md` |
| `/spec adopt` (optional `--iteration N`) | Read `steps/adopt.md` |
| `/spec seed` | Read `steps/seed.md` |
| `/spec review` | Read `steps/review.md` |
| `/spec revise` | Read `steps/revise.md` |
| `/spec check` | Read `steps/check.md` |
| `/spec implement` (optional `<phase-N>`) | Read `steps/implement.md` |
| `/spec verify` | Read `steps/verify.md` |
| `/spec close` | Read `steps/close.md` |
| `/spec decide` or `/spec decide "<text>"` | Read `steps/decide.md` |
| `/spec help` | Read `steps/help.md` |

## File layout (in user's cwd)

```
spec/
├── spec.md                              # current iteration target
├── takeaway.md                          # appears after first /spec close
├── decisions.log                        # cross-iteration, append-only
├── state.yaml                           # phase machine state (skill-writes only)
└── archive/                             # flat, v-prefixed; working + snapshots
    ├── v<NNN>-YYYY-MM-DD-HHMM-interview.md
    ├── v<NNN>-YYYY-MM-DD-HHMM-<persona>.md   # ontologist / contrarian / simplifier
    ├── v<NNN>-YYYY-MM-DD-HHMM-verify.md
    ├── v<NNN>-YYYY-MM-DD-HHMM-spec.md    # snapshot written at close
    └── v<NNN>-YYYY-MM-DD-HHMM-takeaway.md  # snapshot written at close
```

Create `spec/` and `spec/archive/` on first use.

## state.yaml schema

```yaml
schema_version: 1
iteration: <int>                 # 1 by default at initialization; user may override with --iteration N via /spec interview or /spec adopt; otherwise increments on post-close interview
mode: greenfield | iteration | adopted
phase: interviewing | seeded | in-review | revised | converged | verified | closed
started_at: <ISO timestamp>
last_command: <command string>
last_command_at: <ISO timestamp>
spec_sha: <git SHA | null>       # last commit touching spec/spec.md
latest_interview: <archive filename | null>
latest_review_stamp: <prefix | null>     # e.g., "v002-2026-04-22-1100"
latest_verify: <archive filename | null>
phases_implemented: [<int>]              # phase numbers user confirmed via /spec implement; default []. Cleared on /spec close (next iteration restarts).
```

**Skill is the sole writer.** Accepted GAP rationales from `/spec close` live in `takeaway.md`, not here.

## `/spec` bare — state detection

1. **If `spec/state.yaml` exists and parses:**
   - Report: `iteration <n>, mode <mode>, phase <phase>`.
   - Suggest next command per the phase mapping below.

2. **If `spec/state.yaml` is missing:**
   - **Empty / missing `spec/`** → suggest `/spec interview` (greenfield) for a new project, or `/spec adopt` if the user has an existing codebase (and optionally a rough spec doc) they want to bring under the skill's care.
   - **`spec/` exists with artifacts** → attempt auto-reconstruction:
     a. Scan `spec/archive/` for the highest `v<n>` prefix seen.
     b. Check presence of `spec.md`, `takeaway.md`, latest verify report.
     c. Infer `phase`, `iteration`, `mode` from what's present.
     d. Present the inferred state to the user in plain text and ask: *"Does this match your mental model? If yes, I'll write a fresh state.yaml."*
     e. On user confirmation, write `state.yaml` and proceed. On decline, stop and let the user reconcile manually.

3. **Phase → next command mapping:**

| Phase | Suggested next |
|---|---|
| `interviewing` | Continue `/spec interview` (resume) |
| `seeded` | `/spec review` |
| `in-review` | `/spec revise` |
| `revised` | `/spec check` (or `/spec review` for another loop). |
| `converged` | `/spec implement` to walk the `## Implementation phases` one phase at a time (each phase is implemented in a fresh session; the skill orchestrates kickoff and per-phase audit). When all phases are confirmed, run `/spec verify` for the full-spec audit. If the spec has no phases declared, implement freely in a fresh session and run `/spec verify` directly. |
| `verified` | `/spec close` (or `/spec implement` to re-audit a phase, harmless). |
| `closed` | `/spec interview` (start next iteration) |

## Mode

- **`greenfield`** — no prior spec. First iteration. Interview opens with "what are you trying to build?" Seed uses `templates/spec.md`.
- **`iteration`** — a prior iteration has closed through this skill. Interview opens with context ingestion (prior `takeaway.md`). Seed uses `templates/spec-iteration.md` with ACs tagged `[delta]` / `[regression]`.
- **`adopted`** — existing brownfield project being brought under the skill's care for the first time via `/spec adopt`. Applies only to the adoption iteration; subsequent iterations use `mode: iteration`. Interview resumes a pre-populated session file (synthesized `[from-code]` / `[from-rough-spec]` context). Seed uses `templates/spec-iteration.md` with ACs additionally tagged `[adopted]` (unverified claim about existing behavior — distinct from `[regression]`, which connotes verified existing behavior to preserve).

First `/spec interview` from `[no state]` asks "greenfield or iteration/brownfield?" — if the user selects brownfield adoption, redirect them to `/spec adopt` instead. Otherwise answer is recorded in `state.yaml` as `mode`.

**Iteration-specific work is gated by `phase: closed`.** A user cannot start iteration N+1 until iteration N has been closed with `/spec close` (which produces the takeaway). The state machine enforces this — no need for ad-hoc checks.

## Initialization arguments

Entry-point subcommands that can create a fresh `spec/state.yaml` accept an optional iteration override:

- `/spec interview --iteration N` — only honored when invoked from `[no state.yaml]` (greenfield init) or from `phase: closed` (iteration N+1 init). Elsewhere a warning is printed and the flag ignored.
- `/spec adopt --iteration N` — only honored at bootstrap (the only phase `/spec adopt` accepts).

`N` must be a positive integer. Default is `1` for greenfield/adopt; `prior_iteration + 1` for post-close interview. Bare positional integer (`/spec interview 3`, `/spec adopt 3`) is accepted as a shorthand.

## Commits

After any step that writes `spec/spec.md`, `spec/takeaway.md`, `spec/decisions.log`, or produces archive snapshots, propose the exact `git add` + `git commit` command. Do not run it. `state.yaml` is committed alongside any step that transitions phase.
