# Daedalus — Design

Status: agreed 2026-11-06 · v1 scope locked · Kanban board: `daedalus`

## What it is

Daedalus manages the home server container stacks. One Kotlin core, four frontends over it: CLI (scriptable), TUI (interactive dashboard), GUI (desktop window), MCP server (agent tools). Each stack host runs a small `daedalus-agent` daemon; daedalus talks to agents over HTTP — no SSH in v1. Authentication goes through the fleet's existing IdP (authentik) — no per-agent tokens, and neither client nor agent keeps persistent state beyond an append-only audit log.

- Repo: `java-alchemist/daedalus` · package `software.ascendant.daedalus`
- Build pattern follows `/opt/data/akctl` (Gradle 8.10.x, GraalVM native image)
- Sibling of akctl (authentik API client) — no shared code, different domain

## v1 scope

| In | Out (later) |
|---|---|
| Hosts: tva only | odin / turnstone-stack (multi-service outlier) |
| Inventory: `stacks list`, `stack show` | Drift checks (`check`) |
| Lifecycle wrappers: up/down/restart/upgrade/serve/logs | Stack creation/scaffolding (`create`) |
| All four frontends, built in parallel from day one | Secrets inventory/rotation |
| daedalus-agent daemon + one-time SSH bootstrap (incl. authentik identity init) | SSH fallback client (HostClient interface leaves the door open) |
| Human OIDC login (authorization code + PKCE) with per-user action attribution | Group-based authorization (who may run which target) |

## Architecture

```
daedalus-core   pure Kotlin, no UI deps — fleet.yaml model · HostClient interface
                (AgentClient: HTTP+JSON to per-host daedalus-agent) · operations
                with structured (JSON-serializable) result types
  ├── cli        picocli subcommands → native binary `daedalus`
  ├── tui        Mordant dashboard → same binary (`daedalus tui`; bare `daedalus` = TUI)
  ├── gui        Compose Desktop window → jpackaged JVM app (NOT native: skiko reflection
  │              on GraalVM native image is fiddly — not worth it for v1)
  └── mcp        MCP server, stdio transport (`daedalus mcp`) → same native binary

daedalus-agent  per-host daemon, native binary from the same repo — see below
```

## daedalus-agent (host daemon)

One `daedalus-agent` per host. Replaces SSH as the transport: structured data without parsing, clean log streaming, no key/BatchMode quirks; adding a host later = deploy one more binary.

- **Runs as:** systemd service under the operator user (`bfrey`, `Restart=always`) — never root. Stack-user operations go through the same passwordless sudoers rules the Makefiles use (conventions §7), so blast radius is unchanged from today's SSH workflow.
- **Server:** JDK `HttpServer` + kotlinx.serialization JSON — no web framework (~5 endpoints, stays GraalVM-native-friendly).
- **Endpoints:**

  | Endpoint | Purpose |
  |---|---|
  | `GET /health` | liveness + agent version |
  | `GET /stacks` | fleet inventory: per-stack containers (structured podman data), images+digests, git clean/dirty, ts_state presence |
  | `POST /stacks/<name>/ops/<target>` | run a make target (`up/down/restart/upgrade/serve/logs`) with streaming output; non-zero exit propagates |

