# New Stack Guide

Step-by-step instructions for creating a new container stack on any host in the fleet, following the conventions.

## Prerequisites

- `podman` and `podman-compose` installed on the host
- Tailscale connected to your tailnet with admin access to create machines and auth keys
- Access to the GitHub repo (`java-alchemist`) for creating the new stack repository
- An existing stack already running (to copy patterns from — use `jellyfin-stack` as the reference template)

## Step 1: Create the Repository

```bash
# On your workstation or the target host
mkdir /opt/data/staging/<stack>-stack && cd /opt/data/staging/<stack>-stack
git init -b main
```

Create these files (see templates below):

- `README.md` — architecture, services table, secrets layout, ops commands
- `.env.example` — env var template with comments
- `podman-compose.yml` — services definition (app + tailscale + dozzle-agent + watchtower)
- `serve-config/serve.json` — Tailscale SERVE routes
- `setup.sh` — provisioning script (bootstrap + local-setup)
- `Makefile` — standard targets
- `shared-deps` — list of shared files this stack reads from `/opt/data/stack-env/`

When making changes to any stack, always update the **server-stack-wiki** in parallel: per-stack docs under `stacks/<stack>-name.md`, and conventions if the change affects fleet-wide patterns. The wiki is the authoritative knowledge base for how stacks work.

Then push to GitHub:
```bash
git add -A && git commit -m "initial: <app> stack skeleton"
gh repo create java-alchemist/<stack>-stack --public  # or --private
git push -u origin main
```

## Step 2: Write podman-compose.yml

Use `jellyfin-stack/podman-compose.yml` as the reference template. Every stack needs at minimum:

1. **The app container** — your actual service (label it for watchtower)
2. **Tailscale sidecar** — userspace networking + SERVE reverse proxy (no label, not managed by watchtower)
3. **Dozzle agent** — log collection (mTLS gRPC to hub, labeled for watchtower)
4. **Watchtower** — auto-updates labeled services hourly with rolling restart and rollback

```yaml
networks:
  <stack>-net:
    name: <stack>-net
    driver: bridge

services:
  app:
    image: <image>:<tag>
    container_name: <stack>-app
    labels:
      - "com.centurylinklabs.watchtower.enable=true"   # watchtower manages this
    environment:
      PUID: "0"           # rootless mode — uid 0 = host stack user
      PGID: "0"
    networks: [<stack>-net]
    volumes:
      - ./secrets/.<secret>_password:/run/secrets/<secret>_password:ro   # file-based secrets
      # ... app config bind mounts
    restart: unless-stopped

  tailscale:
    image: tailscale/tailscale:latest
    container_name: <stack>-tailscale
    environment:
      TS_AUTH_KEY_FILE: /run/secrets/tailscale_authkey
      TS_HOSTNAME: <app>                       # becomes <app>.<tailnet>.ts.net
      TS_STATE_FILENAME: state.soc
      TS_SERVE_CONFIG: /etc/tailscale/serve.json
    networks: [<stack>-net]
    volumes:
      - ./ts_state:/var/lib/tailscale           # persistent node identity
      - ./secrets/.tailscale-authkey:/run/secrets/tailscale_authkey:ro
      - ./serve-config/serve.json:/etc/tailscale/serve.json:ro
      - /dev/net/tun:/dev/net/tun
    cap_add: [NET_ADMIN]
    restart: unless-stopped

  dozzle-agent:
    image: amir20/dozzle:latest
    container_name: <stack>-dozzle-agent
    labels:
      - "com.centurylinklabs.watchtower.enable=true"   # watchtower manages this
    command: agent --addr :7007 --cert /run/secrets/cert.pem --key /run/secrets/key.pem
    environment:
      PODMAN_HOST: unix:///run/user/${PUID}/podman/podman.sock
    networks: [<stack>-net]
    volumes:
      - ${PODMAN_SOCKET}:/run/user/${PUID}/podman/podman.sock
      - ./secrets/.dozzle-cert.pem:/run/secrets/cert.pem:ro
      - ./secrets/.dozzle-key.pem:/run/secrets/key.pem:ro
    restart: unless-stopped

  # ── Stateful services (databases, etc.) — do NOT label for watchtower ──
  # ... databases, caches, etc.

  # ── Watchtower: auto-updates labeled services hourly ──
  watchtower:
    image: containrrr/watchtower:latest-ce
    container_name: <stack>-watchtower
    environment:
      WATCHTOWER_LABEL_ENABLE: "true"
      WATCHTOWER_POLL_INTERVAL: "3600"
      WATCHTOWER_ROLLING_RESTART: "true"
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_INCLUDE_STOPPED: "false"
    volumes:
      - ${PODMAN_SOCKET}:/var/run/docker.sock:ro
    restart: unless-stopped
```

