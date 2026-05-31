---
description: Phase B — rebuild a project from rebuild-docs/ alone, without ever touching the original source
argument-hint: [path-to-rebuild-docs] [target-output-dir]
allowed-tools: Read, Grep, Glob, Write, Edit, Bash, Task
---

# Clean-Room Rebuild (Phase B)

You are the **orchestrator** for a clean-room rebuild. Ground truth is `rebuild-docs/`
produced by Phase A. The hard rule: **no agent involved in this rebuild — including you —
reads the original project's source.** The docs are the only permitted input. The
`cleanroom-architect` is the Truth oracle over those docs; coding agents receive only the
scoped packets it produces.

Inputs: `$1` = path to `rebuild-docs/` (ask if missing). `$2` = target output directory
for the rebuilt project (default: a fresh, empty sibling dir — confirm it's empty).

## Procedure

1. **Confirm the wall.** Verify `$1` exists and `$2` is empty (or create it). The original
   source must NOT be inside `$2` and must not be read during this run.
2. **Load the plan.** Read only `00-MANIFEST.md` and `09-build-plan.md` from `$1` to get
   the project type, canonical inventory counts, and ordered build steps. (Don't bulk-read
   every doc into your own context — that's the architect's job.)
3. **For each build-plan step, in order:**
   a. Ask the **`cleanroom-architect`** agent: *"Packet for build step: <step>."* It
      returns a self-contained spec packet (exact surface, pseudocode, verbatim facts,
      scope fence, acceptance checks) — sourced only from `rebuild-docs/`.
   b. If the packet contains `GAP:` lines, resolve them: re-check the docs, or ask the
      user. Never let a coding agent fill a gap by guessing.
   c. Spawn a **general-purpose coding agent** with ONLY that packet plus the path to the
      growing rebuild in `$2`. Instruct it explicitly: build exactly the packet's scope,
      add nothing outside the fence, do not look for or read any original source.
   d. Integrate its output into `$2`.
   e. Ask the architect to **judge** the produced files against the step's acceptance
      criteria. On FAIL or scope violation, send the discrepancy back to a coding agent and
      repeat until it passes.
4. **Final acceptance.** When all steps pass, run the full `10-acceptance.md` surface-parity
   check via the architect: all routes/models/deps present at exact shapes/versions, and no
   features beyond the inventories. Report results.
5. **Summary.** Report what was built, which acceptance criteria passed, and any unresolved
   `GAP:`s that needed user input.

Keep the loop tight: one step → one packet → one coding agent → one verdict. The architect
guards fidelity and the clean-room wall; you sequence the work and integrate.
