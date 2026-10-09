# Architecture and storage

Status: implementation design. See [the decision register](08-decisions-and-sources.md) for alternatives that remain open.

## System boundaries

```text
Application: upstream Drizzle / optional typed model API
                          |
TypeScript SDK and thin framework adapters
                          |
Browser host: worker RPC, fetch, storage handles, tab coordination
                          |
One WASM module: Rust execution + reactivity + sync + upstream SQLite C
                          |
OPFS-backed SQLite: application tables and protected local metadata
                          |
Versioned authenticated synchronization protocol
                          |
Native Rust server: routing, authorization, per-database ordering
                          |
Native SQLite files: canonical tables, durable feed, mutation receipts
```

Both targets share mutation normalization, typed values, protocol validation, reconciliation rules, and reactive-query scheduling where appropriate. Native networking, browser APIs, and storage primitives are platform boundaries. A single WASM binary does not eliminate the JavaScript host needed to call browser APIs.

## Proposed repository layout

Create directories when the milestone puts real code and tests in them, not as empty scaffolding.

```text
crates/
  core/          # Shared protocol/state machines; no browser or Tokio dependency
  sqlite/        # SQLite build, narrow unsafe FFI, connection and transaction layer
  server/        # Native authority, database manager, policy and transport adapters
  wasm/          # Browser ABI and host imports using the same core and SQLite layer
packages/
  client/        # Worker host, SDK, typed errors and lifecycle
  drizzle/       # Upstream Drizzle driver/wrapper
  solid/         # Solid 2 adapter
  react/         # React adapter
examples/
  minimal/       # Small vanilla integration fixture
  playground/    # Showcase after the end-to-end sync gate
spec/            # Wire/schema fixtures and executable scenario descriptions
tests/           # Cross-target and end-to-end tests
tools/           # Build/test automation, not a product CLI
plans/           # This implementation plan
```

Physical crate boundaries can be consolidated if they add boilerplate without isolation. Do not duplicate semantic implementations merely to fit this layout.

## Direct SQLite compilation: first technical gate

Acquire the current upstream amalgamation from an authoritative source, record its version and checksum, and compile it with the Rust code for each target. Record the Rust toolchain, C compiler/runtime, flags, and dependency lockfiles. Follow current releases through deliberate upgrades; do not use unrecorded floating toolchains. No Zig compiler is required by the agreed architecture.

Enable `SQLITE_ENABLE_SESSION` and `SQLITE_ENABLE_PREUPDATE_HOOK` for the capture experiment. These enable existing upstream features, not a maintained SQLite fork. The session extension has documented key/table requirements; our schema gate must enforce the supported subset. [S1]

The first browser experiment should evaluate Rust's Emscripten target with SQLite C compiled for the same ABI, runtime, memory, and linker configuration. Rust documents this target and its compatibility caveats. This is an experiment, not a declaration that every Rust SQLite binding works there unchanged. [S2]

A custom freestanding ABI is an alternative only when there is a documented reason; it must provide SQLite's necessary C runtime and host interfaces. Do not pair an Emscripten C object with an incompatible Rust target and describe the failure as a database issue. Do not silently replace the integrated module with separately loaded SQLite and Rust WASM modules.

The gate must prove native and browser SQL execution, session symbol availability, persistence, error propagation, and a live query update. Use a small direct FFI layer if a higher-level crate cannot meet the build target; do not invent a large binding framework.

## Browser persistence and ownership

Implement or adapt an OPFS-backed SQLite VFS compatible with the integrated binary. Evaluate the upstream SAH-pool design as the initial reference, including its exclusive ownership constraints. Official browser VFS helpers may depend on upstream glue; validate reuse rather than assuming drop-in compatibility. [S3]

Specify and test open/read/write/truncate/size/sync/close/delete operations, journal handling, short reads/writes, lock lifetime, directory initialization, and file recovery. Report synchronous storage failures faithfully. Use the journal mode actually supported by the selected VFS; do not assume native WAL/shared-memory behavior transfers unchanged to browser storage.

