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

## Known Issue — podman 4.9 + cgroupv2 breaks auto-update (found 2026-10-19)

Watchtower recreates containers via the API, copying the old container's HostConfig
(including `MemorySwappiness`). Podman bakes that into the OCI spec and crun fails:

```
Error response from daemon: crun: cannot set memory swappiness with cgroupv2: OCI runtime error
```

The damage is one-way: watchtower has already SIGTERM'd and removed the running
container, so the service is left **created but never started**. Worse, watchtower
only scans *running* labeled containers (hourly `Session done Scanned=N` count drops
as things die), so it **never retries** — the outage persists until a manual recreate.

Incident: on tva 2026-10-19, hourly updates killed `jellyfin-app` (new ls55 image)
and `jellyfin-dozzle-agent` (v11.3.3); both stayed down for days while the tailscale
sidecar and watchtower itself ran fine. Recovery: `make upgrade` in the stack dir
(plain `podman start` does NOT work — the bad field is baked into the saved spec).

Mitigations until podman/crun is fixed upstream:
- Pin version tags in `.env` instead of floating `latest` — watchtower only acts when
  the pinned tag's digest changes (delays, doesn't prevent).
- Drop the label from low-value services (e.g. dozzle-agent).
- Compose-level healthchecks do NOT help here: start fails before any healthcheck runs,
  so there is nothing for rollback to act on.

Fix status (verified 2026-10-23):
- **No crun release fixes this.** Current crun main still errors by design when the OCI
  spec carries a swappiness field under cgroup v2 (`src/libcrun/cgroup-resources.c`:
  `swappiness_present && cgroup2 → error`). Cgroup v2 has no per-cgroup swappiness knob,
  so ignoring the field loses nothing (Docker ignores it on v2 too).
- **Podman ≥5.0 fixes it** (guard present since v5.0.0 in
  `pkg/api/handlers/compat/containers_create.go`: rootless + cgroupv2 + non-systemd
  manager → swappiness is not written to the spec at all). Ubuntu 24.04 ships only
  podman 4.9.x, so this means a source build.

Two remediation paths for tva:
1. **Surgical (recommended first):** patch crun 1.14.1's swappiness block to skip
   instead of error under cgroup v2, rebuild, install the binary. Works with existing
   podman 4.9.3; fully reversible (remove the binary). Re-apply if Ubuntu ever updates
   crun via apt.
2. **Proper:** upgrade podman to ≥5.0 (source build on 24.04) — fixes this for all
   API-based recreators fleet-wide, but touches the core runtime of every stack.

Verification canary after either fix: `podman run --rm --memory 1g --memory-swappiness 0
alpine true` must succeed (before the fix it exits 126 with the crun error). Then force
one real watchtower cycle by retagging an older local image as `latest`, and confirm in
the journal that the recreate succeeds.

All stacks with watchtower are exposed (dns pihole/caddy, dozzle hub, immich-server/redis).
