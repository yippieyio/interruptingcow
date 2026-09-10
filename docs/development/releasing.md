# Releasing

> **Status: design phase.** Nothing has been released and the release automation
> doesn't exist yet. This is the intended policy and process.

## Versioning

InterruptingCow follows **[Semantic Versioning 2.0.0](https://semver.org/)**:
`MAJOR.MINOR.PATCH`.

### Pre-1.0

While the version is `0.x`:

- The **wire protocol** and the **database schema** may change in a **minor**
  release.
- Every such change is called out explicitly in [`CHANGELOG.md`](../../CHANGELOG.md)
  and, for the protocol, in an [ADR](../adr/0000-adr-process.md).
- `PATCH` releases remain strictly bug fixes.

### At and after 1.0

- **MAJOR** — a breaking change to the wire protocol, the REST API, the CLI, or
  the supported deployment; or a database migration that cannot be rolled back.
- **MINOR** — backward-compatible features. Additive protocol changes (new
  optional fields, new message types a client may ignore) are minor.
- **PATCH** — backward-compatible bug fixes only.

### Protocol version vs. release version

The wire protocol has its **own integer version**, carried in the `hello`
handshake (see [the protocol document](../protocol.md#versioning-and-compatibility)).
It moves independently of the release version:

- Additive protocol change → no protocol-version bump.
- Breaking protocol change → protocol-version bump, plus at least a MINOR
  (pre-1.0) or MAJOR (post-1.0) release, plus an ADR.

The daemon advertises a minimum and current supported protocol version and
rejects clients below the minimum. This is what lets the client and daemon ship
on different days.

## Cutting a release _(planned)_

Releases are cut from `main` by a maintainer.

1. Ensure `main` is green and everything intended for the release is merged.
2. Confirm `CHANGELOG.md`'s `[Unreleased]` section is accurate (it's assembled
   from [Conventional Commit](../../CONTRIBUTING.md#commit-messages) messages).
   Move it under a new `## [X.Y.Z] - YYYY-MM-DD` heading.
3. Tag: `git tag -s vX.Y.Z -m "vX.Y.Z"` (signed once a signing key is set up),
   and push the tag.
4. The `release.yml` workflow _(planned)_ then:
   - re-runs the full build and test suite against the tag,
   - builds the multi-arch `daemon` container image and pushes it to **GHCR**
     (`ghcr.io/OWNER/interruptingcow-daemon:X.Y.Z` and `:latest`),
   - attaches a **versioned deploy bundle** (`deploy/` plus the built web assets)
     to the GitHub Release,
   - generates the release notes from the commits since the previous tag,
   - records the image digest in the release notes.
5. The release uses only `GITHUB_TOKEN` / GHCR credentials — it does not run for
   fork PRs and does not expose secrets to them.

## Supported versions

Once releases begin, [`SECURITY.md`](../../SECURITY.md) states which versions get
security fixes. The intent: the latest minor, and the previous minor for a
transition period.

## Upgrading an instance

Operator-side upgrade and rollback steps live in the
[runbook](../operations/runbook.md). In short: pull the new pinned image tag,
run `icow migrate`, restart the pod; roll back by pulling the previous tag and,
if a migration must be undone, running its down-migration.

## Deployment

Deployment to the reference/production host is **pull-based and manual**: an
operator re-pulls a pinned tag and restarts. There is no automated deploy in the
initial scope; a `deploy.yml` (SSH or Podman-remote) is a possible later
addition and would get its own ADR if it changes the security posture.
