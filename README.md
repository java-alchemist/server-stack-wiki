# Server Stack Wiki

Living documentation for the container fleet — how each stack repo works, how they share state, and how to set up a new one.

## Fleet Overview (tva example)

The conventions below apply to any host running this fleet pattern. The table shows the current deployment on `tva` as an example.

| Stack | Repo | Host Path (on tva) | User | Tailscale Node | Purpose |
|-------|------|--------------------|------|----------------|---------|
| DNS | `dns-stack` | `/opt/data/home/dns-stack` | `dns` | `dns` | Pi-hole + Caddy (DNS/HTTPS infra) |
| Dozzle | `dozzle-stack` | `/opt/data/dozzle-stack` | `dozzle` | `dozzle` | Central log viewer hub |
| Jellyfin | `jellyfin-stack` | `/opt/data/jellyfin-stack` | `jellyfin` | `jellyfin` | Media streaming server |
| Immich | `immich-stack` | `/opt/data/immich-stack` | `immich` | `immich` | Self-hosted photo library |

All repos live under `java-alchemist` on GitHub. No host ports — everything is tailnet-only via Tailscale SERVE.

## Per-Stack Documentation

Deep dives into how each stack works, its services, data layout, secrets, and gotchas:

- **[dns-stack](stacks/dns-stack.md)** — Pi-hole DNS resolver + Caddy HTTPS reverse proxy
- **[dozzle-stack](stacks/dozzle-stack.md)** — Central container log viewer hub (mTLS gRPC)
- **[jellyfin-stack](stacks/jellyfin-stack.md)** — Media streaming server with VA-API transcoding
- **[immich-stack](stacks/immich-stack.md)** — Self-hosted photo library (PostgreSQL + Valkey)

## Conventions and Guides

- **[conventions.md](conventions.md)** — Shared patterns every stack follows
- **[deployment.md](deployment.md)** — How setup and teardown provision/deprovision stacks (user/group creation, ownership, sudoers, secrets)
- **[new-stack-guide.md](new-stack-guide.md)** — Step-by-step to create a new container stack from scratch

## 3rd-Party Documentation

Backed-up vendor docs for each stack's services live under `docs/<stack>-name/` in this wiki repo. Stack repos themselves stay lean (compose, Makefile, setup script only).
