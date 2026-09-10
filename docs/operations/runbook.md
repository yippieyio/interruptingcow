# Operations Runbook

> **Status: design phase.** There is no software to run and no deployment
> artifacts yet. This runbook describes the **intended** operational model so it
> can be reviewed now and is ready when the daemon and `deploy/` exist. Commands
> are illustrative.

## Audience

Someone self-hosting InterruptingCow for a small group — on a VPS or a home
server, a single host. Not a cluster operator.

## What you're running

A **Podman pod** with three containers ([ADR 0005](../adr/0005-podman-pod-external-state.md)):

| Container | Role | State |
|---|---|---|
| `reverse-proxy` | TLS termination, serves the web app, one origin, forwards `/api` and `/stream` | none (config only) |
| `daemon` | the service | **none on disk** |
| `postgres` | the datastore | **data directory bind-mounted from the host** |

Critical rule: **durable state lives on host volumes, never inside the pod.**

## Host prerequisites

- Podman (rootless is fine and preferred).
- A DNS name pointing at the host, and ports 80/443 reachable (the proxy handles
  ACME).
- A host directory for PostgreSQL data, e.g. `/srv/icow/pgdata`, and one for
  secret material, e.g. `/srv/icow/secrets/` (mode `700`).
- A backup destination (a second disk, another host, or object storage).

## First install _(planned)_

```sh
# 1. Get the deploy bundle for the release you want (attached to the GitHub Release)
#    or clone the repo at the tag.

# 2. Prepare host directories
mkdir -p /srv/icow/pgdata /srv/icow/secrets
chmod 700 /srv/icow/secrets

# 3. Create the secrets (never commit these, never bake into an image)
#    - the PostgreSQL password
#    - ICOW_JWT_SECRET
#    - ICOW_WORLD_CRED_KEY  (encrypts stored game credentials at rest)
#    Put them where deploy/ expects (documented in deploy/README.md).

# 4. Configure
cp deploy/.env.example deploy/.env
#    Set the domain, the database URL, token lifetimes, the recovery grace
#    window, the log level. Every key is documented in the file.

# 5. Bring up the pod
podman kube play deploy/pod.yaml

# 6. Initialise the database and create accounts
podman exec -it icow-daemon icow migrate
podman exec -it icow-daemon icow user add alice
podman exec -it icow-daemon icow world add alice --name ExampleMUSH \
    --host mush.example.net --port 4201
```

Then open `https://your-domain/`, log in as the account you created, and add it
to your home screen.

## Everyday operations

### Check health

```sh
podman pod ps
podman ps --pod
curl -fsS https://your-domain/healthz     # checks DB pool + event-loop
podman logs --tail 100 icow-daemon
```

`/healthz` is what the proxy and the pod healthcheck use. A failing `/healthz`
means the daemon can't reach PostgreSQL or its event loop is stalled.

### Manage users and worlds

```sh
podman exec -it icow-daemon icow user add <handle>
podman exec -it icow-daemon icow user passwd <handle>
podman exec -it icow-daemon icow user remove <handle>
podman exec -it icow-daemon icow world add <handle> --name <n> --host <h> --port <p> [--tls] [--encoding <enc>]
podman exec -it icow-daemon icow world list <handle>
podman exec -it icow-daemon icow token mint <handle>
```

There is **no self-serve signup**; accounts are created here.

### Read the logs

Structured JSON via `podman logs`. The daemon **never logs message content,
passwords, tokens, or game credentials** — only identifiers and lifecycle
events. If you need more detail temporarily, raise `ICOW_LOG_LEVEL` to `debug`
and restart the daemon container; lower it again afterward.

## Upgrades

```sh
# 1. Note the current image digest (for rollback)
podman inspect --format '{{.ImageDigest}}' icow-daemon

# 2. Back up first (see below). Always.

# 3. Pull the new pinned tag and re-play the pod
#    (edit deploy/pod.yaml to the new tag, or use the new deploy bundle)
podman kube play --replace deploy/pod.yaml

# 4. Run migrations
podman exec -it icow-daemon icow migrate

# 5. Verify
curl -fsS https://your-domain/healthz
```

Expect a brief interruption while the pod restarts. Active game connections are
dropped by the restart and then **re-established by our-fault recovery** if they
were connected and within the grace window
([ADR 0006](../adr/0006-connection-recovery-semantics.md)).

### Rollback

```sh
# Re-play the pod at the previous image tag/digest.
podman kube play --replace deploy/pod.yaml    # with the tag reverted

# If the upgrade ran a migration that the old version can't tolerate, run its
# down-migration first:
podman exec -it icow-daemon icow migrate --down <version>
```

Every migration ships with a tested rollback. A migration that is genuinely
irreversible (rare, and post-1.0 it forces a MAJOR release) will say so in the
release notes — for those, restore from backup instead.

## Backups

See [backup-restore.md](backup-restore.md) for the full procedure. The essentials:

- Scheduled `pg_dump` (a nightly cron is the baseline), written to the host and
  copied off-host.
- Optionally, WAL archiving for point-in-time recovery.
- Also back up `/srv/icow/secrets/` — **without `ICOW_WORLD_CRED_KEY` you cannot
  decrypt stored game credentials**, and without the JWT secret all sessions are
  invalidated on restore.
- **Test the restore.** A backup you haven't restored is a hypothesis. The
  restore procedure is marked *untested* in the docs until someone has actually
  run it end to end.

## Incident response

### The daemon is crash-looping

1. `podman logs icow-daemon` — the reason is almost always at startup:
   invalid configuration (zod validation) or PostgreSQL unreachable.
2. Fix the config / bring PostgreSQL up.
3. If it started after a bad upgrade, [roll back](#rollback).

### PostgreSQL won't start after an ungraceful shutdown / power loss

1. This is expected to be **recoverable** — PostgreSQL replays its WAL on
   startup. Give it time; watch `podman logs icow-postgres`.
2. Committed data is intact. At most one un-flushed batch of very recent
   scrollback is lost (the log writer flushes on clean shutdown; a hard kill can
   drop the in-flight batch).
3. If the data directory is genuinely corrupt (disk failure, not just an unclean
   stop), [restore from backup](backup-restore.md).

### Disk is full

- The `lines` table is the growth. It is **partitioned by month**; drop old
  partitions per your retention policy (documented once retention tooling
  exists). `pg_dump` old partitions off-host first if you want to keep them.

### A user reports lost connections after a restart

- Check whether the restart was within `ICOW_RECOVERY_GRACE_SECONDS`. A longer
  outage intentionally does **not** auto-recover.
- Check the world's `auto_recover` flag.
- A connection the **game server** dropped is never auto-reconnected by design —
  the user reconnects it from the client. This is
  [ADR 0006](../adr/0006-connection-recovery-semantics.md), not a bug.

### Suspected security incident

Follow [`SECURITY.md`](../../SECURITY.md). Rotate `ICOW_JWT_SECRET` (invalidates
all sessions), consider rotating `ICOW_WORLD_CRED_KEY` (requires re-entering
world credentials), review access logs, and preserve logs before restarting.

## Monitoring _(planned)_

Baseline signals worth alerting on once metrics exist:

- `/healthz` failing.
- Daemon restart rate.
- PostgreSQL connection saturation; replication/WAL archiving lag if used.
- Disk usage on the data volume.
- Log-writer queue depth (back-pressure = the database can't keep up).
- Count of Sessions in `disconnected` trending up (a game server or the host's
  network may be having problems).