**Image strategy:** Use floating tags (`latest`, `release`, `<major>-alpine`) for services watchtower manages. Pin by digest only for stateful services (databases) that cannot be upgraded automatically — schema migrations can't roll back safely. See [conventions §10](../conventions.md#10-watchtower-auto-update).

## Step 3: Write serve-config/serve.json

Routes HTTPS and raw TCP through the tailscale sidecar:

```json
{
  "<app>.<tailnet>.ts.net": {
    "443": {"httpsUpstream": "http://app:<INTERNAL_PORT>"}
  },
  "<app>-dozzle.<tailnet>.ts.net": {
    "7007": {"tcpUpstream": "dozzle-agent:7007"}
  }
}
```

- `:443` → app container (HTTPS, Tailscale handles TLS termination)
- `:7007` → dozzle-agent (raw TCP for mTLS gRPC log collection)

## Step 4: Write .env.example

Template for the generated `.env`. Every stack needs at minimum:

```bash
# ── Podman socket (set by setup.sh, do not change) ──
PODMAN_SOCKET=/run/user/$(id -u)/podman/podman.sock
PUID=0
PGID=0

# ── Tailscale ──
TS_HOSTNAME=<app>                     # node name on tailnet
```

Add app-specific variables with clear comments. Never put secrets here — they are generated by `setup.sh` into the real `.env`.

## Step 5: Write shared-deps

Declare which files this stack reads from `/opt/data/stack-env/`:

```
# Shared files from $STACK_ENV_DIR (default /opt/data/stack-env)
.tailscale-authkey
.dozzle-cert.pem
.dozzle-key.pem
```

If the stack generates its own secrets (e.g., a DB password), those go into `secrets/` during local-setup but are NOT listed in shared-deps.

## Step 6: Write setup.sh

Copy from `jellyfin-stack/setup.sh` as your template and customize these variables at the top:

```bash
STACK_NAME="<stack>-stack"
STACK_USER="<app>"
STACK_GROUP="<app>"
EXTERNAL_DATA_DIRS="/path/to/external/data/if/any"   # omit if none
```

The script must support these subcommands (called via `case "$1"`):
- `setup` — runs `_bootstrap` (root) then `local-setup` (stack user)
- `bootstrap` / `_bootstrap` — root-only host provisioning (user, group, linger, sudoers, ownership)
- `local-setup` — stack-user phase: generate `.env`, copy shared secrets from `$STACK_ENV_DIR/`, create dirs, register dozzle agent endpoint
- `teardown` / `local-teardown` — cleanup

Key things to customize in local-setup:
1. What secrets to generate (random passwords, API keys) and write to `secrets/.<name>` or `.env`
2. Which directories to create (config bind mounts, ts_state, etc.)
3. The dozzle agent registration line — use the app's Tailscale hostname

## Step 7: Write Makefile

Copy from `jellyfin-stack/Makefile`. Change these three variables at the top:

```makefile
STACK          := <app>         # short name for container prefix filter
COMPOSE        ?= podman-compose.yml
STACK_USER     := <app>         # system user
```

The rest of the Makefile is identical across stacks — all targets use `$(STACK)`, `$(COMPOSE)`, and `$(STACK_USER)` as parameters. Self-escalation guards are already in place.

For the `serve` target, customize to match your stack's container names:
```makefile
serve:
	@[ "$$(id -un)" = "$(STACK_USER)" ] || { ... sudo guard ... }; \
	$(ENV_LAYER) podman-compose -f $(COMPOSE) restart tailscale; \
	sleep 10; \
	podman logs --tail 300 <stack>-tailscale 2>&1 | grep -q "applying serve config" && echo "serve config applied" || { ... error ... }
```

