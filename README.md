# Server Stack Wiki

Living documentation for the tva container fleet — how each stack works, how they share state, and how to set up a new one.

## Fleet Overview

| Stack | Repo | Host Path | User | Tailscale Node | Purpose |
|-------|------|-----------|------|----------------|---------|
| DNS | `dns-stack` | `/opt/data/home/dns-stack` | `dns` | `dns` | Pi-hole + Caddy (DNS/HTTPS infra) |
| Dozzle | `dozzle-stack` | `/opt/data/dozzle-stack` | `dozzle` | `dozzle` | Central log viewer hub |
| Jellyfin | `jellyfin-stack` | `/opt/data/staging/jellyfin-stack` | `jellyfin` | `jellyfin` | Media streaming server |
| Immich | `immich-stack` | `/opt/data/staging/immich-stack` | `immich` | `immich` | Self-hosted photo library |

All repos live under `java-alchemist` on GitHub. No host ports — everything is tailnet-only via Tailscale SERVE.

## Docs

- **[conventions.md](conventions.md)** — Shared patterns every stack follows
- **[new-stack-guide.md](new-stack-guide.md)** — Step-by-step to create a new container stack from scratch
