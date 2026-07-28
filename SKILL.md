---
name: mermaid-diagram
description: >
  Generate a validated Mermaid diagram from a content description. Covers all 23 Mermaid diagram
  types with a decision matrix, the full v11 node shape vocabulary (including the @{ shape } API),
  classDef semantic colour patterns, and mmdc syntax validation. Use whenever any skill or task
  needs to produce a Mermaid diagram — concept maps, architecture diagrams, sequence flows, state
  machines, timelines, ER diagrams, C4 views, quadrant charts, and more. Invoke instead of
  generating Mermaid inline when diagram quality matters.
---

Read `{SKILL_ROOT}/references/mermaid-diagrams.md` before writing any syntax — even when the
diagram type is already known, the reference governs node shapes, classDef patterns, and
anti-patterns to avoid.

---

## Step 1 — Receive Input and Parse Explicit Overrides

The caller provides:
- A description of what the diagram should convey (content + purpose)
- Any explicit user constraints (e.g., "use a sequence diagram", "blue for the API nodes",
  "dark theme", "left-to-right layout")

**Parse overrides before making any decisions.** For each diagram dimension, check whether
the user has specified it:

| Dimension | If user specified | If not specified |
|---|---|---|
| Diagram type | Use it; skip type-selection | Select from the decision matrix |
| Colour / theme | Apply to `classDef` or `%%{init}%%` | Use the semantic palette from `mermaid-diagrams.md` |
| Layout direction | Honour it | Apply direction heuristics from `mermaid-diagrams.md` |
| Node count / max complexity | Honour it | Default to ~15 nodes max |

**The override rule:** an explicit user instruction replaces only the specific decision it
addresses. All other decisions follow the skill's normal logic. A user saying "make it a sequence
diagram" bypasses type-selection but does not bypass node labelling, `autonumber`, `classDef`
colour coding, or `mmdc` validation.

---

## Step 2 — Read the Reference

Read `{SKILL_ROOT}/references/mermaid-diagrams.md`.

It contains the full decision matrix, node shape vocabulary (`@{ shape }` API), `classDef`
semantic patterns, `linkStyle` patterns, direction heuristics, and anti-patterns. Every
dimension not specified by the user is decided from this reference.

---

## Step 3 — Make Remaining Decisions

For each dimension not overridden by the user, decide now and write down the choices before
generating:

- **Diagram type** — from the decision matrix (Section 1 of the reference)
- **Node shapes** — from the shape vocabulary (Section 2); prefer `@{ shape }` API for semantic clarity
- **`classDef` colours** — from the semantic palette (Section 4); apply to problem/solution/process/agent/data/external nodes
- **Edge labels** — every edge must have a label; unlabelled edges are an anti-pattern
- **Direction** — from heuristics (Section 6): `LR` for pipelines, `TD` for hierarchies
- **Theme** — `%%{init: {'theme': 'neutral'}}%%` by default
- **`autonumber`** — always include for `sequenceDiagram`

---

## Step 4 — Generate the Diagram

Write the Mermaid block applying all resolved decisions. Keep it under ~15 nodes.
Refer to the anti-patterns section of the reference and explicitly avoid each one.

---

## Step 5 — Validate with `mmdc`

Write the diagram source to a temp file and validate:

```bash
mmdc -i /tmp/mermaid-diagram.mmd -o /tmp/mermaid-diagram.svg 2>&1
```

**On failure:**
1. Read the error message — it identifies the exact line and token that failed
2. Fix the syntax
3. Re-validate

Repeat up to **2 retry cycles**. If still failing after 2 retries, return the diagram with
an inline warning comment noting the validation failure and the error.

---

## Step 6 — Return the Validated Block

Return the diagram as a fenced Mermaid code block:

````
```mermaid
<diagram source>
```
````

If the caller is a skill embedding this in a larger document, return only the block —
no surrounding prose unless the caller asked for an explanation.

---

## Future: Interactive Refinement

An optional interactive flow (preview in browser → refine → save to disk) is planned as a
secondary mode. It is not implemented.

See `{SKILL_ROOT}/references/future-interactive-flow.md` for design notes, a detailed account
of how the superseded `draw-mermaid-diagram` skill approached this, and evaluation questions
to answer when this feature is built.
