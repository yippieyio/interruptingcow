# @interruptingcow/protocol

> **Status: design phase — no code yet.** This package contains only this README.

The **wire contract** between the InterruptingCow daemon and the web client: the
single source of truth for every message and data shape they exchange.

## Purpose

- Define **every WebSocket message** and **every REST DTO** as a
  [zod](https://zod.dev) schema.
- Export the **TypeScript types inferred from** those schemas (`z.infer`), so the
  static types and the runtime validation can never drift apart.
- Provide a **`Codec` interface** that abstracts the wire encoding, with a
  `JsonCodec` implementation. A binary codec can be added later without changing
  any message handler.
- Carry the **protocol version constant** and the compatibility rules.

## Boundaries

- Imported by **both** `@interruptingcow/daemon` and `@interruptingcow/web`.
- **No runtime dependencies beyond zod.** No Node APIs, no browser APIs — it must
  run in both.
- Contains **no transport code** (no WebSocket, no HTTP) and **no business
  logic** — only shapes, types, and the codec.

## Planned layout

```
src/
  messages/     # WebSocket message unions: hello, subscribe, input, line, status, …
  dto/          # REST request/response DTOs
  codec.ts      # Codec interface; JsonCodec; (a binary codec later)
  version.ts    # protocol version + compatibility helpers
  index.ts      # barrel export
test/
  samples/      # one fixture per message — also used by the daemon/web contract tests
```

## Reference

- [Wire protocol specification](../../docs/protocol.md)
- [ADR 0003: JSON + zod wire protocol](../../docs/adr/0003-json-zod-wire-protocol.md)

## Testing (planned)

`pnpm --filter @interruptingcow/protocol test` — codec round-trips, schema
acceptance/rejection, and the sample corpus.
