# VRCNext Plugins

Plugins for [VRCNext](https://github.com/shinyflvre/VRCNext) — **without modifying VRCNext.**

VRCNext ships no plugin API, so the host installs itself as a VRCNext *custom theme*, a folder in
the config directory that app updates leave alone. The **VRCNext Bridge**, a small loopback
daemon, clones plugin repositories, checks them and compiles them with the host into the one
bundle the page runs. Users install a plugin by pasting its repository URL into the Plugins tab.

📖 **[Documentation](https://vrcnext-plugins.github.io/)** · 🚀 **[Getting started](https://vrcnext-plugins.github.io/getting-started.html)**

```bash
# Linux / macOS — Windows and details: https://vrcnext-plugins.github.io/install.html
curl -fsSL https://raw.githubusercontent.com/vrcnext-plugins/vrcnext-plugin-system/main/install/install.sh | bash
```

## Repositories

| Repository | What it is |
| :--- | :--- |
| [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) | The host, the typed `@vrcnext/plugin-api`, and the installer. |
| [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge) | The native daemon: installs, verifies and compiles plugins; state, notifications, HTTP. Rust, loopback only. |
| [vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin) | The template: every capability, one file each. Start a plugin here. |
| [vrcnext-plugins.github.io](https://github.com/vrcnext-plugins/vrcnext-plugins.github.io) | The documentation site — the only documentation for everything here. |

## Plugins

| Plugin | What it does |
| :--- | :--- |
| [Club Security](https://vrcnext-plugins.github.io/plugin-club-security.html) | Reports who joins your instance and whether they meet your club's rules. |
| [Bio Updater](https://vrcnext-plugins.github.io/plugin-bio-updater.html) | Writes your bio, status, pronouns and links from templates, on a schedule. |
| [VRCNext Patches](https://vrcnext-plugins.github.io/plugin-patches.html) | Small fixes VRCNext does not make: media-library folders, unattended sign-in. |

## For plugin authors

One repository is one plugin: `plugin.json` and `main.ts` at the root, written against
`@vrcnext/plugin-api`. Nothing is built on your side — the bridge compiles it. Every release must
be [signed](https://vrcnext-plugins.github.io/signing.html), and the bridge refuses source that
breaks the [source policy](https://vrcnext-plugins.github.io/source-policy.html).

## A note on trust

Plugins run inside the VRCNext page with the full authority of your VRChat session. There is no
sandbox: the [permission model](https://vrcnext-plugins.github.io/permissions.html) makes each
capability declared and each use confirmed, installs are confirmed on your desktop, and every
plugin is pinned to its author's signing key — but installing one is still like running a
program from that repository. See the [security model](https://vrcnext-plugins.github.io/security.html).

Everything here is released into the public domain under the Unlicense.
