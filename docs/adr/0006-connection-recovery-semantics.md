# ADR 0006: Connection-recovery semantics — "forever" ≠ chase server drops

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

InterruptingCow's headline promise is that it keeps your world connections alive
"forever" so you can detach and reattach freely. We need to pin down exactly what
that means when connections fail, because the obvious reading ("always keep every
world connected, reconnecting whenever anything drops") is **not** what we want.

Forces and observations:

- **A MU\* server closing your connection is often meaningful.** It may be a
  `@boot`, an idle disconnect the game itself enforces, a ban, a reboot, or a
  crash. Silently reconnecting can be wrong — it can fight an intentional
  disconnect, spam a server that's rejecting you, or reconnect you into a
  situation you'd want to know about first.
- **Auto-reconnect loops are a classic way to hammer a server** and get an IP
  blocked, which is bad manners and counterproductive.
- **But the daemon's own failures are different.** If the daemon restarts, the
  host loses power, or the daemon's network drops, connections that were healthy
  and wanted are lost through no decision of the user's. Restoring those is the
  entire point of a persistent daemon.
- We need a way to tell "our outage" from "a deliberate stop hours ago" at boot,
  without a user-intent signal for every case.

## Decision

**The daemon never voluntarily drops an upstream connection, and never
automatically reconnects one that the server or the user closed. It
re-establishes connections automatically only when recovering from its own
outage.**

Concretely:

1. **No voluntary drops.** No idle timeout, no disconnect when the last client
   detaches, no periodic recycle. An optional per-world application-level
   keepalive (a periodic no-op write) may be enabled solely to keep NAT/firewall
   state; it does not drive reconnect logic.

2. **Server closes the connection → `disconnected`, and stop.** The Session and
   its scrollback are retained. A `status` with `detail: server-closed` (or
   `socket-error` for an unclean failure) is sent. **No backoff loop, no retry.**
   The Session stays `disconnected` until the user sends `connect`.

3. **User closes the connection → `closed`.** Exempt from all recovery.

4. **Our-fault boot recovery** (`recovery.ts`, runs once at daemon start):
   for each world, re-establish the connection **only if all** of:
   - the user's intent was connected (`intended_connected = true` at last
     checkpoint), **and**
   - the Session's state at the last checkpoint was `connected` (not
     `disconnected`, not `closed`), **and**
   - the world's `auto_recover` flag is on (default true), **and**
   - the daemon was down for less than a configurable **grace window** (the
     heuristic for "this was our outage").

   Recovery attempts are **bounded** with backoff. Success → `connected`
   (`detail: recovered`). Exhaustion → left `disconnected`
   (`detail: recovery-failed`), not retried further.

5. Every Session **checkpoints** `{ intended_connected, state, last_activity_at,
   last_checkpoint_at, scrollback_cursor }` to `session_state` so recovery has
   the data to make this decision.

`auto_recover` gates **our-fault recovery only**. It is explicitly **not** a
"reconnect when the server drops me" switch — no such switch exists.

## Consequences

**Easier / better behaviour:**

- The daemon is a well-behaved network citizen: it never hammers a server that
  dropped it.
- A meaningful disconnect (boot, ban, reboot) stays visible to the user instead
  of being papered over.
- After a daemon restart or a power blip, the user's connections come back
  without action, which is the persistent-daemon value proposition.
- The rules are simple to state and to test: four decision inputs, one grace
  window.

**Harder / the cost:**

- **Users must manually reconnect after a server-side drop.** This is a
  deliberate choice, but it's a UX cost and must be surfaced clearly in the
  client (a prominent "disconnected — reconnect?" affordance). Documented in the
  [user guide](../user-guide/worlds.md).
- The grace window is a heuristic. Too short and a slow restart skips recovery;
  too long and a deliberate overnight stop gets undone. It is configurable and
  per-world `auto_recover` lets a user opt a world out entirely.
- Auto-login on recovery (replaying stored world credentials) must be handled
  carefully so recovery doesn't, e.g., double-login.

**Committed to:**

- The `connecting / connected / disconnected / closed` state model and the
  `status` `detail` vocabulary in [the protocol](../protocol.md).
- Checkpointing enough state to drive recovery.
- Never adding a blanket auto-reconnect-on-server-drop without a superseding ADR.

## Alternatives considered

- **Always reconnect everything, with backoff.** The naïve "forever". Rejected:
  fights intentional disconnects, risks IP bans, hides meaningful events.
- **Reconnect on server drop, but only N times with long backoff.** Softer, but
  still reconnects into situations the user should see first (bans, boots), and
  still adds load to a server that just rejected us. If we ever want this, it
  should be an explicit, per-world, user-enabled option — a future ADR, not the
  default.
- **Never reconnect anything automatically, even after our own restart.**
  Simplest possible rule, but throws away the main benefit of a persistent
  daemon — a power blip would drop everyone.
- **Ask the user (a queued prompt) before every recovery.** Too heavy for the
  common case (a 20-second daemon restart); the grace window + `auto_recover`
  flag captures the intent with far less friction.
