# /spec interview — Socratic interview

Conduct a Socratic interview to turn a vague project idea (or iteration request) into the inputs needed for a precise specification. You (the main agent) are the interviewer — do not delegate.

## State machine

**Allowed from stages:** (no state.yaml — first run) · `closed` (starts iteration N+1) · `interviewing` (resume existing — covers both `mode: iteration`/`greenfield` resumes *and* `mode: adopted` continuation after `/spec adopt`)
**Transitions to:** `interviewing`. The stage stays `interviewing` until `/spec seed` runs — the gate merely *permits* `/spec seed`.
**Re-run behavior:** If stage is `interviewing`, resume the existing session file (read `latest_interview` from state.yaml). If stage is `closed`, start a new iteration (iteration number = `prior + 1` unless `--iteration N` is provided).

Before executing: read `spec/state.yaml`. If missing, treat as first run. If stage is neither allowed nor missing, report mismatch and suggest the right command.

After completing each turn: update state.yaml with `stage`, `mode`, `iteration`, `started_at` (on first question of iteration), `last_command`, `last_command_at`, `latest_interview`.

## Arguments

- `--iteration N` *(optional)* — only honored when this invocation is **initializing** state.yaml (first run) or **starting a fresh iteration** from `closed`. Elsewhere, print a one-line warning and ignore. Default: `1` for first run, `prior + 1` post-close. Accepts `--iteration N`, `--iteration=N`, or bare positional `N`. Reject non-positive integers.

## Mode determination (first substantive turn)

If resuming (stage was already `interviewing`, including an adoption initialized by `/spec adopt`): skip this section and jump to the appropriate protocol below based on `mode` in state.yaml.

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
     If yes, triage before spawning (SKILL.md principle 6 — either gate skips Explore):
     - **Already-has-context:** if you already know enough about the named paths/code in this session to write a solid `## Current state [from-code]` section with real `file:line` refs, do it yourself — no sub-agent. Prefer this when true to save tokens.
     - **Quantitative narrowness:** if the user named ≤ 3 specific files or a single narrow directory, read those paths directly with Read and Bash, then append the section yourself.
     - **Otherwise** (broad directory, "entire codebase", or many paths, *and* insufficient session context): spawn Agent with `subagent_type: "Explore"`, prefer cheaper model if available (SKILL.md principle 6), prompt narrowly scoped to the user-specified paths with output appended to the session file. The sub-agent prompt must include the **write-fallback instruction from SKILL.md principle 7** — attempt to append to the session file; on Write denial, retry once, then dump full content in the reply. After the sub-agent returns, verify the append landed; if not, write it from the parent session.

3. **Deferred-items triage** (before intent questions; after context ingestion). If `spec/deferred.md` exists and contains at least one item:

   a. Read it. Sort items **stalest-first**: descending by `Defer count`, then ascending by `Last touched`. Items with `Defer count ≥ 3` are flagged with a `⚠ stale` marker in the prompt below.

   b. Present:

   > Deferred items on the plate from prior iterations:
   >
   > - D-001 — <title>  (deferred since iter <n>; last touched iter <m>; defer count <c>) <⚠ stale if applicable>
   > - D-002 — ...
   >
   > For each: **include** in this iteration, **continue deferring**, or **drop** (with rationale)?

   c. **Per-item resolution.** Apply the user's choice:
      - **Include** — remove the item from `spec/deferred.md`. Append a short summary of the item to the interview session file under a new `### Deferred items pulled in` subsection (so `/spec seed` sees them as iteration intent). Propose a `spec/decisions.md` entry via `steps/decide.md`'s auto-invocation protocol (mandatory Consent gate): title `Promoted D-XXX into iteration <n> spec`, decision `Pulled deferred item D-XXX into this iteration's commitments.`, rationale supplied by user (ask if they didn't say why they want to pull it in now), related `Promoted from D-XXX`, context `during /spec interview triage (iteration <n>)`. If the user declines the gate, the deferred-item removal still stands — only the log entry is skipped.
      - **Continue deferring** — bump `Last touched` to today + current iteration; increment `Defer count`. No `decisions.md` entry.
      - **Drop** — remove from `spec/deferred.md`. Propose a `spec/decisions.md` entry via `steps/decide.md`'s auto-invocation protocol (mandatory Consent gate): title `Dropped deferred item D-XXX`, decision `Dropped deferred item D-XXX without implementing.`, rationale supplied by user verbatim, context `during /spec interview triage (iteration <n>)`. If the user declines the gate, the drop still stands — only the log entry is skipped.

   d. If `spec/deferred.md` is empty or missing, skip this step silently. (Mention "No deferred items on the plate." only if the user explicitly asks.)

4. **Intent-focused questions.** Ask about *what should change*, not *what exists*. Examples:
   - "Given <existing thing>, should this iteration extend it, replace it, or leave it alone and build alongside?"
   - "What must *not* break?"
   - "What's the motivating trigger for this iteration?"
   - Apply the **condensed diamond** at interpretation forks.

5. **Tag every answer** in the transcript:
   - `[from-code]` — facts about the current codebase (from user, orchestrator, or Explore)
   - `[from-user]` — user decisions, preferences, priorities (not externally verifiable)
   - `[from-research]` — external facts (API docs, compatibility, pricing) with source reference

6. **Cover these axes:** Motivation, Current state (ingested above), Change delta, Invariants, Goal, Constraints, Success criteria, Scope boundary.

## Adoption protocol (`mode: adopted`)

This runs when `/spec adopt` already created the session file and transitioned stage to `interviewing`. `latest_interview` in state.yaml points to the pre-populated file.

The adoption interview is a **two-part** flow with an explicit checkpoint between them. Part A reconciles the written spec artifact (which `/spec seed` will produce next) against shipped code — it is documentation hygiene disguised as Q&A. Part B is the forward-looking Socratic interview about what this iteration is *for*. Do not blur the two: finish Part A and pass its mini-gate before starting Part B.

### Setup (both parts)

1. **Open the existing session file.** Do **not** create a new one. Read its `## Adopted context (pre-interview)` section in full — it contains `[from-code]` bullets, `[from-rough-spec]` claims (if any), and an Ambiguities list produced at adoption time.

