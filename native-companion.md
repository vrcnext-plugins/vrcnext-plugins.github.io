---
title: VRCNext Bridge
---

# VRCNext Bridge

[← Back to index](./)

[vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge) is the native companion of
the plugin system, and it is **mandatory**. VRCNext's page can only speak HTTP and WebSockets:
it cannot clone a repository, run a compiler, keep a file, open a UDP socket or connect to D-Bus.
Everything that needs one of those happens in the bridge, a small Rust daemon on loopback.

```
page  ──WebSocket──▶  vrcnext-bridge  ──pure-Rust git (HTTPS)──▶  plugin repositories
                            │
                            ├── spawns bin/esbuild ─────────────▶  <VRCNext config>/custom-themes/
                            │      (checksum-verified, fixed args)     vrcnext-plugin-system/vrcnext-plugin-host.js
                            ├── state.json     enabled flags, saved grants, plugin settings
                            ├──UDP────────────────────────────────▶  WayVR / XSOverlay-protocol overlay
                            └──D-Bus───────────────────────────────▶  the desktop's notification daemon
```

Without it there is no install, no state and no build. The [installer](getting-started.md) sets
it up, registers it to start with your session, and prints the pairing token.

## What it does

| Service | Does |
| :--- | :--- |
| `plugins` | install / list / check_updates / update / uninstall / build — clones, validates [plugin.json](plugin-json.md) and the [source policy](source-policy.md), compiles the bundle |
| `state` | the page's key-value store, `state.json`: enabled flags, saved permission grants, plugin settings |
| `notify` | notifications to VR overlays and the desktop, individually targetable |
| `logs` | appends the page's log lines to `plugins.log` |

### The build

The page runs **one static bundle**: the theme file VRCNext loads, containing the host and every
installed plugin. After every install, update or uninstall the bridge verifies `esbuild`'s SHA-256
against the checksum the installer wrote beside it, writes a generated import table —

```ts
import p0 from '../plugins/friend-alerts/main.ts';
import m0 from '../plugins/friend-alerts/plugin.json';
export const COMPILED_PLUGINS = [{ manifest: m0, plugin: p0 }] as const;
```

— and runs esbuild with a fixed argument list, a cleared environment and a 60 s deadline. A
failed build leaves the previous bundle untouched. Then it pushes a `build` event and the page
shows **Rebuilt — reload to apply** with a Reload button. It never reloads on its own, because
a reload discards whatever you were doing in VRCNext.

This is why the page never evaluates code at runtime and never fetches a manifest: everything it
runs was checked by the bridge before it was compiled in. The build module is the one place the
bridge spawns a process, and nothing from a plugin, a manifest or a request reaches the command
line — the plugin id list is regex-validated and lands in a generated file.

### Where things live

```
~/.vrcnext-plugins/                   Windows: %LOCALAPPDATA%\vrcnext-plugins\
  bin/vrcnext-bridge[.exe]
  bin/esbuild[.exe]  bin/esbuild.sha256
  host/packages/{api,host}/src/       host sources, read-only input for the build
  plugins/<id>/                       one git clone per installed plugin
  build/static-plugins.ts             generated import table
  state.json  token  bridge.log
```

The bundle goes to `~/.config/VRCNext/custom-themes/vrcnext-plugin-system/` (`%APPDATA%\VRCNext`
on Windows), beside its source map and an `info.json`.

## Pairing

The page holds one WebSocket to `/v1/ws` for the life of VRCNext. The first frame must be a
`hello` carrying the **pairing token**; until the bridge answers `welcome` the socket carries
nothing, and a wrong token, any other first frame, or five seconds of silence closes it. A
failed hello costs a rate-limit token, so guessing is slow.

The token is 32 random bytes, generated on the bridge's first start and stored readable only by
you at `~/.vrcnext-plugins/token`. It is the page's proof that it is allowed to talk to the daemon
— any web page open in a browser on the same machine is not. The installer prints it once;
`vrcnext-bridge --print-token` prints it again, `--rotate-token` replaces it (every paired page
must re-pair).

