# Daedalus — Design

Status: agreed 2026-11-06 · v1 scope locked · Kanban board: `daedalus`

## What it is

Daedalus manages the home server container stacks. One Kotlin core, four frontends over it: CLI (scriptable), TUI (interactive dashboard), GUI (desktop window), MCP server (agent tools). Each stack host runs a small `daedalus-agent` daemon; daedalus talks to agents over HTTP — no SSH in v1.

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
| daedalus-agent daemon + one-time SSH bootstrap | SSH fallback client (HostClient interface leaves the door open) |

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

- **Auth:** Bearer token, random 32-char key generated at install. Host-side: `/opt/data/daedalus-agent/.token` (600). Client-side: `~/.config/daedalus/tokens.yaml` — local only, never committed; fleet.yaml stays secret-free.
- **Network:** binds the tailnet interface on port 8461 (no host ports published elsewhere; no SERVE needed for an API).
- **Ops execution:** command templates are configurable via env/config — production defaults run the real `make`/podman commands; test mode points at stub scripts. This is what makes containerized integration testing possible without nested podman.
- **Bootstrap:** one-time SSH install via `daedalus agent install --host tva` (copies binary + systemd unit, generates token) — after that, SSH is never needed again.

## Fleet model

`fleet.yaml` at the root of this repo (server-stack-wiki) — single source of truth. Daedalus reads it from a local clone (default `/opt/data/server-stack-wiki`, override `--wiki` / `$DAEDALUS_WIKI`). Adding a host or stack = one YAML block + commit.

```yaml
hosts:
  tva: { ssh: bfrey@tva, agent_port: 8461 }   # ssh kept for bootstrap/manual recovery only
stacks:
  dns:    { host: tva, path: /opt/data/dns-stack }
  server: { host: tva, path: /opt/data/server-stack, containers: false }   # owns stack-env; no podman
```

## Operations → agent calls

Daedalus never touches containers directly — the agent runs the same make targets a human would.

| daedalus | agent call | host action |
|---|---|---|
| inventory (`stacks list`, `stack show`) | `GET /stacks` | `podman ps -a --format json` as stack user (via sudoers), git status, ts_state check — audit ticket confirms the sudoers rule covers this per stack |
| `stack up\|down\|restart\|upgrade\|serve <n>` | `POST /stacks/<n>/ops/<target>` | `cd <path> && make <target>` |
| `stack logs <n> [-s svc] [--tail N]` | `POST /stacks/<n>/ops/logs` (params) | `make logs [SERVICE=svc TAIL=N]` |

## Frontend notes

- **TUI** — k9s-style: fleet table with live refresh (~5 s), select → detail pane (containers/images/git state), keys to run actions + log tail. No logic beyond core.
- **GUI** — Compose Desktop themed to IntelliJ's design language (Darcula/new-UI palette, JetBrains fonts via custom MaterialTheme). Close to the IDEA look; not literal IDE chrome. Fleet table → stack detail → actions/logs.
- **MCP** — stdio transport for local agent integration (Hermes MCP server entry: `command: daedalus, args: [mcp]`). Tools map 1:1 to core ops: `stacks_list`, `stack_show`, `stack_up/down/restart/upgrade/serve`, `stack_logs`. Structured JSON results straight from core data classes.

## Testing

Full-coverage bar — the build fails on `./gradlew check` if it isn't met (JaCoCo gates). JUnit 6 (Jupiter — requires Java 17+ / Kotlin 2.2; we build on GraalVM JDK 21) in every module; no live-tva dependency anywhere in CI — e2e against the real fleet is a manual step inside T10's verification.

- **Unit:** core — fleet.yaml parsing, DTO serialization round-trips, AgentClient against MockWebServer (auth failure, timeout, agent-down errors); cli — argument parsing + command construction; mcp — tool dispatch mapping.
- **Contract tests (core ↔ agent boundary):** golden JSON fixtures in `core/src/test/resources/contract/` — both modules serialize/deserialize the same files and a test asserts they agree on every field. This is where client/server bugs live, so it gets first-class treatment.
- **Testcontainers:** integration tests run the real `daedalus-agent` in a container (built from this project) with host ops stubbed via the configurable command templates — full HTTP stack + process-execution path exercised without touching tva and without nested podman. Covers `/health`, `/stacks` (all 9 stacks incl. `containers: false`), op streaming, and exit-code propagation. Git clean/dirty logic is tested against real repos in temp dirs (git is local; no container needed).
- **Coverage gates:** JaCoCo — core and agent ≥ 90% line coverage on `check`; cli/tui/gui/mcp ≥ 70%. UI rendering itself is excluded from the gate; all state/transformation logic lives in testable view-model classes so the UI layer stays thin.

## Ticket map (board `daedalus`)

```
T1 scaffold repo+GitHub ─┬→ T9 agent daemon → T10 deploy to tva ──┐
                         │                                        ├→ T4 CLI inventory
T2 audit fleet.yaml ─────┴→ T3 core (HostClient + ops) ───────────┤→ T5 CLI lifecycle
                                                                  ├→ T6 TUI (Mordant)
                                                                  ├→ T7 GUI (Compose Desktop)
                                                                  └→ T8 MCP server
```

Both infra chains (agent, core) start immediately and run in parallel; all four frontends fan out once core + a live agent exist.
