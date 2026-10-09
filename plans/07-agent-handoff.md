# Agent handoff

Read 00-start-here.md and 08-decisions-and-sources.md first, then the other plans. Work only in /work/projects/syncqlight, the Work-drive Git repository with origin https://github.com/ShaulLavo/syncqlight.git. Inspect git status and remote before edits. Do not reset, force push, change visibility, or modify unrelated repositories.

Implement SyncQLight independently in Rust with upstream SQLite compiled directly into the integrated browser WASM module and native server. TypeScript is for SDK and adapters. Fregat and sqlite-sync are reference material only; do not extract or depend on their code.

Begin with M0, then M1. Build real compiling code and tests in small reviewable increments. Pin toolchains, verify unfamiliar Rust/SQLite APIs against source, isolate unsafe FFI, run formatting and tests after each change. Do not replace the integrated WASM goal with a separate SQLite JS package without explicitly documenting a blocker and requesting architectural review.

The first demonstrable milestone is native and browser Rust+SQLite execution, persistent OPFS, rollback-safe live query, and a no-gap subscription. The next is two separate persistent replicas synchronizing through the authoritative server.

Treat conflict matrix, runtime framework, capture mechanism, browser VFS, acknowledged-base representation and higher-level server-function runtime as open until their specified tests/decision gates. Do not present examples as shipped APIs or claim tests ran unless executed.

For each PR: summarize design, files changed, exact test commands and results, benchmark methodology if any, unsupported cases, and unresolved questions. Keep README absent unless explicitly requested.
