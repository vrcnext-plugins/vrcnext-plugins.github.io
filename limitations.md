---
title: Limitations
---

# Limitations

[← Back to index](./)

An honest list of what this system **cannot** do, and why. Each VRCNext entry was verified
against the VRCNext source at 2026.61.2 rather than assumed.

## Impossible without patching VRCNext

| Want | Why not |
| :--- | :--- |
| **Externally reachable HTTP routes** | VRCNext's server is a C# `HttpListener` with a fixed route table. The page cannot add to it; the router wraps `fetch`, which external requests never touch. |
| **Custom `vrcn://` prefixes** | `DeepLinkService.Parse` validates against a closed list and returns `null` otherwise, so unknown prefixes are dropped in C# before the page sees them. |
| **Writing to disk from a plugin** | The page has no filesystem access. Everything persistent — settings, enabled flags, saved grants — goes through the [bridge](native-companion.md)'s state store, which is the only thing that can write a file. |
| **Interleaving context-menu items** | `getMenuConfig` is module-closure-scoped; contributions are appended after VRCNext's own items. |
| **New OSC ports through VRCNext** | On Windows VRCNext owns the sockets and plugins share its one OSCQuery advertisement. (On Linux VRCNext holds nothing, and the bridge binds them instead — see below.) |

## By design in the plugin system

| Want | Why not |
| :--- | :--- |
| **Loading a plugin without the bridge** | The page runs one static bundle that only the bridge produces. It never evaluates code at runtime, so there is no path for a URL, a blob or a pasted script to become a running plugin. |
| **Hot reload** | Every change is a commit, push, **Update** (confirmed on the desktop), rebuild, **Reload**. Slower than a dev server, and the price of never running code the bridge has not checked. |
| **npm dependencies** | The bridge has no package manager. Anything a plugin imports must be committed into the repository, where it is scanned by the [source policy](source-policy.md) and counts against the 200-file / 2 MiB limits. |
| **Reaching `window`, `fetch`, `localStorage`, …** | Refused at install by the source policy, so that everything a plugin does outside its own panels goes through `ctx.*` and the [permission gate](permissions.md) can see it. |
| **A sandbox** | A plugin is compiled into the same bundle as the host and runs with the page's authority. Permissions and the policy make what it does declared and confirmed; they do not contain a plugin that is determined to misbehave. See [Security model](security.md). |
| **Silent installs** | Install, update and uninstall are confirmed natively — a notification on Linux, a message box on Windows — because the page cannot be trusted to confirm code being added to itself. Without a prompt channel the bridge refuses. The one exception is a bridge started with `--dev`, which approves every prompt itself and only announces it; that flag is for developing the system, not for running it. |

## Windows-only in VRCNext

`MessageRouter.IsWindowsOnlyAction` drops any action whose name starts with one of these
prefixes **followed by an uppercase letter**, before it reaches any handler. On Linux the whole
feature is unreachable from the page, and VRCNext hides the matching sidebar entries.

| Prefix | Feature | Plugin impact |
| :--- | :--- | :--- |
| `osc` | OSC Tool | **`ctx.osc` goes to the bridge instead** — still check `ctx.osc.available` |
| `vro` | VR wrist overlay | `ctx.notifications.desktop()` no-ops |
| `chatbox` | Custom Chatbox | `ctx.bridge.send('chatbox…')` dropped |
| `vf` | Voice Fight | dropped |
| `kxd` | Kikitan XD (STT) | dropped |
| `sf` / `st` / `fs` | Space Flight / Space Turn / FrameShot | dropped |
| `dp` | Discord Presence | dropped |
| `vc` | VRCVideoCacher | dropped |
| `as` | Avatar Scaling | dropped |

`startRelay` / `stopRelay` (Media Relay) are filtered by exact name as well.

`afTrayNotify` is **not** in the prefix list, so it reaches its handler on Linux — but the tray
and overlay work inside it is `#if WINDOWS`, so nothing happens. That is why
`ctx.notifications.desktopAvailable` exists.

### Two of these gates have a way around them

These gates are in VRCNext, and the page cannot escape them. A **separate process** is not subject
to them at all — which is what the [VRCNext Bridge](native-companion.md) is, so where the bridge
can do the work itself, the gate stops mattering:

- **Notifications.** `ctx.native` reaches VR overlays and the desktop notification daemon on Linux
  and Windows alike, because the daemon talks to them directly rather than asking VRCNext to.
- **OSC.** The bridge's `osc` service binds VRChat's ports itself, so on Linux `ctx.osc` sends and
  receives through the bridge rather than through VRCNext. They are the same sockets either way
  (VRChat listens on 9000 and sends to 9001), a plugin cannot tell which path carried a message,
  and VRCNext's own path is preferred wherever it works — two programs should not fight over one
  OSCQuery advertisement. Loopback only, in both directions.

The rest of the prefix table has no way around it: chatbox, Media Relay, Discord Presence and the
others are features inside VRCNext, not sockets someone else can hold.

Everything else — events, actions without those prefixes, the game log, UI, context menus,
routes, settings, logging — works on both platforms.

## Fragile by nature

These work, but depend on VRCNext internals with no stability guarantee:

- **CSS class names and selectors** (`vrcn-panel-card`, `sf-toggle-row`, `#tab0`, `#sidebarEl`).
  Centralised in `packages/host/src/ui/dom.ts` so a break is loud and one-line to fix.
- **`showTab(i)` indexes by position** among `.tab` elements, not by id. Injected tabs compute
  their index at injection time.
- **The Photino bridge shape** (`window.external.sendMessage` / `receiveMessage`). On Linux the
  inbound side is an array that `receiveMessage` pushes onto, which is what lets the host observe
  events without displacing VRCNext's handler.
- **`request()` correlation** is by event type only — VRCNext has no request ids.
- **Event payloads** other than the seven verified ones are `unknown` on purpose.

## Environmental gotchas

- **Pin the port.** The pairing token lives in the page's `localStorage`, scoped to
  `http://localhost:<LocalHttpPort>`; VRCNext picks a new random port when its saved one is
  taken, and a new origin means pasting the token again. Install with `--pin-port`. (Plugins and
  settings are unaffected: they live in the bridge's state store, not in the page.)
- **Start the bridge inside the graphical session.** Desktop notifications and the install
  confirmation both need a session bus; a bridge started without one refuses every install with
  `approval_unavailable`. The installer's systemd unit orders after `graphical-session.target`
  for this reason.
- **NVIDIA on Linux:** VRCNext re-executes itself to set `WEBKIT_DISABLE_DMABUF_RENDERER=1`, so
  you will see two processes. Export it yourself to skip the double launch.
- **`install_vrcnext.sh` does not register `vrcn://`** — its `.desktop` lacks `MimeType=` and
  `%u`. The AppImage build does.
- **Windows and macOS are untested.** The installer, the message-box prompt and the Scheduled
  Task were written carefully and have not been run; this system has only been developed against
  Linux.

## Verification status

The build, the type gate and the unit tests are green: the permission broker, the state client,
the settings store, the compiled-table reader, the bridge socket and every source-policy rule are
tested without a DOM, and the settings form is tested in JSDOM.

Beyond that, the boot sequence, the manager panel, install, update, the desktop confirmation and
the published plugins have been run inside a real VRCNext against a real bridge, **on Linux
only**. What that does not cover: Windows and macOS end to end, VR overlay output in a headset,
and the OSC paths against a live VRChat. Treat those as unverified.

[← Security model](security.md) · [API reference →](api-reference.md)
