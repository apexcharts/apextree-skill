---
name: apextree
description: >
  AI skill for building ApexTree organizational / hierarchical SVG tree charts.
  Use whenever the user asks to create, configure, render, or troubleshoot an
  org chart, hierarchy diagram, family tree, decision tree, file-system tree,
  or directional tree visualization with `apextree`. Covers the recursive
  `NestedNode` data shape, five grow directions (including `'radial'` with
  `layoutType: 'cluster'` dendrograms), spring motion, live `updateData()`
  diffs, focus mode, expandable cards, semantic zoom, lazy children, edge
  styles, the `'data'` contentKey for org-card layouts, custom `nodeTemplate`,
  search / breadcrumb / selection APIs, family `--apx-*` theme tokens, and
  framework integration (React / Vue / Angular). In
  React / Vue / Angular projects, prefer the framework wrapper packages
  (`react-apextree`, `vue-apextree`, `ngx-apextree`) over the core API.
metadata:
  author: ApexCharts
  version: "1.5.1"
  library_version: "2.1.1"
  category: data-visualization
  tags: [tree, hierarchy, org-chart, diagram, charts, svg, apextree]
  docs: https://apexcharts.com/docs/apextree/
  npm: apextree
  github: https://github.com/apexcharts/apextree
---

# ApexTree AI Skill

> **Framework wrapper detection — check `package.json` before generating code.**
> - `react` → use **`react-apextree`** instead of the core API.
> - `vue` → use **`vue-apextree`**.
> - `@angular/core` → use **`ngx-apextree`**.
>
> Wrappers handle `destroy()` automatically on unmount, accept reactive props, and forward events as idiomatic framework events. Use the core API directly only when no framework is detected, or when the user explicitly asks for vanilla. See `references/framework-wrappers.md`.

## 1. Critical Rules

1. **Data is a single `NestedNode`** (the root) passed to `render(data)`, **not** to the constructor. Constructor takes `(element, options)`; `render(rootNode)` paints.
2. **Every node needs `id`, `name`, and `children`.** Leaves have `children: []` — not `undefined` and not omitted.
3. **`id` must be unique across the entire tree.** Selection, edge highlighting, search, and breadcrumb all key off `id`.
4. **`render(data)` returns a `Graph` instance.** Save it: it exposes `collapse`, `expand`, `updateData`, `focus`, `fitScreen`, `centerOnNode`, search APIs, and selection APIs.
5. **`enableExpandCollapseZoom: true` (default)** re-fits the viewBox on collapse / expand, capped by `maxZoomNodeSpan` (default `8`) so a view of a few remaining nodes doesn't balloon. `maxZoomNodeSpan: 0` restores the pre-2.0 tight fit; `enableExpandCollapseZoom: false` keeps the camera locked.
6. **`enableSelection` is `'single' | 'multi' | false`**, not a boolean. `false` means selection is off; `'single'` and `'multi'` are the active modes.
7. **`onSelectionChange(listener)` is on the `Graph`** returned by `render`, not on `tree` and not in `options`.
8. **Use `contentKey: 'data'`** to switch the built-in template into org-card mode (avatar, name, title, subtitle, badge, accent stripe). Don't write a custom `nodeTemplate` for that — the built-in one is already there.
9. **For custom rendering use `nodeTemplate(content, context?)`** where `content` is the value at `contentKey` and the optional `context` carries `{ direction, cardImagePosition, expanded, lod }`. Set `contentKey: 'data'` for structured payloads.
10. **`direction` is `'top' | 'bottom' | 'left' | 'right' | 'radial'`** and controls where the root sits and which way the tree grows. `'radial'` centers the root with each depth on a ring; combine with `layoutType: 'cluster'` for a dendrogram (all leaves on the outer ring).
11. **Per-node options** live on `node.options` and can override font, border, tooltip, and node visuals for a single node, including `nodeWidth` / `nodeHeight` since 2.0, so mixed-size siblings lay out correctly.
12. **Call `destroy()`** before dropping the instance in React / Vue / Angular. As of 2.1.0 it also stops the spring animation loop and detaches every listener; it is idempotent. Without it, a tree torn down mid-animation keeps its frame loop running against detached DOM.
13. **Set the license key once** at app startup with `ApexTree.setLicense('KEY')` to remove the watermark.
14. **Motion is on by default since 2.0.** Springs drive layout, so node positions read from the DOM right after `render()` are the seed (nodes start stacked on the root), not the settled layout. Set `enableAnimation: false` for synchronous final positions (the 1.15 behavior), or wait for the springs to settle before measuring.
15. **Use `graph.updateData(newData)` for live data.** It diffs against the live tree and animates; collapse state, selection, focus, and expanded cards all survive. `graph.construct(newData)` is a hard rebuild that drops that state.
16. **The collapsed-descendant count renders inside the expand/collapse button** (which widens into a pill) since 2.0. There is no separate badge element; `collapseBadge*` options style the pill.

