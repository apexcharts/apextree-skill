# Framework Wrappers — React, Vue, Angular

## React — `react-apextree`

```bash
npm install react-apextree apextree
```

```jsx
import Tree from 'react-apextree';

const data = {
  id: '1', name: 'CEO',
  children: [
    { id: '2', name: 'CTO', children: [
      { id: '4', name: 'Eng Lead', children: [] },
    ]},
    { id: '3', name: 'CFO', children: [] },
  ],
};

export default function OrgChart() {
  return (
    <Tree
      data={data}
      width={800}
      height={500}
      nodeWidth={120}
      nodeHeight={40}
      direction="top"
      enableSelection="multi"
      onSelectionChange={(ids) => console.log(ids)}
    />
  );
}
```

Vanilla pattern (when you need direct `Graph` access):

```jsx
import { useEffect, useRef } from 'react';
import { ApexTree } from 'apextree';

function OrgChart({ data, options }) {
  const ref = useRef(null);
  const graphRef = useRef(null);

  useEffect(() => {
    const tree = new ApexTree(ref.current, options);
    graphRef.current = tree.render(data);
    return () => tree.destroy();         // 2.1.0: idempotent, stops the spring loop
  }, [options]);          // construct only when options change

  useEffect(() => {
    graphRef.current?.updateData(data);  // hot-swap data: diffs + animates, state survives
  }, [data]);

  return <div ref={ref} />;
}
```

## Vue 3 — `vue-apextree`

```bash
npm install vue-apextree apextree
```

```vue
<script setup>
import { ref } from 'vue';
import VueTree from 'vue-apextree';

const data = ref({
  id: '1', name: 'CEO',
  children: [
    { id: '2', name: 'CTO', children: [
      { id: '4', name: 'Eng Lead', children: [] },
    ]},
    { id: '3', name: 'CFO', children: [] },
  ],
});

function onSelectionChange(ids) {
  console.log('selected:', ids);
}
</script>

<template>
  <VueTree
    :data="data"
    :width="800"
    :height="500"
    :node-width="120"
    :node-height="40"
    direction="top"
    enable-selection="multi"
    @selection-change="onSelectionChange"
  />
</template>
```

## Angular — `ngx-apextree`

```bash
npm install ngx-apextree apextree
```

```ts
import { Component } from '@angular/core';
import { NgxApextree } from 'ngx-apextree';
import type { NestedNode } from 'apextree';

@Component({
  selector: 'app-org',
  standalone: true,
  imports: [NgxApextree],
  template: `
    <ngx-apextree
      [data]="data"
      [width]="800"
      [height]="500"
      [nodeWidth]="120"
      [nodeHeight]="40"
      direction="top"
      enableSelection="multi"
      (selectionChange)="onSelectionChange($event)">
    </ngx-apextree>
  `,
})
export class OrgComponent {
  data: NestedNode = {
    id: '1', name: 'CEO',
    children: [
      { id: '2', name: 'CTO', children: [
        { id: '4', name: 'Eng Lead', children: [] },
      ]},
      { id: '3', name: 'CFO', children: [] },
    ],
  };

  onSelectionChange(ids: string[]) {
    console.log('selected:', ids);
  }
}
```

## Web components (shadow DOM)

Since 2.2.0 the core API works inside a shadow root: pass an element inside it as the container. Styles, tooltips and keyboard focus stay inside that root, and CSS variables set on the page or the host element (such as the `--apx-*` tokens) reach the chart. The tree adds its own nodes next to whatever the container held when it was constructed, and `destroy()` keeps those children.

```js
import { ApexTree } from 'apextree';

class OrgTree extends HTMLElement {
  connectedCallback() {
    if (!this.el) {
      this.el = document.createElement('div');
      this.attachShadow({ mode: 'open' }).appendChild(this.el);
    }
    this.tree = new ApexTree(this.el, { width: 800, height: 500 });
    this.graph = this.tree.render(this.data);
  }
  disconnectedCallback() {
    this.tree?.destroy();
  }
}
customElements.define('org-tree', OrgTree);
```

## Common pitfalls in framework code

| ❌ | ✅ |
|---|---|
| Recreating the chart in every render without `destroy()` | Always return `() => tree.destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy`. Since 2.1.0 `destroy()` also stops the spring animation loop and is idempotent; without it a tree torn down mid-animation keeps ticking against detached DOM |
| Mutating the tree (`data.value.children.push(...)`) | Replace the reference: `data.value = { ...data.value, children: [...] }` (or `graph.updateData(...)` in vanilla) |
| Re-constructing the chart for every data change | Construct once on mount; call `graph.updateData(newData)` for data swaps (diffs and animates; collapse/selection/focus survive) |
| Vue template using camel-case event | Vue normalises to kebab-case: `@selection-change` (not `@selectionChange`) |
| Calling `setLicense` per component | Call `ApexTree.setLicense(KEY)` exactly once at app startup |
| Importing `apextree/ignite-ui` inside a late-mounting component | Import it once in the app entry, before the first tree renders (free, no licence key; apextree 2.3.0). Leave `theme` out of the options so a dark Ignite UI theme can apply, and after swapping the Ignite UI stylesheet call `refreshTokens()` on the wrapper: the React ref, the Vue template ref or the Angular component (from the next react-apextree and vue-apextree releases after 2.1.1, and the next ngx-apextree release after 1.2.1). With older wrappers, call it on the ApexTree instance |
