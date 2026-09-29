---
title: Contributing
---

# Contributing

[← Back to index](./)

For people and agents working *on* these repositories, rather than writing a plugin against them.
If you are writing a plugin, start at [Getting started](getting-started.md) instead.

Every repository carries an `AGENTS.md` at its root, and that file is the authority for it. This
page is the map and the handful of rules that hold everywhere.

## The repositories

| Repository | What it is | Gate |
| :--- | :--- | :--- |
| [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) | The host injected as a VRCNext theme, the typed `@vrcnext/plugin-api`, and the installer. | `./scripts/check.sh` |
| [vrcnext-bridge](https://github.com/vrcnext-plugins/vrcnext-bridge) | The native daemon: install pipeline, source policy, signing, services. Rust. | `./scripts/build.sh` |
| [vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin) | The template plugin, and a submodule of the plugin system. | `npm run check` |
| [vrcnext-club-security-plugin](https://github.com/vrcnext-plugins/vrcnext-club-security-plugin), [vrcnext-bio-updater-plugin](https://github.com/vrcnext-plugins/vrcnext-bio-updater-plugin), [vrcnext-patches-plugin](https://github.com/vrcnext-plugins/vrcnext-patches-plugin) | The published plugins. | `npm run check` |
| [vrcnext-plugins.github.io](https://github.com/vrcnext-plugins/vrcnext-plugins.github.io) | This site. | Liquid parse + link check |

Checkouts sit side by side, so a sibling directory is the local path to any of them.

## Rules that hold everywhere

- **The gate passes before the commit.** Nothing filtered, nothing skipped. A gate that is
  inconvenient is still the gate.
- **Documentation lives only here.** Each repository keeps a compact `README.md` and its
  `AGENTS.md`; everything else is a page on this site. Behaviour and its page change in the same
  piece of work, not in a follow-up.
- **Workflows run only by hand.** Every workflow's one live trigger is `workflow_dispatch`, with
  `push`, `pull_request`, tags and schedules commented out rather than deleted, so a fork can
  restore CI by removing the comment markers. Do not dispatch one yourself, and never add a check
  that waits on a workflow completing.
- **Commit messages end with a `Co-Authored-By:` trailer** naming the model that wrote the commit.
- **Never drive VRCNext with a synthetic mouse or keyboard**, and reload or restart it sparingly:
  both re-authenticate against the VRChat API. Inspect the page through the bridge's `remote`
  service instead — `vrcnext-eval '<async body>'`, with the bridge started with `--dev`. See
  [Running the bridge](bridge-running.md#remote-control).

## Releases are signed by hand

The bridge refuses any plugin tree whose `plugin.sig` does not match its files, and **the
signature covers every tracked file**, so any commit at all invalidates it. Each plugin
repository has a `sign.yml` workflow for this, but workflows are not automatic here, so signing
is a manual step at the end of a change:

```bash
node scripts/sign-plugin.mjs sign --key <your key file>
node scripts/sign-plugin.mjs verify
git commit plugin.sig -m "Sign <the hash it signed>"
```

A plugin repository whose last commit is not a signing commit will fail its next install or
update with `unsigned: the files do not match the signature`. [Signing plugins](signing.md) has
the detail.

## The local loop

The bridge compiles the host from `~/.vrcnext-plugins/host/packages/*/src` with its own pinned
esbuild, so deploying a host change is a source copy plus one rebuild:

```bash
./scripts/deploy-to-bridge.sh      # gate, copy sources without tests, POST /v1/plugins/build
vrcnext-eval 'location.reload()'   # the owner's step, batched once at the end
```

A *plugin* change goes the long way round, because the bridge only knows how to clone over
HTTPS: commit, sign, push, **Update** in the Plugins tab, reload. That is slower than a hot
reload, and it is the price of never evaluating code the bridge has not checked.

## House style

The two things most often got wrong:

- **Tests live beside the code they test, never inline** — `<module>/tests.rs` in the bridge
  (enforced by `source_limits.rs`, which also refuses a source file over 1000 lines),
  `*.test.ts` next to the module in TypeScript. Tests are stripped before deployment and never
  reach the bundle.
- **Suppressions are scoped and explained.** `#[allow(..., reason = "...")]` over a blanket
  attribute; the same for ESLint. `unsafe_code` is forbidden outside the Windows crate.

Two invariants are worth naming because breaking them is a security bug rather than a style one:
`ctx.native` reaches only the bridge services in `PLUGIN_BRIDGE_SERVICES`, and the permission
vocabulary is mirrored in both the bridge's `manifest.rs` and the API's `permissions.ts` — change
one without the other and installs fail with `manifest_invalid`.

## Working on this site

GitHub Pages builds it with Jekyll (`jekyll-theme-cayman`, kramdown GFM); preview with
`bundle exec jekyll serve`. Every page is Markdown at the root with `title` front matter, an `#`
heading equal to it, and a `[← Back to index](./)` line, and links to its siblings as `page.md` —
Jekyll rewrites the extension.

Two things bite:

- **Liquid runs on every page**, so a brace-brace or a brace-percent in an example — a plugin's
  template syntax, most often — breaks the build or renders the page blank. Wrap those blocks in
  Liquid's own `raw` / `endraw` tags, as [Club Security](plugin-club-security.md) does for its
  whole page. Check the lot with:

  ```bash
  ruby -rliquid -e 'Dir["*.md"].each { |f| Liquid::Template.parse(File.read(f)) }'
  ```
- **A new page goes in three places**: the file, the table on `index.md`, and `_data/nav.yml`,
  which is what the navigation bar is built from.

[← Back to index](./)
