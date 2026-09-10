# ADR 0008: Offer per-user origin information to destinations

- **Status:** Proposed
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer
- **Supersedes:** —
- **Superseded by:** —

## Context

InterruptingCow is a shared daemon. Every upstream connection it makes to a
destination — a MUSH, MUCK, MOO, or MUD — originates from the **single IP address
of the host the daemon runs on**, regardless of which user the connection belongs
to or where that user physically is.

MU\* destinations almost always log the connecting IP address, and many use it as
a policy input:

- **Alt limits.** A destination that caps a person at, say, ten "alt" characters
  commonly enforces that by counting characters per source IP. Two InterruptingCow
  users on the same instance are counted as one person: their alt allocations are
  drawn from the same pool, and one user's characters can exhaust the limit for
  the other.
- **Identity conflation.** Staff who see several characters from one IP may treat
  them as belonging to one person. Private information one user shares with staff
  can be associated with, or disclosed to, the other.
- **Shared fate on bans.** A site ban is frequently an IP ban. If one
  InterruptingCow user is banned from a destination, every other user of that
  instance is very likely banned too, and vice versa.

These are not defects in the destinations. Limiting by IP address is often the
only practical tool a volunteer-run game has to defend against ban evasion,
spam, and abuse.

Traditional MU\* clients run on the user's own machine, so the destination sees
the user's own address and these mechanisms work as intended. A shared proxy
breaks that assumption. The web-proxy world solved the analogous problem with
forwarded-identity headers — `X-Forwarded-For`, `Forwarded` (RFC 7239),
`X-Forwarded-Host`, `X-Forwarded-Proto`, `Proxy-Authorization` — that let an
origin server recover the real client when it trusts the proxy in front of it. A
MU\* connection is a telnet-ish TCP stream, not HTTP, so there is no drop-in
equivalent, but the need is the same: give a destination a way to tell one
InterruptingCow user from another if it wants to.

Two hard constraints frame the decision:

- **A destination is never obligated to honour anything we send.** Whatever
  channel we use, the information is advisory. Destinations may ignore it,
  distrust it (it is trivially spoofable by a hostile proxy), or have no code to
  read it. We cannot and should not try to make it authoritative.
- **A carrier already exists, but adoption does not.** The MU\* protocol family
  has a purpose-built mechanism for precisely this: **MNES** (Mud New Environment
  Standard) layers on the `NEW-ENVIRON` telnet option (RFC 1572, option 39) and
  defines an **`IPADDRESS`** variable whose documented rationale is *"suggested
  for proxies adding MTTS support to also report the client's real IP address …
  so a MUD can ban specific users without having to ban an entire proxy"*. It
  also carries `CLIENT_NAME` / `CLIENT_VERSION`. `NEW-ENVIRON` additionally has a
  `USERVAR` type for non-standard variables, which could carry a pseudonymous
  token. **GMCP** (option 201, JSON packages) is the other plausible carrier —
  clients already send `Core.Hello { "client", "version" }`, and a namespaced
  package (e.g. `Client.Forwarded`) would fit its model. The open question is not
  "is there a channel" but "will destination software read it" — MNES support in
  MU\* servers is uneven, and any `USERVAR`/GMCP convention would be new. That
  needs its own investigation and probably coordination with destination authors,
  so the exact wire choice is deferred to a follow-up ADR. See
  [Telnet & MU\* protocols](../telnet-protocols.md) for the full survey of
  candidate carriers.

## Decision

**InterruptingCow will provide a mechanism to convey per-user origin information
to destinations, as a courtesy that a destination may use to distinguish
individual users of one instance. The mechanism is advisory, off unless
configured, and its wire transport is deferred to a later ADR.**

Concretely:

1. **A `ForwardedIdentity` seam in the upstream gateway.** The daemon defines an
   interface that, given a Session (its user and world), produces the
   origin-information payload to present to the destination, and a point in the
   connection lifecycle at which the upstream gateway offers it. The MVP ships a
   **no-op implementation** that sends nothing.

