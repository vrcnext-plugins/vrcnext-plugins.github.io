---
title: Permissions
---

# Permissions

[← Back to index](./)

Two layers, both visible to the user. The first is a ceiling declared in `plugin.json`; the
second is a prompt the first time the plugin touches something concrete inside it. Neither is a
sandbox — a plugin runs with the authority of the page — but together they make what a plugin
does *declared* and *confirmed* rather than silent.

## Layer one: the declared ceiling

`permissions` in [`plugin.json`](plugin-json.md) lists the categories a plugin may ever use.
Enabling the plugin opens a modal that says, in as many words, that it runs with the authority of
this page and there is no sandbox, then lists each category with its description and a
*Low / Medium / High risk* badge, plus the exact `hosts`, `actions` and `events` it pre-declares.
**Enable** grants those; **Cancel** leaves the plugin disabled.

A category that is not declared is refused outright: the call throws `PermissionError`, nothing
is asked, and the refusal is logged. That is deliberate — an undeclared capability is a bug in
the plugin, and asking the user about it would turn every bug into a consent dialog.

`optionalPermissions` are categories the plugin may ask for later, at a moment that makes sense
to the user (a "Fetch stars" button rather than activation):

```ts
if (!(await ctx.permissions.request('network'))) return;   // opens the prompt for that category
const response = await ctx.http.fetch('https://api.github.com/…');
```

`request` resolves `true` at once when the category is already granted, and rejects when the
permission is not in `optionalPermissions` — a required one is granted at enable or the plugin
does not run, and an undeclared one can never be requested. `ctx.permissions.has(p)` is the
cheap synchronous check for rendering state.

## Layer two: first use

Inside a declared category, each *concrete* target is confirmed the first time the plugin
touches it. Targets pre-declared in `plugin.json` (`hosts`, `actions`, `events`) were granted at
enable, so they never prompt. Everything else does, one modal at a time; identical concurrent
requests share one prompt. The wording below is the user-facing contract, taken from the host.

| Category | Asked | Title |
| :--- | :--- | :--- |
| `network` | per host | *Plugin {name} ({id}) wants to request data from {host}* for GET/HEAD, *… wants to send data to {host}* for anything else. Details: method, URL, headers, body. |
| `host:actions` | per action name | *… wants to call VRCNext action {action}*. Details: the payload. |
| `host:events` | per event name | *… wants to listen to {event}*; `onAny` asks *… wants to listen to every VRCNext event*. |
| `host:intercept` | once per plugin | *… wants to observe and drop actions VRCNext sends to its backend* |
| `native` | per `service/method` | *… wants to call the bridge: {service}/{method}*. Details: the parameters. |
| `osc` | once per plugin | *… wants to send and receive OSC through VRCNext* |
| `gamelog` | once per plugin | *… wants to read the VRChat game log* |
| `clipboard` | read and write separately | *… wants to read the clipboard* / *… wants to write to the clipboard* |
| `notifications`, `context-menu`, `routes` | never | the declared category suffices |

Details are collapsed by default and truncated at 4 KiB with a note saying how much was cut.

### The four answers

| Button | Effect |
| :--- | :--- |
| **Confirm** | Allowed until VRCNext restarts. |
| **Confirm & Save** | Remembered through the bridge's state store; survives reloads. |
| **Deny** | The call rejects with a `PermissionError` naming the category and target, and the same target is not asked again this session. |
| **Uninstall** | Removes the plugin (confirmed on the desktop like any uninstall); the pending call rejects. |

The steady state after an answer is one map lookup per call, so gating costs nothing on the hot
path.

### How it feels from inside the plugin

Async methods await the answer. Subscriptions and fire-and-forget sends return at once and take
effect once the user answers, so `activate` never blocks on a modal; an unsubscribe before the
answer simply cancels the pending subscription. A denied call throws `PermissionError` — a plain
`Error` subclass with `permission` and `target` fields — so you can catch it and fall back:

```ts
try {
  await ctx.clipboard.writeText(id);
} catch (error) {
  if (error instanceof PermissionError) ctx.ui.toast({ message: 'Clipboard access was denied.', ok: false });
  else throw error;
}
```

## Revoking

**Manage Plugins → Permissions** on each plugin's card lists every saved grant with a **Revoke**
button and a **Forget all**. A revoked grant is simply asked about again next time; nothing
restarts.

## Installs are confirmed outside the page

Installing, updating or uninstalling a plugin puts code into the page, and the page cannot be the
thing that confirms that — any script already running there, a plugin included, could click its
own dialog. So the bridge asks through something the page cannot reach: on Linux a desktop
notification with **Confirm** and **Deny** buttons, on Windows a topmost Yes/No message box. No
answer within two minutes is a refusal. See [VRCNext Bridge](native-companion.md).

## Writing a plugin that prompts well

- Pre-declare the hosts, actions and events you know you need. One consent modal at enable is
  better than five prompts in the first minute.
- Put optional capabilities in `optionalPermissions` and request them from a button, not from
  `activate`.
- Do not retry a denied call in a loop; the answer is cached for the session and the loop just
  logs refusals.
- Ask for what you use. A plugin that declares `host:actions` "just in case" earns a *High risk*
  badge it does not need.

[← plugin.json](plugin-json.md) · [Source policy →](source-policy.md)
