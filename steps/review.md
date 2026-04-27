# /spec review — Three-persona review

Three skeptical review passes against the current `spec/spec.md`. Each persona runs as a parallel Opus sub-agent.

## State machine

**Allowed from phases:** `seeded` · `revised` · `in-review` (re-run with warning)
**Transitions to:** `in-review`
**Re-run behavior:** If already `in-review`, warn: *"Reviews for current spec.md already exist (stamp X). Generate a fresh set?"* On yes, new timestamp prefix; old review files stay in archive.

Read state.yaml first; validate phase. Write state.yaml on completion.

## Protocol

1. **Check preconditions.** Read state.yaml; validate phase. Confirm `spec/spec.md` exists. If uncommitted changes exist, warn and ask whether to review the uncommitted version (usually yes).

2. **Determine timestamp.** Iteration `v<NNN>` (zero-padded to 3 digits) from state.yaml + current `YYYY-MM-DD-HHMM`. Prefix: `v<NNN>-<YYYY-MM-DD-HHMM>`. All three review files share this prefix.

3. **Spawn three sub-agents in parallel.** Single message with three Agent tool calls. For each:
   - `subagent_type: "general-purpose"`
   - `model: "sonnet"`
   - Prompt (must embed the **write-fallback instruction from SKILL.md principle 7**):

     > Adopt the role at `<skill-base-dir>/personas/<PERSONA>.md` (resolve `<skill-base-dir>` to the absolute path announced at skill invocation before passing to the sub-agent). Read `spec/spec.md`.
     >
     > If mode is iteration (check state.yaml, or detect iteration-specific sections like `## Motivation` / `## Change delta` / `## Invariants` in spec.md), also read `spec/takeaway.md` if it exists, and apply the persona file's `## In iteration mode` section in addition to the main protocol.
     >
     > Apply the persona protocol. Write your critique to `spec/archive/<PREFIX>-<PERSONA>.md` (use the absolute path). Under 400 words. Be specific — reference section headings or AC labels.
     >
     > Attempt the Write tool; if denied, retry once; if still denied, include the full intended critique verbatim in your final reply inside a fenced code block labeled with the absolute target path, so the parent session can write it. Never report success without either having written the file or having dumped its full content.
     >
     > Report back: the file path and (if Write was denied) the full content dump.

   Where `<PERSONA>` is one of `ontologist`, `contrarian`, `simplifier`, and `<PREFIX>` is from step 2.

4. **Verify each critique exists.** For each of the three expected critique files, check whether it was written. For any file that is missing, extract the dumped content from that sub-agent's reply and write it directly using the Write tool in the parent session.

5. **Confirm and headline.** After all three critiques exist on disk, report a one-line headline per persona (no cross-cutting analysis — `/spec revise` produces the structured breakdown from the same files):

   > Three critiques written (`<PREFIX>-{ontologist,contrarian,simplifier}.md`).
   > - **Ontologist:** <one line>
   > - **Contrarian:** <one line>
   > - **Simplifier:** <one line>

6. **Update state.yaml:** `phase: in-review`, `latest_review_stamp: <PREFIX>`, `last_command: /spec review`, `last_command_at: <timestamp>`.

7. **Propose commit.** Per the global commits policy in `SKILL.md`, review produces archive snapshots and transitions phase, so propose (do not run) the exact commands — both the three critique files and `state.yaml` in one atomic commit:

   ```
   git add spec/archive/<PREFIX>-ontologist.md spec/archive/<PREFIX>-contrarian.md spec/archive/<PREFIX>-simplifier.md spec/state.yaml
   git commit -m "spec: review iteration <N> (<PREFIX>)"
   ```

   Substitute the concrete `<PREFIX>` from step 2 and `<N>` from state.yaml.

8. **Next step.** Tell the user to commit (or skip and batch later), then run `/spec revise` — it will read the critique files and produce the structured per-concern breakdown to drive the revision dialogue.
