---
title: Events & actions
---

# Events & actions

[← Back to index](./)

VRCNext's frontend talks to its C# backend over a Photino channel: roughly **474 outbound
actions** and **310 inbound events**. Plugins get both, behind three permissions:

| Permission | Tone | Unlocks | Confirmed on first use |
| :--- | :--- | :--- | :--- |
| `host:events` | low | `ctx.events`, `ctx.deepLinks` | each event name not in `plugin.json`'s `events`; `onAny` once |
| `host:actions` | high | `ctx.bridge.send`, `ctx.bridge.request` | each action name not in `actions`, with the payload shown |
| `host:intercept` | high | `ctx.bridge.interceptOutbound` | once per plugin |

Names you list in `plugin.json` are granted when the user enables the plugin; anything else is a
prompt the first time. Declare what you know you need — one modal at enable beats five in the
first minute. See [Permissions](permissions.md).

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
