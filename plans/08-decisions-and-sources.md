# Decisions, proposals, and sources

Updated 2026-10-09. This file distinguishes explicit owner choices from engineering proposals.

## Established direction
- Name: SyncQLight; new public GitHub repository ShaulLavo/syncqlight on /work/projects/syncqlight.
- Rust shared implementation and native server; TypeScript SDK/adapters. Earlier Zig choice superseded.
- Upstream SQLite C compiled directly into one browser WASM module with Rust, plus native build; browser OPFS.
- Server-authoritative ordering, local-first durable writes, first-class reactivity.
- Multiple logical SQLite databases, complete authorized workspace replicas initially.
- Drizzle wrapper/driver, not a fork; optional simpler typed APIs and framework adapters including Solid 2.
- Independent codebase: Fregat and sqlite-sync are prior art, not code dependencies.
- No dedicated CLI requirement; Game of Life is a showcase, not a stress test.

## Proposed defaults, not yet approved
- Tokio/Axum for server hosting.
- HTTP plus long polling before WebSockets.
- Concrete row/column effects for ordinary Drizzle writes; named authoritative operations later.
- Later accepted server transaction wins same-column conflicts; different-column patches merge.
- Initial bootstrap gates data-dependent writes, then offline writes allowed.
- One in-flight submission per replica initially.
- Stable immutable mutation IDs and durable receipts; conservative blocked pending suffix.
- Reactive SQL reruns for correctness, incremental patches for safe cases.

## Open technical gates
1. Rust/SQLite C compilation target and compatible browser OPFS VFS.
2. SQLite session changesets versus transactional observation queue and trigger/cascade semantics.
3. Accepted-base representation and bounded pending replay.
4. Drizzle callback transaction isolation and worker RPC.
5. Production auth/policy integration and custom server application-function runtime.
6. Migration/retention/backup epoch policy and resource limits.

## References
- SQLite session extension: https://www.sqlite.org/sessionintro.html
- SQLite WASM persistence: https://sqlite.org/wasm/doc/trunk/persistence.md
- SQLite preupdate hook: https://www.sqlite.org/c3ref/preupdate_hook.html
- SQLite authorizer: https://www.sqlite.org/c3ref/set_authorizer.html
- Rust build scripts: https://doc.rust-lang.org/cargo/reference/build-scripts.html
- Tokio: https://tokio.rs/
- PowerSync Drizzle integration: https://docs.powersync.com/client-sdks/orms/js/drizzle
- Fregat reference: https://github.com/ShaulLavo/fregat
- sqlite-sync reference: https://github.com/ShaulLavo/sqlite-sync

These are design references, not test evidence. The old chat attachment targeted Zig and sqlite-sync; this repository plan supersedes it.
