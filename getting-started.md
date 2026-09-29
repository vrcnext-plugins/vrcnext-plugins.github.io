---
title: Getting started
---

# Getting started

[← Back to index](./)

## 1. Install the plugin system

One command installs everything: the VRCNext Bridge daemon, a pinned `esbuild`, the plugin host
sources, autostart, the first bundle build, and the pairing token.

**Linux / macOS**

```bash
curl -fsSL https://raw.githubusercontent.com/vrcnext-plugins/vrcnext-plugin-system/main/install/install.sh | bash
```

**Windows** (PowerShell 5.1 or newer)

```powershell
iwr -useb https://raw.githubusercontent.com/vrcnext-plugins/vrcnext-plugin-system/main/install/install.ps1 | iex
```

> Piping a script into a shell runs code from the internet with your user's rights. If you would
> rather not do that blind, download the script first, read it, then run it. Both scripts verify
> the sha256 of everything they download; nothing verifies the script itself.

Requirements: `curl` and `tar` on Linux/macOS; Windows 10 1803+ for the built-in `tar.exe`. No
Node, no git, no Rust. Re-running the installer is an upgrade: binaries and host sources are
replaced, your plugins, state and token are kept, and the bundle is rebuilt. Flags (`--pin-port`,
`--version`, `--dry-run`), the on-disk layout and uninstall steps are in the
[installer](install.md) page.

The installer ends by printing a **pairing token** in a box, and three steps.

## 2. Pair VRCNext with the bridge

1. In VRCNext, enable the theme under **Settings → Design → Themes** (the installer does this for
   you if VRCNext was closed while it ran).
2. Open the **Plugins** tab. Until the bridge is connected it shows only the Bridge card, in one
   of four states: *Not detected*, *Running, connecting…*, *Running, not paired*, *Connected*.
3. Paste the token into the card and press **Pair**.

The token is the page's proof that it is allowed to talk to the daemon; an arbitrary web page
open in a browser on the same machine does not have it. It is stored in the page's
`localStorage`, which is why the installer offers `--pin-port`: VRCNext picks a new random port
when its saved one is taken, and a new port is a new origin, so you would have to paste it
again. To see the token later: `vrcnext-bridge --print-token`.

## 3. Start a plugin from the template

Press **Use this template** on
[vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin), or clone
it. It is a plugin that already works — every capability the host provides, one capability per
file — with a strict `tsconfig.json`, an ESLint config mirroring the host's rules, and the
signing tool you will need to publish it.

```
plugin.json        the manifest
plugin.sig         the signature over everything else
main.ts            default-exports definePlugin({...})
src/sections/**    one capability each; delete the ones you do not need
README.md          optional
```

Starting from it means **deleting**, not assembling: remove a section's line from `main.ts`, its
file, and whatever it needed from `permissions`, `events` and `hosts`. Declaring less is the
point — every category is shown to the user with a risk tone when they enable your plugin.

Rename the `id` in both `plugin.json` and `main.ts` — they must agree. Then:

```bash
npm install --save-dev typescript eslint @eslint/js typescript-eslint
npm install @vrcnext/plugin-api      # or a file: link to a checkout of packages/api
npm run check
```

`check` typechecks and lints. The ESLint config flags most of the [source policy](source-policy.md)
before the bridge does, so a plugin that passes `check` is unlikely to be refused at install.

`main.ts`:

```ts
import { definePlugin, type PluginId } from '@vrcnext/plugin-api';

export default definePlugin({
  id: 'my-plugin' as PluginId,
  activate(ctx) {
    ctx.logger.info('Hello from my-plugin.');
    ctx.ui.toast({ message: 'my-plugin is running.' });
  },
});
```

Nothing is bundled on your side. The bridge compiles the repository together with the host, so
`main.ts` is imported as TypeScript straight from the clone. There is no `dist/` to commit.

## 4. Install it

Push to the default branch, then in VRCNext open **Plugins**, paste the repository's `https://`
URL into **Install a plugin** and press **Install**. What happens next:

1. The bridge asks you to **confirm on your desktop** — a notification with Confirm/Deny on
   Linux, a message box on Windows. The page cannot answer this for you; that is the point.
2. It clones the repository, validates `plugin.json`, scans the sources against the policy, and
   rebuilds the bundle. Each step shows in the panel as it happens.
3. A toast says **Rebuilt — reload to apply**. Press **Reload**. The page never reloads on its
   own.
4. Enable the plugin. A modal lists every permission it declares, with a description and a risk
   badge, plus the exact hosts, actions and events it pre-declares. **Enable** grants those;
   **Cancel** leaves it disabled.

Anything the plugin then touches for the first time inside a declared category — a new host, an
action name, an event — is confirmed by one more prompt. See [Permissions](permissions.md).

## Development loop

The bridge only knows how to clone over HTTPS, so iterating means: commit, push, **Update** in
the Plugins tab (confirmed on the desktop again), **Reload**. That is slower than a hot reload,
and it is the price of never evaluating code the bridge has not checked.

To see what a plugin is doing, use **Plugins → Logs** rather than devtools: it shows plugin, host
and bridge output with level and scope filters. Devtools do work if you need them:

```bash
WEBKIT_INSPECTOR_SERVER=127.0.0.1:2999 /opt/vrcnext/VRCNext
```

VRCNext leaves WebKit developer extras enabled, so `Ctrl+Shift+I` may also work. Its frontend
calls `preventDefault()` on `contextmenu`, so right-click → Inspect is suppressed.

[← Back to index](./) · [Plugin anatomy →](plugin-anatomy.md)
