# 0020. Atomic Aggregate Update Port and Targeted Writes on Aurora DSQL

* **Status**: Proposed
* **Date**: 2026-11-24
* **Deciders**: HomeHarmony Engineering Team

## Context and Problem Statement

Sprint 2's initial `FamilyRepository` port exposed separate `find_by_id` (committed in `BEGIN READ ONLY`) and `save(&Family)` (full-aggregate `INSERT ... ON CONFLICT DO UPDATE` inside `retry_on_occ`) operations. Under concurrent requests on Amazon Aurora DSQL (snapshot isolation + Optimistic Concurrency Control), this split pattern exhibits four defects:

1. **OCC retry replays stale snapshots (lost updates / member resurrection)**: On `SQLSTATE 40001` / `OC000` / `OC001`, `retry_on_occ` re-executes `save` with the pre-conflict `PreparedFamilySave` without re-reading database state or re-running domain checks, silently overwriting concurrent profile edits or resetting a soft-deleted member's `deleted_at` back to `NULL`.
2. **One-time invite token double-spend**: Two concurrent `POST /families/{id}/members` calls using the same token can both read `used = false` before either commits, and unconditional `ON CONFLICT (token) DO UPDATE SET used = EXCLUDED.used` allows both to succeed.
3. **Unmapped duplicate membership violation**: Two concurrent joins by the same `user_id` generate distinct `MemberId` PKs and hit `UNIQUE (family_id, user_id)` (`SQLSTATE 23505`), surfacing as `500 INTERNAL_ERROR` instead of `409 MEMBER_ALREADY_EXISTS`.
4. **Unbounded historical loading & 500-row write bricking**: Loading all historical `family_invite_tokens` on every `find_by_id` and unconditionally re-upserting all rows fails permanently once a family accumulates `> 500` cumulative members and historical tokens, and causes N+1 write round-trips per mutation.

## Decision Drivers

* Preserve pure domain aggregates (`domain/`) with zero SQL/I/O dependencies (`ADR-0003`)
* Guarantee serializable per-family invariants under Aurora DSQL OCC retries (`40001`/`OC000`/`OC001`)
* Keep write transactions bounded to O(changed rows) without schema changes or artificial history truncation
* Maintain RPITIT static-dispatch ports (`Send + Sync`, no unnecessary `'static`)

## Considered Options

1. **Option 1: Atomic `update` + `insert_new` repository port with `SELECT ... FOR UPDATE` and snapshot-diff writes**
2. **Option 2: Add an optimistic `version BIGINT` column to `family_families` and return `409` on conflict**
3. **Option 3: Continue full-aggregate `save(&Family)` with manual retry loops in every HTTP handler**

## Decision Outcome

Chosen option: **"Option 1: Atomic `update` + `insert_new` repository port with `SELECT ... FOR UPDATE` and snapshot-diff writes"**, because it solves all four concurrency and scalability issues structurally inside the persistence adapter without adding schema migrations or leaking SQL/retry loops into handlers or `domain/`.

* `FamilyRepository::find_by_id` becomes read-only without historical invite tokens (`Vec::new()`).
* `FamilyRepository::insert_new(&Family)` handles aggregate creation (`POST /families`) using plain `INSERT`s.
* `FamilyRepository::update<T, F>(family_id, TokenScope, op)` executes `BEGIN` → `SELECT ... FROM family_families WHERE id = $1 FOR UPDATE` → load members + scoped token (`TokenScope::None` or `TokenScope::One(token)`) → capture `Snapshot` (`BTreeMap`) → run `op(&mut family)` → diff `before`/`after` snapshots → write only changed rows (with `UPDATE family_invite_tokens SET used = $3 WHERE token = $1 AND family_id = $2 AND used = false` conditional guard, tenant `WHERE family_id` guards, and `23505` mapped to `MemberAlreadyExists`) → `COMMIT`, all inside a single `retry_on_occ` loop.

### Positive Consequences

* OCC conflicts automatically re-read the latest committed family state and re-run pure domain invariants.
* Per-mutation SQL writes drop from O(members + all historical tokens) to O(changed rows, typically 1–2 rows), eliminating the 500-row bricking cliff.
* Domain errors returned by `op` roll back cleanly and are never retried.
* EventBridge events are returned from the committed attempt only and published best-effort after commit (`ADR-0017`).

### Negative Consequences / Trade-offs

* `op` closures passed to `update` may execute more than once during OCC retries and must remain pure with respect to external state (requiring `.clone()` of captured inputs inside the closure).
* Callers that inspect historical invite tokens outside `accept_invite` cannot rely on `find_by_id` loading them.

## Compliance & Invariants

* Enforced via compile-time RPITIT port signatures in `backend/domain/src/ports.rs`, snapshot-diff persistence in `backend/infrastructure/src/repositories/{family_repository,family_writes}.rs`, and Layer 2a PostgreSQL 16 concurrency tests in `backend/infrastructure/tests/family_repository_pg.rs`.