---

## 2. Data Format — `NestedNode`

```ts
interface NestedNode<T = undefined> {
  id: string;                                          // REQUIRED, unique across the whole tree
  name: string;                                        // REQUIRED, display label (when contentKey is 'name')
  children: NestedNode<T>[];                           // REQUIRED — empty array for leaves
  hasChildren?: boolean;                               // advertise unloaded children (lazy loading via loadChildren)
  data?: T;                                            // arbitrary payload, used when contentKey: 'data'
  options?: Partial<FontOptions & NodeOptions & TooltipOptions>;   // per-node overrides (incl. nodeWidth / nodeHeight)
}
```

Minimal example:

```js
import { ApexTree } from 'apextree';

const data = {
  id: '1', name: 'CEO',
  children: [
    { id: '2', name: 'CTO', children: [
      { id: '4', name: 'Eng Lead', children: [] },
      { id: '5', name: 'QA Lead',  children: [] },
    ]},
    { id: '3', name: 'CFO', children: [] },
  ],
};

const tree = new ApexTree(document.getElementById('chart'), {
  width: 800, height: 600,
  nodeWidth: 120, nodeHeight: 40,
  childrenSpacing: 70, siblingSpacing: 30,
  direction: 'top',
});

const graph = tree.render(data);
```

### Org-card mode (built-in `'data'` template)

Set `contentKey: 'data'` and put structured fields into `node.data`. The built-in template renders avatar / name / title / subtitle / badge / accent stripe (and optional `meta` icon+label rows) automatically. Since 2.0 the payload may also carry `tags` (keyword chips, always visible) plus `stats`, `progress`, `actions`, and `details`, which are revealed only when the card is expanded in place (see expandable cards in `references/graph-api.md`):

```js
const data = {
  id: 'ceo', name: 'Alice',
  data: {
    name: 'Alice Johnson',
    title: 'Chief Executive Officer',
    subtitle: 'Executive',
    imageURL: 'https://example.com/alice.jpg',
    accentColor: '#6366f1',
    badge: { text: 'Active', color: '#EEF2FF' },
    meta: [{ icon: 'bi bi-envelope', label: 'alice@corp.com' }, { label: 'ext. 4021' }],  // extra icon+label rows
  },
  children: [],
};

new ApexTree(el, {
  contentKey: 'data',
  nodeWidth: 220, nodeHeight: 80,
}).render(data);
```

### Custom `nodeTemplate`

```js
new ApexTree(el, {
  contentKey: 'data',
  nodeWidth: 200, nodeHeight: 80,
  nodeTemplate: (c) => `
    <div style="display:flex;align-items:center;gap:8px;padding:8px;">
      <img src="${c.img}" style="width:32px;height:32px;border-radius:50%;" />
      <div>
        <div style="font-weight:600;">${c.name}</div>
        <div style="font-size:11px;color:#666;">${c.role}</div>
      </div>
    </div>`,
}).render(data);
```

> The optional 2nd arg carries layout context: `nodeTemplate: (c, { direction, cardImagePosition }) => …` — adapt avatar placement or flip the layout by growth direction without reaching for globals.

### Per-node style overrides

```js
const data = {
  id: 'ceo', name: 'CEO',
  options: { nodeBGColor: '#EEF2FF', borderColor: '#A5B4FC' },
  children: [
    { id: 'cto', name: 'CTO',
      options: { nodeBGColor: '#ECFDF5', borderColor: '#6EE7B7' },
      children: [] },
  ],
};
```

