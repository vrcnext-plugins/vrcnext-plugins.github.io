---
title: Using plugins
---

# Using plugins

[← Back to index](./)

Everything on this page happens in VRCNext's **Plugins** tab, under Settings. It assumes the
plugin system is already [installed](install.md) and paired; if the tab shows only a Bridge card,
finish that first.

## Finding plugins

There is **no plugin store, registry or search**. A plugin is a GitHub repository, and you install
it by pasting its URL. That is deliberate: nothing can appear in your VRCNext that you did not
paste in yourself, and there is no index that can be gamed into recommending something.

So plugins reach you the way a repository does — from this site, from whoever wrote it, from a
friend. The ones published here:

| Plugin | What it does |
| :--- | :--- |
| [Club Security](plugin-club-security.md) | Reports who joins your instance and whether they meet your club's rules. |
| [Bio Updater](plugin-bio-updater.md) | Writes your bio, status, pronouns and links from templates. |
| [VRCNext Patches](plugin-patches.md) | Small fixes VRCNext does not make: media-library folders, unattended sign-in. |
| [The template plugin](plugin-example.md) | Every capability, one per file — mostly for people writing one. |

Installing a plugin is equivalent to running a program from that repository: it runs inside the
VRCNext page with your VRChat session's authority. The checks described below are real, and none
of them make an untrustworthy author safe. See the [security model](security.md).

## Installing one

1. Paste the repository's `https://` URL into **Install a plugin** and press **Install**.
2. **Confirm on your desktop.** A notification with *Confirm* and *Deny* appears on Linux, a
   message box on Windows. The page cannot answer this for you — that is the whole point of it.
   Dismissing it, or leaving it for two minutes, is a refusal.
3. The bridge clones the repository, checks the manifest, scans the sources against the
   [source policy](source-policy.md), verifies the [signature](signing.md), and rebuilds the
   bundle. Each step appears in the panel as it happens.
4. A toast says **Rebuilt — reload to apply**. Press **Reload**. The page never reloads itself.
5. Switch the plugin on with the toggle on its row. A modal lists every
   [permission](permissions.md) it declares, each with a description and a risk badge, plus the
   exact hosts, actions and events it asks for in advance. **Enable** grants those; **Cancel**
   leaves it off.

After that, the first time the plugin reaches for something inside a declared category that it
did not pre-declare — another host, another action — you are asked once more, with four answers:
**Confirm** (until VRCNext restarts), **Confirm & Save** (remembered), **Deny** (refused for this
session), **Uninstall**.

## Living with them

Each row in **Installed plugins** expands. Inside:

| | |
| :--- | :--- |
| **Toggle** | On the row itself. Switching off stops the plugin without removing it or losing its settings. |
| **Settings** | The plugin's own settings card, when it has one. |
| **Source** | The repository it came from, with **Open**. |
| **Signed by** | The key fingerprint it is pinned to. Compare it against the one the author publishes. |
| **Declared** | Every permission category it asked for, as chips. |
| **Allowed so far** | The individual things you have saved a yes for. Click a chip to take that one back, or **Forget all**. |
| **Uninstall** | Removes the plugin. Rebuild and reload to finish. |

A row badged **Disabled** is installed and off. **Reload to load** means the bundle on disk is
newer than the page — press Reload. **Removed — reload** is the same the other way around.

## Updates

A plugin with a newer commit on its default branch shows an **Update available** row with how far
behind it is, and an **Update** button; **Update all** does every one of them. Each update is
confirmed on the desktop separately, then the bundle rebuilds and you reload.

Two things an update cannot do:

- **Go backwards.** An update whose version is lower, or whose signature is older than the one
  you have installed, is refused. A repository that is rolled back cannot roll you back with it.
- **Quietly change author.** Each plugin is pinned to the key it was installed under. A tree
  signed by a *different* key is a separate confirmation every time, which names the change. If
  you ever see that and were not expecting it, stop and ask the author before confirming.

## When something is wrong

**Plugins → Logs** is the first place to look: it shows plugin, host and bridge output together,
filtered by level and scope, and it is what a plugin author will ask you for. See
[Logging](logging.md).

| Symptom | What it usually is |
| :--- | :--- |
| The tab shows only the Bridge card | The bridge is not running or not paired. Press **Re-check**; if it says *Running, not paired*, paste the token again. |
| *Running, not paired* after an update to VRCNext | VRCNext picked a new port, which is a new origin, so the saved token was left behind. Paste it again (`vrcnext-bridge --print-token`), or pin the port — see [Installer](install.md). |
| Install refused, naming a file | The [source policy](source-policy.md) rejected something in the repository. That is the author's to fix; the page names the rule. |
| Install refused as unsigned | The repository's `plugin.sig` does not match its files. Also the author's to fix. |
| A plugin is installed but does nothing | Check the toggle, then the Logs panel. A plugin that threw while starting says so there. |

Removing the whole system, rather than one plugin, is under [Installer](install.md#uninstall).

[← Back to index](./) · [Permissions →](permissions.md)
