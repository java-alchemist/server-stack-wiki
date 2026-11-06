# ha-stack

Home Assistant Core in rootless Podman. Own Tailscale node **`ha`**; UI at `https://ha.<tailnet>.ts.net`. Repo: `github.com/java-alchemist/ha-stack`, deployed on tva at `/opt/data/ha-stack`.

## Role in the Fleet

Self-hosted smart-home hub (ZHA for Zigbee via a SONOFF Dongle Plus MG24). HA Core is **stateful** — versioned config dir with forward-only migrations — so it is deliberately NOT watchtower-managed: upgrades are manual (`make upgrade` after bumping `HA_VERSION`).

## Setup

```bash
git clone <repo> && cd ha-stack
make setup     # one sudo prompt (bootstrap) → per-stack phase as user 'ha'
make up        # start the stack (self-heals rendered compose + secrets first)
make serve     # re-apply tailscale serve config if it ever drifts
```

`make setup` converges host prerequisites (user/group, subuid/subgid, linger, sudoers rule, **repo ownership**) and then runs `local-setup` as the stack user. Re-run it any time — it is idempotent. In particular it re-converges two-tier repo ownership: operator owns tracked files, group = stack user with group-write (2775), so `local-setup` can create runtime state at the repo root (rendered compose, `devices/`).

## Services

| Container | Image | Role |
|---|---|---|
| `ha-tailscale` | `tailscale/tailscale:latest` | Userspace tailscaled + SERVE (:443 → ha-app, raw TCP :7007 → dozzle-agent). `TS_USERSPACE: "true"` — kernel TUN mode is not reliable in rootless netns. Not watchtower-managed. |
| `ha-app` | `homeassistant/home-assistant:${HA_VERSION}` | HA Core. Config bind-mounted at `./ha/config`. `stop_grace_period: 60s` — vendor requirement, HA needs >10s to close its SQLite DB cleanly on SIGTERM or the next boot logs dirty-shutdown warnings. Pinned via `.env`; no watchtower label (stateful). |
| `ha-dozzle-agent` | `amir20/dozzle:latest` (`agent`) | Reports this stack's containers to the central Dozzle hub over mTLS gRPC on :7007. Reads the stack user's rootless podman socket. |

No host ports — everything is exposed through Tailscale SERVE only. No watchtower service by design (see Role).

## Data

| Data | Location | Type |
|------|----------|------|
| Entire HA state | `./ha/config/` bind mount | `.storage` SQLite DBs, automations, scripts, `secrets.yaml`, user accounts — back up this directory; restoring it restores everything. Pre-created 750 by local-setup. |
| Tailscale node identity | `./ts_state/state.soc` bind mount | Persistent identity — DO NOT delete (losing it = new tailnet hostname). |

## Secrets

| File | Source | Used By |
|---|---|---|
| `.tailscale-authkey` | Copied from `$STACK_ENV_DIR` by setup (per-stack key copy, owned by `ha`) | Tailscale sidecar auth (entrypoint reads it into `TS_AUTH_KEY`) |
| `.dozzle-cert.pem`, `.dozzle-key.pem` | Copied from `$STACK_ENV_DIR` (generated once by dozzle-stack) | Dozzle agent mTLS credential — THE credential for agent auth |

No plaintext env vars for credentials; secrets are mounted as files under `./secrets/`.

## SERVE Config

```json
{
  "Web": { "${TS_CERT_DOMAIN}:443": { "Handlers": { "/": { "Proxy": "http://ha-app:8123" } } } },
  "TCP": { "443": { "HTTPS": true }, "7007": { "TCPForward": "ha-dozzle-agent:7007" } }
}
```

Routes `https://ha.<tailnet>.ts.net` → ha-app on port 8123. Raw TCP :7007 forwards to the dozzle agent — without it, dials to `<node>.ts.net:7007` hit a dead end (userspace tailscaled only delivers traffic it forwards).

## Operations

Standard Makefile targets apply (`up`, `down`, `restart`, `upgrade`, `serve`, `status`, `logs`, `validate`). HA-specific upgrade flow — no watchtower, deliberate bumps:

```bash
# in .env: bump HA_VERSION, then
make upgrade    # pull → down (dep order) → up
```

### USB device passthrough (add-device convention)

