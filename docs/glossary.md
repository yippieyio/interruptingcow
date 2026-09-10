# Glossary

Terminology used across the InterruptingCow documentation — the text-world
lineage, the telnet layer, and the project's own vocabulary.

## Text worlds

**MUD** (Multi-User Dungeon)
: The original family of text-based multiplayer games, from 1978 onward.
  Combat- and level-oriented in the classic form. Used loosely as the umbrella
  term for all of the below.

**MUSH** (Multi-User Shared Hallucination)
: A text world oriented toward social interaction and collaborative
  role-play, with in-world building and a soft-coding language. TinyMUSH,
  PennMUSH, RhostMUSH, TinyMUX are common servers.

**MUCK**
: Closely related to MUSH; the TinyMUCK lineage (e.g. Fuzzball). Also
  social/role-play oriented, with the MUF programming language.

**MOO** (MUD, Object-Oriented)
: A text world whose entire environment is live, user-editable objects in an
  object-oriented database. **LambdaMOO** (1990) is the canonical server and the
  spiritual ancestor of this project's name.

**MU\***
: Shorthand for "MUSH / MUCK / MOO / MUD and relatives" — the whole category.
  Pronounced "moo-star" or just spelled out. This project targets MU\* servers
  generically.

**Soft code**
: In-world scripting (MUSHcode, MUF, MOO-code) that players and builders write
  to give objects behavior. Not relevant to the client, but explains why worlds
  send richly formatted, sometimes markup-laden text.

**Page**, **tell**
: Private directed messages between players ("Jane pages: are you around?").
  These are the events a user most wants a notification for.

**Pose**, **emote**
: Third-person actions in role-play ("Jane waves to the room").

**Room spam**
: High-volume, low-importance output — movement, weather, ambient poses — that a
  user does *not* want a notification for. The reason push must be gated on
  triggers.

## Telnet and the wire

**Telnet**
: The 1970s terminal protocol most MU\* servers still speak. In practice a MU\*
  connection is a mostly-plaintext TCP stream with occasional telnet
  **option negotiation** interleaved.

**IAC** (Interpret As Command)
: Byte `255`. Introduces a telnet command sequence within the stream. The
  daemon's telnet state machine exists to handle these without corrupting the
  visible text.

**WILL / WONT / DO / DONT**
: The four telnet negotiation verbs. One side offers (`WILL`/`WONT`) or requests
  (`DO`/`DONT`) an option; the other accepts or refuses.

**SB / SE** (Subnegotiation Begin / End)
: Frame arbitrary option-specific data between `IAC SB <option> … IAC SE`. How
  NAWS, CHARSET, GMCP, MXP, MSSP all carry their payloads.

**NAWS** (Negotiate About Window Size)
: Telnet option `31`. The client tells the server its terminal width and height.
  InterruptingCow answers this from the web client's reported dimensions.

**TTYPE** / **MTTS** (Terminal Type / Mud Terminal Type Standard)
: Telnet option `24`. The client identifies its terminal type; MTTS extends this
  with a capability bitmask (ANSI, UTF-8, 256 colour, etc.).

**EOR** (End Of Record, option `25`) / **GA** (Go Ahead)
: Markers a server can send to delimit a **prompt** from a normal line, so a
  client can render prompts specially.

**SGA** (Suppress Go Ahead, option `3`)
: Switches the connection to full-duplex character-at-a-time-friendly operation;
  almost always negotiated.

**CHARSET** (option `42`)
: Negotiates the character encoding. InterruptingCow prefers UTF-8 when offered
  and otherwise falls back to the per-world configured encoding.

**ANSI SGR** (Select Graphic Rendition)
: The `ESC [ … m` escape sequences that set colour and text style. Passed
  through as data and rendered by the client (optionally pre-parsed into
  `spans` by the daemon).

**GMCP** (Generic MUD Communication Protocol)
: A telnet subnegotiation option carrying structured JSON messages (character
  stats, room info, etc.) out of band from the text. *Planned; stubbed.*

**MXP** (MUD eXtension Protocol)
: In-band HTML-like markup for clickable links, sendable commands, and styling.
  *Planned; stubbed.*

**MCCP2** (MUD Client Compression Protocol, v2)
: A telnet option that zlib-compresses the server→client stream. *Planned;
  stubbed.*

**MSSP** (MUD Server Status Protocol)
: Lets a server advertise metadata (name, player count, uptime). *Planned;
  stubbed.*

## Project vocabulary

