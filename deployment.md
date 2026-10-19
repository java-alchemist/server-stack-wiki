# Deployment: Setup and Teardown

How `make setup` and `make teardown` provision and deprovision a stack on a host. Every stack's `setup.sh` follows this flow — idempotent, convergent, one sudo prompt for the operator.

## Quick Reference

```bash
# Deploy (first time or re-converge)
cd /path/to/<stack>-stack
make setup           # one sudo prompt if host prerequisites are missing

# Day-to-day ops (passwordless via installed sudoers rule)
make up              # start the stack
make upgrade         # pull latest images, recreate containers
make local-setup     # re-converge generated files after git pull

# Remove everything
make teardown        # removes containers, volumes, secrets, user, group, sudoers
```

## Setup Flow (`make setup`)

The entry point is `setup` in the Makefile, which calls `bash ./setup.sh setup`. The script runs as a two-phase flow: an optional **bootstrap** (root) and always a **local-setup** (stack user).

### Phase Detection

Before doing anything, the script checks if host prerequisites are met. If all are present, it skips bootstrap entirely and goes straight to local-setup — no sudo prompt at all.

Triggers that force a bootstrap (one sudo):
- Stack user does not exist (`id <user>` fails)
- Stack user is not in the `server-stack` group (needed to read shared secrets from `$STACK_ENV_DIR/`)
- Repo root ownership doesn't match `<operator>:<stack-user> 2775` (happens on fresh clone)
- External data dirs exist but are owned by a different user

### Bootstrap Phase (`_bootstrap`, runs as root)

Creates host-level prerequisites. Every step is idempotent — re-running converges without errors.

| Step | What it does | Idempotent? |
|------|-------------|-------------|
| 1. Determine operator | Resolves the invoking user from `$SUDO_USER` or repo file ownership | — |
| 2. Create stack user | `useradd -m -s /bin/bash <stack-user>` with home dir and auto-assigned subuid/subgid ranges. If user exists, repairs missing home dir. | ✓ (skips if present) |
| 3. Assign subuid/subgid ranges | Reads `/etc/login.defs` for `SUB_UID_COUNT`/`SUB_GID_COUNT`; allocates the next free ascending block after the current max end. Checks `/etc/subuid` and `/etc/subgid` directly (not getent, since this system has no nsswitch entry for subuid). | ✓ (skips if present) |
| 4. Group memberships | Adds stack user to `server-stack` (shared secrets access). Some stacks also add to `render`/`video` for GPU device access. | ✓ (skips if already member) |
| 5. Enable linger | `loginctl enable-linger <stack-user>` — keeps rootless podman running after logout. Verifies it took effect before skipping. | ✓ (checks via loginctl) |
| 6. Two-tier repo ownership | `chown -R <operator>:<stack-user> <repo-dir>` then `chmod 2775`. The setgid bit on the root dir means new files created by the stack user inherit the operator's group, so git operations work without permission errors. `.env` is re-chowned to `<stack-user>:<stack-user>` after the recursive chown (it holds secrets). | ✓ (checks current owner) |
| 7. Runtime dirs ownership | `chown -R <stack-user>:<stack-user>` on runtime/state dirs inside the repo (`ts_state/`, app config bind mounts, etc.). Re-asserted every run so a migration never leaves them operator-owned. | ✓ |
| 8. External data dirs | Creates missing dirs (`mkdir -p`), then `chown -R <stack-user>:<stack-group>` if ownership differs from the stack user. Handles migrations from old combined stacks where the dir was owned by a different user. | ✓ (checks current owner) |
| 9. Install sudoers rule | Writes `/etc/sudoers.d/<stack>-ops` allowing the operator to run specific Makefile targets as `<stack-user>` without a password, scoped to `CWD=<repo-dir>`. Validates via `visudo -cf` before installing. Targets: `setup`, `local-setup`, `up`, `down`, `restart`, `upgrade`, `serve`, `status`, `logs`, `validate`, `teardown`, `local-teardown` (plus any stack-specific targets). | ✓ (overwrites with same content) |

### Local-Setup Phase (`local-setup`, runs as stack user)

Generates per-stack files and directories. Called after bootstrap, or independently via `make local-setup`.

