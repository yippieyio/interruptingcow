# InterruptingCow

**A daemon that keeps your MUSH / MUCK / MOO connections alive while you're away —
so you can close your laptop, pick up your phone, and still be connected.**

> _Knock knock._ — _Who's there?_ — _Interrupting cow._ — _Interrupting c—_ **MOO.**
>
> The daemon interrupts your absence. It holds the line so the game never notices
> you left. (The name also nods to [LambdaMOO][lambdamoo] — this project's
> spiritual home — by way of the author's earlier client, "How Now Moo Cow".)

---

## Status: design phase

**There is no running software yet.** This repository currently contains only
documentation, governance, and architecture decisions — the blueprint for what
will be built. See [`ROADMAP.md`](ROADMAP.md) for the plan and
[`docs/`](docs/index.md) for the full design.

If you're here to help shape it: read [`docs/architecture.md`](docs/architecture.md),
then open a [Discussion][discussions] or an
[ADR proposal](docs/adr/0000-adr-process.md).

---

## The problem

Playing on a text-based world (MUSH, MUCK, MOO, MUD) means holding a live TCP
connection to the game server. The moment your client closes — laptop lid, phone
sleep, train tunnel, flaky Wi-Fi — that connection drops. You miss pages and
tells sent while you were gone, you lose the room's conversation, and you come
back to a cold reconnect and a login screen.

Desktop clients like MUSHclient, Potato, and TinyFugue are excellent, but they
run on one machine and die with it.

## What InterruptingCow does

A small, long-running **daemon** connects to your worlds on your behalf and
**never voluntarily drops those connections**. You attach to it from an
**installable web client** (a PWA that works on iOS, Android, and desktop), read
and type, then detach. The daemon stays connected. Every line to and from every
world is **logged and full-text searchable**, so nothing said while you were away
is lost.

Later (see the [roadmap](ROADMAP.md)):

- **Triggers, aliases, gags, highlights** — the power-user toolkit.
- **Push notifications**, gated on your triggers, so your phone buzzes for a page
  but stays quiet for room spam.
- **Zero-knowledge encryption** of your logs, so whoever runs the server can't
  read them.

### What "forever" does _not_ mean

The daemon never closes a connection on its own — no idle timeout, no dropping
you when the last browser tab detaches. But it does **not** chase a server that
drops _you_. If the game closes the connection, InterruptingCow shows it as
`disconnected` and waits for you to reconnect. The only time it reconnects
automatically is recovering from _its own_ outage — a daemon restart, a host
power loss, a network blip — where connections that were healthy before the
incident are re-established. This distinction is deliberate; see
[ADR&nbsp;0006](docs/adr/0006-connection-recovery-semantics.md).

## Architecture at a glance

```
   telnet / TLS                WebSocket + REST (HTTPS)
game servers  <────────>  InterruptingCow daemon  <────────>  Web client (PWA)
 (MUSH/MOO/…)              ├─ Session manager: one persistent      iOS · Android
                          │  connection per (user, world)         · desktop
                          ├─ Line pipeline: decode → log →
                          │  trigger-eval → fan out to clients
                          ├─ PostgreSQL: users, worlds, session
                          │  state, full-text-searchable logs
                          └─ Reverse proxy (Caddy/nginx) in front
```

- **One language:** TypeScript everywhere — daemon (Node.js LTS), web client
  (Angular), and a shared `protocol` package that is the single source of truth
  for the wire format (zod-validated on both ends).
- **One database:** PostgreSQL. Chosen over SQLite because logs grow fast and
  because future per-user log encryption needs it. State lives on host volumes,
  outside the container, and must survive a hard power loss.
- **Deployment:** a Podman pod (proxy + daemon + Postgres).

Full detail: [`docs/architecture.md`](docs/architecture.md). Wire protocol:
[`docs/protocol.md`](docs/protocol.md). The reasoning behind each big choice:
[`docs/adr/`](docs/adr/0000-adr-process.md).

## Documentation

| If you want to… | Read |
|---|---|
| Understand the system | [`docs/architecture.md`](docs/architecture.md) |
| Understand the client/daemon protocol | [`docs/protocol.md`](docs/protocol.md) |
| Know why a decision was made | [`docs/adr/`](docs/adr/0000-adr-process.md) |
| Contribute code (soon) | [`CONTRIBUTING.md`](CONTRIBUTING.md) · [`docs/development/`](docs/development/getting-started.md) |
| Run your own instance (soon) | [`docs/operations/runbook.md`](docs/operations/runbook.md) |
| Use the web client (soon) | [`docs/user-guide/`](docs/user-guide/install.md) |
| Look up a term | [`docs/glossary.md`](docs/glossary.md) |
| See what's planned | [`ROADMAP.md`](ROADMAP.md) |

## Contributing

Contributions will be welcome once there's code to contribute to. For now, the
useful contributions are design review and discussion:

- Open a [GitHub Discussion][discussions] for questions and ideas.
- Propose an architecture change via the
  [ADR process](docs/adr/0000-adr-process.md).
- File issues for gaps or contradictions in the documentation.

When code lands, [`CONTRIBUTING.md`](CONTRIBUTING.md) has the full workflow:
fork → topic branch → PR, [Conventional Commits][conv-commits], and a
[Developer Certificate of Origin](CONTRIBUTING.md#developer-certificate-of-origin-dco)
sign-off (`git commit -s`).

Everyone participating is expected to follow the
[Code of Conduct](CODE_OF_CONDUCT.md).

## License

InterruptingCow is licensed under the
**[GNU Affero General Public License v3.0 or later](LICENSE)** (`AGPL-3.0-or-later`).

The AGPL is deliberate: it guarantees that anyone who runs a modified version of
this software as a network service — not just anyone who distributes it — must
make their source available to that service's users. If you host InterruptingCow,
you must offer your users the (possibly modified) source. See
[ADR&nbsp;0007](docs/adr/0007-licensing-and-workflow.md).

---

_InterruptingCow is not affiliated with any MU\* server, client, or the
LambdaMOO project. "MUSH", "MUCK", "MOO", and "MUD" are generic terms for
categories of text-based multiplayer worlds._

[lambdamoo]: https://en.wikipedia.org/wiki/LambdaMOO
[discussions]: https://github.com/OWNER/InterruptingCow/discussions
[conv-commits]: https://www.conventionalcommits.org/en/v1.0.0/
