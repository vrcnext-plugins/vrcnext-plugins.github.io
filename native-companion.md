---
title: Native companion
---

# Native companion

`ctx.native` reaches [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge), an
optional local daemon. It exists because VRCNext's page can only speak HTTP: it cannot open a UDP
socket, connect to D-Bus, or reach a unix socket. Anything needing one of those has to happen in a
native process.

Today that means **VR overlay and desktop notification targets**. On Linux this is the only route
to either, since VRCNext's own notification features are Windows-gated — see the
[platform matrix](limitations.md).

## It is optional, and your plugin must act like it

Most users will not have it installed. Every method degrades cleanly:

| Method | Without the companion |
| :--- | :--- |
| `available` | `false` |
| `ready` | resolves `false` |
| `probe()` | resolves `false` |
| `describe()` | resolves `undefined` |
| `targets()` | resolves `[]` |
| `notify()` | resolves `{ ok: false, delivered: [], failed: [] }` |
| `call()` | **rejects** |

`notify()` and `targets()` never reject. Only `call()` does, and its error carries the companion's
own `code` and `status` — a `bad_request` there means the daemon is running fine and your request
was wrong, which is also why a rejected request does **not** flip `available` to `false`.

A missing companion is a normal state, not an exception — and a rejected promise inside an event
handler is an unhandled rejection waiting to happen, which is why `notify()` resolves instead.

### Await `ready`, do not read `available`, inside `activate`

This is the one easy mistake. The host probes the companion at boot, and that probe is usually
**still in flight** when your plugin activates. `available` is a synchronous snapshot of it, so
branching on it there is a race — true if the daemon answers quickly, false if it does not, and
your VR notifications vanish with no error.

```ts
// Correct: one shared probe, awaited.
if (!(await ctx.native.ready)) {
  ctx.logger.info('No companion; skipping VR notifications.');
  return;
}
```

```ts
// Racy: may be false purely because the probe has not come back yet.
if (!ctx.native.available) return;
```

`available` is the right choice *later* — in a settings panel, say, where you want to render the
current state without awaiting anything. `ready` resolves once and is shared between all readers.

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

Omitting `sinks` means every target the companion has. `overrides` patches fields for one target
alone; anything not patched is inherited. An override naming a target you are not delivering to is
ignored, so carrying presentation for an overlay the user does not run costs nothing.

## Discover targets; do not hard-code them

```ts
const targets = await ctx.native.targets();
// [{ name: 'wayvr', health: 'unknown', honours: ['height', 'opacity', ...], description: '...' }]
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

The companion is a service host, not a notification daemon. `call()` is the forward-compatible
path — a companion that grows a new service is usable immediately, without waiting for a
plugin-system release:

```ts
const result = await ctx.native.call('notify', 'targets', {});
```

`describe()` tells you what a running companion actually offers:

```ts
const description = await ctx.native.describe();
// { version: '0.1.0', services: { notify: { methods: [...], targets: [...] } } }
```

## Limits

Requests are validated and bounded by the companion. Exceeding a limit produces a rejection from
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

There is also a rate limit — 5/s with a burst of 10 by default. A notification puts pixels in front
of someone wearing a headset; a runaway loop is otherwise an accident that needs them to take it
off.

## If you moved the daemon

The bridge's `--listen` is configurable, so the host's endpoint is too. **Plugins → Plugin
System → Native companion** has an editable *Endpoint* field; changing it re-probes immediately and
remembers the choice. `ctx.native.endpoint` reports where the host is currently looking.

## Installing it

See the [bridge README](https://github.com/vrcnext-plugins/vrcnext-bridge) and its
[running guide](https://github.com/vrcnext-plugins/vrcnext-bridge/blob/main/docs/running.md).
**Plugins → Plugin System** in VRCNext shows whether the host connected, which targets exist, and
has a **Re-check** button for after you have just started it.

## Security, briefly

The companion listens on loopback only, and requires `Content-Type: application/json` specifically
so that browsers must send a CORS preflight — which it refuses for origins that are not
loopback. That is what stops an arbitrary web page the user has open from pushing notifications
into their headset. No sink may ever execute a program. The full reasoning is in the
[bridge README](https://github.com/vrcnext-plugins/vrcnext-bridge#security).

From a plugin author's point of view the relevant part is narrower: the companion is not a sandbox
escape you are being handed, it is a narrow set of message-passing capabilities. See the
[security model](security.md) for what plugins can already do without it.
