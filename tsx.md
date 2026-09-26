---
title: Using TSX
---

# Using `.tsx` in a plugin

[← Back to index](./)

**Short answer: yes — but you bring your own renderer.**

## VRCNext uses no framework at all

Verified against VRCNext 2026.60.5: no React, no Preact, no Vue, no JSX anywhere. No `.tsx` or
`.jsx` files, no `package.json`, no bundler. The entire frontend is hand-written vanilla
JavaScript, loaded as plain `<script>` tags, manipulating the DOM directly.

So there is **no host renderer to share**. If you want JSX you must bring your own, and it adds
to the size of the bundle every user loads.

## What the bridge will compile

The [VRCNext Bridge](native-companion.md) compiles your repository with esbuild, which handles
`.tsx` natively and reads the `jsx` / `jsxImportSource` settings from the `tsconfig.json`
nearest to the file. The source policy scans `.tsx` and `.jsx` too, so renaming a file changes
nothing.

Two things follow from "the bridge has no package manager":

- **The renderer must be committed into the repository.** There is no `node_modules` on the
  bridge; `import { render } from 'preact'` resolves nowhere. Vendor the renderer's source (or
  its ESM build) under `src/` and import it by relative path.
- **It is scanned like your own code** and counts against the 200-file / 2 MiB limits. A
  renderer that touches `window.` or `globalThis.` anywhere in its source will be refused with
  the file and line. Whether Preact's or React's distributed builds pass the
  [source policy](source-policy.md) has **not** been verified; check before building on one.

## Preact, vendored

~3 KB minified and self-contained, so it is the proportionate choice. Locally you still want
the types for `npm run check`:

```bash
npm i -D typescript eslint preact
```

`tsconfig.json`:

```jsonc
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "jsxImportSource": "./src/vendor/preact",
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022",
    "lib": ["ES2022", "DOM"]
  }
}
```

`src/panel.tsx`:

```tsx
import { render } from './vendor/preact/index.js';

interface StatsProps {
  readonly joins: number;
}

function Stats({ joins }: StatsProps) {
  return (
    <div class="mp-grid">
      <div class="mp-stat">
        <div class="mp-label">Player joins</div>
        <div class="mp-value">{joins}</div>
      </div>
    </div>
  );
}

export function mountStats(container: HTMLElement, joins: number): void {
  render(<Stats joins={joins} />, container);
}
```

`main.ts`:

```ts
import { definePlugin, type PluginId } from '@vrcnext/plugin-api';
import { mountStats } from './src/panel.js';   // .js specifier, .tsx source

export default definePlugin({
  id: 'my-plugin' as PluginId,
  activate(ctx) {
    let joins = 0;
    ctx.ui.addNavTab({
      label: 'My Plugin',
      icon: 'extension',
      render: (tab) => { mountStats(tab, joins); },
    });
    ctx.gameLog.onType('OnPlayerJoined', () => { joins += 1; });
  },
});
```

There is no build step on your side: push, and the bridge compiles it with the host.

## Unmount on deactivate

Preact's `render(null, container)` tears the tree down. The host removes your panel element
anyway, but components holding timers or subscriptions need the explicit unmount:

```ts
const handle = ctx.ui.addNavTab({ /* … */ });
ctx.disposables.add(() => { render(null, handle.element); });
```

## React works too, but weigh it

React + react-dom is roughly 45 KB minified versus Preact's ~3 KB, every plugin vendors its own
copy — there is no sharing between plugins — and all of it ends up in the one bundle VRCNext
loads at startup. For panels made of a few cards and a list, plain DOM or `ctx.ui.kit` is the
proportionate choice, and needs nothing vendored at all.

## Style it with the host's classes

Whatever renderer you pick, keep using VRCNext's CSS classes and variables (see
[UI injection](ui.md)) so your panel follows the user's theme. JSX changes how you build the
markup, not what the markup should be.

[← Logging](logging.md) · [Publishing & updates →](publishing.md)
