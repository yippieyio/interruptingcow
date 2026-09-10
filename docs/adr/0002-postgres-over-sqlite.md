# ADR 0002: PostgreSQL over SQLite

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow needs one datastore for: user accounts and tokens, world
configuration, per-session checkpoint state, and — the demanding part — a
**complete, full-text-searchable log of every line to and from every world**.

Forces:

- **Log volume grows fast.** A single active role-play session can produce a
  megabyte of text in an evening. Every user, every world, indefinitely. This is
  the bulk of the data by orders of magnitude.
- **Full-text search across all history** is a product feature, not a
  nice-to-have.
- **Backups must be straightforward** and the data must survive a hard power
  loss.
- **Zero-knowledge per-user log encryption is a real post-MVP goal.** Encrypted
  logs cannot be server-side full-text indexed at all — search for those worlds
  moves to the client, or to per-user encrypted indexes.
- Self-hosted at friends-scale; the operator should not need to be a DBA.
- SQLite would be the lightest possible option (one file, trivial backup, no
  server process).

## Decision

**We will use PostgreSQL as the single datastore for everything** — accounts,
tokens, worlds, session state, and the line logs.

- Line logs go in a `lines` table with a generated `tsvector` column and a GIN
  index for full-text search, **partitioned by month** so retention is "drop a
  partition" and the index stays healthy as volume grows.
- The future encrypted path is a separate `lines_enc` table of ciphertext blobs
  (added in a later migration); those worlds bypass server-side search.
- Access goes through a thin **repository layer**; no ORM. Nothing outside that
  layer touches the driver.
- The Postgres data directory lives on a **host volume, outside the container**
  (see [ADR 0005](0005-podman-pod-external-state.md)). Backups are `pg_dump`
  plus optional WAL archiving.

## Consequences

**Easier:**

- Handles many gigabytes of log across many large per-world histories without the
  single-writer and single-file limitations that would eventually bite SQLite
  here.
- Mature full-text search (`tsvector`/`tsquery`, GIN) and native table
  partitioning for retention.
- Concurrent readers and writers without application-level locking gymnastics —
  the log writer, REST history queries, and recovery can all hit the database at
  once.
- Streaming-friendly for large result sets; `pg_dump`/restore and WAL archiving
  are well-understood operational tools.
- The right substrate for the encryption work: store ciphertext as `bytea`,
  switch search strategy per world, no engine change.

**Harder / the cost:**

- **A server process to run and back up.** More than "copy one file". Mitigated
  by shipping it as a container in the pod and documenting the procedure; an
  external managed Postgres is also supported.
- The operator has a real database to care about (upgrades, `pg_dump` schedule).
  The [runbook](../operations/runbook.md) covers this.
- Slightly more moving parts in local development (a Postgres for integration
  tests) — handled with a throwaway container.

**Committed to:**

- The repository layer as the only SQL boundary.
- Every migration having a tested rollback.
- Keeping the schema partition-friendly for `lines`.

## Alternatives considered

- **SQLite (single file, FTS5).** The lightest option and the easiest backup. A
  single file genuinely handles many GB and FTS5 is capable. Rejected because the
  log workload is write-heavy and unbounded, multi-writer concurrency is real
  here, and the encryption future points at Postgres anyway — we would likely be
  migrating within a year, and migrations always slip. Starting on Postgres
  avoids a forced move later.
- **SQLite for metadata + append-only log files per world.** Grep-able, cheap to
  archive, easy to hand a user their own logs. Rejected: two backup targets,
  filesystem-bound search, and a lot of custom code for rotation, retention, and
  indexing that Postgres gives us for free.
- **A separate search engine (e.g. an inverted-index service) alongside a
  primary store.** Rejected as far too much operational weight for friends-scale;
  Postgres full-text search is more than sufficient here.
- **A document database.** No advantage for this highly relational, append-heavy,
  search-oriented workload, and weaker durability/backup story for a self-hoster.
