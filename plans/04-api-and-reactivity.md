# API and first-class reactivity

Status: illustrative API, not implemented.

Upstream Drizzle is the query builder. SyncQLight provides a SQLite-compatible driver and wrapper with a watch method. Do not fork Drizzle. PowerSync's wrapper is prior art, not a required dependency.

Example:

    import { openReplica } from "@syncqlight/client";
    import { wrapWithDrizzle } from "@syncqlight/drizzle";
    import * as schema from "./schema";

    const replica = await openReplica({
      database: "workspace:acme",
      endpoint: "https://sync.example.com",
      getToken,
    });
    await replica.ready();
    const db = wrapWithDrizzle(replica, { schema });
    const watched = db.watch(db.select().from(schema.tasks));
    const unsubscribe = watched.subscribe(snapshot => console.log(snapshot));
    await db.insert(schema.tasks).values({ id: "task-1", title: "Ship" });

Names are provisional. Drizzle's SQL construction is separate from SyncQLight's observation and synchronization.

Callback transactions must be real local SQLite transactions and one atomic sync submission. Transaction-scoped worker RPC, isolation, rollback on cancellation, and stable request identity are required. Awaiting a write means local commit; remote acceptance has a separate observable receipt/barrier.

## Engine-owned reactive contract
The native core coordinates initial query and subscription atomically; tracks conservative SQL table dependencies; shares equivalent subscriptions; reruns affected queries after commit; compares results; publishes consistent revisions; and cleans up disposed queries. Filtered, sorted, joined and limited SQL cannot be patched by assuming currently displayed rows are the only dependencies. Fine-grained patches are optional proven-safe optimizations.

A framework-neutral interface exposes getSnapshot(), subscribe(listener), and dispose(). Snapshots represent loading, ready, error, revision, and data completeness. Reconciliation never exposes intermediate restore/replay state.

Solid 2 gets an idiomatic async-reactive adapter; its server-function live API is not itself a local SQLite change detector. React gets a compatible external-store adapter; vanilla JS uses the core. Framework adapters never implement conflict resolution.

Optional model/state wrappers and named authoritative operations can be added later. Arbitrary shared TypeScript functions require an application JS runtime; the native Rust server cannot execute arbitrary JS closures magically.

The Game of Life demo showcases persisted SQLite-backed state, live updates and two-device sync. Large board sizes are optional, not its purpose.
