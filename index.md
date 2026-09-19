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
install tree, so app updates leave it alone and a `git pull` on a VRCNext clone stays clean.

Users install plugins by pasting a **repository URL**. One repository can house many plugins,
and both plugins and the host auto-update.

## Thirty-second version

```ts
import { definePlugin, type PluginId } from '@vrcnext/plugin-api';

export default definePlugin({
  id: 'my-plugin' as PluginId,
  activate(ctx) {
    ctx.gameLog.onType('OnPlayerJoined', (entry) => {
      ctx.notifications.toast({ message: `${entry.detail} joined.` });
    });
  },
});
```

Bundle it to one ESM file, list it in `vrcnext-plugins.json` at your repo root, push, and paste
the repo URL into the Plugins tab.

## What plugins can do

| Capability | API |
| :--- | :--- |
| ~310 VRCNext host events, verified payloads typed | `ctx.events` |
| ~474 backend actions, with request/response and outbound interception | `ctx.bridge` |
| OSC send and receive through VRCNext's sockets *(Windows only)* | `ctx.osc` |
| VRChat game log stream and backlog | `ctx.gameLog` |
| Sidebar tabs, dashboard cards, settings cards, custom CSS | `ctx.ui` |
| In-app toasts and confirm modals; OS tray + SteamVR overlay *(Windows only)* | `ctx.notifications` |
| Context-menu items, dividers and submenus | `ctx.contextMenu` |
| In-page HTTP routes with path parameters | `ctx.router` |
| Deep links VRCNext delivers | `ctx.deepLinks` |
| Typed, persisted, schema-rendered settings | `ctx.settings` |
| Levelled logging with a live in-app panel | `ctx.logger` |

## Documentation

**Start here**

| Page | What it covers |
| :--- | :--- |
| [Getting started](getting-started.md) | Install the host, scaffold a plugin, see it load. |
| [Plugin anatomy](plugin-anatomy.md) | `definePlugin`, lifecycle, disposal, the context object. |
| [Settings](settings.md) | Declarative schema, type inference, persistence. |

**Capabilities**

| Page | What it covers |
| :--- | :--- |
| [Events & the bridge](events-and-bridge.md) | Host events, sending actions, interception. |
| [UI injection](ui.md) | Nav tabs, dashboard cards, settings cards, CSS. |
| [Notifications](notifications.md) | In-app, desktop tray, **SteamVR overlay**, confirm modals. |
| [Context menus](context-menus.md) | Items, dividers, submenus, entity targets. |
| [OSC](osc.md) | Sending and receiving OSC through VRCNext. |
| [Game log](game-log.md) | The VRChat log stream and backlog. |
| [Routes & deep links](routes-and-links.md) | In-page HTTP routes, `vrcn://` links, and their limits. |
| [Logging](logging.md) | The logger, the Logs panel, downloading logs. |

**Reference**

| Page | What it covers |
| :--- | :--- |
| [Publishing](publishing.md) | Manifest format, versioning, auto-update. |
| [Using TSX](tsx.md) | JSX in plugins, and why VRCNext itself has no framework. |
| [Security model](security.md) | What plugins can do, and what the host does about it. |
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
   Logs panel and its download button are the supported equivalent.

Separately, several VRCNext features are **Windows-only in VRCNext itself** — OSC, the VR
overlay, the chatbox and more. See the [platform matrix](limitations.md).

Everything else on this site is implemented and type-checked against VRCNext **2026.60.5**.

## Repositories

| Repository | Purpose |
| :--- | :--- |
| [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) | The runtime, the typed `@vrcnext/plugin-api` contract, and the installer. |
| [vrcnext-plugins.github.io](https://github.com/vrcnext-plugins/vrcnext-plugins.github.io) | This site. |

## A note on trust

Plugins run inside the VRCNext page with the full authority of the user's VRChat session. There
is no sandbox. Installing a plugin is equivalent to running a binary from that repository — see
the [security model](security.md).
