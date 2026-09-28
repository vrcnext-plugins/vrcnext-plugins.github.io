---
title: UI injection
---

# UI injection

[← Back to index](./)

All UI is built from **VRCNext's own CSS classes**, so plugin panels inherit the user's theme,
font-size offset and any active custom theme. Hand-rolling a `<button>` loses all three.

## Where plugin UI appears

`ctx.ui` needs no permission: it can only draw the plugin's own panels.

The host's own pages are two sections in VRCNext's Settings tab, after a divider below
VRCNext's own: **Plugin System** (the Bridge card, status, platform matrix, diagnostics, the
live log) and **Plugins** (install by URL, enable, update, uninstall, saved permissions, and
every plugin's settings card). The **Plugins** group the host adds to the sidebar, mirrored as a
**Plugins** menu in the top bar, only holds shortcuts to those two sections.

Everything the host uses to put itself in the app is a plugin API too: sections and dividers in
Settings, shortcut groups in the sidebar, tabs, dashboard cards. Your plugin's own `addNavTab`
entries are separate top-level sidebar buttons — they are not placed inside the host's group.
The sidebar is for pages that are really pages; shortcuts to things that live elsewhere go in a
group of your own (below).

## Sidebar tab

```ts
const panel = ctx.ui.addNavTab({
  label: 'My Plugin',
  icon: 'extension',          // Material Symbols Rounded ligature
  render: (tab) => {
    tab.appendChild(ctx.ui.createCard('Status', 'monitoring'));
  },
  onVisibility: (visible) => { if (visible) startPolling(); else stopPolling(); },
});

panel.visible;                // the same state, to read on demand
```

`render` is called **once, lazily**, the first time the tab is opened.

VRCNext shows one tab at a time, and `onVisibility` fires whenever yours becomes the one on
screen or stops being it — including when the user switches to one of VRCNext's own tabs, which
your plugin never hears about otherwise. Use it to **stop work nobody can see**: a poll, a clock,
a redraw. `panel.visible` is `false` until the tab is first opened.

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

The card appears in a Settings section **of its own**, named after the plugin and filed below the
host's Plugin System and Plugins sections, under a divider. The plugin's row in the Plugins list
gets a **Settings** button that jumps there. A plugin with several cards gets one section
carrying all of them.

`render` draws the card's own content, and the plugin's schema is rendered as rows underneath —
on the **first** settings card only, so a plugin with several cards does not get the same form
painted onto each of them. Pass `settings: true` when the rows belong on a later card, or
`settings: false` to keep a card free of them.

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

## Settings section

A section of your own in VRCNext's Settings tab, after the host's. `attach` files any card or
container under it; `open` jumps there. Ids are namespaced by plugin id, so `main` becomes
`my-plugin.main` in the page. Disposing the section — or disabling the plugin — removes the nav
item and every attached block, and VRCNext falls back to General if it was on screen.

```ts
ctx.ui.addSettingsDivider();
const section = ctx.ui.addSettingsSection({ id: 'main', label: 'My Plugin', icon: 'extension' });
// id: lowercase letters, digits, '.', '_' and '-' only; anything else throws.
section.attach(k.card({ title: 'Status', icon: 'info', children: [/* rows */] }));

// A settings card can be filed here instead of in the plugin's automatic section.
ctx.ui.addSettingsCard({ title: 'My Plugin', icon: 'tune', section });

// Two cards: the schema rows go on the second one.
ctx.ui.addSettingsCard({ title: 'Status',  icon: 'info', settings: false, render: (c) => {/* … */} });
ctx.ui.addSettingsCard({ title: 'Options', icon: 'tune', settings: true });
```

## Sidebar group

A collapsible group in the sidebar, mirrored as a menu in the top bar, whose entries are
shortcuts. This is exactly what the host's **Plugins** group is. Use it to reach a Settings
section, a modal, a tab; use `addNavTab` for a page of its own.

```ts
ctx.ui.addSidebarGroup({
  id: 'shortcuts',
  label: 'My Plugin',
  icon: 'extension',
  entries: [
    { id: 'settings', label: 'Settings', icon: 'tune', activate: () => { section.open(); } },
    { id: 'hello', label: 'Say hi', icon: 'waving_hand', activate: () => { ctx.ui.toast({ message: 'Hi.' }); } },
  ],
});
```

Survives VRCNext rebuilding its sidebar, and is removed with the plugin.

## Choosing a VRChat thing

A [picker setting](settings.md#pickers) covers the usual case. When the choice belongs to a button
rather than a settings row, open the same picker directly. This needs the `vrchat` permission and
asks the user once, like any [VRChat lookup](vrchat-data.md):

```ts
const ids = await ctx.ui.pickEntity({
  kind: 'user',                      // or 'world' | 'avatar' | 'group' | 'instance'
  multiple: true,
  scopes: ['friends', 'favorites'],  // omit for every scope the kind has
  selected: current,
});
if (ids === undefined) return;       // cancelled
```

It resolves with the chosen ids. No permission is involved: the picker reads VRCNext's data on the
user's behalf, and the plugin sees only the result.

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
| `k.statusCard({ tone, label, action })` | The dot-and-label strip VRCNext uses atop its tool tabs. `tone` is `online`, `warn` or `offline`. |
| `k.section(label, children)` | An uppercase label followed by its rows. |

**Prefer `k.grid` over a stack of full-width cards.** Most cards are short key/value lists, and a
full-width row on a maximised window leaves a lot of dead space between the label and its value.

### Content

| Call | Produces |
| :--- | :--- |
| `k.row({ label, value, detail, stacked })` | A label/value row. A string `value` renders as muted text; pass a node for anything else. `stacked` puts the value on its own full-width line. Rows rule themselves between siblings. |
| `k.toggleRow({ label, value, onChange, detail })` | The same row with VRCNext's switch. |
| `k.button({ label, icon, onClick, active, disabled, round })` | A `vrcn-button`. |
| `k.buttonRow(...children)` | A spaced horizontal strip. |
| `k.textField({ value, placeholder, onCommit })` | A `vrcn-edit-field`. Commits on blur and Enter, never per keystroke. |
| `k.textArea({ value, placeholder, rows, onCommit })` | A multi-line `vrcn-edit-field`. Put it in `k.row({ stacked: true })` so it gets the full width. |
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

Every injector returns a `PanelHandle` with `.element`, `.visible` and `.dispose()`, and is
disposed automatically with the plugin. `visible` asks whether the user can see that panel: for a
nav tab, whether it is the tab on screen; for a settings card, whether its section is the one
open; for injected CSS, never.

## A warning about icons

VRCNext ships a **subset** of Material Symbols Rounded (~568 KB), not the whole face. A perfectly
valid icon name that VRCNext itself does not use has no glyph and renders as **its own literal
text** — `cable` shows up as the word "CABLE". There is no runtime error and no console warning.

Pick icons you have actually seen in the app. To list what is definitely present:

```bash
grep -rho 'msi">[a-z_0-9]*' frontend/ | sed 's/.*>//' | sort -u
```

[← Events & actions](events-and-bridge.md) · [Notifications →](notifications.md)
