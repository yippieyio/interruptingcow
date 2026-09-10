# Backup and Restore

> **Status: design phase.** No software, no deployment, and — importantly — **this
> procedure has not been tested**. Do not trust it until someone has performed a
> full restore from a real backup and updated this document to say so. A backup
> you have never restored is a hypothesis, not a safety net.

## What must be backed up

| Item | Where (planned) | If you lose it |
|---|---|---|
| **PostgreSQL data** | host bind-mount, e.g. `/srv/icow/pgdata` | everything: accounts, worlds, session state, **all logs** |
| **`ICOW_WORLD_CRED_KEY`** | `/srv/icow/secrets/` | stored game credentials become undecryptable; users must re-enter them |
| **`ICOW_JWT_SECRET`** | `/srv/icow/secrets/` | all active sessions invalidated (users log in again); not catastrophic |
| **PostgreSQL password** | `/srv/icow/secrets/` | you can reset it, but back it up with the rest |
| **`deploy/` config** (`pod.yaml`, `.env`, proxy config) | your deploy checkout / bundle | reconstructable, but back it up to save time |
| **WAL archive** (if you enable PITR) | a separate host path or object storage | you lose point-in-time recovery between full dumps |

The database is the irreplaceable part. The secrets are small and must be backed
up **separately and encrypted** — a database backup plus the `WORLD_CRED_KEY`
together is enough to read stored game credentials, so don't keep them in the
same place unprotected.

## Backup

### Baseline: nightly logical dump

```sh
# Run from the host, or in a sidecar with the postgres client.
podman exec icow-postgres pg_dump \
    --format=custom --compress=9 \
    --file=/tmp/icow-$(date +%F).dump \
    "$ICOW_DATABASE_URL"

podman cp icow-postgres:/tmp/icow-$(date +%F).dump /srv/icow/backups/
# then copy /srv/icow/backups/ off-host (rsync, restic, rclone, …)
```

Schedule it with cron/systemd-timer. Keep a rotation (e.g. 7 daily, 4 weekly).
`pg_dump --format=custom` restores with `pg_restore` and supports selective
restore.

### Also back up the secrets and config

```sh
tar -czf - -C /srv/icow secrets deploy | \
    age -r <your-age-recipient> > /srv/icow/backups/icow-secrets-$(date +%F).tar.gz.age
# copy off-host
```

Use real encryption (`age`, `gpg`) for anything containing a key.

### Optional: WAL archiving for point-in-time recovery

Configure PostgreSQL `archive_mode = on` and an `archive_command` that ships
completed WAL segments to `/srv/icow/wal-archive/` (and off-host). With a base
backup plus the WAL archive you can restore to any moment, not just the last
nightly dump. This is worth setting up once the log volume makes a lost day
painful.

## Restore

> Practice this on a throwaway host **before** you need it. Then update the
> "tested" note at the top of this file.

### Full restore from a logical dump

```sh
# 1. Stop the pod (or at least the daemon) so nothing writes during the restore.
podman pod stop icow

# 2. Bring up a fresh, empty PostgreSQL with the SAME major version, pointed at a
#    clean data directory. (Or drop and recreate the database in place.)

# 3. Restore
podman cp /srv/icow/backups/icow-<date>.dump icow-postgres:/tmp/restore.dump
podman exec icow-postgres pg_restore \
    --clean --if-exists --no-owner \
    --dbname="$ICOW_DATABASE_URL" \
    /tmp/restore.dump

# 4. Restore the secrets and config to /srv/icow/secrets and deploy/ if lost.

# 5. Start the pod
podman kube play --replace deploy/pod.yaml

# 6. Verify
curl -fsS https://your-domain/healthz
podman exec -it icow-daemon icow migrate      # apply any migrations newer than the dump
```

### Point-in-time recovery (if WAL archiving is enabled)

Restore the most recent base backup into a clean data directory, set up a
`recovery` configuration pointing at the WAL archive with your target time or
LSN, and start PostgreSQL in recovery mode. Follow the PostgreSQL documentation
for your major version; the specifics change between versions and this document
will pin them once we actually run a supported version.

## After any restore

- **Session state may be stale.** `session_state` reflects the moment of the
  backup. On daemon start, our-fault recovery will look at those checkpoints and
  the grace window; a restore hours later is *outside* the grace window, so
  worlds will come up `disconnected` and users reconnect them. This is correct
  behavior, not data loss — the *logs* are intact.
- **Verify a sample.** Log in as a test user, open a world's history, run a
  full-text search, confirm recent lines are present up to the backup point.
- **Check the secrets round-trip.** If you restored `ICOW_WORLD_CRED_KEY`, a
  world with stored credentials should still connect with auto-login.

## What a hard power loss looks like (no restore needed)

An ungraceful shutdown — `podman pod kill`, a real power cut — is **not** a
restore scenario in the normal case:

- PostgreSQL replays its WAL on the next start and comes up consistent.
- Committed data (every line that reached the database) is intact.
- The log writer flushes its batch on a *clean* shutdown; a hard kill can lose
  the single in-flight batch of the most recent scrollback. Nothing else.

Restore from backup only when the data directory itself is damaged (disk
failure), not merely when the process died uncleanly.
