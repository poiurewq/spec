# /spec defer — Shelve an item for a future iteration

Capture a feature request, bug report, or "we should do this someday" idea to `spec/deferred.md` so it isn't lost between iterations. **Future-tense backlog.** Distinct from `/spec decide` (past-tense, what we already decided) — see Segregation below.

`spec/deferred.md` is mutable, not append-only. Items are added here, presented at the next iteration's interview for triage, and removed when promoted into a spec or dropped. The interview is where promotion happens; this command is for capture and lightweight housekeeping.

## State machine

**Allowed from phases:** any.
**Refused** only when `spec/` does not exist — tell the user: "No `spec/` directory exists yet. Run `/spec setup` first to onboard this project."
**Transitions to:** unchanged. `/spec defer` never moves the phase machine.
**Re-run behavior:** always allowed.

## When this runs

- **Explicitly** — user runs `/spec defer "<text>"`, `/spec defer "<item 1>" "<item 2>" ...`, `/spec defer` (interactive), or `/spec defer --resolve D-XXX drop "<rationale>"`.
- **Automatically** — during any step, when the user says "let's not do this now, maybe later" or otherwise signals a real item should be shelved rather than committed. Append silently; one-line user-facing mention: "Deferred: <title> (D-XXX)." Don't propose a mid-step commit — the parent step's commit proposal will include `spec/deferred.md`.

## File format

`spec/deferred.md` is a flat markdown list. If the file doesn't exist, create it with this header on first write:

```markdown
# Deferred items

_Backlog of feature requests, bug reports, and ideas not yet committed to any iteration. Triaged at each iteration's interview._
```

Each item:

```markdown
## D-001 — <short title>
- First seen: YYYY-MM-DD (iteration N, via <source>)
- Last touched: YYYY-MM-DD (iteration N)
- Defer count: <int>
- Category: <feature | bug | refactor | chore | ...>
- Description: <one to three sentences>
- Notes: <free-form, optional; user may edit between interviews>
```

**Field rules:**

