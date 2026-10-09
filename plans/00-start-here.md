# SyncQLight implementation plan

Status: planned; no engine implementation is claimed by these documents.
Updated: 2026-10-09.
Repository: https://github.com/ShaulLavo/syncqlight
Canonical working directory: `/work/projects/syncqlight` on the Linux Work drive.

## Product

SyncQLight makes ordinary SQLite tables local, durable, reactive, and automatically synchronized with an authoritative server. Developers use a TypeScript SDK and an upstream Drizzle integration, with optional simpler typed APIs. They should not have to build an upload service, outbox, subscription system, or reconciliation loop for normal CRUD.

SQLite is the application data model. This is not a document database disguised as SQL, a new ORM, a general signals framework, or a command-line product.

## Established direction

- A new, public repository: `ShaulLavo/syncqlight`.
- Rust for the shared engine and native server; TypeScript for the browser host, SDK, and framework adapters. The earlier Zig plan is superseded.
- Upstream SQLite C compiled directly with the browser Rust engine into one WASM binary, and linked natively on the server. Neither libSQL nor Turso is the storage dependency.
- First-class reactive queries, local transaction atomicity, durable offline work, and server-ordered reconciliation.
- Multiple independently addressable databases. Start with a complete authorized workspace/database replica, not mandatory query-driven partial replication.
- Ordinary Drizzle schemas and query builders through our driver/wrapper, with `watch()` supplied by SyncQLight. Do not fork Drizzle.
- Framework-independent core with an idiomatic Solid 2 adapter, plus React and vanilla consumers.
- Independent implementation. Fregat and sqlite-sync are optional reading material, not dependencies or extraction targets.
- A polished database-backed Game of Life showcase; optional size and diagnostic controls, not a stress-test dashboard.

This plan supersedes the earlier chat attachment named `sqlite-sync-build-plan.md`. Do not import its Zig choice, old repository destination, or extraction/migration assumptions into this project.

## Read order

| File | Purpose |
|---|---|
| [01-architecture.md](01-architecture.md) | Rust boundaries, direct SQLite/WASM build, storage, ownership, and capture. |
| [02-sync-protocol.md](02-sync-protocol.md) | Mutation meaning, proposed conflict rules, outbox, ordering, reconciliation, and recovery. |
| [03-server.md](03-server.md) | Native service, many databases, authorization, acceptance, transport, and operations. |
| [04-api-and-reactivity.md](04-api-and-reactivity.md) | Concrete proposed SDK/Drizzle APIs, watched queries, transactions, and adapters. |
| [05-delivery-plan.md](05-delivery-plan.md) | Dependency-ordered milestones, work items, exit gates, and implementation sequence. |
| [06-verification.md](06-verification.md) | Executable contract, fault matrix, real-browser tests, and performance measurements. |
| [07-agent-handoff.md](07-agent-handoff.md) | Instructions for the implementation agent and exact first deliverable. |
| [08-decisions-and-sources.md](08-decisions-and-sources.md) | Agreed versus proposed choices, bounded open decisions, authoritative references. |

## What is not settled

Server runtime, exact browser C runtime/VFS integration, changeset representation, acknowledged-base storage, conflict defaults, business-trigger policy, and application-function runtime are implementation decisions with explicit gates. Tokio is a recommendation, not an owner-approved requirement. Server authority is established; every conflict policy is not.

API examples are intended contracts, not released APIs. Package names are proposed and have not been reserved in registries. Read the decision register before treating a proposal as an agreement.

## First demonstrable result

One shared Rust/SQLite core running natively and in one browser WASM module. A real OPFS-backed transaction survives close/reopen, a failed transaction stays invisible to live queries, and an initial subscription cannot miss a commit. Then connect two genuinely separate persistent replicas to the native server and test offline edits, restart, retry, rejection, and convergence.

Do not spend the first implementation phase publishing empty crates or building the website. A workspace skeleton is useful only as part of the compiling, tested vertical slice.

## Completion tracking

All implementation milestones currently start as **not started**. The presence of these plans is not implementation progress. Each milestone must record its commit, commands actually run, test results, supported targets, and remaining limitations before its gate is marked complete.
