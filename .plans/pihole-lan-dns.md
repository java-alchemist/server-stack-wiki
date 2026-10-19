# Plan: Make Pi-hole DNS available on the LAN for Unifi controller

## Context

Pi-hole runs in a rootless Podman container on `tva` (192.168.1.82) and is currently bound to `127.0.0.1:5053`. Rootless Podman cannot bind below port 1024, so we can't publish DNS directly on host port 53 from the container. The Unifi controller (on another router) needs a standard DNS server at `192.168.1.82:53` to use Pi-hole for ad/tracker blocking across the LAN.

## Architecture After Change

```
LAN clients ──→ 192.168.1.82:53 (host dnsmasq forwarder) ──→ 127.0.0.1:5053 (Pi-hole container)
Tailnet clients ──→ dns.<tailnet>.ts.net:53 (Tailscale MagicDNS) ──→ Pi-hole container
Unifi controller ──→ 192.168.1.82:53 (host dnsmasq forwarder) ──→ Pi-hole
```

## Changes Required

### 1. Add a host-side dnsmasq DNS forwarder (root service)

Install and configure `dnsmasq` on the host to listen on all interfaces port 53, forwarding everything to Pi-hole on loopback:5053.

**File:** `/etc/dnsmasq.d/lan-forward`
```
# Forward all DNS queries to Pi-hole running in rootless Podman container
listen-address=127.0.0.1, 192.168.1.82
no-resolv                              # don't read /etc/resolv.conf for upstreams
server=127.0.0.1#5053                  # forward to Pi-hole container
cache-size=1000                        # small cache since Pi-hole already caches
log-queries                            # debug logging (disable after verifying)
```

Then enable `systemd-resolved` to not conflict:
```bash
# Ensure systemd-resolved doesn't hold port 53 — it uses 127.0.0.53 which is fine
# but we need to make sure dnsmasq gets the LAN interface binding
sudo systemctl enable --now dnsmasq
```

**Why not just change the compose port mapping?** We could add a second bind like `192.168.1.82:5353:53` — but Unifi controller needs standard port 53, and rootless Podman can't publish below 1024. The dnsmasq forwarder bridges that gap at the host level where it has permission to bind `:53`.

### 2. Update `setup.sh` in dns-stack (bootstrap section)

Add a bootstrap step that installs and configures the LAN DNS forwarder so new deployments get it automatically. In `_bootstrap()`:

```bash
# Install dnsmasq LAN forwarder (bridges host :53 → loopback :5053 for rootless Podman)
sudo apt-get install -y dnsmasq || sudo yum install -y dnsmasq  # distro-aware
sudo tee /etc/dnsmasq.d/lan-forward > /dev/null << 'EOF'
listen-address=127.0.0.1, $(ip -4 addr show eno1 | grep inet | awk '{print $2}' | cut -d/ -f1)
no-resolv
server=127.0.0.1#5053
cache-size=1000
EOF
sudo systemctl enable --now dnsmasq
```

Add a matching teardown step to disable and remove the forwarder on `make teardown`.

### 3. Update `.env.example` / compose comments

Document the port bridging in the Pi-hole service block so future readers understand why two ports exist:

```yaml
ports:
  # Rootless Podman can't bind below 1024 — DNS published on high port loopback only.
  # A host-side dnsmasq forwarder bridges :53 (host) → :5053 (loopback) for LAN access.
  - "127.0.0.1:${PIHOLE_DNS_PORT:-5053}:53/udp"
  - "127.0.0.1:${PIHOLE_DNS_PORT:-5053}:53/tcp"
```

### 4. Update wiki docs (`server-stack-wiki`)

- **`stacks/dns-stack.md`** — Add a "LAN DNS Access" section documenting the dnsmasq forwarder, its purpose, and how to configure Unifi (point controller at `192.168.1.82`).
- **`conventions.md` §5 Networking** — Note that dns-stack is special: it has a host-side dnsmasq forwarder bridging port 53 → 5053 for rootless Podman compatibility.

## Unifi Configuration (on the router)

Set the DNS server in Unifi network settings to `192.168.1.82`. The controller will pass this as DHCP option 6 to LAN clients, so everything resolves through Pi-hole automatically.

In Unifi Controller → Settings → Networks → [your network] → DHCP Server:
- **DNS 1:** `192.168.1.82` (Pi-hole — primary)
- **DNS 2:** `1.1.1.1` (Cloudflare — fallback if Pi-hole is down)

## Risks / Considerations

| Concern | Mitigation |
|---------|-----------|
| dnsmasq conflicts with systemd-resolved | Handled automatically — setup.sh disables `DNSStubListener` in `/etc/systemd/resolved.conf`. The conflict occurs because dnsmasq's systemd wrapper adds `--local-service` (binds all of `127.0.0.0/8`) which overlaps with systemd-resolved's stub listener on `127.0.0.53:53`. |
| Pi-hole down → all DNS fails | Handled at the Unifi level — set DNS 2 to `1.1.1.1` (Cloudflare) as backup. Clients fall back automatically if Pi-hole doesn't respond. No fallback needed in dnsmasq config. |
| Host firewall blocks :53 on LAN interface | Ensure UFW/iptables allows UDP+TCP 53 on the LAN interface (`eno1`). Most home setups don't have host firewalls enabled, but worth verifying. |
| Tailscale MagicDNS still works? | Yes — Tailscale routes DNS through its own resolver chain regardless of what's on port 53. Pi-hole is reachable via tailnet IP:5053 or the container bridge network. |

## Rollback Plan

If anything goes wrong, stop dnsmasq and everything returns to current state (Pi-hole only on loopback :5053):
```bash
sudo systemctl disable --now dnsmasq
sudo rm /etc/dnsmasq.d/lan-forward
```
