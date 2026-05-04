# Data Format — NestedNode, Org-Card, nodeTemplate

## `NestedNode` shape

```ts
interface NestedNode<T = undefined> {
  id: string;                                   // REQUIRED, globally unique across the tree
  name: string;                                 // REQUIRED, label (when contentKey: 'name')
  children: NestedNode<T>[];                    // REQUIRED — empty array for leaves
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
}
```

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

`nodeTemplate(content)` receives the value at `contentKey` (so it's a `string` for `'name'` mode and your `data` object for `'data'` mode). Return an HTML string.

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

## Direction & layout

| `direction` | Effect |
|---|---|
| `'top'` *(default)* | Root at the top, children flow downward. |
| `'bottom'` | Root at the bottom, children flow upward. |
| `'left'` | Root on the left, children flow rightward. |
| `'right'` | Root on the right, children flow leftward. |

`groupLeafNodes: true` stacks leaf siblings vertically (useful for wide org charts where leaves are people).

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `{ id: 'leaf', name: 'X' }` (missing `children`) | `{ id: 'leaf', name: 'X', children: [] }` |
| Re-using an `id` between branches | Make ids globally unique |
| Hand-rolling avatar HTML | Use `contentKey: 'data'` and let the built-in template render the org card |
| Setting global colors then overriding for one node | Use `node.options` instead of branching at the global level |
| Mutating `children` in place | Replace the reference — `graph.construct(newData)` |
