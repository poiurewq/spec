# <Project Name> — Specification (iteration <n>)

> **Template for iterations** on an existing spec or brownfield project. For greenfield, the skill uses `spec.md` instead.
> Status: draft | reviewed | converged
> Revision: <n>
> Prior iteration: `spec/archive/<date>-v<n-1>-takeaway.md` (or "brownfield — no prior iteration through this skill")
> Last updated: YYYY-MM-DD

## Motivation

<Why this iteration exists. One or two sentences. Reference the prior takeaway, a user report, a bug, compliance, or a new capability being enabled.>

## Current state `[from-code]`

<Brief factual snapshot of what exists and is relevant to this iteration. Use `file:line` refs. Keep tight — full detail lives in the prior `takeaway.md`; this section exists to orient review personas quickly.>

## Change delta

### Added
- <new capability 1>
- <new capability 2>

### Modified
- <changed behavior 1 — from X to Y>

### Removed
- <deprecated capability 1 and migration path>

## Invariants `[must not change]`

- <existing behavior that must continue to work>
- <non-regression requirement>

## Migration `(if applicable)`

<User-facing, data, or API transitions. Keep the header even if empty (write "None.").>

## Goal

<One sentence. What does success look like *for this iteration specifically*?>

## Constraints

- <hard limit 1>
- <hard limit 2>

## Success criteria

- <measurable outcome 1>
- <measurable outcome 2>

## Out of scope

- <explicitly excluded 1>
- <explicitly excluded 2>

## Acceptance criteria (MECE)

> Tree must be **mutually exclusive** and **collectively exhaustive** relative to Goal + Success criteria + Invariants. Each leaf independently testable. Tag each AC with one of:
>
> - **`[delta]`** — new or modified behavior introduced by this iteration.
> - **`[regression]`** — existing behavior that must be preserved (previously verified; `/spec verify` re-confirms it).
> - **`[adopted]`** — *adoption iterations only.* A claim about what the system already does, inherited from the rough spec / Explore summary and ratified in interview, but not yet rigorously verified. `/spec verify` converts these from claim to fact — a GAP on an `[adopted]` AC means the claim was wrong, not that something regressed.

### AC1. <top-level criterion> `[delta]`
- AC1.1. <sub-criterion, testable> `[delta]`
- AC1.2. <sub-criterion, testable> `[delta]`

### AC2. <top-level criterion> `[regression]`
- AC2.1. <sub-criterion, testable> `[regression]`

### AC3. <top-level criterion> `[adopted]`   _(example — only appears in adoption iterations)_
- AC3.1. <sub-criterion, testable> `[adopted]`

## Implementation phases

> Ordered phases for the **`[delta]` work in this iteration**. `[regression]` and `[adopted]` ACs are not phased here — they are verification targets for `/spec verify`, not implementation work.
>
> **Invariant:** phase N must be implementable to completion without any work from phase N+1 or later — dependencies flow only backward. The implementer can work strictly in numerical order.
>
> Each phase references `[delta]` AC IDs from the tree above. Every `[delta]` leaf must appear in exactly one phase. If the iteration's delta is one coherent unit, use a single phase rather than inventing splits. If this iteration has no `[delta]` ACs (pure adoption ratification with no code changes), write "None — this iteration introduces no new implementation work." and omit the phase headings.

### Phase 1. <short name>
**Delivers:** <what works at the end of this phase, in one line>
**Unblocks:** <what later phases need from this — or "foundation" if it's the bedrock of the delta>
- AC1.1
- AC1.2

### Phase 2. <short name>
**Delivers:** <…>
**Depends on:** Phase 1 (<which piece>)
- AC2.1   _(if AC2 is `[delta]`; skip `[regression]`/`[adopted]` ACs)_

## Open questions

<Keep the header even if empty.>