2. **The payload models the useful subset of the forwarded-header family**,
   named for its HTTP analogue so the intent is legible:
   - a stable **per-user pseudonymous identifier** scoped to this instance (the
     `X-Forwarded-For` analogue — it need not be, and by default is not, the
     user's real IP address),
   - optionally the user's **real client IP** and/or a coarse network hint, only
     when the operator has opted in and the user has consented,
   - the **instance identity** (the `X-Forwarded-Host` / `Forwarded by=`
     analogue) so a destination knows which proxy is speaking,
   - room for a **shared-secret credential** (the `Proxy-Authorization` analogue)
     for the case where a destination and an operator have arranged mutual trust.

3. **Privacy-preserving by default.** The default payload is the pseudonymous
   per-user identifier and the instance identity — enough for a destination to
   count alts and scope bans per user without learning where the user is. Real IP
   forwarding is a per-instance, per-user opt-in, never automatic.

4. **The transport is a follow-up ADR.** This record commits to the capability
   and the seam, not to a specific telnet option, subnegotiation, or MU\*
   protocol package. The follow-up ADR's starting point is **MNES `IPADDRESS`
   plus a `NEW-ENVIRON` `USERVAR` for the pseudonymous token** (the mechanism
   that already exists for this purpose), with **a GMCP package** as the
   alternative; it adopts whichever destination software is realistically willing
   to read, and specifies a minimal convention only if neither is.

5. **Documented as advisory on both sides.** User-facing and operator-facing docs
   must state plainly that destinations are under no obligation to read or act on
   this information, and that until a destination does, the shared-IP
   consequences (shared alt pools, identity conflation, shared-fate bans) apply
   in full.

## Consequences

**Easier:**

- There is a defined place to add forwarded-identity support without touching the
  Session core or the line pipeline — the same "plan for everything, build the
  MVP" pattern as the terminal gateway and the trigger engine.
- Operators who coordinate with a friendly destination have a sanctioned path to
  give that destination per-user visibility, instead of an ad-hoc hack.
- The privacy default is conservative: nothing is disclosed unless someone turns
  it on.

**Harder / the cost:**

- **It does not fix the shared-IP problem by itself.** Until destinations
  implement a matching reader, users of a shared instance still share alt limits
  and ban fate. The honest framing of this in the docs is a permanent
  responsibility, not a temporary caveat.
- The information is **spoofable** and must never be treated by anyone as
  authoritative identity. If a destination comes to rely on it for policy, a
  hostile or buggy proxy can lie. The follow-up ADR's transport choice has to
  make the trust model explicit (which is what the `Proxy-Authorization` analogue
  is for).
- Forwarding a real client IP, even opt-in, is a genuine privacy disclosure and
  needs a clear consent flow and operator documentation, or it will be
  misconfigured.
- We take on a small amount of design debt: the seam exists before the transport
  does, so an early reader of the code sees an interface with only a no-op
  implementation.

**Committed to:**

- A `ForwardedIdentity` interface as the extension point, with a no-op default.
- A pseudonymous, privacy-preserving default payload; real-IP forwarding as
  opt-in only.
- Never presenting this information as authoritative, and never making a
  destination's acceptance of it a requirement for InterruptingCow to function.
- A follow-up ADR for the wire transport before any non-no-op implementation
  ships.

## Alternatives considered

- **Do nothing; document the shared-IP consequences and stop there.** The
  shared-IP consequences still need documenting regardless (that part is not
  optional). But leaving no seam means that when a destination *does* want
  per-user data, adding the capability is a retrofit through the connection core.
  Rejected: the seam is cheap and matches how the rest of the system defers
  features.
- **Give each user their own outbound IP address** (an address pool, per-user
  SOCKS/VPN egress). This actually solves alt limits and ban scoping at the
  network layer. Rejected for the MVP and likely beyond: it is a large
  operational burden (address allocation, routing, cost), pushes complexity onto
  every operator, and is impractical for the small self-hosted instances that are
  the target. Could be revisited as an operator-level deployment option, not a
  daemon feature.
- **Pick a transport now** (e.g. mandate MNES `IPADDRESS` + a `NEW-ENVIRON`
  `USERVAR`, or a GMCP `Client.Forwarded` package). Rejected: the mechanism
  exists (MNES documents `IPADDRESS` for exactly the proxy case) but real MU\*
  server support for reading it is uneven, and a pseudonymous-token convention
  would be new either way. Choosing well needs a survey of what destination
  software actually ingests today and probably a conversation with server
  authors. Doing it hastily risks a format nobody implements. Deferred to its own
  ADR — with MNES as the leading candidate.
- **Forward the user's real IP by default**, mirroring `X-Forwarded-For`
  literally. Rejected: a proxy whose purpose includes letting people connect
  without exposing their home address should not disclose that address by
  default. The pseudonymous identifier gives destinations what they need for
  policy (a stable per-user token) without the disclosure.
- **Make honouring the information a condition of connecting** (refuse
  destinations that don't acknowledge it). Rejected outright: InterruptingCow has
  no standing to impose requirements on destinations, and its value to users is
  connecting to the games that exist, as they are.
