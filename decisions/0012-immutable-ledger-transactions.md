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
1. The user invokes `POST /transactions/{id}/amend` with the revised payload.
2. The domain creates two new linked immutable records in a single unit of work:
   * **Reversal Entry**: An exact mirror of the original transaction with `kind = 'reversal'`, which offsets the financial impact of the original. It sets `amendment_of_id = original.id`.
   * **Corrected Entry**: A new transaction containing the corrected amounts, categories, or accounts, also setting `amendment_of_id = original.id`.
3. The original transaction is marked with `deleted_at = now()` (soft-delete) to hide it from standard UI transaction feeds while preserving it in historical audit logs.

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