2. **Append a new `## Socratic interview` section** below the pre-populated context. Under it, create two subsections up front — `### Part A — Ratify current state` and `### Part B — Intent for this iteration` — so transcript entries land under the correct part as you go.

### Part A — Ratify current state (reconciliation)

The goal of Part A is a single output: a current-state snapshot the user affirms is faithful to shipped reality. No forward-looking questions here.

3. **Orienting turn (scoped to the summary only).** Distill the pre-populated context into 3–5 bullets and ask: *"Does this summary accurately describe what exists today? Flag anything wrong, missing, or stated as fact that's actually aspirational. (We'll walk the individual ambiguities next — this turn is just about the high-level shape.)"* Record corrections as `[from-user]` and note which pre-populated claims they override.

4. **Walk the ambiguities (three-way resolution).** For each item in `### Ambiguities and tensions`, present it and ask the user to pick one of three resolutions. Make all three explicit — do not let the conversation default to "code wins":
   - **(a) Code is truth, spec was wrong** → the eventual `spec/spec.md` should describe the code's behavior. Tag the resolution `[from-user]` overriding the `[from-rough-spec]` claim.
   - **(b) Spec is truth, code has a bug** → the eventual `spec/spec.md` should keep the spec's claim; the code needs to change to match. Tag the resolution `[build-change-todo]` with a one-line description of the needed code change. These TODOs flow into `/spec seed` as deferred work / acceptance criteria, not dropped.
   - **(c) Both stale** → the user articulates a third answer; tag `[from-user]` and note both prior claims are superseded.

   **Modifier — `[iteration-scope]`:** any of (a)/(b)/(c) may additionally be flagged `[iteration-scope]` when the user affirms the resolution is true *for this iteration only* and expects to revisit it in a future iteration (e.g., "no `SettingsViewModel` yet — we'll extract one in a future iteration when logic grows"). This keeps the current spec purely descriptive of shipped reality while recording the forward intent. Items tagged `[iteration-scope]` are appended to `spec/deferred.md` per `steps/defer.md`'s automatic-invocation pattern, with source `via /spec interview Part A (iteration <n>)`, category default `refactor` (override at user's request), and description summarizing the transitional shape (e.g., "Extract `SettingsViewModel` once logic grows beyond inline state"). They appear in a future iteration's interview triage like any other deferred item. Use sparingly — most resolutions are permanent; this modifier is for shapes the user explicitly names as transitional. Do **not** project the revisit onto a concrete future `iter-N` (see *No forward iteration-number projection* below).

   Apply the **condensed diamond** when resolution shape has more than these three interpretations (rare, but e.g., timing thresholds may have a range of reasonable values).

5. **Part A mini-gate (ratified-snapshot checkpoint).** When every ambiguity is resolved, produce a consolidated **Ratified current state** block in the session file — the original pre-interview context with each ambiguity replaced by its chosen resolution, inline. Present it and ask: *"Is this now a faithful, drift-free description of what exists? This is the baseline Part B will build on; we won't re-open these points after this."*
   - **Any "no":** re-open the specific items the user names; return to step 4 for those items only.
   - **"Yes":** proceed to Part B. The ratified snapshot is frozen for this iteration.

### Part B — Intent for this iteration (Socratic)

The goal of Part B is to articulate *what iteration N is for*, given the ratified baseline. The adoption iteration is special — its delta may legitimately be empty (pure ratification) or it may include real changes.

