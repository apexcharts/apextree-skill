# ApexTree AI Skill

AI coding skill for building [ApexTree](https://apexcharts.com/docs/apextree/) organizational / hierarchical SVG tree charts. Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant.

> **Separate skill — one of the ApexCharts ecosystem skills.** This is the dedicated skill for **ApexTree** (`apextree`), shipped as its own `apextree-skill` package and repo — distinct from the core `apexcharts-skill` and the other product skills. Each product has its own library and skill; use the one that matches yours:
>
> | Product | npm library | Skill package & repo |
> |---|---|---|
> | ApexCharts — charts | `apexcharts` | [`apexcharts-skill`](https://github.com/apexcharts/apexcharts-skill) |
> | ApexGantt — Gantt / timeline | `apexgantt` | [`apexgantt-skill`](https://github.com/apexcharts/apexgantt-skill) |
> | **ApexTree** — hierarchy / org charts · *this skill* | `apextree` | `apextree-skill` |
> | ApexSankey — flow / Sankey | `apexsankey` | [`apexsankey-skill`](https://github.com/apexcharts/apexsankey-skill) |
> | Apex Grid — data grid | `apex-grid` | [`apexgrid-skill`](https://github.com/apexcharts/apexgrid-skill) |

## What This Does

AI models routinely get tree-chart code wrong: passing data to the constructor, missing `children: []` on leaves, treating `enableSelection` as a boolean, hand-rolling org-card markup when the built-in template covers it. This skill ships structured reference files so the assistant generates correct ApexTree code on the first try.

### Coverage

- **`NestedNode` data shape** — `id` / `name` / `children` rules, recursive children, per-node options
- **Org-card mode** — `contentKey: 'data'` and the built-in avatar / title / badge template
- **Custom `nodeTemplate`** — receiving the value at `contentKey`
- **Lifecycle** — `setLicense()`, construct, `render()`, the returned `Graph`, `destroy()`
- **Graph API** — collapse/expand, layout direction, search, breadcrumb, selection, exporting
- **Framework wrappers**: `react-apextree`, `vue-apextree`, `ngx-apextree`

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apextree-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apextree-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### As an npm dependency

```bash
npm install apextree-skill
```

```js
import { skillFile, referencePath } from 'apextree-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const api   = await readFile(referencePath('graph-api.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                         # Main entry point
├── .cursorrules                     # Self-contained Cursor / Windsurf version
├── references/
│   ├── data-format.md               # NestedNode, org-card, nodeTemplate
│   ├── graph-api.md                 # collapse/expand, search, selection, layout
│   └── framework-wrappers.md        # React, Vue, Angular
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [ApexTree Documentation](https://apexcharts.com/docs/apextree/)
- [ApexTree GitHub](https://github.com/apexcharts/apextree)
- [npm: apextree](https://www.npmjs.com/package/apextree)
- [react-apextree](https://www.npmjs.com/package/react-apextree)
- [vue-apextree](https://www.npmjs.com/package/vue-apextree)
- [ngx-apextree](https://www.npmjs.com/package/ngx-apextree)

## License

MIT
