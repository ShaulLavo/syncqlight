# SyncQLight: Conversation decisions and design history

Updated: 2026-10-09. This is a structured record of our design conversations, **not** a verbatim transcript and **not** a claim that any code has shipped.

> **IMPORTANT FOR AGENTS:** This document contains superseded proposals. The latest direction is in **Current direction** and **Open questions**. Earlier files in plans/ describe a prior native Rust HTTP server and Drizzle-first API; they must not be treated as immutable requirements. Before implementing server/API milestones, reconcile those plans with this record and seek approval for genuine product choices.

## Current direction, as last discussed

- **Product:** SyncQLight, a SQLite-native, local-first, reactive sync engine with a Convex-inspired function API. The developer should think in terms of **one logical database**, not manually selecting local vs remote execution.
- **Storage/runtime:** upstream SQLite C directly linked into the Rust engine. Browser Rust+SQLite WASM with OPFS; native Rust core on the server via Node/Bun bindings (N-API or another proven binding). TypeScript for developer-facing application functions, client, and adapters. The owner chose Rust after considering Zig.
- **Authority:** server-ordered acceptance, optimistic local state, and **Matthew Weidner-style reconciliation without mandatory CRDTs**. Durable local pending work, acknowledged base + pending replay, coherent query publication. FugueMax is text-specific prior art, not the SQL conflict algorithm.
- **Server hosting, latest proposal:** the **user's Node or Bun process** hosts SyncQLight's authoritative function dispatcher and embedded Rust database core. Avoid a second general-purpose Elysia/Hono router and avoid requiring an independently deployed Rust HTTP server. Function registration and RPC transport are library responsibilities. We should not build a general HTTP framework unless genuinely needed.
- **Developer API, latest proposal:** Convex-style query/mutation functions with typed RPC calls, a DB context that can execute **ordinary Drizzle queries or parameterized raw SQL**, and framework-native reactive subscriptions. Strong interest in **no generated API files**, via a TypeScript router/function type exported as a type to the client. This requires TS server functions to execute in the user's Node/Bun runtime; Rust owns the database/sync machinery.
- **Schema:** the user prefers reusing Drizzle's SQLite schema and query builder rather than creating a SyncQLight ORM/schema DSL. Drizzle should be optional for low-level SQL access. Do not claim we have settled whether Drizzle is a required schema dependency.
- **Scopes:** intended product may use **selective, authorized, explicit scopes**, with full-database/full-workspace replication as a valid simple mode. The user does not want scope plumbing or self user IDs in every query. Identity comes from authenticated server context. A watched registered query may declare/acquire required scope automatically. Avoid confusing scope selection with conflict resolution.
- **Reactivity:** first-class in the core from day zero, framework-agnostic subscriptions and thin Solid 2/React adapters. Solid 2 async reactivity and server-function live are relevant but not themselves SQLite change detection.
- **User experience:** local writes can return after durable local commit, and synchronize later. The system must not falsely claim authoritative acceptance. A separate bounded wait can throw unavailable/timeout while preserving queued work. Acknowledgment and canonical feed inclusion are distinct.
- **Code/repo:** new public GitHub repo https://github.com/ShaulLavo/syncqlight at `/work/projects/syncqlight` on the Work drive. No home-directory checkout. The README placeholder was deliberately deleted. No engine code has been implemented as of this record. Fregat and sqlite-sync are reference projects, **not dependencies**.
- **Brand:** SyncQLight (earlier Syncqlite, Synchronistic, Rivulet). The product is a sync engine, not a CLI product. Possible tagline: “Your SQLite. Everywhere.”

## Conversation evolution and superseded approaches

