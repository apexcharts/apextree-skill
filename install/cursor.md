# Installing ApexTree Skill for Cursor

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apextree-skill/main/.cursorrules
```

Restart Cursor or open a new window. Cursor automatically reads `.cursorrules` files in the project root as context.

## For Windsurf

Same approach — Windsurf also reads `.cursorrules`.

## Verification

Ask Cursor to generate an ApexTree with multi-select enabled and a selection-change listener. It should set `enableSelection: 'multi'` (not `true`) and call `graph.onSelectionChange(...)` on the value returned by `tree.render()`.
