# /spec verify — Audit code against spec

Render a per-AC evidence report comparing `spec/spec.md` to the current codebase — either in-session (narrow scope or already-has-context) or via an `Explore` sub-agent after triage. **The orchestrator gathers evidence. You render the verdict.**

## State machine

**Allowed from stages:** `converged` · `verified` (re-run)
**Transitions to:** `verified`
**Re-run behavior:** Allowed freely. Latest report supersedes; old reports retained in archive.

Read state.yaml first; validate stage. Write state.yaml on completion.

## Protocol

1. **Check preconditions.** Read state.yaml; validate stage. If not `converged` or `verified`, suggest the correct step (commonly `/spec check` if not yet converged).

2. **Determine filename.** Iteration `v<NNN>` (zero-padded to 3 digits) + timestamp from state.yaml: `spec/archive/v<NNN>-<YYYY-MM-DD-HHMM>-verify.md`.

3. **Capture current spec.md SHA.** Use `git log -1 --format=%H -- spec/spec.md` to record it; include in the report header for traceability.

4. **Triage and gather evidence.** Before deciding to spawn, count the total leaf ACs and invariant entries in `spec/spec.md` (read it now if not already done). Also estimate source breadth: `find . -mindepth 1 -maxdepth 2 -not -path '*/\.*' -not -path '*/node_modules/*' -type d | wc -l`.

   **Search-scope allowlist (sibling repos + extra in-repo paths).** Before gathering evidence — whether direct or via sub-agent — load optional `spec/verify-allowlist.yaml` from the project root if it exists. Schema:

   ```yaml
   sibling_repos:            # paths relative to project root (often ../other-repo)
     - ../docs-site
   extra_paths:              # optional in-repo (or relative) trees always in scope
     - path/to/retained-but-excluded
   ```

   There is **no skill-level directory-name convention** (no hard-coded `Dormant/`, `_archive/`, etc.). Projects list whatever trees agents might otherwise skip — build-excluded sources, retained assets, vendor mirrors — under `extra_paths`. Sibling deliverables go under `sibling_repos`.

   Apply these rules to every evidence pass (orchestrator direct path **and** Explore sub-agent prompt):

   1. **Sibling repos.** For each entry under `sibling_repos`, resolve the path relative to the project root and treat that directory tree as readable evidence scope. Prefer `Read` / `grep` / `find` on the resolved absolute path. If an AC names a concrete path under `../…` (even when no allowlist file exists), resolve and Read it relative to the project root — never mark **UNCLEAR** solely because the file is outside the git root.
   2. **Paths embedded in ACs.** Any filesystem path that appears in an AC must be opened directly relative to the project root; do not rely on a root-only recursive grep that may skip it (gitignore, unusual layout, or agent default scope).
   3. **Extra in-repo paths.** For each entry under `extra_paths`, resolve relative to the project root and include that tree in search/read scope. Use this for project-specific locations that remain in git but are easy to miss (e.g. excluded from the build target). Naming is entirely project-owned.
   4. **Missing allowlist file.** If `spec/verify-allowlist.yaml` is absent, still apply (2). Sibling roots and extra trees are only auto-expanded when listed in the allowlist or explicitly path-referenced by an AC.
   5. **Verdict discipline.** Use **UNCLEAR** only when the path was searched (project tree + allowlisted siblings + allowlisted extra paths + named AC paths) and evidence is still insufficient. Do **not** use UNCLEAR as a stand-in for "outside my default cwd" or "looked excluded from the build."

   Document this mechanism for skill consumers in `README.md` (§ Search-scope allowlist).

   - **Direct path — handle in-session** if **either** holds:
     1. **Already-has-context** (SKILL.md principle 6): you already know enough about the relevant code in this session to write a complete per-AC evidence report with real `file:line` refs without a fresh broad scan — skip Explore and gather/write the report yourself. Prefer this when true to save tokens.
     2. **Quantitative narrowness:** total leaf ACs + invariants ≤ 5, AND source code is concentrated in ≤ 3 directories.
     In either case: use Bash `grep`/`find` and Read as needed to locate evidence for each AC and invariant, honoring the allowlist rules above. Write the report yourself to `spec/archive/<FILENAME>` using the exact structure shown in the sub-agent prompt below. Proceed to step 5.
   - **Sub-agent path** — if neither direct-path gate holds (wide scope *and* insufficient session context for a reliable report): use the Agent tool with `subagent_type: "Explore"` and `model: "sonnet"`. The prompt must include the **write-fallback instruction from SKILL.md principle 7** **and** the search-scope allowlist block below (expand with the project's actual allowlist contents if the file exists). Prompt:

   > Read `spec/spec.md`. For **each acceptance criterion** in the AC tree (at every level — AC1, AC1.1, AC1.2, etc.):
   >
   > - Search the codebase for evidence the AC is implemented.
   > - Render one verdict: **PASS**, **GAP**, or **UNCLEAR**.
   > - Include specific `file:line` references for every claim.
   > - Include a one-line reason.
   >
   > **Search scope (mandatory):**
   > - If `spec/verify-allowlist.yaml` exists, read it.
   >   - For every `sibling_repos` entry, resolve relative to the project root and search/read inside it when ACs need external evidence.
   >   - For every `extra_paths` entry, resolve relative to the project root and include that tree in scope (project-specific retained/excluded/archive trees — no fixed directory names).
   > - For any path that appears in an AC text (including `../…` sibling paths), resolve relative to the project root and **Read the file directly**. Do not stop at the project git root.
   > - Never render **UNCLEAR** only because a file lives under an allowlisted sibling, an allowlisted extra path, or a path named in an AC; search there first. UNCLEAR only after those locations were checked.
   >
   > If the spec has an Invariants section (iteration mode), give each invariant its own entry with the same PASS/GAP/UNCLEAR verdict.
   >
   > **Do not render an overall pass/fail judgment.** Produce the raw per-item evidence so the user can judge. No hedging language in verdicts — pick one of the three values.
   >
   > Write the report to `spec/archive/<FILENAME>` using this structure:
   >
   > ```
   > # Verify report — iteration <n> — <timestamp>
   > > spec.md SHA at verify: <full-SHA>
   > 
   > ## AC1 — <title>
   > **Verdict:** PASS|GAP|UNCLEAR
   > **Evidence:** <file:line refs>
   > **Reason:** <one line>
   > 
   > ## AC1.1 — <title>
   > ...
   > 
   > ## Invariants (if applicable)
   > ### Invariant 1 — <title>
   > ...
   > 
   > ## Summary
   > Total: <n>, PASS: <n>, GAP: <n>, UNCLEAR: <n>
   > ```
   >
   > Attempt the Write tool for the absolute report path. If denied, retry once; if still denied, include the full intended report verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can write it. Never report success without either having written the file or having dumped its full content.
   >
   > Report back: the file path, the summary line, and (if Write was denied) the full content dump.

5. **Verify the report exists.** After the sub-agent returns, check whether the report was actually written. If not, extract the dumped content and write it directly using the Write tool in the parent session.

6. **Present the report to the user.** Read the file back. Show:
   - The summary line (counts).
   - A terse list of every GAP and UNCLEAR with one-line reasons.
   - Reference to full report path for deeper review.

7. **Update state.yaml:** `stage: verified`, `latest_verify: <filename>`, `last_command: /spec verify`, `last_command_at`.

8. **Next step.** Tell the user:
   - **If any GAPs/UNCLEARs:** "Either address them in code and re-run `/spec verify`, or when ready, run `/spec close` — you'll be asked to provide a rationale for each outstanding gap before closing."
   - **If all PASS:** "Run `/spec close` to finalize the iteration and generate the takeaway."

9. **Propose commit** (optional — verify reports are working artifacts but small):
   ```
   git add spec/archive/<verify-file> spec/state.yaml
   git commit -m "spec: verify v<n> at <short-SHA>"
   ```
