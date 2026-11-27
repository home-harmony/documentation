# 0012. Ledger Transactions Are Immutable — Corrections Via Reversal Pattern

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

The `ledger_transactions` table serves as the authoritative source of truth for all household financial movements. In previous iterations, generic CRUD semantics (including `PUT /transactions/{id}` and direct column updates) were considered for user convenience.

However, mutable ledger entries violate fundamental financial accounting principles:
1. **Audit Trail Destruction**: Updating amounts, dates, or accounts in place destroys historical auditability.
2. **CDC Inconsistency**: Change Data Capture (CDC) streams rely on immutable row histories. Hard-deletes and in-place updates create race conditions in asynchronous balance snapshotting and budget recalculations.
3. **Double-Entry Discipline**: Financial discrepancies must be corrected by explicit offsetting entries rather than retroactively rewriting historical ledger states.

## Decision Drivers

* **Financial Integrity & Double-Entry Alignment** — every financial correction must leave an immutable, traceable audit trail.
* **CDC & Stream Reliability** — downstream consumers (budget aggregates, account snapshots) must receive deterministic change streams without ambiguous in-place updates.
* **Simplicity of Concurrency Control** — append-only transactions eliminate OCC update-conflicts on historical transaction records.

## Considered Options

1. **In-Place Updates (`UPDATE ledger_transactions SET ...`)** — rejected due to audit loss and CDC race hazards.
2. **Physical Delete and Re-Insert** — rejected because Aurora DSQL CDC drops prior row data on physical deletes and causes gap anomalies.
3. **Accounting Reversal & Amendment Pattern** — chosen.

## Decision Outcome

Chosen option: **Option 3 — Accounting Reversal & Amendment Pattern**.

Transactions in `ledger_transactions` are strictly append-only and immutable. Once created, a transaction cannot be updated or hard-deleted.

### Correction Workflow (`POST /transactions/{id}/amend`)

To correct an inaccurate transaction:
1. The user invokes `POST /transactions/{id}/amend` with the revised payload and a client `idempotency_key` (stored on the **Corrected Entry**; the **Reversal Entry** receives a deterministic SHA-256 companion UUID derived from `(family_id, idempotency_key)` so `UNIQUE (family_id, idempotency_key)` is satisfied cleanly).
2. The domain creates two new linked immutable records in a single atomic OCC unit of work:
   * **Reversal Entry**: An exact accounting negation of the original transaction with `kind = 'reversal'` and `amendment_of_id = original.id`. It swaps `source_account_id` ↔ `destination_account_id` (and, for cross-currency transfers/withdrawals where `destination_amount` is present, also swaps `amount` ↔ `destination_amount` so each leg stays in its account's native currency).
   * **Corrected Entry**: A new transaction (with `TransactionId` UUID v7 ordered strictly after the `Reversal` entry) containing the corrected amounts, categories, or accounts, also setting `amendment_of_id = original.id`.
3. **Preserving `original.deleted_at = NULL` for Snapshot Balance Integrity (`ADR-0006`)**: Amending an original transaction intentionally keeps `original.deleted_at = NULL` (reserving `deleted_at = now()` strictly for `DELETE /transactions/{id}`). Because `original + Reversal == 0`, same-month balance sums cancel out without double-negation, **and** amending a transaction from a prior closed month naturally applies the net adjustment after that month's `AccountBalanceSnapshot` without rewriting historical snapshots. Default list views (`GET /transactions`) exclude superseded originals (any row referenced by an active `Reversal`'s `amendment_of_id`) and `Reversal` rows unless `?include_audit=true` is requested.

### Schema Support in `ledger_transactions`

```sql
-- Schema adjustments for ledger_transactions
ALTER TABLE ledger_transactions 
  ADD COLUMN amendment_of_id UUID,
  ADD COLUMN destination_amount_value NUMERIC(19,4),
  ADD COLUMN destination_amount_currency CHAR(3);

-- Updated kind check
ALTER TABLE ledger_transactions 
  DROP CONSTRAINT IF EXISTS ledger_transactions_kind_check,
  ADD CONSTRAINT ledger_transactions_kind_check 
    CHECK (kind IN ('income', 'expense', 'transfer', 'cash_withdrawal', 'loan_payment', 'reversal'));
```

### Positive Consequences

* Unbroken financial audit trail conforming to accounting standards.
* Perfect CDC stream characteristics — consumers process clean inserts without mutating past states.
* Reversals are self-documenting and auditable by any family member with appropriate permissions.

### Negative Consequences / Trade-offs

* Amending a transaction creates two new rows in the database, slightly increasing storage footprint.
* UI list views must filter out reversed and reversal entries by default, showing only active or amended representations unless "Show Audit History" is enabled.

## Compliance & Invariants

* `PUT /transactions/{id}` **is strictly prohibited** and must return `405 Method Not Allowed`.
* `DELETE /transactions/{id}` **must only perform soft-delete** (`deleted_at = now()`) and must emit a `TransactionDeleted` domain event that triggers balance recalculation.
* Amending a transaction **must** generate a `reversal` kind record with `amendment_of_id` populated.
