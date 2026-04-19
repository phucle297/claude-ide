---
name: napi-rs-bridge
description: Patterns for the Rust/TypeScript boundary via napi-rs in the Claude Code IDE indexer
type: skill
---

# napi-rs Bridge Patterns

Use when writing or modifying code that crosses the Rust ↔ TypeScript boundary.

## Setup

The Rust indexer compiles to a native `.node` addon loaded by the Extension Host:

```rust
// Cargo.toml
[lib]
crate-type = ["cdylib"]

[dependencies]
napi = { version = "2", features = ["async"] }
napi-derive = "2"
```

```typescript
// TypeScript side
const { IndexerHandle } = require('./indexer.node');
```

## Async pattern (prefer this)

Long-running operations (indexing, search) must be async to avoid blocking the Extension Host event loop:

```rust
#[napi]
pub async fn search(query: String, limit: u32) -> napi::Result<Vec<SearchResult>> {
    tokio::spawn(async move {
        // heavy work here
    }).await.map_err(|e| napi::Error::from_reason(e.to_string()))
}
```

## Data crossing the boundary

- Primitive types (String, u32, bool, f64) cross natively
- Return `Vec<T>` where T is a `#[napi(object)]` struct — becomes a JS array of objects
- Avoid sending large byte arrays back and forth; prefer returning file paths or IDs and letting each side read its own store

## SQLite coordination

Both Rust (rusqlite) and Node.js (better-sqlite3) open the same SQLite file. Rules:
- WAL mode enabled at DB creation time — allows concurrent reads from both sides
- Only one writer at a time; use Rust for write-heavy operations (indexing), Node.js for reads (chat history queries)
- Never open the same connection from multiple threads in Rust — use a connection pool (r2d2 + r2d2_sqlite)

## Error handling

Convert Rust errors to napi errors at the boundary — never panic across the FFI:

```rust
fn my_op() -> napi::Result<String> {
    do_thing().map_err(|e| napi::Error::from_reason(e.to_string()))
}
```

## Build

```bash
cargo build --release          # produces target/release/libindexer.so
npx napi build --release       # copies .node file to the extension output dir
```
