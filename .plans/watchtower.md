# Watchtower Plan — Fleet Auto-Update with Rollback

## Decisions (per user)

| Image | Strategy | Rationale |
|-------|----------|-----------|
| `tailscale/tailscale` (×4) | Float to `latest` + watchtower | Tailscale is conservative; SERVE configs don't break across versions |
| `pihole/pihole` | Already `latest` + watchtower | — |
| `caddy` | Keep `2-alpine` + watchtower | Minor tag gets security patches, Caddy 2.x stable API |
| `amir20/dozzle` (×4) | Already `latest` + watchtower | — |
| `linuxserver/jellyfin` | Already `${VERSION:-latest}` + watchtower | — |
| `immich-server` | Already `${VERSION:-release}` + watchtower | `release` tag moves with each Immich release |
| `valkey/valkey` | Change from digest-pinned → `9-alpine` + watchtower | Cache store, no state to corrupt on rollback |
| `immich postgres` | **Keep digest-pinned**, NO watchtower | DB migrations can't roll back; upgrade manually in lockstep with immich-server |

## Watchtower Configuration Per Stack

Each stack gets a `watchtower` service added to its compose. It monitors only that stack's containers (rootless podman state is per-user).

```yaml
watchtower:
  image: docker.io/containrrr/watchtower:latest-ce
  container_name: <stack>-watchtower
  networks: [<stack>-net]
  environment:
    TZ: ${TZ:-UTC}
    WATCHTOWER_CLEANUP: "true"          # remove old images after update
    WATCHTOWER_ROLLING_RESTART: "true"  # recreate one at a time
    WATCHTOWER_MONITOR_ONLY: "false"    # actually perform updates
    WATCHTOWER_POLL_INTERVAL: "3600"    # check hourly
    WATCHTOWER_LABEL_ENABLE: "true"     # only watch containers with com.centurylinklabs.watchtower=true label
  volumes:
    - ${PODMAN_SOCKET}:/var/run/docker.sock
```

Each updatable service gets a `com.centurylinklabs.watchtower.enable` label (`"true"`). The postgres container and dozzle-agent get no label so watchtower skips them.

### Rollback Behavior

Watchtower uses healthchecks to decide rollback:
1. Pulls new image, stops old container, starts new one
2. If the new container has a healthcheck defined and it fails within the grace period (default 8s), watchtower stops the new container and restarts the old one from the cached image
3. `--rolling-restart` ensures only one container is down at a time

Services without explicit healthchecks in compose rely on the image's built-in HEALTHCHECK — or we can add one. Watchtower also supports `WATCHTOWER_HEALTHCHECKMANAGEMENT=true` to auto-generate basic HTTP healthchecks for services that expose an API.

## Changes Required Per Stack

### dns-stack
- Change tailscale image: `v1.102.5` → `latest`
- Add watchtower service
- Add watchtower labels to all updatable containers (pihole, caddy, dozzle-agent) — skip tailscale sidecar

### dozzle-stack  
- Change tailscale image: `v1.102.5` → `latest`
- Add watchtower service
- Add watchtower label to hub container (dozzle-agent not present here)

### jellyfin-stack
- Change tailscale image: `v1.102.5` → `latest`
- Add watchtower service
- Add watchtower labels to jellyfin-app and dozzle-agent — skip tailscale sidecar

### immich-stack
- Change tailscale image: `v1.102.5` → `latest`
- Change valkey image: digest-pinned → `9-alpine`
- Add watchtower service
- Add watchtower labels to immich-server, redis, dozzle-agent — skip postgres (digest-pinned) and tailscale sidecar

## Deployment Order

1. Push compose changes + label additions to each repo (git commit + push)
2. Pull on the target server: `cd /path/to/<stack>-stack && git pull`
3. Run `make upgrade` — this pulls new images including watchtower and recreates containers with labels
4. Watchtower starts automatically and begins monitoring

## Notes

- Tailscale sidecars are excluded from watchtower because they manage their own node identity (state.soc) and SERVE config. Auto-recreating them is harmless but unnecessary — the tailscale image handles upgrades gracefully on its own via `make upgrade`.
- Dozzle-agent: included in watchtower updates since it's just a log collector with no state.
- Watchtower itself gets the label too, so it self-updates (standard practice).
