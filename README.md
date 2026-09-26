# rsync

Container images with [rsync](https://rsync.samba.org/), which copies and synchronizes files locally, over SSH or through an rsync daemon, and sends only the parts of files that changed. rsync is compiled from the signed release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when rsync publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the rsync project. Report problems with the image in this repository and rsync bugs [upstream](https://github.com/RsyncProject/rsync/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/rsync \
  -a --delete photos/ photos-backup/
```

The same images can also be pulled as `randomcontainers.com/rsync`.

Show what the same command would change, without copying anything:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/rsync \
  -a --delete --dry-run --itemize-changes photos/ photos-backup/
```

The entrypoint runs `rsync` under `tini` in `/work`, so relative paths start in the directory you mount. To copy between two directories on the host, mount both:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v /srv/data:/src:ro -v /mnt/backup:/dst \
  ghcr.io/randomcontainers/rsync -a --delete /src/ /dst/data/
```

The [rsync manual](https://download.samba.org/pub/rsync/rsync.1) covers the options and filter rules, and [rsyncd.conf](https://download.samba.org/pub/rsync/rsyncd.conf.5) the daemon configuration.

## What is in the image

- `rsync` in `/usr/local/bin`, with ACLs (`-A`), extended attributes (`-X`), zstd and LZ4 compression next to zlib, xxHash checksums, OpenSSL digests (SHA-1 checksums, SHA-256 and SHA-512 for daemon logins) and internationalized host names.
- `ssh` from the OpenSSH client, which rsync uses as its remote shell.
- The distro's libacl, popt, OpenSSL, zstd, LZ4, xxHash and libidn2. rsync's own copy of zlib is compiled in.

Not included: `rsync-ssl`, a bash script that needs openssl, stunnel or gnutls-cli; `rrsync`, which needs Python; and the man pages. The configure flags and the output of `rsync --version` are in `/usr/local/share/randomcontainers/rsync/buildinfo`.

## Default or slim

rsync's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is an rsync release such as `3.5.1`. `<minor>` and `<major>` are its shorter forms, `3.5` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current rsync version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`-a` includes `-o` and `-g`, but only root can hand files to another owner. As any other user, rsync keeps modes and times, and the copies belong to the UID and GID the container runs as. To keep owners and groups, run the container with `--user 0:0`, or pass `--fake-super` to store them in extended attributes instead.

## Copying over SSH

rsync starts `ssh` from the image as its remote shell. ssh exits with `No user exists for uid` when the UID has no entry in `/etc/passwd`, and the only regular user in the image is UID 1000 (`ubuntu` on Ubuntu, `app` on Alpine). Run SSH transfers as UID 1000 and pass the key and `known_hosts` file explicitly:

```sh
docker run --rm --user 1000:1000 -v "$PWD:/work" -v "$HOME/.ssh:/ssh:ro" \
  ghcr.io/randomcontainers/rsync -a \
  -e "ssh -i /ssh/id_ed25519 -o UserKnownHostsFile=/ssh/known_hosts" \
  site/ deploy@example.com:/var/www/site/
```

UID 1000 has to be able to read the key, and the server has to be in `known_hosts` already: without a terminal, ssh cannot ask whether to trust a new host key and refuses to connect. Files that rsync writes to `/work` belong to UID 1000. For another UID, add a user for it in your own image (see [Extending the slim image](#extending-the-slim-image)).

## Running an rsync daemon

A config file for one writable module, with `use chroot = no` because only root can chroot:

```ini
use chroot = no
log file = /work/rsyncd.log

[backup]
    path = /work/backup
    read only = no
    auth users = backup
    secrets file = /work/rsyncd.secrets
    auth digest = sha512
```

`rsyncd.secrets` holds a `backup:<password>` line and must not be readable by other users (`chmod 600`). `auth digest = sha512` refuses clients that can only log in with MD4 or MD5: rsync before 3.2.7, rsync built without OpenSSL (Alpine's rsync package, for one) and the openrsync that ships with macOS. The daemon does not create the module directory, so create `backup/` next to the config file first. Then start the daemon in the foreground on a port above 1024 and publish that port:

```sh
docker run -d --name rsyncd --user "$(id -u):$(id -g)" -p 873:8730 -v "$PWD:/work" \
  ghcr.io/randomcontainers/rsync --daemon --no-detach --port=8730 --config=/work/rsyncd.conf
```

Clients connect with `rsync -a files/ rsync://backup@<host>/backup/` and give the password in `RSYNC_PASSWORD` or with `--password-file`. The daemon protocol is not encrypted, so run it only on a network you trust. rsync releases often fix security issues in path handling and the daemon, and the images pick up a new release about a day after it is published.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new rsync release and are rebuilt when the base image changes. Switch to root to change the image, then back. This adds a user for UID 1001, so that ssh can run as that UID:

```dockerfile
FROM ghcr.io/randomcontainers/rsync:slim-ubuntu@sha256:...
USER root
RUN groupadd --gid 1001 deploy \
 && useradd --uid 1001 --gid 1001 --create-home deploy
USER 1001:1001
```

On Alpine, start from `slim-alpine` and use `addgroup -g 1001 deploy && adduser -D -u 1001 -G deploy deploy`. The entrypoint is `["tini", "--", "rsync"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new rsync releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/rsync/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages rsync needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/rsync:latest \
  --repo randomcontainers/rsync --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/rsync:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the rsync release signing keys in `keys/rsync-release.gpg`: Andrew Tridgell's (fingerprint `9FEF 112D CE19 A0DC 7E88 2CB8 1BB2 4997 A853 5F6F`) and Zen Dodd's (fingerprint `C0E1 0545 0570 4F75 7D92 5AB2 6A64 E8AA 2EFB CF5D`), who signed 3.5.1.

## Updates

The project checks the releases of [RsyncProject/rsync](https://github.com/RsyncProject/rsync/releases) every 15 minutes and skips pre-releases. A release is picked up once it is 24 hours old. Its tarball is checked against the SHA-256 digest GitHub recorded for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. A release signed with a key that is not in `keys/rsync-release.gpg` fails to build until the key is added. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t rsync:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

rsync is licensed under the GNU General Public License, version 3 or later (GPL-3.0-or-later), with an added permission to link it with the OpenSSL and xxHash libraries. The build compiles in rsync's copy of zlib, which is under the zlib license, so the image's license label is `GPL-3.0-or-later AND Zlib`. rsync's `COPYING` and the zlib license (`COPYING.zlib`) are in `/usr/local/share/randomcontainers/rsync/licenses/`. The OpenSSH client and the other Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- rsync: every version has a GitHub release in this repository, named `v<version>`, with the exact `rsync-<version>.tar.gz` that was compiled and its signature. The build applies no patches. `/usr/local/share/randomcontainers/rsync/source` lists that release and the upstream download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