---

## 3. Top-Level Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `width` / `height` | `number \| string` | `'100%'` / `'auto'` | Canvas size. `'auto'` sizes to content. |
| `viewPortWidth` / `viewPortHeight` | `number` | `800` / `600` | Internal SVG viewport. |
| `direction` | `'top'\|'bottom'\|'left'\|'right'\|'radial'` | `'top'` | Where the root sits. `'radial'` centers the root with each depth on a ring. |
| `layoutType` | `'tree'\|'cluster'` | `'tree'` | Rank placement. `'cluster'` pins every leaf to the deepest rank: outer ring (radial) or bottom row (cartesian), the classic dendrogram look. |
| `contentKey` | `string` | `'name'` | Key on the data object used as the label. Set to `'data'` for structured payloads. |
| `siblingSpacing` / `childrenSpacing` | `number` | `50` / `50` | Horizontal / vertical spacing (px). |
| `nodeWidth` / `nodeHeight` | `number` | `50` / `30` | Node dimensions (px). |
| `theme` | `'light'\|'dark'\|'custom'\|string` | `'light'` | Built-in palette. `'custom'` disables CSS-var injection so host vars win. Any other string (2.1.0) names a theme registered on the shared family registry and behaves like `'custom'` plus that theme's `--apx-*` tokens. See Theming below. |
| `enableAnimation` | `boolean` | `true` | Spring-driven motion (2.0). `false` restores fully synchronous layout: positions are final immediately after `render()`. |
| `motion` | `{ spring?, stagger? }` | `{ spring: 'crisp', stagger: 'wave' }` | Motion tuning. `spring: 'crisp'\|'gentle'\|'snappy'` picks the spring feel; `stagger: 'wave'\|'none'` radiates a reflow outward from the toggled node or moves everything together. |
| `enableExpandCollapse` | `boolean` | `true` | Show expand/collapse buttons on parent nodes. |
| `enableExpandCollapseZoom` | `boolean` | `true` | Re-fit viewBox on collapse / expand. |
| `maxZoomNodeSpan` | `number` | `8` | Caps zoom-in on the collapse/expand re-fit: the fitted view spans at least this many node-widths/heights. `0` disables the cap (pre-2.0 tight fit). Never shrinks the chart below 1:1. |
| `enableToolbar` | `boolean` | `false` | Zoom/pan/export toolbar. |
| `enableSearch` | `boolean` | `false` | Search input in the toolbar. |
| `enableSelection` | `'single'\|'multi'\|false` | `false` | Selection mode. |
| `enableBreadcrumb` | `boolean` | `false` | Path-from-root breadcrumb. |
| `groupLeafNodes` | `boolean` | `false` | Stack leaves vertically. |
| `highlightOnHover` | `boolean` | `true` | Highlight node + connecting edges on hover. |
| `edgeStyle` | `'orthogonal'\|'curved'\|'straight'` | `'orthogonal'` | Connector shape. |
| `edgeColorMode` | `'default'\|'node'` | `'default'` | `'node'` = each edge inherits the child node's `borderColor`. |
| `nodeTemplate` | `(content, context?) => string` | built-in | Custom node HTML. Optional 2nd arg `context` = `{ direction, cardImagePosition, expanded, lod }` (2.0 adds the card-expansion flag and the semantic-zoom tier). |
| `cardImagePosition` | `'left' \| 'top'` | `'left'` | Org-card avatar placement; `'top'` centers the avatar above the text. Also surfaced to `nodeTemplate` via `context`. |
| `enableTooltip` | `boolean` | `false` | Hover tooltip. |
| `onNodeClick` | `(node) => void` | — | Click callback (raw node data). |
| `a11y` | `{ enabled?, label? }` | `{ true, 'Organizational chart' }` | WCAG 2.1 AA + keyboard nav. |
| `locale` | `{ direction?, messages? }` | `{ direction: 'ltr' }` | Text/layout direction and string overrides. `direction: 'rtl'` mirrors the tree; `messages` is a `Partial<TreeMessages>`. See Localization / RTL below. |

