# jellyfin-stack

Jellyfin media server. Own Tailscale node **`jellyfin`**; UI at `https://jellyfin.<tailnet>.ts.net`. Split out of the deprecated `media-stack` on 2026-10-16.

## Role in the Fleet

Self-hosted media streaming server. Handles video/audio playback with VA-API hardware transcoding. Libraries are bind-mounted from network storage (`/mnt/samba_storage/jellyfin`). The Tailscale node identity was migrated from `media-stack`'s named volume, so the hostname/cert remain unchanged.

## Services

| Container | Image | Role |
|---|---|---|
| `jellyfin-tailscale` | `tailscale/tailscale:v1.102.5` (pinned) | Userspace tailscaled + SERVE (:443 → jellyfin app, raw TCP :7007 → dozzle-agent). Pinned to v1.102.5 — verify before bumping as Tailscale skips some patch versions. |
| `jellyfin-app` | `linuxserver/jellyfin:latest` (tag from `.env`) | The media server. Config/playlists are bind mounts (`./jellyfin/config/`, `./playlists/`). Libraries from `/mnt/samba_storage/jellyfin` (read-only). VA-API transcoding via render node device passthrough. |
| `jellyfin-dozzle-agent` | `amir20/dozzle:latest` (`agent`) | Reports this stack's containers to the central Dozzle hub over mTLS gRPC on :7007. |

No host ports — everything is exposed through Tailscale SERVE only.

## Data

| Data | Location | Type |
|------|----------|------|
| Jellyfin config + metadata | `./jellyfin/config/` bind mount | Version-controlled path, data on disk |
| Playlists | `./playlists/` bind mount | On disk (bind mount so migration works) |
| Media libraries | `/mnt/samba_storage/jellyfin` (CIFS bind mount, read-only) | Network storage — host dir must exist before `make up` |
| Tailscale node state | `./ts_state/state.soc` bind mount | Persistent identity — DO NOT delete or the node gets a new hostname |

## Secrets

| File | Source | Used By |
|---|---|---|
| `.tailscale-authkey` | Copied from `$STACK_ENV_DIR` by setup | Tailscale sidecar auth |
| `.dozzle-cert.pem`, `.dozzle-key.pem` | Copied from `$STACK_ENV_DIR` (generated once by dozzle-stack) | Dozzle agent mTLS credential |

## SERVE Config

```json
{
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": { "/": { "Proxy": "http://jellyfin:8096" } }
    }
  },
  "TCP": {
    "443": { "HTTPS": true },
    "7007": { "TCPForward": "jellyfin-dozzle-agent:7007" }
  }
}
```

Routes `https://jellyfin.<tailnet>.ts.net` → jellyfin container on port 8096. Raw TCP :7007 forwards to the dozzle agent for log collection.

## Hardware Transcoding (VA-API)

Jellyfin gets a single render node passed through for VA-API transcoding — `JELLYFIN_RENDER_NODE` in `.env` (default `/dev/dri/renderD128`). On hosts without GPU access, comment out the matching line under `devices:` in `podman-compose.yml`.

If Jellyfin logs show permission errors on the render node:
```bash
# Add the stack user to the video group for /dev/dri access
sudo usermod -aG video jellyfin
```

## Migration from media-stack (2026-10-16)

- `jellyfin/config/` and `playlists/` moved in as bind mounts (same host paths inside the container — no reconfiguration needed).
- Tailscale node identity: old named volume `jellyfin_ts_state` → `./ts_state/state.soc`. Compose keeps `TS_STATE_FILENAME=state.soc`; do not change it or the node loses its identity.
- Media libraries were always host bind mounts (`/mnt/samba_storage/jellyfin`) — untouched.

## Operations

Standard Makefile targets apply. Additional jellyfin-specific:

| Target | Description |
|--------|-------------|
| `jellyfin-migrate JELLYFIN_CONFIG_DIR=...` | Import old Jellyfin config (from Docker or apt-installed instance) into `./jellyfin/config/`. For apt, the source is `/etc/jellyfin` — stop the old service first (`sudo systemctl disable --now jellyfin`). |

## Gotchas

- Container paths must match what `jellyfin.xml` references. Host path == container path for media libraries (the migrated config points to absolute paths).
- If adding a new library folder, append `- /path/to/folder:/path/to/folder:ro` under volumes in compose and ensure the host dir exists before `make up`.
- `security_opt: label=disable` is required on rootless Podman for bind mounts that cross SELinux contexts (the Samba drive).