Host-side access for USB devices (Zigbee dongles, DVB antennas, …) is managed by **generated** udev rules — never hand-written. Two sides, one source of truth per side:

| Side | Where | Managed by |
|---|---|---|
| Host rule → stable `/dev/<name>` owned by `ha` | `/etc/udev/rules.d/90-ha-<name>.rules` (rendered from `udev/device.rules.template`) | `make add-device SRC=<node> NAME=<name>` / `remove-device NAME=<name>` (one sudo prompt each) |
| Container passthrough declaration | `./devices/<name>` — **gitignored**, one file per device, filename = `/dev/<name>`, content ignored | same targets (they create/remove the file), or plain `touch`/`rm` + re-render (no sudo) |
| Container passthrough itself | `devices:` block in rendered `podman-compose.yml` (gitignored, generated from `podman-compose.template.yml`) | automatic — re-rendered by every `make local-setup` / `add-device` / `remove-device`; `make up` self-heals a missing render via `local-setup` |

Example — the SONOFF Dongle Plus MG24 (Zigbee 3.0 + Thread) for ZHA. Note: it is a **Silicon Labs EFR32MG24** — in the ZHA flow pick Radio Type **`ezsp`**. Its USB ID `10C4:EA60` only identifies the CP210x bridge chip and is shared with TI CC2652P dongles (which use `znp`) — never infer the protocol from VID:PID; check the model's SoC.

```bash
make list-devices                              # find the dongle's node (here: /dev/ttyUSB0)
make add-device SRC=/dev/ttyUSB0 NAME=sonoff-pmg24   # udev rule + devices/ declaration + re-render, one step
make down && make up
make probe-device                            # verify each declared device is alive INSIDE ha-app (open OK vs ENODEV stale bind vs permission)
# HA UI: Settings → Devices & Services → Add Integration → ZHA → port /dev/sonoff-pmg24
```

To drop a passthrough without touching the host (e.g. temporarily): `rm devices/<name> && make local-setup`. To remove everything: `make remove-device NAME=<name>` (one sudo).

Conventions and guarantees:

- **Stable names:** ttyUSB/dvb numbers reshuffle with enumeration order; the symlink doesn't. Rules match by USB id (resolved from sysfs at install time), never by node number.
- **Ownership, not group access:** rootless podman bind-mounts device nodes into containers (no CAP_MKNOD), so host ownership carries through — uid 0 in-container *is* `ha` on the host; foreign GIDs can't be mapped into the userns. That's why rules set `OWNER=` instead of `GROUP=`+`MODE=`.
- **Fail-loud:** a declared-but-unplugged device makes podman refuse to start ha-app (`stat /dev/<name>: no such file or directory`). Replug + `make up`. `local-setup` warns about missing declared devices; the podman error is the hard gate.
- **Declaration names are validated** (lowercase alnum/dash) at render time — a typo fails loud instead of silently missing from the container. Hidden files in `devices/` are ignored.
- **Multi-node USB devices** (e.g. dual-tuner DVB sticks exposing dvr0+video0) are not supported by `add-device` yet — the USB-id match would claim both nodes for one name; hand-write those rules with per-node KERNEL matches if needed.

## Gotchas

- **No mDNS/SSDP broadcast discovery** through the rootless slirp4netns bridge — integrations relying on auto-discovery (Chromecast, some Zigbee/Thread flows) need manual configuration. Property of the fleet's no-host-port model, not this stack.
- The MG24's **Thread** radio is not usable with HA Core (border routing is an HAOS-only feature) — this stack uses the Zigbee side via ZHA.
- **Bluetooth integration** needs `/run/dbus:/run/dbus:ro` mounted into ha-app — omitted by default (tva has no BT hardware); add it to the template if that changes.
- `security_opt: label=disable` is required on rootless Podman for bind mounts crossing SELinux contexts.
- HA 2026.8+ migrated `http:` settings into UI-managed `.storage/http`; local-setup strips any leftover seeded `http:` block from `configuration.yaml` and patches `use_x_forwarded_for` + trusted proxies (the ha-net subnet) into `.storage/http`. Don't re-add an `http:` block — it raises a permanent UI warning.
- Repo ownership is two-tier (operator-owned, group = stack user with g+w). If the tree ever ends up operator-only 755 or missing the group-write bit, `make setup` detects it and re-converges via bootstrap (one sudo prompt) — don't hand-chown.
