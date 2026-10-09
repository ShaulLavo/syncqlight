# Delivery milestones

Status: implementation not started. Each gate requires real executed tests and commit evidence.

## M0: Foundation and executable specification
Pin current Rust and SQLite versions; create only needed workspace structure; set up CI; define typed protocol and deterministic reference model. Encode duplicates, conflict matrix, rejected pending work, cursors, and crash semantics. Read Fregat and sqlite-sync only as references, never dependencies or extracted code.
Gate: model tests pass without network/UI.

## M1: Integrated SQLite engine
Compile upstream SQLite C directly into native Rust and one browser WASM module. Build a compatible OPFS VFS/worker host, transactions, and minimal live query observation.
Gate: same SQL fixtures pass native/browser, OPFS survives reload, rollback publishes nothing, initial subscription has no gap.

## M2: Durable outbox and reconciliation
Implement stable mutation/request IDs, atomic outbox, capture, accepted base, pending replay, blocked outcomes, coherent query publication. Build minimal Drizzle wrapper and vanilla consumer.
Gate: real engine matches reference traces and crash injection never separates edit from pending work.

## M3: Native authority
Implement multiple database routing, scoped authentication, policy, durable ordered feed/receipts, bootstrap, push/pull and reconnect.
Gate: two independent persistent browser replicas edit offline, restart, reconnect and converge. Lost response after server commit never duplicates effects.

## M4: SDK completion
Implement Drizzle callback transactions, receipts, lifecycle, multi-tab ownership, Solid 2 and React adapters.
Gate: same application schema works with vanilla JS, Solid and React, no sync algorithms in adapters.

## M5: Recovery and showcase
Implement schema migration, expired cursor recovery, retention, backup/restore, quotas, backpressure, benchmarks, deployment packaging, polished Game of Life.
Gate: reproducible fault/performance evidence and real two-device showcase.

Never mark a milestone complete because code merely compiles. Record command, environment, pass/fail, limitations, and commit for every gate.
