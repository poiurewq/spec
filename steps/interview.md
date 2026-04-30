# /spec interview — Socratic interview

Conduct a Socratic interview to turn a vague project idea (or iteration request) into the inputs needed for a precise specification. You (main Opus agent) are the interviewer — do not delegate.

## State machine

**Allowed from phases:** (no state.yaml — first run) · `closed` (starts iteration N+1) · `interviewing` (resume existing — covers both `mode: iteration`/`greenfield` resumes *and* `mode: adopted` continuation after `/spec adopt`)
**Transitions to:** `interviewing`. The phase stays `interviewing` until `/spec seed` runs — the gate merely *permits* `/spec seed`.
**Re-run behavior:** If phase is `interviewing`, resume the existing session file (read `latest_interview` from state.yaml). If phase is `closed`, start a new iteration (iteration number = `prior + 1` unless `--iteration N` is provided).

Before executing: read `spec/state.yaml`. If missing, treat as first run. If phase is neither allowed nor missing, report mismatch and suggest the right command.

After completing each turn: update state.yaml with `phase`, `mode`, `iteration`, `started_at` (on first question of iteration), `last_command`, `last_command_at`, `latest_interview`.

## Arguments

- `--iteration N` *(optional)* — only honored when this invocation is **initializing** state.yaml (first run) or **starting a fresh iteration** from `closed`. Elsewhere, print a one-line warning and ignore. Default: `1` for first run, `prior + 1` post-close. Accepts `--iteration N`, `--iteration=N`, or bare positional `N`. Reject non-positive integers.

## Mode determination (first substantive turn)

If resuming (phase was already `interviewing`, including an adoption initialized by `/spec adopt`): skip this section and jump to the appropriate protocol below based on `mode` in state.yaml.

If starting fresh (no state.yaml or coming from `closed`):

1. **Infer default:**
   - `spec/takeaway.md` exists, OR state.yaml shows prior `closed` → **iteration**
   - Non-empty project dir with substantial prior code → suggest **adoption**; do not silently default to iteration.
   - Otherwise → **greenfield**

2. **Ask:** "Starting fresh (greenfield), iterating on an existing closed spec, or adopting a brownfield project that has code (and maybe a rough spec doc) but has never been through this skill? [default: <inferred>]"

3. **If user picks adoption:** stop this command and instruct them to run `/spec adopt` instead (optionally with `--iteration N`). Adoption is a distinct bootstrap with its own ingestion step — it cannot be done from inside `/spec interview`.

4. **Otherwise, record** the answer as `mode` (`greenfield` or `iteration`) in state.yaml. Create state.yaml if missing. If `--iteration N` was supplied and this is an initializing run, use `N`; otherwise default (`1` for greenfield, `prior + 1` post-close).

## Greenfield protocol

1. **Open session file.** Iteration `v<NNN>` (zero-padded to 3 digits) + timestamp from state.yaml: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-interview.md`. Header: project working title (ask if unclear) + date + `Mode: greenfield`. Record filename in state.yaml as `latest_interview`.

2. **Opening question:** "In one or two sentences, what are you trying to build, and for whom?"

3. **Iterate Socratically.** After each answer:
   - Append question + answer to the session file.
   - Ask the single question that most reduces ambiguity. Favor ontological framings:
     - "What IS this, really — stripped of surface details?"
     - "Is that a root need, or a symptom?"
     - "What must exist for this to work?"
     - "What are you assuming that might not be true?"
   - **Condensed diamond:** when multiple interpretations exist, surface ≥3 explicitly and ask the user to pick (or say "none of these"). Record losers via `steps/decide.md`.

4. **Cover four axes before gating:** Goal, Constraints, Success criteria, Scope boundary.

## Iteration protocol

1. **Open session file.** Same path scheme: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-interview.md`. Header: working title + date + `Mode: iteration` + `Prior takeaway: spec/takeaway.md` (or note brownfield if no prior takeaway).

