---
title: Source policy
---

# Source policy

[← Back to index](./)

A plugin is compiled into the same bundle as the host and runs with the page's authority. There
is no sandbox to put it in, so the next best thing is to refuse, *before compiling*, the handful
of constructs that let code reach past the `ctx.*` API: evaluating strings, touching `window` or
storage directly, opening its own sockets, loading code at runtime.

Before every install and update the bridge scans every `.ts`, `.tsx`, `.mts`, `.cts`, `.js`,
`.jsx`, `.mjs` and `.cjs` file in the repository (the extensions esbuild resolves, so renaming a
file buys nothing). One hit refuses the operation with `policy: file:line rule`; an update that
fails leaves the previously installed tree untouched.

## The shape rules

These run **first**, on every scanned file, because everything below is a search for names and
that only works on code a person could have read. A minified bundle, a string of
`\x65\x76\x61\x6c` escapes or a `_0x3a2b[17]` lookup table passes every rule in the next
table without hiding anything from the runtime — and the point was never the grep, it was that a
dishonest plugin is obvious in review.

| Rule | Refuses |
| :--- | :--- |
| `minified` | a line longer than **1000 characters** |
| `packed source` | a file of **20 or more non-empty lines** whose mean line is over **250 characters** |
| `escape sequences` | more than **eight** `\xNN` or `\uNNNN` escapes in a row |
| `mangled identifiers` | `_0x` followed by four hex digits — the signature `javascript-obfuscator` leaves |
| `opaque blob` | an unbroken run of **256+** characters from `[A-Za-z0-9+/=_-]` with **≥ 3.5 bits** of Shannon entropy per character |
| `invisible characters` | zero-width characters and bidirectional overrides (`U+200B–U+200F`, `U+202A–U+202E`, `U+2060–U+2064`, `U+2066–U+2069`, `U+FEFF`), which make the source read differently than it runs — the Trojan Source attack. A byte-order mark at the very start of a file is fine. |

Nothing here detects obfuscation in general; that is undecidable, and unreadable code can be
written in plain ASCII at honest line lengths. What it does is take the cheap, tool-generated
kind off the table and make the expensive kind look like what it is.

**One honest thing is refused on purpose:** a binary asset inlined as a `data:` URI trips
`opaque blob`, because a text scan cannot tell it from a payload. Ship images as files, or
draw them.

Real plugin sources are nowhere near these limits — the longest line across the published
plugins is about 210 characters — but if one of them fires on code you consider ordinary, that
is worth reporting.

## The rules

In the order the bridge reports them:

| Rule | Matches |
| :--- | :--- |
| `eval` | the word `eval`, called or not: `(0, eval)(s)` is refused too |
| `Function` | the word `Function`: `new Function(…)`, or a reference to call later |
| `globalThis` | the word `globalThis`, so `globalThis['fe' + 'tch']` is refused as well as `globalThis.x` |
| `window` | the word `window`, with no exceptions, not even `window.location.href` |
| `self` | `self.` or `self[`; a function named `self()` is fine |
| `top`, `parent`, `frames` | the name followed by `[` or by a member only the window has (`parent.postMessage`, `top.location`); `parent.appendChild(li)` and `rect.top` are fine |
| `opener` | the word `opener` |
| `defaultView` | `defaultView`, even after a dot: `el.ownerDocument.defaultView` is the window |
| `__receiveMessageCallbacks` | anywhere; it is VRCNext's own message channel |
| `Reflect.get` | `Reflect.get(` |
| `document.cookie` | `document.cookie` |
| `document[` | `document[`, so `document['cook' + 'ie']` is refused |
| `document.location`, `location.href`, `location.assign` | navigating the page away, which is also a way to send data out |
| `localStorage`, `sessionStorage`, `indexedDB` | the name, even after a dot, so `self.localStorage` is refused |
| `XMLHttpRequest`, `fetch`, `WebSocket`, `EventSource` | the word, called or referenced. `ctx.http.fetch(` and `ctx.router.fetch(` are fine, and so is `fetchStars` |
| `sendBeacon` | the name, even after a dot, so `const n = navigator; n.sendBeacon(…)` is refused |
| `Worker`, `SharedWorker`, `importScripts` | the word |
| `dynamic import` | `import(`, however it is spaced: `import (` too |
| `script tag` | `<script` anywhere |
| `innerHTML assignment`, `outerHTML assignment` | `.innerHTML =`, `.innerHTML +=`, and the same for `outerHTML` (a comparison or a read is fine) |
| `insertAdjacentHTML` | `insertAdjacentHTML` anywhere |
| `srcdoc` | the name, even after a dot |
| `createElement of a code-loading tag` | `createElement(` of `script`, `iframe`, `frame`, `object` or `embed` |
| `setTimeout with a string`, `setInterval with a string` | the timer whose first argument is a string literal |
| `constructor` | `.constructor` or `['constructor']`: `Object.getPrototypeOf(async () => {}).constructor` is a `Function` constructor without the word. A class's own `constructor(…) {` is fine |
| `import outside the plugin` | an import specifier that climbs past the plugin's root (`'../../state.json'`), an absolute path, or a `data:`/`file:` URL |
| `imports a test file` | an import of a `*.test.*` or `*.spec.*` file (see below) |
| `require` | the word `require` |
| `process` | `process.` |
| `references a tooling config` | a source file, or any `.json` file, that names one of the exempt files below, in any letter case. `package.json`'s `scripts` block is not checked. |

