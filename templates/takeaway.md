# <Project Name> — Takeaway: iteration <n>

> Source spec: `../archive/v<NNN>-<YYYY-MM-DD-HHMM>-spec.md` (converged YYYY-MM-DD at SHA `<abc>`)
> Implementation verified: YYYY-MM-DD (report: `archive/v<NNN>-<YYYY-MM-DD-HHMM>-verify.md`)
> Verdict: PASS | PASS-WITH-ACCEPTED-GAPS

## Shipped state `[from-code]`

<Factual bullets describing what now exists. Each traceable to a file or module. Examples:>

- `src/api/charge.ts` — implements idempotent charge flow with 24h key TTL
- `src/db/migrations/003_add_idempotency_keys.sql` — adds `idempotency_keys` table
- CLI `--dry-run` flag available in all write commands

## Deviations from spec

### Accepted gaps
- **AC<ref>** (<short title>) — not implemented. **Rationale:** <one or two sentences from user, captured in /spec close>.

### Other deviations
- <AC that shipped differently than spec'd, with rationale if known>

## Discoveries during implementation

- <Things learned that the spec didn't anticipate — candidates for the next iteration's seed context>
- <Performance characteristics, user behavior insights, external constraints surfaced during build>
- <Assumptions from the spec that turned out to be wrong>

## Key decisions (this iteration)

See `../decisions.log` under the `# Iteration <n>` header. Load-bearing ones summarized:

- <short title> — <one-line rationale>
- <short title> — <one-line rationale>

## Open for next iteration

- <known follow-ups, deferred items with rationale>
- <items from "Discoveries" worth elevating into next iteration's Motivation>
