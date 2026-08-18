# Data Format — NestedNode, Org-Card, nodeTemplate

## `NestedNode` shape

```ts
interface NestedNode<T = undefined> {
  id: string;                                   // REQUIRED, globally unique across the tree
  name: string;                                 // REQUIRED, label (when contentKey: 'name')
  children: NestedNode<T>[];                    // REQUIRED — empty array for leaves
  hasChildren?: boolean;                        // advertise unloaded children (lazy loading)
  data?: T;                                     // arbitrary payload (used when contentKey: 'data')
  options?: Partial<NodeOptions & FontOptions & TooltipOptions>;   // per-node overrides
}
```

**Key rules:**

- `id` is unique across the *entire* tree (not just within a level). Selection, search, breadcrumb, edge highlighting, and lineage tracking all key off it.
- `children` is **never** omitted on a leaf — pass `children: []`.
- `name` is the default display label. Switch via `contentKey` if your data uses a different field.

## Default `'name'` mode

```js
const data = {
  id: '1', name: 'A',
  children: [
    { id: '2', name: 'B', children: [] },
    { id: '3', name: 'C', children: [
      { id: '4', name: 'D', children: [] },
    ]},
  ],
};

new ApexTree(el, { width: 700, height: 400, nodeWidth: 120, nodeHeight: 40 })
  .render(data);
```

## Org-card mode (`contentKey: 'data'`)

When `contentKey` is `'data'` and the payload contains any of `imageURL`, `title`, `subtitle`, `badge`, or `accentColor`, the built-in template renders a structured org-chart card automatically — no custom `nodeTemplate` needed.

```ts
interface OrgNodeData {
  name?: string;             // primary label (top line)
  title?: string;            // job title (second line)
  subtitle?: string;         // department (third line)
  imageURL?: string;         // avatar URL — rendered as 40×40 circular image on the left
  accentColor?: string;      // colored left-stripe (any CSS color)
  badge?: { text: string; color?: string };   // status chip (upper-right)
  meta?: { icon?: string; label: string }[];  // (1.13.0) extra icon+label rows under subtitle; `icon` is a CSS icon class
  tags?: string[];                            // (2.0) keyword chips under the title, always visible
  stats?: { label: string; value: string }[]; // (2.0) key/value rows, shown only when the card is expanded
  progress?: { value: number; label?: string; color?: string };  // (2.0) meter (value clamped 0-100), expanded only
  actions?: { label: string; href?: string }[];                  // (2.0) links, expanded only
  details?: string;                           // (2.0) free-text paragraph, expanded only
}
```

The expanded-only fields pair with expandable cards: the built-in card shows a chevron when a node has detail content, and `graph.expandCard(id)` / `toggleCard(id)` reveal it (needs `autoNodeHeight` enabled to grow the card; see `references/graph-api.md`).

Avatar placement is controlled by the `cardImagePosition` option (`'left'` default, or `'top'` to center the avatar above the text). A custom `nodeTemplate` receives it — plus the tree `direction` — via its optional 2nd `context` argument: `nodeTemplate: (content, { direction, cardImagePosition }) => …`.

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
  },
  children: [
    {
      id: 'cto', name: 'Bob',
      data: {
        name: 'Bob Lee',
        title: 'CTO',
        subtitle: 'Engineering',
        imageURL: 'https://example.com/bob.jpg',
        accentColor: '#10b981',
      },
      children: [],
    },
  ],
};

