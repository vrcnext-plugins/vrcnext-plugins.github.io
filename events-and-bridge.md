---
title: Events & actions
---

# Events & actions

[← Back to index](./)

VRCNext's frontend talks to its C# backend over a Photino channel: **415 actions** the backend
dispatches and **311 events** it sends, as read from its source (see
[Checking against VRCNext](#checking-against-vrcnext)). Plugins get both, behind three
permissions:

| Permission | Tone | Unlocks | Confirmed on first use |
| :--- | :--- | :--- | :--- |
| `host:events` | low | `ctx.events`, `ctx.deepLinks` | granted at enable, for the names in `plugin.json`'s `events` |
| `host:actions` | high | `ctx.bridge.send`, `ctx.bridge.request` | granted at enable, for the names in `actions`, with the payloads shown |
| `host:intercept` | high | `ctx.bridge.interceptOutbound` | once per plugin |

**The two lists are exhaustive.** An action or event name that is not in `plugin.json` is
refused — it throws, and no prompt is offered. Both are constants in your source, so your
manifest can name every one you will ever use, and the enable dialog can therefore show the
user a complete list of what the plugin reaches. `"*"` declares the whole stream, which is what
`ctx.events.onAny` needs; the dialog shows that too.

**One event is different.** `vrcPrefillLogin` is VRCNext filling its login form from the saved
account, and it carries the **password** in plain text. Declaring it grants nothing: the first
subscription asks the user, in those words, at high tone. `onAny` never delivers it — a plugin
that wants it has to ask for it by name. `host:intercept` sees the outbound `vrcLogin` and
`vrc2FA` actions, which is part of why it is high risk.

This is deliberately unlike `hosts`, where an undeclared host still prompts: a URL can come
from a setting the user typed, an action name cannot. See [Permissions](permissions.md).

## Checking against VRCNext

VRCNext publishes no protocol, so the plugin system reads one out of its C# source:
[`protocol/vrcnext-protocol.json`](https://github.com/vrcnext-plugins/vrcnext-plugin-system/blob/main/protocol/vrcnext-protocol.json)
lists every action with the arguments its handler reads and whether a Linux build drops it,
every event with its payload fields, and the frontend's element ids and classes, pinned to the
VRCNext commit it was read from. `@vrcnext/plugin-api` exports the names as types:
`VrcnextAction`, `VrcnextEvent`, `VrcnextWindowsOnlyAction` and `VrcnextEventFields`.

Check your plugin against it from a clone of the plugin system:

```bash
node scripts/check-vrcnext-protocol.mjs ../my-plugin
```

It reports an action or event VRCNext does not have (in your sources and in `plugin.json`), an
event used as an action, an argument the action's handler never reads, a selector naming an
element VRCNext does not have, and — as a warning — an action VRCNext drops on Linux and macOS.
Nothing is asked of the network; it reads your files and the JSON.

(`ctx.bridge` is the VRCNext bridge — the Photino channel — not the
[VRCNext Bridge](native-companion.md) daemon, which is `ctx.native`. The name predates the
daemon.)

## Receiving events

```ts
// Typed — payload.friendName is a string.
ctx.events.on('friendTimelineEvent', (payload) => {
  ctx.logger.info(`${payload.friendName} → ${payload.type}`);
});

ctx.events.once('setPlatform', (payload) => {
  if (payload.isLinux) ctx.logger.info('Running on Linux.');
});

// Every event, for diagnostics.
ctx.events.onAny((envelope) => ctx.logger.debug(envelope.type));

// Promise form, abortable.
const payload = await ctx.events.next('vrcMyProfile', ctx.signal);
```

Subscriptions are registered on your disposable bag, so forgetting to unsubscribe is not a leak.
A subscription to an event that still needs the user's answer returns at once and starts
delivering once they confirm — `activate` never blocks on a prompt — and unsubscribing before
the answer simply cancels it.

## Typed vs unknown payloads

Only payloads **read directly out of the VRCNext C# source** are typed. Everything else arrives
as `unknown` and must be narrowed:

```ts
ctx.events.on('vrcCurrentInstance', (payload) => {
  if (typeof payload !== 'object' || payload === null) return;
  const worldName = (payload as { worldName?: unknown }).worldName;
  if (typeof worldName === 'string') { /* … */ }
});
```

This is deliberate. A guessed payload type is worse than none: it compiles, then fails at
runtime. Currently typed: `setPlatform`, `log`, `toast`, `dbMigrationProgress`, `openDeepLink`,
`friendTimelineEvent`, `customThemes`.

To type more for your own build, augment the map:

```ts
declare module '@vrcnext/plugin-api' {
  interface VrcnextEventMap {
    vrcCurrentInstance: { readonly worldName: string; readonly empty?: boolean };
  }
}
```

## Sending actions

```ts
ctx.bridge.send('vrcLaunchAndJoin', { location: '', vr: false });
ctx.bridge.send('vrcUpdateStatus', { status: 'join me', statusDescription: 'Come say hi' });
```

> Actions operate on the user's **real VRChat account**. An action name not pre-declared in
> `plugin.json` is confirmed by the user the first time, with your payload shown verbatim —
> *Plugin {name} ({id}) wants to call VRCNext action vrcUpdateStatus*. After that, `send` is
> fire-and-forget. Treat every call as outward-facing.

### Request/response

```ts
const themes = await ctx.bridge.request(
  'getCustomThemes',
  {},
  { expect: 'customThemes', timeoutMs: 5000, signal: ctx.signal },
);
```

> **Correlation is by event type only.** VRCNext has no request ids, so two concurrent requests
> expecting the same response type can resolve with each other's payload. Use `request` for
> read-style actions; do not rely on it where a mix-up would matter.

## Intercepting outbound traffic

```ts
ctx.bridge.interceptOutbound((action, raw) => {
  ctx.logger.debug(`→ ${action}`);
  if (action === 'vrcDeleteAvatar') return false;  // drop it
  return undefined;                                 // let it through
});
```

Returning `false` drops the message before it reaches the backend — including VRCNext's own
traffic. Powerful and easy to misuse, which is why it is its own *High risk* category and is
confirmed once per plugin. Log before you block.

[← Settings](settings.md) · [UI injection →](ui.md)
