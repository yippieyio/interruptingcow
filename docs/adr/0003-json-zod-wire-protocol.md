# ADR 0003: JSON + zod wire protocol in a shared package

- **Status:** Accepted
- **Date:** 2026-09-10
- **Deciders:** Lead maintainer

## Context

The web client and the daemon exchange two kinds of traffic: request/response
(auth, world management, history search) and a live bidirectional stream (lines
to and from worlds, status changes, client input). We need to define that
contract.

Forces:

- **The two ends are separate deployables on separate release cadences.** The
  contract must be explicit and versioned, not implied.
- **The contract must not drift.** A field the daemon adds and the client forgets
  to handle, or a type mismatch, should be caught early.
- **Debuggability matters** for a small project — being able to read the traffic
  in browser dev tools is valuable.
- **Line volume is modest.** A busy session is hundreds of short text lines per
  minute, not a high-throughput binary feed.
- Both ends are TypeScript ([ADR 0001](0001-all-typescript.md)), so a shared
  package is natural.
- We may, someday, want a more compact encoding.

## Decision

**We will define the wire protocol once, as zod schemas, in
`packages/protocol`,** and validate against them on both ends.

- **Every** WebSocket message and REST DTO is a zod schema. TypeScript types are
  **inferred from** the schemas (`z.infer`), so types and runtime validation
  share one source and cannot diverge.
- WebSocket messages are **discriminated unions** on a `t` field. Client→server
  and server→client namespaces are disjoint.
- Encoding is **JSON** for now, behind a **`Codec` interface**
  (`encode(msg) → bytes`, `decode(bytes) → msg`). `JsonCodec` is the only
  implementation initially; a binary codec can be dropped in later without
  touching any message handler.
- The **`hello` handshake carries an integer protocol version.** The daemon
  advertises a minimum and current supported version and rejects clients below
  the minimum. Additive changes don't bump it; breaking changes bump it and get
  an ADR + changelog entry.
- `packages/protocol` has **no runtime dependency beyond zod**.
- A fixture corpus (one sample per message) lives in the package's tests and
  doubles as the daemon↔client contract-test fixtures.

## Consequences

**Easier:**

- Adding or changing a message is one edit in one place; both ends get the new
  type and the new validation.
- Malformed or unexpected messages are rejected at the boundary with a clear
  error, not propagated as `undefined` into handler code.
- Traffic is human-readable in dev tools and logs.
- Independent client/daemon releases are safe because compatibility is
  negotiated.

**Harder / the cost:**

- JSON is more bytes on the wire than a binary format and costs a
  parse/serialize per message. At this volume it is not a concern; if it ever
  becomes one, the `Codec` seam is where we address it.
- zod validation has a small per-message CPU cost. Acceptable, and a strong
  safety return.
- Discipline required: **no message may be sent that isn't in a schema.** That's
  the point, but it means the schema PR comes first.

**Committed to:**

- `packages/protocol` staying dependency-light and being the single source of
  wire truth.
- Versioning discipline on breaking changes.
- The `Codec` abstraction, so the encoding stays swappable.

## Alternatives considered

- **A binary schema/RPC framework (Protocol Buffers, Cap'n Proto, MessagePack
  with a schema).** More compact and faster to (de)serialize, with codegen for
  types. Rejected for now: the volume doesn't justify it, JSON's readability
  aids debugging, and it would add a codegen step and a non-TypeScript schema
  language. The `Codec` seam keeps this available as a later optimisation.
- **Hand-written TypeScript `interface`s with no runtime validation.** Zero
  runtime cost, but nothing checks that what arrives on the wire actually matches
  the type — exactly the drift we're trying to prevent.
- **A single shared library imported as source (not a package).** Works, but a
  proper workspace package gives clean boundaries, its own tests, and its own
  versioning.
- **GraphQL / tRPC.** tRPC in particular fits an all-TypeScript stack, but it's
  oriented toward request/response RPC; our core is a long-lived push stream of
  domain events, which a plain typed WebSocket message union models more directly.
