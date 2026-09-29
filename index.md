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
    ctx.gameLog.onType('gl_player_join', (entry) => {
      ctx.notifications.toast({ message: `${entry.message} joined.` });
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
| Typed, persisted, schema-rendered settings, including pickers over VRCNext's own data | `ctx.settings` | none |
| Reading users, avatars, worlds, groups and your instance without opening a dialog | `ctx.vrchat` | `vrchat` |
| Levelled logging with a live in-app panel | `ctx.logger` | none |

## Documentation

**Using VRCNext plugins**

| Page | What it covers |
| :--- | :--- |
| [Installer](install.md) | The one-line install, what it sets up, its flags, and how to remove it again. |
| [Using plugins](using-plugins.md) | Finding, installing, enabling, updating and removing a plugin — and what to check when one misbehaves. |

**Writing one — start here**

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
| [VRChat data](vrchat-data.md) | Friends, favourites, groups, instances and lookups — without opening VRCNext's dialogs. |
| [Routes & deep links](routes-and-links.md) | In-page HTTP routes, `vrcn://` links, and their limits. |
| [Logging](logging.md) | The logger, the Logs panel, the bridge's log file. |

**Plugins**

| Plugin | What it does |
| :--- | :--- |
| [Club Security](plugin-club-security.md) | Per-club presets that report who joins your instance and whether they meet the club's rules. |
| [Bio Updater](plugin-bio-updater.md) | Writes your bio, status, pronouns and links from templates, fitted by priority rather than cut. |
| [VRCNext Patches](plugin-patches.md) | Small fixes VRCNext does not make: media-library folders, unattended sign-in. |
| [The template plugin](plugin-example.md) | Every capability, one per file — start a plugin from it. |

**Reference**

| Page | What it covers |
| :--- | :--- |
| [Publishing & updates](publishing.md) | Versioning, how updates reach users, `apiVersion`. |
| [Signing plugins](signing.md) | The signing key, `plugin.sig`, and what the bridge checks. |
| [Using TSX](tsx.md) | JSX in plugins: vendor the renderer, and why VRCNext itself has no framework. |
| [Security model](security.md) | What plugins can do, and what the system does about it. |
| [Limitations](limitations.md) | What is genuinely impossible without patching VRCNext. |
| [API reference](api-reference.md) | Every exported type, one page. |

**Contributing**

| Page | What it covers |
| :--- | :--- |
| [Contributing](contributing.md) | Working on these repositories: the gates, the rules that hold everywhere, signing, the local loop. |

**Internals**

| Page | What it covers |
| :--- | :--- |
| [Bridge reference](bridge.md) | The daemon's protocol, services, install pipeline, build and security design. |
| [Running the bridge](bridge-running.md) | Autostart, flags, remote control, troubleshooting. |
| [Plugin system internals](plugin-system.md) | How the host boots and is built, its repository layout, development. |
| [Design: the pre-bundled architecture](design-aot-bundle.md) | The original plan, kept for its reasoning. |

## Repositories

| Repository | Purpose |
| :--- | :--- |
| [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) | The host, the typed `@vrcnext/plugin-api` contract and the installer. |
| [vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin) | The template repository: a working plugin using every capability, one per file. |
| [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge) | The native companion daemon. Rust, loopback only, one WebSocket. |
| [vrcnext-club-security-plugin](https://github.com/vrcnext-plugins/vrcnext-club-security-plugin), [vrcnext-bio-updater-plugin](https://github.com/vrcnext-plugins/vrcnext-bio-updater-plugin), [vrcnext-patches-plugin](https://github.com/vrcnext-plugins/vrcnext-patches-plugin) | The plugins above. |
| [vrcnext-plugins.github.io](https://github.com/vrcnext-plugins/vrcnext-plugins.github.io) | This site. |

## A note on trust

Plugins run inside the VRCNext page with the full authority of the user's VRChat session. There
is no sandbox. The [permission model](permissions.md) makes each capability declared and each
concrete use confirmed, and the [source policy](source-policy.md) keeps plugins on the `ctx.*`
path — neither contains a plugin that is determined to misbehave. Every tree must also be
[signed](signing.md) by its author, and a plugin stays pinned to the key it was installed with,
so a repository changing hands cannot quietly become an update. Installing a plugin is still
equivalent to running a program from that repository. See the [security model](security.md).
