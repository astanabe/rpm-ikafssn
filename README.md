# rpm-ikafssn

A standalone DNF / YUM channel for [ikafssn](https://github.com/astanabe/ikafssn).

This repository hosts the GPG-signed DNF channel served from
`https://rpm.ikafssn.org` (GitHub Pages with the `rpm.ikafssn.org`
custom domain). Unlike the Conda channel, the `.rpm` packages
themselves are stored *in this repository* under `el<releasever>/<arch>/`,
because the YUM metadata references each `.rpm` by a relative path
and GitHub Pages cannot serve real HTTP redirects.

The channel always reflects the latest ikafssn release; older versions
are not retained. Past versions can still be downloaded directly from
[ikafssn's GitHub Releases](https://github.com/astanabe/ikafssn/releases).

## Installation

One-time setup (run as root or via `sudo`):

```bash
sudo tee /etc/yum.repos.d/ikafssn.repo > /dev/null <<'EOF'
[ikafssn]
name=ikafssn
baseurl=https://rpm.ikafssn.org/el$releasever/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://rpm.ikafssn.org/ikafssn-archive-keyring.asc
EOF
sudo dnf install ikafssn
```

Subsequent upgrades:

```bash
sudo dnf upgrade ikafssn
```

Both the package signature (`gpgcheck=1`) and the repository metadata
signature (`repo_gpgcheck=1`) are verified against the channel's
public key.

## Supported releasevers

| `releasever` | Distribution | Architectures |
|---|---|---|
| `9` | AlmaLinux / Rocky Linux / RHEL 9 | `x86_64`, `aarch64` |
| `10` | AlmaLinux / Rocky Linux / RHEL 10 | `x86_64`, `aarch64` |

## Layout

```
CNAME, README.md, ikafssn-archive-keyring.asc              (static; one-time)
el<releasever>/<arch>/ikafssn-<ver>.el<rv>.<arch>.rpm      (regenerated per release; rpm --addsign'd)
el<releasever>/<arch>/repodata/repomd.xml                  (regenerated per release)
el<releasever>/<arch>/repodata/repomd.xml.asc              (detached GPG signature of repomd.xml)
el<releasever>/<arch>/repodata/primary.xml.zst             (regenerated per release)
el<releasever>/<arch>/repodata/filelists.xml.zst           (regenerated per release)
el<releasever>/<arch>/repodata/other.xml.zst               (regenerated per release)
```

## How this channel works

`dnf install ikafssn` triggers:

1. DNF fetches `https://rpm.ikafssn.org/el<rv>/<arch>/repodata/repomd.xml`
   plus its `repomd.xml.asc` detached signature, verified against the
   public key declared in `gpgkey=` (`repo_gpgcheck=1`).
2. DNF fetches the referenced `primary.xml.zst` / `filelists.xml.zst`
   and resolves the package set.
3. DNF downloads the requested `.rpm` from
   `el<rv>/<arch>/ikafssn-<ver>.el<rv>.<arch>.rpm`, verifies its
   embedded GPG signature (`gpgcheck=1`), and installs it.

## Release flow

When a new ikafssn release is published, the channel layout under
`el9/` and `el10/` is regenerated from scratch. This is automated by
ikafssn's
[`release.yml`](https://github.com/astanabe/ikafssn/blob/main/.github/workflows/release.yml)
workflow (`update-rpm-channel` job): after all `.rpm` artifacts are
uploaded to the release, the helper `recipe/rpm-publish.sh`
- wipes `el*/`,
- downloads the four `.rpm` assets,
- re-signs each `.rpm` in place with `rpm --addsign`,
- runs `createrepo_c --general-compress-type zstd`,
- detach-signs `repomd.xml` with `gpg --detach-sign --armor`,

and pushes the result to `main` here.

## Verifying the public key

The same `ikafssn-archive-keyring.asc` is committed at the root of
the [astanabe/ikafssn](https://github.com/astanabe/ikafssn)
repository, so the key can be fetched out-of-band and compared against
the one served by this channel before trusting it.

## License

Apache-2.0 (matching ikafssn).
