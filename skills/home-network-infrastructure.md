---
name: home-network-infrastructure
description: Use when building or maintaining DNS/HTTPS infrastructure for home server stacks — Pi-hole + Caddy on rootless Podman with Tailscale networking, .home.arpa DNS records, and offline-resilient HTTPS via Let's Encrypt.
---

# Home Network Infrastructure (DNS + HTTPS)

Design and maintain the dns-stack that provides hostname-based DNS resolution and HTTPS reverse proxy for all home services — works over Tailscale AND local LAN without internet connectivity after initial cert issuance.

## Architecture

**Three containers on a rootless Podman bridge network:**

| Container | Purpose | Ports (rootless) |
|---|---|---|
| **tailscale** | Sidecar, userspace mode (`TS_USERSPACE=true`), gives stack its Tailscale IP | none — owns the namespace |
| **pihole** | DNS resolver + `.home.arpa` custom records; runs with **host networking** so FTL binds :53 directly and sees real client source IPs (port publishing would NAT everything through slirp4netns) | host netns `:53`, web UI `:10053` all interfaces |
| **caddy** | Plain-HTTP reverse proxy (`auto_https off` — LE can't issue for `.home.arpa`; tailnet clients use per-app ts.net URLs where Tailscale owns TLS), routes by hostname | `80:80`, `8080:80` fallback |

## Why `.home.arpa` not `.local`?

- **`.local` is reserved for mDNS** — OS intercepts these queries before they reach Pi-hole. On many devices (especially macOS/iOS), `.local` triggers multicast resolution that competes with DNS, causing slow or failed lookups.
- **`home.arpa`** is IANA-reserved specifically for private home networks. Goes straight to your DNS resolver every time, on all platforms. Zero collision risk with public domains.

## DNS Resolution Flow

```
Device → Pi-hole (via DHCP from router, e.g., UniFi Travel Router)
       → resolves jellyfin.home.arpa → Tailscale IP of the node running Jellyfin
       → connects to that IP :443 → Caddy routes by Host header → backend service
```

### dnsmasq config (`pihole/dnsmasq.d/99-custom-dns.conf`):

```conf
# Each record points to the Tailscale IP of the node running that service.
# Get IPs with: tailscale ip -4 <hostname>
address=/jellyfin.home.arpa/100.X.X.X
address=/hermes.home.arpa/100.X.X.X
```

### Caddyfile (`caddy/Caddyfile`) — per-site ACME issuers are MANDATORY

```caddyfile
{
    # Trust anchor for the ACME channel (step-ca serves its API over HTTPS with a
    # self-signed cert, SAN=home.arpa). File is fetched into ./pki/ by make local-setup.
    acme_ca_root /etc/caddy/pki/root.crt
}
jellyfin.home.arpa {
    tls {                       # MUST be multi-line — 'tls { ca ... }' one-liner is a parse error
        ca https://home.arpa:9000/acme/caddy/directory
    }
    reverse_proxy https://jellyfin.<tailnet-domain>.ts.net
}
ca.home.arpa:80 {               # explicit :80 key = HTTP-only site (bootstrap install page)
    reverse_proxy https://ca.<tailnet-domain>.ts.net
}
```

**Why per-site, not global `acme_ca`:** certmagic classifies `*.home.arpa` as an INTERNAL name (`SubjectIsInternal` in caddyserver/certmagic), and Caddy's Caddyfile adapter assigns its own self-signed local CA to internal names whenever a site has no explicit issuer — so a global `acme_ca` option NEVER applies to `.home.arpa` sites (docs: automatic-https → "Hostname requirements"). Symptom when wrong: sites serve an untrusted "Caddy Local Authority" cert and the home CA is never contacted.

**Caddyfile syntax traps that crash-loop dns-caddy silently** (process dies ~250ms after "using config from file", often with no error line in podman logs):
- `tls { ca <url> }` on one line → `Unexpected next token after '{' on same line`. Block contents must start on the next line.
- There is NO per-site `tls off` (single arg must be `internal`, `force_automate`, or an email) and no `protocol http` directive. HTTP-only sites use a `host:80` site key.
- Always verify with `caddy validate --config <file>` before deploying (docker run the caddy image; pipe files in via tar|stdin — bind mounts of scratch paths may not be visible to the daemon).

**TLS comes from ca-stack** (separate stack, step-ca): ACME directory at `https://home.arpa:9000/acme/<provisioner>/directory` (step-ca v0.30.x serves TLS on :9000; URL host must be home.arpa = SAN of its API cert). Issuance is hard-restricted to an exact-name allowlist in ca-stack's `ca/ca.json` (`authority.policy.x509.allow.dns`). HTTP-01 validation loops through Pi-hole + Caddy :80, so FTL hosts records for the app name AND apex `home.arpa` (→ tva LAN IP) must exist BEFORE first issuance — on Pi-hole v6 via `FTLCONF_dns_hosts` in .env (`make local-setup` rebuilds it from live tailnet state), NOT the legacy dnsmasq.d mount. Devices enroll trust from the install page at `https://ca.<tailnet>.ts.net/`.

**Backends must be reachable addresses, not bare service names.** `jellyfin` only resolves inside jellyfin-stack's bridge network — from dns-net it is a dead name (502). Target the stack's **host-published port** via `host.containers.internal:<port>` (resolves to tva's LAN IP under rootless; self-heals if the IP changes), which requires each app stack to publish that port on the host. Apps on *other machines* can't be fronted this way — they keep their own ts.net :443 URLs.

## Secrets Pattern

**All secrets are file-mounted, never env vars.** Follows the Zero-Visibility pattern.

- `./secrets/.pihole_password` → mounted at `/run/secrets/pihole_password`, loaded via custom entrypoint that sets `WEBPASSWORD`
- `${STACK_ENV_DIR}/.tailscale-authkey` → shared across all stacks, NEVER overwritten by setup.sh

### Pi-hole entrypoint (reads secret file):

```yaml
entrypoint: ["/bin/sh", "-c"]
command: ["if [ -f /run/secrets/pihole_password ]; then export WEBPASSWORD=$(cat /run/secrets/pihole_password); fi; exec /container/start.py"]
volumes:
  - type: bind
    source: ./secrets/.pihole_password
    target: /run/secrets/pihole_password
    read_only: true
```

## Setup Script (`setup.sh`)

Generates missing secrets, validates shared ones, creates directory structure. **Key rules:**

1. Generates `secrets/.pihole_password` (random 24-byte base64) only if file doesn't exist
2. Validates `${STACK_ENV_DIR}/.tailscale-authkey` exists — does NOT create or overwrite it
3. Creates `.env` from `.env.example` only if missing
4. Creates `ts_state/`, `pihole/dnsmasq.d/`, `caddy/` directories

## Offline Resilience

- **Let's Encrypt certs** are valid 90 days and auto-renew when internet returns
- Browsers validate against built-in trust stores — no outbound connection needed for HTTPS connections once the cert is issued
- Pi-hole resolves `.home.arpa` records locally — no upstream DNS needed for internal names

## Rootless Constraints

**Low ports ARE bindable here**: setup persists `net.ipv4.ip_unprivileged_port_start=53` (`/etc/sysctl.d/99-pihole.conf`), so rootless containers can publish :53, :80, etc. Caddy uses host `:80` for this reason (LAN clients get port-less URLs like `http://jellyfin.home.arpa`).
Pi-hole avoids publishing entirely — **host networking** gives FTL a direct :53 bind plus real client IPs in the dashboard.

## Adding a New Service

1. Add the name to ca-stack's allowlist (`ca/ca.json` → `authority.policy.x509.allow.dns`) — step-ca refuses names outside it.
2. Add FTL hosts records (Pi-hole v6): `<name>.home.arpa` → tva LAN IP in dns-stack `.env` `DNS_HOSTS_RECORDS` (rebuild via `make local-setup`).
3. Add a Caddy route WITH the per-site tls block:
   ```caddyfile
   newservice.home.arpa {
       tls {
           ca https://home.arpa:9000/acme/caddy/directory
       }
       reverse_proxy https://newservice.<tailnet-domain>.ts.net
   }
   ```
4. Validate (`caddy validate`) before `make up`.

## Stack Location

`/opt/data/home/dns-stack/` on tva (managed via WSL). Independent of turnstone-stack — separate repo, separate user.