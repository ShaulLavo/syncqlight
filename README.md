# SyncQLight

**Your SQLite. Everywhere.**

A SQLite-native, local-first sync engine being built with Rust.

## Status

Repository initialized. The sync engine, server, SDK, and adapters are not implemented yet. The capabilities below describe the intended product, not available features.

## Product direction

- Upstream SQLite, with SQLite's C source compiled directly into the engine.
- A shared Rust sync core for the native server and browser WebAssembly build.
- One integrated browser WASM binary, with worker execution and OPFS persistence.
- Server-authoritative ordering, durable local mutations, and reconciliation.
- Multiple databases, initially with complete authorized workspace/database replicas.
- First-class reactive SQL queries with transaction-consistent publication.
- A TypeScript SDK and a Drizzle driver/wrapper, not a Drizzle fork.
- Optional higher-level typed APIs and thin Solid 2, React, and vanilla integrations.

## Implementation boundaries

This is a fresh implementation. Fregat and the earlier sqlite-sync experiment are reference material only; neither is a dependency or a shared-core extraction target.

Server runtime selection, exact conflict rules, change-capture strategy, and custom server-function execution remain design decisions. A document CRDT is not required for ordinary SQL synchronization.

There is no dedicated product CLI planned. Build tooling and a server executable are separate concerns.