| Step | What it does | Idempotent? |
|------|-------------|-------------|
| 1. Create secrets dir | `mkdir -p ./secrets` with mode 700, owned by stack user | ✓ |
| 2. Copy Tailscale auth key | Reads from `$STACK_ENV_DIR/.tailscale-authkey`, copies to `./secrets/.tailscale-authkey` (mode 600). Exits fatally if the shared key is missing — this means dozzle-stack hasn't run setup yet or the server-stack group isn't provisioned. | ✓ (overwrites) |
| 3. Copy Dozzle mTLS pair | Copies `.dozzle-cert.pem` and `.dozzle-key.pem` from `$STACK_ENV_DIR/` to `./secrets/`. Exits fatally if missing — dozzle-stack's bootstrap must have run first. | ✓ (overwrites) |
| 4. Stack-specific secrets | Generates secrets unique to this stack (e.g., Immich: DB password via `openssl rand -hex 32 > ./secrets/.db_password`). Only creates the file if it doesn't exist — existing passwords are never regenerated. | ✓ (skips if exists) |
| 5. Generate .env | Copies `.env.example` to `.env` if `.env` doesn't exist. Appends `PODMAN_SOCKET=/run/user/<uid>/podman/podman.sock`. Existing values are never clobbered — re-runs only add missing lines. Sets ownership to `<stack-user>:<stack-user>`, mode 600. | ✓ (skips if exists) |
| 6. Enable podman.socket | `systemctl --user enable podman.socket` + start. Required so the rootless Podman API socket stays alive for dozzle-agent. Sets `XDG_RUNTIME_DIR` explicitly because `sudo -u` strips it. | ✓ (no-op if enabled) |
| 7. Create ts_state dir | `mkdir -p ./ts_state`, mode 750, owned by stack user. This is where the Tailscale node identity persists — pre-created so it's never world-readable. | ✓ |
| 8. Register dozzle agent | If compose has a `dozzle-agent:` service, writes to `$STACK_ENV_DIR/dozzle-agents.d/<stack>` with format `<cert-domain>:7007\|<stack-name>`. Resolves the cert domain from `.env` (`TS_CERT_DOMAIN` or `<APP>_TS_CERT_DOMAIN`). Only writes if the registry dir exists and is writable. | ✓ (overwrites) |

### Execution Path

```
make setup  →  bash ./setup.sh setup
                     │
              running as root?
                     ├─ yes → do_bootstrap() → runuser -u <stack> make local-setup
                     │
              └─ no → check prerequisites
                        │
                   all present?
                          ├─ yes → sudo -n -u <stack> make local-setup (passwordless)
                          └─ no  → sudo bash ./setup.sh _bootstrap → sudo -n -u <stack> make local-setup
```

## Teardown Flow (`make teardown`)

Removes everything the stack created on the host. Runs in reverse order of setup: stack-level cleanup first, then host-level cleanup. Repo files and `.git` are preserved so `make setup` can re-provision without losing tracked changes.

### Phase Detection

Same entry point pattern — if run as root, does both phases in one pass. If run as operator, tries passwordless escalation to the stack user for local-teardown, then sudo to host teardown.

If the stack user was already deleted (e.g., manual cleanup), it gracefully skips the local phase and only removes host-level artifacts.

### Local-Teardown Phase (`local-teardown`, runs as stack user)

| Step | What it does |
|------|-------------|
| 1. Stop containers + remove volumes | `podman-compose down -v --remove-orphan` — stops all containers, removes the bridge network, and deletes named volumes (including DB data). External bind mounts are NOT touched. |
| 2. Remove secrets/ | `rm -rf ./secrets` — destroys all secret files |
| 3. Remove runtime dirs | `rm -rf ./ts_state/` and any app-specific config bind mount dirs (`./jellyfin/config/`, `./playlists/`, etc.) |
| 4. Remove .env | Deletes the generated `.env` (`.env.example` remains) |
| 5. Deregister dozzle agent | Removes `$STACK_ENV_DIR/dozzle-agents.d/<stack>` so the hub stops trying to dial this stack's agent |

**What is NOT removed:** External data directories on network storage (`/mnt/samba_storage/...`) are always left intact — those hold user data that may be needed if re-provisioning. The repo directory, `.git`, and tracked files remain for `make setup` to re-run.

### Host-Teardown Phase (`_host-teardown`, runs as root)

| Step | What it does |
|------|-------------|
| 1. Remove sudoers rule | Deletes `/etc/sudoers.d/<stack>-ops` — prevents stale passwordless rules after removal |
| 2. Disable linger | `loginctl disable-linger <stack-user>` — stops rootless podman from surviving logout |
| 3. Delete stack user | `userdel -r <stack-user>` — removes the user and home dir (which is usually empty for a stack account). Warns if it fails (active sessions or processes holding the UID) |
| 4. Delete stack group | `groupdel <stack-group>` — only if no other users are members. Warns if it fails (still in use) |

### Execution Path

```
make teardown  →  bash ./setup.sh teardown
                       │
                  running as root?
                       ├─ yes → runuser -u <stack> make local-teardown || true
                       │        bash ./setup.sh _host-teardown
                       │
              └─ no → check if stack user exists
                         │
                    exists?
                       ├─ yes → sudo -n -u <stack> make local-teardown (passwordless)
                       │          (if passwordless rule missing: one sudo to re-bootstrap, then retry)
                       │          sudo bash ./setup.sh _host-teardown
                       └─ no  → skip stack phase → still run _host-teardown for stale artifacts
```

## Teardown vs Setup Relationship

Teardown is **not** the exact inverse of setup. It removes runtime state and host-level identity, but preserves:

- **Repo files + `.git`** — so `make setup` can re-provision from the same clone without losing tracked changes like compose edits or custom Caddyfiles
- **External data directories** (`/mnt/samba_storage/...`) — user media lives here and should survive stack removal
- **Shared `$STACK_ENV_DIR/` contents** — the shared Tailscale key, dozzle mTLS pair, and `common.env` are managed by whichever stack generated them (e.g., deleting dozzle-stack's mTLS cert would break every other stack)

After teardown, running `make setup` again fully re-provisions from scratch: creates the user, copies secrets, generates new passwords if needed. The only thing that won't come back is named Podman volumes (DB data) — those must be restored from backup or accepted as fresh.
