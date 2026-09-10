# ADR 0001: All TypeScript, one workspace

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow has three code artifacts: a **daemon** that holds long-lived
socket connections and serves an API, a **web client**, and the **wire protocol**
that binds them. We need to choose the implementation language(s).

Relevant forces:

- **The workload is I/O-bound, not CPU-bound.** The daemon mostly holds idle
  sockets, relays lines, and does light text processing (later: regex matching
  for triggers). It is not doing heavy computation.
- **Small, trusted user base.** Friends-scale, self-hosted. We are not chasing
  tens of thousands of concurrent connections per node.
- **The maintainer is fluent in TypeScript and Angular.** Velocity matters for a
  volunteer project; an unfamiliar stack would slow the first year significantly.
- **The protocol must not drift** between the two ends.
- **Long uptime.** The daemon is meant to run for months. Memory discipline
  matters.
- We considered a **Rust or Go daemon** paired with a TypeScript client, for a
  smaller resource footprint and no GC pauses.

## Decision

**We will write the entire project in TypeScript**, on **Node.js LTS**, in a
single **pnpm workspace** with three packages:

- `packages/protocol` — zod schemas and inferred types for every message and DTO;
  a codec abstraction. No runtime dependency beyond zod. Imported by both other
  packages.
- `packages/daemon` — the service, on Node.js LTS.
- `packages/web` — the Angular PWA.

**We will pin to an active Node LTS line** and prefer standard/WHATWG APIs where
they exist, isolating the genuinely Node-specific pieces (the `net`/`tls`
listeners, the native SQLite/Postgres driver) behind interfaces, so that a future
runtime change is contained.

## Consequences

**Easier:**

- One language, one toolchain, one mental model. One `pnpm` install, one lint/
  format config, one test runner.
- The protocol is a shared package the compiler checks on both sides — the type
  and the runtime validation are literally the same source.
- Fast onboarding for the maintainer and for the large pool of TypeScript
  developers.
- Node's `net`/`tls` and the mature `pg` driver are exactly what the daemon
  needs; thousands of idle sockets are not a problem for it.

**Harder / to watch:**

- **Long-uptime memory discipline** is on us: bounded buffers, no unbounded
  caches, care with closures capturing large objects. (We want bounded scrollback
  buffers anyway.)
- Single-threaded by default. If some future feature is CPU-heavy (it isn't
  today), it goes in a `worker_thread`, not the main loop.
- We forgo the tighter resource footprint a Rust/Go daemon would have. At
  friends-scale this is not a real cost; at a hypothetical much larger scale it
  could be reconsidered.

**Committed to:**

- Keeping `packages/protocol` dependency-light and the single source of wire
  truth.
- Node LTS; upgrading as LTS lines roll.

## Alternatives considered

- **Rust daemon + TypeScript client.** Best footprint and no GC, but two
  toolchains, a wire protocol duplicated across the language boundary (kept in
  sync by hand or codegen), and a materially slower first year. Over-investing in
  the runtime while under-investing in features, for this scale.
- **Go daemon + TypeScript client.** Goroutine-per-connection is a clean fit and
  deployment is a single binary, but it still means two languages and a
  duplicated protocol for no capacity benefit we actually need.
- **Bun or Deno instead of Node.** Attractive DX and speed, but less production
  soak time for multi-month-uptime socket servers and a thinner ecosystem for the
  native bits. We keep the code reasonably portable so this can be revisited, but
  Node LTS is the safe default for a daemon whose job is to stay up.
