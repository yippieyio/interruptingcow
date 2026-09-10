# History and Search

> **Status: design phase.** Not built yet — this is the intended behavior.

InterruptingCow logs **every line** to and from **every world** — what the game
sends and what you send — and keeps it. Nothing said while you were away is lost,
and you can search all of it.

## Scrollback

Each world has a **scrollback**: its history, in order.

- When you open a world, InterruptingCow shows you a chunk of **recent**
  scrollback right away — enough to pick up the thread — and then the live stream
  continues from there. This is the **backfill**.
- Scroll up to load older lines.
- Lines you sent are marked distinctly from lines the game sent.

Being disconnected — by you, or by the game — does **not** clear scrollback. When
you reconnect, the history is still there and the new lines append to it.

## Search

A **search** box lets you look through a world's entire logged history, not just
what's currently on screen.

- **Full-text search** — type words or phrases; InterruptingCow finds the lines
  that match, across every session, going back as far as your operator's
  retention settings keep.
- **By time** — narrow to a date or a range.
- Results link back into the scrollback at that point, so you can read the
  surrounding context.

Use it to find that scene from last month, the name someone mentioned, the
command syntax you worked out and forgot.

## Where history is kept

On the **server**, in its database — not on your device. That's why:

- It's the same from every device you sign in on.
- It survives you reinstalling the app or getting a new phone.
- Its safety depends on your operator's backups (see the
  [operations guide](../operations/backup-restore.md)).

## Retention

Your operator decides how long history is kept. By default InterruptingCow keeps
everything; an operator with limited disk may set a retention period (e.g. keep
the last N months). Ask your operator what applies to your instance.

## Privacy and encryption

Today, your operator's database contains your logs in readable form — the same
as any server you'd play through. **Zero-knowledge encryption** is planned: an
option to encrypt a world's logs with a key derived from a passphrase only you
know, so that even the person running the server can't read them. When that
lands, search for an encrypted world happens on your device rather than on the
server. See the [roadmap](../../ROADMAP.md).

## Exporting

Not in the first release. Being able to download a world's log (for your own
archive, or to move it elsewhere) is on the list of things to add. The
[AGPL](../../LICENSE) guarantees you can run your own InterruptingCow; a
first-class export is a convenience on top of that.
