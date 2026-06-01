---
name: clean-room-consult-source
description: PHASE B consent-gated escape hatch for clean-room rebuilds. When a rebuild agent needs a SPECIFIC FACT the docs don't fully provide — an exact JSON schema, a precise request/response or data shape, an exact constant, enum, signature, or copy string — invoke this to request scoped access to the original source. It prompts the USER for approval; on approval it reads ONLY the minimal relevant code, extracts the needed fact under the verbatim rule, writes it back into rebuild-docs/ so future agents never need to re-ask, logs the consult, and returns the fact. On denial it records a GAP. Agents call this autonomously instead of guessing — and must NEVER read the original source any other way.
---

# Clean-Room Consult Source (Phase B escape hatch)

During a rebuild, the wall is absolute: agents build from `rebuild-docs/` and never touch
the original source. This skill is the **one sanctioned exception** — a narrow, consented,
logged crossing for when the docs genuinely lack a *specific fact* needed to proceed. It
exists so the wall stays intact: every crossing is explicit, minimal, user-approved, and
folded back into the docs so it happens at most once per fact.

Reading the original source in any other way breaks the clean room. If you need a fact and
can't get it from the docs or the architect, you call this — you do not go look yourself.

## When to invoke

Invoke only when ALL of these hold:
- A specific fact is **missing or ambiguous** in `rebuild-docs/` and the `cleanroom-architect`
  confirms it (a `GAP:`), AND
- the missing thing is a **FACT** — a JSON schema, exact request/response or data shape, a
  constant/enum value, a public signature, an exact copy string, a design token — not
  procedural logic, AND
- it actually **blocks progress** (you can't make a faithful module without it).

**Do NOT invoke** to obtain procedural logic, algorithms, or implementation bodies — those
must stay paraphrased as pseudocode in `06-logic.md`. The escape hatch is for facts, never a
back door to copy the original's implementation. Exhaust the docs and the architect first.

## Procedure

1. **State the need precisely.** Write: the exact fact required, the doc gap it fills (which
   `NN-*.md` should contain it), why it blocks the current module, and the **smallest source
   artifact** that would resolve it (a specific file, and ideally a specific symbol/region).
2. **Get consent — ask the user.** Use the user-facing question tool to request approval.
   Present three things: *what* you need, the *narrow scope* you propose to read (that one
   file/symbol — not a survey), and the source location. Offer options:
   - **Approve** — read the proposed scope at the known source path.
   - **Approve, but I'll give the path / re-clone** — the clone may have been deleted; the
     user supplies a path or re-clones the original.
   - **Deny** — do not read source.
3. **On approval — scoped read only.** Read ONLY the minimal artifact you named (targeted
   Grep/Read of the specific file/symbol/lines). Do not browse, do not read neighbors "for
   context." Apply the **verbatim rule**: copy the fact verbatim (it is an interface fact);
   if resolving it requires any procedural detail, capture that as pseudocode, never pasted.
4. **Fold it back into the docs.** Write the extracted fact into the correct `rebuild-docs/`
   file (schema/shape → `04-data-model.md` or `05-interfaces.md`; constant/enum → `04`/`03`;
   token/copy → `07-ui-style.md`; etc.). If it adds an inventory item, update that file's
   `[INVENTORY]` and the counts in `00-MANIFEST.md`. The docs must now be self-sufficient for
   this fact.
5. **Log the consult.** Append a `SOURCE CONSULT` entry to `REBUILD-PROGRESS.md` (append-only):
   what was needed, the file/symbol read, the fact extracted, and which doc was updated. This
   keeps the rebuild's provenance auditable.
6. **Return the fact** to the calling agent so it continues building.

**On denial:** do NOT read source. Record a `GAP:` in `REBUILD-PROGRESS.md` and the relevant
doc, then either proceed with a clearly-labeled documented assumption or surface the blocker
to the user — never silently guess.

## Guardrails (the wall stays up)
- **Consent every time** — no source read without an explicit user approval for that request.
- **Minimal scope** — only the named artifact; the smallest read that answers the need.
- **Facts, not logic** — interface facts verbatim; procedural detail paraphrased to pseudocode.
- **Fold back + log** — every crossing updates the docs and the journal, so it's one-time and
  auditable. The aim is a wall with a guarded, recorded gate — not a wall with holes.
