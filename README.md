# Server Stack Wiki

Living documentation for the container fleet — how each stack repo works, how they share state, and how to set up a new one.

## Fleet Overview

The conventions below apply to any host running this fleet pattern. Two hosts are currently deployed.

### tva

| Stack | Repo | Host Path (on tva) | User | Tailscale Node | Purpose |
|-------|------|--------------------|------|----------------|---------|
| DNS | `dns-stack` | `/opt/data/home/dns-stack` | `dns` | `dns` | Pi-hole + Caddy (DNS/HTTPS infra) |
| Dozzle | `dozzle-stack` | `/opt/data/dozzle-stack` | `dozzle` | `dozzle` | Central log viewer hub |
| Home Assistant | `ha-stack` | `/opt/data/ha-stack` | `ha` | `ha` | Smart-home hub (HA Core + ZHA Zigbee) |
| Jellyfin | `jellyfin-stack` | `/opt/data/jellyfin-stack` | `jellyfin` | `jellyfin` | Media streaming server |
| Immich | `immich-stack` | `/opt/data/immich-stack` | `immich` | `immich` | Self-hosted photo library |
| Synapse | `synapse-stack` | `/opt/data/synapse-stack` | `synapse` | `synapse` | Matrix homeserver + Element Web (chat/federation) |
| Authentik | `authentik-stack` | `/opt/data/authentik-stack` | `authentik` | `authentik` | Identity provider — users, groups, SSO/OIDC |
| CA | `ca-stack` | `/opt/data/ca-stack` | `ca` | `ca` | Home certificate authority (step-ca) + cert-install page |
| Server env | `server-stack` | `/opt/data/server-stack` | — (operator-driven) | — | Owns the shared `/opt/data/stack-env`; no containers |

### freya (Kinoite 45 — Ryzen 9800X3D, Radeon AI PRO R9700)

| Stack | Repo | Host Path (on freya) | User | Tailscale Node | Purpose |
|-------|------|----------------------|------|----------------|---------|
| Radiance | `radiance-stack` | `/opt/data/radiance-stack` | `radiance` | `radiance` | Qwen3.8-Flash-Next LLM inference (OpenAI-compatible API) on the R9700 |
| Server env | `server-stack` | `/opt/data/server-stack` | — (operator-driven) | — | Owns the shared `/opt/data/stack-env`; no containers |

freya is a live tailnet node: `https://radiance.tail6a7569.ts.net` serves (the sidecar's SERVE route and `make test` health checks pass).

All repos live under `java-alchemist` on GitHub. No host ports — everything is tailnet-only via Tailscale SERVE (loopback bindings for operator tooling excepted).

## Per-Stack Documentation

Deep dives into how each stack works, its services, data layout, secrets, and gotchas:

- **[dns-stack](stacks/dns-stack.md)** — Pi-hole DNS resolver + Caddy HTTPS reverse proxy
- **[dozzle-stack](stacks/dozzle-stack.md)** — Central container log viewer hub (mTLS gRPC)
- **[ha-stack](stacks/ha-stack.md)** — Home Assistant Core + ZHA; USB device passthrough via `make add-device` convention
- **[jellyfin-stack](stacks/jellyfin-stack.md)** — Media streaming server with VA-API transcoding
- **[immich-stack](stacks/immich-stack.md)** — Self-hosted photo library (PostgreSQL + Valkey)
- **[radiance-stack](stacks/radiance-stack.md)** — Qwen3.8-Flash-Next LLM inference (AMD ROCm) with tailnet SERVE access
- **[synapse-stack](stacks/synapse-stack.md)** — Matrix homeserver (Synapse) + Element Web, federation over SERVE :8448

## Conventions and Guides

- **[conventions.md](conventions.md)** — Shared patterns every stack follows
- **[deployment.md](deployment.md)** — How setup and teardown provision/deprovision stacks (user/group creation, ownership, sudoers, secrets)
- **[new-stack-guide.md](new-stack-guide.md)** — Step-by-step to create a new container stack from scratch
- **[review-2026-11.md](review-2026-11.md)** — Full-fleet review, 2026-11-27 (per-stack findings + cross-cutting items; supersedes [audit.md](audit.md))
- **[kanban.md](kanban.md)** — Tracking projects with Hermes' built-in kanban board (CLI, boards, dependencies, agent-dispatched tasks)
- **[Daedalus design](.plans/daedalus.md)** — Fleet control plane tool (Kotlin/GraalVM; CLI/TUI/GUI/MCP over one core); machine-readable fleet definition in [fleet.yaml](fleet.yaml)

## 3rd-Party Documentation

Backed-up vendor docs for each stack's services live under `docs/<stack>-name/` in this wiki repo. Stack repos themselves stay lean (compose, Makefile, setup script only).