Prefer one owning dedicated worker for each physical replica. All tabs route reads, writes, and live subscriptions through that owner. Use Web Locks for exclusive ownership and owner-generation fencing. A SharedWorker may broker clients, but the design must not depend on spawning a dedicated worker from a SharedWorker without proving that platform capability.

Begin with safe exclusive ownership and an explicit database-in-use result. Add brokered multi-tab access and recovery before declaring the browser SDK complete. Tabs must not open the same exclusive storage pool independently. Worker replacement reloads durable state and rebuilds subscriptions; transient UI revisions are not durable replication cursors.

Namespace physical replica storage by endpoint identity, principal, logical database ID, and schema/history identity as appropriate. Logging out or changing account must not expose another principal's cached data. Never use raw client database names as storage paths.

OPFS failures must not silently select memory storage. User deletion, quota exhaustion, private mode, or eviction can remove or prevent persistence; expose this limit. Memory mode is explicit and useful for tests only unless the application deliberately opts in. [S3]

## Rust/JS boundary and ownership

Keep one owner of each SQLite connection. Clearly separate borrowed pointers, owned SQLite allocations, Rust allocations, and JS copies. Memory is freed by the allocating layer. No borrowed SQLite value outlives statement reset/finalization, and no JS typed view is retained across memory growth without refreshing it.

Use handles and bounded messages, not arbitrary JS objects crossing the ABI. Transfer batch results/deltas rather than making a host call per row or cell. Use `i64`-safe encodings and JavaScript `bigint` or tagged decimal values; never coerce unsafe integers to Number. Preserve SQL NULL, text, BLOB, float, and integer distinctions. Define malformed UTF-8 and floating-point edge behavior explicitly in encoding fixtures.

Panics must not unwind through C or JS boundaries. Errors cross as stable structured categories with useful SQLite codes and operation IDs, not string-only crashes. A fatal instance failure must stop further work and reopen through recovery, not continue using a potentially invalid connection.

## Capture, transactions, and reactivity

A local write owns a SQLite transaction that contains both application changes and durable pending-mutation metadata. A reported successful local commit cannot leave only one side persisted. A failed write rolls back both, including statements that otherwise have partial `FAIL` semantics. Readonly queries are not wrapped as replicated mutations.

Evaluate transaction-scoped session changesets as the default capture candidate. They represent net row effects, not a chronological event log; recreate/reset capture at the correct boundary. Verify rollback, savepoint, BLOB, composite-key, generated-value, and cascade behavior experimentally. A separately implemented transactional observation queue is an alternative if a concrete required behavior cannot be handled cleanly. [S1]

Do not install another preupdate callback over one owned by SQLite's session extension. Commit hooks must not reenter the connection; a callback running is not proof that a transaction durably committed. Observation and publication occur after the owning transaction completes successfully. [S4, S5]

Maintain distinct execution modes: local authoring, accepted remote application, pending replay, and migration. Remote application/replay must not produce new outbound mutations. Reactor invalidations derive from final committed state across modes and publish only coherent revisions. No mandatory permanent full-row JSON log for each reactive notification.

## Acknowledged state versus optimistic projection

The protocol needs an accepted base plus a pending overlay, but not two public data models. Production must not copy the entire database for every transaction. During M2 compare affected-row undo/replay with protected acknowledged-row storage and choose one. Measure cost against actual pending queues and record crash recovery rules.

Any inverse/delta retained for a pending operation must be recomputed when replay changes its realized local effects. SQL constraints, trigger policies, and server canonical changes must remain coherent during restore/replay. Other requests cannot read intermediate restoration state.

## Sources

References resolve in [08-decisions-and-sources.md](08-decisions-and-sources.md): S1 SQLite session extension; S2 Rust Emscripten target; S3 SQLite browser persistence; S4 session/preupdate ownership; S5 commit/rollback callbacks.
