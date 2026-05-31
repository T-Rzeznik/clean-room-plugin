# clean-room

Clean-room "vibe reverse engineering" for Claude Code. Extract any project into a folder
of strict, **bounded** rebuild specs, then reconstruct it 1:1 from those specs alone —
coding agents never see the original source, so the rebuild is independent by construction.

## The two phases

**Phase A — Extract** (`/clean-room-extract` skill)
Reads the original source and writes `rebuild-docs/`:
- `cleanroom-surveyor` (read-only) understands the code → structured SURVEY.
- `cleanroom-scribe` turns the SURVEY into 11 numbered spec docs.

**Phase B — Rebuild** (`/clean-room-rebuild` command)
Points only at `rebuild-docs/` and reconstructs the project:
- `cleanroom-architect` is the *Truth* — the read-only oracle over the docs (never the
  source). It hands scoped spec packets to coding agents and judges their output.
- The command orchestrates: per build step → packet → coding agent → acceptance check.

## What's in rebuild-docs/

```
00-MANIFEST  01-product-spec  02-architecture  03-stack  04-data-model
05-interfaces  06-logic  07-ui-style  08-gotchas  09-build-plan  10-acceptance
```

Three mechanisms make the docs *bounded scaffolding*, not loose notes:
- **Closed inventories** — exhaustive lists (routes, models, deps, env vars, files); the
  rebuild produces exactly those, no more, no less.
- **Scope fences** — every spec doc has `## Out of scope — do NOT build`.
- **Acceptance gates** — per-feature pass/fail checks for self-verification.

**Verbatim line:** interface *facts* (constants, schemas, copy text, design tokens,
signatures) are copied verbatim; procedural *logic* is paraphrased to pseudocode and never
pasted (`06-logic.md`). That keeps the surface exact while keeping the implementation
independently rebuilt.

## Install (local)

```
/plugin marketplace add "C:/Users/tommy/Desktop/Coding Stuff/clean-room-plugin"
/plugin install clean-room@trzeznik-tools
```

## Use

```
/clean-room-extract            # Phase A: generates rebuild-docs/ for the current repo
/clean-room-rebuild ./rebuild-docs ./rebuilt   # Phase B: reconstruct from docs alone
```
