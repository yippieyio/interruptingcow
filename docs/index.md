# InterruptingCow Documentation

> **Status: design phase.** Nothing here is built yet. These documents describe
> what InterruptingCow *will be* and *why it is shaped that way*, so the design
> can be reviewed before code depends on it.

InterruptingCow is a daemon that keeps your MUSH / MUCK / MOO connections alive
while your client is away. See the [project README](../README.md) for the
elevator pitch.

## Start here

| Document | What it covers |
|---|---|
| [Architecture](architecture.md) | The whole system: components, the Session model, the line pipeline, data model, durability, deployment. **Read this first.** |
| [Wire protocol](protocol.md) | How the web client and the daemon talk: the WebSocket message set, the REST API, the handshake, versioning. |
| [Telnet & MU\* protocols](telnet-protocols.md) | Reference: the upstream (game-facing) telnet options and MU\* protocols — what each does, its format, and whether we speak it. |
| [Glossary](glossary.md) | MU\*, telnet, and project-specific terminology. |

## Decisions

| Document | What it covers |
|---|---|
| [ADR process](adr/0000-adr-process.md) | How architecture decisions are proposed, discussed, and recorded. |
| [ADR template](adr/template.md) | Copy this to start a new ADR. |
| [ADR index](#architecture-decision-records) | The decisions made so far. |

## For developers

| Document | What it covers |
|---|---|
| [Getting started](development/getting-started.md) | The intended path from clone to running tests. |
| [Coding standards](development/coding-standards.md) | Style, module boundaries, error handling, logging, tests. |
| [Environment](development/environment.md) | Toolchain, devcontainer, configuration model. |
| [Testing](development/testing.md) | The test pyramid, the fake MU\* server, coverage expectations. |
| [Releasing](development/releasing.md) | Versioning policy and the release flow. |
| [Troubleshooting](development/troubleshooting.md) | Common problems (to be filled in as features land). |

## For operators

| Document | What it covers |
|---|---|
| [Runbook](operations/runbook.md) | Start, stop, upgrade, roll back, monitor. |
| [Backup & restore](operations/backup-restore.md) | Protecting the data that must survive a power loss. |

## For users

| Document | What it covers |
|---|---|
| [Installing the app](user-guide/install.md) | Adding the PWA to iOS, Android, and desktop. |
| [Managing worlds](user-guide/worlds.md) | Adding worlds, connecting, what "kept alive" means. |
| [History & search](user-guide/history.md) | Scrollback, backfill, full-text search. |
| [Notifications](user-guide/notifications.md) | How push will work (planned). |

## Architecture Decision Records

| # | Title | Status |
|---|---|---|
| [0000](adr/0000-adr-process.md) | The ADR process | Accepted |
| [0001](adr/0001-all-typescript.md) | All TypeScript, one workspace | Accepted |
| [0002](adr/0002-postgres-over-sqlite.md) | PostgreSQL over SQLite | Accepted |
| [0003](adr/0003-json-zod-wire-protocol.md) | JSON + zod wire protocol | Accepted |
| [0004](adr/0004-defer-terminal-gateway.md) | Defer the inbound terminal gateway | Accepted |
| [0005](adr/0005-podman-pod-external-state.md) | Podman pod, state on host volumes | Accepted |
| [0006](adr/0006-connection-recovery-semantics.md) | Connection-recovery semantics | Accepted |
| [0007](adr/0007-licensing-and-workflow.md) | Licensing, DCO, and development workflow | Accepted |
| [0008](adr/0008-forwarded-user-identity.md) | Offer per-user origin information to destinations | Proposed |

_"Accepted" here means "decided as part of the initial design". Any of these can
be revisited with a superseding ADR._

_ADR 0008 is **Proposed** — open for review. It proposes the capability and the
seam; the wire transport for the forwarded-identity payload is left to a
follow-up ADR._
