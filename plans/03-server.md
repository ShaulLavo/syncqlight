# Native Rust server

Status: proposed architecture, not implemented.

A native service manages multiple independent SQLite databases. One authoritative write order per database; separate databases progress independently. No cross-database atomicity or multi-primary consensus in v1.

Tokio plus Axum is the current recommendation, not an owner-locked requirement. Synchronous SQLite execution must not block async executor workers. Use bounded connection-owner workers and explicit transaction ownership.

## Server endpoints
- Handshake: protocol/schema/history negotiation and replica registration.
- Bootstrap: consistent authorized application snapshot and cursor.
- Push: one bounded atomic mutation, stable identity and digest.
- Pull: ordered canonical transactions after a cursor, optionally long-polling.
- Outcome lookup: recover lost mutation replies.
- Checkpoint: finite catch-up target.

Start with HTTP and bounded long polling. WebSockets may later change transport, not state semantics.

## Security
Map logical database IDs through a server-owned registry, never raw caller-provided file paths. Verify credentials and database membership for every operation, including continuing streams. Use injected auth and write-policy interfaces with deny-by-default production behavior. Never expose raw arbitrary SQL execution or private sync metadata to clients.

Server-generated writes must pass through tracked transactions. In one SQLite transaction, apply canonical effects and store feed entry plus retry receipt. Retried IDs with different payloads are errors. Limit request sizes, queue lengths, snapshot lifetimes, and per-principal rates. Keep policy decisions free of external side effects inside SQLite transactions.

## Operations
Ship a reproducible native binary/container with persistent data directory, health/readiness, bounded shutdown, structured logs and metrics. Use SQLite-consistent backups and test restoration with history epoch rotation. Integrate with existing deployment tooling through standard deployment contracts. No dedicated CLI product is required.

## Gate
Two browser replicas converge through this server; two databases remain isolated. Auth failures, rejected transactions, duplicate retries, server restart, lost commit reply, schema mismatch, and history restore have tested outcomes.
