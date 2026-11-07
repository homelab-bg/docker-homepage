# docker-homepage

A [Homepage](https://gethomepage.dev) dashboard for the homelab, configured for
Docker label-based auto-discovery - other services get added to the dashboard
by adding `homepage.*` labels to their own `docker-compose.yml`, not by
editing anything in this repo.

This repo is public - `.env.example` uses placeholder values (a generic
RFC1918 address, a fake domain) deliberately. Real values only ever go in
your own gitignored `.env`.

## Why these choices

- **Local Docker socket only**, not a remote TCP connection - both this and
  `docker-vikunja` are deployed on the same host (`docker-green`), so
  cross-host label discovery isn't needed yet. `config/docker.yaml` is
  wired for a single `my-docker` instance; adding a second, remote instance
  later is a few lines in that file, not a rebuild of this repo.
- **Docker-socket mounted `:ro`** - Homepage only lists/queries containers
  and reads their stats for widgets, it never starts/stops/manages them, so
  it gets read-only access, not the unrestricted socket access some of this
  engagement's other stacks (Portainer, Traefik) legitimately need.
- **`config/` as a bind mount**, not a named volume - unlike acme.sh's state
  or a database, Homepage's config (`docker.yaml`, `services.yaml`,
  `settings.yaml`, etc) is plain YAML meant to be hand-edited; a bind mount
  keeps it visible and editable directly in the repo clone, same as
  `docker-vscode`/`docker-traefik-portainer`. Overridable via
  `HOMEPAGE_CONFIG_PATH` though - see "Deploying via Portainer" below, the
  relative default doesn't work there.
- **No authentication configured** - not asked for, and this sits behind
  Traefik on an internal-only hostname regardless (see the project's
  Traefik-is-LAN-only note) - add `HOMEPAGE_AUTH_*` later if that changes.
- **Widget credentials via `HOMEPAGE_FILE_*` + Docker `secrets:`**, not
  plaintext env vars - same reasoning as `docker-postfix`'s AWS keys.
  Homepage substitutes a `HOMEPAGE_FILE_*` env var's file *contents* for a
  matching `{{HOMEPAGE_FILE_*}}` placeholder in `config/*.yaml` (confirmed
  via Homepage's own docs), so a widget's token never appears in a config
  file or as a `docker inspect`-visible value.

## Setup

```sh
cp .env.example .env
cp config/docker.yaml.example config/docker.yaml
mkdir -p secrets
echo -n "your-real-proxmox-token-id" > secrets/proxmox_username
echo -n "your-real-proxmox-token-secret" > secrets/proxmox_password
# edit .env: HOMEPAGE_ALLOWED_HOSTS at minimum
docker compose up -d
```

Both secrets files must exist for `docker compose up` to start at all -
Compose reads a `secrets: file:` source eagerly, regardless of whether
`services.yaml` actually has a Proxmox widget. Create them with placeholder
content if not using the widget yet.

`config/docker.yaml.example` is a generic local-socket template - the real
`config/docker.yaml` (gitignored, like `.env`) is where any remote host
entries actually live, since those name real internal hosts. Homepage
generates its own default `settings.yaml`/`services.yaml`/`bookmarks.yaml`/
`widgets.yaml` into the same bind-mounted `config/` directory on first start
if they don't already exist - those are gitignored too once they name real
services, for the same reason. Edit them directly, or through Homepage's own
built-in config editor in the UI.

### Making another service show up on the dashboard

Add labels to that service's own `docker-compose.yml` - nothing changes here:

```yaml
labels:
  homepage.group: Media
  homepage.name: Vikunja
  homepage.icon: vikunja.png
  homepage.href: https://vikunja.lan.example.internal
  homepage.description: Task tracker
```

### Traefik (optional)

To put Homepage behind the shared Traefik instance with a real HTTPS
hostname instead of the bare `:3000` port:

```sh
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d
```

Requires `HOMEPAGE_HOST` and the external `traefik` network (`NETWORK` in
`.env`) to already exist - see `docker-traefik-portainer`.

### Deploying via Portainer

If this is deployed as a Portainer **git-repository** stack (not a plain
`docker compose up` from a real clone), set `HOMEPAGE_CONFIG_PATH` *and*
`HOMEPAGE_SECRETS_PATH` to real, pre-created absolute paths (e.g. a
CephFS/ZFS dataset) as stack environment variables, not just in `.env` -
Portainer git-stacks don't read a repo `.env` file at all.

Confirmed live: Portainer CE has a known, unfixed limitation
([#2368](https://github.com/portainer/portainer/issues/2368), duplicate of
#2046) where a relative bind-mount path in a git-based stack resolves against
Portainer's own internal `/data` view, not the real host filesystem - even
though Portainer's own data directory is itself bind-mounted from somewhere
else entirely (e.g. `docker-traefik-portainer`'s `DATA_DIR`). The container
ends up bind-mounted from whatever real directory happens to sit at the
literal path `/data/compose/<stack-id>/config` on the host root filesystem -
not the cephfs/ZFS-backed location anyone would expect - and Homepage just
writes its own stock default config there, with no error or warning that
anything's wrong. "Relative Path Volumes" that actually works is a Business
Edition-only feature; an absolute host path is the realistic fix on CE.

## Observability

Deploy the [`docker-grafana-stack`](https://github.com/homelab-bg/docker-grafana-stack)
Alloy agent alongside this container (agent-only profile) if it isn't already
running on this host - `STACK_NAME` above is already wired for it.

## Known limitations / not yet done

- Confirmed live (docker-green, deployed via Portainer): needs
  `HOMEPAGE_CONFIG_PATH` set to a real absolute path, or it silently runs on
  Homepage's own stock default config - see "Deploying via Portainer" above.
