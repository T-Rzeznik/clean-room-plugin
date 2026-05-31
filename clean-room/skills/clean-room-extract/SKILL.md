---
name: clean-room-extract
description: PHASE A of clean-room reverse engineering. Read an existing codebase and generate a folder of strict, bounded "rebuild-docs/" that let a fresh agent rebuild the project 1:1 from specs alone. TRIGGER when the user wants to reverse-engineer / clean-room / spec-out a codebase into rebuild documentation, extract a project into rebuild-docs, or prep a faithful clone. This is EXTRACTION ONLY — it reads source and writes docs; the rebuild is Phase B (/clean-room-rebuild).
---

# Clean-Room Extract (Phase A)

Turn an existing project into `rebuild-docs/` — a folder of `.md` specs exhaustive and
bounded enough for a *fresh* agent who will **never see the original source** to rebuild
the project faithfully (1:1).

You may read the original source here. The agent who later rebuilds (Phase B) may not.
The docs are the entire wall between them: anything you fail to capture cannot be rebuilt.

## Prime directive

> **The rebuild's quality equals the docs' quality. Underspecify → the clone diverges;
> overspecify → no agent can hold it. Aim for *complete and bounded*, not *long*.**

What makes these docs **bounded scaffolding** rather than loose notes:

1. **Closed inventories** — every surface is an exhaustive enumerated list (every route,
   command, component, model, dependency+version, env var, file). Rebuild contract:
   *produce exactly these items, no more, no less.*
2. **Scope fences** — every spec doc carries an explicit `## Out of scope — do NOT build`.
3. **Acceptance gates** — `10-acceptance.md` gives per-feature pass/fail checks so the
   rebuild self-verifies.

## Verbatim rule (the clean-room line)

- **Copy verbatim — interface facts:** constants, schemas, config keys, UI copy text,
  design tokens, public signatures, file/dir names, exact error/enum values.
- **Paraphrase to pseudocode — NEVER paste:** procedural bodies, algorithms, business
  logic, control flow. These live in `06-logic.md` as language-neutral pseudocode.

## Workflow this fits into

You run this **from inside the NEW (destination) repo** — the empty repo where the rebuild
will live (and where this plugin is installed). The **SOURCE** is a *separate* local
checkout elsewhere on disk (e.g. a `git clone` of the original in a sibling directory).
Extraction reads the SOURCE and writes `rebuild-docs/` into the CURRENT repo. Everything
after extraction stays in the current repo; the SOURCE is never written to and can be
deleted once `rebuild-docs/` exists.

## How to run it (orchestration)

### Step 0 — Resolve source & output
- **SOURCE** = the path the user gives to the cloned original. **REQUIRED — do NOT default
  to the current repo** (the current repo is the empty destination). If it wasn't given,
  ask for it. Confirm the SOURCE is outside the current repo.
- **OUTPUT** = `rebuild-docs/` in the **current working directory** (the new repo). Never
  write into the SOURCE. Confirm before overwriting an existing `rebuild-docs/`.

### Step 1 — Survey (dispatch the reader)
Spawn the **`cleanroom-surveyor`** agent (read-only) pointed at the SOURCE path. It detects
project type and returns a structured SURVEY: project type + all inventories +
pseudocode-ready notes on logic. Wait for it; the SURVEY is its return value.

### Step 2 — Write the docs (dispatch the writer)
Spawn the **`cleanroom-scribe`** agent. Pass it: the SURVEY from Step 1, the SOURCE path
(read-only, for grabbing exact verbatim facts), and the OUTPUT path `./rebuild-docs` in the
current repo. It writes the fixed doc set into `./rebuild-docs/`:

```
00-MANIFEST.md   01-product-spec.md   02-architecture.md   03-stack.md
04-data-model.md 05-interfaces.md     06-logic.md          07-ui-style.md
08-gotchas.md    09-build-plan.md     10-acceptance.md
```

(The scribe owns the section templates and the verbatim/fence rules.)

### Step 3 — Completeness gate (do not skip)
After the scribe finishes, verify and report:
- [ ] Every source file appears in `02-architecture.md`'s file map.
- [ ] Every manifest dependency is listed with a version in `03-stack.md`.
- [ ] Every route/command/public signature is in `05-interfaces.md`.
- [ ] Every entity / env var / config key is captured.
- [ ] Every spec doc has an `## Out of scope` fence (or states "none").
- [ ] `06-logic.md` contains zero pasted source bodies.
- [ ] `09-build-plan.md` builds only items present in the inventories.
- [ ] No two docs contradict; `00-MANIFEST.md` holds the canonical facts.

Then print a coverage summary: project type, docs written, any N/A docs, inventory
counts (routes, models, deps, env vars), and any gaps you could not resolve.

> If the two agents are unavailable, you may perform Steps 1–2 inline yourself — but keep
> the reader/writer separation in spirit: survey fully *before* writing.

When this is done, the user runs **`/clean-room-rebuild`** (Phase B) from this same repo —
pointed only at `./rebuild-docs/` — to reconstruct the project into this repo. The SOURCE
clone is no longer needed and may be deleted.
