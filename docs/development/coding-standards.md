# Coding Standards

> **Status: design phase.** These are the standards the codebase will be held to.
> They're written now so the first code follows them rather than retrofitting.
> Tooling that enforces a rule is noted; where there's no tool yet, it's a review
> expectation.

## Language and tooling

- **TypeScript**, `strict` mode on. No implicit `any`; no `// @ts-ignore` without
  a comment explaining why and a linked issue.
- **ESLint + Prettier** _(planned)_ define formatting and lint rules. Formatting
  is not a matter of opinion in review — the formatter decides. `pnpm format`
  applies it; `pnpm verify` checks it.
- **ES modules**, `type: "module"`. Target the pinned Node LTS.
- Prefer standard/WHATWG APIs over Node-only ones where equivalent; isolate the
  genuinely Node-specific pieces behind interfaces (see
  [ADR 0001](../adr/0001-all-typescript.md)).

## Project structure and boundaries

The module boundaries are the architecture. Respect them:

- **`packages/protocol` is the only cross-process contract.** If the daemon and
  the client need to agree on a shape, it's a zod schema there — nowhere else.
- **The repository layer is the only code that touches SQL / the database
  driver.** Everything else depends on `*Repo` interfaces. No query strings
  outside `packages/daemon/src/persistence/`.
- **The telnet/upstream layer never imports persistence or HTTP.** It speaks
  bytes and events.
- **The line pipeline is the single hook point** for logging, triggers, and
  notifications. Don't add a second path for "just this one feature".
- **Stubs implement the same interface as the real thing.**
  `NoopTriggerEngine`, `NoopNotifier`, `NullTerminalGateway` are swapped in the
  composition root, not special-cased by callers.
- Construction and wiring happen in **`main.ts` (the composition root)**. Modules
  receive their dependencies; they don't reach out and build them.

## Naming

- Files: `kebab-case.ts`. One primary export per file where practical; the file
  name matches it (`session-manager.ts` → `SessionManager`).
- Types and classes: `PascalCase`. Interfaces are **not** prefixed with `I`
  except for the small set already established in the architecture
  (`ITransport`, `ITriggerEngine`, `INotifier`, `ITerminalGateway`,
  `*Repo`) — match the surrounding code.
- Functions and variables: `camelCase`. Constants that are truly constant:
  `SCREAMING_SNAKE_CASE`.
- Booleans read as assertions: `isConnected`, `hasScrollback`, `canRetry`.
- Avoid abbreviations except well-known ones (`id`, `url`, `db`, `ws`).

## Async and resources

- `async`/`await`; no bare promise chains for control flow.
- Every `await` that can reject is either handled or deliberately allowed to
  propagate to a boundary that handles it. No unhandled rejections.
- **Bounded everything.** Buffers (scrollback), queues (the log writer), retry
  counts (recovery) all have explicit limits. This is a long-uptime daemon;
  unbounded growth is a bug.
- Clean up: sockets, timers, listeners, and database clients are released on the
  paths that acquire them, including error paths. Prefer a `try/finally` or a
  disposable pattern over hoping.
- Respect back-pressure. The log writer must not accept work faster than the
  database drains it.

## Error handling

- **Throw `Error` (or a subclass), never a string or a plain object.**
- Define a small set of typed errors for conditions callers need to distinguish
  (e.g. `AuthError`, `UnknownWorldError`, `ProtocolError`). Don't invent a new
  class per call site.
- **Errors crossing the wire** become the protocol's `error` message with a
  stable `code`; the internal error and stack stay server-side in the log.
- Fail fast on programmer error and invalid config (refuse to start). Be
  resilient to *expected* runtime failure (a world unreachable, a server
  dropping us) — that's a state, not a crash.
- Never swallow an error silently. If it's genuinely ignorable, log it at
  `debug` with a reason.

## Logging

- **Structured logging** (pino is the intended library). Log objects with fields,
  not interpolated strings: `log.info({ worldId, state }, "session state changed")`.
- Levels: `error` (needs attention), `warn` (unexpected but handled), `info`
  (significant lifecycle events), `debug` (developer detail, off in production).
- **Never log secrets or message content.** No passwords, tokens, game
  credentials, or lines of user scrollback in logs. Log identifiers and counts,
  not payloads.
- Include correlating ids (`worldId`, `userId`, request id) so a session can be
  traced.

## Comments and documentation

- **TSDoc on every exported symbol** — what it's for, not a restatement of the
  signature. Note invariants, units, and ownership ("caller must close the
  returned handle").
- **Stub classes carry a doc comment naming their future implementation** and
  what will trigger building it.
- Comment the **why**, especially where the code is deliberately not the obvious
  thing (the recovery grace window, the no-retry-on-server-close rule — link the
  ADR).
- Match the comment density and style of the surrounding code.
- Keep `docs/` in sync when behavior, config, or the protocol changes — same PR.

## Configuration

- All configuration is loaded once, **validated against a zod schema at boot**,
  and passed as a typed object. No `process.env` reads scattered through the
  code.
- Every key is documented in `.env.example` with a safe default or an obvious
  placeholder.
- "It started" must imply "the config is valid".

## Dependencies

- Add a dependency deliberately. Prefer the standard library and small,
  well-maintained packages. A large dependency for a small need gets pushback in
  review.
- A structurally significant new dependency (a framework, a datastore driver, a
  crypto library) warrants an [ADR](../adr/0000-adr-process.md).
- `packages/protocol` stays dependency-light — zod and nothing else at runtime.
- Pin versions; commit the lockfile; let Dependabot _(planned)_ do the bumping.

## Tests

Full detail in [testing.md](testing.md). The standards that are also *coding*
standards:

- Tests live beside the code (`*.spec.ts`); integration tests that need a real
  PostgreSQL go in `test/integration/`.
- **Every bug fix includes a regression test** that fails before the fix.
- **Every migration includes a test** that applies and rolls it back.
- Test behavior through public interfaces, not private internals.
- No network calls to real services in tests — the fake MU\* server stands in for
  game servers; a throwaway PostgreSQL stands in for the database.
- Deterministic: no reliance on wall-clock timing, on the ordering of unordered
  collections, or on a specific free port. Inject clocks and randomness so tests
  control them.

## Security-adjacent code

- Password hashing is Argon2id via a vetted library — never hand-rolled.
- Token handling: short-lived access tokens, rotating refresh tokens, server-side
  revocation. Constant-time comparison for secrets.
- Treat **all** input from game servers and from clients as hostile: validate at
  the boundary (zod for messages, the telnet state machine for upstream bytes),
  bound the sizes, and never `eval` or interpolate it into anything.
- Game credentials and (later) log-encryption keys never appear in logs, errors,
  API responses, or the container image.

## Commit and PR hygiene

- [Conventional Commits](../../CONTRIBUTING.md#commit-messages); DCO `-s`
  sign-off on every commit.
- One logical change per PR. Refactors separate from behavior changes.
- The PR description says what and why and links the issue and any ADR.