### 2.x feature options (all inert by default)

Every option below defaults to off or neutral, so an upgraded 1.15 config renders the same apart from the changed defaults called out in Critical Rules 5, 14, and 16.

| Option | Type | Default | Notes |
|---|---|---|---|
| `autoNodeHeight` | `{ enabled, minHeight?, maxHeight?, extraHeight? }` | `{ enabled: false }` | Measure each card's content and set its height (width stays fixed). Precedence: explicit per-node size, then measured, then global. Returns nothing without a DOM (SSR/jsdom safe). Cartesian directions only; grouped-leaf and radial stay uniform. |
| `cardExpansion` | `{ clickToExpand? }` | `{ clickToExpand: false }` | Expandable cards: a card expands in place to reveal `stats` / `progress` / `actions` / `details` (distinct from expanding children). The `expandCard` / `collapseCard` / `toggleCard` API and the built-in chevron work regardless; `clickToExpand` adds the card-body click gesture. Needs `autoNodeHeight` enabled to actually grow the card. |
| `focus` | `{ clickToFocus?, dimOpacity? }` | `{ clickToFocus: false, dimOpacity: 0.7 }` | Spotlight mode: `graph.focus(id)` dims everything outside the node's lineage and visible subtree and springs the camera to frame it. Escape or `clearFocus()` restores. |
| `semanticZoom` | `{ enabled, compactBelow?, dotBelow? }` | `{ enabled: false }`, `compactBelow: 90`, `dotBelow: 42` | Level-of-detail by on-screen node width: full card, then name + role plate below `compactBelow` px, then a color-slab name plate below `dotBelow` px. Layout and camera never change; only node content re-tiers. Custom templates get `context.lod` (`'full'\|'compact'\|'dot'`). |
| `edgeFlow` | object | `{ color: '#5C6BC0', width: 2, speed: 60, dashLength: 8, gapLength: 6, direction: 'toChild', followFocus: false }` | Styles the animated active path lit by `graph.setActivePath(ids)`. Disabled under reduced motion (edges stay statically highlighted). |
| `enableCommandPalette` | `boolean` | `false` | Cmd/Ctrl-K overlay: fuzzy-jump to a node, expand all, collapse all, fit to screen. Plain DOM overlay, keyboard-navigable, Escape dismisses. |
| `loadChildren` | `(ctx) => Promise<NestedNode[] \| null \| undefined>` | (none) | Lazy children. Mark a node `hasChildren: true` with no loaded `children`; expanding it shows a spinner (`aria-busy`), calls this with `{ id, name, data }`, and splices the result in. Empty array drops the expand affordance; a rejected promise leaves the node collapsed for retry. |
| `nodeWrapper` | `(ctx) => { className?, attributes? } \| void` | (none) | Stamp extra classes / `data-*` attributes on each node's wrapper `<g>` (context menus, drag handles, framework boundaries). Identity attributes ApexTree owns are ignored. Per-node `options.nodeWrapper` wins over the global hook. Context: `{ id, name, data, depth, collapsed, expanded, hasChildren }`. |
| `countBadgeEnabled` | `boolean` | `false` | Always-visible count badge on every node (independent of collapse state). Tune with `countBadgeSource: 'descendants'\|'children'\|'data'` (default `'descendants'`), `countBadgeDataKey` (default `'count'`), `countBadgeThreshold` (default `1`), plus `countBadgeBGColor` / `countBadgeFontColor` / `countBadgeFontSize`. |
| `expandCollapseButtonSize` | `number` | `15` | Button diameter (was hardcoded 14 pre-2.0). Purely visual: an invisible hit ring keeps the tap target at `max(size + 4, 24)` px. |
| `expandCollapseButtonIconColor` | `string` | `'#475467'` | Color of the `+`/`-` glyph. Set alongside `expandCollapseButtonBGColor` when theming. |
| `expandCollapseButtonHaloColor` | `string` | `''` (off) | Opaque ring outside the button in the color painted behind the nodes, so the button punches through the card border and incoming edge. Opt-in. |
| `expandCollapseOnNodeClick` | `boolean` | `false` | Clicking anywhere on a node with children toggles its expansion; `onNodeClick` fires after the toggle. |
| `externalLabel.collisionStrategy` | `'none'\|'hide'\|'leaves'` | `'none'` | Thin overlapping external labels per ring in a dense `direction: 'radial'` layout; ignored elsewhere. `'leaves'` also drops inner-node labels. |

