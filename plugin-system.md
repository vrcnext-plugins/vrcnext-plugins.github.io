---
title: Plugin system internals
---

# Plugin system internals

[← Back to index](./)

How [vrcnext-plugin-system](https://github.com/vrcnext-plugins/vrcnext-plugin-system) — the host that runs inside VRCNext and the typed `@vrcnext/plugin-api` — is built, laid out and worked on. Plugin authors do not need this page; start at [Getting started](getting-started.md).

## How the build works

```
~/.vrcnext-plugins/                     Windows: %LOCALAPPDATA%\vrcnext-plugins\
  bin/vrcnext-bridge  bin/esbuild  bin/esbuild.sha256
  host/packages/{api,host}/src/         host sources, from a release tarball
  plugins/<id>/                         one git clone per plugin
  build/static-plugins.ts               generated import table
  state.json  token  bridge.log
```

After every install, update or uninstall the bridge verifies `esbuild`'s checksum, writes
`build/static-plugins.ts` —

```ts
import p0 from '../plugins/friend-alerts/main.ts';
import m0 from '../plugins/friend-alerts/plugin.json';
export const COMPILED_PLUGINS = [{ manifest: m0, plugin: p0 }] as const;
```

— and runs `esbuild host/packages/host/src/index.ts --bundle --format=iife --target=es2022
--platform=browser --minify --sourcemap=linked --alias:@vrcnext/plugin-api=… --alias:@vrcnext/static-plugins=…`
into the theme folder, then pushes `build` over the socket. The page shows **Rebuilt — reload to
apply** with a Reload button; it never reloads on its own. The repository's own
`scripts/build.sh` runs the identical flag list with the alias pointed at
`packages/host/static-plugins.dev.ts`, which lists the
example plugin, so `npm run check` bundles a host that runs a plugin.

At boot the host connects to the bridge (`ws://127.0.0.1:42081/v1/ws`, endpoint and token from
`localStorage`), sends `hello`, and on `welcome` reads the host namespace of the state store —
enabled flags and saved grants — then activates the enabled plugins from `COMPILED_PLUGINS` in
dependency order. Until it is connected the Plugins section shows only the Bridge card: not
detected, running, unpaired (with the token field), or connected.

## Where the host lives

The host's own pages are two sections in VRCNext's **Settings** tab, after a divider below
VRCNext's own:

| Section | What it is |
| :--- | :--- |
| **Plugin System** | Bridge card; status, platform support matrix, diagnostics, about; the live plugin + host + bridge log with level and scope filters, copy, clear and download. |
| **Plugins** | Install by URL with progress; enable with consent; updates with a commits-behind badge and changelog; uninstall; saved permissions. Every plugin's settings card follows. |

The **Plugins** group in the sidebar (mirrored in the top menu bar) holds shortcuts to those two
sections and nothing else: sidebar tabs of their own are for plugins.

## Repository layout

| Path | Contents |
| :--- | :--- |
| `packages/api` | `@vrcnext/plugin-api` — the typed contract plus pure logic (manifest parser, permission vocabulary). No DOM. |
| `packages/host` | The runtime injected into VRCNext. `permissions/` is the prompt machinery, `plugins/` the manager and gated context, `state/` the bridge state client. |
| `packages/host/static-plugins.dev.ts` | The plugin table for the repo's own build. |
| `protocol/` | `vrcnext-protocol.json`, generated from VRCNext's source. |
| `examples/example-plugin` | A submodule of [vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin): the template plugin, checked here against the live API and bundled into the dev host. Clone with `--recurse-submodules`. |
| [`vrcnext-club-security-plugin`](https://github.com/vrcnext-plugins/vrcnext-club-security-plugin) | A complete plugin in its own repository: game log, VRChat data, presets, native and Discord notifications, tests. |
| `install/` | The one-line installers; see [Installer](install.md). |
| `scripts/` | `build.sh`, `check.sh`, `deploy-to-bridge.sh`, `install-into-vrcnext.sh` (copies the dev bundle into the theme folder), `sign-plugin.mjs`, and the protocol tools `gen-vrcnext-protocol.mjs` / `check-vrcnext-protocol.mjs` (see [Checking against VRCNext](events-and-bridge.md#checking-against-vrcnext)). |

All documentation lives on this site. The bridge is documented in [Bridge reference](bridge.md).

## Development

```bash
git submodule update --init      # examples/example-plugin; or clone with --recurse-submodules
npm ci
npm run check      # typecheck → tooling typecheck → lint → tests → VRCNext protocol → version agreement → build
```

`examples/example-plugin` is a submodule of the standalone
[vrcnext-example-plugin](https://github.com/vrcnext-plugins/vrcnext-example-plugin) repository,
so there is one copy of it rather than two that drift. The gate type-checks it against the live
`packages/api` through `tsconfig.example.json` — the workspace's own
view of those files, kept here so the submodule stays a standalone repository. Changes to the
plugin are committed in the submodule and pushed there; this repository records which commit of
it the gate ran against.

Strict by policy: `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`,
`typescript-eslint` `strictTypeChecked`, no `any`, no non-null assertions, no `enum`, no lint
suppressions. Size limits are machine-enforced: 100 lines per function, 1000 per file, 4
parameters, depth 3 (ESLint), with a 600-line soft warning from `size-limits.test.ts`.

The permission broker, the state client, the settings store, the compiled-table reader and the
bridge socket are unit-tested without a DOM. The modals, the manager panel and the boot sequence
are not, and neither has been exercised inside a running VRCNext against a real bridge yet.