## Step 8: Write README.md

Follow the `jellyfin-stack/README.md` format:

```markdown
# <app>-stack

<One sentence describing what this stack runs — rootless Podman, convention-aligned with dns/dozzle stacks.>

Owns Tailscale node **`<app>`** — UI at `https://<app>.<tailnet>.ts.net`.

## Services

| Container | Image | Role |
|---|---|---|
| `<stack>-tailscale` | ... | userspace tailscaled + SERVE (:443 → app, raw TCP :7007 → agent). Not managed by watchtower. |
| `<stack>-app` | ... | the main service; config as bind mounts, ... Managed by watchtower. |
| `<stack>-dozzle-agent` | amir20/dozzle:latest (`agent`) | reports containers to central Dozzle hub over mTLS gRPC on :7007. Managed by watchtower. |
| `<stack>-watchtower` | containrrr/watchtower:latest-ce | auto-updates labeled services hourly with rolling restart and rollback. |

## Secrets layout

All credentials are **files** under `secrets/` (700 dir, 600 files, owned by `<app>`) — never env vars in `.env`:

| File | Source | Used by |
|---|---|---|
| `.tailscale-authkey` | copied from `/opt/data/stack-env` by setup | tailscale sidecar |
| `.dozzle-cert.pem`, `.dozzle-key.pem` | copied from `/opt/data/stack-env` | dozzle-agent mTLS credential |
| ... | ... | ... |

## Setup

```sh
git clone git@github.com:java-alchemist/<stack>-stack.git /opt/data/<stack>-stack
cd /opt/data/<stack>-stack && make setup
make up
```

## Ops

(Standard Makefile targets table — copy from jellyfin-stack)
```

## Step 9: First Run

Commit everything and push to GitHub, then pull it onto the target host:

```bash
# On your workstation (or wherever you've been editing)
cd /opt/data/staging/<stack>-stack
git add -A && git commit -m "initial: <app> stack"
git push

# SSH to the target host and deploy
ssh <host>
cd /opt/data/ && mkdir -p <target-path>/ && mv staging/<stack>-stack <target-path>/<stack>-stack 2>/dev/null || true
cd <target-path>/<stack>-stack
make setup    # one sudo prompt for host bootstrap
make up       # starts the stack (pulls images, applies SERVE)
make serve    # re-apply SERVE routes if needed
```

Verify:
1. `make status` — all containers running
2. `https://<app>.<tailnet>.ts.net` loads in your browser
3. Dozzle hub at `https://dozzle.<tailnet>.ts.net` shows the new stack's containers

## Step 10: Post-Setup Checklist

- [ ] Stack user exists and has linger enabled (`loginctl show-user <app> -p Linger`)
- [ ] Operator can edit repo files without sudo (group `server-stack` grants write)
- [ ] `.env` is gitignored, `secrets/` is gitignored
- [ ] `ts_state/state.soc` exists after first tailscale auth (node identity persisted)
- [ ] Dozzle agent endpoint registered in `$STACK_ENV_DIR/dozzle-agents.d/<stack>`
- [ ] SERVE routes work: HTTPS on :443, dozzle-agent reachable on :7007
- [ ] `make validate` passes (compose config dry-run)
- [ ] README.md services table matches actual containers

## Agent Workflow for Changes

When an agent needs to modify a stack:

1. **Work locally:** Edit files in the repo working tree
2. **Validate:** Run `make validate` to check compose config before committing
3. **Commit and push:** `git add -A && git commit -m "<descriptive message>" && git push`
4. **Pull on the target server:** SSH to the host, navigate to the stack dir, run:
   ```bash
   cd /opt/data/<path>/<stack>-stack
   git pull
   make local-setup    # re-converge generated files if needed
   make upgrade        # recreate containers with new config (preserves state volumes)
   make serve          # re-apply SERVE routes if networking changed
   ```

Never commit secrets — they are generated by `setup.sh` and live in gitignored `secrets/`. If you need a new secret, add the generation step to `setup.sh local-setup` and run `make local-setup` on the target host.
