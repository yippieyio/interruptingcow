# Troubleshooting (developers)

> **Status: design phase.** There's no software to troubleshoot yet. This is a
> placeholder with the categories we expect to need. Each will be filled in as
> the corresponding feature lands — with real symptoms, causes, and fixes, not
> guesses.

Operator-facing problems (running an instance) belong in the
[runbook](../operations/runbook.md). User-facing problems belong in the
[user guide](../user-guide/install.md). This page is for people hacking on the
code.

## Anticipated categories

### Local environment

- `pnpm install` fails or resolves the wrong versions → Node/pnpm version
  mismatch; Corepack not enabled. _(to be written)_
- The dev PostgreSQL won't start → Podman machine not running (macOS); port in
  use. _(to be written)_
- Migrations fail on a fresh database → order/dependency issue in a migration.
  _(to be written)_

### Tests

- Integration tests can't reach PostgreSQL → the throwaway database didn't come
  up; connection string. _(to be written)_
- Flaky end-to-end tests → timing assumptions, un-awaited state, port
  contention; how to get Playwright traces. _(to be written)_
- Contract tests fail after a protocol edit → a schema changed without
  regenerating the sample corpus. _(to be written)_

### The daemon

- The daemon exits immediately on start → configuration failed zod validation;
  read the message, check `.env` against `.env.example`. _(to be written)_
- A world won't connect → host/port/TLS/encoding wrong; the upstream server
  refusing; TLS handshake failure. _(to be written)_
- Telnet negotiation misbehaves (garbled output, wrong width) → an option not
  handled; test against the fake MU\* server with negotiation logging.
  _(to be written)_
- A Session stuck in `connecting` → connect attempt neither succeeding nor
  failing; socket timeout handling. _(to be written)_
- Our-fault recovery didn't fire (or fired when it shouldn't) → grace window;
  `intended_connected` / checkpoint state; `auto_recover` flag. See
  [ADR 0006](../adr/0006-connection-recovery-semantics.md). _(to be written)_
- Memory growth over long uptime → an unbounded buffer or cache; which one and
  how to find it. _(to be written)_

### The web client

- The PWA won't install → manifest invalid; not served over HTTPS; iOS
  specifics. _(to be written)_
- The service worker won't update → caching; how to force a refresh in dev.
  _(to be written)_
- The WebSocket won't connect or keeps dropping → proxy config; token expiry;
  `hello` rejected. _(to be written)_
- ANSI rendering wrong → SGR parsing; span merging. _(to be written)_

### Push notifications _(feature not built)_

- Permission denied / not delivered / iOS-only quirks / EU unavailability.
  _(to be written when the feature exists)_

## Contributing to this page

When you fix a non-obvious problem, add it here: the **symptom** as someone would
search for it, the **cause**, and the **fix**. A troubleshooting entry per real
bug is as valuable as the fix.
