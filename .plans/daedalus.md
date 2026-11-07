# Daedalus — Design

Status: agreed 2026-11-06 · v1 scope locked · Kanban board: `daedalus`

## What it is

Daedalus manages the home server container stacks. One Kotlin core, four frontends over it: CLI (scriptable), TUI (interactive dashboard), GUI (desktop window), MCP server (agent tools).

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

## Architecture

```
daedalus-core   pure Kotlin, no UI deps — fleet.yaml model · SSH runner · operations
                with structured (JSON-serializable) result types
  ├── cli        picocli subcommands → native binary `daedalus`
  ├── tui        Mordant dashboard → same binary (`daedalus tui`; bare `daedalus` = TUI)
  ├── gui        Compose Desktop window → jpackaged JVM app (NOT native: skiko reflection
  │              on GraalVM native image is fiddly — not worth it for v1)
  └── mcp        MCP server, stdio transport (`daedalus mcp`) → same native binary
```

## Fleet model

`fleet.yaml` at the root of this repo (server-stack-wiki) — single source of truth. Daedalus reads it from a local clone (default `/opt/data/server-stack-wiki`, override `--wiki` / `$DAEDALUS_WIKI`). Adding a host or stack = one YAML block + commit.

```yaml
hosts:
  tva: { ssh: bfrey@tva }        # BatchMode ssh, tailnet-resolved hostname
stacks:
  dns:    { host: tva, path: /opt/data/dns-stack }
  server: { host: tva, path: /opt/data/server-stack, containers: false }   # owns stack-env; no podman
```

## Operations → make targets

Daedalus never touches containers directly — it speaks the same language a human would. Every lifecycle op is `ssh <host> 'cd <path> && make <target>'`; Makefiles self-escalate to the stack user via sudoers (conventions §7).

| daedalus | host command |
|---|---|
| `stack up\|down\|restart\|upgrade\|serve <n>` | `make up\|down\|restart\|upgrade\|serve` |
| `stack logs <n> [-s svc] [--tail N]` | `make logs [SERVICE=svc TAIL=N]` |
| inventory (containers) | structured: `podman ps -a --format json` as stack user if the sudoers rule permits, else parse `make status` — decide during the fleet-audit ticket |

## Frontend notes

- **TUI** — k9s-style: fleet table with live refresh (~5 s), select → detail pane (containers/images/git state), keys to run actions + log tail. No logic beyond core.
- **GUI** — Compose Desktop themed to IntelliJ's design language (Darcula/new-UI palette, JetBrains fonts via custom MaterialTheme). Close to the IDEA look; not literal IDE chrome. Fleet table → stack detail → actions/logs.
- **MCP** — stdio transport for local agent integration (Hermes MCP server entry: `command: daedalus, args: [mcp]`). Tools map 1:1 to core ops: `stacks_list`, `stack_show`, `stack_up/down/restart/upgrade/serve`, `stack_logs`. Structured JSON results straight from core data classes.

## Ticket map (board `daedalus`)

```
T1 scaffold repo+GitHub ─┐
                         ├→ T3 core → T4 CLI inventory
T2 audit fleet.yaml ─────┘         ├→ T5 CLI lifecycle
                                   ├→ T6 TUI (Mordant)
                                   ├→ T7 GUI (Compose Desktop)
                                   └→ T8 MCP server
```

All four frontends start as soon as core lands — parallel by design.
