# Working on the documentation site

The only documentation for every VRCNext Plugins repository. GitHub Pages builds it with Jekyll
(`jekyll-theme-cayman`, kramdown GFM); preview with `bundle exec jekyll serve`.

## Rules

- Every page is Markdown at the root with `title` front matter, an `# H1` equal to it, and a
  `[← Back to index](./)` line. Link to other pages as `page.md` (Jekyll rewrites to `.html`).
- Every page is listed in a table on `index.md`.
- Tables that mirror code must match it: `source-policy.md` ↔ the bridge's `policy.rs` and
  `obfuscation.rs`; `permissions.md` and `plugin-json.md` ↔ `packages/api/src/permissions.ts`;
  `api-reference.md` ↔ `@vrcnext/plugin-api`'s exports; the event counts ↔
  `protocol/vrcnext-protocol.json`. Check the code, not an older page.
- Say what is not possible or not safe plainly; do not describe planned behaviour as shipped.
- Before committing, check every relative link and anchor resolves (a heading's anchor is its
  lower-cased text, punctuation dropped, spaces as `-`).
- Commit messages end with a `Co-Authored-By:` trailer naming the model that wrote the commit.
