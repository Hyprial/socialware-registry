# Socialware Registry

This repository is the discovery index used by `hyprial install`. It contains no
packages, runs no installation scripts, and resolves no dependencies; each
socialware repository continues to own its own installation and start commands.

## The catalog the installer actually reads

`catalog-v2.json` is the live document. The installer pins its address in code
(`src/hyprial/installers/core.py`, `_DEFAULT_REGISTRY`):

```
https://raw.githubusercontent.com/Hyprial/socialware-registry/main/catalog-v2.json
```

**That address is on the public mirror, not on this host, and that is not an
accident.** `code.hyprial.com` answers every anonymous request with a 303 to
`/user/login` — public repositories included (measured 2026-09-17). A user
installing from the open internet cannot read anything here, so the mirror is
the only address that works for them. `.forgejo/workflows/publish-mirror.yml`
publishes these documents to `github.com/Hyprial/socialware-registry` on every
push to `main`, and then verifies anonymously that the published URL really
serves a valid catalog.

`HYPRIAL_INSTALL_REGISTRY` overrides that address. It names the **catalog
document** — an `https://` URL, a `file://` URL, or a local path — not a Git
repository.

### Schema (`hyprial.catalog/v2`)

```json
{
  "schema": "hyprial.catalog/v2",
  "apps": {
    "<name>": {
      "release":  "https://…/<artifact>",
      "sha256":   "<64 hex characters>",
      "version":  "<semver>",
      "commit":   "<40-hex git commit>",
      "manifest": "<path inside the artifact>"
    }
  }
}
```

v2 is **release-only**. An entry carrying `git`/`ref` is rejected by the
installer, because a catalog that names a Git ref cannot pin what the user will
actually receive. Git remains the developer path, not the install path.

`scripts/check_catalog.py` enforces these rules before anything is published,
and again afterwards against what the mirror serves.

## Current state: zero apps, deliberately

`apps` is empty. It is empty because nothing can honestly be listed yet: a v2
entry needs an artifact that an anonymous user can download and whose sha256 can
be pinned, and no socialware has one. An empty catalog is the truthful answer —
it makes `hyprial install` report *"no apps available"* instead of failing with
`INSTALL_REGISTRY_UNAVAILABLE`, which claimed the registry was unreachable when
in fact it was simply never published.

Adding an app is therefore not a change to this file alone. It needs, first, a
published release artifact at a stable anonymous URL.

**`gui` is not one of them, and will not be.** The GUI now ships inside hyprial
itself — its source is `src/gui` in the harness-bridge monorepo (ruled by Allen,
2026-09-17), so it arrives with the product and is not separately installable.
The catalog will never need a `gui` entry.

## `catalog.json` (v1, kept for the record — do not rely on it)

`catalog.json` is the original index: it mapped a name to a Git repository and a
ref. It is dead three times over, and each one alone would be enough:

* both entries point at `code.hyprial.com`, which no anonymous user can read;
* its `git`/`ref` shape is one the current installer rejects outright;
* its `gui` entry names `dsh-h2b-talk`, which is no longer the distribution path
  at all — the GUI ships inside hyprial now.

It is kept so the history of those two entries is not lost, and it is published
to the mirror unchanged. Nothing reads it.

The `gui-tested` tag it refers to still exists, but on `dsh-h2b-talk` — measured
2026-09-17: `refs/tags/gui-tested` is present there and absent from
harness-bridge, while harness-bridge's own `dev-track` resolves, so the check
distinguishes "absent" from "unreadable". Whatever that tag still gates, it is
not an install path the catalog can address.
