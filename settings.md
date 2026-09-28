---
title: Settings
---

# Settings

[← Back to index](./)

Declare a schema once. The host derives the persisted value type *and* renders the settings UI
from it — you never write a form or a parser.

```ts
const settings = {
  enabled: { kind: 'boolean', label: 'Enabled', default: true },
  threshold: { kind: 'number', label: 'Threshold', default: 5, min: 0, max: 10, step: 1 },
  note: { kind: 'string', label: 'Note', default: '', placeholder: 'Optional' },
  template: { kind: 'string', multiline: true, label: 'Template', default: 'Hi {name}' },  // a text area on its own line
  mode: {
    kind: 'select',
    label: 'Mode',
    default: 'all',
    options: [
      { value: 'all', label: 'All friends' },
      { value: 'favorites', label: 'Favorites only' },
    ],
  },
} as const satisfies SettingsSchema;
```

> `as const satisfies SettingsSchema` is load-bearing. `as const` preserves the literal option
> values; `satisfies` checks the shape without widening it. Drop either and `mode` becomes
> `string`.

## The kinds

| kind | Value | Control |
| :--- | :--- | :--- |
| `boolean` | `boolean` | switch |
| `number` | `number` | text field, or a slider |
| `string` | `string` | text field, or a text area |
| `color` | `'#rrggbb'` | colour swatch |
| `time` | `'HH:MM'` | time field |
| `select` | one option value | dropdown |
| `multiselect` | option values, in declared order | toggle buttons |
| `user` `world` `avatar` `group` `instance` | a VRChat id, or an array of them | picker over VRCNext's own data |
| `embed` | an `EmbedTemplate` | Discord-embed editor |
| `object` | an object shaped like `fields` | its fields, indented |
| `list` | an array of objects shaped like `item` | cards you add, duplicate, reorder, remove |
| `custom` | whatever you say | a control you render |

Every kind takes `label`, and optionally `description`, `hidden` and `disabled`. The last two may
be **predicates** over the plugin's other values, re-evaluated after every change:

```ts
webhookUrl: {
  kind: 'string',
  label: 'Webhook URL',
  default: '',
  disabled: (values) => values['postToDiscord'] !== true,
},
```

### Numbers

`min`, `max` and `step` bound a text field. `slider: true` makes it a slider; `markers` makes it a
slider with labelled ticks. **The thumb rests on the markers and nowhere else**, which is what
Equicord's marker sliders do and almost always what you want; `stickToMarkers: false` allows the
values in between. `integer: true` keeps the value whole — the field refuses a fraction and a
stored one is rounded. `unit` is shown after the value.

```ts
volume:  { kind: 'number', label: 'Volume', default: 50, markers: [0, 25, 50, 75, 100], unit: '%' },
timeout: { kind: 'number', label: 'Timeout', default: 30, min: 1, max: 60, slider: true, integer: true, unit: 's' },
```

Snapping is enforced on the stored value too, not only on the control, so a value written by
`ctx.settings.set` lands on a marker as well.

### Strings

`placeholder`, `multiline` (a text area on its own line, committed on blur), `maxLength`, and
`format: 'password' | 'url'`. A `password` field is masked in the UI — it is **not** secret
storage; see the warning at the bottom of this page.

### Choices

`select` stores one option value and infers the literal union. `multiselect` stores several, always
in the order the options were declared, and honours `min`/`max` counts. Both accept a `description`
per option.

`select` is VRCNext's own dropdown — the host hands the `<select>` to the app's `initVnSelect`,
so it opens the same panel its settings do. `multiselect` draws VRCNext's toggle pills, the ones
its theme and cursor pickers use; more than three of them stack under the label instead of
crowding the end of the row.

```ts
days: {
  kind: 'multiselect',
  label: 'Days',
  default: ['mon'],
  options: [{ value: 'mon', label: 'Monday' }, { value: 'tue', label: 'Tuesday' }],
},
// values.days: readonly ('mon' | 'tue')[]
```

### Pickers

