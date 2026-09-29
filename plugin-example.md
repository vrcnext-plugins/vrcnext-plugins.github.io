---
title: The template plugin
---

# The template plugin

[← Back to index](./)

Source: [vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin)

A working VRCNext plugin that uses **every capability the host provides**, one per file, so you
can start from it and delete what you do not need.

> Press **Use this template** on GitHub, or clone it. It installs and runs as-is — enable it and
> you get a nav tab, a dashboard card, a settings section, context-menu entries, OSC, the game
> log, notifications and an in-page HTTP route, all of them live.

## Making it yours

1. **Rename it.** `id`, `name`, `description`, `author` and `homepage` in `plugin.json`, and
   `id` in `main.ts` to match — the host refuses to activate a plugin whose two ids disagree.
   An id is `[a-z0-9][a-z0-9-]{1,39}` and is also the directory the bridge clones into.
2. **Delete the sections you do not need.** Every `install*` call in `main.ts` is one file under
   `src/sections/`. Remove the line, remove the file, and remove what it needed from
   `permissions`, `events` and `hosts`. Declaring less is the point: each category is shown to
   the user with a risk tone when they enable your plugin.
3. **Trim `src/settings.ts`.** It demonstrates every setting kind the host can render, which is
   far more than a real plugin wants. The host derives both the stored type and the control from
   this schema, so there is no form and no parser anywhere in the plugin.
4. **Make a signing key and sign it.** See [Signing plugins](signing.md). The bridge will not install an
   unsigned repository.

## What is in here

| File | Capability |
| :--- | :--- |
| `main.ts` | the manifest of sections; the only file you must edit |
| `src/settings.ts` | every setting kind, including a `list`, an `embed` and a `custom` control |
| `src/state.ts` | the session state the sections share, and the one function they log through |
| `src/sections/events.ts` | typed host events, the untyped escape hatch, settings changes |
| `src/sections/game-log.ts` | the VRChat game log: live, by type, and the backlog |
| `src/sections/osc.ts` | OSC in and out, guarded on `available` |
| `src/sections/deep-links.ts` | `wrld:` / `usr:` links VRCNext delivers |
| `src/sections/routes.ts` | in-page HTTP routes, reachable from this page and nothing else |
| `src/sections/context-menu.ts` | items, dividers, submenus, and the clipboard |
| `src/sections/notifications.ts` | toasts, VRCNext-styled notifications, OS and VR |
| `src/sections/native.ts` | the bridge: VR overlay and desktop targets, asked for by name |
| `src/sections/http.ts` | outbound HTTP behind an optional permission |
| `src/sections/vrchat.ts` | friends, groups, the current instance, avatars — no dialogs |
| `src/sections/ui-nav-tab.ts` | a whole tab from `ctx.ui.kit` — **the UI file to copy** |
| `src/sections/ui-dashboard.ts` | the same job in plain DOM, plus a modal and an entity picker |
| `src/sections/ui-settings-section.ts` | a Settings section of the plugin's own, and sidebar shortcuts |

Everything registered through `ctx` — listeners, panels, routes, timers on `ctx.disposables` —
is torn down when the plugin is disabled, and `ctx.signal` aborts at the same moment. That is
why `deactivate` is empty.

## Working on it

```bash
npm install
npm run check      # tsc --noEmit && eslint .
```

The typed API is not on npm: `@vrcnext/plugin-api` is a `file:` link to a checkout of
[vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) beside the
plugin, the layout every plugin in this organisation uses:

```
projects/
  vrcnext-plugin-system/
  my-plugin/
```

Nothing is bundled on your side: the bridge compiles the repository together with the host, so
`main.ts` is imported as TypeScript straight from the clone. `tsconfig.json` is the host's own
strict configuration and `eslint.config.mjs` mirrors its rules and most of the
[source policy](source-policy.md) — no `any`, no non-null assertions, no `enum`, 100 lines per
function, 1000 per file, 4 parameters, depth 3 — so anything `check` accepts installs. Before
pushing, also run the [protocol check](events-and-bridge.md#checking-against-vrcnext), then
[sign](signing.md) the release.

The manifest fields are on [plugin.json](plugin-json.md); the refused constructs on
[Source policy](source-policy.md).
