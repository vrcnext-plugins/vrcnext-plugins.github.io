---
title: UI injection
---

# UI injection

[← Back to index](./)

All UI is built from **VRCNext's own CSS classes**, so plugin panels inherit the user's theme,
font-size offset and any active custom theme. Hand-rolling a `<button>` loses all three.

## Where plugin UI appears

Installing the host adds a **Plugins** group to the sidebar (a divider plus a puzzle-piece
entry) and mirrors it as a **Plugins** menu in the top bar. Both drive the same three tabs:
**Manage Plugins**, **Logs**, and **Plugin System**.

Your plugin's own `addNavTab` entries are separate top-level sidebar buttons — they are not
placed inside that group.

## Sidebar tab

```ts
ctx.ui.addNavTab({
  label: 'My Plugin',
  icon: 'extension',          // Material Symbols Rounded ligature
  render: (tab) => {
    tab.appendChild(ctx.ui.createCard('Status', 'monitoring'));
  },
});
```

`render` is called **once, lazily**, the first time the tab is opened.

VRCNext's `navRender()` does `navEl.innerHTML = ''` whenever the nav editor saves or the layout
changes, which drops injected buttons entirely. The host re-attaches via a `MutationObserver` —
no polling, no lost button.

## Dashboard card

```ts
ctx.ui.addDashboardCard({
  title: 'My Plugin',
  icon: 'insights',
  order: 10,                  // lower sorts earlier among plugin cards
  render: (card) => { card.appendChild(buildChart()); },
});
```

Also re-attached automatically when VRCNext re-renders the dashboard.

## Settings card

```ts
ctx.ui.addSettingsCard({
  title: 'My Plugin',
  icon: 'tune',
  render: (card) => {
    card.appendChild(
      ctx.ui.createToggleRow('Enable feature', ctx.settings.get('enabled'), (checked) => {
        void ctx.settings.set('enabled', checked);
      }),
    );
  },
});
```

## Toasts

```ts
ctx.ui.toast({ message: 'Saved.' });
ctx.ui.toast({ message: 'Could not reach the API.', ok: false });
```

Routed through VRCNext's own toast renderer when available, else to the plugin log.

## Custom CSS

```ts
ctx.ui.injectCss(`
  .mp-grid { display: grid; gap: 10px; }
  .mp-value { color: var(--tx0); font-variant-numeric: tabular-nums; }
`);
```

Removed on deactivate. **Use VRCNext's CSS variables** rather than literal colours so your panel
follows the theme:

| Variable | Use |
| :--- | :--- |
| `--tx0` / `--tx2` / `--tx3` | Primary / secondary / tertiary text |
| `--bg-input` | Inset panel background |
| `--fs-off` | User font-size offset — add it: `calc(13px + var(--fs-off, 0px))` |

Prefix your class names (`.mp-`) — the page is a single shared document.

## The kit

`ctx.ui.kit` is the easy path. Describe the shape; it emits VRCNext's own markup, so you never
need a class name, a stylesheet, or a `createElement` tree to get a panel that looks like it
shipped with the app.

```ts
const k = ctx.ui.kit;

container.append(
  k.layout(
    k.grid([
      k.card({ title: 'Status', icon: 'info', children: [
        k.row({ label: 'Connection', value: k.badge('ok', 'Online') }),
        k.row({ label: 'Last seen', value: '2 minutes ago', detail: 'From the game log.' }),
        k.stat({ label: 'Events today', value: String(count) }),
      ]}),
      k.card({ title: 'Controls', icon: 'tune', children: [
        k.toggleRow({ label: 'Announce joins', value: on, onChange: setOn }),
        k.buttonRow(
          k.button({ label: 'Reset', icon: 'refresh', onClick: reset }),
          k.button({ label: 'Export', icon: 'download', onClick: exportAll }),
        ),
      ]}),
    ]),
  ),
);
```

### Layout

| Call | Produces |
| :--- | :--- |
| `k.layout(...children)` | The padded, gapped, scrolling column VRCNext gives its settings tabs. **Wrap every tab in one.** |
| `k.grid(children, { min })` | A responsive grid of cards. Fits as many columns of at least `min` px (default 280) as the width allows. |
| `k.pair(a, b)` | Exactly two cards, side by side. |
| `k.card({ title, icon, children, span })` | A `vrcn-panel-card`. `span` makes it occupy several grid columns. |
| `k.statusCard({ online, label, action })` | The dot-and-label strip VRCNext uses atop its tool tabs. |
| `k.section(label, children)` | An uppercase label followed by its rows. |

**Prefer `k.grid` over a stack of full-width cards.** Most cards are short key/value lists, and a
full-width row on a maximised window leaves a lot of dead space between the label and its value.

### Content

| Call | Produces |
| :--- | :--- |
| `k.row({ label, value, detail })` | A label/value row. A string `value` renders as muted text; pass a node for anything else. Rows rule themselves between siblings. |
| `k.toggleRow({ label, value, onChange, detail })` | The same row with VRCNext's switch. |
| `k.button({ label, icon, onClick, active, disabled, round })` | A `vrcn-button`. |
| `k.buttonRow(...children)` | A spaced horizontal strip. |
| `k.textField({ value, placeholder, onCommit })` | A `vrcn-edit-field`. Commits on blur and Enter, never per keystroke. |
| `k.dropdown({ options, selected, onChange })` | A `vrcn-dropdown`. |
| `k.badge(tone, text)` | A coloured pill: `ok`, `warn`, `err`, `accent`, `cyan`, `neutral`. |
| `k.stat({ label, value, tone })` | A big number with a caption. |
| `k.description(text)` / `k.sectionLabel(text)` / `k.valueText(text)` / `k.emptyState(text)` | Body copy, group label, muted value, empty-state line. |

### Conditionals and updates

Children accept `Node | string | false | null | undefined`, and anything falsy is dropped — so a
conditional child needs no ceremony:

```ts
k.card({ title: 'Debug', children: [
  alwaysRow,
  isAdmin && k.row({ label: 'Internal id', value: id }),
]})
```

Everything returns plain DOM, so updating is ordinary DOM work. To refresh a card, keep it and
replace its contents:

```ts
const card = k.card({ title: 'Live', icon: 'radar' });
const body = k.box();
card.appendChild(body);

function refresh(): void {
  k.setChildren(body, rows.map((r) => k.row({ label: r.name, value: r.value })));
}
```

### Lower-level builders

Still available, and used by the kit internally:

| Call | Produces |
| :--- | :--- |
| `ctx.ui.createPanelLayout()` | Same as `k.layout()`. |
| `ctx.ui.createCard(title, icon)` | A `vrcn-panel-card` with a header. |
| `ctx.ui.createToggleRow(label, checked, onChange)` | An `sf-toggle-row` with a themed switch. |

Every injector returns a `PanelHandle` with `.element` and `.dispose()`, and is disposed
automatically with the plugin.

## A warning about icons

VRCNext ships a **subset** of Material Symbols Rounded (~568 KB), not the whole face. A perfectly
valid icon name that VRCNext itself does not use has no glyph and renders as **its own literal
text** — `cable` shows up as the word "CABLE". There is no runtime error and no console warning.

Pick icons you have actually seen in the app. To list what is definitely present:

```bash
grep -rho 'msi">[a-z_0-9]*' frontend/ | sed 's/.*>//' | sort -u
```

[← Events & the bridge](events-and-bridge.md) · [Notifications →](notifications.md)
