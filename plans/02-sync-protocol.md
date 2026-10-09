# Sync protocol and reconciliation

Status: proposed v1 contract, not implemented.

## Identity and authority
One logical database has one authoritative server commit order, history epoch, schema fingerprint and monotonic cursor. Each replica has a durable identity/incarnation; each mutation has a stable (replica, incarnation, sequence) ID. Reactive revisions are not server cursors.

V1 fully replicates authorized application tables for a workspace. Private server tables never enter bootstrap or feeds.

## Local write semantics
Ordinary SQL and Drizzle mutations capture concrete row/column effects, not the original WHERE clause for server reevaluation. Each local transaction is one atomic sync submission. The local SQLite edit, mutation identity, outbox payload, and receipt commit together. A successful await means local durability, not remote acceptance. A lost worker response must be resolved using a stable local request receipt rather than rerunning SQL blindly.

Initial bootstrap gates data-dependent writes until a complete consistent snapshot exists. After bootstrap, offline reads and writes are allowed.

## Proposed conflict defaults
- Different columns: merge patches if constraints and policy permit.
- Same scalar column: later accepted server transaction wins.
- Update missing/deleted row: reject without resurrection.
- Delete already missing row: authorized idempotent no-op.
- Insert with colliding key: reject, except identical mutation retry.
- Unique/FK/CHECK or policy violation: reject entire transaction.
- Rejected pending edit: preserve it and conservatively block potentially dependent later edits.
- Same mutation ID, same digest: return original durable outcome; different digest: protocol error.

These are proposals requiring executable tests, not established owner decisions. Ordinary row effects do not preserve arbitrary business intent; later named authoritative actions can express increments or server-evaluated predicates.

## Server acceptance
Authenticate and authorize each submission, validate typed bounded changes and current rows, serialize per-database commits. Atomically apply canonical effects, append a feed entry, allocate a cursor, and persist a deduplication receipt. Reply accepted only after successful commit. Semantic rejection has a durable terminal receipt; temporary infrastructure errors remain retryable. Commit-before-reply failure is safe to retry using the same identity.

## Reconciliation
Invariant: visible state = accepted state at local cursor + successfully replayed pending edits.

In one hidden local transaction: validate epoch/schema/feed continuity; restore accepted base; apply contiguous canonical feed transactions; incorporate terminal outcomes; retain optimistic edits until accepted effects appear in the feed; replay pending work in local order; preserve blocked work; atomically persist visible tables, accepted base, cursor, receipts and outbox transitions. Publish one coherent reactive update only after commit. Never recapture replay as a new mutation.

The accepted-base representation and undo mechanism are open engineering decisions. Full snapshots may serve as a reference model, but copying an entire SQLite database per mutation is not the production design.

## Bootstrap, retention and safety
Install a consistent authorized application snapshot at a recorded cursor; catch up from there. Expired cursors trigger explicit resnapshot with pending work preserved. Backup rollback rotates history epoch. Schema mismatch pauses incompatible writes. Retention must not let a very old duplicate mutation become a fresh submission. Permission revocation stops future service, but cannot erase offline bytes.

Support stable non-null primary keys, including composite, and lossless NULL/BLOB/i64/float/text transport. Prove behavior for defaults, cascades, triggers and constraint ordering. V1 excludes client DDL, arbitrary extension loading, attached databases, virtual-table replication, and primary-key mutation.

See 06-verification.md for required tests.
