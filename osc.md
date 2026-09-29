---
title: OSC
---

# OSC

[← Back to index](./)

> ## OSC is Windows-only *in VRCNext* — the bridge covers Linux
>
> `MessageRouter.IsWindowsOnlyAction` drops any action starting with `osc` followed by an
> uppercase letter, so **`oscSend`, `oscSendRaw`, `oscConnect` and `oscDisconnect` never reach
> the backend on Linux.** VRCNext hides its own OSC Tool tab there, and its Action Flow OSC
> blocks are equally inert — they emit the same two actions.
>
> Where VRCNext holds nothing, the [bridge](native-companion.md)'s `osc` service holds the
> sockets instead, and `ctx.osc` routes to it. You write the same code either way. Still guard,
> because an older bridge has no `osc` service:
>
> ```ts
> if (!ctx.osc.available) {
>   ctx.logger.info('No OSC on this setup; skipping.');
>   return;
> }
> ```
>
> Calls made when `available` is `false` log a warning rather than failing silently.

`ctx.osc` needs the `osc` permission, and the user confirms it once per plugin on first use —
*Plugin {name} ({id}) wants to send and receive OSC through VRCNext*.

Plugins never open a socket themselves. On Windows VRCNext owns them: it sends to
`127.0.0.1:9000`, listens on `9001` with `SO_REUSEADDR`, and advertises an extra receive port
over OSCQuery, and plugins ask it to send and subscribe to what it receives. On Linux the bridge
binds the same two ports on loopback and pushes what arrives to the page. VRCNext's own path is
preferred wherever it works, so there is still one OSCQuery advertisement and nothing fights
VRChat over port 9001.

## Sending

```ts
ctx.osc.connect();                      // starts VRCNext's OSC service if idle

ctx.osc.send('VRCEmote', 'int', 3);     // → /avatar/parameters/VRCEmote
ctx.osc.send('IsHappy', 'bool', true);
ctx.osc.send('Blend', 'float', 0.75);

ctx.osc.sendRaw('/chatbox/input', 'bool', true);   // arbitrary address
```

Overloads are typed: `send(name, 'bool', 3)` is a compile error.

> VRChat's chatbox enforces ≤144 characters and ~1500 ms between messages. VRCNext applies this
> to its own chatbox feature; `sendRaw` does not, so respect it yourself or VRChat will drop
> your messages.

## Receiving

```ts
ctx.osc.onParam((event) => {
  // event.name is without the /avatar/parameters/ prefix
  ctx.logger.debug(`${event.name} = ${String(event.value)}`);
});

ctx.osc.onAvatarChange((event) => {
  ctx.logger.info(`Avatar ${event.avatarId}, ${event.parameters.length} parameters`);
});
```

`onAvatarChange` fires when VRChat reports an avatar switch, carrying the new parameter list
with `hasInput` / `hasOutput` flags.

## Caveats

- **Check `available` first.** It is true when VRCNext carries OSC (Windows) or when the bridge
  offers its `osc` service (any platform, a bridge new enough to have one).
- **`onAvatarChange` carries no parameter list on the bridge path.** VRChat sends only the avatar
  id on `/avatar/change`; the list VRCNext's own event adds is empty rather than invented.
- **Parameters arrive only while OSC is enabled in VRChat.** If nothing fires, OSC is off in the
  VRChat radial menu, or another app has bound 9001 without `SO_REUSEADDR`.
- `onParam` can fire *very* often — dozens of times a second with a parameter-heavy avatar.
  Filter by name early and never do DOM work per event.
- `connect()` is shared. Calling `disconnect()` stops OSC for VRCNext and every other plugin, so
  do not call it unless you own the feature.

[← Context menus](context-menus.md) · [Game log →](game-log.md)