2. **Context ingestion (before intent questions):**
   - **If `spec/takeaway.md` exists** — read it. Ask user to confirm it still reflects shipped reality; note any drift.
   - **Otherwise (brownfield first iteration)** — ask the user to point at the prior spec (if any), relevant code paths, and any design docs. Then offer:
     > "Should I spawn an Explore sub-agent to summarize the relevant code into a `## Current state [from-code]` section for this interview, or would you rather describe current state yourself?"
     If yes, triage before spawning: if the user named ≤ 3 specific files or a single narrow directory, read those paths directly with Read and Bash, then append the `## Current state [from-code]` section yourself — no sub-agent needed. Otherwise (broad directory, "entire codebase", or many paths) spawn Agent with `subagent_type: "Explore"`, `model: "sonnet"`, prompt narrowly scoped to the user-specified paths with output appended to the session file. The sub-agent prompt must include the **write-fallback instruction from SKILL.md principle 7** — attempt to append to the session file; on Write denial, retry once, then dump full content in the reply. After the sub-agent returns, verify the append landed; if not, write it from the parent session.

3. **Intent-focused questions.** Ask about *what should change*, not *what exists*. Examples:
   - "Given <existing thing>, should this iteration extend it, replace it, or leave it alone and build alongside?"
   - "What must *not* break?"
   - "What's the motivating trigger for this iteration?"
   - Apply the **condensed diamond** at interpretation forks.

4. **Tag every answer** in the transcript:
   - `[from-code]` — facts about the current codebase (from user or Explore sub-agent)
   - `[from-user]` — user decisions, preferences, priorities (not externally verifiable)
   - `[from-research]` — external facts (API docs, compatibility, pricing) with source reference

5. **Cover these axes:** Motivation, Current state (ingested above), Change delta, Invariants, Goal, Constraints, Success criteria, Scope boundary.

## Adoption protocol (`mode: adopted`)

This runs when `/spec adopt` already created the session file and transitioned phase to `interviewing`. `latest_interview` in state.yaml points to the pre-populated file.

The adoption interview is a **two-phase** flow with an explicit checkpoint between them. Phase A reconciles the written spec artifact (which `/spec seed` will produce next) against shipped code — it is documentation hygiene disguised as Q&A. Phase B is the forward-looking Socratic interview about what this iteration is *for*. Do not blur the two: finish Phase A and pass its mini-gate before starting Phase B.

### Setup (both phases)

1. **Open the existing session file.** Do **not** create a new one. Read its `## Adopted context (pre-interview)` section in full — it contains `[from-code]` bullets, `[from-rough-spec]` claims (if any), and an Ambiguities list produced by the adoption Explore sub-agent.

2. **Append a new `## Socratic interview` section** below the pre-populated context. Under it, create two subsections up front — `### Phase A — Ratify current state` and `### Phase B — Intent for this iteration` — so transcript entries land under the correct phase as you go.

### Phase A — Ratify current state (reconciliation)

The goal of Phase A is a single output: a current-state snapshot the user affirms is faithful to shipped reality. No forward-looking questions here.

3. **Orienting turn (scoped to the summary only).** Distill the pre-populated context into 3–5 bullets and ask: *"Does this summary accurately describe what exists today? Flag anything wrong, missing, or stated as fact that's actually aspirational. (We'll walk the individual ambiguities next — this turn is just about the high-level shape.)"* Record corrections as `[from-user]` and note which pre-populated claims they override.

4. **Walk the ambiguities (three-way resolution).** For each item in `### Ambiguities and tensions`, present it and ask the user to pick one of three resolutions. Make all three explicit — do not let the conversation default to "code wins":
   - **(a) Code is truth, spec was wrong** → the eventual `spec/spec.md` should describe the code's behavior. Tag the resolution `[from-user]` overriding the `[from-rough-spec]` claim.
   - **(b) Spec is truth, code has a bug** → the eventual `spec/spec.md` should keep the spec's claim; the code needs to change to match. Tag the resolution `[build-change-todo]` with a one-line description of the needed code change. These TODOs flow into `/spec seed` as deferred work / acceptance criteria, not dropped.
   - **(c) Both stale** → the user articulates a third answer; tag `[from-user]` and note both prior claims are superseded.

   **Modifier — `[iteration-scope]`:** any of (a)/(b)/(c) may additionally be flagged `[iteration-scope]` when the user affirms the resolution is true *for this iteration only* and expects to revisit it in a future iteration (e.g., "no `SettingsViewModel` yet — we'll extract one next iteration when logic grows"). This keeps the current spec purely descriptive of shipped reality while recording the forward intent. Items tagged `[iteration-scope]` also get a one-line entry in `spec/decisions.log` under the current iteration header: "Deferred: <short description> — revisit in iteration N+1." Use sparingly — most resolutions are permanent; this modifier is for shapes the user explicitly names as transitional.

   Apply the **condensed diamond** when resolution shape has more than these three interpretations (rare, but e.g., timing thresholds may have a range of reasonable values).

