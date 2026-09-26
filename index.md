---
title: VRCNext Plugins
---

# VRCNext Plugins

Build plugins for [VRCNext](https://github.com/shinyflvre/VRCNext) — **without modifying
VRCNext.**

VRCNext ships no plugin API. The
[plugin system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) adds one by installing
itself as a VRCNext *custom theme*: a folder under `~/.config/VRCNext/custom-themes/` whose
JavaScript VRCNext injects into its own page. That folder lives in the config directory, not the
install tree, so app updates leave it alone.

The page runs **one static bundle** containing the host and every installed plugin. The
[VRCNext Bridge](native-companion.md), a small native daemon, produces it: it clones plugin repositories
over HTTPS with pure-Rust git, checks them, and runs a pinned `esbuild`. The page never evaluates
code at runtime and never fetches anything on its own. Without the bridge there is no install,
no state and no build — it is mandatory, and the installer sets it up in one command.

Users install a plugin by pasting its **repository URL** into the Plugins tab. One repository is
one plugin: `plugin.json` and `main.ts` at the root.

## Thirty-second version

```ts
import { definePlugin, type PluginId } from '@vrcnext/plugin-api';

export default definePlugin({
  id: 'my-plugin' as PluginId,
  activate(ctx) {
    ctx.gameLog.onType('OnPlayerJoined', (entry) => {
      ctx.ui.toast({ message: `${entry.detail} joined.` });
    });
  },
});
```

Put that in `main.ts`, declare `"permissions": ["gamelog"]` in `plugin.json`, push, and paste
the repository URL into the Plugins tab. The bridge compiles it; nothing is built on your side.

## What plugins can do

| Capability | API | Permission |
| :--- | :--- | :--- |
| VRCNext host events, verified payloads typed | `ctx.events` | `host:events` |
| Backend actions, with request/response | `ctx.bridge` | `host:actions` |
| Observe or drop the actions VRCNext sends | `ctx.bridge.interceptOutbound` | `host:intercept` |
| Outbound HTTP — the only `fetch` a plugin has | `ctx.http` | `network` |
| VR overlay and desktop notifications through the bridge *(any platform)* | `ctx.native` | `native` |
| OSC send and receive through VRCNext's sockets *(Windows only)* | `ctx.osc` | `osc` |
| VRChat game log stream and backlog | `ctx.gameLog` | `gamelog` |
| In-app toasts and confirm modals; OS tray + SteamVR overlay *(Windows only)* | `ctx.notifications` | `notifications` |
| Context-menu items, dividers and submenus | `ctx.contextMenu` | `context-menu` |
| In-page HTTP routes with path parameters | `ctx.router` | `routes` |
| Clipboard read and write | `ctx.clipboard` | `clipboard` |
| Deep links VRCNext delivers | `ctx.deepLinks` | `host:events` (`openDeepLink`) |
| Sidebar tabs, dashboard cards, settings cards, custom CSS, `ui.kit` | `ctx.ui` | none |
| Typed, persisted, schema-rendered settings | `ctx.settings` | none |
| Levelled logging with a live in-app panel | `ctx.logger` | none |

## Documentation

**Start here**

| Page | What it covers |
| :--- | :--- |
| [Getting started](getting-started.md) | The one-line install, pairing, your first plugin from the template. |
| [Plugin anatomy](plugin-anatomy.md) | The flat repository, `definePlugin`, lifecycle, disposal, the context object. |
| [plugin.json](plugin-json.md) | Every manifest field and its rules. |
| [Permissions](permissions.md) | Declared categories, first-use prompts, what the user sees. |
| [Source policy](source-policy.md) | What the bridge refuses to compile, and why. |
| [Settings](settings.md) | Declarative schema, type inference, persistence. |

**Capabilities**

| Page | What it covers |
| :--- | :--- |
| [Events & actions](events-and-bridge.md) | Host events, sending actions, interception. |
| [UI injection](ui.md) | Nav tabs, dashboard cards, settings cards, CSS, the kit. |
| [Notifications](notifications.md) | In-app, desktop tray, **SteamVR overlay**, confirm modals; VR and desktop via the bridge. |
| [VRCNext Bridge](native-companion.md) | What the daemon does, pairing, status, `ctx.native`. |
| [Context menus](context-menus.md) | Items, dividers, submenus, entity targets. |
| [OSC](osc.md) | Sending and receiving OSC through VRCNext. |
| [Game log](game-log.md) | The VRChat log stream and backlog. |
| [Routes & deep links](routes-and-links.md) | In-page HTTP routes, `vrcn://` links, and their limits. |
| [Logging](logging.md) | The logger, the Logs panel, the bridge's log file. |

**Reference**

| Page | What it covers |
| :--- | :--- |
| [Publishing & updates](publishing.md) | Versioning, how updates reach users, `apiVersion`. |
| [Using TSX](tsx.md) | JSX in plugins: vendor the renderer, and why VRCNext itself has no framework. |
| [Security model](security.md) | What plugins can do, and what the system does about it. |
| [Limitations](limitations.md) | What is genuinely impossible without patching VRCNext. |
| [API reference](api-reference.md) | Every exported type, one page. |

## Honest scope

Three things are **not possible** from a theme-injected host, and this project does not pretend
otherwise:

1. **Real HTTP routes.** VRCNext's web server is a C# `HttpListener` with a fixed route table.
   Plugin routes work inside the page only — `curl` cannot reach them.
2. **Custom `vrcn://` prefixes.** VRCNext validates the link type in C# against a closed list and
   drops anything else before the page ever sees it.
3. **Writing to VRCNext's log file.** There is no page→C# action that logs arbitrary text. The
   Logs panel, its download button and the bridge's `plugins.log` are the supported equivalents.

Separately, several VRCNext features are **Windows-only in VRCNext itself** — OSC, the VR
overlay, the chatbox and more. See the [platform matrix](limitations.md). For notifications
there is a way around this: the [VRCNext Bridge](native-companion.md) reaches VR overlays and the desktop
on any platform, because it is a separate process rather than the page.

Everything else on this site is implemented and type-checked against VRCNext **2026.60.5**.

## Repositories

| Repository | Purpose |
| :--- | :--- |
| [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) | The host, the typed `@vrcnext/plugin-api` contract, the plugin template and the installer. |
| [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge) | The native companion daemon. Rust, loopback only, one WebSocket. |
| [vrcnext-plugins.github.io](https://github.com/vrcnext-plugins/vrcnext-plugins.github.io) | This site. |

## A note on trust

Plugins run inside the VRCNext page with the full authority of the user's VRChat session. There
is no sandbox. The [permission model](permissions.md) makes each capability declared and each
concrete use confirmed, and the [source policy](source-policy.md) keeps plugins on the `ctx.*`
path — neither contains a plugin that is determined to misbehave. Installing a plugin is still
equivalent to running a program from that repository. See the [security model](security.md).
