---
title: Publishing & updates
---

# Publishing & updates

[← Back to index](./)

There is no registry and no repository manifest. A plugin is published the moment its repository
is reachable over `https://` with a valid [`plugin.json`](plugin-json.md) and `main.ts` at the
root. Users install it by pasting the repository URL into the Plugins tab; the
[bridge](native-companion.md) clones the default branch and compiles it.

## What to push

```
plugin.json        the manifest
main.ts            default-exports definePlugin({...})
src/**             optional, imported from main.ts
README.md          optional
```

Nothing is built on your side and there is no `dist/` to commit. The bridge imports `main.ts` as
TypeScript straight from the clone, so what users run is what is in your default branch. Any
dependency must be committed into the repository — the bridge runs no package manager — and it
is scanned by the [source policy](source-policy.md) like your own code.

Run the template's `npm run check` before pushing. Its ESLint config flags most policy rules
locally, and a repository that fails the policy or the manifest schema is refused at install
with the file, line and rule, which is a poor first impression.

## Supported repository URLs

Any `https://` git remote the bridge can clone anonymously: GitHub, a self-hosted Gitea or
Forgejo, GitLab. No shorthand, no `/tree/<branch>` suffix — the bridge clones the remote's
default branch, depth 1. To ship from a specific branch, make it the default branch of that
repository.

## How updates reach users

Nothing is automatic. The manager checks for updates when the user opens it or presses the
button: the bridge fetches each clone's origin and reports how many commits it is behind, with
a changelog of commit summaries (up to 50; a clone is shallow, so a count of 50 means "at
least"). The user presses **Update** (or **Update all**), confirms **on the desktop** — one
prompt per plugin — and the bridge clones afresh, re-validates the manifest and the policy,
swaps the new tree in only if it passes, rebuilds, and the page offers **Reload**. There is
never a merge, and a failed validation leaves the old tree exactly as it was.

Consequences:

- **Updates are tracked by commit, not by `version`.** Bump `version` anyway — it is what the
  user sees in the manager — but forgetting to does not hide a release.
- **Every push to the default branch is a release.** Users see it as "N commits behind" with
  your commit summaries as the changelog, so write summaries a user can read. Develop on a
  branch and merge when ready.
- **Settings and saved permission grants survive an update.** They live in the bridge's state
  store under the plugin's id, not in the clone.
- **A new permission or target needs the user's consent again.** New entries in `permissions`
  show in the enable modal; a new host, action or event is confirmed on first use.

## `apiVersion`

`apiVersion` is checked before activation. If the host's `@vrcnext/plugin-api` version does not
satisfy your range, the plugin is refused with a message naming both versions. Only exact
versions, `^`, `~` and comparator lists (`>=0.2.0 <0.4.0`) are accepted — see
[plugin.json](plugin-json.md). Widen the range only when you have actually tested against the
newer API.

## Updating the host itself

The host is compiled into the bundle from sources the installer placed under
`~/.vrcnext-plugins/host/`. Re-running the installer is an upgrade: it replaces the bridge, the
pinned `esbuild` and the host sources, keeps your plugins, state and token, and rebuilds. The
page does not check for host releases on its own.

[← Using TSX](tsx.md) · [Security model →](security.md)
