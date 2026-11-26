# radiance-stack

Qwen3.8-Flash-Next LLM inference on freya's Radeon AI PRO R9700 (AMD ROCm), served by [Radiance](https://codeberg.org/StillDeadCode/radiance) — an OpenAI-compatible API. Own Tailscale node **`radiance`**; the API is at `https://radiance.<tailnet>.ts.net`.

## Role in the Fleet

The radiance stack provides LLM inference for the fleet: one container serves the
**Qwen3.8-Flash-Next** model (114 GiB `.rad` container, IQR-V4 experts) on a single RDNA4 card.
Requests are OpenAI-compatible (`/v1/chat/completions`, `model: "flashnext"`), so any client that
speaks the OpenAI API can point at the tailnet URL with an API key.

Unlike the other stacks, this one is compute-bound rather than data-bound: the model file lives on a
dedicated XFS NVMe volume and is read directly (direct I/O) by the server; the routed-expert pool
(42 GiB) is pinned in host RAM so experts never leave the card.

## Architecture

```
tailnet client ──→ https://radiance.<tailnet>.ts.net   (SERVE :443 → radiance-server:8000)
                      │
                 radiance-tailscale  (userspace sidecar; node identity in ./ts_state/)
                      │  bridge network radiance-net
                 radiance-server     (ROCm on /dev/kfd + /dev/dri, GPU index 0 = R9700)
                      │  bind mount ro
                 /var/mnt/models/rad/qwen3.8-next-flash-iqr-v4.rad   (XFS NVMe, direct I/O)

operator on freya ──→ http://127.0.0.1:8000   (loopback-only binding; make status/test)
```

- The server runs off upstream's `network_mode: host`, on the shared bridge instead, so the sidecar
  can proxy by container name — inference traffic is small JSON, and model I/O / KV cache never cross a socket.
- A loopback-only port binding (`127.0.0.1:8000`) keeps operator tooling working on the box itself;
  nothing is exposed to the LAN (fleet rule: no host ports).

## Services

| Container | Image | Role |
|---|---|---|
| `radiance-tailscale` | `tailscale/tailscale:latest` | Userspace tailscaled + SERVE (:443 → server:8000). Standard sidecar env block (conventions §5); serve config in `serve-config/serve.json`. |
| `radiance-server` | `stilldeadcode/radiance:latest` | Radiance engine; serves the `flashnext` entry from `radiance.yaml`. GPU via `/dev/kfd` + `/dev/dri` device mounts, `ROCR_VISIBLE_DEVICES=0`. API key enters through the environment (`api-key-env`), never the config file. |

## Data & Secrets

| Path | What |
|---|---|
| `/var/mnt/models/rad/` | `.rad` model containers on a dedicated XFS NVMe volume (3.7 TB), mounted ro at `/models`. Operator-owned (`bfrey:ai-models`); `make download` fetches the 114 GiB file (resumable). |
| `./radiance.yaml` | Server config (model entry, port 8000, expert-pool sizing) — committed, mounted ro. |
| `./.env` | `RADIANCE_API_KEY`, GPU index, image tag, TS node vars (`TS_HOSTNAME`, `TAILNET_DOMAIN`). 640 user:group. |
| `./secrets/.tailscale-authkey` | Per-stack copy of the shared key (copy-or-fail from `$STACK_ENV_DIR`; only server-stack provisions it). |
| `./ts_state/` | Tailscale node identity (`state.soc`) — never delete or the node gets a new hostname. |

**API-key deviation:** `RADIANCE_API_KEY` lives in `.env`, not under `secrets/`. Upstream radiance
reads the key from an environment variable named by the config's `api-key-env`; it has no file-based
secret path, so the fleet's "all credentials are files" rule is relaxed for this one value (group-readable 640, never committed).

## Operations

Standard Makefile targets apply. Radiance-specific:

