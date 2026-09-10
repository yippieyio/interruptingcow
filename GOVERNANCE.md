# Project Governance

This document describes how decisions are made in InterruptingCow and how people
take on responsibility for the project.

> **Current state:** the project is maintained by a single person (see
> [`MAINTAINERS.md`](MAINTAINERS.md)) and is in its design phase. The model below
> is written for where the project intends to go. Until there is a second
> maintainer, "the maintainers" and "the lead maintainer" are the same person,
> and the lead maintainer decides. The point of writing this down now is so the
> transition to a real team is a matter of adding names, not inventing process.

## Principles

1. **Decisions are made in the open.** Significant choices happen on issues,
   pull requests, Discussions, or Architecture Decision Records — not in private
   channels. Private discussion may happen, but the decision and its rationale
   are recorded publicly.
2. **Architecture changes are written down.** Any change to the shape of the
   system goes through an [ADR](docs/adr/0000-adr-process.md).
3. **Lazy consensus.** Most proposals proceed if no one with standing objects
   within a reasonable review window. Silence is assent.
4. **The license is not up for renegotiation.** InterruptingCow is
   `AGPL-3.0-or-later` and contributions are inbound = outbound. Because the
   project uses a [DCO](CONTRIBUTING.md#developer-certificate-of-origin-dco) and
   not a CLA, there is no copyright assignment and a relicense would require the
   agreement of all copyright holders. This is intentional.

## Roles

### Contributor

Anyone who submits an issue, pull request, ADR proposal, documentation fix, or
substantive Discussion comment. No prior approval required. Contributors are
expected to follow [`CONTRIBUTING.md`](CONTRIBUTING.md) and the
[Code of Conduct](CODE_OF_CONDUCT.md).

### Committer

A contributor who has been granted write access to the repository. Committers
can:

- Merge pull requests that meet the [merge requirements](#merge-requirements).
- Triage issues (label, close, assign, milestone).
- Approve and run CI for fork PRs.

Committers are expected to stay within their area of competence and to defer to
the owner of an area (per `.github/CODEOWNERS`, planned) for changes there.

**Becoming a committer:** a maintainer nominates a contributor who has a track
record of good, sustained contributions and good judgment in review. The
nomination is confirmed by lazy consensus of the maintainers (no objection within
one week). There is no fixed contribution count; quality and reliability matter
more than volume.

### Maintainer

A committer who also takes responsibility for the health and direction of the
project or a major part of it. Maintainers:

- Set and adjust the [roadmap](ROADMAP.md).
- Own areas of the codebase in `.github/CODEOWNERS` (planned) and are the
  required reviewers there.
- Decide ADRs that don't reach consensus (see [Decision making](#decision-making)).
- Manage releases.
- Administer the repository (settings, branch protection, secrets, integrations).
- Handle Code of Conduct and security reports.

**Becoming a maintainer:** an existing maintainer nominates a committer who has
shown sustained ownership — not just merging PRs, but shepherding an area,
reviewing thoughtfully, mentoring contributors, and following through. Confirmed
by consensus of the existing maintainers. Added to `MAINTAINERS.md` and the
relevant `CODEOWNERS` entries in the same PR.

### Lead maintainer

One maintainer serves as the tie-breaker of last resort and the point of
accountability for the project as a whole (repository ownership, domain,
trademarks if any, final say when maintainers deadlock). Initially the project
founder. If the role needs to change hands, the lead maintainer nominates a
successor from among the maintainers; if the lead maintainer is unavailable, the
maintainers select one by simple majority.

## Decision making

| Kind of decision | How it's made |
|---|---|
| Routine change (bug fix, small feature, docs) | Normal PR review; one committer/maintainer approval + green CI |
| Change to an owned area (protocol, persistence, auth) | Requires approval from that area's owner in `CODEOWNERS` |
| Architectural change | [ADR](docs/adr/0000-adr-process.md), discussed as a PR; merged by lazy consensus of maintainers |
| Contested ADR | If consensus isn't reached, the maintainers decide by simple majority; the lead maintainer breaks a tie. The dissent is recorded in the ADR. |
| Roadmap / scope | Maintainers, informed by Discussions and issues |
| New maintainer / committer | Nomination + confirmation as described in [Roles](#roles) |
| License, governance, Code of Conduct changes | Consensus of **all** maintainers; for the license specifically, see the note in [Principles](#principles) |
| Security response | The maintainer(s) handling the report, per [`SECURITY.md`](SECURITY.md) |

A "reasonable review window" is normally one week for ADRs and governance
changes, shorter for routine work. Anyone may ask for more time to review.

## Merge requirements

A pull request may be merged when **all** of the following hold:

- CI is green (once CI exists).
- Every commit carries a valid DCO `Signed-off-by`.
- The PR title is a valid Conventional Commit.
- There is at least one approving review from someone other than the author.
- If it touches an owned area, that area's owner has approved.
- Unresolved review threads are resolved or explicitly deferred with agreement.

Merges to `main` are **squash merges** to keep history linear.

## Inactivity and emeritus status

A maintainer or committer who has been inactive for an extended period (roughly
six months, no hard rule) may be moved to **emeritus** in `MAINTAINERS.md` by the
other maintainers, after an attempt to make contact. Emeritus members keep the
credit, lose write access and required-reviewer status, and can be reinstated on
request without re-nomination.

## Changing this document

Changes to `GOVERNANCE.md` follow the process in the
[Decision making](#decision-making) table: consensus of all maintainers, proposed
as a PR, one-week review window.

## Acknowledgements

This governance model borrows from common practice in small-to-mid open-source
projects, including the Node.js, Docusaurus, and Contributor Covenant project
governance documents, adapted to a project that is currently one person with room
to grow.