A **word** match needs something other than an identifier character on both sides, and no dot
in front. So `retrieval(x)` does not trip `eval`, `prefetch(` does not trip `fetch`, and a
member access like `ctx.http.fetch` is left alone. Taking a reference is refused like a call:
`const f = fetch` and `(0, fetch)(url)` both trip `fetch`. The rules marked "even after a dot"
exist because those names reach the same object through another path.

Also enforced: at most **200 source files** and **2 MiB** of source in total, no symlinks inside
the clone. `.git/` and non-source files such as `README.md` are not scanned.

**The build checks too.** A text scan can only guess what a specifier means; esbuild knows. Every
build asks it for the list of files it bundled and refuses the bundle unless each one is the
host, the generated import table, or a file inside an installed plugin's own directory — and a
plugin's files may import only each other and `@vrcnext/plugin-api`. So `state.json` (every
plugin's settings), a config file elsewhere on disk, the host's internal modules and another
plugin's files cannot reach a bundle however the import is spelled. An install or update whose
build fails is rolled back.

Both lists live in Rust modules in the bridge (`policy.rs` and `obfuscation.rs` under
`crates/vrcnext-bridge-plugins/src/`) with a unit test per rule, and this page mirrors them.

## What it is and is not

It is a **text scan**, not a parser. It is meant to catch honest mistakes and make dishonest
ones obvious in a review, not to be unbypassable, and the bridge's own documentation says so.
A plugin that wants to reach `window` badly enough can find a spelling this list does not cover.
It does **not** confine a plugin to `ctx.*`. Plain DOM work is allowed, and the DOM has ways
out that no rule here covers: an `<img>` or `<link>` whose URL a plugin sets is a request to that
URL, CSS `url()` is another, and the page's own elements — VRCNext's login form included — can be
read like any other. What the policy does is make the obvious routes (`fetch`, storage, `eval`,
the window) fail loudly, so that what is left looks deliberate in a review. Those routes are
gated by [permissions](permissions.md) the user can see; the rest is trust in the author, which
is what [signing](signing.md) is for.

Plain DOM work is allowed. `document.createElement`, `addEventListener`, `textContent`,
`IntersectionObserver` and friends are how plugin UI gets built; the policy is about reaching
*outside* the plugin's own panels, not about building them.

## Finding problems before the bridge does

The [template](https://github.com/vrcnext-plugins/vrcnext-example-plugin)'s `eslint.config.mjs` reports most of these as lint errors, so `npm run check` in
your repository catches them locally. The bridge's message names the file, line and rule, so the
rest are one-line fixes.

## What is not scanned

Three narrow exemptions, all for files that never reach the bundle: the tooling configs at the
repository root (`eslint.config.*`, `vitest.config.*`), `scripts/sign-plugin.mjs`, the
[signing tool](signing.md), which runs under Node and has to name `process` and `Buffer` to do
its job, and test files (`*.test.*`, `*.spec.*`), since a fake of `ctx.http` has to write
`fetch`. The first two are exempt by exact path.

Importing an exempt file would pull it into the bundle unscanned, which is the whole thing the
exemption must not allow. So a source file whose import specifier names one is refused (in any
letter case; naming one in a comment is fine), any `.json` file that names a tooling config or
the signer is refused (an `imports` alias in `package.json`, a `paths` entry in a tsconfig), and
the build refuses a bundle that has a test file among its inputs.

## Dependencies

The bridge runs no package manager. Anything you import must be committed into the repository,
and it is scanned like your own code and counts against the 200-file and 2 MiB limits. A library
that touches `window.` will be refused. Keep dependencies small and vendored, or do without.

[← Permissions](permissions.md) · [Settings →](settings.md)
