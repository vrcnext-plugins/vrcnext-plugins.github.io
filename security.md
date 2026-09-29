---
title: Security model
---

# Security model

[← Back to index](./)

## Plugins are not sandboxed

A plugin runs inside the VRCNext page with the **full authority of that page**: the user's
logged-in VRChat session, their Discord webhook URLs, their settings, and the hundreds of backend
actions VRCNext exposes. It is compiled into the same bundle as the host. There is no iframe, no
realm, no process boundary.

**Installing a plugin is equivalent to running a program from that repository.** The system
makes what a plugin does *declared* and *confirmed*; it does not contain a plugin that is
determined to misbehave, and saying otherwise would invite users to install plugins they have
not vetted.

## What the system actually does

Real, verifiable controls. Each one narrows how a plugin gets in or what it does without asking;
none of them holds against a malicious plugin the user has already enabled.

| Control | Detail |
| :--- | :--- |
| **Nothing runs that the bridge did not compile** | The page runs one static bundle. It never evaluates code at runtime, never fetches a manifest, never loads a blob. The only way for code to reach the page is a git clone through the bridge. |
| **Author signatures** | Every tree carries an Ed25519 `plugin.sig` over its files, bound to the plugin id. A plugin is pinned to the key it was installed under; an unknown key, and any later change of key, is confirmed natively and separately. See [Signing](signing.md). |
| **Native confirmation** | Install, update and uninstall are confirmed outside the page — a desktop notification with Confirm/Deny on Linux, a message box on Windows — because the page cannot be trusted to confirm code being added to itself. No channel, no answer within two minutes: refused. |
| **Transport** | `https://` only, cloned with pure-Rust git (no shell, no system `git`), depth 1, 120 s deadline. Symlinks in the clone refuse the install. |
| **Manifest validation** | [`plugin.json`](plugin-json.md) is parsed by the bridge at install and update, and again by the host at boot, through the same parser. Unknown fields and unknown permission names are errors. |
| **Source policy** | Every source file is scanned before compiling; `eval`, `window`, `fetch`, storage, sockets, dynamic import and friends refuse the install, called or merely referenced, with file, line and rule. See [Source policy](source-policy.md). |
| **Shape policy** | Before those names, source shaped so a reviewer cannot read it is refused: minified or packed files, runs of `\xNN` escapes, `_0x`-mangled identifiers, long high-entropy blobs, zero-width and bidirectional-override characters. A name-based scan is worthless on code nobody could read. |
| **Declared ceiling** | Every capability that reaches outside the plugin's own panels needs a category from `permissions`. Enabling shows each one with a risk tone; an undeclared category throws `PermissionError` and is never asked about. |
| **First-use prompts** | Inside a category, each concrete target — a host, an action name, an event, a bridge method — is confirmed the first time, with the exact payload in the details. See [Permissions](permissions.md). |
| **The only `fetch`** | `ctx.http.fetch` asks about each host, forces the plugin's abort signal, and is the only HTTP a plugin can do. Through the bridge it follows no redirects and reaches only public addresses, so an approved host cannot bounce a request onto this machine or its network. See [How `ctx.http.fetch` travels](api-reference.md#how-ctxhttpfetch-travels). |
| **Checksum-pinned compiler** | The bridge refuses to spawn `esbuild` unless its SHA-256 matches the checksum written at install; arguments are fixed and nothing from a plugin reaches them. |
| **Paired, loopback-only daemon** | The bridge listens on loopback with no override, refuses non-loopback origins, and every socket must present the pairing token in its first frame. |
| **Identity and version** | `plugin.id` in `main.ts` must equal the manifest's; `apiVersion` is enforced before activation. |
| **Reversibility** | Disabling disposes listeners, timers, UI, routes and menu contributions; a throwing listener, provider or route is logged and skipped, not fatal. |

## What it does not do

- No sandbox or iframe isolation — plugins share the page's global scope and the host's bundle.
- No protection *between* plugins. One plugin can reach another's DOM; the state store is
  namespaced per plugin, but the namespace is a convention the host follows, not a wall.
- **No review** of what a repository serves. A signature says who published a tree, not that the
  tree is safe, and nothing vouches for a fingerprint but the author — accepting one the first
  time is trust on first use. What protects you beyond that is that nothing changes on your
  machine until you press **Update** and confirm on the desktop, and that a changed signing key
  stops the update and asks.
- Signing does not help if the author's own key is stolen, or if you accept a fingerprint
  without checking it against what the author publishes.
- The source policy is a text scan, not a parser. It catches honest mistakes and makes dishonest
  ones obvious in a review; a determined author can find a spelling it does not cover.
  (Imports are checked for real: the build refuses any file outside the plugin's own directory.)
- **The DOM is not covered by any permission.** A plugin can read what the page shows, including
  what VRCNext puts in its own login form, and an element it creates can load a URL (`<img src>`,
  CSS `url()`), which is a request `ctx.http` never sees.
- The permission prompts are rendered by the host, in the page. A plugin that has already
  escaped the `ctx.*` path could, in principle, click its own prompt. That is exactly why the
  operations that *add* code — install, update, uninstall — are confirmed natively instead.
- The installer is a script piped into a shell. It verifies the sha256 of everything it
  downloads; nothing verifies the script itself. Read it first if that matters to you.

## Guidance for users

- Only install repositories you would trust with your VRChat account.
- Check the fingerprint at the "trust a new signing key" prompt against the one the author
  publishes. If an update says the key changed, stop and ask them before confirming.
- Read the enable modal. A plugin that asks for `host:actions` or `host:intercept` (both *High
  risk*) can act on your account or drop VRCNext's own traffic; a plugin that draws its own UI
  needs neither.
- Prefer **Confirm** over **Confirm & Save** for anything you would not expect the plugin to do
  every session. Saved grants are listed under **Settings → Plugins → Permissions** with a Revoke
  button.
- Confirm installs and updates only when *you* pressed the button in VRCNext moments ago. A
  desktop prompt you did not expect is a Deny.

## Guidance for plugin authors

- Declare exactly what you use. Pre-declare the `actions` and `events` you know you need so the
  user sees one modal at enable instead of a prompt a minute later. Declare your `hosts` too;
  each still prompts on first use.
- Never log tokens, auth headers or message content — `ctx.logger` output is user-visible,
  mirrored to the bridge's `plugins.log`, and gets pasted into bug reports.
- Never put secrets in settings. The state store is a plain JSON file on the user's disk and
  every plugin in the page can reach the host's state client.
- Treat anything from a route handler or a fetched API as hostile: validate type, length,
  content.
- Use `ctx.signal` on every long-lived operation so a disabled plugin stops immediately.
- Be conservative with actions that mutate the user's account. There is no undo, and the prompt
  the user sees shows your payload verbatim.

[← Publishing & updates](publishing.md) · [Limitations →](limitations.md)
