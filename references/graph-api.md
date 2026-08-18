# Graph API — Collapse, Search, Selection, Layout, Export

`tree.render(data)` returns a `Graph` instance. Keep it — most runtime control flows through it.

Since 2.0, collapse/expand and data updates reconcile the DOM instead of rebuilding it: nodes spring to their new positions (interruptible, velocity-preserving), and node identity, selection, and focus survive. Tune with `motion: { spring: 'crisp' | 'gentle' | 'snappy', stagger: 'wave' | 'none' }` or disable everything with `enableAnimation: false` (synchronous layout, as in 1.15).

## Layout & view

| Method | Description |
|---|---|
| `changeLayout(direction)` | Switch direction at runtime (`'top'\|'bottom'\|'left'\|'right'\|'radial'`). |
| `fitScreen()` | Re-fit the viewBox to all visible nodes. |
| `centerOnNode(id)` | Pan / zoom the camera to center a node, keeping current zoom. |
| `zoom(factor)` | Step the zoom relative to the live viewBox: `zoom(0.2)` is 20 percent in, negative values zoom out. Never wrongly refused after programmatic camera moves. |

```js
graph.changeLayout('left');
graph.centerOnNode('vp-engineering');
graph.fitScreen();
```

`direction: 'radial'` places the root at the center with each depth on a ring; `layoutType: 'cluster'` pins every leaf to the outer ring (dendrogram). For dense radial charts with external labels, set `externalLabel.collisionStrategy: 'hide'` or `'leaves'` to cull colliding labels per ring.

## Expand & collapse

| Method | Description |
|---|---|
| `collapse(id)` | Hide all descendants of `id`. |
| `expand(id)` | Show all immediate children of `id`. |
| `expandAll()` / `collapseAll()` | Toggle the whole tree in a single reflow. `collapseAll` leaves only the root visible; each level keeps its own collapsed state, so a later `expand` reveals one level at a time. |
| `expandToDepth(n)` | Show the tree down to depth `n` (root = 0). `expandToDepth(1)` shows the root and its direct children. |
| `expandSubtree(id)` / `collapseSubtree(id)` | Expand or collapse a node and every descendant in a single reflow. |

`enableExpandCollapseZoom: true` *(default)* re-fits the viewBox on collapse / expand. Set to `false` to keep the camera locked when toggling subtrees. The re-fit is capped by `maxZoomNodeSpan` (default `8`): the fitted view always spans at least that many node-widths/heights, so collapsing down to two nodes doesn't balloon them to fill the canvas. `maxZoomNodeSpan: 0` restores the old tight fit.

A collapsed node's hidden-descendant count renders inside the expand/collapse button, which widens into a pill (`collapseBadge*` options style it). `expandCollapseOnNodeClick: true` makes the whole node body a toggle.

### Lazy children

Mark a node `hasChildren: true` with no loaded `children` and supply the `loadChildren` option; expanding it shows a spinner (`aria-busy`) while the promise settles, then splices the subtree in through the same reconciler:

```js
new ApexTree(el, {
  loadChildren: async ({ id, name, data }) => fetchReports(id),   // resolve to NestedNode[]
}).render(data);
```

Return `[]` for a node that turned out to have no children (its expand affordance is dropped). A rejected promise leaves the node collapsed so the user can retry.

## Live data: `updateData()`

```js
graph.updateData(nextQuarter);   // diffs, then springs
```

`updateData(data)` diffs the new dataset against the live tree and reconciles: nodes present in both keep their DOM wrapper and spring to their new positions, new ids grow in from their parent, departed ones retract and exit. Collapse state, selection, focus, and expanded cards all survive, as does anything the browser hangs off the surviving elements (keyboard focus, hover, text selection).

This is the path for live data: a websocket tick, a poll, a filter change, stepping between historical snapshots. Use `tree.render(data)` only for the first render (it also rebuilds the toolbar and chrome) and `construct(data)` only for a deliberate hard reset. With `enableAnimation: false`, `updateData` still produces the correct end state, just without motion.

## Focus (spotlight) mode

