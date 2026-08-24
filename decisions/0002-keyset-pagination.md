# 0002. Enforce Keyset (Cursor-Based) Pagination with UUID v7

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
In distributed relational databases like Aurora DSQL, queries using `OFFSET N` perform full index scans to discard $N$ rows. As users navigate deeper pages, DPU consumption and query latency scale linearly with page depth, resulting in high cloud costs and degraded user experience.

## Decision Drivers
* DPU efficiency and cost minimization in Aurora DSQL.
* Consistent $O(1)$ query execution time regardless of page depth.
* Preventing phantom/skipped items during real-time transaction insertions.

## Considered Options
1. **Keyset (Cursor-Based) Pagination** using `(occurred_at, id) < ($cursor_time, $cursor_id)`
2. **Offset / Limit Pagination** (`OFFSET 50 LIMIT 50`)

## Decision Outcome
Chosen option: **Keyset (Cursor-Based) Pagination**, because it executes via direct index seeks on composite indexes, eliminating table scans and deep pagination performance degradation.

### Positive Consequences
* Stable $O(1)$ query latency across all page depths.
* Immune to pagination drift when new transactions are inserted concurrently.
* Supported by time-sortable **UUID v7** identifiers which provide deterministic ordering tie-breakers.

### Negative Consequences / Trade-offs
* Random access to arbitrary page numbers (e.g. "Jump to Page 47") is not supported; pagination is sequential (Next / Previous cursor).
* Requires clients to handle opaque base64-encoded cursor tokens.

## Compliance & Invariants
* `OFFSET` is explicitly banned in all SQL queries and API contracts.
* All list queries must accept `cursor: Option<String>` and return `next_cursor: Option<String>`.
* Composite index `(family_id, occurred_at, id)` created on transaction tables.