new ApexTree(el, {
  contentKey: 'data',
  nodeWidth: 220, nodeHeight: 80,
  childrenSpacing: 80, siblingSpacing: 30,
}).render(data);
```

## Custom `nodeTemplate`

`nodeTemplate(content)` receives the value at `contentKey` (so it's a `string` for `'name'` mode and your `data` object for `'data'` mode). Return an HTML string. The optional 2nd `context` argument carries `{ direction, cardImagePosition }` plus, since 2.0, `expanded` (this card's in-place expansion state; mark a custom toggle with `data-apextree-card-toggle`) and `lod` (`'full' | 'compact' | 'dot'`, the semantic-zoom tier; always `'full'` when `semanticZoom` is off).

```js
new ApexTree(el, {
  contentKey: 'data',
  nodeWidth: 200, nodeHeight: 80,
  nodeTemplate: (c) => `
    <div style="display:flex;align-items:center;gap:8px;padding:8px;height:100%;">
      <img src="${c.img}" style="width:32px;height:32px;border-radius:50%;" />
      <div style="overflow:hidden;">
        <div style="font-weight:600;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;">${c.name}</div>
        <div style="font-size:11px;color:#666;">${c.role}</div>
      </div>
    </div>`,
}).render({
  id: 'ceo', name: 'Alice',
  data: { name: 'Alice', role: 'CEO', img: 'https://...' },
  children: [],
});
```

## Per-node style overrides via `node.options`

Any `NodeOptions` / `FontOptions` / `TooltipOptions` field can be overridden for a single node:

```js
const data = {
  id: 'ceo', name: 'CEO',
  options: {
    nodeBGColor: '#EEF2FF',
    nodeBGColorHover: '#E0E7FF',
    borderColor: '#A5B4FC',
    borderColorHover: '#6366F1',
  },
  children: [
    {
      id: 'cto', name: 'CTO',
      options: { nodeBGColor: '#ECFDF5', borderColor: '#6EE7B7' },
      children: [],
    },
  ],
};
```

This is the right tool for *highlighting* specific nodes (e.g. critical roles) — easier than a custom `nodeTemplate`.

## Per-node sizing (2.0)

`node.options` may also carry `nodeWidth` / `nodeHeight` to override the global size for a single node; edges anchor to each card's own half-size, so mixed-size siblings separate correctly.

For content-driven heights, enable measured sizing globally (or per node):

```js
new ApexTree(el, {
  autoNodeHeight: { enabled: true, minHeight: 0, maxHeight: 0, extraHeight: 0 },  // 0 = no clamp
}).render(data);
```

Each card's content is measured in a reused offscreen box and its height set to fit (width stays fixed). Precedence: explicit per-node size, then measured, then global. Without a DOM (SSR, jsdom) it measures nothing, so output stays deterministic. Cartesian directions only; grouped-leaf stacks and radial layouts stay at the uniform global size.

## Lazy children (2.0)

For trees too large or dynamic to ship up front, mark a node `hasChildren: true` with `children: []` and provide the `loadChildren` option. Expanding the node shows a spinner, awaits the promise, and splices the returned `NestedNode[]` in. See `references/graph-api.md` for details.

## Direction & layout

| `direction` | Effect |
|---|---|
| `'top'` *(default)* | Root at the top, children flow downward. |
| `'bottom'` | Root at the bottom, children flow upward. |
| `'left'` | Root on the left, children flow rightward. |
| `'right'` | Root on the right, children flow leftward. |
| `'radial'` *(2.0)* | Root at the center, each depth on a ring radiating outward. |

`layoutType: 'tree' | 'cluster'` (default `'tree'`) controls rank placement: `'cluster'` pins every leaf to the deepest rank, the outer ring in radial or the bottom row in cartesian (the classic dendrogram look).

`groupLeafNodes: true` stacks leaf siblings vertically (useful for wide org charts where leaves are people).

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `{ id: 'leaf', name: 'X' }` (missing `children`) | `{ id: 'leaf', name: 'X', children: [] }` |
| Re-using an `id` between branches | Make ids globally unique |
| Hand-rolling avatar HTML | Use `contentKey: 'data'` and let the built-in template render the org card |
| Setting global colors then overriding for one node | Use `node.options` instead of branching at the global level |
| Mutating `children` in place | Replace the reference: `graph.updateData(newData)` (diffs and animates; `construct` is a hard rebuild) |
| Omitting `children: []` on a lazy node | `hasChildren: true` still needs `children: []` plus a `loadChildren` option |
