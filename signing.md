---
title: Signing plugins
---

# Signing plugins

[← Back to index](./)

Every plugin the bridge installs or updates has to carry an Ed25519 signature made by its
author, and it has to be the **same key** the plugin was installed under. An unsigned repository,
a signature that does not match the files, or a signature by a different key stops the install
before anything is written to disk.

This is the one control that survives a repository changing hands. A URL is not an identity: an
account takeover rewrites every branch at once, and the clone still looks exactly like the plugin
you installed last month. A key the attacker does not hold does not.

## What the bridge checks

1. `plugin.sig` exists at the repository root and is version 1, `ed25519`.
2. Its `id` matches `plugin.json`'s, so a valid signature cannot be lifted onto another plugin.
3. Its `digest` matches the files actually in the clone — everything except `.git/` and
   `plugin.sig` itself, hashed by path and content in a fixed order.
4. The signature verifies against the public key the file carries.
5. **The key is one you have accepted.** A key this machine has not seen is a second desktop
   confirmation showing the fingerprint. An update signed by a key other than the one the plugin
   is pinned to is a third, separate confirmation, every time — being a trusted author of *one*
   plugin never grants another.

Steps 1–4 are cryptography. Step 5 is you: the fingerprint means nothing until you compare it
with what the author publishes. Settings → Plugins → **Signing keys** lists every key you have
accepted, what it has signed, and lets you forget one (which uninstalls nothing; it only means
the next thing that key signs is confirmed again).

## Making a key

Once, on a machine you control, in the plugin repository:

```bash
node scripts/sign-plugin.mjs keygen
```

That writes the private key to `vrcnext-signing-key.txt` (mode `0600`; `--out FILE` picks another
path) and prints the public key
and its fingerprint. There is no recovery: lose the file and your users confirm a key change on
their next update.

Keep the private key out of the repository. Publish the **fingerprint** in your README, so users
have something to compare against. `node scripts/sign-plugin.mjs fingerprint [--key FILE]` prints
it again at any time:

```
Signed by  4c82-4d21-20fc-0d00-1154-ef7a-156c-faea
```

## Signing a release

The signature covers the tracked tree, so commit first, then sign:

```bash
node scripts/sign-plugin.mjs sign --key ~/keys/vrcnext-signing-key.txt
node scripts/sign-plugin.mjs verify
git commit -am "Sign"
```

The signer refuses a dirty working tree on purpose — the bytes on your disk would not be the
bytes anyone else receives.

### From GitHub Actions

Put the 64 hex characters from `keygen` in a repository secret named `VRCNEXT_SIGNING_KEY`
(Settings → Secrets and variables → Actions), and copy `scripts/plugin-sign-workflow.yml` from
the plugin system into `.github/workflows/sign.yml`. Every push to `main` signs the
tree and commits `plugin.sig` back.

The secret never leaves the runner; only the signature is pushed. The tradeoff is a short window
between your push and the signing job finishing, during which the branch is unsigned and the
bridge refuses to install it. Signing locally and committing `plugin.sig` alongside the code has
no such window.

## The format

`plugin.sig`, at the repository root:

```json
{
  "version": 1,
  "algorithm": "ed25519",
  "id": "club-security",
  "publicKey": "<64 hex>",
  "digest": "<64 hex>",
  "signature": "<128 hex>",
  "signedAt": 1759000000
}
```

`digest` is `sha256` over `"vrcnext-plugin-tree-v1\n"` followed by, for every file in path
order, its `/`-separated relative path, a NUL, its length as eight little-endian bytes, and its
contents. Symlinks and submodules are skipped, as they are in the clone. The signature covers
`"vrcnext-plugin-signature-v1\n" + id + "\n" + digest + "\n"`, which is what binds it to one
plugin. `signedAt` is advisory — the signer chose it, and nothing depends on it.

The fingerprint is the first 16 bytes of the public key hex string's `sha256`, in groups of
four. It exists to be compared by eye, which is why it is grouped rather than run together.

## What signing does not give you

- **It is not a review.** A signature says who published the tree, not that the tree is safe. A
  signed plugin still runs with the full authority of the page; see the
  [security model](security.md).
- **It is not a registry.** Nothing vouches for a fingerprint but the author. Trusting one the
  first time is trust on first use.
- **It does not protect a key you lost.** Whoever holds the private key can publish as you.
  Treat it like an account password, keep it out of the repository, and rotate it (your users
  will be asked to confirm the change) if you suspect otherwise.
