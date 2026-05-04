# Graph API — Collapse, Search, Selection, Layout, Export

`tree.render(data)` returns a `Graph` instance. Keep it — most runtime control flows through it.

## Layout & view

| Method | Description |
|---|---|
| `changeLayout(direction)` | Switch direction at runtime (`'top'\|'bottom'\|'left'\|'right'`). |
| `fitScreen()` | Re-fit the viewBox to all visible nodes. |
| `centerOnNode(id)` | Pan / zoom the camera to center a node, keeping current zoom. |

```js
graph.changeLayout('left');
graph.centerOnNode('vp-engineering');
graph.fitScreen();
```

## Expand & collapse

| Method | Description |
|---|---|
| `collapse(id)` | Hide all descendants of `id`. |
| `expand(id)` | Show all immediate children of `id`. |

`enableExpandCollapseZoom: true` *(default)* re-fits the viewBox on collapse / expand. Set to `false` to keep the camera locked when toggling subtrees.

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
graph.construct(newData);            // replace the tree, re-render
```

`construct(newData)` is significantly cheaper than destroy + re-render. Use it whenever the tree topology changes.

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

Useful when integrating with custom UI (a side panel showing the selected node's children, an export filter, etc.).

## Keyboard shortcuts

When the toolbar is rendered with `enableToolbar: true` and `enableSearch: true`:

- **`/`** — focus the search input
- **`Esc`** — clear the search

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

When `groupLeafNodes` is on, the side-bracket connector for stacked leaves stays orthogonal regardless of `edgeStyle`.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `enableSelection: true` | `enableSelection: 'single'` or `'multi'` |
| Subscribing to selection in options | `graph.onSelectionChange((ids) => …)` after `render()` |
| Calling `tree.collapse(id)` | `graph.collapse(id)` — collapse/expand are on `Graph`, not on `ApexTree` |
| Running `findNodesByQuery` without painting results | Always pair it with `setSearchHighlight(matches)` |
| Replacing tree data via destroy + construct | Use `graph.construct(newData)` — same instance, much cheaper |