- **Auth:** through the fleet's existing IdP (authentik) — no per-agent tokens, nothing stored on host or client except the audit log. Bootstrap creates one OAuth2 application `daedalus` with two grants (endpoints under `/application/o/daedalus/`, same pattern as Matrix SSO's `matrix-synapse`): **authorization code + PKCE** for humans — CLI/TUI/GUI open a browser, the operator signs in to authentik (already SSO'd), and daedalus receives a JWT carrying *that user's* identity; and **client_credentials** for non-interactive use (the MCP server) under the `daedalus` service account. Access tokens live in memory only; refresh tokens go to the OS keychain, never plaintext files. Agents verify every JWT against `/application/o/daedalus/jwks/` (signature + audience + expiry), fail closed if unverifiable, and **attribute every action to the token's `sub`** — an append-only audit log at `/opt/data/daedalus-agent/audit.log` (JSON lines: timestamp, user, stack, target, exit code) records who did what. No action runs under a generic or unattributed identity; the only long-lived credential is the generated client secret, kept as a gitignored dotfile in the daedalus repo (`secrets/.authentik-client-secret`, mode 600).
- **Network:** binds the tailnet interface on port 8461 (no host ports published elsewhere; no SERVE needed for an API).
- **Ops execution:** command templates are configurable via env/config — production defaults run the real `make`/podman commands; test mode points at stub scripts. This is what makes containerized integration testing possible without nested podman.
- **Bootstrap:** one-time SSH install via `daedalus agent install --host tva`: copies binary + systemd unit, clones server-stack-wiki on the host (the agent reads fleet.yaml from it), and — because tva hosts authentik — initializes daedalus' identity in authentik using the existing admin key at `/opt/data/stack-env/.authentik_api_key` (idempotent: OAuth app `daedalus` with both grants + loopback redirect URI `http://127.0.0.1:8462/callback`, plus the service account). The installer relays the generated client secret to WSL exactly once, where it lands in the gitignored dotfile above. After that, SSH is never needed again.

## Fleet model

`fleet.yaml` at the root of this repo (server-stack-wiki) — single source of truth. Daedalus reads it from a local clone (default `/opt/data/server-stack-wiki`, override `--wiki` / `$DAEDALUS_WIKI`). Adding a host or stack = one YAML block + commit.

```yaml
hosts:
  tva: { ssh: bfrey@tva, agent_port: 8461, authentik_url: https://authentik.tail6a7569.ts.net }   # ssh kept for bootstrap/manual recovery only
stacks:
  dns:    { host: tva, path: /opt/data/dns-stack }
  server: { host: tva, path: /opt/data/server-stack, containers: false }   # owns stack-env; no podman
```

## Operations → agent calls

Daedalus never touches containers directly — the agent runs the same make targets a human would. Every request carries the requesting identity's JWT; the agent attributes it in the audit log.

| daedalus | agent call | host action |
|---|---|---|
| inventory (`stacks list`, `stack show`) | `GET /stacks` | `make status` parse (running containers only — raw podman is NOT in the sudoers rules; see decision below), git status, ts_state check |
| `stack up\|down\|restart\|upgrade\|serve <n>` | `POST /stacks/<n>/ops/<target>` | `cd <path> && make <target>` + audit entry (user, target, exit code) |
| `stack logs <n> [-s svc] [--tail N]` | `POST /stacks/<n>/ops/logs` (params) | `make logs [SERVICE=svc TAIL=N]` + audit entry |

## Frontend notes

- **TUI** — k9s-style: fleet table with live refresh (~5 s), select → detail pane (containers/images/git state), keys to run actions + log tail. No logic beyond core. Shows the logged-in user; all actions attributed to them.
- **GUI** — Compose Desktop themed to IntelliJ's design language (Darcula/new-UI palette, JetBrains fonts via custom MaterialTheme). Close to the IDEA look; not literal IDE chrome. Fleet table → stack detail → actions/logs. Header shows the logged-in user and prompts OIDC login when no session exists.
- **MCP** — stdio transport for local agent integration (Hermes MCP server entry: `command: daedalus, args: [mcp]`). Tools map 1:1 to core ops: `stacks_list`, `stack_show`, `stack_up/down/restart/upgrade/serve`, `stack_logs`. Structured JSON results straight from core data classes. Runs non-interactively under the `daedalus` service account (client_credentials) — audit entries attribute those actions to the machine identity, never a human.

## Testing

Full-coverage bar — the build fails on `./gradlew check` if it isn't met (JaCoCo gates). JUnit 6 (Jupiter — requires Java 17+ / Kotlin 2.2; we build on GraalVM JDK 21) in every module; no live-tva or live-authentik dependency anywhere in CI — e2e against the real fleet is a manual step inside T10's verification.

- **Unit:** core — fleet.yaml parsing, DTO serialization round-trips, AgentClient against MockWebServer (bad token, expired JWT, timeout, agent-down errors), OIDC PKCE flow against a MockWebServer fake authentik (code exchange, refresh); cli — argument parsing + command construction; mcp — tool dispatch mapping.
- **Contract tests (core ↔ agent boundary):** golden JSON fixtures in `core/src/test/resources/contract/` — both modules serialize/deserialize the same files and a test asserts they agree on every field. This is where client/server bugs live, so it gets first-class treatment.
- **Testcontainers:** integration tests run the real `daedalus-agent` in a container (built from this project) with host ops stubbed via the configurable command templates — full HTTP stack + process-execution path exercised without touching tva and without nested podman. Covers `/health`, `/stacks` (all 9 stacks incl. `containers: false`), op streaming, exit-code propagation, token rejection (agent auth verified against a **stub JWKS** — no live authentik in CI), and audit-log attribution for user vs machine tokens. Git clean/dirty logic is tested against real repos in temp dirs (git is local; no container needed).
- **Coverage gates:** JaCoCo — core and agent ≥ 90% line coverage on `check`; cli/tui/gui/mcp ≥ 70%. UI rendering itself is excluded from the gate; all state/transformation logic lives in testable view-model classes so the UI layer stays thin.

## Ticket map (board `daedalus`)

```
T1 scaffold repo+GitHub ─┬→ T9 agent daemon ──────────────────────┐
                         │                                        ├→ T10 deploy to tva → T4 CLI inventory
T2 audit fleet.yaml ────┴→ T3 core → T11 OIDC user login ─────────┘        ├→ T5 CLI lifecycle
                                                                            ├→ T6 TUI (Mordant)
                                                                            ├→ T7 GUI (Compose Desktop)
                                                                            └→ T8 MCP server
```

Both infra chains start immediately and run in parallel; all four frontends fan out once core + login + a live agent exist.

## Inventory data source — decision (2026-11-07, T2 fleet audit)

Live verification on tva (`sudo -n -l` + direct probe): each stack's sudoers rule allows ONLY the enumerated `/usr/bin/make <target>` commands with `CWD=<stackdir>`. Raw podman is NOT permitted — probe `sudo -n -u dns podman ps -a --format json` → "a password is required" (rc=1).

**Decision: daedalus-agent parses `make status` output for container inventory in v1.** Consequences:
- Inventory covers RUNNING containers only (`podman ps` without `-a`); stopped/exited containers are invisible. Acceptable for v1 — the boot-time autostart convention means a healthy stack has all its containers running, and an empty table is itself a signal (stack down).
- No image digests in inventory; `stack show` reports what the status table carries (name, image tag, state/health). Digest-level detail deferred.
- All 8 container stacks' rules include `status`; server-stack has no containers (`containers: false`) — its inventory is git state only.
- If structured podman data becomes required later: extend each stack's sudoers rule with `/usr/bin/podman ps -a --format json` (requires re-running `_bootstrap` per stack, root) — follow-up, not v1.
