# @interruptingcow/web

> **Status: design phase — no code yet.** This package contains only this README.

The **InterruptingCow web client**: an Angular application delivered as an
installable Progressive Web App (PWA).

## Purpose

- Let a user **attach** to their worlds' live streams, read and type, and
  **detach** — without disconnecting the underlying game connections.
- Render the terminal (ANSI colour, input history, tab-complete), the world list
  with live status, and full-text **history search**.
- Be **installable** on iOS (16.4+, Home Screen), Android, and desktop, with an
  offline app shell.
- Store as little as possible on the device — a login token and a few UI
  preferences. Worlds, scrollback, and settings come from the server.

## Boundaries

- Depends on `@interruptingcow/protocol` for the WebSocket message types and REST
  DTOs, consumed through its `Codec`.
- Talks only to the daemon's HTTPS API (REST + WebSocket), same origin, via the
  reverse proxy — no direct game connections, no CORS surface.
- Must expose a visible **Source** link (AGPL §13 —
  [ADR 0007](../../docs/adr/0007-licensing-and-workflow.md)).

## Planned layout

```
src/app/
  core/         # token storage, auth interceptor, WsClientService (uses @interruptingcow/protocol)
  data/         # typed REST clients: worlds, history, settings
  features/
    terminal/   # output pane (ANSI → spans), input line, history, tab-complete
    worlds/     # world list, live status, connect/disconnect, attach/detach
    history/    # scrollback view + full-text search
    settings/   # account, per-world prefs, Source link; triggers route = "coming soon" stub
  app.config.ts
```

## Reference

- [Architecture: the web client](../../docs/architecture.md#the-web-client)
- [User guide](../../docs/user-guide/install.md)
- [Wire protocol](../../docs/protocol.md)

## Testing (planned)

`pnpm --filter @interruptingcow/web test` — component tests (ANSI rendering,
input history, world-list status) against a mocked `WsClientService`, plus
contract tests against `@interruptingcow/protocol`'s sample corpus. End-to-end
coverage lives in the top-level `e2e/` suite.
