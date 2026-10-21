# oriel-plugins

The plugin registry for [Oriel](https://github.com/liamg/oriel): which plugins exist, where their code lives, and
which versions were reviewed. Oriel reads only the signed index this repo publishes; nothing here is code.

## Layout

```
plugins/
  <owner>/<name>/plugin.toml
```

The path is the plugin's id. `<owner>` must be the GitHub owner of the plugin's repository, so only acme can publish
`acme/*`.

### A reviewed plugin

Every version is pinned to a commit and reviewed in a pull request here.

```toml
repo        = "github.com/acme/oriel-jira"
description = "Jira tickets in Oriel"

[[versions]]
version  = "0.4.2"
commit   = "9c1e07a…"                     # the full 40-character hash
reviewed = "https://github.com/liamg/oriel-plugins/pull/123"
archives = { "darwin/arm64" = "sha256:…", "linux/amd64" = "sha256:…" }
built    = "ci"                          # ci: rebuilt from the commit and matched; author: not reproducible

[[versions]]
version = "0.4.1"
yanked  = "crashes on Linux"            # stays installed, but isn't offered for new installs
```

### A trusted plugin

Oriel follows the author's semver tags, after `trusted_hold` (24h by default). Trust is a decision about the author,
made in a reviewed PR.

```toml
repo    = "github.com/liamg/oriel-surround"
trusted = true
since   = "2026-10-13"

[revoked]
"0.3.0" = "exfiltrated tokens: see #140"   # disabled everywhere at the next index fetch
```

Revoking any version of a trusted plugin also removes `trusted`, in the same PR.

## Publishing

In your plugin's repository, tag the version (`v0.4.2`) and, if it ships binaries, attach the archives
`oriel plugin pack` builds to the tag's GitHub release. Then:

```sh
oriel plugin publish
```

That forks this repo and opens the PR that adds the `[[versions]]` entry, or the `plugin.toml` for a new plugin. A
trusted plugin has nothing to publish: Oriel sees the tag.

## What CI checks on a PR

`.github/workflows/pr.yml` runs `oriel plugin registry check` and comments on the PR. The comment leads with the
capability diff since the last reviewed version, links the source diff, and calls out any `gate` capability.

- The path's owner is the repo's owner, and the PR's author is that owner or a member of the org.
- The tag `v<version>` points at the recorded commit.
- `oriel plugin pack` passes at that commit: the manifest, every visible `oriel.*` call's capability declared, and
  20 MB at most.
- Every platform in `binaries` has an archive. Go plugins are rebuilt with `-trimpath` and must match, `built = "ci"`.
  Pin your Go version with a `toolchain` line in `go.mod` so the rebuild uses the same one. Other binaries are
  checked against the release's archives and recorded as `built = "author"`.
- Go dependencies are checked against Oriel's allowlist (`deps/allow.txt`). One outside it doesn't fail the PR, but
  the install screen shows it.

## The index

On every merge to `main`, `.github/workflows/index.yml` flattens every `plugin.toml` into `index.json`, signs it
with the registry's ed25519 key into `index.sig`, and publishes both as the assets of a new release. Oriel embeds the
public key and refuses an index that doesn't verify.

## Secrets

- `ORIEL_REGISTRY_KEY`: the private signing key, as `oriel plugin registry keygen --out key` writes it.
- `ORIEL_READ_TOKEN`: a fine-grained token with Contents: read on `liamg/oriel`, to build the checker. It also makes
  the PR check's GitHub calls. It can go once `liamg/oriel` is public.
