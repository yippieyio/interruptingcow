# @interruptingcow/daemon

> **Status: design phase — no code yet.** This package contains only this README.

The **InterruptingCow service**: a long-running process that holds connections to
MU\* game servers on behalf of users, serves the web client, and owns the
database.

## Purpose

- Maintain **one persistent connection per (user, world)** — a **Session** — that
  outlives any attached client and is never dropped voluntarily.
- Relay lines both ways through a single **line pipeline** that logs everything
  and provides the hook points for triggers and notifications.
- Expose a **REST + WebSocket API** for the web client (auth, world management,
  live stream, history search).
- Persist accounts, worlds, session checkpoints, and the full line log to
  **PostgreSQL**.
- Provide the **`icow` admin CLI** (migrations, user/world management, tokens).

## Key behavior

- **"Forever" ≠ chasing server drops.** If a game server closes a connection, the
  Session goes `disconnected` and stays there until the user reconnects. The
  daemon only re-establishes connections automatically when recovering from its
  **own** outage (restart, power loss), within a grace window. See
  [ADR 0006](../../docs/adr/0006-connection-recovery-semantics.md).
- **No inbound plaintext telnet.** The only way in is the HTTPS API. A secure
  terminal gateway is a stubbed seam for later
  ([ADR 0004](../../docs/adr/0004-defer-terminal-gateway.md)).
- **The repository layer is the only code that touches SQL.**

## Boundaries

- Depends on `@interruptingcow/protocol` for all client-facing message shapes.
- Node.js LTS. Standard APIs preferred; Node-specific pieces (the `net`/`tls`
  listeners, the `pg` driver) sit behind interfaces.
- Holds **no durable state on disk** — everything is in PostgreSQL, whose data
  directory lives on a host volume
  ([ADR 0005](../../docs/adr/0005-podman-pod-external-state.md)).

## Planned layout

```
src/
  main.ts            # composition root — wires concrete implementations together
  config/            # env config, validated against a zod schema at boot
  http/              # Fastify: REST routes/ + WebSocket ws/ + static serving + /healthz
  session/           # Session, SessionManager, recovery, line-pipeline  ← the core
  gateways/
    upstream/        # ITransport (plain/TLS) + telnet IAC negotiation (+ stub options)
    terminal/        # ITerminalGateway + NullTerminalGateway (stub)
  triggers/          # ITriggerEngine + NoopTriggerEngine (stub)
  notifications/     # INotifier + NoopNotifier (stub)
  persistence/       # repositories/ + migrations/ + log-writer  ← the only SQL
  auth/              # Argon2id, JWT, refresh-token rotation
  cli/               # the `icow` command
test/
  integration/       # tests that need a real PostgreSQL and/or a fake socket
```

## Reference

- [Architecture](../../docs/architecture.md)
- [ADRs](../../docs/adr/0000-adr-process.md)

## Testing (planned)

`pnpm --filter @interruptingcow/daemon test` (unit) and `… test:integration`
(against a throwaway PostgreSQL and the fake MU\* server fixture). See
[testing.md](../../docs/development/testing.md).
