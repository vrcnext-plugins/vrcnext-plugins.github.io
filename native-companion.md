---
title: VRCNext Bridge
---

# VRCNext Bridge

`ctx.native` reaches [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge), an
optional native companion daemon. It exists because VRCNext's page can only speak HTTP and
WebSockets: it cannot open a UDP socket, connect to D-Bus, or reach a unix socket. Anything
needing one of those has to happen in a native process.

The host keeps **one WebSocket** to the bridge open for the life of the page. Every call goes over
it with a correlation id, so several can be in flight at once; the host's log records are mirrored
over it to the bridge's log file; and the bridge's own log lines come back and appear in the Logs
panel under the `bridge` scope. A plain HTTP probe of `/v1/health` is what tells "running" from
"not installed".

Today that means **VR overlay and desktop notification targets**. On Linux this is the only route
to either, since VRCNext's own notification features are Windows-gated — see the
[platform matrix](limitations.md).

## It is optional, and your plugin must act like it

Most users will not have it installed. Every method degrades cleanly:

| Method | Without the bridge |
| :--- | :--- |
| `available` | `false` |
| `status` | `'not_detected'` |
| `ready` | resolves `false` |
| `probe()` | resolves `false` |
| `describe()` | resolves `undefined` |
| `targets()` | resolves `[]` |
| `notify()` | resolves `{ ok: false, delivered: [], failed: [] }` |
| `call()` | **rejects** |

`notify()` and `targets()` never reject. Only `call()` does: with a `NativeRequestError` carrying
the bridge's own `code` when the daemon answered and said no — a `bad_request` there means it is
running fine and your request was wrong, which is why a refused request does **not** change
`status` — or with a transport error when the socket did not open within the call's timeout.

A missing bridge is a normal state, not an exception — and a rejected promise inside an event
handler is an unhandled rejection waiting to happen, which is why `notify()` resolves instead.

### Await `ready`, do not read `available`, inside `activate`

This is the one easy mistake. The host probes the bridge at boot, and that probe is usually
**still in flight** when your plugin activates. `available` is a synchronous snapshot of it, so
branching on it there is a race — true if the daemon answers quickly, false if it does not, and
your VR notifications vanish with no error.

```ts
// Correct: one shared probe, awaited.
if (!(await ctx.native.ready)) {
  ctx.logger.info('No bridge; skipping VR notifications.');
  return;
}
```

```ts
// Racy: may be false purely because the probe has not come back yet.
if (!ctx.native.available) return;
```

`available` is the right choice *later* — in a settings panel, say, where you want to render the
current state without awaiting anything. `ready` resolves once and is shared between all readers.

### Three states, not two

`status` is the finer answer, for anything that shows the user a colour:

| `status` | Meaning | Shown as |
| :--- | :--- | :--- |
| `not_detected` | nothing answered the health probe — not installed, or not started | grey |
| `running_not_connected` | the daemon answered, but the socket is not open yet, or any more | yellow |
| `connected` | the socket is open; calls go through | green |

The page cannot tell an uninstalled daemon from an installed one that is stopped, so there is
deliberately no state claiming to.

A call made while the socket is down does not fail at once: it forces a reconnect and waits up to
its timeout. The first call after the user starts the daemon is therefore the one that works, not
the one that tells them to retry.

## Targeting

