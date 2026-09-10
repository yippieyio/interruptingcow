# Contributing to InterruptingCow

Thank you for your interest. This document explains how the project takes
contributions.

> **The project is in its design phase — there is no code yet.** Until the first
> implementation lands, the contributions that help most are:
>
> - **Design review** — read [`docs/architecture.md`](docs/architecture.md) and
>   [`docs/protocol.md`](docs/protocol.md) and tell us what's wrong or missing.
> - **Discussion** — open a [GitHub Discussion][discussions] for questions and
>   ideas.
> - **ADR proposals** — to propose an architectural change, follow the
>   [ADR process](docs/adr/0000-adr-process.md).
> - **Documentation fixes** — broken links, contradictions, unclear passages.
>
> The workflow below describes how code contributions **will** work. The
> automated gates (CI, coverage, linting) are described as intended behavior;
> they are not yet wired up.

---

## Table of contents

- [Code of Conduct](#code-of-conduct)
- [Ways to contribute](#ways-to-contribute)
- [Development workflow](#development-workflow)
- [Branching](#branching)
- [Commit messages](#commit-messages)
- [Developer Certificate of Origin (DCO)](#developer-certificate-of-origin-dco)
- [Pull requests](#pull-requests)
- [Testing expectations](#testing-expectations)
- [Proposing architectural changes](#proposing-architectural-changes)
- [Reporting security issues](#reporting-security-issues)
- [License of contributions](#license-of-contributions)

---

## Code of Conduct

Everyone participating in this project — issues, pull requests, discussions,
chat — is expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Ways to contribute

| Contribution | How |
|---|---|
| Report a bug | Open an issue using the **Bug report** template |
| Request a feature | Open an issue using the **Feature request** template, or start a Discussion first if it's open-ended |
| Fix documentation | Open a PR directly |
| Propose an architecture change | [ADR proposal](docs/adr/0000-adr-process.md) |
| Ask a question | [GitHub Discussions][discussions] (not the issue tracker) — see [`SUPPORT.md`](SUPPORT.md) |
| Report a vulnerability | **Privately** — see [`SECURITY.md`](SECURITY.md), not a public issue |

## Development workflow

Once the codebase exists, the loop will be:

1. **Fork** the repository to your own account.
2. **Clone** your fork and add the principal repo as `upstream`.
3. Create a **topic branch** off `main` (see [Branching](#branching)).
4. Make your change, with tests.
5. Run the local checks (the intended command is `pnpm verify` — lint, typecheck,
   unit + integration tests).
6. **Commit** with a [Conventional Commit](#commit-messages) message and a
   [DCO sign-off](#developer-certificate-of-origin-dco) (`git commit -s`).
7. **Push** to your fork and open a **pull request** against `upstream/main`.
8. Address review feedback. A maintainer squash-merges once checks pass and the
   PR has an approving review.

The detailed environment setup will live in
[`docs/development/getting-started.md`](docs/development/getting-started.md).

### Keeping your branch current

Rebase onto `upstream/main` rather than merging it in:

```sh
git fetch upstream
git rebase upstream/main
```

`main` uses a linear history; PRs are squash-merged.

## Branching

- **`main`** is always releasable and protected. No direct pushes.
- Work happens on **short-lived topic branches**, one logical change each:
  - `feat/<slug>` — a new capability
  - `fix/<slug>` — a bug fix
  - `docs/<slug>` — documentation only
  - `refactor/<slug>` — behavior-preserving restructuring
  - `test/<slug>` — tests only
  - `chore/<slug>` / `ci/<slug>` / `build/<slug>` — tooling, deps, pipeline
- No long-running feature branches. Large features land as vertical slices behind
  stubs or flags — the architecture is built for this.

## Commit messages

We use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>(<optional scope>): <subject>

<optional body>

<optional footer(s)>
```

- **type**: `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`,
  `chore`.
- **scope** (optional): the affected area — `protocol`, `daemon`, `web`,
  `persistence`, `auth`, `session`, `deploy`, `docs`.
- **subject**: imperative mood, lower-case, no trailing period.
- Breaking changes: add `!` after the type/scope (`feat(protocol)!: …`) **and** a
  `BREAKING CHANGE:` footer explaining the migration.

Examples:

```
feat(session): re-establish healthy connections after daemon restart
fix(protocol): reject line messages with a negative timestamp
docs(adr): add 0008 on per-user log encryption
```

Because PRs are squash-merged, **the PR title must itself be a valid Conventional
Commit** — it becomes the commit subject on `main`. The release changelog is
generated from these messages.

## Developer Certificate of Origin (DCO)

This project uses the **Developer Certificate of Origin** instead of a Contributor
License Agreement. There is no form to sign. Instead, you certify the origin of
each contribution by adding a `Signed-off-by` line to every commit:

```
Signed-off-by: Jane Roe <jane.roe@example.com>
```

`git commit -s` (or `-S -s` if you also GPG-sign) adds this automatically, using
your configured `user.name` and `user.email`. The name and email must be real and
match the commit author; pseudonyms and anonymous contributions cannot be
accepted under the DCO.

By signing off, you agree to the following (DCO 1.1, verbatim):

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.
1 Letterman Drive
Suite D4700
San Francisco, CA, 94129

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

Once CI is in place, a check will enforce that every commit in a PR carries a
valid `Signed-off-by` line matching its author. If you forget, you can fix a
branch with:

```sh
git rebase --signoff upstream/main
git push --force-with-lease
```

## Pull requests

A good PR:

- Does **one thing**. Split unrelated changes.
- Has a description that says **what** and **why**, and links the issue it closes
  (`Closes #123`).
- Includes tests for new behavior and a regression test for any bug fix.
- Updates documentation in the same PR when it changes behavior, config, or the
  protocol.
- Notes any schema migration and its rollback.
- Links the relevant ADR if it implements or changes an architectural decision.
- Includes a screenshot or short clip for user-visible web changes.

Review assignment will follow `.github/CODEOWNERS` (planned): changes to the
protocol, persistence, or authentication require review from the owner of that
area.

### PRs from forks

CI on fork PRs runs with **least privilege** — no repository secrets, no image
push, no deploy. It runs linting, type-checking, and the unit / integration /
end-to-end test suites, all of which are designed to need no secrets. First-time
contributors' workflow runs require a maintainer to approve them (GitHub's
default); this is kept on deliberately.

A maintainer may push small fixups to your PR branch if you enabled
"Allow edits from maintainers".

## Testing expectations

The full strategy is in
[`docs/development/testing.md`](docs/development/testing.md). In short:

- **Every package** carries practical unit tests beside its source.
- **Integration tests** run against a real, throwaway PostgreSQL.
- **A fake MU\* server fixture** stands in for real game servers — CI never
  connects to a live MUSH.
- **Every bug fix** adds a test that fails before the fix and passes after.
- **Every migration** has a test that applies it and rolls it back.
- Coverage is reported on each PR and must not regress on the diff.

## Proposing architectural changes

Anything that changes the shape of the system — a new component, a different
storage engine, a protocol change, a new external dependency of significance —
goes through an **Architecture Decision Record**. See
[`docs/adr/0000-adr-process.md`](docs/adr/0000-adr-process.md) for the process and
[`docs/adr/template.md`](docs/adr/template.md) for the template. Open the ADR as
its own PR so the decision can be discussed before code depends on it.

## Reporting security issues

**Do not open a public issue for a security vulnerability.** InterruptingCow is
designed to hold months of users' private conversations; disclosure is taken
seriously. See [`SECURITY.md`](SECURITY.md) for the private reporting channel.

## License of contributions

InterruptingCow is licensed under
[`AGPL-3.0-or-later`](LICENSE). By contributing, you agree that your
contributions are licensed under the same terms — **inbound = outbound**. Your
DCO sign-off is your assertion that you have the right to make the contribution
under that license.

Add the SPDX header to every new source file:

```ts
// SPDX-License-Identifier: AGPL-3.0-or-later
```

[discussions]: https://github.com/OWNER/InterruptingCow/discussions
