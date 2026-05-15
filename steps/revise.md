# /spec revise — Incorporate critiques

Guide the user through addressing the latest review critiques before proposing a revised `spec/spec.md`. **Three distinct turns — do not collapse them.**

## State machine

**Allowed from stages:** `in-review`
**Transitions to:** `revised`
**Re-run behavior:** Needs fresh reviews to consume. If called outside `in-review`, suggest the right prior step.

Read state.yaml first; validate stage. Write state.yaml after Turn 3.

## Turn 1 — Read and summarize

1. Read `spec/spec.md` and the three review files matching `spec/archive/<latest_review_stamp>-{ontologist,contrarian,simplifier}.md` (stamp from state.yaml).

2. Present a structured summary to the user:
   - One section per persona.
   - Each concern numbered with a one-line extract + location reference (section heading or AC label in `spec.md`).
   - `## Cross-cutting`: where personas agree (high weight) or conflict (user judgment).

3. **Stop.** Do not propose changes yet. End the turn with:

   > Walk me through how you want to address each concern:
   > - **Accept** — state the change in your own words.
   > - **Reject** — state why.
   > - **Unsure** — we'll think through it together.

## Turn 2 — User addresses critiques

User responds per concern. During dialogue:
- For any "unsure" item, apply **condensed diamond** — enumerate ≥3 ways to address the critique with tradeoffs before helping the user converge.
- When a resolution implies a **structural choice** in `spec.md` — splitting vs. merging an AC, where a new section/subsection lands, an AC-renumbering scheme — raise that fork *here*, in dialogue: enumerate the options with tradeoffs and let the user pick. Do not defer it silently to Turn 3. The goal is that Turn 3 *transcribes* already-decided structure rather than inventing it.
- Log significant decisions to `spec/decisions.log` per `steps/decide.md` (under the current iteration's header).
- Do **not** propose a revised `spec.md` yet. Wait until the user signals all concerns are addressed or explicitly deferred.

## Turn 3 — Propose the revision

Only after the user signals done:

1. Produce the revision. Choose format by change size:
   - **Small changes** (a handful of line edits, 1–2 AC tweaks) → show a unified diff inline and wait for approval before writing.
   - **Large / structural** (section rewrites, AC renumbering, multiple subsections touched) → **first present a short structural outline and wait for a go-ahead**: sections touched, the AC-renumbering map (old → new), where any new sections/subsections land, and the MECE re-check result. Keep it a plan, not the rewritten file — do **not** dump the full new `spec.md` into chat (the `Write` permission prompt already surfaces full content for approval; duplicating it inline wastes tokens on large files). On go-ahead, call `Write` on `spec/spec.md` directly; the message accompanying the `Write` call restates the outline at a high level so the user can skim before approving the write itself. If the user steers at the outline, or denies the write, adjust and re-present the outline.

2. **MECE re-check.** Verify AC tree is still mutually exclusive + collectively exhaustive after changes. In iteration mode, also verify delta/regression tags are still correct. Do this *before* calling `Write` (or showing the diff, or presenting the structural outline) — never after.

3. After `spec/spec.md` is written (whether via direct `Write` or post-diff approval), proceed to step 4.

4. **Update state.yaml:** `stage: revised`, `last_command: /spec revise`, `last_command_at`. `spec_sha` updates after commit.

5. **Propose commit:**
   ```
   git add spec/spec.md spec/decisions.log spec/state.yaml
   git commit -m "spec: revise per review <YYYY-MM-DD>"
   ```

6. **Next step.** Tell the user they can either:
   - Run `/spec review` (another review pass — if the revision was significant and they want fresh critique before moving on), or
   - Run `/spec check` (to verify the spec is ready to implement).
