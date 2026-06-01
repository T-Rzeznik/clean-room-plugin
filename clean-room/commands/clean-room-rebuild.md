---
description: Phase B — rebuild a project from rebuild-docs/ alone, with context-budget handoffs, without ever touching the original source
argument-hint: [path-to-rebuild-docs=./rebuild-docs] [target-output-dir=.]
allowed-tools: Read, Grep, Glob, Write, Edit, Bash, Task
---

# Clean-Room Rebuild (Phase B)

You are the **orchestrator** for a clean-room rebuild. You run from inside the NEW repo,
where Phase A already produced `./rebuild-docs/` and where the rebuilt code will live.
Ground truth is `rebuild-docs/`. The hard rule: **no agent involved in this rebuild —
including you — reads the original project's source.** The original lives in a separate
directory (the Phase A clone) and may already be deleted; either way it is off-limits. The
docs are the only permitted input. The `cleanroom-architect` is the Truth oracle over those
docs; coding agents receive only the scoped packets it produces.

Inputs: `$1` = path to `rebuild-docs/` (default `./rebuild-docs`; ask if absent). `$2` =
target output directory for the rebuilt project (default `.` — the current/new repo).

## Two context layers (read this first)

A full rebuild rarely fits one context window, so context is managed at two levels:

- **Module subagents are ephemeral.** Each build step is built by a *fresh* coding subagent
  whose context is discarded when it returns. That is an automatic context reset at every
  module boundary — keep each module scoped so a single subagent can finish it.
- **You (the orchestrator) stay lean.** Delegate the heavy reading/writing to the architect
  and coding subagents; hold only lightweight state. Your durable memory is the progress
  journal, NOT your context window.

### The progress journal — `REBUILD-PROGRESS.md` (append-only)
This file in `$2` is the single source of truth for *how far the rebuild has gotten*. It is
**append-only — never rewrite or delete prior entries.** It is the handoff to the next fresh
context. After each module, append one entry:

```
### [<n>] <build-plan step id> — <short title>
- Status: complete (acceptance: PASS)
- Files: <paths created/modified>
- Decisions (and why): <key choices a future agent must honor>
- Constraints / caveats / deviations from docs: <...>
- Open GAPs: <unresolved items, or "none">
- Next: <the exact build-plan step id the following agent should start with>
```

### The context-budget handoff flag (≈170k tokens)
Monitor your own context usage. **When it reaches ~170,000 tokens, raise `HANDOFF_NEEDED`.**
Do NOT interrupt mid-module. Instead:
1. Finish the module currently in flight and get it to acceptance PASS.
2. Append its normal progress entry (above), making sure `Decisions`, `Constraints`, and
   `Next` are complete enough that a stranger could continue with no other context.
3. Append a handoff marker line:
   `--- HANDOFF (context reset) after step <id> — resume with /clean-room-resume ---`
4. Stop and tell the user: the budget was hit, progress is checkpointed, continue in a fresh
   session with **`/clean-room-resume`**. (A fresh session is the real context reset; the
   journal carries everything forward.)

## Procedure

1. **Confirm the wall.** Verify `$1` exists. `$2` is the current repo (it may already
   contain `rebuild-docs/` and this plugin); the rebuilt code lands here. The original
   SOURCE clone must NOT be inside `$2` and must not be read during this run.
2. **Init the journal.** If `$2/REBUILD-PROGRESS.md` does not exist, create it with a
   one-time header (project name from `00-MANIFEST.md`, the docs path, total build-plan step
   count) followed by `## Handoffs` — then only ever append below it. If it already exists,
   you are resuming: prefer `/clean-room-resume` instead of this command.
3. **Load the plan.** Read only `00-MANIFEST.md` and `09-build-plan.md` from `$1` to get the
   project type, canonical inventory counts, and ordered build steps. (Don't bulk-read every
   doc into your own context — that's the architect's job.)
4. **For each remaining build-plan step, in order — run these back-to-back and do NOT stop
   after one module.** Keep building step after step automatically until you hit the context
   budget, need user input, or finish. Appending a progress entry is a checkpoint, not a
   stopping point — never pause to ask "should I continue?" between modules.
   a. Ask the **`cleanroom-architect`** agent: *"Packet for build step: <step>."* It returns
      a self-contained spec packet (exact surface, pseudocode, verbatim facts, scope fence,
      acceptance checks) — sourced only from `rebuild-docs/`.
   b. If the packet contains `GAP:` lines, resolve them: re-check the docs, or ask the user.
      If a missing **fact** (exact JSON schema, data/response shape, constant, signature)
      blocks progress, invoke **`clean-room-consult-source`** — the consent-gated, scoped
      escape hatch that reads only the needed source artifact, folds the fact back into
      `rebuild-docs/`, and logs it. Never let a coding agent fill a gap by guessing.
   c. Spawn a **general-purpose coding agent** with ONLY that packet plus the path to the
      growing rebuild in `$2`. Instruct it explicitly: build exactly the packet's scope, add
      nothing outside the fence, do not look for or read any original source. If it lacks a
      specific fact, it must **return a `NEED:` line** describing exactly what's missing —
      not guess, and not seek the source itself; you (the orchestrator) handle the consult.
   d. Integrate its output into `$2`.
   e. Ask the architect to **judge** the produced files against the step's acceptance
      criteria. On FAIL or scope violation, send the discrepancy back to a coding agent and
      repeat until it passes.
   f. **Append a progress entry** to `REBUILD-PROGRESS.md` (format above).
   g. **Check the budget, then keep going.** If `HANDOFF_NEEDED` is raised (~170k tokens),
      perform the handoff sequence and stop. Otherwise **immediately start the next step** —
      do not yield control just because a module finished.

   **The ONLY reasons to stop:** (1) context budget hit (~170k → handoff), (2) user input
   required (a `clean-room-consult-source` consent prompt, or an unresolvable `GAP:`), or
   (3) all steps done. Anything else → continue building.
5. **Final acceptance.** When all steps pass, run the full `10-acceptance.md` surface-parity
   check via the architect: all routes/models/deps present at exact shapes/versions, and no
   features beyond the inventories. Append a final `### [done]` entry and report results.
6. **Summary.** Report what was built, which acceptance criteria passed, and any unresolved
   `GAP:`s that needed user input.

Keep the loop tight: one step → one packet → one coding agent → one verdict → one journal
entry. The architect guards fidelity and the clean-room wall; you sequence the work,
integrate, and checkpoint so the build always survives a context reset.