The page keeps the token and the endpoint in `localStorage`, scoped to VRCNext's origin
`http://localhost:<LocalHttpPort>`. VRCNext picks a new random port when its saved one is taken,
and a new port is a new origin, so you would have to paste the token again. The installer's
`--pin-port` fixes the port for that reason.

### Developer mode is off by default

The page only needs the socket. Without flags the bridge serves exactly two paths: `/v1/ws` and
`GET /v1/health`, which says only that it is up and which version. Everything else is for
developing and answers 404 unless the bridge was started with `--dev`: the per-call REST surface
(`GET /v1/describe`, `POST /v1/<service>/<method>` with the token as a Bearer header) and
`POST /v1/remote/eval`, which runs JavaScript inside the VRCNext page. Every call still needs the
token, and anyone holding it can drive the page while `--dev` is on, so leave it off unless you
are developing.

### The Bridge card

Until the bridge is connected, the **Plugins** tab shows only the Bridge card. Its four states:

| State | Meaning |
| :--- | :--- |
| **Not detected** | nothing answered `GET /v1/health` at the endpoint. Not installed, or not started. |
| **Running, connecting…** | health answered; the socket is not open yet. |
| **Running, not paired** | the bridge refused the `hello`. Paste the token and press **Pair**. |
| **Connected** | the socket is open. The rest of the tab appears. |

The card has a **Re-check** button for after you have just started the daemon, and an
**Endpoint** field for a bridge started with a different `--listen` (loopback only; there is no
override). The page cannot tell an uninstalled daemon from an installed one that is stopped, so
there is deliberately no state claiming to.

## Installs are confirmed on the desktop

Installing, updating or uninstalling a plugin puts code into the page, and the page cannot be
the thing that confirms that: any script already running there — a plugin included — could click
its own dialog. So the bridge asks through something the page cannot reach:

- **Linux:** a desktop notification titled *Install a plugin?* (or *Update plugin …?*,
  *Uninstall plugin …?*) with the URL and two buttons, **Confirm** and **Deny**. It has critical
  urgency so it does not expire on its own; closing it is a Deny.
- **Windows:** a topmost Yes/No message box titled *VRCNext Bridge*.

No answer within two minutes is a refusal. If the bridge has no way to ask — typically no
session bus because it started outside the graphical session — every install answers
`approval_unavailable` until it is restarted where a prompt can appear. There is no flag to skip
the prompt. `build` and `state` never prompt: they put nothing new into the page.

## From a plugin: `ctx.native`

Needs the `native` permission. When a plugin runs, the bridge is connected by construction — the
host does not activate plugins until the socket is up — so there is nothing to probe and no
`available` flag. Three methods:

```ts
interface NativeApi {
  targets(): Promise<readonly NativeTarget[]>;
  notify(options: NativeNotifyOptions): Promise<NativeNotifyResult>;
  call(service: string, method: string, params?: unknown): Promise<unknown>;
}
```

`call` reaches the `notify` service only. The bridge's other services belong to the host: `state`
holds every plugin's settings and the permission grants themselves, `outbound` and `osc` are
behind `ctx.http` and `ctx.osc` with gates of their own, and `plugins`, `sql`, `logs` and
`remote` are not a plugin's to call. Naming one rejects with a `PermissionError` before anything
is sent — a grant is remembered per `service/method`, not per parameters, so one "allow" of
`state/set` would otherwise let a plugin rewrite its own grants.

Every one is a bridge call, and each `service/method` is confirmed by the user the first time
the plugin uses it — *Plugin {name} ({id}) wants to call the bridge: notify/send*, with the
parameters in the details. See [Permissions](permissions.md). A denied call rejects with a
`PermissionError`.

### Notifications, targetable