6. **Intent questions.** Common shapes:
   - "Is this iteration meant to *ratify* the current behavior (no delta, just formalize the spec), to *extend* it, or to *correct* drift between the rough spec and reality?" (If the user named `[build-change-todo]` items in Part A, those are candidates for the "correct drift" answer.)
   - "What must *not* change?" (invariants)
   - "What's the trigger for adopting the skill now — bug, compliance, handoff, team growth?"
   - Cover axes: Motivation, Change delta (may be empty — that's valid), Invariants, Scope boundary, Success criteria for the adoption iteration.
   - Apply the **condensed diamond** at interpretation forks.

7. **Tag every answer** in the transcript (applies to both parts):
   - `[from-code]` — facts about the current codebase (user- or Explore-sourced)
   - `[from-rough-spec]` — claims inherited from the rough spec doc
   - `[from-user]` — user decisions, preferences, priorities (not externally verifiable)
   - `[from-research]` — external facts with source reference
   - `[build-change-todo]` — Part A ambiguity resolutions where the code needs to change to match the spec's retained claim (adoption mode only)
   - `[iteration-scope]` — modifier on a Part A resolution indicating the current answer holds only for this iteration and is expected to be revisited in a future iteration (adoption mode only; always paired with one of the primary tags; never name a concrete future `iter-N`)
   - `[invariant-provisional]` — modifier on a Part B invariant indicating the invariant stands, but is explicitly *revisable under pressure* from specific real-world evidence (e.g., a user-surfaced bug that motivates refinement of a checklist or mechanism). Distinct from a regular invariant (change requires `/spec decide`) and from `[iteration-scope]` (scheduled change in a future iteration): the trigger is evidence-driven refinement, not an iteration boundary. Use when the user wants to commit to an invariant's shape but acknowledge it hasn't been battle-tested enough to freeze rigidly. Adoption mode only; always paired with an invariant description and a named trigger (what evidence would motivate revision).

## Timebox (Constraints axis, all modes)

When covering Constraints, surface the timebox explicitly rather than leaving it implicit. Offer three framings and let the user pick: a **hard deadline** (ship by date X), a **soft target** (aim for X, slip if needed), or **open-ended** (no timebox — proceed until done). Open-ended is a fully legitimate, complete answer — do not nudge the user toward setting a deadline. Record the chosen framing in the `## Seed handoff` Constraints axis summary.

## Clarity gate (all modes)

When coverage is sufficient (or after ~12 questions, whichever first), read and present `templates/clarity-gate.md`. In iteration or adopted modes, also present the appended iteration-specific items. **Ask the user to self-rate each item yes/no. Do not score for them.**

- **Any "no":** one more targeted question on that axis, then re-present the gate.
- **All "yes":**
  1. Append a `## Seed handoff` section to the session file. This is the durable carrier of interview nuance into `/spec seed` — which drafts in its own fresh session reading only this file from disk, never this conversation — so anything not written here is lost to seed. In the user's own phrasing:
     - **Axis summary** — one or two sentences per axis: Goal, Constraints, Success criteria, Scope boundary (plus Motivation, Current state, Change delta, Invariants in iteration & adopted modes).
     - **Emphasis** — what the user stressed or returned to; relative priorities they signaled but that aren't obvious from the axis summary alone.
     - **Ruled-out interpretations** — readings the user explicitly rejected and why (easy to lose; cross-reference any `decisions.md` entries logged during this interview).
     - **Open tensions** — unresolved trade-offs or items the user flagged as uncertain, so seed routes them to `## Open questions` rather than silently picking.
  2. Update state.yaml: `latest_interview` set, `stage` remains `interviewing` (transitions to `seeded` only when `/spec seed` runs).
  3. Tell the user: "Interview complete. Start a new conversation and run `/spec seed` — it drafts the spec interactively from this interview's `## Seed handoff`. Drafting is a generative step you can steer, so it gets its own fresh session."
  4. Propose (optional):
     ```
     git add spec/archive/<file>.md spec/state.yaml
     git commit -m "spec: interview v<n>"
     ```

## No forward iteration-number projection

Do not project work onto specific future iteration numbers. When the user defers work, sketches runway, or flags something as "reserved for later," record it as **"a future iteration"** and, where useful, a temporal-distance qualifier (**near-term / medium-term / long-term**) — never a concrete `iter-N`. References to the **current** iteration and to **past / baseline** iterations are factual and remain as-is. Future iteration numbers are guesses that ossify into false commitments; the deferral is real, the iteration assignment is not. If the user names a concrete future number themselves, rephrase to the distance qualifier (or plain "a future iteration") when writing session notes, seed handoff, and deferred items — confirm the rephrase if the number seemed load-bearing to them.

## Decision capture

When the user makes a non-obvious choice mid-interview (ruling out scope, picking an interpretation with rationale, selecting one structuring over alternatives), **propose** an entry via `steps/decide.md`'s auto-invocation protocol — never append silently. Every proposal must traverse the Consent gate in `steps/decide.md`; many candidate entries during an interview are minor and the user will rightly skip them. Batch proposals at natural pause points in the interview (e.g., end of an axis, before the clarity gate) rather than interrupting every turn.

Anti-rationalization rule: when composing the rationale field, use only the reason the user gave in their own words. If they picked an interpretation without explaining why — and the choice is non-trivial enough to consider logging — ask them ("What made (b) the right read?") before proposing the entry. Do not synthesize a rationale from context. **Founder-said-so is a valid answer:** if the user declines further justification, record it as product-owner prerogative (e.g. `Rationale: product-owner decision (no further justification given).`) rather than inventing a plausible "why" or pressing past a clear decline. Iteration entries go under a `# Iteration <n>` header; create the header if starting a new iteration.
