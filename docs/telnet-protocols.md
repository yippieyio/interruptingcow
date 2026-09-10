# Telnet and the MU\* protocol family

> **Reference, not a decision.** This document surveys the upstream (game-facing)
> protocols InterruptingCow's telnet layer has to understand or may choose to
> speak. It exists so contributors don't have to re-derive it from scattered MU\*
> wikis. Decisions about *which* of these we implement, and when, live in the
> [architecture doc](architecture.md#upstream-gateway-transports-and-telnet), the
> [roadmap](../ROADMAP.md), and the ADRs — notably
> [ADR 0008](adr/0008-forwarded-user-identity.md), which relies on the MNES
> findings below.

For the *downstream* protocol (how the web client and daemon talk to each other)
see [protocol.md](protocol.md). For one-line definitions see the
[glossary](glossary.md#telnet-and-the-wire).

---

## Base telnet (RFC 854)

A MU\* connection is a single TCP byte stream that is *mostly* plaintext lines of
game text, with **option negotiations** interleaved. Every negotiation byte
sequence is introduced by **`IAC`** (Interpret As Command, byte `255`).

### Negotiation verbs

| Bytes | Name | Meaning |
|---|---|---|
| `IAC WILL <opt>` | WILL | "I will enable option `<opt>` on my side" (an offer) |
| `IAC WONT <opt>` | WONT | "I won't / I'm disabling `<opt>`" |
| `IAC DO <opt>` | DO | "Please enable `<opt>` on your side" (a request) |
| `IAC DONT <opt>` | DONT | "Don't / stop `<opt>`" |

The peer accepts or refuses. By convention each option's spec says which side
offers (`WILL`) and which side requests (`DO`).

### Subnegotiation

Once an option is on, arbitrary option-specific payload is framed as:

```
IAC SB <opt> <payload bytes...> IAC SE
```

`SB` = Subnegotiation Begin (`250`), `SE` = Subnegotiation End (`240`). **Every
structured MU\* protocol below is a payload format inside this envelope.** A
literal `255` byte inside a payload is escaped as `IAC IAC`.

### Defensive parsing

Data from a game server is untrusted. The IAC state machine must not be
crashable by malformed sequences: a truncated `IAC SB` with no `IAC SE`, an
unknown option number, an `IAC` at end of buffer (partial), over-long
subnegotiation payloads, or negotiation for options mid-stream. Unknown options
are refused (`DONT`/`WONT`), not ignored.

---

## Core options InterruptingCow negotiates

These are committed to in the
[architecture](architecture.md#upstream-gateway-transports-and-telnet).

| Option | # | RFC | What it does | Who offers | InterruptingCow's part |
|---|---|---|---|---|---|
| **SGA** — Suppress Go Ahead | 3 | 858 | Drops the half-duplex "go ahead" signal; effectively puts the link in full-duplex. Almost universally negotiated. | either | accept |
| **TTYPE** — Terminal Type | 24 | 1091 | Server asks the client what terminal it is; see **MTTS** below for the MU\* convention layered on it. | server `DO`, client `WILL` | answer with client name, a terminal type, and the MTTS bitmask |
| **EOR** — End Of Record | 25 | 885 | Lets the server send `IAC EOR` to mark the end of a **prompt** line (as opposed to a normal newline-terminated line), so clients can render or hold prompts specially. | server | detect prompts |
| **NAWS** — Negotiate About Window Size | 31 | 1073 | Client reports terminal width and height as four bytes (`IAC SB NAWS <w-hi> <w-lo> <h-hi> <h-lo> IAC SE`). Re-sent whenever the size changes. | client `WILL` | report the web client's reported dimensions; re-send on `resize` |
| **CHARSET** | 42 | 2066 | Negotiates the stream's character encoding. One side sends `REQUEST` with a list; the other `ACCEPTED`/`REJECTED`. | either | prefer `UTF-8` when offered; else fall back to the per-world configured encoding |

`GA` (Go Ahead) is byte `249`; it is what SGA suppresses. There is no option
number for GA itself.

---

## MU\*-specific protocols

All of these ride inside `IAC SB <opt> ... IAC SE`. Support across game servers
is **uneven** — MUSH/MUCK/MOO servers typically implement only a handful, and
which handful varies. InterruptingCow negotiates what it needs and refuses the
rest without breaking.

### Terminal & environment identification

#### MTTS — Mud Terminal Type Standard

- **Carrier:** the TTYPE option (24). No separate option number.
- **Flow:** the server sends `IAC SB TTYPE SEND IAC SE` repeatedly; the client
  answers `IAC SB TTYPE IS "<string>" IAC SE` each time, cycling through:
  1. **client name**, upper-case (e.g. `"MUSHCLIENT"`, `"INTERRUPTINGCOW"`)
  2. **terminal type** — one of `"DUMB"`, `"ANSI"`, `"VT100"`, `"XTERM"`
  3. **`"MTTS <n>"`** where `<n>` is the sum of the capability bits below
  4. repeating the MTTS value again signals "no more".
- **Capability bitmask:**

  | Bit | Meaning | Bit | Meaning |
  |---|---|---|---|
  | 1 | ANSI colour | 128 | **Proxy** — the client is not the user's own machine |
  | 2 | VT100 | 256 | Truecolour (24-bit) |
  | 4 | UTF-8 | 512 | MNES support |
  | 8 | 256 colour | 1024 | MSLP support |
  | 16 | Mouse tracking | 2048 | SSL/TLS on this connection |
  | 32 | OSC colour palette | | |
  | 64 | Screen reader | | |

- **Relevance to InterruptingCow:** bit **128 (proxy)** is the honest signal that
  we are a shared intermediary, not an end-user client. Bit **512** advertises
  that we also speak MNES (below).

#### NEW-ENVIRON — RFC 1572

- **Option:** 39.
- **Purpose:** exchange named environment variables between the two ends.
- **Command codes:** `IS` = 0, `SEND` = 1, `INFO` = 2.
- **Type codes:** `VAR` = 0 (well-known names), `VALUE` = 1, `ESC` = 2,
  `USERVAR` = 3 (caller-defined names).
- **Flow:** only the side that is `DO NEW-ENVIRON` may send `SEND`; only the side
  that is `WILL NEW-ENVIRON` may reply `IS`. The server sends
  `IAC SB NEW-ENVIRON SEND [VAR "X" ...] IAC SE`; a bare `SEND` requests the
  default set. The client replies
  `IAC SB NEW-ENVIRON IS VAR "X" VALUE "..." ... IAC SE`.
- **`VAR` vs `USERVAR`:** the distinction exists so a server can tell values the
  client software derived (`VAR`) from values a user configured (`USERVAR`).

#### MNES — Mud New Environment Standard

- **Carrier:** NEW-ENVIRON (39). A MU\* convention, not a separate option.
- **Standard `VAR`s:** `CLIENT_NAME`, `CLIENT_VERSION`, `CHARSET`,
  `TERMINAL_TYPE`, `MTTS`, and **`IPADDRESS`**.
- **`IPADDRESS`** — MNES documents this variable as *"suggested for proxies
  adding MTTS support to also report the client's real IP address. This allows a
  MUD to ban specific users without having to ban an entire proxy."*
- **Relevance to InterruptingCow:** this is the **purpose-built mechanism** for
  the shared-source-IP problem described in
  [ADR 0008](adr/0008-forwarded-user-identity.md). The follow-up transport ADR's
  starting point is MNES `IPADDRESS` for an opted-in real IP, plus a
  `NEW-ENVIRON` `USERVAR` carrying a pseudonymous per-user token for the default
  (privacy-preserving) case. Adoption is the open question: MNES support in
  server codebases is patchy, and a `USERVAR` convention would be new.

### Out-of-band structured data

#### GMCP — Generic MUD Communication Protocol

- **Option:** 201.
- **Message form:**
  `IAC SB GMCP "<Package.SubPackage.Message>" <json> IAC SE`. The JSON payload is
  optional; when present it is separated from the package name by one space and
  is UTF-8.
- **Negotiation:** server sends `IAC WILL GMCP`; client replies `IAC DO GMCP`.
  Clients should not initiate.
- **Handshake convention:** on connect the client sends
  `Core.Hello { "client": "<name>", "version": "<version>" }`, then
  `Core.Supports.Set [ "Char 1", "Room 1", ... ]` (and later `.Add` / `.Remove`)
  to declare which packages it understands.
- **Packages are server-defined.** Common ones: `Char.Vitals`, `Char.Login`,
  `Room.Info`, `Comm.Channel`. Each server documents its own set.
- **Direction:** bidirectional. This is why GMCP is a *plausible alternative*
  carrier for [ADR 0008](adr/0008-forwarded-user-identity.md) — a namespaced
  package such as `Client.Forwarded { ... }` fits the model and clients already
  send identifying data in `Core.Hello`. It would still be a new convention that
  destinations must choose to read.
- **Status in InterruptingCow:** registered but **stubbed** — a module exists so
  implementing it later is local.

#### MSDP — Mud Server Data Protocol

- **Option:** 69.
- **Payload:** binary key/value with nesting. Byte codes: `MSDP_VAR` = 1,
  `MSDP_VAL` = 2, `MSDP_TABLE_OPEN` = 3, `MSDP_TABLE_CLOSE` = 4,
  `MSDP_ARRAY_OPEN` = 5, `MSDP_ARRAY_CLOSE` = 6. Form:
  `IAC SB MSDP MSDP_VAR "NAME" MSDP_VAL "VALUE" IAC SE`.
- **Direction:** bidirectional. A client sends `LIST` to discover variables,
  `REPORT` to subscribe to a variable's changes, `SEND` for a one-shot request.
  Reportable variables are the server's predefined set, not arbitrary.
- **Purpose:** the same job as GMCP (out-of-band game data — stats, room, combat)
  but older and not JSON.
- **Status in InterruptingCow:** not currently planned. Listed for completeness;
  GMCP covers the same need and is more common in the social-MU\* servers we
  target.

#### MSSP — Mud Server Status Protocol

- **Option:** 70.
- **Payload:** `IAC SB MSSP MSSP_VAR "name" MSSP_VAL "value" ... IAC SE`
  (`MSSP_VAR` = 1, `MSSP_VAL` = 2).
- **Direction:** **server → client only.** The client never sends MSSP data.
- **Purpose:** lets a server advertise metadata — name, player count, uptime,
  codebase, contact — mainly for MUD-listing crawlers.
- **Status in InterruptingCow:** registered but **stubbed**. We would only ever
  *read* it.

### Rendering & transport

#### MXP — MUD eXtension Protocol

- **Option:** 91.
- **Payload:** HTML-like tags mixed **in band** with the text stream —
  `<b>`, `<color>`, `<send>` (a clickable command), `<a>` (a link), plus a
  "line tag" mechanism and security "modes" (open / secure / locked) that limit
  which tags the server may use where.
- **Direction:** primarily server → client; the client acts on `<send>` by
  transmitting the given command.
- **Status in InterruptingCow:** registered but **stubbed**. Rendering MXP safely
  (it is untrusted markup) is real work; deferred.

#### MCCP2 — MUD Client Compression Protocol, v2

- **Option:** 86. (MCCP1 was option 85, with a botched subnegotiation; v2
  supersedes it and is the one anyone uses.)
- **Mechanism:** the server sends `IAC SB MCCP2 IAC SE`, and **everything after
  that `IAC SE` on the server→client stream is a raw zlib (RFC 1950) stream.**
  The client must decompress from that byte onward. Client→server traffic is
  unaffected.
- **Direction:** compresses server → client only.
- **Status in InterruptingCow:** registered but **stubbed**. Straightforward to
  add (zlib inflate in the receive path, before the telnet decode) when wanted.

#### MSLP — Mud Server Link Protocol

- Referenced by an MTTS capability bit (1024) and occasionally seen in the wild;
  concerns server-to-server links / referrals. **Out of scope** for a client and
  not planned. Listed only because the MTTS bit is documented above.

---

## Quick reference

| Protocol | Option # | Carrier | Direction | InterruptingCow |
|---|---|---|---|---|
| SGA | 3 | native | both | negotiate |
| TTYPE / MTTS | 24 | native | server asks / client answers | answer, incl. proxy bit |
| EOR | 25 | native | server → client | detect prompts |
| NAWS | 31 | native | client → server | report dimensions |
| NEW-ENVIRON | 39 | native | server asks / client answers | carrier for MNES |
| MNES | (39) | NEW-ENVIRON | client → server | **candidate for ADR 0008** |
| MSDP | 69 | subneg (binary KV) | both | not planned |
| MSSP | 70 | subneg (KV) | server → client | stub (read only) |
| MCCP2 | 86 | subneg + zlib stream | server → client | stub |
| MXP | 91 | in-band markup | server → client | stub |
| GMCP | 201 | subneg (JSON) | both | stub; alt. carrier for ADR 0008 |
| CHARSET | 42 | subneg | both | negotiate; prefer UTF-8 |

---

## Sources

- Base telnet: RFC 854 (protocol), RFC 855 (options), plus per-option RFCs cited
  above (858 SGA, 885 EOR, 1073 NAWS, 1091 TTYPE, 1572 NEW-ENVIRON, 2066
  CHARSET).
- MU\* protocols: the protocol pages at
  [tintin.mudhalla.net/protocols](https://tintin.mudhalla.net/protocols/)
  (GMCP, MSDP, MSSP, MNES, MTTS, MCCP), the
  [GMCP writeup at gammon.com.au/gmcp](https://www.gammon.com.au/gmcp), and the
  MXP specification as published by Zuggsoft.
- These are community standards without a single authoritative registry; where
  servers diverge from the pages above, the server's own behaviour wins and the
  telnet state machine must degrade gracefully.