On Linux this is the only route to VR overlays and the desktop, since VRCNext's own
notification features are Windows-gated — see the [platform matrix](limitations.md). Each
destination is a separately addressable **sink**, which is the point: one call can go to VR only,
the desktop only, or both with different presentation.

```ts
// VR only — nothing on the monitor.
await ctx.native.notify({
  title: 'Friend online',
  content: 'Tupper is in Great Pug',
  sinks: ['wayvr'],
  height: 220,
  opacity: 0.85,
  alwaysShow: true,
});

// Both, presented differently in each.
await ctx.native.notify({
  title: 'Player joined',
  content: 'on the desktop',
  timeoutSecs: 4,
  overrides: { wayvr: { content: 'tall panel in VR', height: 220, opacity: 0.85 } },
});
```

Omitting `sinks` means every target the bridge has. `overrides` patches fields for one target;
an override naming a target you are not delivering to is ignored, so carrying presentation for an
overlay the user does not run costs nothing.

**Discover targets rather than hard-coding them.** `targets()` returns each sink's `name`,
`health` (`up`, `unknown`, `down` — `unknown` is honest for fire-and-forget UDP) and the fields it
`honours`:

| Field | `wayvr` | `freedesktop` |
| :--- | :--- | :--- |
| `title`, `content`, `timeoutSecs`, `icon`, `sourceApp`, `sound` | yes | yes |
| `volume`, `audioPath`, `height`, `opacity`, `alwaysShow`, `useBase64Icon` | yes | no |
| `urgency` | no | yes |

**Partial delivery is success.** `notify()` resolves `{ ok, delivered, failed }`; `ok` is true
when at least one target accepted, because that is what happened. It never rejects for an overlay
that is not running — that is a normal state, not an exception. Inspect `failed` if you care
which.

### Any service

`call(service, method, params)` reaches any bridge service over the shared socket, so a bridge
that grows a new service is usable from a plugin immediately, without a plugin-system release.
It rejects when the socket closes before the answer, on timeout, or when the bridge answers with
an error — the latter as a `NativeRequestError` carrying the bridge's own `code`.

### Limits

The bridge validates and bounds every request. Exceeding a limit is a rejection from `call()` or
a `failed` entry from `notify()`:

| Field | Limit |
| :--- | :--- |
| `title` | 200 characters, no control characters, required |
| `content` | 2000 characters |
| `icon` | 128 KB — but a base64 icon over ~65 KB cannot fit in a UDP datagram and `wayvr` refuses it |
| `timeoutSecs` | 0–60; `0` means the target's default |
| `volume`, `opacity` | 0–1 |
| `height` | 16–1024 |
| `sinks` | at most 8 names, 32 characters each |

There is also a rate limit — 5/s with a burst of 10 by default, shared between the socket and
plain HTTP. A notification puts pixels in front of someone wearing a headset; a runaway loop is
otherwise an accident that needs them to take it off.

## Security, briefly

Loopback only, with no override flag. HTTP calls must be `application/json` so browsers must
preflight, and the preflight is refused for non-loopback origins; the WebSocket upgrade checks
`Origin` itself, since CORS does not cover it; the pairing token covers what an origin check
cannot. No service may execute a program or write to a caller-chosen path, except the build
module's one checksum-verified binary. The full reasoning is in the
[bridge README](https://github.com/vrcnext-plugins/vrcnext-bridge#security); what it means for
plugins is on the [security model](security.md) page.

## Running it by hand

The installer registers autostart (systemd user unit, launchd agent, or a Scheduled Task). To
run, inspect or troubleshoot it directly — `--print-token`, `--data-dir`, "no session bus",
`esbuild checksum mismatch` — see the bridge's
[running guide](https://github.com/vrcnext-plugins/vrcnext-bridge/blob/main/docs/running.md).

[← Notifications](notifications.md) · [Context menus →](context-menus.md)
