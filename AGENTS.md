# Working on the documentation site

The only documentation for every VRCNext Plugins repository. GitHub Pages builds it with Jekyll
(`jekyll-theme-cayman`, kramdown GFM); preview with `bundle exec jekyll serve`.

## Rules

- Every page is Markdown at the root with `title` front matter, an `# H1` equal to it, and a
  `[← Back to index](./)` line. Link to other pages as `page.md` (Jekyll rewrites to `.html`).
- Every page is listed in a table on `index.md`.
- Jekyll runs every page through Liquid, so `{{` or `{%` anywhere (template examples) breaks the
  build or renders blank: wrap them in `{% raw %}` … `{% endraw %}`. Check with
  `ruby -rliquid -e 'Dir["*.md"].each { |f| Liquid::Template.parse(File.read(f)) }'`.
- Tables that mirror code must match it: `source-policy.md` ↔ the bridge's `policy.rs` and
  `obfuscation.rs`; `permissions.md` and `plugin-json.md` ↔ `packages/api/src/permissions.ts`;
  `api-reference.md` ↔ `@vrcnext/plugin-api`'s exports; the event counts ↔
  `protocol/vrcnext-protocol.json`. Check the code, not an older page.
- Say what is not possible or not safe plainly; do not describe planned behaviour as shipped.
- Before committing, check every relative link and anchor resolves (a heading's anchor is its
  lower-cased text, punctuation dropped, spaces as `-`).
- Commit messages end with a `Co-Authored-By:` trailer naming the model that wrote the commit.
- **Workflows run only by hand.** Every workflow's only trigger is `workflow_dispatch`; automatic
  triggers (`push`, `pull_request`, tags, schedules) stay commented out. Never run a workflow
  yourself — the owner dispatches them.

## Other repositories

Each has its own `AGENTS.md`; read the one for any repository you change. Checkouts sit side by
side, so the local path is a sibling directory.

| Repository | Local | What it is |
| :--- | :--- | :--- |
| [`vrcnext-plugin-system`](https://github.com/vrcnext-plugins/vrcnext-plugin-system/blob/main/AGENTS.md) | `../vrcnext-plugin-system/AGENTS.md` | host, `@vrcnext/plugin-api`, installer, VRCNext protocol tools |
| [`vrcnext-bridge`](https://github.com/vrcnext-plugins/vrcnext-bridge/blob/main/AGENTS.md) | `../vrcnext-bridge/AGENTS.md` | the native daemon: install pipeline, source policy, signing, services |
| [`vrcnext-example-plugin`](https://github.com/vrcnext-plugins/vrcnext-example-plugin/blob/main/AGENTS.md) | `../vrcnext-example-plugin/AGENTS.md` | the template plugin; a submodule of the plugin system |
| [`vrcnext-club-security-plugin`](https://github.com/vrcnext-plugins/vrcnext-club-security-plugin/blob/main/AGENTS.md) | `../vrcnext-club-security-plugin/AGENTS.md` | Club Security plugin |
| [`vrcnext-bio-updater-plugin`](https://github.com/vrcnext-plugins/vrcnext-bio-updater-plugin/blob/main/AGENTS.md) | `../vrcnext-bio-updater-plugin/AGENTS.md` | Bio Updater plugin |
| [`vrcnext-patches-plugin`](https://github.com/vrcnext-plugins/vrcnext-patches-plugin/blob/main/AGENTS.md) | `../vrcnext-patches-plugin/AGENTS.md` | VRCNext Patches plugin |
