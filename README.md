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
  `docker-vscode`/`docker-traefik-portainer`.
- **No authentication configured** - not asked for, and this sits behind
  Traefik on an internal-only hostname regardless (see the project's
  Traefik-is-LAN-only note) - add `HOMEPAGE_AUTH_*` later if that changes.

## Setup

```sh
cp .env.example .env
# edit .env: HOMEPAGE_ALLOWED_HOSTS at minimum
docker compose up -d
```

`config/docker.yaml` is committed pre-seeded with the local socket entry;
Homepage generates its own default `settings.yaml`/`services.yaml`/
`bookmarks.yaml`/`widgets.yaml` into the same bind-mounted `config/` directory
on first start if they don't already exist - edit those directly, or through
Homepage's own built-in config editor in the UI.

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

## Observability

Deploy the [`docker-grafana-stack`](https://github.com/homelab-bg/docker-grafana-stack)
Alloy agent alongside this container (agent-only profile) if it isn't already
running on this host - `STACK_NAME` above is already wired for it.

## Known limitations / not yet done

- Not yet deployed or tested live - compose files and config are built and
  reasoned through against Homepage's own docs, but `docker compose up -d`
  hasn't actually been run anywhere yet.
- No GitHub remote yet - working from this local repo until one's created.
