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
| `select` value no longer in `options` | Default used. |
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