1. **sqlite-sync origin.** Started by auditing an old browser SQLite reactivity project, trigger/log/worker overhead, hooks, custom SQLite WASM, Drizzle and Solid. Game of Life should demonstrate SQLite-backed persistence and live updates elegantly, not be a benchmark torture test. The old repo stays reference material.
2. **Product reset.** The owner explicitly wanted **a sync engine**, not merely a reactive SQLite library. Compared Convex, Jazz, Zero, Electric, PGlite, TanStack DB, SQLSync and others.
3. **CRDT discussion.** Loro, Yjs, FugueMax and custom editor CRDTs were considered. Chose **server authority + Weidner-style pending replay** for ordinary SQL; no mandatory document CRDT. Optional collaboration-specific CRDT support remains a future possibility, not a v1 dependency.
4. **SQLite versus Turso/libSQL.** Chose **upstream SQLite** for control over our sync semantics, with SQLite session/preupdate facilities as candidates. Turso's Rust engine/CDC and managed deployment were studied, but not selected. A custom build configuration does not imply maintaining a SQLite fork.
5. **Rust versus Zig.** Zig was appealing for compiling C SQLite directly, and agent skill/version concerns were discussed. Ultimately chose **Rust**, which can also compile/link SQLite C into the WASM module, for its ecosystem and Rust-based sync/CRDT possibilities. Browser OPFS VFS remains a major integration spike. No need to keep both Rust and Zig implementations.
6. **Fregat audit.** Fregat contains text-specific FugueMax Host/Participant and a partly generic session protocol. The owner explicitly chose **not to extract or reuse its core**; study algorithms/tests as prior art only. SQL transaction semantics differ from text operation envelopes and peer-host handoffs.
7. **First repository plans.** Initial plans favored a native Rust server, complete authorized workspace replicas, raw SQL/Drizzle-first API, explicit scopes later, and optional named actions. Those are **historical plans**, not all current choices.
8. **Replication scope research.** Linear, Notion and libraries showed that selective sync is not equivalent to a cache or a global clone. Scopes/channels/streams/query-driven sync differ. Proposed complete local datasets within explicit authorized scopes, with completeness checkpoints, overlap retention, moves between scopes and revocation. Full-workspace replication remains a simple mode. Exact selective-sync semantics remain open.
9. **API exploration.** Considered raw SQL + `watch`, Drizzle wrapper, model patch helpers, typed named operations, generated Rust RPC contracts, and shared query definitions. The user wanted something simpler, one logical database, and no repeated own-user IDs.
10. **Convex-style shift.** User leaned toward copying Convex's function API. Discussed `query` and `mutation`, auth context, live subscriptions, and typed RPC. The user suggested **Drizzle inside the function context**, plus raw SQL, and avoiding generated files by using TypeScript type inference across a shared router type.
11. **Hosting correction.** A Node/Bun SDK with `sync.elysia()` was rejected as redundant. Then a proposed SyncQLight general HTTP router was questioned. Final insight: **Convex functions are addressed through internal function registration/dispatch; users don't write HTTP routes for each query/mutation.** The owner wants a library that supplies that dispatcher inside the user's Node/Bun process, not a duplicate full web framework.
12. **Latest question.** The owner asked to “write down all our sessions.” This document records decisions, proposed API direction, and open questions, rather than prematurely treating sketches as final contracts.

## Proposed API sketch (NOT agreed syntax)