| Method | Description |
|---|---|
| `focus(id)` | Dim everything outside the node's lineage and visible subtree and spring the camera to frame it. Returns `true` when applied, `false` for an unknown or currently-hidden node. |
| `clearFocus()` | Remove the spotlight and spring the camera back to the whole tree. Escape does the same. |
| `getFocusedNodeId()` | Id of the spotlighted node, or `null`. |

Options: `focus: { clickToFocus: false, dimOpacity: 0.7 }`. `clickToFocus: true` focuses a node on card click (clicking it again clears). Focus survives collapse/expand and layout changes as long as the node stays rendered.

## Active path & edge flow

| Method | Description |
|---|---|
| `setActivePath(ids)` | Light up the lineage from the root down to the given node(s): edges recolor and a dashed pulse flows along them. Single id or list; `[]` clears. Survives collapse/expand. |
| `clearActivePath()` | Restore normal edge styling. |
| `getActivePath()` | Ids currently flowing, or `[]`. |

Style via the `edgeFlow` option, default `{ color: '#5C6BC0', width: 2, speed: 60, dashLength: 8, gapLength: 6, direction: 'toChild', followFocus: false }`. The animation is a marching stroke-dash on the plain SVG edge path (Safari-safe) and is disabled under reduced motion; the active edges then stay statically highlighted.

## Expandable cards

A node's card can expand in place to reveal a detail section (`stats`, `progress`, `actions`, `details` in `OrgNodeData`), distinct from expanding its children:

| Method | Description |
|---|---|
| `expandCard(id)` / `collapseCard(id)` / `toggleCard(id)` | Toggle a single card's detail section. |
| `setExpandedCards(ids)` | Replace the set of expanded cards wholesale, one reflow. |
| `getExpandedCards()` | Ids of every expanded card. |

`cardExpansion: { clickToExpand: true }` toggles on a card-body click; the built-in chevron and the API work regardless. Expansion only grows the card when the height can change, so enable `autoNodeHeight: { enabled: true }`. Custom templates receive an `expanded` flag in `NodeTemplateContext` and can mark their own toggle with `data-apextree-card-toggle`. Cartesian directions only.

## Search

| Method | Description |
|---|---|
| `findNodesByQuery(query)` | Returns node ids whose resolved labels contain `query` (case-insensitive). Empty query returns `[]`. |
| `setSearchHighlight(matchIds)` | Apply highlight state to the DOM (paints lineage to root). Pass `[]` to clear. |

```js
const matches = graph.findNodesByQuery('engineer');
graph.setSearchHighlight(matches);
if (matches[0]) graph.centerOnNode(matches[0]);
```

The built-in search input (enabled with `enableSearch: true`) wires these together automatically.

## Selection

`enableSelection` is **`'single' | 'multi' | false`**, not a boolean.

| Method | Description |
|---|---|
| `getSelection()` | Returns selected node ids in insertion order. |
| `setSelection(ids)` | Replace selection. In `'single'` mode only the first id is applied. |
| `clearSelection()` | Reset. |
| `onSelectionChange(listener)` | Subscribe; pass `null` to unsubscribe. |

```js
const tree = new ApexTree(el, { enableSelection: 'multi' });
const graph = tree.render(data);

graph.onSelectionChange((ids) => {
  console.log('Selected:', ids);
});

// programmatic
graph.setSelection(['ceo', 'cto']);
graph.clearSelection();
```

Selected nodes get `aria-selected="true"` and a visible ring. The listener receives the new ids array on every change.

## Breadcrumb

```js
new ApexTree(el, { enableBreadcrumb: true });
```

The breadcrumb shows the path from root to the most recently clicked node and updates on every click. Subscribe to segment clicks:

```js
graph.setBreadcrumbHandler((nodeId) => {
  if (nodeId === null) return;       // breadcrumb cleared
  graph.centerOnNode(nodeId);
});
```

Pass `null` to remove the handler.

## Data swaps

```js
graph.updateData(newData);           // preferred: diff + animate, state survives
graph.construct(newData);            // hard rebuild, state dropped
```

Prefer `updateData(newData)` (see Live data above). `construct(newData)` is still cheaper than destroy + re-render, but it rebuilds the tree from scratch and drops collapse / selection / focus state.

## Export

