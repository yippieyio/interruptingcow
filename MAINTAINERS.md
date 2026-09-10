# Maintainers

This file lists the people responsible for InterruptingCow. See
[`GOVERNANCE.md`](GOVERNANCE.md) for what these roles mean and how they change.

## Lead maintainer

| Name | GitHub | Areas |
|---|---|---|
| pepper | **TODO: @handle** | Everything (project founder) |

> The lead maintainer is currently the sole maintainer. Contact address for
> project matters that aren't security or conduct: **TODO** (add before the
> project accepts outside contributions).

## Maintainers

_None yet besides the lead maintainer._

## Committers

_None yet._

## Area ownership

This table mirrors the intended `.github/CODEOWNERS` (which does not exist yet).
Changes to an owned area require review from its owner.

| Area | Path(s) | Owner |
|---|---|---|
| Wire protocol | `packages/protocol/**`, `docs/protocol.md` | Lead maintainer |
| Daemon core (sessions, pipeline, recovery) | `packages/daemon/src/session/**` | Lead maintainer |
| Persistence & migrations | `packages/daemon/src/persistence/**` | Lead maintainer |
| Authentication | `packages/daemon/src/auth/**` | Lead maintainer |
| Upstream / telnet | `packages/daemon/src/gateways/upstream/**` | Lead maintainer |
| Web client | `packages/web/**` | Lead maintainer |
| Deployment | `deploy/**` | Lead maintainer |
| Docs & ADRs | `docs/**` | Lead maintainer |

## Emeritus

_None._

## How to become a maintainer or committer

See [Roles](GOVERNANCE.md#roles) in `GOVERNANCE.md`. In short: sustained,
high-quality contribution and good judgment in review, nomination by an existing
maintainer, confirmation by consensus. There is no contribution quota.