Shared server module (TypeScript, executed by user's Node/Bun host):

```ts
import { defineApp } from "syncqlight/server";
import { eq } from "drizzle-orm";
import * as schema from "./schema";

export const app = defineApp({ schema, authenticate });

export const api = app.router({
  tasks: {
    mine: app.query(async ({ db, auth }) => {
      const user = await auth.requireUser();
      return db.select().from(schema.tasks)
        .where(eq(schema.tasks.ownerId, user.id));
    }),
    complete: app.mutation(async ({ db, auth }, { taskId }) => {
      const user = await auth.requireUser();
      // Validate ownership and apply one tracked transaction.
      await db.update(schema.tasks)
        .set({ completed: true })
        .where(eq(schema.tasks.id, taskId));
    }),
  },
});

export type Api = typeof api;
```

Client (type-only import, no generated files):

```ts
import { createClient } from "syncqlight/client";
import type { Api } from "../server/app";

const client = createClient<Api>({ url: "/sync", getToken });
const tasks = client.tasks.mine.watch();
const receipt = await client.tasks.complete({ taskId });
await receipt.waitForAcceptance({ timeoutMs: 10_000 });
```

These names are **examples only**. The server function model, bundling and local prediction are not solved just because the types infer. No-codegen type inference requires a type-accessible shared module; production clients must not import server secrets or execute server-only code. Runtime validation and deployed contract compatibility remain necessary.

Raw SQL should remain possible through a parameterized SQLite execution interface; Drizzle is an optional ergonomic query builder. The same tracked transaction path must capture all supported writes.

## Critical semantic boundaries

### Weidner reconciliation is not selective sync
- **Conflict resolution:** authoritative order, accepted base, pending replay; no general CRDT needed.
- **Replication scope:** which authorized rows/tables are on the device, whether complete, and how they are acquired.
- A full replica may still be stale after disconnect. A scope may be complete at checkpoint X without being current at the server.

### One logical database does not mean lying about acceptance
- `await mutation` may mean **durably committed locally**, with optimistic UI updates.
- `waitForAcceptance({ timeoutMs })` may reject promptly offline or on deadline, without cancelling the pending mutation.
- A terminal server rejection is different from a transient transport failure.
- A late rejection must be observable after component unmount/reload.

### Authentication is not the same as a built-in identity provider
- User supplies authentication integration; server validates credentials and derives identity.
- `tasks.mine()` should not require passing one's own user ID.
- A project ID may be needed to select a resource, but never grants access by itself.
- Query coverage and write permissions are separate checks.

### Scopes are data contracts, not UI pagination
- Watching tasks for a project can require the **whole authorized project dataset**, even when only 20 of 2,000 tasks are rendered.
- Explicit scopes are a promising middle ground between whole-workspace cloning and arbitrary query-driven sync.
- Moves between scopes, overlapping subscriptions, joins, revocation, retention, and query completeness must be defined.
- Automatic scope acquisition should not silently label an incomplete SQL result complete.

### SQL effects versus operation intent
- A raw SQL update on a local subset captures concrete changed rows; it does not automatically rerun its predicate on all server rows.
- A named authoritative operation can intentionally evaluate a condition against current server state.
- If TS mutation functions execute both locally and on the server, define deterministic prediction, versioning, side effects, and auth context; shared types alone do not ensure identical behavior.
- Local SQLite transactions, server acceptance and remote feed application must remain atomic/coherent.

## Current open questions (resolve before implementation hardens APIs)

1. **Function registration:** file-based discovery like Convex, explicit router/registry, or both? How does the user's Node/Bun server start and mount the sync dispatcher without becoming a general web framework?
2. **Client prediction:** how do shared TypeScript handlers run locally without leaking server secrets? Are some functions server-only? How do runtime/auth contexts differ? Should ordinary local SQL effects be an alternative mutation mode?
3. **Schema/typing:** is Drizzle required for schema, optional for queries, or completely optional? How are runtime validators and migrations produced without a duplicate schema?
4. **No codegen:** can inferred types span client/server without bundling server code? How is deployed API compatibility validated? Does any build-time metadata generation become necessary for scope/query analysis?
5. **Selective replication:** explicit scope definitions and automatic acquisition, authorization, complete checkpoints, cross-scope moves, joins, offline retention and revocation. Full-replica v1 vs scoped-v1 remains unresolved.
6. **Execution location:** when does a registered query execute against local SQLite versus the authoritative server? What does an offline query do if required scope was never downloaded?
7. **Mutation meaning:** tracked concrete row changes versus rerunning registered named operations, and whether both are exposed under one calling convention.
8. **Conflict defaults:** same-column collisions, delete/update, uniqueness, dependent rejected work, replay determinism, and server-only policies.
9. **Rust host integration:** native Node/Bun binding choice, connection/thread ownership, embedded protocol host, worker/browser OPFS integration, crash recovery.
10. **Auth and authorization:** how the server's verified identity maps to replica scopes and locally predicted queries without making client-provided identity trusted.
11. **Reactivity:** core subscription/diff contract, query dependency tracking, scope readiness, Solid 2 async adapter and React external store.
12. **Product boundary:** standalone sync library with function dispatcher versus an opinionated full-stack framework. Most recent discussion rejects a general-purpose HTTP framework; don't accidentally build one.

## Research references and projects

- Linear: selective, permission-filtered sync; workspace log, sync groups/subscriptions.
- Notion: SQLite local caching, explicit offline completeness/retention reasons.
- PowerSync: SQLite + Drizzle, Sync Streams, upload queue.
- Zero/Replicache: shared query/function references, mutation replay, query-driven data.
- Convex: function registry, typed query/mutation calls, reactive subscriptions; no user-defined HTTP route per function.
- Dexie Cloud: conditional server-evaluated operations.
- SQLSync/Graft: full-log replication and partial replication tradeoffs.
- LiveStore: event-sourced local SQLite projection.
- Jazz: classic CoValues vs newer relational/Rust direction.
- TanStack DB: reactive collections and incremental query machinery.
- Syncular: scopes and completeness; RxDB, WatermelonDB, Couchbase Lite, Electric, PGlite, Firestore, SQLite AI SQLite Sync as contrasting designs.
- Fregat: Weidner/FugueMax/editor prior art only, no shared core extraction.
- https://mattweidner.com/2025/05/21/text-without-crdts.html
- https://www.sqlite.org/sessionintro.html
- https://sqlite.org/wasm/doc/trunk/persistence.md
- https://docs.convex.dev/functions/overview
- https://docs.powersync.com/client-sdks/orms/js/drizzle

## Next planning action

Before implementation, revise the old native-server/Drizzle-first plan files to align with the newest **embedded Rust core + user's Node/Bun host + Convex-style typed functions** direction. First write a short API/host design RFC and pressure-test it against: authenticated `mine()`, first-time offline scope, local optimistic mutation, server rejection, project-wide operation, same-column conflict, and a reconnect after crash. Record owner decisions rather than silently promoting recommendations into requirements.