- **ID (`D-NNN`)** — zero-padded to 3 digits. Compute next ID by scanning the file for the highest existing `D-NNN` and incrementing. IDs are stable for the item's lifetime; do not reuse IDs of dropped items.
- **First seen** — `<date> (iteration N, via <source>)`. Iteration `N` is read from `state.yaml` at write time (use `0` if `state.yaml` doesn't exist yet — pre-onboarding capture is rare but possible). Sources: `/spec defer (standalone)`, `/spec defer during interview`, `/spec reconcile bucket: defer`, `/spec close gap resolution`, etc. — record what command produced the item.
- **Last touched** — bumped to today's date and current iteration on any of: continued at interview, edited via `/spec defer`, or recategorized. Initial value matches `First seen`.
- **Defer count** — incremented **only** when the user selects "continue deferring" at an iteration interview. Not bumped by creation, edits, or category changes. Initial value `0`.
- **Category** — suggest one of `feature` / `bug` / `refactor` / `chore`; accept any string the user writes. The file stays flat — no per-category sections.
- **Description / Notes** — free-form. Description is the substance; Notes is an optional context line.

## Protocol — explicit invocation

### Capture mode (`/spec defer "<text>"` or batch or interactive)

1. **Validate `spec/` exists.** Refuse per the State machine section if not.

2. **Parse arguments:**
   - One or more quoted strings → each is a candidate item description.
   - No arguments → interactive: prompt "Enter items to defer one at a time; type `done` when finished."
   - For each item, ask: "Category? (feature / bug / refactor / chore, or other)" — accept any answer; default `feature` if user just hits enter.

3. **Compute next ID.** Read `spec/deferred.md` if present; find highest existing `D-NNN`; next ID is that + 1 (zero-padded to 3 digits). If file is missing, start at `D-001`.

4. **Compose entries.** For each item, build the markdown block per the format above:
   - Title: short summary (≤ ~60 chars). If user gave a long description, ask for a title or auto-derive (first sentence, trimmed).
   - First seen: today's date + current iteration (from `state.yaml`) + source `via /spec defer (standalone)`.
   - Last touched: same as First seen.
   - Defer count: `0`.
   - Description: the user's text.
   - Notes: empty.

5. **Append to `spec/deferred.md`.** Create the file with header if it doesn't exist. Append items in invocation order at the end of the file.

6. **Show the user** the IDs and titles assigned: "Deferred: D-001 — Add CSV export. D-002 — Webhook 429 handling."

7. **Propose commit:**
   ```
   git add spec/deferred.md
   git commit -m "defer: <comma-separated short titles>"
   ```

   If more than 3 items, use `git commit -m "defer: <N> items (D-XXX..D-YYY)"`.

### Resolve mode (`/spec defer --resolve D-XXX drop "<rationale>"`)

For housekeeping outside an interview — drop a deferred item the user no longer wants on the plate.

1. **Validate.** Only `drop` is accepted. **Refuse `--resolve D-XXX promote`** with the message: "Promotion happens during `/spec interview` (next iteration's triage) or as a side effect of `/spec reconcile` (when drift matches a deferred item). It cannot be invoked standalone."

2. **Find the item.** If `D-XXX` doesn't exist in `spec/deferred.md`, error and stop.

3. **Remove the item from `spec/deferred.md`.**

4. **Append to `spec/decisions.log`** per `steps/decide.md`'s auto-invocation protocol:
   - Title: `Dropped deferred item D-XXX — <item title>`
   - Decision: `Dropped deferred item D-XXX without implementing.`
   - Rationale: `<user's rationale>`
   - Related: `Dropped from D-XXX`
   - Context: `via /spec defer --resolve (iteration <n>)`

5. **Propose commit:**
   ```
   git add spec/deferred.md spec/decisions.log
   git commit -m "defer: drop D-XXX — <short title>"
   ```

## Protocol — automatic invocation

When another step's dialogue reveals an item that should be shelved (user says "not this iteration, maybe later" or similar):

1. **If `spec/deferred.md` does not exist, create it first** with the standard header:
   ```markdown
   # Deferred items

   _Backlog of feature requests, bug reports, and ideas not yet committed to any iteration. Triaged at each iteration's interview._
   ```
   Then compute next ID and append to `spec/deferred.md` per Capture mode steps 3–5, with source set per the calling step (e.g., `via /spec reconcile bucket: defer`, `via /spec close gap resolution`, `via /spec interview triage`).

2. Emit one-line mention to the user: "Deferred: <title> (D-XXX)."

3. **Do not propose a mid-step commit.** The parent step's commit proposal must include `spec/deferred.md` in its `git add` line.

## Segregation: defer vs. decide

Mental rule for new users:

- **Past-tense → `decisions.log`.** What we decided (positively, negatively-permanent, or meta). Append-only.
- **Future-tense, committed → `spec/spec.md`.** What we are building this iteration.
- **Future-tense, uncommitted → `spec/deferred.md`.** Backlog. Mutable.

Cross-references between the two files happen on every state change of a deferred item:

- **Item created via `/spec defer`** → entry in `deferred.md`. No `decisions.log` entry (the decision to defer is implicit; logging it is noise).
- **Item dropped via `/spec defer --resolve drop` or interview triage** → removed from `deferred.md`; *one-line `decisions.log` entry* recording the drop with rationale.
- **Item promoted via interview triage** → removed from `deferred.md`; *one-line `decisions.log` entry* recording the promotion ("Promoted D-XXX into iteration N spec"); the item then takes its place as one or more ACs in the new iteration's `spec.md`.
- **Item promoted via `/spec reconcile` (drift-match case)** — same as interview promotion: removed from `deferred.md`; `decisions.log` entry; the spec edit lands in the current iteration's `spec.md` rather than a new one.

## Notes

- **Tolerance for hand-edited entries.** Users can refine descriptions and notes between interviews. The skill does not lock the file. If a user-added entry is missing skill-managed metadata fields (e.g., no Defer count), tolerate it — fill in sensible defaults the next time the item is touched (e.g., interview triage).
- **No staleness auto-prune.** Items linger until the user drops them. The interview triage sorts stalest-first (descending Defer count, ascending Last touched) so old items get attention first; that's the only nudge.
- **Empty file handling.** When `spec/deferred.md` is empty (only the header) or missing, the interview triage step is a no-op and prints "No deferred items on the plate."