Each destination is a separately addressable **sink**. This is the whole point of the API: one call
can go to VR only, the desktop only, or both with different presentation.

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
```

```ts
// Both, presented differently in each.
await ctx.native.notify({
  title: 'Player joined',
  content: 'on the desktop',
  timeoutSecs: 4,
  overrides: {
    wayvr: { content: 'tall panel in VR', height: 220, opacity: 0.85 },
  },
});
```

Omitting `sinks` means every target the bridge has. `overrides` patches fields for one target
alone; anything not patched is inherited. An override naming a target you are not delivering to is
ignored, so carrying presentation for an overlay the user does not run costs nothing.

## Discover targets; do not hard-code them

```ts
const targets = await ctx.native.targets();
// [{ name: 'wayvr', health: 'down', honours: ['height', 'opacity', ...], description: '...' }]
```

Hard-coding `'wayvr'` works today and breaks the moment someone runs a different overlay. Each
target reports what it `honours`, because not every field means something everywhere:

| Field | `wayvr` | `freedesktop` |
| :--- | :--- | :--- |
| `title`, `content`, `timeoutSecs`, `icon`, `sourceApp`, `sound` | yes | yes |
| `volume`, `audioPath` | yes | no |
| `height`, `opacity`, `alwaysShow` | yes | no |
| `urgency` | no | yes |
| `useBase64Icon` | yes | no — the spec takes a pixel buffer, not an encoded image |

A field a target does not honour is ignored, not an error.

## Partial delivery is success

```ts
const result = await ctx.native.notify({ title: 'Hello' });
// { ok: true, delivered: ['freedesktop'], failed: [{ sink: 'wayvr', error: '...' }] }
```

`ok` is true when **at least one** target accepted — because that is what happened: the
notification reached the user somewhere. Inspect `failed` if you care which.

## Services beyond notifications

The bridge is a service host, not a notification daemon. `call()` is the forward-compatible
path — a bridge that grows a new service is usable immediately, without waiting for a
plugin-system release:

```ts
const result = await ctx.native.call('notify', 'targets', {});
```

`describe()` tells you what a running bridge actually offers:

```ts
const description = await ctx.native.describe();
// { version: '0.1.0', services: { notify: { methods: [...], targets: [...] } } }
```

## Limits

Requests are validated and bounded by the bridge. Exceeding a limit produces a rejection from
`call()`, or a `failed` entry from `notify()`:

| Field | Limit |
| :--- | :--- |
| `title` | 200 characters, no control characters, required |
| `content` | 2000 characters; newlines and tabs allowed |
| `icon` | 128 KB — but a base64 icon over ~65 KB cannot fit in a UDP datagram and `wayvr` will refuse it |
| `timeoutSecs` | 0–60; `0` means the target's default |
| `volume`, `opacity` | 0–1, finite |
| `height` | 16–1024 |
| `sinks` | at most 8 names, 32 characters each |
| request body | 192 KB |

There is also a rate limit — 5/s with a burst of 10 by default, shared between the socket and
plain HTTP so switching transports buys nothing. A notification puts pixels in front of someone
wearing a headset; a runaway loop is otherwise an accident that needs them to take it off.

## If you moved the daemon

The bridge's `--listen` is configurable, so the host's endpoint is too. **Plugins → Plugin
System → VRCNext Bridge** has an editable *Endpoint* field; changing it reconnects and re-probes
immediately and remembers the choice. `ctx.native.endpoint` reports where the host is currently looking.

## Installing it

See the [bridge README](https://github.com/vrcnext-plugins/vrcnext-bridge) and its
[running guide](https://github.com/vrcnext-plugins/vrcnext-bridge/blob/main/docs/running.md).
**Plugins → Plugin System** in VRCNext shows the three-state dot, which targets exist, and has a
**Re-check** button for after you have just started it. The dot follows the socket, so it turns
green on its own once the daemon is up.

## Security, briefly

The bridge listens on loopback only, and requires `Content-Type: application/json` on HTTP calls
specifically so that browsers must send a CORS preflight — which it refuses for origins that are
not loopback. A WebSocket has no preflight, so the bridge checks the upgrade's `Origin` header
itself and refuses the same origins. That is what stops an arbitrary web page the user has open
from pushing notifications into their headset. No sink may ever execute a program. The full reasoning is in the
[bridge README](https://github.com/vrcnext-plugins/vrcnext-bridge#security).

From a plugin author's point of view the relevant part is narrower: the bridge is not a sandbox
escape you are being handed, it is a narrow set of message-passing capabilities. See the
[security model](security.md) for what plugins can already do without it.
