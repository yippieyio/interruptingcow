# Roadmap

This is **direction, not a set of promises**. It's maintained by a volunteer
project with no deadlines. Priorities shift; features get cut or deferred.
Milestones on GitHub track the current state; this file is the narrative.

The detailed, living design lives in [`docs/`](docs/index.md). This page is the
sequence.

## Guiding principle

> Clean and elegant architecture first, then a minimum viable product, then the
> nice-to-haves.

Plan for everything; build the MVP slice. Anticipated features get their seams,
interfaces, and stub classes in place early so adding them later is not a
rewrite.

---

## M0 — Skeleton & architecture

**Goal:** the project is a real, navigable codebase with every architectural seam
in place, even though it does almost nothing yet.

- Monorepo scaffold (pnpm workspace): `protocol`, `daemon`, `web` packages.
- The shared `protocol` package: every wire message as a zod schema with inferred
  types; a codec abstraction.
- Daemon skeleton: composition root, config loading and validation, health
  endpoint, structured logging, graceful shutdown.
- **All the interfaces and stub classes**, each documented with what it will
  become: the inbound terminal gateway, the upstream transport, the trigger
  engine, the notifier, every repository.
- The ADRs written up (`docs/adr/`).
- CI: lint, type-check, unit and integration tests, on every push and PR.
- Deployment scaffold: a Podman pod definition that stands up even though the
  daemon is nearly empty.
- The community-health and documentation set (this repository's first commit
  already covers most of it).

## M1 — MVP: persistent connections that outlive your client

**Goal:** the core promise, end to end. Connect to a world from the web client,
close the browser, come back, and you're still connected with your scrollback
intact.

- **PostgreSQL persistence:** users, tokens, worlds, session state, and
  full-text-searchable line logs. Migrations with tested rollbacks. Batched,
  back-pressure-aware log writes.
- **Authentication:** admin-provisioned accounts (a CLI, no self-serve signup),
  Argon2id password hashing, short-lived access tokens plus a long-lived refresh
  token. The web client stores only a token.
- **The `icow` admin CLI:** create users and worlds, mint tokens, run migrations.
- **Upstream connections:** plaintext and TLS, with real telnet option
  negotiation (NAWS, TTYPE, EOR/GA, SUPPRESS-GO-AHEAD, CHARSET). Compression and
  the richer MU\* protocols are registered but stubbed.
- **The Session model:** one persistent connection per (user, world). The daemon
  never drops it voluntarily. A server-initiated disconnect is surfaced and
  *not* automatically retried. Our-fault recovery (restart, power loss, network
  loss) re-establishes connections that were healthy before the incident, within
  a grace window, bounded. See
  [ADR 0006](docs/adr/0006-connection-recovery-semantics.md).
- **The line pipeline:** decode → normalize → log → (trigger hook, no-op for now)
  → fan out to attached clients.
- **WebSocket + REST API:** attach/detach, live line stream, scrollback backfill
  on attach, full-text history search.
- **The web client (PWA):** installable on iOS 16.4+, Android, and desktop. A
  terminal pane with ANSI colour, an input line with history and tab-complete, a
  world list with live status, and a history search view. A settings area with a
  visible **Source** link (AGPL §13) and a "triggers — coming soon" placeholder.
- **A fake MU\* server fixture** for integration and end-to-end tests. CI never
  touches a live game.
- **End-to-end tests** for the headline journeys, plus a durability test that
  hard-kills the pod and verifies the database recovers with no data loss.

**M1 is done when:** a small group of friends can use it daily as their MU\*
client, with confidence that a crash or a redeploy won't lose their connections
or their logs. Tagged `v0.1.0`.

## M2 — The cake gets iced

Roughly in priority order; each is independently shippable and the M0/M1 seams
already exist for all of them.

- **Triggers, aliases, gags, highlights.** Per-world rule sets, stored
  server-side, edited in the web client. This is the prerequisite for
  notifications — without it, push would fire on every line.
- **Push notifications.** Web Push (VAPID), driven by trigger rules, so your
  phone buzzes for a page and stays silent for room spam. iOS requires the PWA to
  be installed to the home screen; not available in the EU per Apple.
- **Zero-knowledge log encryption.** Per-user keys derived from a passphrase;
  encrypted worlds' logs are ciphertext at rest and searched client-side; a
  server administrator cannot read them.
- **An inbound terminal gateway.** A way to attach from a real terminal — over
  TLS with token auth, or an embedded SSH server with registered public keys.
  Deliberately deferred; plaintext telnet in is a non-starter
  ([ADR 0004](docs/adr/0004-defer-terminal-gateway.md)).
- **Richer MU\* protocols.** MCCP2 compression, GMCP, MXP, MSSP — for structured
  data and nicer rendering.
- **Optional TOTP two-factor authentication.**

## Beyond M2

Unscheduled, plausible, not committed:

- Self-serve registration and multi-tenant operation (a much larger scope:
  signup, email verification, quotas, abuse handling).
- Shared/observer connections (multiple people watching one session).
- A binary protocol codec, if line volume ever justifies it.
- Native mobile wrappers, if the PWA proves insufficient.
- Signed releases with SBOM and provenance.

## How to influence the roadmap

Open a [Discussion][discussions]. Roadmap and scope decisions are made by the
maintainers, informed by Discussions and issues — see
[`GOVERNANCE.md`](GOVERNANCE.md).

[discussions]: https://github.com/OWNER/InterruptingCow/discussions