5. **Phase A mini-gate (ratified-snapshot checkpoint).** When every ambiguity is resolved, produce a consolidated **Ratified current state** block in the session file — the original pre-interview context with each ambiguity replaced by its chosen resolution, inline. Present it and ask: *"Is this now a faithful, drift-free description of what exists? This is the baseline Phase B will build on; we won't re-open these points after this."*
   - **Any "no":** re-open the specific items the user names; return to step 4 for those items only.
   - **"Yes":** proceed to Phase B. The ratified snapshot is frozen for this iteration.

### Phase B — Intent for this iteration (Socratic)

The goal of Phase B is to articulate *what iteration N is for*, given the ratified baseline. The adoption iteration is special — its delta may legitimately be empty (pure ratification) or it may include real changes.

6. **Intent questions.** Common shapes:
   - "Is this iteration meant to *ratify* the current behavior (no delta, just formalize the spec), to *extend* it, or to *correct* drift between the rough spec and reality?" (If the user named `[build-change-todo]` items in Phase A, those are candidates for the "correct drift" answer.)
   - "What must *not* change?" (invariants)
   - "What's the trigger for adopting the skill now — bug, compliance, handoff, team growth?"
   - Cover axes: Motivation, Change delta (may be empty — that's valid), Invariants, Scope boundary, Success criteria for the adoption iteration.
   - Apply the **condensed diamond** at interpretation forks.

7. **Tag every answer** in the transcript (applies to both phases):
   - `[from-code]` — facts about the current codebase (user- or Explore-sourced)
   - `[from-rough-spec]` — claims inherited from the rough spec doc
   - `[from-user]` — user decisions, preferences, priorities (not externally verifiable)
   - `[from-research]` — external facts with source reference
   - `[build-change-todo]` — Phase A ambiguity resolutions where the code needs to change to match the spec's retained claim (adoption mode only)
   - `[iteration-scope]` — modifier on a Phase A resolution indicating the current answer holds only for this iteration and is expected to be revisited in a future iteration (adoption mode only; always paired with one of the primary tags)
   - `[invariant-provisional]` — modifier on a Phase B invariant indicating the invariant stands, but is explicitly *revisable under pressure* from specific real-world evidence (e.g., a user-surfaced bug that motivates refinement of a checklist or mechanism). Distinct from a regular invariant (change requires `/spec decide`) and from `[iteration-scope]` (scheduled change in a known future iteration): the trigger is evidence-driven refinement, not an iteration boundary. Use when the user wants to commit to an invariant's shape but acknowledge it hasn't been battle-tested enough to freeze rigidly. Adoption mode only; always paired with an invariant description and a named trigger (what evidence would motivate revision).

## Clarity gate (all modes)

When coverage is sufficient (or after ~12 questions, whichever first), read and present `templates/clarity-gate.md`. In iteration or adopted modes, also present the appended iteration-specific items. **Ask the user to self-rate each item yes/no. Do not score for them.**

- **Any "no":** one more targeted question on that axis, then re-present the gate.
- **All "yes":**
  1. Append `## Conclusion` to the session file summarizing each axis in the user's own phrasing.
  2. Update state.yaml: `latest_interview` set, `phase` remains `interviewing` (transitions to `seeded` only when `/spec seed` runs).
  3. Tell the user: "Interview complete. Start a new conversation and run `/spec seed` to draft the spec."
  4. Propose (optional):
     ```
     git add spec/archive/<file>.md spec/state.yaml
     git commit -m "spec: interview v<n>"
     ```

## Decision capture

When the user makes a non-obvious choice mid-interview (ruling out scope, picking an interpretation with rationale, selecting one structuring over alternatives), append to `spec/decisions.log` per `steps/decide.md` inline — no ceremony, one-line mention: "Logged decision: <title>." Iteration entries go under a `# Iteration <n>` header; create the header if starting a new iteration.