### Theming: family `--apx-*` tokens (2.1.0)

Every ApexCharts-family chart reads five shared CSS custom properties, so a page can state its brand once: `--apx-accent` (interactive/selected), `--apx-fore` (text), `--apx-grid` (borders, connectors), `--apx-surface` (background plane), and `--apx-series-1` … `--apx-series-N` (ordered palette, 1-based, stops at the first gap). Declare them on `:root` and they reach every chart on the page.

Resolution precedence (highest first): product CSS variable (`--apex-tree-*`) > explicit option > `--apx-*` token > built-in default. An option set to a value equal to its built-in default is indistinguishable from one left alone, so the token wins there; pin with a product variable if needed.

Named themes: `registerTheme('acme', { tokens: { accent, fore, grid, surface } })` from `@apex/commons` records a token set on a registry shared by the whole family. Reference it via `theme: 'acme'`: behaves like `'custom'` (no CSS injection) plus that theme's tokens, resolved one layer below the CSS cascade. The theme name is always written to `data-apex-tree-theme` on the container.

### Reduced motion

As of 2.1.0 the tree mirrors the OS `prefers-reduced-motion` setting onto the container as the `apextree-reduced-motion` class automatically, from the first render onward. When set, animations are skipped and elements appear at their final state. The class can also be added by hand to force it.

### Localization / RTL

The `locale` option controls both text direction and every user-facing string. It is fully additive: the default `{ direction: 'ltr' }` with English strings reproduces the pre-i18n output exactly.

```js
new ApexTree(el, {
  direction: 'top',
  locale: {
    direction: 'rtl',
    messages: {
      searchPlaceholder: 'بحث…',
      searchMatchCount: (n) => `${n} نتيجة`,
    },
  },
}).render(data);
```

- **`direction: 'ltr' | 'rtl' | 'auto'`** (default `'ltr'`). `'rtl'` mirrors the tree horizontally and sets `dir="rtl"` on the container, so node text and the search / breadcrumb chrome flow right-to-left; `'auto'` defers to the document/element direction. RTL mirroring is tuned for the vertical (`'top'` / `'bottom'`) growth directions.
- **`messages`** is a `Partial<TreeMessages>` overriding any subset of strings. Unset keys keep their English defaults (exported as `DEFAULT_TREE_MESSAGES`), so you only translate what you need. Plain labels are strings; values that embed runtime data (`searchMatchCount`, `nodeAriaLabel`) are functions, so each locale controls its own grammar and pluralization.

`TreeMessages` keys: `rootAriaLabel`, `searchPlaceholder`, `searchAriaLabel`, `searchMatchCount(count)`, `breadcrumbAriaLabel`, `expandNodeLabel`, `collapseNodeLabel`, `nodeAriaLabel(ctx)`, plus (2.0) `expandCardLabel`, `collapseCardLabel`, `loadingNodeLabel`, and the command-palette strings `commandPalettePlaceholder`, `commandPaletteAriaLabel`, `commandNoResults`, `commandExpandAll`, `commandCollapseAll`, `commandFitScreen`. The `nodeAriaLabel` function receives a `NodeAriaContext` of `{ name, level, position, total, state? }`. The legacy `a11y.label` still overrides `rootAriaLabel` when set.

---

## 4. Lifecycle

```js
import { ApexTree } from 'apextree';

ApexTree.setLicense('KEY');                  // optional, once at app startup

const tree = new ApexTree(el, options);      // 1. construct
const graph = tree.render(rootNode);         // 2. paint — REQUIRED. KEEP the returned Graph.

graph.collapse('node-2');                    // imperative API
graph.expand('node-2');
graph.expandAll(); graph.collapseAll();      // batch verbs (single reflow)
graph.changeLayout('left');                  // switch direction at runtime
graph.fitScreen();
graph.centerOnNode('node-7');
graph.focus('node-7');                       // spotlight; clearFocus() or Escape restores
graph.exportToSvg();

graph.updateData(nextRoot);                  // live data: diff + spring; state survives
graph.construct(newRoot);                    // hard rebuild: replace data + re-render

tree.destroy();                              // before unmount; idempotent (2.1.0)
```

