# Future Feature: Interactive Preview → Refine → Save

This file documents the design intent for the interactive flow and preserves a detailed account
of how the now-removed `draw-mermaid-diagram` skill implemented it — so that when this feature
is built, that prior approach can be evaluated and either adopted, adapted, or replaced.

---

## Intended Flow (Not Yet Implemented)

After the automated generation + validation cycle completes, the skill would optionally enter
an interactive mode:

1. Open the diagram in a browser tab with live reload enabled
2. The user inspects it and requests changes in natural language
   (e.g., "move the database to the right", "add an error path from API to Client")
3. The skill updates the Mermaid source, re-validates with `mmdc`, and the browser refreshes
4. Repeat until the user is satisfied
5. The user approves → the diagram is saved to a specified path on disk

This mode is user-facing and session-driven. It is distinct from the primary automated flow,
which is caller-driven and returns a block without any browser interaction.

---

## How `draw-mermaid-diagram` Approached This

The skill installed at `~/.claude/skills/draw-mermaid-diagram/` (now removed) implemented
an interactive-only flow using two MCP tools from the `veelenga/claude-mermaid` npm package:

### `mermaid_preview`

Renders the diagram and opens a browser tab with live reload via WebSocket.

Parameters:
- `diagram` — the Mermaid source string
- `preview_id` — a descriptive kebab-case identifier (e.g., `auth-flow`, `architecture`).
  Reusing the same ID on subsequent calls updates the existing browser tab rather than opening
  a new one — this is the core of the iterative refinement loop.
- `format` — `svg` for live reload (PNG does not support it)
- `theme` — `default`, `forest`, `dark`, or `neutral`
- `background` — `white`, `transparent`, or a hex colour
- `width`, `height`, `scale` — size and quality controls

### `mermaid_save`

Saves the last-previewed diagram to disk.

Parameters:
- `save_path` — destination path (e.g., `./docs/diagram.svg`)
- `preview_id` — must match the ID used in the preceding `mermaid_preview` call
- `format` — must match the format from the preview

### Workflow

```
mermaid_preview(preview_id="X") → browser tab opens
→ user feedback
→ mermaid_preview(preview_id="X") again → same tab refreshes
→ repeat
→ mermaid_save(preview_id="X") → file saved
```

### Limitations of the `draw-mermaid-diagram` Approach

- **Interactive-only**: had no automated generation mode; required a human in the loop for
  every diagram
- **No type-selection logic**: defaulted to whatever the LLM chose (usually `graph TD`)
- **No `classDef` guidance**: used only inline `style id fill:#...` on individual nodes
- **No v11 syntax**: covered only 5 of 23 diagram types, no `@{ shape }` API, no `%%{init}%%`
- **No validation**: relied entirely on browser rendering to surface syntax errors; silent
  failures were common
- **MCP dependency**: required the `veelenga/claude-mermaid` MCP server to be running locally.
  The server was not configured in this environment, making the skill entirely non-functional.

---

## Evaluation Questions for When This Is Built

Before implementing the interactive flow, answer:

1. **MCP or `mmdc --watch`?**
   The `veelenga/claude-mermaid` server opens a browser and live-reloads via WebSocket.
   `mmdc` has no built-in watch mode, but a shell loop (`while inotifywait ...; do mmdc ...; done`)
   could approximate it. Is the MCP server the right dependency to introduce, or is a simpler
   approach sufficient?

2. **Optional or always-on?**
   Should interactive mode be triggered explicitly by the user (e.g., a `--interactive` flag or
   a follow-up prompt), or should the skill always offer it after automated generation?

3. **Same skill or a separate extension?**
   Keeping both flows in one SKILL.md keeps the skill self-contained but makes it longer.
   A separate `mermaid-diagram-interactive` skill or a sub-command pattern might be cleaner.

4. **MCP server availability as a runtime check?**
   If the MCP server is absent, the skill should degrade gracefully to automated-only rather
   than failing. Should availability be checked at invocation time or assumed from configuration?

5. **Save format?**
   `draw-mermaid-diagram` saved to SVG. Should the interactive flow save SVG, PNG, or embed
   the Mermaid source in the output document? Consider that SVG is rendered by browsers but
   not by GitHub or Confluence natively without a build step.
