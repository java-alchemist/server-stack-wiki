# Kanban — Project Tracking with Hermes' Built-in Board

Hermes ships a built-in kanban: tasks live in SQLite (`kanban.db` under the Hermes home), are visible in the **web app UI** (the "kanban" panel), and are fully scriptable from the CLI. Use it to track multi-step projects — some tickets as plain tracking items, others handed to agent workers for execution.

## CLI Basics

```bash
H=/opt/hermes/bin/hermes          # not on PATH by default in this environment

$H kanban list                    # all tasks on the active board (alias: ls)
$H kanban show <id>               # task + comments + event history
$H kanban boards list             # boards; ● marks the active one
```

Task IDs look like `t_5227c550`. Every command accepts `--json` for machine-readable output.

## Boards

One board per project keeps things clean (a shared `default` board also exists):

```bash
$H kanban boards create akctl --name "akctl CLI"   # slug + display name
$H kanban boards switch akctl                      # persist selection to <home>/kanban/current
$H kanban boards list | rm | archive | rename | set-wd ...
```

Active-board resolution order: `HERMES_KANBAN_BOARD` env var → `<home>/kanban/current` file (written by `boards switch`) → `default`. The dispatcher sets the env var on worker spawn, so workers always see the right board.

## Creating Tickets

### Tracking ticket (no agent work)

Omit `--assignee` and the task simply sits on the board — nothing gets dispatched:

```bash
$H kanban create "akctl: build container image" \
  --body 'Dockerfile ready in repo root. cp build/native/nativeCompile/akctl . && docker build -t akctl .'
# → Created t_deb4d80c  (ready, assignee=-)
```

Use `--body-file PATH` (or `-` for stdin) when the body has newlines or flag-like lines that would mangle shell quoting.

### Agent-executed task

Add `--assignee <profile>` and the gateway picks it up automatically (`kanban.dispatch_in_gateway: true`, every 60s by default). The worker runs in its own terminal session with a workspace of your choice:

```bash
$H kanban create "Rotate synapse federation keys" \
  --body 'Run make rotate-keys on tva, verify /_matrix/federation/v1/version still serves.' \
  --assignee default \
  --workspace dir:/opt/data/synapse-stack   # or: scratch | worktree | worktree:<path>
```

Useful flags: `--priority`, `--skill <name>` (load a skill into the worker), `--model` / `--provider` overrides, `--max-retries`, `--idempotency-key` (safe re-runs), `--parent` (repeatable; same as `link`).

## Dependencies

```bash
$H kanban link <parent-id> <child-id>    # child waits for parent
$H kanban unlink <parent-id> <child-id>
```

A linked child starts in `todo`; it flips to `ready` automatically when its parent completes. This is how you express "restart synapse after the user account exists" without manual sequencing.

## Statuses

`triage → todo → scheduled → ready → running → review / done`, plus `blocked` and `archived`.

- New tasks land in `ready` (or `todo` if they have unfinished parents).
- `--initial-status blocked|running` at create time for the two non-default starts.
- A task that fails its worker `failure_limit` times (default 2) is circuit-broken to `blocked`; unblock with a status edit or reassign.
- `$H kanban complete <id>...`, `$H kanban block <id>...`, `$H kanban comment <id> "note"`, `$H kanban assign|reassign|reclaim` for the rest of the lifecycle.

## Config (`config.yaml`)

```yaml
kanban:
  dispatch_in_gateway: true        # gateway auto-dispatches assigned tasks
  dispatch_interval_seconds: 60    # poll cadence
  failure_limit: 2                 # worker failures before circuit-break to blocked
  default_assignee: ''             # leave empty so unassigned = pure tracking ticket
  auto_decompose: true             # decompose large goals into subtasks
```

## Worked Example — the akctl Project (Nov 2026)

Six tickets on a `default` board, dependency chain for the critical path:

```
▶ t_5227c550  ready   akctl: live-test against real authentik (2026.8.5)
▶ t_58da9bed  ready   akctl: create GitHub repo java-alchemist/akctl and push
◻ t_a9f88b54  todo    Create rahzul authentik account via akctl        ← waits on live-test
◻ t_ae056d24  todo    Restart synapse + full SSO test in Element Web   ← waits on rahzul
▶ t_deb4d80c  ready   akctl: build container image (optional)
▶ t_4181c821  ready   akctl: port app provisioning from bash scripts (future)

$H kanban link t_5227c550 t_a9f88b54
$H kanban link t_a9f88b54 t_ae056d24
```

Pattern that worked well: every ticket's body carries the *exact* commands and file paths needed to execute it cold (repo path, binary location, blocker description) — so any agent or human can pick up a ticket without session context.