**Re-renders**: prefer `graph.updateData(newData)`: it diffs the tree, animates survivors to their new positions, and keeps collapse / selection / focus / expanded-card state. `graph.construct(newData)` is a full rebuild; `tree.render(newData)` additionally rebuilds the toolbar and chrome, so use it only for the first render or a deliberate hard reset. With `enableAnimation: false`, `updateData` still produces the correct end state, just without motion.

**Measuring layout**: with motion on (the default), node positions read from the DOM right after `render()` are the seed, not the settled layout. Use `enableAnimation: false` when you need synchronous final positions.

---

## 5. Public API

### Tree (`ApexTree`)

| Method | Description |
|---|---|
| `new ApexTree(el, opts?)` | Construct; applies dimensions to `el`. |
| `render(rootNode)` | Paint. Returns a `Graph`. **Required.** |
| `destroy()` | Tear down: stops the spring loop, detaches every listener, releases the chart context (2.1.0). Idempotent. Call before dropping the instance. |
| `getInstanceId()` | Unique chart instance id. |
| `ApexTree.setLicense(key)` | Static; call once at app startup. Keys are ECDSA P-256 signature-verified as of 1.15.0; unsigned keys keep working until 2027-07-31. |

### Graph (returned by `render`)

| Method | Description |
|---|---|
| `updateData(data)` | Diff against the live tree and reconcile: survivors spring to new positions, new ids grow in, departed ones retract. Collapse / selection / focus / expanded-card state survives. The path for live data. |
| `construct(data)` | Replace tree data and re-render (hard rebuild, state dropped). |
| `changeLayout(direction)` | Switch direction at runtime (`'top'\|'bottom'\|'left'\|'right'\|'radial'`). |
| `collapse(id)` / `expand(id)` | Programmatic expand/collapse. |
| `expandAll()` / `collapseAll()` | Expand or collapse every node in one reflow. |
| `expandToDepth(n)` | Show the tree down to depth `n` (root = 0). |
| `expandSubtree(id)` / `collapseSubtree(id)` | Expand or collapse a node and all its descendants in one reflow. |
| `fitScreen()` | Re-fit viewBox to all visible nodes. |
| `centerOnNode(id)` | Pan/zoom the camera to a node, keeping the current zoom. |
| `zoom(factor)` | Step the zoom multiplicatively on the live viewBox (`zoom(0.2)` is 20 percent in). |
| `focus(id)` | Spotlight a node: dim outside its lineage and visible subtree, spring the camera to frame it. Returns `false` for unknown or unrendered nodes. |
| `clearFocus()` / `getFocusedNodeId()` | Remove the spotlight / id of the spotlighted node or `null`. |
| `setActivePath(ids)` | Flow a marching dash along the root-to-node lineage(s). Single id or list; `[]` clears. Styled by `edgeFlow`. |
| `clearActivePath()` / `getActivePath()` | Clear the flow / ids currently flowing. |
| `expandCard(id)` / `collapseCard(id)` / `toggleCard(id)` | Expand a node's card in place to reveal its detail section (distinct from expanding children). |
| `setExpandedCards(ids)` / `getExpandedCards()` | Replace the set of expanded cards in one reflow / list them. |
| `getNodeMap()` | Map of all node ids → node objects. |
| `getRootNodeId()` | Id of the root. |
| `getNodeLabel(id)` | Resolved display label for a node. |
| `findNodesByQuery(q)` | Returns node ids whose labels contain `q` (case-insensitive). |
| `setSearchHighlight(ids)` | Highlight nodes; pass `[]` to clear. |
| `getSelection()` | Currently selected node ids (insertion order). |
| `setSelection(ids)` | Replace selection. In `'single'` mode only the first id is applied. |
| `clearSelection()` | Reset. |
| `onSelectionChange(listener)` | Subscribe; pass `null` to unsubscribe. |
| `setBreadcrumbHandler(fn)` | Click-on-segment callback for the breadcrumb. |
| `exportToSvg()` | Trigger SVG file download. |

