# Testing

> **Status: design phase.** No tests exist yet. This is the strategy the project
> commits to, so the first code is written test-first against it.

Testing is **first-class**. Every package ships practical unit tests from its
first commit; integration and end-to-end suites grow with the features they
cover; everything runs in CI on every push and pull request.

## The layers

| Layer | Scope | Tools _(planned)_ | Location | Runs |
|---|---|---|---|---|
| **Unit** | Pure logic in isolation: the codec, zod schemas, the telnet IAC state machine, the ANSI parser, the recovery grace-window policy, token rotation, the trigger-engine interface contract. No network, no database. | Vitest | `*.spec.ts` beside the source | every push / PR; must stay fast |
| **Integration** | A real, throwaway PostgreSQL. Migrations apply **and roll back**; every `*Repo` against real SQL; the log writer's batching, back-pressure, and shutdown flush; full-text search correctness; `SessionManager` against a fake upstream socket (detach-survives, server-drop-no-retry, our-fault recovery in and out of the grace window); HTTP routes and the WebSocket handshake. | Vitest + `pg` + node `net` fakes | `packages/*/test/integration/` | every push / PR |
| **Contract** | The daemon and the web client agree on the wire. For each protocol message, a sample is generated from its zod schema; both the daemon's serializers and the client's `WsClientService` must round-trip it. | Vitest + shared fixtures in `packages/protocol/test/samples/` | protocol + daemon + web | every push / PR |
| **Component (web)** | Angular pieces in isolation — ANSI-to-spans rendering, input history and tab-complete, the world-list status, the history search view — with a mocked `WsClientService`. | Vitest + Angular Testing Library | `packages/web/src/**/*.spec.ts` | every push / PR |
| **End-to-end** | The whole system: an ephemeral Podman pod (daemon + PostgreSQL + the **fake MU\* server**), driven through the PWA with Playwright. See the journeys below. | Playwright + Podman | `e2e/specs/` | PR + `main` |
| **Durability** | Ungraceful `podman pod kill` mid-session; bring the pod back; assert PostgreSQL recovers, all committed lines are present, at most one un-flushed batch of scrollback is lost, no corruption. | shell script + `psql` assertions | `e2e/specs/durability.sh` | `main` / nightly |
| **Static** | `eslint`, `tsc --noEmit`, `prettier --check`, dependency audit, CodeQL. | CI | — | every push / PR |

## The end-to-end journeys

The suite must cover, at minimum:

1. **Detach survives.** Log in → connect a world → exchange lines → fully close
   the browser → wait (have the fake server emit a "page" meanwhile) → reopen →
   scrollback backfills including what arrived while away → the live stream
   resumes.
2. **Server drop, no chase.** The fake server closes the connection → the client
   shows `disconnected` with `server-closed` → **no reconnect attempt is made** →
   an explicit reconnect works.
3. **Our-fault recovery.** Restart just the daemon within the grace window → a
   world that was intentionally connected comes back automatically → a world the
   user had disconnected stays disconnected.
4. **History search.** Full-text search returns hits across multiple sessions.
5. **PWA installability.** The web app manifest is valid and the service worker
   registers.
6. **Durability.** As in the table above.

## The fake MU\* server fixture

`e2e/fixtures/fake-mush/` is a small TypeScript TCP server that:

- speaks enough telnet to complete option negotiation (NAWS, TTYPE, EOR, SGA,
  CHARSET),
- emits scripted or canned output on command,
- can be told to drop the connection, stall, or send malformed IAC sequences.

It's used by the integration and end-to-end layers and is testable on its own.
**CI never connects to a real game server.** A manual smoke test against a real
public MUSH is a documented checklist in
[`CONTRIBUTING.md`](../../CONTRIBUTING.md), not an automated job.

## Conventions

- **Test behavior through public interfaces**, not private internals.
- **Deterministic.** Inject clocks and randomness; don't depend on wall-clock
  timing, on the iteration order of unordered collections, or on a specific free
  port.
- **Every bug fix adds a regression test** that fails before the fix and passes
  after.
- **Every migration has a test** that applies it to a database at the previous
  version and asserts the resulting shape, plus a rollback.
- Prefer a real throwaway PostgreSQL over mocking the database; prefer the fake
  MU\* server over mocking sockets.
- Keep unit tests fast enough that `pnpm test` stays a sub-10-seconds-per-package
  habit.

## Coverage

- Reported with `vitest --coverage` (v8) and surfaced on each PR.
- Target roughly **85% lines** on the daemon core — `session/`, `persistence/`,
  `auth/`, `gateways/upstream/telnet/`. The number is a guide, not a gate; what's
  enforced is **no regression on the PR's diff**.

## What CI enforces _(planned)_

Branch protection on `main` requires green:

- `ci.yml` — format check, lint, typecheck, unit + integration + contract +
  component tests (with a `postgres` service container).
- `e2e.yml` — build the images, stand up the pod, run Playwright; on `main` also
  run the durability script.

CodeQL runs advisory. Fork PRs run all of the above **without secrets** — the
suites are designed not to need any.
