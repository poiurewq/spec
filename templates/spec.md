# <Project Name> — Specification

> **Template for greenfield projects.** For iterations on an existing spec, the skill uses `spec-iteration.md` instead.
> Status: draft | reviewed | converged
> Revision: <n>
> Last updated: YYYY-MM-DD

## Goal

<One sentence. What is this, and for whom?>

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

> This tree must be **mutually exclusive** (no AC overlaps another) and **collectively exhaustive** (every success criterion traces to at least one AC; every AC traces back to a goal or success criterion). Each leaf must be independently testable.

### AC1. <top-level criterion>
- AC1.1. <sub-criterion, testable>
- AC1.2. <sub-criterion, testable>

### AC2. <top-level criterion>
- AC2.1. <sub-criterion, testable>

## Implementation phases

> Ordered phases for building this. **Invariant:** phase N must be implementable to completion without any work from phase N+1 or later — dependencies flow only backward (later phases may depend on earlier ones, never the reverse). The implementer can work strictly in numerical order.
>
> Each phase references AC IDs from the tree above. ACs may be split across phases at the leaf level if a parent AC has independently shippable sub-criteria. Every leaf AC must appear in exactly one phase. If the work is genuinely one coherent unit, use a single phase rather than inventing splits.

### Phase 1. <short name>
**Delivers:** <what works at the end of this phase, in one line>
**Unblocks:** <what later phases need from this — or "foundation" if it's the bedrock>
- AC1.1
- AC2.1

### Phase 2. <short name>
**Delivers:** <…>
**Depends on:** Phase 1 (<which piece>)
- AC1.2

## Open questions

<If any, list here with one-line context each. Keep the header even if empty.>