A `user`, `world`, `avatar`, `group` or `instance` setting stores the **id** (`usr_…`, `wrld_…`,
`avtr_…`, `grp_…`, or a full instance location). With `multiple: true` it stores an array. The host
renders a picker over what VRCNext already knows, and `scopes` limits where it may look:

| kind | scopes |
| :--- | :--- |
| `user` | `friends` `favorites` `recent` `instance` `search` |
| `world` | `favorites` `recent` `current` `search` |
| `avatar` | `own` `favorites` `recent` `search` |
| `group` | `mine` `search` |
| `instance` | `current` `friends` `manual` |

```ts
doorStaff: { kind: 'user',  label: 'Door staff', default: [], multiple: true, scopes: ['friends'] },
homeWorld: { kind: 'world', label: 'Home world', default: '' },   // every world scope
```

Your plugin never sees the picker and needs no permission for it: the user chooses, you get ids.
Resolve them later with [`ctx.vrchat`](api-reference.md#vrchat-data), or open the same picker
yourself with `ctx.ui.pickEntity`.

The picker's field filters the scope on screen as it is typed, and is the search query itself
when the scope is `search`. In the settings row, each chosen thing is a profile row with its own
×, named rather than shown as an id — the id is what your plugin stores, not what the user
picked.

### Discord embeds

An `embed` setting stores a template for every part of an embed — title, description, colour,
author, thumbnail, image, footer, and a list of fields — and each text is a
[template](api-reference.md#templates). `variables` lists the names you will supply, so the editor
can show them.

```ts
report: {
  kind: 'embed',
  label: 'Discord report',
  default: { title: '{name} joined', color: '{resultColor}', timestamp: true },
  variables: ['name', 'resultColor', 'world'],
},

// at send time
const embed = renderEmbed(ctx.settings.get('report'), values);
if (embed !== undefined) {
  const result = await postWebhook({
    http: ctx.http,
    logger: ctx.logger,
    url,
    payload: discordWebhookPayload(embed, { username: 'My plugin' }),
  });
  if (!result.ok) ctx.logger.warn(String(result.error));
}
```

`renderEmbed` fills every text, drops what rendered empty, refuses URLs that are not `http(s)`,
cuts everything to Discord's limits, and returns `undefined` when nothing is left — Discord
rejects an empty embed. `color` accepts `#rrggbb`, a decimal, or a name (`green`, `orange`, `red`,
`blue`, `yellow`, `grey`). `discordWebhookPayload` never allows mentions.

Post with [`postWebhook`](api-reference.md#posting-to-a-discord-webhook) rather than calling
`ctx.http.fetch` yourself: it checks the URL, logs the payload, explains a refusal and never
throws.

### Objects and lists

`object` nests a schema under one key. `list` stores an array of objects shaped like `item`, and
the user adds, duplicates, reorders and removes them; `titleKey` names the card, `addLabel` the
button, `max` the ceiling.

```ts
presets: {
  kind: 'list',
  label: 'Presets',
  titleKey: 'name',
  addLabel: 'Add preset',
  default: [],
  item: {
    name:  { kind: 'string', label: 'Name', default: 'New' },
    world: { kind: 'world',  label: 'World', default: '' },
    embed: { kind: 'embed',  label: 'Embed', default: {} },
  },
},
// values.presets: readonly { name: string; world: string; embed: EmbedTemplate }[]
```

Both nest freely — a list item may hold a list, a picker or an embed. Every control inside one is
the same control as at the top level, and an edit anywhere still results in exactly one write of
the top-level value, so `onChange` fires once.

An `object` may carry a `toggle`: a switch on its header, with its fields shown only while the
switch is on.

```ts
templates: {
  kind: 'object',
  label: 'Templates',
  toggle: {
    label: 'Use custom templates',
    description: "Off means the plugin's own wording, which changes as the plugin improves.",
    default: false,
  },
  fields: {
    template: { kind: 'string', multiline: true, label: 'Report', default: DEFAULT_TEMPLATE },
  },
},
// values.templates: { enabled: boolean; template: string }
```

The switch's own state lives in the object under `enabled` — `TOGGLE_KEY` — beside the fields it
guards, so you read the answer and the text it gates in one place instead of keeping a loose
boolean next to them and hoping the two stay in step.

Its fields are **hidden, not cleared**: the values are the user's. Someone who turns a custom
template off and on again gets back the text they wrote, not an empty box. Read it as a fallback
rather than a branch at every use:

```ts
const text = preset.templates.enabled ? preset.templates.template : DEFAULT_TEMPLATE;
```

Note what that buys the user beyond tidiness: with the switch off they follow the plugin's own
wording, so every improvement you ship reaches them without them editing anything.

A single setting can be hidden the same way with a `hidden` predicate, which is the better fit
when the thing being gated is one field rather than a group.

### Your own control

When no kind fits, `custom` hands you the row. `coerce` decides what may be stored, exactly as the
built-in kinds do for theirs: return the cleaned value, or `undefined` to fall back to the
default. Wrap it in `defineCustomSetting<T>` so `T` survives.

```ts
windowSize: defineCustomSetting<WindowSize>({
  kind: 'custom',
  label: 'Window size',
  default: { width: 800, height: 600 },
  coerce: (value) => (isWindowSize(value) && value.width >= 320 ? value : undefined),
  render: (host) => {
    const input = document.createElement('input');
    input.value = String(host.value.width);
    input.addEventListener('change', () => {
      void host.setValue({ ...host.value, width: Number(input.value) })
        .catch(() => { host.setError('At least 320 wide.'); });
    });
    host.onChange((next) => { input.value = String(next.width); });
    return input;
  },
}),
```

`host.setValue` rejects when `coerce` refuses, which is how a validation message reaches the user;
`host.setError` puts it under the control. The value must be plain JSON.

## Reading and writing

```ts
ctx.settings.get('enabled');   // boolean
ctx.settings.get('mode');      // 'all' | 'favorites'  ← not string
ctx.settings.values;           // the whole object, synchronous

await ctx.settings.set('mode', 'favorites');  // 'nope' is a compile error
await ctx.settings.reset();
```

`values` is synchronous because everything is loaded once at activation — reading a setting
inside a hot event handler should not await the bridge. Writes are debounced 200 ms and
coalesced, and anything still pending is flushed when the plugin is disabled.

## Reacting to changes

```ts
ctx.disposables.add(
  ctx.settings.onChange((values) => {
    ctx.logger.info(`Mode is now ${values.mode}.`);
  }),
);
```

Fires for writes from your code *and* from the settings UI.

## Validation and migration

Every persisted value is re-validated against the schema on load. Anything that no longer
matches falls back to the default:

| Situation | Result |
| :--- | :--- |
| Stored value has the wrong type | Default used. |
| Number outside `min`/`max` | Clamped into range. |
| Number off a `markers` tick | Snapped to the nearest one. |
| String longer than `maxLength` | Cut. |
| `select` value no longer in `options` | Default used. |
| `multiselect` value no longer offered | Dropped; the rest are kept. |
| A picker value that is not an id of that kind | Dropped. |
| An `object` or `list` field that no longer matches | That field falls back to its default; the rest of the object survives. |
| Key removed from the schema | Ignored. |
| Key added to the schema | Default used. |

So changing a setting's kind or removing an option between versions is safe — no migration code,
no crash on upgrade.

## Where it is stored

In the [VRCNext Bridge](native-companion.md)'s state store — `~/.vrcnext-plugins/state.json`,
namespace `plugin:<id>`, one key per setting. The page has no storage of its own worth trusting:
it cannot write a file, and anything origin-scoped in the browser is lost the moment VRCNext
picks a new port. Settings survive plugin updates (the clone is replaced, the state is not) and
are deleted on uninstall. Each value is at most 64 KiB serialised.

> **Never put secrets here.** The state store is a plain JSON file on the user's disk, and every
> plugin in the page can reach the host's state client. There is no secret storage in this
> system.

[← Source policy](source-policy.md) · [Events & actions →](events-and-bridge.md)
