# b4hub

`b4hub` is the service behind community sets in [b4](https://github.com/DanielLavrushin/b4).
b4 routers share, rate and download sets through it. The central hub is
`https://hub.b4core.app`; its signing key is compiled into b4, so a router with community
sets enabled talks to it without any configuration.

The same binary runs anywhere else in two roles:

- **Central hub** (`b4hub serve`): owns the store, moderates shares, signs and publishes
  the catalogue. There is one publisher key per hub network; routers trust exactly one key.
- **Child hub** (`b4hub mirror`): keeps a byte-identical copy of a parent's signed
  catalogue and relays signed records (shares, votes, reports) to the parent. Routers
  pointed at a child see the same catalogue as routers pointed at the parent, verified
  with the parent's key. A child announces itself with `--announce`; once the parent's
  operator approves it and its health checks pass, the parent lists it in the manifest
  and routers learn it as an alternative address.

An operator who wants an independent network (own moderation, own key, no relation to
`hub.b4core.app`) runs `b4hub serve` and gives routers the hub address and key id in
`system.hub` settings. An operator who wants to serve their routers locally while staying
part of the community runs `b4hub mirror`.

The source code lives in the b4 repository under
[`hub/`](https://github.com/DanielLavrushin/b4/tree/main/hub). This repository holds what
an operator needs: the [releases](https://github.com/DanielLavrushin/b4hub/releases), the
deployment bundle in [deploy/](deploy/) (Docker Compose, systemd unit, nginx vhost,
environment template) and this guide.

## Releases

`b4hub` has its own version, independent of the b4 version. Each release carries
`b4hub-linux-amd64.tar.gz` and `b4hub-linux-arm64.tar.gz` with `SHA256SUMS`, and pushes
`lavrushin/b4hub:<version>` (also `ghcr.io/daniellavrushin/b4hub:<version>`) for linux/amd64
and linux/arm64. `lavrushin/b4hub:latest` is the newest release that is not a pre-release.

A release is built from one commit of the b4 repository, because the hub compiles against
the router source tree, which holds the wire format. `b4hub version` prints both, for
example `1.0.0 (b4 v1.82.1-5-g1a2b3c4)`, and the release notes link the commit.
Compatibility between routers and hubs does not depend on either number: the wire format
is versioned separately and each shared set carries the minimum b4 version it needs. The
rule for an operator is to run the newest published hub.

Hub versions 1.82.0rc2 and 1.82.0 were shipped inside b4 releases under the b4 version
number. 1.0.0 is the first standalone release and supersedes them.

## Run with Docker

The quickest way to run a hub of either role. [deploy/docker/](deploy/docker/) holds a
Compose file with `b4hub` and a Caddy front that obtains and renews the certificate on its
own; the only requirement is a host with ports 80 and 443 reachable under a DNS name.

```sh
mkdir b4hub && cd b4hub
curl -fsSLO https://raw.githubusercontent.com/DanielLavrushin/b4hub/main/deploy/docker/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/DanielLavrushin/b4hub/main/deploy/docker/Caddyfile
curl -fsSL -o .env https://raw.githubusercontent.com/DanielLavrushin/b4hub/main/deploy/docker/.env.example
```

`.env` selects the role. A central hub keeps `B4HUB_COMMAND=serve` and needs
`B4HUB_ADMIN_PASSWORD`; a child of `hub.b4core.app` sets `B4HUB_COMMAND=mirror --announce`
and `B4HUB_UPSTREAM=https://hub.b4core.app` instead. `HUB_DOMAIN` is the public name in both
cases; it becomes `B4HUB_PUBLIC_URL` and the Caddy site.

```sh
docker compose run --rm b4hub keygen     # once; prints the key id, writes hub.key into the volume
docker compose up -d
docker compose logs -f b4hub
```

The Compose network is pinned to `10.84.0.0/24` and `B4HUB_TRUSTED_PROXIES` covers it, so the
hub takes the client address from Caddy's `X-Forwarded-For` instead of attributing every
request to the Caddy container; without that, network scoring, login limits and ingest quotas
would all see one client. If the subnet collides with a network already on the host, change it
in both places. Nothing else may reach `b4hub` on that network: a peer in the trusted range can
claim any client address.

The signing key sits in the `b4hub-data` volume; `docker compose down -v` deletes it together
with the store, so back up `hub.key` (`docker compose cp b4hub:/var/lib/b4hub/hub.key .`) before
anything that removes volumes. The image runs as uid 7100; a bind mount in place of the named
volume has to be owned by that uid. Updating is `docker compose pull && docker compose up -d`;
the store migrates on start.

The image without Compose:

```sh
docker run --rm -v b4hub:/var/lib/b4hub lavrushin/b4hub keygen
docker run -d --name b4hub -v b4hub:/var/lib/b4hub -p 127.0.0.1:7100:7100 \
  -e B4HUB_PUBLIC_URL=https://hub.example.net -e B4HUB_ADMIN_PASSWORD=... lavrushin/b4hub
```

behind any TLS-terminating proxy that forwards `X-Forwarded-Proto: https`. A proxy on the host
reaches the published port through Docker's bridge, so the hub sees the bridge gateway
(`172.17.0.1` by default) as the peer, not loopback; `-e B4HUB_TRUSTED_PROXIES=172.17.0.1` makes
its `X-Forwarded-For` count. The setting, also `--trusted-proxies`, takes addresses or CIDRs
separated by commas; loopback is always trusted.

## Build

From a checkout of the b4 repository:

```sh
make hub-build HUB_VERSION=1.0.0          # current platform, out/b4hub
make hub-linux-amd64 HUB_VERSION=1.0.0    # linux/amd64, out/assets/b4hub-linux-amd64.tar.gz (+ .sha256)
make hub-linux-arm64 HUB_VERSION=1.0.0    # linux/arm64
make hub-linux-all HUB_VERSION=1.0.0      # both
make hub-docker HUB_VERSION=1.0.0         # lavrushin/b4hub:1.0.0 from hub/Dockerfile, context = repo root
```

All of them build the moderation console first (`pnpm build` in `hub/ui`) and embed it. A plain
`go build` produces a binary whose `/admin` answers 503 "not built into this binary".
`HUB_VERSION` defaults to `dev`; the b4 source tree (`git describe`) is embedded alongside and
shown by `b4hub version` and on the console's Overview page.

## Install a central hub (systemd)

This is how `hub.b4core.app` runs.

1. Copy the binary: `sudo install -m0755 b4hub /usr/local/bin/b4hub`.
2. Service account and data directory:

   ```sh
   sudo useradd --system --home /var/lib/b4hub --shell /usr/sbin/nologin b4hub
   sudo install -d -o b4hub -g b4hub -m 0750 /var/lib/b4hub
   ```

3. Signing identity, once:

   ```sh
   sudo -u b4hub b4hub keygen --data /var/lib/b4hub
   ```

   The command prints the key id. Back up `/var/lib/b4hub/hub.key` offline: routers trust
   the hub by this key, and a lost key means every router has to be reconfigured with a new
   one. Routers of a self-hosted hub set the address and this key id under `system.hub`.

4. Environment: `sudo install -d -m 0750 /etc/b4hub`, then `/etc/b4hub/env` from
   [deploy/env.example](deploy/env.example) with a real `B4HUB_ADMIN_PASSWORD`, owned
   `root:b4hub`, mode `0640`. The password is the only credential of the console at `/admin`;
   changing it ends every signed-in session.
5. Unit: copy [deploy/b4hub.service](deploy/b4hub.service) to `/etc/systemd/system/`, set
   `B4HUB_PUBLIC_URL` to the public address, then
   `sudo systemctl daemon-reload && sudo systemctl enable --now b4hub`.
   The unit listens on `127.0.0.1:7100`; nothing else is exposed directly.
6. nginx: copy [deploy/nginx-hub.conf](deploy/nginx-hub.conf) to `/etc/nginx/conf.d/`, set
   `server_name` and the certificate paths, `sudo nginx -t && sudo systemctl reload nginx`.
   The vhost proxies the router endpoints (`/b4/hub/...`), the console (`/admin`) and the
   console's hashed assets (`/admin/assets/`, cached as immutable). `X-Forwarded-Proto`
   must be `https`: it makes the session cookie `Secure`.
7. Geo databases download on the first start and daily after that. The first catalogue is
   built once a share is approved, or on demand from the console's Catalogue page
   (`sudo -u b4hub b4hub build --data /var/lib/b4hub` does the same).

## Install a child hub (systemd)

Same binary, own data directory and identity, `mirror` instead of `serve`:

```sh
sudo -u b4hub b4hub keygen --data /var/lib/b4hub
b4hub mirror --data /var/lib/b4hub --upstream https://hub.b4core.app \
  --listen 127.0.0.1:7100 --public-url https://hub.example.net --announce
```

`--upstream-key` pins the parent's key id; without it the key compiled into b4 is trusted,
which is right for a child of `hub.b4core.app`. In the unit, replace `serve` in `ExecStart`
and put `B4HUB_UPSTREAM` in `/etc/b4hub/env`. A child has no moderation console and no
admin password; the nginx vhost is the same minus the `/admin` locations.

## Update

From a release: download the tarball for the architecture, check it against `SHA256SUMS`,
then

```sh
tar -xzf b4hub-linux-amd64.tar.gz
sudo install -m0755 b4hub /usr/local/bin/b4hub && sudo systemctl restart b4hub
```

From a b4 checkout, `make hub-deploy HUB_VERSION=1.0.1` cross-compiles for linux/amd64,
copies the binary to `HUB_DEPLOY_HOST` (a `user@host` with passwordless `sudo`, optionally
`HUB_DEPLOY_KEY` for the SSH key; both belong in the gitignored `.env`), installs it as
`/usr/local/bin/b4hub`, restarts the unit and prints `systemctl is-active` and
`b4hub version`.

Data under `/var/lib/b4hub` is untouched by an update; schema migrations run on start, the
key never changes. When [deploy/nginx-hub.conf](deploy/nginx-hub.conf) changed, the vhost on
the box has to be updated by hand, since the deployed copy carries the box's certificate paths.

## Checks

```sh
curl -s https://hub.example.net/admin/api/session     # {"configured":true,"authenticated":false,"version":"1.0.0"}
curl -s -o /dev/null -w '%{http_code}\n' https://hub.example.net/b4/hub/manifest.json
sudo -u b4hub b4hub moderate --data /var/lib/b4hub list
```

`configured:false` means `B4HUB_ADMIN_PASSWORD` is missing from the environment. The
`moderate` subcommands (`list`, `approve`, `reject`, `hide`, `ban`, `mirrors`,
`approve-mirror`, `reject-mirror`) act on the same store as the console.

## Publishing a release

1. Add the entry for the new version at the top of
   [`hub/CHANGELOG.md`](https://github.com/DanielLavrushin/b4/blob/main/hub/CHANGELOG.md)
   in the b4 repository and merge it.
2. Run the [Release workflow](https://github.com/DanielLavrushin/b4hub/actions/workflows/release.yml)
   in this repository with the version and the b4 ref to build from, normally `main`. It runs
   the hub tests, builds both tarballs, pushes the images and publishes the release here with
   the changelog entry and the b4 commit it was built from. A version that is already released
   or has no changelog entry is refused. A draft run builds everything and publishes nothing.

The workflow needs the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets in this repository;
`ghcr.io` is pushed with the workflow's own token, so the `b4hub` package must belong to this
repository or grant it write access.
