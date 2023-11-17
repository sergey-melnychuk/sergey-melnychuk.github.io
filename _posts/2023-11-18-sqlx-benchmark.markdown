---
layout: post
title:  "Benchmarking sqlx: postgres vs sqlite"
date:   2023-11-17 22:32:42 +0200
categories: rust sqlx postgres sqlite
---

I was wondering if [sqlx](https://github.com/launchbadge/sqlx) is a good choice when similar workflow needs to be run against an embedded database (sqlite) and a "remote" one (postgres, running on the same host or on a neighbour node). Surprisingly popular combo recently.

Workflow:
- merkle tree updates
  - keys & values: 32 bytes long
  - hash function: keccak
  - `N` entries (rows) in the tree
  - `K` inserts + updates combined
- each update hits `log2(N) * 2` rows
  - in a single transaction

Get a grasp of a performance difference:
- sqlx + sqlite vs vanilla rusqlite
- sqlx + sqlite vs sqlx + postgres

Plan:
- tree table
  - columns: 
    - id
    - parent_id
    - is_left
    - key (optional - present only in leaves)
    - hash
      - node: `keccak(keccak(L subtree) || keccak(R subtree))`
      - leaf: `keccak(key || value)`
  - indices:
    - (id, parent_id): btree
    - key: btree
- merkle tree impl
  - mutable in-memory
    - insert: tree re-alignment + traversal
    - update: tree traversal
  - return list of updated entries on each update
    - `add(key, val) -> [...]`
  - framework-agnostic transaction abstraction
  - unit-testing
- data generation
  - random yet deterministic
- measure
  - insert initial state
  - perform updates

References:
- [15k inserts/s with Rust and SQLite](https://kerkour.com/high-performance-rust-with-sqlite)
- [rust-tide-sqlx-crud](https://github.com/sergey-melnychuk/rust-tide-sqlx-crud)
- [diesel metrics](https://github.com/diesel-rs/metrics/#inserts)