---

## 6. Pitfalls — ❌ Wrong vs ✅ Correct

### 1. Data on the constructor
❌ `new ApexTree(el, { id: '1', name: 'A', children: [...] })`
✅ `new ApexTree(el, options).render(rootNode)`

### 2. Missing `children: []` on a leaf
❌ `{ id: '5', name: 'Leaf' }` — runtime errors when iterating.
✅ `{ id: '5', name: 'Leaf', children: [] }`

### 3. Duplicate ids
❌ Two nodes with `id: 'a'` — selection and edge highlighting target the wrong one.
✅ Make ids globally unique across the whole tree.

### 4. Missing `render()`
❌ `new ApexTree(el, opts)` — nothing paints.
✅ `tree.render(data)`.

### 5. Treating `enableSelection` as a boolean
❌ `enableSelection: true` — TypeScript errors / no selection.
✅ `enableSelection: 'single'` or `enableSelection: 'multi'`.

### 6. Subscribing to selection changes via options
❌ `onSelectionChange` doesn't exist on `TreeOptions`.
✅ `graph.onSelectionChange((ids) => …)` after `render()`.

### 7. Custom org-card with a custom template
❌ Hand-rolling avatar HTML when the built-in works.
✅ Set `contentKey: 'data'` and supply `data: { name, title, subtitle, imageURL, accentColor, badge }` — the built-in template renders the org card.

### 8. Forgetting `destroy()` in React / Vue
❌ ResizeObservers leak across unmounts.
✅ Return `() => tree.destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy`.

### 9. Re-rendering the whole tree just to swap data
❌ `tree.destroy(); new ApexTree(el, opts).render(newData)`.
✅ `graph.construct(newData)` — same chart instance, re-renders with new data.

### 10. Mutating the data tree in place
❌ `data.children.push(newChild)` — chart doesn't update.
✅ `graph.construct({ ...data, children: [...data.children, newChild] })`.

### 11. Hovering shows nothing
❌ `enableTooltip` is `false` by default.
✅ `enableTooltip: true` (and optionally pass a `tooltipTemplate`).

### 12. License watermark in production
❌ No license call.
✅ `ApexTree.setLicense('KEY')` once at app startup.

### 13. Measuring node positions right after `render()`
❌ Reading `data-x` / `data-y` or `foreignObject` coordinates synchronously after `render()`: with motion on (the 2.0 default) they hold the seed positions (nodes stacked on the root), not the settled layout.
✅ `new ApexTree(el, { enableAnimation: false }).render(data)` when you need synchronous final positions, or wait for the springs to settle.

### 14. Live data through `construct()`
❌ `graph.construct(newData)` on every websocket tick: a hard rebuild that drops collapse / selection / focus state; nodes jump.
✅ `graph.updateData(newData)`: diffs, animates survivors, state survives.

### 15. Styling or testing the old collapse badge element
❌ Targeting a separate badge element below the button. It no longer exists (2.0); the count renders inside the button, which widens into a pill.
✅ Target the expand/collapse button; `collapseBadge*` options and `--apex-tree-badge-*` variables style the pill.

### 16. Expandable cards that never grow
❌ `cardExpansion` / `toggleCard(id)` with a fixed `nodeHeight`: the card toggles but its height cannot change.
✅ Enable `autoNodeHeight: { enabled: true }` so the expanded card re-measures and siblings spring apart.

---

## 7. Reference Routing Table

| Topic | Reference File |
|---|---|
| `NestedNode` shape, org-card mode, per-node options, `nodeTemplate`, lazy children, per-node sizing | `references/data-format.md` |
| Graph API, `updateData`, batch expand/collapse, focus, active path, expandable cards, search, breadcrumb, selection, layout, exporting, teardown | `references/graph-api.md` |
| React / Vue / Angular wrappers | `references/framework-wrappers.md` |
