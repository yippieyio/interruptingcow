# Managing Worlds

> **Status: design phase.** Nothing here is usable yet — this is the intended
> behavior, written for review.

A **world** is one game you connect to: its address, and how InterruptingCow
should talk to it. You can have as many as you like.

## Before you start: everyone on this instance shares one IP address

InterruptingCow connects to games on your behalf from **the server it runs on**,
not from your own computer or phone. So every connection InterruptingCow makes —
yours and every other user's on the same instance — **appears to the game to come
from the same IP address.**

Most games log the connecting IP address, and many use it to enforce policy. That
has consequences you should understand before you rely on InterruptingCow for a
game you care about:

- **Shared alt limits.** A game that limits you to, say, ten alternate characters
  ("alts") usually counts them per IP address. If you share an instance, that
  limit is shared. *Example:* Dick and Jane both use Jane's InterruptingCow to
  play on LimitedMOO, which allows ten alts per person. Jane has four. Dick can
  now create at most six before the instance is over LimitedMOO's limit — even
  though Dick and Jane are different people.
- **Identity conflation.** Staff who see several characters from one address may
  assume they're one person. Something you tell staff in confidence could be
  associated with, or revealed to, another user of your instance.
- **Shared fate on bans.** Site bans are often IP bans. If one user of an
  instance is banned from a game, the others very likely are too — and if you're
  banned, so are they.

None of this is the games misbehaving. Limiting by IP address is often the only
practical way a volunteer-run game can defend itself against ban evasion and
abuse.

**What to do about it:**

- If any of the above matters to you — you have alts to protect, you value
  keeping your play compartmentalised, or you can't afford a shared-fate ban —
  **run your own single-user instance**, or use a traditional client from your
  own machine for that game.
- If you share an instance, treat it as you would a shared household connection:
  coordinate alt counts with the other users, and assume a game's staff may see
  your characters as linked.

InterruptingCow intends to offer games a *courtesy* mechanism to tell individual
users of one instance apart (the MU\* equivalent of a web proxy's
"X-Forwarded-For"), but **games are under no obligation to use it**, and it does
not exist yet. Until a game supports something like it, everything above applies
in full. The reasoning is in
[ADR 0008](../adr/0008-forwarded-user-identity.md).

## Adding a world

In the MVP, worlds are added for you by your operator (there's no signup, and
world management in the app comes a little later). You give them:

| Setting | What it is |
|---|---|
| **Name** | a label for you (e.g. "ExampleMUSH") |
| **Host** | the server's address (e.g. `mush.example.net`) |
| **Port** | the server's port (e.g. `4201`) |
| **TLS** | whether the server expects an encrypted (TLS) connection — most don't; some do |
| **Encoding** | the character set the server uses if it's not UTF-8 (e.g. `latin1`); usually leave as UTF-8 |
| **Auto-login** (optional) | credentials for the game itself, so InterruptingCow can log you in automatically. Stored encrypted. |

## Connecting and disconnecting

Each world shows a **status**:

| Status | Meaning |
|---|---|
| **Connecting** | InterruptingCow is establishing the connection |
| **Connected** | you're on; the server holds this open indefinitely |
| **Disconnected** | not connected right now — but your scrollback is kept |
| **Closed** | you deliberately disconnected this world |

- **Connect** — tell InterruptingCow to connect (or reconnect) the world.
- **Disconnect** — tell it to drop the connection. It stays dropped until you
  connect again.

## What "kept alive" means

When you're **connected**, InterruptingCow **never hangs up on its own**:

- Closing the app doesn't disconnect you.
- Detaching on your phone and reattaching on your laptop doesn't disconnect you.
- There's no idle timeout.

When you reopen the app you'll see the lines that arrived while you were away,
and the live conversation continues.

## When the *game* disconnects you

If the **game server** closes your connection — a reboot, an idle kick, a
`@boot`, a ban — the world goes to **Disconnected**, and InterruptingCow **does
not automatically reconnect it.** This is deliberate:

- A disconnect from the game often means something you should see (you were
  booted, the game is down, you're banned).
- Automatically hammering a server that just dropped you is bad form and can get
  your address blocked.

So the world waits, showing **Disconnected**, until you press **Connect**. Your
scrollback is untouched.

## When *InterruptingCow* restarts

If the InterruptingCow **server itself** restarts — an update, a reboot, a power
blip — that's not your fault or the game's. In that case InterruptingCow **does**
reconnect the worlds that were connected before, automatically, as soon as it's
back (as long as the outage was brief). Worlds you had deliberately
**Disconnected** or **Closed** stay that way.

Each world has an **Auto-recover** option (on by default). Turn it off for a
world you'd rather always reconnect by hand.

_(This behavior is specified in
[ADR 0006](../adr/0006-connection-recovery-semantics.md) if you want the full
reasoning.)_

## Scrollback and history

Everything said in a world — by you and by the game — is saved. See
[History & search](history.md). Disconnecting or being disconnected never loses
history.

## Multiple devices

Worlds, scrollback, and settings live on the server. Sign in on another device
and everything is there. You can be attached from several devices at once; what
you type on one appears on all of them.
