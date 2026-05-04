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
    return () => tree.destroy();
  }, [options]);          // construct only when options change

  useEffect(() => {
    graphRef.current?.construct(data);   // hot-swap data
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

## Common pitfalls in framework code

| ❌ | ✅ |
|---|---|
| Recreating the chart in every render without `destroy()` | Always return `() => tree.destroy()` from `useEffect` cleanup / `onBeforeUnmount` / `ngOnDestroy` |
| Mutating the tree (`data.value.children.push(...)`) | Replace the reference — `data.value = { ...data.value, children: [...] }` (or `graph.construct(...)` in vanilla) |
| Re-constructing the chart for every data change | Construct once on mount; call `graph.construct(newData)` for data swaps |
| Vue template using camel-case event | Vue normalises to kebab-case: `@selection-change` (not `@selectionChange`) |
| Calling `setLicense` per component | Call `ApexTree.setLicense(KEY)` exactly once at app startup |
