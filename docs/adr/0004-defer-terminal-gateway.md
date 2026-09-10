# ADR 0004: Defer the inbound terminal gateway; no plaintext telnet in

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

The original concept for InterruptingCow included letting users connect to the
daemon with a **classic telnet client**, the way they connect to a MU\* server —
a familiar, lightweight way to attach from a terminal.

On reflection this is in tension with the project's security posture:

- **Plaintext telnet carries credentials and content in the clear.** The daemon
  holds months of users' private conversations; putting a login and that stream
  on an unencrypted socket is a serious liability, especially if the port is ever
  exposed beyond localhost.
- **Telnet cannot do modern authentication.** No Argon2id challenge, no token
  exchange worth the name — at best a plaintext password prompt.
- The realistic secure alternatives are **a TLS-wrapped line terminal** (users
  need a client that does TLS, or an `stunnel`/SSH tunnel) or **an embedded SSH
  server** with public-key auth. Both are real implementation and maintenance
  work — an SSH server is also an additional listening attack surface to keep
  patched.
- The **web client is the primary interface** and covers the actual product
  need. Terminal access is a convenience, not a requirement.

## Decision

**We will not ship any inbound terminal access in the MVP, and we will not ship
plaintext telnet into the daemon at all.**

- The daemon defines an **`ITerminalGateway`** interface and ships
  **`NullTerminalGateway`**, which binds no listener.
- The interface is documented with its intended future implementations
  (`TlsTerminalGateway`, an SSH-based gateway) so a secure terminal front-end can
  be added later against a stable seam, without touching the session core.
- If a plaintext line listener is ever added for local tunneling convenience, it
  must be **off by default and bindable only to loopback**, with the risk
  documented loudly.

## Consequences

**Easier:**

- Smaller MVP surface; no listener to secure, rate-limit, or fuzz in the first
  release.
- The security story is simple to state: the only way in is HTTPS to the web
  client.
- The seam means adding secure terminal access later is additive, not a
  refactor.

**Harder / the cost:**

- Users who would prefer a terminal client can't have one yet. Mitigated: the
  web client is installable and works on desktop.
- When we do build it, we take on either a TLS line-protocol design or an
  embedded SSH server (with its own patching responsibility). That work is
  acknowledged and deferred, not avoided.

**Committed to:**

- `ITerminalGateway` as the extension point.
- No modern-auth-incompatible plaintext protocol as a supported inbound path.

## Alternatives considered

- **Ship plaintext telnet now, document the risk.** Rejected: too easy to
  misconfigure into an internet-exposed cleartext credential-and-content leak,
  for a product whose whole value is holding private data.
- **Ship a TLS line terminal in the MVP.** Rejected for scope: it needs a small
  protocol design (auth handshake over the TLS stream, framing) and a client
  story. Worth doing later; not worth delaying the MVP.
- **Ship an embedded SSH server in the MVP.** Rejected for scope and surface:
  best security and familiarity for terminal users, but a significant dependency
  and a listening service to keep current. A strong candidate for a later ADR.
- **Never support terminal access.** Rejected: it's a legitimate want in this
  community and the seam is cheap to leave in place.
