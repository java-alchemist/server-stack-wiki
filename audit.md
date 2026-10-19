# Stack Audit — Convention Adherence

Findings from reviewing all four active stack repos against the wiki conventions. Grouped by severity.

## dns-stack (`/opt/data/home/dns-stack`)

### ✗ Must fix

- **No README.md** — Every repo must have one with services table, secrets layout, and ops commands per convention §2

### ⚠ Should fix

*(Fixed)* Tailscale image pinned to `v1.102.5` — [`ccb0ce8`](https://github.com/java-alchemist/dns-stack/commit/ccb0ce8)
- **Stack group name includes `-stack` suffix** — `STACK_GROUP="dns-stack"` vs `dns`. Jellyfin/immich use the short name (`jellyfin`, `immich`). Inconsistent with the other two newer stacks
- **User shell is `/usr/sbin/nologin`** — Jellyfin/immich use `/bin/bash`. Convention says "nologin" for defense-in-depth but both approaches work. If standardizing, pick one and update conventions

## dozzle-stack (`/opt/data/dozzle-stack`)

### ⚠ Should fix

*(Fixed)* Tailscale image pinned to `v1.102.5` — [`d1d00fc`](https://github.com/java-alchemist/dozzle-stack/commit/d1d00fc)
- **Stack group name includes `-stack` suffix** — `STACK_GROUP="dozzle-stack"` vs `dozzle`. Same inconsistency as dns-stack
- **User shell is `/usr/sbin/nologin`** — Same discrepancy as dns-stack vs jellyfin/immich

## jellyfin-stack (`/opt/data/staging/jellyfin-stack`)

### ⚠ Should fix

*(Fixed)* Systemd autostart added (`install-service`/`enable-service` targets + `systemd/jellyfin-stack.service`) — [`e6facd5`](https://github.com/java-alchemist/jellyfin-stack/commit/e6facd5)
*(Fixed)* STACK_GROUP explicitly created via `groupadd` in bootstrap — same commit

## immich-stack (`/opt/data/staging/immich-stack`)

### ⚠ Should fix

*(Fixed)* Systemd autostart added — [`effa359`](https://github.com/java-alchemist/immich-stack/commit/effa359)
*(Fixed)* STACK_GROUP explicitly created via `groupadd` in bootstrap — same commit

## Cross-Cutting Issues

| Issue | Affected | Convention | Status |
|-------|----------|------------|--------|
| **Tailscale image unpinned** | dns-stack, dozzle-stack | Jellyfin/immich pin to `v1.102.5` | ✅ Fixed in both repos |
| **Inconsistent STACK_GROUP naming** | dns (`dns-stack`) vs jellyfin (`jellyfin`) | Convention doesn't specify short name — should standardize to the short stack name without `-stack` suffix for consistency | Minor |
| **Inconsistent user shell** | dns/dozzle: `/usr/sbin/nologin` vs jellyfin/immich: `/bin/bash` | Convention says "nologin" in §3 table but two stacks use bash. Should pick one and document it | Minor |

## Wiki Corrections Needed

The following convention text needs updating based on what the actual repos do:

1. **SERVE config format** — dns-stack uses `TCPForward` while jellyfin/immich use a mix of `HTTPS` + `Proxy` handlers. The conventions §5 example should reflect the actual pattern used (which varies by stack)
2. **Tailscale sidecar image path** — Conventions say `tailscale/tailscale:v1.102.5` but dns/dozzle use `docker.io/tailscale/tailscale:latest`. Should standardize to pinned version with full registry prefix