**Daemon**
: The long-running InterruptingCow server process. Holds the upstream
  connections, serves the web client, owns the database.

**World**
: A configured game connection belonging to a user — host, port, TLS flag,
  encoding, and options. A user has many worlds.

**Session**
: The daemon's live object for one persistent connection to one world for one
  user. Owns the socket, the line pipeline, and the attached client subscribers.
  Has a state: `connecting`, `connected`, `disconnected`, `closed`.

**SessionManager**
: The daemon component that owns every Session, routes client subscriptions, and
  runs recovery at boot.

**Attach / detach**
: A web client **attaches** to a Session to receive its live stream and send
  input, and **detaches** when it goes away. Detaching does not stop the Session.

**Line pipeline**
: The single path every line takes: decode → normalize → log → trigger-evaluate
  → fan out (receive), or encode → send → log (transmit). The one place features
  hook in.

**Scrollback**
: The history of lines for a world. Stored in full in the database; a recent
  slice is **backfilled** to a client when it attaches; all of it is
  full-text searchable.

**Backfill**
: The chunk of recent scrollback the daemon sends a client right after it
  subscribes, so the user has context before the live stream resumes.

**"Forever"**
: The property that the daemon never voluntarily closes an upstream connection.
  It does **not** imply reconnecting after the *server* closes one.

**Our-fault recovery**
: The narrow case where the daemon *does* reconnect automatically: after its own
  outage (restart, power loss, local network loss), re-establishing connections
  that were healthy and intended-connected before the incident, within a grace
  window. See [ADR 0006](adr/0006-connection-recovery-semantics.md).

**Grace window**
: The maximum downtime for which our-fault recovery will act. A restart that
  took longer is assumed to be a deliberate stop, and connections are left as
  they were.

**Trigger**
: A user-defined rule matching lines from a world and producing actions.
  *Planned.*

**Action** (of a trigger)
: One of `gag` (hide the line), `highlight` (style it), `send` (write a command
  back), `notify` (raise a notification). *Planned.*

**Gag**
: A trigger action (and the classic MU\*-client term) for suppressing matching
  lines from the display while still logging them.

**Notifier**
: The daemon component that turns `notify` actions into delivered notifications
  — Web Push in the planned implementation. No-op in the MVP.

**Terminal gateway**
: The (deferred) component that would let a user attach from a real terminal
  rather than the web client — over TLS or SSH, never plaintext telnet. Stubbed
  as `NullTerminalGateway`. See
  [ADR 0004](adr/0004-defer-terminal-gateway.md).

**Protocol package**
: `packages/protocol` — the shared zod schemas and inferred TypeScript types that
  define every client↔daemon message and REST DTO. The single source of truth for
  the wire format.

**Codec**
: The pluggable encoder/decoder for WebSocket frames. `JsonCodec` now; a binary
  codec possible later without changing message handlers.

**icow**
: The daemon's admin command-line tool (migrations, user and world management,
  token minting).

## Standards and licenses

**PWA** (Progressive Web App)
: A web application that can be installed to a device's home screen / app list
  and run in its own window, with a service worker for the offline shell. The
  form the InterruptingCow client takes.

**Service worker**
: A background script the browser runs for an installed PWA — caches the app
  shell, and (later) receives Web Push messages.

**Web Push** / **VAPID**
: The browser standard for server-sent push notifications to an installed PWA.
  VAPID is the key scheme that identifies the sending server. *Planned.*

**Argon2id**
: The password-hashing function InterruptingCow uses for account passwords.

**JWT** (JSON Web Token)
: The format of the short-lived access token.

**AGPL-3.0-or-later**
: The GNU Affero General Public License, version 3 or later — InterruptingCow's
  license. Unlike the GPL, it requires that users of a *network service* running
  the software be offered its (possibly modified) source. See
  [`LICENSE`](../LICENSE) and [ADR 0007](adr/0007-licensing-and-workflow.md).

**DCO** (Developer Certificate of Origin)
: A lightweight, sign-off-based alternative to a CLA for certifying that a
  contributor has the right to submit their contribution. See
  [`CONTRIBUTING.md`](../CONTRIBUTING.md).

**ADR** (Architecture Decision Record)
: A short document capturing one significant decision, its context, and its
  consequences. See [the ADR process](adr/0000-adr-process.md).

**Zero-knowledge encryption** (here)
: Encrypting a user's logs with a key the server never has in usable form, so the
  operator cannot read them. *Planned.*
