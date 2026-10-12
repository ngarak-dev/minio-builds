# Installing MinIO on Ubuntu with these builds

A fresh MinIO install on Ubuntu 22.04 / 24.04 (amd64 or arm64) is automated by the
**MinIO fresh install scripts for Ubuntu** gist:

- Gist: <https://gist.github.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c>
- Full reference (every option, helper script, troubleshooting):
  [README.md in the gist](https://gist.github.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c#file-readme-md)
- Other ways to download it (zip, `git clone`, pinned revision):
  [INSTALL-FROM-GIST.md](https://gist.github.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c#file-install-from-gist-md)

By default the scripts download `minio` and `mc` from the
[releases of this repository](https://github.com/ngarak-dev/minio-builds/releases)
and verify their SHA256 before installing. Every step is idempotent: re-running never
regenerates existing credentials, never overwrites the secrets in `/etc/default/minio`
and never touches the data directory.

> These are **unofficial, unmodified** builds of the upstream MinIO tags, not
> affiliated with or supported by MinIO, Inc. Production needs a build your
> organisation has approved (see [Download source](#download-source)).

## 1. Download the scripts

On the Ubuntu server:

```bash
sudo apt-get update && sudo apt-get install -y wget
wget -qO- https://gist.githubusercontent.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c/raw/get.sh | bash
cd minio-install-scripts
```

`get.sh` downloads every file of the newest gist revision into `./minio-install-scripts`,
syntax-checks the scripts and sets permissions (`*.sh` 755, `config.env` 600). An existing
`config.env` is kept; the new one is saved as `config.env.new`. Options: `--dir PATH`,
`--ref <revision>`, `--force`, `--run -- <installer options>`.

Prefer to read it before running it:

```bash
wget -O get.sh https://gist.githubusercontent.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c/raw/get.sh
less get.sh
bash get.sh --dir ~/minio-install-scripts
```

## 2. Configure

All options are documented inline in `config.env`. Put machine-specific values and
secrets in `config.local.env` next to it (git-ignored, overrides `config.env`);
environment variables override both, e.g. `sudo NGINX_PROFILE=local ./install-all.sh`.

The settings most installs touch:

| Setting | Default | Meaning |
|---|---|---|
| `MINIO_RELEASE` | `2025-10-15T17-29-55Z` | Server release from this repository, or `latest` |
| `MC_RELEASE` | `2025-08-13T08-35-41Z` | Client release; must be the one shipped in the chosen release (see its `VERSIONS` asset) |
| `DOWNLOAD_SOURCE` | `github` | `github` (this repository), `sourceforge` or `custom` (internal mirror) |
| `DATA_DIR` | `/data/minio` | Data directory (`DATA_IS_DEDICATED_MOUNT="yes"` if it is a separate disk) |
| `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` | `minio-root-admin`, prompt | Root account; an empty password is generated; set on first install only |
| `BUCKET_NAME`, `APP_USER`, `APP_POLICY_MODE` | `app-documents`, `app_storage`, `readwrite` | Application bucket and its least-privilege user |
| `NGINX_PROFILE` | `A` | `A` private IP, `B` domains + Let's Encrypt, `local`, `none` |
| `PRIVATE_IP`, `UFW_TRUSTED_SOURCE` | — | Profile A listen address and who may reach the MinIO ports |
| `API_DOMAIN`, `CONSOLE_DOMAIN`, `CERTBOT_EMAIL` | — | Profile B |

Nginx profiles:

| Profile | Result | Client URLs |
|---|---|---|
| `A` | Nginx on `PRIVATE_IP:9000/9002` in front of MinIO on `127.0.0.1`; HTTP or TLS (`NGINX_A_TLS=selfsigned` / `custom`) | `http(s)://PRIVATE_IP:9000` / `:9002` |
| `B` | Nginx on 80/443 for `API_DOMAIN` / `CONSOLE_DOMAIN`, certificates from Let's Encrypt or your own | `https://API_DOMAIN` / `https://CONSOLE_DOMAIN` |
| `local` | No proxy, for local VMs / development | `http://localhost:9000` / `:9002` |
| `none` | Nginx untouched; MinIO on localhost only | — |

## 3. Install

```bash
sudo ./install-all.sh --dry-run   # effective config (no secrets) + steps; changes nothing
sudo ./install-all.sh             # prompts for secrets; Enter = generate
sudo ./show-credentials.sh        # Console login, app keys, Laravel .env (masked)
```

Unattended: `sudo ./install-all.sh --yes`, or in one go from the download:

```bash
curl -fsSL https://gist.githubusercontent.com/ngarak-dev/3eaced97b592b1f6b947f880585aca1c/raw/get.sh | bash -s -- --run -- --yes
```

Resume after a failure with `--from NN`, run single steps with `--only NN[,NN]`, skip
with `--skip NN[,NN]`. Each run is logged to `/var/log/minio-install/`.

| Step | What it does |
|---|---|
| `00-preflight.sh` | Config summary; OS, architecture, disk and port checks |
| `01-install-dependencies.sh` | curl, openssl, nginx, certbot, jq, ufw, xfsprogs |
| `02-prepare-storage.sh` | `minio` system user and data directory (0750) |
| `03-install-minio-server.sh` | Downloads the server from this repository, verifies SHA256, installs `/usr/local/bin/minio` |
| `04-configure-minio-service.sh` | `/etc/default/minio` (600), systemd unit, start + health check |
| `05-install-mc-client.sh` | Downloads `mc` from the same release, verifies SHA256, creates the `local` admin alias |
| `06-create-bucket-and-iam.sh` | Private bucket(s), versioning/lock/quota/lifecycle, least-privilege policy and users, upload/download test |
| `07-configure-nginx.sh` | Nginx profile and TLS |
| `08-configure-firewall.sh` | UFW: SSH first, MinIO ports only from trusted sources |
| `09-verify.sh` | Pass/fail checks for service, health, `mc`, buckets, users and proxy |

## 4. Operate

```bash
sudo ./status.sh                              # service, versions, health, buckets, users, firewall
sudo ./backup.sh --install-cron "30 2 * * *"  # config/IAM archive + bucket mirrors, nightly
sudo ./add-bucket.sh backups --quota 50GiB --save
sudo ./add-user.sh reporting --mode readonly --buckets app-documents --save
sudo ./rotate-credentials.sh --app
sudo ./uninstall.sh --dry-run                 # keeps data and secrets unless asked
```

## 5. Upgrade to a newer build

When a new release appears [here](https://github.com/ngarak-dev/minio-builds/releases):

```bash
sudo ./upgrade.sh --dry-run                  # current vs target, checks the download
sudo ./upgrade.sh --release latest --yes     # newest release of this repository
sudo ./upgrade.sh --rollback                 # restore the previous binary
```

`upgrade.sh` backs up the current binary, health-checks MinIO after the switch and rolls
back automatically if it is unhealthy. Releases older than `RELEASE.2025-10-15T17-29-55Z`
lack security fixes and are for labs only (the scripts warn).

## Download source

| `DOWNLOAD_SOURCE` | Where `minio` / `mc` come from |
|---|---|
| `github` (default) | `releases/download/RELEASE.<tag>/minio.linux-<arch>.RELEASE.<tag>` of this repository (+ `.sha256sum`); `mc` from the same release; `latest` via the GitHub API, `mc` version from the release's `VERSIONS` asset |
| `sourceforge` | Third-party mirror of the old upstream downloads (no binaries for 2025-10-15 or newer) |
| `custom` | Your mirror: `MINIO_BINARY_URL`, `MINIO_SHA256_URL`, `MC_BINARY_URL`, `MC_SHA256_URL` (placeholders `{RELEASE}` `{ARCH}` `{FILE}`) |

To use a fork of this repository, set `BUILDS_REPO="<owner>/minio-builds"`; `BUILDS_TAG`
picks a specific release.

## Not automated

- Formatting and mounting a dedicated data disk (XFS, `/etc/fstab` by UUID).
- DNS records for profile B and inbound port 80 for certbot.
- Moving the generated secrets into your secrets manager.
- Monitoring and the go-live checklist.
