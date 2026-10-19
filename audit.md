# Stack Audit — Convention Adherence

Findings from reviewing all four active stack repos against the wiki conventions. Grouped by severity.

## dns-stack (`/opt/data/home/dns-stack`)

### ✗ Must fix

- **No README.md** — Every repo must have one with services table, secrets layout, and ops commands per convention §2

### ⚠ Should fix

- **Tailscale image unpinned** — Uses `tailscale:latest`. Convention pinning note (jellyfin-stack comment): "verify before bumping as Tailscale skips some patch versions." Should be a version tag like `v1.102.5`
- **Stack group name includes `-stack` suffix** — `STACK_GROUP="dns-stack"` vs `dns`. Jellyfin/immich use the short name (`jellyfin`, `immich`). Inconsistent with the other two newer stacks
- **User shell is `/usr/sbin/nologin`** — Jellyfin/immich use `/bin/bash`. Convention says "nologin" for defense-in-depth but both approaches work. If standardizing, pick one and update conventions

## dozzle-stack (`/opt/data/dozzle-stack`)

### ✗ Must fix

- **Tailscale image unpinned** — Uses `tailscale:latest`. Same issue as dns-stack

### ⚠ Should fix

- **Stack group name includes `-stack` suffix** — `STACK_GROUP="dozzle-stack"` vs `dozzle`. Same inconsistency as dns-stack
- **User shell is `/usr/sbin/nologin`** — Same discrepancy as dns-stack vs jellyfin/immich

## jellyfin-stack (`/opt/data/staging/jellyfin-stack`)

### ✗ Must fix

- **No `shared-deps` file** — Convention §4 requires this to declare which files are read from `$STACK_ENV_DIR/`. The setup.sh copies them anyway, but the declaration is missing so there's no manifest of what it depends on

### ⚠ Should fix

- **No systemd autostart** — No `install-service`/`enable-service` targets or `systemd/` dir
- **Does not explicitly create STACK_GROUP** — Sets `STACK_GROUP="jellyfin"` but does not call `groupadd`. Relies on the implicit primary group created by `useradd -m`. Teardown tries to `groupdel` it, which works because useradd creates a matching group. Works today but is fragile if someone changes the useradd flags

## immich-stack (`/opt/data/staging/immich-stack`)

### ⚠ Should fix

- **No systemd autostart** — No `install-service`/`enable-service` targets or `systemd/` dir
- **Does not explicitly create STACK_GROUP** — Same as jellyfin-stack (relies on implicit primary group)

## Cross-Cutting Issues

| Issue | Affected | Convention | Severity |
|-------|----------|------------|----------|
| **No systemd autostart units** | All 4 stacks | Convention §3: "Autostart via systemd unit (`User=<stack>`)" | ⚠ — current stacks are managed manually or by other means, but the convention says they should have it |
| **Tailscale image unpinned** | dns-stack, dozzle-stack | Jellyfin/immich pin to `v1.102.5` with a comment about verifying versions. dns/dozzle use `:latest` which pulls silently on upgrade | ⚠ |
| **Inconsistent STACK_GROUP naming** | dns (`dns-stack`) vs jellyfin (`jellyfin`) | Convention doesn't specify short name — should standardize to the short stack name without `-stack` suffix for consistency | Minor |
| **Inconsistent user shell** | dns/dozzle: `/usr/sbin/nologin` vs jellyfin/immich: `/bin/bash` | Convention says "nologin" in §3 table but two stacks use bash. Should pick one and document it | Minor |

## Wiki Corrections Needed

The following convention text needs updating based on what the actual repos do:

1. **SERVE config format** — dns-stack uses `TCPForward` while jellyfin/immich use a mix of `HTTPS` + `Proxy` handlers. The conventions §5 example should reflect the actual pattern used (which varies by stack)
2. **Tailscale sidecar image path** — Conventions say `tailscale/tailscale:v1.102.5` but dns/dozzle use `docker.io/tailscale/tailscale:latest`. Should standardize to pinned version with full registry prefix
