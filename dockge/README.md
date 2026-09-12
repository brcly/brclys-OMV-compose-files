# Dockge stacks

These are the same services as the top-level OMV-Extras compose files, converted to run under
[Dockge](https://github.com/louislam/dockge) instead of the OMV-Extras "Compose Files" plugin.

## What changed

OMV-Extras' compose plugin uses its own template syntax (`${{ sf:"appdata" }}`, `${{ tz }}`,
`${{ uid:"appuser" }}`, `${{ gid:"users" }}`) that it substitutes before handing the file to
`docker compose`. Dockge doesn't do that substitution — it just runs plain `docker compose` in
each stack's folder, using standard `${VAR}` interpolation from a per-stack `.env` file. So:

| OMV-Extras template          | Replaced with | Set in `.env` as        |
|-------------------------------|----------------|--------------------------|
| `${{ sf:"appdata" }}`        | `${APPDATA}`   | absolute host path       |
| `${{ sf:"data" }}`           | `${DATA}`      | absolute host path       |
| `${{ tz }}`                  | `${TZ}`        | e.g. `America/New_York`  |
| `${{ uid:"appuser" }}`       | `${PUID}`      | numeric UID              |
| `${{ gid:"users" }}`         | `${PGID}`      | numeric GID              |

Each stack folder here has its own `compose.yaml` (Dockge's default filename) and its own `.env`
with `CHANGEME` placeholders — edit the paths/TZ/PUID/PGID in each `.env` to match your system
before deploying. Everything else (ports, images, container names, network layout) is unchanged.

## Using with Dockge

Point Dockge's stacks directory at this `dockge/` folder (or copy each subfolder into wherever
Dockge is configured to look), then edit each stack's `.env` before starting it.

## One-time network setup

Several stacks join `external: true` networks (`internal_bridge`, `lan_macvlan`,
`piholeunbound-net`, `bitmagnet`) that were previously created for you by OMV-Extras. Dockge has
no equivalent network manager, so create these once on the Docker host before deploying, e.g.:

```
docker network create internal_bridge
docker network create -d macvlan \
  --subnet=<your LAN subnet> --gateway=<your LAN gateway> \
  -o parent=<your LAN interface> lan_macvlan
docker network create -d bridge --subnet=192.168.55.0/24 --gateway=192.168.55.1 bitmagnet
```

Adjust the macvlan subnet/gateway/parent interface to match your LAN. `piholeunbound-net` is a
plain bridge network and only needs `docker network create piholeunbound-net`.

## arr-stack-gluetun split

The old combined `arr-stack-gluetun` stack is now one stack per app: `gluetun`, `flaresolverr`,
`qbittorrent`, `radarr`, `sonarr`, `prowlarr`, and `bitmagnet` (which still bundles its
`bitmagnet-postgres` sidecar, since it's not useful on its own).

Everything except `gluetun` and the `postgres` service inside `bitmagnet` joins gluetun's network
via `network_mode: container:gluetun` instead of the compose-only `service:gluetun` shorthand, so
it keeps working across separate Dockge stacks. Two things to know:

- **Start order matters.** Plain `docker compose` has no cross-stack `depends_on`, so start the
  `gluetun` stack first, then the others. If you deploy them all at once and a container fails to
  start because `gluetun` isn't up yet, just restart it once gluetun is healthy.
- **`bitmagnet` network is shared** between the `gluetun` and `bitmagnet` stacks (gluetun and
  postgres both need static IPs on it), so it's created externally once (see above) instead of
  being owned by either stack.
