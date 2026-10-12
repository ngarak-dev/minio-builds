# minio-builds

**Unofficial, unmodified** Linux builds of the [MinIO](https://github.com/minio/minio)
object storage server, built from upstream release tags by GitHub Actions, plus the
official [mc](https://github.com/minio/mc) client binaries re-published next to them.

> **Not affiliated with, endorsed by or supported by MinIO, Inc.**
> MinIO is a trademark of MinIO, Inc.; the name is used here only to identify the
> upstream software these binaries are built from. No source code is changed.
> For supported, signed builds, talk to MinIO, Inc.

Why this exists: the community repository `minio/minio` is archived, `dl.min.io` no longer
serves community binaries (HTTP 410), and the security-fix release
`RELEASE.2025-10-15T17-29-55Z` was published as source code only.

## Install on Ubuntu

The [MinIO fresh install scripts for Ubuntu](https://gist.github.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c)
install these binaries on Ubuntu 22.04 / 24.04 (amd64, arm64): server, systemd service,
`mc`, bucket and least-privilege user, Nginx, UFW, verification, backups and upgrades.

```bash
wget -qO- https://gist.githubusercontent.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c/raw/get.sh | bash
cd minio-install-scripts
sudo ./install-all.sh --dry-run   # review the effective config first
sudo ./install-all.sh
```

Step-by-step walkthrough: [GUIDE.md](GUIDE.md).

## Releases

One GitHub release per upstream server tag, e.g.
[`RELEASE.2025-10-15T17-29-55Z`](https://github.com/ngarak-dev/minio-builds/releases/tag/RELEASE.2025-10-15T17-29-55Z).
Each release contains:

| Asset | Description |
|---|---|
| `minio.linux-amd64.RELEASE.<tag>`, `minio.linux-arm64.RELEASE.<tag>` | Server binaries (upstream file naming) |
| `minio-linux-amd64`, `minio-linux-arm64` | Same binaries under stable names |
| `mc.linux-<arch>.RELEASE.<mc-tag>`, `mc-linux-<arch>` | Official upstream `mc` binaries (upstream SHA256 verified before re-publishing) |
| `*.sha256sum` | One per binary: `<sha256>  <file name>` |
| `SHA256SUMS` | Checksums of every asset |
| `VERSIONS` | `MINIO_RELEASE=`, `MC_RELEASE=`, upstream commits, Go version, workflow run URL |
| `minio-<tag>-source.tar.gz`, `mc-<mc-tag>-source.tar.gz` | Corresponding source (`git archive` of the upstream tags) |
| `LICENSE` | GNU AGPL v3 |

The release body records the upstream commit SHA, the Go toolchain and the workflow run
that produced the binaries. `minio --version` prints the upstream release tag.

Stable download URLs:

```bash
# newest release
https://github.com/ngarak-dev/minio-builds/releases/latest/download/minio-linux-arm64
https://github.com/ngarak-dev/minio-builds/releases/latest/download/VERSIONS
# pinned release
https://github.com/ngarak-dev/minio-builds/releases/download/RELEASE.2025-10-15T17-29-55Z/minio.linux-amd64.RELEASE.2025-10-15T17-29-55Z
```

## Verify a download

```bash
TAG=RELEASE.2025-10-15T17-29-55Z
ARCH=$(dpkg --print-architecture)   # amd64 | arm64
BASE=https://github.com/ngarak-dev/minio-builds/releases/download/$TAG
curl -fLO "$BASE/minio-linux-$ARCH" -fLO "$BASE/SHA256SUMS"
sha256sum -c --ignore-missing SHA256SUMS
chmod +x "minio-linux-$ARCH" && "./minio-linux-$ARCH" --version
```

## How the binaries are built

[`.github/workflows/build-minio.yml`](.github/workflows/build-minio.yml):

1. Checks out `minio/minio` at the requested tag and installs Go from its `go.mod`.
2. Builds natively on linux/amd64 and linux/arm64 runners with the same command as upstream
   `make build`:
   `CGO_ENABLED=0 go build -tags kqueue -trimpath --ldflags "$(MINIO_RELEASE=RELEASE go run buildscripts/gen-ldflags.go <release time>)"`
3. Runs `minio --version` on each architecture and checks it reports the tag.
4. Downloads the `mc` binaries from the `minio/mc` GitHub release and verifies their
   upstream `.sha256sum` files.
5. Publishes everything (plus source tarballs, LICENSE, checksums) as one release.

Build another tag (the release is created/marked latest automatically; an older tag
does not replace the current "latest"):

```bash
gh workflow run build-minio.yml -R ngarak-dev/minio-builds \
  -f minio_tag=RELEASE.2025-10-15T17-29-55Z \
  -f mc_tag=RELEASE.2025-08-13T08-35-41Z \
  -f force=false          # true = rebuild and overwrite an existing release
gh run watch -R ngarak-dev/minio-builds
```

A weekly scheduled run builds the newest upstream tags only if no release exists yet.

## Used by

The [MinIO fresh install scripts for Ubuntu](https://gist.github.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c)
(`DOWNLOAD_SOURCE="github"`, the default; see [GUIDE.md](GUIDE.md)) download from this
repository: pinned releases from `releases/download/RELEASE.<tag>/`, `latest` via the
GitHub API (`/repos/ngarak-dev/minio-builds/releases/latest`, falling back to the
`releases/latest` redirect), the matching `mc` version from the release's `VERSIONS`
file, and checksums from the `.sha256sum` assets.

## License

The MinIO server and client are licensed under the
[GNU Affero General Public License v3.0](LICENSE). Each release includes the complete
corresponding source for the binaries it ships (source tarballs attached to the release;
upstream tags: [minio/minio](https://github.com/minio/minio/tags),
[minio/mc](https://github.com/minio/mc/tags)). The workflow and documentation in this
repository are provided under the same license.
