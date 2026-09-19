---
title: Notifications
---

# Notifications

[← Back to index](./)

VRCNext has four distinct notification surfaces. They are **not** interchangeable, and one of
them is Windows-only.

| Surface | Reaches | Availability |
| :--- | :--- | :--- |
| In-app toast | The VRCNext window | Everywhere |
| Notification toast | The window, styled as an invite / friend request | Everywhere |
| Desktop + VR | OS tray toast **and** the SteamVR wrist overlay | **Windows only** |
| Confirmation modal | The window, blocking | Everywhere |

## In-app toast

```ts
ctx.notifications.toast({ message: 'Saved.' });
ctx.notifications.toast({ message: 'Could not reach the API.', ok: false });
```

## Notification-styled toast

```ts
ctx.notifications.notifToast({
  kind: 'invite',          // 'invite' | 'friendRequest' | 'notification'
  sender: 'SomeUser',
  message: 'wants to hang out',
});
```

Renders in VRCNext's own invite/friend-request style rather than the plain toast.

## Desktop and VR, in one call

```ts
if (ctx.notifications.desktopAvailable) {
  ctx.notifications.desktop({
    title: 'Player joined',
    subtitle: 'SomeUser',
    accent: 'info',                  // 'accent' | 'info' | 'ok' | 'warn' | 'err'
    imageUrl: 'https://…/avatar.png',
    friendId: 'usr_…',
  });
}
```

This maps to VRCNext's `afTrayNotify`, which fans out to **three** places at once:

1. The OS tray toast (Windows toast / your desktop's notification daemon).
2. The SteamVR wrist overlay's **notification list** (persistent entry).
3. The wrist overlay's **toast queue** (transient popup in VR).

> ### Yes, VRCNext has an in-game VR overlay
>
> It is a SteamVR wrist overlay, run in a separate `--vr-subprocess` process, with its own
> notification list, toast queue, toast sounds and TTS. Plugins reach it through
> `notifications.desktop()` — there is no separate JS action to target the overlay alone, since
> the `vro_*` protocol is internal between VRCNext and its VR subprocess.
>
> Delivery is best-effort and depends on things a plugin cannot detect:
>
> - The whole action is wrapped in `#if WINDOWS`. **On Linux it is a no-op** — `desktopAvailable`
>   is `false` and the call is logged instead.
> - The tray toast additionally requires **Minimize to tray** *and* **Tray notifications**
>   enabled in VRCNext settings.
> - The VR parts require SteamVR running and the wrist overlay connected.

## Confirmation modal

```ts
const confirmed = await ctx.notifications.confirm({
  title: 'Reset settings',
  message: 'Restore every setting to its default?',
  confirmLabel: 'Reset',
  icon: 'restart_alt',
});
if (confirmed) await ctx.settings.reset();
```

Resolves `true` **only** on confirm. Cancelling, pressing Escape, or clicking the backdrop all
resolve `false` — VRCNext's modal only calls back on confirm, so the host watches for the
element's removal to guarantee the promise always settles.

## Degradation

Every method looks its global up at call time. If a VRCNext update renames or removes one, the
call falls back to the plugin log rather than throwing inside your event handler. You will see
a line in the Logs panel instead of a broken plugin.

[← UI injection](ui.md) · [Logging →](logging.md)

## Beyond VRCNext's own surfaces

Everything above goes through VRCNext, which means it inherits VRCNext's platform gates — the tray
toast and the SteamVR overlay are Windows-only, and `ctx.notifications.desktopAvailable` is `false`
on Linux.

The [native companion](native-companion.md) is the way around that. It is a separate local process,
so it is not subject to VRCNext's gating, and it can address a VR overlay and the desktop
notification daemon as separate targets:

```ts
await ctx.native.notify({
  title: 'Friend online',
  sinks: ['wayvr'],        // VR only — nothing on the monitor
  height: 220,
  opacity: 0.85,
});
```

It is optional and must be installed separately, so guard on `ctx.native.available`.