```js
graph.exportToSvg();                 // triggers a file download
```

Or trigger from a custom button:

```js
const btn = document.getElementById('export');
btn.addEventListener('click', () => graph.exportToSvg());
```

## Introspection

| Method | Returns |
|---|---|
| `getNodeMap()` | `Record<string, Node>` — all nodes by id. |
| `getRootNodeId()` | id of the root. |
| `getNodeLabel(id)` | resolved display label for a node. |
| `getMessages()` | `TreeMessages` — resolved, localized strings for this chart (English defaults merged with `locale.messages`). |
| `getIsRtl()` | `boolean` — whether the current `locale.direction` resolves to right-to-left. |

Useful when integrating with custom UI (a side panel showing the selected node's children, an export filter, etc.).

`getMessages()` / `getIsRtl()` reflect the `locale` option (added in 1.14.0). The related exported types are `LocaleOptions` (`{ direction?: TextDirection, messages?: Partial<TreeMessages> }`), `TreeMessages`, `NodeAriaContext`, `TextDirection` (`'ltr' | 'rtl' | 'auto'`), and the `DEFAULT_TREE_MESSAGES` constant holding the English defaults.

```js
graph.getIsRtl();       // true when locale.direction resolves to rtl
graph.getMessages();    // { searchPlaceholder, nodeAriaLabel, … }
```

## Keyboard shortcuts & command palette

When the toolbar is rendered with `enableToolbar: true` and `enableSearch: true`:

- **`/`** — focus the search input
- **`Esc`** — clear the search

`enableCommandPalette: true` adds a Cmd/Ctrl-K overlay for fuzzy-jumping to a node (Enter centers it), expanding or collapsing everything, and fitting the view. It is a plain DOM overlay (Safari-safe), keyboard-navigable, dismissed with Escape, and localizable via `locale.messages` (`commandPalettePlaceholder`, `commandPaletteAriaLabel`, `commandNoResults`, `commandExpandAll`, `commandCollapseAll`, `commandFitScreen`).

Wire your own shortcuts via:

```js
graph.setKeyboardShortcutHandlers({
  onFocusSearch: () => searchInput.focus(),
  onClearSearch: () => clearSearch(),
});
```

## Edge styles & coloring

| Option | Values | Effect |
|---|---|---|
| `edgeStyle` | `'orthogonal'` *(default)* | Right-angle elbows with rounded corners (classic org chart look). |
| `edgeStyle` | `'curved'` | Smooth Bézier curve from parent to child (similar to d3-org-chart). |
| `edgeStyle` | `'straight'` | Direct line from parent anchor to child anchor. |
| `edgeColorMode` | `'default'` *(default)* | All edges use `edgeColor`. |
| `edgeColorMode` | `'node'` | Each edge inherits the `borderColor` of the child node it connects into. |

When `groupLeafNodes` is on, the side-bracket connector for stacked leaves stays orthogonal regardless of `edgeStyle`. In `direction: 'radial'`, edges are cubic dendrogram links curved around the center.

## Teardown

```js
tree.destroy();   // before dropping the instance (component unmount, route change)
```

As of 2.1.0, `destroy()` stops the spring animation loop, detaches every listener the graph and its controls installed, and releases the chart context. It is idempotent. Without it, a tree torn down mid-animation leaves its frame loop running against detached DOM until the springs settle.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `enableSelection: true` | `enableSelection: 'single'` or `'multi'` |
| Subscribing to selection in options | `graph.onSelectionChange((ids) => …)` after `render()` |
| Calling `tree.collapse(id)` | `graph.collapse(id)` — collapse/expand are on `Graph`, not on `ApexTree` |
| Running `findNodesByQuery` without painting results | Always pair it with `setSearchHighlight(matches)` |
| Replacing tree data via destroy + construct | Use `graph.updateData(newData)`: same instance, state survives, motion is continuous |
| Reading node positions right after `render()` | With motion on they hold the seed, not the settled layout; use `enableAnimation: false` or wait for the springs to settle |
| Expecting `toggleCard(id)` to grow the card | Enable `autoNodeHeight: { enabled: true }`; a fixed `nodeHeight` cannot change |
