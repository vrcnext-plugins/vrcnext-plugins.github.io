---
title: plugin.json
---

# `plugin.json`

[← Back to index](./)

The manifest at the root of every plugin repository. The bridge validates it when a plugin is
installed or updated, and the host validates it again when it reads the compiled plugin table
at boot — through the same parser, `parsePluginManifest` from `@vrcnext/plugin-api`, so a
manifest that passes one passes the other. Parsing never throws: a bad manifest yields a list
of problems the UI shows verbatim.

```jsonc
{
  "id": "friend-alerts",            // [a-z0-9][a-z0-9-]{1,39}; equals plugin.id in main.ts
  "name": "Friend alerts",
  "version": "1.2.0",               // semver
  "apiVersion": "^0.2.0",           // semver range against @vrcnext/plugin-api
  "description": "…",               // <= 200 characters
  "author": "…", "homepage": "…",   // optional
  "tags": ["notifications"],        // optional, <= 8
  "permissions": ["host:events", "native"],   // categories it may ever use
  "optionalPermissions": ["network"],         // asked for later via ctx.permissions.request
  "actions": ["getFriends"],        // VRCNext actions granted at enable (host:actions)
  "events": ["friendOnline"],       // host events granted at enable (host:events)
  "hosts": ["api.example.com"]      // hosts granted at enable (network); no wildcards
}
```

## Fields

| Field | Rules |
| :--- | :--- |
| `id` | Required. `[a-z0-9][a-z0-9-]{1,39}`. Must equal `id` in `main.ts`; also the clone directory name on the bridge. |
| `name` | Required. Shown in the manager and in every permission prompt. |
| `version` | Required. Plain `MAJOR.MINOR.PATCH` — no prerelease, no build metadata. Shown in the manager; updates are tracked by git commit, not by this field. |
| `apiVersion` | Required. A **range** of `@vrcnext/plugin-api` versions, checked before activation. Only four shapes are accepted: exact (`0.2.0`), caret (`^0.2.0`), tilde (`~0.2.1`) and space-separated comparator lists (`>=0.2.0 <0.4.0`). The parser is forty lines on purpose — the api package is compiled straight from source with no `node_modules` — so `||`, `x`, `*` and hyphen ranges are not ranges here. |
| `description` | Optional (empty when absent); at most 200 characters. Shown in the manager and the enable modal. |
| `author`, `homepage` | Optional strings. |
| `tags` | Optional, at most 8. Any string is accepted; the manager filters by the suggested ones (`PLUGIN_TAGS`): `accessibility`, `activity`, `audio`, `chat`, `customisation`, `developer`, `friends`, `fun`, `media`, `notifications`, `osc`, `privacy`, `ui`, `utility`. |
| `permissions` | The categories the plugin may ever use. Granted when the user enables it. May be `[]`. An undeclared category is refused outright at call time. |
| `optionalPermissions` | Categories the plugin may ask for later through `ctx.permissions.request(p)`. |
| `actions` | Exact VRCNext action names `ctx.bridge` may send without a prompt. Needs `host:actions`. |
| `events` | Exact host event names `ctx.events` may subscribe to without a prompt. Needs `host:events`. `openDeepLink` here covers `ctx.deepLinks`. |
| `hosts` | Exact hosts `ctx.http.fetch` may reach without a prompt. Bare host names, optionally with a port; no scheme, no path, no wildcards. Needs `network`. |
| `dependencies` | Optional in the host parser: ids of plugins that must be running before this one activates. **The bridge does not accept it yet** — its manifest schema has no such field and refuses unknown fields — so a manifest that uses it is refused at install. |

Lists are de-duplicated; empty or non-string entries are errors. An unknown permission name is an
error, not a warning — the vocabulary is fixed.

## The permission vocabulary

Exactly these, from `PERMISSIONS` in `@vrcnext/plugin-api`, each with the one-line description
and risk tone the consent modal shows:

| Permission | Tone | Description |
| :--- | :--- | :--- |
| `host:events` | low | Receive VRCNext events, limited to the event names listed in plugin.json. |
| `host:actions` | high | Send actions to VRCNext on your behalf, limited to the action names listed in plugin.json. Actions act on your real VRChat account. |
| `host:intercept` | high | Observe and drop every action VRCNext sends to its backend, including its own. |
| `network` | medium | Make HTTP requests, limited to the hosts listed in plugin.json. |
| `notifications` | low | Show toasts, confirmation dialogs and desktop notifications. |
| `native` | medium | Call the VRCNext Bridge: VR overlay and desktop notification targets. |
| `osc` | medium | Send and receive OSC avatar parameters through VRCNext. |
| `gamelog` | medium | Read the VRChat game log, live and its backlog. |
| `context-menu` | low | Add entries to right-click menus. |
| `routes` | low | Serve in-page HTTP routes under /plugins/&lt;id&gt;/. |
| `clipboard` | medium | Read and write the clipboard. |

`ui`, `settings`, `logger` and `disposables` need no permission: they can only affect the
plugin's own panels and its own stored values.

"Limited to the names listed" is what the declared lists buy at enable time. Names *not* listed
are still reachable inside a declared category — the user is asked on first use. See
[Permissions](permissions.md) for exactly when.

## What the bridge checks beyond the parser

At install and update the bridge also refuses a manifest with unknown fields, and refuses the
repository if it has no `plugin.json` at the root (`no_manifest`), if the id is already installed
(`already_installed`), or if the sources fail the [source policy](source-policy.md). The error
message names the problem with a stable prefix (`manifest_invalid: …`, `policy: file:line rule`)
so it can be read straight off the panel.

[← Plugin anatomy](plugin-anatomy.md) · [Permissions →](permissions.md)
