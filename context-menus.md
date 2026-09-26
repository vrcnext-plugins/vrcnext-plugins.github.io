---
title: Context menus
---

# Context menus

[← Back to index](./)

Needs the `context-menu` permission (low risk: it only adds entries to menus). The example
also writes the clipboard, which is `ctx.clipboard` behind the `clipboard` permission — the
user is asked the first time — because `navigator.clipboard` is not reachable through the
[source policy](source-policy.md).

```ts
ctx.contextMenu.contribute((target) => [
  { kind: 'divider' },
  {
    kind: 'item',
    icon: 'content_copy',
    label: 'Copy element tag',
    onSelect: () => ctx.clipboard.writeText(target.element.tagName),
  },
  {
    kind: 'submenu',
    icon: 'tune',
    label: 'Mode',
    items: () => MODES.map((m) => ({
      kind: 'item' as const,
      icon: 'radio_button_checked',
      label: m.label,
      checked: ctx.settings.get('mode') === m.value,
      onSelect: () => void ctx.settings.set('mode', m.value),
    })),
  },
]);
```

## Entry kinds

| Kind | Fields |
| :--- | :--- |
| `item` | `icon`, `label`, `onSelect`, optional `danger`, `checked` |
| `submenu` | `icon`, `label`, `items()` — resolved when opened, so it can reflect live state |
| `divider` | none |

## Targeting

```ts
// Only on friend rows.
ctx.contextMenu.contributeFor('.friend-row', (target) => [ /* … */ ]);
```

`target.entity` is populated when a VRChat entity can be read off the clicked element or one of
its ancestors. VRCNext carries ids in inline `onclick` handlers (`openFriendDetail('usr_…')`,
`navOpenModal('world','wrld_…')`) and in `data-*` attributes (`data-uid`, `data-wid`,
`data-avid`, `data-gid`, `data-location`, …); the host reads both, nearest ancestor first, and
derives the type from the id prefix. An instance location (`wrld_…:12345~…`) resolves as
`{ type: 'instance', id: location }`.

```ts
if (target.entity?.type === 'user') {
  entries.push({ kind: 'item', icon: 'badge', label: 'Copy user id',
                 onSelect: () => ctx.clipboard.writeText(target.entity.id) });
}
```

## Standalone menus

```ts
element.addEventListener('contextmenu', (event) => {
  event.preventDefault();
  ctx.contextMenu.open(event.clientX, event.clientY, entries);
});
```

## How it works, and what that costs

VRCNext builds its menus inside a module closure — `getMenuConfig` is unreachable from outside.
So contributions are **appended to the rendered menu** rather than merged into its item list: a
capture-phase `contextmenu` listener records the target, and a `MutationObserver` appends plugin
buttons once VRCNext has drawn its own.

Consequences worth knowing:

- Plugin entries always land **at the bottom**. You cannot interleave with native items.
- Your buttons carry their own listeners, so VRCNext's internal callback array is untouched.
- Providers run on **every menu open** — keep them cheap.
- A provider that throws is logged and skipped; the menu still opens.

[← VRCNext Bridge](native-companion.md) · [OSC →](osc.md)
