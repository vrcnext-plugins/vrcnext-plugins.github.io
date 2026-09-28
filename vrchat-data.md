---
title: VRChat data
---

# VRChat data

[← Back to index](./)

`ctx.vrchat` reads what VRCNext knows about VRChat — your friends, your favourites, your groups,
the instance you are in, and any user, avatar, world or group by id — **without a dialog opening
over what the user is doing**.

That last part is the whole point of this API. VRCNext fetches those things for its own modals and
announces the result as an event its page then renders: asking for a user with
`ctx.bridge.send('vrcGetFriendDetail', …)` paints the profile modal, and asking which avatar
someone wears *opens* the avatar dialog. The host therefore wraps the message callbacks Photino
registered before its own, and withholds a reply it asked for from VRCNext. The plugin gets the
data; the screen does not change.

```ts
const instance = await ctx.vrchat.currentInstance();
if (instance !== undefined) {
  for (const user of instance.users) {
    const profile = await ctx.vrchat.user(user.id);
    ctx.logger.info(`${user.displayName}: 18+ ${profile?.ageVerificationStatus ?? 'unknown'}`);
  }
}
```

Everything here is **read-only**. Nothing in this API sends a friend request, joins an instance or
changes anything on the account; for that you need `host:actions` and `ctx.bridge`.

## Permission

`ctx.vrchat` needs the `vrchat` permission (medium), confirmed once per plugin on first use —
*Plugin {name} ({id}) wants to read VRChat data through VRCNext*. One answer covers every method.

`self()` is the exception to the "await" rule: it is synchronous, because VRCNext pushes the
signed-in account, and it returns `undefined` until the grant is in. It is also the fullest
object here — a `VrcSelf` carries your bio, bio links, pronouns, languages, date joined, current
avatar, home location and whether VRChat is running, because the push already contained all of
it.

## Lists

VRCNext keeps these lists anyway and pushes them when they change. The host mirrors the latest of
each, so a call is usually free and never a request the user did not cause.

```ts
ctx.vrchat.self();                    // VrcSelf | undefined, synchronous
await ctx.vrchat.friends();           // your friend list
await ctx.vrchat.favoriteFriends();
await ctx.vrchat.favoriteFriendGroups();  // the same, in their groups, with your names for them
await ctx.vrchat.moderationCounts();  // how many you blocked, muted, hid the avatar of, …
await ctx.vrchat.recentPlayers();     // players VRCNext recorded near you, newest first
await ctx.vrchat.favoriteWorlds();
await ctx.vrchat.recentWorlds();      // worlds you visited
await ctx.vrchat.ownAvatars();        // avatars you uploaded
await ctx.vrchat.favoriteAvatars();
await ctx.vrchat.recentAvatars();     // avatars you wore
await ctx.vrchat.myGroups();
await ctx.vrchat.currentInstance();   // VrcInstance | undefined, with its players
await ctx.vrchat.friendInstances();   // where your friends are, grouped by location
```

Pass `{ cached: false }` to insist on asking VRCNext again:

```ts
const instance = await ctx.vrchat.currentInstance({ cached: false, signal: ctx.signal });
```

A reply to one of these is left for VRCNext to handle as well — that is what refreshes the lists
it shows — so a list call is the one place this API does touch its UI, by keeping it current.

## Lookups

```ts
await ctx.vrchat.user(id);            // full profile, as the user dialog would show it
await ctx.vrchat.userBasic(id);       // name, picture, status — free for a friend
await ctx.vrchat.userGroups(id);      // the groups they show publicly
await ctx.vrchat.userTimeline(id);    // VRCNext's ten most recent records about them
await ctx.vrchat.instanceAvatar(id);  // which avatar a player in your instance wears
await ctx.vrchat.avatar(id);          // including pcRank / questRank / iosRank
await ctx.vrchat.world(id);
await ctx.vrchat.group(id);
```

Each resolves `undefined` when VRCNext has no answer — a legacy account with no `usr_` id, an
avatar no database knows, a timeout. **That is a normal outcome, not an error**: write code that
reports "unknown" rather than code that treats absence as a "no".

Answers are cached for a minute and identical concurrent lookups share one request, so a settings
card that resolves twenty ids costs twenty requests, not four hundred.

## Searches

```ts
const page = await ctx.vrchat.searchUsers('tupper');          // { results, offset, hasMore }
const more = await ctx.vrchat.searchUsers('tupper', { offset: 20 });
await ctx.vrchat.searchWorlds('club', { sort: 'popularity' });
await ctx.vrchat.searchGroups('club');
await ctx.vrchat.searchAvatars('cat');                        // the avatar databases
```

Twenty per page. These are real VRChat API calls made on the user's behalf — search on what the
user asked for, not on a timer.

## Avatar ranks

`pcRank`, `questRank` and `iosRank` are VRChat's own words — `Excellent`, `Good`, `Medium`, `Poor`,
`VeryPoor` — and `''` when unknown. Compare them with `rankIndex`, which is 0 for `Excellent` and
4 for `VeryPoor`:

```ts
const have = rankIndex(avatar.pcRank);
const want = rankIndex('Medium');
if (have === undefined) reportUnverified();       // not "too heavy" — unknown
else if (have > want) reportTooHeavy();
```

## Instances

`VrcInstance` carries the parsed location so you do not have to take it apart:

```ts
const { worldId, instanceId, instanceType, groupId, region, users, capacity } = instance;
```

For a location string from somewhere else — a friend's `location`, a timeline event — the api
exports VRCNext's own parser:

```ts
import { parseLocation, isGroupInstance } from '@vrcnext/plugin-api';

const at = parseLocation(friend.location);
at.key;            // 'wrld_…:12345' — identifies the instance across visits
at.instanceType;   // 'group-plus' | 'friends' | 'public' | … | '' when not an instance
isGroupInstance(at.instanceType);
```

## Pickers

When what you want is for the *user* to choose something, do not build a list — declare a
[picker setting](settings.md#pickers), or open the same picker on demand:

```ts
const ids = await ctx.ui.pickEntity({ kind: 'user', scopes: ['friends', 'favorites'] });
```

A picker declared in your settings needs no permission, because the user fills the form.
`ctx.ui.pickEntity` needs `vrchat` and asks once, because what it returns goes to your plugin.
Either way you get only what the user chose.

## What this does not give you

- **Anything that writes.** Use `host:actions`.
- **Live presence events.** Friends coming online arrive as `friendTimelineEvent` through
  [`ctx.events`](events-and-bridge.md); this API is for asking, not for watching.
- **Fields VRCNext does not fetch.** What is missing is missing at the source, not filtered here.
  Unknown strings are `''`, unknown numbers `0`, unknown tri-states `undefined`.

[← Game log](game-log.md) · [Routes & deep links →](routes-and-links.md)
