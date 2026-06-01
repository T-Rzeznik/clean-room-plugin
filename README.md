# clean-room

**Clean-room "vibe reverse engineering" for [Claude Code](https://claude.com/claude-code).**

Extract any project into a folder of strict, **bounded** rebuild specs — then reconstruct
it 1:1 from those specs alone. The agents that do the rebuild never see the original
source, so the result is independent by construction, not just by promise.

This repo is a Claude Code **plugin marketplace** containing one plugin: `clean-room`.

---

## Why

"Clean-room design" means one team documents *behavior* without copying implementation, and
a second team rebuilds *only* from those docs — the docs are a wall between the original and
the clone. This plugin automates both sides of that wall with specialized agents whose
**tool access** enforces the separation:

| Agent | Phase | Can read the original source? |
|-------|-------|-------------------------------|
| `cleanroom-surveyor` | A · read | ✅ reads source → structured survey |
| `cleanroom-scribe`   | A · write | ✅ reads source for verbatim facts → writes docs |
| `cleanroom-architect`| B · truth | ❌ **docs only** — the wall |
| coding agents        | B · build | ❌ spawned with only a scoped spec packet |

The one sanctioned crossing is **`/clean-room-consult-source`** — a consent-gated escape
hatch (below). The agents never read source directly; this is the only door, and it is
guarded, scoped, and logged.

---

## The two phases

**Phase A — Extract** (`/clean-room-extract`)
Reads the original source and writes `rebuild-docs/`:
- `cleanroom-surveyor` (read-only) understands the code → a structured SURVEY.
- `cleanroom-scribe` turns the SURVEY into 11 numbered spec docs.

**Phase B — Rebuild** (`/clean-room-rebuild <path-to-rebuild-docs> <target-dir>`)
Points only at `rebuild-docs/` and reconstructs the project:
- `cleanroom-architect` is the *Truth* — a read-only oracle over the docs (never the
  source). It hands scoped spec packets to coding agents and judges their output.
- The command orchestrates the loop: per build step → packet → coding agent → acceptance.

**Phase B (continued) — Resume** (`/clean-room-resume`)
Real rebuilds rarely fit one context window, so Phase B manages context at two levels:
- **Per-module subagents** are ephemeral — each build step is built by a fresh coding
  subagent whose context is discarded on return (an automatic reset at every module).
- **`REBUILD-PROGRESS.md`** is an *append-only* journal in the new repo. After every module
  the orchestrator appends what it built, the decisions/constraints/caveats a future agent
  must honor, and the next step. When context approaches **~170k tokens**, it finishes the
  current module, writes a handoff entry, and stops. `/clean-room-resume` then continues the
  build in a *fresh* context, reading the journal as its entire handoff. This repeats until
  the build is done — so a project of any size can be rebuilt across many context windows.

**Phase B (continued) — Consult source** (`/clean-room-consult-source`)
The escape hatch for when the docs genuinely lack a *specific fact* (an exact JSON schema,
data/response shape, constant, or signature). A rebuild agent calls it autonomously instead
of guessing; it **prompts you for approval**, and only on consent reads the *one* relevant
source artifact. It copies the fact under the verbatim rule (facts only — never procedural
logic), **folds it back into `rebuild-docs/`** so it's never re-asked, and logs the crossing
in `REBUILD-PROGRESS.md`. The wall stays up — with a single guarded, recorded gate.

---

## What's in `rebuild-docs/`

```
00-MANIFEST     01-product-spec   02-architecture   03-stack       04-data-model
05-interfaces   06-logic          07-ui-style       08-gotchas     09-build-plan
10-acceptance
```

Three mechanisms make the docs **bounded scaffolding**, not loose notes:

- **Closed inventories** — exhaustive lists (every route, model, dependency+version, env
  var, file). The rebuild produces *exactly* those items, no more, no less.
- **Scope fences** — every spec doc ends with `## Out of scope — do NOT build`, so a
  vibe-agent can't hallucinate features the original never had.
- **Acceptance gates** — per-feature pass/fail checks so the rebuild self-verifies.

**The verbatim line:** interface *facts* (constants, schemas, UI copy, design tokens,
public signatures) are copied verbatim; procedural *logic* is paraphrased to pseudocode and
never pasted (`06-logic.md`). The surface stays exact while the implementation is rebuilt
independently.

---

## Install

Requires [Claude Code](https://claude.com/claude-code).

Add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add T-Rzeznik/clean-room-plugin
/plugin install clean-room@trzeznik-tools
```

> `T-Rzeznik/clean-room-plugin` is the GitHub `owner/repo`. You can also pass a full URL
> (`https://github.com/T-Rzeznik/clean-room-plugin`) or a local path to a clone.

To update later:

```
/plugin marketplace update trzeznik-tools
```

---

## Use

The plugin lives in the **new repo** you're building into. The software you're recreating
is cloned into a **separate directory** that stays read-only — and that physical separation
is what enforces the clean-room wall.

```
# 1. Create and enter the new (empty) repo — this is where the rebuild will live.
mkdir my-rebuild && cd my-rebuild && git init

# 2. Install the plugin (see Install above), then clone the ORIGINAL somewhere SEPARATE.
git clone https://github.com/some/original ../original-clone

# 3. Phase A — extract. Point it at the clone; docs are written into THIS repo.
/clean-room-extract ../original-clone
#   → creates ./rebuild-docs/ here. The clone is now optional and can be deleted:
rm -rf ../original-clone

# 4. Phase B — rebuild into THIS repo, from the docs alone.
/clean-room-rebuild
#   (defaults: docs = ./rebuild-docs, output = . )

# 5. If the build hits its context budget and checkpoints, continue in a fresh
#    session as many times as needed — it picks up from REBUILD-PROGRESS.md:
/clean-room-resume
```

Extraction reads the external clone and writes only into the current repo. The rebuild
agents read only `./rebuild-docs` — never the original source (which may already be gone).

---

## Repo layout

```
clean-room-plugin/
├─ .claude-plugin/marketplace.json        # marketplace "trzeznik-tools"
└─ clean-room/                            # the plugin
   ├─ .claude-plugin/plugin.json
   ├─ README.md
   ├─ skills/
   │  ├─ clean-room-extract/SKILL.md         # Phase A orchestrator
   │  ├─ clean-room-resume/SKILL.md          # Phase B resume (continues from REBUILD-PROGRESS.md)
   │  └─ clean-room-consult-source/SKILL.md  # Phase B consent-gated source escape hatch
   ├─ agents/
   │  ├─ cleanroom-surveyor.md            # Phase A reader  (read-only)
   │  ├─ cleanroom-scribe.md              # Phase A writer  (owns the doc templates)
   │  └─ cleanroom-architect.md           # Phase B "Truth" oracle (docs-only)
   └─ commands/clean-room-rebuild.md      # Phase B orchestrator (+ context handoffs)
```

## License

MIT — see [LICENSE](LICENSE).
