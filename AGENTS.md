# SyncQLight development

Read plans/00-start-here.md and plans/08-decisions-and-sources.md before changing the project. This repository currently contains setup documentation only; there is no engine implementation or test suite yet.

## Established direction

- Implement the shared synchronization core and native server in Rust.
- Use upstream SQLite. Compile its C implementation directly with the browser engine into one WASM binary.
- Treat reactivity, local durability, and atomic transactions as first-class requirements.
- Keep the core framework-independent. Drizzle is an adapter/wrapper, not a fork.
- Use TypeScript for the SDK and optional framework adapters, including Solid 2 and React.
- Start with full authorized workspace/database replication and server-authoritative ordering.
- Build an independent implementation. Study Fregat and sqlite-sync for ideas, but do not extract their core or add them as dependencies.

## Engineering expectations

- Distinguish agreed requirements from proposals and unresolved decisions.
- Do not claim unimplemented APIs, completed tests, or benchmark results.
- Prefer current stable toolchains and dependencies; verify releases and record exact versions for reproducible builds.
- Prove native compilation, integrated browser WASM, and persistent browser storage before expanding the architecture.
- Keep database edits and their pending synchronization records atomic.
- Distinguish local commit, pending upload, remote acceptance, and rejection.
- Publish coherent query results after commit/reconciliation, never intermediate rollback/replay states.
- Test duplicates, disconnects, restarts, rejected transactions, and concurrent edits using separate replicas.
- Do not invent a dedicated product CLI or require a document CRDT for ordinary SQL synchronization.
- Keep secrets, runtime databases, and generated build output out of commits.
- Never force-push, change repository visibility, or modify unrelated projects without explicit authorization.
