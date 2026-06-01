---
name: clean-room-resume
description: PHASE B continuation for clean-room reverse engineering. Resume an in-progress rebuild in a FRESH context by reading REBUILD-PROGRESS.md (the append-only handoff journal) plus rebuild-docs/, then continuing the build loop from the next step. TRIGGER when a clean-room rebuild was paused/handed off (context budget hit, session ended) and the user wants to continue building, or whenever REBUILD-PROGRESS.md exists and work remains. Reads only rebuild-docs/ — never the original source.
---

# Clean-Room Resume (Phase B continuation)

A clean-room rebuild is too large for one context window, so it proceeds in segments,
checkpointing to `REBUILD-PROGRESS.md` (append-only) at every module and at each context
handoff. This skill is how a **fresh** orchestrator picks the build back up with full
knowledge of what came before — the journal is the entire handoff; you were not present for
the earlier segments and must not assume anything not written there.

The hard rules are unchanged: ground truth is `./rebuild-docs/`; **no agent reads the
original source**; the `cleanroom-architect` is the Truth oracle; coding subagents get only
scoped packets. The original clone is off-limits (and likely deleted).

## Resume procedure

1. **Read the handoff.** Read `REBUILD-PROGRESS.md` in full (it is append-only, so the whole
   file is the history). From it determine:
   - which build-plan steps are already complete (collect every entry's step id),
   - the **`Next:`** pointer from the most recent entry / the last `--- HANDOFF ---` marker,
   - all `Decisions`, `Constraints/caveats`, and open `GAPs` you must honor going forward.
   Also read `00-MANIFEST.md` and `09-build-plan.md` from `./rebuild-docs/` for the full
   ordered step list and inventory counts.
2. **Reconcile.** Cross-check completed entries against the build plan to confirm the next
   step. If the journal and the on-disk code disagree (e.g. a step marked done is missing),
   trust the code state, note the discrepancy in a new progress entry, and re-derive the next
   step. If `Next:` is ambiguous or absent, pick the earliest build-plan step with no
   completed entry.
3. **Continue the loop.** From that next step, run the exact same per-step loop as
   `/clean-room-rebuild` — for each remaining step:
   architect packet → resolve any `GAP:` (for a blocking missing *fact*, use the consent-gated
   **`clean-room-consult-source`** escape hatch, never direct source reads) → spawn a fresh
   coding subagent (scope-fenced, no source; it returns `NEED:` instead of guessing) →
   integrate → architect judges acceptance (retry on FAIL) → **append a progress entry**.
4. **Respect the same context budget.** Monitor your context usage; at **~200,000 tokens**
   raise `HANDOFF_NEEDED`, finish the in-flight module, append a complete progress entry plus
   a `--- HANDOFF (context reset) after step <id> — resume with /clean-room-resume ---`
   marker, then stop and tell the user to continue in a fresh session with
   `/clean-room-resume`. Each segment honors `Decisions`/`Constraints` recorded by earlier
   segments so the rebuild stays internally consistent across handoffs.
5. **Finish.** When no build-plan steps remain, run the full `10-acceptance.md` surface-parity
   check via the architect, append a final `### [done]` entry, and report results plus any
   unresolved `GAPs`.

Always append, never rewrite. The journal must remain a faithful, ordered record so the next
fresh context can resume exactly as you did.
