---
title: Plugin anatomy
---

# Plugin anatomy

[← Back to index](./)

## One flat repository, one plugin

```
plugin.json        the manifest; validated by the bridge at install and by the host at boot
main.ts            default-exports definePlugin({...}); its id must equal plugin.json's
src/**             optional, imported from main.ts
README.md          optional
```

No build step, no `dist/`, no `package.json` needed for the bridge (the template has one for
your own `check` script). The bridge imports `main.ts` as TypeScript straight from the clone and
compiles it into the same bundle as the host. Anything you import must be in the repository:
the bridge runs no package manager, and `@vrcnext/plugin-api` is aliased to the host's own copy.

The manifest is documented on its own page: [plugin.json](plugin-json.md).

## The module contract

`main.ts` **default-exports one object**. The host verifies it has a string `id` equal to the
manifest's and a callable `activate` before doing anything else.

```ts
import { definePlugin, type PluginId, type SettingsSchema } from '@vrcnext/plugin-api';

export default definePlugin({
  id: 'my-plugin' as PluginId,
  settings,                 // optional
  activate(ctx) { /* … */ },
  deactivate() { /* … */ }, // optional
});
```

### Why `definePlugin`

It is an identity function whose only job is to pin the settings schema generic. Without it, an
inline plugin object widens `S` to `SettingsSchema` and every setting degrades to
`boolean | number | string`. With it, `ctx.settings.get('mode')` keeps its literal union.

### The id must match three times

`id` in `plugin.json`, `id` on the `definePlugin` object, and the clone directory
`~/.vrcnext-plugins/plugins/<id>` are the same string, `[a-z0-9][a-z0-9-]{1,39}`. It becomes a
filesystem path on the bridge, which is why the format is narrow and the bridge checks it before
touching disk.

## Lifecycle

| Phase | What happens |
| :--- | :--- |
| **Install** | You confirm on the desktop. The bridge clones, validates `plugin.json`, scans the [source policy](source-policy.md), rebuilds the bundle. Nothing is executed. Reload to load it. |
| **Enable** | The consent modal lists the declared permissions. Enable → `apiVersion` checked → `activate(ctx)` called. |
| **Disable** | `deactivate()` awaited → every `ctx` registration disposed → pending settings writes flushed. |
| **Update** | Confirmed on the desktop. Fresh clone, re-validated, swapped in only if it passes, rebuilt. Reload to run it. Settings and saved grants survive. |
| **Uninstall** | Confirmed on the desktop. Clone, state entries and settings removed, rebuilt. The card shows *Removed — reload* until you do. |

If `activate` throws, the host disposes everything the plugin registered and rolls the enabled
flag back, so the UI never shows a plugin as running when it is not.

Every gated capability on `ctx` throws a `PermissionError` when its category is not declared —
it never silently no-ops, so a missing declaration is found on the first run rather than in a
user's bug report.

## Disposal

Everything registered through `ctx` is tracked and reversed automatically:

```ts
activate(ctx) {
  ctx.events.on('friendTimelineEvent', handler);  // auto-removed
  ctx.ui.addNavTab({ /* … */ });                  // auto-removed
  ctx.router.get('stats', handler);               // auto-unmounted

  // Anything you create yourself, register explicitly:
  const timer = setInterval(tick, 30_000);
  ctx.disposables.add(() => { clearInterval(timer); });
}
```

Disposers run in **reverse registration order**, and one throwing does not strand the rest.

### `ctx.signal`

Aborts on deactivate. `ctx.http.fetch` attaches it for you, so a request still in flight when the
plugin is disabled is cancelled; pass it yourself to anything else long-lived:

```ts
const backlog = await ctx.gameLog.history(ctx.signal);
```

## The context object

| Member | Permission | Page |
| :--- | :--- | :--- |
| `ctx.id`, `ctx.version` | — | Manifest identity. |
| `ctx.logger` | — | [Logging](logging.md). `.scoped('sync')` for sub-scopes. |
| `ctx.settings` | — | [Settings](settings.md) |
| `ctx.permissions` | — | [Permissions](permissions.md): `has(p)`, `request(p)`. |
| `ctx.events`, `ctx.deepLinks` | `host:events` | [Events & actions](events-and-bridge.md), [Routes & deep links](routes-and-links.md) |
| `ctx.bridge` | `host:actions`, `host:intercept` | [Events & actions](events-and-bridge.md) |
| `ctx.http` | `network` | [Permissions](permissions.md) |
| `ctx.ui` | — | [UI injection](ui.md) |
| `ctx.notifications` | `notifications` | [Notifications](notifications.md) |
| `ctx.native` | `native` | [VRCNext Bridge](native-companion.md) |
| `ctx.contextMenu` | `context-menu` | [Context menus](context-menus.md) |
| `ctx.osc` | `osc` | [OSC](osc.md) |
| `ctx.gameLog` | `gamelog` | [Game log](game-log.md) |
| `ctx.router` | `routes` | [Routes & deep links](routes-and-links.md) |
| `ctx.clipboard` | `clipboard` | [Permissions](permissions.md) |
| `ctx.disposables`, `ctx.signal` | — | Above. |

[← Getting started](getting-started.md) · [plugin.json →](plugin-json.md)
