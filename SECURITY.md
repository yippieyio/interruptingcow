# Security Policy

InterruptingCow is designed to hold **months of users' private conversations** —
role-play, tells, pages, whatever people say in the worlds they connect to. A
compromise of an InterruptingCow instance is a compromise of everything its users
have said through it. We take security reports seriously and ask that you report
them responsibly.

> **Project status:** design phase. There is no released software and no
> deployed instance to attack yet. This policy describes how reports will be
> handled once there is code. If you find a security-relevant flaw in the
> **design** as documented, please still tell us privately using the channel
> below rather than opening a public issue.

## Supported versions

| Version | Supported |
|---|---|
| _none released yet_ | — |

Once releases begin, this table will list which versions receive security fixes.
The intent is to support the latest minor release, and the previous minor release
for a transition period.

## Reporting a vulnerability

**Do not open a public GitHub issue, Discussion, or pull request for a security
vulnerability.**

Report privately through one of:

1. **GitHub private vulnerability reporting** — on the repository's **Security**
   tab, "Report a vulnerability". This is the preferred channel once the repo is
   public. _(To be enabled by the maintainers.)_
2. **Email** — **TODO: security contact address** (e.g. `security@<project-domain>`
   or a named maintainer). Encrypt with the maintainer's PGP key if the report is
   sensitive: **TODO: PGP key fingerprint / link**.

Please include:

- A description of the issue and its impact.
- Steps to reproduce, or a proof of concept.
- Affected version(s) or commit, and configuration relevant to the issue.
- Whether the issue is already known publicly or being exploited.

## What to expect

This is a volunteer project; the timelines below are targets, not contractual
guarantees.

| Stage | Target |
|---|---|
| Acknowledgement of your report | within 3 working days |
| Initial assessment (severity, affected versions) | within 10 working days |
| Fix developed and reviewed privately | depends on severity and complexity |
| Coordinated disclosure & release | by agreement with you; default 90 days from report, sooner if a fix is ready and users are at risk |

We will keep you informed of progress, credit you in the advisory and release
notes unless you ask us not to, and coordinate the public disclosure date with
you.

## Scope

In scope:

- The daemon, the web client, the shared protocol package, the deployment
  configuration in this repository, and the documented operational procedures.
- Authentication and session handling, token issuance and revocation.
- Handling of stored logs and world credentials, including the planned
  encryption features.
- The telnet/TLS handling of untrusted data from upstream game servers.

Out of scope:

- Vulnerabilities in third-party game servers you connect to.
- Vulnerabilities in dependencies that are already public and have an upstream
  fix — please just open a normal PR bumping the dependency, unless the way we
  use it makes the impact worse.
- Social-engineering, physical attacks, or attacks requiring a already-compromised
  host or a malicious administrator (the AGPL/self-host model assumes the operator
  is trusted; the planned zero-knowledge log encryption is the exception and
  reports about *that* threat model are very much in scope).
- Denial of service from simply pointing the daemon at a hostile server that
  floods it, **unless** it leads to crash, data loss, or cross-user impact.

## Hardening and supply chain

Planned, not yet in place (tracked on the [roadmap](ROADMAP.md)):

- Dependency scanning and automated updates (Dependabot).
- Static analysis (CodeQL) on every pull request.
- Secret scanning and push protection on the repository.
- Pinned, digest-referenced container base images.
- Signed releases with a software bill of materials (SBOM) and build provenance.

## Disclosure of this policy's gaps

The **TODO** markers above (contact address, PGP key) must be filled in before the
repository is made public and before it accepts outside contributions. An
unmonitored security address is a security problem in itself.
