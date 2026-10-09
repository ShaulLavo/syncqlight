# Verification matrix

Status: planned tests, not executed results.

## Reference model
Duplicate ID and payload mismatch; same/different column conflicts; update after delete; missing row; rejected dependent suffix; server ordering; acknowledgment before feed; feed gap; expired cursor; schema mismatch; offline restart.

## SQLite native/browser
NULL, BLOB, i64 extremes, composite keys, savepoints, rollback, deferred constraints, cascades, trigger semantics, capture exclusion of internal tables, OPFS persistence, worker death, multi-tab ownership, initial query/subscription race, filtered/joined query invalidation, coherent revisions.

## Crash injection
Before/after local commit, outbox write, worker response, network send, authoritative commit, server response, feed application, cursor update, and resnapshot installation. Verify no lost committed edit, duplicated accepted effect, or mismatched cursor.

## Security
Forged actor, invalid token, cross-database access, path traversal, unauthorized columns, internal tables, malformed/oversized payloads, policy rejection, revoked membership, replayed stale IDs.

## E2E
Use two separate browser profiles with independent OPFS databases, not two tabs sharing one worker. Disconnect both, mutate, close, reopen, reconnect; verify convergence and durable receipts. Also test two isolated logical server databases.

## Performance
Measure local write, notification, WASM boundary, worker messages, sync lag, bootstrap size, memory, outbox/feed growth. Record machine, OS, browser, compiler, SQLite version, workload, warmups, samples and percentiles. Never invent results.

For every milestone save exact commands, test results, failing/skipped cases, source commit and limitations.
