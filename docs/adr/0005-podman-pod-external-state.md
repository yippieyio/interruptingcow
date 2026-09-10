# ADR 0005: Podman pod for deployment; durable state on host volumes

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow is self-hosted at friends-scale. It has three runtime pieces: a
reverse proxy (TLS termination, static assets, single origin), the daemon, and
PostgreSQL. We need a deployment shape that is simple for one operator to run and
that meets a hard requirement:

> **All persistent state must survive a hard power loss, and nothing durable may
> be lost when containers are recreated.**

Forces:

- The operator runs a single host (a VPS or a home server), not a cluster.
- Rootless containers are preferable for a home/VPS setup.
- The proxy and the daemon are **disposable** — they hold no state.
- PostgreSQL's data directory, the game-credential encryption key material, and
  any log archives are **not** disposable.
- Redeploys (new image, restart) happen and must be safe.

## Decision

**We will deploy as a Podman pod** of three containers sharing a network
namespace: `reverse-proxy`, `daemon`, `postgres`.

- The **`postgres` container may run in the pod, but its storage may not.** Its
  data directory is a **host bind-mount** (e.g. `/srv/icow/pgdata`). Recreating
  the pod or any container must not touch it. The same applies to the
  game-credential key material and any log-file archives.
- An **external or managed PostgreSQL is a fully supported alternative**: set a
  connection string and drop the `postgres` container. Either way the storage is
  outside the pod's writable layer.
- PostgreSQL runs with `fsync=on`, `synchronous_commit=on`,
  `full_page_writes=on`, `wal_level=replica`.
- The daemon holds **no on-disk state**.
- The pod restarts the daemon on failure; the daemon exposes `/healthz`
  (database pool + event-loop check) that the proxy and pod healthcheck consult.
- **Deployment is pull-based and manual** for now: the operator re-pulls a pinned
  image tag and restarts the pod, per the [runbook](../operations/runbook.md).
  Automated deploy is a documented future step, not part of the initial build.
- Backups: scheduled `pg_dump`, optionally WAL archiving to a second host path or
  offsite. The restore procedure is documented and is marked **untested until it
  has actually been exercised**.

## Consequences

**Easier:**

- One `podman kube play` (or equivalent) brings the whole system up.
- Redeploying the daemon or proxy is safe by construction — they have nothing to
  lose.
- Backup is a database concern with standard tools, against a directory the
  operator controls directly.
- Rootless Podman fits the target hosts.

**Harder / to watch:**

- The operator must set up the host directory and its permissions correctly, and
  must actually run and test backups. The runbook covers this, and the durability
  claims are only as good as the operator's backup discipline.
- Podman pod networking and volume semantics are a thing the operator has to
  learn a little of.
- No built-in HA or rolling deploy. Acceptable at this scale; a brief restart
  window is fine.

**Committed to:**

- State never living in a container's writable layer.
- The PostgreSQL durability settings above.
- Keeping the deployment reproducible from the committed `deploy/` artifacts.

## Alternatives considered

- **Docker / docker-compose.** Equivalent capability; Podman chosen for rootless
  operation and pod semantics on the target hosts. The compose file could be
  provided as an alternative later.
- **Bare-metal systemd services (no containers).** Fewer layers, but more
  host-specific setup and a messier upgrade story; containers give reproducible
  builds and clean rollbacks.
- **Kubernetes.** Wildly disproportionate for one host and one operator.
- **Postgres data in a named container volume instead of a host bind-mount.**
  Rejected: less transparent to back up and inspect, and easier to destroy by
  accident with a stray `podman volume` command. A host path the operator chose
  is clearer and safer for this audience.
- **A managed database from the start.** Supported, not required — imposing a
  cloud dependency on a self-hoster who wants everything on one box would be the
  wrong default.
