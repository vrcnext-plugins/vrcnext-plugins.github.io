---
title: Club Security
---
{% raw %}

# Club Security

[← Back to index](./)

Source: [vrcnext-club-security-plugin](https://github.com/vrcnext-plugins/vrcnext-club-security-plugin)

Reports who joins your instance, and whether they meet your club's rules.

Everything that varies per club is a **preset**: which instances it watches, what it expects of a
joiner, and where its reports go. A join is reported once per enabled preset whose filters match
the instance, so one account can run the door for several clubs at once.

Every report carries a verdict, and it is the worst of the preset's checks:

| Verdict | Means |
| :--- | :--- |
| ✅ green | Every requirement was checked and holds. |
| ⚠️ orange | Something could not be checked: an unknown avatar rank, a hidden age status, a membership the player does not show publicly. |
| ⛔ red | A requirement was checked and does not hold: a VeryPoor avatar under a Medium floor, a verified account that is not 18+, a non-friend where friendship is required. |

The verdict reaches your templates as `{result}`, `{resultText}`, `{resultEmoji}` and
`{resultColor}` — the last one colours the Discord embed's bar.

## A preset

| Section | Setting | Meaning |
| :--- | :--- | :--- |
| Identity | Preset name, Enabled | The name appears in every report as `{preset}`. |
| Filters | Instance types, Group, Worlds | One of each kind. Empty means "any". A pickable group and worlds, not pasted ids. |
| Requirements | Require 18+, PC / Quest / iOS avatar rank at least, Must be a member of, Must be on my friend list, Trust score at least | Each one becomes a check with its own verdict. The rank floors are one ordered scale: `Unknown or better` accepts anything and so checks nothing, while `VeryPoor or better` — the next rung — accepts every rank that exists but not an avatar nothing could rank. |
| Exceptions | Never check these people | Picked from your friends, favourites or the instance. Staff join and change avatar without a report. |
| Avatars | Warn when someone here switches avatar | Re-checks the avatar limits alone against the new avatar. |
| Channels | In-app toast, Desktop, VR overlay, Discord | Per preset, so a strict club can post to Discord while a relaxed one only toasts. |
| Formats | Report template, VR overlay template, Discord embed | The text report, the plain-text one for VR, and the embed. |

## The Actions card

Three buttons on the Club Security tab, each running the presets you have enabled:

| Button | What it does | Sends? |
| :--- | :--- | :--- |
| **Test me** | Runs *your own* account through every enabled preset, bypassing the self check and the exception list that normally keep you out of reports. With VRChat closed it uses the last instance VRCNext recorded you in, and a clearly fake `Example World` when it has none. | Yes |
| **Check everyone here** | Every player in your instance against every enabled preset, up to twelve. The door check: who in this room would the rules have turned away? | No — panel only |
| **Replay last join** | The last player VRCNext recorded, through every preset, to their channels. Works with VRChat closed. | Yes |

All three ignore the instance filters: a preset that would not have watched this instance still
says what it would have said, and notes that in the log. Hover a button for the detail.

**Test me** delivers to the channels, because "how would I be treated" includes the
notification — a desktop toast that never arrives and a webhook that stays silent are exactly
what it is worth pressing to find. It is one player on one click. **Check everyone here** does
not, because twelve players across several presets in a single click would flood the channel.

## Avatar switches

A club's avatar rules are usually broken *after* the door, by someone who came in on a light
avatar and changed. With **Warn when someone here switches avatar** on, a preset re-checks its
PC and Quest limits whenever a player already in your instance changes into another avatar, and
reports it with the same three colours. Nothing else is re-read — their age status and their
memberships did not change with their avatar — so the report carries the avatar checks only.

VRChat's log says nothing about other people's avatars, so this comes from the instance VRCNext
already keeps: a switch is noticed at the next refresh, within about half a minute.

## Testing the channels

**Replay last join** takes the most recent player VRCNext recorded near you and runs them
through every enabled preset again, each with its own requirements, so the test shows what that
preset would really have reported. It reads VRCNext's own records — the recent players and
their timeline — so it works with VRChat closed, which is when a webhook is usually being set
up. Filters are not applied, because the point is to exercise the channels; a preset that would
not have watched that instance says so in the plugin log.

## The default report

```
✅ Tupper joined · Saturday Night
✅ 18+ verified: 18+
✅ PC avatar rank: Good
✅ Group member: member
Avatar: Ava (PC Good · Quest Poor)
```

First line = title, rest = body. A line whose placeholders all came out empty is left out, which
is why the rejoin line only appears for someone who has been in that instance before.

The Discord embed is separate from that text and has its own editor. Out of the box it carries
seven fields — Requirements, Avatar, Activity, Moderation, Info, Names and Recently — where
Activity, Moderation and Info mirror the cards VRCNext shows on a profile. Each field's value is an ordinary template over the
raw variables below, so a club reorders the rows, rewords the labels or deletes a field outright.
A row whose value is empty renders as nothing and Discord drops the blank line, so a profile
VRCNext knows little about quietly shrinks instead of filling with "Unknown"; Moderation lists
only what you have actually done, so it disappears for a player you never moderated.

Four switches — **Add the Activity field**, **Add the Moderation field**, **Add the Info field**
and **Add the Names field** — drop those fields from the plugin's own embed without writing one.
They apply only while **Use a custom embed** is off and are hidden while it is on, because a club
with its own embed deletes the field it does not want, and the field's name is theirs to change
by then anyway.

A fifth, **Count the person in the title**, drops the `for the 5th time` clause — the whole
clause, not just the number, so the title does not end in a dangling "for the". Same rule: only
while **Use a custom embed** is off.

### Names is a weaker claim than it looks

VRChat publishes no name history, and VRCNext keeps one current display name per player and
records no rename. What the Names field shows is incidental: every instance record VRCNext wrote
kept a copy of the name the player carried **at that moment**, so the distinct names across their
records are the ones VRCNext happened to witness — while you were in an instance with them, since
it started recording, on this machine. A rename between two of your meetings leaves no trace, and
a player you met once has no history at all.

Two sources answer it, and the switch decides which. **Off**, the names come from the players on
the ten timeline records a report already fetched — free, and enough to reveal a rename for most
players who have ever renamed, but nobody's whole history: a player whose renames are spread over
years of records shows one name where the database shows eight. **On**, the same question is asked
of every record there is.

It is empty both for a player who has never renamed and
for one whose renames nobody saw. Those two are not the same thing, and nothing can tell them
apart, which is why the field says nothing rather than "no previous names". The current name is
left out — it is already the title and the author line — so the field disappears entirely for
most people, and appears for the ones worth a second look.

## The database switch

**Read VRCNext's database for the full picture**, in the plugin's own settings, is off by default
and asks for the optional `sql` permission the moment you turn it on. Decline, or leave it off,
and nothing breaks: every fact behind it has a free source in what VRCNext already holds in the
page, and the reports get shallower rather than wrong.

Why there are two sources at all: VRCNext answers `getTimelineForUser` with **ten records** —
hardcoded, the mini-timeline its own profile modal draws — and the page caches only the window it
has fetched. Anything that counts a whole history is therefore unanswerable from the page at any
cost, which is the only reason the database is read.

| Fact | Switch off | Switch on |
| :--- | :--- | :--- |
| Names they went by | The names on the players of those ten records. Reveals a rename for most players who have renamed, to whatever depth ten records reach | Every name in every record, each dated |
| Activity log, and its pinned oldest line | The oldest of the ten records | VRCNext's genuinely oldest record — usually the day you met |
| `eventCount` / `eventOrdinal`, the count in the title | Instances among those ten, so it stops climbing at ten | Every instance VRCNext ever recorded them in |
| `dbEntries` | **Left out.** A count of the whole database has no cheaper substitute, and a floor of "at least ten" would say nothing | Every row that mentions them |
| Everything else — requirements, avatar, ranks, trust, moderation, Info, `meets`, `firstMet`, `timeTogether` | Unaffected: all of it is on the payload VRCNext already sent, or in the lists it keeps in the page | Identical |

The access is read-only and local. The bridge opens the file `SQLITE_OPEN_READ_ONLY` with
`query_only` set, so a statement that asks to write is refused by SQLite itself; VRChat is never
asked anything, and nothing leaves the machine that was not already going to Discord.

`{name}` is short for `{{ name }}`; the full template language (conditions, filters, `{% if %}`
blocks) is in the plugin system's API reference. The VR template is separate because WayVR draws
with a single font and shows nothing for emoji.

Every name below is also a chip under the template editors: hover it for a description, click
it to copy `{name}`. Text is checked as you type, and a name the plugin does not provide turns
the field red rather than rendering as nothing an hour later.

| Kind | Variables |
| :--- | :--- |
| Verdict | `result` `resultText` `resultEmoji` `resultColor` `checksText` `checksPlainText` `failedText` `unverifiedText` |
| Player | `name` `userId` `profileUrl` `userImageUrl` `platform` `platformEmoji` `isFriend` `friendText` `ageVerified` `ageVerifiedText` `ageVerifiedEmoji` `ageStatus` |
| Trust | `trustScore` `trustScoreText` `trustScoreEmoji` `trustText` — the standing as a percentage, empty when the profile could not be read. Whether it is *required* is the preset's **Trust score at least**; these render it wherever you want it |
| Avatar | `avatar` `avatarId` `avatarImageUrl` `avatarLink` `avatarPlain` `ranksText` `pcRank` `pcRankText` `pcRankEmoji` `questRank` `questRankText` `questRankEmoji` `iosRank` `iosRankText` `iosRankEmoji` — `ranksText` lists one line per platform the avatar is built for, so a platform it has no build for is left out rather than reported as unknown |
| Activity | `logText` — the player's recent records as Discord lines |
| How you know them | `meets` `meetsText` `timeTogether` `timeTogetherSeconds` `firstMet` `firstMetAgo` `firstMetSince` `lastSeenAgo` `lastSeenSince` `dbEntries` — VRCNext's own records of the two of you. `dbEntries` counts every row in its database that mentions them and needs the `sql` permission |
| Names they went by | `nameHistoryText` `previousNames` `nameCount` — the names VRCNext recorded for them before this one, as bullet lines with the date each was last seen, as one comma-separated line, and as a count. Deeper with the database switch on, still present without it; see the caveat above. Empty for a player it has only ever seen under one name |
| What you did to them | `blocked` `muted` `chatMuted` `avatarHidden` `interactOff`, each with an `…Emoji` — your own moderation, read from the lists VRCNext already holds. Empty, not `false`, when a list was never loaded |
| Who they are | `trustRank` `languages` `pronouns` `status` `statusText` `statusDescription` `note` `dateJoined` `joinedAgo` `joinedSince` `lastLoginAgo` `lastLoginSince` `lastActivityAgo` `lastActivitySince` `allowAvatarCopying` `allowAvatarCopyingText` |
| Club | `preset` `inGroup` `inGroupText` `inGroupEmoji` |
| What the preset asked for | `presetRequiredAge` `presetRequiredFriend` `presetRequiredPcRank` `presetRequiredQuestRank` `presetRequiredIosRank` `presetRequiredGroup` `presetRequiredTrustScore` — the floors this preset set, as opposed to what the joiner turned out to be. A requirement it does not check has no value, so a line naming one is dropped rather than printing "any" |
| History | `rejoin` `rejoinText` `rejoinEmoji` `rejoinAgo` `rejoinSince` `rejoinAt` — all about *this* instance |
| How well the club knows them | `eventCount` `eventOrdinal` — how many of this preset's instances VRCNext has seen them in, this one included, as a number and as `5th`. One per instance, so a weekend spent in one room counts once. A player can be new to this room and a regular at the door, which is what `rejoin` cannot say. With the `sql` permission it counts every instance in VRCNext's database; without it, only the ten records a timeline read returns, so it stops climbing at ten. Empty when VRCNext has no history at all, so the title counts nothing rather than claiming a hundredth visit is the first |
| Place and time | `world` `worldId` `instanceType` `instanceId` `location` `time` `date` `timestamp` |

`avatar`, `avatarLink` and `avatarPlain` are empty when VRCNext cannot name the avatar, which
drops the line — and, in an embed, the whole field — rather than printing "Unknown".

Booleans (`ageVerified`, `inGroup`, `rejoin`, `isFriend`) are empty when unknown, so
`{{ "yes" if rejoin else "no" }}` and `{% if inGroup == false %}…{% endif %}` both behave. A
template that does not parse is reported in the log and the default is used instead.

## Where each fact comes from

Everything goes through `ctx.vrchat`, which reads VRCNext's data **without opening its dialogs**.

| Fact | Source | Caveat |
| :--- | :--- | :--- |
| 18+ verification | The joiner's profile. `18+` passes, `verified` without `18+` fails, `hidden` or missing is unverified. | Legacy accounts without a `usr_` id cannot be looked up at all. |
| Avatar and ranks | The avatar the player wears, resolved through VRCNext's avatar databases, then its performance ranks. | Only works when one of the databases knows the avatar; an unknown rank is unverified, never a failure. |
| Group membership | The groups the user shows publicly. | A member who hides the membership is unverified, not a failure. |
| Friendship | Your friend list. | — |
| Trust score | The profile score VRChat stopped showing, rebuilt from account age, 18+ status, a bio and groups joined. Asked for with the **Trust score at least** slider; at 0 nothing is checked and the score is left out of the report. | Badges and uploaded content are not in what VRCNext pushes, so they are left out of the total rather than counted as failures. A profile that could not be read is unverified, never a failure. |
| Recent activity | VRCNext's timeline, worded by the plugin system: `Blocked by you`, ``Met again in `Jellybean #52792` (Friends+)``, `Friend request from **X**`. **One instance is always exactly one line**, however many times they have been there and whatever mix of records VRCNext filed for those visits — a rejoin adds a `×2` rather than another line, and the first meeting shares the line of the place it happened in. A group instance names its group when VRCNext knows it, and the last row is the oldest record there is — usually the day you met — with a `...` row above it for what sits between. | A timeline read returns **ten records**, so without the `sql` permission the log and its pinned oldest row reach back only that far. With it, the pinned row is VRCNext's genuinely oldest record. |
| Activity, Moderation and Info | The profile payload VRCNext already sends, plus the moderation lists it keeps in the page. No extra lookup. | `dbEntries` is the exception: it is a count of the whole database, so it has no cheaper substitute and is simply left out without the switch. A value VRCNext does not have renders empty, so the row — and an all-empty field — is dropped rather than printing "Unknown". |
| Names they went by | The name each instance record kept at the time: from the players on the ten timeline records for free, or from every record there is with the database switch on. One indexed read either way. | Only names VRCNext itself witnessed — not a name history from VRChat, which publishes none. Without the database the depth is whatever those ten records cover. `event_players.joined_at` is empty on nearly every row, so the dates come from the player's own arrival, not that column. |
| Rejoin | VRCNext's timeline: the player's ten most recent events, each with its location. Yes when one of them is this exact instance (same world **and** instance id) from before this join. | Survives restarts and reaches back to when VRCNext was installed, but only ten events deep per player. |

Each lookup runs in parallel and degrades to "unknown" on its own timeout rather than holding up
the report.

### Pictures

Discord fetches an embed's images from its own servers, and VRCNext serves pictures it has
downloaded from its cache on this machine — a `http://localhost:…/imgcache/…` address that is
nothing anywhere else. So each picture is looked for in up to three places, cheapest first, and
the debug log says which one answered:

1. **What the lookup already handed over.** For a picture VRCNext has not cached yet, that is
   VRChat's own address, and using it costs nothing.
2. **The address VRCNext recorded** when it downloaded the file — one indexed read of VRCNext's
   own database, on this machine.
3. **One more uncached lookup**, only with **Allow making extra API requests** on. This is the
   only step that may make VRCNext ask VRChat again, which is why it is off by default: VRChat
   rate-limits, and a busy club is a lot of joins.

With the switch off, a picture none of the free answers could supply is simply left out — better
an empty thumbnail than one only your own machine can load.

## Your own joins

The signed-in account comes from `ctx.vrchat.self()`. VRChat logs an `OnPlayerJoined` line for the
local player too, and right after it one line for every player already in the instance. The plugin
ignores your own line and treats joins inside the settle window after it (or after a world change)
as "already here".

## Layout and permissions

Flat, like every plugin: `plugin.json`, `main.ts`, `src/`. The manifest declares `gamelog` (joins
and world changes), `vrchat` (the read-only data above), `notifications` (toast, Windows tray
toast), `native` (bridge targets) and `network` with `discord.com` as its only host. No VRCNext
actions and no raw events: the `vrchat` capability covers everything this plugin reads.

`sql` is **optional**, not declared: the plugin never touches the database unless you turn
**Read VRCNext's database for the full picture** on, which is when it asks. Decline and nothing
breaks — every fact behind it falls back to what VRCNext is already holding in the page, and the
reports get shorter rather than wrong. Revoke it later from the permissions panel and the switch
turns itself back off at the next change rather than pretending it still has access.

## Installing

In VRCNext, **Settings → Plugins → Install a plugin**, with the repository URL:

```
https://github.com/vrcnext-plugins/vrcnext-club-security-plugin
```

The [bridge](native-companion.md) clones it, checks the manifest, the [source policy](source-policy.md) and the [signature](signing.md), asks you to confirm on the desktop, and rebuilds.

Every release is signed by this key; check it against the one VRCNext shows when it asks whether to trust a new key, and if an update ever says the key changed, stop and ask before confirming:

```
1bc6-e13e-c44c-3bd0-f5a8-5618-8b9b-919c
```
{% endraw %}
