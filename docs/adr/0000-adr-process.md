# ADR 0000: The ADR process

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow intends to be a long-lived, community-maintained project. Design
decisions made now — the language, the datastore, the connection semantics — will
be load-bearing for years, and contributors who arrive later need to understand
*why* things are the way they are, not just *what* they are. Undocumented
decisions get re-litigated, or worse, silently eroded.

We want a lightweight, in-repository record of significant decisions that travels
with the code and is reviewed like the code.

## Decision

We keep **Architecture Decision Records** in `docs/adr/`, following the
widely-used lightweight format popularised by Michael Nygard.

### What warrants an ADR

Anything that changes the *shape* of the system or would be expensive to reverse:

- Choice of language, runtime, framework, or datastore.
- The public wire protocol, or a breaking change to it.
- A new component, or removal/merging of one.
- A new external dependency of structural significance.
- Cross-cutting policies: security model, licensing, branching, release strategy.
- Connection, consistency, or durability semantics.

Routine choices (a utility library, a file layout within a module, a bug-fix
approach) do **not** need an ADR.

### Format

Copy [`template.md`](template.md) to `NNNN-short-kebab-title.md`, where `NNNN` is
the next unused four-digit number. Sections:

- **Status** — `Proposed`, `Accepted`, `Rejected`, `Deprecated`, or
  `Superseded by ADR-XXXX`.
- **Date**, **Deciders**.
- **Context** — the forces at play: requirements, constraints, what problem this
  solves. Written so a newcomer understands the situation without prior
  knowledge.
- **Decision** — what we're doing, in the active voice ("We will…").
- **Consequences** — what becomes easier, what becomes harder, what we're now
  committed to, what we'll need to watch.
- **Alternatives considered** — the other real options and why they lost. This is
  the section that saves the most future time.

### Lifecycle

1. Open the ADR as its **own pull request**, status `Proposed`.
2. Discuss on the PR. A "reasonable review window" is about one week; anyone may
   ask for more time.
3. Merge when it reaches lazy consensus of the maintainers (see
   [`GOVERNANCE.md`](../../GOVERNANCE.md)). Flip the status to `Accepted` (or
   `Rejected`, kept for the record) in the same PR before merge.
4. A later decision that overturns an earlier one is a **new** ADR that sets the
   old one's status to `Superseded by ADR-XXXX`. **Do not rewrite history** in an
   accepted ADR beyond status and cross-links; if the reasoning changed, that's a
   new record.
5. Code that implements a decision should reference its ADR in a comment or its
   PR description.

### Numbering

- `0000` — this document.
- `0001`+ — decisions, in the order they were made.
- Numbers are never reused, even for rejected or superseded ADRs.

## Consequences

- **Easier:** onboarding; resisting decision churn; understanding trade-offs that
  were already weighed.
- **Harder:** significant changes now carry a small writing cost and a review
  cycle before code can depend on them. This is intentional friction.
- We commit to keeping the [ADR index](../index.md#architecture-decision-records)
  current.

## Alternatives considered

- **A wiki or an external design-doc tool.** Rejected: drifts from the code, not
  reviewed with it, often not read.
- **Commit messages and PR descriptions only.** Rejected: not discoverable; the
  "why" is scattered and lost to history.
- **A heavier RFC process** (separate repo, formal stages, shepherds). Rejected
  as disproportionate for a project this size. Can be revisited if the
  contributor base grows enough to need it — via an ADR, naturally.
