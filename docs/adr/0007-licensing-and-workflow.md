# ADR 0007: Licensing, contributor sign-off, and development workflow

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow is intended to be a serious, long-lived, community-maintained
open-source project. Before code exists, we should fix the ground rules that are
expensive to change later: the license, how contributions are legally accepted,
and how work flows into the repository.

Forces:

- **The project should stay open.** The maintainer's stated goal is a guarantee
  that the software — and improvements to it — remain free software, including
  when someone runs it as a service for others.
- **InterruptingCow is fundamentally a network service.** A plain copyleft
  license (GPLv3) only obligates people who *distribute* the software; someone
  running a modified version as a hosted service for other people has no
  obligation to share their changes. That's precisely the case we care about.
- The maintainer is **not** trying to preserve a path to a proprietary or
  commercial-SaaS offering; "scuttling" that is acceptable.
- It's a **hobbyist project**; contribution friction should be low.
- Contributions come from a principal repo plus **forks** (outside contributors).
- `main` should stay releasable and its history readable.

## Decision

### License: AGPL-3.0-or-later

The project is licensed under the **GNU Affero General Public License, version 3
or later** (`AGPL-3.0-or-later`). The full text is in [`LICENSE`](../../LICENSE).

- The AGPL's §13 extends copyleft to network use: anyone who runs a modified
  InterruptingCow as a service must offer its users the corresponding source.
- Every source file carries an SPDX identifier:
  `// SPDX-License-Identifier: AGPL-3.0-or-later` (comment syntax per language).
- The **web client must expose a visible "Source" link** so users of a running
  instance can obtain the (possibly modified) source, satisfying §13 in practice.
- Contributions are **inbound = outbound**: by contributing you license your
  work under `AGPL-3.0-or-later`.
- A possible future **permissive carve-out for `packages/protocol`** (to let
  third-party clients exist without being AGPL) is explicitly *not* done now and
  would require its own ADR.

### Contributor sign-off: DCO, not a CLA

The project uses the **Developer Certificate of Origin 1.1**. Every commit must
carry a `Signed-off-by` line matching its author (`git commit -s`). The full DCO
text and instructions are in [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

- No CLA, no copyright assignment. Copyright stays with contributors.
- This means a future relicense would need every copyright holder's agreement —
  which is the intended safeguard, not an accident.
- Once CI exists, a check enforces the sign-off on every commit in a PR.

### Workflow: trunk-based, fork-and-PR, Conventional Commits

- **`main`** is always releasable and protected: no direct pushes, linear
  history, required review, required status checks once they exist.
- Work happens on **short-lived topic branches** (`feat/…`, `fix/…`, `docs/…`,
  etc.), one logical change each. No long-running feature branches — large
  features land as vertical slices behind stubs or flags.
- Outside contributors work from **forks** and open PRs against `main`.
- **[Conventional Commits](https://www.conventionalcommits.org/)** for messages;
  because PRs are **squash-merged**, the PR title must itself be a valid
  Conventional Commit (it becomes the commit subject). The changelog is generated
  from these.
- **Fork PR CI runs with least privilege** — no repository secrets, no image
  push, no deploy. The test suites are designed to need no secrets.
- Reviews are routed by `CODEOWNERS`; changes to the protocol, persistence, or
  auth require the area owner's approval.

The pipeline details (workflow files, release automation) are implementation and
will be built in the skeleton phase; this ADR fixes the *policy*.

## Consequences

**Easier / what we gain:**

- The openness guarantee actually covers the case that matters for this software
  (hosted service), not just redistribution.
- No CLA friction or infrastructure; contributing is `git commit -s` and a PR.
- Copyright stays distributed, which structurally protects the license choice.
- A readable, linear `main`; an auto-generated changelog; predictable review
  routing.

**Harder / the cost:**

- **AGPL deters some adopters** — companies with blanket AGPL bans won't deploy
  or contribute. This is an accepted, deliberate trade for a hobbyist project
  that values the guarantee over adoption breadth.
- The §13 source-offer obligation is a real requirement we must honor in the
  product (the "Source" link) and document for operators.
- DCO requires contributors to use a real name and email and to sign off; the CI
  check will reject unsigned commits, which occasionally trips people up (fixable
  with `git rebase --signoff`).
- Squash-merge loses intermediate commit granularity on `main` (acceptable; the
  PR retains it).

**Committed to:**

- `AGPL-3.0-or-later`, SPDX headers, and the client "Source" link.
- DCO sign-off on every commit; no CLA.
- Trunk-based development with protected `main` and Conventional Commits.

## Alternatives considered

- **GPL-3.0-or-later.** Strong copyleft, but no network-use trigger — a hosted
  modified version needn't share changes. Rejected because that is exactly the
  scenario the maintainer wants covered.
- **MPL-2.0 / Apache-2.0 / MIT.** Permissive; would let closed commercial
  services build on InterruptingCow without contributing back. Contradicts the
  stated goal.
- **A CLA (or DCO + CLA).** Enables a future relicense by a single entity. Not
  wanted here — the maintainer is fine being unable to relicense unilaterally,
  and a CLA adds friction and infrastructure disproportionate to a hobby project.
- **Dual licensing (AGPL + a paid commercial license).** A common model for
  network-service copyleft projects. Rejected: the maintainer explicitly doesn't
  care about a commercial offering, and dual licensing would require a CLA.
- **Git-flow (long-lived `develop`, release branches).** Heavier than needed;
  trunk-based with slices behind stubs fits a small team and keeps integration
  continuous.