| Target | Description |
|--------|-------------|
| `download` | Fetch the 114 GiB model container into `RADIANCE_MODELS` (resumable; runs as operator — owns `/var/mnt/models`) |
| `rotate-key` | Generate a new `RADIANCE_API_KEY` in `.env` (shown once) and recreate the server to apply it |
| `serve` | Re-apply the tailscale serve config (restarts the sidecar) |
| `test` | Full test suite (`tests/run-tests.sh`) — first fleet stack with one: config/secrets/runtime/tailnet/autostart checks. SKIP is acceptable (e.g. model still loading); any FAIL exits non-zero |

Model load takes minutes after every container start — the healthcheck allows a 15-minute
`start_period`, so `make status` will report "not answering yet" for a while. That is normal, not an error.

## Gotchas

- **Kinoite host:** no dnf — host tools are layered with `rpm-ostree ex <pkg>` (e.g., `sudo rpm-ostree ex make`, effective after reboot).
- **GPU index 0 only.** The R9700 is `pci-03:00.0` → `renderD128`; the Granite Ridge iGPU (index 1) must not be used (`ROCR_VISIBLE_DEVICES=0`).
- **Model generation:** the engine (radiance 2.x, what `latest` ships) refuses block-FP8/E4M3 containers at startup — the old `qwen3.8-next-flash-fp8-iq4r-moe.rad` (a 1.x container) will not load. The working file is `qwen3.8-next-flash-iqr-v4.rad`.
- **No dozzle-agent yet:** freya predates full tailnet integration, so this stack does not run a dozzle agent and its containers are invisible to the central log viewer until freya's `$STACK_ENV_DIR` has the shared mTLS pair (generated by dozzle-stack's bootstrap).
- **Group model (freya variant):** a single `radiance-stack` group covers both repo management and `.env`/`secrets/` access for operator + stack user, instead of tva's split (`server-stack` shared for the repo, per-stack group for secrets). The stack user additionally joins the fleet-wide `server-stack` group so it can read shared secrets from `$STACK_ENV_DIR` (files are `root:server-stack` 640) — bootstrap adds this and re-bootstraps automatically if it is missing.
- **SELinux (Kinoite, enforcing):** confined containers cannot write to `usr_t`-labeled dirs even with correct ownership — the sidecar died with `mkdir /var/lib/tailscale/dns-cache: permission denied`. Bootstrap/local-setup apply the private container label (`chcon -Rt container_file_t ts_state/`, recursive so state files inside keep it); if a recreated dir loses it, re-run `make setup`.
- **podman ≥ 6 inspect lies about devices:** `HostConfig.Devices` is always `[]` even when the container was created with `--device` (verified on podman 6.2 — the only trace is in `.Config.CreateCommand`). Device checks must run INSIDE a running container (`podman exec <c> test -e /dev/kfd`) and retry across crash-loop gaps; never trust the inspect JSON.
- **R9700 VRAM coexistence:** flashnext (~20 GiB) + LM Studio's Qwen3.8-27B (~17+ GiB) > 31.86 GiB — one big tenant per card. Unload LM Studio before `make gpu` and keep it unloaded until `/health` is OK; VRAM grabbed mid-load kills the load with a bare "device error" (kernel init refuses the instance) and the container crash-loops until the card frees up. Note: any diagnostic run *during* a Bionic session necessarily sees LM Studio loaded — check `make vram` at the moment that matters.
- **ts_state/ CONTENTS must stay stack-user-owned.** The bootstrap's recursive repo chown re-owns everything — including runtime state. Older versions re-asserted only the *directory*, leaving `tailscaled.state` operator-owned; tailscaled then could not open it, fell back to an in-memory store, and lost the tailnet node identity on every restart (`store.New failed ... permission denied`, sidecar crash-looping). Bootstrap now re-asserts recursively (`chown -R`, `chcon -Rt`), SELFHEAL fails closed when any file under `ts_state/` is not owned by the stack user (printing the exact fix command), and test #4 verifies contents ownership + label. On an old install: `sudo chown -R radiance:radiance-stack ts_state/`, or delete the state file for a fresh tailnet identity.
